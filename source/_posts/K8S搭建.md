---
title: K8S搭建
date: 2018-01-25 08:57:38
categories:
- 容器
- kubernates
tags:
- docker
- k8s
- kubernates
---
# K8S搭建文档 
## 硬件环境 ##
****三台centos7主机****

****两台做master(高可用)****

****1台做node****

****V1.6.3****

## 部署准备 ##
### Step 1   登录各个主机首先配置静态ip

### Step 2   关闭各个主机SELINUX&&防火墙
 setenforce 0
 systemctl stop firewalld.service
 systemctl disable firewalld.service 
### Step 3   设置各个主机hostname
 echo 192-168-232-133.master  /etc/hostname

 echo "127.0.0.1 192-168-232-133.master"  /etc/hosts  
 
 sysctl kernel.hostname=192-168-232-133.master

### Step 4   配置各个主机ip到dns的域名解析(需要配置所有集群中的主机)
 vi /etc/hosts
 192.168.232.133 192-168-232-133.master

### Step 5   启动各个主机network
 service network restart

### Step 6   验证主机节点网络和域名解析和主机名

### Step 7   安装工具
yum install vim lrzsz tmux -y

### Step 8   安装docker（1.12.6）

yum install -y docker-engine-selinux-1.12.6-1.el7.centos.noarch.rpm  
yum install -y docker-engine-1.12.6-1.el7.centos.x86_64.rpm 

### Step 9  启动docker  

systemctl start docker

## 部署K8S ##

### Step 10  导入k8s1.6.3&calico所需镜像 

docker load -i  k8s1.6.3/k8s-1.6.3-images/dnsmasq.tar  
docker load -i  k8s1.6.3/k8s-1.6.3-images/dns-nanny.tar  
docker load -i  k8s1.6.3/k8s-1.6.3-images/dns.tar  
docker load -i  k8s1.6.3/k8s-1.6.3-images/kube-apiserver.tar  
docker load -i  k8s1.6.3/k8s-1.6.3-images/kube-controller-manager.tar  
docker load -i  k8s1.6.3/k8s-1.6.3-images/kube-proxy.tar  
docker load -i  k8s1.6.3/k8s-1.6.3-images/kube-scheduler.tar  
docker load -i  k8s1.6.3/k8s-1.6.3-images/pause.tar  
docker load -i  k8s1.6.3/k8s-1.6.3-images/sidecar.tar  
docker load -i  k8s1.6.3/calico/cni.tar  
docker load -i  k8s1.6.3/calico/node.tar  
docker load -i  k8s1.6.3/calico/policy.tar

