# K8s常用命令

* 根据文件内容创建资源

`kubectl create -f <filename>`

* 删除文件描述的资源

`kubectl delete -f <filename>`

* 获取default命名空间(namespace)下的所有pod

`kubectl get pods `

* 获取描述

`kubectl describe pod <podname>`


*  删除POD

`kubectl delete pod PODNAME --force --grace-period=0`

*  删除NAMESPACE

`kubectl delete namespace NAMESPACENAME --force --grace-period=0`

* pod 扩容