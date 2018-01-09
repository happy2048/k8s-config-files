## kubernetes1.8.6手动安装

说明： 本教程是对[《Kubernetes v1.8.x 全手動苦工安裝教學》](https://kairen.github.io/2017/10/27/kubernetes/deploy/manual-v1.8/)或者[《Kubernetes 1.8.x 全手动安装教程》](https://www.kubernetes.org.cn/3096.html)的修改补充，因为感觉原文的安装有点混乱，所以这里特意整理了一下，其中有些文字直接使用原文的，望见谅。

### 环境准备

本次安装的版本为：
	
	kubernetes v1.8.6
	etcd v3.2.9
	calico v2.6.2
	docker v1.12.0
预先准备信息

本教程的节点信息如下：

| IP address  |  Role | hostname  |
| ------------ | ------------ | ------------ |
| 10.61.0.160  | master,etcd node  | kuber-master  |
| 10.61.0.161  |  node,etcd node|  kuber-node1 |
| 10.61.0.162  | node,etcd node  |  kuber-node2 |
| 10.61.0.163 |  node |  kuber-node3 |
| 10.61.0.164 |  node |   kuber-node4|
说明：

	1.集群使用的网段为10.96.0.0/12，原文中是设置的这个网段，它是一个虚网段，所以我们不需要配置物理网卡上配置这个网段的ip，既然是虚网段，我们还是按照原作者选择的10.96.0.0/12的网段，并且使用其中的10.96.0.10作为集群ip。
	2.10.61.0.0/24这个网段是我的物理网段，如果这个网段需要根据实际情况做出相应的调整。
	3.etcd集群我这选3个节点，原文中选用一个节点，这个可靠性不高。

首先安装前要确认以下几项都准备好了：

1.节点需要彼此互通，并且kuber-master能够免密码登陆其他节点。

2.所有防火墙与selinux关闭，如Centos:

	systemctl stop firewalld && systemctl disable firewalld
	setenforce 0
	vim /etc/selinux/config
	SELINUX=disabled(/etc/selinux/config中此项变为disabled)
3.所有节点需要设定/etc/host解析到所有主机。(这里如果节点比较少，可以使用这种方式，如果节点比较多，可以考虑使用dnsmasq，或者bind)

	[root@kuber-master ~]# cat /etc/hosts
	127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
	::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
	10.61.0.160 kuber-master
	10.61.0.161 kuber-node1
	10.61.0.162 kuber-node2
	10.61.0.163 kuber-node3
	10.61.0.164 kuber-node4
4.所有节点都安装docker引擎，这里我直接选择yum安装(以下步骤在所有节点上执行)：

	[root@kuber-master ~]# yum install epel-release
	[root@kuber-master ~]# yum install docker -y
	[root@kuber-master ~]# systemctl enable docker && systemctl restart docker
	[root@kuber-master ~]# docker version 
	Client:
	 Version:         1.12.6
	 API version:     1.24
	 Package version: docker-1.12.6-68.gitec8512b.el7.centos.x86_64
	 Go version:      go1.8.3
	 Git commit:      ec8512b/1.12.6
	 Built:           Mon Dec 11 16:08:42 2017
	 OS/Arch:         linux/amd64

	Server:
	 Version:         1.12.6
	 API version:     1.24
	 Package version: docker-1.12.6-68.gitec8512b.el7.centos.x86_64
	 Go version:      go1.8.3
	 Git commit:      ec8512b/1.12.6
	 Built:           Mon Dec 11 16:08:42 2017
	 OS/Arch:         linux/amd64

5.所有节点需要设定/etc/sysctl.d/k8s.conf的系统参数。

	[root@kuber-master ~]# cat  /etc/sysctl.d/k8s.conf 
	net.ipv4.ip_forward = 1
	net.bridge.bridge-nf-call-ip6tables = 1
	net.bridge.bridge-nf-call-iptables = 1

6.所有节点必须保持时区和时间一致，否则etcd集群会出现时钟漂移，这里有两种方法，一种是本地安装一个ntp server，另一种是直接使用网上的ntp server，我这里就选择第二种（以下步骤在所有节点上进行），添加最后一行到/etc/crontab（首先确定ntpdate是否安装）：

首先更改时区：

	[root@kuber-master ~]# date
	Sun Jan  7 20:41:06 EST 2018
	[root@kuber-master ~]# rm -rf /etc/localtime 
	[root@kuber-master ~]# ln -sv /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
	‘/etc/localtime’ -> ‘/usr/share/zoneinfo/Asia/Shanghai’
	[root@kuber-master ~]# date
	Mon Jan  8 09:41:58 CST 2018
然后添加定时时间同步任务：

	[root@kuber-master ~]# cat /etc/crontab
	SHELL=/bin/bash
	PATH=/sbin:/bin:/usr/sbin:/usr/bin
	MAILTO=root

	# For details see man 4 crontabs

	# Example of job definition:
	# .---------------- minute (0 - 59)
	# |  .------------- hour (0 - 23)
	# |  |  .---------- day of month (1 - 31)
	# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
	# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
	# |  |  |  |  |
	# *  *  *  *  * user-name  command to be executed

	*/2 * * * *  root  ntpdate cn.ntp.org.cn &> /dev/null
	
7.在kuber-master上安装cfssl工具，这将会用来建立 TLS certificates。

	[root@kuber-master ~]# export CFSSL_URL="https://pkg.cfssl.org/R1.2"
	[root@kuber-master ~]# wget "${CFSSL_URL}/cfssl_linux-amd64" -O /usr/local/bin/cfssl
	[root@kuber-master ~]# wget "${CFSSL_URL}/cfssljson_linux-amd64" -O /usr/local/bin/cfssljson
	[root@kuber-master ~]# chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
	
8.准备需要用到的配置文件

	[root@kuber-master tmp]# cd /opt
	[root@kuber-master opt]# git clone https://github.com/happy2048/k8s-config-files.git
	[root@kuber-master ~]# echo "export K8S_CONFIG_FILES=/opt/k8s-config-files" >> ~/.bashrc
	[root@kuber-master ~]# echo  "export KUBE_APISERVER=https://10.61.0.160:6443" >>  ~/.bashrc
	[root@kuber-master ~]# source ~/.bashrc 
	[root@kuber-master ~]# echo $K8S_CONFIG_FILES
	/opt/k8s_config_files
	
### Etcd安装

在开始安装 Kubernetes 之前，需要先将一些必要系统创建完成，其中 Etcd 就是 Kubernetes 最重要的一环，Kubernetes 会将大部分信息储存于 Etcd 上，来提供给其他节点索取，以确保整个集群运作与沟通正常。

1.使用yum安装etcd（该步骤所有etcd节点上进行，这里我只展示在kuber-master的操作，其他etcd节点执行一样）

	[root@kuber-master ~]# yum install epel-release -y
	[root@kuber-master ~]# yum install etcd -y
	[root@kuber-master ~]# mkdir /etc/etcd/ssl && chown etcd:etcd  /etc/etcd/ssl
	
2.建集群CA与Certificates（以下步骤在kuber-master进行)

2.1 建立/etc/etcd/ssl文件夹，然后进入目录完成以下操作。

	[root@kuber-master etcd]#  cd /etc/etcd/ssl
	
