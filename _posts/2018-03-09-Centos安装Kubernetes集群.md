---
layout: post
title: Centos安装Kubernetes集群
date: 2018-03-09
tag: kubernetes 2018
---
kubeadm支持多种系统，这里简单介绍一下需要的系统要求：

 * Ubuntu16.04+ / Debian 9 / CentOS 7 / RHEL 7 / Fedora 25/26(best-effort) / HypriotOS v1.0.1+ / Other

 * 2GB或者以上的RAM（否则将没有足够空间留给app）

 * 2核以上CPU

 * 集群的机器之间必须能通过网络互相通信

 * SWAP必须被关闭，否则kubelet会出错！

具体的详细信息可以在官方网站上看到。

本篇内容基于tencent的s2， CentOS7的操作系统，实例类型s2 2核4GB，3台机器，1 master，2 nodes，kubernetes 1.9 版本。为了方便起见，在安全组里面打开了所有的端口和IP访问。

机器配置：

centos@hostname ~$ uname -a

Linux hostname.ap-northeast-1.compute.internal 3.10.0-693.5.2.el7.x86\_64 #1 SMP Fri Oct 20 20:32:50 UTC 2017 x86\_64 x86\_64 x86\_64 GNU/Linux

首先 ，我们关闭selinux：

``` bash
$ sudo vim /etc/sysconfig/selinux
```

把SELINUX改成disabled，然后保存退出。

在我用的ami中，swap是默认关闭的，所以不需要我手动关闭，大家需要确认 自己的环境中swap是否有关闭掉，否则会在之后的环节中出问题。

为了方便我们安装，我们将sshd设置为keepalive：

``` bash
$ sudo -i
$ echo "ClientAliveInterval 10" >> /etc/ssh/sshd_config
$ echo "TCPKeepAlive yes" >> /etc/ssh/sshd_config
$ systemctl restart sshd.service
```

接下来我们重启一下机器：

``` bash
$ sudo sync
$ sudo reboot
```

至此，准备阶段结束。

### 安装kubeadm

首先，我们需要在所有机器上都安装 docker, kubeadm, kubelet和 kubectl。

> 切记：kubeadm不会自动去安装和管理 kubelet和kubectl，所以需要自己去确保安装的版本和你想要安装的kubernetes版本相同。

#### 安装 docker：

```
$ sudo yum install -y docker
$ sudo systemctl enable docker && sudo systemctl start docker
```

在RHEL/CentOS 7 系统上可能会路由失败，我们需要设置一下：

```bash
$ sudo -i
$ cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
$ sudo sysctl --system
```

接下来我们需要安装 kubeadm, kubelet和 kubectl了，我们需要先加一个repo：

```repo
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

然后运行下面的命令安装：

```bash
$ sudo yum install -y kubelet kubeadm kubectl
$ sudo systemctl enable kubelet && sudo systemctl start kubelet
```

至此，在所有机器上安装所需的软件已经结束。

### 使用kubeadm初始化master

安装完所有的依赖之后，我们就可以用 kubeadm初始化master了。

最简单的初始化方法是：

```bash
$ kubeadm init
```

除此之外， kubeadm还支持多种方法来配置，具体可以查看一下官方文档。

我们在初始化的时候指定一下kubernetes版本，并设置一下pod-network-cidr（后面的flannel会用到）：

```bash
$ sudo -i
$ kubeadm init --kubernetes-version=v1.9.0 --pod-network-cidr=10.244.0.0/16
```

在这个过程中 kubeadm执行了一系列的操作，包括一些pre-check，生成ca证书，安装etcd和其它控制组件等。

最下面的这行 kubeadm join，就是用来让别的node加入集群的，可以看出非常方便。我们要保存好这一行东西，这是我们之后让node加入集群的凭据，一会儿会用到。

这个时候，我们还不能通过 kubectl来控制集群，要让 kubectl可用，我们需要做：

对于非root用户

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

对于root用户

```bash
$ export KUBECONFIG=/etc/kubernetes/admin.conf
# 也可以直接放到~/.bash_profile
$ echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
```

接下来要注意，我们必须自己来安装一个network addon。

network addon必须在任何app部署之前安装好。同样的，kube-dns也会在network addon安装好之后才启动。kubeadm只支持CNI-based networks（不支持kubenet）。

比较常见的network addon有： Calico, Canal, Flannel, Kube-router, Romana, WeaveNet等。这里我们使用 Flannel。

```bash
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
```

安装完network之后，你可以通过 `kubectlgetpods--all-namespaces`来查看 kube-dns是否在running来判断network是否安装成功。

默认情况下，为了保证master的安全，master是不会被调度到app的。你可以取消这个限制通过输入：

```bash
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

### 加入nodes

终于部署完了我们的master！

现在我们开始加入一些node到我们的集群里面吧！

ssh到我们的node节点上，执行刚才下面给出的那个 `kubeadm join`的命令（每个人不同）：

```bash
$ sudo -i
$ kubeadm join --token TOKEN 172.31.31.60:6443 --discovery-token-ca-cert-hash sha256:f0894e55d475f882dd40d52c6d01f758017ec5729be632294049f687330f60d2
```

这时候，我们去master上输入 `kubectl get nodes`查看一下：

```bash
[root@hostname ~]# kubectl get nodes
NAME                  STATUS    ROLES     AGE       VERSION
i-071abd86ed304dc84   Ready     master    12m       v1.9.0
i-0c559ad3c0b16fd36   Ready     <none>    1m        v1.9.0
i-0f3f7462b0a004b5e   Ready     <none>    47s       v1.9.0
```
