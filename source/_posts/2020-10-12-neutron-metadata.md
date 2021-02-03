---
title: neutron metadata 服务详解
date:  2020-10-12
tags:
    - openstack
    - neutron
---

本文简单介绍一下neutron中metadata是如何对接nova-metadtata为虚拟机提供metadata服务的。

<!-- more -->

# 1. Metadata服务简介
OpenStack的Metadata服务主要是为虚拟机提供配置信息。在虚拟机启动的时候，会向Metadata服务请求自己的metadata信息，然后执行cloud-init的时候完成个性化配置。

## 1.1 工作流程
整个的工作流程如下图所示。经过neutron-metadata-agent进行转发的原因是，nova-api-metadata服务运行在管理网络，而虚拟机是运行在业务网络，两者可能不能通信。而在neutron-metadata-agent之前又嫁接一层neutron-ns-metadata-proxy的原因是在openstack中，不同的网络可以配置相同的CIDR，nova-api-metadata无法根据虚拟机的ip来判定虚拟机的id，所以需要在neutron-ns-metadata-agent服务添加network-id后者router-id。这样就可以根据instance ip，network-id/router-id来获取到唯一的虚拟机id。
![](/img/openstack/neutron/metadata-1.png)

1. 虚拟机启动的时候会配置默认路由  
虚拟机启动的时候，会进行默认路由的配置，配置访问169.254.169.254的路由信息
2. 虚拟机请求169.254.169.254  
   请求到达`qrouter-<network-id>`或者`qdhcp-<network-id>`的网络命名空间。如果该网络没有配置router，是到达`qdhcp-<network-id>`的网络命名空间，如果配有router，会到达`qrouter-<network-id>`的网络命名空间。
3. 请求到达`qrouter-<network-id>`或者`qdhcp-<network-id>`网络命名空间  
请求到达网络命名空间以后，经由neutron-ns-metadata-proxy，添加router-id或者network-id，然后通过socket转发给neutron-metadata-agent。
4. 请求通过unix socket转发给neutron-metadata-agent服务  
neutron-metadata-agent通过请求头部的network-id/router-id以及请求的ip地址，获取到instance id，将请求转发给nova-api-metadata。
5. neutron-metadata-agent服务将请求转发给nova-api-metadata。  
nova-api-metadata根据instance id回复metadata response。

# 2. 基于`qrouter-<network-id>`命名空间的metadata服务实现

当openstack中的网络连接router时，neutron-metadata-agent通过`qrouter-<network-id>`网络命名空间来提供metadata服务。

此时，登录虚拟机，看一下路由规则。可以看到在请求169.254.169.254时，通过gateway进行转发，而gateway是配置在`qrouter-<network-id>` 网络命名空间下的
```bash
#虚拟机路由
[root@test ~]# ip route
default via 10.0.0.1 dev eth0
10.0.0.0/24 dev eth0 proto kernel scope link src 10.0.0.14
169.254.169.254 via 10.0.0.1 dev eth0 proto static

#qrouter-网络命名空间信息
[root@tstack-con01 ~]# ip netns exec qrouter-f8156a50-cf9e-430c-a8a6-64a87e41fd01 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
31: qr-6a921526-09: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:6f:53:5d brd ff:ff:ff:ff:ff:ff
    inet 10.1.0.1/24 brd 10.1.0.255 scope global qr-6a921526-09
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe6f:535d/64 scope link
       valid_lft forever preferred_lft forever
32: qr-853fe99d-e7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:5d:07:11 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/24 brd 10.0.0.255 scope global qr-853fe99d-e7
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe5d:711/64 scope link
       valid_lft forever preferred_lft forever
```
那么请求在`qrouter-<network-id>`的命名空间是如何转发的呢？答案是通过iptables

```bash
[root@tstack-con01 ~]# ip netns exec qrouter-f8156a50-cf9e-430c-a8a6-64a87e41fd01 iptables-save|grep 169
-A neutron-l3-agent-PREROUTING -d 169.254.169.254/32 -i qr-+ -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 9697
-A neutron-l3-agent-PREROUTING -d 169.254.169.254/32 -i qr-+ -p tcp -m tcp --dport 80 -j MARK --set-xmark 0x1/0xffff
```
会把对169.254.169.254，80端口的请求转发到9697端口。而9697端口就运行着neutron-ns-metadata-proxy服务，该服务实际上是一个haproxy进行。

```
[root@tstack-con01 ~]# ip netns exec qrouter-f8156a50-cf9e-430c-a8a6-64a87e41fd01 lsof -i:9697
COMMAND     PID    USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
haproxy 1592727 neutron    4u  IPv4 27760235      0t0  TCP *:9697 (LISTEN)
[root@tstack-con01 ~]# ip netns exec qrouter-f8156a50-cf9e-430c-a8a6-64a87e41fd01 ps -aux|grep 1592727
neutron  1592727  0.0  0.0  47940  1196 ?        Ss   Feb01   0:01 haproxy -f /var/lib/neutron/ns-metadata-proxy/f8156a50-cf9e-430c-a8a6-64a87e41fd01.conf
root     1856176  0.0  0.0 112708   952 pts/1    S+   00:34   0:00 grep --color=auto 1592727
[root@tstack-con01 ~]#
```
查看一下配置文件/var/lib/neutron/ns-metadata-proxy/

