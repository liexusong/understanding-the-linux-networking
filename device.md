# Linux 网卡数据收发过程分析

一般来说，网卡主要有两个重要的功能：**接收数据** 和 **发送数据**。

所以，当网卡接收到数据包后，要通知 Linux 内核有数据需要处理。另外，网卡驱动应该提供让 Linux 内核把数据把发送出去的接口。

`net_device` 结构是 Linux 为了适配不同类型的网卡设备而抽象出来的对象，不同的网卡驱动只需要按 Linux 的规范来填充 `net_device` 结构的各个成员变量，Linux 内核就能够识别出网卡，并工作起来。

下面我们将分析网卡设备接收和发送数据包的实现原理。

## net_device 结构

`net_device` 结构是 Linux 内核对网卡设备的抽象，但由于历史原因，`net_device` 结构的定义十分复杂。

不过本文主要分析网卡设备收发数据的实现，所以不会分析 `net_device` 结构的所有成员。下面主要列出收发数据相关的成员，如下：

```c
struct net_device
{
    char                name[IFNAMSIZ];  // 设备名字
    ...
    unsigned int        irq;             // 中断号
    ...
    int (*init)(struct net_device *dev); // 设备初始化设备的接口
    ...
    int (*open)(struct net_device *dev); // 打开设备时调用的接口
    int (*stop)(struct net_device *dev); // 关闭设备时调用的接口

    // 发送数据接口
    int (*hard_start_xmit)(struct sk_buff *skb,struct net_device *dev);
    ...
};
```

下面介绍一下各个成员的作用：

*   `name`：设备的名字。用于在终端显示设备的名字或者通过设备名字来搜索设备。
*   `irq`：中断号。当网卡从网络接收到数据包后，需要产生一个中断来通知 Linux 内核有数据包需要处理，而 `irq` 就是网卡驱动注册到内核中断服务的中断号。
*   `init`、`open`、`stop`：分别为设备的初始化接口，打开接口和关闭接口。
*   `hard_start_xmit`：当需要通过网卡设备发送数据时，可以调用这个接口来发送数据。

所以，一个网卡驱动必须完成以下两个工作：

*   通过实现 `net_device` 结构的 `hard_start_xmit` 方法来提供发送数据的功能。
*   通过向内核注册硬件中断服务，来通知内核处理网卡设备接收到的数据包。

也就是说，发送数据的功能是由 `net_device` 结构的 `hard_statr_xmit` 方法提供，而通知内核处理接收到的数据包的功能是由网卡的硬件中断提供的。

**图1** 展示了网卡接收和发送数据的过程：

![network-card](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/network-card.png)

**图1** 网卡接收和发送数据过程

上图展示的是 `NS8390网卡` 接收和发送数据的过程（红色括号为接收过程，蓝色括号为发送过程），从上图可以发现，`NS8390网卡驱动` 完成了两件事情：

*   将 `net_device` 结构的 `hard_start_xmit` 方法设置为 `ei_start_xmit`。
*   向 Linux 内核注册了 `ei_interrupt` 硬件中断服务。

所以，当网卡接收到数据包时，会触发 `ei_interrupt` 中断服务来通知内核有数据包需要处理。而当需要通过网卡发送数据时，将会调用 `ei_start_xmit` 方法把数据发送出去。

## 接收数据过程

当网卡从网络中接收到数据包后，会触发 `ei_interrupt` 中断服务，我们来看看 `ei_interrupt` 中断服务的实现：

```c
void ei_interrupt(int irq, void *dev_id, struct pt_regs *regs)
{
    struct net_device *dev = dev_id;
    long e8390_base;
    int interrupts, nr_serviced = 0;
    struct ei_device *ei_local;

    e8390_base = dev->base_addr;
    ei_local = (struct ei_device *)dev->priv;

    spin_lock(&ei_local->page_lock);
    ...
    // (1) 通过读取网卡的中断类型来进行相应的操作
    while ((interrupts = inb_p(e8390_base + EN0_ISR)) != 0 
            && ++nr_serviced < MAX_SERVICE)
    {
        ...
        // (2) 如果中断类型为接收到数据包
        if (interrupts & (ENISR_RX + ENISR_RX_ERR)) {
            ei_receive(dev); // (3) 则从网卡读取数据
        }
        ...
    }
    ...
    spin_unlock(&ei_local->page_lock);
    return;
}
```

