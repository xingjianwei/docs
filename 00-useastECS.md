# useast ECS
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
ExecStart=/usr/bin/dockerd --insecure-registry xxxx.com:5000
systemctl stop docker
systemctl daemon-reload
sudo systemctl start docker 
私有仓库管理：
http://xxxx.com:8080/

## 上网代理shadowsocks

[参考文档1](https://store.docker.com/community/images/mritd/shadowsocks)

[参考文档2](https://store.docker.com/community/images/smounives/shadowsocksr-docker)

docker pull mritd/shadowsocks

### Server 端

`docker run -dt --name ss -p 6443:6443 -p 6500:6500/udp mritd/shadowsocks -s "-s :: -s 0.0.0.0 -p 6443 -m aes-256-cfb -k test123 --fast-open" -k "-t 127.0.0.1:6443 -l :6500 -mode fast2" -x`

以上命令相当于执行了

`ss-server -s :: -s 0.0.0.0 -p 6443 -m aes-256-cfb -k test123 --fast-open
kcptun -t 127.0.0.1:6443 -l :6500 -mode fast2`

### Client 端

`docker run -dt --name ss -p 1080:1080 mritd/shadowsocks -m "ss-local" -s "-c /etc/shadowsocks-libev/test.json`

以上命令相当于执行了

`ss-local -c /etc/shadowsocks-libev/test.json`
关于 shadowsocks-libev 和 kcptun 都支持哪些参数请自行查阅官方文档，本镜像只做一个拼接

注意：kcptun 映射端口为 udp 模式(6500:6500/udp)，不写默认 tcp；shadowsocks 请监听 0.0.0.0

### 环境变量支持

环境变量	作用	取值
SS_MODULE	shadowsocks 启动命令	ss-local、ss-manager、ss-nat、ss-redir、ss-server、ss-tunnel
SS_CONFIG	shadowsocks-libev 参数字符串	所有字符串内内容应当为 shadowsocks-libev 支持的选项参数
KCP_CONFIG	kcptun 参数字符串	所有字符串内内容应当为 kcptun 支持的选项参数
KCP_FLAG	是否开启 kcptun 支持	可选参数为 true 和 false，默认为 fasle 禁用 kcptun
使用时可指定环境变量，如下

`docker run -dt --name ss -p 6443:6443 -p 6500:6500/udp -e SS_CONFIG="-s 0.0.0.0 -p 6443 -m aes-256-cfb -k test123 --fast-open" -e KCP_CONFIG="-t 127.0.0.1:6443 -l :6500 -mode fast2" -e KCP_FLAG="true" mritd/shadowsocks`