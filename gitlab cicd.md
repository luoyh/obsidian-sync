
### 镜像准备
```bash
docker pull docker:29.6.1-cli
gitlab/gitlab-runner:v18.11.4
```
### 启动gitlab-runner docker环境

```sh
docker run -d --name gitlab-runner --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /home/local/data/gitlab-runner/config:/etc/gitlab-runner \
  gitlab/gitlab-runner:v18.11.4
```

### 注册项目

```bash
docker exec -it gitlab-runner gitlab-runner register
# 最后的cli环境可输入上面的docker:29.6.1-cli
```

### 设置映射目录
```toml
# /home/local/data/gitlab-runner/config/config.toml
# 主要是这种缓存目录与项目目录
# 基本上gitlab-runner都是在/builds目录下
concurrent = 1
check_interval = 0
connection_max_age = "15m0s"
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "test java"
  url = "http://192.168.30.55:31715/"
  id = 2
  token = "0ab31220aeb3d56b00eed009de357a"
  token_obtained_at = 2026-06-30T03:39:22Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
      AssumeRoleMaxConcurrency = 0
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "docker:29.6.1-cli"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/home/local/data/gitlab-runner/cache:/cache", "/home/local/data/gitlab-runner/builds:/builds", "/home/local/data/repos:/data/repos", "/var/run/docker.sock:/var/run/docker.sock"]
    volume_keep = false
    pull_policy = ["if-not-present"]
    shm_size = 0
    network_mtu = 0

[[runners]]
  name = "test www"
  url = "http://192.168.30.55:31715/"
  id = 4
  token = "43053691498efe8c88fef430ef4ef1"
  token_obtained_at = 2026-07-01T06:04:49Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
      AssumeRoleMaxConcurrency = 0
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "docker:29.6.1-cli"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/home/local/data/gitlab-runner/cache:/cache", "/home/local/data/gitlab-runner/builds:/builds", "/home/local/data/repos:/data/repos", "/var/run/docker.sock:/var/run/docker.sock"]
    volume_keep = false
    pull_policy = ["if-not-present"]
    shm_size = 0
    network_mtu = 0

```


### kubectl的镜像
```bash
# kubectl.dockerfile
FROM alpine/kubectl:1.36.2
  
RUN sed -i 's#https://dl-cdn.alpinelinux.org/alpine#https://mirrors.tuna.tsinghua.edu.cn/alpine#g' /etc/apk/repositories

#RUN apk add --no-cache gettext
RUN apk update && apk add --no-cache \
    bash \
    gettext \
    coreutils
ENTRYPOINT []
CMD ["/bin/sh"]

# build
docker build -t vvtf/apline-kubectl:1 -f kubectl.dockerfile .
```

### pnpm镜像
```bash
# pnpm.dockerfile
FROM node:24-alpine3.23
RUN npm install --global "pnpm@latest-11"
CMD ["node"]

# build
 docker build -t vvtf/pnpm:1 -f pnpm.dockerfile .
```

### 前端测试使用pnpm docker编译项目
```bash
docker run --rm \
  -v ${PWD}:/app \
  -w /app \
  -v /home/local/data/repos/npm:/pnpm-store \
  -e PNPM_STORE_PATH=/pnpm-store \
  -e NODE_OPTIONS="--max-old-space-size=8196" \
  -e CI=true \
  tqbb/pnpm:1 \
  sh -c "pnpm config set registry http://192.168.0.3:8081/repository/npm-group/ && \
  	  pnpm config set store-dir /pnpm-store && \
  	  pnpm approve-builds --all && \
          pnpm install && \
          pnpm add file:element-plus && \
          pnpm run build:docker2"

```

