# Spinlock的实现与memory order

如 poco 注释所说，spinlock 比价适合于对小的代码块进行加锁，这个小指的是持有锁的时间短。

其实现机制是 busy-wait, 也就是不让出cpu，比较适用于多核心的情况。其优点是减少的因进入 sleep 而导致的切换调度开销

<br>

## 1. 实现流程
----
<br>

其核心思想是使用 c++ 中提供的 一个原子操作来控制变量的状态，如下：

``` cpp
    std::atomic_flag _flag = ATOMIC_FLAG_INIT;
```

这个类型可以在 lock-free 的情况下修改对应的值。而加锁解锁则是竞争访问和修改这个变量

<br>

我们看看 加锁 和 释放锁 的逻辑：

``` cpp
inline void SpinlockMutex::lock()
{
	while (_flag.test_and_set(std::memory_order_acquire));
}

inline void SpinlockMutex::unlock()
{
	_flag.clear(std::memory_order_release);
}
```

核心函数是 ```test_and_set()```:

> Atomically changes the state of a std::atomic_flag to set (true) and returns the value it held before.

也就是说不管之前 _flag 之前是什么，在执行这个方法结束后，_flag 的值都会变成 true。同时返回之前的值

所以我们加锁的逻辑就是：

* 我们只需要判断之前有没有人持有这个锁(原有的 _flag = false)，如果有人持有，我们就不断循环判断。

问题来了，死循环判断，不耗费 cpu 吗?

* 首先，肯定是这样的

* 但是我们的让出cpu，就会进行上下文调度切换，同样会耗费 cpu 资源

* 所以我们要权衡，那种操作的消耗更小，如果对应只有几个指令周期的临界区。忙等的消耗，要远小于上下文切换的开销。这就是自旋锁存在的一个意义

<br>

## 2. memory order
----
<br>

仅仅靠原子指令实现不了最资源的访问控制。因为 编译器 和 cpu 可能并不会按照我们的代码顺序去执行。为了进行优化，可能会打乱执行顺序，所以我们要手动设置 编译器和 cpu 乱序执行的行为。

如下：
``` cpp
For example, consider the following sequence of events:

	CPU 1		CPU 2
	===============	===============
	{ A == 1; B == 2 }      //这里是 A, B 的初始值
	A = 3;		x = B;
	B = 4;		y = A;

The set of accesses as seen by the memory system in the middle can be arranged
in 24 different combinations:

	STORE A=3,	STORE B=4,	y=LOAD A->3,	x=LOAD B->4
	STORE A=3,	STORE B=4,	x=LOAD B->4,	y=LOAD A->3
	STORE A=3,	y=LOAD A->3,	STORE B=4,	x=LOAD B->4
	STORE A=3,	y=LOAD A->3,	x=LOAD B->2,	STORE B=4
	STORE A=3,	x=LOAD B->2,	STORE B=4,	y=LOAD A->3
	STORE A=3,	x=LOAD B->2,	y=LOAD A->3,	STORE B=4
	STORE B=4,	STORE A=3,	y=LOAD A->3,	x=LOAD B->4
	STORE B=4, ...
	...

and can thus result in four different combinations of values:

	x == 2, y == 1
	x == 2, y == 3
	x == 4, y == 1
	x == 4, y == 3


Furthermore, the stores committed by a CPU to the memory system may not be
perceived by the loads made by another CPU in the same order as the stores were
committed.
```

而 c++ 中不同的 memory order 则规定了哪些操作可以执行重排，那些不可以。

<br><br><br>

## 参考资料
----
<br>

* [Memory model synchronization modes](http://gcc.gnu.org/wiki/Atomic/GCCMM/AtomicSync)
* [详解c++ atomic原子编程中的Memory Order](https://www.jb51.net/article/214304.htm#_label4)
* [Memory Consistency Models: A Tutorial](https://www.cs.utexas.edu/~bornholt/post/memory-models.html)
* [Memory barriers](https://mariadb.org/wp-content/uploads/2017/11/2017-11-Memory-barriers.pdf)