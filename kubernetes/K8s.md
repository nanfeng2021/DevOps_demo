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

maria-pod.yml

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

