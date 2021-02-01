# 安装k8s

## 前期准备

### step 1 关闭selinux
	sed -i 's/SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
	reboot
  
### step 2 检查系统内核版本,要求3.8以上
	uanme -a
  
### step 3 关闭防火墙
	systemctl stop firewalld
	systemctl disable firewalld
### step 4 安装yum源,并安装必要的软件
	curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
	yum install wget net-tools telnet tree nmap sysstat lrzsz dos2unix bind-utils -y
	yum install supervisor -y
启动supervisor,用还监控服务,挂掉后自动拉起来
	
	systemctl start  supervisord
	systemctl enable supervisord
## 在dns服务器上安装dns服务
  安装bind9
### step 1
  	yum install bind -y
### step 2 修改配置文件
  	sed -i 's/listen-on port 53.*/listen-on port 53 { 0.0.0.0; };/g' /etc/named.conf
  	sed -i 's/allow-query.*/allow-query     { any; };/g' /etc/named.conf
  	sed -i "/allow-query/a\\\tforwarders\\t{ 10.4.7.254; };" /etc/named.conf   #10.4.7.254 为上级网关地址
  	sed -i 's/dnssec-enable.*/dnssec-enable no;/g' /etc/named.conf
  	sed -i 's/dnssec-validation.*/dnssec-validation no;/g' /etc/named.conf
  	named-checkconf   #检测配置
### step 3添加动态域
在/etc/named.rfc1912.zones 中添加

	zone "host.com" IN {
        type master;
        file "host.com.zone";
        allow-update { 10.4.7.11; };
	};
	zone "od.com" IN {
        type master;
        file "od.com.zone";
        allow-update { 10.4.7.11; };
	};
### step 4添加/var/named/host.com.zone
	
	$ORIGIN	host.com.
	$TTL	600; 10 minutes
	@	IN	SOA	dns.host.com. dnsadmin.host.com. (
					2021012801	; serial
					10800		; refresh (3 hours)
					900			; retry (15 minutes)
					604800		; expire (1 week)
					86400		; minimum (1 day)
					)
			NS	dns.host.com.
	$TTL	60 ; 1 minute
	dns	A	10.4.7.11
	HDSS7-11	A	10.4.7.11
	HDSS7-12	A	10.4.7.12
	HDSS7-21	A	10.4.7.21
	HDSS7-22	A	10.4.7.22
	HDSS7-200	A	10.4.7.200


### step 5 vi /var/named/od.com.zone
	$ORIGIN od.com.
	$TTL 600  ; 10 minutes
	@	IN	SOA	dns.od.com. dnsadmin.od.com. (
					2021012801      ; serial
					10800           ; refresh (3 hours)
					900             ; retry (15 minutes)
					604800          ; expire (1 week)
					86400           ; minimum (1 day)
					)
			NS	dns.od.com.
	$TTL	60	; 1 minute
	dns	A	10.4.7.11

### step 6 启动服务
	systemctl start named
	netstat -anp|grep 53 #检查服务
	dig -t A hdss7-21.host.com @10.4.7.11 +short
	sed -i 's/nameserver.*/nameserver 10.4.7.11/' /etc/resolv.conf  #更改dns server

## 准备证书签发环境

### step 1 安装cfssl
	curl -s -L -o /bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
	curl -s -L -o /bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
	curl -s -L -o /bin/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
	chmod +x /bin/cfssl*
### 集群相关证书类型
**client certificate**： 用于服务端认证客户端,例如etcdctl、etcd proxy、fleetctl、docker客户端

**server certificate**: 服务端使用，客户端以此验证服务端身份,例如docker服务端、kube-apiserver

**peer certificate**: 双向证书，用于etcd集群成员间通信

根据认证对象可以将证书分成三类：服务器证书<u>server cert</u>，客户端证书<u>client cert</u>，对等证书<u>peer cert</u>(表示既是server cert又是client cert)，在kubernetes 集群中需要的证书种类如下：

