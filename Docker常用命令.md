### Docker常用命令
#### 从docker官方拉取镜像(以nginx为例)
https://hub.docker.com/r/library/nginx/tags/
#### 查看官方镜像的使用手册
https://hub.docker.com/r/library/nginx/
#### 拉取镜像
```
docker pull nginx:1-alpine-perl
```
注意：nginx:1-alpine-perl表示 "镜像名:tag"，不写镜像仓库地址，默认访问官方镜像仓库。
#### 查看镜像
```
docker image ls
```
#### 用镜像启动一个容器
```
以指定目录挂载
docker container run -d --name nginx --mount type=bind,src=/data/wwwroot,dst=/usr/share/nginx/html -p 88:80 nginx:1-alpine-perl
以标签名方式挂载
docker container run -d --name nginx --mount src=wwwroot,dst=/usr/share/nginx/html -p 88:80 nginx:1-alpine-perl
```
#### 删除所有正在运行的容器
```
docker container rm -f $(docker container ps -a | awk '{print $1}')
```
#### 查看正在运行的容器
```
docker container ps -a
```
#### 使用dockerfile构建一个镜像
```
docker image build -f Dockerfile -t nginx:v1 .
```
#### 查看镜像分层
```
docker history nginx:v1
```
