# Kubernetes1.10.3离线部署安装说明文档

 K8S集群部署有几种方式：`kubeadm`、`minikube`和二进制包。前两者属于自动部署，简化部署操作，但不适用与生产环境部署。
## 高可用简介
   在`Kubernetes`体系中，`Master`服务扮演着总控中心的角色，主要的三个服务：
`kube-apiserver`、`kube-controller-mansger`和`kube-scheduler`。
    通过不断与工作节点上的`Kubelet`和`kube-proxy`进行通信来维护整个集群的健康工作状态。如果`Master`的服务无法访问到某个`Node`，则会将该`Node`标记为不可用，不再向其调度新建的Pod。但对`Master`自身则需要进行额外的监控，使`Master`不成为集群的单故障点，所以对`Master`服务也需要进行高可用方式的部署。
以`Master`的`kube-apiserver`、`kube-controller-mansger`和`kube-scheduler`三个服务作为一个部署单元，类似于`etcd`集群的典型部署配置。使用至少三台服务器安装`Master`服务，并且使用`Active-Standby-Standby`模式，保证任何时候总有一套Master能够正常工作。
## HA原理
所有工作节点上的`Kubelet`和`kube-proxy`服务则需要访问`Master`集群的统一访问入口地址。下图展示了一种典型的部署方式。
![](_v_images/20201129123002951_4141.png)

#### 配置说明：
1、所有组件可以通过`kubelet static pod`的方式启动和管理，由`kubelet static pod`机制保证宿主机上各个组件的高可用, 注意`kubelet`要添加配置`--allow-privileged=true`;


2、管理`static pod`的`kubelet`的高可用通过`systemd`来负责；


3、当然，你也可以直接通过进程来部署这些组件，`systemd`来直接管理这些进程；（我们选择的是这种方式，降低复杂度。）


4、上图中，`etcd`和`Master`部署在一起，三个`Master`节点分别部署了三个`etcd`，这三个`etcd`组成一个集群；（当然，如果条件允许，建议将`etcd`集群和`Master`节点分开部署。）


5、每个`Master`中的`apiserver`、`controller-manager`、`scheduler`都使用`hostNetwork`,`controller-manager`和`scheduler`通过`localhost`连接到本节点的`apiserver`，而不会和其他两个`Master`节点的`apiserver`连接；


6、外部的`rest-client`、`kubectl`、`kubelet`、`kube-proxy`等都通过`TLS`证书，在`LB`节点做`TLS Termination`，`LB`出来就是`http`请求发到经过LB策略`（RR）`到对应的`apiserver instance`；


7、`apiserver`到`kubelet server`和`kube-proxy server`的访问也类似，`Https`到`LB`这里做`TLS Termination`，然后`http`请求出来到对应`node`的`kubelet`/`kube-proxy server`；


8、`apiserver`的`HA`通过经典的`haproxy + keepalived`来保证，集群对外暴露`VIP`；


9、`controller-manager`和`scheduler`的`HA`通过自身提供的`leader`选举功能`（--leader-select=true）`，使得3个`controller-manager`和`scheduler`都分别只有一个是`leader`，`leader`处于正常工作状态，当`leader`失败，会重新选举新`leader`来顶替继续工作；


因此，该HA方案中，通过`haproxy+keepalived`来做`apiserver`的`LB`和`HA`，`controller-manager`和`scheduler`通过自身的`leader`选举来达到`HA`，`etcd`通过`raft`协议保证`etcd cluster`数据的一致性，达到`HA`；


**注意：**`LB`所在的节点，注意确保`ip_vs model`已加载、`ip_forward`和`ip_nonlocal_bind`已开启；


当然上图内部结构太过复杂，难以理解。简化后的架构如下所示：
![](_v_images/20201129122912115_18668.png =929x)

## 下面介绍离线手工部署K8S的方法：
  搭建`kubernetes`集群最大的麻烦其实不在于其复杂度，而在于有`GFW`, 所以为了避免墙带来的麻烦, 也为了加深对`kubernetes`的理解, 这里将使用纯手工离线的方式进行部署
