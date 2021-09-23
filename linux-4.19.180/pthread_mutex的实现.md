# pthread_mutex的实现

pthread_mutex 的实现在 glibc 库中，支持可重入和不可重入。pthread_mutex 用到了 CAS 操作更新锁的状态


<br>

## 1. 相关的数据结构
----
<br>

以 x86 为例

``` cpp

// sysdeps/nptl/bits/pthreadtypes.h
typedef union
{
  struct __pthread_mutex_s __data;
  char __size[__SIZEOF_PTHREAD_MUTEX_T];
  long int __align;
} pthread_mutex_t;

// sysdeps/x86/nptl/bits/struct_mutex.h:22
struct __pthread_mutex_s
{
  int __lock;               // 锁的状态 0 没有被持有。 这个变量的操作是通过 CAS 系列函数操作的
  unsigned int __count;     // 如果是可重入锁的话，记录加锁的次数
  int __owner;              // 当前持有锁的线程
#ifdef __x86_64__
  unsigned int __nusers;
#endif
  /* KIND must stay at this position in the structure to maintain
     binary compatibility with static initializers.  */
  int __kind;               // 持有的是那种类型的
#ifdef __x86_64__
  short __spins;
  short __elision;
  __pthread_list_t __list;
# define __PTHREAD_MUTEX_HAVE_PREV      1
#else
  unsigned int __nusers;
  __extension__ union
  {
    struct
    {
      short __espins;
      short __eelision;
# define __spins __elision_data.__espins
# define __elision __elision_data.__eelision
    } __elision_data;
    __pthread_slist_t __list;
  };
# define __PTHREAD_MUTEX_HAVE_PREV      0
#endif
};
```

<br>

## 2. 加锁机制
----
<br>

pthread_mutext 的实现在 ```glibc/nptl/pthread_mutex_lock.c:72```。

我们先看一下正常加锁的调用栈：

``` cpp
int PTHREAD_MUTEX_LOCK (pthread_mutex_t *mutex)
    |
    |-> LLL_MUTEX_LOCK_OPTIMIZED (mutex);
        |
        |-> lll_mutex_lock_optimized (mutex)
            |
            |-> lll_lock (mutex->__data.__lock, private);
                |
                |-> __lll_lock (&(futex), private)
                    |
                    |-> atomic_compare_and_exchange_bool_acq (__futex, 1, 0)))  // 先尝试在用户空间加一下锁
                    |-> __lll_lock_wait (__futex, private); // 上一步的加锁尝试失败，需要进入等待
```

其核心也就是 ```___lll_lock_wait()``` 函数：

``` cpp
void
__lll_lock_wait (int *futex, int private)
{
  if (atomic_load_relaxed (futex) == 2)
    goto futex;

  while (atomic_exchange_acquire (futex, 2) != 0)
    {
    futex:
      LIBC_PROBE (lll_lock_wait, 1, futex);

      //这里最终会进行系统调用: futex
      futex_wait ((unsigned int *) futex, 2, private); /* Wait if *futex == 2.  */
    }
}
```

其流程如下：

1. 加锁机制是使用 CAS 操作对 __lock 进行尝试更新，如果更新成功，说明加锁成功，直接返回。

2. 否则执行 futex 系统调用切换到 kernel， 进入休眠等待。

3. 当 mutex 被释放时，当前 task 被唤醒，再次执行上述检测机制


<br>

## 3. 解锁机制
----
<br>

我们只看一下关键的部分 ```sysdeps/nptl/lowlevellock.h:145``` ，如下

``` cpp
#define __lll_unlock(futex, private)					\
  ((void)								\
  ({									\
     int *__futex = (futex);						\
     int __private = (private);						\
     int __oldval = atomic_exchange_rel (__futex, 0);			\
     if (__glibc_unlikely (__oldval > 1))				\
       {								\
         if (__builtin_constant_p (private) && (private) == LLL_PRIVATE) \
           __lll_lock_wake_private (__futex);                           \
         else                                                           \
           __lll_lock_wake (__futex, __private);			\
       }								\
   }))
```

可以看到里边有根据 ```__oldval > 1``` 的判断。这个其实是一个优化的技巧。在解释这个技巧前，我们先看一下 futex 这个系统调用

