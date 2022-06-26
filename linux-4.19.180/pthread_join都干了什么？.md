# pthread_join的作用是什么？

先来看看系统手册的描述：

The pthread_join() function suspends execution of the calling thread until the target thread terminates unless the target thread has already terminated.

基本就是挂起当前线程，知道被等待的线程结束。

<br>

## 1. pthread_join 的内部逻辑
----
<br>

pthread_join 的实现在 glibc 下的 ```nptl/pthread_join.c``` (最初是 redhat 的 NPTL 项目)。

先看看其调用栈:

``` cpp
___pthread_join()
    |-> __pthread_clockjoin_ex()
```

最终调用的是 __pthread_clockjoin_ex 函数，这个函数是在 pthread_join_common.c 文件里边。

因为代码并不长，我们贴实现

``` cpp
int
__pthread_clockjoin_ex (pthread_t threadid, void **thread_return,
                        clockid_t clockid,
                        const struct __timespec64 *abstime, bool block)
{
  struct pthread *pd = (struct pthread *) threadid;

  /* Make sure the descriptor is valid.  */
  if (INVALID_NOT_TERMINATED_TD_P (pd))
    /* Not a valid thread handle.  */
    return ESRCH;

  /* Is the thread joinable?.  */
  if (IS_DETACHED (pd))
    /* We cannot wait for the thread.  */
    return EINVAL;

  struct pthread *self = THREAD_SELF;
  int result = 0;

  LIBC_PROBE (pthread_join, 1, threadid);

  if ((pd == self
       || (self->joinid == pd
	   && (pd->cancelhandling
	       & (CANCELED_BITMASK | EXITING_BITMASK
		  | TERMINATED_BITMASK)) == 0))
      && !(self->cancelstate == PTHREAD_CANCEL_ENABLE
	   && (pd->cancelhandling & (CANCELED_BITMASK | EXITING_BITMASK
				     | TERMINATED_BITMASK))
	       == CANCELED_BITMASK))
    /* This is a deadlock situation.  The threads are waiting for each
       other to finish.  Note that this is a "may" error.  To be 100%
       sure we catch this error we would have to lock the data
       structures but it is not necessary.  In the unlikely case that
       two threads are really caught in this situation they will
       deadlock.  It is the programmer's problem to figure this
       out.  */
    return EDEADLK;

  /* Wait for the thread to finish.  If it is already locked something
     is wrong.  There can only be one waiter.  */
  else if (__glibc_unlikely (atomic_compare_exchange_weak_acquire (&pd->joinid,
								   &self,
								   NULL)))
    /* There is already somebody waiting for the thread.  */
    return EINVAL;

  /* BLOCK waits either indefinitely or based on an absolute time.  POSIX also
     states a cancellation point shall occur for pthread_join, and we use the
     same rationale for posix_timedjoin_np.  Both clockwait_tid and the futex
     call use the cancellable variant.  */
  if (block)
    {
      /* During the wait we change to asynchronous cancellation.  If we
	 are cancelled the thread we are waiting for must be marked as
	 un-wait-ed for again.  */
      pthread_cleanup_push (cleanup, &pd->joinid);

      /* We need acquire MO here so that we synchronize with the
         kernel's store to 0 when the clone terminates. (see above)  */
      pid_t tid;
      while ((tid = atomic_load_acquire (&pd->tid)) != 0)
        {
         /* The kernel notifies a process which uses CLONE_CHILD_CLEARTID via
	    futex wake-up when the clone terminates.  The memory location
	    contains the thread ID while the clone is running and is reset to
	    zero by the kernel afterwards.  The kernel up to version 3.16.3
	    does not use the private futex operations for futex wake-up when
	    the clone terminates.  */
	  int ret = __futex_abstimed_wait_cancelable64 (
	    (unsigned int *) &pd->tid, tid, clockid, abstime, LLL_SHARED);
	  if (ret == ETIMEDOUT || ret == EOVERFLOW)
	    {
	      result = ret;
	      break;
	    }
	}

      pthread_cleanup_pop (0);
    }

  void *pd_result = pd->result;
  if (__glibc_likely (result == 0))
    {
      /* We mark the thread as terminated and as joined.  */
      pd->tid = -1;

      /* Store the return value if the caller is interested.  */
      if (thread_return != NULL)
	*thread_return = pd_result;

      /* Free the TCB.  */
      __nptl_free_tcb (pd);
    }
  else
    pd->joinid = NULL;

  LIBC_PROBE (pthread_join_ret, 3, threadid, result, pd_result);

  return result;
}
```

可以看到函数大致分成三步：

1. 第一部分是错误检查，包括是否 detached、deadlock 检查等

2. 如果开启 block 选项，则当前线程陷入等待

3. 获取线程的返回结果，并清理 TCB

我觉得第3步是 pthread_join 的核心用途，就是得到线程的执行结果。

看到这里有两个问题比较感兴趣：

1. __nptl_free_tcb 中的 TCB（线程控制块）都包含哪些信息

2. 等待-唤醒的通知机制

<br>

## 2. TCB 都包含得了哪些信息？
----
<br>

