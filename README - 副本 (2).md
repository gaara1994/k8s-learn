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



## firewalld

```
systemctl disabled firewalld.service
systemctl stop firewalld.service
systemctl status firewalld.service
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

重启系统后再次查看

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



## 升级内核

```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```



```
yum -y install https://mirrors.tuna.tsinghua.edu.cn/elrepo/kernel/el7/x86_64/RPMS/elrepo-release-7.0-5.el7.elrepo.noarch.rpm
```



https://mirrors.tuna.tsinghua.edu.cn/elrepo/kernel/el7/x86_64/RPMS/elrepo-release-7.0-5.el7.elrepo.noarch.rpm

```
yum --enablerepo="elrepo-kernel" -y install kernel-lt.x86_64
```



```
 grub2-set-default 0
 grub2-mkconfig -o /boot/grub2/grub.cfg
 reboot
```





```
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF
```



```
modprobe br_netfilter
lsmod | grep br_netfilter
```



```
[root@k8s-master ~]# sysctl -p /etc/sysctl.d/k8s.conf 
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0

```



## ipvs

```
yum -y install ipset ipvsadm

```

```
vim /etc/sysconfig/modules/ipvs.modules
```

```
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
```



```
chmod 755 /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
```



```
lsmod | grep -e ip_vs -e nf_conntrack
```







```
tar -zxvf cri-containerd-1.7.17-linux-amd64.tar.gz -C /
mkdir /etc/containerd
```



```
containerd config default > /etc/containerd/config.toml
```

vim /etc/containerd/config.toml

```
sandbox_image = "registry.k8s.io/pause:3.9"
```



```
systemctl enable containerd.service
containerd --version
```



## runc



```
yum install -y gperf gcc
wget https://github.com/opencontainers/runc/releases/download/v1.1.12/libseccomp-2.5.4.tar.gz
tar -zxvf libseccomp-2.5.4.tar.gz 
cd libseccomp-2.5.4/
./configure
make
make install
```



```
[root@k8s-master libseccomp-2.5.4]# find / -name "libseccomp.so"
/root/libseccomp-2.5.4/src/.libs/libseccomp.so
/usr/local/lib/libseccomp.so
```



```
wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
chmod +x runc.amd64
```



```
[root@k8s-master ~]# which runc
/usr/local/sbin/runc
[root@k8s-master ~]# rm -rf /usr/local/sbin/runc
[root@k8s-master ~]# mv runc.amd64 /usr/local/sbin/runc
```

```
[root@k8s-master ~]# 
[root@k8s-master ~]# runc --version
runc version 1.1.12
commit: v1.1.12-0-g51d5e946
spec: 1.0.2-dev
go: go1.20.13
libseccomp: 2.5.4

```



## kube



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



vim /etc/sysconfig/kubelet

```
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"T

```

systemctl enable kubelet

现在启动不起来，等集群配置就好了。





## 7.转发 IPv4 

并让 iptables 看到桥接流量

所有机器

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```