2.2 复制ca-config.json与etcd-ca-csr.json文件到本目录下，并产生 CA 密钥（此步骤在kuber-master上进行）：

	[root@kuber-master ssl]# cp $K8S_CONFIG_FILES/pki/ca-config.json  $K8S_CONFIG_FILES/pki/etcd-ca-csr.json ./
	[root@kuber-master ssl]#  cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare etcd-ca
	2018/01/08 10:30:32 [INFO] generating a new CA key and certificate from CSR
	2018/01/08 10:30:32 [INFO] generate received request
	2018/01/08 10:30:32 [INFO] received CSR
	2018/01/08 10:30:32 [INFO] generating key: rsa-2048
	2018/01/08 10:30:32 [INFO] encoded CSR
	2018/01/08 10:30:32 [INFO] signed certificate with serial number 343991803789219532784889668850923759971181192623
	[root@kuber-master ssl]# ls
	ca-config.json  etcd-ca.csr  etcd-ca-csr.json  etcd-ca-key.pem  etcd-ca.pem
	
ps: 本步骤中的两个json文件无需做更改,其中etcd-ca-csr.json中names字段的值解释如下：

	"C": country
	"L": locality or municipality (such as city or town name)
	"O": organisation
	"OU": organisational unit, such as the department responsible for owning the key; it can also be used for a "Doing Business As" (DBS) name
	"ST": the state or province
	
2.3 复制etcd-csr.json文件到本目录下，并产生 kube-apiserver certificate 证书(本步骤不能直接运行命令，需要对etcd-csr.json做相应的修改，文件修改说明如下)（此步骤在kuber-master上进行）：

	[root@kuber-master ssl]# cp $K8S_CONFIG_FILES/pki/etcd-csr.json  ./
	[root@kuber-master ssl]# cfssl gencert \
	>  -ca=etcd-ca.pem \
	> -ca-key=etcd-ca-key.pem \
	> -config=ca-config.json \
	> -profile=kubernetes \
	> etcd-csr.json | cfssljson -bare etcd
	
修改属主：

	[root@kuber-master ~]# chown -R etcd:etcd /etc/etcd/ssl

ps: etcd-csr.json文件中hosts字段ip需要把所有etcd节点的ip和主机名都写进去，我这是三个节点所以就写了三个，另外再加一个127.0.0.1,上述命令出现警告可忽略。

2.4 将生成的相关认证文件复制到所有节点，使用如下脚本执行：

	[root@kuber-master ssl]# cat /tmp/copy.sh 
	#!/bin/bash
	for i in kuber-node1 kuber-node2 kuber-node3 kuber-node4;do
		ssh $i mkdir -pv /etc/etcd
		if ! ssh $i id etcd &>/dev/null;then
			ssh $i groupadd -g 992 etcd
			ssh $i useradd -u 995 -g 992 -M etcd
		fi
		scp -r /etc/etcd/ssl $i:/etc/etcd
		ssh $i chown etcd:etcd -R /etc/etcd/ssl
	done
	
2.5 这里我选择静态文件配置etcd，当然也可以像原文那样选择直接在命令行配置，配置文件的地址为/etc/etcd/etcd.conf（配置文件的修改在所有的etcd节点上进行,下面是在kuber-master进行的，不同的节点配置文件的信息有点不同，具体如下）:

	[root@kuber-master ssl]# cp $K8S_CONFIG_FILES/config/etcd.conf  /etc/etcd/etcd.conf
	[root@kuber-master ssl]# scp /etc/etcd/etcd.conf kuber-node1:/etc/etcd
	[root@kuber-master ssl]# scp /etc/etcd/etcd.conf kuber-node2:/etc/etcd
	[root@kuber-master ssl]# cat /etc/etcd/etcd.conf 
	#[Member]
	#ETCD_CORS=""
	ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
	#ETCD_WAL_DIR=""
	ETCD_LISTEN_PEER_URLS="https://0.0.0.0:2380"
	ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,https://0.0.0.0:2379"
	#ETCD_MAX_SNAPSHOTS="5"
	#ETCD_MAX_WALS="5"
	ETCD_NAME="etcd-master"
	#ETCD_SNAPSHOT_COUNT="100000"
	#ETCD_HEARTBEAT_INTERVAL="100"
	#ETCD_ELECTION_TIMEOUT="1000"
	#ETCD_QUOTA_BACKEND_BYTES="0"
	#
	#[Clustering]
	ETCD_INITIAL_ADVERTISE_PEER_URLS="https://kuber-master:2380"
	ETCD_ADVERTISE_CLIENT_URLS="https://kuber-master:2379"
	#ETCD_DISCOVERY=""
	#ETCD_DISCOVERY_FALLBACK="proxy"
	#ETCD_DISCOVERY_PROXY=""
	#ETCD_DISCOVERY_SRV=""
	ETCD_INITIAL_CLUSTER="etcd-master=https://kuber-master:2380,etcd-node1=https://kuber-node1:2380,etcd-node2=https://kuber-node2:2380"
	ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-1"
	ETCD_INITIAL_CLUSTER_STATE="new"
	#ETCD_STRICT_RECONFIG_CHECK="true"
	#ETCD_ENABLE_V2="true"
	#
	#[Proxy]
	#ETCD_PROXY="off"
	#ETCD_PROXY_FAILURE_WAIT="5000"
	#ETCD_PROXY_REFRESH_INTERVAL="30000"
	#ETCD_PROXY_DIAL_TIMEOUT="1000"
	#ETCD_PROXY_WRITE_TIMEOUT="5000"
	#ETCD_PROXY_READ_TIMEOUT="0"
	#
	#[Security]
	ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
	ETCD_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
	ETCD_CLIENT_CERT_AUTH="true"
	ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
	ETCD_AUTO_TLS="true"
	ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd.pem"
	ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
	ETCD_PEER_CLIENT_CERT_AUTH="true"
	ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
	ETCD_PEER_AUTO_TLS="true"
	#
	#[Logging]
	#ETCD_DEBUG="false"
	#ETCD_LOG_PACKAGE_LEVELS=""
	#ETCD_LOG_OUTPUT="default"
	#
	#[Unsafe]
	#ETCD_FORCE_NEW_CLUSTER="false"
	#
	#[Version]
	#ETCD_VERSION="false"
	#ETCD_AUTO_COMPACTION_RETENTION="0"
	#
	#[Profiling]
	#ETCD_ENABLE_PPROF="false"
	#ETCD_METRICS="basic"
	#
	#[Auth]
	#ETCD_AUTH_TOKEN="simple"
	
配置文件说明：

	1.ETCD_DATA_DIR如果这个目录在还没启动etcd时就已经存在了，说明你曾经装过etcd，请把所有节点的/var/lib/etcd下的所有文件都删除，保留/var/lib/etcd为空目录，否则etcd将不会启动成功
	2.ETCD_INITIAL_ADVERTISE_PEER_URLS和ETCD_ADVERTISE_CLIENT_URLS需要写本etcd节点的ip或者主机名,保证其他节点能够解析，举例来说,如果我这个节点是kuber-master,那么我这就写https://kuber-master:2380 ，如果我这个节点是kuber-node1，那么我就写https://kuber-node1:2380
	3.ETCD_NAME需要给每个节点取一个不同的名字，例如kuber-master这台机器，我取名为etcd-master，kuber-node1我取名为etcd-node1
	4.ETCD_INITIAL_CLUSTERxuya需要写入所有的节点
	5.ETCD_INITIAL_CLUSTER_STATE首次运行写"new"，运行成功后修改为existing,再重启etcd服务
	
