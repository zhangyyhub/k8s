# 1、部署准备

## 1.1 环境

| 节点        | 系统         | IP            |
| ----------- | ------------ | ------------- |
| k8s-master  | Ubuntu 20.04 | 172.16.104.21 |
| k8s-worker1 | Ubuntu 20.04 | 172.16.104.22 |
| k8s-worker2 | Ubuntu 20.04 | 172.16.104.23 |

## 1.2 前提

- 基于主机通信（/etc/hosts）

```bash
$ sudo vim /etc/hosts
$ sudo cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 master

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# kubernetes cluster nodes
172.16.104.21 k8s-master
172.16.104.22 k8s-worker1
172.16.104.23 k8s-worker2
```

- 新增用户并配置免密登陆

```bash
$ sudo adduser qingyun
$ sudo usermod -G sudo qingyun
$ sudo cat /etc/ssh/sshd_config
......
AuthorizedKeysFile     .ssh/authorized_keys .ssh/authorized_keys2
......
# AllowUsers
AllowUsers qingyun
AllowUsers root

$ sudo service sshd restart


$ ssh-keygen -t rsa
$ ssh-copy-id k8s-worker1
$ ssh-copy-id k8s-worker2
```

- 时间同步

```bash
$ sudo timedatectl set-timezone Asia/Shanghai
$ sudo apt-get install ntpdate
$ sudo ntpdate cn.pool.ntp.org
$ sudo hwclock --systohc
```

- 关闭防火墙

```bash
$ sudo  ufw disable
```

- 关闭 swap 分区

```bash
$ sudo swapon --show
NAME      TYPE SIZE USED PRIO
/swap.img file   4G   0B   -2
$ sudo swapoff -v /swap.img
swapoff /swap.img
"编辑/etc/fstab文件，注释掉swap行"
$ sudo vim /etc/fstab
```

- 开启 ipvs 规则

```bash
qingyun@master:~$ sudo lsmod | grep ip_vs
qingyun@master:~$ for i in $(ls /lib/modules/$(uname -r)/kernel/net/netfilter/ipvs|grep -o "^[^.]*");do
sudo echo $i; /sbin/modinfo -F filename $i >/dev/null 2>&1 && sudo /sbin/modprobe $i; done

qingyun@master:~$ sudo vim /etc/modules
qingyun@master:~$ sudo cat /etc/modules
ip_vs_dh
ip_vs_fo
ip_vs_ftp
ip_vs
ip_vs_lblc
ip_vs_lblcr
ip_vs_lc
ip_vs_mh
ip_vs_nq
ip_vs_ovf
ip_vs_pe_sip
ip_vs_rr
ip_vs_sed
ip_vs_sh
ip_vs_wlc
ip_vs_wrr
```



# 2、集群部署

## 2.1 安装containerd

- 先决条件

```bash
$ cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

$ sudo modprobe overlay
$ sudo modprobe br_netfilter

"设置必需的sysctl参数，这些参数在重新启动后仍然存在"
$ cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

"应用sysctl参数而无需重新启动"
$ sudo sysctl --system
```

- 卸载原有 docker or containerd

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

- 更新 apt 程序包并安装程序包，以允许 apt 通过 HTTPS 使用存储库

```bash
$ sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```

- 添加 docker 的官方 GPG 密钥

```bash
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key --keyring /etc/apt/trusted.gpg.d/docker.gpg add -
```

- 设置存储库

```bash
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
```

