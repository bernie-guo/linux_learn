## KubeEdge
### KubeEdge介绍
- 1、优势
  kubernetes + 容器的组合大大提高了用户创建部署应用的效率。kubernetes 可以把 n 台主机整合成一个集群，用户在 master 节点上通过编写一个 yaml 或者 json 格式的配置文件，也可以通过命令等请求 Kubernetes API 创建应用，就直接将应用部署到集群上的各个节点上，该配置文件中还包含了用户想要应用程序保持的状态，从而生成用户想要的环境。
  Kubernetes 作为容器编排的标准，自然会想把它应用到边缘计算上，即通过 kubernetes 在边缘侧部署应用

  

  但是 kubernetes 在边缘侧部署应用时遇到了一些问题，例如：

  边缘侧设备没有足够的资源运行一个完整的 Kubelet
  一些边缘侧设备是 ARM 架构的，然而大部分的 Kubernetes 发行版并不支持 ARM 架构
  边缘侧网络很不稳定，甚至可能完全不通，而 kubernetes 需要实时通信，无法做到离线自治
  很多边缘设备都不支持TCP/IP 协议
  Kubernetes 客户端（集群中的各个Node节点）是通过 list-watch 去监听 Master 节点的 apiserver 中资源的增删改查，list-watch 中的 watch 是调用资源的 watch API 监听资源变更事件，基于 HTTP 长连接实现，而维护一个 TCP 长连接开销较大。从而造成可扩展性受限。

  

  为了解决包含但不限于以上 Kubernetes 在物联网边缘场景下的问题，从而产生了KubeEdge 。对应以上问题：
  KubeEdge 保留了 Kubernetes 的管理面，重新开发了节点 agent，大幅度优化让边缘组件资源占用更低很多
  KubeEdge 可以完美支持 ARM 架构和 x86 架构
  KubeEdge 有离线自治功能，可以看 MetaManager 组件的介绍
  KubeEdge 丰富了应用和协议支持，目前已经支持和计划支持的有：MQTT、BlueTooth、OPC UA、Modbus等。
  KubeEdge 通过底层优化的多路复用消息通道优化了云边的通信的性能，可以看 EdgeHub 组件的介绍

- 2、 云边通信
  CloudHub: CloudHub 是一个 Web Socket 服务端，用于大量的 edge 端基于 websocket 或者 quic 协议连接上来。负责监听云端的变化, 缓存并发送消息到 EdgeHub。
  EdgeHub: 是一个 Web Socket 客户端，负责将接收到的信息转发到各edge端的模块处理；同时将来自个edge端模块的消息通过隧道发送到cloud端。提供可靠和高效的云边信息同步。
  如何配置通信协议

- 3、 云上部分
  EdgeController: 用于控制 Kubernetes API Server 与边缘的节点、应用和配置的状态同步。
  DeviceController: DeviceController 是一个扩展的 Kubernetes 控制器，管理边缘设备，确保设备信息、设备状态的云边同步。

- 4、边缘部分
  MetaManager: MetaManager 模块后端对应一个本地的数据库（sqlLite），所有其他模块需要与 cloud 端通信的内容都会被保存到本地 DB 种一份，当需要查询数据时，如果本地 DB 中存在该数据，就会从本地获取，这样就避免了与 cloud 端之间频繁的网络交互；同时，在网络中断的情况下，本地的缓存的数据也能够保障其稳定运行（比如你的智能汽车进入到没有无线信号的隧道中），在通信恢复之后，重新同步数据。是边缘节点自治能力的关键；
  Edged: 是运行在边缘节点的代理，用于管理容器化的应用程序。算是个重新开发的轻量化 Kubelet，实现 Pod，Volume，Node 等 Kubernetes 资源对象的生命周期管理
  EventBus: EventBus 是一个与 MQTT 服务器（mosquitto）交互的 MQTT 客户端，为其他组件提供订阅和发布功能。
  ServiceBus: ServiceBus是一个运行在边缘的HTTP客户端，接受来自云上服务的请求，与运行在边缘端的HTTP服务器交互，提供了云上服务通过HTTP协议访问边缘端HTTP服务器的能力。
  DeviceTwin: DeviceTwin 负责存储设备状态并将设备状态同步到云，它还为应用程序提供查询接口。

## KubeEdge部署
### 系统配置
- 1、集群环境
kube-cloud 云端   192.168.1.66  k8s、docker、cloudcore
kube-edge1 边缘端 192.168.1.56  k8s、edgecore
kube-edge2 边缘端 192.168.1.218 k8s、edgecore

