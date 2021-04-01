# IP数据包接收过程

在《网络中断下半部处理》一文中介绍过，当网卡接收到网络数据包后，会由网卡驱动通过调用 `netif_rx` 函数把数据包添加到待处理队列中，然后唤起网络中断下半部处理。

而网络中断下半部处理由 `net_rx_action` 函数完成的，其主要功能就是从待处理队列中获取一个数据包，然后根据数据包的网络层协议类型来找到相应的处理接口来处理数据包。我们通过 图1 来展示 `net_rx_action` 函数的处理过程：

![net_rx_action](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/net_rx_action.png)

图1 `net_rx_action` 处理过程

网络层的处理接口保存在 `ptype_base` 数组中，其元素的类型为 `packet_type` 结构，而索引为网络层协议的类型，所以通过网络层协议的类型就能找到对应的处理接口，我们先来看看 `packet_type` 结构的定义：

```c
struct packet_type
{
    unsigned short      type;   // 网络层协议类型
    struct net_device   *dev;   // 绑定的设备对象, 一般设置为NULL
    int                 (*func)(struct sk_buff *, struct net_device *, struct packet_type *); // 处理接口
    void                *data;  // 私有数据域, 一般设置为NULL
    struct packet_type  *next;
};
```

`ptype_base` 数组的定义如下：

```c
static struct packet_type *ptype_base[16];
```

从 `ptype_base` 数组的定义可知，其只有 16 个元素，那么如果内核超过 16 种网络层协议怎么办？我们可以从网络层处理接口的注册函数 `dev_add_pack` 中找到答案：

```c
void dev_add_pack(struct packet_type *pt)
{
    int hash;

    br_write_lock_bh(BR_NETPROTO_LOCK);

    // 如果类型为ETH_P_ALL, 表示需要处理所有协议的数据包
    if (pt->type == htons(ETH_P_ALL)) { 
        netdev_nit++;
        pt->next = ptype_all;
        ptype_all = pt;
    } else {
        // 对网络层协议类型与15进行与操作, 得到一个 0 到 15 的整数
        hash = ntohs(pt->type) & 15;

        // 通过 next 指针把冲突的处理接口连接起来
        pt->next = ptype_base[hash];
        ptype_base[hash] = pt;
    }

    br_write_unlock_bh(BR_NETPROTO_LOCK);
}
```

从 `dev_add_pack` 函数的实现可知，内核把 `ptype_base` 数组当成了哈希表，而键值就是网络层协议类型，哈希函数就是对协议类型与 15 进行与操作。如果有冲突，就通过 `next` 指针把冲突的处理接口连接起来。

我们再来看看 IP 协议是怎样注册处理接口的，如下代码：

```c
/* /net/ipv4/ip_output.c */

static struct packet_type ip_packet_type = {
     __constant_htons(ETH_P_IP),
     NULL,
     ip_rcv,  // 处理 IP 协议数据包的接口
     (void*)1,
     NULL,
 };
 
 void __init ip_init(void)
{
     // 注册 IP 协议处理接口
     dev_add_pack(&ip_packet_type);
     ...
 }
```

从上面的代码可以看到，在 `ip_init` 函数中调用了 `dev_add_pack(&ip_packet_type)` 函数来注册了 IP 协议的处理接口，而 IP 协议的处理接口为 `ip_rcv` 函数。

我们再看看 `net_rx_action` 函数的处理：

```c
static void net_rx_action(struct softirq_action *h)
{
    ...
    for (;;) {
        ...
        {
            struct packet_type *ptype, *pt_prev;
            unsigned short type = skb->protocol; // 网络层协议类型
 
            pt_prev = NULL;
            ...
            // 从 ptype_base 数组中查找网络层处理接口
            for (ptype = ptype_base[ntohs(type) & 15]; 
                 ptype; 
                 ptype = ptype->next) 
            {
                if (ptype->type == type // 如果协议类型匹配
                    && (!ptype->dev || ptype->dev == skb->dev))
                {
                    if (pt_prev) {
                        atomic_inc(&skb->users);
                        // 如处理IP协议数据包的 ip_rcv() 函数
                        pt_prev->func(skb, skb->dev, pt_prev);
                    }

                    pt_prev = ptype;
                }
            }
            ...
        }
        ...
    }
    ...
    return;
}
```

