# etcd test
## 1.etcd-member-list
### Download and install

```sh
ETCD_VER=v3.4.17
DOWNLOAD_URL=https://github.com/etcd-io/etcd/releases/download
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test
```

### Start etcd locally

为避免跟本地的 hostNetwork 的 etcd 容器冲突，我们需要修改 etcd 的监听端口

- `initial-cluster`：初始化集群，需要列所有 member 地址

> What is the difference between listen-<client,peer>-urls, advertise-client-urls or initial-advertise-peer-urls?
>
> listen-client-urls and listen-peer-urls specify the local addresses etcd server binds to for accepting incoming connections. To listen on a port for all interfaces, specify 0.0.0.0 as the listen IP address.
>
> advertise-client-urls and initial-advertise-peer-urls specify the addresses etcd clients or other etcd members should use to contact the etcd server. The advertise addresses must be reachable from the remote machines. Do not advertise addresses like localhost or 0.0.0.0 for a production setup since these addresses are unreachable from remote machines.

```sh
etcd --listen-client-urls 'http://localhost:12379' \
 --advertise-client-urls 'http://localhost:12379' \
 --listen-peer-urls 'http://localhost:12380' \
 --initial-advertise-peer-urls 'http://localhost:12380' \
 --initial-cluster 'default=http://localhost:12380'
```

### Member list

```sh
etcdctl member list --write-out=table --endpoints=localhost:12379
```

```sh
etcdctl --endpoints=localhost:12379 put /key1 val1
etcdctl --endpoints=localhost:12379 put /key2 val2
etcdctl --endpoints=localhost:12379 get --prefix /
etcdctl --endpoints=localhost:12379 get --prefix / --keys-only
etcdctl --endpoints=localhost:12379 watch --prefix /
```

```sh
etcdctl --endpoints=localhost:12379 put /key val1
etcdctl --endpoints=localhost:12379 put /key val2
etcdctl --endpoints=localhost:12379 put /key val3
etcdctl --endpoints=localhost:12379 put /key val4
etcdctl --endpoints=localhost:12379 get /key -wjson
etcdctl --endpoints=localhost:12379 watch --prefix / --rev 0
etcdctl --endpoints=localhost:12379 watch --prefix / --rev 1
etcdctl --endpoints=localhost:12379 watch --prefix / --rev 2
```
## 2.put-data
### 启动新 etcd 集群

```sh
docker run -d registry.aliyuncs.com/google_containers/etcd:3.5.0-0 /usr/local/bin/etcd
```

### 进入 etcd 容器

```sh
docker ps|grep etcd
docker exec –it <containerid> sh
```

### 存入数据

```sh
etcdctl put x 0
```

### 读取数据

```sh
etcdctl get x -w=json
{"header":{"cluster_id":14841639068965178418,"member_id":10276657743932975437,"revision":2,"raft_term":2},"kvs":[{"key":"eA==","create_revision":2,"mod_revision":2,"version":1,"value":"MA=="}],"count":1}
```

### 修改值

```sh
etcdctl put x 1
```

### 查询最新值

```sh
etcdctl get x
x
1
```

### 查询历史版本值

```sh
etcdctl get x --rev=2
x
0
```
## 2.2.lease
设置 Key 的生存周期: grant
```sh
etcdctl --endpoints=localhost:12379 lease grant 30
etcdctl --endpoints=localhost:12379 lease list
etcdctl --endpoints=localhost:12379 --lease 1cf77c6926d4620e put /a b
etcdctl --endpoints=localhost:12379 get --prefix /
etcdctl --endpoints=localhost:12379 lease keep-alive 1cf77c6926d4620e
```
## 3.dr
### 创建 Snapshot

```sh
etcdctl --endpoints https://127.0.0.1:3379 --cert /tmp/etcd-certs/certs/127.0.0.1.pem --key /tmp/etcd-certs/certs/127.0.0.1-key.pem --cacert /tmp/etcd-certs/certs/ca.pem snapshot save snapshot.db
```

### 恢复数据

```sh
etcdctl snapshot restore snapshot.db \
--name infra2 \
--data-dir=/tmp/etcd/infra2 \
--initial-cluster infra0=http://127.0.0.1:3380,infra1=http://127.0.0.1:4380,infra2=http://127.0.0.1:5380 \
--initial-cluster-token etcd-cluster-1 \
--initial-advertise-peer-urls http://127.0.0.1:5380
```
## 4.alarm
### 设置 etcd 存储大小

```sh
etcd --quota-backend-bytes=$((16*1024*1024))
```

### 写爆磁盘

```sh
while [ 1 ]; do dd if=/dev/urandom bs=1024 count=1024 | ETCDCTL_API=3 etcdctl put key || break; done
```

