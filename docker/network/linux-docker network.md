[TOC]

ip a  代替ifconfig的命令 可以查看网卡的ip、mac等
ip link list 显示网络设备的运行状态  ip link只能看链路层的状态，看不到ip地址
ip netns list 显示namespace
ip netns exec test1 ip a

[https://wangchujiang.com/linux-command/c/ip.html#!kw=ip%20a](https://wangchujiang.com/linux-command/c/ip.html#!kw=ip a)

# Docker namespace

## 创建两个namespace 就是一个独立的容器

`ip netns add test1`
`ip netns add test2`

此时我们执行`ip netns exec test1 set dev lo up` 无法up该namespace 因为没有创建一对veth，于是我们先创建一对veth-peer


## 创建一对veth-pair

`ip link add veth-test1 type veth peer name veth-test2`

(veth-pair 就是一对的虚拟设备接口，和 tap/tun 设备不同的是，它都是成对出现的。一端连着协议栈，一端彼此相连着,https://www.cnblogs.com/bakari/p/10613710.html)

此时我们执行 ip link 可以看到宿主机已经有了两个网卡



## 将veth-pair设置到namespace里去

先把 veth-test1 添加到 test1 这个namespace里去

```
[root@docker-node1 ~]# ip netns exec test1 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

`ip link set veth-test1 netns test1`
`ip link set veth-test2 netns test2 `
这行命令会把 上面 的 宿主机的link加到test1  这个namespace里去



## 给veth-pair添加ip地址

这个时候 veth-test1 2 都添加到了 test1 test2里去 但是这个时候 这两个namespqce都是没有ip的只有mac地址 状态也是down的

下面我们去配置ip地址 给 veth-test1 2 分别添加ip地址 掩码是24

`ip netns exec test1 ip addr add 192.168.1.1/24 dev veth-test1`
`ip netns exec test2 ip addr add 192.168.1.2/24 dev veth-test2`



## 启动接口

ip netns exec test1 ip link 再次执行 会发现 还是没有ip地址 还是都是down的 

`ip netns exec test1 ip link set dev veth-test1 up`
`ip netns exec test2 ip link set dev veth-test2 up`

此时已经可以从两个namespace拼通了

ip netns exec test1 ping 192.168.1.2



# Docker Network

## Docker bridge

### 容器之间如何通信
通过`docker network ls` 展示出3种网络模式

```
[root@docker-node1 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
3d7583035603        bridge              bridge              local
add93e527ff1        host                host                local
14985b5a112b        none                null                local
```

docker 默认网络 bridge 这种网络就是我们执行 ip a 时看到的docker0接口

在我们执行

`docker network inspect 3d7583035603 或者 docker network inspect bridge`时我们可以看到连到这个bridge上的容器

我们的container 会连到这个docker0这个bridge上去, 通过该命令可以查看bridge信息



```
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
```



eth0@if12 跟 vethb89494d@if11 是一对veth pair  vethb89494d@if11 是连到了docker0这个接口上面



brctl show可以查看机器上的bridge 与 对应的链接情况

vethb89494d@if11  连到了 docker0  vethb89494d@if11 又跟 docker 容器里的 eth0@if12 是一对 veth-pair



这解释了容器之间如何通信



### 容器如何链接外网

NAT 通过ip table实现



### Docker link

#### --link

在创建容器test1的时候 加上 --link test2  test2为其他容器的名字 容器在test1 就可以直接 ping test2 不需要知道test2的ip地址  直接使用test2 代替 ip地址

#### 自定义bridge 创建网络

我们通过命令 docker network create -d bridge my-bridge 创建一个my-bridge网络

```
[vagrant@docker-node1 ~]$ docker network create -d bridge my-bridge
f402c072ea9b6e960c90f5b711a7a983a1251f05de550aca3fa10b2af2af8c57
[vagrant@docker-node1 ~]$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
3d7583035603        bridge              bridge              local
add93e527ff1        host                host                local
f402c072ea9b        my-bridge           bridge              local
14985b5a112b        none                null                local
```



此时我们自行 brctl show 可以看到多了一个bridge

```
[vagrant@docker-node1 ~]$ brctl show
bridge name	bridge id		STP enabled	interfaces
br-f402c072ea9b		8000.02426e21f556	no
docker0		8000.0242134ff385	no		vethb89494d
```



##### 在创建容器的时候指定bridge

我们创建容器的时候 再通过 `--network my-bridge`指定自己创建的bridge

`docker run -d --name test3 --network my-bridge  busybox /bin/sh -c "while true; do sleep 3600; done"`

再次 执行`brctl show`就可以看到我们的my-bridge已经有接口了 已经有容器连过去了
bridge name	bridge id		STP enabled	interfaces
br-f402c072ea9b		8000.02426e21f556	no		veth4402057
docker0		8000.0242134ff385	no		vethb89494d

同时我们执行docker network inspect也可以看到 这个bridge 多了个容器



##### 指定已经存在的容器

`docker network connect my-bridge test1`  这样的话就把test1也链接my-bridge了，但是这个时候test1 即连接到了my-bridge  也连到了 老的bridge



##### 特殊说明

在连接到自己创建的my-bridge上的两个容器不需要在执行 `docker link`  默认就link好了 emmmm不是很懂，已经可以直接通过名字就ping通了



### 端口映射

-p 指定端口映射



## Docker none

创建容器的时候 指定`--network none` 创建的容器将挂到none 这个网络里去，通过`docker network inspect none` 可以看到挂到none网络下的 容器 没有mac地址 没有ip地址， 在容器里 也只有一个 回还口  这种网络 模式只有exec 才可以进到容器里面去  

```
none 网络就是什么都没有的网络，一些对安全性有求高并且不需要联网的应用可以使用 none 网络。
比如：某个容器的唯一用途是生成随机密码，就可以放到 none 网络中避免密码被窃取。
```



## Docker host

创建容器的时候 指定`--network host` 创建的容器将挂到host 这个网络, 通过`docker network inspect none` 可以看到挂到none网络下的 容器 没有mac地址也没有ip地址，但是我们进入容器里面执行 `ip a`容器内部的网络跟外部的网络是一样的

没有自己独立的 network namespace 跟主机共享一套



端口可能会有冲突



## 多机通信

VXLAN

overlay  underlay



# Docker compose



docker-compose up

docker-compose up -d

docker-compose up -f xxxx.yml  -f默认为docker-compose.yml 

docker-compose ps 查看当前的compose service

docker-compose stop 停止service 不会删除

docker-compose start

docker-compose down 停止并删除

docker-compose exec xxservice bash ...exec 后面跟service

docker-compose up --scale xxservice=Num 单机。。。。



# Docker Swarm