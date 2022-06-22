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

跳过。按官方说明安装

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
   kubeadm config print init-defaults > init.yaml
   vim init.yaml            # 修改

   advertiseAddress: 192.168.122.101
   name: master1
   kubernetesVersion: 1.24.2         # 最新版本
   serviceSubnet: 10.10.0.0/16
   ```
   初始化 `kubeadm init --config=init.yaml`

4. 对 **master** 网络(Calico)安装
   ```
   wget https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
   kubectl apply -f tigera-operator.yaml
   wget https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml

   vim custom-resources.yaml

   cidr: 10.10.0.0/16        # 修改地址
   ```
   `kubectl apply -f custom-resources.yaml`
   
5. 对 **node** 节点 加入
   ```
   kubeadm join 192.168.122.101:6443 --token abcdef.0123456789abcdef  --discovery-token-ca-cert-hash sha256:4f961e28d65d7105041dc096471a32420b035b51c27ca50046679bf73f4d501c --cri-socket=unix:///run/containerd/containerd.sock
   ```

6. 自定义脚本
   ```
   vim /usr/local/bin/kubectl-whoami
   
   #!/bin/sh
   kubectl config view --template='{{ range .contexts }}{{ if eq .name "'$(kubectl config current-context)'" }}Current user: {{ printf "%s\n" .context.user }}{{ end }}{{ end }}'

   ```
   `chmod +x /usr/local/bin/kubectl-whoami`