

```bash

# maven docker
# -w /usr/src/project把工作目录设置到此
# -w ${PWD}:/usr/src/project映射文件目录
# -v /mnt/e/.roy/data/.m2:/root/.m2/repository 映射本地仓库
# 映射settngs.xml文件, 可修改镜像等
# -v /data/apps/tqbb/src/m2conf/settings.xml:/usr/share/maven/conf/settings.xml 
docker run -ti --rm \
-w /usr/src/project \
-v ${PWD}:/usr/src/project \
-v /mnt/e/.roy/data/.m2:/root/.m2/repository \
-v /data/apps/tqbb/src/m2conf/settings.xml:/usr/share/maven/conf/settings.xml \
maven:3.9.9-eclipse-temurin-17 \
mvn clean package -Dmaven.test.skip=true

```