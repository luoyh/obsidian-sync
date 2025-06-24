

### wsl禁止使用windows的path

```conf

# /etc/wsl.conf
[boot]
systemd=true

[interop]
enabled=false
appendWindowsPath=false

[network]
generateHosts = false


```

### 使用windows的clash
```conf

# ~/.bashrc

hostip=host.local
export https_proxy="http://${hostip}:7897"
export http_proxy="http://${hostip}:7897"
export all_proxy="socks5://${hostip}:7897"


export no_proxy="*"
unset http_proxy
unset https_proxy
unset all_proxy
unset socks_proxy

# history
export HISTSIZE=10000
export HISTFILESIZE=10000
export HISTCONTROL=ignoredups
export HISTIGNORE="ls:exit:history:pwd:ll"
export HISTTIMEFORMAT="[%F %T]: "

```

### 开启ssh

```bash
apt update
apt install -y openssh-server
vim /etc/ssh/sshd_config

    Port 62222
    #AddressFamily any
    ListenAddress 0.0.0.0
    PasswordAuthentication yes
    PermitRootLogin yes

## windows
C:\WINDOWS\system32>netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=62222 connectaddress=172.19.198.135 connectport=62222
C:\WINDOWS\system32>netsh advfirewall firewall add rule name=WSL2 dir=in action=allow protocol=TCP localport=62222



```

