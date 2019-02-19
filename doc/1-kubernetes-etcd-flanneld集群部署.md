
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
```
k8s-master1	192.168.50.10	k8s-master	etcd、kube-apiserver、kube-controller-manager、kube-scheduler
k8s-node1	192.168.50.21	k8s-node	etcd、kubelet、docker、kube_proxy
k8s-node2	192.168.50.22	k8s-node	etcd、kubelet、docker、kube_proxy
```

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
```

##MASTER 服务器，基础安装 和证书生成

### 下载软件
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
vim /k8s/etcd/cfg/etcd.conf  
```
etcd.conf  文件内容
``` 
#[Member]
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/data1/etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.50.10:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.50.10:2379,http://127.0.0.1:2379"
 
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.50.10:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.50.10:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.50.10:2380,etcd02=https://192.168.50.21:2380,etcd03=https://192.168.50.21:2380"
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

配置服务，但此时不启动etcd01
```
systemctl daemon-reload
systemctl enable etcd
```


## etcd.step.3

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
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.50.10:2380,etcd02=https://192.168.50.21:2380,etcd03=https://192.168.50.21:2380"
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