- 2、禁用开机启动防火墙
`# systemctl disable firewalld `
- 3、永久禁用SELinux
编辑文件/etc/selinux/config，将SELINUX修改为disabled，如下：
```
# sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux
SELINUX=disable
```
- 4、关闭系统Swap（可选）
Kbernetes 1.8 开始要求关闭系统的Swap，如果不关闭，默认配置下kubelet将无法启动，如下：
```
# sed -i 's/.*swap.*/#&/' /etc/fstab
#/dev/mapper/centos-swap swap           swap   defaults     0 0
```
- 5、所有机器安装Docker
```
# yum install wget container-selinux -y
# wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm
# yum erase runc -y
# rpm -ivh containerd.io-1.2.6-3.3.el7.x86_64.rpm
注意：上面的步骤在centos7中无须操作
# update-alternatives --set iptables /usr/sbin/iptables-legacy
# yum install -y yum-utils device-mapper-persistent-data lvm2 && yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo && yum makecache
# yum -y install docker-ce
# systemctl enable docker.service && systemctl start docker

说明：如果想安装指定版本的Docker
# yum -y install docker-ce-18.06.3.ce
```

- 6、重启系统
`# reboot`

### cloud节点部署K8s

- 1、配置yum源
就算使用官方建议的多副本 + 联邦，仍然会遇到一些问题:
```
[root@ke-cloud ~]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

- 2、安装kubeadm、kubectl
```
  [root@ke-cloud ~]# yum makecache
  [root@ke-cloud ~]# yum install -y kubelet kubeadm kubectl ipvsadm
```
说明：如果想安装指定版本的kubeadm
`[root@ke-cloud ~]# yum install kubelet-1.17.0-0.x86_64 kubeadm-1.17.0-0.x86_64 kubectl-1.17.0-0.x86_64`

- 3、配置内核参数
```
[root@ke-cloud ~]# cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
EOF

[root@ke-cloud ~]# sysctl --system
[root@ke-cloud ~]# modprobe br_netfilter
[root@ke-cloud ~]# sysctl -p /etc/sysctl.d/k8s.conf

加载ipvs相关内核模块
如果重新开机，需要重新加载（可以写在 /etc/rc.local 中开机自动加载）
[root@ke-cloud ~]# modprobe ip_vs
[root@ke-cloud ~]# modprobe ip_vs_rr
[root@ke-cloud ~]# modprobe ip_vs_wrr
[root@ke-cloud ~]# modprobe ip_vs_sh
[root@ke-cloud ~]# modprobe nf_conntrack_ipv4

查看是否加载成功
[root@ke-cloud ~]# lsmod | grep ip_vs
```



- 4、拉取镜像

用命令查看版本当前kubeadm对应的k8s组件镜像版本，如下：
```
[root@ke-cloud ~]# kubeadm config images list
I0716 20:10:22.666500    6001 version.go:251] remote version is much newer: v1.18.6; falling back to: stable-1.17
W0716 20:10:23.059486    6001 validation.go:28] Cannot validate kubelet config - no validator is available
W0716 20:10:23.059501    6001 validation.go:28] Cannot validate kube-proxy config - no validator is available
k8s.gcr.io/kube-apiserver:v1.17.9
k8s.gcr.io/kube-controller-manager:v1.17.9
k8s.gcr.io/kube-scheduler:v1.17.9
k8s.gcr.io/kube-proxy:v1.17.9
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.5
```

使用kubeadm config images pull命令拉取上述镜像，如下：
```
[root@ke-cloud ~]# kubeadm config images pull
I0716 20:11:12.188139    6015 version.go:251] remote version is much newer: v1.18.6; falling back to: stable-1.17
W0716 20:11:12.580861    6015 validation.go:28] Cannot validate kube-proxy config - no validator is available
W0716 20:11:12.580877    6015 validation.go:28] Cannot validate kubelet config - no validator is available
[config/images] Pulled k8s.gcr.io/kube-apiserver:v1.17.9
[config/images] Pulled k8s.gcr.io/kube-controller-manager:v1.17.9
[config/images] Pulled k8s.gcr.io/kube-scheduler:v1.17.9
[config/images] Pulled k8s.gcr.io/kube-proxy:v1.17.9
[config/images] Pulled k8s.gcr.io/pause:3.1
[config/images] Pulled k8s.gcr.io/etcd:3.4.3-0
[config/images] Pulled k8s.gcr.io/coredns:1.6.5
```

查看下载下来的镜像，如下：
```
[root@ke-cloud ~]# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.17.9             ddc09a4c2193        19 hours ago        117MB
k8s.gcr.io/kube-controller-manager   v1.17.9             c7f1dde319ee        19 hours ago        161MB
k8s.gcr.io/kube-apiserver            v1.17.9             7417868987f3        19 hours ago        171MB
k8s.gcr.io/kube-scheduler            v1.17.9             f7b1228fa995        19 hours ago        94.4MB
k8s.gcr.io/coredns                   1.6.5               70f311871ae1        8 months ago        41.6MB
k8s.gcr.io/etcd                      3.4.3-0             303ce5db0e90        8 months ago        288MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        2 years ago         742kB
```

- 5、配置Kubelet(可选)

