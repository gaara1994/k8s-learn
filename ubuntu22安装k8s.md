# 环境准备

## 1.虚拟机网卡配置

![Snipaste_2024-05-13_19-52-45](.\assets\Snipaste_2024-05-13_19-52-45.png)



![Snipaste_2024-05-13_20-06-12](.\assets\Snipaste_2024-05-13_20-01-46.png)



![Snipaste_2024-05-13_20-01-46](.\assets\Snipaste_2024-05-13_20-03-03.png)



1.取消DHCP分配IP的方式。

2.子网段设置为 `10.0.0.0`。

![Snipaste_2024-05-13_20-03-03](.\assets\Snipaste_2024-05-13_20-03-03.png)





![Snipaste_2024-05-13_20-03-35](.\assets\Snipaste_2024-05-13_20-03-35.png)

修改网关

![Snipaste_2024-05-13_20-03-57](.\assets\Snipaste_2024-05-13_20-03-57.png)



为什么网关ip是 `10.0.0.2` 呢，因为 ` 10.0.0.2` 是VMnet8的ip。

```
以太网适配器 VMware Network Adapter VMnet8:

   连接特定的 DNS 后缀 . . . . . . . :
   本地链接 IPv6 地址. . . . . . . . : fe80::fdd3:c4af:7ac5:cbbd%16
   IPv4 地址 . . . . . . . . . . . . : 10.0.0.1
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . :
```

![Snipaste_2024-05-13_20-04-26](.\assets\Snipaste_2024-05-13_20-04-26.png)



## 2.虚拟机ip规划

```shell
10.0.0.130  k8s-master
10.0.0.131  k8s-node1
```



## 3.修改静态ip

```shell
cd /etc/netplan/
sudo cp 00-installer-config.yaml 00-installer-config.yaml.bak
sudo vim 00-installer-config.yaml
```

在最下面添加如下内容

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:  # 将此更改为你的网络接口名称  
      dhcp4: no
      addresses:
        - 10.0.0.130/24  # 设置你的静态 IP 地址和子网掩码  
      routes:
        - to: default
          via: 10.0.0.2  # 网关地址
      nameservers:
          addresses: [8.8.8.8,8.8.4.4]  # 设置你的 DNS 服务器地址
```

重启网络服务

```
sudo netplan apply
```



## 5.firewalld

```shell
sudo systemctl disable ufw.service
sudo systemctl stop ufw.service
sudo systemctl status ufw.service
```



## 6.禁用swap

修改配置文件

```shell
sudo vim /etc/fstab
```

找到这一行并注释掉

```shell
#/swap.img      none    swap    sw      0       0
```



## 7.修改hostname

```shell
hostnamectl set-hostname  k8s-master
exec /bin/bash
```



## 8.添加域名

所有机器

```shell
vim /etc/hosts
```

```shell
#添加域名
10.0.0.130  k8s-master
10.0.0.131  k8s-node-1
```



## 9.添加网桥过滤和地址转发功能

```
sudo vim /etc/modules-load.d/k8s.conf
```

```shell
overlay
br_netfilter
```

加载网桥过滤器模块

```
sudo modprobe overlay
sudo modprobe br_netfilter
```

```shell
lsmod | grep br_netfilter
```

会有下面的输出

```shell
br_netfilter           22256  0 
bridge                151336  1 br_netfilter
```

设置所需的 sysctl 参数，参数在重新启动后保持不变

```shell
 sudo vim /etc/sysctl.d/k8s.conf
```

```shell
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward 
```

```shell
# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```



## 12.containerd

https://containerd.io/downloads/

```shell
wget https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
```

```shell
mkdir containerd
tar -zxf containerd-1.7.14-linux-amd64.tar.gz -C containerd
sudo cp -p containerd/bin/* /usr/local/bin/
```

```shell
containerd --version
```



开机启动

```shell
sudo vim /lib/systemd/system/containerd.service
```

```shell
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStart=/usr/local/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
```

```shell
sudo systemctl daemon-reload
sudo systemctl enable containerd
sudo systemctl start containerd
sudo systemctl status containerd
```



生成containerd 的配置文件

```shell
sudo mkdir -p /etc/containerd
#切换root
su
#这行需要root账户
sudo containerd config default > /etc/containerd/config.toml
sudo cp -p /etc/containerd/config.toml /etc/containerd/config.toml.bak
```



```shell
sudo vim /etc/containerd/config.toml
```



```shell
SystemdCgroup = false
#修改如下
SystemdCgroup = true
```



```shell
sandbox_image = "registry.k8s.io/pause:3.8"
#修改如下
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
```



```shell
sudo systemctl restart containerd
sudo systemctl status containerd
```