### 查看 endpoint 状态

```sh
ETCDCTL_API=3 etcdctl --write-out=table endpoint status
```

### 查看 alarm

```sh
ETCDCTL_API=3 etcdctl alarm list
```

### 清理碎片

```sh
ETCDCTL_API=3 etcdctl defrag
```

### 清理 alarm

```sh
ETCDCTL_API=3 etcdctl alarm disarm
```
## 5.defrag
### Keep one hour of history

```sh
etcd --auto-compaction-retention=1
```

### Compact up to revision 3

```sh
etcdctl compact 3
etcdctl defrag

Finished defragmenting etcd member[127.0.0.1:2379]
```
## 6.statefulset-etcd
### Install helm

https://github.com/helm/helm/releases

### Download bitnami etcd

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm pull bitnami/etcd
tar -xvf etcd-6.8.4.tgz
vi etcd/values.yaml
```

And set persistence to false:

```yaml
persistence:
  ## @param persistence.enabled If true, use a Persistent Volume Claim. If false, use emptyDir.
  ##
  enabled: false
```

### Install etcd by helm chart

```sh
helm install my-release ./etcd
```

### Start etcd client

```sh
kubectl run my-release-etcd-client --restart='Never' --image docker.io/bitnami/etcd:3.5.0-debian-10-r94 --env ROOT_PASSWORD=$(kubectl get secret --namespace default my-release-etcd -o jsonpath="{.data.etcd-root-password}" | base64 --decode) --env ETCDCTL_ENDPOINTS="my-release-etcd.default.svc.cluster.local:2379" --namespace default --command -- sleep infinity
```

```sh
kubectl exec --namespace default -it my-release-etcd-client -- bash
etcdctl --user root:$ROOT_PASSWORD put /message Hello
etcdctl --user root:$ROOT_PASSWORD get /message
```
## etcd-binary-setup
### 创建工作目录

```sh
mkdir -p /data/k8s-work
cd /data/k8s-work
```

### 下载 cfssl 工具

```sh
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64

chmod +x cfssl*
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

### 生成 ca 配置文件

```sh
cat > ca-csr.json <<"EOF"
{
"CN": "kubernetes",
"key": {
"algo": "rsa",
"size": 2048
},
"names": [
{
"C": "CN",
"ST": "Shanghai",
"L": "Shanghai",
"O": "cncamp",
"OU": "cncamp"
}
],
"ca": {
"expiry": "87600h"
}
}
EOF
```

### 生成 ca 证书文件

```sh
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

### 配置 ca 证书策略

```sh
cat > ca-config.json <<"EOF"
{
  "signing": {
      "default": {
          "expiry": "87600h"
        },
      "profiles": {
          "kubernetes": {
              "usages": [
                  "signing",
                  "key encipherment",
                  "server auth",
                  "client auth"
              ],
              "expiry": "87600h"
          }
      }
  }
}
EOF
```

### 配置 etcd 请求 csr 文件

```sh
cat > etcd-csr.json <<"EOF"
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.34.2"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "CN",
    "ST": "Shanghai",
    "L": "Shanghai",
    "O": "cncamp",
    "OU": "cncamp"
  }]
}
EOF
```

### 生成证书

```sh
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson  -bare etcd
```

```sh
wget https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz
tar -xvf etcd-v3.5.0-linux-amd64.tar.gz
cp -p etcd-v3.5.0-linux-amd64/etcd* /usr/local/bin/
```

### 生成 etcd 配置文件

```sh
cat >  etcd.conf <<"EOF"
#[Member]
ETCD_NAME="etcd1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.34.2:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.34.2:2379,http://127.0.0.1:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.34.2:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.34.2:2379"
ETCD_INITIAL_CLUSTER="etcd1=https://192.168.34.2:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

### Reference

https://www.jianshu.com/p/b02c428950df

## etcd-k8s
```sh
alias ks=kubectl -n kube-system
ks get po etcd-cadmin -oyaml
```

```yaml
spec:
  containers:
    - command:
        - etcd
        - --advertise-client-urls=https://192.168.34.2:2379
        - --cert-file=/etc/kubernetes/pki/etcd/server.crt
        - --client-cert-auth=true
        - --data-dir=/var/lib/etcd
        - --initial-advertise-peer-urls=https://192.168.34.2:2380
        - --initial-cluster=cadmin=https://192.168.34.2:2380
        - --key-file=/etc/kubernetes/pki/etcd/server.key
        - --listen-client-urls=https://127.0.0.1:2379,https://192.168.34.2:2379
        - --listen-metrics-urls=http://127.0.0.1:2381
        - --listen-peer-urls=https://192.168.34.2:2380
        - --name=cadmin
        - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
        - --peer-client-cert-auth=true
        - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
        - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
        - --snapshot-count=10000
        - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

```sh
ks exec -it etcd-cadmin sh
ETCDCTL_API=3
alias ectl='etcdctl --endpoints https://127.0.0.1:2379 \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key'

