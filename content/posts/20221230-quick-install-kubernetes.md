---
title: "安装原生Kubernetes单机版"
date: 2022-12-30T14:09:00+08:00
draft: false
---

使用一台物理机安装Kubernetes，非Minikube，非Kind；使用原生的kubelet、kubeadm进行单机版Kubernetes的部署。

<!--more-->

# 环境准备

## 更新kernel 

如果宿主机是CentOS7，则需要更新Kernel到较新的版本，如果是CentOS8则无需更新。

## 更新HostName

由于Kubernetes使用Hostname来进行配置主机，所以这里配置宿主机的`hostname`为`k8s-master`

```bash
$ echo 'k8s-master' > /etc/hostname
$ hostname k8s-master
$ echo '127.0.0.1 k8s-master' >> /etc/hosts

# 如果有多台Node设备，则需要将Node设备的Hostname设置到/etc/hosts中
```

## 设置SSH免密访问

在k8s-master上创建SSH密钥，并同步到所有Node主机

```bash
$ ssh-keygen -t rsa
$ ssh-copy-id -i /root/.ssh/id_rsa.pub k8s-master

# 如果有多台Node设备，则需要将ssh密钥拷贝到所有的Node主机上
```

## 关闭交换分区

```bash
$ swapoff -a
$ sed -ri 's/.*swap.*/#&/' /etc/fstab
$  sysctl -w vm.swappiness=0
```

## 安装容器管理工具

> 截止到目前（2022.12.30），kubernetes 1.26版本不再支持docker作为其容器管理工具，
> 所以这里选择安装1.23版本的Kubernetes，可以支持使用docker作为容器管理工具。

安装docker-ce，详细步骤点击： [CentOS](https://docs.docker.com/engine/install/centos/)  [Ubuntu](https://docs.docker.com/engine/install/ubuntu/)


## 设置docker配置

需要修改docker的cgroup driver 为systemd， 同时storage driver为overlay2
也是overlay2需要高版本kernel的支持。

```bash
$ cat /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "storage-driver": "overlay2",
  "registry-mirrors": ["https://registry.docker-cn.com"]
}

$ service docker restart
```

# 安装Kubernetes工具

需要安装的工具包括：
- kubeadm
- kubelet
- kubectl

由于国内环境，这里采用aliyun的源进行安装指定1.23.15-0版本

**Debian/Ubuntu**
```bash
$ apt-get update && apt-get install -y apt-transport-https
$ curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
$ cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
$ apt-get update
$ apt-get install -y kubelet-1.13.15-0 kubeadm-1.13.15-0 kubectl-1.13.15-0
```

**CentOS/RHEL/Fedora**
```bash
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
$ setenforce 0
$ yum install -y kubelet-1.13.15-0 kubeadm-1.13.15-0 kubectl-1.13.15-0
$ systemctl enable kubelet && systemctl start kubelet
```

# 初始化集群

**安装集群**
```bash
# 使用docker作为容器管理工具，需指定1.23版本，同时指定aliyun镜像仓库
$ kubeadm init --apiserver-advertise-address=10.226.193.7 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version=v1.23.15 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --v=5
# --apiserver-advertise-address: kube-apiserver对外侦听的地址；由于kubeadm默认使用eth0的地址作为侦听地址，在某些情况下不适用
# --image-repository: 由于一些不可描述的原因，国内无法下载kubernetes的容器镜像，这里使用阿里云的镜像仓库
# --kubernetes-version=v1.23.15: 指定安装kubernetes的版本，1.23版本是支持docker作为容器支持的
```


**初始化网络插件**

安装好集群后，使用`kubectl get node` 查看节点状态，会显示`NotReady`，这是由于集群还没有初始化网络。
这里安装flannel网络模块：

```bash
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

**取消主节点不可调度限制**

由于是单机版，所以需要去掉主节点的`NoSchedule`的taint标记，否则的话安装Pod会无法找到可用的Node节点而失败。

```bash
$ kubectl taint nodes k8s-master node-role.kubernetes.io/master:NoSchedule-
```


