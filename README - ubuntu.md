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
10.0.0.130  master.k8s.com  k8s-master
10.0.0.131  node1.k8s.com   k8s-node1
10.0.0.132  node2.k8s.com   k8s-node2
10.0.0.133  harbor.k8s.com  k8s-harbor
```



## 3.修改静态ip

```shell
cp /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.yaml.bak

vim /etc/netplan/00-installer-config.yaml
```

在最下面添加如下内容

```yaml
network:  
  version: 2  
  ethernets:  
    ens33:  
      dhcp4: no  
      addresses:  
        - 10.0.0.130/24  
      routes:  
        - to: default  
          via: 10.0.0.2  
      nameservers:  
        addresses: [8.8.8.8, 114.114.114.114]
```

重启网络服务

```shell
netplan apply
```

四台机器同样的方式修改为静态ip。





## 5.修改hostname

```shell
hostnamectl set-hostname  k8s-master
exec /bin/bash
```

其他的也依次修改



## 1.关闭防火墙

```
systemctl stop ufw.service
systemctl disable ufw.service
```





## 2.禁用swap

修改配置文件

```shell
vim /etc/fstab
```

找到这一行并注释掉

```shell
#/swap.img      none    swap    sw      0       0
```

reboot 重启系统后再次查看

```shell
root@k8s-master:~# free -m
              total        used        free      shared  buff/cache   available
Mem:           7927         282        7259           1         384        7401
Swap:             0           0           0
```









## 6.添加域名

所有机器

```
vim /etc/hosts
```

```shell
...
#添加域名
10.0.0.130   master.k8s.com  k8s-master
10.0.0.131   node-1.k8s.com  k8s-node-1
10.0.0.132   node-1.k8s.com  k8s-node-1
10.0.0.133   harbor.k8s.com  k8s-harbor
```



## 时区

```shell
root@k8s-master:~# date
Sun 19 May 2024 09:40:41 AM UTC
root@k8s-master:~# timedatectl set-timezone Asia/Shanghai
root@k8s-master:~# date
Sun 19 May 2024 05:41:59 PM CST
```



## 配置内核转发

```shell
#创建加载内核模块文件
cat << EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

```



```
#本次执行,手动加载此模块
modprobe overlay
modprobe br_netfilter

```



```
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```





