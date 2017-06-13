# kubernetes安装

[和我一步步部署 kubernetes 集群](https://github.com/opsnull/follow-me-install-kubernetes-cluster)

[以上文档的本地版本](file:///Users/xingjianwei/github/xingjianwei/follow-me-install-kubernetes-cluster/README.md)


kubelet参数pod-infra-container-image:
在minion端的kubelet配置文件中发现这个参数，默认值是这样的
`KUBELET_POD_INFRA_CONTAINER=--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest`
这个是一个基础容器，每一个Pod启动的时候都会启动一个这样的容器。如果你的本地没有这个镜像，kubelet会连接外网把这个镜像下载下来。最开始的时候是在Google的registry上，因此国内因为GFW都下载不了导致Pod运行不起来。现在每个版本的Kubernetes都把这个镜像打包，你可以提前传到自己的registry上，然后再用这个参数指定。



## 安装准备

172.16.210.101-103

hostnamectl set-hostname  kuber-admin
kuber-node1-2

http://mirrors.aliyun.com/

```
sudo yum install -y wget curl vim
sudo wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
sudo yum makecache
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo yum makecache
sudo yum update
reboot
```

添加hosts绑定:

```
sudo cat << EOF >> /etc/hosts
172.16.210.101    kuber-admin
172.16.210.102    kuber-node1
172.16.210.103    kuber-node2
EOF
```

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

禁用selinux:

```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
```
没有关闭防火墙。最后安装docker步骤会永久关闭防火墙。


## 安装步骤

`mkdir -p /root/local/bin`

[01-组件版本和集群环境](file:///Users/xingjianwei/github/xingjianwei/follow-me-install-kubernetes-cluster/01-组件版本和集群环境.md)

[02-创建CA证书和秘钥](/Users/xingjianwei/github/xingjianwei/follow-me-install-kubernetes-cluster/02-创建CA证书和秘钥.md)
将生成的 CA 证书、秘钥文件、配置文件拷贝到所有机器的 /etc/kubernetes/ssl 目录下
```
scp ca* root@kuber-node1:/etc/kubernetes/ssl/
scp ca* root@kuber-node2:/etc/kubernetes/ssl/
```
检验证书的工作需要等到生成master节点的kubernetes.pem后进行。

[03-部署高可用Etcd集群](file:///Users/xingjianwei/github/xingjianwei/follow-me-install-kubernetes-cluster/03-部署高可用Etcd集群.md)
```
scp /root/local/bin/etcd* root@kuber-node1:/root/local/bin/
scp /root/local/bin/etcd* root@kuber-node2:/root/local/bin/
```
环境变量中`NODE_IP`和`NODE_NAME`每个服务器不同，创建证书的过程需要在每台节点上执行。

在admin节点生成node1节点的 etcd 证书签名请求：
```
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.16.210.102"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

```
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
ssh root@kuber-node1  "mkdir -p /etc/etcd/ssl"
scp etcd*.pem root@kuber-node1:/etc/etcd/ssl/
rm etcd*.pem etcd.csr  etcd-csr.json

```
在admin节点生成node2节点的 etcd 证书签名请求：
```
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.16.210.103"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

```
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
ssh root@kuber-node2  "mkdir -p /etc/etcd/ssl"
scp etcd*.pem root@kuber-node2:/etc/etcd/ssl/
rm etcd*.pem  etcd.csr  etcd-csr.json

```

```
sudo firewall-cmd --zone=public --add-port=2380/tcp --permanent
sudo firewall-cmd --zone=public --add-port=2379/tcp --permanent
sudo firewall-cmd --reload
```

(04-部署Kubectl命令行工具)[file:///Users/xingjianwei/github/xingjianwei/follow-me-install-kubernetes-cluster/04-部署Kubectl命令行工具.md]

O 指定该证书的 Group 为 system:masters，kubelet 使用该证书访问kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限； 

(05-部署Flannel网络)[file:///Users/xingjianwei/github/xingjianwei/follow-me-install-kubernetes-cluster/05-部署Flannel网络.md]

在admin节点生成node1节点的 flanneld 证书签名请求：
```
cat > flanneld-csr.json <<EOF
{
  "CN": "flanneld",
  "hosts": [
    "127.0.0.1",
    "172.16.210.102"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
```
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
ssh  root@kuber-node1 "mkdir -p /etc/flanneld/ssl"
scp flanneld*.pem root@kuber-node1:/etc/flanneld/ssl/
rm flanneld*.pem  flanneld.csr  flanneld-csr.json
scp flannel/{flanneld,mk-docker-opts.sh} root@kuber-node1:/root/local/bin/
```

在admin节点生成node1节点的 flanneld 证书签名请求：

```
cat > flanneld-csr.json <<EOF
{
  "CN": "flanneld",
  "hosts": [
    "127.0.0.1",
    "172.16.210.103"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
```
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
ssh  root@kuber-node2 "mkdir -p /etc/flanneld/ssl"
scp flanneld*.pem root@kuber-node2:/etc/flanneld/ssl/
rm flanneld*.pem  flanneld.csr  flanneld-csr.json
scp flannel/{flanneld,mk-docker-opts.sh} root@kuber-node2:/root/local/bin/
```
mk-docker-opts.sh 脚本将分配给 flanneld 的 Pod 子网网段信息写入到 /run/flannel/docker 文件中，后续 docker 启动时使用这个文件中参数值设置 docker0 网桥；

[06-部署Master节点](file:///Users/xingjianwei/github/xingjianwei/follow-me-install-kubernetes-cluster/06-部署Master节点.md)

[安全端口启动服务](http://www.cnblogs.com/yangxiaoyi/p/6921594.html)

需要在开启kuber服务的节点上打开端口：
```
sudo firewall-cmd --zone=public --add-port=6443/tcp --permanent
sudo firewall-cmd --reload
```

[07-部署Node节点](file:///Users/xingjianwei/github/xingjianwei/follow-me-install-kubernetes-cluster/07-部署Node节点.md)

`MASTER_IP`每个节点都是相同的，`NODE_IP`每个节点都要分别设置。

```
export  PATH=/root/local/bin:$PATH
scp docker/docker* root@kuber-node1:/root/local/bin/
scp docker/docker* root@kuber-node2:/root/local/bin/
scp docker/completion/bash/docker root@kuber-node1:/etc/bash_completion.d/
scp docker/completion/bash/docker root@kuber-node2:/etc/bash_completion.d/
```
在执行
`iptables -F && sudo iptables -X && sudo iptables -F -t nat && sudo iptables -X -t nat`后，加上
`iptables -P FORWARD ACCEPT`
如果不加，会出现服务IP和节点IP ping不通的现象，dashboard也无法启动。
```
cat /etc/docker/daemon.json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn", "hub-mirror.c.163.com"],
  "insecure-registries": ["http://172.16.210.3:5000"],
  "max-concurrent-downloads": 10
}
```

```
scp -r ./server/bin/{kube-proxy,kubelet} root@kuber-node1:/root/local/bin/
scp -r ./server/bin/{kube-proxy,kubelet} root@kuber-node2:/root/local/bin/
scp bootstrap.kubeconfig root@kuber-node1:/etc/kubernetes/
scp bootstrap.kubeconfig root@kuber-node2:/etc/kubernetes/
scp kube-proxy*.pem root@kuber-node1:/etc/kubernetes/ssl/
scp kube-proxy*.pem root@kuber-node2:/etc/kubernetes/ssl/
rm kube-proxy.csr  kube-proxy-csr.json kube-proxy*.pem
scp kube-proxy.kubeconfig root@kuber-node1:/etc/kubernetes/
scp kube-proxy.kubeconfig root@kuber-node2:/etc/kubernetes/
```
可以使用以下命令验证pod：
```
kubectl get nodes
kubectl get pods  -o wide|grep nginx-ds
kubectl get svc |grep nginx-ds
```

[08-部署DNS插件](file:///Users/xingjianwei/github/xingjianwei/follow-me-install-kubernetes-cluster/08-部署DNS插件.md)
```
docker load < gcr.io-google_containers-k8s-dns-kube-dns-amd64-1.14.1.tar 
docker load < gcr.io-google_containers-k8s-dns-dnsmasq-nanny-amd64-1.14.1.tar 
docker load < gcr.io-google_containers-k8s-dns-sidecar-amd64-1.14.1.tar
```
开始因为gcr的image pull的版本号写错了的问题，最后的ping测试没有成功。
修改后成功

[09-部署Dashboard插件](file:///Users/xingjianwei/github/xingjianwei/follow-me-install-kubernetes-cluster/09-部署Dashboard插件.md)
`docker load < gcr.io-google_containers-kubernetes-dashboard-amd64-v1.6.0.tar`


```
kubectl get pods --all-namespaces  -o wide
kubectl proxy --address='172.16.210.101' --port=8086 --accept-hosts='^*$'
kubectl cluster-info
  Kubernetes master is running at https://172.16.210.101:6443
  KubeDNS is running at https://172.16.210.101:6443/api/v1/proxy/namespaces/kube-system/services/kube-dns
  kubernetes-dashboard is running at https://172.16.210.101:6443/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard
```
由于设置了proxy，可以访问`http://172.16.210.101:8086/ui`

也可以直接访问安全端口`http://172.16.210.101:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard`

由于 kube-apiserver 开启了 RBAC 授权，而浏览器访问 kube-apiserver 的时候使用的是匿名证书，所以访问安全端口会导致授权失败。

导入证书

将生成的admin.pem证书转换格式

```
cd /etc/kubernetes/ssl
openssl pkcs12 -export -in admin.pem  -out admin.p12 -inkey admin-key.pem
```
将生成的admin.p12证书导入的你的电脑(chrome->设置->高级->管理证书)；macos系统中在`钥匙串访问`中导入项目，导出的时候记住你设置的密码，导入的时候还要用到。

如果不导入证书，需要使用**非安全**端口访问 kube-apiserver：


