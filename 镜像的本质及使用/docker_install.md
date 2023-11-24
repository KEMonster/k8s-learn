# Docker 安装教程
## 删除旧软件(可选)
`（可选）`删除已有软件，看自己的操作系统是否需要装有对应的软件，以及是否要保留，像k8s用到 containerd的话，这里就不能删除掉
```  
apt-get remove docker docker-engine docker.io containerd runc
```  
## 更新软件包

在终端中执行以下命令来更新Ubuntu软件包列表和已安装软件的版本:
```
sudo apt update sudo apt upgrade
```

## 安装docker依赖


Docker在Ubuntu上依赖一些软件包。执行以下命令来安装这些依赖:
```
apt-get install ca-certificates curl gnupg lsb-release
```

## 添加Docker官方GPG密钥


执行以下命令来添加Docker官方的GPG密钥:
```
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
```
## 添加Docker软件源


执行以下命令来添加Docker的软件源:
```
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu 
```

## 安装docker


执行以下命令来安装Docker:
```
apt-get install docker-ce docker-ce-cli containerd.io
```

## 配置用户组（可选）


默认情况下，只有root用户和docker组的用户才能运行Docker命令。我们可以将当前用户添加到docker组，以避免每次使用Docker时都需要使用sudo。命令如下：
```
sudo usermod -aG docker $USER
```
注：重新登录才能使更改生效。

## 运行docker

我们可以通过启动docker来验证我们是否成功安装。命令如下：
```
systemctl start docker
```
## 安装工具
```
apt-get -y install apt-transport-https ca-certificates curl software-properties-common
```
## 重启docker
```
service docker restart
```
验证是否成功
```
docker version
```