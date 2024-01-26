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

#### Deployment

管理Pod，让应用不宕机

`kubectl api-resources`



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: xxx-dep
```



```yaml
export out="--dry-run=client -o yaml"
kubectl create deploy ngx-dep --image=nginx:alpine $out
```

example

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



#### DaemonSet

* 目标是在集群的每个节点上运行且仅运行一个Pod

* 跟Deployment类似，管理控制Pod，但管理调度策略不同。

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

##### `kubectl taint` 

`kubectl taint` + `节点名`  `污点名`  `污点` 

去掉污点+ `-`

```shell
kubectl taint node master node-role.kubernetes.io/master:NoSchedule-
```



tolerations

```yaml
tolerations:
- key: node-role.kubernetes.io/master
  effect: NoSchedule
  operator: Exists
```



#### Service

`TCP/IP`

`kubectl expose`  `--port` `--target-port`



```yaml
apiVersion: v1
kind: Service
metadata:
  name: xxx-svc
```

example-service.yml

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



#### Ingress

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



### PersistentVolume

`PersistentVolume`  `PersistentVolumeClaim`  `StorageClass`



#### NFS





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



`kubectl rollout status`



`kubectl rollout pause`



`kubectl rollout resume`



`kubectl rollout history`

