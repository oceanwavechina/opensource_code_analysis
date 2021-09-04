# pthread_mutex的实现

pthread_mutex 的实现在 glibc 库中，支持可重入和不可重入。pthread_mutex 用到了 CAS 操作更新锁的状态。同时会自适应的选择自旋方式还是内核调用睡眠方式。


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

1. 加锁机制是使用 CAS 操作对 __lock 进行尝试更新，如果更新成功，说明加锁成功，直接返回。
2. 否则执行 futex 系统调用切换到 kernel， 进入休眠等待。
3. 当 mutex 被释放时，当前 task 被唤醒，再次执行上述检测机制


关于 futex， 这个是把当前 task 陷入等待状态，直到等待的事件触发。比如锁被释放

``` cpp
DESCRIPTION
       The futex() system call provides a method for waiting until a certain condition becomes true.  It is typically used as a blocking construct in the context of shared-memory synchronization.  When using futexes, the majority of
       the synchronization operations are performed in user space.  A user-space program employs the futex() system call only when it is likely that the program has to block for a longer time until the condition becomes true.  Other
       futex() operations can be used to wake any processes or threads waiting for a particular condition.
```

<br>

## 3. 锁 与 中断
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