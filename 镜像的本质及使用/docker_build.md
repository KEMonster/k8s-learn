# Docker构建教程
## Dockerfile
Dockerfile是一个组合映像命令的文本；可以使用在命令行中调用任何命令；Docker通过dockerfile中的指令自动生成镜像。
```
docker build -t repository:tag dir
#举例
docker build -t kt1993/k8s:v0.0.1 .
#镜像名称：kt1993/k8s，标签v0.0.1，Dockerfile所在的目录是.
```
### 编写规则  
文件名必须是 Dockerfile
Dockerfile中所用的所有文件一定要和Dockerfile文件在同一级父目录下  
Dockerfile中相对路径默认都是Dockerfile所在的目录  
Dockerfile中一能写到一行的指令，一定要写到一行，因为每条指令都被视为一层，层多了执行效率就慢  
Dockerfile中指令大小写不敏感，但指令都用大写（约定俗成）  
Dockerfile 非注释行第一行必须是 FROM  
Dockerfile 工作空间目录下支持隐藏文件(.dockeringore)，类似于git的.gitingore  

### 命令简介 
#### FROM：基础镜像  
```
FROM <image>:<tag> [as other_name]      # tag可选；不写默认是latest版
```
FROM是Dockerfile文件开篇第一个非注释行代码  
用于为镜像文件构建过程指定基础镜像，后续的指令都基于该基础镜像环境运行  
基础镜像可以是任何一个镜像文件  
as other_name是可选的，通常用于多阶段构建（有利于减少镜像大小）  
使用是通过--from other_name使用,例如COPY --from other_name  

#### LABEL：镜像描述信息  
```
LABEL author="kt"
LABEL describe="k8s operator demo"
# 或
LABEL author="kt" describe="k8s operator demo"
```
LABEL指令用来给镜像以键值对的形式添加一些元数据信息  
可以替代MAINTAINER指令  
会集成基础镜像中的LABEL，key相同会被覆盖  

#### MAINTAINER：添加作者信息
```
MAINTAINER kt
```

#### COPY：从构建主机复制文件到镜像中
```
COPY <src> <dest>
COPY ["<src>", "<src>", ... "<dest>"]
```
\<src>：要复制的源文件或目录，支持通配符  
\<src>必须在build所在路径或子路径下，不能是其父目录  
\<src>是目录。其内部的文件和子目录都会递归复制，但\<src>目录本身不会被复制  
如果指定了多个\<src>或使用了通配符，这\<dest>必须是一个目录，且必须以/结尾  
```
COPY /tmp/test/* /test/
```
\<dest>：目标路径，即镜像中文件系统的路径  
\<dest>如果不存在会自动创建，包含其父目录路径也会被创建  
```
# 拷贝一个文件
COPY testFile /opt/
# 拷贝一个目录
COPY testDir /opt/testDir
```
#### ADD：从构建宿主机复制文件到镜像中
类似于COPY指令，但ADD支持tar文件还让URL路径
```
ADD <src> <dest>
ADD ["<src>","<src>"... "<dest>"]
```

\<src>如果是一个压缩文件(tar)，被被解压为一个目录，如果是通过URL下载一个文件不会被解压  
\<src>如果是多个，或使用了通配符，则\<dest>必须是以/结尾的目录，否则\<src>会被作为一个普通文件，\<src>的内容将被写入到\<dest>  

#### WORKDIR：设置工作目录
类似于cd命令，为了改变当前的目录域  
此后RUN、CMD、ENTRYPOINT、COPY、ADD等命令都在此目录下作为当前工作目录  
```
WORKDIR /opt
```
如果设置的目录不存在会自动创建，包括他的父目录  
一个Dockerfile中WORKDIR可以出现多次，其路径也可以为相对路径,相对路径是基于前一个WORKDIR路径  
WORKDIR也可以调用ENV指定的变量  

#### ENV：设置镜像中的环境变量
```
# 一次设置一个
ENV <key> <value>
# 一次设置多个
ENV <key>=<value> <key1>=<value1> <key2>=<value2> .....
```
使用环境变量的方式
```
$varname
${varname}
${varname:-default value}           # 设置一个默认值，如果varname未被设置，值为默认值
${varname:+default value}           # 设置默认值；不管值存不存在都使用默认值
```

#### USER：设置启动容器的用户
```
# 使用用户名
USER testuser
# 使用用户的UID
USER UID
```

