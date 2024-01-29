# K8s

### Master

* API Server

  系统唯一入口、RESTful API

* etcd

  KV数据库、高可用、只能通过apiserver访问

* Controller Manager

  监控运维

* Kube Scheduler

  调度Pod运行

  

### Worker

* Kubelet

  与apiserver通信、结点的代理

* Kube Proxy

  转发TCP/UDP、实现反向代理

* Container runtime 

  OCI标准

  eg: Docker | CRI-O|containerd









### minikube



###### install

```shell
# Intel x86_64
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Apple arm64
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-arm64

sudo install minikube /usr/local/bin/
```



```shell
minikube version
```



### kubectl

```
curl -Lo kubectl    http://kubernetes.oss-cn-hangzhou.aliyuncs.com/kubernetes-release/release/v1.22.1/bin/linux/amd64/kubectl
mv kubectl /usr/bin
chmod a+x /usr/bin/kubectl
```



``` shell
alias kubectl="minikube kubectl --"
```

自动补全

```shell
source <(kubectl completion bash)
```

#### API对象

**apiVersion、kind、metadata、spec**



##### spec

1、containers

* ports
* imagePullPolicy
* env
* command
* args





`kubectl api-resources`

`kubectl explain`  

`--dry-run=client -o yaml` 生成YAML模板



```yml
apiVersion: v1
kind: Pod
metadata:
  name: ngx-pod
  labels:
    env: demo
    owner: chrono

spec:
  containers:
  - image: nginx:alpine
    name: ngx
    ports:
    - containerPort: 80
```







`kubectl apply`	

`kubectl delete`





##### Pod

`kubectl run`





##### Job

`kubectl explain job`

`kubectl create`



* activeDeadlineSeconds

  设置Pod运行的超时时间

* backoffLimit

  设置Pod的失败重试次数

* completions

  Job完成需要运行多少个Pod，默认是1个

* parallelism

  与completions相关，表示允许并发运行的Pod数量，避免过多占用资源



cron-job.yml

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: echo-cj
spec:
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - image: busybox
            name: echo-cj
            imagePullPolicy: IfNotPresent
            command: ["/bin/echo"]
            args: ["hello", "world"]
          restartPolicy: OnFailure
  schedule: "*/1 * * * *"
```



##### ConfigMap/Secret

env-pod.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod

spec:
  containers:
  - env:
      - name: COUNT
        valueFrom:
          configMapKeyRef:
            name: info
            key: count
      - name: GREETING
        valueFrom:
          configMapKeyRef:
            name: info
            key: greeting
      - name: USERNAME
        valueFrom:
          secretKeyRef:
            name: user
            key: name
      - name: PASSWORD
        valueFrom:
          secretKeyRef:
            name: user
            key: pwd

    image: busybox
    name: busy
    imagePullPolicy: IfNotPresent
    command: ["/bin/sleep", "300"]
```



###### volume

vol-pod.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vol-pod

spec:
  volumes:
  - name: cm-vol
    configMap:
      name: info
  - name: sec-vol
    secret:
      secretName: user
      
  containers:
  - volumeMounts:
    - mountPath: /tmp/cm-items
      name: cm-vol
    - mountPath: /tmp/sec-items
      name: sec-vol
      
    image: busybox
    name: busy
    imagePullPolicy: IfNotPresent
    command: ["/bin/sleep", "300"]
```



### Demo

###### MariaDB

maria-cm.yml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: maria-cm
  
data:
  DATABASE: 'db'
  USER: 'wp'
  PASSWORD: '123'
  ROOT_PASSWORD: '123'
```

mariadb-pod.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: maria-pod
  labels:
    app: wordpress
    role: database
  
spec:
  containers:
  - image: mariadb:10
    name: maria
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 3306
    
    envFrom:
    - prefix: 'MARIADB_'
      configMapRef:
        name: maria-cm
```

###### wordpress

wp-cm.yml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wp-cm
  
data:
  HOST: '172.17.0.7'
  USER: 'wp'
  PASSWORD: '123'
  NAME: 'db'
```

wp-pod.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: wp-pod
  labels:
    app: wordpress
    role: website
    
spec:
  containers:
  - image: wordpress:5
    name: wp-pod
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 80
    
    envFrom:
    - prefix: "WORDPRESS_DB_"
      configMapRef:
        name: wp-cm
