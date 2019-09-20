## 安装sonarqube的原因

* 通过sonarqube能进一步避免可能发生的bug
* 团队内部代码质量的一个分析利器
* 代码规范

## 目标
- [X] 在 k8s 中运行 sonarqube
- [X] 实现 k8s 中 sonarqube 数据持久化
- [ ] ~~集成现有 gitlab~~（暂时不考虑）
- [X] 兼容已集成了pipeline的项目
## 在 k8s 中运行 sonarqube
### 安装
1. 在 mysql 中创建 sonar 用户
2. 选择 sonarqube:7.7-community
3. sonarqube:7.7-community 上传到私库，并打tag：`hub.docker.gemii.cc:7443/lizroom/sonarqube:7.7`
4. 通过 env 设置数据库属性
5. 增加 initContainers 设置虚拟内存 与 jvm 属性。
这么做的原因可参考下方 **问题记录** ：`sonarqube 内部es版本问题`。

* initContainers
```
     initContainers:
        - name: init-sysctl
          image: busybox:1.27.2
          command:
            - sysctl
            - -w
            - vm.max_map_count=262144
          env:
            - name: ES_JAVA_OPTS
              value: -Xms2g -Xmx2g
            - name: bootstrap.memory_lock
              value: "true"
            - name: discovery.type
              value: single-node
          securityContext:
            privileged: true
```
### 暴露端口
* sonar-svc.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: sonar
spec:
  ports:
      - port: 9000
      protocol: TCP
      targetPort: 30090
  selector:
    app: sonar
```

## 持久化
### 安装nfs
为了实现持久化，需要外部文件系统来作为存储介质，所以在 k8s 所有节点上安装了 NFS 。
nfs的安装可参考网上配置。
* 共享目录
开发服务器作为主节点共享了文件夹`data/k8s/sonarqube/` 。
### 创建pv与pvc
* sonar-storage.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sonar
  labels:
    app: sonar
spec:
  accessModes:       
    - ReadWriteOnce
  capacity:          
    storage: 5Gi
  mountOptions:   #NFS挂在选项
    - hard  
  nfs:            #NFS设置
    server: 192.168.0.142
    path: /root/data/k8s/sonarqube
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: sonar
  labels:
    app: sonar
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi  #设置大小为5G
  selector:
    matchLabels:
      app: sonar
```
创建完成后可以根据命令 `kubectl get pv` 和 `kubectl get pvc` 查看状态。`Bound` 说明 `pv` 与 `pvc` 成功绑定了
![title](https://raw.githubusercontent.com/YpandaGit/pics/master/gemii/2019/09/03/1567479869435-1567479869470.png)

### 容器绑定pvc
```
      containers:
        - name: sonar
          image: hub.docker.gemii.cc:7443/lizroom/sonarqube:7.7
          resources:
            requests:
              cpu: 500m
              memory: 1024Mi
            limits:
              cpu: 2000m
              memory: 2048Mi
          volumeMounts:
          - mountPath: "/opt/sonarqube/data/"
            name: nfs
            subPath: data
          - mountPath: "/opt/sonarqube/extensions/"
            name: nfs
            subPath: extensions
          env:
          - name: "SONARQUBE_JDBC_USERNAME"
            value: "sonar"
          - name: "SONARQUBE_JDBC_URL"
            value: "jdbc:mysql://192.168.0.152:3306/liz-sonar?useSSL=false&useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance"
          - name: "SONARQUBE_JDBC_PASSWORD"
            value: "sonar"
          ports:
          - containerPort: 9000
            protocol: TCP
      volumes:
      - name: nfs
        persistentVolumeClaim:
          claimName: sonar
```

## 使用方法
在jenkins上配置全局属性
```
            <profile>
            <id>sonar</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
            <sonar.jdbc.url>jdbc:mysql://192.168.0.152:3306/liz-sonar</sonar.jdbc.url>
            <sonar.jdbc.username>sonar</sonar.jdbc.username>
            <sonar.jdbc.password>sonar</sonar.jdbc.password>
                <!-- Optional URL to server. Default value is http://localhost:9000 -->
                <sonar.host.url>
                  http://192.168.0.142:30090
                </sonar.host.url>
            </properties>
        </profile>
```
接下来在项目中的 pom 文件里添加 plugin
```
 			<plugin>
				<groupId>org.sonarsource.scanner.maven</groupId>
				<artifactId>sonar-maven-plugin</artifactId>
				<version>3.6.1.1688</version>
				<executions>
					<execution>
						<id>s2</id>
						<phase>package</phase>
						<goals>
							<goal>sonar</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
```

在 jenkins 构建的时候就会自动分析了。

## 总结
关于 sonarqube 的 K8S 配置总共有三个文件，分别是`sonar-rc.yaml`、`sonar-storage.yaml`、`sonar-svc.yaml`，在开发环境（142）的目录`/root/kuberepo/liz-room-kube/liz-sonarkube`。

* sonar用户名：admin，密码：admin
* sonarqube地址：[http://192.168.0.142:30090/projects](http://192.168.0.142:30090/projects)

### 问题记录

* sonarqube 版本选择
> 根据官方描述，为了保证更好的体验与性能，sonarqube7.9 及以上不再支持 mysql ，全面投入 postgreSql ，所以针对于 7.9+ 版本请自行安装 postgreSQL 。

* sonarqube 内部es版本问题
当 es 是 6.0+ 版本时，在不做任何特殊处理的情况下，会出现以下错误：
```
WARN  es[][o.e.d.c.s.Settings] [http.enabled] setting was deprecated in Elasticsearch and will be removed in a future release! See the breaking changes documentation for the next major version.
ERROR: [1] bootstrap checks failed
[1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```

### 参考文档
* [Kubernetes部分Volume类型介绍及yaml示例](https://www.cnblogs.com/zhenyuyaodidiao/p/6594541.html)
* [data-persistence](https://github.com/gregbkr/kubernetes-kargo-logging-monitoring#8-data-persistence)
* [Elasticsearch 6.X #33](https://github.com/gregbkr/kubernetes-kargo-logging-monitoring#8-data-persistence)
* [running-sonarqube-on-azure-kubernetes](https://medium.com/@akamenev/running-sonarqube-on-azure-kubernetes-92a1b9051120)