### Step 11  安装kubeadm1.5.1&kubelet&kubectl（以上步骤每台都执行）  
yum localinstall k8s1.6.3/centos/*.rpm -y  

### Step 12  安装etcd3.1.7(每台执行)
tar xvf etcd-v3.1.7-linux-amd64.tar  
cd etcd-v3.1.7-linux-amd64  
tmux
`./etcd --name infra0 --initial-advertise-peer-urls http://192.168.10.127:2380 \
      --listen-peer-urls http://192.168.10.127:2380 \
      --listen-client-urls http://192.168.10.127:2379,http://127.0.0.1:2379 \
      --advertise-client-urls http://192.168.10.127:2379 \
      --initial-cluster-token etcd-cluster-1 \
      --initial-cluster infra0=http://192.168.10.127:2380,infra1=http://192.168.10.130:2380,infra2=http://192.168.10.129:2380 \
      --initial-cluster-state new
`

 `./etcd --name infra1 --initial-advertise-peer-urls http://192.168.10.130:2380 \
      --listen-peer-urls http://192.168.10.130:2380 \
      --listen-client-urls http://192.168.10.130:2379,http://127.0.0.1:2379 \
      --advertise-client-urls http://192.168.10.130:2379 \
      --initial-cluster-token etcd-cluster-1 \
      --initial-cluster infra0=http://192.168.10.127:2380,infra1=http://192.168.10.130:2380,infra2=http://192.168.10.129:2380 \
      --initial-cluster-state new
`

`./etcd --name infra2 --initial-advertise-peer-urls http://192.168.10.129:2380 \
      --listen-peer-urls http://192.168.10.129:2380 \
      --listen-client-urls http://192.168.10.129:2379,http://127.0.0.1:2379 \
      --advertise-client-urls http://192.168.10.129:2379 \
      --initial-cluster-token etcd-cluster-1 \
      --initial-cluster infra0=http://192.168.10.127:2380,infra1=http://192.168.10.130:2380,infra2=http://192.168.10.129:2380 \
      --initial-cluster-state new
`

### Step 13   修改kubelet配置文件（每台机器都修改）
vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
把`KUBELET_CGROUP_ARGS=—cgroup-driver=systemd`改成`KUBELET_CGROUP_ARGS=—cgroup-driver=cgroupfs`  

### Step 14    重启kubelet&&并加为系统自启动
[root@hor1 k8s1.6.3]# service kubelet restart  
[root@hor1 k8s1.6.3]# systemctl daemon-reload  
[root@hor1 k8s1.6.3]# systemctl enable kubelet  
[root@hor1 k8s1.6.3]# systemctl enable docker  
 
### Step 15  修改Bridge(重启后失效,每台机器执行)，可加入到/etc/rc.d/rc.local  
echo 1  /proc/sys/net/bridge/bridge-nf-call-iptables   
echo 1  /proc/sys/net/bridge/bridge-nf-call-ip6tables

### Step 16  初始化主节点
 
[root@hor1 k8s1.6.3]# kubeadm init --config kubeadm.config   
[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.  
[init] Using Kubernetes version: v1.6.3
[init] Using Authorization mode: RBAC
[preflight] Running pre-flight checks
[preflight] Starting the kubelet service
[certificates] Generated CA certificate and key.
[certificates] Generated API server certificate and key.
[certificates] API Server serving cert is signed for DNS names [192-168-10-127.master   kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.10.127]  
[certificates] Generated API server kubelet client certificate and key.  
[certificates] Generated service account token signing key and public key.  
[certificates] Generated front-proxy CA certificate and key.  
[certificates] Generated front-proxy client certificate and key.    
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"   
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"  
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"  
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"  
[apiclient] Created API client, waiting for the control plane to become ready  
[apiclient] All control plane components are healthy after 17.544851 seconds  
[apiclient] Waiting for at least one node to register  
[apiclient] First node has registered after 4.004183 seconds  
[token] Using token: 3f877e.e592d381fbbbe358  
[apiconfig] Created RBAC rules  
[addons] Created essential addon: kube-proxy  
[addons] Created essential addon: kube-dns  
  
Your Kubernetes master has initialized successfully!  

To start using your cluster, you need to run (as a regular user):  
  
  sudo cp /etc/kubernetes/admin.conf $HOME/  
  sudo chown $(id -u):$(id -g) $HOME/admin.conf  
  export KUBECONFIG=$HOME/admin.conf  

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:  
  http://kubernetes.io/docs/admin/addons/  

You can now join any number of machines by running the following on each node  
as root:  

  kubeadm join --token 3f877e.e592d381fbbbe358 192.168.10.127:8080  



### Step 17   主机点可用
sudo cp /etc/kubernetes/admin.conf $HOME/  
 sudo chown $(id -u):$(id -g) $HOME/admin.conf  
 export KUBECONFIG=$HOME/admin.conf   

### Step 18    加入Calico网络  
kubectl apply -f beijing-calico.yml    

### Step 19   查看各组件各pod的状态    
kubectl get pods --all-namespaces  

### Step 20 加入子节点
   kubeadm join --token 3f877e.e592d381fbbbe358 192.168.10.127:8080    

### Step 21    验证dns无误  
kubectl create -f busybox.yml  
kubectl exec -it busybox -- nslookup kubernetes.default

主节点搭建完成，一个主节点，两个子节点
6/12/2017 4:57:18 PM 
----------
## 高可用节点搭建（1.6未实践，参考1.5） ##

  




