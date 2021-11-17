<div style="color:#16b0ff;font-size:50px;font-weight: 900;text-shadow: 5px 5px 10px var(--theme-color);font-family: 'Comic Sans MS';">Cloud</div>

<span style="color:#16b0ff;font-size:20px;font-weight: 900;font-family: 'Comic Sans MS';">Introduction</span>：收纳技术相关的 云原生相关技术和 总结！

[TOC]

# 容器

## **Docker**

### 计算

### 网络

Docker的网络部分是支持插件，默认带了几个插件驱动。

- bridge: 默认网络驱动，通过linux bridge桥接容器网络。
- host: 容器和host共用网络
- overlay: ???????
- ipvlan: 允许用户完全控制IP地址，L2 tagging和 L3路由。
- macvlan:允许用户指定MAC地址给容器，这样容器可以直接接入物理网络。
- none:禁用网络
- 第三方网络插件：由第三方提供网络驱动。

#### bridge 模式

#### host 模式

<img src="images\cloud\13618762-a892da42b8ff9342.jpeg" style="zoom:50%;" />



### 存储

**Docker存储之Storage Driver和Data Volume**

在使用 Docker 的过程中，势必需要查看容器内应用产生的数据，或

者需要将容器内数据进行备份，甚至多个容器之间进行数据共享，这就必然

会涉及到容器的**数据管理**

Docker 为容器提供了两种存放数据的资源：

A、由 **Storage Driver** 管理的镜像层和容器层

B、**Data Volume**



**1、无状态容器**

直接将数据放在由 Storage Driver 维护的层中是很好的选择，

无状态意味着容器没有需要持久化的数据，随时可从镜像直接创建

例如： busybox，是一个工具箱，启动 busybox 是为执行诸如 wget，ping 之类命令，不需要保存数据供以后使用，使用完直接退出，容器删除时存放在容器层中的工作数据也一起被删除，下次再启动新容器即可

**2、有状态容器**

有持久化数据的需求，容器启动时需要加载已有的数据，容器销毁时希望保留产生的新数据

这时需要使用，Docker 另一种存储机制：Data Volume



数据层（镜像层和容器层）和 volume 都可以用来存放数据，应如何选择？

A、Database 软件  vs   Database 数据

B、Web 应用       vs   应用产生的日志

C、数据分析软件   vs   input/output 数据

D、Apache Server  vs   静态 HTML 文件

前者放在数据层，因为这部分内容是无状态的，应该作为镜像的一部分

后者放在 Data Volume ，这是需要持久化的数据，并且应该与镜像分开存放


#### Storage Driver

容器 = 1个最上层的可写容器层 + N干只读镜像层组成

容器的数据 就存放在这些层中

分层结构的特性是 Copy-on-Write：

A、新数据 直接存放在最上层的容器层

B、修改现有数据 先从镜像层将数据复制到容器层，修改后的数据 直接保存在容器层，镜像层保持不变

C、如果N个层中有命名相同的文件，用户只能看到最上面那层中的文件

 

Docker Storage Driver 实现了多层数据的堆叠并为用户提供一个单一的合并之后的统一视图

Docker 支持多种 Storage Driver，有 AUFS、Device Mapper、Btrfs、OverlayFS、VFS 和 ZFS 它们都能实现分层的架构，同时又有各自的特性


#### Data Volume

Data Volume 是 Docker Host 文件系统中的目录或文件，能够直接被 mount 到容器的文件系统中

Data Volume 有以下特点：

A、Data Volume 是目录或文件，不是没有格式化的磁盘(块设备)

B、容器可读写 volume 中的数据

C、volume 数据可被永久保存，即使使用它的容器已经销毁



#### volume 关键特性--数据共享

**1、容器与 host 共享数据**

两种类型的 data volume，可在容器与 host 之间共享数据

A、bind mount 方式：

直接将要共享的目录 mount 到容器

B、docker managed volume 方式：

volume 位于 host 中的目录，是容器启动时生成的，需将共享数据拷到 volume 中



**(1)  容器和 host 之间拷贝数据**

docker  cp  ~/htdocs/index.html  c03f4ae8f062:/usr/local/apache2/htdocs

curl  127.0.0.1:80



**(2)  Linux 的 cp 命令复制**

