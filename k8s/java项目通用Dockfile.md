引入了arthas方便线上问题排查
```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
COPY --from=hengyunabc/arthas:latest /opt/arthas /opt/arthas
ADD *.jar app.jar
#RUN bash -c 'touch /app.jar'
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["java","-Duser.timezone=Asia/Shanghai","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```