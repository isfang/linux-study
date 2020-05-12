[TOC]

ip a  代替ifconfig的命令 可以查看网卡的ip、mac等
ip link list 显示网络设备的运行状态  ip link只能看链路层的状态，看不到ip地址
ip netns list 显示namespace
ip netns exec test1 ip a

[https://wangchujiang.com/linux-command/c/ip.html#!kw=ip%20a](https://wangchujiang.com/linux-command/c/ip.html#!kw=ip a)


##### 创建两个namespace 就是一个独立的容器

`ip netns add test1`
`ip netns add test2`

此时我们执行`ip netns exec test1 set dev lo up` 无法up该namespace 因为没有创建一对veth，于是我们先创建一对veth-peer



##### 创建一对veth-pair

`ip link add veth-test1 type veth peer name veth-test2`

(veth-pair 就是一对的虚拟设备接口，和 tap/tun 设备不同的是，它都是成对出现的。一端连着协议栈，一端彼此相连着,https://www.cnblogs.com/bakari/p/10613710.html)

此时我们执行 ip link 可以看到宿主机已经有了两个网卡



##### 将veth-pair设置到namespace里去

先把 veth-test1 添加到 test1 这个namespace里去
[root@docker-node1 ~]# ip netns exec test1 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

`ip link set veth-test1 netns test1`
`ip link set veth-test2 netns test2 `
这行命令会把 上面 的 宿主机的link加到test1  这个namespace里去



##### 给veth-pair添加ip地址

这个时候 veth-test1 2 都添加到了 test1 test2里去 但是这个时候 这两个namespqce都是没有ip的只有mac地址 状态也是down的

下面我们去配置ip地址 给 veth-test1 2 分别添加ip地址 掩码是24

`ip netns exec test1 ip addr add 192.168.1.1/24 dev veth-test1`
`ip netns exec test2 ip addr add 192.168.1.2/24 dev veth-test2`



##### 启动接口

ip netns exec test1 ip link 再次执行 会发现 还是没有ip地址 还是都是down的 

`ip netns exec test1 ip link set dev veth-test1 up`
`ip netns exec test2 ip link set dev veth-test2 up`

此时已经可以从两个namespace拼通了

ip netns exec test1 ping 192.168.1.2



##### docker bridge

通过`docker network ls` 展示出3种网络模式

[root@docker-node1 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
3d7583035603        bridge              bridge              local
add93e527ff1        host                host                local
14985b5a112b        none                null                local



docker 默认网络 bridge 这种网络就是我们执行 ip a 时看到的docker0接口



在我们执行

`docker network inspect 3d7583035603 或者 docker network inspect bridge`时我们可以看到连到这个bridge上的容器

我们的container 会连到这个docker0这个bridge上去, 通过该命令可以查看bridge信息



[root@docker-node1 ~]# ip a
12: vethb89494d@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 6a:df:aa:a9:47:28 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::68df:aaff:fea9:4728/64 scope link
       valid_lft forever preferred_lft forever
[root@docker-node1 ~]# docker exec test1 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever



eth0@if12 跟 vethb89494d@if11 是一对veth pair  vethb89494d@if11 是连到了docker0这个接口上面



brctl show可以查看机器上的bridge 与 对应的链接情况

vethb89494d@if11  连到了 docker0  vethb89494d@if11 又跟 docker 容器里的 eth0@if12 是一对 veth-pair



这解释了容器之间如何通信



##### 容器如何链接外网

NAT 通过ip table实现

