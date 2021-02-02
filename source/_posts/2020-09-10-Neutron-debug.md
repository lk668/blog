---
title: Neutron网络troubleshooting
date: 2020-09-10
tags:
    - openstack
    - neutron
---

本文简单介绍一下如何对openstack neutron网络做troubleshooting

<!-- more -->

# 1. 如何查看网络设备

```bash 
# 可以查看网卡设备，以及网卡类型(openvswitch/tap/tun/veth)
ip -d link show

#如何查看veth的对端地址
ethtool -S qvb3ce0ad77-c9
[root@tstack-con01 ~]# ethtool -S qvb3ce0ad77-c9
NIC statistics:
     peer_ifindex: 23
```

# 2. 如何监听ovs上patch类型接口的数据包
在linux下tap和veth设备都是正常的linux网络设备，可以使用ip/tcpdump来查看和监控状态。但是在ovs下，对于patch类型的接口，像patch-tun，只能对ovs是可见的，我们就不能够使用`tcpdump -i patch-tun`命令，会报错patch-tun设备不存在。

那么对于patch-tun这种patch类型的ovs接口，如何进行数据追踪呢？步骤如下。

1. 创建一个`dummpy`类型的端口snooper0
```bash
ip link add name snooper0 type dummy
ip link set dev snooper0 up
```
2. 将该端口绑定到`br-int`网桥，因为`patch-tun`位于该网桥    
```bash
    ovs-vsctl add-port br-int snooper0
```
3. 创建端口流量镜像来绑定snooper0和patch-tun  
```bash
#将进出patch-tun的流量全部镜像到snooper0网卡
ovs-vsctl -- set Bridge br-int mirrors=@m  -- --id=@snooper0 \
  get Port snooper0  -- --id=@patch-tun get Port patch-tun \
  -- --id=@m create Mirror name=mymirror select-dst-port=@patch-tun \
  select-src-port=@patch-tun output-port=@snooper0 select_all=1
```
4. 现在就可以通过监控snooper0网卡来获取patch-tun的流量  
```bash
 tcpdump -i snooper0
```
5. 清除配置信息
```bash
ovs-vsctl clear Bridge br-int mirrors
ovs-vsctl del-port br-int snooper0
ip link delete dev snooper0
```


# 3. 数据包传输问题定位
我们可以使用`ping`命令来快速定位问题

1. 从虚拟机ping外部主机，看是否能ping通，如果能，就没有什么问题
2. 如果不能ping通外部主机，从虚拟机ping虚拟机所在的主机，如果可以ping通，说明是计算节点和计算节点的gateway之间有什么问题
3. 如果不能ping通虚拟机所在的主机，那问题就出现在虚拟机和计算主机之间。This includes the bridge connecting the compute node’s main NIC with the vnet NIC of the instance.
4. 最后，在该计算几点再创建一台虚拟机，两台虚拟机相互ping，如果可以ping通，那基本定位为是主机防火墙的问题。

# 4. tcpdump
另外一个强大的工具是`tcpdump`。我们可以使用下面的命令来监控`ICMP`数据包

```bash
tcpdump -i any -n -v 'icmp[icmptype] = icmp-echoreply or icmp[icmptype] = icmp-echo'
```

然后在虚拟机、计算节点主机、外部主机分别运行该命令。然后利用ping来检查哪条链路出现了问题

# 5. iptables
在OpenStack中，我们的安全组是通过iptables来实现的。我们可以使用下面的命令来获取当前主机的iptables配置
```bash
iptables-save
```

# 6. 如何手动解绑定浮动IP
如果在执行解绑定浮动IP的时候，失败了，数据库无法释放该浮动ip，那么可以通过下面的命令，修改数据库来释放浮动IP

