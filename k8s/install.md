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
安装v1.13.5

	#需要科学上网
	https://storage.googleapis.com/kubernetes-release/release/v1.13.5/kubernetes-server-linux-amd64.tar.gz
	#百度云
	https://pan.baidu.com/s/1AbvCi5F9xBCVJBKXmK8xFA
	# 提取码
	097n
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
	mv kubernetes kubernetes-v1.13.5
	ln -s kubernetes-v1.13.5 kubernetes

### step3 签发client 证书
此证书是用api-server和etcd通信使用的. api-server客户端,etcdserver端
在hdss7-11.host.com(签发证书服务器)上编辑/opt/certs/client-csr.json

	{
	    "CN": "k8s-node",
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
	    ]
	}

生成证书

	cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client-csr.json |cfssljson -bare client

### step4 给apiserver签发证书
调用apiserver时使用的证书,
编辑/opt/certs/apiserver-csr.json

	{
	    "CN": "k8s-apiserver",
	    "hosts":[
			"127.0.0.1",
			"192.168.0.1",
			"10.4.7.1",
			"10.4.7.10",
			"10.4.7.11",
			"10.4.7.12",
			"10.4.7.21",
			"10.4.7.22",
			"10.4.7.200",
			"kubernetes.default",
			"kubernetes.default.svc",
			"kubernetes.default.svc.cluster",
			"kubernetes.default.svc.local"
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

生成证书

	cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server apiserver-csr.json |cfssljson -bare apiserver

上传证书到`/opt/kubernetes/server/bin/certs`

	mkdir -p /opt/kubernetes/server/bin/certs
* ca-key.pem
* ca.pem
* client-key.pem
* client.pem
* apiserver-key.pem
* apiserver.pem
### step5 添加配置文件 
	
	mkdir -p /opt/kubernetes/server/bin/conf 
在`/opt/kubernetes/server/bin/conf/audit.yaml`文件中添加
	
	apiVersion: audit.k8s.io/v1beta1 # This is required.
	kind: Policy
	# Don't generate audit events for all requests in RequestReceived stage.
	omitStages:
	  - "RequestReceived"
	rules:
	  # Log pod changes at RequestResponse level
	  - level: RequestResponse
	    resources:
	    - group: ""
	      # Resource "pods" doesn't match requests to any subresource of pods,
	      # which is consistent with the RBAC policy.
	      resources: ["pods"]
	  # Log "pods/log", "pods/status" at Metadata level
	  - level: Metadata
	    resources:
	    - group: ""
	      resources: ["pods/log", "pods/status"]
	
	  # Don't log requests to a configmap called "controller-leader"
	  - level: None
	    resources:
	    - group: ""
	      resources: ["configmaps"]
	      resourceNames: ["controller-leader"]
	
	  # Don't log watch requests by the "system:kube-proxy" on endpoints or services
	  - level: None
	    users: ["system:kube-proxy"]
	    verbs: ["watch"]
	    resources:
	    - group: "" # core API group
	      resources: ["endpoints", "services"]
	
	  # Don't log authenticated requests to certain non-resource URL paths.
	  - level: None
	    userGroups: ["system:authenticated"]
	    nonResourceURLs:
	    - "/api*" # Wildcard matching.
	    - "/version"
	
	  # Log the request body of configmap changes in kube-system.
	  - level: Request
	    resources:
	    - group: "" # core API group
	      resources: ["configmaps"]
	    # This rule only applies to resources in the "kube-system" namespace.
	    # The empty string "" can be used to select non-namespaced resources.
	    namespaces: ["kube-system"]
	
	  # Log configmap and secret changes in all other namespaces at the Metadata level.
	  - level: Metadata
	    resources:
	    - group: "" # core API group
	      resources: ["secrets", "configmaps"]
	
	  # Log all other resources in core and extensions at the Request level.
	  - level: Request
	    resources:
	    - group: "" # core API group
	    - group: "extensions" # Version of group should NOT be included.
	
	  # A catch-all rule to log all other requests at the Metadata level.
	  - level: Metadata
	    # Long-running requests like watches that fall under this rule will not
	    # generate an audit event in RequestReceived.
	    omitStages:
	      - "RequestReceived"

### step 6 创建启动脚本

编辑`/opt/kubernetes/server/bin/kube-apiserver.sh`

	#!/bin/bash
	./kube-apiserver \
	  --apiserver-count 2 \
	  --audit-log-path /data/logs/kubernetes/kube-apiserver/audit-log \
	  --audit-policy-file ./conf/audit.yaml \
	  --authorization-mode RBAC \
	  --client-ca-file ./certs/ca.pem \
	  --requestheader-client-ca-file ./certs/ca.pem \
	  --enable-admission-plugins NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
	  --etcd-cafile ./certs/ca.pem \
	  --etcd-certfile ./certs/client.pem \
	  --etcd-keyfile ./certs/client-key.pem \
	  --etcd-servers https://10.4.7.12:2379,https://10.4.7.21:2379,https://10.4.7.22:2379 \
	  --service-account-key-file ./certs/ca-key.pem \
	  --service-cluster-ip-range 192.168.0.0/16 \
	  --service-node-port-range 3000-29999 \
	  --kubelet-client-certificate ./certs/client.pem \
	  --kubelet-client-key ./certs/client-key.pem \
	  --log-dir  /data/logs/kubernetes/kube-apiserver \
	  --tls-cert-file ./certs/apiserver.pem \
	  --tls-private-key-file ./certs/apiserver-key.pem \
	  --v 2

创建必要的目录和权限

	mkdir /data/logs/kubernetes/kube-apiserver/audit-log -p
	chmod +x /opt/kubernetes/server/bin/kube-apiserver.sh
	cd /opt/kubernetes/server/bin

编辑`/etc/supervisord.d/kube-apiserver.ini`

	[program:kube-apiserver]
	command=/opt/kubernetes/server/bin/kube-apiserver.sh            ; the program (relative uses PATH, can take args)
	numprocs=1                                                      ; number of processes copies to start (def 1)
	directory=/opt/kubernetes/server/bin                            ; directory to cwd to before exec (def no cwd)
	autostart=true                                                  ; start at supervisord start (default: true)
	autorestart=true                                                ; retstart at unexpected quit (default: true)
	startsecs=22                                                    ; number of secs prog must stay running (def. 1)
	startretries=3                                                  ; max # of serial start failures (default 3)
	exitcodes=0,2                                                   ; 'expected' exit codes for process (default 0,2)
	stopsignal=QUIT                                                 ; signal used to kill process (default TERM)
	stopwaitsecs=10                                                 ; max num secs to wait b4 SIGKILL (default 10)
	user=root                                                       ; setuid to this UNIX account to run the program
	redirect_stderr=false                                           ; redirect proc stderr to stdout (default false)
	stdout_logfile=/data/logs/kubernetes/kube-apiserver/apiserver.stdout.log        ; stdout log path, NONE for none; default AUTO
	stdout_logfile_maxbytes=64MB                                    ; max # logfile bytes b4 rotation (default 50MB)
	stdout_logfile_backups=4                                        ; # of stdout logfile backups (default 10)
	stdout_capture_maxbytes=1MB                                     ; number of bytes in 'capturemode' (default 0)
	stdout_events_enabled=false                                     ; emit events on stdout writes (default false)
	stderr_logfile=/data/logs/kubernetes/kube-apiserver/apiserver.stderr.log        ; stderr log path, NONE for none; default AUTO
	stderr_logfile_maxbytes=64MB                                    ; max # logfile bytes b4 rotation (default 50MB)
	stderr_logfile_backups=4                                        ; # of stderr logfile backups (default 10)
	stderr_capture_maxbytes=1MB                                     ; number of bytes in 'capturemode' (default 0)
	stderr_events_enabled=false                                     ; emit events on stderr writes (default false)

启动服务

	supervisorctl update

## apiserver 高可用
hdss7-11/12.host.com
### step1 nginx配置
安装nginx

	yum install nginx -y

编辑`/etc/nginx/nginx.conf` 在最下方加入

	stream {
	    upstream kube-apiserver {
	        server 10.4.7.21:6443     max_fails=3 fail_timeout=30s;
	        server 10.4.7.22:6443     max_fails=3 fail_timeout=30s;
	    }
	    server {
	        listen 7443;
	        proxy_connect_timeout 2s;
	        proxy_timeout 900s;
	        proxy_pass kube-apiserver;
	    }
	}

启动nginx
	
	systemctl start nginx
	systemctl enable nginx

### step2 keepalived
hdss7-11/12.host.com
安装keepalived

	yum install keepalived -y
编辑`/etc/keepalived/check_port.sh`
	
	#!/bin/bash
	#keepalived 监控端口脚本
	#使用方法：
	#在keepalived的配置文件中
	#vrrp_script check_port {#创建一个vrrp_script脚本,检查配置
	#    script "/etc/keepalived/check_port.sh 6379" #配置监听的端口
	#    interval 2 #检查脚本的频率,单位（秒）
	#}
	CHK_PORT=$1
	if [ -n "$CHK_PORT" ];then
	        PORT_PROCESS=`ss -ltn|grep $CHK_PORT|wc -l`
	        if [ $PORT_PROCESS -eq 0 ];then
	                echo "Port $CHK_PORT Is Not Used,End."
	                exit 1
	        fi
	else
	        echo "Check Port Cant Be Empty!"
	fi

配置keepalived
编辑`/etc/keepalived/keepalived.conf` 
主

	! Configuration File for keepalived
	
	global_defs {
	   router_id 10.4.7.11
	
	}
	
	vrrp_script chk_nginx {
	    script "/etc/keepalived/check_port.sh 7443"
	    interval 2
	    weight -20
	}
	
	vrrp_instance VI_1 {
	    state MASTER
	    interface ens33
	    virtual_router_id 251
	    priority 100
	    advert_int 1
	    mcast_src_ip 10.4.7.11
	    nopreempt
	
	    authentication {
	        auth_type PASS
	        auth_pass 11111111
	    }
	    track_script {
	         chk_nginx
	    }
	    virtual_ipaddress {
	        10.4.7.10
	    }
	}

备

	! Configuration File for keepalived
	global_defs {
		router_id 10.4.7.12
	}
	vrrp_script chk_nginx {
		script "/etc/keepalived/check_port.sh 7443"
		interval 2
		weight -20
	}
	vrrp_instance VI_1 {
		state BACKUP
		interface ens33
		virtual_router_id 251
		mcast_src_ip 10.4.7.12
		priority 90
		advert_int 1
		authentication {
			auth_type PASS
			auth_pass 11111111
		}
		track_script {
			chk_nginx
		}
		virtual_ipaddress {
			10.4.7.10
		}
	}

启动keepalived

	chmod +x /etc/keepalived/check_port.sh
	systemctl start keepalived
	systemctl enable keepalived
	ip addr |grep 10.4.7.10
## 部署controller-manager
hdss7-21/22.host.com

### step 1 创建启动脚本

编辑`/opt/kubernetes/server/bin/kube-controller-manager.sh`

	#!/bin/sh
	./kube-controller-manager \
	  --cluster-cidr 172.7.0.0/16 \
	  --leader-elect true \
	  --log-dir /data/logs/kubernetes/kube-controller-manager \
	  --master http://127.0.0.1:8080 \
	  --service-account-private-key-file ./certs/ca-key.pem \
	  --service-cluster-ip-range 192.168.0.0/16 \
	  --root-ca-file ./certs/ca.pem \
	  --v 2

### step 2 调整文件权限，创建目录

	cd /opt/kubernetes/server/bin
	chmod +x /opt/kubernetes/server/bin/kube-controller-manager.sh
	mkdir -p /data/logs/kubernetes/kube-controller-manager

### step3 启动程序

编辑`/etc/supervisord.d/kube-conntroller-manager.ini`

	[program:kube-controller-manager]
	command=/opt/kubernetes/server/bin/kube-controller-manager.sh                     ; the program (relative uses PATH, can take args)
	numprocs=1                                                                        ; number of processes copies to start (def 1)
	directory=/opt/kubernetes/server/bin                                              ; directory to cwd to before exec (def no cwd)
	autostart=true                                                                    ; start at supervisord start (default: true)
	autorestart=true                                                                  ; retstart at unexpected quit (default: true)
	startsecs=22                                                                      ; number of secs prog must stay running (def. 1)
	startretries=3                                                                    ; max # of serial start failures (default 3)
	exitcodes=0,2                                                                     ; 'expected' exit codes for process (default 0,2)
	stopsignal=QUIT                                                                   ; signal used to kill process (default TERM)
	stopwaitsecs=10                                                                   ; max num secs to wait b4 SIGKILL (default 10)
	user=root                                                                         ; setuid to this UNIX account to run the program
	redirect_stderr=false                                                             ; redirect proc stderr to stdout (default false)
	stdout_logfile=/data/logs/kubernetes/kube-controller-manager/controll.stdout.log  ; stdout log path, NONE for none; default AUTO
	stdout_logfile_maxbytes=64MB                                                      ; max # logfile bytes b4 rotation (default 50MB)
	stdout_logfile_backups=4                                                          ; # of stdout logfile backups (default 10)
	stdout_capture_maxbytes=1MB                                                       ; number of bytes in 'capturemode' (default 0)
	stdout_events_enabled=false                                                       ; emit events on stdout writes (default false)
	stderr_logfile=/data/logs/kubernetes/kube-controller-manager/controll.stderr.log  ; stderr log path, NONE for none; default AUTO
	stderr_logfile_maxbytes=64MB                                                      ; max # logfile bytes b4 rotation (default 50MB)
	stderr_logfile_backups=4                                                          ; # of stderr logfile backups (default 10)
	stderr_capture_maxbytes=1MB                                                       ; number of bytes in 'capturemode' (default 0)
	stderr_events_enabled=false                                                       ; emit events on stderr writes (default false)

启动程序
	
	supervisorctl update

## 部署kube-scheduler
hdss7-21/22.host.com

### step 1 创建启动脚本
编辑`/opt/kubernetes/server/bin/kube-scheduler.sh`

	#!/bin/sh
	./kube-scheduler \
	  --leader-elect  \
	  --log-dir /data/logs/kubernetes/kube-scheduler \
	  --master http://127.0.0.1:8080 \
	  --v 2

### step2 调整文件权限，创建目录
	chmod +x /opt/kubernetes/server/bin/kube-scheduler.sh
	mkdir -p /data/logs/kubernetes/kube-scheduler
### step3 启动程序
编辑`/etc/supervisord.d/kube-scheduler.ini`

	[program:kube-scheduler]
	command=/opt/kubernetes/server/bin/kube-scheduler.sh                     ; the program (relative uses PATH, can take args)
	numprocs=1                                                               ; number of processes copies to start (def 1)
	directory=/opt/kubernetes/server/bin                                     ; directory to cwd to before exec (def no cwd)
	autostart=true                                                           ; start at supervisord start (default: true)
	autorestart=true                                                         ; retstart at unexpected quit (default: true)
	startsecs=22                                                             ; number of secs prog must stay running (def. 1)
	startretries=3                                                           ; max # of serial start failures (default 3)
	exitcodes=0,2                                                            ; 'expected' exit codes for process (default 0,2)
	stopsignal=QUIT                                                          ; signal used to kill process (default TERM)
	stopwaitsecs=10                                                          ; max num secs to wait b4 SIGKILL (default 10)
	user=root                                                                ; setuid to this UNIX account to run the program
	redirect_stderr=false                                                    ; redirect proc stderr to stdout (default false)
	stdout_logfile=/data/logs/kubernetes/kube-scheduler/scheduler.stdout.log ; stdout log path, NONE for none; default AUTO
	stdout_logfile_maxbytes=64MB                                             ; max # logfile bytes b4 rotation (default 50MB)
	stdout_logfile_backups=4                                                 ; # of stdout logfile backups (default 10)
	stdout_capture_maxbytes=1MB                                              ; number of bytes in 'capturemode' (default 0)
	stdout_events_enabled=false                                              ; emit events on stdout writes (default false)
	stderr_logfile=/data/logs/kubernetes/kube-scheduler/scheduler.stderr.log ; stderr log path, NONE for none; default AUTO
	stderr_logfile_maxbytes=64MB                                             ; max # logfile bytes b4 rotation (default 50MB)
	stderr_logfile_backups=4                                                 ; # of stderr logfile backups (default 10)
	stderr_capture_maxbytes=1MB                                              ; number of bytes in 'capturemode' (default 0)
	stderr_events_enabled=false                                              ; emit events on stderr writes (default false)

启动程序
	
	supervisorctl update

### 部署kubelet
hdss7-21/22.host.com

### step1 签发kubelet证书

在证书服务器hdss7-11上编辑`/opt/certs/kubelet-csr.json`

	{
	    "CN": "kubelet-node",
	    "hosts": [
	    "127.0.0.1",
	    "10.4.7.10",
	    "10.4.7.21",
	    "10.4.7.22",
	    "10.4.7.23",
	    "10.4.7.24",
	    "10.4.7.25",
	    "10.4.7.26",
	    "10.4.7.27",
	    "10.4.7.28"
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

生成kubelet证书和私钥

	cd /opt/certs
	cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server kubelet-csr.json | cfssljson -bare kubelet

把证书上传到hdss7-21/22的`/opt/kubernetes/server/bin/certs`目录下

* **kubelet-key.pem**
* **kubelet.pem**


### step2 创建配置
创建软连接

	ln -s /opt/kubernetes/server/bin/kubectl /usr/bin/kubectl

**set-cluster**
	
	mkdir /opt/kubernetes/server/bin/conf -p
	cd /opt/kubernetes/server/conf
	kubectl config set-cluster myk8s \
	  --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
	  --embed-certs=true \
	  --server=https://10.4.7.10:7443 \
	  --kubeconfig=kubelet.kubeconfig

**set-credentials**

	cd /opt/kubernetes/server/bin/conf
	kubectl config set-credentials k8s-node \
	 --client-certificate=/opt/kubernetes/server/bin/certs/client.pem \
	 --client-key=/opt/kubernetes/server/bin/certs/client-key.pem \
	 --embed-certs=true --kubeconfig=kubelet.kubeconfig 
	
**set-context**

	cd /opt/kubernetes/server/bin/conf
	kubectl config set-context myk8s-context \
	  --cluster=myk8s \
	  --user=k8s-node \
	  --kubeconfig=kubelet.kubeconfig

**use-context**

	cd /opt/kubernetes/server/bin/conf
	kubectl config use-context myk8s-context --kubeconfig=kubelet.kubeconfig
创建完成后把`/opt/kubernetes/server/conf/bin/kubelet.kubeconfig` 同步到其他运算节点

**创建资源配置文件**

编辑`/opt/kubernetes/server/bin/conf/k8s-node.yaml`

	apiVersion: rbac.authorization.k8s.io/v1
	kind: ClusterRoleBinding
	metadata:
	  name: k8s-node
	roleRef:
	  apiGroup: rbac.authorization.k8s.io
	  kind: ClusterRole
	  name: system:node
	subjects:
	- apiGroup: rbac.authorization.k8s.io
	  kind: User
	  name: k8s-node
**应用资源配置文件**

	cd /opt/kubernetes/server/bin/conf
	kubectl create -f k8s-node.yaml
	kubectl get clusterrolebinding k8s-node

**查看etcd中配置文件

	export ETCDCTL_API=3
	/opt/etcd/etcdctl get / --prefix --keys-only 

### step3 启动kubelet
准备基础镜像
**此处应该放到私有仓库**

	docker pull kubernetes/pause
**创建kubelet启动脚本**

主机hdss7-21.host.com

编辑`/opt/kubernetes/server/bin/kubelet.sh`

	#!/bin/sh
	./kubelet \
	  --anonymous-auth=false \
	  --cgroup-driver systemd \
	  --cluster-dns 192.168.0.2 \
	  --cluster-domain cluster.local \
	  --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice \
	  --fail-swap-on="false" \
	  --client-ca-file ./certs/ca.pem \
	  --tls-cert-file ./certs/kubelet.pem \
	  --tls-private-key-file ./certs/kubelet-key.pem \
	  --hostname-override 10.4.7.21 \
	  --image-gc-high-threshold 20 \
	  --image-gc-low-threshold 10 \
	  --kubeconfig ./conf/kubelet.kubeconfig \
	  --log-dir /data/logs/kubernetes/kube-kubelet \
	  --pod-infra-container-image kubernetes/pause:latest \
	  --root-dir /data/kubelet

ps.每个节点的主机配置稍有不通,注意`--hostname-override`

**启动节点**

	chmod +x /opt/kubernetes/server/bin/kubelet.sh
	mkdir -p /data/logs/kubernetes/kube-kubelet /data/kubelet

编辑`/etc/supervisord.d/kube-kubelet.ini`

	[program:kube-kubelet]
	command=/opt/kubernetes/server/bin/kubelet.sh                 ; the program (relative uses PATH, can take args)
	numprocs=1                                                        ; number of processes copies to start (def 1)
	directory=/opt/kubernetes/server/bin                              ; directory to cwd to before exec (def no cwd)
	autostart=true                                                    ; start at supervisord start (default: true)
	autorestart=true              									  ; retstart at unexpected quit (default: true)
	startsecs=22                  									  ; number of secs prog must stay running (def. 1)
	startretries=3                									  ; max # of serial start failures (default 3)
	exitcodes=0,2                 									  ; 'expected' exit codes for process (default 0,2)
	stopsignal=QUIT               									  ; signal used to kill process (default TERM)
	stopwaitsecs=10               									  ; max num secs to wait b4 SIGKILL (default 10)
	user=root                                                         ; setuid to this UNIX account to run the program
	redirect_stderr=false                                             ; redirect proc stderr to stdout (default false)
	stdout_logfile=/data/logs/kubernetes/kube-kubelet/kubelet.stdout.log   ; stdout log path, NONE for none; default AUTO
	stdout_logfile_maxbytes=64MB                                      ; max # logfile bytes b4 rotation (default 50MB)
	stdout_logfile_backups=4                                          ; # of stdout logfile backups (default 10)
	stdout_capture_maxbytes=1MB                                       ; number of bytes in 'capturemode' (default 0)
	stdout_events_enabled=false                                       ; emit events on stdout writes (default false)
	stderr_logfile=/data/logs/kubernetes/kube-kubelet/kubelet.stderr.log   ; stderr log path, NONE for none; default AUTO
	stderr_logfile_maxbytes=64MB                                      ; max # logfile bytes b4 rotation (default 50MB)
	stderr_logfile_backups=4                                          ; # of stderr logfile backups (default 10)
	stderr_capture_maxbytes=1MB   									  ; number of bytes in 'capturemode' (default 0)
	stderr_events_enabled=false   									  ; emit events on stderr writes (default false)

**启动**
	
	
	supervisorctl update

**检查节点**

	kubectl get node
**添加ROLES**
	
	kubectl label node 10.4.7.21 node-role.kubernetes.io/master=
	kubectl label node 10.4.7.21 node-role.kubernetes.io/node=
	kubectl label node 10.4.7.22 node-role.kubernetes.io/master=
	kubectl label node 10.4.7.22 node-role.kubernetes.io/node=

## 部署kube-proxy

### step 1签发证书
hdss7-11.host.com
编辑`/opt/certs/kube-proxy-csr.json`

	{
	    "CN": "system:kube-proxy",
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
	
生成证书

	cd /opt/certs
	cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client kube-proxy-csr.json | cfssljson -bare kube-proxy-client

上次证书到运算节点`/opt/kubernetes/server/bin/certs`

* kube-proxy-client-key.pem
* kube-proxy-client.pem

**安装lvs**

	yum install ipvsadm -y

### step2 创建配置

**set-cluster**

	cd /opt/kubernetes/server/bin/conf
	kubectl config set-cluster myk8s \
	  --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
	  --embed-certs=true \
	  --server=https://10.4.7.10:7443 \
	  --kubeconfig=kube-proxy.kubeconfig

**set-credentials**
	
	cd /opt/kubernetes/server/bin/conf
	kubectl config set-credentials kube-proxy \
	  --client-certificate=/opt/kubernetes/server/bin/certs/kube-proxy-client.pem \
	  --client-key=/opt/kubernetes/server/bin/certs/kube-proxy-client-key.pem \
	  --embed-certs=true \
	  --kubeconfig=kube-proxy.kubeconfig
	
**set-context**
		
	cd /opt/kubernetes/server/bin/conf
	kubectl config set-context myk8s-context \
	  --cluster=myk8s \
	  --user=kube-proxy \
	  --kubeconfig=kube-proxy.kubeconfig

**use-context**
	
	cd /opt/kubernetes/server/bin/conf
	kubectl config use-context myk8s-context --kubeconfig=kube-proxy.kubeconfig

ps.把kube-proxy.kubeconfig 上传至其他节点`/opt/kubernetes/server/bin/conf`下


### step 3 创建kube-proxy启动脚本

编辑 `/opt/kubernetes/server/bin/kube-proxy.sh`

	#!/bin/sh
	./kube-proxy \
	  --cluster-cidr 172.7.0.0/16 \
	  --hostname-override 10.4.7.21 \
	  --proxy-mode=ipvs \
	  --ipvs-scheduler=nq \
	  --kubeconfig ./conf/kube-proxy.kubeconfig

ps. 其他节点做响应修改

	chmod +x /opt/kubernetes/server/bin/kube-proxy.sh
	mkdir -p /data/logs/kubernetes/kube-proxy

编辑`/etc/supervisord.d/kube-proxy.ini`


	[program:kube-proxy]
	command=/opt/kubernetes/server/bin/kube-proxy.sh                 ; the program (relative uses PATH, can take args)
	numprocs=1                                                           ; number of processes copies to start (def 1)
	directory=/opt/kubernetes/server/bin                                 ; directory to cwd to before exec (def no cwd)
	autostart=true                                                       ; start at supervisord start (default: true)
	autorestart=true                                                     ; retstart at unexpected quit (default: true)
	startsecs=22                                                         ; number of secs prog must stay running (def. 1)
	startretries=3                                                       ; max # of serial start failures (default 3)
	exitcodes=0,2                                                        ; 'expected' exit codes for process (default 0,2)
	stopsignal=QUIT                                                      ; signal used to kill process (default TERM)
	stopwaitsecs=10                                                      ; max num secs to wait b4 SIGKILL (default 10)
	user=root                                                		         ; setuid to this UNIX account to run the program
	redirect_stderr=false                                           		 ; redirect proc stderr to stdout (default false)
	stdout_logfile=/data/logs/kubernetes/kube-proxy/proxy.stdout.log     ; stdout log path, NONE for none; default AUTO
	stdout_logfile_maxbytes=64MB                                    		 ; max # logfile bytes b4 rotation (default 50MB)
	stdout_logfile_backups=4                                        		 ; # of stdout logfile backups (default 10)
	stdout_capture_maxbytes=1MB                                     		 ; number of bytes in 'capturemode' (default 0)
	stdout_events_enabled=false                                     		 ; emit events on stdout writes (default false)
	stderr_logfile=/data/logs/kubernetes/kube-proxy/proxy.stderr.log     ; stderr log path, NONE for none; default AUTO
	stderr_logfile_maxbytes=64MB                                    		 ; max # logfile bytes b4 rotation (default 50MB)
	stderr_logfile_backups=4                                        		 ; # of stderr logfile backups (default 10)
	stderr_capture_maxbytes=1MB   						                           ; number of bytes in 'capturemode' (default 0)
	stderr_events_enabled=false   						                           ; emit events on stderr writes (default false)

**启动**
	
	supervisorctl  update

**验证**
	
	ipvsadm -Ln  
	kubectl get svc

##验证kubernetes集群##

编辑`/root/nginx-ds.yaml`

	apiVersion: extensions/v1beta1
	kind: DaemonSet
	metadata:
	  name: nginx-ds
	  labels:
	    addonmanager.kubernetes.io/mode: Reconcile
	spec:
	  template:
	    metadata:
	      labels:
	        app: nginx-ds
	    spec:
	      containers:
	      - name: my-nginx
	        image: nginx:1.7.9
	        ports:
	        - containerPort: 80

**应用资源配置，并检查**

	kubectl create -f nginx-ds.yaml
	kubectl get pods
	kubectl get pods -o wide