

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
```bash
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce 0
systemctl stop firewalld && systemctl disable firewalld
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.conf.all.forwarding=1
net.ipv6.conf.all.forwarding=1
EOF
```
