---
title: ovn为外部主机提供dhcp服务
date: 2020-12-21
tags:
    - ovn
---

本文简单介绍一下OVN是如何通过external logical port来为外部主机提供DHCP服务。

<!-- more -->
## 1. 简介

**Native OVN services for external logical ports**

以下是摘自OVN官网，对external logical ports的介绍

To support OVN native services (like DHCP/IPv6 RA/DNS lookup) to the cloud resources which are external, OVN supports external logical ports.

Below are some of the use cases where external ports can be used.

* VMs connected to SR-IOV nics - Traffic from these VMs by passes the kernel stack and local ovn-controller do not bind these ports and cannot serve the native services.
* When CMS supports provisioning baremetal servers. 

OVN will provide the native services if CMS has done the below  configuration in the OVN Northbound Database.

* A row is created in **Logical_Switch_Port**, configuring the **addresses** column and setting the type to **external**.

* **ha_chassis_group** column is configured.

* The HA chassis which belongs to the HA chassis group has the **ovn-bridge-mappings** configured and has proper L2 connectivity so that it can receive the DHCP and other related request packets from these external resources.

* The Logical_Switch of this port has a **localnet port**.

* Native OVN services are enabled by configuring the DHCP options like the way it is done for the normal logical ports.

It is recommended to use the same HA chassis group for all the external ports of a logical switch. Otherwise, the physical switch might see MAC flap issue when different chassis provide the native services. For example when supporting native DHCPv4 service, DHCPv4 server mac (configured in options:server_mac column in table DHCP_Options) originating from different ports can cause MAC flap issue. The MAC of the logical router IP(s) can also flap if the same HA chassis group is not set for all the external ports of a logical switch.

## 2. 配置流程

### 2.1 安装ovs
```bash
yum install -y openvswitch
systemctl enable --now openvswitch
```

### 2.2 安装/配置ovn
```bash
yum install -y unbound-libs ovn ovn-host ovn-central 

systemctl enable --now ovn-northd
systemctl enable --now ovn-controller

配置ovn controller。（在执行ovn-northd的节点执行下面的命令）。ovn-northd启动的时候会启动两个ovsdb-server进程，一个是服务northbound database，一个是用来服务southbound database,为了让其他的几点连接到southbound database，我们需要将ovsdb-server监听tcp服务，就需要执行下面的命令
ovn-sbctl set-connection ptcp:6642
```

### 2.3 创建br-int网桥,配置ovs

```
ovs-vsctl add-br br-int
设置br-int所支持的openflow协议
ovs-vsctl set bridge br-int \
    protocols=OpenFlow10,OpenFlow11,OpenFlow12,OpenFlow13,OpenFlow14,OpenFlow15

登录每个计算节点，将ovn-controller连接到southbound database。通过ovs的external_ids来配置。其中10.0.2.128是ovn-northd所在的几点
ovs-vsctl set open_vswitch .  \
  external_ids:ovn-remote=tcp:10.0.2.128:6642 \
  external_ids:ovn-encap-ip=$(ip addr show ens33| awk '$1 == "inet" {print $2}' | cut -f1 -d/) \
  external_ids:ovn-encap-type=geneve \
  external_ids:system-id=$(hostname)
验证
ovs-vsctl list open_vswitch
```

### 2.4 配置external logical port


