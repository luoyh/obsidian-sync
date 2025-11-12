

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


| 名称                                        | 说明             |
| ----------------------------------------- | -------------- |
| rke2.linux-amd64.tar.gz                   | rke2安装包        |
| rke2-images.linux-amd64.tar.zst           | rke2安装镜像资源包    |
| sha256sum-amd64.txt                       | rke2验证文件       |
| rke2-install.sh                           | rke2安装脚本       |
| docker-28.5.2.tgz                         | docker安装包      |
| tqbb-images.linux-amd64.tar               | 土桥行服务镜像资源包     |
| tqbb-deps.linux-amd64.tar                 | 土桥行依赖镜像资源包     |
| tqbb-jt808.jar                            | 土桥行部标808协议解析服务 |
| mysql.tgz                                 | mysql资源包       |
| doris-2.1.10.tgz                          | doris资源包       |
| tqbb-apps.yaml                            | 土桥行服务配置        |
| helm-v3.19.1-linux-amd64.tar.gz           | helm           |
| flink-kubernetes-operator-1.13.0-helm.tgz | flink helm安装包  |
| flink-cluster.yaml                        | flink集群配置      |
| openjdk-17.0.2_linux-x64_bin.tar.gz       | jdk包           |

# 四 安装部署

## 4.1 rke2

### master

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

#### 5. 配置内置mirror

```bash
# 配置内置mirror, 创建/etc/rancher/rke2/config.yaml, 输入embedded-registry: true
[root@k8s-master1 rke2-artifacts]# mkdir -p /etc/rancher/rke2/
[root@k8s-master1 rke2-artifacts]# cat <<EOF > /etc/rancher/rke2/config.yaml 
embedded-registry: true
EOF

# 创建/etc/rancher/rke2/registries.yaml, 输入如下内容
[root@k8s-master1 rke2-artifacts]# cat <<EOF > /etc/rancher/rke2/registries.yaml 
mirrors:
  docker.io:
  registry.k8s.io:
EOF
```

#### 6. 安装
```bash
INSTALL_RKE2_ARTIFACT_PATH=/root/rke2-artifacts sh rke2-install.sh
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
2. 在k8s-master2上面执行前面(1~5)步骤
3. 在/etc/rancher/rke2/config.yaml新增如下内容
```bash
server: https://k8s-master1:9345
token: K10829816a052835a015d6188c4a701cc31eca8e4aeee355e8597d87fa1068bf213::server:e8462d3b06a44f6ebc90891bb5542be1
```
4. 安装
```bash
INSTALL_RKE2_ARTIFACT_PATH=/root/rke2-artifacts sh rke2-install.sh
```
5. 启动服务
```bash
systemctl enable rke2-server.service
systemctl start rke2-server.service
```
6. 在k8s-master1查看节点状态
```bash
[root@k8s-master1 ~]# kubectl get nodes
NAME          STATUS   ROLES                AGE    VERSION
k8s-master1   Ready    control-plane,etcd   154m   v1.34.1+rke2r1
k8s-master2   Ready    control-plane,etcd   15m    v1.34.1+rke2r1

```

### worker

> 在worker节点执行上面master的1~5步骤

#### 修改config.yaml
新增server和token
```bash
[root@k8s-worker1 rke2-artifacts]# cat /etc/rancher/rke2/config.yaml 
embedded-registry: true
server: https://192.168.3.201:9345
token: K10829816a052835a015d6188c4a701cc31eca8e4aeee355e8597d87fa1068bf213::server:e8462d3b06a44f6ebc90891bb5542be1

