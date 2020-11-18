#### k8s核心
		-1.核心组件
			- 配置存储中心-->etcd服务
			
			- 主控节点（master）
				- kube-apiserver服务
				- kube-controller-manager服务
				- kube-scheduler服务
			- 运算节点（node）
				- kube-kubelet服务
				- kube-proxy服务
		-2.CLI客户端
			- kubectl
		-3.核心附件
			- CNI网络插件->flannnel/calico
			- 服务发现用插件->coredns
			- 服务暴露用插件->reefik
			- GUI管理插件->Dashboard
