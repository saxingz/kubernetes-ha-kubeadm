# 一、实践环境准备
## 1. 服务器说明
我们这里使用的是五台centos-7.6的虚拟机，具体信息如下表：

| 系统类型 | IP地址 | 节点角色 | CPU | Memory | Hostname |
| :------: | :--------: | :-------: | :-----: | :---------: | :-----: |
| centos-7.8 | 172.16.249.130 | master |   \>=2    | \>=2G | m1 |
| centos-7.8 | 172.16.249.131 | master |   \>=2    | \>=2G | m2 |
| centos-7.8 | 172.16.249.132 | master |   \>=2    | \>=2G | m3 |
| centos-7.8 | 172.16.249.135 | worker |   \>=2    | \>=2G | s1 |
| centos-7.8 | 172.16.249.136 | worker |   \>=2    | \>=2G | s2 |

## 2. 系统设置（所有节点）
#### 2.1 主机名
主机名必须每个节点都不一样，并且保证所有点之间可以通过hostname互相访问。
```bash
# 查看主机名
$ hostname
# 修改主机名
$ hostnamectl set-hostname <your_hostname>
# 配置host，使所有节点之间可以通过hostname互相访问
$ vi /etc/hosts
# <node-ip> <node-hostname>
```
#### 2.2 安装依赖包
```bash
# 更新yum
$ yum update
# 安装依赖包
$ yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp
```
#### 2.3 关闭防火墙、swap，重置iptables
```bash
# 关闭防火墙
$ systemctl stop firewalld && systemctl disable firewalld
# 重置iptables
$ iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
# 关闭swap
$ swapoff -a
# 使用下面的命令对文件/etc/fstab操作，注释 /dev/mapper/centos_master-swap  swap  swap    defaults        0 0 这行
$ sed -i 's/.*swap.*/#&/' /etc/fstab
# 关闭selinux
$ vim /etc/selinux/config 
# 将SELINUX=enforcing改为SELINUX=disabled
$ setenforce 0
# 查看selinux状态
$ sestatus
# 关闭dnsmasq(否则可能导致docker容器无法解析域名)
$ service dnsmasq stop && systemctl disable dnsmasq
```
#### 2.4 系统参数设置

```bash
# 制作配置文件
$ cat > /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
EOF
# 生效文件
$ sysctl -p /etc/sysctl.d/kubernetes.conf
# 执行sysctl -p 时出现下面的错误
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: No such file or directory
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory
# 解决方法：运行命令 modprobe br_netfilter 然后再执行 sysctl -p /etc/sysctl.d/kubernetes.conf
$ modprobe br_netfilter
# 查看
$ ls /proc/sys/net/bridge
bridge-nf-call-arptables bridge-nf-filter-pppoe-tagged
bridge-nf-call-ip6tables bridge-nf-filter-vlan-tagged
bridge-nf-call-iptables bridge-nf-pass-vlan-input-dev
```
## 3. 安装docker（所有节点）
```bash
# 1. 方法一: 通过yum源的方式安装
# 创建所需目录
$ mkdir -p /opt/kubernetes/docker && cd /opt/kubernetes/docker
# 清理原有版本, 如果系统没有安装过跳过
$ yum remove -y docker* container-selinux
# 安装依赖包
$ yum install -y yum-utils device-mapper-persistent-data lvm2
# 设置yum源
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# 可以查看所有仓库中所有docker版本，并选择特定版本安装：
$ yum list docker-ce --showduplicates | sort -r
# 安装指定版本docker, 如果不指定版本号，将安装最新版本的docker
$ sudo yum install -y docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
# 示例-安装docker版本是: 19.03.8
$ yum install -y docker-ce-19.03.8 docker-ce-cli-19.03.8 containerd.io
# 开机启动
$ systemctl enable docker && systemctl start docker

# 2. 方法二: 通过rpm方式安装
# 手动下载rpm包【示例docker 19.03.8版本的安装】
$ wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.13-3.2.el7.x86_64.rpm
$ wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-19.03.8-3.el7.x86_64.rpm
$ wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-cli-19.03.8-3.el7.x86_64.rpm
# 安装rpm包
$ yum localinstall -y *.rpm
# 开机启动
$ systemctl enable docker && systemctl start docker


# 设置参数
# 1.查看磁盘挂载
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        98G  2.8G   95G   3% /
devtmpfs         63G     0   63G   0% /dev
/dev/sda5      1015G  8.8G 1006G   1% /tol
/dev/sda1       197M  161M   37M  82% /boot
# 2.设置docker启动参数
# - 设置docker数据目录：选择比较大的分区（我这里是根目录就不需要配置了，默认为/var/lib/docker）
# - 设置cgroup driver, 防止文件驱动不一致，导致镜像无法启动（默认是cgroupfs，主要目的是与kubelet配置统一，这里也可以不设置后面在kubelet中指定cgroupfs）
# docker info 可以查看到cgroup driver的类型
$ cat <<EOF > /etc/docker/daemon.json
{
    "graph": "/docker/data/path",
    "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
# 启动docker服务
systemctl start docker

# daemon.json 详细配置示例
{
  "debug": false,
  "experimental": false,
  "graph": "/home/docker-data",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
    "https://fy707np5.mirror.aliyuncs.com"
  ],
  "insecure-registries": [
    "hub.zy.com",
    "172.16.249.159:8082"
  ]
}
# 启动docker服务
systemctl restart docker
```

