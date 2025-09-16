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
