---
title: Kubeadm 搭建 Kubernetes 集群
date: 2017-06-15 10:27:15
tags: [Docker,应用编排,Kubernetes,Systemd,Flannel]
categories: 云平台
---
### Overview
![](monitor.jpeg)

本文简单介绍如何利用 Kubeadm 搭建 Kubernetes 1.6.4 集群的方法，网络方案采用Flannel(Overlay)。不对原理和架构有过多阐述和讲解。

废话少说，立马开始！

### 环境
四台机器（CentOS7.1）
```txt
192.168.80.23  (master)
192.168.80.24
192.168.80.25
192.168.80.26
```
Docker 版本: 1.12.3
Kubernetes 版本: 1.6.4

**PS：确保服务器 yum 安装器可以科学上网**

### Systemd: 系统级进程管理工具，管理 Docker 与 Kubelet
这里不管是 Docker 也好，Kubelet 也罢，都是通过 Systemd 来管理进程的，因此有必要对此做一定的了解。

![systemd 底层架构](systemd.png)

Systemd 取代了之前 Centos6 init.d 的进程管理方式，然而 Systemd 并不是一个命令，而是一组命令的集合，通过这个集合我们可以对我们的进程进行全方位的控制。当然我们不仅仅可以用 Systemd 管理 docker，我们还可以管理 docker 的容器，但这里不做过多介绍。
* systemctl: 主命令，管理系统和服务
* systemd-analyze: 查看启动耗时
* hostnamectl: 查看主机信息
* journalctl: 查看服务日志工具

可以参考这篇文章 http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html