```shell
yantao@k8s-master:~$ sudo ctr version
Client:
  Version:  v1.7.14
  Revision: dcf2847247e18caba8dce86522029642f60fe96b
  Go version: go1.21.8

Server:
  Version:  v1.7.14
  Revision: dcf2847247e18caba8dce86522029642f60fe96b
  UUID: d7ccbb0f-c77f-4264-a14e-d41ddf939cc
```





## 13.runc

https://github.com/opencontainers/runc/releases/tag/v1.1.12

```shell
wget https://github.com/opencontainers/runc/releases/download/v1.1.12/libseccomp-2.5.4.tar.gz
```

安装编译工具

```shell
sudo apt update
sudo apt install gperf gcc make -y
```

```shell
tar -zxvf libseccomp-2.5.4.tar.gz
cd libseccomp-2.5.4/
./configure
sudo make && sudo make install
```



```shell
cd
wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo chmod +x runc.amd64
sudo mv runc.amd64 /usr/local/sbin/runc

```

```shell
runc --version
```





## 14.kube

安装

```
su
```

```shell
apt-get update && apt-get install -y apt-transport-https
curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl

```

修改配置文件

```shell
vim /etc/sysconfig/kubelet
```

```shell
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
```

```shell
systemctl enable kubelet && systemctl start kubelet && systemctl status kubelet
```



```shell
[root@k8s-master ~]# kubeadm config images list
I0520 11:46:21.947549    5792 version.go:256] remote version is much newer: v1.30.1; falling back to: stable-1.28
registry.k8s.io/kube-apiserver:v1.28.10
registry.k8s.io/kube-controller-manager:v1.28.10
registry.k8s.io/kube-scheduler:v1.28.10
registry.k8s.io/kube-proxy:v1.28.10
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.12-0
registry.k8s.io/coredns/coredns:v1.10.1

```



## 15.init

**master机器**

```shell
kubeadm init \
--apiserver-advertise-address=10.0.0.130 \
--kubernetes-version=v1.28.10 \
--pod-network-cidr=10.244.0.0/16 \
--service-cidr=10.254.0.0/16 \
--image-repository registry.aliyuncs.com/google_containers
```



失败的话执行 kubeadm reset --force



下面的初始化成功了

```shell
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

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

kubeadm join 10.0.0.130:6443 --token 3wdes5.q0m3rvt65551m6ee \
	--discovery-token-ca-cert-hash sha256:0f7339facb4fd22a1bbaf3913d2cbcb436f72a4e7b0ccd2c612d380e1be762dc 
```



未完待续

## 16.calico

安装 Tigera Calico 运算符和自定义资源定义。（通过创建必要的自定义资源来安装 Calico）

```shell
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml -O
kubectl apply -f tigera-operator.yaml
```

通过自定义资源方式安装

```shell
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/custom-resources.yaml -O
```

```shell
vim custom-resources.yaml
```

修改为--pod-network-cidr的值

```shell
cidr: 10.244.0.0/16
```

```shell
kubectl apply -f custom-resources.yaml
```

等所有的都是running

```shell
watch kubectl get pods -n calico-system
```



## 17.worker

```shell
kubeadm join 10.0.0.128:6443 --token 9jhj9d.97pb9w09n2mx5p3v \
	--discovery-token-ca-cert-hash sha256:59cfa3a5f42963cecac6eea6b8d269c9be5dd3e0a4bb89569e96867d121d123b 
```



如果失败

```shell
kubeadm reset
```



## 删除节点

master

```shell
kubectl get nodes
```



```shell
kubectl drain k8s-node-1 --delete-emptydir-data --force --ignore-daemonsets --grace-period=30

kubectl delete node k8s-node-1
```





## 部署测试

1.deployment

```shell
vim nginx-deployment.yaml
```

```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
        image: nginx:1.18-alpine
        ports:
        - containerPort: 80
```

```shell
kubectl apply -f nginx-deployment.yam
```



2.deployment

```shell
vim nginx-service.yaml
```

```shell
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

```shell
kubectl apply -f nginx-service.yaml
```

3.验证

```shell
[root@k8s-master ~]# kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           2m31s
[root@k8s-master ~]# kubectl get services
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP      10.254.0.1      <none>        443/TCP        70m
nginx-service   LoadBalancer   10.254.66.200   <pending>     80:30226/TCP   2m5sxxxxxxxxxx [root@k8s-master ~]# kubectl get deploymentsNAME               READY   UP-TO-DATE   AVAILABLE   AGEnginx-deployment   2/2     2            2           2m31s[root@k8s-master ~]# kubectl get servicesNAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGEkubernetes      ClusterIP      10.254.0.1      <none>        443/TCP        70mnginx-service   LoadBalancer   10.254.66.200   <pending>     80:30226/TCP   2m5skubectl get deploymentskubectl get services
```



http://10.0.0.128:30226/