2.6 启动etcd服务（本步骤在所有节点上执行，并且需要同时启动，不能是第一个节点启动成功后再启动第二个etcd节点，建议开多个终端，分别执行，否则etcd服务可能执行不成功）

	[root@kuber-master ~]# systemctl enable etcd
	[root@kuber-master ~]# systemctl start etcd
	[root@kuber-node2 ~]# systemctl status  etcd
	
如果服务没有启动,使用journalctl -xe查看，如果出现以下错误信息，需要修改/etc/etcd/ssl下所有文件的属主为etcd：

	Jan 08 12:13:47 kuber-master etcd[17420]: open /etc/etcd/ssl/etcd-key.pem: permission denied
	Jan 08 12:13:47 kuber-master systemd[1]: etcd.service: main process exited, code=exited, status=1/FAILURE
	Jan 08 12:13:47 kuber-master systemd[1]: Failed to start Etcd Server.
	
2.7 检查etcd集群健康状态，在任意一个etcd节点上执行如下步骤（我这是在kuber-master上执行，但是访问的是kuber-node2节点）：
	
	[root@kuber-master ~]# etcdctl --cert-file=/etc/etcd/ssl/etcd.pem  \
	--key-file=/etc/etcd/ssl/etcd-key.pem \
	--ca-file=/etc/etcd/ssl/etcd-ca.pem \
	--endpoint https://kuber-node2:2379 cluster-health
	
显示信息如下：

	member 14b69ef4fd7851b4 is healthy: got healthy result from https://kuber-node1:2379
	member 773eb256b3c437b8 is healthy: got healthy result from https://kuber-master:2379
	member ed5a8029cadfb1e3 is healthy: got healthy result from https://kuber-node2:2379
	
### Calico安装

原文的calico安装放在比较靠后的地方，导致依赖它的服务启动不起来，这里我们把它首先安装。

Calico 是一款纯 Layer 3 的数据中心网络方案(不需要 Overlay 网络)，Calico 好处是他已与各种云原生平台有良好的整合，而 Calico 在每一个节点利用 Linux Kernel 实现高效的 vRouter 来负责数据的转发，而当数据中心复杂度增加时，可以用 BGP route reflector 来达成。

1 保证每个节点上的docker服务启动，因为calico服务运行在容器中的（我这在kuber-master上执行的，其他节点也一样）
	
	[root@kuber-master ssl]# systemctl status docker 
	● docker.service - Docker Application Container Engine
	   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
	   Active: active (running) since Sun 2018-01-07 20:28:53 CST; 16h ago
	   
2.下载配置cni组件（此步骤在kuber-master上进行）	

	[root@kuber-master ~]# mkdir -p /opt/cni/bin && cd /opt/cni/bin
	[root@kuber-master ~]# export CNI_URL="https://github.com/containernetworking/plugins/releases/download"
	[root@kuber-master bin]# wget  "${CNI_URL}/v0.6.0/cni-plugins-amd64-v0.6.0.tgz" 
	[root@kuber-master bin]# tar -xf cni-plugins-amd64-v0.6.0.tgz
	
如果下载不成功可以使用我已经下好的，把wget那条命令换成如下的命令：

	[root@kuber-master bin]# cp $K8S_CONFIG_FILES/software/cni-plugins-amd64-v0.6.0.tgz  ./
	
3 下载calico(此步骤在kuber-master进行)：

	[root@kuber-master bin]# wget https://github.com/projectcalico/calicoctl/releases/download/v1.6.1/calicoctl -O /usr/bin/calicoctl
	[root@kuber-master bin]# chmod +x /usr/bin/calicoctl
	export CALICO_URL="https://github.com/projectcalico/cni-plugin/releases/download/v1.11.0"
	[root@kuber-master bin]# export CALICO_URL="https://github.com/projectcalico/cni-plugin/releases/download/v1.11.0"
	[root@kuber-master bin]# wget -N -P /opt/cni/bin ${CALICO_URL}/calico
	[root@kuber-master bin]# wget -N -P /opt/cni/bin ${CALICO_URL}/calico-ipam
	[root@kuber-master bin]# chmod +x calico calico-ipam
	
4 接着修改CNI plugins配置文件，以及 calico-node.service：

	[root@kuber-master bin]# mkdir -pv /etc/cni/net.d
	[root@kuber-master bin]# cp $K8S_CONFIG_FILES/network/10-calico.conf /etc/cni/net.d/ 
	[root@kuber-master bin]# cp $K8S_CONFIG_FILES/network/calico-node.service /lib/systemd/system/
	[root@kuber-master ssl]# systemctl enable calico-node
	[root@kuber-master ssl]# systemctl start calico-node
	
修改/etc/cni/net.d/10-calico.conf，修改信息如下：

	etcd_endpoints需要把所有的etcd节点访问url加进去，这样可以避免其中一个etcd节点出了问题不影响客户端访问etcd集群
	
修改/lib/systemd/system/calico-node.serviceye，修改信息如下：

	ETCD_ENDPOINTS的值变为所有etcd节点的访问url
	如果有多个网卡，需要在IP_AUTODETECTION_METHODhe和IP6_AUTODETECTION_METHOD指定集群通信的网卡。

5 使用脚本将配置文件复制到所有节点上：

	[root@kuber-master ssl]# cat /tmp/calico-copy.sh 
	#!/bin/bash
	for i in {kuber-node1,kuber-node2,kuber-node3,kuber-node4};do
		scp -r /opt/cni $i:/opt
		scp -r /etc/cni $i:/etc
		scp /lib/systemd/system/calico-node.service $i:/lib/systemd/system
		ssh $i systemctl enable calico-node.service
		ssh $i systemctl start calico-node.service
	done
	
如果无法启动，使用journalctl -xe查看原因，如果出现以下原因：

	 WARNING: Unable to auto-detect an IPv4 address using interface regexes [eth0]: no valid IPv4 addresses found on the host interfaces
	 ERROR: Couldn't autodetect a management IPv4 address:
	 
可能的情况就是/lib/systemd/system/calico-node.serviceye的IP_AUTODETECTION_METHOD值设置不对，比如我这的10.61.0.160是在eth2上，这时需要把IP_AUTODETECTION_METHOD和IP6_AUTODETECTION_METHOD的值改为eth2。

启动以后需要等一段时间，因为要拉取镜像，在每个节点上可以使用docker ps查看容器是否启动起来了。

6 在kuber-master上查看 Calico nodes:

	[root@kuber-master ssl]# cat <<EOF > ~/calico-rc
	export ETCD_ENDPOINTS="https://kuber-master:2379,https://kuber-node1:2379,https://kuber-node2:2379"
	export ETCD_CA_CERT_FILE="/etc/etcd/ssl/etcd-ca.pem"
	export ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
	export ETCD_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
	EOF

	[root@kuber-master ssl]# . ~/calico-rc
	[root@kuber-master ssl]# calicoctl get node -o wide
	NAME           ASN       IPV4             IPV6   
	kuber-master   (64512)   10.61.0.160/24          
	kuber-node1    (64512)   10.61.0.161/24          
	kuber-node2    (64512)   10.61.0.162/24          
	kuber-node3    (64512)   10.61.0.163/24          
	kuber-node4    (64512)   10.61.0.164/24          

### Kubernetes Master

