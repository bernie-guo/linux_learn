## 部署K8S
#### 第一步：环境介绍
- 1.Master节点（192.168.117.134）
	docker
	etcd
	flannel
	kube-apiserver
	kube-scheduler
	kube-controller-manager
	
- 2.Node节点（192.168.117.135）
	docker
	flannel
	kubelet
	kube-proxy	
- 3.client节点（）
	客户端
- 4.docker镜像仓库
			阿里云镜像管理

#### 第二步：环境准备
- 1.设置master节点和node节点的主机名
	master执行：hostnamectl --static set-hostname  k8s-master
	node执行：hostnamectl --static set-hostname  k8s-node-1
	
	edge执行：hostnamectl --static set-hostname  k8s-edge-1
	
- 重启生效：reboot
	
- 2.修改master和node上的hosts
	在master和slave的/etc/hosts文件中均加入以下内容：
	
	```
	cat > /etc/hosts << EOF
	192.168.117.139   		k8s-master
	192.168.117.139  			etcd
	192.168.117.139  			registry
	192.168.117.142  	vimk8s-node-1
	192.168.117.146     k8s-edge-1
	EOF
	```
	
- 3.关闭master和slave上的防火墙
	systemctl disable firewalld.service
	systemctl stop firewalld.service

#### 第三步：部署msater节点
- 1.etcd安装
	- 安装命令：yum install etcd -y
	- 编辑etcd的默认配置文件/etc/etcd/etcd.conf
		修改：
		ETCD_LISTEN_CLIENT_URLS="http://本机地址:2379,http://127.0.0.1:2379"
		ETCD_ADVERTISE_CLIENT_URLS="http://etcd:2379"
	- 设置etcdctl版本为2：export ETCDCTL_API=2
	- 启动etcd并设置开机自启动
		systemctl start etcd 
		systemctl enable etcd
	- 查看etcd健康指标
		etcdctl -C http://etcd:2379 cluster-health
- 2.安装flannel
	- 安装命令：yum install flannel
	- 配置flannel：/etc/sysconfig/flanneld
	  修改：FLANNEL_ETCD_ENDPOINTS="http://etcd:2379"
	- 配置etcd中关于flannel的key：etcdctl mk /atomic.io/network/config '{ "Network": "10.0.0.0/16" }'
	- 启动并设置开机自启动
		systemctl start flanneld.service
		systemctl enable flanneld.service
- 3.安装docker
	
	- 安装命令：yum install docker -y
	- 开启docker服务：service docker start
	- 设置docker开启自启动：chkconfig docker on
	- 添加阿里云镜像加速器
	```
				sudo mkdir -p /etc/docker
				sudo tee /etc/docker/daemon.json <<-'EOF'
				{
				  "registry-mirrors": ["https://hzn76ac5.mirror.aliyuncs.com"]
				}	
				EOF
				sudo systemctl daemon-reload
				sudo systemctl restart docker
	```
- 4.安装k8s
	- 安装：yum install kubernetes
	- 配置/etc/kubernetes/apiserver文件
		修改：
		KUBE_API_ADDRESS="--address=0.0.0.0"
		KUBE_ETCD_SERVERS="--etcd-servers=http://etcd:2379"
		KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
	- 配置/etc/kubernetes/config文件
		修改：KUBE_MASTER="--master=http://k8s-master:8080"
	- 启动k8s各个组件
		systemctl start kube-apiserver.service
		systemctl start kube-controller-manager.service
		systemctl start kube-scheduler.service
	- 设置k8s各组件开机启动
		systemctl enable kube-apiserver.service
		systemctl enable kube-controller-manager.service
		systemctl enable kube-scheduler.service

#### 第四步：部署node节点
- 1.安装flannel
	- 安装命令：yum install flannel
	- 配置flannel：/etc/sysconfig/flanneld
		修改：FLANNEL_ETCD_ENDPOINTS="http://etcd:2379"
	- 启动并设置开机自启动
		systemctl start flanneld.service
		systemctl enable flanneld.service
- 2.安装docker
	
	- 安装命令：yum install docker -y
	- 开启docker服务：service docker start
	- 设置docker开启自启动：chkconfig docker on
	- 添加阿里云镜像加速器
	```
				sudo mkdir -p /etc/docker
				sudo tee /etc/docker/daemon.json <<-'EOF'
				{
				  "registry-mirrors": ["https://hzn76ac5.mirror.aliyuncs.com"]
				}	
				EOF
				sudo systemctl daemon-reload
				sudo systemctl restart docker
	```
- 3.安装k8s
	
	- 安装：yum install   kubernetes  
		
	- 配置/etc/kubernetes/config文件
		修改：KUBE_MASTER="--master=http://k8s-master:8080"
	- 配置/etc/kubernetes/kubelet
		修改：
		KUBELET_ADDRESS="--address=0.0.0.0"
		KUBELET_HOSTNAME="--hostname-override=k8s-node-1"
		KUBELET_API_SERVER="--api-servers=http://k8s-master:8080"
	- 启动k8s各个组件
		systemctl start kubelet.service
		systemctl start kube-proxy.service
	- 设置k8s各组件开机启动
		systemctl enable kubelet.service
		systemctl enable kube-proxy.service

