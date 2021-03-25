# Netfilter 与 iptables 原理

`Netfilter` 可能了解的人比较少，但是 `iptables` 用过 Linux 的都应该知道。本文主要介绍 `Netfilter` 与 `iptables` 的原理，而下一篇将会介绍 `Netfilter` 与 `iptables` 的实现。

## 什么是 Netfilter

`Netfilter` 顾名思义就是网络过滤器，其主要功能就是对进出内核协议栈的数据包进行过滤或者修改，有名的 `iptables` 就是建立在 `Netfilter` 之上。

`Netfilter` 主要通过向内核协议栈中不同的位置注册 `钩子函数` 来对数据包进行过滤或者修改操作，这些位置主要分为 5 个：`PRE_ROUTING`、`LOCAL_IN`、`FORWARD`、`LOCAL_OUT` 和 `POST_ROUTING`，如下图所示：

![netfilter-hooks](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/netfilter-hooks.png)

这 5 个位置的意义如下：

* `PRE_ROUTING`：路由前，就是数据包进入内核协议栈，但还没有进行路由查找
* `LOCAL_IN`：进入本地
* `FORWARD`：转发
* `LOCAL_OUT`：本地输出
* `POST_ROUTING`：路由后



