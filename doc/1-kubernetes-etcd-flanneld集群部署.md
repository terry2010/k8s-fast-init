

# kubernetes+etcd+flanneld 集群部署
### 文档涉及版本
    kubernetes 1.13.1
    etcd 3.3.10
    flanneld 0.10


### 下载链接
```
Client Binaries
https://dl.k8s.io/v1.13.1/kubernetes-client-linux-amd64.tar.gz
Server Binaries
https://dl.k8s.io/v1.13.1/kubernetes-server-linux-amd64.tar.gz
Node Binaries
https://dl.k8s.io/v1.13.1/kubernetes-node-linux-amd64.tar.gz
etcd
https://github.com/etcd-io/etcd/releases/download/v3.3.10/etcd-v3.3.10-linux-amd64.tar.gz
https://github.com/etcd-io/etcd/releases/download/v3.3.12/etcd-v3.3.12-linux-amd64.tar.gz
flannel
https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz
```

### 服务器角色


 角色         |ip    |主机名|服务列表
 -------------------|-|-|-
k8s-master1|	192.168.50.10|	k8s-master|	etcd、kube-apiserver、kube-controller-manager、kube-scheduler
k8s-node1   |	192.168.50.21|	k8s-node|	etcd、kubelet、docker、kube_proxy
k8s-node2   |	192.168.50.22|	k8s-node|	etcd、kubelet、docker、kube_proxy
 

### 防火墙设置

 服务器角色 |	端口
---------------|-
 etcd |	2379、2380
 Master |	6443、8472
 Node |	8472
 LB |	8443

@todo ：先关闭防火墙，生产机的上线流程写完了再改
```
systemctl stop firewalld
chkconfig iptables off  
```

## MASTER 服务器，基础安装 和证书生成

### 在所有服务器下载软件
```
wget https://dl.k8s.io/v1.13.1/kubernetes-server-linux-amd64.tar.gz
wget https://dl.k8s.io/v1.13.1/kubernetes-client-linux-amd64.tar.gz
wget https://dl.k8s.io/v1.13.1/kubernetes-node-linux-amd64.tar.gz

wget https://github.com/etcd-io/etcd/releases/download/v3.3.10/etcd-v3.3.10-linux-amd64.tar.gz
wget https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
```

## 证书生成
### 下载cfssl

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo

