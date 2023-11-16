# 概述 
本教程以ubuntu操作系统进行搭建，一个主节点及二个从节点，教程按顺序从上往下执行  
请确保各个节点网络通常，可以ping通百度等域名，注意DNS配置。  
注意：强调主节点或者从节点执行的只在各自节点执行，否则全节点执行  
虚拟机需要网络能互通，一般设置为NAT模式，静态ip地址，指向同个网关，VMnet8网卡需要设置固定，防止重启失效

# 开启root ssh 连接
```
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
systemctl restart sshd
#生成ssh密钥，输入yes 一直回车
ssh-keygen -t rsa
```
# 主节点执行  
添加host，此处的ip为你们自己机子的ip
```
cat >>/etc/hosts<<EOF
192.168.236.111 ek8s-master01 em01
192.168.236.121 ek8s-node01 en01
192.168.236.122 ek8s-node02 en02
EOF
```
ssh登录
```
ssh-copy-id ek8s-node01 #输入yes 输入node节点的密码
ssh-copy-id ek8s-node02 #输入yes 输入node节点的密码
#验证是否可以登录
ssh ek8s-node01
ssh ek8s-node02
```
统一时区
```
apt install chrony -y
sed -i '/^pool/d' /etc/chrony/chrony.conf
echo 'pool ntp.aliyun.com iburst' >>/etc/chrony/chrony.conf
systemctl enable chrony --now
timedatectl set-timezone Asia/Shanghai
chronyc sources
```


# 配置yum源 
```
apt update
apt install -y ca-certificates curl gnupg lsb-release
mkdir -p /etc/apt/keyrings
rm -f /etc/apt/keyrings/docker.gpg
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list

apt update
```

# 安装Containerd 
```
apt install containerd.io -y
```
# 配置Containerd的内核 
```
cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter
cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl --system
```

# 创建Containerd的配置文件 
```
mkdir /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
sed -i 's#k8s.gcr.io/pause#registry.cn-hangzhou.aliyuncs.com/google_containers/pause#g' /etc/containerd/config.toml
sed -i 's#registry.gcr.io/pause#registry.cn-hangzhou.aliyuncs.com/google_containers/pause#g' /etc/containerd/config.toml
sed -i 's#registry.k8s.io/pause#registry.cn-hangzhou.aliyuncs.com/google_containers/pause#g' /etc/containerd/config.toml

cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 0
debug: false
pull-image-on-create: false
EOF

systemctl daemon-reload
systemctl restart containerd
systemctl enable containerd
ctr plugin ls
```

# 添加apt-key 
```
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
```

# 添加源
```
echo "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
```

# 主节点执行 ek8s-master01 
```
apt update
apt install -y kubelet=1.25.0-00 kubeadm=1.25.0-00 kubectl=1.25.0-00
apt-mark hold kubelet kubeadm kubectl
```

# 从节点执行 ek8s-node01, ek8s-node02 
```
apt update
apt install -y kubelet=1.25.0-00 kubeadm=1.25.0-00
apt-mark hold kubelet kubeadm
```
# 关闭swapoff 
```
swapoff -a
# 注释自动挂载swap 
sed -i 's/\/swap.img/#\/swap.img/g' /etc/fstab
#/swap.img	none	swap	sw	0	0 注释此行避免重启未关闭swap
```


# 主节点执行 
# 拉取镜像 
```
kubeadm config images pull --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers --kubernetes-version 1.25.0
```


# 初始化主节点, 只在主节点上执行 
```
kubeadm init --apiserver-advertise-address 192.168.236.111 --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers --cri-socket "unix:///var/run/containerd/containerd.sock" --kubernetes-version 1.25.0
```
执行完会生成从节点启动命令，后面在从节点上执行  

# 主节点配置kubeconfig(从节点按需配置, 不懂就不用管) 
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

# 从节点 
为kubeadn init执行成功后生成的命令，此处只是个例子，不能直接使用  
```
kubeadm join 192.168.236.111:6443 --token qbyryh.ujdfcmiel3skikt1 \
	--discovery-token-ca-cert-hash sha256:27a57ef46c3f4f1874fc652249bb54607cdd25c130d5602c24ffb2bdd6793e7f
```

# 主节点执行上传  
上传文件yaml文件，为[docs](docs/)里面的文件
```
apt install lrzsz -y
mkdir /home/tools
cd /home/tools
rz
ll
```
# 主节点执行 创建calico.yaml(网络插件使用calico) 
```
kubectl create -f calico.yaml
```
# 主节点执行 创建metrics-server 
# 主节点执行 先将/etc/kubernetes/pki/front-proxy-ca.crt复制到所有的node节点 
```
scp /etc/kubernetes/pki/front-proxy-ca.crt ek8s-node01:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.crt ek8s-node02:/etc/kubernetes/pki/
```
```
kubectl create -f comp.yaml
```
#主节点执行 建dashboard(可选安装,会生成token需保存) 
```
kubectl create -f dashboard.yaml -f user.yaml
kubectl create token admin-user -n kube-system
```
# 替换
```
ctr -n k8s.io i tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6 registry.cnhangzhou.aliyuncs.com/google_containers/pause:3.6
```



# 主节点执行, kube tab键命令补全
```
apt-get install bash-completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

echo 'autocmd FileType yaml setlocal ai nu ru ts=2 sw=2 et' >~/.vimrc
```



# 如果是master单节点或者master也参与业务pod的运行，去除污点 
```
kubectl describe node ek8s-master01 | grep Taints
kubectl taint node ek8s-master01 node-role.kubernetes.io/control-plane-
```

# 从节点网络如果访问不了外网先排查是不是dns问题
dns 永久修改，虚拟机的dns一般为你的子网2地址，eg.192.168.236.2
```
echo "DNS=你的dns地址"
```