# Kubernetes_Notes
阅读Kubernetes书籍所做笔记

书籍 《Kubernetes in Action》

- [1.Kubernetes介绍](#1kubernetes介绍)
- [2.开始使用Kubernetes和Docker](#2开始使用kubernetes和docker)
- [3.pod：运行于Kubernetes中的容器](#3pod运行于kubernetes中的容器)
- [4.副本机制和其他控制器：部署托管的pod](#4副本机制和其他控制器部署托管的pod)
- [5.服务：让客户端发现pod并与之通信](#5服务让客户端发现pod并与之通信)
- [6.卷：将磁盘挂载到容器](#6卷将磁盘挂载到容器)
- [7.ConfigMap和Secret：配置应用程序](#7configmap和secret配置应用程序)
- [8.从应用访问pod元数据及其他资源](#8从应用访问pod元数据及其他资源)
- [9.Deployment：声明式地升级应用](#9deployment声明式地升级应用)
- [10.StatefulSet：部署有状态的多副本应用](#10statefulset部署有状态的多副本应用)
- [11.了解Kubernetes机理](#11了解kubernetes机理)
- [12.Kubernetes API服务器的安全防护](#12kubernetes-api服务器的安全防护)
- [13.保障集群内节点和网络安全](#13保障集群内节点和网络安全)
- [14.计算资源管理](#14计算资源管理)
- [15.自动横向伸缩pod与集群节点](#15自动横向伸缩pod与集群节点)
- [16.高级调度](#16高级调度)
- [17.开发应用的最佳实践](#17开发应用的最佳实践)
- [18.Kubernetes应用扩展](#18kubernetes应用扩展)

 ### 1.Kubernetes介绍
**单体应用到微服务：**<br>
单体应用由许多个组件组成，这些组件紧密地耦合在一起，它们在同一个操作系统种运行。复杂的大型单体应用，水平扩展与垂直扩展皆有不便，于是，微服务便应运而生。微服务将大型单体应用拆分为小的可独立部署的微服务组件。<br>
让同一个团队参与应用的开发、部署、运维的整个生命周期，这种实践被称为DevOps。

**容器与虚拟机的区别：**<br>
虚拟机实现操作系统级别的隔离，而linux容器实现了进程级别的隔离。容器更加轻量。

**Linux容器怎样实现隔离：**<br>
1. linux命名空间。对于一个进程，可在一个命名空间中运行它，进程将只能看见同一个命名空间下的资源。
2. 限制进程的可用资源。cgroups是一个Linux内核功能，通过cgroups,限制一个进程或一组进程的资源使用。

Docker是首个使容器能在不同机器之间移植的平台。Docker本身不提供进程隔离，容器隔离是在linux内核上使用命名空间、cgroups完成的。  

Docker并非唯一的容器平台。Kubernetes并非一个专为Docker容器设计的容器编排系统，rkt也是一个运行容器的平台，亦被支持。为编排大规模容器，Kubernetes应运而生。

**Kubernetes架构：**
<img src="https://gallery.angelo.org.cn/images/2020/04/27/QQ20200427191738.png">

架构为分布式系统，分为master(控制)节点与node(工作)节点。<br><br>
API服务器：对外暴露的通信接口。<br>
Scheduler：调度器，将Pod分配至节点。<br>
Controller Manager：执行集群级别的功能，如复制组件、处理节点失败等。<br>
etcd：持久化存储集群配置。<br><br>
Kubelet:同API服务器进行通信。<br>
kube-proxy：负责组件间的负载均衡。<br>
容器运行时：运行的容器。

### 2.开始使用Kubernetes和Docker

使用Docker尝试运行简单的node.js应用：<br>
将以下代码保存为`app.js`。
```javascript
const http = require('http');
const os = require('os');
console.log("server is starting...")

var handler = function(request, response) {
    console.log("Receiving request from " + request.connection.remoteAddress);
    response.writeHead(200);
    response.end("You've hit" + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8000);
```
为使代码运行，还需nodejs环境。我们编写Dockerfile，供docker读取，以指定镜像的构建过程。
将以下内容保存为`Dockerfile`：
```Dockerfile
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```
使用`docker build -t <镜像名> .`构建自己的容器镜像。构建镜像时docker客户端会将当前目录所有文件上传至守护进程，因此当前目录不应存放与构建镜像无关的文件。<br>
镜像由多层组成，不同镜像可能共享同分层，这优化了镜像的存储与传输。

构建镜像完成后，便可从镜像生成容器。 <br>
`docker run --name <容器名>  -p 8080:8080 -d <镜像名>`

使用`docker login`后，可将镜像推送至自己的仓库：
`docker tag <镜像名> <dockerhub ID>/<镜像名> `<br>
`docker push <dockerhub ID>/<镜像名>`

Kubernetes用于管理大量容器。minikube是一个迷你版的kubernetes集群，可运行本地单节点集群。
minikube可在虚拟机上运行，可在宿主机上运行。<br>
`minikube start --vm-driver=<driver>` 指定虚拟机driver,一般为VirtualBox或KVM。设置为none则在宿主机运行。

**小插曲：** <br>
若在虚拟机中运行minukube，需给予足够的硬件资源，并开启虚拟化。启动失败多次后，我分配给ubuntu虚拟机2核+4G内存+30GB磁盘，才启动成功。并且，中间有许多次失败为网络原因，即使配置代理，也常出现"time out"等字样(原因不明)。后仿照一篇中文教程，指定国内镜像后方才运行成功：<br>
`minikube start --registry-mirror=https://registry.docker-cn.com`
<br><br>因此，当阅读英文资料配置环境时，由于网络等原因，有必要使用中文资料进行参考，在中文资料中可能寻得问题所在。
<br><br>
实际运行时，经发现，minikube开启后ubuntu虚拟机内存占用达到3GB，且宿主机中的风扇始终转不停。可见启动minikube资源占用较大。<br><br>
**注：dockerhub服务器位于国外，由于网络原因，拉取镜像时可能失败。可使用`minikube ssh`进入运行Kubernetes的虚拟机,然后创建/etc/docker/daemon.json(有则修改)，指定国内镜像源(具体见[设置国内镜像源](https://blog.angelo.org.cn/docker/))。然后重启docker服务。** 

安装Kubectl后，可经由APIServer进行与Kubernetes的交互。<br><br>
Kubernetes不直接处理单个容器，它使用多个容器共存的理念，这组容器便是pod。pod是一组紧密相关的容器，它们运行在同一个工作节点上。每个pod就如同一台独立的主机，具有自己的IP、主机名、进程等。<br><br>**容器、pod、工作节点三者的关系：**
<img src="https://gallery.angelo.org.cn/images/2020/04/28/QQ20200428103407.png">

**理解在Kubernetes中运行容器的过程：**
<img src="https://gallery.angelo.org.cn/images/2020/04/28/QQ20200428131822.png">

运行Kubectl时，它通过向APIServer发起REST HTTP请求，在集群中创建一个新的ReplicationController对象，ReplicationController创建一个新的pod。主节点将pod调度至一个工作节点。工作节点接收调度后，从dockerhub拉取镜像并创建容器。

每个pod虽具有IP，但此IP为集群内部的IP，因而无法从集群外访问pod。可创建LoadBalancer服务，创建外部的负载均衡，通过负载均衡的公共IP访问pod。创建服务对象:
`kubectl expose rc kubia --type=LoadBalancer --name kubia-http` <br>
列出服务：`kubectl get services`,查看外部IP。

`ReplicationController`的角色：<br>
`ReplicationController`用以复制pod，并控制pod的副本数恒定。<br>
Kubernetes将所有pod由一个服务对外暴露，ReplicationController控制pod的稳定。
<img src="https://gallery.angelo.org.cn/images/2020/04/28/QQ20200428134623.png"> <br><br>

便捷的扩容：<br>
`kubectl scale rc kubia --replicas=3`
查看`kubectl get rc`即可看见实例被扩展至3个。<br>

通过扩容操作，总结**Kubernetes基本原则之一**：<br>
不是告诉Kubernetes应该执行什么操作，而是声明性地改变系统的期望状态，并让Kubernetes检查当前的状态是否与期望的状态一致。


### 3.pod：运行于Kubernetes中的容器
pod是Kubernetes中最重要的核心概念。pod应包含紧密耦合的容器组。显然，一个pod中的所有容器位于同一个工作节点上。<br><br>
多个容器比单个容器包含多个进程好：<br>
容器被设计为每个容器只运行一个进程(除非进程本身产生子进程)。在实践中一个容器运行多个进程，增加了管理负担，是不合理的。<br>

有必要将pod视为独立的机器，每个机器只托管一个特定的应用。如前端服务与数据库服务便应分别放进不同pod中，方便各自扩容。将不同服务放进一个pod的做法不值得推荐。<br>
为充分利用计算资源，pod应分配至多个节点上。

容器不应包含多个进程，pod也不应包含多个并不需要运行在同一主机上的容器。
<img src="https://gallery.angelo.org.cn/images/2020/04/29/QQ20200429140741.png">

使用`kubectl get po <pod名> -o yaml`可查看定义pod的yaml文件。<br>
使用`kubectl get po <pod名 -o wide`可查看pod详细信息。<br>

**yaml文件的组成为:**<br>
- API版本、yaml描述的资源类型
- metadata(元数据)
- spec(内容)
- status(状态)

创建pod时，不需提供status部分。

一个基本的yaml文件：
```yaml
apiVersion: v1
kind: Pod
metadata:   #元数据
    name: kubia-manual
spec:   #内容
    containers:
    - image: luksa/kubia
      name: kubia
      ports:
      - containerPort: 8080
        protocol: TCP
```
**注:表示属性字段的冒号后有必要加空格，以表示此为map类型(可从语法高亮看出)，否则会被Kubernetes当作string类型，导致解析yaml时失败。如图：**

<img src="https://gallery.angelo.org.cn/images/2020/05/02/QQ20200502125100.png">

图中我使用了别名`k`代表`kubectl`。
<br><br>

使用`kubectl explain pods`获取pod描述文件的组成结构。更加的，使用如`kubectl explain pod.spec`可查看更深层次的结构。

使用`kubectl create`创建pod:<br>
`kubectl create -f <描述pod的yaml文件>`

**`kubectl create`可创建Kubernetes资源，
`kubectl apply`可修改Kubernetes资源。**

使用`kubectl logs <pod名>`即可查看该pod的日志，便无需ssh登录该pod所在节点并使用"docker logs <容器名>"了。若pod包含多个容器，则需使用`- c <容器名>`选项显式指定容器。

可通过`kubectl port-forward <pod名> <宿主机端口>:<pod端口>`进行端口转发，与pod通信。端口转发是一种测试特定pod的有效方法。

当pod数量较多时，使用标签组织pod。
创建pod时指定标签，将下面内容保存为kubia.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: kubia-manual-v2
    labels:     #添加了标签
      creatioon_method: manual
      env: prod
spec:
    containers:
    - image: luksa/kubia
      name: kubia
      ports:
      - containerPort: 8080
      protocol: TCP
```
`kubectl create -f kubia.yaml`创建出了带2个标签的pod。<br>

`kubectl get po --show-labels`查看标签。<br>
`kubectl get po -L creation_method,env`查看带有"creation_method"与"env"标签的pod。<br>
`kubectl label po kubia-manual creation_method=manual`为"kubia-manual""pod添加"creation_method"标签。<br>
`kubectl label kubia-manual-v2 env=debug --overwrite`更改"kubia-manual-v2"pod的"env"标签为"debug"。<br>
`kubectl get po -l creation_method=manual`选择"creation_method"标签为"manual"的pod。<br>
`kubectl get po -l env`选择所有包含标签"env"的pod。<br>
`kubectl get po -l '!env'`选择所有不包含"env"标签的pod。

标签可配合标签选择器使用。<br>
标签可附加到任何Kubernetes对象上。使用标签选择器可将pod调度至特定节点，这在一些情况下有所应用。<br><br>
`kubectl label node <节点名> gpu=true`向节点添加"gpu"标签。<br>
`kubectl get nodes -l gpu=true`选择含"gpu"标签且为"true"的节点。<br><br>
通过创建新的yaml文件，将pod调度至特定节点。将以下文件保存为"kubia-gpu.yaml"：
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: kubia-gpu
spec:
    nodeSelector:   
      gpu: "true"   #只将pod部署至含标签"gpu=true"的节点上
     containers:
     - image: luksa/kubia
       name: kubia
```
`kubectl create -f kubia-gpu.yaml`将pod部署至含标签"gpu=true"的节点上。<br>

为pod添加注解：<br>
`kubectl anontate pod kubia-manual mycompany . com/someannotation="foo bar"`为"kubia-manual"pod添加注解。<br>

为将复杂系统拆分为不同组，允许不同团队使用同一集群，可使用命名空间。<br>
列出所有命名空间:`kubectl get ns` <br>
列出指定命名空间的pod:`kubectl get po --namespace kube-system` 或 `kubectl get po -n kube-system` <br>
可通过yaml文件创建一个命名空间：
```yaml
apiVersion: v1
kind: Namespace
metadata:
    name: custom-namespace
```
使用yaml创建命名空间，体现了Kubernetes中所有内容都是一个API对象。

还可使用`kube create`创建命名空间:<br>
`kubectl create namespace custom-namespace`

在metadata字段中添加"namespace: custom-namespace"，创建资源时会指定"custom-namespace"命名空间。或`kubectl create -f <yaml文件> -n custom-namespace`也可指定。<br>

切换命名空间：
`alias kcd="kubectl config set-context $(kubectl config current-context) --namespace"`，然后可使用
`kcd some-namespace`。
<br>
按名称删除pod：
`kubectl delete po <pod名>` <br>
使用标签选择器删除pod：
如`kubectl delete po -l creation_method=manual`<br>
通过删除命名空间以删除pod:
`kubectl delete ns custom-namespace`<br>
删除当前命名空间中所有pod:
`kubectl delete po --all`<br><br>
而由于`ReplicationController`的存在，删除一个pod后，会立即出现一个新的pod顶替。为真正删除pod,还需删除该`ReplicationController`。<br>

删除命名空间中几乎所有资源(可删除ReplicationController、pod、所有service):<br>
`kubectl delete all --all`

### 4.副本机制和其他控制器：部署托管的pod
不应直接创建pod，因为如果它们被错误地删除，它们正在运行的节点异常，或者它们从节点中被逐出时，将不会被重新创建。<br><br>
实际中，希望pod能自动保持运行，于是便使用 `ReplicationController`或`Deployment`进行pod的创建与管理。<br>

Kubernetes为检查pod的健康，使用发送探针(liveness probe)的方法检查容器是否还在运行。有三种探测容器的机制：
1. 发送HTTP GET请求，检查返回码。
2. 发送TCP套接字请求，检查能否建立连接。
3. `exec`进入容器内执行命令，检查退出状态码。

可通过编辑yaml文件创建探针(liveness probe)。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy #该镜像包含坏掉的应用
    name: kubia
    livenessProbe:
      httpGet:  #一个httpGet存活探针
        path: / #http请求路径
        port: 8080 #请求的端口
```
该文件定义了一个httpGet探针，探针定期请求容器8080端口的web根路径，确定容器是否健康。Kubernetes会重启不健康的容器。<br>

通过`kubectl get pod <pod名>`可查看容器重启次数。<br>
通过`kubectl describe po <pod名>`可查看重启原因。<br>
定义pod时可设置探测的初始延迟，目的是等待pod就绪后再探测。
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 8080
initialDelaySeconds: 15 #在第一次探测前等待15秒
```
探针任务由承载Pod的节点上的kubectl执行，主节点并不执行。所以当节点崩溃时，探针任务便无法执行了。为确保pod在另一节点启动，需借助`ReplicationController`。<br>
`ReplicationController`始终保持所需数量的pod副本正在运行。

创建`ReplicationController`：
```yaml
apiVersion: v1
kind: ReplicationController #类型
metadata:
  name: kubia   #rc的名字
spec:
  replicas: 3   #pod实例数
  selector: #选择器决定了rc的操作对象
    app: kubia
  template: #创建新pod所用的模板
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080
```
将上述内容保存为`kubia-replica.yaml`。<br>
执行`kubectl create -f kubia-replica.yaml`创建一个`rc`。
<br>

获取有关ReplicationController的信息:<br>
`kubectl get rc`

获取ReplicationController详细信息：<br>`kubectl describe rc <rc名>`

编辑`rc`：`kubectl edit rc <rc名>` <br>
配置"kubectl edit"使用不同的编辑器：
`export KUBE_EDITOR=<编辑器路径>` <br>
编辑`rc`，改变模板中的容器镜像，删除已有pod后，rc会创建出新的pod，可用于升级pod。

使用`kubectl delete`删除`rc`时，其控制的pod也会被删除。而`rc`控制的pod并非`rc`自身组成的部分，因此可在删除`rc`时不删除控制的pod：<br>
`kubectl delete rc <rc名> --cascade==false`

**"ReplicaSet"是新一代的"ReplicationController"，它可以完全替代后者。** 学习`ReplicationControler`的目的是详细了解Kubernetes。

创建ReplicaSet:<br>
```yaml
apiVersion: apps/v1beta2    #并非v1版本
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:    #使用了更简单的matchLabels选择器
      app: kubia
  template:
    metadata:
      labels:
        app:kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
```
将上面内容保存为kubia-replicaset.yaml。
然后`kubectl create -f kubia-replicaset.yaml`。 <br>
凭标签选择器，该ReplicaSet可接管前面被删除的`rc`的pod。<br><br>
可使用`kubectl get`、`kubectl describe`查看ReplicaSet:<br>
`kubectl get rs` <br>
`kubectl describe rs`

删除ReplicaSet: <br>
`kubectl delete rs <rs名>` <br>
删除ReplicaSet会删除其管理的所有pod。

DaemonSet可将pod运行至指定的节点上。
编写DaemonSet:
```yaml
apiVersion: apps/v1beta2    #在apps的API组中
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssh-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
      spec:
        nodeSelector:   
          disk: ssd   #节点选择器会选择含disk=ssd标签的节点
        containers:
        - name: main
        image: luksa/ssh-monitor
```
保存为ssh-monitor-daemonset.yaml。<br>
`kubectl create -f ssh-monitor-daemonset.yaml`创建一个DaemonSet。

查看DaemonSet: `kubectl get ds` <br>
向节点打上标签: `kubectl label node <节点名> 标签`

<br>

`rc`、`rs`、`ds`均会使pod一直处于运行中。若需部署执行任务后便关闭的pod，可使用Job资源。
定义Job：
```yaml
apiVersion: batch/v1    #Job属于batch API组
kind: Job
metadata:
  name: batch-job
spec:
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy:OnFailure
      containiers:
      - name: main
        image: luksa/batch-job
```
pod的`restatPolicy`属性默认为"alwways"，Job不能使用该默认值，因为Job下的pod不会一直运行下去。
<br><br>
查看Job资源:<br>
`kubectl get jobs`

Job中可运行多个pod实例，并以并行或串行的方式运行，通过在Job配置中设置completion和parallelism属性来实现。
顺序运行Job pod,将completions设置为希望pod多少次:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5    #顺序运行5个pod
  template:
    ...
```
并行运行Job pod：
```yaml
...
spec:
  completions: 5    #必须确保5个pod成功完成
  parallelism: 2    #最多可并行运行2个pod
  template:
  ...
```
甚至可在Job运行时更改parallelism属性：<br>
`kubectl scale job <job名> --replicas=<数量>`

可在pod配置中设置`activeDeadlineSeconds`属性，设置超时时间，运行时间超过此时间，Job将标记为失败。

安排Job定期运行，可创建`CronJob`，与linux中的`crontab`类似：
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes
spec:
  schedule: "0,15,30,45 * * * *"    #每天在每小时0、15、30、45分钟运行
  jobTemplate:
    spec:
      template:
      ...
```
可设置`startingDeadlineSeconds`指定截止日期：<br>
```yaml
...
spec:
  schedule: "0,15,30,45 * * * *"
  startingDeadlineSeconds: 15   #最迟必须在预定时间后15秒执行，否则标记为失败
```

### 5.服务：让客户端发现pod并与之通信
pod通常需对来自集群内部的pod，及来自集群外部的客户端HTTP请求的响应。而pod是短暂的，其IP地址常会发生变化，为与一组功能相同的pod进行通信，可使用"服务"，将pod集群暴露为一个固定的IP地址与端口。<br><br>
Kubernetes服务是一种为一组功能相同的pod提供单一不变接入点的资源。
<img src="https://gallery.angelo.org.cn/images/2020/05/01/QQ20200501095801.png">

图中创建了一个前端服务，一个后端服务。

可通过`kubectl expose`创建服务，也可通过配置yaml文件创建服务。
```yaml
apiVersion: v1
kind: sevice
metadata:
  name: kubia
sepc:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia  #具有app=kubia标签的Pod均属于该服务
```
检测服务：
`kubectl get svc`

服务在集群内默认是可访问的，可登录至集群内部的一个pod中，以测试服务：<br>
`kubectl exec <pod名> -- curl -s http://ip:port`<br>
双横杠代表kubectl命令的结束，防止后面容器命令的横杠选项被解释为kubectl选项。

测试流程为：
<img src="https://gallery.angelo.org.cn/images/2020/05/01/QQ20200501113829.png">

可通过配置seriver中`spec.sessionAffinity`为`ClientIP`，使同一IP的客户端的请求被分发至同一pod上。

同一服务可暴露多个端口：
```yaml
...
spec:
  ports:
  - name: http
    port: 80    #pod的8080端口映射为80
    targetPort: 8080
  -name: https
    port: 443   #pod的8443端口映射为443
    targetPort: 8443
  selector:
    app: kubia  #标签选择器
```
pod中可命名端口：
```yaml
kind: Pod
spec:
  containers:
  - name: kubia
    ports:
    - name http     #8080端口命名为http
      containerPort: 8080
    - name https    #8443端口命名为https
      containerPort: 8443
```
<br>

集群中，客户端使用**服务发现**以发现服务。<br>
可使用环境变量或DNS发现服务.
- 环境变量：pod开始运行时，Kubernetes会初始化一系列环境变量指向现在存在的服务。使用`kubectl exec <pod名> env`查看Pod中的环境变量。或`kubectl exec -it <pod名> bash`可执行bash。
- DNS：kube-system命名空间下名为kube-dns的pod，运行DNS服务。pod是否使用内部DNS服务器，由spec中dnsPolicy属性决定。

<br>
服务并非直接与pod相连，有一种资源介于两者之间--EndPoint资源。有时会有"将连接重定向至外部IP与端口，而不是内部pod集群"的需求，可通过EndPoint实现。

可使用`kubectl describe svc <服务名>`查看服务中的EndPoint。
使用`kubectl get endpoints <服务名>`则直接查看。

当创建的服务中不含选择器(缺少选择器，将不会知道服务中包含哪些pod)时，EndPoint便不会随服务自动创建。<br>
创建一个无选择器的服务:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service    
sepc:
  ports:
  - port: 80
```
创建该服务的EndPoint资源：
```yaml
apiVersion: v1
kind: EndPoint
metadata:
  name: external-service    #名称需与服务的名称相同
subsets:
  - addresses:
    - ip: 11.11.11.11   #将连接重定向至这些地址
    - ip: 22.22.22.22
    ports:
    - port: 80      #目标端口
```

可在服务中为外部服务创建别名:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName    #设置type为ExternalName
  externalName: someapi.somecompany.com     #实际提供服务的域名
  ports:
  - port: 80
```

<br>
为将服务暴露给集群外部客户端，有几种方式：<br>

1. 将服务类型设置为NodePort。
2. 将服务类型设置为LoadBalance。
3. 创建一个Ingress资源。

<br>
创建NodePort服务，可使Kubernetes在所有节点上保留一个端口号(所有节点上使用相同端口号)，供外部、集群内部的访问。
<br><br>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-nodeport
spec:
  type: NodePort    #指定type为nodeport
  ports:
  - port: 80
    targetPort: 8080    #背后pod的目标端口号
    nodeport: 30000     #通过集群节点的30000端口可访问该服务
  selector:
    app: kubia
```
创建该服务后，可通过服务下每个节点的IP与30000端口访问该服务。

<br>
创建负载均衡器，可将外部请求负载均衡至各个节点。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-loadbalancer
spec:
  type: LoadBalancer    #定义type为负载均衡器
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```
然后使用`kubectl get svc <服务名>`，会出现外部IP。可通过该IP对服务进行访问。<br>
使用负载均衡器后，外部客户端访问均衡器的80端口，均衡器将请求路由至集群中的某个节点，之后该连接被转发至一个pod实例：
<img src="https://gallery.angelo.org.cn/images/2020/05/01/QQ20200501140119.png">

<br><br>

每个LoadBalancer服务均需要自己的负载均衡器与公网IP，而Ingress只需一个公网IP便能为许多服务提供访问。
<img src="https://gallery.angelo.org.cn/images/2020/05/01/QQ20200501143354.png">
<br>
创建Ingress资源:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: kubia.example.com     #将域名映射至服务
    http:
      paths:    
      - path: /     #所有请求发送至kubia-nodeport的80端口
        backend:
          serviceName: kubia-nodeport
          servicePort: 80 
```
获取Ingress的IP地址: `kubectl get ingresses`<br>
将不同服务映射至主机不同路径:<br>
```yaml
...
- host:kubia.example.com
  http:
    paths:
    - path: /kubia
      backend:
        serviceName: kubia  #对该域名/kubia的请求将转发至kubia服务
        servicePort: 80
    - path: /foo
      backend:
        serviceName: bar    #对该域名/foo的请求将转发至bar服务
        servicePort: 80
```
将不同服务映射至不同主机：
```yaml
spec：
  rules:
  - host: foo.example.com   #对foo.example.com的请求将会转发至foo服务
    http:
      paths:
      - path: /
        backend:
          serviceName: foo
          servicePort: 80
  - host: bar.example.com   #对bar.example.com的请求将会转发至bar服务
    http:
      paths:
      - path: /
        backend:
          serviceName: bar
          servicePort: 80
```
<br>
当使用HTTPS传输时，客户端与Ingress间流量加密，而Ingress与pod间不需加密，为HTTP传输。证书与私钥存储在称为Secret的Kubernetes资源中。

<br><br>

**就绪探针** <br>
不同于存活探针，就绪探针是探测pod是否已就绪。pod需要一定时间加载配置与数据，处理请求需要pod已就绪。同存活探针类型一样，就绪探针有三种类型。
<br><br>
若某个pod报告尚未准备就绪，则将该pod从服务中剔除。若该pod准备就绪，则重新添加至服务中。<br>

尝试修改`ReplicationController`中的pod模板，添加就绪探针，`kubectl edit rc <rc名>`：
```yaml
apiVesion: v1
kind: ReplicationController
...
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        readinessProbe:     #pod中每个容器都会有一个就绪探针
          exec:
            command:
            - ls
            - /var/ready
```
开发人员可通过某种验证手段检测命令是否执行成功，达到就绪探测的目的。

**应始终定义就绪探针，以保证服务的正常进行。**

<br>

到服务的每个连接都被转发至一个随机的Pod上，若要使得客户端可连接该服务的所有pod，则使用headless服务。<br>
**headless服务**：允许客户端能连接到支持服务的每个pod。

将spec项中`clusterIP`字段设置为"None"即可设置为headless服务。
```yaml
apiVersion: v1
kind: service
...
spec:
  clusterIP: None
  ...
```
该服务无集群IP。

<br>
有时希望未准备就绪的pod也能被匹配该pod标签的服务发现，可将下面出现的注解添加至服务中：

```yaml
kind: Service
metadata:
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
```
<br>

**服务是Kubernetes中的重要概念。** 若无法通过服务访问pod，排除服务故障便尤为重要。<br>暂且按原文记下故障排除的流程:
- 首先，确保从集群内连接到服务的集群IP ，而不是从外部。
- 不要通过ping 服务IP 来判断服务是否可访问（ 请记住，服务的集群IP是虚
拟IP，是无法pin g 通的） 。
- 如果己经定义了就绪探针，请确保它返回成功；否则该pod 不会成为服务的
一部分。
- 要确认某个容器是服务的一部分，请使用kubectl get endpoints来检查相应的端点对象。
- 如果尝试通过FQDN 或其中一部分来访问服务（ 例如，myservice.mynamespace.svc.cluster.local 或myservice.mynamespace），但并不起作用，请查看是否可以使用其集群IP 而不是FQDN 来访问服务。
- 检查是否连接到服务公开的端口，而不是目标端口。
- 尝试直接连接到pod IP 以确认pod 正在接收正确端口上的连接。
- 如果甚至无法通过pod 的IP 访问应用，请确保应用不是仅绑定到本地主机。

### 6.卷：将磁盘挂载到容器
**pod中的每个容器都有自己独立的文件系统，因为文件系统来自于容器镜像。**

为使pod中的容器能共同操作文件，使容器协同工作，进行数据的持久化存储，可定义存储卷(Volume)。将卷挂载至pod中的每个容器中，使得pod中的容器可操作相同的文件。

卷有许多类型。<br>
**注：持久卷、节点都不属于任何命名空间**

emptyDir用于存储临时数据。<br>
测试emptyDir卷的挂载:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: luksa/fortune
    name: html-generator  #第一个容器名为html-generator，运行luksa/fortune镜像
    volumeMounts:
    - name: html
      mountPath: /var/htdocs  #名为html的卷挂载于/var/htdocs
  - image: nginx:alpine
    name: web-server  #第二个容器名为web-server,运行nginx镜像
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true  #与上面相同的卷挂载于/usr/share/nginx/html，设置为只读
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:    #一个名为html的emptyDir卷，挂载于上面两个容器中
  - name: html
    emptyDir: {}

```
使用`kubectl create -f <yaml>`创建该pod后，可用`kubectl port-forward fortune 8080:80`将该pod的80端口映射至宿主机8080端口，然后使用`curl http://localhost:8080`访问。

emptyDir是在承载pod的节点的磁盘上创建的，其性能取决于节点磁盘性能。emptyDir还可指定在temfs(内存)上创建，以加快存取速度。将emptyDir的`medium`设置为Memory:
```yaml
...
volumes:
  - name: html
  emptyDir:
    medium: Memory
```
<br>

gitRepo卷是从emptyDir卷中填充git仓库内容的卷。
```yaml
...
volumns:
- name: html
  gitRepo:
    repository: <git仓库地址>
    revision: master    #主分支
    directory: .    #将repo克隆至卷的根目录
```

**emptyDir与gitRepo均不能持久化存储数据，它们会随pod的删除而删除。**

`hostPath`用于pod访问工作节点上的文件系统，它可指向节点文件系统上的特定目录与或文件。可用于持久化存储数据。<br>而在实践中`hostPath`并不适合多节点集群。因为新pod可能被调度至与原来pod不同的节点，如此新pod便无法读取原pod数据了。
<br>

实际需求为：当pod被调度至新节点时，pod仍能使用之前的数据。换言之，数据需要从集群可访问。<br>
实践中，数据存储至某种类型的网络存储(NAS)中。应根据基础设施，创建持久化存储卷。

在minikube中进行创建持久卷的实验：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  volumes:
  - name: mongodb-data
    hostPath:   #因为单节点，所以创建hostPath资源
      path: /tmp/mongodb    #访问minikube单节点的问文件系统
  containers:
  - image: mongo    #拉取mongodb镜像
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db   #挂载容器的/data/db目录
    ports:
    - containerPort: 27017
      protocol: TCP
```
创建完毕后，可验证是否为持久化存储。
`kubectl exec -it mongodb mongo`进入容器的mongdb命令行，插入一条数据后，退出并删除该pod：

<img src="https://gallery.angelo.org.cn/images/2020/05/02/QQ20200502161549.png">


然后创建一个pod，进入该pod的容器后，查看数据是否还存在。结果是存在的，说明数据被持久化存储了下来。(疑点：宿主机上并未发现/tmp/mongodb目录)
<br><br>

为将pod与底层基础架构解耦，使pod开发人员无需了解集群中真实网络存储的基础结构，`PersistentVolume`（持久卷，简称PV）、`PersistentVolumeClaim`(持久卷声明，简称PVC)应运而生。<br>

当集群用户需要在pod中使用持久化存储时，首先创建持久卷声明，指定最低容量要求与访问模式，然后用户将持久卷声明提交至Kubernetes API服务器，Kubernetes将找到可用的持久卷并将其绑定至持久卷声明：

<img src="https://gallery.angelo.org.cn/images/2020/05/02/QQ20200502150959.png">

集群管理员创建持久卷：
```yaml
apiVersion: v1
kind: PersistentVolume  #类型为持久卷
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 1Gi    #容量为1G
  accessModes:
  - ReadWriteOnce   #可被单个客户端挂载为读写模式
  - ReadOnlyMany    #可被多个客户端挂载为只读模式
  persistentVolumeReclaimPolicy: Retain     #持久卷声明释放后，持久卷将保留
  hostPath:
    path: /tmp/mongodb
```

使用`kubevtl get pv`可查看持久卷。
<img src="https://gallery.angelo.org.cn/images/2020/05/02/QQ20200502162756.png">

通过创建持久卷声明以获取持久卷：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim     #类型为pvc
metadata:
  name: mongodb-pvc
spec:
  resources:
    requests:
      storage: 1Gi  #申请1G存储空间
  accessModes:
  - ReadWriteOnce   #允许单个客户端访问
  storageClassName: ""  #有关动态配置，后面讲到
```
创建好声明后，Kubernetes会将满足条件的持久卷绑定之该持久卷声明。<br>
`kubectl get pvc`可查看持久卷声明的状态。

若要在pod中使用持久卷，需在pod的yaml配置文件中声明：
```yaml
...
spec:
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc    #通过名称引用持久卷声明
  ...
```

通过`pv`与`pvc`，研发人员无需再关心底层实际使用的存储技术。
<br><br>

为自动执行创建持久卷的任务，集群管理员可创建一个持久卷配置，并定义一个或多个StorageClass对象，让用户选择持久卷类型。用户可在持久卷声明中引用StorageClass，配置程序在配置持久卷时将采用。

首先，管理员创建一个或多个storageClass资源：
```yaml
apiVersion: storage.k8s.io/v1   #API类型
kind: StorageClass  #类别为sc
metadata:
  name: fast
provisioner: k8s.io/minikube-hostpath   #用于配置持久卷的插件
parameters:     #传递给parameters的参数
  type: pd-ssd
```

可通过`kubectl get sc`查看storageClass资源。<br>

创建StorageClass资源后，用户可在持久卷中声明中应用存储类：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  storageClassName: fast
  resources:
    requests:
      storage: 100Mi
  accessModes: 
  - ReadWriteOnce
```

可查看到所创建的pvc与动态配置的pv:
<img src="https://gallery.angelo.org.cn/images/2020/05/02/QQ20200502171045.png">

StorageClass的优点：使得pvc定义可跨不同集群移植。

持久卷动态配置的完整图示为：
<img src="https://gallery.angelo.org.cn/images/2020/05/02/QQ20200502171805.png">

### 7.ConfigMap和Secret：配置应用程序
为像容器中传递配置参数，以及保护一些敏感配置，可使用`ConfigMap`与`Secret`。

可用于配置应用程序的方法：
- 向容器传递命令行参数
- 为每个容器设置自定义环境变量
- 通过特殊类型的卷将配置文件挂在至容器中

向容器传递命令行参数：<br>
`ENTRYPOINT`定义容器启动时执行的程序。
`CMD`指定床传递给`ENTRYPOINT`的参数。<br>

在Kubernetes中定义容器时，镜像的ENTRYPOINT与CMD可被覆盖。在容器定义中设置属性`command`与`args`即可。
```yaml
kind: Pod
spec:
  containers:
  - image: some/image
    command: ["/bin/command"]   #指定命令
    args: ["arg1", "arg2", "arg3"]  #指定参数
```
command字段与args字段在创建后无法被修改。

**创建一个传输命令行参数的应用:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune2s
spec:
  containers:
  - image: luksa/fortune:args
    args: ["2"]     #传递一个参数
    name: html-generator  #第一个容器名为html-generator，运行luksa/fortune镜像
    volumeMounts:
    - name: html
      mountPath: /var/htdocs  #名为html的卷挂载于/var/htdocs
  - image: nginx:alpine
    name: web-server  #第二个容器名为web-server,运行nginx镜像
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true  #与上面相同的卷挂载于/usr/share/nginx/html，设置为只读
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:    #一个名为html的emptyDir卷，挂载于上面两个容器中
```
少量参数时可使用上述数组表示，多参数情况下可采用如下标记：
```yaml
args
- foo
- bar
- "12"  #字符串值无需用引号标记，数值需要。
```

**为容器设置环境变量：**
可为pod中每个容器定义环境变量。<br>
```yaml
kind: Pod
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
      value: "30"
    name: html-generator
...
```
也可使用`$(VAR)`引用其他环境变量。

<br>
为将配置与pod定义中解耦，可使用ConfigMap资源。应用无需直接读取ConfigMap，甚至根本不需知道其是否存在。ConfigMap的内容通过环境变量或卷文件的形式传递给容器。
<br>
<br>

使用`kubectl create configmap`创建ConfigMap：<br>
`kubectl create configmap fortune-config --from-literal=sleep-interval=25`
创建了一个名为fortune-config的ConfigMap，包含配置条目`sleep-interval=25`。

还可从文件/文件夹内容创建ConfigMap：
`kubectl create configmap my-config --from-file=config-file.conf`。


将ConfigMap中的条目传递给容器：
```yaml
apiVersion:
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL    #设置环境变量：INTERVAL
      valueFrom:    #用configmap初始化
        configMapKeyRef:
          name: fortune-config  #引用的configmap名称
          key: sleep-interval   #环境变量被设置为configmap下对应键的值
...
```
如果引用不存在的configmap，容器将会启动失败。

一次性传递configmap所有条目给容器：
```yaml
spec:
  containers:
  - image: some-image
    envFrom:    #使用envFrom字段而非env字段
    - prefix: CONFIG_   #可选，所有环境变量均包含前缀CONFIG_
      configMapRef:     #引用名为my-config-map的configMap
        name: my-config-map
```

<br>
当传递较大的配置文件给容器时，可使用configMap卷将条目暴露为文件，运行于容器中的进程可通过读取文件内容获得对应的值。
<br>


**注：将卷挂载至容器某文件夹，该文件夹中原本的文件会被隐藏。**

还可为configMap中的文件设置权限：
```yaml
volumes:
- name: config
  configMap:
    name: fortune-config
    defaultMode: "6600"     #设置所有文件的权限为-rw-rw----
```

使用环境变量或命令行参数无法在容器运行时更新配置，而configMap可以。修改configMap时，使用该configMap的容器中的配置文件也会更新。



为了存储与分发敏感信息(证书、私钥等)，Kubernetes提供了`Secret`资源。Kubernetes仅将Secret分发至需要访问Secret的pod所在节点以保障安全性。同时，Secret只保存于内存中，永不写入物理存储。<br>
可使用`kubectl get secrets`查看Secret，使用`kubectl describe secrets`查看详细信息。<br>

创建一个一般类型的Secret资源：<br>
`kubectl create secret generic <secret名> --from-file=<包含的文件>`

或者从yaml文件中创建：
```yaml
apiVersion: v1
kind: Secret
stringData:     #stringData可用来设置非加密内容
  foo: plain text
data:   #默认数据均经过base64编码
  https.cert: LSOtLSlCRUdJTiBDRVJUSUZJQOFURSOtLSOtCklJSURCekNDQ...
  https.key: LSOtL SlCRUdJT 工B SUOEgUFJJVkFURSBLRVkt LSOtLQpNSUlFcE...
``` 
尝试将Secret中的证书挂载至nginx中：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-https
spec:
  containers:
  - image: luksa/fortune:env
    name: html-generator
    env:
    - name: INTERVAL
      valueFrom: 
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: certs   #配置nginx从/etc/nginx/certs读取证书与密钥，所以需要将secret卷挂载于此
      mountPath: /etc/nginx/certs/
      readOnly: true
    ports:
    - containerPort: 80
    - containerPort: 443
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      items:
      - key: my-nginx-config.conf
        path: https.conf
  - name: certs
    secret:     #引用含证书与密钥的secret卷
      secretName: fortune-https
```
<br>
Secret与configMap组合使用的图示：
<img src="https://gallery.angelo.org.cn/images/2020/05/03/QQ20200503191129.png">
<br>

<br><br>
当从私有仓库拉取镜像时，也需要使用Secret配置密钥。如创建docker私有仓库的Secret：<br>
`kubectl create secret docker-registry <secret名> --docker-username=<用户名> --docker-password=<密码> --docker-email=<邮箱>`

在创建pod时使用该Secret：
```yaml
apiVersion: v1
kind: Pod
metadata:
  private-pod
spec:
  imagePullSecrets:     #该字段引用了前面创建的docker私有仓库secret
  - name: <secret名>
  containers:
  - image: <用户名>/private:tag
    name: main
```

### 8.从应用访问pod元数据及其他资源
对无法预知的数据，如pod的IP、主机名等，使用configMap或Secret无法传递配置数据。<br>
这时，可使用`Downward API`传递这些元数据。`Downward API`提供了简单的方式，将pod和容器中的元数据传递至它们内部运行的进程。`Downward API`允许通过环境变量或文件来传递pod和容器的元数据。

通过环境变量将pod的容器的元数据传递至容器中：
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: downward
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep","9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    env:
    - name: POD_NAME    #引用pod元数据中的名称字段，而非具体的值
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:  
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
    - name: CONTAINER_CPU_REQUEST_MILLICORES
      valueFrom:
        resourceFieldRef:   #容器请求的CPU与内存容量的引用字段为resourseFieldRef，而不是fieldRef
          resource: requests.cpu
          divisor: 1m   #对于资源相关的字段，定义一个基数单位，从而生成每一部分的值
    - name: CONTAINER_MEMORY_LIMIT_KIBIBYTES
      valueFrom:
        resourceFieldRef:
          resource: limits.memory
          divisor: 1Ki
```
进入容器中查看这些环境变量：
<img src="https://gallery.angelo.org.cn/images/2020/05/04/QQ20200504093022.png">

或通过创建`Downward API`卷挂载至容器中，以传递元数据:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
  labels:   #通过downward卷暴露这些标签与注解
    foo: bar
  annotations:
    key1: value1
    key2: |
      multi
      line
      value
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    volumeMounts:   
    - name: downward
      mountPath: /etc/downward  #将该downward卷挂载至容器的/etc/downward
  volumes:
  - name: downward  #将卷的名字设定为downward
    downwardAPI:
      items:
      - path: "podName"
        fieldRef:
          fieldPath: metadata.name  #pod的name将被写入/etc/downward/podName文件中，下面类似
      - path: "podNamespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "containerCpuRequestMilliCores"
        resourceFieldRef:
          containerName: main
          resource: requests.cpu
          divisor: 1m
      - path: "containerMemoryLimitBytes"
        resourceFieldRef:
          containerName: main
          resource: limits.memory
          divisor: 1
```
进入容器中查看已挂载的内容：
<img src="https://gallery.angelo.org.cn/images/2020/05/04/QQ20200504094806.png">

<br><br>
通过`downward API`获取的元数据是相当有限的，若要获取更多的元数据，需使用直接访问Kubernetes API服务器的方式。<br>
使用`kubectl cluster-info`可查看服务器的URL。<br>

API服务器具有REST(Representation State Transfer)风格。<br>

访问Kubernetes API服务器的步骤：
1. 确定API服务器的位置
2. 验证API服务器的证书，防止冒名者
3. 通过API服务器的身份验证

此时`kube-proxy`的作用便体现出来。可通过它与API服务器进行交互，无需手动处理上面的步骤了。

在本机可使用`kubectl proxy`打开代理通道，然后访问打开的端口。<br>

在pod中，每个pod都默认配置了有关Kubernetes的环境变量。通过查询环境变量可获得API服务器的相关信息。<br>
或直接使用域名形式访问API服务器(因为每个服务都有一个DNS入口)。<br>

尝试创建一个pod以访问API服务器：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl
spec:
  containers:
  - name: main
    image: tutum/curl   #使用包含curl的镜像
    command: ["sleep", "9999999"]   #运行一个长时间延迟的sleep命令保证容器一直处于运行状态
```
使用`kubectl exec -it curl bash`进入容器。<br>

直接使用`curl https://kubernetes`访问API服务器会提示无法验证证书。此时可使用`Secret`挂载至容器中的CA公钥来验证API服务器的证书。<br>
(名为default-token-xyz的Secret默认挂载于容器中的/var/run/secrets/kubernetes.io/serviceaccount/)
<img src="https://gallery.angelo.org.cn/images/2020/05/04/QQ20200504110850.png">

已成功验证证书(通过设置`CURL_CA_BUNDLE`变量为CA公钥路径，便无需再指定--cacert选项了)。接下来便需获取API服务器的授权，在HTTP头指定token：
<img src="https://gallery.angelo.org.cn/images/2020/05/04/QQ20200504110850bbb142d4b8818894.png">
**由于基于角色的访问控制存在，pod中含有的默认ServiceAccount凭证不具有列出集群状态的权限，因此返回403**

<br><br>
为简化与API服务器的交互，可使用ambassador容器。<br>ambassador译为使节，可将容器通过ambassador访问API服务器的过程形象地比喻为：容器通过API服务器驻扎于pod中的使节来访问API服务器。

<br>
容器中的应用通过HTTP协议连接ambassador，由ambassaddor通过HTTPS连接API服务器。<br><br>

创建带有附加ambassador容器的pod：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-with-ambassador
spec:
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "999999"]
  - name: ambassador    #ambassador容器，运行kubectl-proxy镜像
    image: luksa/kubectl-proxy:1.6.2
```
该pod包含两个容器。使用`kubectl exec -it curl-with-ambassador -c main bash`进入main容器。默认情况下，ambassador容器绑定8001端口，由于pod中两个容器共享包括localhost地址在内的相同的网络接口，所以可直接在main容器中访问本地8001端口：
<img src="https://gallery.angelo.org.cn/images/2020/05/04/QQ2020050411085042a293d8abd1e7de.png">
**疑点：已连接至API服务器，却仍显示无授权。**

<br>
ambassador可屏蔽连接外部服务的复杂性，简化在主容器中运行的应用。但是需要运行额外的进程，并且消耗资源：
<img src="https://gallery.angelo.org.cn/images/2020/05/04/QQ20200504121137.png">


<br><br>
当执行复杂API请求时，可使用Kubernetes API客户端。有多种语言编写的客户端。

### 9.Deployment：声明式地升级应用
运行在pod中的应用程序有更新的需求。目前为止有两种升级pod的方法：
1. 更新`ReplicationController`的pod模板，然后删除其控制的旧版本pod，`ReplicationController`会生成新版本pod。(缺点：将有一段时间停机)
2. 部署好新版本pod，待新版本pod正常运行后，再删除旧版本pod。(缺点：同时运行旧版本与新版本，消耗双倍硬件资源)

<br>
还可以执行滚动升级操作，通过逐渐减少旧版本pod增加新版本pod的方式更新：
<img src="https://gallery.angelo.org.cn/images/2020/05/05/QQ20200505091609.png">

<br><br><br>

尝试使用`ReplicationController`进行滚动更新：<br>
创建一个v1版本的pod与暴露其的service：
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia-v1
spec:
  replicas: 3   #rc控制3个pod
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
---                             #YAMLwen文件可包含多个资源定义，以三个横杠分开
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  type: LoadBalancer
  selector:     #service指向所有app=kubia标签的pod
    app: kubia
  ports:
  - port: 80
    targetPort: 8080
```
由于使用minikube，无法获取外部IP并直接访问。尝试进入其中一个pod访问集群服务：
<img src="https://gallery.angelo.org.cn/images/2020/05/05/QQ20200505095121.png">
目前版本为v1。
<br>

使用`kubectl rolling-update kubia-v1 kubia-v2 --image=luksa/kubia:v2`进行将名为kubia-v1的rc升级为名为kubia-v2的rc，镜像被更换为v2版本:<br>
**Warning! `kubectl rolling-update`在当前版本中已被弃用，无法使用该指令！**

<br>
下面使用更高阶的资源--Deployment进行应用程序的部署，并以声明的方式升级应用。可直接定义单个Deployment资源期望的状态，让Kubernetes去处理。<br>
<br>

当创建一个Deployment时，ReplicaSet也会随之创建。实际的pod是由Deployment的ReplicaSet管理的，不是由Deployment直接创建与管理。
<br>

创建一个Deployment资源：
```yaml
apiVersion: apps/v1beta1    #Deployment属于apps API组
kind: Deployment    #种类为Deployment
metadata:
  name: kubia   #Deployment可同时管理多个版本，名称中不再需要版本号
spec:
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
```
保存为kubia-deployment-v1.yaml，并`kubectl creata -f kubia-deployment-v1.yaml --record`。(--record选项可让Deployment显示历史版本号)

使用`kubectl set image`更改镜像：<br>
``kubectl set image deployment kubia nodejs=luksa/kubia:v2``

执行完此命令，`kubia` Deployment的pod模板镜像会被更改为luksa/kubia:v2：
<img src="https://gallery.angelo.org.cn/images/2020/05/05/QQ20200505104124.png">
<img src="https://gallery.angelo.org.cn/images/2020/05/05/QQ20200505104333.png">

修改资源的若干不同方式：<br>
|  方法   | 作用  |
|  ----  | ----  |
| `kube edit`  | 使用默认编辑器打开资源配置，如：`kubectl edit deployment kubia`|
| `kubectl patch`  | 修改单个资源属性 |
| `kubectl apply`  | 通过一个完整的YAML或json文件，应用其中新的值来修改对象，如果指定的对象不存在则会被创建  |
|  `kubectl replace` | 将原有对象替换为YAML/JSON文件中定义的新对象，运行前要求对象必须存在 |
| `kubectl set iamge`  |  修改资源内的镜像 |


回滚操作：<br>
``kubectl rollout undo deployment kubia``可将Deployment回退至上个版本。

查看滚动升级历史：<br>
``kubectl rollout history deployment kubia``可查看历史版本

回滚至一个Deployment版本：<br>
``kubectl rollout undo deployment kubia --to-revision=1``回到了版本1

暂停滚动升级：<br>
``kubectl rollout pause deployment kubia``
可验证新版本是否正常工作后，再继续升级

恢复滚动升级：<br>
``kubectl rollout resume deployment kubia``

可通过设置maxSurge与maxUnavailable属性以控制滚动更新速率:<br>
```yaml
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1   #决定了滚动升级过程中最多允许超出的pod实例的数量
      maxUnavailable: 0     #决定了滚动升级过程中能够允许多少pod实例处于不可用状态
    type: RollingUpdate
```

<br><br>
阻止出错版本的滚动升级：<br>
`minReadySeconds`属性指定新创建的pod至少需成功运行多久后，才能将其视为可用。配合就绪探针，可检测新pod是否成功运行，若出现失败，则新版本的滚动升级将被阻止。<br>

尝试部署一个出错的应用并设置探针：
```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  minReadySeconds: 10   #设置最少正常运行10s
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0     #确保升级过程中pod被挨个替换
    type: RollingUpdate
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v3
        name: nodejs
        readinessProbe:     #定义一个就绪探针每隔1s执行一次
          periodSeconds: 1
          httpGet:  #就绪探针发送HTTP GET请求至容器
            path: /
            port: 8080
```
使用`kubectl apply -f`升级Deployment后，由于就绪探针返回失败，因此滚动更新将被阻塞。<br>
默认情况下，若10分钟内不能完成滚动升级，则视为失败。失败后需使用``kubectl rollout undo deployment kubia``取消滚动升级。

与处理多个`ReplicationController`对象相比，管理单个Deployment更为容易。

### 10.StatefulSet：部署有状态的多副本应用
由一个ReplicaSet资源创建的pod共用相同的模板，它们只能共享相同的持久卷声明与持久卷：
<img src="https://gallery.angelo.org.cn/images/2020/05/05/QQ20200505154529.png">

若要使得每个pod实例都具有单独存储，StatefulSet应运而生。<br>

ReplicaSet中每个pod并非独一无二，它们都是无状态的，任何时候都可被一个全新的pod替换。而StatefulSet中每个pod实例都是不可替代的个体，都拥有稳定的名字与状态。

<br>
为给pod提供稳定的网络标识，一个StatefulSet要求创建一个用来记录每个pod网络标记的headless service。通过这个service，每个pod将有独立DNS记录。
<br>

为使用StateFul部署应用，需创建以下对象：
- 存储数据的持久卷
- Stateful必需的一个控制service
- Stateful本身

对于每一个pod实例，Stateful都会创建一个绑定到一个持久卷上的持久卷声明。<br>
下面创建一个含3个pod副本的StatefulSet资源。

首先创建3个持久卷：
```yaml
kind: List  #可使用list，一次创建多个资源
apiVersion: v1
items:
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-a
  spec:
    capacity:
      storage: 1Mi  #每个持久军大小为1M
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle  #当卷被声明释放后，空间会被再回收利用
    hostPath:
      path: /tmp/pv-a
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-b
  spec:
    capacity:
      storage: 1Mi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    hostPath:
      path: /tmp/pv-b
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-c
  spec:
    capacity:
      storage: 1Mi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    hostPath:
      path: /tmp/pv-c
```
创建service：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  clusterIP: None   #Statefulset的控制service必须是headless模式
  selector:
    app: kubia  #所有标签为app=kubia的pod都属于这个service
  ports:
  - name: http
    port: 80
```
创建Statefulset:
```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: kubia
spec:
  serviceName: kubia
  replicas: 2
  template:
    metadata:
      labels:
        app: kubia  #Statefulset创建的pod都具有app=kubia标签
    spec:
      containers:
      - name: kubia
        image: luksa/kubia-pet  #该容器中的服务会保存HTTP POST来的数据，当以HTTP GET请求时会显示保存的数据
        ports:
        - name: http
          containerPort: 8080
        volumeMounts:
        - name: data    #pod中的容器将把pvc数据卷嵌入指定目录
          mountPath: /var/data
  volumeClaimTemplates:     #创建持久卷声明
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 1Mi
      accessModes:
      - ReadWriteOnce
```

<br>

可进入pod访问服务，或创建一个普通service以访问，或通过API服务器与Pod通信。

通过API服务器访问pod：<br>
为避免麻烦的验证与授权，先使用`kubectl proxy`带打开代理，再访问8001端口对应路径：
<img src="https://gallery.angelo.org.cn/images/2020/05/05/QQ202005051545294216bf92fa2890af.png">
流量路径为：
<img src="https://gallery.angelo.org.cn/images/2020/05/05/QQ20200505180759.png">

工作状态正常。现使用`kubectl delete po kubia-0`删除一个有状态pod，检查重新调度的pod是否关联了相同的存储：
<img src="https://gallery.angelo.org.cn/images/2020/05/05/QQ20200505192717.png">
可确认Statefulset会使用一个完全一致的pod来替换被删除的pod。

可通过编辑Statefulset配置文件更新Pod模板，让它使用新的镜像。Statefulset更像ReplicaSet,而不是Deployment，所以在模板被修改后，需要手动删除旧的pod副本，然后Statefulset会依据新的模板重新调度它们。<br>

自Kubernetes 1.7版本始，StatefulSet支持与Deployment和DaemonSet一样的滚动升级。


当节点与网络断开时，pod的状态将变为Unkonwn。以正常方式`kubectl delete po`无法删除该pod，因为删除pod需要该节点通知API服务器pod已终止，而此时节点又与网段断开，无法完成。<br>
需要使用`kubectl delete po <pod名> --force --grace-period 0`强制删除pod。<br>
再次列举pod，可看到新的pod被创建出来。

### 11.了解Kubernetes机理
再次回顾下Kubernetes中的各个组件。<br>
控制节点上的组件：
- etcd分布式持久化存储
- API服务器
- 调度器
- 控制管理器

工作节点上的组件：
- kubelet
- kube proxy
- 运行的容器

附加组件：
- Kubernetes DNS服务器
- dashboard
- Ingress控制器
- Heapster(容器集群监控)
- 容器网络接口插件

**Kuberbetes如何使用etcd：**<br>
<p>etcd是一个响应快、分布式、一致的key-value存储。Kubernetes中的集群状态、元数据以持久化方式存储至etcd。<br></p>
<p>因为etcd是分布式的，故可以运行多个etcd实例来获取高可用性和更好的性能。</p>
<p>API服务器可直接与etcd通信，而其他组件通过API服务器间接地读取、写入数据到etcd。</p>
<p>连接到etcd集群不同节点的客户端，得到的要么是当前的实际状态，要么是之前的状态。etcd使用RAFT一致性算法保证etcd集群一致性。一致性算法要求集群过半节点参与才能进行到下一状态，所以etcd实例数量往往是奇数。</p>

**API服务器做了什么：**
<p>API服务器实现了乐观锁机制，提供了一致的方式将对象存储至etcd。</p>
<p>API服务器通过认证插件认证客户端，通过授权插件授权客户端，通过准入控制插件验证修改资源的请求：</p>
<img src="https://gallery.angelo.org.cn/images/2020/05/06/QQ20200506184420.png">
  
<p>通知客户端资源变更：</p>
<img src="https://gallery.angelo.org.cn/images/2020/05/06/QQ20200506185132.png">

<br>
<br>

**调度器：**<br>
调度器依据调度算法，将新的Pod调度至节点上。可在集群中设置多个调度器，但一个pod在某个时刻只能被一个调度器调度。

**控制器：**<br>
前面已涉及许多控制器如Deployment、Statefulset等，它们的作用为使集群朝着期望的状态收敛。
控制器包括：
- Replication 管理器（ ReplicationController 资源的管理器）
- ReplicaSet 、DaemonS et 以及Job 控制器
- Deployment 控制器
- State如！ Set 控制器
- Node 控制器
- Service 控制器
- Endpoints 控制器
- Namespace 控制器
- PersistentVolume 控制器
- 其他

**kubelet：**<br>
在API服务器中创建Node资源并注册节点，然后启动p。<br>
也是运行容器存活的组件，当探针报错时它会重启容器。

**kube-proxy：**<br>
通过建立iptables规则使得服务在自己的运行节点上可寻址。将目的IP为服务IP/端口的报文解析，修改目的地址，这样数据包就会被重定向至支持服务的一个pod。

<br>
为使不同节点上的pod进行通信，Kubernetes需要配置一个跨pod网络，可以使flannel、Weave Net等：
<img src="https://gallery.angelo.org.cn/images/2020/05/06/QQ20200506185132d61dbdc71b0d0afc.png">

<br><br>

一个高可用集群示例：
<img src="https://gallery.angelo.org.cn/images/2020/05/06/QQ20200506220338.png">

### 12.Kubernetes API服务器的安全防护
API服务器上的认证插件会返回已经认证过用户的用户名和组。<br>

ServiceAccount是一种资源，每个pod都与一个ServiceAccount相关联，它代表了运行在pod中应用程序的身份证明。<br>
可使用`kubectl get sa`查看ServiceAccount资源。

为保障集群的安全性，应为不同权限的pod创建不同的ServiceAccount。<br>

创建一个名为foo的ServiceAccount：<br>
`kubectl create serviceaccount foo`

可使用`kubectl describe sa foo`查看详细信息

sc中包含了可被pod挂载的密钥，可通过对sc进行配置，包含注解`kubernetes.io/enforce-mountable-secrets="true"`，让pod只允许挂载其中列出的可挂载密钥。

为将sc中的密钥赋值给pod，可在pod定义中的`spec.serviceAccountName`设置sc的名称。下面创建一个非默认sc的pod：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-custom-sa
spec:
  serviceAccountName: foo
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "999999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```
使用`kubectl exec -it curl-custom-sa -c main curl localhost:8001/api/v1/pods`查看集群中pod的状态。

<br>
Kubernetes API服务器默认使用RABC(Role Based Access Control)插件进行访问控制。

RBAC规则是通过四种资源来进行配置的，它们可以分为两个组：
- Role与ClusterRole
- RoleBinding与ClusterRoleBinding

Role资源定义了哪些操作可以在哪些资源上进行。<br>
定义一个Role资源：
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: foo    #所在命名空间
  name: service-reader
rules:
- apiGroups: [""]
  verbs: ["get", "list"]    #该role可获取独立的serivce并且列出所有允许的服务
  resources: ["service"]    #这条规则与服务有关
```

还需要将角色绑定至所在命名空间中的ServiceAccount上：
`kubectl create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo`<br>
除此之外，roleBinding还可引用来自其他命名空间中的ServiceAccount。

### 13.保障集群内节点和网络安全

**在pod中使用宿主机的命名空间：** <br>
部分Pod需要在宿主节点的默认命名空间中运行，以允许它们看到操作节点级别的资源和设备，以下是一个使用宿主机网络接口的例子：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-host-network
spec:
  hostNetwork: true     #添加hostNetwork=true字段，使用宿主节点的网络命名空间
  containers:
  - name: main
    image: alpine
    command: ["sleep", "999999"]
```
创建完毕后，使用`kubectl exec pod-with-host-path`可查看到宿主机上所有的网络接口。
<br>
<br>

pod可以在拥有自己命名空间的同时绑定节点上的端口，以下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-hostport
spec:
  containers:
  - image: luksa/kubia
    name: kubia
    ports:
    - containerPort: 8080
      hostPort: 9000    #通过hostPort字段绑定至宿主机9000端口
      protocol: TCP
```
hostPort与NodePort不同，NodePort通过服务暴露特定端口，经过该端口的流量会被随机分发至服务中的一个pod；而hostPath暴露一个宿主机上的端口，经过该节点上的流量会被转发至该pod。
<br>
<br>

pod可使用宿主节点的PID与IPC(进程间通信)命名空间，pod将被允许看到宿主机上的全部进程，或通过IPC机制与它们通信，以下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-host-pid-and-ipc
spec:
  hostPID: true     #使用宿主节点的PID命名空间
  hostIPC: true     #使用宿主节点的IPC命名空间
  containers:
  - name: main
    image: alpine
    command: ["sleep", "999999"]
```
创建完毕后，使用`kubectl exec pod-with-host-pid-and-ipc ps aux`可查看到宿主机上进程。
<br>
<br>

**配置节点的安全上下文：** <br>
安全上下文可配置许多内容：
- 指定容器中运行进程的用户（用户ID ）。
- 阻止容器使用root 用户运行。
- 使用特权模式运行容器，使其对宿主节点的内核具有完全的访问权限。
- 与以上相反，通过添加或禁用内核功能，配置细粒度的内核访问权限。
- 设置SELinux，加强对容器的限制。
- 阻止进程写入容器的根文件系统。


使用指定用户运行容器：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-as-user-guest
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsUser: 405    #指明一个用户ID，而不是用户名
```
创建完毕后，使用`kubectl exec pod-as-user-guest id`可查看到用户为guest。
<br>
<br>

阻止容器以root权限运行：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-run-as-non-root
spec:
  containers:
  - name: main
    image: alpine
    command: ["sleep", "999999"]
    securityContext:
      runAsNonRoot: true    #这个容器只允许以非root用户运行
```
<br>
<br>

如同kube-proxy修改宿主机的iptables规则使得Kubernetes中的服务规则生效，有时pod需获取宿主机内核的完整权限，该pod需要在特权模式下运行：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-privileged
spec:
  containers:
  - name: main
    image: alpine
    command: ["sleep", "999999"]
    securityContext:
      privileged: true      #这个容器将在特权模式下运行
```
<br>
<br>

kubernetes允许为特定容器添加内核功能，以允许对容器进行更加精细的权限控制，限制攻击者潜在侵入的影响。以下是为容器添加修改系统时间的功能：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-add-setting-capability
spec:
  containers:
  - name: main
    image: alpine
  command: ["/bin/sleep", "999999"]
  securityContext:
    capabilities:
      add:
      - SYS_TIME    #添加了SYS_TIME内核功能
```
<br>

也可以禁用指定的内核功能，以下为禁用chown：
```yaml
apiVserion: v1
kind: Pod
metadata:
  name: pod-drop-chown-capability
spec:
  containers:
  - name: main
    image: alpine
    command: ["sleep", "999999"]
    securityContext:
      capabilities:
        drop:
        - CHOWN     #在这里禁止了容器修改文件所有者
```
<br>
<br>

有时应用在容器的根文件系统中提供服务，这时为避免攻击者将恶意代码写入，可使用容器根文件系统写入的方法：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-readonly-filesystem
spec:
  containers:
  - name: main
    image: alpine
    command: ["sleep", "999999"]
    securityContext:
      readOnlyRootFilesystem: true  #根文件系统不允许写入
    volumeMounts:
    - name: my-volume
      mountPath: /volume    #向/volume写入是允许的，因为这个目录挂载了一个存储卷
      readOnly: false
  volumes:
  - name: my-volume
    emptyDir:
```
不仅可以设置容器级别的安全上下文，还可设置pod级别的安全上下文，在`pod.spec.securityContext`中设置。
<br>
<br>

**限制pod使用安全相关的特性：** <br>
 PodSecurityPolicy是一种集群级别（无命名空间）的资源，它定义了用户能否在pod中使用各种安全相关的特性。一个PodSecurityPolicy资源可定义以下事项：
 - 是否允许pod使用宿主节点的PID 、IPC 、网络命名空间
 - pod允许绑定的宿主节点端口
 - 容器运行时允许使用的用户ID
 - 是否允许拥有特权模式容器的pod
 - 允许添加哪些内核功能，默认添加哪些内核功能，总是禁用哪些内核功能
 - 允许容器使用哪些SELinux 选项
 - 容器是否允许使用可写的根文件系统
 - 允许容器在哪些文件系统组下运行
 - 允许pod 使用哪些类型的存储卷

可使用`kubectl get psp`查看。 <br>
一个样例：
```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: default
spec:
  hostIPC: false    #不允许使用宿主机的IPC、PID、网络命名空间
  hostPID: false
  hostNetwork: false
  hostPorts:
  - min: 10000  #容器只能绑定宿主节点的范围端口
    max: 11000
  - min: 13000
    max: 14000
  privileged: false     #容器不能在特权模式下运行 
  readOnlyRootFilesystem: true  #使用只读的根文件系统
  runAsUser:    #容器可以任意用户与组运行
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  volumes:  #pod可使用所有类型的存储卷
  - '*'
```
创建该PodSecurityPolicy后，集群中不能再部署使用宿主节点的PID、IPC、网络命名空间的Pod了，容器的根文件系统将变为只读。

配置允许、默认添加、禁止使用的内核功能：
```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
spec:
  allowCapabilities:    #允许容器添加
  - SYS_TIME    
  defaultAddCapabilities:   #为每个容器自动添加
  - CHOWN
  requireddropCapabilities:     #要求容器禁用
  - SYS_ADMIN
  - SYS_MODULE
```
限制pod可使用的存储卷类型：
```yaml
kind: PodSecurityPolicy
spec:
  volumes:
  - emptyDir
  - configMap
  - secret
  - downwardAPI
  - persistentVolumeClaim
```
<br>
<br>

有时需让不同用户遵照不同规则，可使用RBAC将PodSecurityPolicy分配给不同用户。设现在已有default与privileged两个psp。<br>
创建两个clusterRole，分别使用不同的psp：<br>
``kubectl create clusterrole psp-default --verb=use --resource=podsecuritypolicies --resource-name=default``

``kubectl create clusterrole psp-privileged --verb=use --resource=podsecuritypolicies --resource-name=privileged``

将psp-default这个clusterRole绑定至已认证用户组：<br>
``kubectl create clusterrolebinding psp-all-users --clusterrole=psp-default --group=system:authenticated``

将psp-privileged绑定至一个用户Bob：<br>
``kubectl create clusterrolebinding psp-bob --clusterrole=psp-privileged --user=bob``

现在，普通用户使用default政策，而bob可使用default政策与特权政策了。
<br>
<br>


**隔离pod的网络：** <br>
同一命名空间中的限制访问：

假设在foo namespace中正运行一个数据库pod，以及一个使用该数据库的网页服务器pod,其他pod也在这个命名空间中运行，然而它们不允许连接数据库。<br>
这时需要在该命名空间中创建一个NetworkPolicy资源：
```yaml
apiVeriosn: networking.k8s.io/v1
kind: NetworkPolicy
matadata:
  name: postgres-netpolicy
spec:
  podSelector:
    matchLabels:    #通过标签选择器选中app=database的标签
      app: database
  ingress:
  - form:   #只允许来自具有app=webserver标签的pod的访问
    - podSelector:
        matchLabels:
          app: webserver
    ports:
    - port: 5432    #允许对这个端口的访问
```
创建后，仅允许含app=webserver标签的pod访问app=database标签的pod。
<br>
<br>

不同命名空间中的网络隔离：
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: shoppingcart-netpolicy
spec:
  podSelector:
    matchLabels:    #该策略应用于具有app=shopping-cart标签的Pod
      app: shopping-cart    
  ingress:
  - from:
    - namespaceSelector:    #只有在具有tenant=manning标签的命名空间中运行的Pod可访问该服务
        matchLabels:
          tenant: manning
    ports:
    - port: 80
```

还可使用CIDR隔离网络：
```yaml
ingress:
- from:
  - ipBlock:
      cidr: 192.168.1.0/24  #在入向规则中指明网段
```

限制pod的对外访问流量：
```yaml
spec:
  podSelector:
    matchLabels:    #该策略应用于包含app=webserver标签的pod
      app: webserver
  egress:   #限制pod出网流量
  - to:
    - podSelector:
        matchLabels:    #webserver的pod只能与有app=database的标签的pod通信
          app: database
```

### 14.计算资源管理
**为pod中的容器申请资源：**<br>
创建Pod时可为每个容器单独指定CPU和内存的资源请求量。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: request-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      requests:
        cpu: 200m   #申请至少200毫核的CPU，即1/5核
        memory: 10Mi    #申请10MB内存
```
而调度器在调度pod只关注资源的request量，并不关注实际使用量。例如，有三个pod共申请了节点80%的CPU资源，而实际使用量为70%。但第四个占用25%CPU的pod无法调度至该节点上。

<br>
<br>

**限制容器的可用资源：**
一个包含CPU和内存限制的pod：<br>
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      limits:   #因为没有指定requests，它   将被设置为与limits相同的值
        cpu: 1  #最大允许使用1核CPU
        memory: 20Mi    #最大允许使用20Mi内存
```
与requests不同的是，所有limits的综合可超过节点的总资源。<br>

创建该pod，并进入容器内使用`top`查看资源占用：
<img src="https://gallery.angelo.org.cn/images/2020/05/16/QQ20200516105229.png">
内存占用量大大超出了20Mi，且CPU始终未到100%。这是因为在容器中看到的是节点的内存，而不是容器本身的内存。
容器内同样也可看到节点上所有的CPU核。

<br>
<br>

**了解pod QoS等级：** <br>
既然limits可以超额分配，那么当新pod调度至节点时，若占用内存总量超出了节点资源，被杀死的将会是哪个pod？Kubernetes无法自己做出决策，此时便需要依照QoS来决策。
Kubernetes将pod划分为三种QoS等级：
- BestEffort(优先级最低)
- Burstable
- Guaranteed(优先级最高)

当创建pod时未指定requests与limits，pod为BestEffort类型。
当创建pod时指定的requests与limits相等，pod为Guaranteed类型。
其他情况下为Burstable类型。

<br>
<br>

**为命名空间中的pod设置默认的requests和limits：** <br>
通过创建一个LimitRange资源，避免必须为每个容器配置requests与limit：
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example
spec:
  limits:
  - type: Pod   #指定整个pod的资源limits
    min:    #pod中所有容器的CPU和内存的请求量的最小值
      cpu: 50m 
      memory: 5Mi
    max:    #pod中所有容器的CPU和内存的请求量的最大值
      cpu: 1
      memory: 1Gi
  - type: Container     #指定每个容器的资源限制
    defaultRequest:     #没有指定请求量时的默认值
      cpu: 100m
      memory: 10Mi
    default:    #没有指定限制量时的默认值
      cpu: 200m
      memory: 100Mi
    min:
      cpu: 50m
      memory: 5Mi
    max:
      cpu: 1
      memory: 1Gi
    maxLimitRequestRatio:   #requests与limits的最大比值
      cpu: 4
      memory: 10
  - type: PersistentVolumeClaim     #还可指定请求PVC存储容量的最小值与最大值
    min:
      storage: 1Gi
    max:
      storage: 10Gi
```

为每个命名空间指定自己的LimitRange，可以区分不同团队所需的资源。

<br>
<br>

**限制命名空间中的可用资源总量：** <br>
LimitRange只应用于单个pod。当需限制命名空间中的资源总量时，可创建一个ResourceQuota对象实现：
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cpu-and-mem
spec:
  hard:
    requests.cpu: 400m
    requests.memory: 200Mi
    limits.cpu: 600m
    limits.memory: 500Mi
```
应用于当前命名空间。<br>
可使用`kubectl describe quota`查看限额。

<br>

ResourceQuota同样可为持久化存储指定配额：
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage
spec:
  hard:
    requests.storage: 500Gi     #可声明存储总量
    ssd.storageclass.storage.k8s.io/requests.storage: 300Gi    #StorageClass ssd的可申请的存储量
    standard.storageclass.storage.k8s.io/erquests.storage: 1Ti     #低性能的HDD存储限制为1TB
```

<br>

ResourceQuota还可限制可创建对象的个数：
```yaml
apiversion: v1
kind: ResourceQuota
metadata:
  name: objects
spec:
  hard:
    pods: 10    #这个命名空间最多创建10个pod
    replicationcontrollers: 5   #最多5个rc
    secrets: 10
    configmaps: 10
    persistentvolumeclaims: 4
    services: 5     #最多5个service
    services.loadbalancers: 1   #最多1个loadbalancer类型的服务
    services.nodeports: 2   #最多2个nodeport类型的服务
    ssd.storageclass.storage.k8s.io/persistentvolumeclaims: 2  #最多声明2个StorageClass为ssd的PVC
```

<br>
<br>

**监控pod的资源使用量：** <br>
对资源的实际使用量进行实时监控，以设置最佳的requests与limits值。如何监控一个在Kubernetes中运行的应用？需使用Heapster附加组件。 <br>
在minikube中开启Heapster组件：
``minikube addons enable heapster``

<br>

开启Heapster后，可使用``kubectl top node``显示节点资源使用量。
使用`kubectl top pod`查看每个pod使用了多少资源，使用--container选项查看容器的资源使用量。

<br>
当需要保存并分析历史资源的使用统计信息时(如过去某段时间的资源使用信息)，往往使用InfluxDB来存储统计数据，然后使用Grafana对数据进行可视化分析。

在minikube中打开Heapster后，可使用`minikube service monitoring-grafana -n kube-system`在默认浏览器中打开Grafana控制台：
<img src="https://gallery.angelo.org.cn/images/2020/05/16/QQ20200516105229133dc77cc656aacb.png">

### 15.自动横向伸缩pod与集群节点
Kubernetes可进行pod与节点级别的自动伸缩。

**pod的横向自动伸缩：** <br>
通过创建一个HorizontalpodAutoscaler(HPA)资源来启用和配置Horizontal 控制器，pod的横向自动伸缩由Horizontal控制器执行。<br>
获取pod的度量数据的过程：
<img src="https://gallery.angelo.org.cn/images/2020/05/17/QQ20200517084835.png">

<br>

进行基于CPU使用量的自动伸缩。首先创建一个Deployment资源：
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3   #副本数为3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
        resources:
          requests:
            cpu: 100m   #每个pod请求100毫核的CPU
```
然后创建一个HPA对象，将它指向该Deployment。可使用`kubectl autoscale`命令： <br>
``kubectl autoscale deployment kubia --cpu-percent=30 --min=1 --max=5`` <br>
设置了目标CPU使用率为30%，制定了副本最小和最大数量。

可使用`kubectl get hpa`显示HPA资源。

<br>

还可进行基于内存使用量的自动伸缩，这与基于CPU用量的自动伸缩相似。而容器的内存占用并不是自动伸缩的一个好度量，因为，如果增加副本数不能导致被观测度量平均值的线性下降，autoscale就不能正常工作了。

<br>
<br>

**pod的纵向自动伸缩：** <br>
某些应用无法横向伸缩。对这些应用而言，唯一的选项是纵向伸缩--给它们更多CPU与内存。

写作本书时尚未有pod的纵向自动伸缩。 <br>
而目前GKE(Google Kubernetes Engine)实现了纵向伸缩，文档为：
[配置pod自动纵向扩缩](https://cloud.google.com/kubernetes-engine/docs/how-to/vertical-pod-autoscaling?hl=zh-cn)

<br><br>

**集群节点的横向伸缩：** <br>
若所有节点满载，则需扩展节点。若Kubernetes运行于自建基础架构，需添加一台物理机，并将其加入集群若集群运行于云端基础架构之上，则可向云端API做调用。 <br>
Kubernetes的Cluster Autoscaler支持从云服务提供者请求更多节点。
<br>

启动Cluster Autoscaler 的方式取决于你的Kubemetes集群运行在哪家云服务提供商。具体应参考对应文档。
<br><br>

有一些服务要求至少保证一定数量的pod持续运行，当缩容时，则需指定最少需要维持运行的Pod数量。可通过创建podDisruptionBudget资源实现。 <br>
``kubectl create pdb kubia-pdb --selector=app=kubia --min-available=3`` <br>
即可通过标签选择器限定至少运行的pod数量。

### 16.高级调度
Kubernetes有许多机制使得pod调度至指定节点，或不让pod调度至指定节点。
<br><br>

**使用污点和容忍度阻止pod调度至特定节点：** <br>
一个pod只有容忍了节点的污点，才能被调度到该节点上。 <br>
可使用`kubectl describe node`查看节点的`taint`字段表示的污点信息。每个污点可关联一个效果，效果包含：
- NoSchedule表示如果pod没有容忍这些污点，pod则不能被调度至包含这些污点的节点上。
- PreferNoSchedule表示尽量阻止pod被调度至这个节点上。
- NoExecute表示在该节点上运行着的pod，如果没有容忍这个NoExecute污点，将会从这个节点去除。

<br>

可使用`kubectl taint node <node名> node-type=NoSchedule`在节点上添加自定义污点。<br>
在区分生产环境与测试环境时，需在Pod清单中添加容忍选项：
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: prod
spec:
  replicas: 5
  template:
    spec:
      ...
      tolerations:      #此处污点容忍度允许pod被调度至生产环境节点上
      - key: node-type
        operator: Equal
        value: production
        effect: NoSchedule
```

<br><br>

**使用节点亲缘性将pod调度至特定节点上：** <br>
与节点选择器类似，每个pod可以定义自己的节点亲缘性规则，这些规则可以允许你指定硬件性质或者偏好。 <br>
如果将节点选择器替换为节点亲缘性规则，则pod定义将会如下所示：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: gpu
            operator: In
            values:
            - "true"
```
`requiredDuringScheduling...`表明了该字段下定义的规则，为了让Pod能调度至该节点上，明确指出了该节点必须包含的空间。 <br>
`IgnoredDuringExecution`表明不会影响已经在节点上运行的pod。
<br>

当调度pod优先考虑某些节点时，首先给节点加上标签，然后指定优先级节点亲缘性规则，下面展示了优先选择zone1中的dedicated节点：
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: pref
spec:
  template:
    ...
    spec:
      affinity:
        nodeAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:   #指定优先级，这不是必需的
        - weight: 80    #节点优先调度至zone1
          preference:
            matchExpressions: 
            - key: availability-zone
              operator: In
              values:
              - zone1
              - weight: 20  
                preference:     #同时优先调度pod到独占节点，但是该优先级zone1优先级的1/4
                  matchExpressions:
                  - key: share-type 
                    operaotr: In
                    values:
                    - dedicated
```

<br><br>

**使用pod亲缘性与非亲缘性对pod进行协同部署：** <br>
可在pod定义中指定pod亲缘性：
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 5
  template:
    ...
    spec:
      affinity:
        podAffinity:    #定义节点亲缘性规则
          requiredDuringSchedulingIgnoredDuringExecution:    #定义一个强制性要求，而不是偏好
          - topologyKey: kubernetes.io/hostname #topologyKey决定了pod被调度的范围
            labelSelector:
              matchLabels:  #本次部署的pod，必须被调度至匹配pod选择器的节点上
                app: backend
      ...
```
注：除了使用简单的matchLabels字段，还可使用表达能力更强的matchExpressions字段。 <br>

<br>
使用pod的非亲缘性分开调度pod时，只需将podAffinity字段换成podAntiAffinity即可。

### 17.开发应用的最佳实践
一个典型应用中所使用的各个Kubernetes组件：
<img src="https://gallery.angelo.org.cn/images/2020/05/19/QQ20200519090059.png">

<br><br>

**了解pod的生命周期：** <br>
在实际应用中，需注意pod：
- 预料到本地IP和主机名会发生变化
- 预料到写入磁盘的数据会消失
- 使用存储卷来跨容器持久化存储数据

有时需以固定顺序启动Pod中的容器，这是通过在pod中包含一个叫做init的容器来实现的。<br>
一个pod可用拥有任意数量的init容器。init容器是顺序执行的，并且仅当最后一个init容器执行完毕才会去启动主容器。 <br>
init容器可在pod配置文件中的spec.initContainers中定义，下面为检测第7章fortune服务的一个init容器的yaml内容：
```yaml
spec:
  initContainers:
  - name:init
    image: busybox
    command:
    - sh
    - -c
    - 'while true; do echo "waiting for service to come up..."; wget http://fortune -q -T l -O /dev/null > /dev/null 2>/dev/null && break; sleep 1; done; echo "service is up.Staring main container."'
```

<br>
部署人员也可在容器中设置容器"启动后hook"与"终止前hook"，在容器启动后与终止前执行某些特殊操作。

<br><br>

**确保所有的客户端请求都得到了妥善处理：** <br>
妥善关闭一个应用包括如下步骤：
1. 等待几秒钟，然后停止接收新的连接
2. 关闭所有没有请求过来的长连接
3. 等待所有的请求都完成
4. 然后完全关闭应用

<br>

**让应用在Kubernetes中方便运行和管理：**
- 合理给镜像打标签，正确使用ImagePullPolicy
- 使用多维度而不是单维度的标签
- 通过注解描述每个资源
- 将日志输出至标准终端以便轻易通过`kubectl logs`查看


### 18.Kubernetes应用扩展
**自定义API对象：** <br>
只需向API服务器提交`CustomResourceDefinitions`对象，即可定义新的资源类型。以下为定义一个网站类资源的CRD：
```yaml
apiVersion: apiextensions.k8s.io/v1beta1    #CRD所属的API集群和版本号
kind: CustomResourceDefinition
metadata:
  name: websites.extensions.example.com     #自定义对象的全名
spec:
  scope: Namespaced  #期望这种资源属于某个命名空间下
  group: extensions.example.com     #定义API集群、网站资源版本
  version: v1
  names:    #指定自定义对象名称的各种形式
    kind: Website
    singular: website
    plural: websites
```
将其发布至Kubernetes:
<img src="https://gallery.angelo.org.cn/images/2020/05/20/QQ20200520085226.png">
<br><br>

然后便可创建自定义网站实例：
```yaml
apiVersion: extensions.example.com/v1
kind: Website
metadata:
  name: kubia
spec:
  gitRepo: https://github.com/luksa/kubia-website-example.git    #从git仓库拉取
```
<img src="https://gallery.angelo.org.cn/images/2020/05/20/QQ202005200852269747914fb9432344.png">
<br><br>

可使用`ke get websites`查看生成的自定义网站资源：
<img src="https://gallery.angelo.org.cn/images/2020/05/20/QQ20200520085226b649a1c4bf01ebab.png">

与所有Kubernetes资源一样，可以创建并列出自定义资源实例。 <br>

为让该网站运行一个通过服务暴露的web服务器pod，需构建和部署一个网站控制器。控制器将创建Deployment资源，而不是直接创建非托管pod，这样便能确保既能被管理，还能在节点遇到故障时继续正常工作。 <br>
以下是一个控制器示例：
```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: website-controller
spec:
  replicas: 1   #单个副本
  template:
    metadata:
      name: website-controller
      labels:
        app: website-controller
    spec:
      serviceAccountName: website-controller    #在特殊服务账户下运行
      containers:   #两个容器：主容器和代理sidecar容器
      - name: main
        image: luksa/website-controller
      - name: proxy
        image: luksa/kubectl-proxy:1.6.2
```
由于该Deployment在特殊的服务账户下运行，因此需要在部署控制器前创建一个服务账户： <br>
``kubectl create serviceaccount website-controller``

若在集群中启用了基于角色的访问控制(RABC)，此时需要将web-controller服务账户绑定至cluster-admin的ClusterRole：
``kubectl create clusterrolebinding website-controller --clusterrole=cluster-admin --serviceaccount=default:website-controller``
<img src="https://gallery.angelo.org.cn/images/2020/05/20/QQ20200520085226241551d3de5b64ca.png">
<br><br>

然后便可部署控制器了。
<br><br>

**基于Kuberbetes搭建的平台：** <br>
Kubernetes中的组件可以轻松扩展，它被大量新一代PaaS产品采用。 <br>
基于Kubernetes搭建的平台有：
- 实现应用程序快速开发、便捷部署、轻松扩展、长期维护的红帽OpenShift平台
- Microsoft的Helm平台
