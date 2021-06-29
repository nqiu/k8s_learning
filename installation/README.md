# K8s安装教程

K8S（Kubernetes）在国外安装很简单，因为kubernetes依赖的镜像从gcr.io（Google Container Registry）获取，该网址在国内被墙了，所以需要其它办法获取镜像。
本文通过搭建一个一主两从的k8s集群，介绍国内搭建k8s的详细过程。

## 环境

| 角色 | 机器名 | 机器IP | OS |
|---|---|---|---|
|  Master | master.io | 172.29.20.111 | CentOS 7 |
|  Worker Node | worker1.io | 172.29.20.112 | CentOS 7 |
|  Worker Node | worker2.io | 172.29.20.113 | CentOS 7 |

## 安装

我们使用官方kubeadm工具安装

### 安装docker

为了在Pod中运行容器，需要在每台机器上安装容器运行时，本文使用docker作为容器运行时。
在每台机器上安装docker，设置开机启动，非root用户设置等，详细安装过程参照[官方文档](https://docs.docker.com/get-started/)

### 安装kubeadm, kubelet, kubectl

需要在每台机器上安装以下安装包：

* `kubeadm`: 用来初始化集群的命令
* `kubelet`: 在集群中的每个节点上用来启动Pod和容器等
* `kubectl`: 用来与集群通信的命令行工具

`kubelet`和`kubectl`的版本需要与通过`kubeadm`安装的控制平台的版本相匹配，否则存在版本偏差的风险，控制平面与kubelet间的相差一个次要版本不一致是支持的，`kubelet`版本不可以高于API服务器的版本。

国内可以选择使用中科大的源或阿里的源。
<!-- 
#### 基于Debian的发行版

1. 添加中科大的源

    ```bash
    cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
    deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
    EOF
    ```

2. 更新apt包索引，安装kubelet, kubeadm, kubectl

    ```bash
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
    sudo apt-get install -y kubeadm kubelet kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```
-->

#### 基于Red Hat的发行版

1. 添加阿里的源

    ```bash
    cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-$basearch
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
    EOF
    ```

2. 更新索引，安装kubelet, kubeadm, kubectl

    ```bash
    sudo yum makecache
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    sudo yum install -y kubeadm kubectl kubelet
    ```

<!-- 
#### 无包管理器的情况

(todo)
-->

### 获取镜像列表

由于官方镜像被墙，所以需要先获取依赖的镜像列表，然后从国内源下载，在Master机器上，运行以下命令

```bash
# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.21.2
k8s.gcr.io/kube-controller-manager:v1.21.2
k8s.gcr.io/kube-scheduler:v1.21.2
k8s.gcr.io/kube-proxy:v1.21.2
k8s.gcr.io/pause:3.4.1
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns/coredns:v1.8.0
```

然后通过以下脚本从阿里云获取：

```bash
images=(  # 下面的镜像应该去除"k8s.gcr.io/"的前缀，版本换成上面获取到的版本
    kube-apiserver:v1.12.2
    kube-controller-manager:v1.12.2
    kube-scheduler:v1.12.2
    kube-proxy:v1.12.2
    pause:3.4.1
    etcd:3.4.13-0
    coredns:v1.8.0
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```

！！！kubeadm拉镜像时可以指定registry，在国内可以用以下命令提前下载所需镜像

```bash
kubeadm config images pull --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```

> **注**：Worker Node需要依赖部分镜像，简单起见，可以在每台机器都安装所有镜像

### 初始化集群

在Master机器上，使用kubeadm初始化集群，可以通过--kubernetes-version选项指定k8s版本，详细选项请阅读kubeadm说明

```bash
# kubeadm init --kubernetes-version="v1.21.2" --pod-network-cidr="10.244.0.0/16"

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.29.75.235:6443 --token r760l1.8v1dhgdqil258wco \
        --discovery-token-ca-cert-hash sha256:57e2cafeb8d0fd04a098524fc09c301400cfe28290327356b7ec62de75769966
```

### 安装网络插件

本文安装flannel网络插件，flannel[官文文档](https://github.com/flannel-io/flannel)，其它插件的安装可参考插件的官方文档。

```bash
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 集群加入Worker Node

根据Master机器上初始化集群命令运行完成后的打印信息，在Worker Node机器上运行命令加入集群

```bash
# kubeadm join 172.29.75.235:6443 --token r760l1.8v1dhgdqil258wco \
        --discovery-token-ca-cert-hash sha256:57e2cafeb8d0fd04a098524fc09c301400cfe28290327356b7ec62de75769966
```

### 其它配置

#### 设置master节点可以运行pod

在Master机器上运行以下命令，允许Master运行Pod，分担工作负载

```bash
# kubectl taint nodes --all node-role.kubernetes.io/master-
```

### 常见问题

#### 查看集群状态

可以通过以下命令查看集群节点信息

```bash
# kubectl get nodes -o=wide
```

可以通过以下命令查看集群pod信息

```bash
# kubectl get pods -n kube-system
```

#### 定位问题

可以通过以下命令查看node详细信息，包含运行异常信息

```bash
# kubectl describe node <node_name> -n kube-system
```

可以通过以下命令查看pod详细信息，包含运行异常信息

```bash
# kubectl describe pod <pod_name> -n kube-system
```

### References

* [安装kubeadm](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
