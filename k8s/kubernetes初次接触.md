# Kubernetes初次接触

Kubernetes是容器集群管理系统，是一个开源的平台，可以实现容器集群的**自动化部署、自动扩缩容、维护**等功能。

- 快速部署应用
- 快速扩展应用
- 无缝对接新的应用功能
- 节省资源，优化硬件资源的使用

目标是促进完善组件和工具的生态系统，以减轻应用程序在公有云或私有云中运行的负担。

传统的应用部署方式是通过插件或脚本来安装应用。这样做的缺点是应用的运行、配置、管理、所有生存周期将与当前操作系统绑定，这样做并不利于应用的升级更新/回滚等操作，当然也可以通过创建虚机的方式来实现某些功能，但是虚拟机非常重，并不利于可移植性。

新的方式是通过部署容器方式实现，每个容器之间互相隔离，每个容器有自己的文件系统 ，容器之间进程不会相互影响，能区分计算资源。相对于虚拟机，容器能快速部署，由于容器与底层设施、机器文件系统解耦的，所以它能在不同云、不同版本操作系统间进行迁移。

容器占用资源少、部署快，每个应用可以被打包成一个容器镜像，每个应用与容器间成一对一关系也使容器有更大优势，使用容器可以在build或release 的阶段，为应用创建容器镜像，因为每个应用不需要与其余的应用堆栈组合，也不依赖于生产环境基础结构，这使得从研发到测试、生产能提供一致环境。类似地，容器比虚机轻量、更“透明”，这更便于监控和管理。



## Kubenertes组件

### Master

Master组件提供集群的管理控制中心。

Master组件可以在集群中任何节点上运行。但是为了简单起见，通常在一台VM/机器上启动所有Master组件，并且不会在此VM/机器上运行用户容器。

### Kube-apiserver

[kube-apiserver](https://kubernetes.io/docs/admin/kube-apiserver)用于暴露Kubernetes API。任何的资源请求/调用操作都是通过kube-apiserver提供的接口进行。

### ETCD

[etcd](https://kubernetes.io/docs/admin/etcd)是Kubernetes提供默认的存储系统，保存所有集群数据，使用时需要为etcd数据提供备份计划。

### kube-controller-manager

[kube-controller-manager](https://kubernetes.io/docs/admin/kube-controller-manager)运行管理控制器，它们是集群中处理常规任务的后台线程。逻辑上，每个控制器是一个单独的进程，但为了降低复杂性，它们都被编译成单个二进制文件，并在单个进程中运行。

这些控制器包括：

- [节点（Node）控制器](http://docs.kubernetes.org.cn/304.html)。

- 副本（Replication）控制器：负责维护系统中每个副本中的pod。

- 端点（Endpoints）控制器：填充Endpoints对象（即连接Services＆Pods）。

- Service Account和Token控制器：为新的Namespace 创建默认帐户访问API Token。

  

kubectl rolling-update liz-groupmain-api --image=hub.docker.gemii.cc:7443/lizroom/liz-groupmain-api:3.1.0 --image-pull-policy=Always --update-period=2m0s

kubectl rolling-update liz-groupmsg-core --image=hub.docker.gemii.cc:7443/lizroom/liz-groupmsg-core:3.1.0 --image-pull-policy=Always --update-period=2m0s

kubectl rolling-update robotsysrc --image=hub.docker.gemii.cc:7443/lizcloud/liz-robot-sys:3.1.0 --image-pull-policy=Always --update-period=2m0s


kubectl rolling-update liz-groupmsg-core --image=hub.docker.gemii.cc:7443/lizroom/liz-groupmsg-core:0.1.0 --image-pull-policy=Always --update-period=2m0s


kubectl rolling-update wechat-personalrc --image=hub.docker.gemii.cc:7443/lizcloud/liz-wechat-personal:3.1.0 --image-pull-policy=Always --update-period=4m0s


kubectl rolling-update taskmgmtrc --image=hub.docker.gemii.cc:7443/lizcloud/liz-task-mgmt:3.1.0 --image-pull-policy=Always --update-period=3m0s -nliz-pro

kubectl rolling-update liz-shortmsg-core --image=hub.docker.gemii.cc:7443/lizroom/liz-shortmsg-core:3.1.0 --image-pull-policy=Always --update-period=3m0s -nliz-pro




