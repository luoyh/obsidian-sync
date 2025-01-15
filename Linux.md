

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