* etcd 节点需要标识自己服务的server cert，也需要client cert与etcd集群其他节点交互，当然可以分别指定2个证书，也可以使用一个对等证书
* master 节点需要标识 apiserver服务的server cert，也需要client cert连接etcd集群，这里也使用一个对等证书
* kubectl calico kube-proxy 只需要client cert，因此证书请求中 hosts 字段可以为空
* kubelet证书比较特殊，不是手动生成，它由node节点TLS BootStrap向apiserver请求，由master节点的controller-manager 自动签发，包含一个client cert 和一个server cert

### step 2 创建CA证书签名请求


**配置证书生成策略，规定CA可以颁发那种类型的证书**/opt/certs/ca-config.json

	{
		"signing": {
			"default": {
				"expiry": "175200h"
			},
			"profiles": {
				"server": {
					"expiry": "175200h",
					"usages": [
						"signing",
						"key encipherment",
						"server auth"
					]
				},
				"client":{
					"expiry": "175200h",
					"usages": [
						"signing",
						"key encipherment",
						"client auth"
					]
				},
				"peer":{
					"expiry": "175200h",
					"usages": [
						"signing",
						"key encipherment",
						"server auth",
						"client auth"
					]
				}
			}
		}
	}


**编辑配置文件**/opt/certs/ca-csr.json

	{
		"CN": "ming",
		"hosts":[
		],
		"key": {
		    "algo": "rsa",
		    "size": 2048
		},
		"names": [
		    {
		        "C": "CN",
		        "L": "nanji",
		        "ST": "nanji",  
				"O": "jszh",          
		        "OU": "zcb"
		    }    
		],
		"ca" : {
			"expiry": "175200h"
		}
	}
**生成ca证书**

	cd /opt/certs/
	cfssl gencert -initca ca-csr.json |cfssljson -bare ca

## 部署docker环境
在HDSS7-200/21/22.host.com上部署docker

### step 1安装
	curl -fsSL https://get.docker.com |bash -s docker --mirror Aliyun

### step 2 配置文件
	mkdir /etc/docker
	vi /etc/docker/daemon.json
配置文件如下,注意根据需求修改bip

	{
		"graph":"/data/docker",
		"storage-driver":"overlay2",
		"insecure-registries": [],
		"registry-mirrors": ["https://bv3z4ezp.mirror.aliyuncs.com"],
		"bip":"172.7.21.1/24",
		"exec-opts":["native.cgroupdriver=systemd"],
		"live-restore":true
	}
启动docker
	
	mkdir /data/docker -p 
	systemctl start docker
	systemctl enable docker
	docker info

## 部署etcd集群
安装 hdss7-12/21/22.host.com 3台

### step1 签发证书
**创建etcd证书请求文件**
在hdss7-11.host.com上执行

	cd /opt/certs/
	vi etcd-peer-csr.json
内容为:

	{
	    "CN": "k8s-etcd",
	    "hosts":[
			"10.4.7.11",
			"10.4.7.12",
			"10.4.7.21",
			"10.4.7.22"
	    ],
	    "key": {
	        "algo": "rsa",
	        "size": 2048
	    },
	    "names": [
	        {
	            "C": "CN",
	            "L": "nanji",
	            "ST": "nanji",  
	            "O": "jszh",          
	            "OU": "zcb"
	        }    
	    ]
	}

**生成证书**

	cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer etcd-peer-csr.json |cfssljson -bare etcd-peer

### step2 安装etcd

**添加etcd用户**

	useradd  -s /sbin/nologin -M etcd

**安装etcd**

	wget https://github.com/etcd-io/etcd/releases/download/v3.1.19/etcd-v3.1.19-linux-amd64.tar.gz
	tar xvf etcd-v3.1.19-linux-amd64.tar.gz -C /opt
	cd /opt 
	mv etcd-v3.1.19-linux-amd64/ etcd-v3.1.19
	ln -s /opt/etcd-v3.1.19/ /opt/etcd
	cd /opt/etcd
	mkdir -p /opt/etcd/certs /data/etcd /data/logs/etcd-server
**上传etcd证书**
上传证书到/opt/etcd/certs目录下,