在cloud端配置kubelet并非必须，主要是为了验证K8s集群的部署是否正确，也可以在云端搭建Dashboard等应用。
1）获取Docker的cgroups
[root@ke-cloud ~]# DOCKER_CGROUPS=$(docker info | grep 'Cgroup' | cut -d' ' -f4)
[root@ke-cloud ~]# echo $DOCKER_CGROUPS
cgroupfs

2）配置kubelet的cgroups
```
[root@ke-cloud ~]# cat >/etc/sysconfig/kubelet<<EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=$DOCKER_CGROUPS --pod-infra-container-image=k8s.gcr.io/pause:3.1"
EOF
```
3）启动kubelet
```
[root@ke-cloud ~]# systemctl daemon-reload
[root@ke-cloud ~]# systemctl enable kubelet && systemctl start kubelet
特别说明：在这里使用systemctl status kubelet会发现报错误信息，这个错误在运行kubeadm init 生成CA证书后会被自动解决，此处可先忽略。
```
- 6、初始化集群
使用kubeadm init命令进行集群初始化，初始化完成必须要记录下初始化过程最后的命令，如下图所示：
```
[root@ke-cloud ~]# kubeadm init --kubernetes-version=v1.17.9 \
                      --pod-network-cidr=10.244.0.0/16 \
                      --apiserver-advertise-address=192.168.1.66 \
                      --ignore-preflight-errors=Swap

[init] Using Kubernetes version: v1.17.9

...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.66:6443 --token mskyxo.1gvtjenik78cm1ip \
    --discovery-token-ca-cert-hash sha256:89fd70b84d6beaff3c0223f288a675080034d108b5673cf66502267291078a04
```

进一步配置kubectl
```
[root@ke-cloud ~]# rm -rf $HOME/.kube
[root@ke-cloud ~]# mkdir -p $HOME/.kube
[root@ke-cloud ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@ke-cloud ~]# chown $(id -u):$(id -g) $HOME/.kube/config
```

查看node节点
```
[root@ke-cloud ~]# systemctl status kubelet
NAME       STATUS     ROLES    AGE    VERSION
ke-cloud   NotReady   master   3m3s   v1.17.0
```
- 7、配置网络插件（可选）
特别说明： 版本会经常更新，如果配置成功，就手动去https://raw.githubusercontent.com/coreos/flannel/master/Documentation/ 下载最新版yaml文件
1）下载flannel插件的yaml文件
`[root@ke-cloud ~]# cd ~ && mkdir flannel && cd flannel`
`[root@ke-cloud ~]# curl -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`
2）启动
`[root@ke-cloud ~]# kubectl apply -f ~/flannel/kube-flannel.yml`
3）查看
```
[root@ke-cloud ~]# kubectl get node
NAME       STATUS   ROLES    AGE   VERSION
ke-cloud   Ready    master   12h   v1.17.0
[root@ke-cloud ~]# kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-6955765f44-k65xx           1/1     Running   0          12h
coredns-6955765f44-r2q6g           1/1     Running   0          12h
etcd-ke-cloud                      1/1     Running   0          12h
kube-apiserver-ke-cloud            1/1     Running   0          12h
kube-controller-manager-ke-cloud   1/1     Running   0          12h
kube-flannel-ds-amd64-lrsrh        1/1     Running   0          12h
kube-proxy-vr44d                   1/1     Running   0          12h
kube-scheduler-ke-cloud            1/1     Running   0          12h

[root@ke-cloud ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   12h
注意： 只有网络插件也安装配置完成之后，node才能会显示为ready状态。
```

- 8、K8s安装dashboard（可选）

### KubeEdge的安装与配置

- 1、Cloud端配置
cloud端负责编译KubeEdge的相关组件与运行cloudcore。

- 准备工作
1）下载golang
```
[root@ke-cloud ~]# wget https://golang.google.cn/dl/go1.14.4.linux-amd64.tar.gz
[root@ke-cloud ~]# tar -zxvf go1.14.4.linux-amd64.tar.gz -C /usr/local
```
​		2）配置golang环境
`[root@ke-cloud ~]# vim /etc/profile`
文件末尾添加：

```
# golang env
export GOROOT=/usr/local/go
export GOPATH=/data/gopath
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```
```
[root@ke-cloud ~]# source /etc/profile
[root@ke-cloud ~]# mkdir -p /data/gopath && cd /data/gopath
[root@ke-cloud ~]# mkdir -p src pkg bin
```
​	3）下载KubeEdge源码
`[root@ke-cloud ~]# git clone https://github.com/kubeedge/kubeedge $GOPATH/src/github.com/kubeedge/kubeedge`

- 部署cloudcore
可以通过本地部署的方式部署KubeEdge（这需要单独编译cloudcore和edgecore，部署方式参考： https://docs.kubeedge.io/en/latest/setup/local.html ） ，也可以通过keadm的方式部署KubeEdge（部署方式参考： https://docs.kubeedge.io/en/latest/setup/keadm.html ） 。