我们在来看看 ```__lll_lock_wake()``` 的调用栈：

``` cpp

__lll_lock_wake (__futex, __private);
    |
    |-> lll_futex_wake (futex, 1, private);

```

其中 ```lll_futex_wake()`` 定义如下, 即，最多唤醒 nr 个 waiter：
``` cpp
/* Wake up up to NR waiters on FUTEXP.  */
# define lll_futex_wake(futexp, nr, private)                             \
  lll_futex_syscall (4, futexp,                                         \
		     __lll_private_flag (FUTEX_WAKE, private), nr, 0)
```

也就是会唤醒 1 个 waiter 尝试获取锁，这样就避免了惊群的问题。

<br>

## 4. futex 系统调用
----
<br>

关于 futex， 这个是把当前 task 陷入等待状态，直到等待的事件触发。比如锁被释放

``` cpp
DESCRIPTION
       The futex() system call provides a method for waiting until a certain condition becomes true.  It is typically used as a blocking construct in the context of shared-memory synchronization.  When using futexes, the majority of
       the synchronization operations are performed in user space.  A user-space program employs the futex() system call only when it is likely that the program has to block for a longer time until the condition becomes true.  Other
       futex() operations can be used to wake any processes or threads waiting for a particular condition.
```

其函数原型如下：

``` cpp
    int futex(int *uaddr, int futex_op, int val,
                 const struct timespec *timeout,   /* or: uint32_t val2 */
                 int *uaddr2, int val3);
```

这个函数有个比较 tricky 的地方：其中第三个参数 val 为当前调用者期望的值，也就是说如果当前值 *uaddr == 0 的情况下：

1. 我们如果执行以下调用：

    futex(uaddr, wait, 0, ...) 
    
    那当前 task 就会进入休眠等待状态，直到 *uaddr 的值发生改变(即 != 0)

1. 我们如果执行以下调用：

    futex(uaddr, wait, 1, ...) 
    
    那当前 task 就会立即返回 EWOILDBLOCK，而不会休眠等待状态

当 futex 执行 wake 逻辑时， val 表示的是最多唤醒的个数。在 glibc 中(```nptl/pthread_mutex_unlock.c:177```) 

```futex_wake ((unsigned int *) &mutex->__data.__lock, 1, private);```
FUTEX_WAKE: 最多唤醒val个等待在uaddr上进程。


<br>

## 5. futex 的内核实现
----
<br>

在锁的实现上，我们使用到的 futex 的逻辑有两个：

* futex_wait: 这个是把当前 task 放到等待队列，然后进入休眠， 等 待*uaddr 的值发生改变时被唤醒

* futex_wake: 这个是当 *uaddr 的值改变时，唤醒所有等待的 task


futex 的实现在 ```kernel/futex.c:3920``` , 其调用栈如下：

``` cpp
SYSCALL_DEFINE6(futex, u32 __user *, uaddr, int, op, u32, val, ...)
        |
        |-> copy_from_user(&ts, utime, sizeof(ts)) // 拷贝超时时间
        |-> do_futex(uaddr, op, val, tp, uaddr2, val2, val3)
            |
            |-> // switch op & FUTEX_CMD_MASK
            |-> futex_wait(uaddr, flags, val, timeout, val3)
            |-> futex_wake(uaddr, flags, val, val3);
            |-> ......