### java示例
```yml
# java项目示例,支持多模块,并且最后发布到k8s
stages:
  - init 
  - build-jar
  - build-image
  - apply-k8s
  
variables:
  V_ENV: "auto"
  DOCKER_API_VERSION: "1.44"
  TAG_NAME: "test"
  # 这个是k8s的授权文件的base64编码
  KUBECFG_DATA: "xxx"

  ENVIRONMENT: "test"
  REGISTRY: "harbor.test.com"
  HARBOR_NAMESPACE: "tqxing-test"
  K8s_NAMESPACE: "tqxing-test"
  
  HARBOR_USER: "1234"
  HARBOR_PASSWORD: "1234"
  
  v_gate: "0"
  v_auth: "0"
  v_center: "0"
  v_admin: "1"
  v_stream: "0"
  v_chat: "0"
  v_sms: "0"
  v_forward: "0"
  v_map: "0"
  v_alarm: "0"
  v_face: "0"
  v_push: "0"
  v_integ: "0"

# 新版本的gitlab可使用workflow.rules是否触发pipeline
workflow:
  rules:
    - if: $V_ENV == 'auto'
      when: never
    - when: always

before_script:
  - 'echo "build java: $V_ENV"'

init_variables:
  stage: init
  image: alpine:3.23.5
  only:
    variables:
      - $V_ENV == 'test'
  tags: 
    - test
  script:
    - |
      SERVICES=""
      [ "$v_gate" == "1" ] && SERVICES="$SERVICES tuqiaoxing-gate"
      [ "$v_auth" == "1" ] && SERVICES="$SERVICES tuqiaoxing-auth"
      [ "$v_center" == "1" ] && SERVICES="$SERVICES tuqiaoxing-center"
      [ "$v_admin" == "1" ] && SERVICES="$SERVICES tuqiaoxing-admin-biz"
      [ "$v_stream" == "1" ] && SERVICES="$SERVICES tuqiaoxing-stream"
      [ "$v_chat" == "1" ] && SERVICES="$SERVICES tuqiaoxing-chat"
      [ "$v_sms" == "1" ] && SERVICES="$SERVICES tuqiaoxing-sms"
      [ "$v_forward" == "1" ] && SERVICES="$SERVICES tuqiaoxing-forward"
      [ "$v_map" == "1" ] && SERVICES="$SERVICES tuqiaoxing-map"
      [ "$v_alarm" == "1" ] && SERVICES="$SERVICES tuqiaoxing-alarm-scheduler"
      [ "$v_face" == "1" ] && SERVICES="$SERVICES tuqiaoxing-face"
      [ "$v_push" == "1" ] && SERVICES="$SERVICES tuqiaoxing-push"
      [ "$v_integ" == "1" ] && SERVICES="$SERVICES tuqiaoxing-integ"
      
      # 去除首尾空格
      SERVICES=$(echo $SERVICES | xargs)
      if [ -z "$SERVICES" ]; then
        echo "没有选择构建的服务!!!"
        exit 1
      fi
      echo "选中的服务: $SERVICES"
      # 将变量写入 build.env，供下游 Job 使用
      echo "SERVICES_TO_BUILD=$SERVICES" 
      export SERVICES_TO_BUILD=$SERVICES
      # 注意加引号，防止空格分割
      echo "export SERVICES_TO_BUILD=\"$SERVICES\"" > build.env
      cat build.env
  artifacts:
    paths:
      - build.env
    expire_in: 1 hour
    
build-jar-job:
  stage: build-jar
  image: maven:3.9.12-eclipse-temurin-17
  tags:
    - test
  only:
    variables:
      - $V_ENV == 'test'
  dependencies:
    - init_variables
  script:
    - source build.env
    - echo "========= build jar =========="
    - echo "maven 本地缓存仓库 /data/repos/m2"
    # 使用主机上的m2本地仓库与主机上的settings.xml配置
    - mvn clean package -Dmaven.test.skip=true -Dmaven.repo.local=/data/repos/m2 -s /data/repos/maven-settings.xml -T 1C -P test
    - echo "避免jar文件过大使用artifact上传失败,把构建的jar放到本地映射路径下"
    - echo "SERVICES_TO_BUILD=$SERVICES_TO_BUILD"
    - |
      for service in $SERVICES_TO_BUILD; do
        echo "当前构建并推送的镜像是：${service}"
        workspace=$service
        if [ "$service" == "tuqiaoxing-admin-biz" ]; then
          workspace="tuqiaoxing-admin/tuqiaoxing-admin-biz"
        fi
        IMAGE_TAG=$CI_PIPELINE_IID
        echo "mv $workspace/target/$service.jar to /data/repos/targets/"
        # mkdir -p /builds/tmp
        # 当然这里也可以放到/builds/tmp, 因为/builds也映射到主机的
        mv $workspace/target/$service.jar /data/repos/targets/
      done

build-image-job:
  stage: build-image
  tags:
    - test
  only:
    variables:
      - $V_ENV == 'test'
  #image: docker:29.6.1-cli
  dependencies:
    - init_variables
  before_script:
    - docker login -u $HARBOR_USER -p $HARBOR_PASSWORD $REGISTRY
  script:
    - source build.env
    - echo "==== build docker image and run ====="
    - echo "SERVICES_TO_BUILD=$SERVICES_TO_BUILD"
    - |
      for service in $SERVICES_TO_BUILD; do
        echo "当前构建并推送的镜像是：${service}"
        workspace=$service
        if [ "$service" == "tuqiaoxing-admin-biz" ]; then
          workspace="tuqiaoxing-admin/tuqiaoxing-admin-biz"
        fi
        cd $workspace
        mkdir -p target
        echo "mv /data/repos/targets/$service.jar to target/"
        mv /data/repos/targets/$service.jar target/
        docker build -t "$REGISTRY/$HARBOR_NAMESPACE/${service}:${CI_PIPELINE_IID}" .
        echo "build image successfuly and push to remote docker repos: $REGISTRY/$HARBOR_NAMESPACE/${service}:${CI_PIPELINE_IID}"
        docker push $REGISTRY/$HARBOR_NAMESPACE/${service}:${CI_PIPELINE_IID}
        cd $CI_PROJECT_DIR
      done


apply-k8s-job:
  stage: apply-k8s
  image: tqbb/apline-kubectl:1
#    name: alpine/kubectl:1.36.2
#    entrypoint: [""]
  tags:
    - test
  only:
    variables:
      - $V_ENV == 'test'
  dependencies:
    - init_variables
  before_script:
    - echo "$KUBECFG_DATA" | base64 -d > kubeconfig
  script:
    - source build.env
    - echo "==== build docker image and run ====="
    - echo "SERVICES_TO_BUILD=$SERVICES_TO_BUILD"
    - |
      for service in $SERVICES_TO_BUILD; do
        echo "当前构建并推送的镜像是：${service}"
        export APP_NAME=$service
        
        echo "appname=$APP_NAME"
        case $service in
          tuqiaoxing-gate) export NODEPORT=31800; export PORT=31800 ;;
          tuqiaoxing-auth) export NODEPORT=31801; export PORT=31801 ;;
          tuqiaoxing-center) export NODEPORT=31802; export PORT=8848 ;;
          tuqiaoxing-admin-biz) export NODEPORT=31803; export PORT=31803 ;;
          tuqiaoxing-stream) export NODEPORT=31806; export PORT=31806 ;;
          tuqiaoxing-chat) export NODEPORT=31808; export PORT=31808 ;;
          tuqiaoxing-sms) export NODEPORT=31809; export PORT=31809 ;;
          tuqiaoxing-forward) export NODEPORT=31810; export PORT=31810 ;;
          tuqiaoxing-map) export NODEPORT=31811; export PORT=31811 ;;
          tuqiaoxing-alarm-scheduler) export NODEPORT=31812; export PORT=31812 ;;
          tuqiaoxing-face) export NODEPORT=31813; export PORT=31813 ;;
          tuqiaoxing-push) export NODEPORT=31814; export PORT=31814 ;;
          tuqiaoxing-monitor) export NODEPORT=31815; export PORT=31815 ;;
          tuqiaoxing-integ) export NODEPORT=31816; export PORT=31816 ;;
          *) echo "未知服务: $service"; continue ;;
        esac
        
        export BUILD_NUMBER=${CI_PIPELINE_IID}
        if [ "$service" == "tuqiaoxing-center" ]; then
          envsubst < deploy/test/nacos-devops.yaml | kubectl --kubeconfig=kubeconfig apply -f -
        else
          envsubst < deploy/test/devops.yaml | kubectl --kubeconfig=kubeconfig apply -f -
        fi
      done
```

