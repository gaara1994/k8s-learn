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
10.0.0.100  master.k8s.com  k8s-master
10.0.0.101  node1.k8s.com   k8s-node1
10.0.0.102  node2.k8s.com   k8s-node2
10.0.0.103  harbor.k8s.com  k8s-harbor
```



## 3.修改静态ip

```shell
vim /etc/sysconfig/network-scripts/ifcfg-ens33
```

在最下面添加如下内容

```shell
#修改
ONBOOT=yes
#添加
BOOTPROTO="static"
IPADDR=10.0.0.100  # 您的静态 IP 地址
NETMASK=255.255.255.0  # 子网掩码
GATEWAY=10.0.0.2   # 默认网关（请根据您的网络设置进行更改）
DNS1=8.8.8.8           # DNS 服务器地址（您可以指定一个或多个）
DNS2=8.8.4.4
```

重启网络服务

```shell
systemctl restart network.service	
```

四台机器同样的方式修改为静态ip。



## 1.关闭selinux

所有机器

```shell
vim /etc/selinux/config
```

```shell
SELINUX=enforcing
#修改为
SELINUX=disabled
```



## 1.关闭防火墙

```
systemctl stop firewalld.service 
systemctl disable firewalld.service
```





## 2.禁用swap

所有机器

```shell
[root@localhost ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:           4056         346        3486          11         223        3481
Swap:          4095           0        4095
```

修改配置文件

```shell
vim /etc/fstab
```

找到这一行并注释掉

```shell
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

reboot 重启系统后再次查看

```shell
[root@localhost ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:           4056         318        3537          11         200        3512
Swap:             0           0           0
```





## 5.修改hostname

```shell
[root@localhost ~]# hostnamectl set-hostname  k8s-master
[root@localhost ~]# exec /bin/bash
[root@k8s-master ~]#
```

其他的也依次修改





## 6.添加域名

所有机器

```
vim /etc/hosts
```

```shell
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
#添加域名
10.0.0.100   master.k8s.com  k8s-master
10.0.0.101   node-1.k8s.com  k8s-node-1
10.0.0.102   node-1.k8s.com  k8s-node-1
10.0.0.103   harbor.k8s.com  k8s-harbor
```



```
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3 安装
sudo yum -y install containerd.io cri-tools

#Step 4 启动 containerd服务并开机配置自启动
systemctl enable containerd.service
systemctl start containerd.service 
systemctl status containerd.service

```



```
cat >/etc/containerd/config.toml <<EOF
disabled_plugins = ["restart"]
[plugins.linux]
shim_debug = true
[plugins.cri.registry.mirrors."docker.io"]
endpoint = ["https://frz7i079.mirror.aliyuncs.com"]
[plugins.cri]
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.2"
EOF

```

```
systemctl restart containerd.service 
systemctl status containerd.service 
```



```

#配置 containerd网络 配置
cat >/etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

```





```
#配置k8s网络配置
cat >/etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
#加载 overlay br_netfilter模块
modprobe overlay
modprobe br_netfilter

```





k8s源



```
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/repodata/repomd.xml.key
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet

```

