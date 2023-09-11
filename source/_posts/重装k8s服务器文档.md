---
title: openEuler安装K8S文档
category: K8S
tag: 
 - K8S
 - openEuler
 - 教程
 - arm64
---

# 重装k8s服务器文档

## 任务背景：项目计划在两台服务器开启k8s集群，来运行虚拟机和pod，目前其中一台还没有安装k8s，因此任务是在这台openeuler20.04lts服务器上安装k8s集群。

## 依照之前安装k8s所写文档（k8s-centos）进行安装k8s

### 关闭防火墙 或者 开启防火墙端口

`sudo systemctl stop firewalld.service
sudo systemctl disable firewalld.service`

```shell
# master
firewall-cmd --zone=public --add-port=6443/tcp --permanent # Kubernetes API server
所有
firewall-cmd --zone=public --add-port=2379/tcp --permanent # etcd server client
API kube-apiserver, etcd
firewall-cmd --zone=public --add-port=2380/tcp --permanent # etcd server client
API kube-apiserver, etcd
firewall-cmd --zone=public --add-port=10250/tcp --permanent # Kubelet API 自身,
控制面
firewall-cmd --zone=public --add-port=10259/tcp --permanent # kube-scheduler 自
身
firewall-cmd --zone=public --add-port=10257/tcp --permanent # kube-controllermanager 自身
firewall-cmd --zone=trusted --add-source=192.168.111.200 --permanent # 信任集群中各
个节点的IP
firewall-cmd --zone=trusted --add-source=192.168.111.201 --permanent # 信任集群中各
个节点的IP
firewall-cmd --zone=trusted --add-source=192.168.111.202 --permanent # 信任集群中各
个节点的IP
firewall-cmd --add-masquerade --permanent # 端口转发
firewall-cmd --reload
firewall-cmd --list-all
firewall-cmd --list-all --zone=trusted
# 工作节点
firewall-cmd --zone=public --add-port=10250/tcp --permanent # Kubelet API 自身,
控制面
firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent # NodePort
Services† 所有
firewall-cmd --zone=trusted --add-source=192.168.111.200 --permanent # 信任集群中各
个节点的IP
firewall-cmd --zone=trusted --add-source=192.168.111.201 --permanent # 信任集群中各
个节点的IP
firewall-cmd --zone=trusted --add-source=192.168.111.202 --permanent # 信任集群中各
个节点的IP
firewall-cmd --add-masquerade --permanent # 端口转发
firewall-cmd --reload
firewall-cmd --list-all
firewall-cmd --list-all --zone=trusted
```

### 关闭swap和selinux

```shell
free -h //查看是否关闭
sudo swapoff -a //暂时关闭
sudo sed -i 's/.*swap.*/#&/' /etc/fstab //永久关闭
free -h
getenforce
cat /etc/selinux/config
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
cat /etc/selinux/config
```

### 安装docker  和 containerd和runc组件

docker使用dnf可以直接安装

containerd 建议从官网下载tar包或deb包安装。

runc 必须从官网下载最新版，因为dnf源中的runc版本过低，无法使用。runc的包名为libseccomp-devel

### 配置docker cgroups和源

```shell
# 对docker换源和cgroup设置
cat<<EOF | sudo tee /etc/docker/daemon.json
{
"registry-mirrors": ["https://2u8f97e8.mirror.aliyuncs.com"],
"exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
```

### 设置内核参数

```shell
# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1 //这两条让网络连接走iptables（禁用可以不设置）
net.ipv4.ip_forward = 1 //是否允许将一个网络接口收到的数据包转发到另一个网络接口
EOF
# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

### 设置k8s仓库

```shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
# 是否开启本仓库
enabled=1
# 是否检查 gpg 签名文件
gpgcheck=0
# 是否检查 gpg 签名文件
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 安装kubelet kubeadm kubectl

```shell
sudo yum install -y kubelet-1.26.2-0 kubeadm-1.26.2-0 kubectl-1.26.2-0 --
disableexcludes=kubernetes --nogpgcheck
```

版本号选择你要安装的版本号，work尽量和master保持一致（1.23.1）。

### master中执行

```shell
kubeadm init --image-repository=registry.aliyuncs.com/google_containers
# 指定集群的IP
# kubeadm init --image-repository=registry.aliyuncs.com/google_containers --apiserveradvertise-address=192.168.80.60
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
# 或者在环境变量中添加：export KUBECONFIG=/etc/kubernetes/admin.conf
# 添加完环境变量后，刷新环境变量：source /etc/profile
kubectl cluster-info
# 初始化失败后，可进行重置，重置命令：kubeadm reset -
# 执行成功后，会出现类似下列内容：
# kubeadm join 192.168.80.60:6443 --token f9lvrz.59mykzssqw6vjh32 \
# --discovery-token-ca-cert-hash
sha256:4e23156e2f71c5df52dfd2b9b198cce5db27c47707564684ea74986836900107
# 查看join命令
# kubeadm token create --print-join-command
```

### work节点上执行

```shell
# 运行的内容来自上方执行结果
kubeadm join 192.168.80.60:6443 --token f9lvrz.59mykzssqw6vjh32 \
--discovery-token-ca-cert-hash
sha256:4e23156e2f71c5df52dfd2b9b198cce5db27c47707564684ea74986836900107
```

```shell
cp /etc/kubenetes/*.config ~/.kube/config
```

之后使用kubectl get node 可以看到node的状态为not ready

### 配置work网络（网络有问题的节点都需要进行）

```shell
yum install -y wget
wget --no-check-certificate
https://projectcalico.docs.tigera.io/archive/v3.25/manifests/calico.yaml
```

```shell
# 在 - name: CLUSTER_TYPE 下方添加如下内容
- name: CLUSTER_TYPE
value: "k8s,bgp"
# 下方为新增内容
- name: IP_AUTODETECTION_METHOD
value: "interface=网卡名称"
```

```shell
# 配置网络
kubectl apply -f calico.yaml
```

之后等网络配置完成。

## 遇到的问题：

### 安装程序的时候提示根目录空间占用100%，发现是pcp（性能监控软件）的日志占用了很大空间。解决方法是直接使用rm -rf删除了这些日志/var/log/pcp/pmlogger/openEuler1/。

#### 关于pcp：

- Performance Co-Pilot (`pcp`) 提供了支持系统级性能监控和管理的框架和服务。它为系统中的所有性能数据提供了统一的抽象，以及用于询问、检索和处理该数据的许多工具。
- 这些生成的log，在openeuler系统没有设置自动清理，导致了日志积累。

### 使用kubeadm join报错runtime not running或者 no such file or dictionary

原因：现在kubelet默认runtime是containerd，一般的原因都是没有正常安装containerd造成的

解决方法：安装containerd

### join后总是显示节点notReady

原因：docker的一些容器启动错误，去docker查看问题所在

解决方法：docker start 一个容器，查看报错

### docker启动容器报错runc error

原因：runc错误

解决方法：卸载本地的 libseccomp-devel包，重装最新版本（runc包含在这个包中，yum仓库版本不够新）

### kubectl get 命令报错 The connection to the server localhost:8080 was refused – did you specify the right host or port?

原因：kubectl 无法获取到集群信息，一般都是因为配置文件~/.kube/下没有配置文件

解决方法：首先当你kubeadm join成功后，在/etc/kubenetes/下有一个conf文件，将它复制为~/.kube/config即可解决。

### calico镜像pull缓慢或出错

将containerd 源从
`registry.cn-hangzhou.aliyuncs.com/google_containers/pause`

换成
``registry.aliyuncs.com/k8sxio/pause`