### 前端pnpm示例

```yml
stages: 
  - build-tar
  - build-image
  - apply-k8s
  
variables:
  V_ENV: "auto"
  DOCKER_API_VERSION: "1.44"
  TAG_NAME: "test"
  KUBECFG_DATA: "1234"

  ENVIRONMENT: "test"
  REGISTRY: "harbor.test.com"
  HARBOR_NAMESPACE: "tqxing-test"
  K8s_NAMESPACE: "tqxing-test"
  APP_NAME: "tuqiaoxing-ui"
  
  HARBOR_USER: "1234"
  HARBOR_PASSWORD: "1234"


before_script:
  - 'echo "build env: $V_ENV"'

    
build-tar-job:
  stage: build-tar
  image: tqbb/pnpm:1
  tags:
    - test
  only:
    variables:
      - $V_ENV == 'test'
  before_script:
    # pnpm的安全策略
    - | 
      cat > pnpm-workspace.yaml << EOF
      allowBuilds:
        '@parcel/watcher': true
        core-js: true
        es5-ext: true
        esbuild: true
        vue-demi: true
      EOF
  script:
    - echo "========= build ui =========="
    - echo "pnpm 本地缓存仓库 /data/repos/npm"
    - export PNPM_STORE_PATH=/pnpm-store
    - export NODE_OPTIONS="--max-old-space-size=8196"
    - export CI=true
    - pnpm config set registry http://219.153.12.25:31723/repository/npm-group/
    - pnpm config set store-dir /pnpm-store
    - pnpm approve-builds --all
    - pnpm install
    - pnpm add file:element-plus
    - pnpm run build:docker2
    - echo "避免tar文件过大使用artifact上传失败,把构建的tar放到本地映射路径下"
    - mv docker/dist.tar.gz /data/repos/targets/dist.tar.gz

build-image-job:
  stage: build-image
  tags:
    - test
  only:
    variables:
      - $V_ENV == 'test'
  #image: docker:29.6.1-cli
  before_script:
    - docker login -u $HARBOR_USER -p $HARBOR_PASSWORD $REGISTRY
  script:
    - echo "==== build docker image and push ====="
    - mv /data/repos/targets/dist.tar.gz docker/
    - cd docker
    - docker build -f Dockerfile -t $REGISTRY/$HARBOR_NAMESPACE/$APP_NAME:$CI_PIPELINE_IID .
    - echo "build image successfuly and push to remote docker repos $REGISTRY/$HARBOR_NAMESPACE/$APP_NAME:$CI_PIPELINE_IID"
    - docker push $REGISTRY/$HARBOR_NAMESPACE/$APP_NAME:$CI_PIPELINE_IID

apply-k8s-job:
  stage: apply-k8s
  image: tqbb/apline-kubectl:1
#    name: alpine/kubectl:1.36.2
#    entrypoint: [""]
  tags:
    - test
  only:
    variables:
      - $V_ENV == 'test'
  before_script:
    - echo "$KUBECFG_DATA" | base64 -d > kubeconfig
  script:
    - echo "==== apply k8s yaml ====="
    - echo "SERVICES_TO_BUILD=$SERVICES_TO_BUILD"
    - |
      export BUILD_NUMBER=${CI_PIPELINE_IID}
      envsubst < deploy/test/devops.yaml | kubectl --kubeconfig=kubeconfig apply -f -


```


