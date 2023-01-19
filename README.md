# learn-docker

## 利用VXLAN使不同服务器间的容器二层网络通信

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
