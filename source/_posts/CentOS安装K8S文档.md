---
title: Centos7 安装 k8s
categories:
 - K8S
tags:
 - Centos
 - K8S
 - 教程
---

# Centos7 安装 k8s 环境

> 安装centos7虚拟机

* Vmware 下载 mini 包[x86_64下载地址](http://mirrors.oit.uci.edu/centos/7.9.2009/isos/x86_64/)

* 硬件至少 2G 2核

> 初始配置虚拟机网络

* 更改hostname
  
  * echo 'master' >> /etc/hostname

* 更改静态ip
  
  * 打开 /etc/sysconfig/network-scripts/ifcfg-ens33  ，网卡名称有的使eth0 有的是ens33，使用 ip addr查看
  
  * 将 BOOTPROTO=dhcp 改为 static
  
  * 将ONBOOT=no 改为yes
  
  * 末尾加上你的配置
    
    ```shell
    # /etc/sysconfig/network-scripts/ifcfg-ens33
    IPADDR=192.168.111.201
    
    NETMASK=255.255.255.0
    
    GATEWAY=192.168.111.2
    
    DNS1=8.8.8.8
    ```

* /etc/hosts 加上 ip ：hostname
  
  ```shell
  # /etc/hosts
  192.168.111.200 master
  
  192.168.111.201 node1
  
  192.168.111.202 node2
  ```

* systemctl restart network 

* 使配置生效，此时能ping通外网

> 使用vscode或其他应用连接虚拟机

> 同步时间 补全命令

* ```shell
  sudo yum -y install ntpdate
  sudo ntpdate ntp1.aliyun.com
  sudo systemctl status ntpdate
  sudo systemctl start ntpdate
  sudo systemctl status ntpdate
  sudo systemctl enable ntpdate
  ```

* ```shell
  sudo yum -y install bash-completion
  source /etc/profile
  ```

> 关闭防火墙 或者 开启防火墙端口

* ```shell
  sudo systemctl stop firewalld.service 
  sudo systemctl disable firewalld.service
  ```

* ```shell
  # 控制面板
  firewall-cmd --zone=public --add-port=6443/tcp --permanent # Kubernetes API server    所有
  firewall-cmd --zone=public --add-port=2379/tcp --permanent # etcd server client API    kube-apiserver, etcd
  firewall-cmd --zone=public --add-port=2380/tcp --permanent # etcd server client API    kube-apiserver, etcd
  firewall-cmd --zone=public --add-port=10250/tcp --permanent # Kubelet API    自身, 控制面
  firewall-cmd --zone=public --add-port=10259/tcp --permanent # kube-scheduler    自身
  firewall-cmd --zone=public --add-port=10257/tcp --permanent # kube-controller-manager    自身
  firewall-cmd --zone=trusted --add-source=192.168.111.200 --permanent # 信任集群中各个节点的IP
  firewall-cmd --zone=trusted --add-source=192.168.111.201 --permanent # 信任集群中各个节点的IP
  firewall-cmd --zone=trusted --add-source=192.168.111.202 --permanent # 信任集群中各个节点的IP
  firewall-cmd --add-masquerade --permanent # 端口转发
  firewall-cmd --reload
  firewall-cmd --list-all
  firewall-cmd --list-all --zone=trusted
  
  # 工作节点
  firewall-cmd --zone=public --add-port=10250/tcp --permanent # Kubelet API    自身, 控制面
  firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent # NodePort Services†    所有
  firewall-cmd --zone=trusted --add-source=192.168.111.200 --permanent # 信任集群中各个节点的IP
  firewall-cmd --zone=trusted --add-source=192.168.111.201 --permanent # 信任集群中各个节点的IP
  firewall-cmd --zone=trusted --add-source=192.168.111.202 --permanent # 信任集群中各个节点的IP
  firewall-cmd --add-masquerade --permanent # 端口转发
  firewall-cmd --reload
  firewall-cmd --list-all
  firewall-cmd --list-all --zone=trusted
  ```

> 关闭swap 和 selinux

* ```shell
  free -h  //查看是否关闭
  sudo swapoff -a  //暂时关闭
  sudo sed -i 's/.*swap.*/#&/' /etc/fstab //永久关闭
  free -h
  ```
  
  ```shell
  getenforce
  cat /etc/selinux/config
  sudo setenforce 0
  sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  cat /etc/selinux/config
  ```

> 安装docker 和 containerd，1.24.0版本开始，k8s使用containerd代替docker

* cgroup（control group）是 Linux 内核提供的一种机制，用于对进程进行资源限制、统计和控制。通过 cgroup，我们可以对一个或多个进程的 CPU、内存、IO、网络等资源进行限制，从而保证系统资源的合理分配，防止某些进程占用过多的资源导致系统崩溃或变慢。

* ```shell
  # https://docs.docker.com/engine/install/centos/
  sudo yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
  sudo yum install -y yum-utils device-mapper-persistent-data lvm2
  sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo 
  # yum --showduplicates list docker-ce
  sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
  sudo yum install -y containerd
  
  # 启动 docker 时，会启动 containerd
  # sudo systemctl status containerd.service
  sudo systemctl stop containerd.service
  
  # 设置containerd 的默认配置文件
  sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak
  sudo containerd config default > $HOME/config.toml
  sudo cp $HOME/config.toml /etc/containerd/config.toml
  
  # 对containerd 换源和设置cgroup的设置
  sudo sed -i "s#registry.k8s.io/pause#registry.aliyuncs.com/k8sxio/pause#g" /etc/containerd/config.toml
  # https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#containerd-systemd
  # 确保 /etc/containerd/config.toml 中的 disabled_plugins 内不存在 cri
  sudo sed -i "s#SystemdCgroup = false#SystemdCgroup = true#g" /etc/containerd/config.toml
  
  # containerd 忽略证书验证的配置
  #      [plugins."io.containerd.grpc.v1.cri".registry.configs]
  #        [plugins."io.containerd.grpc.v1.cri".registry.configs."192.168.0.12:8001".tls]
  #          insecure_skip_verify = true
  sudo systemctl enable --now containerd.service
  
  sudo systemctl start docker.service
  ```

```shell
sudo systemctl enable docker.service
sudo systemctl enable docker.socket
sudo systemctl list-unit-files | grep docker

sudo mkdir -p /etc/docker

# 对docker换源和cgroup设置
cat<<EOF | sudo tee /etc/docker/daemon.json 
 {
 "registry-mirrors": ["https://2u8f97e8.mirror.aliyuncs.com"],
 "exec-opts": ["native.cgroupdriver=systemd"]
 }
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
sudo docker info

sudo systemctl status docker.service
sudo systemctl status containerd.service
```

> 安装k8s

* 添加k8s 仓库

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
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

EOF
```

* 设置内核参数

```shell
# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1 
net.bridge.bridge-nf-call-ip6tables = 1  //这两条让网络连接走iptables（禁用可以不设置）
net.ipv4.ip_forward  = 1  //是否允许将一个网络接口收到的数据包转发到另一个网络接口

EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

* 安装1.26.2版本

```shell
sudo yum install -y kubelet-1.26.2-0 kubeadm-1.26.2-0 kubectl-1.26.2-0 --disableexcludes=kubernetes --nogpgcheck

systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl enable kubelet
```

以上内容需要在节点和master机器全部运行一次

----

> master上执行

```shell
kubeadm init --image-repository=registry.aliyuncs.com/google_containers
# 指定集群的IP
# kubeadm init --image-repository=registry.aliyuncs.com/google_containers --apiserver-advertise-address=192.168.80.60

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 或者在环境变量中添加：export KUBECONFIG=/etc/kubernetes/admin.conf
# 添加完环境变量后，刷新环境变量：source /etc/profile

kubectl cluster-info

# 初始化失败后，可进行重置，重置命令：kubeadm reset -

# 执行成功后，会出现类似下列内容：
# kubeadm join 192.168.80.60:6443 --token f9lvrz.59mykzssqw6vjh32 \
# --discovery-token-ca-cert-hash sha256:4e23156e2f71c5df52dfd2b9b198cce5db27c47707564684ea74986836900107     

# 查看join命令 
# kubeadm token create --print-join-command
```

如果init失败，查看kubelet的运行日志

```shell
journalctl -xefu kubelet
```

> node上执行

```shell
# 运行的内容来自上方执行结果
kubeadm join 192.168.80.60:6443 --token f9lvrz.59mykzssqw6vjh32 \
--discovery-token-ca-cert-hash sha256:4e23156e2f71c5df52dfd2b9b198cce5db27c47707564684ea74986836900107 

#
# kubeadm token create --print-join-command

# kubeadm join 192.168.80.60:6443 --token f9lvrz.59mykzssqw6vjh32 \
# --discovery-token-unsafe-skip-ca-verification
```

> 此时node还是not ready  pod 也没有正常运行，需要在master配置网络

```shell
yum install -y wget
wget --no-check-certificate https://projectcalico.docs.tigera.io/archive/v3.25/manifests/calico.yaml
```

```shell
# 修改 calico.yaml 文件
vim calico.yaml
```

```shell
# 在 - name: CLUSTER_TYPE 下方添加如下内容
- name: CLUSTER_TYPE
  value: "k8s,bgp"
  # 下方为新增内容
- name: IP_AUTODETECTION_METHOD
  value: "interface=网卡名称"


# INTERFACE_NAME=ens33
# sed -i '/k8s,bgp/a \            - name: IP_AUTODETECTION_METHOD\n              value: "interface=INTERFACE_NAME"' calico.yaml
# sed -i "s#INTERFACE_NAME#$INTERFACE_NAME#g" calico.yaml
```

```shell
# 配置网络
kubectl apply -f calico.yaml
```

之后等待网络配置完成

k8s搭建成功

---

> kubeadm 有问题访问[对 kubeadm 进行故障排查 | Kubernetes](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)

> 网络问题

* node加入master之后get nodes 失败

* ![](http://gohoy.top/i/2023/08/11/rd09c0-1.png)

* master 显示该node notready

* pods 网络问题 sandbox create failed
  
  * ![](http://gohoy.top/i/2023/08/11/rd0dze-1.png)
  
  * ![](http://gohoy.top/i/2023/08/11/rd0hum-1.png)

* 主要问题：calico镜像pull缓慢或出错
  
  * ![](http://gohoy.top/i/2023/08/11/rd0od9-1.png)
  
  * ![](http://gohoy.top/i/2023/08/11/rd0n9o-1.png)
  
  * ![](http://gohoy.top/i/2023/08/11/rd182i-1.png)

等了非常久 calico pull成功

最后将containerd 源从

`registry.cn-hangzhou.aliyuncs.com/google_containers/pause`

换成

`registry.aliyuncs.com/k8sxio/pause`

pull镜像可以成功了

> 实践

* 尝试搭建nginx

* ```shell
  cat > nginx.yaml << EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
    labels:
      app: nginx
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx:1.23.2
          ports:
          - containerPort: 80
  
  EOF
  
  cat nginx.yaml
  
  # 创建deployment
  kubectl apply -f nginx.yaml
  ```

* 

* pull 结束后

* ```shell
  # 控制面板：设置服务
  kubectl expose deployment nginx-deployment --type=NodePort --name=nginx-service
  # 或者
  # kubectl create service nodeport nginx-service --tcp=80:80 --node-port=32767 --selector=app=nginx
  ```

* ```shell
  # 控制面板：查看pod,svc
  kubectl get pod,svc -o wide
  ```

* ![](http://gohoy.top/i/2023/08/11/rd1djq-1.png)

* ![](http://gohoy.top/i/2023/08/11/rd1tb5-1.png)

* 停止pods
  
  * 删除deployment