```

```shell
kubectl port-forward wp-pod 8080:80 &
```



proxy.conf

```json
server {
  listen 80;
  default_type text/html;

  location / {
      proxy_http_version 1.1;
      proxy_set_header Host $host;
      proxy_pass http://127.0.0.1:8080;
  }
}
```



```shell
docker run -d --rm \
    --net=host \
    -v ./proxy.conf:/etc/nginx/conf.d/default.conf \
    nginx:alpine
```



### verification env



```shell
minikube start --kubernetes-version=v1.23.3 --image-mirror-country='cn' --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers' --force
```

有效

```shell
minikube start  --kubernetes-version=v1.23.3 --driver=docker  --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers' --force
```



### kubeadm

`init` `join` `upgrade` `reset`

#### 准备工作

```shell
# 1、修改每个节点主机名
sudo vi /etc/hostname

# 2、Docker作为container runtime底层支持
# 修改docker配置daemon.json
# 把cgroup的驱动程序改成systemd，然后重启Docker守护进程
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker

# 3、让kubernetes检查、转发网络流量
# 修改iptables配置，启动'br_netfilter'模块
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1 # better than modify /etc/sysctl.conf
EOF

sudo sysctl --system

# 4、修改'/etc/fstb', 关闭swap分区，提升kubernetes的性能
sudo swapoff -a
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
```

#### 安装kubeadm

```shell
# Google下载不稳定，选择其它软件源
sudo apt install -y apt-transport-https ca-certificates curl

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

sudo apt update


# kubeadm、kubelet、kubectl
sudo apt install -y kubeadm=1.23.3-00 kubelet=1.23.3-00 kubectl=1.23.3-00

# verify version
kubeadm version
kubectl version --client

# lock version
sudo apt-mark hold kubeadm kubelet kubectl
```



#### 下载Kubernetes组件镜像

```shell
# apiserver|etcd|scheduler
# 国内访问困难，把镜像下载到本地
kubeadm config images list --kubernetes-version v1.23.3

k8s.gcr.io/kube-apiserver:v1.23.3
k8s.gcr.io/kube-controller-manager:v1.23.3
k8s.gcr.io/kube-scheduler:v1.23.3
k8s.gcr.io/kube-proxy:v1.23.3
k8s.gcr.io/pause:3.6
k8s.gcr.io/etcd:3.5.1-0
k8s.gcr.io/coredns/coredns:v1.8.6

# 第一种，minikube打包，拷贝
# 第二种，从国内的镜像网站下载，然后改名
repo=registry.aliyuncs.com/google_containers

