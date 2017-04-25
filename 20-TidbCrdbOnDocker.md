# # docker swarm 搭建tidb和crdb
## 参考文档链接
## 集群服务器环境
4节点Docker集群系统已经安装完成。如下：
172.16.210.180~183
root/Skxx
### 配置环境变量：
```
nmcli connection show
nmcli con mod enp2s0 ipv4.dns "192.168.100.1 8.8.8.8"
nmcli con up enp2s0
hostnamectl set-hostname dockerx.localdomain
```
### 升级Centos

```
cd /etc/yum.repos.d/
mkdir bak
mv *.repo bak/
curl -L -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo \
&& curl -L -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum makecache
yum -y update
```
重启服务器：
`reboot`

## 安装docker

```
sudo yum -y remove docker docker-common container-selinux
sudo yum -y remove docker-selinux
```

```
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
sudo yum makecache fast
sudo yum install -y docker-ce
```


```
sudo systemctl enable docker.service
sudo systemctl start docker 
sudo docker run hello-world
```
## 配置docker参数，支持私有仓库
vi /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd --insecure-registry beagledata.com:5000
systemctl stop docker
systemctl daemon-reload
sudo systemctl start docker 
私有仓库管理：
http://beagledata.com:5080/
加速：
https://4b517mxr.mirror.aliyuncs.com
## docker swarm 初始化

`docker swarm init`

显示如下：

```
Swarm initialized: current node (rvjykhpfhal2z7p0i4rnaputw) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1do9eqddn48dj3ncm03m2201y79g889k1n49ywp4mh6qqxa5aa-dobqd6j96qk5eahzr9g1gbffi \
    172.16.210.180:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

--advertise-addr 172.16.210.180 为可选，如不指定，默认选择eth0网卡的IP。指定时，不支持指定浮动IP。

打开防火墙端口：

```
firewall-cmd --permanent --zone=public  --add-port=2377/tcp
firewall-cmd --permanent --zone=public  --add-port=7946/udp
firewall-cmd --permanent --zone=public  --add-port=4789/udp
firewall-cmd --list-all --permanent
firewall-cmd --reload
sudo systemctl restart firewalld

```

```必须启动firewall，否则安装shipyard时报错iptables。

firewall-cmd --zone=trusted --add-interface=docker0
firewall-cmd --zone=trusted --add-interface=enp2s0
（firewall-cmd --get-default-zone firewall-cmd --set-default-zone=trusted）
firewall-cmd --reload
systemctl restart docker
```


在其它节点执行以上join命令，成功后显示：

```
This node joined a swarm as a worker.
```
### 查看swarm节点
`docker node ls`
### 查看节点详细信息
`docker info`
### 使节点离开集群
`docker swarm leave`
## 在swarm集群上创建服务
###  创建overlay network
建立容器之前先创建一个overlay的网络，用来保证在不同主机上的容器网络互通的网络模式。
`docker network create -d overlay xxxx`
### 查询docker network 列表
`docker network ls`
### 容器的创建
在同一个名叫xxxx的overlay网络里新建三个相同的web容器副本，和一个 redis副本，并且每个web容器都提供统一的端口映射关系。
docker service create –replicas 3 –name frontend –network xxxx –publish 80:80/tcp frontend_image:latest
docker service create –name redis –network xxxx redis:latest
### 查看服务
docker service ls
### 查看服务运行在哪个节点上
`docker service ps <SERVICE-ID>`
### 服务扩展
docker service scale frontend=6
每台节点上都run一个相同的副本（global模式的service, 就是在swarm集群的每个节点创建一个容器副本）
docker service create –mode=global –name extend_frontend frontend_image:latest

swarm集群负载均衡service有两种方式—VIP和DNSRR:
VIP模式每个service会得到一个virtual IP地址作为服务请求的入口。基于virtual IP进行负载均衡.
DNSRR模式service利用DNS解析来进行负载均衡, 这种模式在旧的Docker Engine下不稳定，所以不推荐.
### docker 命令
删除当前没有被使用的一切项目
docker system prune
删除所有不使用的镜像并且不再请求确认（在不运行容器的系统中会删除所有镜像）
docker image prune --force --all
删除所有孤立的网络
docker network prune
删除qcow文件
停止docker服务后
rm -fr  ~/Library/Containers/com.docker.docker/Data/*
## shipyard 
curl -s https://shipyard-project.com/deploy | bash -s
alpine:latest
rethinkdb:latest
shipyard/shipyard:latest
shipyard/docker-proxy:latest
microbox/etcd:latest
swarm:latest
ehazlett/curl:latest
(服务器上安装http://172.16.210.180:81/，本机安装 http://127.0.0.1:8080/)




```
curl -s https://shipyard-project.com/deploy | bash -s -- -h
Shipyard Deploy uses the following environment variables:
  ACTION: this is the action to use (deploy, upgrade, node, remove)
  DISCOVERY: discovery system used by Swarm (only if using 'node' action)
  IMAGE: this overrides the default Shipyard image
  PREFIX: prefix for container names
  SHIPYARD_ARGS: these are passed to the Shipyard controller container as controller args
  TLS_CERT_PATH: path to certs to enable TLS for Shipyard
  PORT: specify the listen port for the controller (default: 8080)
  IP: specify the address at which the controller or node will be available (default: eth0 ip)
  PROXY_PORT: port to run docker proxy (default: 2375)