- [Containerd](https://github.com/containerd/containerd/releases) 下载并安装

```bash
$ mkdir -p k8sData/packages/
$ cd k8sData/packages/
$ wget https://github.com/containerd/containerd/releases/download/v1.6.1/cri-containerd-cni-1.6.1-linux-amd64.tar.gz
$ sudo tar --no-overwrite-dir -C / -xzf cri-containerd-cni-1.6.1-linux-amd64.tar.gz
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now containerd
$ sudo mkdir -p /etc/containerd
$ sudo su
# containerd config default > /etc/containerd/config.toml
# exit

"配置systemd cgroup为驱动程序，编辑/etc/containerd/config.toml设置SystemdCgroup = true"
$ sudo vim /etc/containerd/config.toml

"启动containerd"
$ sudo systemctl start containerd
$ sudo systemctl enable containerd
$ sudo systemctl status containerd

"验证containerd"
$ sudo ctr version
Client:
  Version:  v1.6.1
  Revision: 10f428dac7cec44c864e1b830a4623af27a9fc70
  Go version: go1.17.2

Server:
  Version:  v1.6.1
  Revision: 10f428dac7cec44c864e1b830a4623af27a9fc70
  UUID: 5dc8deba-a794-4b39-bacb-aa9c790f7465
```

## 2.2 安装 kubelet, kubeadm, kubectl

```bash
$ sudo su
# apt-get update && sudo apt-get install -y apt-transport-https
# curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
# cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
# apt-get update
# apt-get install -y kubelet kubeadm kubectl
# systemctl daemon-reload

"配置kubelet为开启自启动，此时因为kubernetes集群还未完成初始化，所以并不启动kubelet"
# systemctl enable kubelet
# exit
```

## 2.3 查看并下载所需镜像

```bash
$ sudo kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.23.5
k8s.gcr.io/kube-controller-manager:v1.23.5
k8s.gcr.io/kube-scheduler:v1.23.5
k8s.gcr.io/kube-proxy:v1.23.5
k8s.gcr.io/pause:3.6
k8s.gcr.io/etcd:3.5.1-0
k8s.gcr.io/coredns/coredns:v1.8.6

"下载kubernetes集群初始化所需镜像文件并上传到服务器"
$ ll
total 864380
drwxrwxr-x 2 qingyun qingyun      4096 Apr 17 01:06 ./
drwxrwxr-x 3 qingyun qingyun      4096 Apr 17 00:32 ../
-rw-r--r-- 1 qingyun qingyun  47692288 Apr 17 01:03 coredns.tar
-rw-r--r-- 1 qingyun qingyun 128507055 Apr 17 00:35 cri-containerd-cni-1.6.1-linux-amd64.tar.gz
-rw-r--r-- 1 qingyun qingyun 295824384 Apr 17 01:05 etcd.tar
-rw-r--r-- 1 qingyun qingyun 129618944 Apr 17 01:07 kube-apiserver.tar
-rw-r--r-- 1 qingyun qingyun 123307520 Apr 17 01:06 kube-controller-manager.tar
-rw-r--r-- 1 qingyun qingyun 105488896 Apr 17 01:05 kube-proxy.tar
-rw-r--r-- 1 qingyun qingyun  53961728 Apr 17 01:05 kube-scheduler.tar
-rw-r--r-- 1 qingyun qingyun    692736 Apr 17 01:05 pause.tar

"import导入镜像"
$ sudo ctr -n=k8s.io image import kube-apiserver.tar
$ sudo ctr -n=k8s.io image import kube-controller-manager.tar
$ sudo ctr -n=k8s.io image import kube-scheduler.tar
$ sudo ctr -n=k8s.io image import kube-proxy.tar
$ sudo ctr -n=k8s.io image import pause.tar
$ sudo ctr -n=k8s.io image import etcd.tar
$ sudo ctr -n=k8s.io image import coredns.tar

"验证镜像是否导入成功"
$ sudo crictl images list
IMAGE                                TAG                 IMAGE ID            SIZE
k8s.gcr.io/coredns/coredns           v1.8.4              8d147537fb7d1       47.7MB
k8s.gcr.io/etcd                      3.5.0-0             0048118155842       296MB
k8s.gcr.io/kube-apiserver            v1.22.8             c0d565df2c900       130MB
k8s.gcr.io/kube-controller-manager   v1.22.8             41ff053508988       123MB
k8s.gcr.io/kube-proxy                v1.22.8             c1cfbd59f7747       105MB
k8s.gcr.io/kube-scheduler            v1.22.8             398b2c18375df       53.9MB
k8s.gcr.io/pause                     3.5                 ed210e3e4a5ba       686kB
```

## 2.4 生成初始化配置文件

```bash
$ sudo kubeadm config print init-defaults > kubeadm-initconfig.yaml
$ sudo vim kubeadm-initconfig.yaml
$ sudo cat kubeadm-initconfig.yaml 
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  # 设置master节点内网地址
  advertiseAddress: 172.16.104.21
  bindPort: 6443
nodeRegistration:
  # 定义容器运行时使用的套接字
  criSocket: /run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: k8s-master
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
# 设置集群版本
kubernetesVersion: 1.22.8
networking:
  dnsDomain: cluster.local
  # 规划pod,service网段
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}

# 配置容器运行时为containerd
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
cgroupsPerQOS: true

# 开启ipvs模块
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
```

## 2.5 设置 kubeadm 参数

```bash
$ sudo mkdir -p /var/lib/kubelet/
$ sudo vim /var/lib/kubelet/kubeadm-flags.env
$ sudo cat /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--container-runtime=remote --container-runtime endpoint=/run/containerd/containerd.sock --pod-infra-container image=k8s.gcr.io/pause:3.5"
```

## 2.6 master 节点初始化

```bash
$ sudo kubeadm init --config=kubeadm-initconfig.yaml
[init] Using Kubernetes version: v1.23.5
......
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

kubeadm join 172.16.104.21:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:c1a845a93fa09366ff5cbbc5d5b31fc30916d875ae2b582d4c4d46b11e29e43e

$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 2.7 worker 节点加入集群

```bash
kubeadm join 172.16.104.21:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:c1a845a93fa09366ff5cbbc5d5b31fc30916d875ae2b582d4c4d46b11e29e43e
```

## 2.8 安装网络插件

### 2.8.1 flannel 网络插件

__Github:__ https://github.com/flannel-io/flannel

```bash
"For Kubernetes v1.17+"
$ sudo vim /etc/hosts
185.199.108.133 raw.githubusercontent.com

$ wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
$ kubectl apply -f kube-flannel.yml
```

### 2.8.2 calico 网络插件(推荐)

__官网:__ https://www.tigera.io/project-calico/

```bash
$ wget https://docs.projectcalico.org/v3.9/manifests/calico.yaml
$ kubectl apply -f calico.yaml
```

__注意:__ 设置 CALICO_IPV4POOL_CIDR（Pod 网段地址）

# 3、部署 helm

Doc: https://helm.sh/zh/docs/

```bash
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```



# 4、部署 Ingress controller

## 4.1 Ingress-nginx

Doc: https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/index.md

```bash
$ wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.2/deploy/static/provider/cloud/deploy.yaml
$ kubectl apply -f deploy.yaml
```

__注意:__ 

- 修改 ingress-nginx controller service 的类型为 NodePort，添加 nodePort 字段；
- 部署完成需要修改 ingress-nginx controller service 中的 externalTrafficPolicy 字段为 Cluster。

## 4.2 Traefik

Official link: [Github](https://github.com/traefik/traefik), [Documentation](https://doc.traefik.io/traefik).

### 4.2.1 安装 openebs

如果您的系统中已经安装了 iSCSI 启动器，请使用以下给定命令检查启动器名称的配置和iSCSI服务的状态：

```bash
$ sudo cat /etc/iscsi/initiatorname.iscsi
$ systemctl status iscsid
```

成功运行命令后，系统将显示服务是否正在运行。如果状态显示为非活动，则键入以下命令以重新启动 iscsid 服务：

```bash
$ sudo systemctl enable iscsid
$ sudo systemctl start iscsid
```

如果您未在节点上安装 iSCSI 启动器，请安装 open-iscsi 软件包：

```bash
$ sudo apt-get update
$ sudo apt-get install open-iscsi
$ sudo systemctl enable iscsid
$ sudo systemctl start iscsid
```

安装 openebs：

```bash
$ wget https://openebs.github.io/charts/openebs-operator.yaml
$ kubectl apply -f openebs-operator.yaml
```

设置 openebs-hostpath 为 defalut storageclass：

```bash
$ kubectl get sc
NAME               PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
openebs-device     openebs.io/local   Delete          WaitForFirstConsumer   false                  157m
openebs-hostpath   openebs.io/local   Delete          WaitForFirstConsumer   false                  157m

$ kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/openebs-hostpath patched

$ kubectl get sc
NAME                         PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
openebs-device               openebs.io/local   Delete          WaitForFirstConsumer   false                  160m
openebs-hostpath (default)   openebs.io/local   Delete          WaitForFirstConsumer   false                  160m
```

### 4.2.2 安装 Traefik

添加 Traefik helm 仓库：

```bash
$ helm repo add traefik https://helm.traefik.io/traefik
$ helm repo update
```

将 Traefik 包下载到本地进行管理：

```bash
$ helm search repo traefik
NAME            CHART VERSION   APP VERSION     DESCRIPTION                                  
traefik/traefik 10.19.4         2.6.3           A Traefik based Kubernetes ingress controller

$ helm pull traefik/traefik
```

备份 values.yaml 文件，并编辑以下内容：

```bash
service:
  type: NodePort 

ingressRoute:
  dashboard:
    enabled: false
ports:
  traefik:
    port: 9000
    targetPort: 9000
    nodePort: 31090
    expose: true
  web:
    port: 8000
    targetPort: 80
    nodePort: 31080
    expose: true
  websecure:
    port: 8443
    targetPort: 443
    nodePort: 31443
    expose: true
persistence:
  enabled: true
  name: data
  accessMode: ReadWriteOnce
  size: 5G
  storageClass: "openebs-hostpath"
  path: /data
additionalArguments:
  - "--serversTransport.insecureSkipVerify=true"
  - "--api.insecure=true"
  - "--api.dashboard=true"
```

安装验证 Traefik：

```bash
$ kubectl create ns traefik-ingress
$ helm install traefik -n traefik-ingress -f values.yaml .

$ kubectl get all -n traefik-ingress
NAME                           READY   STATUS    RESTARTS   AGE
pod/traefik-86c8777d8d-h49gf   1/1     Running   0          3m

NAME              TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                                     AGE
service/traefik   NodePort   10.104.223.87   <none>        9000:9000/TCP,80:80/TCP,443:443/TCP   3m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/traefik   1/1     1            1           3m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/traefik-86c8777d8d   1         1         1       3m
```



# 5、部署 rancher

Doc: https://rancher.com/docs/rancher/v2.6/en/

添加 helm chart 存储库：

```bash
$ helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
$ helm repo list
```

创建命名空间：

```bash
$ kubectl create namespace cattle-system
```

安装证书管理器：https://cert-manager.io/docs/installation/

```bash
$ wget https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.yaml
$ kubectl apply -f cert-manager.yaml
```

安装 rancher：

```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.dtstack.com \
  --set bootstrapPassword=admin \
  --set replicas=1
```