for name in `kubeadm config images list --kubernetes-version v1.23.3`; do

    src_name=${name#k8s.gcr.io/}
    src_name=${src_name#coredns/}

    docker pull $repo/$src_name

    docker tag $repo/$src_name $name
    docker rmi $repo/$src_name
done


## 结合，两种方法镜像IMAGE ID比较
```

#### 安装Master节点

* `--pod-network-cidr` 设置集群里Pod的IP地址
* `--apiserver-advertise-address` 设置apiserver的IP地址
* `--kubernetes-version` 指定kubernetes的版本号

```shell
sudo kubeadm init \
    --pod-network-cidr=10.10.0.0/16 \
    --apiserver-advertise-address=192.168.10.210 \
    --kubernetes-version=v1.23.3

# step 1
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
# step 2
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.10.210:6443 --token tv9mkx.tw7it9vphe158e74 \
  --discovery-token-ca-cert-hash sha256:e8721b8630d5b562e23c010c70559a6d3084f629abad6a2920e87855f8fb96f3
  
# verify version
kubectl version
kubectl get node
```



#### 安装Flannel网络插件

https://github.com/flannel-io/flannel/

```shell
# 把Network改成'--pod-network-cidr'设置的地址段 
net-conf.json: |
    {
      "Network": "10.10.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
    
# install
kubectl apply -f kube-flannel.yml

# verify
kubectl get node
```



#### 安装Worker节点

```shell
sudo \
kubeadm join 192.168.10.210:6443 --token tv9mkx.tw7it9vphe158e74 \
  --discovery-token-ca-cert-hash sha256:e8721b8630d5b562e23c010c70559a6d3084f629abad6a2920e87855f8fb96f3
  
# verify
kubectl get node

kubectl run ngx --image=nginx:alpine
kubectl get pod -o wide
```



#### Console节点

```shell
scp `which kubectl` chrono@192.168.10.208:~/
scp ~/.kube/config chrono@192.168.10.208:~/.kube
```



#### 注意点

* 安装Kubernetes之前需要修改主机的配置，包括主机名、Docker配置、网络设置、交换分区等
* Kubernetes组件镜像放在gcr.io，国内下载比较麻烦，可以考虑从minikube或者国内镜像网站获取
* 安装Master节点需要使用命令`kubeadm init`，安装Worker节点需要使用命令`kubeadm join`，还要部署Flannel等网络插件才能让集群正常工作



https://github.com/chronolaw/k8s_study/tree/master/admin





### Deployment

管理Pod，让应用不宕机

`kubectl api-resources`



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: xxx-dep
```



```shell
export out="--dry-run=client -o yaml"
kubectl create deploy ngx-dep --image=nginx:alpine $out
```

example.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: ngx-dep
  name: ngx-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ngx-dep
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: ngx-dep
    spec:
      containers:
      - image: nginx:alpine
        name: nginx
        resources: {}
status: {}
```

example-deploy.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: ngx-dep
  name: ngx-dep
  
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ngx-dep
      
  template:
    metadata:
      labels:
        app: ngx-dep
    spec:
      containers:
      - image: nginx:alpine
        name: nginx
```

`replicas` `selector`

#### replicas

期望副本数量



#### selector

筛选要被管理的Pod对象

`selector.matchLabels` 与`template.metadata.labels` ， `app: ngx-dep`



#### kubectl scale

```shell
kubectl scale --replicas=5 deploy ngx-dep
```





### DaemonSet

* 目标是在集群的每个节点上运行且仅运行一个Pod
* 跟Deployment类似，管理控制Pod，但管理调度策略不同。

特殊业务需求，每个节点都要运行Pod

* 网络应用（如kube-proxy）
* 监控应用（如Prometheus）监控节点的状态、实时上报信息
* 日志应用（如Fluentd）搜集容器运行时产生的日志数据
* 安全应用  执行安全审计、入侵检查、漏洞扫描等

daemonset template

https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/daemonset/



`kubectl api-resources`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: xxx-ds
```



template

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: redis-ds
  labels:
    app: redis-ds

spec:
  selector:
    matchLabels:
      name: redis-ds

  template:
    metadata:
      labels:
        name: redis-ds
    spec:
      containers:
      - image: redis:5-alpine
        name: redis
        ports:
        - containerPort: 6379
```



```yaml
export out="--dry-run=client -o yaml"

# change "kind" to DaemonSet
# delete "spec.replicas"
kubectl create deploy redis-ds --image=redis:5-alpine $out
```



#### Taint&Toleration

https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/taint-and-toleration/

`kubectl describe node`



##### `kubectl taint` 

`kubectl taint` + `节点名`  `污点名`  `污点` 

去掉污点+ `-`

```shell
kubectl taint node master node-role.kubernetes.io/master:NoSchedule-
```



tolerations

` kubectl explain ds.spec.template.spec.tolerations`

```yaml
tolerations:
- key: node-role.kubernetes.io/master
  effect: NoSchedule
  operator: Exists
```



### Service

`TCP/IP`

`kubectl expose`  `--port` `--target-port`



```yaml
apiVersion: v1
kind: Service
metadata:
  name: xxx-svc
```



```shell
export out="--dry-run=client -o yaml"
kubectl expose deploy ngx-dep --port=80 --target-port=80 $out
```

example-svc.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ngx-svc
  
spec:
  selector:
    app: ngx-dep
    
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```



ngx-conf.conf

```json
apiVersion: v1
kind: ConfigMap
metadata:
  name: ngx-conf

data:
  default.conf: |
    server {
      listen 80;
      location / {
        default_type text/plain;
        return 200
          'srv : $server_addr:$server_port\nhost: $hostname\nuri : $request_method $host $request_uri\ndate: $time_iso8601\n';
      }
    }
```

ngx-dep-deploy.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx-dep

spec:
  replicas: 2
  selector:
    matchLabels:
      app: ngx-dep

  template:
    metadata:
      labels:
        app: ngx-dep
    spec:
      volumes:
      - name: ngx-conf-vol
        configMap:
          name: ngx-conf

      containers:
      - image: nginx:alpine
        name: nginx
        ports:
        - containerPort: 80

        volumeMounts:
        - mountPath: /etc/nginx/conf.d
          name: ngx-conf-vol
```

```shell
# startup
kubectl apply -f cm.yml
kubectl apply -f deploy.yml
kubectl apply -f svc.yml

# verify
kubectl exec -it ngx-dep-6796688696-r2j6t -- sh
```



以域名方式使用Service，"**IP地址.命名空间.pod.cluster.local**"，需要把IP地址里的"`.`"改成"`-`"

Service对象由一个关键字段"**type**"，默认"**ClusterIP**",还有其它三种类型"**ExternalName**","**LoadBalancer**","**NodePort**" 。前两种一般由云服务厂商提供

#### NodePort

`kubectl expose`  `--type=NodePort`

```yaml
apiVersion: v1
...
spec:
  ...
  type: NodePort
```

问题点

* 端口数量有限，30000~32767
* 会在每个节点都开端口，使用kube-proxy路由到真正的后端Service，对于有很多计算节点的大集群来说会带来一些通信成本
* 它要求向外界暴露节点的IP地址，为了安全还需要在集群外再搭一个反向代理，增加了方案的复杂度

小结

* Pod生命周期很短暂，会不停地创建销毁，所以需要用Service来实现负载均衡。它由Kubernetes分配固定的IP地址，能够屏蔽后端的Pod变化
* Service对象使用与Deployment、DaemonSet相同的"selector"字段，选择要代理的后端Pod，是松耦合关系
* 基于DNS插件，能够以域名的方式访问Service，比静态IP地址更方便
* 命名空间是Kubernetes用来隔离对象的一种方式，实现了逻辑上的对象分组，Service的域名里就包含了命名空间限定
* Service的默认类型是"ClusterIP"，只能在集群内部访问，如果改成"NodePort"，就会在节点上开启一个随机端口号，让外界也能够访问内部的服务





### Ingress

https://github.com/nginxinc/kubernetes-ingress

`Ingress` 、`Ingress Controller`  、`Ingress Class`

* `--class` 指定Ingress从属的Ingress Class对象
* `--rule` 指定路由规则，基本形式是"URI=Service"，http request -> service -> Pod



```yaml
export out="--dry-run=client -o yaml"
kubectl create ing ngx-ing --rule="ngx.test/=ngx-svc:80" --class=ngx-ink $out
```

example-ingress.yml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ngx-ing
  
spec:

  ingressClassName: ngx-ink
  
  rules:
  - host: ngx.test
    http:
      paths:
      - path: /
        pathType: Exact
        backend:
          service:
            name: ngx-svc
            port:
              number: 80
```

example-ingress-class.yml

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: ngx-ink

spec:
  controller: nginx.org/ingress-controller
```





#### IngressController

https://github.com/nginxinc/kubernetes-ingress

```shell
kubectl apply -f common/ns-and-sa.yaml
kubectl apply -f rbac/rbac.yaml
kubectl apply -f common/nginx-config.yaml
kubectl apply -f common/default-server-secret.yaml

kubectl apply -f common/crds/k8s.nginx.org_virtualservers.yaml
kubectl apply -f common/crds/k8s.nginx.org_virtualserverroutes.yaml
kubectl apply -f common/crds/k8s.nginx.org_transportservers.yaml
kubectl apply -f common/crds/k8s.nginx.org_policies.yaml
```

ingress.yml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ngx-ing
  
spec:

  ingressClassName: ngx-ink
  
  rules:
  - host: ngx.test
    http:
      paths:
      - path: /
        pathType: Exact
        backend:
          service:
            name: ngx-svc
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: ngx-ink

spec:
  controller: nginx.org/ingress-controller

```



ingress-controller-kic.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx-kic-dep
  namespace: nginx-ingress

spec:
  replicas: 1
  selector:
    matchLabels:
      app: ngx-kic-dep

  template:
    metadata:
      labels:
        app: ngx-kic-dep
    
    spec:
      containers:
      - image: nginx/nginx-ingress:2.2-alpine
        args:
          - -ingress-class=ngx-ink
```





### PersistentVolume

`PersistentVolume`  `PersistentVolumeClaim`  `StorageClass`



* `storageClassName`

* `accessMode`
  * ReadWriteOnce 存储卷可读可写，但只能被一个节点的Pod挂载
  * ReadOnlyMany 存储卷只读不可写，可以被任意节点上的Pod多次挂载
  * ReadWriteMany 存储卷可读可写，可以被任意节点上的Pod多次挂载

* `capacity` 国际标准，Ki/Mi/Gi
* `hostPath` 指定存储卷的本地路径

host-path-pv.yml

```yaml
# PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: host-10m-pv

spec:
  storageClassName: host-test
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Mi
  hostPath:
    path: /tmp/host-10m-pv/
    
```

host-path-pvc.yml

```yaml
#PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: host-5m-pvc

spec:
  storageClassName: host-test
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Mi
```

host-path-pod.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: host-pvc-pod

spec:
  volumes:
  - name: host-pvc-vol
    persistentVolumeClaim:
      claimName: host-5m-pvc

  containers:
    - name: ngx-pvc-pod
      image: nginx:alpine
      ports:
      - containerPort: 80
      volumeMounts:
      - name: host-pvc-vol
        mountPath: /tmp
```

小结：

* PersistentVolume简称为PV，是Kubernetes对存储设备的抽象，由系统管理员维护，需要描述清楚存储设备的类型、访问模式、容量等信息
* PersistentVolumeClaim简称为PVC，代表Pod向系统申请存储资源，它声明对存储的要求，Kubernetes会查找最合适的PV然后绑定
* StorageClass抽象特定类型的存储系统，归类分组PV对象，用来简化PV/PVC的绑定过程
* HostPath是最简单的一种PV，数据存储在节点本地，速度快但不能跟随Pod迁移

### NFS

#### server

```shell
sudo apt -y install nfs-kernel-server

mkdir -p /tmp/nfs

/tmp/nfs 192.168.10.0/24(rw,sync,no_subtree_check,no_root_squash,insecure)

sudo exportfs -ra
sudo exportfs -v

sudo systemctl start  nfs-server
sudo systemctl enable nfs-server
sudo systemctl status nfs-server

showmount -e 127.0.0.1
```



#### client

```shell
sudo apt -y install nfs-common

showmount -e ${ip_address}

mkdir -p /tmp/test
sudo mount -t nfs 192.168.10.208:/tmp/nfs /tmp/test
touch /tmp/test/x.yml
```



nfs-static-pv.yml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-1g-pv

spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1Gi

  nfs:
    path: /tmp/nfs/1g-pv
    server: 192.168.10.208

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-1g-pv

spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1Gi

  nfs:
    path: /tmp/nfs/1g-pv
    server: 192.168.10.208

---
apiVersion: v1
kind: Pod
metadata:
  name: nfs-static-pod

spec:
  volumes:
  - name: nfs-pvc-vol
    persistentVolumeClaim:
      claimName: nfs-static-pvc

  containers:
    - name: nfs-pvc-test
      image: nginx:alpine
      ports:
      - containerPort: 80

      volumeMounts:
        - name: nfs-pvc-vol
          mountPath: /tmp
```



#### Provisioner

https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner



小结：

* 在Kubernetes集群里，网络存储系统更适合数据持久化，NFS是最容易使用的一种网络存储系统，需要实现安装好服务端和客户端

* 可编写PV定义NFS静态存储卷，要指定NFS服务器的IP地址和共享目录名

* 使用NFS动态存储卷必须要部署相应的Provisioner

* 动态存储卷不需要手工定义PV，而是要定义StorageClass，由关联的Provisioner自动创建PV完成绑定

  

### StatefulSet

管理有状态应用

* 启动顺序
* 依赖关系
* 网络标识
* 数据持久化

`kubectl api-resources`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: xxx-sts
```

example-redis-sts.yml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-sts

spec:
  serviceName: redis-svc
  replicas: 2
  selector:
    matchLabels:
      app: redis-sts

  template:
    metadata:
      labels:
        app: redis-sts
    spec:
      containers:
      - image: redis:5-alpine
        name: redis
        ports:
        - containerPort: 6379
```



example-redis-svc.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-svc

spec:
  selector:
    app: redis-sts

  ports:
  - port: 6379
    protocol: TCP
    targetPort: 6379
```

加入`volumeClaimTemplate`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-pv-sts

spec:
  serviceName: redis-pv-svc

  volumeClaimTemplates:
  - metadata:
      name: redis-100m-pvc
    spec:
      storageClassName: nfs-client
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 100Mi

  replicas: 2
  selector:
    matchLabels:
      app: redis-pv-sts

  template:
    metadata:
      labels:
        app: redis-pv-sts
    spec:
      containers:
      - image: redis:5-alpine
        name: redis
        ports:
        - containerPort: 6379

        volumeMounts:
        - name: redis-100m-pvc
          mountPath: /data
```

总结：

1. StatefulSet的YAML描述和Deployment几乎完全相同，只是多了一个关键字段`serviceName`
2. 要为StatefulSet里的Pod生成稳定的域名，需要定义Service对象，它的名字必须和StatefulSet里的`serviceName`一致
3. 访问StatefulSet应该使用每个Pod的单独域名，形式是`Pod名.服务名`,不应该使用Service的负载均衡功能
4. 在StatefulSet里可以用字段`volumeClaimTemplates`直接定义PVC，让Pod实现数据持久化存储



### Kubectl rollout

**应用的"版本更新"实际上更新的是整个Pod。**

ngx-v1.yml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ngx-conf

data:
  default.conf: |
    server {
      listen 80;
      location / {
        default_type text/plain;
        return 200
          'ver : $nginx_version\nsrv : $server_addr:$server_port\nhost: $hostname\n';
      }
    }
    
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: 
    kubernetes.io/change-cause: v1, ngx=1.21
  labels:
    app: ngx-dep
  name: ngx-dep

spec:
  minReadySeconds: 15
  replicas: 4
  selector:
    matchLabels:
      app: ngx-dep
  template:
    metadata:
      labels:
        app: ngx-dep
    spec:
      volumes:
      - name: ngx-conf-vol
        configMap:
          name: ngx-conf

      containers:
      - image: nginx:1.21-alpine
        name: nginx
        ports:
        - containerPort: 80

        volumeMounts:
        - mountPath: /etc/nginx/conf.d
          name: ngx-conf-vol
          
---
apiVersion: v1
kind: Service
metadata:
  name: ngx-svc
  
spec:
  selector:
    app: ngx-dep
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```



```shell
# 转发状态
kubectl port-forward svc/ngx-svc 8080:80 &
curl 127.1:8080
```





`kubectl rollout status`



`kubectl rollout pause`



`kubectl rollout resume`



`kubectl rollout history`

```shell

# 
kubectl rollout history deploy --revision=4

# --to-revision回退到任意一个历史版本
kubectl rollout undo deploy ngx-dep

```



小结：

1. 在Kubernetes里应用的版本不仅仅是容器镜像，而是整个Pod模板，为了便于处理使用了摘要算法，计算模板的Hash值作为版本号
2. Kubernetes更新应用采用的是滚动更新策略，减少旧版本Pod的同时增加新版本Pod，保证在更新过程中服务始终可用
3. 管理应用更新使用的命令是`kubectl rollout`，子命令有`status`、`history`、`undo`等。
4. Kubernetes会记录应用的 更新历史，可以使用`history --revision`查看每个版本的详细信息，也可以在每次更新时添加注解`kubernetes.io/change-cause`。



### Resouces&Probe

#### Resources

容器资源配额

example.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ngx-pod-resources

spec:
  containers:
  - image: nginx:alpine
    name: ngx

    resources:
      requests:
        cpu: 10m
        memory: 100Mi
      limits:
        cpu: 20m
        memory: 200Mi
```

`containers.resources`

* "**requests**", 容器要申请的资源
* "**limits**"，容器使用资源的上限



#### Probe

kubernetes为检查应用状态定义了三种探针

* **Startup**，启动探针，检查应用是否启动成功，适合那些有大量初始化工作要做，启动很慢的应用
* **Liveness**，存活探针，检查应用是否正常运行，是否存在死锁，死循环
* **Readiness**，就绪探针，用来检查应用是否可以接受流量，是否能够对外提供服务

如果Pod配置了探针，也有不同的处理策略

* 如果Startup探针失败，就会尝试反复重启，后面的两个探针也不会启动
* 如果Liveness探针失败，Kubernetes会认为容器发生了异常，也会重启容器
* 如果Readiness探针失败，Kubernetes会认为容器虽然在运行，但内部有错误，不能正常提供服务，就会把容器从Service对象的负载均衡集合中排除，不会给它分配流量

配置关键字段

* **periodSeconds**, 执行探测动作的时间间隔，默认是10秒探测一次
* **timeoutSeconds**，探测动作的超时时间，如果超时就认为探测失败，默认是1秒
* **successThreshold**，连续几次探测成功才认为是正常，对于StartupProbe和livenessProbe来说它只能是1
* **failureThreshold**，连续探测失败几次才认为是真正发生了异常，默认是3次

探测方式，Kubernetes支持3种：Shell、TCP Socket、HTTP GET

* **exec**，执行一个Linux命令，比如ps、cat等等。
* **tcpSocket**，使用TCP协议尝试连接容器的指定端口
* **httpGet**，连接端口并发送HTTP GET请求

要使用探针，必须在开发应用时预留出"检查口"，这样Kubernetes才能调用探针获取信息。

ngx-cm.yml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ngx-conf

data:
  default.conf: |
    server {
      listen 80;
      location = /ready {
        return 200 'I am ready';
      }
    }
```

ngx-pod-probe.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ngx-pod-probe

spec:
  volumes:
  - name: ngx-conf-vol
    configMap:
      name: ngx-conf

  containers:
  - image: nginx:alpine
    name: ngx
    ports:
    - containerPort: 80
    volumeMounts:
    - mountPath: /etc/nginx/conf.d
      name: ngx-conf-vol

    startupProbe:
      periodSeconds: 1
      exec:
        command: ["cat", "/var/run/nginx.pid"]

    livenessProbe:
      periodSeconds: 10
      tcpSocket:
        port: 80

    readinessProbe:
      periodSeconds: 5
      httpGet:
        path: /ready
        port: 80
```



小结：

1. 资源配额使用的是cgroup技术，可以限制容器使用的CPU和内存数量，让Pod合理利用系统资源，也能够让Kubernetes更容易调度Pod
2. Kubernetes定义了Startup、Liveness、Readiness三种健康探针，他们分别探测应用的启动、存活和就绪状态
3. 探测状态可以使用Shell、TCP Socket、HTTP Gget三种方式，还可以调整探测的频率和超时时间等参数



### Namespace



```shell
kubectl create ns test-ns 
kubectl get ns

# 命名空间的资源配额
export out="--dry-run=client -o yaml"
kubectl create quota dev-qt $out
```



test-ns-pod.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ngx
  namespace: test-ns

spec:
  containers:
  - image: nginx:alpine
    name: ngx
```



example.yml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev-ns

---

apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-qt
  namespace: dev-ns

spec:
  hard:
    requests.cpu: 10
    requests.memory: 10Gi
    limits.cpu: 10
    limits.memory: 20Gi

    requests.storage: 100Gi
    persistentvolumeclaims: 100

    pods: 100
    configmaps: 100
    secrets: 100
    services: 10

    count/jobs.batch: 1
    count/cronjobs.batch: 1
    count/deployments.apps: 1
```

在ResourceQuota里可以设置各类资源配额

* CPU和内存配额
* 存储容量配额
* 核心对象配额
* 其他API对象配额



#### 默认资源配额

`kubectl explain limits`



limit-range-lr.yml

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev-ns

spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 200m
      memory: 50Mi
    default:
      cpu: 500m
      memory: 100Mi
  - type: Pod
    max:
      cpu: 800m
      memory: 200Mi
```



小结：

1. namespace是一个逻辑概念，没有实体，它的目标是为资源和对象划分一个逻辑边界，避免冲突
2. ResourceQuota对象可以为namespace添加资源配额，限制全局的CPU、内存和API对象数量
3. LimitRange对象可以为容器或者Pod添加默认的资源配额，简化对象的创建工作

### Metrics Server & Prometheus



#### Metrics Server

https://github.com/kubernetes-sigs/metrics-server



```shell
# Metrics Server依赖
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml


```

1、修改YAML，需要在deploymeny对象里，加上一个额外的运行参数`--kubelet-insecure-tls`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
spec:
  ... ... 
  template:
    spec:
      containers:
      - args:
        - --kubelet-insecure-tls
        ... ... 
```

2、预下载Metrics Server的镜像

```shell
# reference
repo=registry.aliyuncs.com/google_containers

name=k8s.gcr.io/metrics-server/metrics-server:v0.6.1
src_name=metrics-server:v0.6.1

docker pull $repo/$src_name

docker tag $repo/$src_name $name
docker rmi $repo/$src_name
```



```shell
kubectl apply -f components.yaml

kubectl get pod -n kube-system

kubectl top node
kubectl top pod -n kube-system
```



#### HorizontalPodAutoscaler

**hpa**-水平自动伸缩对象, 适用于Deployment和StatefulSet

HorizontalPodAutoscaler的能力完全基于Metrics Server，它从Metrics Server获取当前应用的运行指标，主要是CPU使用率，再依据预定的策略增加或减少Pod的数量

example-ngx.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx-hpa-dep

spec:
  replicas: 1
  selector:
    matchLabels:
      app: ngx-hpa-dep

  template:
    metadata:
      labels:
        app: ngx-hpa-dep
    spec:
      containers:
      - image: nginx:alpine
        name: nginx
        ports:
        - containerPort: 80

        resources:
          requests:
            cpu: 50m
            memory: 10Mi
          limits:
            cpu: 100m
            memory: 20Mi
---

apiVersion: v1
kind: Service
metadata:
  name: ngx-hpa-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: ngx-hpa-dep
```

注意点：**`spec`一定要用`resources`字段写清楚资源配额**，否则HorizontalPodAutoscaler会无法获取Pod的指标，也就无法实现自动化扩缩容。

```shell
export out="--dry-run=client -o yaml"              # 定义Shell变量
kubectl autoscale deploy ngx-hpa-dep --min=2 --max=10 --cpu-percent=5 $out
```

hpa-example.yml

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: ngx-hpa

spec:
  maxReplicas: 10
  minReplicas: 2
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ngx-hpa-dep
  targetCPUUtilizationPercentage: 5
```

压力测试

```shell
kubectl run test -it --image=httpd:alpine -- sh

ab -c 10 -t 60 -n 1000000 'http://ngx-hpa-svc/'
```



#### Prometheus

Prometheus Server之外，还有三个重要的组件

* Push Gateway ：用来适配一些特殊的监控目标，把默认的Pull模式转变为Push模式

* Alert Manager：告警中心，预先设定规则，发现问题时就通过邮件等方式告警

* Grafana：图形化界面，可以定制大量直观的监控仪表盘

  

```shell
wget https://github.com/prometheus-operator/kube-prometheus/archive/refs/tags/v0.11.0.tar.gz
```



1、修改`prometheus-service.yml` 、`grafana-service.yml`

可以给他们添加`type: NodePort`

2、修改`kubeStateMetrics-deployment.yaml`、`prometheusAdapter-deployment.yaml`，因为它们里面有两个存放在 gcr.io 的镜像，必须解决下载镜像的问题。

国内无法下载，只能下载后再上传到DockerHub

```shell
image: k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.5.0
image: k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.9.1

image: chronolaw/kube-state-metrics:v2.5.0
image: chronolaw/prometheus-adapter:v0.9.1

docker pull pengyc2019/prometheus-adapter:v0.9.1
docker tag pengyc2019/prometheus-adapter:v0.9.1 k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.9.1
docker rmi pengyc2019/prometheus-adapter:v0.9.1
docker push k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.9.1

```





```shell
kubectl create -f manifests/setup
kubectl create -f manifests

kubectl get svc -n monitoring
kubectl get pod -n monitoring
```



小结：

1. Metris Server是一个Kubernetes插件，能够收集系统的核心资源指标，相关的命令`kubectl top`
2. Prometheus是云原生监控领域的"事实标准"，用PromQL语言来查询数据，配合Grafana可以展示直观的图形界面，方便监控
3. HorizontalPodAutoscaler实现了应用的自动水平伸缩功能，它从Metrics Server获取应用的运行指标，再实时调整Pod数量，可以很好地应对突发流量