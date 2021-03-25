# IP数据包接收过程

在《网络中断下半部处理》一文中介绍过，当网卡接收到网络数据包后，会由网卡驱动通过调用 `netif_rx` 函数把数据包添加到待处理队列中，然后唤起网络中断下半部处理。

而网络中断下半部处理由 `net_rx_action` 函数完成的，其主要功能就是从待处理队列中获取一个数据包，然后根据数据包的网络层协议类型来找到相应的处理接口来处理数据包。我们通过 图1 来展示 `net_rx_action` 函数的处理过程：

![net_rx_action](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/net_rx_action.png)

图1 `net_rx_action` 处理过程