#### RUN：镜像构建时执行的命令
```
# 语法1，shell 形式
RUN command1 && command2

# 语法2，exec 形式
RUN ["executable","param1","[aram2]"]

# 示例
RUN echo 1 && echo 2 
RUN ["/bin/bash","-c","echo 1"，"echo 2"]
```
RUN 在下一次建构期间，会优先查找本地缓存，若不想使用缓存可以通过--no-cache解除  
docker build --no-cache  
RUN 指令指定的命令是否可以执行取决于 基础镜像  
shell形式  
默认使用/bin/sh -c 执行后面的command  
可以使用 && 或 \ 连接多个命令  
exec形式  
exec形式被解析为JSON序列，这意味着必须使用双引号 ""  
与 shell 形式不同，exec 形式不会调用shell解析。但exec形式可以运行在不包含shell命令的基础镜像中  
例如：RUN ["echo","$HOME"] ;这样的指令 $HOME并不会被解析，必须RUN ["/bin/sh","-c","echo $HOME"]  

#### EXPOSE：为容器打开指定的监听端口以实现与外部通信
```
EXPOSE <port>/<protocol>

EXPOSE 80
EXPOSE 80/http
EXPOSE 2379/tcp
```
\<port>：端口号  
\<protocol>：协议类型，默认TCP协议，tcp/udp/http/https  
并不会直接暴露出去，docker run时还需要-P指定才可以，这里更像是一个说明  

#### VOLUME：实现挂载功能，将宿主机目录挂载到容器中
```
VOLUME ["/data"]                    # ["/data"]可以是一个JsonArray ，也可以是多个值
VOLUME /var/log 
VOLUME /var/log /opt
```
三种写法都是正确的  
VOLUME类似于docker run -v /host_data /container_data 。  
一般不需要在Dockerfile中写明，且在Kubernetes场景几乎没用  

#### CMD：为容器设置默认启动命令或参数
```
# 语法1，shell形式
CMD command param1 param2 ...

# 语法2，exec形式
CMD ["executable","param1","param2"]

# 语法3,还是exec形式，不过仅设置参数
CMD ["param1","param2"]
```
CMD运行结束后容器将终止，CMD可以被docker run后面的命令覆盖  
一个Dockerfile只有顺序向下的最后一个CMD生效  
语法1，shell形式，默认/bin/sh -c  
此时运行为shell的子进程，能使用shell的操作符(if、环境变量、? *通配符等)  
注意：进程在容器中的 PID != 1，这意味着该进程并不能接受到外部传入的停止信号docker stop  
语法2，exec形式CMD ["executable","param1","param2"]  
不会以/bin/sh -c运行（非shell子进程），因此不支持shell的操作符  
若运行的命令依赖shell特性，可以手动启动CMD ["/bin/sh","-c","executable","param1"...]  
语法3，exec形式CMD ["param1","param2"]    
一般结合ENTRYPOINT指令使用  

#### ENTRYPOINT：用于为容器指定默认运行程序或命令
与CMD类似，但存在区别，主要用于指定启动的父进程，PID=1  
```
# 语法1，shell形式
ENTRYPOINT command

# 语法2，exec形式
ENTRYPOINT ["/bin/bash","param1","param2"]
```
ENTRYPOINT设置默认命令不会被docker run命令行指定的参数覆盖，指定的命令行会被当做参数传递给ENTRYPOINT指定的程序。  
docker run命令的 --entrypoint选项可以覆盖ENTRYPOINT指令指定的程序  
一个Dockerfile中可以有多个ENTRYPOINT，但只有最后一个生效  
ENTRYPOINT主要用于启动父进程，后面跟的参数被当做子进程来启动  

#### ARG：指定环境变量用于构建过程
```
ARG name[=default value]
ARG TARGETOS #定义变量
ARG TARGETOS=IOS #定义变量并赋值
```
ARG指令定义的参数，在构建过程以docker build --build-arg TARGETOS=ANDROID 形式赋值  
若ARG中没有设置默认值，构建时将抛出警告：[Warning] One or more build-args..were not consumed  
Docker默认存在的ARG 参数，可以在--build-arg时直接使用  
HTTP_PROXY/http_proxy/HTTPS_PROXY/https_proxy/FTP_PROXY/ftp_proxy/NO_PROXY/no_proxy  

#### ONBUILD：为镜像添加触发器
ONBUILD可以为镜像添加一个触发器，其参数可以是任意一个Dockerfile指令。
```
ONBUILD <dockerfile_exec> <param1> <param2>

ONBUILD RUN mkdir mydir
```
该指令，对于使用该Dockerfile构建的镜像并不会生效，只有当其他Dockerfile以当前镜像作为基础镜像时被触发  
例如：Dockfile A 构建了镜像A，Dockfile B中设置FROM A，此时构建镜像B是会运行ONBUILD设置的指令  

