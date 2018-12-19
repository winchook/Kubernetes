### 部署flannel网络

#### 环境是阿里云服务器2c4g：
```
172.31.0.15-node01
172.31.0.13-node02
```
注意：etcd和flanneld使用同一套证书

#### 登录任意一个etcd节点，执行如下命令，写入分配的子网段到etcd，供flannel使用：
```
cd /opt/etcd/ssl
/opt/etcd/bin/etcdctl \
--ca-file=ca.pem --cert-file=server.pem --key-file=server-key.pem \
--endpoints="https://172.31.0.14:2379,https://172.31.0.15:2379,https://172.31.0.13:2379" \
set /coreos.com/network/config '{"Network":"172.17.0.0/16","Backend":{"Type":"vxlan"}}'
```

#### 登录任意一个节点确保可以查看到刚才写入的数据，验证能否获取到刚才写入的数据
```
cd /opt/etcd/ssl
/opt/etcd/bin/etcdctl \
--ca-file=ca.pem --cert-file=server.pem --key-file=server-key.pem \
--endpoints="https://172.31.0.14:2379,https://172.31.0.15:2379,https://172.31.0.13:2379" \
get /coreos.com/network/config
```

#### 下载flannel二进制包
```
cd /root/k8s/
wget https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
tar zxf flannel-v0.10.0-linux-amd64.tar.gz
flanneld表示启动文件
mk-docker-opts.sh表示脚本用于生成一个子网并写入文件中，docker来读取这个文件指定这个子网启动
```

#### 创建kubernetes组件的目录：
```
mkdir -p /opt/kubernetes/{cfg,bin,ssl}
```

#### 复制二进制启动文件到
```
cp flanneld  mk-docker-opts.sh /opt/kubernetes/bin/
```
#### 参照课件中的脚本
```
flannel.sh
```
/bin/bash flannel.sh https://172.31.0.14:2379,https://172.31.0.15:2379,https://172.31.0.13:2379
这时flannel启动会报错，因为后面需要再配置docker

#### 修改docker自启动文件
```
vim /usr/lib/systemd/system/docker.service
#ExecStart=/usr/bin/dockerd -H unix://
EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
因为，需要docker用到这个子网
```
再进行配置子网的配置文件
创建目录：
mkdir /run/flannel
```
[root@k-node01 ~]# cat /run/flannel/subnet.env
DOCKER_OPT_BIP="--bip=172.17.88.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=false"
DOCKER_OPT_MTU="--mtu=1450"
DOCKER_NETWORK_OPTIONS=" --bip=172.17.88.1/24 --ip-masq=false --mtu=1450"
```
#### 重新加载配置文件
```
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
```
#### 重启docker
```
systemctl restart docker
```
#### 复制相关配置文件到node01节点172.31.0.13
```
scp -r /opt/kubernetes 172.31.0.13:/opt/
scp -r /run/flannel 172.31.0.13:/run/
scp /usr/lib/systemd/system/flanneld.service 172.31.0.13:/usr/lib/systemd/system/
```
#### 再172.31.0.13修改docker自启动文件
```
vim /usr/lib/systemd/system/docker.service
#ExecStart=/usr/bin/dockerd -H unix://
EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
```
#### 重新加载172.31.0.13配置文件
```
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
```
#### 重启172.31.0.13 docker
```
systemctl restart docker
```
#### 这时flannel的网段和docker的网段是一样的
```
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:7b:55:0e:39 brd ff:ff:ff:ff:ff:ff
    inet 172.17.19.1/24 brd 172.17.19.255 scope global docker0
       valid_lft forever preferred_lft forever
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
    link/ether 22:0e:0b:11:03:40 brd ff:ff:ff:ff:ff:ff
    inet 172.17.19.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::200e:bff:fe11:340/64 scope link 
       valid_lft forever preferred_lft forever
```
至此，flannel配置完成！

#### 验证可以跨主机通信
```
登录node01，分别ping node02的docker网卡及flannel网卡，互通
[root@k-node01 ~]# ping 172.17.64.1
PING 172.17.64.1 (172.17.64.1) 56(84) bytes of data.
64 bytes from 172.17.64.1: icmp_seq=1 ttl=64 time=0.406 ms
64 bytes from 172.17.64.1: icmp_seq=2 ttl=64 time=0.306 ms
^C
--- 172.17.64.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.306/0.356/0.406/0.050 ms
[root@k-node01 ~]# ping 172.17.64.0
PING 172.17.64.0 (172.17.64.0) 56(84) bytes of data.
64 bytes from 172.17.64.0: icmp_seq=1 ttl=64 time=0.315 ms
64 bytes from 172.17.64.0: icmp_seq=2 ttl=64 time=0.248 ms
64 bytes from 172.17.64.0: icmp_seq=3 ttl=64 time=0.292 ms
64 bytes from 172.17.64.0: icmp_seq=4 ttl=64 time=0.293 ms
64 bytes from 172.17.64.0: icmp_seq=5 ttl=64 time=0.235 ms
^C
--- 172.17.64.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4000ms
rtt min/avg/max/mdev = 0.235/0.276/0.315/0.035 ms
```
#### 再验证容器之间可以通信
```
运行busybox
登录172.31.0.15-node01
[root@k-node01 ~]# docker container run -it --rm busybox
/ # ping 172.17.64.0
PING 172.17.64.0 (172.17.64.0): 56 data bytes
64 bytes from 172.17.64.0: seq=0 ttl=63 time=0.502 ms
64 bytes from 172.17.64.0: seq=1 ttl=63 time=0.340 ms
^C
--- 172.17.64.0 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.340/0.421/0.502 ms
/ # ping 172.17.64.2
PING 172.17.64.2 (172.17.64.2): 56 data bytes
64 bytes from 172.17.64.2: seq=0 ttl=62 time=0.526 ms
64 bytes from 172.17.64.2: seq=1 ttl=62 time=0.358 ms
^C
--- 172.17.64.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.358/0.442/0.526 ms
```
#### 【附】对虚拟网卡进行清理
```
ip link delete docker0
ip link delete flannel.1
```
#### 重启docker
```
system restart docker
```