TCB 实际上就是 ```struct pthread``` 这个数据接口，其定义在 [nptl/descr.h](https://elixir.bootlin.com/glibc/glibc-2.34/source/nptl/descr.h#L128) 下，因为这个结构还是比较长的，这里就不贴完整代码了，只挑几个有意思的成员看一下。

<br>

### 2.1 TCB 中的 header

<br>

``` cpp
struct
{
    /* multiple_threads is enabled either when the process has spawned at
    least one thread or when a single-threaded process cancels itself.
    This enables additional code to introduce locking before doing some
    compare_and_exchange operations and also enable cancellation points.
    The concepts of multiple threads and cancellation points ideally
    should be separate, since it is not necessary for multiple threads to
    have been created for cancellation points to be enabled, as is the
    case is when single-threaded process cancels itself.

    Since enabling multiple_threads enables additional code in
    cancellation points and compare_and_exchange operations, there is a
    potential for an unneeded performance hit when it is enabled in a
    single-threaded, self-canceling process.  This is OK though, since a
    single-threaded process will enable async cancellation only when it
    looks to cancel itself and is hence going to end anyway.  */
    int multiple_threads;
    int gscope_flag;
} header;

```

这个 header 这个结构中有一个 multiple_threads 变量来标识是否开启了多线程。因为在多线程中释放资源跟单线程是不一样的，多线程环境下资源的释放要等所有线程没有被引用时才可以释放。

比如 ```_dl_close()```(elf/dl-close.c) 操作

``` cpp
_dl_close (void *_map)
    |
    |-- _dl_close_worker (struct link_map *map, bool force)
            |
            |--  通过判断 RTLD_SINGLE_THREAD_P 为多线程时，需要 THREAD_GSCOPE_WAIT() 进行等待
                    |
                    |-- __thread_gscope_wait()
```

<br>

### 2.2 TCB 中的 joinid

<br>

``` cpp
  /* If the thread waits to join another one the ID of the latter is
     stored here.

     In case a thread is detached this field contains a pointer of the
     TCB if the thread itself.  This is something which cannot happen
     in normal operation.  */
  struct pthread *joinid;
  /* Check whether a thread is detached.  */
#define IS_DETACHED(pd) ((pd)->joinid == (pd))
```

判断是线程是不是 detached 就是通过这个变量来的，这就要回到 [pthread_create](https://elixir.bootlin.com/glibc/glibc-2.34/source/nptl/pthread_create.c#L649) 时的逻辑了：

``` cpp
    
    versioned_symbol (libc, __pthread_create_2_1, pthread_create, GLIBC_2_34);
        |
        |-- __pthread_create_2_1 (pthread_t *newthread, const pthread_attr_t *attr,
		      void *(*start_routine) (void *), void *arg)



其中，关于 pd->joinid 的赋值如下


  /* Initialize the field for the ID of the thread which is waiting
     for us.  This is a self-reference in case the thread is created
     detached.  */
  pd->joinid = iattr->flags & ATTR_FLAG_DETACHSTATE ? pd : NULL;

```

也就是说如果一个 thread 是 detached， 那个其 joinid 指向的就是它自己的 TCB


<br>

### 2.3 TCB 中的 result

<br>

``` cpp
  /* The result of the thread function.  */
  void *result;

  /* Start position of the code to be executed and the argument passed
    to the function.  */
  void *(*start_routine) (void *);
  void *arg;
```  

我们在 [join()](https://elixir.bootlin.com/glibc/glibc-2.34/source/nptl/pthread_join_common.c#L117) 函数的逻辑中看到了这个变量的修改，被等待的线程结束后结果存放的地方，实际上就是线程函数的返回值。

start_routine 实际上就是我们用户层的线程函数。


<br>

## 2. join的等待和唤醒机制
----
<br>

``` cpp

__pthread_clockjoin_ex (pthread_t threadid, void **thread_return,
                        clockid_t clockid,
                        const struct __timespec64 *abstime, bool block)
    |
    |-- int ret = __futex_abstimed_wait_cancelable64 (
	    (unsigned int *) &pd->tid, tid, clockid, abstime, LLL_SHARED);
            |
            |-- __futex_abstimed_wait_common (futex_word, expected, clockid,
                                       abstime, private, true);
                    |
                    |-- err = __futex_abstimed_wait_common64 (futex_word, expected, op, abstime,
					                private, cancel);
                        |
                        |-- return INTERNAL_SYSCALL_CALL (futex_time64, futex_word, op, expected,
				                        abstime, NULL /* Ununsed.  */, FUTEX_BITSET_MATCH_ANY);
```


在join的实现中我们看到 glibc 最终使用 futex_time64 来使当前线程陷入等待。关于  __futex_abstimed_wait_cancelable64 ，实际上就是添加了超时机制的 futex_wait，源码注释如下。


``` cpp
/* Like futex_wait, but will eventually time out (i.e., stop being blocked)
   after the duration of time provided (i.e., ABSTIME) has passed using the
   clock specified by CLOCKID (currently only CLOCK_REALTIME and
   CLOCK_MONOTONIC, the ones support by lll_futex_supported_clockid). ABSTIME
   can also equal NULL, in which case this function behaves equivalent to
   futex_wait.

   Returns the same values as futex_wait under those same conditions;
   additionally, returns ETIMEDOUT if the timeout expired.

   The call acts as a cancellation entrypoint.  */
```



<br>

## 3. 总结
----
<br>

1. joinable thread 在等待线程结束的同时可以获取线程的结果， 而 detached 则要通过其他方式传递出来

2. detached thread 在线程创建的时候会执行 clean tcb 的操作， 而joinable thread 要在join的时候才会执行。



<br><br><br>


## 参考资料
----

<br>

* [POSIX thread (pthread) libraries](https://www.cs.cmu.edu/afs/cs/academic/class/15492-f07/www/pthreads.html#BASICS)
* [glibc-2.34 nptl部分](https://elixir.bootlin.com/glibc/glibc-2.34/source/nptl)