```bash
mysql> select uuid from instances where hostname = 'hostname';
mysql> select * from fixed_ips where instance_uuid = '<uuid>';
mysql> select * from floating_ips where fixed_ip_id = '<fixed_ip_id>';
mysql> update floating_ips set fixed_ip_id = NULL, host = NULL where
       fixed_ip_id = '<fixed_ip_id>';
```

# 7. DHCP troubleshooting
当我们在openstack中创建网络并开启dhcp功能的时候，neutron会在网络节点创建一个名字为`qdhcp-<network-id>`的linux网络命名空间。在该网络明明空间配置一个和subnet同网段的ip，此ip来提供dhcp和dns功能。我们会在该命名空间下启动一个dnsmasq进程，该进程用来提供dhcp和dns服务。

所以在dhcp出现问题的时候，使用如下步骤进行debug
1. 查看dhcp的请求是否到达了`qdhcp-<network-id>`命名空间的`tapXXXXX`网卡，如果没有到达，网络有问题。可以使用tcpdump来进行数据包的追踪。对于dhcp请求，使用udp协议，client端使用68端口，server端使用67端口  
```
tcpdump -i br100 -n port 67 or port 68

以下是在openstack网络节点qdhcp-768a8089-ac83-4961-8c08-dc14eaf5f78a网络命名空间的执行结果，可以查看到相应的服务
[root@tstack-con01 log]# ip netns exec qdhcp-768a8089-ac83-4961-8c08-dc14eaf5f78a lsof -i:67
COMMAND     PID   USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
dnsmasq 1592657 nobody    4u  IPv4 27757858      0t0  UDP *:bootps
```
2. 如果请求可以正常到达该命名空间，那查看dnsmasq服务时候正常运行。如果没有运行，重启neutron-dhcp-agent
3. 如果dnsmasq正常运行，查看dnsmasq的log，看是否ip已经被分配完了，或者什么其他错误。可以通过`/var/log/syslog`或者`/varlog/messages`来查看dnsmasq的日志

# 8. DNS troubleshooting 
对于dns服务，openstack也是利用`dnsmasq`来实现。可以使用下面的tcpdump命令来进行数据包的追踪

```bash
tcpdump -i br100 -n -v udp port 53
```

以下是在openstack网络节点qdhcp-768a8089-ac83-4961-8c08-dc14eaf5f78a网络命名空间的执行结果，可以查看到相应的dnsmasq服务运行在53端口上
```bash
[root@tstack-con01 log]# ip netns exec qdhcp-768a8089-ac83-4961-8c08-dc14eaf5f78a lsof -i:53
COMMAND     PID   USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
dnsmasq 1592657 nobody    6u  IPv4 27757861      0t0  UDP host-10-1-0-2.openstacklocal:domain
dnsmasq 1592657 nobody    7u  IPv4 27757862      0t0  TCP host-10-1-0-2.openstacklocal:domain (LISTEN)
dnsmasq 1592657 nobody    8u  IPv4 27757863      0t0  UDP localhost:domain
dnsmasq 1592657 nobody    9u  IPv4 27757864      0t0  TCP localhost:domain (LISTEN)
dnsmasq 1592657 nobody   10u  IPv6 27757865      0t0  UDP tstack-con01:domain
dnsmasq 1592657 nobody   11u  IPv6 27757866      0t0  TCP tstack-con01:domain (LISTEN)
dnsmasq 1592657 nobody   12u  IPv6 27757867      0t0  UDP localhost:domain
dnsmasq 1592657 nobody   13u  IPv6 27757868      0t0  TCP localhost:domain (LISTEN)
[root@tstack-con01 log]#
```

# 9. 参考工具
[easyOVS](https://github.com/yeasy/easyOVS) 是一个在openstack平台操控linux bridge， ovs，iptables的工具。他可以自动关联虚拟端口、vm的IP/MAC、VLAN tag、网络命名空间以及vm的iptables规则。用起来非常方便。

`参考文档`
- https://docs.openstack.org/operations-guide/ops-network-troubleshooting.html