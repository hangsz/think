Mellanox 是一家 RDMA 网卡厂商，目前已被 NVIDIA 收购。

RDMA(Remote Direct Memory Access) 全称远程直接数据存取，就是为了解决网络传输中服务器端数据处理的延迟而产生的。RDMA 通过网络把资料直接传入计算机的存储区，将数据从一个系统快速移动到远程系统存储器中，而不对操作系统造成任何影响，这样就不需要用到多少计算机的处理功能。它消除了外部存储器复制和上下文切换的开销，因而能解放内存带宽和 CPU 周期用于改进应用系统性能。

## MLX5 网卡

NVIDIA® Mellanox® ConnectX®-5 网卡可提供先进的硬件分流，以降低 CPU 资源消耗，并推动极高的数据包速率和吞吐量。这可提高数据中心基础设施的效率，并为 Web 2.0、云、数据分析和存储平台提供灵活的高性能解决方案。

[https://www.nvidia.cn/networking/ethernet/connectx-5/](https://www.nvidia.cn/networking/ethernet/connectx-5/)

## 网卡驱动

[https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/)

判断驱动是否安装成功：

```
# ibdev2netdev
mlx5_0 port 1 ==> ens2f0 (Up)   #表示该网口插了网线
mlx5_1 port 1 ==> ens2f1 (Down) #表示该网口没有插网线
```

## idv_devinfo 命令

[https://man7.org/linux/man-pages/man1/ibv_devinfo.1.html](https://man7.org/linux/man-pages/man1/ibv_devinfo.1.html)

## ibdev2netdev 脚本

**ibdev2netdev** is one of the most useful scripts in MLNX_OFED package as it maps the adapter port to the net device.

[https://enterprise-support.nvidia.com/s/article/ibdev2netdev](https://enterprise-support.nvidia.com/s/article/ibdev2netdev)

## 带宽统计

在使用 RDMA 时，发送和接收的数据带宽可以在 app 中自己进行收集，这样我们的程序发送和接收的数据量会很清楚。  
如果想知道当前 RDMA 网卡所发送和接收的带宽可以通过 sysfs 下的相关节点获取。

- 发送数据量（byte）：/sys/class/infiniband/mlx5_0/ports/1/counters/port_xmit_data
- 接收数据量（byte）：/sys/class/infiniband/mlx5_0/ports/1/counters/port_rcv_data

## 连通性探测

Server 端

```
# ibdev2netdev
mlx4_0 port 1 ==> enp1s0 (Down)
mlx5_0 port 1 ==> ens2f0 (Up)
mlx5_1 port 1 ==> ens2f1 (Up)
```

Client 端

```
# ibdev2netdev
mlx5_0 port 1 ==> p5p1 (Down)
mlx5_1 port 1 ==> p5p2 (Up)
mlx5_2 port 1 ==> p4p1 (Down)
mlx5_3 port 1 ==> p4p2 (Down)
```

连通性测试

```
# ibv_rc_pingpong -d mlx5_1 -g 0 192.168.2.4
  local address:  LID 0x0000, QPN 0x0009df, PSN 0xa7f02f, GID fe80::1e34:daff:fe79:c0d
  remote address: LID 0x0000, QPN 0x00011a, PSN 0xd775ee, GID fe80::e42:a1ff:fe41:2d36
8192000 bytes in 0.01 seconds = 5376.21 Mbit/sec
1000 iters in 0.01 seconds = 12.19 usec/iter
```