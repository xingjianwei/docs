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

```
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum makecache
```

确认yum目录中有epel源。



`vim /etc/yum.repos.d/ceph.repo`

添加

```
[ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-jewel/el7/noarch
priority=1
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
```
ceph添加阿里源：

```
[ceph]
name=ceph
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/x86_64/
priority=1
gpgcheck=0
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/noarch/
priority=1
gpgcheck=0
```

```
yum clean all
yum makecache
```

### 安装ceph-deploy

in ceph-admin & node:
sudo yum -y update && sudo yum install -y ceph-deploy

### 节点配置

时间更新同步:

```
yum install -y ntp ntpdate ntp-doc
systemctl start ntpd
systemctl enable ntpd
ntpdate asia.pool.ntp.org
```

设置时区：

```
timedatectl  status
timedatectl list-timezones

timedatectl set-timezone "Asia/Shanghai"

```

添加hosts绑定:

```
sudo cat << EOF >> /etc/hosts
172.16.210.121    ceph-admin
172.16.210.122    ceph-node1
172.16.210.123    ceph-node2
172.16.210.124    ceph-node3
EOF
```

设置防火墙:

```
sudo systemctl start firewalld.service
sudo systemctl enable firewalld.service

sudo firewall-cmd --list-all --permanent
sudo firewall-cmd --zone=public --add-service=ceph-mon --permanent
sudo firewall-cmd --zone=public --add-service=ceph --permanent
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
sudo firewall-cmd --reload
```

禁用selinux:

```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
```

Cephuser 账号:

```
groupadd -g 2000 cephgroup
useradd -c "ceph user" -g cephgroup -u 2000 cephuser
echo "cephuser:xxxx" | chpasswd
```

sudo和tty:

```
cat << EOF >/etc/sudoers.d/cephuser
cephuser ALL = (root) NOPASSWD:ALL
Defaults:cephuser !requiretty
EOF
chmod 0440 /etc/sudoers.d/cephuser
```

SSH免密登录：

in ceph-admin use cephuser:
su - cephuser
ssh-keygen
ssh-copy-id cephuser@ceph-node1

```
cat << EOF > ~/.ssh/config
Host node1
   Hostname ceph-node1
   User cephuser
Host node2
   Hostname ceph-node2
   User cephuser
Host node3
   Hostname ceph-node3
   User cephuser
EOF
```

`reboot`

### 集群安装

cd /home/cephuser
mkdir beagle-cluster
cd beagle-cluster/

想从头再来，可以用下列命令清除配置：

```
ceph-deploy purgedata {ceph-node} [{ceph-node}]
ceph-deploy forgetkeys
```
用下列命令可以连 Ceph 安装包一起清除：

```
ceph-deploy purge {ceph-node} [{ceph-node}]
yum-complete-transaction --cleanup-only
yum history redo last
```

要用 ceph-deploy 创建集群，用 new 命令、并指定几个主机作为初始监视器法定人数。

`ceph-deploy new  ceph-admin`

把 Ceph 配置文件里的默认副本数从 3 改成 2 ，这样只有两个 OSD 也可以达到 active + clean 状态。把下面这行加入 [global] 段：

`osd pool default size = 2`

网络配置:

```
public network = 172.16.210.0/24

cluster network = 192.168.1.0/24
```

安装 Ceph:

```
ceph-deploy install  --repo-url=http://mirrors.aliyun.com/ceph/rpm-jewel/el7 --gpg-url=http://mirrors.aliyun.com/ceph/keys/release.asc  ceph-admin ceph-node1 ceph-node2 ceph-node3
```

单独安装包：
```
sudo yum install -y snappy leveldb gdisk python-argparse gperftools-libs
sudo yum install -y ceph
```

### 初始化磁盘

`ceph-deploy disk list ceph-admin ceph-node1 ceph-node2 ceph-node3`

Add the initial monitor(s) and gather the keys:

`ceph-deploy mon create-initial`

umount磁盘并修改fstab：

`chars='bcdefghijkl';for (( i=0; i<=10; i++ )) ; do umount /dev/sd${chars:$i:1}1;done`

如果磁盘刚umount，需要reboot服务器。

`chars='bcdefghijkl';for (( i=0; i<=10; i++ )) ; do ceph-deploy disk zap ceph-node1:sd${chars:$i:1};done`

创建osd：使用sdb作为cdef的journal-disk，使用sdg作为hijkl的journal-disk

```
ceph-deploy osd prepare {node-name}:{data-disk}[:{journal-disk}]
ceph-deploy osd activate {node-name}:{data-disk-partition}[:{journal-disk-partition}]

chars='cdef';for (( i=0; i<4; i++ )) ;do ceph-deploy osd prepare ceph-node1:sd${chars:$i:1}:/dev/sdb ;done
chars='hijkl';for (( i=0; i<5; i++ )) ;do ceph-deploy osd prepare ceph-node1:sd${chars:$i:1}:/dev/sdg ;done

chars='cdef';for (( i=0; i<4; i++ )) ;do ceph-deploy osd prepare ceph-node2:sd${chars:$i:1}:/dev/sdb ;done
chars='hijkl';for (( i=0; i<5; i++ )) ;do ceph-deploy osd prepare ceph-node2:sd${chars:$i:1}:/dev/sdg ;done

chars='cdef';for (( i=0; i<4; i++ )) ;do ceph-deploy osd prepare ceph-node3:sd${chars:$i:1}:/dev/sdb ;done
chars='hijkl';for (( i=0; i<5; i++ )) ;do ceph-deploy osd prepare ceph-node3:sd${chars:$i:1}:/dev/sdg ;done

chars='cdef';for (( i=0; i<4; i++ )) ;do ceph-deploy --overwrite-conf osd prepare  ceph-admin:sd${chars:$i:1}:/dev/sdb ;done
chars='hijkl';for (( i=0; i<5; i++ )) ;do ceph-deploy --overwrite-conf osd prepare  ceph-admin:sd${chars:$i:1}:/dev/sdg ;done

```

