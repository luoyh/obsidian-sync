

# 一 文档概述
## 1.1 文档目的
 本文档用于北斗土桥物联网服务平台运维部署. 涵盖服务器规划,  服务器设置, 集群搭建, 服务部署, 日志查看, 系统监控, 服务升级等.
## 1.2 使用范围
 - 运维工程师
 - 技术支持人员
 - 测试工程师
## 1.3 环境说明
- 集群环境: RKE2 k8s集群
- 核心服务: 土桥行后端服务, 土桥行前端服务, 数据库, Flink实时计算, Dinky实时计算平台, kafka消息中间件, redis缓存中间件, dolphinscheduler离线计算, 北斗协议服务, Minio OSS服务.

# 二 服务器配置要求
## 2.1 硬件配置

> CPU要求支持avx2指令
>  使用`cat /proc/cpuinfo | grep avx2` 查看是否支持

| 节点         | CPU | 内存   | 硬盘    | 数量       | 说明                  |
| ---------- | --- | ---- | ----- | -------- | ------------------- |
| k8s-master | ≥4  | ≥8G  | ≥500G | 1或者3(奇数) | k8s的master节点, 要求奇数个 |
| k8s-worker | ≥8  | ≥32G | ≥500G | ≥3(奇数)   | k8s的worker节点,要求奇数个  |
| mysql      | ≥8  | ≥32G | ≥1T   | 1        | 业务数据库服务             |
| doris-fe   | ≥8  | ≥32G | ≥500G | 1或者3     | doris fe(大数据分析数据库)  |
| doris-be   | ≥8  | ≥64G | ≥2T   | 1或者3     | doris be(大数据分析数据库)  |
| minio      | ≥4  | ≥16G | ≥1T   | 1        | oss服务               |
| deps       | ≥8  | ≥32G | ≥1T   | 1        | 其它依赖服务(如OCR, 人脸失败)  |
| jtt        | ≥8  | ≥32G | ≥1T   | 2        | 北斗部标协议服务            |

# 三 服务资源包


| 名称                              | 说明             |
| ------------------------------- | -------------- |
| rke2.linux-amd64.tar.gz         | rke2安装包        |
| rke2-images.linux-amd64.tar.zst | rke2安装镜像资源包    |
| sha256sum-amd64.txt             | rke2验证文件       |
| rke2-install.sh                 | rke2安装脚本       |
| docker-28.5.2.tgz               | docker安装包      |
| tqbb-images.linux-amd64.tar     | 土桥行服务镜像资源包     |
| tqbb-deps.linux-amd64.tar       | 土桥行依赖镜像资源包     |
| tqbb-jt808.jar                  | 土桥行部标808协议解析服务 |
| mysql-8.0.43.tar                | mysql资源包       |
| doris-2.1.10.tar                | doris资源包       |
|                                 |                |

# 四 安装部署

## 4.1 rke2

### 在k8s-master节点服务器做以下操作

#### 1. 关闭selinux, 交换分区, 防火墙等

```bash
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce 0
systemctl stop firewalld && systemctl disable firewalld
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
# 设置hostname
hostnamectl set-hostname k8s-master1
```

#### 2. 配置ip转发, 新建/etc/sysctl.d/90-rke2.conf文件
```bash
cat <<EOF > /etc/sysctl.d/90-rke2.conf
net.ipv4.conf.all.forwarding=1
net.ipv6.conf.all.forwarding=1
EOF
```

重启服务器 `reboot`
#### 3. 上传文件

把`rke2.linux-amd64.tar.gz, rke2-images.linux-amd64.tar.zst, sha256sum-amd64.txt, rke2-install.sh` 上传到服务器目录`/root/rke2-artifacts`
```bash
[root@k8s-master1 ~]# mkdir /root/rke2-artifacts && cd /root/rke2-artifacts
# 然后把rke2.linux-amd64.tar.gz, rke2-images.linux-amd64.tar.zst, sha256sum-amd64.txt, rke2-install.sh上传到此目录
[root@k8s-master1 rke2-artifacts]# cd /root/rke2-artifacts
[root@k8s-master1 rke2-artifacts]# pwd
/root/rke2-artifacts

[root@k8s-master1 rke2-artifacts]# ll
-rw-r--r--. 1 root root     25288 11月 10 10:05 rke2-install.sh
-rw-r--r--. 1 root root 799019021 11月 10 10:05 rke2-images.linux-amd64.tar.zst
-rw-r--r--. 1 root root  39700180 11月 10 10:05 rke2.linux-amd64.tar.gz
-rw-r--r--. 1 root root      4252 11月 10 10:05 sha256sum-amd64.txt

```

#### 4. 配置
```bash

[root@k8s-master1 rke2-artifacts]# mkdir -p /var/lib/rancher/rke2/agent/images/
[root@k8s-master1 rke2-artifacts]# touch /var/lib/rancher/rke2/agent/images/.cache.json
[root@k8s-master1 rke2-artifacts]# cp rke2-images.linux-amd64.tar.zst /var/lib/rancher/rke2/agent/images/rke2-images.linux-amd64.tar.zst

```

### 5. 安装
```bash
INSTALL_RKE2_ARTIFACT_PATH=/root/rke2-artifacts sh rke2-install.sh
```

#### 6. 配置内置mirror

```bash
# 配置内置mirror, 创建/etc/rancher/rke2/config.yaml, 输入embedded-registry: true
[root@k8s-master1 rke2-artifacts]# cat /etc/rancher/rke2/config.yaml 
embedded-registry: true

# 创建/etc/rancher/rke2/registries.yaml, 输入如下内容
[root@k8s-master1 rke2-artifacts]# cat /etc/rancher/rke2/registries.yaml 
mirrors:
  docker.io:
  registry.k8s.io:
```

#### 7. 启动

```bash
systemctl enable rke2-server.service
systemctl start rke2-server.service
```

#### 8. 查看输出
```bash
journalctl -fu rke2-server
```

等几分钟后无输出则是启动成功, 使用`systemctl status rke2-server.service`查看状态

#### 9. 环境变量配置
```bash
# 在~/.bashrc加入如下内容
[root@k8s-master1 ~]# vim ~/.bashrc
export PATH=/var/lib/rancher/rke2/bin:$PATH
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export CRI_CONFIG_FILE=/var/lib/rancher/rke2/agent/etc/crictl.yaml
export CTRD_ADDRESS=/run/k3s/containerd/containerd.sock

# 生效
[root@k8s-master1 ~]# . ~/.bashrc

# 查看k8s节点, STATUS状态是Ready就成功了
[root@k8s-master1 ~]# kubectl get nodes
NAME          STATUS   ROLES                AGE    VERSION
k8s-master1   Ready    control-plane,etcd   154m   v1.34.1+rke2r1
```

#### 10.其它master节点(如果需要)
1. 在k8s-master1查看token
```bash 
cat /var/lib/rancher/rke2/server/node-token
K10829816a052835a015d6188c4a701cc31eca8e4aeee355e8597d87fa1068bf213::server:e8462d3b06a44f6ebc90891bb5542be1
```
2. 在k8s-master2上面执行qia