#### 2.4.1 创建/配置 logical switch， logical switch port，dhcp option
```bash
#创建net0的logical switch
ovn-nbctl ls-add net0
#配置logical switch的subnet
ovn-nbctl set logical_switch net0 \
  other_config:subnet="10.0.2.0/24" \
  other_config:exclude_ips="10.0.2.1..10.0.2.10"
#创建DHCP option
ovn-nbctl dhcp-options-create 10.0.2.0/24
#用下面的命令来获取DHCP option的uuid
CIDR_UUID=$(ovn-nbctl --bare --columns=_uuid find dhcp_options cidr="10.0.2.0/24")
ovn-nbctl dhcp-options-set-options ${CIDR_UUID} \
  lease_time=3600 \
  router=10.0.2.3 \
  server_id=10.0.2.3 \
  server_mac=c0:ff:ee:00:00:01
# 用下面的命令来验证
ovn-nbctl list dhcp_options

# 创建逻辑端口
ovn-nbctl lsp-add net0 port1
# 配置端口的mac地址，ip地址配置为dynamic，自动获取。mac地址为物理机或者sriov网卡的mac地址
ovn-nbctl lsp-set-addresses port1 "00:0c:29:e7:ff:ac dynamic"

# 为该端口绑定dhcp options
ovn-nbctl lsp-set-dhcpv4-options port1 $CIDR_UUID

#创建完成以后，可以用下面的命令验证,可以看到分配的ip是10.0.2.11
ovn-nbctl list logical_switch_port
[root@localhost ~]# ovn-nbctl list logical_switch_port
_uuid               : db3d91cb-3ee9-4345-b203-8eecf8a92eff
addresses           : ["00:0c:29:e7:ff:ac dynamic"]
dhcpv4_options      : c074e117-b4a0-4f12-93a8-0f7e2f87e8a9
dhcpv6_options      : []
dynamic_addresses   : "00:0c:29:e7:ff:ac 10.0.2.11"
enabled             : []
external_ids        : {}
ha_chassis_group    : []
name                : port1
options             : {}
parent_name         : []
port_security       : []
tag                 : []
tag_request         : []
type                : ""
up                  : false

# 用下面的命令来模拟dhcp请求，验证是否能正常拿到ip
ovn-trace --summary net0 '
  inport=="port1" &&
  eth.src==00:0c:29:e7:ff:ac &&
  ip4.src==0.0.0.0 &&
  ip.ttl==1 &&
  ip4.dst==255.255.255.255 &&
  udp.src==68 &&
  udp.dst==67'
# 此时用ovn-sbctl dump-flows查看流表，可以看到有两条是用来解析dhcp的流表信息
[root@localhost ~]# ovn-sbctl dump-flows|grep 10.0.2.11
  table=14(ls_in_dhcp_options ), priority=100  , match=(inport == "port1" && eth.src == 00:0c:29:e7:ff:ac && ip4.src == 0.0.0.0 && ip4.dst == 255.255.255.255 && udp.src == 68 && udp.dst == 67), action=(reg0[3] = put_dhcp_opts(offerip = 10.0.2.11, lease_time = 3600, netmask = 255.255.255.0, router = 10.0.2.3, server_id = 10.0.2.3); next;)
  table=14(ls_in_dhcp_options ), priority=100  , match=(inport == "port1" && eth.src == 00:0c:29:e7:ff:ac && ip4.src == 10.0.2.11 && ip4.dst == {10.0.2.3, 255.255.255.255} && udp.src == 68 && udp.dst == 67), action=(reg0[3] = put_dhcp_opts(offerip = 10.0.2.11, lease_time = 3600, netmask = 255.255.255.0, router = 10.0.2.3, server_id = 10.0.2.3); next;)
[root@localhost ~]#
```

#### 2.4.2 配置逻辑端口类型为external
```bash
ovn-nbctl lsp-set-type port1 external
#此时再次查看流表信息，改成external后，流表信息没有了
[root@localhost ~]# ovn-sbctl dump-flows|grep 10.0.2.11
[root@localhost ~]#
```

#### 2.4.3 创建路基路由，并将它与net0逻辑交换机连接
```bash
ovn-nbctl lr-add lr0
ovn-nbctl lrp-add lr0 lr0-net0 a0:10:00:00:00:01 10.0.2.3/24
ovn-nbctl lsp-add net0 net0-lr0
ovn-nbctl set Logical_Switch_Port net0-lr0 type=router \
    options:router-port=lr0-net0 addresses=router

# 添加完router后，再次查看流表信息，有一条用作arp恢复的流表
[root@localhost ~]# ovn-sbctl dump-flows|grep 10.0.2.11
  table=12(lr_in_arp_resolve  ), priority=100  , match=(outport == "lr0-net0" && reg0 == 10.0.2.11), action=(eth.dst = 00:0c:29:e7:ff:ac; next;)

```

#### 2.4.4 创建localnet port。创建br-phys网桥，来绑定物理网卡
```bash
# 创建br-phys网桥，绑定ens37物理网卡
ovs-vsctl add-br br-phys
ovs-vsctl add-port br-phys ens37

# 在net0 逻辑交换机上创建localnet 端口
ovn-nbctl lsp-add net0 ln-public
ovn-nbctl lsp-set-addresses ln-public unknown
ovn-nbctl lsp-set-type ln-public localnet
ovn-nbctl lsp-set-options ln-public network_name=phys

# 将network phys和ovs网桥br-phys做个映射
ovs-vsctl set open . external-ids:pings=phys:br-phys
```

