# CoreOS Tectonic Install

[参考文档1](https://coreos.com/tectonic/docs/latest/install/bare-metal/metal-terraform.html)
[参考文档2](https://coreos.com/matchbox/docs/latest/deployment.html#docker)
[参考文档3](https://coreos.com/blog/matchbox-with-terraform)
[参考文档4](https://coreos.com/tectonic/docs/latest/tutorials/first-app.html)

## 安装准备

172.16.210.101-103

系统IP	IPMIIP	IPMI用户	
172.16.210.101	172.16.210.111	root	
172.16.210.102	172.16.210.112	root	
172.16.210.103	172.16.210.113	root


## Matchbox

### Generate TLS Certificates

```
tar xzvf matchbox-v0.6.1-linux-amd64.tar.gz
cd matchbox-v0.6.1-linux-amd64
cd scripts/tls
export SAN=DNS.1:matchbox.tectoniclocal.com,IP.1:172.16.210.3
./cert-gen

sudo mkdir -p /etc/matchbox
sudo cp ca.crt server.crt server.key /etc/matchbox
```

Save `client.crt`, `client.key`, and `ca.crt` for later use (e.g. `~/.matchbox`).

```
mkdir -p /home/docker/volumes/matchbox/assets
sudo docker run --net=host -d -v /home/docker/volumes/matchbox:/var/lib/matchbox:Z -v /etc/matchbox:/etc/matchbox:Z,ro quay.io/coreos/matchbox:latest -address=0.0.0.0:8090 -rpc-address=0.0.0.0:8091 -log-level=debug
```

## coreos/dnsmasq

Run DHCP, TFTP, and DNS on the host's network:

```
sudo docker run -d --cap-add=NET_ADMIN --net=host quay.io/coreos/dnsmasq \
  -d -q \
  --dhcp-range=172.16.210.100,172.16.210.119 \
  --enable-tftp --tftp-root=/var/lib/tftpboot \
  --dhcp-userclass=set:ipxe,iPXE \
  --dhcp-boot=tag:#ipxe,undionly.kpxe \
  --dhcp-boot=tag:ipxe,http://matchbox.tectoniclocal.com:8090/boot.ipxe \
  --address=/matchbox.tectoniclocal.com/172.16.210.3 \
  --log-queries \
  --log-dhcp
```
Be sure to allow enabled services in your firewall configuration.
`sudo firewall-cmd --add-service=dhcp --add-service=tftp --add-service=dns`