```bash
[root@tstack-con01 ~]# cat /var/lib/neutron/ns-metadata-proxy/f8156a50-cf9e-430c-a8a6-64a87e41fd01.conf

global
    log         /dev/log local0 debug
    log-tag     haproxy-metadata-proxy-f8156a50-cf9e-430c-a8a6-64a87e41fd01
    user        neutron
    group       neutron
    maxconn     1024
    pidfile     /var/lib/neutron/external/pids/f8156a50-cf9e-430c-a8a6-64a87e41fd01.pid
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor
    retries                 3
    timeout http-request    30s
    timeout connect         30s
    timeout client          32s
    timeout server          32s
    timeout http-keep-alive 30s

listen listener
    bind 0.0.0.0:9697
    server metadata /var/lib/neutron/metadata_proxy
    http-request add-header X-Neutron-Router-ID f8156a50-cf9e-430c-a8a6-64a87e41fd01
```
实际上neutron-ns-metadata-proxy用过socket连接neutron-metadata-agent，然后在请求头部添加router-id，neutorn-metadata-agent就会根据请求的ipd地址和router-id获取到虚拟机的id。然后将请求发送给nova-api-metadata服务。

# 3. 基于`qdhcp-<network-id>`命名空间的metadata服务实现

如果网络没有绑定router，那么neutron通过`qdhcp-<network-id>`网络命名空间来处理metadata请求。

此时的实现方式，会在`qdhcp-<network-id>`网络命名空间添加一个169.254.169.254的网卡。

```bash
[root@openstack-con01 ~]# ip netns exec qdhcp-35a559be-9f45-4d4e-849f-6c58b7e8ddd1 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
10: tap768dfac8-23: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:fd:4d:8e brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.2/24 brd 172.16.0.255 scope global tap768dfac8-23
       valid_lft forever preferred_lft forever
    inet 169.254.169.254/16 brd 169.254.255.255 scope global tap768dfac8-23
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fefd:4d8e/64 scope link
       valid_lft forever preferred_lft forever
```

neutron-ns-metadata-proxy运行在80端口，实际就是一个haproxy服务。这样对169.254.169.254的80端口的访问都会直接到达该服务。

```bash
[root@openstack-con01 ~]# ip netns exec qdhcp-35a559be-9f45-4d4e-849f-6c58b7e8ddd1 lsof -i:80
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
haproxy 5443 neutron    4u  IPv4  41711      0t0  TCP *:http (LISTEN)
[root@openstack-con01 ~]# ip netns exec qdhcp-35a559be-9f45-4d4e-849f-6c58b7e8ddd1 ps -aux|grep 5443
neutron     5443  0.0  0.0  47936   868 ?        Ss    2020  15:39 haproxy -f /var/lib/neutron/ns-metadata-proxy/35a559be-9f45-4d4e-849f-6c58b7e8ddd1.conf
root     3203881  0.0  0.0 112708   968 pts/0    S+   18:20   0:00 grep --color=auto 5443
```

看一下haproxy的配置文件，实际上通过socket连接neutron-metadata-agent，然后在请求的头部添加network id。这样neutron-metadata-agent就可以通过虚拟机的ip和network的id来获取instance id，然后想nova-api-metadata请求对应instance的metadata信息。
```bash
[root@openstack-con01 ~]# cat /var/lib/neutron/ns-metadata-proxy/35a559be-9f45-4d4e-849f-6c58b7e8ddd1.conf

global
    log         /dev/log local0 info
    log-tag     haproxy-metadata-proxy-35a559be-9f45-4d4e-849f-6c58b7e8ddd1
    user        neutron
    group       neutron
    maxconn     1024
    pidfile     /var/lib/neutron/external/pids/35a559be-9f45-4d4e-849f-6c58b7e8ddd1.pid
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor
    retries                 3
    timeout http-request    30s
    timeout connect         30s
    timeout client          32s
    timeout server          32s
    timeout http-keep-alive 30s

listen listener
    bind 0.0.0.0:80
    server metadata /var/lib/neutron/metadata_proxy
    http-request add-header X-Neutron-Network-ID 35a559be-9f45-4d4e-849f-6c58b7e8ddd1
```

# 4. 源码分析

1. neutron-ns-metadata-proxy的配置文件生成代码 [view code](https://github.com/openstack/neutron/blob/805359d9a2b53db431c698e946ba62fe3b7213af/neutron/agent/metadata/driver.py#L97)
2. neutron-metadata-agent的处理函数 [view code](https://github.com/openstack/neutron/blob/805359d9a2b53db431c698e946ba62fe3b7213af/neutron/agent/metadata/agent.py#L84)
3. neutron-metadata-agent从请求中获取虚拟机ip，router-id/network-id，然后根据这些信息获取虚拟机的id [view code](https://github.com/openstack/neutron/blob/805359d9a2b53db431c698e946ba62fe3b7213af/neutron/agent/metadata/agent.py#L156)
4. neutron-metadata-agent把请求转发给nova-api-metadata [view code](https://github.com/openstack/neutron/blob/805359d9a2b53db431c698e946ba62fe3b7213af/neutron/agent/metadata/agent.py#L90)

`参考文档`
- https://engineering.linecorp.com/en/blog/openstack-summit-vancouver-2018-recap-2-2/
- https://docs.openstack.org/networking-ovn/latest/contributor/design/metadata_api.html#neutron-and-metadata-today
- https://www.99cloud.net/10213.html%EF%BC%8F