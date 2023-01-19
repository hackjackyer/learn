# learn-docker

## 利用VXLAN使不同服务器间的容器二层网络通信

参考文档
* https://docs.docker.com/network/bridge/
* https://blog.thestateofme.com/2014/06/08/connecting-docker-containers-between-vms-with-vxlan/
* https://www.thegeekdiary.com/arp-command-not-found/amp/
* https://devcoops.com/assign-static-ip-address-docker-container/#:~:text=1%20Step%201.%20Create%20a%20network%20first.%20docker,3%20Step%203.%20Verify%20the%20IP%20address%20assignment.
* https://www.tecmint.com/create-network-bridge-in-rhel-centos-8/
* https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/assembly_using-a-vxlan-to-create-a-virtual-layer-2-domain-for-vms_configuring-and-managing-networking
* https://docs.docker.com/engine/tutorials/networkingcontainers/#:~:text=Docker%20includes%20support%20for%20networking%20containers%20through%20the,own%20drivers%20but%20that%20is%20an%20advanced%20task.
* http://www.billauer.co.il/ipmasq-html.html#:~:text=%20iptables%20-t%20nat%20-A%20POSTROUTING%20-j%20MASQUERADE,will%20appear%20to%20come%20from%20the%20masquerading%20host.

### 实验环境

* vm:centos7
* vm1:192.168.100.1/24
* vm2:192.168.100.2/24
* container:centos:7
* container1:172.17.0.11/16
* container2:172.17.0.12/16

```bash
# 安装容器
yum install -y docker
systemctl start docker
# 拉取测试镜像，操作系统不定，这里以centos为例
docker pull centos:7
# 这里创建以下docker的配置文件仅此而已
nmcli conn add con-name docker0 ifname docker0 type bridge
nmcli conn up docker0
# 创建vxlan并绑定到docker0，以下命令在两台主机上分别输入哦。
connection add type vxlan slave-type bridge con-name docker0-vxlan10 ifname vxlan10 id 10 local 192.168.100.1 remote 192.168.100.2 master docker0
connection add type vxlan slave-type bridge con-name docker0-vxlan10 ifname vxlan10 id 10 local 192.168.100.2 remote 192.168.100.1 master docker0
# 放行vxlan端口
firewall-cmd --permanent --add-port=8472/udp
firewall-cmd --reload
# 启动子桥并查看状态
nmcli conn up docker0-vxlan10
nmcli device
# 容器指定IP启动，防止冲突
docker run --net docker0 --ip 172.17.0.11 -dit docker.io/centos:7
docker run --net docker0 --ip 172.17.0.12 -dit docker.io/centos:7
docker ps
# 进入容器安装相应指令，测试网络是否可以二层通信
docker exec -it containerid bash
yum install -y iproute net-tools
ping 172.17.0.11
ping 172.17.0.12
```