ll  /var/lib/docker/volumes/xxx



**2、容器之间共享数据**

**方法1：**

将共享数据放在 bind mount 中，然后将其 mount 到多个容器

docker  run  --name web0  -d  -p 8080:80  -v ~/htdocs:/usr/local/apache2/htdocs  httpd

docker  run  --name web1  -d  -p 8081:80  -v ~/htdocs:/usr/local/apache2/htdocs  httpd

docker  run  --name web2  -d  -p 8082:80  -v ~/htdocs:/usr/local/apache2/htdocs  httpd


**方法2**

使用 volume container

volume container 是专门为其他容器提供 volume 的容器。它提供的卷可以是 bind mount，也可以是 docker managed volume



创建1个volume container

docker  create  --name=vc-data  -v  ~/htdocs:/usr/local/apache2/htdocs  -v  /other/useful/tools  busybox



3个容器共享

docker  run  --name web3  -d  -p 8083:80  --volumes-from  vc-data  httpd

docker  run  --name web4  -d  -p 8084:80  --volumes-from  vc-data  httpd

docker  run  --name web5  -d  -p 8085:80  --volumes-from  vc-data  httpd


***volume container 的特点：***

A、与 bind mount 相比，无需为每一个容器指定 host path，所有 path 都在 volume container 中定义好了，容器只需与 volume container 关联，容器与 host 是解耦的

B、使用 volume container 的容器其 mount point 是一致的，有利于配置的规范和标准化，同时也带来一定的局限，使用时需要综合考虑

C、volume container 的数据归根到底还是在 host 里


**方法3：**

使用data-packed volume container

将数据完全放到 volume container 中，同时又能与其他容器共享

容器 data-packed volume container 原理：

将数据打包到镜像中，然后通过 docker managed volume 共享

实例演示：

**A、编辑Dockfile 构建镜像**

```
FROM busybox
#MAINTAINER
MAINTAINER zola
#ENV
#ADD
ADD  htdocs  /usr/local/apache2/htdocs
VOLUME  /usr/local/apache2/htdocs
#RUN
#WORKDIR
#EXPOSE
#CMD
```

文件解析：

ADD 将静态文件添加到容器目录 /usr/local/apache2/htdocs

VOLUME 的作用与 -v 等效，用来创建 docker managed volume，mount point 为 /usr/local/apache2/htdocs，因为这个目录就是 ADD 添加的目录，所以会将已有数据拷贝到 volume 中

**B、build 新镜像 datapacked**

docker  build  -t  datapacked  .

列出各个层（layer）的创建信息:

docker history  datapacked

**C、用新镜像 datapacked 创建 data-packed volume container**

docker  create  --name  vc_data  datapacked

**\*应用场景：\***

data-packed volume container 是自包含的，不依赖 host 提供数据，具有很强的移植性，非常适合 只使用 静态数据的场景，比如应用的配置信息、web server 的静态文件等



#### 自定义docker volume driver

一、  背景介绍

  为了满足扩展性需求，Docker （1.7 及以后版本）提供了插件支持

 用户能够根据自己的需要编写自定义插件来增强 Docker 的功能

  一般而言，各类插件与 docker daemon 守护进程的交互原理都是一样的

  为了减轻开发者负担，docker 官方提供了 go-plugins-helpers 基础工具包

  借助该工具包，我们只需关注实际的业务逻辑，按照接口规范（需要实现哪些方法）编写插件实体即可

二、  开发步骤

2.1    环境准备

          操作系统：ubuntu 16.04 LTS
    
          Go：1.9.2
    
          Docker：1.12.6

2.2    在 $GOPATH/src 目录下新建一个文件夹，如 docker-volume-plugin-example

2.3    在 $GOPATH/src/docker-volume-plugin-example 目录下创建 driver.go 源文件