ectl member list

ectl get --prefix --keys-only /

ectl get /registry/services/specs/kube-system/kube-dns -w=json
{"header":{"cluster_id":10396955284310292238,"member_id":2496364209257137704,"revision":596482,"raft_term":8},"kvs":[{"key":"L3JlZ2lzdHJ5L3NlcnZpY2VzL3NwZWNzL2t1YmUtc3lzdGVtL2t1YmUtZG5z","create_revision":251,"mod_revision":251,"version":1,"value":"azhzAAoNCgJ2MRIHU2VydmljZRLfCAqbBwoIa3ViZS1kbnMSABoLa3ViZS1zeXN0ZW0iACokZTc0MmExMWYtMzY4NC00ZThkLWJhOGEtZWZiY2E2YTM0YzMwMgA4AEIICL+hx4oGEABaEwoHazhzLWFwcBIIa3ViZS1kbnNaJQoda3ViZXJuZXRlcy5pby9jbHVzdGVyLXNlcnZpY2USBHRydWVaHQoSa3ViZXJuZXRlcy5pby9uYW1lEgdDb3JlRE5TYhoKEnByb21ldGhldXMuaW8vcG9ydBIEOTE1M2IcChRwcm9tZXRoZXVzLmlvL3NjcmFwZRIEdHJ1ZXoAigGxBQoHa3ViZWFkbRIGVXBkYXRlGgJ2MSIICL+hx4oGEAAyCEZpZWxkc1YxOoMFCoAFeyJmOm1ldGFkYXRhIjp7ImY6YW5ub3RhdGlvbnMiOnsiLiI6e30sImY6cHJvbWV0aGV1cy5pby9wb3J0Ijp7fSwiZjpwcm9tZXRoZXVzLmlvL3NjcmFwZSI6e319LCJmOmxhYmVscyI6eyIuIjp7fSwiZjprOHMtYXBwIjp7fSwiZjprdWJlcm5ldGVzLmlvL2NsdXN0ZXItc2VydmljZSI6e30sImY6a3ViZXJuZXRlcy5pby9uYW1lIjp7fX19LCJmOnNwZWMiOnsiZjpjbHVzdGVySVAiOnt9LCJmOmludGVybmFsVHJhZmZpY1BvbGljeSI6e30sImY6cG9ydHMiOnsiLiI6e30sIms6e1wicG9ydFwiOjUzLFwicHJvdG9jb2xcIjpcIlRDUFwifSI6eyIuIjp7fSwiZjpuYW1lIjp7fSwiZjpwb3J0Ijp7fSwiZjpwcm90b2NvbCI6e30sImY6dGFyZ2V0UG9ydCI6e319LCJrOntcInBvcnRcIjo1MyxcInByb3RvY29sXCI6XCJVRFBcIn0iOnsiLiI6e30sImY6bmFtZSI6e30sImY6cG9ydCI6e30sImY6cHJvdG9jb2wiOnt9LCJmOnRhcmdldFBvcnQiOnt9fSwiazp7XCJwb3J0XCI6OTE1MyxcInByb3RvY29sXCI6XCJUQ1BcIn0iOnsiLiI6e30sImY6bmFtZSI6e30sImY6cG9ydCI6e30sImY6cHJvdG9jb2wiOnt9LCJmOnRhcmdldFBvcnQiOnt9fX0sImY6c2VsZWN0b3IiOnt9LCJmOnNlc3Npb25BZmZpbml0eSI6e30sImY6dHlwZSI6e319fUIAEroBChYKA2RucxIDVURQGDUiBggAEDUaACgAChoKB2Rucy10Y3ASA1RDUBg1IgYIABA1GgAoAAocCgdtZXRyaWNzEgNUQ1AYwUciBwgAEMFHGgAoABITCgdrOHMtYXBwEghrdWJlLWRucxoKMTAuOTYuMC4xMCIJQ2x1c3RlcklQOgROb25lQgBSAFoAYABoAIoBC1NpbmdsZVN0YWNrkgEKMTAuOTYuMC4xMJoBBElQdjSyAQdDbHVzdGVyGgIKABoAIgA="}],"count":1}

# 慎用
etcl delete
```

## 资料
### Kubernetes control plane component: etcd

### References

- [etcd v3.5 docs](https://etcd.io/docs/v3.5/)

- [How etcd works with and without Kubernetes](https://learnk8s.io/etcd-kubernetes)
