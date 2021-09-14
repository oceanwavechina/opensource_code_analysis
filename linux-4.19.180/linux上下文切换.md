# linux上下文切换

linux 下任务的最小单位是 task。上下文切换也就是 task 之间的切换。

task 的定义在 ```include/linux/sched.h:593```


<br>

## 1. 调用栈
----
<br>

调度函数 ```schedule(void)``` 的实现在 ```kernel/sched/core.c:3554```。

``` cpp
schedule(void)
    |
    |-> preempt_disable();  // 关闭抢占
    |-> __schedule(false);  // 核心调度逻辑
    |-> sched_preempt_enable_no_resched();  // 开启抢占
```

接下来看看 ```__schedule(bool preempt)``` 这个函数：

TODO



<br><br><br>

## 参考资料
----

* [linux内核上下文切换解析](https://blog.csdn.net/a7980718/article/details/82807986)
* [Preemption and Context Switching](https://www.informit.com/articles/article.aspx?p=101760&seqNum=3)