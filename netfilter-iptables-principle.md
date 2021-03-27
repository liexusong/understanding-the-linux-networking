# Netfilter 与 iptables 原理

`Netfilter` 可能了解的人比较少，但是 `iptables` 用过 Linux 的都应该知道。本文主要介绍 `Netfilter` 与 `iptables` 的原理，而下一篇将会介绍 `Netfilter` 与 `iptables` 的实现。

## 什么是 Netfilter

`Netfilter` 顾名思义就是网络过滤器，其主要功能就是对进出内核协议栈的数据包进行过滤或者修改，有名的 `iptables` 就是建立在 `Netfilter` 之上。

`Netfilter` 通过向内核协议栈中不同的位置注册 `钩子函数（Hooks）` 来对数据包进行过滤或者修改操作，这些位置称为 `挂载点`，主要有 5 个：`PRE_ROUTING`、`LOCAL_IN`、`FORWARD`、`LOCAL_OUT` 和 `POST_ROUTING`，如下图所示：

![netfilter-hooks](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/netfilter-hooks.png)

这 5 个 `挂载点` 的意义如下：

* `PRE_ROUTING`：路由前。数据包进入IP层后，但还没有对数据包进行路由判定前。

* `LOCAL_IN`：进入本地。对数据包进行路由判定后，如果数据包是发送给本地的，在上送数据包给上层协议前。

* `FORWARD`：转发。对数据包进行路由判定后，如果数据包不是发送给本地的，在转发数据包出去前。

* `LOCAL_OUT`：本地输出。对于输出的数据包，在没有对数据包进行路由判定前。

* `POST_ROUTING`：路由后。对于输出的数据包，在对数据包进行路由判定后。

从上图可以看出，路由判定是数据流向的关键点。

* 第一个路由判定通过查找输入数据包 `IP头部` 的目的 `IP地址` 是否为本机的 `IP地址`，如果是本机的 `IP地址`，说明数据是发送给本机的。否则说明数据包是发送给其他主机，经过本机只是进行中转。

* 第二个路由判定根据输出数据包 `IP头部` 的目的 `IP地址` 从路由表中查找对应的路由信息，然后根据路由信息获取下一跳主机（或网关）的 `IP地址`，然后进行数据传输。

通过向这些 `挂载点` 注册钩子函数，就能够对处于不同阶段的数据包进行过滤或者修改操作。由于钩子函数能够注册多个，所以内核使用链表来保存这些钩子函数，如下图所示：

![netfilter-hooks](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/netfilter-hooks-functions.png)

如上图所示，当数据包进入本地（`LOCAL_IN` 挂载点）时，就会相继调用 `ipt_hook` 和 `fw_confirm` 钩子函数来处理数据包。另外，钩子函数还有优先级，优先级越小越先执行。正因为挂载点是通过链表来存储钩子函数，所以又被称为 `链`，链的名称如下：

`LOCAL_IN` 挂载点：又称为 `INPUT链`。
`LOCAL_OUT` 挂载点：又称为 `OUTPUT链`。
`FORWARD` 挂载点：又称为 `PORWARD链`。
`PRE_ROUTING` 挂载点：又称为 `PREROUTING链`。
`POST_ROUTING` 挂载点：又称为 `POSTOUTING链`。


## 什么是 iptables

`iptables` 是建立在 `Netfilter` 之上的数据包过滤器，也就是说，`iptables` 通过向 `Netfilter` 的挂载点上注册钩子函数来实现对数据包过滤的。`iptables` 的实现比较复杂，所以先要慢慢介绍一下它的一些基本概念。

### 表

从 `iptables` 这个名字可以看出，它一定包含了 `表` 这个概念。`表` 是指一系列规则，可以看成是规则表。`iptables` 通过把这些规则表挂载在 `Netfilter` 的挂载点上，对进出内核协议栈的数据包进行过滤或者修改操作。

`iptables` 定义了 4 种表，每种表都有其不同的用途：

**1. Filter表**

`Filter表` 用于过滤数据包。是 `iptables` 的默认表，因此如果你配置规则时没有指定表，那么就默认使用 `Filter表`，它分别挂载在以下 3 个挂载点上：

* `LOCAL_IN`
* `LOCAL_OUT`
* `FORWARD`

**2. NAT表**

`NAT表` 用于对数据包的网络地址转换(IP、端口)，它分别挂载在以下 3 个挂载点上：

* `PRE_ROUTING`
* `POST_ROUTING`
* `LOCAL_OUT`

**3. Mangle表**

`Mangle表` 用于修改数据包的服务类型或TTL，并且可以配置路由实现QOS，它分别挂载在以下 5 个挂载点上：

* `PRE_ROUTING`
* `LOCAL_IN`
* `FORWARD`
* `LOCAL_OUT`
* `POST_ROUTING`

**4. Raw表**

`Raw表` 用于判定数据包是否被状态跟踪处理，它分别挂载在以下 2 个挂载点上：

* `PRE_ROUTING`
* `LOCAL_OUT`

我们通过下图来展示各个表所挂载的挂载点：

![iptables-hooks](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/iptables-hooks.png)

上图展示了，数据包从网络中进入到内核协议栈的过程中，要执行的 `iptables` 规则，如果在执行某条 `iptables` 规则失败后，会直接把数据包丢弃，不会继续执行下面的规则。

拿一个挂载点来看，如下图所示：

![packet-iptables](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/packet-iptables.png)

也就是说，当数据包从网络中进入到内核协议栈后，在路由判定前会分别执行 `Raw表`、`Mangle表` 和 `NAT表` 中的规则。如果在执行规则时，某一条规则拒绝了数据包，那么数据包便会被丢弃，从而不会继续执行下面的规则。

## 添加 iptables 规则

上面介绍了 `iptables` 的原理，下面主要介绍怎么向 `iptables` 中添加规则。要向 `iptables` 中添加规则，可以使用 `iptables` 命令，其使用格式如下：

```shell
iptables [-t 表名] 选项 [链名] [条件] [-j 控制类型]
```

可选的参数如下：

```
-F 清空规则链。
-L 查看规则链。
-A 在挂载点的末尾加入新规则。
-I num 在挂载点的头部加入新规则。
-D num 删除某一条规则。
-s 匹配源IP地址，加叹号 "!" 表示除这个IP外。
-d 匹配目标IP地址。
-i 网卡名称 匹配从这块网卡流入的数据。
-o 网卡名称 匹配从这块网卡流出的数据。
-p 匹配协议，如tcp、udp、icmp。
--dport num 匹配目标端口号。
--sport num 匹配来源端口号。
```
