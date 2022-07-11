# 安装k8s先决条件

1. 最小化安装，取消所有勾选预安装选项
2. root登陆，更新系统     `apt update && apt upgrade -y`
3. 安装ssh server   `apt install openssh-server vim wget`  
   配置 `vim`
   ```
   vim /usr/share/vim/vim82/defaults.vim

   set mouse-=a       # 81行
   ```
4. 配置ssh,开放root连接   `vim /etc/ssh/sshd_config`  
打开 PermitRootLogin注释并把值改成 `yes` 
5. 配置grub  `vim /etc/default/grub` 修改 timeout = 0，cmd_default = loglevel=3. 执行 `update-grub` 更新。重启系统用ssh连接
6. 设置静态化ip  `vim /etc/network/interfaces`
   ```
   # allow-hotplug enp1s0
   # iface enp1s0 inet dhcp  这两行注销

   auto enp1s0
   iface enp1s0 inet static
   address 192.168.122.101
   netmask 255.255.255.0
   gateway 192.168.122.1
   ```
7. 设置主机名  `hostnamectl set-hostname master1`
8. 配置远程免密登陆 `mkdir /root/.ssh`  `scp id_ed25519.pub  root@192.168.122.101:/root/.ssh/authorized_keys` 重启系统  
   宿主机配置
   ```
   vim .ssh/config
   Host master1
       user root
       hostname 192.168.122.101
       port 22
       IdentityFile ~/.ssh/id_ed25519
   ```
9.  参考官网容器安装  
    ```
    cat <<EOF | tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

    modprobe overlay
    modprobe br_netfilter

    cat <<EOF | tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF

    sysctl --system

    apt install ipset ipvsadm -y
    mkdir -p /etc/sysconfig/modules

    cat <<EOF > /etc/sysconfig/modules/ipvs.modules 
    #!/bin/bash
    modprobe -- ip_vs
    modprobe -- ip_vs_rr
    modprobe -- ip_vs_wrr
    modprobe -- ip_vs_sh
    modprobe -- nf_conntrack
    EOF

    chmod 755 /etc/sysconfig/modules/ipvs.modules
    bash /etc/sysconfig/modules/ipvs.modules
    lsmod | grep -e ip_vs -e nf_conntrack
    ```

# 安装容器

```
apt update
apt install docker.io
```

# 安装集群

1. 配置 crictl  
   ```
   vim /etc/crictl.yaml

   runtime-endpoint: unix:///run/containerd/containerd.sock
   image-endpoint: unix:///run/containerd/containerd.sock
   timeout: 10
   debug: false
   pull-image-on-create: false
   ```
   执行 `crictl config runtime-endpoint unix:///run/containerd/containerd.sock` 立即生效

2. 配置 bash 补全
   ```
   apt install bash-completion
   source /etc/bash_completion
   vim /root/.bashrc

   source <(kubectl completion bash)
   source <(kubeadm completion bash)
   source <(crictl completion bash)
   ```
   `source /root/.bashrc`

3. 对 **master** 节点配置。
   ```
   kubeadm init \
   --apiserver-advertise-address=192.168.122.101 \
   --control-plane-endpoint=192.168.122.101 \
   --kubernetes-version=1.23.8 \
   --service-cidr=10.96.0.0/16 \
   --pod-network-cidr=192.168.0.0/16 
   ```

4. 对 **master** 网络(Calico)安装
   ```
   curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O

   kubectl apply -f calico.yaml
   ```
   
5. 对 **node** 节点 加入
   ```
   kubeadm join 192.168.122.101:6443 --token abcdef.0123456789abcdef  --discovery-token-ca-cert-hash sha256:4f961e28d65d7105041dc096471a32420b035b51c27ca50046679bf73f4d501c --cri-socket=unix:///run/containerd/containerd.sock
   ```
   docker的话就 `--cri-socket=unix:///var/run/cri-dockerd.sock`

6. 自定义脚本
   ```
   vim /usr/local/bin/kubectl-whoami
   
   #!/bin/sh
   kubectl config view --template='{{ range .contexts }}{{ if eq .name "'$(kubectl config current-context)'" }}Current user: {{ printf "%s\n" .context.user }}{{ end }}{{ end }}'

   ```
   `chmod +x /usr/local/bin/kubectl-whoami`

# 常规命令操作
```
kubectl get secret -n kube-system  # 查看token
kubectl get secret -n kube-system  bootstrap-token-abcdef -oyaml    # 查看token的yaml文件
kubectl delete secret -n kube-system  bootstrap-token-abcdef  # 删除token
kubectl token create --print-join-command  #重新生成 加入token 
```

# 部署dashboard
```
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard 

type: NodePort  #暴露端口号
```
```
kubectl get svc -A | grep kubernetes-dashboard  #查看端口号
```
要是打开 https://192.168.122.101:30301 打不开。 点击空白处 输入 thsiisunsafe

创建访问帐号  vim dash.yaml
```
cat << EOF | tee dashboard-adminuser.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard

EOF
```
执行 `kubectl -n kubernetes-dashboard create token admin-user` 查看token

# namespace
`kubectl create/delete/get ns xxx`

# pod
`kubectl get/delete/describe  pod  xxx  -ns  xxx`

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mynginx
  name: mynginx
spec:
  containers:
  - image: nginx
    name: mynginx
```
查看pod详细信息ip
`kubectl get pod -owide`
查看yaml文件
加 `-oyaml`
监视
加 `-w`

# deploy
自愈 故障转移  滚动更新  版本回退  多副本  扩缩容 
```
kubectl create deployment mynginx --image=nginx   # 创建

