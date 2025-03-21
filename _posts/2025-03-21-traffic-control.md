---
layout:     post
title:      "tc端口流量控制"
subtitle:   "实践是检验的唯一标准"
date:       2025-03-21
author:     "Syrup"
header-style: text
tags:
    - Linux
    - System
    - Traffic Control
---

很早之前做大规模实验时要做流量控制，用linux自带的tc挺方便的，也有一些心得。原来丢在了[csdn](https://blog.csdn.net/UuuuuxxxOwO/article/details/116199766)，帮到了不少人。不过后来csdn自动把我的博客改成了闭源收费版，虽然我不再用了，但我也挺生气的，因为自己的知识产权被csdn拿去白嫖赚钱。好不容易登上去改回了原来的原创开放，但也不打算再管了。这里再上传一份留作备用吧。

## tc端口流量控制

tc真的是个巨坑，搞了一天才明白问题出在哪，记录一下。

## tc前置

首先强调一下，**tc只管发包，不管收包的事**。**tc只管发包，不管收包的事**。**tc只管发包，不管收包的事**。重要的事说三遍，坑就出在这里，很容易给绕晕过去。换言之，tc只管出站（出网卡）的流量，不管进来的流量，当然这句话也不完全是，因为即使是本地回环lo，也是这么一回事。

更具体地说，我们将包简单的解构为：[source-ip] | [source-port] | [other-data] | [destination-ip] | [destination-port]

这对后面的说明和理解会有较大的帮助。这个包显然是由 source-ip 机器的 source-port 发送出去的，经由的是 source 机器的网卡。**因此如果要做限流，在 destination 机器上的网卡设置规则是没有意义的，只能在 source 机器的网卡上限制规则。**

## 具体操作

首先简单了解一下tc的三大功能，其他可以忽略，当然要想深入可以去看参考资料的demo。这三个就可以简单完成端口流量控制的目标了，重点其实是后两个。

1. tc-qdisc：队列规则。后面用的是htb层次令牌桶的队列。队列是用来管理消息进出和分发的。
2. tc-class：定义队列规则类。每个类可以简单理解成一个子队列，有着独立的规则。
3. tc-filter：定义过滤器。端口控流就用得到了，如果和iptables配合还可以专门过滤出带有特殊标志的包。实际上就是将包过滤到特定class的队列去。

来一遍实操就懂了，这里默认对外的网卡是`enp8s0`，本地回环是`lo`，这两个都可以实现端口控流。

先删除root已有的队列规则（del是delete，root是根队列，dev是device，enp8s0是网卡名，可以通过ip addr查看）：

`sudo tc qdisc del root dev enp8s0`

重建root队列规则（handle可以简单理解为处理句柄，类似C那种，1:其实是1:0，即缺省是0，所以这个队列的‘名字’就是1:0了，htb是层次令牌桶，即所用的队列类型，default 3指的是如果进来的包没有指定要去的子类，就默认去1:3子类的队列，显然，这里的1是一个大类的公共前缀，而0和3就是小类或者称父类、子类的专属后缀）：

`sudo tc qdisc add root dev enp8s0 handle 1: htb default 3`

在这个队列的基础上建一个子类，设置带宽上限实现该子类的控流（parent用于指定父类，classid与上面的handle类似，用于指定这个子类的名字，rate用于控流，由于ceil和rate默认是一致的，所以如果只是控制流量稳定在一个值，可以不用额外再写ceil，这里控制带宽上限为20mbps）：

`sudo tc class add dev enp8s0 parent 1: classid 1:1 htb rate 20mbit`

接着可以在这个子类上再建两个子类（注意带宽不能超过父类1:1的20mbps）：

`sudo tc class add dev enp8s0 parent 1:1 classid 1:2 htb rate 10mbit`

`sudo tc class add dev enp8s0 parent 1:1 classid 1:3 htb rate 5mbit`

到这里就可以知道，包如果分发给了1:2的子类队列，带宽最高是10mbps，如果没有分发给特定的子类队列，那就会按照默认设置的3，分发给1:3的子类队列，带宽最高是5mbps。

如何分发就涉及到了过滤器了（protocol是协议类别，ip就可以了，其他还有指定tcp，udp等。prio是优先级别。sport是source port的意思，也可以设定dport，即destination port。0xffff是用于设置端口范围的，或者说偏移量，这里的效果是单个端口。flowid就是满足过滤器条件的包分发到哪个流，这里将包分发到1:2的子类队列）：

`sudo tc filter add dev enp8s0 protocol ip parent 1:0 prio 1 u32 match ip sport 12345 0xffff flowid 1:2`

到这里就完成了12345端口的限流，不过注意是**发出的包**限流，而不是**收到的包**限流。如果要将收到的包限流，只有两种方式，1）一个是做一个中转网卡，让那个网卡收到的包再发给enp8s0这个网卡，然后将规则设定到那个中转网卡上，那么**本质上控制的就是中转网卡的发包，进而影响到enp8s0的收包**。2）在对接的服务端的网卡上设置规则，本地客户端收到的包自然也就是限流后的了。

至于过滤器的sport和dport的意义，我们还是拿之前提到的包的结构来说明。比如`sport 12345`指的是这个包是从本地的12345端口发出的包，满足这类条件的包就能通过这个过滤器。而`dport 12345`指的是这个包是本地发出的包（不管是哪个端口的），且包发往的目的端口是12345，满足这类条件的包可通过过滤器，而不是本地收到的发到12345端口的包，即使是本地发到本地12345端口的包，也是在发送的时候就限流了。此外过滤器还能设置目的ip，只要是发往那个ip地址的包就会通过过滤器，这里就不细说了。可以发现这里提到的全部都是发包，这也就是为什么前面一直在强调tc影响的是发包而不是收包了。

## 测试结果

前置环境装个iperf3就可以了：`sudo apt-get install iperf3`

我们假定有两个机器，ip地址分别是`192.168.1.101`和`192.168.1.102`。

将上述tc规则**设置在101机器**的enp8s0网卡上。

接着在**101机器上**的12345端口设置**服务端**（s是server，p是port）：

pc101: `iperf3 -s -p 12345`

在**102机器上**设置**客户端**，并从服务端收取数据（c是client，后接对接服务端的ip。R代表反向，即从服务端接收数据，不加-R选项则是客户端往服务端发数据。i是interval，代表发送间隔，t是time，代表测试时长）：

pc102: `iperf3 -c 192.168.1.101 -p 12345 -R -i 1 -t 3`

可以发现从`192.168.1.101`的`12345`端口发出的包被限流到10mbps了。iperf3不能设置client的端口，所以这里用了`-R`来间接完成从12345端口发包。

当然也可以把`12345`端口设为目的端口，即之前tc的filter的条件设为（将`sport`改成`dport`，还是在**101机器上设置**，记得先用`tc filter delete`删掉原有条件）：

pc101: `sudo tc filter add dev enp8s0 protocol ip parent 1:0 prio 1 u32 match ip dport 12345 0xffff flowid 1:2`

不过这时候反过来，让**102机器**作为服务端，并设置端口为12345。

pc102: `iperf3 -s -p 12345`

然后用**101**机器作为客户端发送数据到服务端，测试带宽（不用带`-R`选项了！）：

pc101: `iperf3 -c 192.168.1.102 -p 12345 -i 1 -t 3`

可以发现**发包带宽**也是被限制在10mbps左右（肯定是会有波动的，但会在10mbps附近一个小范围内）。

不管是哪种方式，tc限流最终都是落在**发包**上。

### 在一台机器上做tc限流

跟之前提到的具体操作几乎没有任何区别，只需要把设备名从`enp8s0`改成`lo`即可，因为本地端口之间的通信走的都是`lo`回环。

设置完tc规则和filter条件后，用两个终端或者tmux开启多窗口的方式来用iperf3分别开服务端和客户端。

后续操作和原理与之前在两台机器上的完全一致，就不多说了。

## 查看网卡的规则

再给一些查看网卡规则的指令，有时候也可以确认规则是否生效，以及流量是否被分配到某一个子类规则队列中，流量和包数有多少等等。

简单查看（设备可以换成lo等等，查看的除了qdisc也可以换成class，filter等）：

`tc qdisc ls dev enp8s0`

详细查看（还可以看class的流量和包数等）：

`tc -s qdisc ls dev enp8s0`

## 其他-利用iptables完成相同工作

也可以用iptables完成分发，前面步骤还是一样的，只有最后的filter不太一样（这里的`handle 10`以及后续的参数指的是将标志为10的包分发给classid为1:2的flow）：

`tc filter add dev enp8s0 parent 1:0 protocol ip prio 1 handle 10 fw classid 1:2`

然后用iptables对指定端口的包设置特殊的标志，完成端口绑定tc队列（只要保证`--set-mark`的值和`handle`的值一致就可以了）。

`iptables -A OUTPUT -t mangle -p tcp --sport 12345 -j MARK --set-mark 10`

由于这些规则还是基于本机的tc设置的，因此控制的还是**发包**（`sport`指定的是本机发包的source端口，即使改成`dport`也是这个包发往的目的端口号）。

## 参考资料

* [tc控制端口带宽](https://blog.csdn.net/cleanarea/article/details/107760143)
* [tc高级控流demo](https://wiki.archlinux.org/index.php/advanced_traffic_control#Stochastic_Fairness_Queueing_%28SFQ%29)
* [tc参数/manpage](https://linux.die.net/man/8/tc)
