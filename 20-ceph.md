# ceph

[参考文档-基于docker部署ceph以及修改docker image](http://www.tuicool.com/articles/ZR7Bre7)

[docker-ceph](https://github.com/ceph/ceph-docker)

[ceph 存储安装部署](http://www.cnblogs.com/pycode/p/6494853.html)

## 安装提示

元数据服务器和监视器必须可以尽快地提供它们的数据，所以他们应该有足够的内存，至少每进程 1GB 。 OSD 的日常运行不需要那么多内存（如每进程 500MB ）差不多了；然而在恢复期间它们占用内存比较大（如每进程每 TB 数据需要约 1GB 内存）。通常内存越多越好。

Ceph 最佳实践指示，你应该分别在单独的硬盘运行操作系统、 OSD 数据和 OSD 日志。

建议每台机器最少两个千兆网卡，现在大多数机械硬盘都能达到大概 100MB/s 的吞吐量，网卡应该能处理所有 OSD 硬盘总吞吐量，所以推荐最少两个千兆网卡，分别用于公网（前端）和集群网络（后端）。集群网络（最好别连接到国际互联网）用于处理由数据复制产生的额外负载，而且可防止拒绝服务攻击，拒绝服务攻击会干扰数据归置组，使之在 OSD 数据复制时不能回到 active + clean 状态。请考虑部署万兆网卡。通过 1Gbps 网络复制 1TB 数据耗时 3 小时，而 3TB （典型配置）需要 9 小时，相比之下，如果使用 10Gbps 复制时间可分别缩减到 20 分钟和 1 小时。在一个 PB 级集群中， OSD 磁盘失败是常态，而非异常；在性价比合理的的前提下，系统管理员想让 PG 尽快从 degraded （降级）状态恢复到 active + clean 状态。

## 安装Jewel LTS过程

### 安装准备
172.16.210.121-124

hostnamectl set-hostname  ceph-admin
ceph-node1-3

http://mirrors.aliyun.com/

```
yum install -y wget curl vim
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum makecache
```

ceph添加阿里源：

`vim /etc/yum.repos.d/ceph.repo`

添加

```
[ceph]
name=ceph
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/x86_64/
gpgcheck=0
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/noarch/
gpgcheck=0
进行yum的makecache
yum makecache
```

### 安装ceph-deploy

in ceph-admin & node:
sudo yum -y update && sudo yum install -y ceph-deploy

### 节点配置