本文选用了在操作上更简单的keadm部署方式。
1）编译kubeadm

```
[root@ke-cloud ~]# cd $GOPATH/src/github.com/kubeedge/kubeedge
[root@ke-cloud ~]# make all WHAT=keadm
```
说明：编译后的二进制文件在./_output/local/bin下，单独编译cloudcore与edgecore的方式如下：
`[root@ke-cloud ~]# make all WHAT=cloudcore && make all WHAT=edgecore`
2）创建cloud节点

```
[root@ke-cloud ~]# keadm init --advertise-address="192.168.1.66"

Kubernetes version verification passed, KubeEdge installation will start...

...

KubeEdge cloudcore is running, For logs visit:  /var/log/kubeedge/cloudcore.log
CloudCore started
```
- 2、edge端配置
edge端也可以通过keadm进行配置，可以将cloud端编译生成的二进制可执行文件通过scp命令复制到edge端。

- 从云端获取令牌
在云端运行将返回令牌，该令牌将在加入边缘节点时使用。keadm gettoken
`[root@ke-cloud ~]# keadm gettoken
8ca5a29595498fbc0648ca59208681f9d18dae86ecff10e70991cde96a6f4199.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1OTUwMzU0Mjh9.YR-N628S5wEFLifC0sM9t-IuIWkgSK-kizFnyAy5Q50`
- 加入边缘节点
keadm join将安装edgecore和mqtt。它还提供了一个标志，通过它可以设置特定的版本。
```
[root@ke-edge1 ~]# keadm join --cloudcore-ipport=192.168.1.66:10000 `--token=8ca5a29595498fbc0648ca59208681f9d18dae86ecff10e70991cde96a6f4199.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1OTUwMzU0Mjh9.YR-N628S5wEFLifC0sM9t-IuIWkgSK-kizFnyAy5Q50

Host has mosquit+ already installed and running. Hence skipping the installation steps !!!

...

KubeEdge edgecore is running, For logs visit:  /var/log/kubeedge/edgecore.log

重要的提示：
• --cloudcore-ipport 标志是强制性标志。
• 如果要自动为边缘节点应用证书，--token则需要。
• 云和边缘端使用的kubeEdge版本应相同。
```
- 验证

边缘端在启动edgecore后，会与云端的cloudcore进行通信，K8s进而会将边缘端作为一个node纳入K8s的管控。
```
[root@ke-cloud ~]# kubectl get node
NAME       STATUS   ROLES        AGE   VERSION
ke-cloud   Ready    master       13h   v1.17.0
ke-edge1   Ready    agent,edge   64s   v1.17.1-kubeedge-v1.3.1

[root@ke-cloud ~]# kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-6955765f44-k65xx           1/1     Running   0          13h
coredns-6955765f44-r2q6g           1/1     Running   0          13h
etcd-ke-cloud                      1/1     Running   0          13h
kube-apiserver-ke-cloud            1/1     Running   0          13h
kube-controller-manager-ke-cloud   1/1     Running   0          13h
kube-flannel-ds-amd64-fgtdq        0/1     Error     5          5m55s
kube-flannel-ds-amd64-lrsrh        1/1     Running   0          13h
kube-proxy-vr44d                   1/1     Running   0          13h
kube-scheduler-ke-cloud            1/1     Running   0          13h

说明：如果在K8s集群中配置过flannel网络插件（见2.7），这里由于edge节点没有部署kubelet，
所以调度到edge节点上的flannel pod会创建失败。这不影响KubeEdge的使用，可以先忽略这个问题。
```

### 运行KubeEdge示例


选用示例：KubeEdge Counter Demo计数器是一个伪设备，用户无需任何额外的物理设备即可运行此演示。计数器在边缘侧运行，用户可以从云侧在Web中对其进行控制，也可以从云侧在Web中获得计数器值。
原理图如下：

详细文档参考： https://github.com/kubeedge/examples/tree/master/kubeedge-counter-demo

- 1、准备工作
1）本示例要求KubeEdge版本必须是v1.2.1+
```
[root@ke-cloud ~]# kubectl get node
NAME       STATUS   ROLES        AGE   VERSION
ke-cloud   Ready    master       13h   v1.17.0
ke-edge1   Ready    agent,edge   64s   v1.17.1-kubeedge-v1.3.1

说明：本文接下来的验证将使用边缘节点ke-edge1进行，如果你参考本文进行相关验证，后续边缘节点名称的配置需要根据你的实际情况进行更改。
```
2）下载示例代码：
`[root@ke-cloud ~]# git clone https://github.com/kubeedge/examples.git $GOPATH/src/github.com/kubeedge/examples`

- 2、创建device model和device

