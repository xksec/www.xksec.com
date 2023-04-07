---
title: "安装原生Kubernetes单机/集群版"
date: 2022-12-30T14:09:00+08:00
draft: false
tags:
  - Kubernetes
---

这里介绍安装原生Kubernetes单机/集群版的方法，非Minikube、非Kind、非Colima等指令； 使用的是原生kubelet、kubeadm来部署Kubernetes。

kubeadm是Kubernetes官方提供的快速安装集群的工具，伴随着Kubernetes的版本发布进行更新。
<!--more-->

# 环境准备

## Linux

如果你有一台Linux环境，并且具备4C以上的CPU、8G以上的内存，并且安装了CentOS7或Ubuntu 20.XX以上版本的操作系统，那么就可以看下一章节了。

## Windows

可以使用VirtualBox + Vagrant 来搭建一个Kubernetes集群环境。模拟 master*1+worker*2的3节点环境。

### 安装软件

分别安装 [Virtualbox](https://www.virtualbox.org/wiki/Downloads) 和 [Valgrant](https://www.vagrantup.com/)

截止到今天`2023.04.07`，virtualbox和vagrant对arm64的支持还非常不完善，对x86的支持比较好。

### 导入镜像

由于国内的网络环境体验比较差，需要手工下载box镜像并导入到vagrant系统中。

先写一个简单的Vagrantfile：

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/centos8"

  config.vm.define "master" do |vb|
    vb.vm.hostname = "master-node"
    vm.vm.network "private_network", ip: "192.168.33.100"
  end
end
```

执行 `vagrant up` 会自动下载box镜像，如果碰到网络环境不好的话，会卡在下载步骤。这时候就需要手动拷贝提示的URL并下载，然后导入：

导入需要使用 `metadata.json`:

```json
{
    "name": "generic/centos8",
    "versions": [{
        "version": "4.2.14",
        "providers": [{
            "name": "virtualbox",
            "url": "file:///D:/Downloads/fe15bfbf-d39d-4ee6-99cb-9624fa4be44f"
        }]
    }]
}
```

执行指令: `vagrant box add ./metadata.json` 即可完成镜像的导入。


### 编写 Vagrantfile

此文件实现了 master*1+worker*2的环境：

```ruby
IP_NW="192.168.33."
IP_START=100

Vagrant.configure("2") do |config|
  config.vm.box = "generic/centos8"
  
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 4096
    vb.cpus = 2
	vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end
  
  config.vm.provision "shell", inline: <<-SHELL
	  echo "$IP_NW$((IP_START)) master-node" >> /etc/hosts
	  echo "$IP_NW$((IP_START+1)) worker-node01" >> /etc/hosts
	  echo "$IP_NW$((IP_START+2)) worker-node02" >> /etc/hosts
  SHELL
  
  config.vm.define "master" do |vb|
    vb.vm.hostname = "master-node"
    vb.vm.network "private_network", ip: IP_NW + "#{IP_START}"
  end

  (1..2).each do |i|
	config.vm.define "node0#{i}" do |node|
		node.vm.hostname = "worker-node0#{i}"
		node.vm.network "private_network", ip: IP_NW + "#{IP_START + i}"
	end
  end 
  
end
```

安装完成后可以使用 `vagrant ssh-config` 获取ssh 配置信息。

### 更改为root登录

```bash
$ vagrant ssh master-node
$ sudo -s
$ cd /root
$ mkdir -p .ssh && chmod 0700 .ssh
$ cp -af /home/vagrant/.ssh/authorized_keys .ssh/
$ chown root:root .ssh/authorized_keys
```

### 清理Vagrant环境

如果vagrant 没有启动成功，那么清理的话分为以下几步：
- 重启Windows，避免virtualbox关闭不了，卡在Shutting的过程
- 在VirtualBox中删除所有的虚拟机
- 删除vagrant工程中的.vagrant目录
- 删除 c:\users\username\.vagrant.d\data\machine-index\* 所有索引文件
- 再次执行 vagrant up 命令


## 更新kernel 

如果使用的宿主机是CentOS7，则需要更新Kernel到较新的版本，如果是CentOS8则无需更新。

```bash
# 导入ElRepo库的gpg密钥
$ rpm –import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# 添加ElRepo的repo配置
$ yum install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
# $ yum install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm
# $ yum install https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm

# 列出可用的内核版本
$ yum list available –disablerepo=’*’ –enablerepo=elrepo-kernel
# 安装lt支持的版本， lt= long-term
$ yum –enablerepo=elrepo-kernel install kernel-lt


# 查看当前内核版本及启动顺序
$ awk -F\' '$1=="menuentry " {print i++ " : " $2}' /boot/grub2/grub.cfg
0 : CentOS Linux (5.4.108-1.el7.elrepo.x86_64) 7 (Core)
1 : CentOS Linux (3.10.0-1160.11.1.el7.x86_64) 7 (Core)
2 : CentOS Linux (3.10.0-1160.el7.x86_64) 7 (Core)
3 : CentOS Linux (0-rescue-20210128140208453518997635111697) 7 (Core)

# 设置内核默认启动顺序，若无此命令则安装 grub2-pc 包
$ grub2-set-default 0

# 重启系统
$ reboot 

```

## 更新HostName

如果非Vagrant环境的话， 需要手动配置Hostname，这里配置宿主机的`hostname`为`master-node`

由于Kubernetes使用Hostname来进行配置主机，所以这里配置宿主机的`hostname`为`master-node`

```bash
$ echo 'master-node' > /etc/hostname
$ hostname master-node
$ echo '127.0.0.1 master-node' >> /etc/hosts

# 如果有多台Node设备，则需要将Node设备的Hostname设置到/etc/hosts中
```

## 设置SSH免密访问

在 `master-node` 上创建SSH密钥，并同步到所有Node主机

```bash
$ ssh-keygen -t rsa
$ ssh-copy-id -i /root/.ssh/id_rsa.pub master-node

# 如果有多台Node设备，则需要将ssh密钥拷贝到所有的Node主机上
```

## 关闭交换分区

```bash
$ swapoff -a
$ sed -ri 's/.*swap.*/#&/' /etc/fstab
$ sysctl -w vm.swappiness=0
```

## 安装容器管理工具

截止到目前（2022.12.30），kubernetes 1.26版本不再支持docker作为其容器管理工具，
所以这里选择安装1.23版本的Kubernetes，可以支持使用docker作为容器管理工具。

安装docker-ce，详细步骤点击：

 - [CentOS 安装步骤](https://docs.docker.com/engine/install/centos/)
 - [Ubuntu 安装步骤](https://docs.docker.com/engine/install/ubuntu/)

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

[阿里云Kubernetes镜像配置方法](https://developer.aliyun.com/mirror/kubernetes/?spm=a2c6h.25603864.0.0.2619274fzqz5ya)

**Debian/Ubuntu**
```bash
$ apt-get update && apt-get install -y apt-transport-https
$ curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
$ cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
$ apt-get update
$ apt-get install -y kubelet-1.23.15-0 kubeadm-1.23.15-0 kubectl-1.23.15-0
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
$ yum install -y kubelet-1.23.15-0 kubeadm-1.23.15-0 kubectl-1.23.15-0
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
$ kubectl taint nodes master-node node-role.kubernetes.io/master:NoSchedule-
```


**提前导入镜像**

如果创建多Node节点的集群，可以使用提前导入镜像的方式，使用以下脚本，将镜像上传到docker.com作为中转：

使用脚本 k8s-outside.sh，从国外站点拉取最新镜像，上传到dockerhub
```bash
#!/bin/bash

repo=ir0cn
#repo=quay.io/xxfe
#repo=registry.cn-hangzhou.aliyuncs.com/xxfe

images=$(kubeadm config images list --kubernetes-version=v1.23.15 2>/dev/null | awk '{print $1}')

for imageName in  ${images[@]}; do
    docker pull $imageName || exit -1
    newImageName=$(echo $imageName | awk -F/ '{print $NF}' | sed 's@:@__@')
    docker tag $imageName $repo/google_containers:$newImageName || exit -1
    docker push $repo/google_containers:$newImageName || exit -1
done
```

使用脚本 k8s-inside.sh，从dockerhub拉取镜像并改名，每台机器均需执行
```bash
#!/bin/bash

repo=ir0cn
#repo=quay.io/xxfe
#repo=registry.cn-hangzhou.aliyuncs.com/xxfe

images=$(kubeadm config images list --kubernetes-version=v1.23.15 2>/dev/null | awk '{print $1}')

for imageName in ${images[@]}; do
    newImageName=$(echo $imageName | awk -F/ '{print $NF}' | sed 's@:@__@')
    docker pull $repo/google_containers:$newImageName || exit -1
    docker tag $repo/google_containers:$newImageName $imageName
    docker rmi $repo/google_containers:$newImageName
done
```


# 重置集群

```bash
kubeadm reset
rm -rf /etc/kubernetes /etc/cni/net.d
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X

```


# 坑

- centos8默认开启了防火墙，kubeadm join 无效; 关闭防火墙 `systemctl stop firewalld`

