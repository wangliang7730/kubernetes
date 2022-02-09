# Install Kubernetes Cluster using kubeadm
Follow this documentation to set up a Kubernetes cluster on __CentOS 7__.

This documentation guides you in setting up a cluster with one master node and one worker node.

## Assumptions
|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Master|kmaster.example.com|172.16.16.100|CentOS 7|2G|2|
|Worker|kworker.example.com|172.16.16.101|CentOS 7|1G|1|

## On both Kmaster and Kworker
Perform all the commands as root user unless otherwise specified
##### Disable Firewall
```
systemctl disable firewalld; systemctl stop firewalld
```
##### Disable swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
##### Disable SELinux
```
setenforce 0
sed -i --follow-symlinks 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
```
##### Update sysctl settings for Kubernetes networking
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
##### Install docker engine
```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce-19.03.12 
systemctl enable --now docker
```
### Kubernetes Setup
##### Add yum repository
```
cat >>/etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
#注释该行使用阿里的镜像库 baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
#注释该行使用阿里的镜像库 gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        #https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg

gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg 
       https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg        
EOF

```
##### Install Kubernetes components
```
#早期版本注释 yum install -y kubeadm-1.18.5-0 kubelet-1.18.5-0 kubectl-1.18.5-0
yum install -y kubeadm-1.20.11-0 kubelet-1.20.11-0 kubectl-1.20.11-0
```
##### Enable and Start kubelet service
```
systemctl enable --now kubelet
```
## On kmaster
##### Initialize Kubernetes Cluster
```
kubeadm init --apiserver-advertise-address=172.16.16.100 --pod-network-cidr=192.168.0.0/16 --kubernetes-version=v1.20.11 --image-repository registry.aliyuncs.com/google_containers #固定镜像源参考https://blog.csdn.net/weixin_30752699/article/details/101417734  #如果报错可以加上 --kubernetes-version=v1.20.11 指定版本
如果还出现错误请 kubeadm reset 操作重新安装和初始化
```
##### Deploy Calico network
```
#注释kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
##### Cluster join command
```
kubeadm token create --print-join-command
```
##### To be able to run kubectl commands as non-root user
If you want to be able to run kubectl commands as non-root user, then as a non-root user perform these
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```
## On Kworker
##### Join the cluster
Use the output from __kubeadm token create__ command in previous step from the master server and run here.

## Verifying the cluster
##### Get Nodes status
```
kubectl get nodes
```
##### Get component status
```
kubectl get cs
```

Have Fun!!

#### 报错及解决
## The connection to the server localhost:8080 was refused
Master节点出现这个报错
首先需要检查Master安装完Kubernetes后是否执行了下面命令。需要注意到是：如果整个过程都是在普通用户下使用sudo安装，则仍然需要在普通用户下执行了下面命令；如果整个过程都在root用户下安装，则还在root用户下执行了下面命令。

mkdir -p $HOME/.kube
#root用户下执行不需要加sudo
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

如果执行了，就到用户目录下检查是否有.kube文件夹，文件夹中是否有config文件，如果都有还是报错，就需要执行下面命令把它加入到环境变量中：

export KUBECONFIG=$HOME/.kube/config

Node节点出现这个报错
kubectl命令需要使用kubernetes-admin来运行，所以需要将主节点中的/etc/kubernetes/admin.conf文件拷贝到从节点用户目录下，然后配置环境变量：

#在Master节点运行下面命令将admin.conf文件拷贝到从节点
sudo scp /etc/kubernetes/admin.conf root@192.168.63.131:~
#在Node节点运行下面命令配置环境变量
export KUBECONFIG=$HOME/admin.conf

