好的，根据您的需求，这是一个详细的分步配置指南，可以帮助您在主机 A 和主机 B 上使用 OVS 和 VXLAN 实现跨地域的二层通信。
首先，我们假设两台主机的物理 IP 地址分别为：
 * 主机 A： 10.10.10.1
 * 主机 B： 10.10.10.2
   我们还将使用一个 VNI（VXLAN Network Identifier）来匹配您的 VLAN 200，我们将 VNI 设为 200。
主机 A 上的配置步骤
 * 创建 OVS 桥:
   如果您还没有 OVS 桥，请首先创建一个。
   sudo ovs-vsctl add-br br0

 * 添加物理网口 eth0 到桥:
   将您的物理网口 eth0 作为 br0 的一个端口。
   sudo ovs-vsctl add-port br0 eth0

 * 配置管理网络端口（VLAN 100）:
   为了让 OVS 处理管理流量，您需要创建一个带 VLAN 100 标签的内部端口。
   sudo ovs-vsctl add-port br0 mgmt tag=100 -- set interface mgmt type=internal
sudo ifconfig mgmt up

   您可以在 mgmt 端口上配置主机的管理 IP 地址。
 * 创建 VXLAN 隧道端口 (vx1)：
   这是最关键的步骤，用于创建 VXLAN 隧道。这个端口将作为 VLAN 200 网络流量的入口，并将其封装发送到主机 B。
   sudo ovs-vsctl add-port br0 vx1 -- set interface vx1 type=vxlan options:remote_ip=10.10.10.2 options:key=200

   * type=vxlan: 指定端口类型为 VXLAN。
   * options:remote_ip=10.10.10.2: 告诉 OVS 隧道的另一端是主机 B 的 IP 地址。
   * options:key=200: 将 VNI 设置为 200，这等同于在 VXLAN 网络中标记 VLAN 200 的流量。
主机 B 上的配置步骤
主机 B 的配置与主机 A 类似，只是 remote_ip 的值需要指向主机 A 的 IP 地址。
 * 创建 OVS 桥:
   sudo ovs-vsctl add-br br0

 * 添加物理网口 eth0 到桥:
   sudo ovs-vsctl add-port br0 eth0

 * 配置管理网络端口（VLAN 100）:
   sudo ovs-vsctl add-port br0 mgmt tag=100 -- set interface mgmt type=internal
sudo ifconfig mgmt up

 * 创建 VXLAN 隧道端口 (vx1)：
   将 VXLAN 隧道的另一端指向主机 A。
   sudo ovs-vsctl add-port br0 vx1 -- set interface vx1 type=vxlan options:remote_ip=10.10.10.1 options:key=200

完成这些步骤后，任何进入主机 A vx1 端口的二层流量，都会被封装并通过 VXLAN 隧道传输到主机 B 的 vx1 端口，反之亦然。这使得它们能够实现跨地域的二层通信，而管理流量则通过 mgmt 端口和 VLAN 100 独立处理。
您希望了解如何验证 VXLAN 隧道是否正常工作，或者如何排除配置过程中的常见问题吗？

验证 VXLAN 隧道是否正常工作，需要从几个不同的层面进行检查，包括物理连通性、OVS 配置和实际的数据包流动。以下是几个关键的验证步骤：
1. 验证 OVS 桥和端口配置
首先，在两台主机上运行 ovs-vsctl show 命令，确认 OVS 桥 (br0)、物理端口 (eth0)、管理端口 (mgmt) 以及最重要的 VXLAN 隧道端口 (vx1) 都已正确添加和配置。
sudo ovs-vsctl show

检查输出中 vx1 端口的 type 是否为 vxlan，以及 options:remote_ip 和 options:key 是否正确指向对端主机的 IP 和预设的 VNI。
2. 验证物理网络连通性
由于 VXLAN 隧道依赖于底层的 IP 网络，因此确保两台主机的物理 IP (10.10.10.1 和 10.10.10.2) 之间能够正常通信是至关重要的。在两台主机上相互 ping 对方的物理 IP 地址。
# 在主机 A 上
ping 10.10.10.2

# 在主机 B 上
ping 10.10.10.1

如果 ping 不通，你需要首先解决底层 IP 网络的问题，例如防火墙规则、路由或网络故障。
3. 验证数据包流动 (VXLAN 隧道)
这是最关键的一步，用于验证二层报文是否确实通过 VXLAN 隧道传输。
 * 方法一：使用 tcpdump 抓包
   在主机 A 和主机 B 上，同时对物理网卡 eth0 进行抓包，并过滤 VXLAN 协议（默认端口 4789）。
   # 在主机 A 和主机 B 上同时运行
sudo tcpdump -i eth0 -n -e port 4789

   然后，让位于两台主机上的虚拟机（或连接到 VLAN 200 网络的设备）之间进行二层通信，例如相互 ping 对方的 IP 地址。
   * 预期结果： 你应该能在两台主机的 tcpdump 输出中看到封装了 UDP 端口 4789 的 VXLAN 报文。这些报文的源 IP 和目的 IP 应该是 10.10.10.1 和 10.10.10.2。如果能看到这些数据包，说明 VXLAN 隧道工作正常。
 * 方法二：使用 ovs-ofctl dump-flows
   在通信过程中，VXLAN 隧道的流量也会在 OVS 内部形成流规则。你可以通过查看 OVS 的流表来验证流量是否经过隧道。
   sudo ovs-ofctl dump-flows br0

   你应该能看到类似 vxlan 端口的规则，这些规则将特定匹配条件下的报文导向 vx1 端口。
如果上述验证步骤都通过了，说明 VXLAN 隧道配置成功，并且二层报文能够正常传输。
现在我们已经详细讨论了验证 VXLAN 隧道的方法。您对如何进一步在 OVS 中管理和监控这些隧道流量，比如查看隧道流量统计或调试更复杂的多点 VXLAN 网络感兴趣吗？