现在就非常清晰了，就是根据数据包的网络层协议类型，然后从 `ptype_base` 数组中找到对应的处理接口处理数据包，如 IP 协议的数据包就调用 `ip_rcv` 函数处理。

## 处理IP数据包

通过上面的分析，我们知道当内核接收到一个 IP 数据包后，会调用 `ip_rcv` 函数处理这个数据包，下面我们来分析一下 `ip_rcv` 函数的实现：

```c
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt)
{
    struct iphdr *iph = skb->nh.iph;

    // 如果数据包不是发送给本机的, 丢弃数据包
    if (skb->pkt_type == PACKET_OTHERHOST)
        goto drop;

    // 如果其他地方也在使用数据包, 复制一份新的数据包
    if ((skb = skb_share_check(skb, GFP_ATOMIC)) == NULL)
        goto out;

    // 数据包的长度比IP头部的长度还小，不合法丢弃
    if (skb->len < sizeof(struct iphdr) || skb->len < (iph->ihl<<2))
        goto inhdr_error;

    // 判断IP头部是否合法
    if (iph->ihl < 5                                 // IP头部长度是否合法
        || iph->version != 4                         // IP协议版本是否合法
        || ip_fast_csum((u8 *)iph, iph->ihl) != 0)   // IP校验和是否正确
        goto inhdr_error;

    {
        __u32 len = ntohs(iph->tot_len);

        // 如果数据包的长度比IP头部的总长度小, 说明数据包不合法, 需要丢弃
        if (skb->len < len || len < (iph->ihl<<2))
            goto inhdr_error;
        __skb_trim(skb, len);
    }

    // 如果所有验证都通过, 那么调用 ip_rcv_finish 函数继续处理数据包
    return NF_HOOK(PF_INET, NF_IP_PRE_ROUTING, skb, dev, NULL, ip_rcv_finish);

    // 这里是丢弃数据包的处理
inhdr_error:
    IP_INC_STATS_BH(IpInHdrErrors);
drop:
    kfree_skb(skb);
out:
    return NET_RX_DROP;
}
```

`ip_rcv` 函数主要对数据包的合法性进行验证，如果数据包是合法的，那么就调用 `ip_rcv_finish` 函数继续对数据包进行处理。

我们继续分析 `ip_rcv_finish` 函数的实现：

```c
static inline int ip_rcv_finish(struct sk_buff *skb)
{
    struct net_device *dev = skb->dev;
    struct iphdr *iph = skb->nh.iph;

    if (skb->dst == NULL) { // 如果数据包的输入路由缓存还没设置
        // 根据目标IP地址获取数据包的输入路由缓存
        if (ip_route_input(skb, iph->daddr, iph->saddr, iph->tos, dev))
            goto drop;
    }
    ...
    // 如果数据包是发送给本机的，那么就调用 ip_local_deliver 进行处理
    return skb->dst->input(skb); 

drop:
    kfree_skb(skb);
    return NET_RX_DROP;
}
```

为了简单起见，我们去掉了对 IP 选项的处理。在上面的代码中，如果数据包的输入路由缓存还没设置，那么先调用 `ip_route_input` 函数获取数据包的输入路由缓存（`ip_route_input` 函数将会在 `路由子系统` 一章介绍，暂时可以忽略这个函数）。

设置好数据包的路由缓存后，就调用路由缓存的 `input` 方法处理数据包。如果数据包是发送给本机的，那么路由缓存的 `input` 方法将会被设置为 `ip_local_deliver`（由 `ip_route_input` 函数设置）。

所有，如果数据包是发送给本机，那么最终会调用 `ip_local_deliver` 函数处理数据包，我们继续来分析 `ip_local_deliver` 函数：

