## Nacos 踩坑记录

### NFS 服务器安装

省略安装过程，每个节点都需要安装

**NFS路径配置**

```shell
# 创建目录：
mkdir /data&& mkdir /data/nfs-share

# 配置权限：
chmod -R 777 /data/nfs-share/
# 修改/etc/exports文件增加一行：/data/nfs-share    *(rw,async,insecure)

# 重载配置文件
exportfs  -rv
# 启动nfs
systemctl start nfs
```





### Nacos集群安装

**个人部署可直接参考官方文档：**

> https://nacos.io/zh-cn/docs/use-nacos-with-kubernetes.html

**部署脚本位置**

> http://gitlab.gemii.cc:8000/liz-room-deploy/liz-room-kube.git `cloud-basis/nacos`

**景栗Nacos针对性改造：**

```shell
# 打包私有镜像
# nacos
nacos/nacos-server:latest -> hub.docker.gemii.cc:7443/lizbasis/nacos-server:latest
# nfs-client
registry.cn-hangzhou.aliyuncs.com/open-ali/nfs-client-provisioner:latest -> docker push hub.docker.gemii.cc:7443/lizbasis/nacos-server:latest
# nacos-peer-finder-plugin
nacos/nacos-peer-finder-plugin:1.0 -> hub.docker.gemii.cc:7443/lizbasis/nacos-peer-finder-plugin:1.0

# 私有镜像打包完成之后需要在yaml文件中替换原有镜像

# nacos service 不能暴露集群ip，会导致nacos集群启动失败,nacos集群原理是通过同步文件内容

# 修改nacos-quick-start.yaml文件mysql配置，例子为开发环境
data:
  mysql.db.name: "nacos"
  mysql.port: "3306"
  mysql.user: "root"
  mysql.password: "gemii!@#$"
  mysql.host: "192.168.0.152"
- name: MYSQL_SERVICE_HOST
  valueFrom:
    configMapKeyRef:
    	name: nacos-cm
    	key: mysql.host

# 修改 deployment文件 NFS地址
        env:
        - name: PROVISIONER_NAME
          value: fuseim.pri/ifs
        - name: NFS_SERVER
          value: 192.168.0.142
        - name: NFS_PATH
          value: /data/nfs-share
      volumes:
      - name: nfs-client-root
        nfs:
          server: 192.168.0.142
          path: /data/nfs-share
# 替换所有yaml文件中的命名空间为当前的命名空间
```

nacos相关文件包含yaml与sql均可在`http://gitlab.gemii.cc:8000/liz-room-deploy/liz-room-kube/cloud-basis`中查看。

完成配置改造后按照下述安装步骤行动即可。



####安装步骤

1. 导入数据库初始化脚本
2. 创建角色：执行脚本`rbac.yaml`
3. 创建 ServiceAccount 和部署 NFS-Client Provisioner： 执行脚本`deployment.yaml`
4. 创建 NFS StorageClass：执行脚本`class.yaml`
5. 创建nacos：执行脚本`nacos-quick-start.yaml`，`nacos-service.yaml`，`nacos-ingress.yaml`



nacos地址

* dev：`http://nacos.dev.gemii.cc:58080/nacos`
* test：`http://nacos.test.gemii.cc:58080/nacos`

**在切换不同环境的时候需要注意域名与端口是否已被占用，根据情况更换就好**



[点击进入示例项目](http://gitlab.gemii.cc:8000/liz-cloud-basis/cloud-basis.git):`http://gitlab.gemii.cc:8000/liz-cloud-basis/cloud-basis.git`





### nacos 集群注册

* maven项目中加入以下依赖，已在父级pom中添加，无需再额外添加，需要去掉eureka依赖

```xml
<!-- 整合nacos服务注册发现配置客户端 -->
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
	<version>2.2.0.RELEASE</version>
</dependency>
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
	<version>2.2.0.RELEASE</version>
</dependency>
```

* 在bootstrap.properties中配置nacos、环境配置、项目名称

```properties
spring.profiles.active=${SPRING_PROFILES_ACTIVE:development}
# 项目名称
spring.application.name=basis-authuser
# 本地开发时需要替换域名与端口
spring.cloud.nacos.config.server-addr=nacos-headless:8848
spring.cloud.nacos.discovery.server-addr=nacos-headless:8848
# nacos配置文件类型
spring.cloud.nacos.config.file-extension=yaml
spring.cloud.nacos.config.group=WECHAT
```

* 启动文件增加注解`@EnableDiscoveryClient`

* 配置类上的注解：@Component、@RefreshScope

* docker镜像打包方式优化

  合理利用镜像缓存+外部依赖的方式减少推拉镜像的时间、降低带宽压力

* 配置文件外部化（这是为了满足nacos刷新配置，选择了外部依赖，配置也需要外部化，不然刷新配置会失败），需要注意的是仅仅将`bootstrap.properties`外部化，其余配置文件均保留在jar包内





### git：提交规范化

​															**type: subject**

![image-20200310190046254](/Users/gemii/Library/Application Support/typora-user-images/image-20200310190046254.png)

* type:
  * `feat`：新增功能；
  * `fix`：修复bug；
  * docs：修改文档；
  * refactor：代码重构，未新增任何功能和修复任何bug;
  * `build`：改变构建流程，新增依赖库、工具等（例如pom、配置文件修改）；
  * `style`：仅仅修改了空格、缩进等，不改变代码逻辑；
  * `perf`：改善性能和体现的修改；
  * chore：非src和test的修改；
  * test：测试用例的修改；
  * ci：自动化流程配置修改；
  * revert：回滚到上一个版本；





















