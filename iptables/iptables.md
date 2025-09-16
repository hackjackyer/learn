你可以通过 Socat 和 Iptables 两种不同的方式实现端口转发。Socat 是在应用层进行转发，而 Iptables 是在网络层进行转发，两者各有优劣。
使用 Socat 实现端口转发
Socat 能够直接在用户空间（user space）进行端口转发。它的优点是配置简单直观，可以跨越多种协议，并且能灵活地进行数据流的转换或记录。
基本语法：
socat TCP-LISTEN:<local_port>,fork TCP:<remote_host>:<remote_port>

 * TCP-LISTEN:<local_port>: 这指定了 Socat 将监听本地的 TCP 端口。
 * fork: 这个选项非常重要，它让 Socat 在接收到一个连接后，为每个新连接都创建一个子进程。这使得 Socat 能够同时处理多个客户端连接。
 * TCP:<remote_host>:<remote_port>: 这是 Socat 将流量转发到的目标地址和端口。
例子：
将本地的 8080 端口转发到 example.com 的 80 端口。
socat TCP-LISTEN:8080,fork TCP:example.com:80

这行命令会启动一个 Socat 进程，它会监听本地的 8080 端口。当有客户端连接到这个端口时，Socat 会建立一个到 example.com:80 的连接，并将两者之间的数据双向转发。
使用 Iptables 实现端口转发
Iptables 是 Linux 内核中的防火墙工具，它在网络层（或者更准确地说是内核的 netfilter 框架）进行数据包的转发。它更高效、更稳定，更适合作为生产环境的端口转发解决方案。
基本语法：
你需要配置两条 Iptables 规则：一条用于 DNAT（目标网络地址转换），另一条用于 SNAT（源网络地址转换）。
 * DNAT 规则： 这条规则将进入的数据包的目标地址和端口修改为真正的目标服务器。
   iptables -t nat -A PREROUTING -p tcp --dport <local_port> -j DNAT --to-destination <remote_host>:<remote_port>

   * -t nat: 指定使用 nat 表。
   * -A PREROUTING: 在数据包进入系统后，但在路由决策前处理。
   * -p tcp: 指定协议为 TCP。
   * --dport <local_port>: 匹配目标端口。
   * -j DNAT --to-destination <remote_host>:<remote_port>: 进行目标地址转换。
 * SNAT 规则： 当转发的流量从服务器返回到客户端时，需要进行源地址转换，以确保数据包能够正确返回到原始客户端。如果没有这条规则，目标服务器可能会直接将响应发送给客户端，导致连接中断。
   iptables -t nat -A POSTROUTING -p tcp -d <remote_host> --dport <remote_port> -j SNAT --to-source <local_ip>

   * -A POSTROUTING: 在数据包即将离开系统时处理。
   * -d <remote_host>: 匹配目标地址。
   * --to-source <local_ip>: 将数据包的源地址修改为本机地址，通常是公网 IP。
例子：
将本地 8080 端口的流量转发到 192.168.1.100 的 80 端口。
# 1. 开启内核转发功能
sysctl net.ipv4.ip_forward=1
# 2. DNAT 规则
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.100:80
# 3. SNAT 规则
iptables -t nat -A POSTROUTING -d 192.168.1.100 -p tcp --dport 80 -j MASQUERADE

这里的 MASQUERADE 是一种特殊的 SNAT 形式，它会自动使用出站接口的 IP 地址作为源地址，非常方便。
现在你已经了解了 Socat 和 Iptables 实现端口转发的方法，你更倾向于哪种方式呢？或者说，你希望了解它们在性能、安全性或配置复杂度方面的具体差异吗？