```


## ETCD 服务部署

### 基础路径创建
```
mkdir -p /k8s/etcd/{bin,cfg,ssl}
cd /k8s/etcd/ssl/
```

## etcd.step.1
*以下操作在etcd01服务器执行*
### etcd 证书生成
##### etcd ca配置
```
cat << EOF | tee ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "etcd": {
         "expiry": "87600h",
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
EOF

```
##### etcd ca证书
```
cat << EOF | tee ca-csr.json
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF

```
####  etcd server证书

```
cat << EOF | tee server-csr.json
{
    "CN": "etcd",
    "hosts": [
    "192.168.50.10",
    "192.168.50.21",
    "192.168.50.22"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF

```

#### 生成etcd ca证书和私钥 初始化ca
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca 
```
执行输出
```
[root@k8s-master ssl]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca 
2019/02/12 11:33:37 [INFO] generating a new CA key and certificate from CSR
2019/02/12 11:33:37 [INFO] generate received request
2019/02/12 11:33:37 [INFO] received CSR
2019/02/12 11:33:37 [INFO] generating key: rsa-2048
2019/02/12 11:33:37 [INFO] encoded CSR
2019/02/12 11:33:37 [INFO] signed certificate with serial number 238191686567694059843016759495808939046554666646
[root@k8s-master ssl]# ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem	 server-csr.json
```
#### 生成etcd server 证书
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=etcd server-csr.json | cfssljson -bare server
```
执行输出
```
[root@k8s-master ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=etcd server-csr.json | cfssljson -bare server
2019/02/12 11:37:16 [INFO] generate received request
2019/02/12 11:37:16 [INFO] received CSR
2019/02/12 11:37:16 [INFO] generating key: rsa-2048
2019/02/12 11:37:16 [INFO] encoded CSR
2019/02/12 11:37:16 [INFO] signed certificate with serial number 107880657375364549897688914639187306835470628394
2019/02/12 11:37:16 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```



## etcd.step.2

#### etcd-node-01 安装
此步操作在etcd01 中执行（本例为master）

基础路径创建
```
mkdir -p /k8s/etcd/{bin,cfg,ssl}
```

解压缩

```
tar -xvf etcd-v3.3.10-linux-amd64.tar.gz
cd etcd-v3.3.10-linux-amd64/
cp etcd etcdctl /k8s/etcd/bin/
```
配置etcd主文件

```
cat << EOF | tee  /k8s/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/data1/etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.50.10:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.50.10:2379,http://127.0.0.1:2379"
 
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.50.10:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.50.10:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.50.10:2380,etcd02=https://192.168.50.21:2380,etcd03=https://192.168.50.22:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

#[Security]
ETCD_CERT_FILE="/k8s/etcd/ssl/server.pem"
ETCD_KEY_FILE="/k8s/etcd/ssl/server-key.pem"
ETCD_TRUSTED_CA_FILE="/k8s/etcd/ssl/ca.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_PEER_CERT_FILE="/k8s/etcd/ssl/server.pem"
ETCD_PEER_KEY_FILE="/k8s/etcd/ssl/server-key.pem"
ETCD_PEER_TRUSTED_CA_FILE="/k8s/etcd/ssl/ca.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
EOF

```

配置etcd启动文件

```
mkdir -p /data1/etcd
vim /usr/lib/systemd/system/etcd.service

```
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/data1/etcd/
EnvironmentFile=-/k8s/etcd/cfg/etcd.conf
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /k8s/etcd/bin/etcd --name=\"${ETCD_NAME}\" --data-dir=\"${ETCD_DATA_DIR}\" --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\" --listen-peer-urls=\"${ETCD_LISTEN_PEER_URLS}\" --advertise-client-urls=\"${ETCD_ADVERTISE_CLIENT_URLS}\" --initial-cluster-token=\"${ETCD_INITIAL_CLUSTER_TOKEN}\" --initial-cluster=\"${ETCD_INITIAL_CLUSTER}\" --initial-cluster-state=\"${ETCD_INITIAL_CLUSTER_STATE}\" --cert-file=\"${ETCD_CERT_FILE}\" --key-file=\"${ETCD_KEY_FILE}\" --trusted-ca-file=\"${ETCD_TRUSTED_CA_FILE}\" --client-cert-auth=\"${ETCD_CLIENT_CERT_AUTH}\" --peer-cert-file=\"${ETCD_PEER_CERT_FILE}\" --peer-key-file=\"${ETCD_PEER_KEY_FILE}\" --peer-trusted-ca-file=\"${ETCD_PEER_TRUSTED_CA_FILE}\" --peer-client-cert-auth=\"${ETCD_PEER_CLIENT_CERT_AUTH}\""
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

配置服务，但此时启动etcd01, 会因为etcd02和etcd03没有启动出现启动失败的报错。 等三台服务器都配置完之后，使用文中命令检查服务状态， 如果还是异常状态，则可以再启动一次。
```
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```


## etcd.step.3
*以下操作在etcd02服务器执行*
#### etcd-node-02 安装
基础路径创建
```
mkdir -p /k8s/etcd/{bin,cfg,ssl}
```
拷贝证书
```
cd /k8s/etcd/ssl/
scp 192.168.50.10:$PWD/*.pem $PWD
```
执行结果
```
[root@k8s-node-1 ssl]# scp  192.168.50.10:$PWD/*.pem $PWD
root@192.168.50.10's password: 
ca-key.pem                                                                                                                                 100% 1679   509.3KB/s   00:00    
ca.pem                                                                                                                                     100% 1265   670.3KB/s   00:00    
server-key.pem                                                                                                                             100% 1679   959.2KB/s   00:00    
server.pem                     
```

解压缩

```
tar -xvf etcd-v3.3.10-linux-amd64.tar.gz
cd etcd-v3.3.10-linux-amd64/
cp etcd etcdctl /k8s/etcd/bin/
```
配置etcd主文件

```
vim /k8s/etcd/cfg/etcd.conf  
```
etcd.conf  文件内容
``` 
#[Member]
ETCD_NAME="etcd02"
ETCD_DATA_DIR="/data1/etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.50.21:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.50.21:2379,http://127.0.0.1:2379"
 
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.50.21:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.50.21:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.50.10:2380,etcd02=https://192.168.50.21:2380,etcd03=https://192.168.50.22:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

#[Security]
ETCD_CERT_FILE="/k8s/etcd/ssl/server.pem"
ETCD_KEY_FILE="/k8s/etcd/ssl/server-key.pem"
ETCD_TRUSTED_CA_FILE="/k8s/etcd/ssl/ca.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_PEER_CERT_FILE="/k8s/etcd/ssl/server.pem"
ETCD_PEER_KEY_FILE="/k8s/etcd/ssl/server-key.pem"
ETCD_PEER_TRUSTED_CA_FILE="/k8s/etcd/ssl/ca.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
```

配置etcd启动文件

```
mkdir -p /data1/etcd
vim /usr/lib/systemd/system/etcd.service
```
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/data1/etcd/
EnvironmentFile=-/k8s/etcd/cfg/etcd.conf
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /k8s/etcd/bin/etcd --name=\"${ETCD_NAME}\" --data-dir=\"${ETCD_DATA_DIR}\" --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\" --listen-peer-urls=\"${ETCD_LISTEN_PEER_URLS}\" --advertise-client-urls=\"${ETCD_ADVERTISE_CLIENT_URLS}\" --initial-cluster-token=\"${ETCD_INITIAL_CLUSTER_TOKEN}\" --initial-cluster=\"${ETCD_INITIAL_CLUSTER}\" --initial-cluster-state=\"${ETCD_INITIAL_CLUSTER_STATE}\" --cert-file=\"${ETCD_CERT_FILE}\" --key-file=\"${ETCD_KEY_FILE}\" --trusted-ca-file=\"${ETCD_TRUSTED_CA_FILE}\" --client-cert-auth=\"${ETCD_CLIENT_CERT_AUTH}\" --peer-cert-file=\"${ETCD_PEER_CERT_FILE}\" --peer-key-file=\"${ETCD_PEER_KEY_FILE}\" --peer-trusted-ca-file=\"${ETCD_PEER_TRUSTED_CA_FILE}\" --peer-client-cert-auth=\"${ETCD_PEER_CLIENT_CERT_AUTH}\""
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

配置服务，启动etcd02 ,
    注意： 没开放端口或者没关闭防火墙会导致启动失败
```
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```


## etcd.step.4
*以下操作在etcd03服务器执行*
#### etcd-node-03 安装
基础路径创建
```
mkdir -p /k8s/etcd/{bin,cfg,ssl}
```
拷贝证书
```
cd /k8s/etcd/ssl/
scp 192.168.50.10:$PWD/*.pem $PWD
```
执行结果
```
[root@k8s-node-1 ssl]# scp  192.168.50.10:$PWD/*.pem $PWD
root@192.168.50.10's password: 
ca-key.pem                                                                                                                                 100% 1679   509.3KB/s   00:00    
ca.pem                                                                                                                                     100% 1265   670.3KB/s   00:00    
server-key.pem                                                                                                                             100% 1679   959.2KB/s   00:00    
server.pem                     
```

解压缩

```
tar -xvf etcd-v3.3.10-linux-amd64.tar.gz
cd etcd-v3.3.10-linux-amd64/
cp etcd etcdctl /k8s/etcd/bin/
```
配置etcd主文件

```
vim /k8s/etcd/cfg/etcd.conf  
```
etcd.conf  文件内容
``` 
#[Member]
ETCD_NAME="etcd03"
ETCD_DATA_DIR="/data1/etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.50.22:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.50.22:2379,http://127.0.0.1:2379"
 
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.50.22:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.50.22:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.50.10:2380,etcd02=https://192.168.50.21:2380,etcd03=https://192.168.50.22:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

#[Security]
ETCD_CERT_FILE="/k8s/etcd/ssl/server.pem"
ETCD_KEY_FILE="/k8s/etcd/ssl/server-key.pem"
ETCD_TRUSTED_CA_FILE="/k8s/etcd/ssl/ca.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_PEER_CERT_FILE="/k8s/etcd/ssl/server.pem"
ETCD_PEER_KEY_FILE="/k8s/etcd/ssl/server-key.pem"
ETCD_PEER_TRUSTED_CA_FILE="/k8s/etcd/ssl/ca.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
```

配置etcd启动文件

```
mkdir -p /data1/etcd
vim /usr/lib/systemd/system/etcd.service
```
文件内容
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/data1/etcd/
EnvironmentFile=-/k8s/etcd/cfg/etcd.conf
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /k8s/etcd/bin/etcd --name=\"${ETCD_NAME}\" --data-dir=\"${ETCD_DATA_DIR}\" --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\" --listen-peer-urls=\"${ETCD_LISTEN_PEER_URLS}\" --advertise-client-urls=\"${ETCD_ADVERTISE_CLIENT_URLS}\" --initial-cluster-token=\"${ETCD_INITIAL_CLUSTER_TOKEN}\" --initial-cluster=\"${ETCD_INITIAL_CLUSTER}\" --initial-cluster-state=\"${ETCD_INITIAL_CLUSTER_STATE}\" --cert-file=\"${ETCD_CERT_FILE}\" --key-file=\"${ETCD_KEY_FILE}\" --trusted-ca-file=\"${ETCD_TRUSTED_CA_FILE}\" --client-cert-auth=\"${ETCD_CLIENT_CERT_AUTH}\" --peer-cert-file=\"${ETCD_PEER_CERT_FILE}\" --peer-key-file=\"${ETCD_PEER_KEY_FILE}\" --peer-trusted-ca-file=\"${ETCD_PEER_TRUSTED_CA_FILE}\" --peer-client-cert-auth=\"${ETCD_PEER_CLIENT_CERT_AUTH}\""
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

配置服务，启动etcd03 ,
    注意： 没开放端口或者没关闭防火墙会导致启动失败
```
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

## etcd.step.5
> 以下操作可以在任何一台启动过etcd的服务器执行
```
 /k8s/etcd/bin/etcdctl --ca-file=/k8s/etcd/ssl/ca.pem --cert-file=/k8s/etcd/ssl/server.pem --key-file=/k8s/etcd/ssl/server-key.pem --endpoints="https://192.168.50.10:2379,https://192.168.50.21:2379,https://192.168.50.22:2379" cluster-health
```
执行结果
```
[root@k8s-node-2 etcd-v3.3.10-linux-amd64]# /k8s/etcd/bin/etcdctl --ca-file=/k8s/etcd/ssl/ca.pem --cert-file=/k8s/etcd/ssl/server.pem --key-file=/k8s/etcd/ssl/server-key.pem --endpoints="https://192.168.50.10:2379,https://192.168.50.21:2379,https://192.168.50.22:2379" cluster-health
member 48aebc8897d84757 is healthy: got healthy result from https://192.168.50.10:2379
member 6b85f157810fe4ab is healthy: got healthy result from https://192.168.50.21:2379
member 834444e7d46c33e4 is healthy: got healthy result from https://192.168.50.22:2379
cluster is healthy
```


## kubernetes 安装

>以下操作在master执行

### 基础路径创建
```
mkdir -p /k8s/kubernetes/{bin,cfg,ssl} 
cd /k8s/kubernetes/ssl/
```
### 证书生成
> 使用cfssl 生成证书， 安装方式参见etcd 安装开始处
> 以下操作在 k8s-master 上执行，且假定cfssl在k8s-master 已经安装完成.
###
## k8s.step.1
### 生成kubernets证书与私钥

#### 制作kubernetes ca证书
```
cd /k8s/kubernetes/ssl
```
```
cat << EOF | tee ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
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
EOF
```
 
```
cat << EOF | tee ca-csr.json
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

执行以下命令
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

```
执行结果
```
[root@k8s-master ssl]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
2019/02/12 12:00:43 [INFO] generating a new CA key and certificate from CSR
2019/02/12 12:00:43 [INFO] generate received request
2019/02/12 12:00:43 [INFO] received CSR
2019/02/12 12:00:43 [INFO] generating key: rsa-2048
2019/02/12 12:00:44 [INFO] encoded CSR
2019/02/12 12:00:44 [INFO] signed certificate with serial number 193610323705675526688752988224199514176925987003
[root@k8s-master ssl]# ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```


#### 制作apiserver证书

```
cat << EOF | tee server-csr.json
{
    "CN": "kubernetes",
    "hosts": [
      "192.168.50.1",
      "127.0.0.1",
      "192.168.50.10",
      "192.168.50.21",
      "192.168.50.22",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

执行命令
```
 cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
 ```
 输出结果
 ```
 [root@k8s-master ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
2019/02/12 12:07:29 [INFO] generate received request
2019/02/12 12:07:29 [INFO] received CSR
2019/02/12 12:07:29 [INFO] generating key: rsa-2048
2019/02/12 12:07:29 [INFO] encoded CSR
2019/02/12 12:07:29 [INFO] signed certificate with serial number 700810050038865385373338867239054062888152574206
2019/02/12 12:07:29 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
[root@k8s-master ssl]# ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem  server.csr  server-csr.json  server-key.pem  server.pem
 ```

#### 制作kube-proxy证书
```
cat << EOF | tee kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

执行命令
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

```
输出结果
```
[root@k8s-master ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
2019/02/12 12:09:50 [INFO] generate received request
2019/02/12 12:09:50 [INFO] received CSR
2019/02/12 12:09:50 [INFO] generating key: rsa-2048
2019/02/12 12:09:50 [INFO] encoded CSR
2019/02/12 12:09:50 [INFO] signed certificate with serial number 481991947134927061933629246120948488949754418059
2019/02/12 12:09:50 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
[root@k8s-master ssl]# ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem  kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem  server.csr  server-csr.json  server-key.pem  server.pem
```

## K8S.STEP.2
### 部署kubernetes server

> 以下还在 k8s-master上执行

#### 解压缩文件
```
tar -zxvf kubernetes-server-linux-amd64.tar.gz 
cd kubernetes/server/bin/
cp kube-scheduler kube-apiserver kube-controller-manager kubectl kubeadm /k8s/kubernetes/bin/

```

#### 部署kube-apiserver组件 创建TLS Bootstrapping Token
> 生成随机字符串， 之后命令中的随机字符串都需要用服务器生成的随机字符串替换

执行命令
```
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```
输出结果
```
[root@k8s-master ~]#  head -c 16 /dev/urandom | od -An -t x | tr -d ' '
f5675ffd8d3d03ef5a6beec27be8dd80
```
执行命令
```
vim /k8s/kubernetes/cfg/token.csv
```
输入内容
> 记得将下文的 f5675ffd8d3d03ef5a6beec27be8dd80 替换为服务器生成的随机字符串
```
f5675ffd8d3d03ef5a6beec27be8dd80,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
```

#### 创建Apiserver配置文件
```
vim /k8s/kubernetes/cfg/kube-apiserver 
```
输入内容
```
KUBE_APISERVER_OPTS="--logtostderr=true \
--v=4 \
--etcd-servers=https://192.168.50.10:2379,https://192.168.50.21:2379,https://192.168.50.22:2379 \
--bind-address=192.168.50.10 \
--secure-port=6443 \
--advertise-address=192.168.50.10 \
--allow-privileged=true \
--service-cluster-ip-range=192.168.0.0/16 \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--enable-bootstrap-token-auth \
--token-auth-file=/k8s/kubernetes/cfg/token.csv \
--service-node-port-range=30000-50000 \
--tls-cert-file=/k8s/kubernetes/ssl/server.pem  \
--tls-private-key-file=/k8s/kubernetes/ssl/server-key.pem \
--client-ca-file=/k8s/kubernetes/ssl/ca.pem \
--service-account-key-file=/k8s/kubernetes/ssl/ca-key.pem \
--etcd-cafile=/k8s/etcd/ssl/ca.pem \
--etcd-certfile=/k8s/etcd/ssl/server.pem \
--etcd-keyfile=/k8s/etcd/ssl/server-key.pem"
```

#### 创建apiserver systemd文件
执行命令
```
vim /usr/lib/systemd/system/kube-apiserver.service 

```
输入内容
```
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
 
[Service]
EnvironmentFile=-/k8s/kubernetes/cfg/kube-apiserver
ExecStart=/k8s/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
```

启动服务
```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
```

检查服务启动状态

```
[root@k8s-master ssl]# systemctl status kube-apiserver
● kube-apiserver.service - Kubernetes API Server
   Loaded: loaded (/usr/lib/systemd/system/kube-apiserver.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-02-12 12:30:46 EST; 15s ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 21922 (kube-apiserver)
    Tasks: 10
   Memory: 273.3M
   CGroup: /system.slice/kube-apiserver.service
           └─21922 /k8s/kubernetes/bin/kube-apiserver --logtostderr=true --v=4 --etcd-servers=https://192.168.50.10:2379,https://192.168.50.21:2379,https://192.168.50.22:2379 --bind-address=192.168.50.10 --secure-port=6443 --advert...

Feb 12 12:31:00 k8s-master kube-apiserver[21922]: I0212 12:31:00.530350   21922 wrap.go:47] POST /apis/rbac.authorization.k8s.io/v1/namespaces/kube-system/rolebindings: (3.798677ms) 201 [kube-apiserver/v1.13.1 (linu...168.50.10:35834]
Feb 12 12:31:00 k8s-master kube-apiserver[21922]: I0212 12:31:00.530638   21922 storage_rbac.go:276] created rolebinding.rbac.authorization.k8s.io/system:controller:cloud-provider in kube-system
Feb 12 12:31:00 k8s-master kube-apiserver[21922]: I0212 12:31:00.549200   21922 wrap.go:47] GET /apis/rbac.authorization.k8s.io/v1/namespaces/kube-system/rolebindings/system:controller:token-cleaner: (2.588941ms) 40...168.50.10:35834]
Feb 12 12:31:00 k8s-master kube-apiserver[21922]: I0212 12:31:00.552475   21922 wrap.go:47] GET /api/v1/namespaces/kube-system: (2.74916ms) 200 [kube-apiserver/v1.13.1 (linux/amd64) kubernetes/eec55b9 192.168.50.10:35834]
Feb 12 12:31:00 k8s-master kube-apiserver[21922]: I0212 12:31:00.570315   21922 wrap.go:47] POST /apis/rbac.authorization.k8s.io/v1/namespaces/kube-system/rolebindings: (3.728849ms) 201 [kube-apiserver/v1.13.1 (linu...168.50.10:35834]
Feb 12 12:31:00 k8s-master kube-apiserver[21922]: I0212 12:31:00.570491   21922 storage_rbac.go:276] created rolebinding.rbac.authorization.k8s.io/system:controller:token-cleaner in kube-system
Feb 12 12:31:00 k8s-master kube-apiserver[21922]: I0212 12:31:00.589798   21922 wrap.go:47] GET /apis/rbac.authorization.k8s.io/v1/namespaces/kube-public/rolebindings/system:controller:bootstrap-signer: (2.653749ms)...168.50.10:35834]
Feb 12 12:31:00 k8s-master kube-apiserver[21922]: I0212 12:31:00.592625   21922 wrap.go:47] GET /api/v1/namespaces/kube-public: (2.443401ms) 200 [kube-apiserver/v1.13.1 (linux/amd64) kubernetes/eec55b9 192.168.50.10:35834]
Feb 12 12:31:00 k8s-master kube-apiserver[21922]: I0212 12:31:00.610995   21922 wrap.go:47] POST /apis/rbac.authorization.k8s.io/v1/namespaces/kube-public/rolebindings: (3.709393ms) 201 [kube-apiserver/v1.13.1 (linu...168.50.10:35834]
Feb 12 12:31:00 k8s-master kube-apiserver[21922]: I0212 12:31:00.611263   21922 storage_rbac.go:276] created rolebinding.rbac.authorization.k8s.io/system:controller:bootstrap-signer in kube-public
Hint: Some lines were ellipsized, use -l to show in full.
[root@k8s-master ssl]# ps -ef |grep kube-apiserver
root      21922      1 47 12:30 ?        00:00:12 /k8s/kubernetes/bin/kube-apiserver --logtostderr=true --v=4 --etcd-servers=https://192.168.50.10:2379,https://192.168.50.21:2379,https://192.168.50.22:2379 --bind-address=192.168.50.10 --secure-port=6443 --advertise-address=192.168.50.10 --allow-privileged=true --service-cluster-ip-range=192.168.50.0/24 --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction --authorization-mode=RBAC,Node --enable-bootstrap-token-auth --token-auth-file=/k8s/kubernetes/cfg/token.csv --service-node-port-range=30000-50000 --tls-cert-file=/k8s/kubernetes/ssl/server.pem --tls-private-key-file=/k8s/kubernetes/ssl/server-key.pem --client-ca-file=/k8s/kubernetes/ssl/ca.pem --service-account-key-file=/k8s/kubernetes/ssl/ca-key.pem --etcd-cafile=/k8s/etcd/ssl/ca.pem --etcd-certfile=/k8s/etcd/ssl/server.pem --etcd-keyfile=/k8s/etcd/ssl/server-key.pem
root      21934  21665  0 12:31 pts/3    00:00:00 grep --color=auto kube-apiserver
[root@k8s-master ssl]# netstat -tulpn |grep kube-apiserve
tcp        0      0 192.168.50.10:6443      0.0.0.0:*               LISTEN      21922/kube-apiserve 
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      21922/kube-apiserve 
```


### 部署kube-scheduler组件 
#### 创建kube-scheduler配置文件

执行命令
```
vim  /k8s/kubernetes/cfg/kube-scheduler 
```
输入内容
```
KUBE_SCHEDULER_OPTS="--logtostderr=true --v=4 --master=127.0.0.1:8080 --leader-elect"

```
>参数备注： 

>--address：在 127.0.0.1:10251 端口接收 http /metrics 请求；kube-scheduler 目前还不支持接收 https 请求； 

>--kubeconfig：指定 kubeconfig 文件路径，kube-scheduler 使用它连接和验证 kube-apiserver； 

>--leader-elect=true：集群运行模式，启用选举功能；被选为 leader 的节点负责处理工作，其它节点为阻塞状态；

创建kube-scheduler systemd文件
```
vim /usr/lib/systemd/system/kube-scheduler.service 

```

输入内容
```
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
 
[Service]
EnvironmentFile=-/k8s/kubernetes/cfg/kube-scheduler
ExecStart=/k8s/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
```

启动服务

```
systemctl daemon-reload
systemctl enable kube-scheduler.service 
systemctl start kube-scheduler.service
```

检查服务状态
```
[root@k8s-master ssl]# systemctl status kube-scheduler.service
● kube-scheduler.service - Kubernetes Scheduler
   Loaded: loaded (/usr/lib/systemd/system/kube-scheduler.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-02-12 12:36:20 EST; 19s ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 21983 (kube-scheduler)
    Tasks: 10
   Memory: 10.8M
   CGroup: /system.slice/kube-scheduler.service
           └─21983 /k8s/kubernetes/bin/kube-scheduler --logtostderr=true --v=4 --master=127.0.0.1:8080 --leader-elect

Feb 12 12:36:21 k8s-master kube-scheduler[21983]: I0212 12:36:21.917261   21983 shared_informer.go:123] caches populated
Feb 12 12:36:22 k8s-master kube-scheduler[21983]: I0212 12:36:22.017849   21983 shared_informer.go:123] caches populated
Feb 12 12:36:22 k8s-master kube-scheduler[21983]: I0212 12:36:22.118056   21983 shared_informer.go:123] caches populated
Feb 12 12:36:22 k8s-master kube-scheduler[21983]: I0212 12:36:22.218653   21983 shared_informer.go:123] caches populated
Feb 12 12:36:22 k8s-master kube-scheduler[21983]: I0212 12:36:22.218718   21983 controller_utils.go:1027] Waiting for caches to sync for scheduler controller
Feb 12 12:36:22 k8s-master kube-scheduler[21983]: I0212 12:36:22.319354   21983 shared_informer.go:123] caches populated
Feb 12 12:36:22 k8s-master kube-scheduler[21983]: I0212 12:36:22.319391   21983 controller_utils.go:1034] Caches are synced for scheduler controller
Feb 12 12:36:22 k8s-master kube-scheduler[21983]: I0212 12:36:22.319440   21983 leaderelection.go:205] attempting to acquire leader lease  kube-system/kube-scheduler...
Feb 12 12:36:22 k8s-master kube-scheduler[21983]: I0212 12:36:22.327059   21983 leaderelection.go:214] successfully acquired lease kube-system/kube-scheduler
Feb 12 12:36:22 k8s-master kube-scheduler[21983]: I0212 12:36:22.428706   21983 shared_informer.go:123] caches populated
```

### 部署kube-controller-manager组件 
#### 创建kube-controller-manager配置文件
执行命令
```
vim /k8s/kubernetes/cfg/kube-controller-manager

```
输入内容
```
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=true \
--v=4 \
--master=127.0.0.1:8080 \
--leader-elect=true \
--address=127.0.0.1 \
--service-cluster-ip-range=192.168.0.0/16 \
--cluster-name=kubernetes \
--cluster-signing-cert-file=/k8s/kubernetes/ssl/ca.pem \
--cluster-signing-key-file=/k8s/kubernetes/ssl/ca-key.pem  \
--root-ca-file=/k8s/kubernetes/ssl/ca.pem \
--service-account-private-key-file=/k8s/kubernetes/ssl/ca-key.pem"
```

创建kube-controller-manager systemd文件

执行命令
```
vim /usr/lib/systemd/system/kube-controller-manager.service
```
输入内容
```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
 
[Service]
EnvironmentFile=-/k8s/kubernetes/cfg/kube-controller-manager
ExecStart=/k8s/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
```
 

启动服务
```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
```

检查服务状态
```
[root@k8s-master ssl]# systemctl status kube-controller-manager
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/usr/lib/systemd/system/kube-controller-manager.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-02-12 12:41:26 EST; 16s ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 22040 (kube-controller)
    Tasks: 8
   Memory: 44.4M
   CGroup: /system.slice/kube-controller-manager.service
           └─22040 /k8s/kubernetes/bin/kube-controller-manager --logtostderr=true --v=4 --master=127.0.0.1:8080 --leader-elect=true --address=127.0.0.1 --service-cluster-ip-range=192.168.50.0/24 --cluster-name=kubernetes --cluster-...

Feb 12 12:41:29 k8s-master kube-controller-manager[22040]: E0212 12:41:29.554869   22040 resource_quota_controller.go:437] failed to sync resource monitors: couldn't start monitor for resource "extensions/v1beta1, Re...etworkpolicies"
Feb 12 12:41:29 k8s-master kube-controller-manager[22040]: I0212 12:41:29.621976   22040 shared_informer.go:123] caches populated
Feb 12 12:41:29 k8s-master kube-controller-manager[22040]: I0212 12:41:29.622030   22040 controller_utils.go:1034] Caches are synced for garbage collector controller
Feb 12 12:41:29 k8s-master kube-controller-manager[22040]: I0212 12:41:29.622039   22040 garbagecollector.go:245] synced garbage collector
Feb 12 12:41:29 k8s-master kube-controller-manager[22040]: I0212 12:41:29.652471   22040 shared_informer.go:123] caches populated
Feb 12 12:41:29 k8s-master kube-controller-manager[22040]: I0212 12:41:29.652743   22040 controller_utils.go:1034] Caches are synced for garbage collector controller
Feb 12 12:41:29 k8s-master kube-controller-manager[22040]: I0212 12:41:29.652759   22040 garbagecollector.go:142] Garbage collector: all resource monitors have synced. Proceeding to collect garbage
Feb 12 12:41:38 k8s-master kube-controller-manager[22040]: I0212 12:41:38.099236   22040 cronjob_controller.go:111] Found 0 jobs
Feb 12 12:41:38 k8s-master kube-controller-manager[22040]: I0212 12:41:38.101947   22040 cronjob_controller.go:119] Found 0 cronjobs
Feb 12 12:41:38 k8s-master kube-controller-manager[22040]: I0212 12:41:38.101963   22040 cronjob_controller.go:122] Found 0 groups
Hint: Some lines were ellipsized, use -l to show in full.
```

### 验证kubeserver服务

设置环境变量
```
vim ~/.bashrc
```
在文件结尾加入
```
export PATH=/k8s/kubernetes/bin:$PATH
```
执行
```
source ~/.bashrc
```

### 最后检查master服务状态

执行命令
```
kubectl get cs,nodes

```
执行结果
```
[root@k8s-master ssl]# kubectl get cs,nodes
NAME                                 STATUS    MESSAGE             ERROR
componentstatus/scheduler            Healthy   ok                  
componentstatus/controller-manager   Healthy   ok                  
componentstatus/etcd-0               Healthy   {"health":"true"}   
componentstatus/etcd-2               Healthy   {"health":"true"}   
componentstatus/etcd-1               Healthy   {"health":"true"}   
```

## kubernetes Node-01 部署

> 第一个节点部署和之后不一样的地方是：
>   需要在master 上 将kubelet-bootstrap用户绑定到系统集群角色
> kubernetes work 节点运行如下组件：  
> docker  
> kubelet  
> kube-proxy  
> flannel  
> 
>系统环境  
>CentOS Linux release 7.4.1708 (Core)  
>Docker版本  
>Server Version: 18.09.0  
>Cgroup Driver: cgroupfs

#### Docker环境安装

```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum list docker-ce --showduplicates | sort -r
yum install docker-ce -y
systemctl start docker && systemctl enable docker
```
或者参考：
https://github.com/terry2010/centos7-fast-init/blob/master/docker/install.sh

[利用aliyun源在centos7快速安装docker  ](https://github.com/terry2010/centos7-fast-init/blob/master/docker/install.sh)

### 基础路径创建
```
mkdir -p /k8s/kubernetes/{bin,cfg,ssl} 

```

#### 部署kubelet
> kublet 运行在每个 worker 节点上，接收 kube-apiserver 发送的请求，管理 Pod 容器，执行交互式命令，如exec、run、logs 等; kublet 启动时自动向 kube-apiserver 注册节点信息，内置的 cadvisor 统计和监控节点的资源使用情况; 为确保安全，只开启接收 https 请求的安全端口，对请求进行认证和授权，拒绝未授权的访问(如apiserver、heapster)
>
安装二进制文件
```
tar zxvf kubernetes-node-linux-amd64.tar.gz
cd kubernetes/node/bin/
cp kube-proxy kubelet kubectl kubeadm /k8s/kubernetes/bin/
```
设置环境变量
```
vim ~/.bashrc
```
在文件结尾加入
```
export PATH=/k8s/kubernetes/bin:$PATH
```
执行
```
source ~/.bashrc
```

复制相关证书到node节点

执行命令
```
cd /k8s/kubernetes/ssl
scp 192.168.50.10:$PWD/*.pem $PWD
```
执行结果
```
[root@k8s-node-1 ssl]# scp 192.168.50.10:$PWD/*.pem $PWD
root@192.168.50.10's password: 
ca-key.pem                                                                                                                                                                                              100% 1675     1.5MB/s   00:00    
ca.pem                                                                                                                                                                                                  100% 1359     1.2MB/s   00:00    
kube-proxy-key.pem                                                                                                                                                                                      100% 1679     1.7MB/s   00:00    
kube-proxy.pem                                                                                                                                                                                          100% 1403     1.6MB/s   00:00    
server-key.pem                                                                                                                                                                                          100% 1679     1.9MB/s   00:00    
server.pem                        
```

#### 生成创建kubelet bootstrap kubeconfig脚本， 并创建 kubeconfig文件




执行命令
```
vim /k8s/kubernetes/cfg/environment.sh

```
输入内容

```
#!/bin/bash
#创建kubelet bootstrapping kubeconfig 
#BOOTSTRAP_TOKEN需要修改为之前在master用urandom生成的随机字符串
BOOTSTRAP_TOKEN=f5675ffd8d3d03ef5a6beec27be8dd80
KUBE_APISERVER="https://192.168.50.10:6443"
#设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/k8s/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
 
#设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
 
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
 
# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
 
#----------------------
 
# 创建kube-proxy kubeconfig文件

kubectl config set-cluster kubernetes \
  --certificate-authority=/k8s/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
 
kubectl config set-credentials kube-proxy \
  --client-certificate=/k8s/kubernetes/ssl/kube-proxy.pem \
  --client-key=/k8s/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
 
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
 
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
执行脚本
```
cd /k8s/kubernetes/cfg
sh environment.sh 
```
执行结果
```
[root@k8s-node-1 cfg]# sh environment.sh 
Cluster "kubernetes" set.
User "kubelet-bootstrap" set.
Context "default" created.
Switched to context "default".
Cluster "kubernetes" set.
User "kube-proxy" set.
Context "default" created.
Switched to context "default".
[root@k8s-node-1 cfg]# ls
bootstrap.kubeconfig  environment.sh  kube-proxy.kubeconfig
```

创建kubelet参数配置模板文件

执行命令
```
vim /k8s/kubernetes/cfg/kubelet.config

```
输入内容
```
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 192.168.50.21
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS: ["192.168.50.10"]
clusterDomain: cluster.local.
failSwapOn: false
authentication:
  anonymous:
    enabled: true
```

创建kubelet配置文件

执行命令
```
vim /k8s/kubernetes/cfg/kubelet
```
输入内容
```
KUBELET_OPTS="--logtostderr=true \
--v=4 \
--hostname-override=192.168.50.21 \
--kubeconfig=/k8s/kubernetes/cfg/kubelet.kubeconfig \
--bootstrap-kubeconfig=/k8s/kubernetes/cfg/bootstrap.kubeconfig \
--config=/k8s/kubernetes/cfg/kubelet.config \
--cert-dir=/k8s/kubernetes/ssl \
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"

```

创建kubelet systemd文件

执行命令
```
vim /usr/lib/systemd/system/kubelet.service 
```
输入内容
```
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service
 
[Service]
EnvironmentFile=/k8s/kubernetes/cfg/kubelet
ExecStart=/k8s/kubernetes/bin/kubelet $KUBELET_OPTS
Restart=on-failure
KillMode=process
 
[Install]
WantedBy=multi-user.target
```
### 将kubelet-bootstrap用户绑定到系统集群角色
```
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```
因为默认是连接localhost:8080端口， 所以会有下面的报错
```
[root@k8s-node-1 cfg]# kubectl create clusterrolebinding kubelet-bootstrap \
>   --clusterrole=system:node-bootstrapper \
>   --user=kubelet-bootstrap
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
切换到 k8s-master 上执行下面的命令，将kubelet-bootstrap用户绑定到系统集群角色

```
kubectl create clusterrolebinding kubelet-bootstrap \
    --clusterrole=system:node-bootstrapper \
    --user=kubelet-bootstrap
```
执行输出结果
```
[root@k8s-master ssl]# kubectl create clusterrolebinding kubelet-bootstrap \
>     --clusterrole=system:node-bootstrapper \
>     --user=kubelet-bootstrap
clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created
```
回到node1 
启动服务
```
systemctl daemon-reload
systemctl enable kubelet 
systemctl start kubelet
```
检查服务状态

```
[root@k8s-node-1 cfg]# systemctl status kubelet
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-02-12 14:15:11 EST; 5s ago
 Main PID: 21681 (kubelet)
    Tasks: 9
   Memory: 15.5M
   CGroup: /system.slice/kubelet.service
           └─21681 /k8s/kubernetes/bin/kubelet --logtostderr=true --v=4 --hostname-override=192.168.50.21 --kubeconfig=/k8s/kubernetes/cfg/kubelet.kubeconfig --bootstrap-kubeconfig=/k8s/kubernetes/cfg/bootstrap.kubeconfig --config=...

Feb 12 14:15:12 k8s-node-1 kubelet[21681]: I0212 14:15:12.288293   21681 server.go:407] Version: v1.13.1
Feb 12 14:15:12 k8s-node-1 kubelet[21681]: I0212 14:15:12.288357   21681 feature_gate.go:206] feature gates: &{map[]}
Feb 12 14:15:12 k8s-node-1 kubelet[21681]: I0212 14:15:12.288432   21681 feature_gate.go:206] feature gates: &{map[]}
Feb 12 14:15:12 k8s-node-1 kubelet[21681]: I0212 14:15:12.288537   21681 plugins.go:103] No cloud provider specified.
Feb 12 14:15:12 k8s-node-1 kubelet[21681]: I0212 14:15:12.288551   21681 server.go:523] No cloud provider specified: "" from the config file: ""
Feb 12 14:15:12 k8s-node-1 kubelet[21681]: I0212 14:15:12.288585   21681 bootstrap.go:65] Using bootstrap kubeconfig to generate TLS client cert, key and kubeconfig file
Feb 12 14:15:12 k8s-node-1 kubelet[21681]: I0212 14:15:12.290188   21681 bootstrap.go:96] No valid private key and/or certificate found, reusing existing private key or creating a new one
Feb 12 14:15:12 k8s-node-1 kubelet[21681]: I0212 14:15:12.308724   21681 bootstrap.go:239] Failed to connect to apiserver: the server has asked for the client to provide credentials
Feb 12 14:15:14 k8s-node-1 kubelet[21681]: I0212 14:15:14.376419   21681 bootstrap.go:239] Failed to connect to apiserver: the server has asked for the client to provide credentials
Feb 12 14:15:16 k8s-node-1 kubelet[21681]: I0212 14:15:16.647004   21681 bootstrap.go:239] Failed to connect to apiserver: the server has asked for the client to provide credentials
``` 

在master上对node节点的csr进行授权

```
 kubectl get nodes  
 kubectl get csr 
 kubectl certificate approve node-csr-s6NbHbQp8M3fxKbRTO9AW6_L6KNi89gQdGByxm6sGn8 
```
执行结果
```
[root@k8s-master bin]# kubectl get nodes  
No resources found.
[root@k8s-master bin]# kubectl get csr 
NAME                                                   AGE     REQUESTOR           CONDITION
node-csr-GEpA4trrDK_r1By85WO6VEeMIvKYlzhVu4WM9AqffQU   7m26s   kubelet-bootstrap   Pending
[root@k8s-master bin]#  kubectl certificate approve node-csr-GEpA4trrDK_r1By85WO6VEeMIvKYlzhVu4WM9AqffQU
certificatesigningrequest.certificates.k8s.io/node-csr-GEpA4trrDK_r1By85WO6VEeMIvKYlzhVu4WM9AqffQU approved
[root@k8s-master bin]# kubectl get nodes  
NAME            STATUS   ROLES    AGE    VERSION
192.168.50.21   Ready    <none>   111s   v1.13.1
```




#### 部署 kube-proxy组件

kube-proxy 运行在所有 node节点上，它监听 apiserver 中 service 和 Endpoint 的变化情况，创建路由规则来进行服务负载均衡
> 此刻回到node-1 服务器
##### 创建 kube-proxy 配置文件
执行命令
```
vim /k8s/kubernetes/cfg/kube-proxy

```
输入内容
```
KUBE_PROXY_OPTS="--logtostderr=true \
--v=4 \
--hostname-override=192.168.50.21 \
--cluster-cidr=192.168.0.0/16 \
--kubeconfig=/k8s/kubernetes/cfg/kube-proxy.kubeconfig"
```

##### 创建kube-proxy systemd文件
执行命令
```
vim /usr/lib/systemd/system/kube-proxy.service 
```
输入内容
```
[Unit]
Description=Kubernetes Proxy
After=network.target
 
[Service]
EnvironmentFile=-/k8s/kubernetes/cfg/kube-proxy
ExecStart=/k8s/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
```

##### 启动 kube-proxy  服务
```
systemctl daemon-reload
systemctl enable kube-proxy 
systemctl start kube-proxy
```

查看服务状态
```
[root@k8s-node-1 cfg]# systemctl status  kube-proxy
● kube-proxy.service - Kubernetes Proxy
   Loaded: loaded (/usr/lib/systemd/system/kube-proxy.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-02-12 12:27:40 EST; 21s ago
 Main PID: 21802 (kube-proxy)
    Tasks: 0
   Memory: 10.8M
   CGroup: /system.slice/kube-proxy.service
           ‣ 21802 /k8s/kubernetes/bin/kube-proxy --logtostderr=true --v=4 --hostname-override=192.168.50.21 --cluster-cidr=192.168.0.0/16 --kubeconfig=/k8s/kubernetes/cfg/kube-proxy.kubeconfig

Feb 12 12:27:53 k8s-node-1 kube-proxy[21802]: I0212 12:27:53.299890   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:54 k8s-node-1 kube-proxy[21802]: I0212 12:27:54.270917   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:55 k8s-node-1 kube-proxy[21802]: I0212 12:27:55.311513   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:56 k8s-node-1 kube-proxy[21802]: I0212 12:27:56.276286   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:57 k8s-node-1 kube-proxy[21802]: I0212 12:27:57.323746   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:58 k8s-node-1 kube-proxy[21802]: I0212 12:27:58.302454   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:59 k8s-node-1 kube-proxy[21802]: I0212 12:27:59.332524   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:28:00 k8s-node-1 kube-proxy[21802]: I0212 12:28:00.332119   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:28:01 k8s-node-1 kube-proxy[21802]: I0212 12:28:01.343378   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:28:02 k8s-node-1 kube-proxy[21802]: I0212 12:28:02.345592   21802 config.go:141] Calling handler.OnEndpointsUpdate
```


 


## kubernetes Node-02以及更多 部署
> kubernetes work 节点运行如下组件：  
> docker  
> kubelet  
> kube-proxy  
> flannel  
> 
>系统环境  
>CentOS Linux release 7.4.1708 (Core)  
>Docker版本  
>Server Version: 18.09.0  
>Cgroup Driver: cgroupfs

#### Docker环境安装

```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum list docker-ce --showduplicates | sort -r
yum install docker-ce -y
systemctl start docker && systemctl enable docker
```
或者参考：
https://github.com/terry2010/centos7-fast-init/blob/master/docker/install.sh

[利用aliyun源在centos7快速安装docker  ](https://github.com/terry2010/centos7-fast-init/blob/master/docker/install.sh)

### 基础路径创建
```
mkdir -p /k8s/kubernetes/{bin,cfg,ssl} 

```

#### 部署kubelet
> kublet 运行在每个 worker 节点上，接收 kube-apiserver 发送的请求，管理 Pod 容器，执行交互式命令，如exec、run、logs 等; kublet 启动时自动向 kube-apiserver 注册节点信息，内置的 cadvisor 统计和监控节点的资源使用情况; 为确保安全，只开启接收 https 请求的安全端口，对请求进行认证和授权，拒绝未授权的访问(如apiserver、heapster)
>
安装二进制文件
```
tar zxvf kubernetes-node-linux-amd64.tar.gz
cd kubernetes/node/bin/
cp kube-proxy kubelet kubectl kubeadm /k8s/kubernetes/bin/
```
设置环境变量
```
vim ~/.bashrc
```
在文件结尾加入
```
export PATH=/k8s/kubernetes/bin:$PATH
```
执行
```
source ~/.bashrc
```

复制相关证书到node节点

执行命令
```
cd /k8s/kubernetes/ssl
scp 192.168.50.10:$PWD/*.pem $PWD
```
执行结果
```
[root@k8s-node-1 ssl]# scp 192.168.50.10:$PWD/*.pem $PWD
root@192.168.50.10's password: 
ca-key.pem                                                                                                                                                                                              100% 1675     1.5MB/s   00:00    
ca.pem                                                                                                                                                                                                  100% 1359     1.2MB/s   00:00    
kube-proxy-key.pem                                                                                                                                                                                      100% 1679     1.7MB/s   00:00    
kube-proxy.pem                                                                                                                                                                                          100% 1403     1.6MB/s   00:00    
server-key.pem                                                                                                                                                                                          100% 1679     1.9MB/s   00:00    
server.pem                        
```

#### 生成创建kubelet bootstrap kubeconfig脚本， 并创建 kubeconfig文件




执行命令
```
vim /k8s/kubernetes/cfg/environment.sh

```
输入内容

```
#!/bin/bash
#创建kubelet bootstrapping kubeconfig 
#BOOTSTRAP_TOKEN需要修改为之前在master用urandom生成的随机字符串
BOOTSTRAP_TOKEN=f5675ffd8d3d03ef5a6beec27be8dd80
KUBE_APISERVER="https://192.168.50.10:6443"
#设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/k8s/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
 
#设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
 
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
 
# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
 
#----------------------
 
# 创建kube-proxy kubeconfig文件

kubectl config set-cluster kubernetes \
  --certificate-authority=/k8s/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
 
kubectl config set-credentials kube-proxy \
  --client-certificate=/k8s/kubernetes/ssl/kube-proxy.pem \
  --client-key=/k8s/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
 
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
 
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

```
执行脚本
```
cd /k8s/kubernetes/cfg
sh environment.sh 
```
执行结果
```
[root@k8s-node-1 cfg]# sh environment.sh 
Cluster "kubernetes" set.
User "kubelet-bootstrap" set.
Context "default" created.
Switched to context "default".
Cluster "kubernetes" set.
User "kube-proxy" set.
Context "default" created.
Switched to context "default".
[root@k8s-node-1 cfg]# ls
bootstrap.kubeconfig  environment.sh  kube-proxy.kubeconfig
```

创建kubelet参数配置模板文件

执行命令
```
vim /k8s/kubernetes/cfg/kubelet.config

```
输入内容
```
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 192.168.50.22
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS: ["192.168.50.10"]
clusterDomain: cluster.local.
failSwapOn: false
authentication:
  anonymous:
    enabled: true
```

创建kubelet配置文件

执行命令
```
vim /k8s/kubernetes/cfg/kubelet
```
输入内容
```
KUBELET_OPTS="--logtostderr=true \
--v=4 \
--hostname-override=192.168.50.22 \
--kubeconfig=/k8s/kubernetes/cfg/kubelet.kubeconfig \
--bootstrap-kubeconfig=/k8s/kubernetes/cfg/bootstrap.kubeconfig \
--config=/k8s/kubernetes/cfg/kubelet.config \
--cert-dir=/k8s/kubernetes/ssl \
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"

```

创建kubelet systemd文件

执行命令
```
vim /usr/lib/systemd/system/kubelet.service 
```
输入内容
```
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service
 
[Service]
EnvironmentFile=/k8s/kubernetes/cfg/kubelet
ExecStart=/k8s/kubernetes/bin/kubelet $KUBELET_OPTS
Restart=on-failure
KillMode=process
 
[Install]
WantedBy=multi-user.target
```  

启动服务
```
systemctl daemon-reload
systemctl enable kubelet 
systemctl start kubelet
```
检查服务状态

```
[root@k8s-node-2 cfg]# systemctl status kubelet
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-02-12 12:16:27 EST; 4s ago
 Main PID: 21644 (kubelet)
    Tasks: 9
   Memory: 19.6M
   CGroup: /system.slice/kubelet.service
           └─21644 /k8s/kubernetes/bin/kubelet --logtostderr=true --v=4 --hostname-override=192.168.50.22 --kubeconfig=/k8s/kubernetes/cfg/kubelet.kubeconfig --bootstrap-kubeconfig=/k8s/kubernetes/cfg/...

Feb 12 12:16:27 k8s-node-2 kubelet[21644]: I0212 12:16:27.875752   21644 feature_gate.go:206] feature gates: &{map[]}
Feb 12 12:16:28 k8s-node-2 kubelet[21644]: I0212 12:16:28.452883   21644 server.go:825] Using self-signed cert (/k8s/kubernetes/ssl/kubelet.crt, /k8s/kubernetes/ssl/kubelet.key)
Feb 12 12:16:28 k8s-node-2 kubelet[21644]: I0212 12:16:28.471237   21644 mount_linux.go:179] Detected OS with systemd
Feb 12 12:16:28 k8s-node-2 kubelet[21644]: I0212 12:16:28.471287   21644 server.go:407] Version: v1.13.1
Feb 12 12:16:28 k8s-node-2 kubelet[21644]: I0212 12:16:28.471327   21644 feature_gate.go:206] feature gates: &{map[]}
Feb 12 12:16:28 k8s-node-2 kubelet[21644]: I0212 12:16:28.471365   21644 feature_gate.go:206] feature gates: &{map[]}
Feb 12 12:16:28 k8s-node-2 kubelet[21644]: I0212 12:16:28.471427   21644 plugins.go:103] No cloud provider specified.
Feb 12 12:16:28 k8s-node-2 kubelet[21644]: I0212 12:16:28.471435   21644 server.go:523] No cloud provider specified: "" from the config file: ""
Feb 12 12:16:28 k8s-node-2 kubelet[21644]: I0212 12:16:28.471455   21644 bootstrap.go:65] Using bootstrap kubeconfig to generate TLS client cert, key and kubeconfig file
Feb 12 12:16:28 k8s-node-2 kubelet[21644]: I0212 12:16:28.473080   21644 bootstrap.go:96] No valid private key and/or certificate found, reusing existing private key or creating a new one
``` 

在master上对node节点的csr进行授权

```
 kubectl get nodes  
 kubectl get csr 
 kubectl certificate approve node-csr-cZCxO0ThIHUDCyjkKwglJU9AQXYwrKv6c8orKQ8tDPg 
 kubectl get nodes  
```
执行结果
```
[root@k8s-master bin]#  kubectl get nodes  
 kubectl get csr NAME            STATUS   ROLES    AGE   VERSION
192.168.50.21   Ready    <none>   15m   v1.13.1
[root@k8s-master bin]#  kubectl get csr 
NAME                                                   AGE   REQUESTOR           CONDITION
node-csr-GEpA4trrDK_r1By85WO6VEeMIvKYlzhVu4WM9AqffQU   23m   kubelet-bootstrap   Approved,Issued
node-csr-cZCxO0ThIHUDCyjkKwglJU9AQXYwrKv6c8orKQ8tDPg   63s   kubelet-bootstrap   Pending
[root@k8s-master bin]#  kubectl certificate approve node-csr-cZCxO0ThIHUDCyjkKwglJU9AQXYwrKv6c8orKQ8tDPg 
certificatesigningrequest.certificates.k8s.io/node-csr-cZCxO0ThIHUDCyjkKwglJU9AQXYwrKv6c8orKQ8tDPg approved
[root@k8s-master bin]# kubectl get nodes  
NAME            STATUS   ROLES    AGE   VERSION
192.168.50.21   Ready    <none>   15m   v1.13.1
192.168.50.22   Ready    <none>   28s   v1.13.1
```




#### 部署 kube-proxy组件

kube-proxy 运行在所有 node节点上，它监听 apiserver 中 service 和 Endpoint 的变化情况，创建路由规则来进行服务负载均衡
> 此刻回到node-2 服务器
##### 创建 kube-proxy 配置文件
执行命令
```
vim /k8s/kubernetes/cfg/kube-proxy

```
输入内容
```
KUBE_PROXY_OPTS="--logtostderr=true \
--v=4 \
--hostname-override=192.168.50.22 \
--cluster-cidr=192.168.0.0/16 \
--kubeconfig=/k8s/kubernetes/cfg/kube-proxy.kubeconfig"
```

##### 创建kube-proxy systemd文件
执行命令
```
vim /usr/lib/systemd/system/kube-proxy.service 
```
输入内容
```
[Unit]
Description=Kubernetes Proxy
After=network.target
 
[Service]
EnvironmentFile=-/k8s/kubernetes/cfg/kube-proxy
ExecStart=/k8s/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
```

##### 启动 kube-proxy  服务
```
systemctl daemon-reload
systemctl enable kube-proxy 
systemctl start kube-proxy
```

查看服务状态
```
[root@k8s-node-1 cfg]# systemctl status  kube-proxy
● kube-proxy.service - Kubernetes Proxy
   Loaded: loaded (/usr/lib/systemd/system/kube-proxy.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-02-12 12:27:40 EST; 21s ago
 Main PID: 21802 (kube-proxy)
    Tasks: 0
   Memory: 10.8M
   CGroup: /system.slice/kube-proxy.service
           ‣ 21802 /k8s/kubernetes/bin/kube-proxy --logtostderr=true --v=4 --hostname-override=192.168.50.22 --cluster-cidr=192.168.0.0/16 --kubeconfig=/k8s/kubernetes/cfg/kube-proxy.kubeconfig

Feb 12 12:27:53 k8s-node-1 kube-proxy[21802]: I0212 12:27:53.299890   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:54 k8s-node-1 kube-proxy[21802]: I0212 12:27:54.270917   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:55 k8s-node-1 kube-proxy[21802]: I0212 12:27:55.311513   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:56 k8s-node-1 kube-proxy[21802]: I0212 12:27:56.276286   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:57 k8s-node-1 kube-proxy[21802]: I0212 12:27:57.323746   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:58 k8s-node-1 kube-proxy[21802]: I0212 12:27:58.302454   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:27:59 k8s-node-1 kube-proxy[21802]: I0212 12:27:59.332524   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:28:00 k8s-node-1 kube-proxy[21802]: I0212 12:28:00.332119   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:28:01 k8s-node-1 kube-proxy[21802]: I0212 12:28:01.343378   21802 config.go:141] Calling handler.OnEndpointsUpdate
Feb 12 12:28:02 k8s-node-1 kube-proxy[21802]: I0212 12:28:02.345592   21802 config.go:141] Calling handler.OnEndpointsUpdate
``` 



 ## Flanneld网络部署
>默认没有flanneld网络，Node节点间的pod不能通信，只能Node内通信，为了部署步骤简洁明了，故flanneld放在后面安装
flannel服务需要先于docker启动。flannel服务启动时主要做了以下几步的工作：
从etcd中获取network的配置信息
划分subnet，并在etcd中进行注册
将子网信息记录到/run/flannel/subnet.env中


> master 上无需部署本服务

> 所有子节点操作一致， 所以就写一个子节点的操作
#### etcd注册网段
执行命令
```
/k8s/etcd/bin/etcdctl --ca-file=/k8s/etcd/ssl/ca.pem --cert-file=/k8s/etcd/ssl/server.pem --key-file=/k8s/etcd/ssl/server-key.pem --endpoints="https://192.168.50.10:2379,https://192.168.50.21:2379,https://192.168.50.22:2379"  set /k8s/network/config  '{ "Network": "192.168.0.0/16", "Backend": {"Type": "vxlan"}}'
```
输出结果
```
[root@k8s-master bin]# /k8s/etcd/bin/etcdctl --ca-file=/k8s/etcd/ssl/ca.pem --cert-file=/k8s/etcd/ssl/server.pem --key-file=/k8s/etcd/ssl/server-key.pem --endpoints="https://192.168.50.10:2379,https://192.168.50.21:2379,https://192.168.50.22:2379"  set /k8s/network/config  '{ "Network": "192.168.0.0/16", "Backend": {"Type": "vxlan"}}'
{ "Network": "192.168.0.0/16", "Backend": {"Type": "vxlan"}}
```
>flanneld 当前版本 (v0.10.0) 不支持 etcd v3，故使用 etcd v2 API 写入配置 key 和网段数据；
写入的 Pod 网段 ${CLUSTER_CIDR} 必须是 /16 段地址，必须与 kube-controller-manager 的 --cluster-cidr 参数值一致；


####  flannel安装
##### 解压安装
```
tar -xvf flannel-v0.10.0-linux-amd64.tar.gz
mv flanneld mk-docker-opts.sh /k8s/kubernetes/bin/
```
##### 配置flanneld

执行命令
```
vim /k8s/kubernetes/cfg/flanneld
```
输入内容
```
FLANNEL_OPTIONS="--etcd-endpoints=https://192.168.50.10:2379,https://192.168.50.21:2379,https://192.168.50.22:2379 -etcd-cafile=/k8s/etcd/ssl/ca.pem -etcd-certfile=/k8s/etcd/ssl/server.pem -etcd-keyfile=/k8s/etcd/ssl/server-key.pem -etcd-prefix=/k8s/network"
```
创建flanneld systemd文件
```
vim /usr/lib/systemd/system/flanneld.service
```
输入内容
```
[Unit]
Description=Flanneld overlay address etcd agent
After=network-online.target network.target
Before=docker.service
 
[Service]
Type=notify
EnvironmentFile=/k8s/kubernetes/cfg/flanneld
ExecStart=/k8s/kubernetes/bin/flanneld --ip-masq $FLANNEL_OPTIONS
ExecStartPost=/k8s/kubernetes/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
```
注意
>mk-docker-opts.sh 脚本将分配给 flanneld 的 Pod 子网网段信息写入 /run/flannel/docker 文件，后续 docker 启动时 使用这个文件中的环境变量配置 docker0 网桥；
flanneld 使用系统缺省路由所在的接口与其它节点通信，对于有多个网络接口（如内网和公网）的节点，可以用 -iface 参数指定通信接口;
flanneld 运行时需要 root 权限；


##### 配置Docker启动指定子网
> 
> 修改
> EnvironmentFile=/run/flannel/subnet.env

> ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
> 即可
```
vim /usr/lib/systemd/system/docker.service 
```
按下面的提示修改内容
```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
 
[Service]
Type=notify
EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
 
[Install]
WantedBy=multi-user.target
```

##### 启动服务
注意启动flannel前要关闭docker及相关的kubelet这样flannel才会覆盖docker0网桥
```
systemctl daemon-reload
systemctl stop docker
systemctl enable flanneld
systemctl start flanneld
systemctl start docker
systemctl restart kubelet
systemctl restart kube-proxy
```

##### 验证服务
```
[root@k8s-node-1 k8s]# cat /run/flannel/subnet.env 
DOCKER_OPT_BIP="--bip=192.168.43.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=false"
DOCKER_OPT_MTU="--mtu=1450"
DOCKER_NETWORK_OPTIONS=" --bip=192.168.43.1/24 --ip-masq=false --mtu=1450"



[root@k8s-node-1 k8s]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:a8:6c:2b brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.21/24 brd 192.168.50.255 scope global noprefixroute dynamic ens33
       valid_lft 80004sec preferred_lft 80004sec
    inet6 fe80::d525:988a:83cb:61c3/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:81:fc:a4:1d brd ff:ff:ff:ff:ff:ff
    inet 192.168.43.1/24 brd 192.168.43.255 scope global docker0
       valid_lft forever preferred_lft forever
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    link/ether fe:1e:3c:c0:83:77 brd ff:ff:ff:ff:ff:ff
    inet 192.168.43.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::fc1e:3cff:fec0:8377/64 scope link 
       valid_lft forever preferred_lft forever
```