Master 是 Kubernetes 的大总管，主要创建apiserver、Controller manager与Scheduler来组件管理所有 Node。本步骤将下载 Kubernetes 并安装至 kuber-master上，然后产生相关 TLS Cert 与 CA 密钥，提供给集群组件认证使用。	

1 下载 Kubernetes 组件

	[root@kuber-master ssl]# mkdir /opt/k8s && cd /opt/k8s
	[root@kuber-master k8s]# wget  https://github.com/kubernetes/kubernetes/releases/download/v1.8.6/kubernetes.tar.gz
	[root@kuber-master k8s]# tar -xf kubernetes.tar.gz
	[root@kuber-master k8s]# bash kubernetes/cluster/get-kube-binaries.sh
	[root@kuber-master k8s]# tar -xf kubernetes/server/kubernetes-server-linux-amd64.tar.gz
	[root@kuber-master k8s]# ls -l kubernetes/server/bin
	[root@kuber-master k8s]# cp  kubernetes/server/bin/kubectl  kubernetes/server/bin/kubelet  /usr/local/bin
	
2 创建集群 CA 与 Certificates

在这部分，将会需要生成 client 与 server 的各组件 certificates，并且替 Kubernetes admin user 生成 client 证书。

2.1 创建pki文件夹，然后进入目录完成以下操作。

	[root@kuber-master k8s]# mkdir -p /etc/kubernetes/pki && cd /etc/kubernetes/pki
	[root@kuber-master pki]# export KUBE_APISERVER="https://10.61.0.160:6443"
	
复制ca-config.json与ca-csr.json文件到当前目录，并生成 CA 密钥：

	[root@kuber-master pki]# cp $K8S_CONFIG_FILES/pki/ca-csr.json  $K8S_CONFIG_FILES/pki/ca-config.json   ./
	[root@kuber-master pki]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca
	
2.2 API server certificate

复制apiserver-csr.json文件，并生成 kube-apiserver certificate 证书：

	[root@kuber-master pki]# cp $K8S_CONFIG_FILES/pki/apiserver-csr.json   ./
	[root@kuber-master pki]# cfssl gencert \
	-ca=ca.pem \
	-ca-key=ca-key.pem \
	-config=ca-config.json \
	-hostname=10.96.0.1,10.61.0.160,127.0.0.1,kubernetes.default,kuber-master \
	-profile=kubernetes \
	apiserver-csr.json | cfssljson -bare apiserver
	
2.3 Front proxy certificate

复制front-proxy-ca-csr.json文件，并生成 Front proxy CA 密钥，Front proxy 主要是用在 API aggregator 上:

	[root@kuber-master pki]# cp $K8S_CONFIG_FILES/pki/front-proxy-ca-csr.json   ./
	[root@kuber-master pki]#  cfssl gencert \
	-initca front-proxy-ca-csr.json | cfssljson -bare front-proxy-ca
	
复制front-proxy-client-csr.json文件，并生成 front-proxy-client 证书：

	[root@kuber-master pki]# cp $K8S_CONFIG_FILES/pki/front-proxy-client-csr.json ./
	[root@kuber-master pki]# cfssl gencert \
	-ca=front-proxy-ca.pem \
	-ca-key=front-proxy-ca-key.pem \
	-config=ca-config.json \
	-profile=kubernetes \
	front-proxy-client-csr.json | cfssljson -bare front-proxy-client
	
2.4 Bootstrap Token
	
由于通过手动创建 CA 方式太过繁杂，只适合少量机器，因为每次签证时都需要绑定 Node IP，随机器增加会带来很多困扰，因此这边使用 TLS Bootstrapping 方式进行授权，由 apiserver 自动给符合条件的 Node 发送证书来授权加入集群。

主要做法是 kubelet 启动时，向 kube-apiserver 传送 TLS Bootstrapping 请求，而 kube-apiserver 验证 kubelet 请求的 token 是否与设定的一样，若一样就自动产生 kubelet 证书与密钥。具体作法可以参考 TLS bootstrapping。
	