上面的代码删除了很多硬件相关的操作，因为本文并不是分析网卡驱动的实现。

`ei_interrupt` 中断服务首先读取中断的类型，保存到 `interrupts` 变量中。然后判断中断类型是否为接收到数据包，如果是就调用 `ei_receive` 函数从网卡处读取数据。

我们继续分析 `ei_receive` 函数的实现：

```c
static void ei_receive(struct net_device *dev)
{
    ...
    while (++rx_pkt_count < 10) 
    {
        int pkt_len;  // 数据包的长度
        int pkt_stat; // 数据包的状态
        ...
        if ((pkt_stat & 0x0F) == ENRSR_RXOK) { // 如果数据包状态是合法的
            struct sk_buff *skb;

            skb = dev_alloc_skb(pkt_len + 2); // 申请一个数据包对象
            if (skb) {
                skb_reserve(skb, 2);
                skb->dev = dev;         // 设置接收数据包的设备
                skb_put(skb, pkt_len);  // 增加数据的长度

                // 从网卡中读取数据(由网卡驱动实现), 并将数据保存到skb中
                ei_block_input(dev, pkt_len, skb, current_offset+sizeof(rx_frame));

                skb->protocol = eth_type_trans(skb, dev); // 从以太网头部中获取网络层协议类型

                netif_rx(skb); // 将数据包上送给内核网络协议栈
                ...
            }
        }
        ...
    }
    ...
    return;
}
```

`ei_receive` 函数主要完成以下几个工作：

*   申请一个 `sk_buff` 数据包对象，并且设置其 `dev` 字段为接收数据包的设备。
*   通过调用 `ei_block_input` 函数从网卡中读取接收到的数据，并保存到刚申请的 `sk_buff` 数据包对象中。`ei_block_input` 函数是由网卡驱动实现的，所以这里不作详细分析。
*   通过调用 `eth_type_trans` 函数从数据包的以太网头部中获取网络层协议类型。
*   调用 `netif_rx` 函数将数据包上送给内核网络协议栈。

当把数据包上送给内核网络协议栈后，数据包的处理就由内核接管。一般来说，内核网络协议栈会通过网络层的 `IP协议` 和传输层的 `TCP协议` 或者 `UDP协议` 来对数据包进行处理，处理完后就会把数据提交给应用层的进程进行处理。

## 发送数据过程

当网络协议栈需要通过网卡设备发送数据时，会调用 `net_device` 结构的 `hard_start_xmit` 方法，而对于 `NS8390网卡` 来说，`hard_start_xmit` 方法会被设置为 `ei_start_xmit` 函数。

也就是说，使用 `NS8390网卡` 发送数据时，最终会调用 `ei_start_xmit` 函数将数据发送出去。我们来看看 `ei_start_xmit` 函数的实现：

```c
static int ei_start_xmit(struct sk_buff *skb, struct net_device *dev)
{
    ...
    length = skb->len; // 数据包的长度
    ...
    disable_irq_nosync(dev->irq);      // 关闭硬件中断
    spin_lock(&ei_local->page_lock);   // 对设备进行上锁, 避免多核CPU对设备的使用
    ...
    // 使用网卡驱动的发送接口把数据发送出去，skb->data 为数据包的数据部分
    ei_block_output(dev, length, skb->data, ei_local->tx_start_page);
    ...
    spin_unlock(&ei_local->page_lock); // 对设备进行解锁
    enable_irq(dev->irq);              // 打开硬件中断
    ...
    return 0;
}
```

删减了硬件相关的操作后，`ei_start_xmit` 函数的实现就非常简单：

*   首先关闭网卡的硬件中断，防止发送过程中受到硬件中断的干扰。
*   调用 `ei_block_output` 函数把数据包的数据发送出去，此函数由网卡驱动实现，这里不作详细分析。
*   打开网卡的硬件中断，让网卡能够继续通知内核。

## 总结

本文主要简单的介绍了网卡设备接收和发送数据包的过程，而网卡设备的初始化过程并没有涉及。当然网卡设备的初始化过程也非常重要，可能会在后面的文章继续分析。