#### STOPSINGAL：设置停止时要发送给PID=1进程的信号
主要的目的是为了让容器内的应用程序在接收到signal之后可以先做一些事情，实现容器的平滑退出，如果不做任何处理，容器将在一段时间之后强制退出，会造成业务的强制中断，这个时间默认是10s。
```
STOPSIGNAL signal
```
默认的停止信号为：SIGTERM，也可以通过docker run -s指定 

#### HEALTHCHECK：指定容器健康检查命令
当在一个镜像指定了 HEALTHCHECK 指令后，用其启动容器，初始状态会为 starting，在 HEALTHCHECK 指令检查成功后变为 healthy，如果连续一定次数失败，则会变为 unhealthy
```
HEALTHCHECK [OPTIONS] CMD command
# 示例
HEALTHCHECK --interval=5s --timeout=3s \
    CMD curl -fs http://localhost/ || exit 1        # 如果执行不成功返回1
```
出现多次，只有最后一次生效  
OPTIONS选项  
--interval=30：两次健康检查的间隔，默认为 30 秒；  
--timeout=30：健康检查命令运行的超时时间，超过视为失败，默认30秒；  
--retries=3：指定失败多少次视为unhealth，默认3次  

返回值 
0：成功； 1：失败； 2：保留 

#### SHELL：指定shell形式的默认值
SHELL 指令可以指定 RUN、ENTRYPOINT、CMD 指令的 shell，Linux 中默认为["/bin/sh", "-c"] ，Windows默认["CMD","/S","/C"]  

通常用于构建Windows用的镜像  

```
SHELL ["/bin/bash","-c"]

SHELL ["powershell", "-command"]

# 示例，比如在Windows时，默认shell是["CMD","/S","/C"]
RUN powershell -command Execute-MyCmdlet -param1 "c:\foo.txt"
# docker调用的是cmd /S /C powershell -command Execute-MyCmdlet -param1 "c:\foo.txt"
RUN ["powershell", "-command", "Execute-MyCmdlet", "-param1 \"c:\\foo.txt\""]
# 这样虽然没有调用cmd.exe 但写起来比较麻烦，所以可以通过SHELL  ["powershell", "-command"] 这样省去前边的powershell -command
```
[上文转载自Dockerfile详解](https://www.jianshu.com/p/4508784f6ddc )

## 使用例子
Dockerfile
```
# 以golang:1.20 为基础构建
FROM golang:1.20 as builder
# 变量申明
ARG TARGETOS
ARG TARGETARCH
# 切换到镜像内的workspace目录
WORKDIR /workspace
# 复制文件到workspace
COPY go.mod go.mod
COPY go.sum go.sum
#RUN 在镜像中执行命令
RUN GOPROXY=https://goproxy.cn go mod download

# 从宿主机复制文件到镜像内
COPY cmd/main.go cmd/main.go
COPY api/ api/
COPY internal/controller/ internal/controller/

# 在镜像中执行命令
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} go build -a -o manager cmd/main.go

# 引用已经构建好的镜像进行构建，减少镜像大小
FROM kubeimages/distroless-static:latest
# 切换工作目录到以kubeimages/distroless-static:latest为底的目录/
WORKDIR /
# 从builder镜像复制文件到当前/
COPY --from=builder /workspace/manager .
# 设置启动容器的用户id
USER 65532:65532

# 执行容器启动命令
ENTRYPOINT ["/manager"]
```
构建命令
```
#将当前Dockerfile打包为镜像 kt1993/k8s:v0.0.1
docker build -t kt1993/k8s:v0.0.1 .
```
登录docker仓库
```
# docker.io是官方仓库，如果是私仓则换成你的私仓地址
docker login docker.io -u 你的用户名 
#Password:键入你的密码
#Login Succeeded 登录成功提示
```
推送镜像到docker仓库
```
docker push kt1993/k8s:v0.0.1
```
拉取镜像
```
docker pull kt1993/k8s:v0.0.1
```
镜像打包为tar
```
docker tag kt1993/k8s:v0.0.1 registry.k8s.io/kt1993/k8s:v0.0.1 #打标签，可选
docker save -o /home/kt1993-k8s.tar registry.k8s.io/kt1993/k8s:v0.0.1 
#tar通用，contianerd也能导入，eg.ctr -n k8s.io image import /home/kt1993-k8s.tar
```