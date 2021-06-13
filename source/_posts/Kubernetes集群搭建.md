---
title: Kubernetes集群搭建
date: 2021-05-23 21:09:09
tags: 
- Ubuntu
- Docker
- Kubernetes
categories: 云原生

---







### 环境规划：

| IP地址         | 操作系统版本       | 内核版本                          | Docker版本             | Kubernetes版本 | 角色   |
| -------------- | ------------------ | --------------------------------- | ---------------------- | -------------- | ------ |
| 192.168.12.201 | Ubuntu 18.04.5 LTS | Linux node-202 4.15.0-130-generic | Docker version 20.10.6 |                | Master |
| 192.168.12.203 | Ubuntu 18.04.5 LTS | Linux node-202 4.15.0-130-generic | Docker version 20.10.6 |                | Node   |
| 192.168.12.203 | Ubuntu 18.04.5 LTS | Linux node-202 4.15.0-130-generic | Docker version 20.10.6 |                | Node   |



### 使用Vagrant创建虚拟机



#### vagrantfile

```text
Vagrant.configure("2") do |config|
   (201..204).each do |i|
        config.vm.define "node-#{i}" do |node|
            # 设置虚拟机的Box
            node.vm.box = "ubuntu-bionic"
            node.vm.box_url = "https://mirrors.tuna.tsinghua.edu.cn/ubuntu-cloud-images/bionic/current/bionic-server-cloudimg-amd64-vagrant.box"

            # 设置虚拟机的主机名
            node.vm.hostname="node-#{i}"

            # 设置虚拟机的IP
            node.vm.network "private_network", ip: "192.168.12.#{i}", netmask: "255.255.255.0"

            # 设置主机与虚拟机的共享目录
            # node.vm.synced_folder "~/Documents/vagrant/share", "/home/vagrant/share"

            # VirtaulBox相关配置
            node.vm.provider "virtualbox" do |v|
                # 设置虚拟机的名称
                v.name = "node-#{i}"
                # 设置虚拟机的内存大小
                v.memory = 8069
                # 设置虚拟机的CPU个数
                v.cpus = 4
            end
        end
   end
end
```



#### 初始化虚拟机

```text
vagrant init
```



#### 配置

在vagrantfile文件所在目录，使用 `vagrant ssh 主机名`登录虚拟机， 修改 `/etc/ssh/sshd_config` 文件，把相应内容改成如下形式：

##### 允许root以root用户登录系统

```text
PermitRootLogin yes
```

##### 允许以ssh形式进行远程登录

```text
PasswordAuthentication yes
```

自此，虚拟机即可采用ssh工具进行远程连接。





### 安装Docker环境：



#### 修改Ubuntu的安装源为阿里云源

```shell
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

把 `sources.list` 文件内容替换成如下内容

```text
deb http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse
```

更新：

```shell
sudo apt update
```



#### 修改Docker的安装源为阿里云源

添加Docker官方的GPG密钥

```she
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

更换镜像

```shel
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
```

更新：

```shell
sudo apt update
```



#### 查看系统可安装的版本：

```shell
apt-cache madison docker-ce
```



具体安装卸载可参考官方文档(https://docs.docker.com/engine/install/ubuntu/)



```shell
sudo apt-get -y install docker-ce
```

查看Docker版本：

```shell
sudo docker version
```



#### 运行非root用户查看Docker信息：

```shell
sudo usermod -aG docker ${USER}
```



#### 配置阿里云提供的Docker镜像加速地址

登录阿里云控制台，搜索 `容器镜像服务`，点击左侧 `镜像工具 -> 镜像加速地址`

```shell
sudo tee /etc/docker/daemon.json <<-'EOF'{    "registry-mirrors": ["https://xxx.mirror.aliyuncs.com"]}EOFsudo systemctl daemon-reloadsudo systemctl restart docker
```







### 安装Kubernetes



#### 前置准备

- 将Docker 的 Cgroup Driver 修改为 systemd，否则的话，在为后续使用kubeadm join命令的时候，会有warning，但是不影响使用

- 查看当前状态 

  - ```shell
    docker info | grep "Cgroup Driver"
    ```

  - ```shel
    {  "exec-opts": ["native.cgroupdriver=systemd"],  "registry-mirrors": ["https://xxx.mirror.aliyuncs.com"]}
    ```

  - 

Ubuntu 下切换用户至root使用的命令：

```shell
sudo -i
```



#### 添加阿里云Kubernetes安装源

```shell
sudo curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo  apt-key add -
```



```shel
cat <<EOF >/etc/apt/sources.list.d/kubernetes.listdeb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial mainEOF
```



更新：

```shell
apt-get update
```



搜索可安装的版本：

```shell
apt-cache madison kubeadm
```

安装指定版本：

```shell
apt-get install -y kubelet=1.20.7-00 kubeadm=1.20.7-00 kubectl=1.20.7-00
```

开机自启动

```shell
systemctl enable kubelet && systemctl start kubelet
```



**以上操作3太虚拟机均需要执行。**



#### 初始化Master节点： 

```text
kubeadm init \ --apiserver-advertise-address 192.168.12.203 \ --image-repository registry.aliyuncs.com/google_containers \ --kubernetes-version 1.20.7 \ --pod-network-cidr 10.10.0.0/16 \ --service-cidr 10.20.0.0/16 \ --upload-certs
```

#### flannel网络插件

注意：

```shell
kubectl apply -f flannel.yml
```

查看状态：

```shell
watch -n 2 kubectl get pods --all-namespaces -o wide
```



#### Node节点加入集群

```shell
kubeadm join 192.168.12.203:6443 --token cpbmqn.5lk0w4qzrcinuo97 \    --discovery-token-ca-cert-hash sha256:6a6e45045a8d93d4f2741fa8a5856e9428f9a47dbd4498a5b46d01cfc5d5a578 --v=5
```



#### 查看状态

```shell
kubectl get nodes
```



```text
kubectl describe pod -n kube-system  kube-flannel-ds-r8
```





### 安装Kubesphere



















