### 3
```yml

stages:
  - build-jar
  - build-image
variables:
  # 虚拟启动环境, 用于触发pipeline, 默认为auto, 不触发pipeline
  # 防止自动触发pipeline, 只在手动触发时运行
  # demo触发demo.moreinsight.ai环境
  # ali触发chengdu的runner部署(未实现)
  # 其它触发dev.moreinsight.ai
  V_ENV: "auto"
  DOCKER_API_VERSION: "1.44"
  TAG_NAME: "shared"

workflow:
  rules:
    - if: $V_ENV == 'auto'
      when: never
    - when: always

default:
  tags:
    - shared
  interruptible: true

before_script:
  - |
    case "$V_ENV" in
      demo)
        export ENV_MOREINSIGHT_PROFILE="demo"
        export ENV_MOREINSIGHT_PORT=7070
        ;;
      ali)
        export ENV_MOREINSIGHT_PROFILE="dev"
        export ENV_MOREINSIGHT_PORT=8080
        export TAG_NAME="chengdu"
        ;;
      *)
        export ENV_MOREINSIGHT_PROFILE="dev"
        export ENV_MOREINSIGHT_PORT=8080
        ;;
    esac
  - export ENV_IMAGE_NAME="moreinsight-admin-${ENV_MOREINSIGHT_PROFILE}"
  - 'echo "profile:port: $ENV_MOREINSIGHT_PROFILE:$ENV_MOREINSIGHT_PORT"'
  - 'echo "name: ${ENV_IMAGE_NAME}"'
  - 'echo "tag: ${TAG_NAME}"'


build-jar-job:
  stage: build-jar
  image: maven:3.9.12-eclipse-temurin-21
  script:
    - echo "========= build jar =========="
    - echo "maven 本地缓存仓库 /builds/backend/.m2"
    - mkdir -p /builds/backend/.m2
    - mkdir -p /builds/backend/buildx
    - mkdir -p /builds/backend/tmp
    - mvn clean package -Dmaven.test.skip=true -Dmaven.repo.local=/builds/backend/.m2 -T 1C
    - echo "避免jar文件过大使用artifact上传失败,把构建的jar放到本地映射路径下"
    - rm -rf /builds/backend/buildx/app.jar
    - mv target/*.jar /builds/backend/buildx/app.jar

build-image-job:
  stage: build-image
  #image: docker:24.0.5-cli
  script:
    - echo "==== build docker image and run ====="
    - echo "remove image ${ENV_IMAGE_NAME}"
    - docker image ls | grep "${ENV_IMAGE_NAME}" || true
    - docker image ls | grep "${ENV_IMAGE_NAME}" | awk '{print $3}' | xargs -r -I {} docker rmi {} 2>/dev/null || true
    - echo "stop container and remove container"
    - docker ps -a | grep "${ENV_IMAGE_NAME}" || true
    - docker ps -q --filter "name=${ENV_IMAGE_NAME}" | xargs -r docker stop
    - docker ps -a -q --filter "name=${ENV_IMAGE_NAME}" | xargs -r docker rm
    - ls -l /var/run/
    - rm -rf target
    - mkdir -p target
    - mv /builds/backend/buildx/app.jar target/app.jar
    - docker build -t "${ENV_IMAGE_NAME}:${CI_JOB_ID}" .
    - docker run -d --name "${ENV_IMAGE_NAME}-${CI_JOB_ID}" -v /builds/backend/tmp:/builds/backend/tmp -e "ENV_MOREINSIGHT_PROFILE=${ENV_MOREINSIGHT_PROFILE}" -p "${ENV_MOREINSIGHT_PORT}:8080" "${ENV_IMAGE_NAME}:${CI_JOB_ID}"
    - sleep 20
    - docker ps -a | grep "${ENV_IMAGE_NAME}" || true
    - docker logs "${ENV_IMAGE_NAME}-${CI_JOB_ID}"


```


### 不是docker环境的gitlab-runner

```yml
stages: 
  - build
  - test
  - deploy

variables:
    V_ENV: 'auto'

workflow:
  rules:
    - if: $V_ENV == 'auto'
      when: never
    - when: always

build-job:
  stage: build
  tags:
    - test
  script:
    - echo "Compiling the code..."
    - sh mvn-build.sh
    - sh /usr/local/apps/test/start.sh stop
    - rm -rf /usr/local/apps/test.jar
    - cp target/test.jar /usr/local/apps/test.jar
    - cd /usr/local/apps/test
    - sh start.sh start
    - echo "Compile complete."

deploy-job:
  stage: deploy
  tags:
    - test
  script:
    - echo "Deploying application..."
    - echo "Application successfully deployed."
```