```


```curl -sSL https://shipyard-project.com/deploy | ACTION=remove  bash -s
deploy: Deploy a new Shipyard instance
upgrade: Upgrade an existing instance (note: you will need to pass the same environment variables as when you deployed to keep the same configuration)
node: Add current Docker engine as a new Swarm node in the cluster
remove: Completely removes Shipyard
```
添加swarm节点：

```curl -sSL https://shipyard-project.com/deploy | ACTION=node DISCOVERY=etcd://172.16.210.180:4001 bash -s
```

### 部署cockroachdb
https://github.com/cockroachdb/docs/blob/gh-pages/orchestrate-cockroachdb-with-docker-swarm-insecure.md
下载imags：

```
cockroachdb/cockroach:beta-20170323
cockroachdb/builder:20170228-215146
cockroachdb/postgres-test:20170308-1644
```

创建overlay network：
`docker network create --driver overlay cockroachdb`
启动CockroachDB集群：

```
docker service create \
--replicas 1 \
--name cockroachdb-1 \
--network cockroachdb \
--mount type=volume,source=cockroachdb-1,target=/cockroach/cockroach-data,volume-driver=local \
--stop-grace-period 60s \
--publish protocol=tcp,mode=ingress,published=8080,target=8080 \
--publish 26257:26257 \
beagledata.com:5000/cockroachdb/cockroach:beta-20170323 start \
--advertise-host=cockroachdb-1 \
--logtostderr \
--insecure 
```

```
关闭防火墙后可以访问任意服务器8080端口
其它形式：
--publish 8080:8080
--publishmode=host,published=8080,target=8080
```

docker service  ps  cockroachdb-1
本地存储卷位置在/var/lib/docker/volumes/cockroachdb-1
启动第二个节点：

```
docker service create \
--replicas 1 \
--name cockroachdb-2 \
--network cockroachdb \
--mount type=volume,source=cockroachdb-2,target=/cockroach/cockroach-data,volume-driver=local \
--stop-grace-period 60s \
beagledata.com:5000/cockroachdb/cockroach:beta-20170323 start \
--advertise-host=cockroachdb-2 \
--join=cockroachdb-1:26257 \
--logtostderr \
--insecure
```
启动第三个节点：

```
docker service create \
--replicas 1 \
--name cockroachdb-3 \
--network cockroachdb \
--mount type=volume,source=cockroachdb-3,target=/cockroach/cockroach-data,volume-driver=local \
--stop-grace-period 60s \
beagledata.com:5000/cockroachdb/cockroach:beta-20170323 start \
--advertise-host=cockroachdb-3 \
--join=cockroachdb-1:26257 \
--logtostderr \
--insecure
```

使用SQL client：
在cockroachdb-1管理节点运行的docker服务器上：
docker container  ls
docker container exec -it c1fbfc3a5985 ./cockroach sql
进入sql客户端：

```
> CREATE DATABASE bank;

> CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);

> INSERT INTO bank.accounts VALUES (1234, 10000.50);