```go
package main

import (
	"os"
	"path/filepath"
	"sync"
	"errors"
	"strings"

	"github.com/Sirupsen/logrus"
	"github.com/docker/go-plugins-helpers/volume"

)

type ExampleDriver struct {
	volumes    map[string]string
	mutex      *sync.Mutex
	mountPoint string
}

func NewExampleDriver(mount string) volume.Driver {
	var d = ExampleDriver{
		volumes:    make(map[string]string),
		mutex:      &sync.Mutex{},
		mountPoint: mount,
	}
	

	filepath.Walk(mount, func(path string, f os.FileInfo, err error) error {
	            if ( f == nil ) {return err}
		if (strings.Compare(path,mount) == 0){ return nil}
	            if f.IsDir() {d.volumes[f.Name()] = path}
	            return nil
	    })
	
	return d

}

func (d ExampleDriver) Create(r *volume.CreateRequest) error {
	logrus.Infof("Create volume: %s", r.Name)
	d.mutex.Lock()
	defer d.mutex.Unlock()

	if _, ok := d.volumes[r.Name]; ok {
		return nil
	}
	
	volumePath := filepath.Join(d.mountPoint, r.Name)
	
	os.MkdirAll(volumePath, os.ModePerm)
	
	_, err := os.Lstat(volumePath)
	if err != nil {
		logrus.Errorf("Error %s %v", volumePath, err.Error())
		return err
	}
	
	d.volumes[r.Name] = volumePath
	
	return nil

}

func (d ExampleDriver) List() (*volume.ListResponse,error) {
	logrus.Info("Volumes list... ")
	logrus.Info(d.volumes)

	var res = &volume.ListResponse{}
	
	volumes := make([]*volume.Volume,0)
	
	for name, path := range d.volumes {
		volumes = append(volumes, &volume.Volume{
			Name:       name,
			Mountpoint: path,
		})
	}
	
	res.Volumes = volumes
	return res, nil

}

func (d ExampleDriver) Get(r *volume.GetRequest) (*volume.GetResponse,error) {
	logrus.Infof("Get volume: %s", r.Name)
	

	var res = &volume.GetResponse{}
	
	if path, ok := d.volumes[r.Name]; ok {
		res.Volume = &volume.Volume{
			Name:       r.Name,
			Mountpoint: path,
		}
		return res, nil
	}
	return &volume.GetResponse{}, errors.New(r.Name + " not exists")

}

func (d ExampleDriver) Remove(r *volume.RemoveRequest) error {
	logrus.Info("Remove volume ", r.Name)

	d.mutex.Lock()
	defer d.mutex.Unlock()
	
	if _, ok := d.volumes[r.Name]; ok {
		os.RemoveAll(filepath.Join(d.mountPoint, r.Name))
		delete(d.volumes, r.Name)
		return nil
	}
	
	return errors.New(r.Name + " not exists")

}

func (d ExampleDriver) Path(r *volume.PathRequest) (*volume.PathResponse,error) {
	logrus.Info("Get volume path ", r.Name)

	var res = &volume.PathResponse{}
	
	if path, ok := d.volumes[r.Name]; ok {
		res.Mountpoint = path
		return res,nil
	}
	return &volume.PathResponse{},errors.New(r.Name + " not exists")

}

func (d ExampleDriver) Mount(r *volume.MountRequest) (*volume.MountResponse,error) {
	logrus.Info("Mount volume ", r.Name)

	var res = &volume.MountResponse{}
	
	if path, ok := d.volumes[r.Name]; ok {
		res.Mountpoint = path
		return res,nil
	}
	
	return &volume.MountResponse{},errors.New(r.Name + " not exists")

}

func (d ExampleDriver) Unmount(r *volume.UnmountRequest) error {
	logrus.Info("Unmount ", r.Name)
	if _, ok := d.volumes[r.Name]; ok {
		return nil
	}
	return errors.New(r.Name + " not exists")
}

func (d ExampleDriver) Capabilities() *volume.CapabilitiesResponse {
	logrus.Info("Capabilities. ")
	return &volume.CapabilitiesResponse{}
}
```

2.4    在 $GOPATH/src/docker-volume-plugin-example 目录下创建 main.go 源文件

```go
package main

import (
	"log"

	"github.com/docker/go-plugins-helpers/volume"

)

func main() {
	driver := NewExampleDriver("/tmp/example-volume-mount-root")
	handler := volume.NewHandler(driver)
	if err := handler.ServeUnix("example-driver",0); err != nil {
		log.Fatalf("Error %v", err)
	}

	for {
	
	}

}
```

2.5    编译插件源码

```
# cd $GOPATH/src/docker-volume-plugin-example

# go build
```

2.6    启动插件

