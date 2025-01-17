# 3.5.2 虚拟网络设备 tun 和 tap

tun 和 tap 是Linux 内核 2.4.x 版本之后引入的虚拟网卡设备，是一种让用户空间可以和内核空间双向传输数据包的虚拟网络设备。这两种设备的区别与含义为：
- tun 设备是一个三层网络层设备，从 /dev/net/tun 字符设备上（稍后介绍）读取的是 IP 数据包，写入的也只能是 IP 数据包，因此常用于一些点对点 IP 隧道，例如 OpenVPN，IPSec 等；
- tap 设备是二层链路层设备，等同于一个以太网设备，从 /dev/tap0 字符设备上读取 MAC 层数据帧，写入的也只能是 MAC 层数据帧，因此常用来作为虚拟机模拟网卡使用；

在 Linux 中，内核空间和用户空间之间数据传输有多种方式，字符设备是其中一种。所以 TAP/TUN 都具有相应的字符设备，用于实现内核空间和用户空间之间传输数据。TAP/TUN 对应的字符设备文件分别为：
- TAP：/dev/tap0；
- TUN：/dev/net/tun。

当用户空间的程序 open() 一个字符设备文件时，会返回一个 fd 句柄，同时字符设备驱动就会创建并注册相应的虚拟网卡网络接口，并以 tunX 或 tapX 命名。当用户空间的程序向 fd 执行 read()/write() 时，就可以和内核网络协议栈读写数据了。

TUN 和 TAP 的工作方式基本不同，只是两者工作的层面不一样。以使用 TUN 设备建立的 VPN 隧道为例（如图 3-20）：普通的用户程序发起一个网络请求，数据包进入内核协议栈时查找路由，下一跳是 tunX 设备。tunX 发现自己的另一端由 VPN 程序打开，所以收到数据包后传输给 VPN 程序。VPN 程序对数据包进行封装操作，“封装”是指将一个数据包包装在另一个数据包中，就像将一个盒子放在另一个盒子中一样。封装后的数据再次被发送到内核，最后通过 eth0 接口（也就是图中的物理网卡）发出。

:::center
  ![](../assets/tun.svg)<br/>
 图 3-20 VPN 中数据流动示意图
:::

将一个数据包封装到另一个数据包的处理方式被称为 “隧道”，隧道技术是构建虚拟逻辑网络的经典做法。容器网络插件 Flannel 早期曾使用 tun 设备实现了 UDP 模式下的跨主网络相互访问，但使用 tun 设备传输数据需要经过两次协议栈，且有多次的封包/解包过程，产生额外的性能损耗。这也是后来 Flannel 弃用 UDP 模式的原因。
