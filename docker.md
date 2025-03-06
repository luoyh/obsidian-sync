
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
# 1
for i in $(docker container ls --format "{{.ID}}"); do docker inspect -f '{{.State.Pid}} {{.Name}}' $i; done
# 2
pgrep containerd-shim
# 3
pgrep -P $pid
# 4
for i in $(pgrep containerd-shim); do pgrep -P $i; done

for i in $(docker container ls --format "{{.ID}}"); do t=$(docker inspect -f '{{.State.Pid}} {{.Name}}' $i); echo "$t, `pgrep -P $(echo $t | awk '{print $1}')`"; done
```