1）创建device model
```
[root@ke-cloud ~]# cd $GOPATH/src/github.com/kubeedge/examples/kubeedge-counter-demo/crds
[root@ke-cloud crds~]# kubectl create -f kubeedge-counter-model.yaml
```
2）创建device
根据你的实际情况修改matchExpressions：
```
[root@ke-cloud ~]# cd $GOPATH/src/github.com/kubeedge/examples/kubeedge-counter-demo/crds
[root@ke-cloud crds~]# vim kubeedge-counter-instance.yaml
apiVersion: devices.kubeedge.io/v1alpha1
kind: Device
metadata:
  name: counter
  labels:
    description: 'counter'
    manufacturer: 'test'
spec:
  deviceModelRef:
    name: counter-model
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
      - key: 'kubernetes.io/hostname'
        operator: In
        values:
        - ke-edge1

status:
  twins:
    - propertyName: status
      desired:
        metadata:
          type: string
        value: 'OFF'
      reported:
        metadata:
          type: string
        value: '0'

[root@ke-cloud crds~]# kubectl create -f kubeedge-counter-instance.yaml
```
- 3、部署云端应用

1）修改代码

云端应用web-controller-app用来控制边缘端的pi-counter-app应用，该程序默认监听的端口号为80，此处修改为8089，如下所示：
```
[root@ke-cloud ~]# cd $GOPATH/src/github.com/kubeedge/examples/kubeedge-counter-demo/web-controller-app
[root@ke-cloud web-controller-app~]# vim main.go
package main

import (
        "github.com/astaxie/beego"
        "github.com/kubeedge/examples/kubeedge-counter-demo/web-controller-app/controller"
)

func main() {
        beego.Router("/", new(controllers.TrackController), "get:Index")
        beego.Router("/track/control/:trackId", new(controllers.TrackController), "get,post:ControlTrack")

        beego.Run(":8089")
}
```
2）构建镜像
```
注意：构建镜像时，请将源码拷贝到GOPATH对应的路径下，如果开启了go mod请关闭。
[root@ke-cloud web-controller-app~]# make all
[root@ke-cloud web-controller-app~]# make docker
```
3）部署web-controller-app
```
[root@ke-cloud ~]# cd $GOPATH/src/github.com/kubeedge/examples/kubeedge-counter-demo/crds
[root@ke-cloud crds~]# kubectl apply -f kubeedge-web-controller-app.yaml
```
- 4、部署边缘端应用

边缘端的pi-counter-app应用受云端应用控制，主要与mqtt服务器通信，进行简单的计数功能。

1）修改代码与构建镜像
需要将Makefile中的GOARCH修改为amd64才能运行该容器。
```
[root@ke-cloud ~]# cd $GOPATH/src/github.com/kubeedge/examples/kubeedge-counter-demo/counter-mapper
[root@ke-cloud counter-mapper~]# vim Makefile
.PHONY: all pi-execute-app docker clean
all: pi-execute-app

pi-execute-app:
        GOARCH=amd64 go build -o pi-counter-app main.go

docker:
        docker build . -t kubeedge/kubeedge-pi-counter:v1.0.0

clean:
        rm -f pi-counter-app

[root@ke-cloud counter-mapper~]# make all
[root@ke-cloud counter-mapper~]# make docker
```
2）部署Pi Counter App
```
[root@ke-cloud ~]# cd $GOPATH/src/github.com/kubeedge/examples/kubeedge-counter-demo/crds
[root@ke-cloud crds~]# kubectl apply -f kubeedge-pi-counter-app.yaml

说明：为了防止Pod的部署卡在`ContainerCreating`，这里直接通过docker save、scp和docker load命令将镜像发布到边缘端
[root@ke-cloud ~]# docker save -o kubeedge-pi-counter.tar kubeedge/kubeedge-pi-counter:v1.0.0
[root@ke-cloud ~]# scp kubeedge-pi-counter.tar root@192.168.1.56:/root
[root@ke-edge1 ~]# docker load -i kubeedge-pi-counter.tar
```
 - 5、运行效果

现在，KubeEdge Demo的云端部分和边缘端的部分都已经部署完毕，如下：
```[root@ke-cloud ~]# kubectl get pods -o wide
NAME                                    READY   STATUS    RESTARTS   AGE     IP             NODE       NOMINATED NODE   READINESS GATES
kubeedge-counter-app-758b9b4ffd-f8qjj   1/1     Running   0          26m     192.168.1.66   ke-cloud   <none>           <none>
kubeedge-pi-counter-c69698d6-rb4xz      1/1     Running   0          2m      192.168.1.56   ke-edge1   <none>           <none>
```
我们现在开始测试一下该Demo运行效果：
1）执行ON命令
在web页面上选择ON，并点击Execute，可以在edge节点上通过以下命令查看执行结果：
`[root@ke-edge1 ~]# docker logs -f counter-container-id`

2）查看counter STATUS
在web页面上选择STATUS，并点击Execute，会在Web页面上返回counter当前的status，如下所示：

2）执行OFF命令
在web页面上选择OFF，并点击Execute，可以再edge节点上通过以下命令查看执行结果：
`[root@ke-edge1 ~]# docker logs -f counter-container-id`


