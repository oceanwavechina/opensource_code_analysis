# epoll 的内部实现

关于 epoll 的水平模式，边缘模式演示的具体例子，[看这里](https://gitee.com/oceanwave/linux_playground/blob/master/linux_api/epoll_lt_or_et.cpp)。

先来看一个 epoll 的简单用法：

``` cpp
int main(int argc, char **argv) {

	struct epoll_event ev, events[5];
	
    // 创建 epoll 文件
    int epfd = epoll_create(1);
	
    // 设置要监控的fd， 以及监听事件和模式
    ev.data.fd = STDIN_FILENO;
	ev.events = EPOLLIN|EPOLLET;	//监听读状态同时设置ET模式

    // 也就是说在 epf 上 ADD (EPOLL_CTL_ADD) 一个需要监控的 fd (STDIN_FILENO)
    // 具体监听的触发事件为 ev (EPOLLIN)
	epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &ev);

    // 轮询等待，直到被事件唤醒或超时
	for(;;) {
		int nfds = epoll_wait(epfd, events, 5, -1);
		for(int i=0; i<nfds; ++i) {
			if(events[i].data.fd == STDIN_FILENO) {
				printf("welcome from: %s\n", __FUNCTION__);
			}
		}
	}
}
```

可以看到，epoll 和 select 在用法上的区别是， epoll只需要添加一次要监听的文件fd， 而select每次轮询都需要重新添加。而且epoll_ctl 添加fd 的个数也没有 1024 的限制了。

<br>

## 1. epoll_create：创建资源和初始化
----

<br>

因为 epoll 也是文件类型，所以它的实现在 ```linux/fs``` 目录下边。其调用栈如下；

``` cpp
SYSCALL_DEFINE1(epoll_create, int, size)
    |
    | -> do_epoll_create()
```

具体创建 epoll 文件是在 ```do_epoll_create()``` 里边， 一下是简化版，只包含了主体流程：

``` cpp
static int do_epoll_create(int flags)
{
	int error, fd;
	struct eventpoll *ep = NULL;
	struct file *file;
    
    ......

	// 分配 struct eventpoll 空间，这个是 epoll 的核心数据结构
	error = ep_alloc(&ep);
	if (error < 0)
		return error;

    // 分配 fd， 并根据 struct eventpoll 创建 inode 信息
	fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
    ......

    // 这里会把 struct eventpoll *ep 对象存储到 file 里边
	file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep, O_RDWR | (flags & O_CLOEXEC));
    ......
	fd_install(fd, file);
	return fd;
}
```

我们要重点看一下 ```struct eventpoll``` 这个数据结构：

``` cpp
struct eventpoll {

    // 当进程/线程 (在 linux 调度管理中都是以 task 的结构存储的) 在调用 epoll_wait() 这个方法的时候
    // 如果 epoll 当时没有 ready 的事件，就会把当前 task (current) 添加到这个 wq 变量中
    // 当有事件发生的时候，会唤醒 wq 中的 task。(其实这里会有一个惊群的问题。下面在看)
    // 为什么会有多个 task 调用这个 epoll_wait？
    //      比如，一个进程，fork 了多个子进程，同时等待事件的发生 (如 nginx 中的 worker)
    //      就会有多个 task，
	wait_queue_head_t wq;

    // epoll 作为一个文件设备(publisher)，也需要给其他监听 当前epoll 对象的实体提供 notify 机制。
    // 这个 poll_wait 就是通知其他 subscriber 状态变化的队列
	wait_queue_head_t poll_wait;

	// 这里存的是所有已经触发的事件
	struct list_head rdllist;

	// 存储的是所有需要监听的 fd
    // 其中 RB tree 的 node 类型是 struct epitem， 
	struct rb_root_cached rbr;

	/*
	 * This is a single linked list that chains all the "struct epitem" that
	 * happened while transferring ready events to userspace w/out
	 * holding ->wq.lock.
	 */
	struct epitem *ovflist;

	/* wakeup_source used when ep_scan_ready_list is running */
	struct wakeup_source *ws;

    // 因为 epoll 本身就是一个文件， 也就是在 create 的时候创建的文件
    // 而 该成员(file) 的 private_data 就是当前这个 struct eventpoll
    // 相当于是指向父对象的的一个指针
    // 其关联是通过 anon_inode_getfile 调用实现的，
    /// 而 anon_inode_getfile 的实现在fs/anon_inodes.c:70， 作用是创建一个匿名inode (anonymous inode)
	struct file *file;
};
```

有几个对象的关系这里先梳理下：
1. 当执行 epoll_create 后，会返回一个fd，也就是对应操作系统的一个文件 ```struct file* file```

2. 作为 epoll 的核心对象 ```struct eventpoll```, 会以 private_data 的成员存储在 file 对象里边
   
   即 ```file->private_data = priv;``` (fs/anon_inode.c:70 anon_inode_getfile())

3. 所以当给定 epoll fd 后其查找流程为：“”

    - 先根据 ```fd``` 找到 ```struct file* file``` 对象
    - 根据 ```struct file* file``` 对象找到 ```struct eventpoll```, 即 ```file->private_data```

4. 而我们的监听列表等信息，都存储在 ```struct eventpoll``` 里边。


<br>

## 2. epoll_ctl: 注册事件
----

<br>

这个函数是跟 select 不一样的地方。在 select 机制中，每次调用 select 其实是同时执行了3个操作：

1. 添加要监听的 fd （用户态到内核态的数据拷贝）
2. 轮询所有被监听的 fd 的事件
3. 把有事件发生的fd返回 (内核态到用户态的数据拷贝)

每次全量添加 fd， 也就成了影响 select 性能的一个地方。所以在 epoll 中把以上这两个操作拆开成了两个独立的步骤。所以在 wait 时操作也就减少了。

这里主要说一下 insert 操作:

1. 首先，添加的需要监听的 fd， 会分配一个 ```struct epitem``` 结构。
   
    这个接口保存了事件的所有信息。epoll 实现中每个被监听的 fd 都对应一个 ```struct epitem```。

2. 把 epitem 添加的红黑树里边。

3. 把当前 task 添加到监听文件的唤醒列表里边

如下是 ep_insert 的实现：

``` cpp
static int ep_insert(struct eventpoll *ep, const struct epoll_event *event,
		     struct file *tfile, int fd, int full_check)
{

	// 红黑树里边是所有需要监听的fd
    // 所以要把新加的fd 放到 红黑树里边
	ep_rbtree_insert(ep, epi);


	// 这里是初始化回调函数，也即是当设备有事件发生的时候会调用这个函数
    // 这个回调函数也就是把我们当前 task 插入到 设备等待队列的关键
    // 回调函数的具体内容我们下边看
	epq.epi = epi;
	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

    //  这里会主动 poll 一下设备
    //  在poll 的过程中会 调用我们刚刚设置的回调函数 ep_ptable_queue_proc
    //  这个回调函数就会把当前 task 插入到 设备的等待队列中
	revents = ep_item_poll(epi, &epq.pt, 1);
}
```

那个关键的回调函数实现如下，看函数注释就很清楚了：

``` cpp
/*
 * This is the callback that is used to add our wait queue to the
 * target file wakeup lists.
 */
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
				 poll_table *pt)
{

    /* 这里有几个参数需要注意下：
        whead：
            这个是设备内部的链表，也就是当设备有事件发生时，会唤醒这个链表上的task
    */

	struct epitem *epi = ep_item_from_epqueue(pt);
	struct eppoll_entry *pwq;

	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
        // 这里还有个回调函数，这个是当有事件时，设备会调用这个函数
		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
		pwq->whead = whead;
		pwq->base = epi;

        // 下边的 add_wait_queue* 就是把当前 eppoll_entry，添加到设备的等待队列中
		if (epi->event.events & EPOLLEXCLUSIVE)
			add_wait_queue_exclusive(whead, &pwq->wait);
		else
			add_wait_queue(whead, &pwq->wait);
		list_add_tail(&pwq->llink, &epi->pwqlist);
		epi->nwait++;
	} else {
		/* We have to signal that an error occurred */
		epi->nwait = -1;
	}
}
```

关于如何注册当前task (subscriber) 到 设备(publisher) 其调用栈如下

``` cpp
ep_item_poll()
    |
    |-> poll_wait()
        |
        |-> p->_qproc(filp, wait_address, p);
            // 即 ep_ptable_queue_proc()  eventpoll.c:1236

```


<br>

## 3. epoll_wait: 获取所有 ready 的 fd
----

<br>

其调用栈如下：

``` cpp
SYSCALL_DEFINE4(epoll_wait)
    |
    | -> do_epoll_wait()
            |
            | -> do_epoll_wait()
                    |
                    | -> ep_poll()
```

<br>

ep_poll 的主题逻辑如下：

可以看到 每次调用 eppoll_wait 拿掉了从用户空间到内核空间的数据拷贝，同时返回的时候只返回已经触发的事件

``` cpp
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, long timeout)
{
	int res = 0, eavail, timed_out = 0;
	u64 slack = 0;
	wait_queue_entry_t wait;
	ktime_t expires, *to = NULL;

	lockdep_assert_irqs_enabled();

    // 1. 计算超时时间，
    // 如果超时等于0，则不阻塞，直接判断 ready list 有没有事件并返回
	if (timeout > 0) {
		struct timespec64 end_time = ep_set_mstimeout(timeout);

		slack = select_estimate_accuracy(&end_time);
		to = &expires;
		*to = timespec64_to_ktime(end_time);
	} else if (timeout == 0) {

		timed_out = 1;
		spin_lock_irq(&ep->wq.lock);
		goto check_events;
	}

fetch_events:

	if (!ep_events_available(ep))
		ep_busy_loop(ep, timed_out);

	spin_lock_irq(&ep->wq.lock);

    // 如果没有事件发生那我们需要阻塞等待了。
	if (!ep_events_available(ep)) {
		ep_reset_busy_poll_napi_id(ep);

		// 把当前 task 添加到等待队列中。
        // current 是获取到当前 task (进程/线程)
        // 这样，当有事件触发时，epoll就可以通知我们(task)
        // linux 下调度的资源都是以 task 为单位的，每个线程都有一个task 对象
        //      只不过 这个 task 对象里边的pid，和资源是一样的。
		init_waitqueue_entry(&wait, current);
		__add_wait_queue_exclusive(&ep->wq, &wait);

        // 轮询, 跳出条件是：
        // 1. 被信号中断
        // 2. 有事件触发了，或是超时了
		for (;;) {
			/*
			 * We don't want to sleep if the ep_poll_callback() sends us
			 * a wakeup in between. That's why we set the task state
			 * to TASK_INTERRUPTIBLE before doing the checks.
			 */
			set_current_state(TASK_INTERRUPTIBLE);
			/*
			 * Always short-circuit for fatal signals to allow
			 * threads to make a timely exit without the chance of
			 * finding more events available and fetching
			 * repeatedly.
			 */
			if (fatal_signal_pending(current)) {
				res = -EINTR;
				break;
			}
			if (ep_events_available(ep) || timed_out)
				break;
			if (signal_pending(current)) {
				res = -EINTR;
				break;
			}

            // 这里程序进入休眠，等待信号或是时间的发生
            // 等醒来就是超时了，返回当前的状态
			spin_unlock_irq(&ep->wq.lock);
			if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
				timed_out = 1;

			spin_lock_irq(&ep->wq.lock);
		}

		__remove_wait_queue(&ep->wq, &wait);
		__set_current_state(TASK_RUNNING);
	}
check_events:
	/* Is it worth to try to dig for events ? */
	eavail = ep_events_available(ep);

	spin_unlock_irq(&ep->wq.lock);

	/*
	 * Try to transfer events to user space. In case we get 0 events and
	 * there's still timeout left over, we go trying again in search of
	 * more luck.
	 */
     // 这里是把触发的事件拷贝到用户空间
     // 跟 select 不一样的是，只会拷贝已经触发的事件，而不是所有设置过的 fd
     // 需要注意的是这个函数对 LT ET 模式有很大影响
     // 这个函数的分析，我们留在下一小节说
	if (!res && eavail &&
	    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
		goto fetch_events;

	return res;
}
```

需要注意，epoll 检查有没有感兴趣的事件触发 跟 select 不同： 

&nbsp; &nbsp; &nbsp; &nbsp; epoll 没有轮询所有 fd 。它判断有没有时间发生的逻辑是检查 ready list 是否为空, 

&nbsp; &nbsp; &nbsp; &nbsp; 但是获取 ready list 中具体的事件则是 poll 每个 fd 设备

如下：

``` cpp

// 检查有没有事件发生
static inline int ep_events_available(struct eventpoll *ep)
{
	return !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;
}

// 挨个 poll 有事件的设备
static int ep_send_events(struct eventpoll *ep,
			  struct epoll_event __user *events, int maxevents)
{
	struct ep_send_events_data esed;

	esed.maxevents = maxevents;
	esed.events = events;

    // 遍历 ready list, 对所有有事件发生的fd 执行 poll 操作
	ep_scan_ready_list(ep, ep_send_events_proc, &esed, 0, false);
	return esed.res;
}
```

<br>

## 4. LT 和 ET 实现上的区别是什么？
----

<br>

关于事件的通知和唤醒，在 select 分析那篇已经说过了，底层机制都一样。这里我们看看 LT 和 ET 在实现上的区别是什么。

接着第 3 小节中的 ep_send_events 函数继续， 其调用栈如下:

``` cpp
ep_send_events()
    |
    | -> for item in readylist: {ep_send_events_proc()}
    | -> // add new ready event to readylist
    | -> // wai
```

其核心在 ```ep_send_events_proc()``` 函数：

``` cpp
static __poll_t ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
			       void *priv)
{
	struct ep_send_events_data *esed = priv;
	__poll_t revents;
	struct epitem *epi;
	struct epoll_event __user *uevent;
	struct wakeup_source *ws;
	poll_table pt;

	init_poll_funcptr(&pt, NULL);

	/*
	 * We can loop without lock because we are passed a task private list.
	 * Items cannot vanish during the loop because ep_scan_ready_list() is
	 * holding "mtx" during this call.
	 */
	for (esed->res = 0, uevent = esed->events;
	     !list_empty(head) && esed->res < esed->maxevents;) {
		epi = list_first_entry(head, struct epitem, rdllink);

		/*
		 * Activate ep->ws before deactivating epi->ws to prevent
		 * triggering auto-suspend here (in case we reactive epi->ws
		 * below).
		 *
		 * This could be rearranged to delay the deactivation of epi->ws
		 * instead, but then epi->ws would temporarily be out of sync
		 * with ep_is_linked().
		 */
		ws = ep_wakeup_source(epi);
		if (ws) {
			if (ws->active)
				__pm_stay_awake(ep->ws);
			__pm_relax(ws);
		}

		list_del_init(&epi->rdllink);

		revents = ep_item_poll(epi, &pt, 1);

		/*
		 * If the event mask intersect the caller-requested one,
		 * deliver the event to userspace. Again, ep_scan_ready_list()
		 * is holding "mtx", so no operations coming from userspace
		 * can change the item.
		 */
		if (revents) {
			if (__put_user(revents, &uevent->events) ||
			    __put_user(epi->event.data, &uevent->data)) {
				list_add(&epi->rdllink, head);
				ep_pm_stay_awake(epi);
				if (!esed->res)
					esed->res = -EFAULT;
				return 0;
			}
			esed->res++;
			uevent++;
			if (epi->event.events & EPOLLONESHOT)
				epi->event.events &= EP_PRIVATE_BITS;
			else if (!(epi->event.events & EPOLLET)) {
				/*
				 * If this file has been added with Level
				 * Trigger mode, we need to insert back inside
				 * the ready list, so that the next call to
				 * epoll_wait() will check again the events
				 * availability. At this point, no one can insert
				 * into ep->rdllist besides us. The epoll_ctl()
				 * callers are locked out by
				 * ep_scan_ready_list() holding "mtx" and the
				 * poll callback will queue them in ep->ovflist.
				 */
                /* 注意在这里 就是 LT ET 的 区别了
                        其实如果我们每次都检查文件设备的状态的话，那就是 level trigger 的，
                        因为如果设备是可读的话，每次调用 file->poll() 都会返回可读状态。
                    而 epoll 并不是每次轮询的，而是把状态改变的通知放到了 rdllink 里边
                        进入到 rdlink 的条件是主动把 current-task 放到设备等待队列中，
                        TODO 如果这次放了，但是没有读，下次在放的都
                        试想这样的情况， 
                            这次事件触发从 rdllink 里边移走，并返回给了上层用户，但是上层没有读，这是设备里边是有数据的
                            问题是，设备状态没有改变(变多), 那就不会再次通知给epoll，也就没有机会再次进入rdllink了。
                            这样上层用户就不会在有机会读到设备的数据了(直到设备状态下次改变。)
                    每次 epoll_wait() 都会把 rdllink 中的元素取走，如果下次在来的时候就没有了。
                    而 level trigger 则是相当于把事件再次放到里边，下次在执行 epoll_wait() 的时候，
                    就会在检查设备的状态 (file->poll)，这样就不会错过没有处理的元素了。

                    select 就没有 ET，因为它每次都轮询所有的 fd。而 epoll 只轮询 rdllink 里边的元素
                */
				list_add_tail(&epi->rdllink, &ep->rdllist);
				ep_pm_stay_awake(epi);
			}
		}
	}

	return 0;
}
```



<br>

## 5. 惊群是怎么处理的？
----

<br>

在linux4.15 版本之后引入了 [EPOLLEXCLUSIVE](https://man7.org/linux/man-pages/man2/epoll_ctl.2.html) 选项，来解决惊群问题，这个选项的玩法是这样的：

1. 如果有多个 task 在 wait 同一个设备的事件，并且这些 task 在 epoll_ctl 的时候都设置了 EPOLLEXCLUSIVE 选项。等事件触发的时候，只会返回 task 中的一个。

2. 如果 task 中有的没有设置 EPOLLEXCLUSIVE ，那这些没有 EPOLLEXCLUSIVE 标记的都会被唤醒。

其实核心就在 wakeup 函数里边了, 在分析 select 的实现中，知道了，当数据可读时，有如下唤醒流程：

``` cpp
tcp_data_queue()
    |
    | -> tcp_data_ready()
            |
            | -> sk->sk_data_ready(sk) ( 即：sock_def_readable)
                    |
                    | -> wake_up_interruptible_sync_poll() 
                            |
                            | -> __wake_up_sync_key((x), TASK_INTERRUPTIBLE, 1, poll_to_key(m))
                                // 注意： 这个函数调用中 nr_exclusive 传的是1
                                    |
                                    | -> __wake_up_common_lock()
                                            |
                                            | -> __wake_up_common()
```

重点就是 ```__wake_up_common()``` 这个函数了：

``` cpp
/*
 * The core wakeup function. Non-exclusive wakeups (nr_exclusive == 0) just
 * wake everything up. If it's an exclusive wakeup (nr_exclusive == small +ve
 * number) then we wake all the non-exclusive tasks and one exclusive task.
 *
 * There are circumstances in which we can try to wake a task which has already
 * started to run but is not in state TASK_RUNNING. try_to_wake_up() returns
 * zero in this (rare) case, and we handle it by continuing to scan the queue.
 */
static int __wake_up_common(struct wait_queue_head *wq_head, unsigned int mode,
			int nr_exclusive, int wake_flags, void *key,
			wait_queue_entry_t *bookmark)
{
	wait_queue_entry_t *curr, *next;
	int cnt = 0;

	lockdep_assert_held(&wq_head->lock);

	if (bookmark && (bookmark->flags & WQ_FLAG_BOOKMARK)) {
		curr = list_next_entry(bookmark, entry);

		list_del(&bookmark->entry);
		bookmark->flags = 0;
	} else
		curr = list_first_entry(&wq_head->head, wait_queue_entry_t, entry);

	if (&curr->entry == &wq_head->head)
		return nr_exclusive;

	list_for_each_entry_safe_from(curr, next, &wq_head->head, entry) {
		unsigned flags = curr->flags;
		int ret;

		if (flags & WQ_FLAG_BOOKMARK)
			continue;

		ret = curr->func(curr, mode, wake_flags, key);
		if (ret < 0)
			break;

        /* 处理惊群就在这里了
             其中 nr_exclusive 是以 exclusive 方式唤醒 task 的个数
             注意我们上边的调用栈中调用 __wake_up_common 时， nr_exclusive 对应的参数是1
             也就是说希望在所有设置了 EXCLUSIVE 标记的 task 中选一个唤醒。
             这个被唤醒的 task  就是队列中最前边那个 exclusive 的 task
        */
		if (ret && (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
			break;

		if (bookmark && (++cnt > WAITQUEUE_WALK_BREAK_CNT) &&
				(&next->entry != &wq_head->head)) {
			bookmark->flags = WQ_FLAG_BOOKMARK;
			list_add_tail(&bookmark->entry, &next->entry);
			break;
		}
	}

	return nr_exclusive;
}
```

<br>

## 6. 总结
----

<br>

1. epoll 的核心是把所有触发事件的 fd 设备，存储到了一个list 里边 rdllink，用来避免遍历所有监听的fd。因为 epoll 要监听的fd 没有数量限制。如果被监听的 fd 太多，那遍历所有就效率太低了。

2. 注意虽然有rdllink ，但这个 list 中的fd 并不一定都返回给用户空间。因为还要对这个 list 中的 fd 的文件，依次执行 poll 操作，以判断是否 “真的有我们关心的设备状态，以及查询最新的设备状态”。这里也就是 LT ，ET 的区别了。LT 比 ET 低效的原因也就是：
   
   epoll，对于已经触发过事件的fd，会不断循环添加到 rdllink 里边，也就是每次调用都会 file->poll() 一下上次有事件触发的设备，看看用户有没有处理。最好的情况下，也会比 ET 多 1 次 file->poll() 操作。







<br><br><br>

## 参考文档

* [The Implementation of epoll](https://idndx.com/the-implementation-of-epoll-1/)
* [彻底学会使用epoll(一)——ET模式实现分析](http://blog.chinaunix.net/uid-28541347-id-4273856.html)
* [The method to epoll’s madness](https://copyconstruct.medium.com/the-method-to-epolls-madness-d9d2d6378642)
* [Implementation of Epoll](https://fd3kyt.github.io/posts/implementation-of-epoll/)
* [epoll implementation](http://baotiao.github.io/2016/03/06/epoll_implementation/)