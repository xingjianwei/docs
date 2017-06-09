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
```
firewall-cmd --permanent --zone=dmz --add-port=8090/tcp
firewall-cmd --permanent --zone=dmz --add-port=8091/tcp
systemctl reload firewalld
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

## Install Tectonic on Bare-Metal with Terraform
```
cd /home/coreos
curl -O https://releases.tectonic.com/tectonic-1.6.2-tectonic.1.tar.gz
tar xzvf tectonic-1.6.2-tectonic.1.tar.gz
cd tectonic
export INSTALLER_PATH=$(pwd)/tectonic-installer/linux/installer
export PATH=$PATH:$(pwd)/tectonic-installer/linux
sed "s|<PATH_TO_INSTALLER>|$INSTALLER_PATH|g" terraformrc.example > .terraformrc
export TERRAFORM_CONFIG=$(pwd)/.terraformrc
terraform get ./platforms/metal
```
Customize the deployment
```
export CLUSTER=tectonic-cluster
mkdir -p build/${CLUSTER}
cp examples/terraform.tfvars.metal build/${CLUSTER}/terraform.tfvars
```
Customizations should be made to build/${CLUSTER}/terraform.tfvars. Edit the following variables to correspond to your matchbox installation:
```
tectonic_metal_matchbox_http_url = "http://matchbox.tectoniclocal.com:8090"
tectonic_metal_matchbox_rpc_endpoint = "matchbox.tectoniclocal.com:8091"
tectonic_metal_matchbox_client_cert = "<<EOD
xxx
EOD"
tectonic_metal_matchbox_client_cert = "<<EOD
xxx
EOD"
tectonic_metal_matchbox_ca = "<<EOD
xx
EOD"

```
Edit additional variables to specify DNS records, list machines, and set a SSH key and Tectonic Console email and password.

Several variables are currently required, but their values are not used.

make tectonic_admin_password_hash:
```
wget https://github.com/coreos/bcrypt-tool/releases/download/v1.0.0/bcrypt-tool-v1.0.0-linux-amd64.tar.gz
./bcrypt-tool/bcrypt-tool
```

```
tectonic_admin_email = "xxxx@xxxx.com"
tectonic_admin_password_hash = ""
tectonic_base_domain = "tectoniclocal.com"
tectonic_cluster_name = "tectonic-cluster"
tectonic_master_count = "1"
tectonic_worker_count = "3"
tectonic_etcd_count = "3"
tectonic_license_path = "/home/coreos"
tectonic_metal_cl_version = "1353.8.0"
tectonic_metal_controller_domain = "node1.example.com"
tectonic_metal_controller_macs = "["xx:xx:xx:xx:xx:xx"]"
tectonic_metal_ingress_domain = "node2.example.com"
tectonic_metal_worker_domains = "["node1.example.com", "node2.example.com", "node3.example.com"]"
tectonic_metal_worker_macs = "["xx", "xx", "xx"]"
tectonic_pull_secret_path = "matchbox.example.com:8091"

```


