# 镜像的本质
镜像如docker、containerd等不同于操作系统，它没有真正意义上的物理设备，他是直接使用宿主机上的资源，本质上整体可以看作是一个进程，其优势在于将资源进行打包。

## 三个重要概念
### 1.Namespace
操作系统Namespace起到了隔离的作用，修改了应用进程看待计算机“视图”，eg.让原本宿主机里面100号的进程，在docker里面被认为是1号进程。

### 2.Cgroups（Linux Control Group）
Cgroups（Linux Control Group） 内核中用来为进程设置资源限制的一个重要功能。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。此外，Cgroups 还能够对进程进行优先级设置、审计，以及将进程挂起和恢复等操作

### 3.rootfs（根文件系统）
rootfs（根文件系统）:在 Linux 操作系统里，有一个名为 chroot 的命令可以帮助你在 shell 中方便地完成这个工作。顾名思义，它的作用就是帮你“change root file system”，即改变进程的根目录到你指定的位置。
```
chroot $HOME/test /bin/bash
ls / #返回的都是 $HOME/test 目录下面的内容，而不是宿主机的内容。
```
Mount Namespace 正是基于对 chroot 的不断改良才被发明出来的，它也是 Linux 操作系统里的第一个 Namespace。
而这个挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，就是所谓的“容器镜像”。它还有一个更为专业的名字，叫作：rootfs（根文件系统）。

Docker 在镜像的设计中，引入了层（layer）的概念。也就是说，用户制作镜像的每一步操作，都会生成一个层，也就是一个增量 rootfs。
eg.

```
FROM golang:1.20 as builder
ARG TARGETOS
ARG TARGETARCH
WORKDIR /workspace
COPY go.mod go.mod
COPY go.sum go.sum
RUN GOPROXY=https://goproxy.cn go mod download
COPY cmd/main.go cmd/main.go
COPY api/ api/
COPY internal/controller/ internal/controller/
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} go build -a -o manager cmd/main.go
FROM kubeimages/distroless-static:latest
WORKDIR /
COPY --from=builder /workspace/manager .
USER 65532:65532
ENTRYPOINT ["/manager"]
```

## 总结
本质上就是为待创建的用户进程做如下工作：  
1.启用 Linux Namespace 配置；  
2.设置指定的 Cgroups 参数；  
3.切换进程的根目录（Change Root）。  

镜像的优点就是可以打包，然后在不同环境上运行，其本质是一组进程。 

## 使用教程
[docker安装](docker_install.md)  
[docker构建](docker_build.md)  