## 4. 安装必要工具（所有节点）
#### 4.1 工具说明
- **kubeadm:**  部署集群用的命令
- **kubelet:** 在集群中每台机器上都要运行的组件，负责管理pod、容器的生命周期
- **kubectl:** 集群管理工具（可选，只要在控制集群的节点上安装即可）

#### 4.2 安装方法

```bash
# 配置yum源（科学上网的同学可以把"mirrors.aliyun.com"替换为"packages.cloud.google.com"）
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装工具
# 找到要安装的版本号
$ yum list kubeadm --showduplicates | sort -r

# 安装指定版本（这里用的是1.18.4）
$ yum install -y kubeadm-1.18.4-0 kubelet-1.18.4-0 kubectl-1.18.4-0 --disableexcludes=kubernetes

# 设置kubelet的cgroupdriver（kubelet的cgroupdriver默认为systemd，如果上面没有设置docker的exec-opts为systemd，这里就需要将kubelet的设置为cgroupfs）
$ sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# 启动kubelet 所有的节点
$ systemctl enable kubelet && systemctl start kubelet

```


## 5. 准备配置文件（任意节点）
#### 5.1 下载配置文件
我这准备了一个项目，专门为大家按照自己的环境生成配置的。它只是帮助大家尽量的减少了机械化的重复工作。它并不会帮你设置系统环境，不会给你安装软件。总之就是会减少你的部署工作量，但不会耽误你对整个系统的认识和把控。
```bash
$ cd ~ && git clone https://gitee.com/salmon_163/kubernetes-ha-kubeadm.git
# 看看git内容
$ ls -l kubernetes-ha-kubeadm
addons/
configs/
scripts/
init.sh
global-configs.properties
```
#### 5.2 文件说明
- **addons**
> kubernetes的插件，比如calico和dashboard。

- **configs**
> 包含了部署集群过程中用到的各种配置文件。

- **scripts**
> 包含部署集群过程中用到的脚本，如keepalive检查脚本。

- **global-configs.properties**
> 全局配置，包含各种易变的配置内容。

- **init.sh**
> 初始化脚本，配置好global-config之后，会自动生成所有配置文件。

#### 5.3 生成配置
这里会根据大家各自的环境生成kubernetes部署过程需要的配置文件。
> 此脚本不支持MACOS系统，切记不要使用MACOS来运行 init.sh
```bash
# cd到之前下载的git代码目录
$ cd kubernetes-ha-kubeadm

# 编辑属性配置（根据文件注释中的说明填写好每个key-value）
$ vi global-config.properties

# 生成配置文件，确保执行过程没有异常信息
$ ./init.sh

# 查看生成的配置文件，确保脚本执行成功
$ find target/ -type f
```
> **执行init.sh常见问题：**
> 1. Syntax error: "(" unexpected
> - bash版本过低，运行：bash -version查看版本，如果小于4需要升级
> - 不要使用 sh init.sh的方式运行（sh和bash可能不一样哦）
> 2. global-config.properties文件填写错误，需要重新生成  
> 再执行一次./init.sh即可，不需要手动删除target

```

```