```
chars='cdef';for (( i=0; i<4; i++ )) ;do ceph-deploy osd activate ceph-node1:sd${chars:$i:1}1:/dev/sdb$[i+1] ;done
chars='hijkl';for (( i=0; i<5; i++ )) ;do ceph-deploy osd activate ceph-node1:sd${chars:$i:1}1:/dev/sdg$[i+1] ;done

chars='cdef';for (( i=0; i<4; i++ )) ;do ceph-deploy osd activate ceph-node2:sd${chars:$i:1}1:/dev/sdb$[i+1] ;done
chars='hijkl';for (( i=0; i<5; i++ )) ;do ceph-deploy osd activate ceph-node2:sd${chars:$i:1}1:/dev/sdg$[i+1] ;done

chars='cdef';for (( i=0; i<4; i++ )) ;do ceph-deploy osd activate ceph-node3:sd${chars:$i:1}1:/dev/sdb$[i+1] ;done
chars='hijkl';for (( i=0; i<5; i++ )) ;do ceph-deploy osd activate ceph-node3:sd${chars:$i:1}1:/dev/sdg$[i+1] ;done

chars='cdef';for (( i=0; i<4; i++ )) ;do ceph-deploy osd activate ceph-admin:sd${chars:$i:1}1:/dev/sdb$[i+1] ;done
chars='hijkl';for (( i=0; i<5; i++ )) ;do ceph-deploy osd activate ceph-admin:sd${chars:$i:1}1:/dev/sdg$[i+1] ;done
```
把配置文件和 admin 密钥拷贝到管理节点和 Ceph 节点

```
ceph-deploy admin {admin-node} {ceph-node}
ceph-deploy admin ceph-admin ceph-node1 ceph-node2 ceph-node3
sudo chmod 744 /etc/ceph/ceph.client.admin.keyring
ceph health
```

STARTING ALL DAEMONS:

```
sudo systemctl start ceph.target       # start all daemons
sudo systemctl status ceph-osd@12      # check status of osd.12
```

STOPPING ALL DAEMONS:

`sudo systemctl stop ceph\*.service ceph\*.target`


有告警信息如下：

`HEALTH_WARN too few PGs per OSD (3 < min 30)`

```
ceph osd pool stats
ceph osd pool set rbd pg_num 1000
ceph osd pool set rbd pgp_num 1000
```

rados lspools 查看存储池
ceph df 检查集群使用情况
ceph mon stat 检查monitor状态
ceph osd stat 检查osd状态
ceph pg stat 检查pg配置组状态
ceph osd lspools 列出存储池

ceph建好后默认有个rbd池，可以考虑删除:

`ceph osd pool delete rbd rbd --yes-i-really-really-mean-it`

MDS:

`ceph-deploy mds create   ceph-admin`

RGW:

By default, the RGW instance will listen on port 7480

`ceph-deploy rgw create ceph-admin`


### Client安装

在管理节点上：

```
ceph-deploy install ceph-client
ceph-deploy admin ceph-client
```
#### CEPH块设备

在 ceph-client 节点上创建一个块设备 image 。
Ceph 块设备也叫 RBD 或 RADOS 块设备。

`rbd create foo --size 4096 [-m {mon-IP}] [-k /path/to/ceph.client.admin.keyring]`

在 ceph-client 节点上，把 image 映射为块设备。

`sudo rbd map foo --name client.admin [-m {mon-IP}] [-k /path/to/ceph.client.admin.keyring]`

在 ceph-client 节点上，创建文件系统后就可以使用块设备了。

`sudo mkfs.ext4 -m0 /dev/rbd/rbd/foo`

此命令可能耗时较长。

在 ceph-client 节点上挂载此文件系统。

```
sudo mkdir /mnt/ceph-block-device
sudo mount /dev/rbd/rbd/foo /mnt/ceph-block-device
cd /mnt/ceph-block-device
```

#### CEPH 文件系统

`yum install ceph-fuse`

创建存储池和文件系统:

```
ceph osd pool create cephfs_data 1000
ceph osd pool create cephfs_metadata 1000
ceph fs new beaglefs cephfs_metadata cephfs_data
```

创建密钥文件:

Ceph 存储集群默认启用认证，你应该有个包含密钥的配置文件（但不是密钥环本身）

* 在密钥环文件中找到与某用户对应的密钥，例如：
`ceph auth list`
`cat ceph.client.admin.keyring`

* 找到用于挂载 Ceph 文件系统的用户，复制其密钥。大概看起来如下所示：
```
[client.admin]
   key = AQCj2YpRiAe6CxAA7/ETt7Hcl9IyxyYciVs47w==
 ```

* 把密钥粘帖进去，大概像这样：

`AQCj2YpRiAe6CxAA7/ETt7Hcl9IyxyYciVs47w==`
* 保存文件，并把其用户名 name 作为一个属性（如 admin.secret ）。

确保此文件对用户有合适的权限，但对其他用户不可见。
```
sudo mkdir /mnt/mycephfs
sudo mount -t ceph 192.168.0.1:6789:/ /mnt/mycephfs -o name=admin,secret=xxxx
```

```
sudo ceph-fuse -k ./ceph.client.admin.keyring -m 192.168.0.1:6789 ~/mycephfs
或者将配置文件和key拷贝到/etc/ceph目录下。
```