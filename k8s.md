
```bash

# 进入pod
> kubectl exec -ti <pod name> -n <namespace> -- bash

# 查看日志
> kubectl logs -f --tail 100 <pod name> -n <namespace> 

# 查看配置, like docker inspect
> kubectl describe configmap <configmap name> -n <namespace>
> kubectl describe pod <pod name> -n <namespace>
> kubectl describe deployment <deployment name> -n <namespace>

# cp
> kubectl cp <pod name>:/a/b.txt /b.txt -n <namespace>

# delete pod then auto restart
> kubectl delete pod <pod name> -n <namespace>

```

## flink

```bash
## 安装flink-kubernetes-operator时最后先导入镜像, 否则会去github上下载镜像, 且镜像名称最好是ghcr.io/xxx
## 1.导入镜像
ctr --address /run/k3s/containerd/containerd.sock -n k8s.io image import flink-operator-1.13.0.docker
## 2.重命名镜像
ctr --address /run/k3s/containerd/containerd.sock \
-n k8s.io image tag \
docker.io/apache/flink-kubernetes-operator:b40c553 \
ghcr.io/apache/flink-kubernetes-operator:b40c553
## 3.设置自定义flink-conf.yaml
cat <<EOF > flink-values.yaml
defaultConfiguration:
  create: true
  append: false
  flink-conf.yaml: |+
    kubernetes.taskmanager.cpu: 1
    kubernetes.service-account: flink
    kubernetes.taskmanager.cpu.amount: 2.0
    kubernetes.internal.taskmanager.replicas: 2
    kubernetes.container.image: flink/flink-tqx:1.16.2
    parallelism.default: 1
    taskmanager.numberOfTaskSlots: 2
    taskmanager.memory.process.size: 2 gb
    # web上是否显示cancel
    web.cancel.enable: true
    jobmanager.memory.process.size: 2 gb
    pekko.ask.timeout: 30s
    akka.ask.timeout: 100s
    state.backend: rocksdb
    state.checkpoints.num-retained: 5
    jobmanager.execution.failover-strategy: region
    high-availability.storageDir: s3://flink-xx/recovery/
    s3.access-key: xxx
    s3.secret-key: xxxxxx
    s3.endpoint: http://1.1.1.1:9000
    s3.path.style.access: true
    state.checkpoints.dir: s3://flink-xx/checkpoints/
    state.savepoints.dir: s3://flink-xx/savepoints/
    jobmanager.archive.fs.dir: s3://flink-xx/completed-jobs/
    env.java.opts.jobmanager: -Duser.timezone=GMT+08
    env.java.opts.taskmanager: -Duser.timezone=GMT+08
    kubernetes.jobmanager.cpu.amount: 2
    internal.cluster.execution-mode: NORMAL
    kubernetes.jobmanager.cpu: 2
    $internal.flink.version: v1_16
    kubernetes.pod-template-file.jobmanager: /tmp/flink_op_generated_podTemplate_1400800315163453345.yaml
EOF

## 3.安装flin-oprator
helm install flink-operator -f flink-values.yaml ./flink-kubernetes-operator-1.13.0-helm.tgz -n test-ns --create-namespace
## 4.设置flink-cluster.yaml
## 这里设置配置会覆盖默认的
cat <<EOF > flink-cluser.yaml
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: test-native-flink-cluster
  namespace: test-ns
spec:
  serviceAccount: flink
  image: flink/flink-test:1.16.2
  flinkVersion: v1_16
  flinkConfiguration:
    #taskmanager.numberOfTaskSlots: 1
  jobManager:
    resource:
      memory: "2048m"
      cpu: 1
  taskManager:
    resource:
      memory: "2048m"
      cpu: 1
EOF

## 5.启动flink-cluster
kubectl apply -f flink-cluster.yaml

## 6.验证, 暴露8081页面可访问
kubectl port-forward service/test-native-flink-cluster-rest 8081 -n test-ns

## 7.删除cluster
kubectl delete flinkdep/tqxing-native-flink-cluster -n test-ns

## 8.删除flink-operator
helm uninstall flink-operator -n test-ns

## 9.安装过程可以通过kubectl describe 查看进度.
## 比如
kubectl describe pod/flink-kubernetes-operator-5c8c4d7c48-cz222 -n test-ns
```