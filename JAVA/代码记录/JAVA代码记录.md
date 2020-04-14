* 通用docker镜像

```dockerfile
FROM openjdk:8-jdk-alpine
VOLUME /tmp
COPY --from=hengyunabc/arthas:latest /opt/arthas /opt/arthas
ADD *.jar app.jar
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["java","-Duser.timezone=Asia/Shanghai","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```



* jar包配置与依赖外部化

```xml
	<build>
		<resources>
			<!-- 控制资源文件的拷贝 -->
			<resource>
				<directory>src/main/resources</directory>
				<includes>
					<include>**/*.*</include>
				</includes>
				<filtering>false</filtering>
				<targetPath>${project.build.directory}/config</targetPath>
			</resource>
			<resource>
				<directory>src/main/resources</directory>
				<includes>
					<include>**/*.yml</include>
					<include>**/*.xml</include>
					<include>**/*.properties</include>
					<include>**/*_cert.p12</include>
				</includes>
				<filtering>false</filtering>
			</resource>
		</resources>
		<pluginManagement>
			<plugins>
				<!-- 设置 SpringBoot 打包插件不包含任何 Jar 依赖包 -->
				<plugin>
					<groupId>org.springframework.boot</groupId>
					<artifactId>spring-boot-maven-plugin</artifactId>
					<configuration>
						<includes>
							<include>
								<groupId>nothing</groupId>
								<artifactId>nothing</artifactId>
							</include>
						</includes>
					</configuration>
				</plugin>
				<!-- 设置应用 Main 参数启动依赖查找的地址指向外部 lib 文件夹 -->
				<plugin>
					<groupId>org.apache.maven.plugins</groupId>
					<artifactId>maven-jar-plugin</artifactId>
					<configuration>
						<archive>
							<manifest>
								<addClasspath>true</addClasspath>
								<classpathLayoutType>custom</classpathLayoutType>
								<classpathPrefix>lib/</classpathPrefix>
								<customClasspathLayout>$${artifact.artifactId}-$${artifact.version}$${dashClassifier?}.$${artifact.extension}</customClasspathLayout>
							</manifest>
						</archive>
					</configuration>
				</plugin>
				<!-- 设置将 lib 拷贝到应用 Jar 外面 -->
				<plugin>
					<groupId>org.apache.maven.plugins</groupId>
					<artifactId>maven-dependency-plugin</artifactId>
					<executions>
						<execution>
							<id>copy-dependencies</id>
							<phase>prepare-package</phase>
							<goals>
								<goal>copy-dependencies</goal>
							</goals>
							<configuration>
								<outputDirectory>
									${project.build.directory}/lib
								</outputDirectory>
							</configuration>
						</execution>
					</executions>
				</plugin>
				<plugin>
					<groupId>com.spotify</groupId>
					<artifactId>docker-maven-plugin</artifactId>
					<version>0.4.13</version>
					<executions>
						<execution>
							<id>build-image</id>
							<phase>package</phase>
							<goals>
								<goal>build</goal>
							</goals>
						</execution>
						<execution>
							<id>tag-image</id>
							<phase>package</phase>
							<goals>
								<goal>tag</goal>
							</goals>
							<configuration>
								<image>${docker.image.prefix}/${project.artifactId}:${project.version}</image>
								<newName>${docker.registry}${docker.image.prefix}/${project.artifactId}:${kubeimage.version}</newName>
							</configuration>
						</execution>
						<execution>
							<id>push-image</id>
							<phase>install</phase>
							<goals>
								<goal>push</goal>
							</goals>
							<configuration>
								<imageName>${docker.registry}${docker.image.prefix}/${project.artifactId}:${kubeimage.version}</imageName>
							</configuration>
						</execution>
					</executions>
					<configuration>
						<serverId>${docker.mvnhup}</serverId>
						<registryUrl>${docker.registry.url}</registryUrl>
						<imageName>${docker.image.prefix}/${project.artifactId}:${project.version}</imageName>
						<dockerDirectory>${project.basedir}/src/main/docker</dockerDirectory>
						<forceTags>true</forceTags>
						<resources>
							<resource>
								<targetPath>/</targetPath>
								<directory>${project.build.directory}</directory>
								<include>${project.build.finalName}.jar</include>
							</resource>
							<resource>
								<targetPath>/</targetPath>
								<directory>${project.build.directory}</directory>
								<include>lib/*</include>
							</resource>
							<resource>
								<targetPath>/</targetPath>
								<directory>${project.build.directory}</directory>
								<include>config/*</include>
							</resource>
						</resources>
					</configuration>
				</plugin>
			</plugins>
		</pluginManagement>
	</build>
```

* 配套 dockerfile

```dockerfile
FROM openjdk:8-jdk-alpine
VOLUME /tmp
COPY --from=docker.io/hengyunabc/arthas /opt/arthas /opt/arthas
RUN apk add tini
ENTRYPOINT ["/sbin/tini", "--"]
ADD lib/ lib/
ADD config/ config/
ADD *.jar app.jar
CMD ["java","-Duser.timezone=Asia/Shanghai","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

* fastjson 列表转换

```java
parseObject
new TypeReference<T>() {
		}
```

