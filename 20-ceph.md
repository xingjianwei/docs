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
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
sudo firewall-cmd --zone=public --add-port=5000/tcp --permanent
sudo firewall-cmd --zone=public --add-port=27017/tcp --permanent
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

CEPH对象网关RGW:

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


## web ui

### 方案对比

[ceph 原生的calamari server](https://github.com/ceph/calamari)

[calamari client-romana](https://github.com/ceph/romana)

选择更轻量级的[inkscope](https://github.com/inkscope/inkscope)

### inkscope安装
[安装文档](https://github.com/inkscope/inkscope/wiki/Inkscope-installation-(V1.1))

* sysprobe：所有相关节点都需要安装。
* inkscope-admviz : 包含inkscope的web控制台文件，含接口和界面，仅需要安装一个，该节点（管理节点）上同时需要按安装flask和mongodb
* inkscope-cephrestapi: 用于安装启动 ceph rest api 的脚本，仅需要安装在提供api接口的节点上，即mon节点。
* inkscope-cephprobe: 用于安装启动 cephprobe 的脚本(整个集群只需一个)，安装在mon节点，脚本主要实现：获取Ceph集群的一些信息，并使用端口（5000）提供服务，将数据存入mongodb数据库中。

下载安装包创建本地仓库：
https://github.com/inkscope/inkscope-packaging

```
su - cephuser
mkdir inkscope_repo
```

拷贝rpm文件到此目录。

```
sudo chown -R root.root inkscope_repo
sudo yum install createrepo
sudo createrepo /home/cephuser/inkscope_repo
sudo chmod -R o-w+r /home/cephuser/inkscope_repo
```
`sudo vim /etc/yum.repos.d/inkscope.repo`
```
[inkScope local]
name=My inkScope Repo
baseurl=file:///home/cephuser/inkscope_repo
enabled=1
gpgcheck=0
```
`sudo yum makecache`

on admin server:
```
yum install inkscope-cephrestapi
yum install inkscope-cephprobe
yum install inkscope-sysprobe
yum install inkscope-admviz
yum install mongodb mongodb-server python-pip python-devel
pip install Flask-Login==0.3.2
```
edit /etc/mongod.conf
```
bind_ip = 0.0.0.0
port = 27017
```
systemctl  enable mongod
systemctl start mongod
`radosgw-admin user create --uid=inkscope --display-name="inkscope" --access-key=inkscope --secret=inkscope --access=full --caps="metadata=*;users=*;buckets=*"`

edit  /opt/inkscope/etc/inkscope.conf
```
cat /opt/inkscope/etc/inkscope.conf
{
    "ceph_conf": "/etc/ceph/ceph.conf",
    "ceph_rest_api": "172.16.210.121:5000",
    "ceph_rest_api_subfolder": "",
    "mongodb_host" : "ceph-admin",
    "mongodb_set" : "mongodb0:27017,mongodb1:27017,mongodb2:27017",
    "mongodb_replicaSet" : "replmongo0",
    "mongodb_read_preference" : "ReadPreference.SECONDARY_PREFERRED",
    "mongodb_port" : 27017,
    "mongodb_user":"ceph",
    "mongodb_passwd":"monpassword",
    "is_mongo_authenticate" : 0,
    "is_mongo_replicat" : 0,
    "influxdb_endpoint":"http://172.16.210.121:8086",
    "cluster": "ceph",
    "platform": "x86_64",
    "status_refresh": 3,
    "osd_dump_refresh": 3,
    "pg_dump_refresh": 60,
    "crushmap_refresh": 60,
    "df_refresh": 60,
    "cluster_window": 1200,
    "osd_window": 1200,
    "pool_window": 1200,
    "mem_refresh": 60,
    "swap_refresh": 600,
    "disk_refresh": 60,
    "partition_refresh": 60,
    "cpu_refresh": 30,
    "net_refresh": 30,
    "mem_window": 1200,
    "swap_window": 3600,
    "disk_window": 1200,
    "partition_window": 1200,
    "cpu_window": 1200,
    "net_window": 1200,
    "inkscope_root" : "172.16.210.121:8080",
    "radosgw_url": "http://172.16.210.121:7480",
    "radosgw_admin": "admin",
    "radosgw_key": "inkscopexxxx",
    "radosgw_secret": "inkscopexxxx"
}
```
edit /etc/httpd/conf/httpd.conf
```
Listen 8080
NameVirtualHost *.8080
```
edit  /etc/httpd/conf.d/inkScope.conf
```
Directory "/var/www/inkscope
ProxyPass /ceph-rest-api/ http://172.16.210.121:5000/api/v0.1/
systemctl restart httpd
```
[参考文档](http://inkscope.blogspot.hk/2015/03/inkscope-and-ceph-rest-api.html)
参考配置文件：

```
<VirtualHost *:8080>
ServerName localhost
ServerAdmin webmaster@localhost

DocumentRoot /var/www/inkscope
<Directory "/var/www/inkscope">
    Options All
    AllowOverride All
</Directory>

ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
<Directory "/usr/lib/cgi-bin">
    AllowOverride None
    Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
    Order allow,deny
    Allow from all
</Directory>

WSGIScriptAlias /inkscopeCtrl /var/www/inkscope/inkscopeCtrl/inkscopeCtrl.wsgi
<Directory "/var/www/inkscope/inkscopeCtrl">
    Order allow,deny
    Allow from all
</Directory>

WSGIScriptAlias /ceph_rest_api /var/www/inkscope/inkscopeCtrl/ceph-rest-api.wsgi
<Directory "/var/www/inkscope/inkscopeCtrl">
     Require all granted
</Directory>

# Possible values include: debug, info, notice, warn, error, crit,
# alert, emerg.
LogLevel warn

ProxyRequests Off  # we want  a "Reverse proxy"

# For a ceph_rest_api in wsgi mode
ProxyPass /ceph-rest-api/ http://debianhost.engba.veritas.com:8080/ceph_rest_api/api/v0.1/

# For a standalone ceph_rest_api, uncomment the next line and comment the previous one
# ProxyPass /ceph-rest-api/ http://<ceph_rest_api_host>:5000/api/v0.1/

ErrorLog /var/log/inkscope/webserver_error.log
CustomLog /var/log/inkscope/webserver_access.log common
```

打开浏览器访问：http://172.16.210.121:8080，可看到Inscope首页信息了，不过此时没有Ceph集群的信息。
```
On the first launch, two users are created :
- one with the admin role (User 'admin' with password 'admin')
- a second with supervizor role (User 'guest', no password)
```
```
mongo
show dbs
use inkscope
show tables
db.inkscope_users.find()
db.inkscope_users.update({password:"792fa0e40153f25192715026abd1e445"},{"$set":{name:"admin1"}})
db.inkscope_users.update({password:"792fa0e40153f25192715026abd1e445"},{"$set":{roles:[ "admin" ]}})
db.inkscope_users.update({password:"792fa0e40153f25192715026abd1e445"},{"$set":{password:"11bdb07177c71765ba91f11a0e8b5979"}})
```
查看文件/var/www/inkscope/inkscopeCtrl/inkscopeCtrlcore.py中的加密方法：
```
import hashlib
password_r = 'admin'
secret_key = "Mon Nov 30 17:20:29 2015"
salted_password = password_r + secret_key
print hashlib.md5(salted_password).hexdigest()
```

on each server:
```
yum install inkscope-sysprobe psutils sysstat
pip install psutil
pip install  pymongo
edit  /opt/inkscope/etc/inkscope.conf
/etc/init.d/sysprobe start
```

在CephProbe所在节点，启动ceph-rest-api服务 
/etc/init.d/ceph-rest-api start  2>&1 > /var/log/ceph/ceph_rest_api

The monitoring graphs need the installation of influxdb and collectd.
collectd：定时收集系统性能的统计信息并提供了一个存储机制的守护进程。主要用它来收集cpu，内存，带宽的数据。
influxdb：分布式的，time series的db

```
yum install collectd
cat <<EOF | sudo tee /etc/yum.repos.d/influxdb.repo
[influxdb]
name = InfluxDB Repository - RHEL \$releasever
baseurl = https://repos.influxdata.com/rhel/\$releasever/\$basearch/stable
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdb.key
EOF
yum install influxdb
sudo firewall-cmd --zone=public --add-port=8086/tcp --permanent
sudo firewall-cmd --zone=public --add-port=8088/tcp --permanent
sudo firewall-cmd --zone=public --add-port=8083/tcp --permanent
sudo firewall-cmd --zone=public --add-port=25826/udp --permanent
sudo firewall-cmd --reload
```
在centos系统下，collectd ceph plugins需要[getsigchld.py](https://github.com/collectd/collectd/tree/master/contrib/python)。
pip install collectd
检查/etc/collectd.d/ceph_plugins.conf文件中开启`Import "getsigchld"`，并将getsigchld.py文件拷贝到/usr/lib64/python2.7/site-packages/ 。
检查 Line 54 in /usr/lib/collectd/plugins/ceph/ceph_latency_plugin.py，将默认的data改为cephfs_data。

```
vi /etc/influxdb/influxdb.conf
reporting-disabled = true

```

```
vi /etc/influxdb/influxdb.conf
[[collectd]]
  enabled = true
  bind-address = "Server "172.16.210.121" "25826""
  database = "collectd"
  typesdb = "/usr/share/collectd/types.db"
```
使用influx命令进入：
```
SHOW DATABASES
CREATE DATABASE collectd
CREATE USER "root" WITH PASSWORD 'root' WITH ALL PRIVILEGES
use collectd
#显示该数据库中所有的表 
show measurements 
select * from cpu_value order by time desc 
```

```
vi /etc/collectd.conf
mkdir -p /usr/lib/collectd/plugins/ceph
cp -p  collectd-ceph-master/plugins/* /usr/lib/collectd/plugins/ceph
```
add a configuration file named ceph_plugins.conf in /etc/collectd/collectd.conf.d/ with this content:
```
vi /etc/collectd.d/ceph_plugins.conf
<LoadPlugin "python">
    Globals true
</LoadPlugin>

<Plugin "python">
    ModulePath "/usr/lib/collectd/plugins/ceph"
    #Import "getsigchld"  #uncomment for centos
    Import "ceph_latency_plugin"
   
    <Module "ceph_latency_plugin">
        Verbose "True"
        Cluster "ceph"
        Interval "60"
    </Module>

    Import "ceph_monitor_plugin"

    <Module "ceph_monitor_plugin">
        Verbose "True"
        Cluster "ceph"
        Interval "60"
    </Module>
    
    Import "ceph_osd_plugin"

    <Module "ceph_osd_plugin">
        Verbose "True"
        Cluster "ceph"
        Interval "60"
    </Module>

    Import "ceph_pg_plugin"

    <Module "ceph_pg_plugin">
        Verbose "True"
        Cluster "ceph"
        Interval "60"
    </Module>

    Import "ceph_pool_plugin"

    <Module "ceph_pool_plugin">
        Verbose "True"
        Cluster "ceph"
        Interval "60"
        TestPool "test"
    </Module>
</Plugin>
```

`systemctl start collectd.service`

测试端口服务是否正常：
`nc -l -u -p 25826`



## 服务器停止
poweroff

## 服务器重启
cep服务会自动启动修复pg。

在node节点：
`/etc/init.d/sysprobe start`
在mon节点：
```
/etc/init.d/cephrestapi start
/etc/init.d/cephprobe start
/etc/init.d/sysprobe start
```

## 日常运维
```
ceph osd stat
osdmap e400: 36 osds: 34 up, 34 in
```
```
ceph osd tree
ceph -w
```
经过一段时间后依旧有一些PGS无法+recovering.
如果一个硬盘故障导致osd节点出现如下的down状态，且一直无法恢复（ reweight列等于0，表示osd已经out此集群）。

一般情况下重启服务或者重启节点即可。

```
systemctl stop ceph-osd@9
umount /var/lib/ceph/osd/ceph-9  卸载挂载分区
ceph osd rm 9  在集群中删除一个osd硬盘
ceph osd crush rm osd.9 在集群中删除一个osd 硬盘 crush map
ceph auth del osd.9 
```
osd9所属的节点为ceph-node2，磁盘为sdc，`mkfs.xfs  -f /dev/sdc`。
```
cd beagle-cluster    否则会找不到keyring。
ceph-deploy  disk zap ceph-node2:/dev/sdc
ceph-deploy osd prepare ceph-node2:sdc:/dev/sdb
ceph-deploy osd activate ceph-node2:sdc1:/dev/sdb1
```

摘掉osd的脚本如下
```
osd_id=`ceph osd tree | grep down | grep osd | awk '{print $3}' | awk -F . '{print $2}'`
ceph osd rm ${osd_id}
ceph osd crush rm osd.${osd_id}
ceph auth del osd.${osd_id}
umount /var/lib/ceph/osd/ceph-${osd_id}
```