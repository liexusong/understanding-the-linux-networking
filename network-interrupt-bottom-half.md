# Linux 网络中断下半部处理

在 [上一篇文章](https://mp.weixin.qq.com/s/u8PgRcMLbYvOC5oAd_fGPA) 中，我们介绍了网卡接收和发过数据在 Linux 内核中的处理过程，我们先来回顾一下网卡接收和发送数据的过程，如 图1 所示：

![](C:\codes\linux-2.4.0\network-docs\images\network-card.png)

图1 网卡接收和发送数据过程

如上图所示，当网卡接收到从网络中发送过来的数据后，网卡会向 CPU 发起一个硬件中断。当 CPU 接收到网卡的硬件中断后，便会调用网卡驱动向内核注册的中断处理服务，如 `NS8390网卡驱动` 会向内核注册 `ei_interrupt` 中断服务。

由于在处理硬件中断服务时会关闭硬件中断，所以在处理硬件中断服务的过程中，如果发生了其他的硬件中断，也不能得到有效的处理，从而导致硬件中断丢失的情况。

为了避免这种情况出现，Linux 内核把中断处理分为：**中断上半部** 和 **中断下半部**，上半部在关闭中断的情况下进行，而下半部在打开中断的情况下进行。

由于中断上半部在关闭中断的情况下进行，所以必须要快速完成，从而避免中断丢失的情况。而中断下半部处理是在打开中断的情况下进行的，所以可以慢慢进行。

一般来说，网卡驱动向内核注册的中断处理服务属于 **中断上半部**，如前面介绍的 `NS8390网卡驱动` 注册的 `ei_interrupt` 中断处理服务，而本文主要分析网卡 **中断下半部** 的处理。

## 数据包上送

在上一篇文章中，我们介绍过 `ei_interrupt`  中断处理服务首先会创建一个 `sk_buff` 数据包对象保存从网卡中接收到的数据，然后调用 `netif_rx` 函数将数据包上送给网络协议栈处理。

我们先来分析一下 `netif_rx` 函数的实现：

```c
int netif_rx(struct sk_buff *skb)
{
    int this_cpu = smp_processor_id(); // 获取当前运行的CPU
    struct softnet_data *queue;
    unsigned long flags;
    ...
    queue = &softnet_data[this_cpu]; // 获取当前CPU的待处理的数据包队列

    local_irq_save(flags); // 关闭本地硬件中断

    // 如果待处理队列的数据包数量没超出限制
    if (queue->input_pkt_queue.qlen <= netdev_max_backlog) {
        if (queue->input_pkt_queue.qlen) {
            ...
enqueue:
            dev_hold(skb->dev); // 增加网卡设备的引用计数器
            __skb_queue_tail(&queue->input_pkt_queue, skb); // 将数据包添加到待处理队列中
            __cpu_raise_softirq(this_cpu, NET_RX_SOFTIRQ);  // 启动网络中断下半部处理
            local_irq_restore(flags);

            return softnet_data[this_cpu].cng_level;
        }
        ...
        goto enqueue;
    }
    ...
drop:
    local_irq_restore(flags); // 打开本地硬件中断
    kfree_skb(skb);           // 释放数据包对象
    return NET_RX_DROP;
}
```

`netif_rx` 函数的参数就是要上送给网络协议栈的数据包，`netif_rx` 函数主要完成以下几个工作：

*   获取当前 CPU 的待处理的数据包队列，在 Linux 内核初始化时，会为每个 CPU 创建一个待处理数据包队列，用于存放从网卡中读取到网络数据包。
*   如果待处理队列的数据包数量没超出 `netdev_max_backlog` 设置的限制，那么调用 `__skb_queue_tail` 函数把数据包添加到待处理队列中，并且调用 `__cpu_raise_softirq` 函数启动网络中断下半部处理。
*   如果待处理队列的数据包数量超出 `netdev_max_backlog` 设置的限制，那么就把数据包释放。

`netif_rx` 函数的处理过程如 图2 所示：

![](C:\codes\linux-2.4.0\network-docs\images\interrupt-bottom-half.png)

图2 `netif_rx` 函数的处理过程

所以，`netif_rx` 函数的主要工作就是把接收到的数据包添加到待处理队列中，并且启动网络中断下半部处理。

对于 Linux 内核的中断处理机制可以参考我们之前的文章 [Linux中断处理](https://mp.weixin.qq.com/s/oTz15wPgximSOpoSWL_j4Q)，这里就不详细介绍了。在本文中，我们只需要知道网络中断下半部处理例程为 `net_rx_action` 函数即可。

## 网络中断下半部处理

上面说了，网络中断下半部处理例程为 `net_rx_action` 函数，所以我们主要分析 `net_rx_action` 函数的实现：

```c
static void net_rx_action(struct softirq_action *h)
{
    int this_cpu = smp_processor_id();                    // 当前运行的CPU
    struct softnet_data *queue = &softnet_data[this_cpu]; // 当前CPU对于的待处理数据包队列
    ...
    for (;;) {
        struct sk_buff *skb;

        local_irq_disable();
        skb = __skb_dequeue(&queue->input_pkt_queue); // 从待处理数据包队列中获取一个数据包
        local_irq_enable();

        if (skb == NULL)
            break;

        ...
        {
            struct packet_type *ptype, *pt_prev;
            unsigned short type = skb->protocol; // 网络层协议类型

            pt_prev = NULL;
            ...
            // 使用网络层协议处理接口处理数据包
            for (ptype = ptype_base[ntohs(type)&15]; ptype; ptype = ptype->next) {
                if (ptype->type == type
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

            if (pt_prev) {
                // 如处理IP协议数据包的 ip_rcv() 函数
                pt_prev->func(skb, skb->dev, pt_prev);
            } else
                kfree_skb(skb);
        }
        ...
    }
    ...
    return;
}
```

`net_rx_action` 函数主要完成以下几个工作：

*   从待处理数据包队列中获取一个数据包，如果数据包为空，那么就退出网络中断下半部。

*   如果获取的数据包不为空，那么就从数据包的以太网头部中获取到网络层协议的类型。然后根据网络层协议类型从 `ptype_base` 数组中获取数据处理接口，再通过此数据处理接口来处理数据包。

    在内核初始化时，通过调用 `dev_add_pack` 函数向 `ptype_base` 数组中注册网络层协议处理接口，如 `ip_init` 函数：

    ```c
    static struct packet_type ip_packet_type = {
        __constant_htons(ETH_P_IP),
        NULL,
        ip_rcv,  // 处理IP协议数据包的接口
        (void*)1,
        NULL,
    };
    
    void __init ip_init(void)
    {
        // 注册网络层协议处理接口
        dev_add_pack(&ip_packet_type);
        ...
    }
    ```

    

    >   ```c
    >   
    >   ```

所以，`net_rx_action` 函数主要从待处理队列中获取数据包，然后根据数据包的网络层协议类型，找到相应的处理接口处理数据包。其过程如 图3 所示：

![](C:\codes\linux-2.4.0\network-docs\images\interrupt-bottom-half-2.png)

从上图可知，`net_rx_action` 函数将数据包交由网络层协议处理接口后就不管了，而网络层协议处理接口接管数据包后，会对数据包进行进一步处理，如判断数据包的合法性（数据包是否损坏、数据包是否发送给本机）。如果数据包是合法的，就会交由传输层协议处理接口处理。

## 总结

本文主要介绍了网络中断下半部的处理，从分析可知，网络中断下半部主要工作是从待处理队列中获取数据包，并且根据数据包的网络层协议类型来找到相应的处理接口，然后把数据包交由网络层协议处理接口进行处理。

对于网络层协议处理接口的相关过程，我们将会在后面的文章继续分析。

