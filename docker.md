
## docker set proxy and wsl set clash

### docker配置service启动
```bash
# 下载docker二进制
wget https://download.docker.com/linux/static/stable/x86_64/docker-28.2.2.tgz
tar zxvf docker-28.2.2.tgz
cp docker/* /usr/bin/
# 下载service
wget https://raw.githubusercontent.com/moby/moby/master/contrib/init/systemd/docker.service
wget https://raw.githubusercontent.com/moby/moby/master/contrib/init/systemd/docker.socket
wget https://raw.githubusercontent.com/containerd/containerd/refs/heads/main/containerd.service
# 修改docker.socket的用户组为root
SocketGroup=root
# 可以看下containerd.service的containerd是不是指向/usr/bin/containerd

# 复制文件到system
cp docker.service /etc/systemd/system/
cp docker.socket /etc/systemd/system/
cp containerd.service /etc/systemd/system/

# 看下是否授权
chmod a+x /etc/systemd/system/docker.service
# 查看依赖
systemctl list-dependencies docker.service

# 修改默认docker目录
vim /etc/docker/daemon.json
{
    "data-root": "/data/docker",
    "proxies": {
        "http-proxy": "http://host.local:7890",
        "https-proxy": "http://host.local:7890",
        "no-proxy": "localhost,127.0.0.1"
    }
}

# 可使用dockerd检查是否成功
/usr/bin/dockerd

# 安装iptables
apt install -y iptables

# 启动
systemctl start docker
# service docker start

```

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

## go-wkhtmltopdf

```Dockerfile
FROM ubuntu:24.04
RUN apt-get update \
  && apt-get install -y wget curl tar \
  && apt-get install -y fontconfig locales language-pack-zh-hans

ENV LANG zh_CN.UTF-8
ENV LC_ALL=zh_CN.UTF-8
RUN mkdir -p /opt/lib
RUN mkdir -p /opt/app
COPY wkhtmltox_0.12.6.1-2.jammy_amd64.deb /opt/lib/wkhtmltox_0.12.6.1-2.jammy_amd64.deb
RUN apt-get install -y --no-install-recommends /opt/lib/wkhtmltox_0.12.6.1-2.jammy_amd64.deb
COPY simkai.ttf /usr/share/fonts/truetype/simkai.ttf
COPY main /opt/app/main
RUN rm -rf /opt/lib/wkhtmltox_0.12.6.1-2.jammy_amd64.deb

WORKDIR /opt/app
expose 8080
CMD ["./main"]

```