#### 第五步：验证集群状态
- 1.查看断点信息
	kubectl get endpoints
- 2.查看集群信息
	kubectl cluster-info
- 3.查看集群中节点状态
	kubectl get nodes

#### 第六步：k8s集群里的概念和操作命令	
- 1.pod：Pod是一组容器的集合，且一个Pod只能运行在一个Node上，Pod是kubernetes调度、部署、扩展的基本单位
	- kubectl run命令创建一个pod：kubectl run pod-example --image=nginx
	- kubectl create命令创建pod：kubectl create -f create_pod.yaml
	- 删除pod：kubectl delete po pod名
	- 查看pod启动情况：kubectl get pods， kubectl get pods -o wide
	- 查看pod的详细信息：kubectl discrible po pod名
	
- 2.Namespace：Namespace是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户组。常见的pods, services, replication controllers和deployments等都是属于某一个namespace的（默认是default），而node, persistentVolumes等则不属于任何namespace。
	- 创建 kubectl create namespace namespace名
	- 删除 kubectl delete namespaces namespace名
	- 查询 kubectl get namespaces
	
- 3.Replication Controller（RC）：保证了在所有时间内，都有特定数量的Pod副本正在运行，如果太多了，Replication controller就杀死几个，如果太少了，Replication Controller会新建几个，和直接创建的pod不同的是，Replication Controller会替换掉那些删除的或者被终止的pod，而不管删除的原因是什么。
	- 创建：kubectl create -f create_rc.json
	- 查看rc启动情况：kubectl get rc
	
- 4.Service 是一个定义了一组Pod的策略的抽象，可以理解为抽象到用户层的一个宏观服务。其实这个概念在Swarm集群里也有，可以参照理解。Service的这样一层抽象最起码可以抵御两个方面的问题：Pod重启后IP地址的变化（保持用我们系统的客户其IP不变）
	提供负载均衡能力
	- 创建service：kubectr create -f redis-master-service.json
	- 查看service的启动情况：kubectl get svc

- 5.关于pod的实例练手

  - 创建yaml文件 vim k8s_pod.yml

    ```kotlin
    apiVersion: v1
    kind: Pod
    metadata:
      name: two-containers
    spec:
    
      restartPolicy: Never
    
      volumes:
      - name: shared-data
        emptyDir: {}
    
      containers:
    
      - name: nginx-container
        image: nginx
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
    
      - name: debian-container
        image: debian
        volumeMounts:
        - name: shared-data
          mountPath: /pod-data
        command: ["/bin/sh"]
        args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
    ```

  - 创建pod：`kubect create -f two-container-pod.yaml`

  - 查看pod和容器信息：`kubectl get pod two-containers --output=yaml`
  	输出内容大致如下:
  	
  	```cpp
  	apiVersion: v1
  	kind: Pod
  	metadata:
  	  ...
  	  name: two-containers
  	  namespace: default
  	  ...
  	spec:
  	  ...
  	  containerStatuses:
  	
  	  - containerID: docker://c1d8abd1 ...
  	    image: debian
  	    ...
  	    lastState:
  	      terminated:  // debian容器已停止
  	        ...
  	    name: debian-container
  	    ...
  	
  	  - containerID: docker://96c1ff2c5bb ...
  	    image: nginx
  	    ...
  	    name: nginx-container
  	    ...
  	    state:
  	      running:  // nginx容器已运行
  	    ...
  	```

	可以看到debian容器已经停止了，nginx容器依然运行
	
	- 我们进入nginx容器的/bin/bash并验证nginx服务器
	
	```
	kubectl exec -it two-containers -c nginx-container -- /bin/bash
	```
	
	执行如下命令来安装curl：

```ruby
root@two-containers:/# apt-get update
root@two-containers:/# apt-get install curl procps
root@two-containers:/# ps aux
```

然后执行`curl localhost`,可以获得输出：

> Hello from the debian container

可见debian容器中写入的东西在nginx容器中获得了，因此文件交换也完成了