* ca.pem
* etcd-peer-key.pem
* etcd-peer.pem

### step3 启动etcd

在/opt/etcd/etcd-server-startup.sh中加入以下命令
注意**listen-peer-urls**,**listen-client-urls**,**initial-advertise-peer-urls**,**advertise-client-urls**,**initial-cluster** 字段按需要进行更改,

	#!/bin/sh
	./etcd --name etcd-server-7-12 \
	 --data-dir /data/etcd/etcd-server \
	 --ca-file ./certs/ca.pem \
	 --cert-file ./certs/etcd-peer.pem \
	 --key-file ./certs/etcd-peer-key.pem \
	 --client-cert-auth \
	 --trusted-ca-file ./certs/ca.pem \
	 --peer-ca-file ./certs/ca.pem \
	 --peer-cert-file ./certs/etcd-peer.pem \
	 --peer-key-file ./certs/etcd-peer-key.pem \
	 --peer-client-cert-auth \
	 --peer-trusted-ca-file ./certs/ca.pem \
	 --listen-peer-urls https://10.4.7.12:2380 \
	 --listen-client-urls https://10.4.7.12:2379,http://127.0.0.1:2379 \
	 --initial-advertise-peer-urls https://10.4.7.12:2380 \
	 --advertise-client-urls https://10.4.7.12:2379,http://127.0.0.1:2379 \
	 --initial-cluster etcd-server-7-12=https://10.4.7.12:2380,etcd-server-7-21=https://10.4.7.21:2380,etcd-server-7-22=https://10.4.7.22:2380 \
	 --log-output stdout
更改权限

	cd /opt/etcd
	chmod +x etcd-server-startup.sh 
	chown -R etcd:etcd /opt/etcd*
	chown -R etcd:etcd /data/etcd/
	chown -R etcd:etcd /data/logs/etcd-server -R

把etcd加入到supervisor监控
编辑/etc/supervisord.d/etcd-server.ini

	#项目名
	[program:etcd-server]
	#脚本目录
	directory=/opt/etcd
	#脚本执行命令
	command=/opt/etcd/etcd-server-startup.sh
	#进程数
	numproces=1
	#supervisor启动的时候是否随着同时启动，默认True
	autostart=true
	#当程序exit的时候，这个program不会自动重启,默认unexpected，设置子进程挂掉后自动重启的情况，有三个选项，false,unexpected和true。如果为false的时候，无论什么情况下，都不会被重新启动，如果为unexpected，只有当进程的退出码不在下面的exitcodes里面定义的
	autorestart=true
	#这个选项是子进程启动多少秒之后，此时状态如果是running，则我们认为启动成功了。默认值为1
	startsecs=30

	startretries =3
	
	#脚本运行的用户身份 
	user = etcd
	
	#日志输出 
	stderr_logfile=/data/logs/etcd-server/etcd-server_err.log 
	stdout_logfile=/data/logs/etcd-server/etcd-server_stdout.log 
	#把stderr重定向到stdout，默认 false
	redirect_stderr = true
	#stdout日志文件大小，默认 50MB
	stdout_logfile_maxbytes = 20MB
	#stdout日志文件备份数
	stdout_logfile_backups = 20

启动
	
	supervisorctl update

## 安装apiserver
主机节点hdss7-21/22.host.com

### step1 下载k8s
在github上面搜索kubernetes,安装v1.20.2版本,下载地址

	#需要科学上网
	https://storage.googleapis.com/kubernetes-release/release/v1.20.2/kubernetes-server-linux-amd64.tar.gz
	#百度云
	https://pan.baidu.com/s/1cqbYRspvMlq33LqO7TnMcw
	#提取码
	ioux
### step2 安装
	#解压文件到/opt 下
	tar xvf kubernetes-server-linux-amd64.tar.gz -C /opt/
	cd /opt
	mv kubernetes kubernetes-v1.20.2
	ln -s kubernetes-v1.20.2 kubernetes

### step3 签发client 证书
此证书是用api-server和etcd通信使用的. api-server客户端,etcdserver端

	
	