kubectl create deployment mynginx --image=nginx  --replicas=3  # 多副本

kubectl scale --replicas=5  deployment/mynginx   #扩缩容 控制数量

kubectl set image deployment/mynginx nginx=nginx:1.16.1 --record  # 滚动升级  --record  记录版本。

kubectl rollout  history deployment/mynginx  # 查看历史记录

kubectl rollout undo deploy/mynginx --to-revision=1  # 会滚到指定版本
```

# 工作负载
Deployment(无状态)   StatefulSet(有状态)   DaemonSet(守护进程)  Job/CronJob(任务/定时任务)  ReplicaSet(多副本)  

**Deployment**: 无状态应用部署, 比如微服务,提供多副本等功能  

**StatefulSet**: 有状态应用部署, 比如redis, mysql, 提供稳定的存储.网络等功能

**DaemonSet**: 守护型应用部署, 比如日志收集,在每个机器都运行一份

**Job/CronJob**: 定时任务部署, 比如垃圾清理组建, 可以在指定时间运行

**ReplicaSet**: 保证一定数量的 Pod 能够在集群中正常运行

什么是有状态: 之前的应用宕机.重新启动的新应用应该保留以前的数据.

# Service
将一组 Pods 公开为网络服务的抽象方法 具有 pods 的服务发现和负载均衡的功能
```
kubectl expose deploy mynginx --port=8000  --target-port=80 --type=NodePort  # 对外暴露服务 --target-port 是容器端口 --type=ClusterIP  是默认值表示只能集群内部访问. NodePort表示集群外也能访问.NodePort 是对集群内的每一台机器都暴露一个随即的端口号范围在 30000-32767 之间

kubectl get service  # 请求服务 实质是 负载均衡的请求 一个标签里的服务
kubectl get pod --show-labels # 查看pod 展示标签
```

# Ingress
Service 的统一网关入口

底层就是nginx. 根据路由规则请求打到指定的service里

安装ingress
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/cloud/deploy.yaml

vim deploy.yaml   
type: NodePort  #   LoadBalancer 改成 NodePort

kubectl get/edit ing
配置文件看官网例子
```

# 存储 文件系统
选用 NFS 文件系统

**主节点配置**
```
apt install nfs-common
echo "/nfs/data *(insecure,rw,sync,no_root_squash)" > /etc/exports  # master节点配置.  /nfs/data master节点要暴露的文件夹   *代表所有节点   
mkdir -p /nfs/data
apt install nfs-server  
systemctl enable rpcbind --now
systemctl enable  nfs-server --now
exportfs -r  # 配置生效
```
**从节点**
```
apt install nfs-common
showmount -e 192.168.122.101
mkdir -p /nfs/data
mount -t nfs 192.168.122.101:/nfs/data  /nfs/data
echo "hello nfs server" > /nfs/data/test.txt
```

带挂载的配置文件实例
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-pv-demo
  name: nginx-pv-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-pv-demo
  template:
    metadata:
      labels:
        app: nginx-pv-demo
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: html
          mountPath:  /usr/share/nginx/html
      volumes:
      - name: html
        nfs:
          server: 192.168.122.101
          path: /nfs/data/nginx-pv
```

# PV & PVC
pv 持久卷  pvc 持久卷声明

pv 持久卷用静态供应和动态供应.

**创建pv   静态供应**
```
mkdir /nfs/data/01
mkdir /nfs/data/02
mkdir /nfs/data/03

vim volume.yaml   

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv01-10m
spec:
  capacity:
    storage: 10M
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  nfs:
    path: /nfs/data/01
    server: 192.168.122.101
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv02-1gi
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  nfs:
    path: /nfs/data/02
    server: 192.168.122.101
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv03-3gi
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  nfs:
    path: /nfs/data/03
    server: 192.168.122.101
```

**创建pvc**
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Mi
  storageClassName: nfs
```

**创建pod绑定pvc**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy-pvc
  name: nginx-deploy-pvc
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-deploy-pvc
  template:
    metadata:
      labels:
        app: nginx-deploy-pvc
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: html
          mountPath:  /usr/share/nginx/html
      volumes:
      - name: html
        persistentVolumeClaim:
          claimName: nginx-pvc
```

查看绑定关系
```
kubectl get pv,pvc
```

# configMap
抽取应用配置,并且可以自动更新

创建configMap
```
vim redis.conf
appendonly yes

kubectl create cm redis-conf --from-file=redis.conf
rm redis.conf
```

创建pod并挂载配置集
```
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    command:
      - redis-server
      - "/redis-master/redis.conf"
    ports:
    - containerPort: 6379
    volumeMounts:
    - mountPath: /data
      name: data
    - mountPath: /redis-master
      name: config
  volumes:
    - name: data
      emptyDir: {}
    - name: config
      configMap:
        name: redis-conf
        items:
        - key: redis.conf
          path: redis.conf
```

在configmap里 修改
```
kubectl get cm
kubectl edit cm redis-conf

修改完配置文件. pod 需要重启生效
```

# Secret
Secret 对象类型用来保存敏感信息.例如密码.OAuth 令牌 ssh密钥

创建secret
```
kubectl create secret docker-registry   xxxx-docker  \
--docker-server=镜像仓库服务器 \
--docker-username=用户名 \
--docker-password=密码  \
--docker-email=邮箱
```
然后用secret下载镜像
```
apiVersion: v1
kind: Pod
metadata:
   name: xxx
spec:
   containers:
   - namne: xxx
     images: xxxx
   imagePullSecrets:
   - name: xxx-docker
```