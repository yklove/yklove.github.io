---
title: 使用 kubeadm 创建一个单主集群
date: 2019-04-04 14:42:42
categories:
- Kubernetes
tags:
- Kubernetes
- k8s
- linux
- docker
---

接触到的一个项目会用到 [kubernetes](https://kubernetes.io/) ，但是没有一个良好的 kubernetes 环境可以供开发使用，只能自己动手搭一套了。

<!-- more -->

## 开始搭建

### 环境说明
- CentOS Linux release 7.5.1804 (Core)

### 安装 shadowsocks、proxychains4 解决网络问题

``` bash
pip install shadowsocks
git clone https://github.com/rofl0r/proxychains-ng.git
cd proxychains-ng
./configure
make && make install
make install-config
```
修改配置文件
``` bash
vi /etc/shadowsocks.json
添加以下内容，服务器自行搭建或者寻找
{
  "server":"xx.xx.xx.xx",
  "server_port":xxx,              
  "local_address": "127.0.0.1",
  "local_port":1080,             
  "password":"xxx",        
  "timeout":300,                
  "method":"aes-256-cfb",    
  "workers": 1              
}
vi /usr/local/etc/proxychains.conf
# 最后一行修改成
socks5 	127.0.0.1 1080
```
启动shadowsock客户端
``` bash
nohup sslocal -c /etc/shadowsocks.json /dev/null 2>&1 &
# 测试是否正常（返回的 json 中显示的是 shadowsocks 服务器地址）
curl --socks5 127.0.0.1:1080 http://httpbin.org/ip
# 测试 proxychains4，能正常连接
proxychains4 curl https://www.google.com
```

### 安装 kubeadm、kubelet、kubectl
首先添加 Kubernetes 仓库
``` bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
```
接下来关闭selinux防火墙
``` bash
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
接下来安装上面提到的三个软件
``` bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
无法安装成功，会提示以下信息，因为某些原因导致网络不通。
``` bash
https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64/repodata/repomd.xml: [Errno 14] curl#7 - "Failed to connect to 2404:6800:4005:807::200e: 网络不可达"
正在尝试其它镜像
```
使用 proxychains4 执行安装命令
``` bash
proxychains4 yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
耐心等待安装完成…
![avatar](/uploads/WX20190404-154303.png)
安装完成！
启动 kubelet
``` bash
systemctl enable --now kubelet
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### 使用 kubeadm 初始化

> 镜像下载方法及脚本来源于 https://blog.51cto.com/purplegrape/2315451

由于 kubeadm 初始化过程中会下载一部分 docker 镜像，而这些镜像也是无法直接下载的
首先查看需要使用哪些镜像
``` bash
kubeadm config images list
# 输出如下结果
k8s.gcr.io/kube-apiserver:v1.14.0
k8s.gcr.io/kube-controller-manager:v1.14.0
k8s.gcr.io/kube-scheduler:v1.14.0
k8s.gcr.io/kube-proxy:v1.14.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```
我们通过 docker.io/mirrorgooglecontainers 下载再转换 docker 镜像标签
批量下载及转换标签脚本如下
``` bash
kubeadm config images list |sed -e 's/^/docker pull /g' -e 's#k8s.gcr.io#docker.io/mirrorgooglecontainers#g' |sh -x
docker images |grep mirrorgooglecontainers |awk '{print "docker tag ",$1":"$2,$1":"$2}' |sed -e 's#mirrorgooglecontainers#k8s.gcr.io#2' |sh -x
docker images |grep mirrorgooglecontainers |awk '{print "docker rmi ", $1":"$2}' |sh -x
docker pull coredns/coredns:1.3.1
docker tag coredns/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1
docker rmi coredns/coredns:1.3.1
```
执行完成之后查看 docker images 应该会存在以下镜像
![avatar](/uploads/WX20190404-162805.png)
接下来执行初始化命令
``` bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```
初始化成功，需要保存好日志中输出的秘钥
![avatar](/uploads/WX20190404-163126.png)
接下来执行：
``` bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
到这里就算是安装好了，但是执行 `kubectl get nodes` 会发现节点状态并不是 `Ready` 而是 `NotReady`
执行 `systemctl status kubelet -l`查找原因：
``` bash
4月 04 16:38:58 21.124 kubelet[15411]: W0404 16:38:58.960008   15411 cni.go:213] Unable to update cni config: No networks found in /etc/cni/net.d
4月 04 16:38:59 21.124 kubelet[15411]: E0404 16:38:59.238355   15411 kubelet.go:2170] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```
因为kubelet配置了network-plugin=cni，但是还没安装，所以状态会是NotReady。接下来安装它：
``` bash
wget https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml 
mkdir -p /etc/cni/net.d
vi /etc/cni/net.d/10-flannel.conflist
# 写入以下内容：
{
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```
最后执行
``` bash
systemctl daemon-reload
systemctl restart kubelet
```
此时查看节点状态是正常的了
![avatar](/uploads/WX20190404-164424.png)

到这里 Kubernetes 就算是安装完成了。

### 安装 Kubernetes web界面
> 参考https://www.cnblogs.com/harlanzhang/p/10045975.html

另外如果要运行与 master 节点上需要执行 
`kubectl taint nodes --all node-role.kubernetes.io/master-`
否则 pod 会一直 pending