### 安装前注意事项：
确保所有服务器时间及时区是同步的
```
环境是阿里云服务器2c4g：
172.31.0.14-master01
172.31.0.15-node01
172.31.0.13-node02
```
### 安装etcd数据库集群
```
确保进程及安装包删除干净
登录master1
创建目录
mkdir -p /root/k8s
上传文件
cfssl.sh
etcd-cert.sh
etcd.sh
这里是用的是CFSSL（CloudFlare的PKI工具包） - CloudFlare's PKI toolkit生成证书，相对OpenSSL而言，简单很多。
cd /root/k8s
mkdir etcd-cert
mv etcd-cert.sh etcd-cert/
```
```
执行脚本cfssl.sh，来下载cfssl这个包，并赋予其可执行权限

cfssl作用：是来生成证书的
cfssljson作用：通过传入json文件来生成证书，和cfssl配合使用
cfssl-certinfo作用：查看生成的证书的信息
```
### 生成证书配置文件
```
cd /root/k8s/etcd-cert

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF
```
### 生成CSR证书签名请求文件
```
cat > ca-csr.json <<EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF
```
### 生成签名证书
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
从而生成了ca.csr ca-key.pem ca.pem可以理解为证书签发机构
下面为etcd来颁发https证书，这里的IP是需要安装etcd的服务器IP
```
```
cat > server-csr.json <<EOF
{
    "CN": "etcd",
    "hosts": [
    "172.31.0.14",
    "172.31.0.15",
    "172.31.0.13"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF
```
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
server.csr
server-key.pem
server.pem
```
到此etcd的证书已经签发完成

### 接下来安装etcd集群
```
下载etcd v3.3.10
wget https://github.com/etcd-io/etcd/releases/download/v3.3.10/etcd-v3.3.10-linux-amd64.tar.gz
解压etcd
tar -zxf etcd-v3.3.10-linux-amd64.tar.gz
cd /root/k8s/etcd-v3.3.10-linux-amd64
etcd表示启动的二进制包
etcdctl客户端工具
```
```
创建etcd存放路径
mkdir -p /opt/etcd/{cfg,bin,ssl} -p
bin表示存放二进制文件
cfg表示存放配置文件
ssl表示存放证书文件
```
```
mv etcd etcdctl /opt/etcd/bin/

拷贝证书到指定目录
cp /root/k8s/etcd-cert/*pem /opt/etcd/ssl/
生成etcd配置文件及systemctl自启动
cd /root/k8s
/bin/bash etcd.sh etcd01 172.31.0.14 etcd02=https://172.31.0.15:2380,etcd03=https://172.31.0.13:2380
```
```
查看证书信息：
cfssl-certinfo -cert /opt/etcd/ssl/server.pem
```
```
分别复制如下文件到另外两台服务器
scp -r /opt/etcd  172.31.0.15:/opt/
scp -r /opt/etcd  172.31.0.13:/opt/
scp /usr/lib/systemd/system/etcd.service 172.31.0.15:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/etcd.service 172.31.0.13:/usr/lib/systemd/system/
```
```
复制到另外两个etcd节点后需要修改
vim /opt/etcd/cfg/etcd
#ETCD_NAME="etcd02"
ETCD_LISTEN_PEER_URLS
ETCD_LISTEN_CLIENT_URLS
ETCD_INITIAL_ADVERTISE_PEER_URLS
ETCD_ADVERTISE_CLIENT_URLS
对应的IP
```
```
关闭服务器的iptables

逐一启动etcd（第一个etcd会报连接超时，因为后面两个节点还未启动，继续启动即可）
systemctl start etcd

查看集群成员
/opt/etcd/bin/etcdctl --ca-file=ca.pem --cert-file=server.pem --key-file=server-key.pem member list
```
```
在任意一个节点均可查看etcd集群状态
cd /opt/etcd/ssl
/opt/etcd/bin/etcdctl \
--ca-file=ca.pem --cert-file=server.pem --key-file=server-key.pem \
--endpoints="https://172.31.0.14:2379,https://172.31.0.15:2379,https://172.31.0.13:2379" \
cluster-health
```
到此etcd https方式集群搭建完成！