```

其中 futex_wait 的实现如下：

``` cpp
static int futex_wait(u32 __user *uaddr, unsigned int flags, u32 val,
		      ktime_t *abs_time, u32 bitset)
{
    // 设置超时时间
	if (abs_time) {
		to = &timeout;

		hrtimer_init_on_stack(&to->timer, (flags & FLAGS_CLOCKRT) ?
				      CLOCK_REALTIME : CLOCK_MONOTONIC,
				      HRTIMER_MODE_ABS);
		hrtimer_init_sleeper(to, current);
		hrtimer_set_expires_range_ns(&to->timer, *abs_time,
					     current->timer_slack_ns);
	}

retry:
	/*
	 * Prepare to wait on uaddr. On success, holds hb lock and increments
	 * q.key refs.
	 */
	ret = futex_wait_setup(uaddr, val, flags, &q, &hb);
	if (ret)
		goto out;

	/* queue_me and wait for wakeup, timeout, or a signal. */
    // 把当前 task 放到 等待队列中
	futex_wait_queue_me(hb, &q, to);

	/* If we were woken (and unqueued), we succeeded, whatever. */
	ret = 0;
	/* unqueue_me() drops q.key ref */
	if (!unqueue_me(&q))
		goto out;
	ret = -ETIMEDOUT;
	if (to && !to->task)
		goto out;

	/*
	 * We expect signal_pending(current), but we might be the
	 * victim of a spurious wakeup as well.
	 */
	if (!signal_pending(current))
		goto retry;

	ret = -ERESTARTSYS;
	if (!abs_time)
		goto out;

    // TODO??
	restart = &current->restart_block;
	restart->fn = futex_wait_restart;
	restart->futex.uaddr = uaddr;
	restart->futex.val = val;
	restart->futex.time = *abs_time;
	restart->futex.bitset = bitset;
	restart->futex.flags = flags | FLAGS_HAS_TIMEOUT;

	ret = -ERESTART_RESTARTBLOCK;

out:
	if (to) {
		hrtimer_cancel(&to->timer);
		destroy_hrtimer_on_stack(&to->timer);
	}
	return ret;
}
```

<br>

## 6. __lock 为什么会有三中状态 ？
----
<br>

有了 futex 函数的基础，我们看一下为什么需要这 3 个状态？

我们先从最原始的想法开始：

``` cpp
class mutex {
public:
    mutex(): val(0) {}

    void lock() {
        while(compare_and_swap(val, 0, 1) != 0) {
            // 等待变成 非1
            futex_wait(&val, 1);
        }
    }

    void unlock() {
        compare_and_swap(val, 1, 0)
        // 唤醒其他等待的 task
        futex_wake(&val, 0);
    }

private:
    int val;
};
```

能看到上述代码的性能问题吗？

* 在 unlock 的时候，会必然调用 futex_wake() 系统调用，即时当前没有其他线程在等待，这样会造成不必要的消耗，因为是系统调用

所以才有了 pthread_mutex 中 3 种 状态的转换，就是为了尽量减少不必要的系统调用

先说一下 ```__pthread_mutex_s.__lock``` 这个变量的状态转换，对于通过 CAS 操作检测这个变量的 task 来说：

- 如果 CmpAndChg(__lock, 0, 1) 
  
  - 返回值(原有值) 为 0，说明之前锁没有被任何人持有，此时我们获得了锁，直接返回。

  - 返回值(原有值) 为 1，说明有人正在持有锁，**但是没有其他waiters**， 也就是我们是第一个waiter

  - 返回值(原有值) 为 2，说明有人正在持有锁，**且已经有其他waiters 在等待**


根据上述状态，当锁的持有者是唯一的持有者时，如果它释放锁，即

``` cpp
    int oldval = compare_and_swap(val, 1, 0);
```

1. 如果发现 oldval == 1，说明除了它，没有人想获取锁，此时就不用调用 futex_wake() 了，节省了系统调用的开销

2. 如果发现 oldval == 2，说明除了它，还有其他人在等待锁，此时就需要调用 futex_wake() 唤醒其他人

综上所述，通过加了一个新的状态，节省了昂贵的系统调用，由此带来的收益还是很划算的。

<br>

## 7. 锁 与 中断
----
<br>

首先锁分为用户空间的锁和内核空间的锁。

如果是用户空间的锁，是不需要关中断的，也就是。当中断发生的时候，会切到kernel，等中断处理完，会在回到用户空间继续执行抢锁操作。

如果是内核空间，比如 spinlock, 就需要关闭中断了。为了保证在加锁执行的过程中，其他 task 不会被调度起来。另外也不可能进入睡眠状态。


<br><br><br>

## 参考资料
----

* [How pthread_mutex_lock is implemented](https://stackoverflow.com/questions/5095781/how-pthread-mutex-lock-is-implemented)
* [what happens if Interrupts occur after mutex lock has been acquired](https://stackoverflow.com/questions/22218365/what-happens-if-interrupts-occur-after-mutex-lock-has-been-acquired)
* [Basics of Futexes](https://eli.thegreenplace.net/2018/basics-of-futexes/)
* Futexes are Tricky