```
#### 安装
```bash
INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_ARTIFACT_PATH=/root/rke2-artifacts sh rke2-install.sh
```

#### 启动
```bash
systemctl enable rke2-agent.service
systemctl start rke2-agent.service
```

#### 查看状态
```bash
journalctl -fu rke2-agent
```

#### 查看节点
在k8s-master1节点执行
```bash
[root@k8s-master1 ~]# kubectl get nodes
NAME   STATUS   ROLES                AGE    VERSION
k8s-master1   Ready    control-plane,etcd   154m   v1.34.1+rke2r1
k8s-master2   Ready    control-plane,etcd   45m    v1.34.1+rke2r1
k8s-worker1   Ready    <none>               10m   v1.34.1+rke2r1
```

## 4.2 安装服务

> 在k8s-master1上执行

### 1. 导入镜像
```bash
ctr --address /run/k3s/containerd/containerd.sock -n k8s.io image import tqbb-images.linux-amd64.tar
```

### 2. 查看导入结果

```bash
ctr --address /run/k3s/containerd/containerd.sock -n k8s.io image ls | grep tqbb
```
### 3. 启动服务
```bash
kubectl apply -f tqbb-apps.yaml
```

### 4. 查看结果
```bash
kubectl get pods -n tqbb -o wide
# 所有状态为Running则成功, 打开网页: http://k8s-master1/
```

## 4.3 mysql安装
### 1. 安装docker
```bash
# 解压docker
tar zxvf docker-28.5.2.tgz
cd docker
# 设置可执行
chmod +x docker/*
# 复制到/usr/bin下
cp docker/* /usr/bin/
# 复制service到systemd
cp config/* /etc/systemd/system/
# 创建用户组
groupadd docker
# 启动docker
systemctl daemon-reload
systemctl enable docker
systemctl start docker

```

### 2. 安装mysql
```bash
# 解压mysql
tar zxvf mysql.tgz
# 进入解压后的mysql目录
cd mysql
mkdir -p /data/mysql
# 复制mysql文件
cp -rf data /data/mysql/
# 导入镜像
docker load -i mysql-8.0.43.tar
# 启动mysql
docker run -dti --memory 32g -p 3306:3306 --restart always -v /data/mysql/data:/var/lib/mysql --name mysql mysql:8.0.43
# 查看结果
docker ps | grep mysql
# 验证
docker exec -ti mysql mysql --version
```

## 4.4 doris安装

### jdk安装
```bash
# 在fe和be服务器都需要安装jdk
tar zxvf openjdk-17.0.2_linux-x64_bin.tar.gz -C /data
# 配置环境变量, 在/etc/profile最后面加上
export JAVA_HOME=/data/jdk-17.0.2
export PATH=$JAVA_HOME/bin:$PATH

# 环境变量生效
. /etc/profile
```

### 第 1 步：部署 FE Master 节点

1. **创建元数据路径**

   在部署 FE 时，建议与 BE 节点数据存储在不同的硬盘上。

   在解压安装包时，会默认附带 doris-meta 目录，建议为元数据创建独立目录，并将其软连接到默认的 `doris-meta` 目录。生产环境应使用单独的 SSD 硬盘，不建议将其放在 Doris 安装目录下；开发和测试环境可以使用默认配置。

   ```SQL
   ## Use a separate disk for FE metadata
   mkdir -p <doris_meta_created>
      
   ## Create FE metadata directory symlink
   ln -s <doris_meta_created> <doris_meta_original>
   ```

2. **修改 FE 配置文件**

   FE 的配置文件在 FE 部署路径下的 conf 目录中，启动 FE 节点前需要修改 `conf/fe.conf`。

    在部署 FE 节点之前，建议调整以下配置：

   ```Bash
   ## modify Java Heap
   JAVA_OPTS="-Xmx16384m -XX:+UseMembar -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=7 -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSClassUnloadingEnabled -XX:-CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=80 -XX:SoftRefLRUPolicyMSPerMB=0 -Xloggc:$DORIS_HOME/log/fe.gc.log.$DATE"
      
   ## modify case sensitivity
   lower_case_table_names = 1
     
   ## modify network CIDR 
   priority_networks = 10.1.3.0/24
      
   ## modify Java Home
   JAVA_HOME = <your-java-home-path>
   ```

| 参数                     | 修改建议                                    |
| ---------------------- | --------------------------------------- |
| JAVA_OPTS              | 指定参数 `-Xmx` 调整 Java Heap，生产环境建议 16G 以上。 |
| lower_case_table_names | 设置大小写敏感，建议调整为 1，即大小写不敏感。                |
| priority_networks      | 网络 CIDR，更具网络 IP 地址指定。在 FQDN 环境中可以忽略。    |
| JAVA_HOME              | 建议 Doris 使用独立于操作系统的 JDK 环境。             |
   
3. **启动 FE 进程**

   通过以下命令可以启动 FE 进程

   ```Shell
   bin/start_fe.sh --daemon
   ```

   FE 进程将在后台启动，日志默认保存在 `log/` 目录。如果启动失败，可通过查看 `log/fe.log` 或 `log/fe.out` 文件获取错误信息。

4. **检查 FE 启动状态**

   通过 MySQL 客户端连接 Doris 集群，初始化用户为 `root`，默认密码为空。

   ```SQL

   mysql -uroot -P<fe_query_port> -h<fe_ip_address>

   ```

   链接到 Doris 集群后，可以通过 `show frontends` 命令查看 FE 的状态，通常要确认以下几项

   - Alive 为 `true` 表示节点存活；

   - Join 为 `true` 表示节点加入到集群中，但不代表当前还在集群内（可能已失联）；

   - IsMaster 为 true 表示当前节点为 Master 节点。

### 第 2 步：部署 FE 集群（可选）

生产环境建议至少部署 3 个节点。在部署过 FE Master 节点后，需要再部署两个 FE Follower 节点。

1. **创建元数据目录**

   参考部署 FE Master 节点，创建 doris-meta 目录

2. **修改 FE Follower 节点配置文件**

   参考部署 FE Master 节点，修改 FE Follower 节点配置文件。通常情况下，可以直接复制 FE Master 节点的配置文件。

3. **在 Doris 集群中注册新的 FE Follower 节点**

   在启动新的 FE 节点前，需要先在 FE 集群中注册新的 FE 节点。

   ```Bash
   ## connect a alive FE node
   mysql -uroot -P<fe_query_port> -h<fe_ip_address>
   
   ## registe a new FE follower node
   ALTER SYSTEM ADD FOLLOWER "<fe_ip_address>:<fe_edit_log_port>"
   ```

   如果要添加 observer 节点，可以使用 `ADD OBSERVER` 命令

   ```Bash
   ## register a new FE observer node
   ALTER SYSTEM ADD OBSERVER "<fe_ip_address>:<fe_edit_log_port>"
   ```

   :::caution 注意
   - FE Follower（包括 Master）节点的数量建议为奇数，建议部署 3 个组成高可用模式。

   - 当 FE 处于高可用部署时（1 个 Master，2 个 Follower），我们建议通过增加 Observer FE 来扩展 FE 的读服务能力
   :::
4. **启动 FE Follower 节点**

   通过以下命令，可以启动 FE Follower 节点，并自动同步元数据。

   ```Shell
   bin/start_fe.sh --helper <helper_fe_ip>:<fe_edit_log_port> --daemon
   ```

   其中，helper_fe_ip 是 FE 集群中任何存活节点的 IP 地址。`--helper` 参数仅在第一次启动 FE 时需要，之后重启无需指定。

5. **判断 Follower 节点状态**

   与 FE Master 节点状态判断相同，添加 Follower 节点后，可通过 `show frontends` 命令查看节点状态，IsMaster 应为 false。

### 第 3 步：部署 BE 节点

1. **创建数据目录**

   BE 进程应用于数据的计算与存储。数据目录默认放在 `be/storage` 下。生产环境通常将 BE 数据与 BE 部署文件分别存储在不同的硬盘上。BE 支持数据分布在多盘上以更好的利用多块硬盘的 I/O 能力。

   ```Bash
   ## Create a BE data storage directory on each data disk
   mkdir -p <be_storage_root_path>
   ```

2. **修改 BE 配置文件**

   BE 的配置文件在 BE 部署路径下的 conf 目录中，启动 BE 节点前需要修改 `conf/be.conf`。

   ```Bash
   ## modify storage path for BE node
   
   storage_root_path=/home/disk1/doris,medium:HDD;/home/disk2/doris,medium:SSD
   
   ## modify network CIDR 
   
   priority_networks = 10.1.3.0/24
   
   ## modify Java Home in be/conf/be.conf
   
   JAVA_HOME = <your-java-home-path>
   ```
   
   参数解释如下：

| 参数        | 修改建议                                                  |
| --------------------------------- | ------------------------------------- |
| priority_networks | 网络 CIDR，更具网络 IP 地址指定。在 FQDN 环境中可以忽略。 |
| JAVA_OPTS                                                    | 指定参数 `-Xmx` 调整 Java Heap，生产环境建议 2G 以上。   |
| JAVA_HOME                                                    | 建议 Doris 使用独立于操作系统的 JDK 环境。                |

3. **在 Doris 中注册 BE 节点**

   在启动 BE 节点前，需要先在 FE 集群中注册该节点：

   ```Bash
   ## connect a alive FE node
   mysql -uroot -P<fe_query_port> -h<fe_ip_address>
      
   ## registe BE node
   ALTER SYSTEM ADD BACKEND "<be_ip_address>:<be_heartbeat_service_port>"
   ```

4. **启动 BE 进程**

   通过以下命令可以启动 BE 进程：

   ```Bash
   bin/start_be.sh --daemon
   ```

   BE 进程在后台启动，日志默认保存在 `log/` 目录。如果启动失败，请检查 `log/be.log` 或 `log/be.out` 文件以获取错误信息。

5. **查看 BE 启动状态**

   连接 Doris 集群后，可通过 `show backends` 命令查看 BE 节点的状态。

   ```Bash
   ## connect a alive FE node
   mysql -uroot -P<fe_query_port> -h<fe_ip_address>
      
   ## check BE node status
   show backends;
   ```

   通常情况下需要注意以下几项状态：

   - Alive 为 true 表示节点存活

   - TabletNum 表示该节点上的分片数量，新加入的节点会进行数据均衡，TabletNum 逐渐趋于平均。

### 第 4 步：验证集群正确性

1. **登录数据库**

   使用 MySQL 客户端登录 Doris 集群。

   ```Bash
   ## connect a alive fe node
   mysql -uroot -P<fe_query_port> -h<fe_ip_address>
   ```

2. **检查 Doris 安装信息**

   通过 `show frontends` 与 `show backends` 可以查看数据库各实例的信息。
   
   ```SQL
   -- check fe status
   show frontends \G

   -- check be status
   show backends \G
   ```

4. **修改 Doris 集群密码**

   在创建 Doris 集群时，系统会自动创建一个名为 `root` 的用户，并默认设置其密码为空。为了提高安全性，建议在集群创建后立即为 `root` 用户设置一个新密码。

   ```SQL
   -- check the current user
   select user();  
   +------------------------+  
   | user()                 |  
   +------------------------+  
   | 'root'@'192.168.88.30' |  
   +------------------------+  
        
   -- modify the password for current user
   SET PASSWORD = PASSWORD('doris_new_passwd');
   ```

5. **创建测试表并插入数据**

   为了验证集群的正确性，可以在新创建的集群中创建一个测试表，并插入测试数据。

   ```SQL
   -- create a test database
   create database testdb;
    
   -- create a test table
   CREATE TABLE testdb.table_hash
   (
       k1 TINYINT,
       k2 DECIMAL(10, 2) DEFAULT "10.5",
       k3 VARCHAR(10) COMMENT "string column",
       k4 INT NOT NULL DEFAULT "1" COMMENT "int column"
   )
   COMMENT "my first table"
   DISTRIBUTED BY HASH(k1) BUCKETS 32;
   ```

   Doris 兼容 MySQL 协议，可以使用 `INSERT` 语句插入数据。

   ```SQL
   -- insert data
   INSERT INTO testdb.table_hash VALUES
   (1, 10.1, 'AAA', 10),
   (2, 10.2, 'BBB', 20),
   (3, 10.3, 'CCC', 30),
   (4, 10.4, 'DDD', 40),
   (5, 10.5, 'EEE', 50);
   
   -- check the data
   SELECT * from testdb.table_hash;
   +------+-------+------+------+
   | k1   | k2    | k3   | k4   |
   +------+-------+------+------+
   |    3 | 10.30 | CCC  |   30 |
   |    4 | 10.40 | DDD  |   40 |
   |    5 | 10.50 | EEE  |   50 |
   |    1 | 10.10 | AAA  |   10 |
   |    2 | 10.20 | BBB  |   20 |
   +------+-------+------+------+
   ```

## 4.5 flink
### 1. 安装helm
```bash
# 解压helm-v3.19.1-linux-amd64.tar.gz并复制到/usr/bin/helm
[root@k8s-master1 ~]# tar zxvf helm-v3.19.1-linux-amd64.tar.gz
[root@k8s-master1 ~]# cp linux-amd64/helm /usr/bin/
```
### 2. 安装flink-operator
```bash
[root@k8s-master1 ~]# helm install flink-operator ./flink-kubernetes-operator-1.13.0-helm.tgz -n tqbb --create-namespace
```

### 3. 配置flink-cluster
```bash
[root@k8s-master1 ~]# kubectl apply -f flink-cluster.yaml
```

### 4. 查看结果
```bash
[root@k8s-master1 ~]# kubectl get pods -n tqbb | grep flink
flink-kubernetes-operator-7c7d4b67b4-4dhjw     2/2     Running   2 (33d ago)   86d
tqxing-native-flink-cluster-7d8f8fd575-4sbx7   1/1     Running   0             33d
```

### 5.验证
```bash
# 转发8081端口, 然后访问http://k8s-master1:8081能打开就成功了,成功后下面这个可以取消,ctrl+c
[root@k8s-master1 ~]# kubectl port-forward svc/tqxing-native-flink-cluster-rest 8081 -n tqbb
```

## 4.6 协议服务
## 4.7 minio安装
### 1. 安装docker
> 参照前面mysql安装小节安装docker

### 2. 安装minio
```bash
# 导入镜像
docker load -i minio.tar
# 启动
docker run -tdi --name minio -p 9000:9000 -p 9001:9001 -v /data/minio:/var/lib/minio --memory 16g --restart always tqbb/oss-minio:0.1
```

### 3. 验证
浏览器打开`http://{ip}:9000`查看
## 4.8 中间服务安装(OCR, 人脸识别)

### 1. 安装docker
> 参照前面mysql安装小节安装docker

### 2. 导入镜像
```bash
# 包含ocr和人脸识别
docker load -i middle.docker.tar
# 启动ocr
docker run -dti --name ocr -p 9003:8080 --memory 8g --restart always tqbb/ocr:0.1
# 启动人脸识别
docker run -dti --name face -p 9004:8080 --memory 16g --restart always tqbb/face:0.1
```

# 五 更新服务

# 六 故障排查
# 七 日志查看