首先建立一个变量来产生BOOTSTRAP_TOKEN，并建立 bootstrap.conf 的 kubeconfig 文件：	
	
	[root@kuber-master pki]# export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
	[root@kuber-master pki]# cat <<EOF > /etc/kubernetes/token.csv
	${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
	EOF
	
	# bootstrap set-cluster
	[root@kuber-master pki]# kubectl config set-cluster kubernetes \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=../bootstrap.conf
	
	# bootstrap set-credentials
	[root@kuber-master pki]# kubectl config set-credentials kubelet-bootstrap \
	--token=${BOOTSTRAP_TOKEN} \
	--kubeconfig=../bootstrap.conf
	
	# bootstrap set-context
	[root@kuber-master pki]# kubectl config set-context default \
	--cluster=kubernetes \
	--user=kubelet-bootstrap \
	--kubeconfig=../bootstrap.conf
	
	# bootstrap set default context
	[root@kuber-master pki]# kubectl config use-context default --kubeconfig=../bootstrap.conf
	
2.5 Admin certificate
	
复制admin-csr.json文件，并生成 admin certificate 证书：

	[root@kuber-master pki]# cp $K8S_CONFIG_FILES/pki/admin-csr.json ./
	[root@kuber-master pki]# cfssl gencert \
	-ca=ca.pem \
	-ca-key=ca-key.pem \
	-config=ca-config.json \
	-profile=kubernetes \
	admin-csr.json | cfssljson -bare admin

接着通过以下指令生成名称为 admin.conf 的 kubeconfig 文件：	
	
	# admin set-cluster
	[root@kuber-master pki]# kubectl config set-cluster kubernetes \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=../admin.conf
	
	# admin set-credentials
	[root@kuber-master pki]# kubectl config set-credentials kubernetes-admin \
	--client-certificate=admin.pem \
	--client-key=admin-key.pem \
	--embed-certs=true \
	--kubeconfig=../admin.conf

	# admin set-context
	[root@kuber-master pki]# kubectl config set-context kubernetes-admin@kubernetes \
	--cluster=kubernetes \
	--user=kubernetes-admin \
	--kubeconfig=../admin.conf
	
	# admin set default context
	[root@kuber-master pki]# kubectl config use-context kubernetes-admin@kubernetes \
	--kubeconfig=../admin.conf

2.6 Controller manager certificate

复制manager-csr.json文件，并生成 kube-controller-manager certificate 证书：

	[root@kuber-master pki]# cp $K8S_CONFIG_FILES/pki/manager-csr.json ./
	[root@kuber-master pki]# cfssl gencert \
	-ca=ca.pem \
	-ca-key=ca-key.pem \
	-config=ca-config.json \
	-profile=kubernetes \
	manager-csr.json | cfssljson -bare controller-manager
接着通过以下指令生成名称为controller-manager.conf的 kubeconfig 文件：

	# controller-manager set-cluster
	[root@kuber-master pki]# kubectl config set-cluster kubernetes \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=../controller-manager.conf
	
	# controller-manager set-credentials
	[root@kuber-master pki]# kubectl config set-credentials system:kube-controller-manager \
	--client-certificate=controller-manager.pem \
	--client-key=controller-manager-key.pem \
	--embed-certs=true \
	--kubeconfig=../controller-manager.conf

	# controller-manager set-context
	[root@kuber-master pki]# kubectl config set-context system:kube-controller-manager@kubernetes \
	--cluster=kubernetes \
	--user=system:kube-controller-manager \
	--kubeconfig=../controller-manager.conf
	
	# controller-manager set default context
	[root@kuber-master pki]# kubectl config use-context system:kube-controller-manager@kubernetes \
	--kubeconfig=../controller-manager.conf

2.7 Scheduler certificate

复制scheduler-csr.json文件，并生成 kube-scheduler certificate 证书：

	[root@kuber-master pki]# cp $K8S_CONFIG_FILES/pki/scheduler-csr.json  ./
	[root@kuber-master pki]# cfssl gencert \
	-ca=ca.pem \
	-ca-key=ca-key.pem \
	-config=ca-config.json \
	-profile=kubernetes \
	scheduler-csr.json | cfssljson -bare scheduler
接着通过以下指令生成名称为 scheduler.conf 的 kubeconfig 文件：

	# scheduler set-cluster
	[root@kuber-master pki]# kubectl config set-cluster kubernetes \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=../scheduler.conf
	
	# scheduler set-credentials
	[root@kuber-master pki]# kubectl config set-credentials system:kube-scheduler \
	--client-certificate=scheduler.pem \
	--client-key=scheduler-key.pem \
	--embed-certs=true \
	--kubeconfig=../scheduler.conf

	# scheduler set-context
	[root@kuber-master pki]# kubectl config set-context system:kube-scheduler@kubernetes \
	--cluster=kubernetes \
	--user=system:kube-scheduler \
	--kubeconfig=../scheduler.conf

	# scheduler set default context
	[root@kuber-master pki]# kubectl config use-context system:kube-scheduler@kubernetes \
	--kubeconfig=../scheduler.conf
	
2.8 Kubelet master certificate

复制kubelet-csr.json文件，并生成 master node certificate 证书：

	[root@kuber-master pki]# cp $K8S_CONFIG_FILES/pki/kubelet-csr.json ./
	
把NODE字符串替换为master主机名(我这是kuber-master):

	[root@kuber-master pki]# sed -i 's@NODE@kuber-master@g' kubelet-csr.json
	[root@kuber-master pki]# cfssl gencert \
	-ca=ca.pem \
	-ca-key=ca-key.pem \
	-config=ca-config.json \
	-hostname=kuber-master,10.61.0.160 \
	-profile=kubernetes \
	kubelet-csr.json | cfssljson -bare kubelet
	
此处的-hostname需要根据实际情况做调整。

接着通过以下指令生成名称为 kubelet.conf 的 kubeconfig 文件：

	# kubelet set-cluster
	[root@kuber-master pki]# kubectl config set-cluster kubernetes \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=../kubelet.conf
	
	# kubelet set-credentials
	[root@kuber-master pki]# kubectl config set-credentials system:node:kuber-master \
	--client-certificate=kubelet.pem \
	--client-key=kubelet-key.pem \
	--embed-certs=true \
	--kubeconfig=../kubelet.conf

	# kubelet set-context
	[root@kuber-master pki]# kubectl config set-context system:node:kuber-master@kubernetes \
	--cluster=kubernetes \
	--user=system:node:kuber-master \
	--kubeconfig=../kubelet.conf
	

	# kubelet set default context
	[root@kuber-master pki]# kubectl config use-context system:node:kuber-master@kubernetes \
	--kubeconfig=../kubelet.conf
	
ps: 需要根据实际情况将system:node:kuber-master 中的kuber-master进行替换。

2.9 Service account key

Service account 不是通过 CA 进行认证，因此不要通过 CA 来做 Service account key 的检查，这边建立一组 Private 与 Public 密钥提供给 Service account key 使用：

	[root@kuber-master pki]# openssl genrsa -out sa.key 2048
	[root@kuber-master pki]# openssl rsa -in sa.key -pubout -out sa.pub
	
确认/etc/kubernetes与/etc/kubernetes/pki有以下文件：

	[root@kuber-master pki]# ls -l 
	total 140
	-rw-r--r-- 1 root root 1025 Jan  8 15:45 admin.csr
	-rw-r--r-- 1 root root  176 Jan  8 15:44 admin-csr.json
	-rw------- 1 root root 1679 Jan  8 15:45 admin-key.pem
	-rw-r--r-- 1 root root 1440 Jan  8 15:45 admin.pem
	-rw-r--r-- 1 root root 1029 Jan  8 15:19 apiserver.csr
	-rw-r--r-- 1 root root  181 Jan  8 15:17 apiserver-csr.json
	-rw------- 1 root root 1675 Jan  8 15:19 apiserver-key.pem
	-rw-r--r-- 1 root root 1533 Jan  8 15:19 apiserver.pem
	-rw-r--r-- 1 root root  278 Jan  8 14:59 ca-config.json
	-rw-r--r-- 1 root root 1025 Jan  8 15:00 ca.csr
	-rw-r--r-- 1 root root  177 Jan  8 14:59 ca-csr.json
	-rw------- 1 root root 1675 Jan  8 15:00 ca-key.pem
	-rw-r--r-- 1 root root 1407 Jan  8 15:00 ca.pem
	-rw-r--r-- 1 root root 1082 Jan  8 15:57 controller-manager.csr
	-rw------- 1 root root 1679 Jan  8 15:57 controller-manager-key.pem
	-rw-r--r-- 1 root root 1497 Jan  8 15:57 controller-manager.pem
	-rw-r--r-- 1 root root  891 Jan  8 15:25 front-proxy-ca.csr
	-rw-r--r-- 1 root root   66 Jan  8 15:24 front-proxy-ca-csr.json
	-rw------- 1 root root 1679 Jan  8 15:25 front-proxy-ca-key.pem
	-rw-r--r-- 1 root root 1143 Jan  8 15:25 front-proxy-ca.pem
	-rw-r--r-- 1 root root  903 Jan  8 15:29 front-proxy-client.csr
	-rw-r--r-- 1 root root   74 Jan  8 15:27 front-proxy-client-csr.json
	-rw------- 1 root root 1671 Jan  8 15:29 front-proxy-client-key.pem
	-rw-r--r-- 1 root root 1188 Jan  8 15:29 front-proxy-client.pem
	-rw-r--r-- 1 root root 1050 Jan  8 16:30 kubelet.csr
	-rw-r--r-- 1 root root  193 Jan  8 16:26 kubelet-csr.json
	-rw------- 1 root root 1679 Jan  8 16:30 kubelet-key.pem
	-rw-r--r-- 1 root root 1509 Jan  8 16:30 kubelet.pem
	-rw-r--r-- 1 root root  217 Jan  8 15:56 manager-csr.json
	-rw-r--r-- 1 root root 1679 Jan  8 16:40 sa.key
	-rw-r--r-- 1 root root  451 Jan  8 16:40 sa.pub
	-rw-r--r-- 1 root root 1058 Jan  8 16:14 scheduler.csr
	-rw-r--r-- 1 root root  199 Jan  8 16:12 scheduler-csr.json
	-rw------- 1 root root 1679 Jan  8 16:14 scheduler-key.pem
	-rw-r--r-- 1 root root 1472 Jan  8 16:14 scheduler.pem
	
### 安装 Kubernetes 核心组件

1 首先复制 Kubernetes 核心组件 YAML 文件，这边我们不透过 Binary 方案来创建 Master 核心组件，而是利用 Kubernetes Static Pod 来创建，因此需下载所有核心组件的Static Pod文件到/etc/kubernetes/manifests目录：

	[root@kuber-master pki]# mkdir -p /etc/kubernetes/manifests && cd /etc/kubernetes/manifests
	[root@kuber-master manifests]# for i in apiserver manager scheduler; do cp $K8S_CONFIG_FILES/master/${i}.yml ./;done
	
2 修改/etc/kubernetes/manifests/apiserver.yml配置文件，修改信息如下：

	1.--advertise-address，ip需要根据实际情况做改动
	2. --etcd-servers，etcd每个节点的访问url都应写入

3 生成一个用来加密 Etcd 的 Key：

	[root@kuber-master manifests]# head -c 32 /dev/urandom | base64
	mwYib16D3v9fpBqOstsfHlgpniZzPjWvPcRGz2iGDtk=
	
4 在/etc/kubernetes/目录下，创建encryption.yml的加密 YAML 文件：

	[root@kuber-master manifests]# cp  $K8S_CONFIG_FILES/master/encryption.yml  /etc/kubernetes
	[root@kuber-master manifests]# sed -i 's@secret:\(.*\)@secret: mwYib16D3v9fpBqOstsfHlgpniZzPjWvPcRGz2iGDtk=@g' /etc/kubernetes/encryption.yml
	
5 在/etc/kubernetes/目录下，创建audit-policy.yml的进阶审核策略 YAML 文件：

	[root@kuber-master manifests]# cp  $K8S_CONFIG_FILES/master/audit-policy.yml  /etc/kubernetes
	
6 下载kubelet.service相关文件来管理 kubelet：

	[root@kuber-master manifests]# mkdir -p /etc/systemd/system/kubelet.service.d
	[root@kuber-master manifests]# cp $K8S_CONFIG_FILES/master/10-kubelet.conf  /etc/systemd/system/kubelet.service.d/
	[root@kuber-master manifests]# cp $K8S_CONFIG_FILES/master/kubelet.service /lib/systemd/system/
	
7 修改配置文件/etc/systemd/system/kubelet.service.d/10-kubelet.conf，修改信息如下：

	1.KUBELET_POD_CONTAINER,如果拉取不到该容器，可以使用如下的根容器：
	registry.access.redhat.com/rhel7/pod-infrastructure:latest
	
	2.由于我这的docker版本的cgroup-driver采用的是systemd，需要显式指定(我这已经加上了，原文没加可能是docker版本不同)，即添加--cgroup-driver=systemd，否则启动服务的时候可能会报下面的错误：
	 kubelet[13924]: error: failed to run Kubelet: failed to create kubelet: misconfiguration: kubelet cgroup driver: "cgroupfs" is different from docker cgroup driver: "systemd"
	 
8 最后创建 var 存放信息，然后启动 kubelet 服务:

	[root@kuber-master manifests]# mkdir -p /var/lib/kubelet /var/log/kubernetes
	[root@kuber-master manifests]#  systemctl enable kubelet.service && systemctl start kubelet.service
	
9 其他说明：
	
	1.如果使用systemctl status kubelet,看见如下信息，不要惊慌，因为apiserver还没启动：
	Failed to list *v1.Pod: Get https://10.61.0.160:6443/api/v1/pods?..
	2.查看监听的端口是否在监听：
	[root@kuber-master manifests]# ss -tunlp | grep kube
	tcp    LISTEN     0      128    127.0.0.1:10248                 *:*                   users:(("kubelet",pid=6216,fd=20))
	tcp    LISTEN     0      128    127.0.0.1:10251                 *:*                   users:(("kube-scheduler",pid=6703,fd=3))
	tcp    LISTEN     0      128    127.0.0.1:10252                 *:*                   users:(("kube-controller",pid=6815,fd=3))
	tcp    LISTEN     0      128      :::10250                :::*                   users:(("kubelet",pid=6216,fd=23))
	tcp    LISTEN     0      128      :::6443                 :::*                   users:(("kube-apiserver",pid=6893,fd=164))
	tcp    LISTEN     0      128      :::10255                :::*                   users:(("kubelet",pid=6216,fd=21))
	tcp    LISTEN     0      128      :::4194                 :::*                   users:(("kubelet",pid=6216,fd=8))
	
10 完成后，复制 admin kubeconfig 文件，并通过简单指令验证：

	[root@kuber-master manifests]# cp /etc/kubernetes/admin.conf ~/.kube/config
	[root@kuber-master manifests]# kubectl get cs
	NAME                 STATUS    MESSAGE              ERROR
	controller-manager   Healthy   ok                   
	scheduler            Healthy   ok                   
	etcd-0               Healthy   {"health": "true"}   
	etcd-2               Healthy   {"health": "true"}   
	etcd-1               Healthy   {"health": "true"}   
	
	[root@kuber-master manifests]# kubectl get nodes
	NAME           STATUS    ROLES     AGE       VERSION
	kuber-master   Ready     master    6m        v1.8.6
	
	[root@kuber-master manifests]# kubectl -n kube-system get po
	NAME                                   READY     STATUS    RESTARTS   AGE
	kube-apiserver-kuber-master            1/1       Running   0          5m
	kube-controller-manager-kuber-master   1/1       Running   0          5m
	kube-scheduler-kuber-master            1/1       Running   0          5m
	
确认服务能够执行 logs 等指令：

	[root@kuber-master manifests]# kubectl -n kube-system logs -f kube-scheduler-kuber-master
	Error from server (Forbidden): Forbidden (user=kube-apiserver, verb=get, resource=nodes, subresource=proxy) ( pods/log kube-scheduler-kuber-master)
	
ps: 这边会发现出现 403 Forbidden 问题，这是因为 kube-apiserver user 并没有 nodes 的资源权限，属于正常。

由于上述权限问题，我们必需创建一个 apiserver-to-kubelet-rbac.yml 来定义权限，以供我们执行 logs、exec 等指令：

	[root@kuber-master manifests]# cd /etc/kubernetes/
	[root@kuber-master kubernetes]# cp $K8S_CONFIG_FILES/master/apiserver-to-kubelet-rbac.yml  ./
	[root@kuber-master kubernetes]# kubectl apply -f apiserver-to-kubelet-rbac.yml 
	clusterrole "system:kube-apiserver-to-kubelet" created
	clusterrolebinding "system:kube-apiserver" created
	[root@kuber-master kubernetes]# kubectl -n kube-system logs -f kube-scheduler-kuber-master
	
### Kubernetes Node

Node 是主要执行容器实例的节点，可视为工作节点。在这步骤我们会下载 Kubernetes binary 文件，并创建 node 的 certificate 来提供给节点注册认证用。Kubernetes 使用Node Authorizer来提供Authorization mode，这种授权模式会替 Kubelet 生成 API request。

在开始前，我们先在master1将需要的 ca 与 cert,以及二进制文件复制到 Node 节点上：

	[root@kuber-master kubernetes]# cat   /tmp/copy-kube.sh
	#!/bin/bash
	for NODE in kuber-node1 kuber-node2 kuber-node3 kuber-node4; do
		ssh ${NODE} "mkdir -p /etc/kubernetes/pki/"
		ssh ${NODE} "mkdir -p /etc/kubernetes/manifests"
		scp /usr/local/bin/kubelet ${NODE}:/usr/local/bin 
		for FILE in pki/ca.pem pki/ca-key.pem bootstrap.conf; do
		  scp /etc/kubernetes/${FILE} ${NODE}:/etc/kubernetes/${FILE}
		done
	done
	
1 设置node节点的配置文件,使用脚本将配置文件传到各个node节点上：

	[root@kuber-master kubernetes]# cat  /tmp/copy-kube-conf.sh
	#!/bin/bash
	for NODE in kuber-node1 kuber-node2 kuber-node3 kuber-node4; do
		ssh ${NODE} "mkdir -p  /etc/systemd/system/kubelet.service.d"
		scp $K8S_CONFIG_FILES/node/10-kubelet.conf ${NODE}:/etc/systemd/system/kubelet.service.d
		scp $K8S_CONFIG_FILES/node/kubelet.service  ${NODE}:/lib/systemd/system
	done
2 授权 Kubernetes Node

当所有节点都完成后，在master节点，因为我们采用 TLS Bootstrapping，所需要创建一个 ClusterRoleBinding：

	[root@kuber-master kubernetes]# kubectl create clusterrolebinding kubelet-bootstrap \
	--clusterrole=system:node-bootstrapper \
	--user=kubelet-bootstrap
	
3 启动各节点的kubelet服务，使用如下脚本进行：

	[root@kuber-master kubernetes]# cat  /tmp/copy-kube-start.sh
	#!/bin/bash
	for NODE in kuber-node1 kuber-node2 kuber-node3 kuber-node4; do
		ssh ${NODE} "systemctl start kubelet"
	done
	
ps: 原文中是先启动各个节点的服务再授权，我经过测试，发现如果在未授权的情况下,各节点的kubelet服务根本无法启动，错误报告如下：

	error: failed to run Kubelet: cannot create certificate signing request: certificatesigningrequests.certificates.k8s.io is forbidden: 
	
只有先授权，才能启动各节点的kubelet服务。

4.检查节点是否成功加入

在kuber-master通过简单指令验证，会看到节点处于pending：

	[root@kuber-master kubernetes]# kubectl get csr
	NAME                                                   AGE       REQUESTOR           CONDITION
	node-csr-9ocxrCiSF6hjpIWXjB2qibfprN6i2YYcE6ttnL2lXKM   2m        kubelet-bootstrap   Pending
	node-csr-IhB-ttp82G_z5Sgh2oyCs1ygRjgj52X2qoOBVJmFpIE   9s        kubelet-bootstrap   Pending
	node-csr-lsrD6n3t5gboFimV28ayVZdzPf_eoV702oMQTreRKUQ   1m        kubelet-bootstrap   Pending
	node-csr-nm4TaOggBqgaWg98WOkaBh44X-Hn-N0AhWUr_VUQtBQ   51s       kubelet-bootstrap   Pending
	
通过 kubectl 来允许节点加入集群：

	[root@kuber-master kubernetes]# kubectl get csr | awk '/Pending/ {print $1}' | xargs kubectl certificate approve
	
	[root@kuber-master kubernetes]# kubectl get csr
	NAME                                                   AGE       REQUESTOR           CONDITION
	node-csr-9ocxrCiSF6hjpIWXjB2qibfprN6i2YYcE6ttnL2lXKM   3m        kubelet-bootstrap   Approved,Issued
	node-csr-IhB-ttp82G_z5Sgh2oyCs1ygRjgj52X2qoOBVJmFpIE   1m        kubelet-bootstrap   Approved,Issued
	node-csr-lsrD6n3t5gboFimV28ayVZdzPf_eoV702oMQTreRKUQ   2m        kubelet-bootstrap   Approved,Issued
	node-csr-nm4TaOggBqgaWg98WOkaBh44X-Hn-N0AhWUr_VUQtBQ   1m        kubelet-bootstrap   Approved,Issued
	
	[root@kuber-master kubernetes]# kubectl get nodes
	NAME           STATUS    ROLES     AGE       VERSION
	kuber-master   Ready     master    50m       v1.8.6
	kuber-node1    Ready     node      51s       v1.8.6
	kuber-node2    Ready     node      51s       v1.8.6
	kuber-node3    Ready     node      51s       v1.8.6
	kuber-node4    Ready     node      51s       v1.8.6

ps: 如果节点加入不成功需要重新加入，得先把/var/lib/kubeletes/下的文件删除，然后重启服务即可。

### Kubernetes Core Addons 部署

当完成上面所有步骤后，接着我们需要安装一些插件，而这些有部分是非常重要跟好用的，如Kube-dns与Kube-proxy等。

**1 Kube-proxy addon**

Kube-proxy 是实现 Service 的关键组件，kube-proxy 会在每台节点上执行，然后监听 API Server 的 Service 与 Endpoint 资源对象的改变，然后来依据变化执行 iptables 来实现网络的转发。这边我们会需要建议一个 DaemonSet 来执行，并且创建一些需要的 certificate。Kubernetes 1.8 kube-proxy 开启 ipvs

首先复制ube-proxy-csr.json文件，并产生 kube-proxy certificate 证书：

	[root@kuber-master pki]# cd /etc/kubernetes/pki
	[root@kuber-master pki]# cp $K8S_CONFIG_FILES/pki/kube-proxy-csr.json ./
	[root@kuber-master pki]# cfssl gencert \
	-ca=ca.pem \
	-ca-key=ca-key.pem \
	-config=ca-config.json \
	-profile=kubernetes \
	kube-proxy-csr.json | cfssljson -bare kube-proxy

接着透过以下指令生成名称为 kube-proxy.conf 的 kubeconfig 文件：

	# kube-proxy set-cluster
	[root@kuber-master pki]# kubectl config set-cluster kubernetes \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=../kube-proxy.conf

	# kube-proxy set-credentials
	[root@kuber-master pki]# kubectl config set-credentials system:kube-proxy \
	--client-key=kube-proxy-key.pem \
	--client-certificate=kube-proxy.pem \
	--embed-certs=true \
	--kubeconfig=../kube-proxy.conf
	
	# kube-proxy set-context
	[root@kuber-master pki]# kubectl config set-context system:kube-proxy@kubernetes \
	--cluster=kubernetes \
	--user=system:kube-proxy \
	--kubeconfig=../kube-proxy.conf
	
	# kube-proxy set default context
	[root@kuber-master pki]# kubectl config use-context system:kube-proxy@kubernetes \
	--kubeconfig=../kube-proxy.conf
	
在kuber-master将kube-proxy相关文件复制到 Node 节点上：

	[root@kuber-master pki]# cat  /tmp/copy-kuber-proxy.sh
	#!/bin/bash
	for NODE in kuber-node1 kuber-node2 kuber-node3 kuber-node4; do
		for FILE in pki/kube-proxy.pem pki/kube-proxy-key.pem kube-proxy.conf; do
			scp /etc/kubernetes/${FILE} ${NODE}:/etc/kubernetes/${FILE}
		done
	done
	
完成后，在kuber-master通过 kubectl 来创建 kube-proxy daemon：

	[root@kuber-master pki]# mkdir -p /etc/kubernetes/addons && cd /etc/kubernetes/addons
	[root@kuber-master addons]# cp $K8S_CONFIG_FILES/addons/kube-proxy.yml  ./ 
	[root@kuber-master addons]# kubectl apply -f kube-proxy.yml
	[root@kuber-master addons]# kubectl -n kube-system get po -l k8s-app=kube-proxy
	NAME               READY     STATUS    RESTARTS   AGE
	kube-proxy-gkfgc   1/1       Running   0          30s
	kube-proxy-hwr9b   1/1       Running   0          30s
	kube-proxy-j8hs9   1/1       Running   0          30s
	kube-proxy-ksfsw   1/1       Running   0          30s
	kube-proxy-p85x8   1/1       Running   0          30s
	
**2 Calico policy controller**

Calico 是一款纯 Layer 3 的数据中心网络方案(不需要 Overlay 网络)，Calico 好处是他已与各种云原生平台有良好的整合，而 Calico 在每一个节点利用 Linux Kernel 实现高效的 vRouter 来负责数据的转发，而当数据中心复杂度增加时，可以用 BGP route reflector 来达成。

前面我们把calico配置好了，我们还差一步就是启动Calico policy controller， 原文把这一步放在kube-dns后面，导致kube-dns容器三个只能启动两个，然后出错。

在kuber-master通过 kubectl 建立 Calico policy controller：

	[root@kuber-master addons]# cp $K8S_CONFIG_FILES/addons/calico-controller.yml  ./
	[root@kuber-master addons]# kubectl apply -f calico-controller.yml
	[root@kuber-master addons]# kubectl -n kube-system get po -l k8s-app=calico-policy
	NAME                                        READY     STATUS    RESTARTS   AGE
	calico-policy-controller-66f794cd67-mbjzw   1/1       Running   0          1m

ps: calico-controller.yml需要修改的地方是ETCD_ENDPOINTS的value值。

**3 Kube-dns addon**

Kube DNS 是 Kubernetes 集群内部 Pod 之间互相沟通的重要 Addon，它允许 Pod 可以通过 Domain Name 方式来连接 Service，其主要由 Kube DNS 与 Sky DNS 组合而成，通过 Kube DNS 监听 Service 与 Endpoint 变化，来提供给 Sky DNS 信息，已更新解析地址。

安装只需要在kuber-master通过 kubectl来创建 kube-dns deployment 即可：

	[root@kuber-master addons]# cp $K8S_CONFIG_FILES/addons/kube-dns.yml  ./
	[root@kuber-master addons]#  kubectl apply -f kube-dns.yml
	[root@kuber-master addons]# kubectl -n kube-system get po -l k8s-app=kube-dns
	NAME                        READY     STATUS    RESTARTS   AGE
	kube-dns-6cb549f55f-xrbht   3/3       Running   3          11m

如果其他服务一切正常，但是kube-dns只是启动了两个容器，另一个无法启动，可以把所有节点的iptables规则清空(iptables -F -t nat)，再重启kubelet服务。

### Kubernetes Extra Addons 部署

本节说明如何部署一些官方常用的 Addons，如 Dashboard、Heapster 等。

**1 Dashboard addon**

Dashboard 是 Kubernetes 社区官方开发的仪表板，有了仪表板后管理者就能够透过 Web-based 方式来管理 Kubernetes 集群，除了提升管理方便，也让资源可视化，让人更直觉看见系统信息的呈现结果。

首先我们要建立kubernetes-dashboard-certs，来提供给 Dashboard TLS 使用：

	[root@kuber-master addons]# mkdir -p /etc/kubernetes/addons/certs && cd /etc/kubernetes/addons 
	[root@kuber-master addons]# openssl genrsa -des3 -passout pass:x -out certs/dashboard.pass.key 2048
	[root@kuber-master addons]# openssl rsa -passin pass:x -in certs/dashboard.pass.key -out certs/dashboard.key
	[root@kuber-master addons]# openssl req -new -key certs/dashboard.key \
	-out certs/dashboard.csr -subj '/CN=kube-dashboard'
	[root@kuber-master addons]# rm certs/dashboard.pass.key
	[root@kuber-master addons]# kubectl create secret generic kubernetes-dashboard-certs \
	--from-file=certs -n kube-system
	[root@kuber-master addons]# cp $K8S_CONFIG_FILES/addons/kube-dashboard.yml  ./
	[root@kuber-master addons]# kubectl apply -f kube-dashboard.yml
	[root@kuber-master addons]# kubectl -n kube-system get po,svc -l k8s-app=kubernetes-dashboard
	NAME                                      READY     STATUS    RESTARTS   AGE
	po/kubernetes-dashboard-747c4f7cf-d4msq   1/1       Running   0          6m

	NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
	svc/kubernetes-dashboard   ClusterIP   10.107.162.31   <none>        443/TCP   6m
	
P.S. 这边会额外创建一个名称为anonymous-open-door Cluster Role Binding，这仅作为方便测试时使用，在一般情况下不要开启，不然就会直接被存取所有 API。
	
	完成后，就可以透过浏览器访问 Dashboard，https://10.61.0.160:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

在 1.7 版本以后的 Dashboard 将不再提供所有权限，因此需要建立一个 service account 来绑定 cluster-admin role：

	[root@kuber-master addons]# kubectl -n kube-system create sa dashboard
	
	[root@kuber-master addons]# kubectl create clusterrolebinding dashboard --clusterrole cluster-admin --serviceaccount=kube-system:dashboard
	[root@kuber-master addons]# kubectl -n kube-system get sa dashboard -o yaml
	apiVersion: v1
	kind: ServiceAccount
	metadata:
	  creationTimestamp: 2018-01-08T13:58:10Z
	  name: dashboard
	  namespace: kube-system
	  resourceVersion: "25035"
	  selfLink: /api/v1/namespaces/kube-system/serviceaccounts/dashboard
	  uid: f65b7438-f47b-11e7-af70-0cc47a130e4c
	secrets:
	- name: dashboard-token-vsvkl

 	[root@kuber-master addons]# kubectl -n kube-system describe secrets dashboard-token-vsvkl
	
把token贴到Kubernetes dashboard。

### Heapster addon

Heapster 是 Kubernetes 社区维护的容器集群监控分析工具。Heapster 会从 Kubernetes apiserver 获得所有 Node 信息，然后再通过这些 Node 来获得 kubelet 上的数据，最后再将所有收集到数据送到 Heapster 的后台储存 InfluxDB，最后利用 Grafana 来抓取 InfluxDB 的数据源来进行可视化。

在kuber-master通过 kubectl 来创建 kubernetes monitor 即可：

	[root@kuber-master addons]# cp $K8S_CONFIG_FILES/addons/kube-monitor.yml .
	[root@kuber-master addons]# kubectl apply -f kube-monitor.yml
	
完成后，就可以透过浏览器存取 Grafana Dashboard，https://10.61.0.160:6443/api/v1/proxy/namespaces/kube-system/services/monitoring-grafana

### 简单部署 Nginx 服务

Kubernetes 可以选择使用指令直接创建应用程序与服务，或者撰写 YAML 与 JSON 档案来描述部署应用程序的配置，以下将创建一个简单的 Nginx 服务：

	[root@kuber-master addons]# kubectl run nginx --image=nginx --port=80
	[root@kuber-master addons]# kubectl expose deploy nginx --port=80 --type=LoadBalancer --external-ip=10.61.0.160
	[root@kuber-master addons]# kubectl get svc,po
	NAME             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
	svc/kubernetes   ClusterIP      10.96.0.1        <none>        443/TCP        4h
	svc/nginx        LoadBalancer   10.103.165.116   10.61.0.160   80:30367/TCP   1m

	NAME                        READY     STATUS    RESTARTS   AGE
	po/nginx-7cbc4b4d9c-h8gqn   1/1       Running   0          1m