> SELECT * FROM bank.accounts;
```
生成示例数据：
`./cockroach gen example-data | ./cockroach sql`

用户及权限：
root用户默认拥有全部权限。
```
./cockroach user set maxroach --password
cockroach sql -e 'GRANT ALL ON DATABASE * TO maxroach'
./cockroach user set root --password
```
```+------+---------+
|  id  | balance |
+------+---------+
| 1234 | 10000.5 |
+------+---------+
(1 row)
```

联通性测试

docker service create --publish 80:80 --network cockroachdb --name nginx beagledata.com:5000/nginx:1.8.1


停止集群：

```
docker service rm cockroachdb-1 cockroachdb-2 cockroachdb-3
docker volume rm cockroachdb-1 cockroachdb-2 cockroachdb-3
```
### 部署TiDB
创建overlay network：
`docker network create --driver overlay tidb`
启动pd1：
docker service create \
--replicas 1 \
--name pd1 \
--network tidb \
--mount type=volume,source=tidb1,target=/data,volume-driver=local \
--stop-grace-period 60s \
beagledata.com:5000/pingcap/pd:latest \
--name="pd1" \
  --data-dir="/data/pd1" \
  --client-urls="http://0.0.0.0:2379" \
  --advertise-client-urls="http://pd1:2379" \
  --peer-urls="http://0.0.0.0:2380" \
  --advertise-peer-urls="http://pd1:2380" \
  --initial-cluster="pd1=http://pd1:2380,pd2=http://pd2:2380,pd3=http://pd3:2380"

启动pd2：
docker service create \
--replicas 1 \
--name pd2 \
--network tidb \
--mount type=volume,source=tidb2,target=/data,volume-driver=local \
--stop-grace-period 60s \
beagledata.com:5000/pingcap/pd:latest \
--name="pd2" \
  --data-dir="/data/pd2" \
  --client-urls="http://0.0.0.0:2379" \
  --advertise-client-urls="http://pd2:2379" \
  --peer-urls="http://0.0.0.0:2380" \
  --advertise-peer-urls="http://pd2:2380" \
  --initial-cluster="pd1=http://pd1:2380,pd2=http://pd2:2380,pd3=http://pd3:2380"
 
 启动pd3：
docker service create \
--replicas 1 \
--name pd3 \
--network tidb \
--mount type=volume,source=tidb3,target=/data,volume-driver=local \
--stop-grace-period 60s \
beagledata.com:5000/pingcap/pd:latest \
--name="pd3" \
  --data-dir="/data/pd3" \
  --client-urls="http://0.0.0.0:2379" \
  --advertise-client-urls="http://pd3:2379" \
  --peer-urls="http://0.0.0.0:2380" \
  --advertise-peer-urls="http://pd3:2380" \
  --initial-cluster="pd1=http://pd1:2380,pd2=http://pd2:2380,pd3=http://pd3:2380"
  
启动 TiKV1：
docker service create \
--replicas 1 \
--name tikv1 \
--network tidb \
--mount type=volume,source=tikv1,target=/data,volume-driver=local \
--stop-grace-period 60s \
beagledata.com:5000/pingcap/tikv:latest \
  --addr="0.0.0.0:20160" \
  --advertise-addr="tikv1:20160" \
  --store="/data/tikv1" \
  --pd="pd1:2379,pd2:2379,pd3:2379"

启动 TiKV2：
docker service create \
--replicas 1 \
--name tikv2 \
--network tidb \
--mount type=volume,source=tikv2,target=/data,volume-driver=local \
--stop-grace-period 60s \
beagledata.com:5000/pingcap/tikv:latest \
  --addr="0.0.0.0:20160" \
  --advertise-addr="tikv2:20160" \
  --store="/data/tikv2" \
  --pd="pd1:2379,pd2:2379,pd3:2379"

启动 TiKV3：
docker service create \
--replicas 1 \
--name tikv3 \
--network tidb \
--mount type=volume,source=tikv3,target=/data,volume-driver=local \
--stop-grace-period 60s \
beagledata.com:5000/pingcap/tikv:latest \
  --addr="0.0.0.0:20160" \
  --advertise-addr="tikv3:20160" \
  --store="/data/tikv3" \
  --pd="pd1:2379,pd2:2379,pd3:2379"

启动 TiDB
docker service create \
--replicas 1 \
--name tidb \
--network tidb \
--stop-grace-period 60s \
--publish 4000:4000 \
--publish 10080:10080 \
beagledata.com:5000/pingcap/tidb:latest \
--store=tikv \
--path="pd1:2379,pd2:2379,pd3:2379"

联通性测试：
使用sql工具4000端口连接正常。
curl http://172.16.210.183:10080/status

## 监控
grafana/grafana
docker run -d -p 3000:3000 \
    -v /var/lib/grafana:/var/lib/grafana \
    -e "GF_SECURITY_ADMIN_PASSWORD=secret" \
    grafana/grafana
    
docker pull prom/pushgateway
docker run -d -p 9091:9091 prom/pushgateway

docker run --name prometheus -d -p 127.0.0.1:9090:9090 quay.io/prometheus/prometheus