#### 2.4.5 创建ha_chassis_group，绑定port1关联到ha_chassis_group
```bash
# 创建ha chassis group
ovn-nbctl ha-chassis-group-add hagrp1
# 获取所有的chassis
ovn-sbctl show 或者ovn-sbctl list chassis
# 将logical switch port所在的chassis添加到ha chassis group
ovn-nbctl ha-chassis-group-add-chassis hagrp1 {chassis name} 30

#获取hagrp1的uuid
hagrp1_uuid=`ovn-nbctl --bare --columns _uuid find ha_chassis_group name="hagrp1"`
#关联port1到hagrp1
ovn-nbctl set Logical_Switch_Port port1 \
ha-chassis-group=$hagrp1_uuid
```

#### 2.4.6 此时再次查看流表
```bash
已经有dhcp相关的流表信息
[root@localhost ~]# ovn-sbctl dump-flows|grep 10.0.2.11
  table=14(ls_in_dhcp_options ), priority=100  , match=(inport == "ln-public" && eth.src == 00:0c:29:e7:ff:ac && ip4.src == 0.0.0.0 && ip4.dst == 255.255.255.255 && udp.src == 68 && udp.dst == 67 && is_chassis_resident("port1")), action=(reg0[3] = put_dhcp_opts(offerip = 10.0.2.11, lease_time = 3600, netmask = 255.255.255.0, router = 10.0.2.3, server_id = 10.0.2.3); next;)
  table=14(ls_in_dhcp_options ), priority=100  , match=(inport == "ln-public" && eth.src == 00:0c:29:e7:ff:ac && ip4.src == 10.0.2.11 && ip4.dst == {10.0.2.3, 255.255.255.255} && udp.src == 68 && udp.dst == 67 && is_chassis_resident("port1")), action=(reg0[3] = put_dhcp_opts(offerip = 10.0.2.11, lease_time = 3600, netmask = 255.255.255.0, router = 10.0.2.3, server_id = 10.0.2.3); next;)
  table=12(lr_in_arp_resolve  ), priority=100  , match=(outport == "lr0-net0" && reg0 == 10.0.2.11), action=(eth.dst = 00:0c:29:e7:ff:ac; next;)
[root@localhost ~]#
```

## 3 验证dhcp功能

登录到mac地址是00:0c:29:e7:ff:ac的虚拟机，执行dhclient ens33。可以看到准确的拿到了ip。
```bash
[root@localhost ~]# dhclient ens33
[root@localhost ~]# ifconfig
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.2.11  netmask 255.255.255.0  broadcast 10.0.2.255
        inet6 fe80::7591:8e8e:dd1a:1537  prefixlen 64  scopeid 0x20<link>
        inet6 fe80::ff75:6c29:b180:65b5  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:e7:ff:ac  txqueuelen 1000  (Ethernet)
        RX packets 145  bytes 26129 (25.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 81  bytes 12217 (11.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 532  bytes 46180 (45.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 532  bytes 46180 (45.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@localhost ~]#
```

## 4. 遇到的问题
1. 重启ovn所在的虚拟机chassis的名字改了，改成了uuid。所以添加chassis到ha-chassis-group的时候一定要用uuid。
2. 重启ovs，ovn-controller并不会自动下发流表，需要重启ovn-controller。
3. ovn-nbctl ha-chassis-group-del-chassis命令不存在，在社区提了个patch，解了。https://github.com/ovn-org/ovn/commit/358d162c071cfb56ce9fd113ccc5e3b599022fe1
4. ovs为网桥关联物理网卡的话，物理网卡不能有ip，不然网络会瘫痪。

`参考文档`

ovn简介：http://galsagie.github.io/2015/04/20/ovn-1/

ovn external logical port代相关代码和文档：
* ovn external logical port简介：https://man7.org/linux/man-pages/man7/ovn-architecture.7.html
* ovn external logical port的bug需求：https://bugzilla.redhat.com/show_bug.cgi?id=1666673
* ovn external logical portd的代码和测试用例：https://github.com/openvswitch/ovs/commit/96080083581275afaec8bc281d6a648aff7ef39e#diff-97d4cf929e4894ef95c4bfde3f896c34
* 
ovn dhcp的一个example ：https://blog.oddbit.com/post/2019-12-19-ovn-and-dhcp/

ovn gateway出网方案：https://segmentfault.com/a/1190000020349044

ovn native dhcp support：https://numans.blog/2016/08/09/native-dhcp-support-in-ovn/

ovn Genevevs VXLAN： https://blog.russellbryant.net/2017/05/30/ovn-geneve-vs-vxlan-does-it-matter/

ovs配置常见问题:（ovs绑定物理网卡时，物理网卡不能配置ip）:https://docs.openvswitch.org/en/latest/faq/issues/