
## docker set proxy and wsl set clash

```bash

vim ~/.bashrc
:wq
hostip=172.31.80.1
export https_proxy="http://${hostip}:7897"
export http_proxy="http://${hostip}:7897"
export all_proxy="socks5://${hostip}:7897"

cat /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=172.31.80.1:7897" 
Environment="HTTPS_PROXY=172.31.80.1:7897"
Environment="NO_PROXY=localhost,127.0.0.1"

systemctl daemon-reload && service docker restart
# systemctl daemon-reload && systemctl restart docker

# check
docker info
>
 HTTP Proxy: 172.31.80.1:7897
 HTTPS Proxy: 172.31.80.1:7897
 No Proxy: localhost,127.0.0.1


# 通过主机进程查看docker容器
# 查看容器的pid
for i in $(docker container ls --format "{{.ID}}"); do docker inspect -f '{{.State.Pid}} {{.Name}}' $i; done
# 通过pgrep查看
pgrep containerd-shim
# 通过容器pid查看
pgrep -P $pid
# 4
for i in $(pgrep containerd-shim); do pgrep -P $i; done

for i in $(docker container ls --format "{{.ID}}"); do t=$(docker inspect -f '{{.State.Pid}} {{.Name}}' $i); echo "$t, `pgrep -P $(echo $t | awk '{print $1}')`"; done


ps --ppid $DID -o pid,ppid,cmd

# whole
for i in $(docker container ls --format "{{.ID}}"); do t=$(docker inspect -f '{{.State.Pid}} {{.Name}}' $i); echo "$t, `ps --ppid $(echo $t | awk '{print $1}') -o pid,ppid,cmd`"; done

```


## openjdk-alpine

```dockerfile
FROM alpine:3.22
#ADD jdk-17.0.2 /opt/jdk
#COPY sgerrand.rsa.pub /etc/apk/keys/sgerrand.rsa.pub
#COPY glibc-2.35-r0.apk /opt/glibc-2.35-r0.apk
#COPY glibc-bin-2.35-r0.apk /opt/glibc-bin-2.35-r0.apk
#COPY glibc-i18n-2.35-r0.apk /opt/glibc-i18n-2.35-r0.apk

RUN echo http://mirrors.aliyun.com/alpine/v3.22/main/ > /etc/apk/repositories && \
    echo http://mirrors.aliyun.com/alpine/v3.22/community/ >> /etc/apk/repositories
RUN apk update && apk upgrade
RUN apk add --no-cache curl wget ca-certificates libgcc libstdc++ ncurses-libs
#RUN apk add --force-overwrite /opt/glibc-2.35-r0.apk /opt/glibc-bin-2.35-r0.apk /opt/glibc-i18n-2.35-r0.apk

ENV JAVA_HOME=/usr/lib/jvm/default-jvm

RUN apk add --no-cache --update openjdk21 && \
    ln -sf "${JAVA_HOME}/bin/"* "/usr/bin/"
RUN apk --no-cache add ttf-dejavu fontconfig
COPY simkai.ttf /usr/share/fonts/TTF/simkai.ttf
#ENV JAVA_HOME=/opt/jdk
#ENV PATH=.:$JAVA_HOME/bin:$PATH
#RUN ln -sf "${JAVA_HOME}/bin/"* "/usr/bin/"

# CMD ["/bin/sh"]
#ENTRYPOINT [ "java" ]

```


## openjdk-alpine-wkhtmltopdf

```dockerfile
FROM surnet/alpine-wkhtmltopdf:3.21.3-024b2b2-full
FROM alpine:3.21

RUN echo http://mirrors.aliyun.com/alpine/v3.21/main/ > /etc/apk/repositories && \
    echo http://mirrors.aliyun.com/alpine/v3.21/community/ >> /etc/apk/repositories
RUN apk update && apk upgrade
RUN apk add --no-cache curl wget ca-certificates libgcc libstdc++ ncurses-libs bash

ENV JAVA_HOME=/usr/lib/jvm/default-jvm

RUN apk add --no-cache --update openjdk21 && \
    ln -sf "${JAVA_HOME}/bin/"* "/usr/bin/"

RUN apk --no-cache add ttf-dejavu fontconfig

COPY --from=surnet/alpine-wkhtmltopdf:3.21.3-024b2b2-full /bin/wkhtmltopdf /bin/wkhtmltopdf
COPY --from=surnet/alpine-wkhtmltopdf:3.21.3-024b2b2-full /bin/wkhtmltoimage /bin/wkhtmltoimage
COPY --from=surnet/alpine-wkhtmltopdf:3.21.3-024b2b2-full /lib/libwkhtmltox* /lib/
COPY simkai.ttf /usr/share/fonts/TTF/simkai.ttf

RUN rm -rf /var/cache/apk/*

```

## springboot-wkhtml2pdf

```dockerfile
FROM openjdk-wkhtml2pdf:21
RUN mkdir -p /opt && chmod -R 777 /opt
COPY app.jar /opt/app.jar

WORKDIR /opt
EXPOSE 80

CMD ["java","-Dserver.port=80","-Dcom.tqbb.html2pdf.localTest=false", "-jar", "/opt/app.jar"]

```


## ubuntu docker 支持中文

```bash
docker pull ubuntu:24.04
docker pull ubuntu:latest
docker run -ti -d --name ubt ubuntu:latest bash

docker exec -ti ubt bash
> apt get update
> apt install -y language-pack-zh-hans
> locale
> vim ~/.bashrc
#> export LANG=zh_CN.UTF-8
#> export LC_ALL=zh_CN.UTF-8
> source ~/.bashrc


```