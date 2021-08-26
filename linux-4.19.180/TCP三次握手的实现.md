# TCP三次握手的实现

这篇我们看看内核中对于 TCP 三次握手的实现。握手的原理不多说了，我们主要着眼于具体的实现流程，包括相关的数据结构，如何维护状态的变迁等。

<br>

## 1. 相关的数据结构和函数
----
<br>

之前写过一篇关于 [socket 创建](https://gitee.com/oceanwave/opensource_code_analysis/blob/master/linux-4.19.180/socket%E7%9A%84%E5%88%9B%E5%BB%BA%E4%B8%8Elinux%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F.md)相关的文章，主要是是从文件系统的角度看 socket。所以有些数据结构没有提到或是一扫而过。在 tcp 实际的实现中，有些核心的数据结构需要提前了解下：

![struct_relation](resources/sock_struct_relationship.png)

<br>

- **struct socket**

    用户层通过调用 socket() 函数就会创建如下结构

    ``` cpp
    struct socket {
        // 注意这里的状态不是 TCP 变迁的状态，而是表示当前是否已经连接，是否正在连接
        socket_state		state;
        short			type;   // 通俗的说就是 TCP 还是 UDP ，还是其它
        unsigned long		flags;  // 阻塞非阻塞等
        struct socket_wq	*wq;    //  socket 本身也是事件源，所有监听 socket 事件发生的task 会放到这里
        struct file		*file;         // 作为一个特殊的文件，socket 会与一个 struct file关联，同时会分配对应的 fd
        struct sock		*sk;            // 这个是 网络数据存储的核心
        const struct proto_ops	*ops;   // 定义 socket 操作的接口集合，比如 bind, listen 等，不同的协议对应不同的实现。
    };
    ```

- **struct sock**

    这个接口太庞大了，包括接收缓冲区，监听队列(backlog), 发送缓冲区等等。总之，网络层的数据基本都在这里了。

- **struct sk_buff**
  
    这个是存储包的一个结构，驱动层收到一个网络包后，会创建这个结构并填充数据，如下:
    |收包流程|sk_buff结构|
    |-|-|
    |![flow](resources/receiving_pack_sbk.jpg)|![struct](resources/sk_buff.jpg)|
  
    
<br>

我们以 server 端的角度，来看三次握手，所以分析源码要从内核中 tcp 层收到数据开始，也就是 TCP 处理逻辑的入口。以 ipv4 为例，其入口是 ```tcp_v4_rcv(struct sk_buff *skb) (net/ipv4/tcp_ipv4.c1695)```

在开始之前，我们需要先看一下 ```tcp_v4_rcv``` 里边的 ```__inet_lookup_skb()``` 函数：

``` cpp
int tcp_v4_rcv(struct sk_buff *skb)
{
    ......

    // 以下两个分别保存了 TCP header  he IP header 
	th = (const struct tcphdr *)skb->data;
	iph = ip_hdr(skb);
lookup:

    // 
	sk = __inet_lookup_skb(&tcp_hashinfo, skb, __tcp_hdrlen(th), th->source,
			       th->dest, sdif, &refcounted);
	if (!sk)
		goto no_tcp_socket;

    ......
}
```

这个函数的逻辑是： 根据 sk_buff 获取对应的 socket 结构对象，是一个反向查找的过程。因为 ```struct sock``` 里边存了当前连接的状态，TCP 的处理是一个状态机，当一个网络包过来的时候，就可以根据 ```struct sock``` 的状态进行处理了。

这个方法的是实现是查找两个状态的 ```struct sock```， 分别是已经连接的和正在监听的：

``` cpp
static inline struct sock *__inet_lookup(struct net *net,
					 struct inet_hashinfo *hashinfo,
					 struct sk_buff *skb, int doff,
					 const __be32 saddr, const __be16 sport,
					 const __be32 daddr, const __be16 dport,
					 const int dif, const int sdif,
					 bool *refcounted)
{
	u16 hnum = ntohs(dport);
	struct sock *sk;

	sk = __inet_lookup_established(net, hashinfo, saddr, sport,
				       daddr, hnum, dif, sdif);
	*refcounted = true;
	if (sk)
		return sk;
	*refcounted = false;
	return __inet_lookup_listener(net, hashinfo, skb, doff, saddr,
				      sport, daddr, hnum, dif, sdif);
}
```


<br>

## 2. 响应 client 的 SYN
----
<br>

由第一部分可以知道，第一步是根据数据包的信息来反向查找对应的 sock 结构，如果没有找到则说明包不需要处理，直接丢弃。

因为我们分析的是三次握手，server 端第一次收到的包必然是 SYN 包，此时还没有建立连接，所以 ```__inet_lookup_skb()``` 会返回处于 listen 状态的 ```struct sock```。

``` cpp
int tcp_v4_rcv(struct sk_buff *skb)
{
    // 查找 listen sock struct
	th = (const struct tcphdr *)skb->data;
	iph = ip_hdr(skb);
lookup:
	sk = __inet_lookup_skb(&tcp_hashinfo, skb, __tcp_hdrlen(th), th->source,
			       th->dest, sdif, &refcounted);
	if (!sk)
		goto no_tcp_socket;


    // 这个才是我们 listen 状态的 sock 要经过的逻辑
	if (sk->sk_state == TCP_LISTEN) {
		ret = tcp_v4_do_rcv(sk, skb);
		goto put_and_return;
	}

put_and_return:
	if (refcounted)
		sock_put(sk);

	return ret;
}
```

找到了入口，我们来看下响应 SYN 的调用栈：

``` cpp
// net/ipv4/tcp_ipv4.c:1695
tcp_v4_rcv()
    |
    |-> tcp_v4_do_rcv(sk, skb)
        |
        |-> tcp_checksum_complete(skb)
        |-> tcp_v4_cookie_check(skb)    // 预防 syn-flood 攻击
        |-> tcp_rcv_state_process(sk, skb)
            |
            |-> // 调用的是函数指针，icsk->icsk_af_ops->conn_request(sk, skb)，实际是如下这个函数
                tcp_conn_request()   // net/ipv4/tcp_input.c:6430
```

其中处理 syn 的主要逻辑在 ```tcp_conn_request()``` 函数里边。

``` cpp
int tcp_conn_request(struct request_sock_ops *rsk_ops,
		     const struct tcp_request_sock_ops *af_ops,
		     struct sock *sk, struct sk_buff *skb)
{
	struct tcp_fastopen_cookie foc = { .len = -1 };
	__u32 isn = TCP_SKB_CB(skb)->tcp_tw_isn;
	struct tcp_options_received tmp_opt;
	struct tcp_sock *tp = tcp_sk(sk);
	struct net *net = sock_net(sk);
	struct sock *fastopen_sk = NULL;
	struct request_sock *req;
	bool want_cookie = false;
	struct dst_entry *dst;
	struct flowi fl;

    // TODO: 如果 **半连接** 队列满了的话。可以使用 syncookies 机制继续响应 client 的请求
    // 而不是直接丢弃
	if ((net->ipv4.sysctl_tcp_syncookies == 2 ||
	     inet_csk_reqsk_queue_is_full(sk)) && !isn) {
		want_cookie = tcp_syn_flood_action(sk, skb, rsk_ops->slab_name);
		if (!want_cookie)
			goto drop;
	}

    // **全连接** 队列，这个也就是 listen 中 backlog 的值
	if (sk_acceptq_is_full(sk)) {
		NET_INC_STATS(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
		goto drop;
	}

    // 创建一个新的 request
	req = inet_reqsk_alloc(rsk_ops, sk, !want_cookie);
	if (!req)
		goto drop;


	if (fastopen_sk) {
	} else {
		tcp_rsk(req)->tfo_listener = false;

        // 把这个新的 requst（半连接）放到队列里边， 注意req 的状态会设置成 TCP_NEW_SYN_RECV
        // 并且添加一个超时定时器
		if (!want_cookie)
			inet_csk_reqsk_queue_hash_add(sk, req,
				tcp_timeout_init((struct sock *)req));
		
        // 给 client 返回 syn 和 ack
        af_ops->send_synack(sk, dst, &fl, req, &foc,
				    !want_cookie ? TCP_SYNACK_NORMAL :
						   TCP_SYNACK_COOKIE);
		
        // 如果使用了cookie 的话是不需要存储的，减轻了 syn-flood 攻击的影响
        if (want_cookie) {
			reqsk_free(req);
			return 0;
		}
	}
	reqsk_put(req);
	return 0;
}
```

总结一下：

当 server 端收到 sync 的时候，正常流程必然会发 syn + ack。除此之外，这里涉及到了两个队列：

1. 半连接队列

    这个其实就是代码中的存储 ```struct request_sock``` 的队列，保存着还没有收到 client ACK 的所有连接请求
    
    其获取方式是 ```reqsk_queue_len(&inet_csk(sk)->icsk_accept_queue)```

2. 已连接(全连接)队列
   
    这个是已经连接建立好的队列

    其获取方式是 ```sk->sk_ack_backlog```

注意代码中两个队列获取的方式不同, 半连接队列是把 sk 指针向上转成了父类 inet_connection_sock 的类型

<br>

## 3. 防范 SYN flood 攻击
----
<br>

在上述分析中，内核实现了 syncookie 机制，这是预防 [syn-flood 攻击](https://en.wikipedia.org/wiki/SYN_flood)的手段。

其基本思想，就是把 半连接状态(初始序列号) 的存储由 server 本地 转移到了 cookie 里边。这样再多的 syn 请求也不会持续占用 server 的存储空间了。玩法是这样的：

1. client 发送 SYN 给 server

2. server 根据 SYN 包中的一些列数据计算出一个唯一标识 cookie， 连同 syn-ack 一起返回给 client

3. client 回复 ACK 的时候要携带刚才的 cookie

4. server 解码 cookie 并校验，如果成功的话，分配资源保存当前连接


那问题来了：

1. sever 端如何知道 client 的 ISN (initial sequence number) 和 本端的 ISN ?
   
    首先本端的初始序列号就是 cookie 本身，也就是 client 回复 ACK 时的确认序列号就是 cookie + 1

    client 端的初始序列号也可以根据 ACK 对导出来， ACK 的序列号-1

2. 那 TCP 握手中的各种选项怎么存储的，比如窗口扩大选项，时间戳等？

    不支持。因为 cookie 本身的限制，这些状态没法存储。

<br>

在内核实现中根据是否使用 cookie，也分成两种逻辑；

1. 不使用 cookie
   
    这种情况就需要我们保存 client 的状态信息到 半连接队列里边，这个队列本身也是要占用资源的，这也就是 syn-flood 攻击的点

2. 使用 cookie
   
    可以理解成 cookie 本身携带了状态信心，也就是状态存到了 cookie 里边，而不是在 server 本地存储。因为不会持续占用 server 本地存储，所以也就化解了 syn-flood 攻击


<br>

## 4. 处理 client 的 ACK
----
<br>

接下来，就是等待 client 的 ACK 包了。我们又要回到 ```tcp_v4_rcv() (net/ipv4/tcp_ipv4.c:1695)``` 这个函数了。

这次我们通过``` __inet_lookup_skb() ``` 查找 sk 的时候，会在半连接队列里边找到状态为TCP_NEW_SYN_RECV

这次的调用栈如下；

``` cpp
tcp_v4_rcv()
    |
    |-> //if (sk->sk_state == TCP_NEW_SYN_RECV)
```

// TODO





<br><br><br>

## 参考资料
----
<br>

* [Receiving TCP Data](https://www.halolinux.us/kernel-architecture/receiving-tcp-data.html)
* [Buffer Management](https://www.slideserve.com/tannar/buffer-management)
* [The Journey of a Packet Through the Linux](https://slidetodoc.com/the-journey-of-a-packet-through-the-linux/)
* [深入浅出TCP中的SYN-Cookies](https://segmentfault.com/a/1190000019292140)