### 安装 Docker （所有节点均要安装）
在[官网的 rpm 仓库](https://yum.dockerproject.org/repo/main/centos/7/Packages/)中找到要下载的 docker 版本 ，然后下载下来并安装
```txt
wget https://yum.dockerproject.org/repo/main/centos/7/Packages/docker-engine-1.12.3-1.el7.centos.x86_64.rpm

yum install docker-engine-1.12.3-1.el7.centos.x86_64.rpm
```
通过 Systemd 启动 docker
```txt
systemctl enable docker
systemctl start docker
```
可以看到 docker 已经安装完毕
```txt
[root@Node1 ~]# docker version
Client:
 Version:      1.12.3
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   6b644ec
 Built:
 OS/Arch:      linux/amd64

Server:
 Version:      1.12.3
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   6b644ec
 Built:
 OS/Arch:      linux/amd64
```

### 安装 Kubectl 客户端工具 （所有节点均要安装）
```txt
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.6.4/bin/linux/amd64/kubectl

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl
```
安装好就放着，不管先。

### 安装 Kubeadm （所有节点均要安装）

```txt
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm kubernetes-cni
systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet
```
这个时候如果科学上网设置正确，就可以顺利完成。

### Kubeadm 初始化集群 （Master 节点）

##### Docker 代理设置

上面 yum 已经把要的包安装完了，可以将 yum 的科学上网代理删去，但是需要进一步配置 docker 的科学上网代理，配置方法很简单
```txt
vim /etc/systemd/system/docker.service.d/http-proxy.conf

[Service]
Environment="HTTP_PROXY=http://192.168.80.15:8118/" "NO_PROXY=localhost,127.0.0.1,docker.jcing.com,docker-registry.bluemoon.com.cn"
```
这里注意要过滤掉本地配置的 Docker 私有镜像库的域名或者 IP 地址，否则私有库用不了。

接着
```txt
systemctl daemon-reload
systemctl restart docker
```
##### 预下载镜像，加快初始化进程

这里列举镜像列表

镜像  | 版本 | 组件
------------- | -------------
gcr.io/google_containers/kube-proxy-amd64 | v1.6.4 | Kubernetes
gcr.io/google_containers/kube-controller-manager-amd64 | v1.6.4 | Kubernetes
gcr.io/google_containers/kube-apiserver-amd64 | v1.6.4 | Kubernetes
gcr.io/google_containers/kube-scheduler-amd64 | v1.6.4 | Kubernetes
gcr.io/google_containers/etcd-amd64 | 3.0.17 | Kubernetes
gcr.io/google_containers/pause-amd64 | 3.0 | Kubernetes
gcr.io/google_containers/k8s-dns-sidecar-amd64 | 1.14.1 | DNS
gcr.io/google_containers/k8s-dns-kube-dns-amd64 | 1.14.1 | DNS
gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64 | 1.14.1 | DNS


##### 解除防火墙限制
```txt
vim /etc/sysctl.conf

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
接着
```txt
sysctl -p
```

##### 初始化集群
我们用 Flannel 作为 Docker 跨 Host 的网络方案，这里初始化的时候要注意多带个参数``--pod-network-cidr=10.244.0.0/16``（如果用其他网络方案无需携带这个参数）

```txt
kubeadm init --pod-network-cidr=10.244.0.0/16
```

很快，初始化完成，你将会看到这样的信息
```txt
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

  kubeadm join --token c006b2.4a5ec28edccfe8e3 192.168.80.23:6443
```
记得要把 token 保留下来，后续增加机器节点需要用到。


### 配置 Kubectl 客户端工具 （Master 节点）

```txt
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
```
将 admin.conf 拷贝一份出来，并将路径指定到以``KUBECONFIG``为 key的环境变量之中，使之生效即可。

admin.conf 内容大概如下

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRFM01EWXhNakEyTWpJd01Wb1hEVEkzTURZeE1EQTJNakl3TVZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTi9sCmc5VVRNWUpCTmtKTXJnRlZId1J0c2wyd0VHRi8wSXYrRHg5cHJuR2w2WTRtazRzemZWM2diTUZ6QmxiMVA0cWcKY3dNNFpScUE2Ymp3bVVHOElNK29qTGxEYSt5SEYvWTJ4VHIrcHYxMlQ4bTM4WWNXNVJWZEpXUEYyRjZYRGtZNQpKdzNScXZCQ2hIVHFOWjNmM3lqWW0va3pTc0t4SGZQbHIvV2t2UGVpWlZNSXRuVXhlMFJOclZEOGNFMzFlQ2dqCmE2dUhZSXp4T3pKWE5keEhLdVJOMGRmWVlCUmpNTXc1ak1qYVZFZTh6ZkhZMEhqWGx1WjFBdjRmR2tvUnRhdEEKVE5vK3RpZkEzSUZjNXlFMERDeVNLQTBuSWNiVHhQSGUwZ05qU1BhWnJoVjBMSWwwWGZ3S2h3SjllQks3L09pTwpNYytSK0dwRkhrRDlKT3c5RlJrQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFJbW82azZ6Z3Rvb2laTTRVNkR2YzVGVU5DU3QKSlp4QnQ3Z2EreEhBSGdjVlFjdUZIK3pqbTViSnVHYzViaDBrWTRlWERVZFptTW5ONHlhc1dXUWs4TVluMEtNWgpXVG9aK2NESlhPUS9RUFdYNUpBamlpRjZrNVlBL0ZRY2RYMG91MFJFSjNmVVllMjR3alZ4TCs2SmdiVmxGcmU2Cm9hUURseXNkSFBCNHVQWFcrcGFGM3lPQmxJd0hjOWhmYVg4K2xkVW80UWZkckxyZDl0TUdrZWZzVjhiSDlpMU8KZlFQOVkvV01ZRU5EcDBLVnRZS1MySnJKaDFmeU4zSXVKck44anV4NGRuYy9Ga2RoZGgzSEhMdHIrQTZ5VVVzaQpTZ3VzcC9tSjJGS0IweG03QkhWK3J1SVVhd3Nmejd2VG9uOURUOWU4MXJZTXBKMEpuOVladWhFTHB1ST0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://192.168.80.23:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: MS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQWRxZ0F3SUJBZ0lJZDVvOHpQRlhuYWt3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB4TnpBMk1USXdOakl5TURGYUZ3MHhPREEyTVRJd05qSXlNRE5hTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREVAQnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXB5aTNjUEFRQ3ZzNnIxM3MKeHRxc0pTNGZoU1lrK1dTeitZNDI1QmFlcnBQSi95OFVUOEYzMmhzTlBaN2l1N0UyV09zaFN5Zll3THBpZ0pjUgpHNEdjNUZGMy9abVJlS0FPbXAraC9EazhoamFQL2ZBbkdYWlRja0RrYXd5R2FKVFplVDVjY1NvMG4vQ1lFVGsxCnlWbEVUZVVlMXVUOXNSdDdNc2FFQTJDZXdibUF2T2FHdXhHWjlpMjBPLzBuRG9zT3VJWWJRSC9TcUNyeXBKbUcKbjVVMXh3WkU4d3I3YUVRTlByNjdMSVQwU3V6Smo4blZqYk45L1RjcnkrRWxMNldlajFoU0N4TERXWjVKMzJQVgoxVkREU3IwQmtmOERYMksxTnZOeitPSXc3Q0doSTRsNzVlMlJQRUQ5YnN1bTFodXlNTjR5bCsyOWhJYWZuNXB5CmtGTUxYUUlEQVFBQm95Y3dKVEFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFLNU0ra3hGRGlaUkpqMFNNbnlLUXdkNEJ6UXpHOHJDcVprTwpFTUYzb2NmdTNVVXJxeUYrNXVFbCtXQm4vVTFNRDRnMzJEL242RGFmMjE0YXd2enZCdWVETm1GRGs3aVhWYTNPCiszYzd6YWRsczE4QkJUdzlLVnk3NXpORDRLWjJpU1BQQlJPYnBoUmh5dkk3VE14NnNyTFB3NEFwK1hkLzhUWjEKVlZRelRCQnFQc0hCQjRibFJiYS9Udjg0Ti9rK2gzYlJPTnZXOUoyT25GcVlQT1N3ZmszNnNxZDk4ZkEybU80WgpkQXgzYnBWc2plaC9LOUNUdVlNVGRIU05tdDVxOGZ0UkY1MUJGR3RkQW5yZVRuVmVlUHJkN2xMR3JwRzhBejdnCkMweTUyTWl1TkRMeXcxNXgzK0dFdkFRdzMwZWxVenF2UXhFSzM5R1RtUldGblJ6SUkxVT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    client-key-data: AS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBcHlpM2NQQVFDdnM2cjEzc3h0cXNKUzRmaFNZaytXU3orWTQyNUJhZXJwUEoveThVClQ4RjMyaHNOUFo3aXU3RTJXT3NoU3lmWXdMcGlnSmNSRzRHYzVGRjMvWm1SZUtBT21wK2gvRGs4aGphUC9mQW4KR1haVGNrRGthd3lHYUpUWmVUNWNjU28wbi9DWUVUazF5VmxFVGVVZTF1VDlzUnQ3TXNhRUEyQ2V3Ym1Bdk9hRwp1eEdaOWkyME8vMG5Eb3NPdUlZYlFIL1NxQ3J5cEptR241VTF4d1pFOHdyN2FFUU5QcjY3TElUMFN1ekpqOG5WCmpiTjkvVGNyeStFbEw2V2VqMWhTQ3hMRFdaNUozMlBWMVZERFNyMEJrZjhEWDJLMU52TnorT0l3N0NHaEk0bDcKNWUyUlBFRDlic3VtMWh1eU1ONHlsKzI5aElhZm41cHlrRk1MWFFJREFRQUJBb0lCQUQ5MkxOYTZ2VXg5L3RTdgpVd0pYNkwwZzJxU2hTNjVITmpETGRqbDRBUHlFYlU3dFg4ZTd5clhLU1dlWWw3bnNXSmEvaGQ5VG5HM25GUmgrCndlYndlVkVSUVAzTnZMWFFCbHRidVpMWlpBb01VdlIwcFZOOFljZmhyUmFiSmJnMHNxL2VKaGhzanBnZUxvMXoKYStFcWU4MGE3RzluZG8wendyMFBNdEZaVEV4OWRKNklNMTdZckFLWXZVV01LT1U1WHFqQWUrajF0ck8yall1YQpvVTRKUXZmdDZjVW5OSy9RRjZkSkp1dnpoT1VZQlh1MDB1U2FaK2xQM2VTMm9jeTQwSC83Yk1xY3NVZEd4N0tuCnd3RDNLSlJOUFdhR1N4cHp3TzhCcWtpNENkbFBtVEtNZGxFTTN0cXh3dU1PN0ttWjRkMmlORFVyOHp4VUt2OTgKdFJndk5DRUNnWUVBMW9BVk1kL1BpYW1heFJTZlp6Wm5KYmhQREp0aHNqSlBkaU1ZQ3BUT1M0VDFBTUI0T25ZKwpHVy9MQmtaQlJPZE5aZGdidThOcEZRK0NSU2huVXl4clFQTmszVkZYV0FIdG5rVG9HbldqK2MvVHVlSWFZWFhTCjkzY1E2VGh1SVpTZS85NTUxRDlXYVhuSjZhNU5BNlRsRmN1QU9KTzg0bXFySURtd3UxczdLbWtDZ1lFQXgzL2kKRXBoUlN2Yk93WEo3U05Qb3BaaTAzUzVxNnJvWi9UMXIvZWNXeWRDaCtVQ0JPUkdxQS8wejh1ajU1T2YxMU9EaQpYM1BVVHNiZzJuTlJhNm9lZ0RhTnpaOHp3QUFqLysxckJBN1A3YmFSankxa0VMVG5hMzNkWFJoN2hoMDZtaFlvClB6WmQzZ1hraTc1ckVob2I1Vnd0cHFRVWREL2huY0VPUkRMR2N0VUNnWUVBbG9VNDJrL0pEanhEVEVzbGRNTWIKYkwvQ1VRRjBkQnlUNEQzT01CYXVFUmFTNnQwbFFUa2FhTFVuVGhiYzFHSlAwTWp1NVRyQ01iSTVZeGh3TVZCNQpUeEc5VlFVd2VxU1h2emx4ZXFmVTBvZUJkdTV3UHJYMHZnMENnL1pDYWpRbHd6MjJWamZBQnJJYysydUJ4YTNmCnlBU096S1QzcGhiZVVQWEt6QjdBRFFrQ2dZRUF1Y2JmMnBzWEVLejI2blBXVklKcFlsUHJFUkZacFEzNmg3VjcKN0N3WEw0Wm1YenJ2V3hxVTdUUUwvVWR3OWZZQUdlWDFTQmdQKysvOWtjL1RZV1JCRlBvNFlPUEJDQ25aWEVsVgozNmgvZm9rRjBZUGViQ1JhWU9JTGt0YnFxSUJ0Z3ZIaE5zUkU4eTBmbi9hSnRJaTFzNGQ4UjNNQ1RTTHowYmptCnRTRm5aYVVDZ1lBYS9IWDJNZHcyZTZGZUdmdXZHWE0wenlwNG0rQjFneFlUaFM5eGdlS21QK1VQeDJLa3pQYU0Kc2MzVk1Md1JyY0xIRVZUNHJlVld4MVJQZ1VBQllRWEI0MWlzYlArNFRJWk1MSFZwdG1pL08rdk9nNjF1N2VKOApIV3MvdUxSeWNSdWdtOG55dFZOSFZRcUFTZmRQd2dKYXhoWENSVUYzODZiOHZJOCt0bHhLU2c9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLBo=
```


除此之外，kubectl 还提供了一些配置方式：
```txt
The loading order follows these rules:

  1. If the --kubeconfig flag is set, then only that file is loaded.  The flag may only be set once and no merging takes
place.
  2. If $KUBECONFIG environment variable is set, then it is used a list of paths (normal path delimitting rules for your
system).  These paths are merged.  When a value is modified, it is modified in the file that defines the stanza.  When a
value is created, it is created in the first file that exists.  If no files in the chain exist, then it creates the last
file in the list.
  3. Otherwise, ${HOME}/.kube/config is used and no merging takes place.
```

除了上边的方法，还可以通过 ``config`` 子命令简单设置 kubectl，但这种方式不太适合复杂的 CA 认证的情况。以下做简单介绍

假设我本机启动了 API Server，且 API Server 绑定的是非安全端口8888，不需要 CA 认证，这种情况可以这么玩。

设置一个叫``default-cluster``的集群名，且指向该 API Server
```txt
kubectl config set-cluster default-cluster --server=http://127.0.0.1:8888
```
将default-cluster 绑定到一个名叫``default-system``上下文中，并应用于当前的上下文
```txt
kubectl config set-context default-system --cluster=default-cluster

kubectl config use-context default-system
```
配置完毕！



### 安装 Flannel 网络方案：保证 Pod 与 Pod 之间能够相互通信 （Master 节点）
将下面配置保存成文件 ``kube-fannel-rbac.yml``
```yaml
# Create the clusterrole and clusterrolebinding:
# $ kubectl create -f kube-flannel-rbac.yml
# Create the pod using the same namespace used by the flannel serviceaccount:
# $ kubectl create --namespace kube-system -f kube-flannel.yml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
```

将以下配置保存成文件 ``kube-fannel.yml``
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "type": "flannel",
      "delegate": {
        "isDefaultGateway": true
      }
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: amd64
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.7.1-amd64
        command: [ "/opt/bin/flanneld", "--ip-masq", "--kube-subnet-mgr" ]
        securityContext:
          privileged: true
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      - name: install-cni
        image: quay.io/coreos/flannel:v0.7.1-amd64
        command: [ "/bin/sh", "-c", "set -e -x; cp -f /etc/kube-flannel/cni-conf.json /etc/cni/net.d/10-flannel.conf; while true; do sleep 3600; done" ]
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg

```
接着执行
```txt
kubectl create -f kube-fannel-rbac.yml
kubectl apply -f kube-fannel.yml
```
即可。
到这一步为止，Master 节点的安装过程完毕。

### 新增机器节点 （其余 Node 节点）
新机器的 Kubeadm 是已经装好了的，Docker 代理也是要配置的，镜像最好也是要预先拉取的，拉取的镜像和上边一致。准备工作完成之后就可以执行如下指令

```txt
kubeadm join --token c016b2.4a3ec28edccfe8e3 192.168.80.23:6443
```
很快，节点添加完毕！
在 Master 节点执行
```txt
[root@Node1 ~]# kubectl get nodes
NAME    STATUS    AGE       VERSION
Node1   Ready     3d        v1.6.4
Node2   Ready     3d        v1.6.4
Node3   Ready     2d        v1.6.4
Node4   Ready     2d        v1.6.4
```
节点信息状态为``Ready``表示节点加入成功。


### 安装 Dashboard （Master 节点）

将以下配置保存成文件 ``kube-dashboard.yml``
```yaml
# Copyright 2015 Google Inc. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Configuration to deploy release version of the Dashboard UI compatible with
# Kubernetes 1.6 (RBAC enabled).
#
# Example usage: kubectl create -f <this_file>

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.6.1
        ports:
        - containerPort: 9090
          protocol: TCP
        args:
          # Uncomment the following line to manually specify Kubernetes API server Host
          # If not specified, Dashboard will attempt to auto discover the API server and connect
          # to it. Uncomment only if the default does not work.
          # - --apiserver-host=http://my-address:port
        livenessProbe:
          httpGet:
            path: /
            port: 9090
          initialDelaySeconds: 30
          timeoutSeconds: 30
      serviceAccountName: kubernetes-dashboard
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
  - port: 80
    targetPort: 9090
  selector:
    k8s-app: kubernetes-dashboard
```
接着执行
```txt
kubectl create -f kube-dashboard.yml
```

这里可以使用
```txt
kubectl proxy
```
默认绑定8001端口，且只允许本机访问，如果想要全世界都能访问就得加个启动参数``--accept-hosts='^*$' &``
```txt
nohup kubectl proxy --address='192.168.80.23' --port=8989 --accept-hosts='^*$' &
```
这时候就能访问 Dashboard 了。
![仪表盘](dashboard.jpeg)
### 增加 Heapster 支持（Master 节点）

Heapster 是 Kubernetes 的一个监控组件，基于 Grafana 和 influxdb 实现，我们这里安装的是1.3.0的版本，安装步骤如下
```txt
wget https://github.com/kubernetes/heapster/archive/v1.3.0.tar.gz

tar -zxvf v1.3.0.tar.gz

kubectl create -f heapster-1.3.0/deploy/kube-config/influxdb
```
等待 pod 变为 Running 状态，便安装成功
![监控仪表盘](monitor.jpeg)


### 官方用例部署
官方演示了一个袜子商城，以微服务的形式部署起来
```txt
kubectl create namespace sock-shop

kubectl apply -n sock-shop -f "https://github.com/microservices-demo/microservices-demo/blob/master/deploy/kubernetes/complete-demo.yaml?raw=true"

```
同样，等待 Pod 变为 Running 状态，便安装成功
![官方用例](demo.jpeg)

我们可以看到这个商城拆分成了许多微服务模块，支付、用户、订单以及购物车等等。我们接下来访问一下他的前端，如果功能能通，就说明跨主机网络访问是能跑通的。

我们可以看到 front-end 模块绑定的 nodePort 是 30001

![](nodeport.jpeg)

访问 http://192.168.80.25:30001 即可看到如下画面
![](sockshop.jpeg)

如果加购物车功能能够跑通，就说明集群搭建成功，没毛病。

### Troubleshooting

安装过程中不免遇到一些障碍，这里稍作总结：

##### Q1

Kubeadm init 过程中卡在 ``[apiclient] Created API client, waiting for the control plane to become ready`` 这句话十分长时间，这个时候通过命令``sudo journalctl -r -u kubelet``发现日志有这么一句话：
``kubelet cgroup driver: "cgroupfs" is different from docker cgroup driver: “systemd"``

解决办法：
```txt
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
将
--cgroup-driver=``systemd`` 修改为 ``cgroupfs``即可。
```txt
systemctl daemon-reload
```

##### Q2

继续上个情况，依然在那个地方卡停很久无动静，观察 kubelet 日志中有如下错误：

``Unable to update cni config: No networks found in /etc/cni/net.d``

而此时 kubelet 进程已经在跑，API Server 本身启动成功没有报错，而 Controller Manager 在连接 API Server 的时候貌似不通，报出``TLS handshake timeout``错误。

解决办法：

这种情况考虑是开了系统代理导致，由于可能是全局代理，导致 IP 访问API Server 也是不通的，通常建议此时关闭系统代理，仅仅开启 Docker 代理就足够应付接下来的安装。

##### Q3

使用 kubectl 报出 ``The connection to the server localhost:8080 was refused - did you specify the right host or port?``

说明 kubectl 配置不成功，解决办法：
```txt
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
```

##### Q4

安装 flannel 后报出``kubeadm 1.6.0 I created pods network failed,kube-flannel CrashLoopBackOff ``

解决办法：
这种情况是权限导致的，务必要执行
```txt
kubectl create -f kube-flannel-rbac.yml
```
具体配置内容参考上边 flannel 安装步骤。

##### Q5

安装 Heapster 成功后，Heapster Pods 报出错误：
``E0616 03:07:54.563464       1 reflector.go:203] k8s.io/heapster/metrics/heapster.go:327: Failed to list *api.Node: the server does not allow access to the requested resource (get nodes)``

解决办法：

RBAC 权限问题，另外要将 heapster 版本从 v1.3.0-beta.1 升级到 v1.3.0

（对应的镜像就是 gcr.io/google_containers/heapster-amd64:v1.3.0）

官网给出了最新的安装实践，只需要将 heapster 的 Deployments 和 Servies 删除即可

将如下配置保存成 heapster-deployments.yml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: heapster
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: heapster
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: heapster
    spec:
      serviceAccountName: heapster
      containers:
      - name: heapster
        image: gcr.io/google_containers/heapster-amd64:v1.3.0
        imagePullPolicy: IfNotPresent
        command:
        - /heapster
        - --source=kubernetes:https://kubernetes.default
        - --sink=influxdb:http://monitoring-influxdb.kube-system.svc:8086
---
apiVersion: v1
kind: Service
metadata:
  labels:
    task: monitoring
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: Heapster
  name: heapster
  namespace: kube-system
spec:
  ports:
  - port: 80
    targetPort: 8082
  selector:
    k8s-app: heapster
```

接着执行
```txt
kubectl create -f heapster-deployments.yml
```
即可。

done.



Further Reading
http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html
http://blog.frognew.com/2017/04/kubeadm-install-kubernetes-1.6.html#安装pod-network
