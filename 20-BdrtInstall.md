# BDRT 编译安装
## MACOS环境设置
git clone git@192.168.100.2:bdtq/beagledata-BDTQ.git
git branch --list
git checkout develop
export JAVA_HOME=$(/usr/libexec/java_home -v 1.8)
java -version
## 源代码编译

`ant`

## solr安装包下载地址
`http://archive.apache.org/dist/lucene/solr/5.5.2/`

## 准备hadoop on docker

```
docker pull kiwenlau/hadoop:1.0

http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html

wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" download.oracle.com/otn-pub/java/jdk/8u111-b14/jdk-8u111-linux-x64.rpm

wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u121-b13/e9e7ea248e2c4826b92b3f075a80e441/jdk-8u121-linux-x64.rpm
修改Dockerfile文件，下载CDH5.8.3安装包。
```
制作镜像文件：

```
cd bdrt/hadoop
docker build -t beagle/hadoop:1.0 .
```
运行镜像：

```
sudo docker network create --driver=bridge hadoop
cd bdrt/hadoop
./start-container.sh
```
进入容器环境后，使用`./start-hadoop.sh`启动hadoop。

## 准备hbase & solr on docker


由于hbase DNSQ问题，如果依靠docker swarm本身的hostname解析方式，在hbase regionserver 会在docker network中自动找到相同的主机，导致服务报错。

所以只能使用hosts文件的解析方式，在容器全部启动后，依据`/usr/local/hbase/conf/regionservers`文件中的主机名，自动建立hosts文件。
制作镜像文件：

```
cd bdrt/hbase
docker build -t beagle/hbase:1.0 .
```
运行镜像：

```
sudo docker network create --driver=bridge hadoop
cd bdrt/hbase
./start-container.sh
```
进入容器环境后，

```
./make-allhost.sh  设置所有节点的hosts文件
./start-hadoop.sh
./start-hbase.sh
$SOLR_HOME/bin/solrinit.sh   初始化bdrt索引
```
测试zk启动是否成功：

```
$ZOOKEEPER_HOME/bin/zkCli.sh -server 127.0.0.1:2181
ls / 使用 ls 命令来查看当前 ZooKeeper 中所包含的内容
ls2 / 查看当前节点数据并能看到更新次数等数据
create /zk "test" 创建一个新的 znode节点“ zk ”以及与它关联的字符串
get /zk 确认 znode 是否包含我们所创建的字符串
set /zk "zkbak" 对 zk 所关联的字符串进行设置
delete /zk 将刚才创建的 znode 删除
quit
help
```
测试hbase启动是否成功：

```
$HBASE_HOME/bin/hbase shell
hbase(main)> list   查看有哪些表
hbase(main)> create 't1',{NAME => 'f1', VERSIONS => 2},{NAME => 'f2', VERSIONS => 2}    创建表
hbase(main)> disable 't1'   删除表
hbase(main)> drop 't1'
hbase(main)> describe 't1'  查看表的结构
hbase(main)> put 't1','rowkey001','f1:col1','value01'   添加数据
hbase(main)> get 't1','rowkey001'   查询某行记录

```
测试solr启动是否成功：

```
$SOLR_HOME/bin/solr create_collection -c my_test -shards 3 -replicationFactor 3
```


## 制作BDRT on Docker

制作镜像文件：

```
cd bdrt/bdrt
docker build --no-cache -t beagle/bdrt:1.0 .
```
运行镜像：

```
docker network create --driver=bridge hadoop
cd bdrt/bdrt
./start-container.sh
```
进入容器环境后，

```
./make-allhost.sh  设置所有节点的hosts文件
./start-hadoop.sh
./start-hbase.sh  已经初始化完成bdrt中所需的solrInit.sh
$BDRT_HOME/bin/bdrt.sh start  启动bdrt server
```

```
进入BDRT控制台：
$BDRT_HOME/bin/bdrt-sql.sh -h hadoop-master

bdrt>
bdrt> :help
bdrt> :plugin list
bdrt> show databases;
bdrt> create database my_test;

```

此时进入 [hbase管理界面](http://127.0.0.1:60010/) ，会监控到hbase表MY_TEST的建立过程。

```
bdrt> use my_test;
bdrt> show tables;
bdrt> CREATE TABLE my_test.accounts (id int PRIMARY KEY, balance double);
bdrt> desc  accounts;
bdrt> select * from accounts;
bdrt> INSERT INTO accounts VALUES (1234, 10000);
insert into accounts values (2222, 10001);
INSERT INTO accounts VALUES (1234, 20000); 不指定列插入时，默认列顺序和建表时不同
select * from accounts where balance=1234;
insert into accounts (id,balance) values (1001, 2000)
insert into accounts (id,balance) values (1002, 2000),(1003, 2000);

```




