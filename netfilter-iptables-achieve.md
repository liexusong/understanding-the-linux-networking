# Netfileter 实现

在《[Netfilter & iptables 原理](https://github.com/liexusong/understanding-the-linux-networking/blob/master/netfilter-iptables-principle.md)》一文中，我们介绍了 `Netfilter` 和 `iptables` 的原理，而本文主要通过源码分析来介绍一下 Netfilter 与 iptables 的实现过程。

## 一、Netfilter 挂载点

我们先来回顾一下 Netfilter 的原理，Netfilter 是通过在网络协议栈的不同阶段注册钩子函数来实现对数据包的处理与过滤，如 图1 所示：

![](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/netfilter-hooks.png)

在 图1 中，蓝色部分就是 Netfilter 挂载钩子函数的位置，所以 Netfilter 定义了 5 个常量来表示这 5 个位置，如下代码：

```c
// 文件：include/linux/netfilter_ipv4.h

#define NF_IP_PRE_ROUTING   0
#define NF_IP_LOCAL_IN      1
#define NF_IP_FORWARD       2
#define NF_IP_LOCAL_OUT     3
#define NF_IP_POST_ROUTING  4
```

上面代码中的常量与 图1 中挂载钩子函数的位置一一对应，如常量 `NF_IP_PRE_ROUTING` 对应着 图1 的 `PRE_ROUTING` 处。

## 二、Netfilter 钩子函数链

前面说过，Netfilter 是通过在网络协议中的不同位置挂载钩子函数来对数据包进行过滤和处理，而且每个挂载点能够挂载多个钩子函数，所以 Netfilter 使用链表结构来存储这些钩子函数，如 图2 所示：

![](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/netfilter-hook-link.png)

如 图2 所示，Netfilter 的每个挂载点都使用一个链表来存储钩子函数列表。在内核中，定义了一个名为 `nf_hooks` 的数组来存储这些链表，如下代码：

```c
// 文件：net/core/netfilter.c

struct list_head nf_hooks[32][5];
```

`struct list_head` 结构是内核的通用链表结构。

从 `nf_hooks` 变量定义为一个二维数组，第一维是用来表示不同的协议（如 IPv4 或者 IPv6，本文只讨论 IPv4，所以可以把 `nf_hooks` 当成是一维数组），而第二维用于表示不同的挂载点，如 图2 中的 5 个挂载点。

## 三、钩子函数

接下来我们介绍一下钩子函数在 Netfilter 中的存储方式。

前面我们介绍过，Netfilter 通过链表来存储钩子函数，而钩子函数是通过结构 `nf_hook_ops` 来描述的，其定义如下：

```c
// 文件：include/linux/netfilter.h

struct nf_hook_ops
{
    struct list_head list; // 连接相同挂载点的钩子函数
    nf_hookfn *hook;       // 钩子函数指针
    int pf;                // 协议类型
    int hooknum;           // 钩子函数所在链
    int priority;          // 优先级
};
```

下面我们对 `nf_hook_ops` 结构的各个字段进行说明：

*   `list`：用于把处于相同挂载点的钩子函数链接起来。
*   `hook`：钩子函数指针，就是用于处理或者过滤数据包的函数。
*   `pf`：协议类型，用于指定钩子函数挂载在 `nf_hooks` 数组第一维的位置，如 IPv4 协议设置为 `PF_INET`。
*   `hooknum`：钩子函数所在链（挂载点），如 `NF_IP_PRE_ROUTING`。
*   `priority`：钩子函数的优先级，用于管理钩子函数的调用顺序。

其中 `hook` 字段的类型为 `nf_hookfn`，`nf_hookfn` 类型的定义如下：

```c
// 文件：include/linux/netfilter.h

typedef unsigned int nf_hookfn(unsigned int hooknum,
                               struct sk_buff **skb,
                               const struct net_device *in,
                               const struct net_device *out,
                               int (*okfn)(struct sk_buff *));
```

我们也介绍一下 `nf_hookfn` 函数的各个参数的作用：

*   `hooknum`：钩子函数所在链（挂载点），如 `NF_IP_PRE_ROUTING`。
*   `skb`：数据包对象，就是要处理或者过滤的数据包。
*   `in`：接收数据包的设备对象。
*   `out`：发送数据包的设备对象。
*   `okfn`：当挂载点上所有的钩子函数都处理过数据包后，将会调用这个函数来对数据包进行下一步处理。

## 四、注册钩子函数

当定义好一个钩子函数结构后，需要调用 `nf_register_hook` 函数来将其注册到 `nf_hooks` 数组中，`nf_register_hook` 函数的实现如下：

```c
// 文件：net/core/netfilter.c

int nf_register_hook(struct nf_hook_ops *reg)
{
    struct list_head *i;

    br_write_lock_bh(BR_NETPROTO_LOCK); // 对 nf_hooks 进行上锁

    // priority 字段表示钩子函数的优先级
    // 所以通过 priority 字段来找到钩子函数的合适位置
    for (i = nf_hooks[reg->pf][reg->hooknum].next;
         i != &nf_hooks[reg->pf][reg->hooknum];
         i = i->next)
    {
        if (reg->priority < ((struct nf_hook_ops *)i)->priority)
            break;
    }

    list_add(&reg->list, i->prev); // 把钩子函数添加到链表中

    br_write_unlock_bh(BR_NETPROTO_LOCK); // 对 nf_hooks 进行解锁

    return 0;
}
```

`nf_register_hook` 函数的实现比较简单，步骤如下：

*   对 `nf_hooks` 进行上锁操作，用于保护 `nf_hooks` 变量不受并发竞争。
*   通过钩子函数的优先级来找到其在钩子函数链表中的正确位置。
*   把钩子函数插入到链表中。
*   对 `nf_hooks` 进行解锁操作。

插入过程如 图3 所示：

![](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/hook-function-instert.png)

如 图3 所示，我们要把优先级为 20 的钩子函数插入到 `PRE_ROUTING` 这个链中，而 `PRE_ROUTING` 链已经存在两个钩子函数，一个优先级为 10， 另外一个优先级为 30。

通过与链表中的钩子函数的优先级进行对比，发现新的钩子函数应该插入到优先级为 10 的钩子函数后面，所以就 如图3 所示就把新的钩子函数插入到优先级为 10 的钩子函数后面。

## 五、触发调用钩子函数

钩子函数已经被保存到不同的链上，那么什么时候才会触发调用这些钩子函数来处理数据包呢？

要触发调用某个挂载点上（链）的所有钩子函数，需要使用 `NF_HOOK` 宏来实现，其定义如下：

```c
// 文件：include/linux/netfilter.h

#define NF_HOOK(pf, hook, skb, indev, outdev, okfn)    \
    (list_empty(&nf_hooks[(pf)][(hook)])               \
        ? (okfn)(skb)                                  \
        : nf_hook_slow((pf), (hook), (skb), (indev), (outdev), (okfn)))
```

首先介绍一下 `NF_HOOK` 宏的各个参数的作用：

*   `pf`：协议类型，就是 `nf_hooks` 数组的第一个维度，如 IPv4 协议就是 `PF_INET`。
*   `hook`：要调用哪一条链（挂载点）上的钩子函数，如 `NF_IP_PRE_ROUTING`。
*   `indev`：接收数据包的设备对象。
*   `outdev`：发送数据包的设备对象。
*   `okfn`：当链上的所有钩子函数都处理完成，将会调用此函数继续对数据包进行处理。

而 `NF_HOOK` 宏的实现也比较简单，首先判断一下钩子函数链表是否为空，如果是空的话，就直接调用 `okfn` 函数来处理数据包，否则就调用 `nf_hook_slow` 函数来处理数据包。我们来看看 `nf_hook_slow` 函数的实现：

```c
// 文件：net/core/netfilter.c

int nf_hook_slow(int pf, unsigned int hook, struct sk_buff *skb,
                 struct net_device *indev, struct net_device *outdev,
                 int (*okfn)(struct sk_buff *))
{
    struct list_head *elem;
    unsigned int verdict;
    int ret = 0;

    elem = &nf_hooks[pf][hook]; // 获取要调用的钩子函数链表

    // 遍历钩子函数链表，并且调用钩子函数对数据包进行处理
    verdict = nf_iterate(&nf_hooks[pf][hook], &skb, hook, indev, outdev, &elem, okfn);
    ...
    // 如果处理结果为 NF_ACCEPT, 表示数据包通过所有钩子函数的处理, 那么就调用 okfn 函数继续处理数据包
    // 如果处理结果为 NF_DROP, 表示数据包被拒绝, 应该丢弃此数据包
    switch (verdict) {
    case NF_ACCEPT:
        ret = okfn(skb);
        break;
    case NF_DROP:
        kfree_skb(skb);
        ret = -EPERM;
        break;
    }

    return ret;
}
```

`nf_hook_slow` 函数的实现也比较简单，过程如下：

*   首先调用 `nf_iterate` 函数来遍历钩子函数链表，并调用链表上的钩子函数来处理数据包。
*   如果处理结果为 `NF_ACCEPT`，表示数据包通过所有钩子函数的处理, 那么就调用 `okfn` 函数继续处理数据包。
*   如果处理结果为 `NF_DROP`，表示数据包没有通过钩子函数的处理，应该丢弃此数据包。

既然 Netfilter 是通过调用 `NF_HOOK` 宏来调用钩子函数链表上的钩子函数，那么内核在什么地方调用这个宏呢？

比如数据包进入 IPv4 协议层的处理函数 `ip_rcv` 函数中就调用了 `NF_HOOK` 宏来处理数据包，代码如下：

```c
// 文件：net/ipv4/ip_input.c

int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt)
{
    ...
    return NF_HOOK(PF_INET, NF_IP_PRE_ROUTING, skb, dev, NULL, ip_rcv_finish);
}
```

如上代码所示，在 `ip_rcv` 函数中调用了 `NF_HOOK` 宏来处理输入的数据包，其调用的钩子函数链（挂载点）为 `NF_IP_PRE_ROUTING`。而 `okfn` 设置为 `ip_rcv_finish`，也就是说，当 `NF_IP_PRE_ROUTING` 链上的所有钩子函数都成功对数据包进行处理后，将会调用 `ip_rcv_finish` 函数来继续对数据包进行处理。

## 六、总结

本文主要介绍了 Netfilter 的实现，因为 Netfilter 是 Linux 网络数据包过滤的框架，而 iptables 就是建立在 Netfilter 之上的。所以，先了解 Netfilter 的实现对分析 iptables 的实现有非常大的帮助。

而在下一章中，我们将会继续分析 iptables 的实现。

