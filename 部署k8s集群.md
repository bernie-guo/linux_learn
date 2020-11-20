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

#### 环境准备
- 1.设置master节点和node节点的主机名
	master执行：hostnamectl --static set-hostname  k8s-master
	node执行：hostnamectl --static set-hostname  k8s-node-1
	重启生效：reboot
- 2.修改master和node上的hosts
	在master和slave的/etc/hosts文件中均加入以下内容：
	```
	127.0.0.1   		k8s-master
	127.0.0.1  			etcd
	127.0.0.1  			registry
	192.168.117.136  	k8s-node-1
	```
- 3.关闭master和slave上的防火墙
	systemctl disable firewalld.service
	systemctl stop firewalld.service

#### 部署msater节点
- 1.etcd安装
	- 安装命令：yum install etcd -y
	- 编辑etcd的默认配置文件/etc/etcd/etcd.conf
		修改：
		ETCD_LISTEN_CLIENT_URLS="http://本机地址：2379,http://127.0.0.1：2379"
		ETCD_ADVERTISE_CLIENT_URLS="http://etcd:2379"
	- 启动etcd并设置开机自启动
		systemctl start etcd 
		systemctl enable etcd
	- 查看etcd健康指标
		etcdctl -C http://etcd:2379 cluster-health
- 2.安装flannel
	- 安装命令：yum install flannel
	- 配置flannel：
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

#### 部署node节点
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
	- 安装：yum install  吗  
			
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

#### 验证集群状态
- 1.查看断点信息
	kubectl get endpoints
- 2.查看集群信息
	kubectl cluster-info
- 3.查看集群中节点状态
	kubectl get nodes

#### k8s集群里的概念和操作命令	
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