- 6.RC和Service练手

  部署一个包含Redis集群、基于PHP的留言板系统

  - 创建redis-master的RC

    redis-master-controller.yaml内容：

    ```cpp
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: redis-master
    spec:
      replicas: 1
      selector:
        name: redis-master
      template:
        metadata:
          name: redis-master
          labels:
            name: redis-master
        spec:
          containers:
          - name: redis-master
            image: kubeguide/redis-master
            ports:
            - containerPort: 6379
    ```

  创建该RC并查看创建结果：

  ```csharp
  kubectl create -f redis-master-controller.yaml
  kubectl get rc
  kubectl get pods
  ```

  - 创建redis-master的**service**

  redis-master-service.yaml内容：

  ```undefined
  apiVersion: v1
  kind: Service
  metadata:
    name: redis-master
    labels:
      name: redis-master
  spec:
    ports:
    - port: 6379
      targetPort: 6379
    selector:
      name: redis-master
  ```

  创建该Service并查看：

  ```csharp
  kubectl create -f redis-master-service.yaml
  kubectl get services
  ```

  - 创建redis-slave的**RC**

  redis-slave-controller.yaml文件内容：

  ```csharp
  apiVersion: v1
  kind: ReplicationController
  metadata:
    name: redis-slave
  spec:
    replicas: 2
    selector: # RC通过spec.selector来筛选要控制的Pod
      name: redis-slave
    template:
      metadata:
        name: redis-slave
        labels: # Pod的label，可以看到这个label与spec.selector相同
          name: redis-slave
      spec:
        containers:
        - name: redis-slave
          image: kubeguide/guestbook-redis-slave
          env:
          - name: GET_HOSTS_FROM
            value: env
          ports:
          - containerPort: 6379
  ```

  创建该RC并查看

  ```csharp
  kubectl create -f frontend-controller.yaml
  kubectl get rc
  kubectl get pods
  ```

  - 创建frontend的**Service**

  frontend-service.yaml文件内容如下：

  ```bash
  apiVersion: v1
  kind: Service
  metadata:
    name: frontend
    labels:
      name: frontend
  spec:
    type: NodePort
    ports:
    - port: 80
      nodePort: 30001
    selector:
      name: frontend
  ```

  创建该Service并查看

  ```csharp
  kubectl create -f frontend-service.yaml
  kubectl get services
  ```

  - 访问：http://your-host:30001/

#### 第七步：制作基础镜像

- 1.登录阿里云镜像仓库

  docker login --username=guoghb782 registry.cn-beijing.aliyuncs.com

- 制作centos7.4镜像

  #### 1、去远端拉取一个最新的centos最基础镜像，基于此镜像来制作

  ```
  docker pull centos
  ```

  #### 2、启动该docker容器

  ```
  docker run -it centos：latest /bin/bash
  ```

  #### 3、在启动的容器中来安装sshd

  ```undefined
  yum -y install openssh-server
  yum -y install openssh-clients
  ```

  #### 4、我们来尝试启动一下sshd服务，会发现有报错

  启动sshd服务命令: `/usr/sbin/sshd -D`
   报如下错误：

  ```jsx
  Could not load host key: /etc/ssh/ssh_host_rsa_key
  Could not load host key: /etc/ssh/ssh_host_ecdsa_key
  Could not load host key: /etc/ssh/ssh_host_ed25519_key
  ```

  我们来解决以上错误：

  ```bash
  ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key -N ""
  ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key -N ""
  ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key -N ""
  ```

  此时再次来启动sshd服务应该无错误了

  #### 5、编辑sshd_config配置文件

  执行命令：`vim /etc/ssh/sshd_config`
   将配置文件中原本`UsePAM yes`换成`UsePAM no`

  #### 6、修改root的密码

  执行命令：`passwd root`
   输入两次密码即可

  #### 7、我们用exit命令来退出容器

  #### 8、基于刚退出的容器我们来制作带ssh功能的centos镜像

  ```
  docker commit bf5b84f8e2d8 docker.io/hansonwang/centos7.4_ssh
  ```

  （1）**注意**此处的bf5b84f8e2d8即为刚才运行的容器的id，可用docker ps -a查看

  （2）**注意**此处的commit格式，必须为docker.io/<你的dockerhub用户名>/centos7.4_ssh

- push镜像到远端

  ```
  docker push docker.io/hansonwang/centos7.4_ssh:latest
  ```

  > 同样需要注意此处的push格式，必须为docker.io/<你的dockerhub用户名/完整的镜像名

- 效果验证

为了验证镜像确实被推到远端，我们将本地刚打包好的镜像删除，然后从远端pull下来运行看看

```
docker pull hansonwang/centos7.4_ssh
```

可以成功pull下来：

测试一下该镜像里是否包含有ssh组件：运行其并用ssh连接到容器中：
 运行容器：`docker run -d -p 2222:22 docker.io/hansonwang/centos7.4_ssh:latest /usr/sbin/sshd -D`
 ssh接入：`ssh root@localhost -p 2222`

#### 部署简单应用

- 在master 节点上创建一个deployment

  ```bash
  kubectl create deployment nginx --image=nginx
  ```

可以看到一个叫nginx的deployment创建成功了。

```bash
root@ubuntu:/home/cong# kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
root@ubuntu:/home/cong# kubectl get deployments
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     1         1         1            1           11m
```

- 创建一个service

  ```bash
  kubectl create service nodeport nginx --tcp 80:80
  ```

  可以看到一个叫nginx的service创建成功了，这里kubectl get svc是kubectl get services的简写。

  ```bash
  root@ubuntu:/home/cong# kubectl create service nodeport nginx --tcp 80:80
  service/nginx created
  root@ubuntu:/home/cong# kubectl get svc
  NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
  kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        3d
  nginx        NodePort    10.107.237.157   <none>        80:30601/TCP   11s
  ```

- 在slave节点上执行下面的命令验证一下nginx有没有部署成功。

```bash
curl localhost:30601
或者
curl kube-slave:30601
```

效果如下:

```bash
root@ubuntu:/home/cong# curl localhost:30601
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

浏览器打开试试，nginx的首页显示出来