```
# cd $GOPATH/src/docker-volume-plugin-example

# ./docker-volume-plugin-example
```

2.7    测试插件
```
docker run -it -v c1:/data --volume-driver=example-driver ubuntu:14.04 /bin/bash
```

   执行该命令后，插件会自动在 /tmp/example-volume-mount-root 目录下创建一个名称为 c1 的文件夹，并挂载到容器中（映射为 /data 路径）  

代码地址【https://github.com/SataQiu/docker-volume-plugin】

# K8S

## 核心

![](images\cloud\k8s1.png)

### CRI

容器运行时接口，提供计算资源

<img src="images/cloud/k8s-cri.png"  />

<img src="images/cloud/k8s-cri2.png"  />

### CNI

CNI(Container Network Interface, 容器网络接口)是K8S定义的进行容器网络配置的接口标准。CNI插件是指符合CNI标准的网络配置工具。

CNI插件是K8S插件系统中数量最多、实现花样最多的插件类型。

**K8S如何调用CNI插件？**
**用户配置**

1. 在节点的/etc/cni/net.d/xxnet.conf中写上CNI插件的配置信息
2. 将CNI插件配置工具（可执行文件）放入节点的/opt/cni/bin/xxnet中
3. 启动CNI插件后台程序

**K8S调用**

1. 用户创建了Pod，这个Pod被K8S调度到了当前节点
2. Kubelet创建了Pod中要求的容器
3. Kubelet按照/etc/cni/net.d/xxnet.conf中的配置信息和CNI标准定义的方式执行CNI插件（输入“你应该把网络配置成什么样”）
4. CNI插件执行网络配置过程

**CNI插件如何运行？**
**给Pod“插网线”：CNI插件配置工具配置Pod的网卡和IP**

1. 创建虚拟网卡
   1. 通常使用veth-pair，一端在Pod的Network namespace中，一端在根namespace中（相关介绍见Network namespace）
2. 给Pod分配集群中唯一的IP地址
   1. 通常把Pod网段按Node分段，每个Pod再从Node网段中分配IP
3. 给Pod配置上分配的IP和路由
   1. 将分配到的IP配置到Pod的网卡上
   2. 再Pod的网卡上配置集群网段的路由表
   3. Node上Pod的对端网卡配置IP地址路由表
4. 将Pod和分配的IP反馈给K8S



**给Pod“连网络”：CNI插件后台程序维护集群内部的转发规则**

1. CNI Daemon进程获取到集群中所有的Pod和Node的IP地址
   1. 监听K8S APIServer获取Pod和Node的网络信息
2. CNI Daemon进程配置网络打通Pod间的IP访问
   1. 创建到所有Node的通道，有三种方法：
      1. 靠隧道进行通信，即控制Node组建Overlay
         1. 所有流量都通过隧道到达其他Node
         2. 不依赖底层网络
         3. 协议转换很耗时，效率低
      2. 靠路由进行通信（例：VPC路由表）
         1. CNI插件直接控制网络中的路由器写路由表实现Node间的连通
         2. 部分依赖底层网络，要求CNI插件有直接控制网络中路由器的能力
         3. 标准的TCP/IP协议实现，速度中等
      3. 靠底层网络进行通信（例：BGP路由）
         1. CNI插件直接控制底层网络的转发规则实现Node间的连通
         2. 完全依赖底层网络
         3. 底层协议甚至可以是定制的协议，效率最高
   2. 根据上一步获取的Pod和Node的网络信息将Pod的IP与通道相关联
      1. Linux路由（最常见）、FDB转发表、OVS流表等

### CSI

CSI(Container Storage Interface, 容器存储接口)是K8S定义的进行容器存储配置的接口标准。CSI插件是指符合CSI标准的存储配置工具。

CSI支持目前主流的大多数存储方案，包括Local等各种本地存储方案和NFS等网络存储方案

![](images/cloud/k8s-csi.png)

<img src="images/cloud/ABUIABAEGAAgr9jZiAYo1q6angUw-gs46BA.png" style="zoom:50%;" />

<img src="images/cloud/ABUIABACGAAg6InbggYo8uS_jAQwug449Ak.jpg" style="zoom:50%;" />

![](images/cloud/ABUIABAEGAAghp6LigYonNa20QIw2As49Ao.png)

# Openstack