### 环境介绍
下面`Kubernetes`集群搭建需要的版本信息:
| 软件        | 版本                                 |
| :---------- | :----------------------------------- |
| OS          | CentOS Linux release 7.2.1511 (Core) |
| Kubernetes  | 1.10.3                               |
| Docker      | 18.09.1-ce                           |
| Etcd        | 3.3.1                                |
| Flannel     | 0.10.0                               |
| CNI-Plugins | 0.7.0                                |



我们将在四台`CentOS`系统的物理机上部署一个2个节点的`kubernetes1.10.3`集群：
节点信息如下：

| 主机名（必须） | IP           | 服务 |
| :----------- | :----------- | :-- |
|              | 10.162.0.100 |     |
|              |              |     |
|              |              |     |
|              |              |     |
|              |              |     |
|              |              |     |
|              |              |     |
|              |              |     |
|              |              |     |
|              |              |     |
|              |              |     |
|              |              |     |
|              |              |     |


Harbor仓库	10.162.0.200；可以用域名解析，也可以直接IP，本教程已事先解析域名 harbor.hzins.com
注：
1、所有操作均为root用户操作
2、必须事先设置好ETCD集群的主机名，否则集群无法正常启动
3、必须保证主机名能够被/etc/hosts解析
4、所有服务器事先全部初始化完成
5、Master和Node节点关闭SWAP分区
在进行集群搭建时, 由于很多镜像都需要翻墙下载, 因此先安装一个Harbor私有仓库。
环境准备
加载网络转发模块modprobe br_netfilter
(此操作在所有Master和Node节点操作) 
vim /etc/default/grub
在GRUB_CMDLINE_LINUX参数中添加cgroup_enable=memory swapaccount=1
yum upgrade
reboot
modprobe br_netfilter

设置环境变量，开启转发选项
echo "net.ipv6.conf.all.disable_ipv6 = 1" >>/etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >>/etc/sysctl.conf
echo "net.ipv6.conf.lo.disable_ipv6 = 1" >>/etc/sysctl.conf
echo "net.ipv4.neigh.default.gc_stale_time=120" >>/etc/sysctl.conf
echo "net.ipv4.conf.all.rp_filter=0" >>/etc/sysctl.conf
echo "net.ipv4.conf.default.arp_announce = 2" >>/etc/sysctl.conf
echo "net.ipv4.conf.lo.arp_announce=2" >>/etc/sysctl.conf
echo "net.ipv4.conf.all.arp_announce=2" >>/etc/sysctl.conf
echo "net.ipv4.tcp_max_syn_backlog = 1024" >>/etc/sysctl.conf
echo "net.ipv4.tcp_synack_retries = 2" >>/etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >>/etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >>/etc/sysctl.conf
echo "net.bridge.bridge-nf-call-arptables = 1 " >>/etc/sysctl.conf
sysctl -p

Harbor镜像仓库部署
1、安装内网docker 
cd /etc/yum.repos.d
curl -o ./docker-ce.repo  http://10.145.0.43/docker-ce/linux/centos/docker-ce.repo
yum install -y yum-utils device-mapper-persistent-data lvm2
yum install docker-ce  #默认安装最新版18.09版本

2、修改Docker存储路径、远程连接方式、启动方式、仓库地址
vim /usr/lib/systemd/system/docker.service  
ExecStartPost=/usr/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
ExecStart=/usr/bin/dockerd --graph /data/docker -H tcp://0.0.0.0 -H unix:///var/run/docker.sock  --insecure-registry harbor.hzins.com（harbor地址）
DOCKER_OPTS="--insecure-registry=harbor.hzins.com"

3、reload配置文件 
systemctl daemon-reload

4、启动docker 
systemctl start docker.service

至此 Docker安装完成  (harbor和Node节点都需要安装Docker)
5、下载harbor离线安装包
wget http://harbor.orientsoft.cn/harbor-v1.4.0/harbor-offline-installer-v1.4.0.tgz
6、安装compose编排工具
yum -y install docker-compose

7、解压
tar zxf harbor-offline-installer-v1.4.0.tgz
cd harbor
vim harbor.cfg    将hostname修改为仓库的域名或者直接写成IP地址
bash install.sh

至此 仓库完成
访问仓库地址 http://harbor.hzins.com/harbor 即可
默认账户：admin
默认密码：Harbor12345
登陆界面之后，创建一个Kubernetes的项目，访问级别设置为：公开