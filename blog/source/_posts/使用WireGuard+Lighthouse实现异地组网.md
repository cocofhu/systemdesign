---
title: 使用WireGuard+Lighthouse实现异地组网
date: 2025-04-29 23:49:34
---

> WireGuard是一个易于配置、快速、安全的基于UDP协议的开源VPN软件。WireGuard具有自定义配置路由转发的能力，所以可以被用来在多个不同地域将设备所在的内网网络通过路由转发的方式串通起来，组建一张属于自己的大内网。

> 通过使用WireGuard组建一个大内网（类似于腾讯的iOA），可以让小伙伴们加入同一个内网， 局域网联机开黑。

> 本文将介绍如何使用lighthouse+WireGuard，通过lighthouse作为跳板机登陆到家庭内网，并和不同地域的伙伴局域网联机。

### 一、安装WireGuard
在腾讯云购买200M的轻量服务器，选择ubuntu镜像，登陆到系统，执行如下命令安装WireGuard和开启转发：
```shell
apt install wireguard -y
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

### 二、生成公钥和私钥
由于WireGuard需要加密连接，所以我们要先生成相关密钥，为了处理方便，我这里吧所有的文件都放在了`/etc/wireguard/`，也就是WireGuard的默认配置路径：
```shell
# 进入路径，并设置权限，权限过高启动的时候有warning
cd /etc/wireguard/
chmod 077 /etc/wireguard
umask 077
# 生成服务器公钥和私钥
wg genkey > server.key
wg pubkey < server.key > server.key.pub
# 生成客户端公钥和私钥
wg genkey > client1.key
wg pubkey < client1.key > client1.key.pub
```

执行上述脚本后我们将得到如下四个文件：
- server.key 服务器的私钥
- server.key.pub 服务器的公钥
- client1.key 客户端1的私钥
- client1.key.pub 客户端1的公钥

如果有更多的客户端需要接入、可以生成更多的client。

### 三、编写服务器的配置文件
我们在`/etc/wireguard/`下创建一个`wg0.conf`文件：
```text
[Interface]
PrivateKey = xxx 
Address = 10.5.1.1 

PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE


ListenPort = 50814 
MTU = 820
[Peer]
PublicKey =  xxxx
AllowedIPs = 10.5.1.11/32,192.168.2.0/24

[Peer]
PublicKey =  xxxx
AllowedIPs = 10.5.1.12/32
```
在第二行和第三行中，我们需要配置服务器的私钥（这里使用server.key里面的内容）和IP（使用自己喜欢的IP，PS：五一放假了）。
然后PostUp、PostDown、ListenPort、MTU可以先保持不动，这里需要注意要放通lighthouse相关的安全组。
最后我们定义了两个客户端，如果你只有一个就删掉一个，在PublicKey这里填写客户端的公钥，AllowedIPs表示的是那些IP通过这个节点作为网关出去(我感觉这个应该叫Route)，服务启动的时候会创建相关的路由表，可以使用`ip route` 查看， 这里我用了一个我的内网IP，想让它把内网IP转发到第一个Peer，从而实现跳板机的功能。

### 四、启动服务
执行以下命令将会读取`wg0.conf`文件，并启动服务
```shell
wg-quick up wg0
```
启动成功后我们可以通过`ip route` 看看机器的路由表：
```shell
10.5.1.11 dev wg0 scope link 
10.5.1.12 dev wg0 scope link 
192.168.2.0/24 dev wg0 scope link
```
可以看到我们配置IP都通过`wg0`网卡出去。

### 五、配置客户端
在内网客户端上安装WireGuard（步骤一），创建`gateway.conf`:
```text
[Interface]
PrivateKey = xxx
Address = 10.5.1.11 
MTU = 820

[Peer]
PublicKey = xxx
AllowedIPs = 10.5.1.0/24 
Endpoint = lighthouse_ip:50814 
```
将PrivateKey改为第二步生成的`client1.key`的内容（节点的私钥），PublicKey改为第二步生成的`server.key.pub`的内容（对端的公钥），AllowedIPs使用`10.5.1.0/24`网段，表示`10.5.1.0/24`走这个接口， 最后将Endpoint改为lighthouse的公网IP（注意放通UDP端口）。

### 六、测试
在局域网节点执行ping命令：
```text
ping 10.5.1.1
```
网络畅通表示配置成功，也可以用`wg show` 查看节点状态。

### 七、开启网卡转发实现跳板机相关功能
使用刚刚client作为内网中转节点，开启转发，这里我的物理网卡是enp1s0：
```shell
# 允许从物理网卡(enp1s0)进入，转发到gateway虚拟接口的流量
iptables -A FORWARD -i enp1s0 -o gateway -j ACCEPT

# 允许从gateway虚拟接口进入，转发到物理网卡(enp1s0)的流量
iptables -A FORWARD -i gateway -o enp1s0 -j ACCEPT

# 对从物理网卡(enp1s0)流出的数据进行源地址伪装
iptables -t nat -A POSTROUTING -o enp1s0 -j MASQUERADE
```
执行上面命令后就可以通过lighthouse直接ssh 192.168.2.x连接上内网机器了。

### 八、更多用户接入开黑
如果需要更多用户接入可以创建更多的client、接入到整改局域网，不同的系统可以通过WireGuard提供的客户端接入。

### 九、Troubleshooting
#### 1、大文件传输失败、延迟巨高，丢包严重
- 可以调整MTU，在测试移动网络和电信网络组网时，使用默认的1420有问题，调小MTU时问题得到解决，具体原因可以百度或者问AI。
#### 2、连接不上
- 放通lighthouse的相关端口，检查相关密钥是否弄混了。




















