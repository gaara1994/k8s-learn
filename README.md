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





## 4.跨主机免密码认证

生成密钥对

```shell
ssh-keygen -t rsa
```

跨主机认证，在master执行。

```shell
for i in 101 102 103
do 
ssh-copy-id root@10.0.0.$i
done
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





## 8. 安装docker-ce

```shell
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3
sudo sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
# Step 4: 更新并安装Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start

# 注意：
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，您可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ce.repo
#   将[docker-ce-test]下方的enabled=0修改为enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]

```



镜像加速，修改驱动为systemd，添加insecure-registries

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://7i36h8m3.mirror.aliyuncs.com"],
  "insecure-registries":["harbor.k8s.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```





 vim /etc/containerd/config.toml

disabled_plugins = []



## 10. harbor

下载

[Releases · goharbor/harbor (github.com)](https://github.com/goharbor/harbor/releases)

```
tar xvf harbor-offline-installer-v2.9.4.tgz
```

安装

```shell

[root@k8s-harbor ~]# tar xvf harbor-offline-installer-v2.9.4.tgz
harbor/harbor.v2.9.4.tar.gz
harbor/prepare
harbor/LICENSE
harbor/install.sh
harbor/common.sh
harbor/harbor.yml.tmpl
[root@k8s-harbor ~]# cd harbor/
[root@k8s-harbor harbor]# cp harbor.yml.tmpl harbor.yml
[root@k8s-harbor harbor]# vim harbor.yml

```

修改以下部分

```
hostname: harbor.k8s.com



#https:
  # https port for harbor, default is 443
#  port: 443
  # The path of cert and key files for nginx
#  certificate: /your/certificate/path
#  private_key: /your/private/key/path

```

![image-20240518135038642](.\assets\image-20240518135038642.png)



在配置文件有看到账号密码

admin

Harbor12345

安装

```shell
[root@k8s-harbor harbor]# ./install.sh
```

```shell

[Step 5]: starting Harbor ...
WARN[0000] /root/harbor/docker-compose.yml: `version` is obsolete
[+] Running 10/10
 ✔ Network harbor_harbor        Created                                                                                                      1.2s
 ✔ Container harbor-log         Started                                                                                                      1.3s
 ✔ Container registryctl        Started                                                                                                      3.5s
 ✔ Container harbor-db          Started                                                                                                      3.5s
 ✔ Container redis              Started                                                                                                      3.6s
 ✔ Container harbor-portal      Started                                                                                                      3.2s
 ✔ Container registry           Started                                                                                                      3.2s
 ✔ Container harbor-core        Started                                                                                                      4.3s
 ✔ Container harbor-jobservice  Started                                                                                                      6.0s
 ✔ Container nginx              Started                                                                                                      6.2s
✔ ----Harbor has been installed and started successfully.----

```



打开网址： http://10.0.0.103/



新建一个项目

![image-20240518135228205](.\assets\image-20240518135228205.png)



## harbor测试

在master上登陆harbor

```shell
[root@k8s-master ~]# docker login harbor.k8s.com
Username: admin
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@k8s-master ~]#
```



```shell
#下载测试镜像
[root@k8s-master ~]# docker tag hello-world:latest  harbor.k8s.com/k8s-image/hello-world:latest
#推送
[root@k8s-master ~]# docker push harbor.k8s.com/k8s-image/hello-world:latest
The push refers to repository [harbor.k8s.com/k8s-image/hello-world]
e07ee1baac5f: Pushed
latest: digest: sha256:f54a58bc1aac5ea1a25d796ae155dc228b3f0e11d046ae276b39c4bf2f13d8c4 size: 525

```



node-1下载测试

```shell
[root@k8s-node1 ~]# docker pull harbor.k8s.com/k8s-image/hello-world:latest
latest: Pulling from k8s-image/hello-world
2db29710123e: Pull complete
Digest: sha256:f54a58bc1aac5ea1a25d796ae155dc228b3f0e11d046ae276b39c4bf2f13d8c4
Status: Downloaded newer image for harbor.k8s.com/k8s-image/hello-world:latest
harbor.k8s.com/k8s-image/hello-world:latest

```







## 安装三个

除了 harbor

```shell
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

查看版本

```shell

[root@k8s-master ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"28", GitVersion:"v1.28.10", GitCommit:"21be1d76a90bc00e2b0f6676a664bdf097224155", GitTreeState:"clean", BuildDate:"2024-05-14T10:51:30Z", GoVersion:"go1.21.9", Compiler:"gc", Platform:"linux/amd64"}

```

查看需要的镜像列表

```
[root@k8s-master ~]# kubeadm config images list
registry.k8s.io/kube-apiserver:v1.28.10
registry.k8s.io/kube-controller-manager:v1.28.10
registry.k8s.io/kube-scheduler:v1.28.10
registry.k8s.io/kube-proxy:v1.28.10
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.12-0
registry.k8s.io/coredns/coredns:v1.10.1

```



编写脚本下载镜像，并推送到私有仓库

```shell
# 获取指定 Kubernetes 版本的镜像列表
images=$(kubeadm config images list --kubernetes-version=v1.28.10 | awk -F '/' '{print $NF}')

# 遍历镜像列表
for image in $images; do
    # 下载原始镜像
    echo "Downloading $image from registry.aliyuncs.com..."
    docker pull registry.aliyuncs.com/google_containers/$image

    # 构建新的镜像名（假设私有仓库地址正确）
    new_image=harbor.k8s.com/k8s-image/$image

    # 为镜像打新标签
    echo "Tagging as $new_image..."
    docker tag registry.aliyuncs.com/google_containers/$image $new_image

    # 推送镜像到私有仓库
    echo "Pushing $new_image to private registry..."
    docker push $new_image

    # 删除原始镜像（可选，取决于你的磁盘空间和需求）
    echo "Removing original image registry.aliyuncs.com/google_containers/$image..."
    docker rmi registry.aliyuncs.com/google_containers/$image
done

echo "All images processed."
```



![image-20240518202429897](.\assets\image-20240518202429897.png)





## kubeadmin 初始化

查看帮助命令

```shell
kubeadm init --help

```



```shell
......
Flags:
      --apiserver-advertise-address string   The IP address the API Server will advertise it's listening on. If not set the default network interface will be used.
      --apiserver-bind-port int32            Port for the API Server to bind to. (default 6443)
      --apiserver-cert-extra-sans strings    Optional extra Subject Alternative Names (SANs) to use for the API Server serving certificate. Can be both IP addresses and DNS names.
      --cert-dir string                      The path where to save and store the certificates. (default "/etc/kubernetes/pki")
      --certificate-key string               Key used to encrypt the control-plane certificates in the kubeadm-certs Secret.
      --config string                        Path to a kubeadm configuration file.
      --control-plane-endpoint string        Specify a stable IP address or DNS name for the control plane.
      --cri-socket string                    Path to the CRI socket to connect. If empty kubeadm will try to auto-detect this value; use this option only if you have more than one CRI installed or if you have non-standard CRI socket.
      --dry-run                              Don't apply any changes; just output what would be done.
      --feature-gates string                 A set of key=value pairs that describe feature gates for various features. Options are:
                                             EtcdLearnerMode=true|false (ALPHA - default=false)
                                             PublicKeysECDSA=true|false (ALPHA - default=false)
                                             RootlessControlPlane=true|false (ALPHA - default=false)
                                             UpgradeAddonsBeforeControlPlane=true|false (DEPRECATED - default=false)
  -h, --help                                 help for init
      --ignore-preflight-errors strings      A list of checks whose errors will be shown as warnings. Example: 'IsPrivilegedUser,Swap'. Value 'all' ignores errors from all checks.
      --image-repository string              Choose a container registry to pull control plane images from (default "registry.k8s.io")
      --kubernetes-version string            Choose a specific Kubernetes version for the control plane. (default "stable-1")
      --node-name string                     Specify the node name.
      --patches string                       Path to a directory that contains files named "target[suffix][+patchtype].extension". For example, "kube-apiserver0+merge.yaml" or just "etcd.json". "target" can be one of "kube-apiserver", "kube-controller-manager", "kube-scheduler", "etcd", "kubeletconfiguration". "patchtype" can be one of "strategic", "merge" or "json" and they match the patch formats supported by kubectl. The default "patchtype" is "strategic". "extension" must bec either "json" or "yaml". "suffix" is an optional string that can be used to determine which patches are applied first alpha-numerically.
      --pod-network-cidr string              Specify range of IP addresses for the pod network. If set, the control plane will automatically allocate CIDRs for every node.
      --service-cidr string                  Use alternative range of IP address for service VIPs. (default "10.96.0.0/12")
      --service-dns-domain string            Use alternative domain for services, e.g. "myorg.internal". (default "cluster.local")
      --skip-certificate-key-print           Don't print the key used to encrypt the control-plane certificates.
      --skip-phases strings                  List of phases to be skipped
      --skip-token-print                     Skip printing of the default bootstrap token generated by 'kubeadm init'.
      --token string                         The token to use for establishing bidirectional trust between nodes and control-plane nodes. The format is [a-z0-9]{6}\.[a-z0-9]{16} - e.g. abcdef.0123456789abcdef
      --token-ttl duration                   The duration before the token is automatically deleted (e.g. 1s, 2m, 3h). If set to '0', the token will never expire (default 24h0m0s)
      --upload-certs                         Upload control-plane certificates to the kubeadm-certs Secret.

Global Flags:
      --add-dir-header           If true, adds the file directory to the header of the log messages
      --log-file string          If non-empty, use this log file (no effect when -logtostderr=true)
      --log-file-max-size uint   Defines the maximum size a log file can grow to (no effect when -logtostderr=true). Unit is megabytes. If the value is 0, the maximum file size is unlimited. (default 1800)
      --one-output               If true, only write logs to their native severity level (vs also writing to each lower severity level; no effect when -logtostderr=true)
      --rootfs string            [EXPERIMENTAL] The path to the 'real' host root filesystem.
      --skip-headers             If true, avoid header prefixes in the log messages
      --skip-log-headers         If true, avoid headers when opening log files (no effect when -logtostderr=true)
  -v, --v Level                  number for the log level verbosity

Use "kubeadm init [command] --help" for more information about a command.

```



```shell
kubeadm init --kubernetes-version=1.28.10 \
--apiserver-advertise-address=10.0.0.100 
--image-repository=harbor.k8s.com/k8s-image/ \
--pod-network-cidr="10.244.0.0/16" \
--service-cidr="10.96.0.0/12"  \
--ignore-preflight-errors=Swap  \
--cri-socket=unix:///var/run/cri-dockerd.sock\
```



```
kubeadm init --kubernetes-version=1.28.10 --apiserver-advertise-address=10.0.0.100 
--image-repository=harbor.k8s.com/k8s-image --pod-network-cidr="10.244.0.0/16" 
--service-cidr="10.96.0.0/12"  --ignore-preflight-errors=Swap  
--cri-socket=unix:///var/run/containerd/containerd.sock
```



```
kubeadm init --kubernetes-version=1.28.10 --apiserver-advertise-address="10.0.0.100" --image-repository=harbor.k8s.com/k8s-image --pod-network-cidr="10.244.0.0/16" --service-cidr="10.96.0.0/12" --ignore-preflight-errors=swap --cri-socket=unix:///var/run/containerd/containerd.sock

```



```
kubeadm init --kubernetes-version=1.28.10 
--apiserver-advertise-address=10.0.0.100  
--pod-network-cidr="10.244.0.0/16" 
--service-cidr="10.96.0.0/12"  
--ignore-preflight-errors=Swap  
--cri-socket=unix:///var/run/containerd/containerd.sock
--image-repository=registry.k8s.io
```

```
kubeadm init --kubernetes-version=1.28.10 --apiserver-advertise-address="10.0.0.100"  --pod-network-cidr="10.244.0.0/16" --service-cidr="10.96.0.0/12" --ignore-preflight-errors=Swap --cri-socket=unix:///var/run/containerd/containerd.sock --image-repository=harbor.k8s.com/k8s-image
```