```c
int ip_local_deliver(struct sk_buff *skb)
{
    struct iphdr *iph = skb->nh.iph;

    // 如果当前数据包是一个IP分片, 那么先对数据包进行分片处理
    if (iph->frag_off & htons(IP_MF|IP_OFFSET)) {
        skb = ip_defrag(skb);
        if (!skb)
            return 0;
    }

    // 接着调用 ip_local_deliver_finish 函数处理数据包
    return NF_HOOK(PF_INET, NF_IP_LOCAL_IN, skb, skb->dev, NULL,
                   ip_local_deliver_finish);
}
```

`ip_local_deliver` 函数首先判断数据包是否为一个 IP 分片（IP 分片将在下一篇文章介绍，暂时可以忽略），如果是就调用 `ip_defrag` 函数对数据包进行分片重组处理。如果数据包不是一个分片或者分片重组成功，那么最终调用 `ip_local_deliver_finish` 函数处理数据包。

`ip_local_deliver_finish` 函数是 IP 层处理数据包的最后一步，我们接着分析 `ip_local_deliver_finish` 函数的实现：

```c
static inline int ip_local_deliver_finish(struct sk_buff *skb)
{
    struct iphdr *iph = skb->nh.iph;         // 获取数据包的IP头部
    ...
    skb->h.raw = skb->nh.raw + iph->ihl * 4; // 设置传输层协议头部指针

    {
        int hash = iph->protocol & (MAX_INET_PROTOS - 1); // 从IP头部获取传输层协议类型
        struct sock *raw_sk = raw_v4_htable[hash];
        struct inet_protocol *ipprot;
        int flag;
        ...
        ipprot = (struct inet_protocol *)inet_protos[hash]; // 传输层协议处理函数
        flag = 0;

        if (ipprot != NULL) { // 调用传输层协议处理函数处理数据包
            if (raw_sk == NULL && ipprot->next == NULL && ipprot->protocol == iph->protocol) {
                return ipprot->handler(skb, (ntohs(iph->tot_len)-(iph->ihl*4)));
            } else {
                flag = ip_run_ipprot(skb, iph, ipprot, (raw_sk != NULL));
            }
        }
        ...
    }
    return 0;
}
```

在上面代码中，我们省略对原始套接字的处理（原始套接字将会在 `原始套接字` 一章中介绍）。`ip_local_deliver_finish` 函数的主要工作如下：

* 通过数据包的 IP 头部获取到上层协议（传输层）类型。

* 根据传输层协议类型从 `inet_protos` 数组中查找对应的处理函数。

* 调用传输层协议的处理函数处理数据包。

`inet_protos` 数组保存了传输层协议的处理函数，其的定义如下：

```c
struct inet_protocol
{
    int (*handler)(struct sk_buff *skb, unsigned short len); // 协议的处理函数
    unsigned char protocol;                                  // 协议类型
    struct inet_protocol *next;                              // 解决冲突
    ...
};

#define MAX_INET_PROTOS	32

struct inet_protocol *inet_protos[MAX_INET_PROTOS];
```

不同的传输层协议处理函数，会根据其协议类型的值保存到 `inet_protos` 数组中。由于 `inet_protos` 数组只有32个元素，所以保存处理函数时，需要将协议值与32进行取模操作，得到一个 0 ~ 31 的值，然后把处理函数保存到 `inet_protos` 数组对应位置上。如果有多个协议发生冲突，那么就通过 `next` 字段连接起来。

通过调用 `inet_add_protocol` 函数，可以向 `inet_protos` 数组注册传输层协议的处理函数。例如 `TCP协议` 的处理函数定义如下：

```c
static struct inet_protocol tcp_protocol = 
{
    tcp_v4_rcv,         /* TCP handler */
    IPPROTO_TCP,        /* protocol ID */
    ...
};
```

所以，当接收到一个 TCP 协议数据包时，将会调用 `tcp_v4_rcv` 函数处理此数据包。

最后，我以一幅图来展示处理 IP 数据包的函数调用链：

![ip-recv-process](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/ip-recv-process.png)

