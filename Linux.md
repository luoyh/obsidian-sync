

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
```