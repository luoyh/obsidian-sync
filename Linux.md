

```bash
# 使用snap安装软件后发现软件错误,
# 然后使用snap remove移除了软件, 
# 然后使用apt安装软件, 
# 结果提示-bash: /snap/bin/xxx: No such file or directory

# 比如安装tree
> snap install tree
> snap remove tree
> apt install tree
> tree .
-bash: /snap/bin/tree: No such file or directory

# 使用hash查看缓存的命令
> hash
# 清除缓存
> hash -r


# /etc/profile
# ~/.bashrc
alias tailf='tail -f ---disable-inotify'

# curl 中文参数, 比如: curl http://localhost/api/v1/test?name=张三
# 使用--data-urlencode
> curl -X GET --data-urlencode 'name=张三' "http://localhost/api/v1/test"
> curl -X GET --data-urlencode 'name=张三' "http://localhost/api/v1/test?id=2"

# 文件行数
> cat test.txt | wc -l
# 日志文件每秒数量
# 2024-11-29 09:46:03,965 INFO  com.xxx hello
> awk '{print $1" "$2}' test.txt | awk -F ',' '{print $1}' | uniq -c


# 输出的内容到复制
> curl -s http://localhost/test/output.json | xclip -sel clip
> echo abcd | xclip -sel clip

# 输出的内容json格式化
> curl -s http://localhost/test/output.json | jq .
> echo '{"id":1, "name":"hello", "data": []}' | jq .

# ssh keep-alive
# interval 30 second send heart beat, 3 time no response then close
> ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 192.168.1.2

```


### print 等待

```bash
#!/bin/bash

file=$(mktemp)
echo $file
progress() {
  pc=0
  tx=("-" "\\" "|" "/")
  while [ -e $file ]
    do
      #echo -n "sec\033[0K\r"
      printf "%s\b" ${tx[$pc]}
      #echo "."
      sleep 0.3
      pc=$(($pc+1))
      if [ $pc -gt 3 ]; then
        pc=0
      fi
    done
}
progress &
#Do all the necessary staff
sleep 3
#now when everything is done
rm -f $file

```


```bash
#!/bin/bash
#ms1=$(date +%s.%3N)
#ms2=$(date +%s.%3N)
#msdiff=$(awk "BEGIN{print $ms2 - $ms1}")
#echo "msdiff=$msdiff"
wait() {
  pc=0
  ms=0.0
  tx=("-" "\\" "|" "/")
  ext=$(ps aux | grep -v grep | grep mvvn | wc -l)
  while [ $ext -gt 0 ]
  do
    #echo -n "$pc sec\033[0K\r"
    #printf "%-5ss %s\b\b\b\b\b\b\b\b" $ms ${tx[$pc]}
    # %-5s left align, %5s right align
    printf "%5ss %s\b\b\b\b\b\b\b\b" $ms ${tx[$pc]}
    sleep 0.1
    pc=$(($pc+1))
    ms=`echo $ms + 0.1 | bc`
    if [ $pc -gt 3 ]; then
      pc=0
    fi
    ext=$(ps aux | grep -v grep | grep mvvn | wc -l)
  done
}

echo "wait for mvn build finished > "

wait &

ext=$(ps aux | grep -v grep | grep mvvn | wc -l)
while [ $ext -gt 0 ]
do
  sleep 1
  ext=$(ps aux | grep -v grep | grep mvvn | wc -l)
done
echo ""
echo "mvn build finished. then build to docker and run > "

#sh local-build-and-start.sh

```


```bash
#!/bin/bash
#ms1=$(date +%s.%3N)
#ms2=$(date +%s.%3N)
#msdiff=$(awk "BEGIN{print $ms2 - $ms1}")
#echo "msdiff=$msdiff"
wait() {
  pc=0
  ms=0.0
  #tx=("X" "^" "~" ">" "<")
  tx=( - \\ \| / )
  ext=$(ps aux | grep -v grep | grep mvvn | wc -l)
  while [ $ext -gt 0 ]
  do
    #echo -n "$pc sec\033[0K\r"
    #printf "%5ss %s\b\b\b\b\b\b\b\b" $ms ${tx[$pc]}
    printf "%5ss %s\b\b\b\b\b\b\b\b" $ms "${tx[pc++%${#tx[@]}]}"
    sleep 0.1
    ms=`echo $ms + 0.1 | bc`
    ext=$(ps aux | grep -v grep | grep mvvn | wc -l)
  done
}

echo "wait for mvn build finished > "

wait &

ext=$(ps aux | grep -v grep | grep mvvn | wc -l)
while [ $ext -gt 0 ]
do
  sleep 1
  ext=$(ps aux | grep -v grep | grep mvvn | wc -l)
done
echo ""
echo "mvn build finished. then build to docker and run > "

#sh local-build-and-start.sh

```


```bash
#/bin/sh
cd /data/libs/graalvm-jdk-23_37.1/bin/build/sms


input=$(/data/kafka_2.13-3.8.0/bin/kafka-consumer-groups.sh --describe  --bootstrap-server 192.168.2.51:30354 --group flink-alarm-consumer 2>&1  | grep jt808_trajectory_topic)

if [ -z "$input" ]; then
    r=$(/home/soft/libs/jdk-22.0.2/bin/java -cp .:$(find /data/libs/graalvm-jdk-23_37.1/bin/build/sms/lib/ -name  "*.jar" | xargs | sed  "s/ /:/g") SimpleSender 13648341599 "kafka连接失败")
    echo "[`date`]: kafka error: $r" >> c.out
    exit 0
fi

read offset end lag <<< $(echo "$input" | awk '{
    offset += $4;
    end += $5;
    lag += $6;
} END {print offset,end,lag}')

echo "[`date`]: current consumer: " >> c.out
echo "$input" >> c.out
echo "[`date`]: offset=$offset, end=$end, lag=$lag" >> c.out

if [[ $lag -gt 1000000 ]]; then
    r=$(/home/soft/libs/jdk-22.0.2/bin/java -cp .:$(find /data/libs/graalvm-jdk-23_37.1/bin/build/sms/lib/ -name  "*.jar" | xargs | sed  "s/ /:/g") SimpleSender 13648341599 "gps积压($lag)")
    echo "[`date`]: >> $lag is great than 1000000 then send sms result: $r" >> c.out
fi
echo "" >> c.out


```


## history

```bash
# 在~/.bashrc增加
# history
# 控制内存中存储的历史命令数量
export HISTSIZE=10000
# 控制历史文件中存储的命令数量
export HISTFILESIZE=10000
# 控制历史记录的过滤（如去重)
# ignoredups   # 忽略连续重复的命令
# erasedups    # 删除历史中所有重复的命令
export HISTCONTROL=ignoredups
# 设置不记录的命令（用 `:` 分隔）
export HISTIGNORE="ls:history:pwd:ll"
# 设置 `history` 时间显示格式
export HISTTIMEFORMAT="%F %T "

# 生效
. ~/.bashrc
```

## 在目录下查找文件内容包含字符串

```bash
grep -r "hello" /path/to/directory
# `-r` 或 `-R`：递归搜索子目录
# `-i`：忽略大小写（如需搜索"Hello"、"HELLO"等变体）
# `-l`：只显示包含匹配项的文件名而不显示具体内容
# `-n`：显示匹配行号
```