### 总结
文本采用的KubeEdge安装和部署流程是非常简易的，具体体现在：
首先在云端使用了kubeadm轻松搭建了一个K8s集群。
然后使用了非常方便、易用的keadm在云端和边缘端搭建了KubeEdge的核心组件：cloudcore和edgecore。部署成功后，KubeEdge边缘节点会作为一个node被K8s管控。
最后用了KubeEdge官方提供的examples（kubeedge-counter-demo）运行了一个示例，这一示例完美地展现了kubeEdge的边缘设备管理、边云协同能力。



cp $GOPATH/src/github.com/kubeedge/kubeedge/build/crds/devices/* $GOPATH/yamls

cp $GOPATH/src/github.com/kubeedge/kubeedge/build/node.json $GOPATH/cloud





ssh 192.168.117.145 "mkdir -p /etc/kubeedge /home/kkbill/kubeedge/edge"

scp -r $GOPATH/certs/* 192.168.117.145:/etc/kubeedge



scp -r $GOPATH/edge/* 192.168.117.145:/home/kkbill/kubeedge/edge













## 1. 概述

**1.1 环境**

- 云端：centos7，用户名为k8smaster。ip 为 192.168.179.131 
- 边缘端：Ubuntu 18.04.3 LTS，用户名为k8slave1。ip 为 192.168.179.142
- KubeEdge 部署需要的组件：（注意，**本安装教程假定你已经在云端机器上安装好了k8s集群和docker，并在边缘端机器上安装好了docker**）
- 云端：docker， kubernetes 集群和 KubeEdge 云端核心模块。
- 边缘端：docker， mqtt 和 KubeEdge 边缘端核心模块。

**1.2 依赖**

- golang：版本1.12.14，移步 https://studygolang.com/dl 下载（由于是源码编译，所以需要用到golang）
- k8s版本：v1.16.0
- mosquitto：直接通过apt-get安装
- KubeEdge：v1.1.0



## 2. 准备



### **2.1 创建部署文件目录**

本安装方法主要适用于初学者，因为初学者对于搭建集群往往是比较陌生的。本文会先创建一个目录，存放各类安装过程中需要的文件。

创建部署工程目录：

```
# mkdir /home/kkbill/kubeedge
```

创建子目录： 

```
# cd /home/kkbill/kubeedge

# mkdir cloud edge certs yamls src
```

- 说明：

  - cloud：云端相关文件，包括 cloudcore 和配置文件。
  - edge：边缘端相关文件，包括 edgecore 和配置文件。
  - certs：证书文件。
  - yamls：一些 yamls 文件。
  - src：源码目录，存放kubeedge源码。

- 

  ### **2.2 KubeEdge 二进制**

获取KubeEdge的方式有两种，一种是直接从 官网(https://github.com/kubeedge/kubeedge/releases) 中下载（本实验版本为kubeedge-v1.1.0-linux-amd64.tar.gz）；另一种方法是通过源码编译得到。这里介绍一下源码编译的方法（嗯，折腾...）

**2.2.1 Golang环境搭建**

(1) 下载golang，并解压

```
# wget https://studygolang.com/dl/golang/go1.12.14.linux-amd64.tar.gz

# tar -C /usr/local -xzf  go1.12.14.linux-amd64.tar.gz
```

(2) 添加环境变量

在~/.bashrc文件末尾添加：

```
# vim ~/.bashrc

export GOPATH=/home/kkbill/kubeedge
export PATH=$PATH:/usr/local/go/bin
```

保存后记得执行 source ~/.bashrc 生效。验证：

```
# go version
# go version go1.12.14 linux/amd64
```

**2.2.2 下载kubeedge源码**

```
# git clone https://github.com/kubeedge/kubeedge.git $GOPATH/src/github.com/kubeedge/kubeedge
```

**2.2.3 检测gcc是否安装**

```
# gcc --version
```

如果没有，则自行安装。

**2.2.4 编译云端**

```
# cd $GOPATH/src/github.com/kubeedge/kubeedge/

# make all WHAT=cloudcore
```

生成二进制 cloudcore 文件位于 cloud 目录。拷贝 cloudcore 和同一目录的配置文件（conf目录）到部署工程目录：

```
# cp -a cloud/cloudcore $GOPATH/cloud/
# cp -a cloud/conf/ $GOPATH/cloud/
```

在编译的时候遇到了第一个坑，就是版本的问题。由于最新clone下来的版本已经不是v1.1.0了，所以，我们需要把代码切回到v1.1.0版本，操作如下：（如果你在这一步无异常，则跳过）

在 $GOPATH/src/github.com/kubeedge/kubeedge 目录下，执行 git tag，并选择 v1.1.0 即可

```
# git tag
...
v1.1.0
...

# git checkout v1.1.0 
```

执行完这一句后，代码就会回到v1.1.0版。

**2.2.5 编译边缘端**

```
# cd $GOPATH/src/github.com/kubeedge/kubeedge/

# make all WHAT=edgecore
```

生成二进制 edgecore 文件位于 edge 目录。拷贝二进制及配置文件到部署工程目录：

```
# cp -a edge/edgecore $GOPATH/edge/

# cp -a edge/conf/ $GOPATH/edge/
```

这里又遇到了第2个坑，即出现如下错误：

```
/usr/local/go/pkg/tool/linux_amd64/link: signal: killed
```

这是由于编译需要较大的内存，而内存不够，造成了OOM。对应的解决办法：增加内存

(1) 创建要作为swap分区的文件:增加1GB大小的交换分区，则命令写法如下，其中的count等于想要的块的数量（bs*count=文件大小）。

```
# dd if=/dev/zero of=/root/swapfile bs=1M count=1024
```

(2) 格式化为交换分区文件:

```
mkswap /root/swapfile #建立swap的文件系统
```

(3) 启用交换分区文件:

```
swapon /root/swapfile #启用swap文件
```

解决这个问题，参考了：

```
https://forum.golangbridge.org/t/go-build-exits-with-signal-killed/513

https://segmentfault.com/a/1190000012219689

https://www.cnblogs.com/spjy/p/7085389.html
```



### **2.3 生成证书**

```
# $GOPATH/src/github.com/kubeedge/kubeedge/build/tools/certgen.sh genCertAndKey edge

# cp -a /etc/kubeedge/* $GOPATH/certs
```

生成的 ca 和 certs 分别位于 /etc/kubeedge/ca 和 /etc/kubeedge/certs 目录，将其拷贝到部署工程目录的 certs 目录。注意，这是在云端机器执行，所以云端已经有了证书，拷贝到 certs 目录，是为了方便分发到边缘节点。



### **2.4 拷贝设备模块和设备CRD yaml 文件**

```
# cp 𝐺𝑂𝑃𝐴𝑇𝐻/𝑠𝑟𝑐/𝑔𝑖𝑡ℎ𝑢𝑏.𝑐𝑜𝑚/𝑘𝑢𝑏𝑒𝑒𝑑𝑔𝑒/𝑘𝑢𝑏𝑒𝑒𝑑𝑔𝑒/𝑏𝑢𝑖𝑙𝑑/𝑐𝑟𝑑𝑠/𝑑𝑒𝑣𝑖𝑐𝑒𝑠/∗

GOPATH/yamls
```



### **2.5 拷贝node.json**

```
# cp 𝐺𝑂𝑃𝐴𝑇𝐻/𝑠𝑟𝑐/𝑔𝑖𝑡ℎ𝑢𝑏.𝑐𝑜𝑚/𝑘𝑢𝑏𝑒𝑒𝑑𝑔𝑒/𝑘𝑢𝑏𝑒𝑒𝑑𝑔𝑒/𝑏𝑢𝑖𝑙𝑑/𝑛𝑜𝑑𝑒.𝑗𝑠𝑜𝑛

GOPATH/cloud 
```

释义：node.json 为节点的配置信息，需要在云端机器执行，作用是将边缘端加入集群（但实际上只是让 k8s 知道有这个节点，还不是真正意义上的加入）



### **2.6 配置云端节点**

打开配置文件 $GOPATH/cloud/conf/controller.yaml ，修改两处 master 的值，将master修改为 k8s 的apiserver的地址，在我的配置中，修改为：https://192.168.179.131:6443

这个地址当然就是云端这台机器的ip，那么端口是怎么来的？你可以从 /etc/kubernetes/manifests/kube-apiserver.yaml 中找到答案，如下图所示：

（当然，这里的前提是你已经装好了kubernetes，但是安装 kubernetes 不在本教程范围。）

 ![img](https://img2020.cnblogs.com/blog/1442950/202003/1442950-20200330213223439-1561103334.png)

 这里提醒一下，我在第一次安装的时候，由于自身对k8s也还非常的不了解，还不知道是查看kube-apiserver.yaml，所以错写成了http，这个错误让我废了很长的时间才解决。

另外，还需要将kubeconfig的值修改为：/root/.kube/config

![img](https://img2020.cnblogs.com/blog/1442950/202003/1442950-20200330214208350-1616304629.png)

 这一步也存在一个坑，如果设置不正确，可能会导致 cloudcore 启动失败，出现如下错误：

E0102 17:53:58.630318 15021 reflector.go:125] github.com/kubeedge/kubeedge/cloud/pkg/devicecontroller/manager/device.go:40: Failed to list *v1alpha1.Device: the server rejected our request for an unknown reason (get devices.devices.kubeedge.io)
E0102 17:53:58.630335 15021 reflector.go:125] github.com/kubeedge/kubeedge/cloud/pkg/devicecontroller/manager/devicemodel.go:40: Failed to list *v1alpha1.DeviceModel: the server rejected our request for an unknown reason (get devicemodels.devices.kubeedge.io)
W0102 17:53:58.632435 15021 controller.go:51] new downstream controller failed with error: the server rejected our request for an unknown reason (get nodes)

这一问题在 https://github.com/kubeedge/kubeedge/issues/1361 有讨论，我也是从这个issue中摸索出了自己的解决方案。



### **2.7 配置边缘节点**

打开配置文件$GOPATH/edge/conf/edge.yaml ，将如下两处ip换成你自己的云端主机的ip。

![img](https://img2020.cnblogs.com/blog/1442950/202003/1442950-20200330215031057-2107395820.png)

 另外，由于在v1.1.0版本中，边缘节点名称为 `fb4ebb70-2783-42b8-b3ef-63e2fd6d242e，这不方便阅读，我们将其修改为edge-node。在edge.yaml文件中把所有 fb4ebb70-2783-42b8-b3ef-63e2fd6d242e 的值替换为 edge-node。`

此外，还有非常重要的一点是，注意该文件中 cgroup-driver 字段的值，默认情况下是 cgroupfs，有些文章不说明具体原因就让你将其修改为 sysemd，这是不负责的。**这里是否需要修改该字段的值取决于你安装的docker的cgroup-driver是什么，原则就是要保持两者一直**，若不一致，就会出现致命的问题。

可以通过 docker info 命令查看已安装的docker的cgroup-driver，确定好后，再决定是否修改edge.yaml 文件中cgroup-driver的值。

这里我也踩过坑，第一次安装时未经检查，就盲目修改了该字段的值，导致最后边缘节点的状态始终是NotReady，这个问题在issue列表中也有讨论，详见 https://github.com/kubeedge/kubeedge/issues/1243 。



### **2.8 安装mqtt**

mqtt只需要在边缘端安装。由于我使用的是ubuntu系统，直接使用apt-get，如下：

\# add-apt-repository ppa:mosquitto-dev/mosquitto-ppa // 添加源
\# apt-get update // 更新
\# apt-get install mosquitto // 安装



## **3. 部署**

前面已经准备好了所需要的配置文件，现在，只需要将证书和边缘端的文件拷贝到边缘机器上，使用 scp 命令进行拷贝。（当然，你也可以在边缘机器上完成上面的2.7节。）

在云端主机上进行操作，192.168.179.142是边缘端的ip。

```
# ssh 192.168.179.142 "mkdir -p /etc/kubeedge /home/kkbill/kubeedge/edge"
# scp -r GOPATH/certs/* 192.168.179.142:/etc/kubeedge # scp -r

GOPATH/edge/* 192.168.179.142:/home/kkbill/kubeedge/edge
```

这里可能会涉及到一些permission deny的问题，这是由于 scp 命令造成的，这个问题不难，自己解决一下。



### **3.1 云端**

首先进入部署工程目录：cd /home/kkbill/kubeedge

**3.1.1 添加边缘端到集群**

```
# kubectl apply -f cloud/node.json
```

**注意**：在执行该命令前，务必修改node.json文件的值，如下，把name字段的值由 fb4ebb70-2783-42b8-b3ef-63e2fd6d242e 的值替换为 edge-node。

![img](https://img2020.cnblogs.com/blog/1442950/202003/1442950-20200330224140558-2135133133.png)

 此时，查看节点状态，边缘端未就绪。如下：

```
# kubectl get nodes 

NAME        STATUS      ROLES   AGE     VERSION
edge-node NotReady    edge       9s
k8smaster  Ready          master    15h       v1.16.0
```

**3.1.2 创建设备模块和设备CRD**

```
# kubectl apply -f yamls
```

 注意，yamls 目录有2个 yaml 文件，此处指定目录，会自动执行目录所有文件。

**3.1.3 运行cloudcore**

```
# cd  cloud

# ./cloudcore
```

可能会出现如下错误：

```
github.com/kubeedge/kubeedge/cloud/pkg/edgecontroller/manager/secret.go:31: Failed to list *v1.Secret: Get https://192.168.179.131:6443/api/v1/secrets?limit=500&resourceVersion=0: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")
```

参考这里的讨论 https://github.com/kubernetes/kubernetes/issues/48378 ，我忘了当时是怎么解决这个问题的了...



### **3.2 边缘端**

该部分在边缘端机器上进行。

首先，在边缘端运行mqtt：

```
# mosquitto -d -p 1883
```

然后进入部署工程目录：cd /home/kkbill/kubeedge/edge，并运行 edgecore：

```
# ./edgecore
```



### **3.3 验证**

在云端查看状态：

```
# kubectl get nodes
NAME        STATUS     ROLES    AGE    VERSION
edge-node   Ready         edge        8h       v1.15.3-kubeedge-v1.1.0
k8smaster    Ready        master     173d   v1.16.0
```

这个时候，edge-node 的状态就变成了 Ready。到这里，基本上算是搭建完成了。