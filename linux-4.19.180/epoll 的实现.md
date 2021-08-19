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

<br>

## 2. epoll_ctl: 增删改 需要监听的fd
----

<br>

这个函数是跟 select 不一样的地方。在 select 机制中，每次调用 select 其实是同时执行了两个操作：

1. 添加要监听的 fd （用户态到内核态的数据拷贝）
2. 轮询所有被监听的 fd 的事件

每次全量添加 fd， 也就成了影响 select 性能的一个地方。所以在 epoll 中把以上这两个操作拆开成了两个独立的步骤。所以在 wait 时操作也就减少了。

这里主要说一下 insert 操作:

1. 首先，添加的需要监听的 fd， 会分配一个 ```struct epitem``` 结构。
   
    这个接口保存了事件的所有信息。epoll 实现中每个被监听的 fd 都对应一个 ```struct epitem```。

2. 把 epitem 添加的红黑树里边。


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
static inline int ep_events_available(struct eventpoll *ep)
{
	return !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;
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

## 5. 惊群是怎么处理的？TODO
----

<br>

<br><br><br>

## 参考文档

* [The Implementation of epoll](https://idndx.com/the-implementation-of-epoll-1/)
* [彻底学会使用epoll(一)——ET模式实现分析](http://blog.chinaunix.net/uid-28541347-id-4273856.html)
* [The method to epoll’s madness](https://copyconstruct.medium.com/the-method-to-epolls-madness-d9d2d6378642)
* [Implementation of Epoll](https://fd3kyt.github.io/posts/implementation-of-epoll/)
* [epoll implementation](http://baotiao.github.io/2016/03/06/epoll_implementation/)