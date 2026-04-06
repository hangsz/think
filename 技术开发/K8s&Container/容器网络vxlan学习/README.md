# 基础知识

## veth设备

- 成对出现，虚拟以太网，一段发送，另一端接收
- ip link add veth 创建
- ip set 将其放入某 namespace中

  

## Route表

- 三层交换设备使用，要转发到目的ip地址，应该走哪个dev发送或接收
- 有 dev 就认为直连，可以直接发arp等。没有dev则递归查询路由表直到找到带有dev的条目

default via 172.21.240.1 dev eth0

dev对应出设备

172.21.240.0/20 dev eth0 proto kernel scope link src 172.21.250.97

## Neigh表或ARP表

- 三层交换设备使用，已经到达ip所在的子网，如何找到对应的dev mac，用于封装二层头

目的ip 走这个dev 目的ip对应的二层mac

172.21.240.1 dev eth0 lladdr 00:15:5d:43:38:23 STALE

## FDB表

- 待补充

# 启动网络

  

## 网络设备

![](技术开发/K8s&Container/容器网络vxlan学习/1.png)

## 启动 vxlan设备

- vxlan用于封装和解封
- vxlan ip一般为pod cidr的第一个地址

  

```
# node
# id 区分不同vlan
# dstport，vlan规范的port 4789，默认8742
# local，三层ip
# dev，和local功能相同，二选一，用于从哪个网卡获取三层ip
ip link add <vxlan name> type vxlan id <id> dstport 4789 local <node ip> dev <node dev>


# 配置ip， vxlan ip一般为podcidr的第一个地址
ip addr add <vxlan ip/mask> dev <vxlan name>
# 启动
ip link set <vxlan name> up

# 会自动添加route表，当新node添加时也需要添加（flannel等cni自动添加）
# 每个node上的vxlan会在其他node上生成route
<vxlan ip/mask> dev <vxlan name>  proto kernel  scope link <vxlan ip>
<dst vxlan ip/mask> dev <vxlan name>  proto kernel  scope link <vxlan ip>

# 会自动更新vxlan fdb条目, mac反查dst ip
<dst vxlan mac> dev <vxlan name> dst <dst node ip> via <node dev> self permanent


# 添加neigh表，添加一条就行
<dst vlan ip/mask> dev <vxlan name> lladdr <dst vxlan mac> STALE
```

## 启动 bridge设备

- 虚拟的二层网络交换机
- bridge用于node上pod互通
- vxlan下，也作为pod默认网关

  

```
# node
ip link add <bridge name> type bridge 


# 配置ip， bridge ip一般为podcidr的第二个地址
ip addr add <bridge ip/mask> dev <bridge name>
# 启动
ip link set <bridge name> up
```

## 配置 veth设备

- 每个pod有独立的netns，container共享同一个netns
- pod的netns中加入veth的一端作为eth0网卡
- 给eth0 添加ip和子网掩码，这个ip由cni分配，一般根据node的pod cidr分配
- 给eth0 添加默认路由，网关为bridge ip

```
# node 执行
# 添加netns
ip netns add <ns name>
# 添加veth对
ip link add <veth name> type veth peer name <veth peer name>
# 一端放入ns中，另一端默认在node，这样node和ns就连接起来了
ip link set <veth peer name> netns <ns name>  
# 启动node网卡veth-ns
ip link set <veth name> up 

# 配置pod ip
ip netns exec <ns name> ip addr add <pod ip/mask> dev <veth peer name> 
# 启动ns网卡
ip netns exec <ns name> ip link set <veth peer name> up 

# 配置默认路由
ip netns exec <ns name> ip route add default via <bridge ip> dev <veth peer name>
```

  

# 跨主机POD发包流程

pod1 在node1上，pod2在node2上。

1. pod 三层
- src为pod1 ip
- dst为pod2 ip
- **查路由表**，确定下一跳ip和二层dev。匹配 default对应的路由条目，下一跳via <bridge ip>，dev <veth peer name>。有dev，认为直连，进入arp流程
- **查neigh表**，确定下一跳ip对应的mac。查不到， **arp广播请求** <bridge ip>对应的mac。<veth peer name> -> <veth name> -> bridge, <bridge name>响应自身mac

1. pod 二层
- dev为<veth peer name>
- src为<veth peer name>dev的mac
- dst为<bridge name>dev的mac

3. pod 二层到 node二层，<veth peer name> -> <veth name> -> bridge，进入node的netns
4. node 二层

- bridge发现mac是自己，接收此包，否则丢掉

5. node 三层

- 发现三层ip不是node的ip，需要转发
- **查路由表**，确定下一跳ip和二层dev。匹配 <dst vxlan ip>对应的路由条目（上面介绍过，vxlan cni自动添加，vxlan和pod一个网段，所以能匹配上），dev为<vxlan name>。有dev，认为直连，进入arp流程
- **查neigh表**，确定下一跳ip对应的mac。匹配 <dst vxlan ip>对应的neigh条目，为<dst vxlan name>的mac（上面介绍过，vxlan cni自动添加）

6. node 二层

- dev为<vxlan name>
- src为<vxlan name> dev的mac
- dst为<dst vxlan name>dev的mac

7. 由于dev为vxlan设备，继续走vxlan设备的额外逻辑
8. underlay 四层

- 添加udp层，默认端口

9. underlay三层

- src为<node ip>，vxlan配置的对应参数 (local 或dev参数，一般对应node ip和物理网卡的ip）
- 查询vxlan fdb表，确定dst ip。匹配<dst vxlan mac>对应的条目， 查询到 <dst node ip>
- dst为<dst node ip>
- **查路由表**，确定下一跳ip和二层dev。匹配 <dst node ip>的node underlay路由条目，进入arp流程
- **查neigh表**，确定下一跳ip对应的mac

10. underlay二层

- src为node dev，dev为vxlan配置的对应参数
- dst为上一步查询到的mac