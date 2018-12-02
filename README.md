# Kubernetes企业实战经验
## 第一篇：docker技术入门与应用实战
*这个板块主要分享docker的安装、镜像的拉取、docker常用命令的使用*
### 环境需求
```
服务器为2c4g配置即可

10.0.0.176-LB-master
10.0.0.178-LB-backup
10.0.0.179-master1
10.0.0.181-master2
10.0.0.182-node1
10.0.0.186-node2
10.0.0.187-Harbor
```
### 安装docker-ce
安装docker-ce社区版
官网参考链接：
https://docs.docker.com/install/linux/docker-ce/centos/#uninstall-old-versions

#### 卸载旧版本docker
```
yum remove docker \
           docker-client \
           docker-client-latest \
           docker-common \
           docker-latest \
           docker-latest-logrotate \
           docker-logrotate \
           docker-selinux \
           docker-engine-selinux \
           docker-engine
```   
#### 安装依赖包
``` 
yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
``` 
#### 安装docker-ce的yum源仓库
```
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
``` 
#### 安装docker-ce（默认安装最新稳定版）
```
yum install docker-ce
```
```
或者指定版本安装
yum list docker-ce --showduplicates | sort -r
for example, docker-ce-18.03.0.ce
yum install docker-ce-<VERSION STRING>
```
#### 启动docker
```
systemctl start docker
```
#### 卸载docker-ce
```
yum remove docker-ce
rm -rf /var/lib/docker*
```


