---
layout: page
title: A Receiver-Driven Low-Latency Transport Protocol
tags: [Transport Protocol, Data Center, Network]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Homa -- A Receiver-Driven Low-Latency Transport Protocol Using Network Priorities 

### 0x00 引言

 最近几年为数据中心设计的新的传输协议不少，这篇是SIGCOMM上最新的一篇(截止写这篇论文时)，总而言之，这篇论文做到了:

```
In simulations, Homa’s latency is roughly equal to pFabric and significantly better than pHost, PIAS, and NDP for almost all message sizes and workloads. Homa can also sustain higher network loads than pFabric, pHost, or PIAS.
-----
Our implementation of Homa achieves 99th percentile round trip latencies less than 15 μs for small messages at 80% network load with 10 Gbps link speeds, and it does this even in the presence of competing large messages. Across a wide range of message sizes and work- loads, Homa achieves 99th percentile latencies at 80% network load that are within a factor of 2–3.5x of the minimum possible latency on an unloaded network. 
```

Homa倾向于为short messages设计。

### 0x01 KEY IDEAS 

Homa的设计有4个key design principles ：

```
(i) transmitting short messages blindly;

(ii) using in-network priorities;

(iii) allocating priorities dynamically at receivers in conjunction with receiver-driven rate control;

(iv) controlled overcommitment of receiver downlinks.
```

Homa这么设计出于一下的考虑：

```
1. There is no time to schedule every packet. 
```

 特别在意延时的情况下，schedule带来的延时都不可以接受。所以有了principle 1。

```
2. Buffering is a necessary evil.
```

没有一个protocol在没有导致buffering的同时实现low latency，讽刺的是，buffering又会带来latency。buffer，latency，throughput，欢喜冤家，emmmmmmm这里是不是可以写一本书了。

```
3. In-network priorities are a must. 
```

由于上一条，为了减少延时，不同的packet区分处理能获得一些效果。这个方法在很多类似的protocol中都有使用。这样就有了principle 2.

```
4. Making best use of limited priorities requires receiver control.
```

 简而言之就是recevier控制更加好。

```
5. Receivers must allocate priorities dynamically.
```

 Homa 使用Receivers动态分配优先级的方式解决了之前类似协议的一些问题(pHost )，比如large的messag使用高优先级带来的问题，只使用一种优先级可能导致的delay。这样就有了principl 3。

```
6. Receivers must overcommit their downlink in a controlled manner.
```

为了解决一些情况下链路利用率低的问题，比如一个sender项多个recevier发送数据(注意Homa使用的是receiver控制的传输方式)。为了解决这个问题，一个receiver可以过量使用downlink，比如同时给几个sender发送可以向receiver发送数据的grants。这样可能造成packet queuing ，但是对于提高利用率来说是必要的。

```
7. Senders need SRPT(shortest remaining processing time first) also. 
```

 排队也可能在sender端出现，Sender知道SRPT能跟好的解决这些问题。

### 0x02 基本设计

先来一张论文中的图：

<img src="/assets/img/homa-arch.png" alt="homa-arch" style="zoom: 33%;" />

Homa有以下特点：

```
Homa contains several unusual features: 
it is receiver-driven; 
it is message-oriented, rather than stream-oriented; 
it is connectionless; 
it uses no explicit acknowledgments; 
and it implements at-least-once semantics, rather than the more traditional at-most-once semantics.
```

#### RPCs, not connections 

 Homa是无连接的，一个来讲client的request message 对应一个来自server的 response message。由一个全局唯一的RPCid表示(id由客户端生成)。有以下的packet类型：

<img src="/assets/img/homa-packet-types.png" alt="homa-packet-types" style="zoom: 33%;" />

#### Basic sender behavior 

   Homa将Message分为两部分，一部分是unscheduled 的部分，这部分是立即会被发送的，可能有一个or多个packet。第二个部分是unscheduled 的部分，这部分只有在收到receiver的GRANT 包之后才能发送。每一个DATA 包都有一个优先级。这个优先级由receiver决定。Sender也使用了SRPT，当多个message的DATA包同时准备好发送的时候，决定先发送的是有最少剩余数据的message的包。sender 不根据优先级决定那些包先发送(由此可以看出，优先级只对路由器产生作用)。此外，控制类型的包，如GRANT and RESEND包的优先级总是高于DATA包。

#### Packet priorities 

   从论文中的描述看，优先级的部分是这个transport protocol的核心的一个部分：

```
The most novel feature in Homa, and the key to its performance, is its use of priorities. Each receiver determines the priorities for all of its incoming DATA packets in order to approximate the SRPT policy. It uses different mechanisms for unscheduled and scheduled packets. 
```

  对于unscheduled 的包，receiver提前根据最近的流量模式决定优先级，然后会将顺带地这些信息发送给sender(by piggybacking it on other packets )，sender会报送最近的每一个receiver的相关信息。

<img src="/assets/img/homa-unscheduled.png" alt="homa-unscheduled" style="zoom: 33%;" />

  对于scheduled 的包，receiver在GRANT包中给其分配优先级，sender使用这个优先级发送DATA包。这样的好处就是可以实时地适应目前的状况。这些主要是根据SRPT 。

#### Lost packets

 Homa认为包丢失是概率很小的事件。主要关注2种类型的包丢失：

```
corruption in the network, and buffer overflow
```

 Homa中，由receiver发现包丢失(这个与TCP完全不相同，我们可以发送，Homa这个协议将很多东西都放到了receiver这边，这种设计目前看来适应datacenter这样的环境还是很不错的)。

```
 Receivers use a simple timeout-based mechanism to detect lost packets. If a long time period (a few milliseconds) elapses without additional packets arriving for a message, the receiver sends a RESEND packet that identifies the first range of missing bytes; the sender will then retransmit those bytes.
```

#### At-least-once semantics 

   Homa实现的语义是至少一次，我们知道至少一次的基本套路就是一直重试到收到确认为止。Homa又是为RPC设计的，那Homa是如何处理被多次执行的呢？答案很简单粗暴：

```
Homa assumes that higher level software will either tolerate redundant executions of RPCs or filter them out.
```

  说白了就是不管，有Application处理。

#### 其它一些东西

 另外paper中关于 Flow control，Overcommitment ，Incast 的部分可以参看原论文。

### 0x03 LIMITATIONS 

   Paper还很罕见的讨论了这个协议的缺点(暴露自己缺点的paper很少见啊)，我们从之前的内容中也可以发现，Homa协议是建立在很多的假设上的，这也让我对其实际的可用性造成了很大的怀疑，它更加像一个在特定的环境下为特定的应用设计的传输协议，没有通用型可言:

```
  Homa is designed for use in datacenter networks and capital- izes on the properties of those networks; it is unlikely to work well in wide-area networks.
  Homa assumes that congestion occurs primarily at host down- links, not in the core of the network. Homa assumes per-packet spraying to ensure load balancing across core links, combined with sufficient overall capacity. .... We hypothesize that congestion in the core of datacenter networks will be uncommon because it will not be cost-effective. ... 
  ...
  Homa also assumes a single implementation of the protocol for each host-TOR link, such as in an operating system kernel running on bare hardware, so that Homa is aware of all incoming and outgoing traffic.
  ...
  Homa assumes that the most severe forms of incast are pre- dictable because they are self-inflicted by outgoing RPCs;
```

  一堆的假设，看不下去了。。。。。

## 参考

1. Homa: A Receiver-Driven Low-Latency Transport Protocol Using Network Priorities, SIGCOMM 2018;
