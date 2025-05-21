
```bash
# 抓取http
tcpdump -i any -A -s 0 'host c.qqearth.com and port 81'

tcpdump -i any -A -s 0 'host c.qqearth.com and (port 81 or port 443)'

```

# 使用 tcpdump 抓取访问百度(baidu.com)的 HTTP 请求包

要抓取当前服务器上访问 `http://baidu.com` 的 HTTP 请求包，可以使用以下 tcpdump 命令：

## 基本命令

```bash
sudo tcpdump -i any -A -s 0 'host baidu.com and (port 80 or port 443)'
```

### 命令解释：
- `-i any`：监听所有网络接口
- `-A`：以ASCII格式打印数据包内容（方便查看HTTP文本）
- `-s 0`：抓取完整数据包（不截断）
- `host baidu.com`：只抓取与baidu.com相关的流量
- `port 80 or port 443`：HTTP(80)和HTTPS(443)端口

## 更精确的HTTP请求抓取

如果只想抓取HTTP请求（不包括响应），可以使用：

```bash
sudo tcpdump -i any -A -s 0 'host baidu.com and port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```

这个命令会匹配TCP负载中开始的"GET "字符串（HTTP请求方法）。

## 保存到文件以供后续分析

```bash
sudo tcpdump -i any -w baidu.pcap 'host baidu.com and (port 80 or port 443)'
```

然后用Wireshark等工具分析保存的baidu.pcap文件。

## HTTPS注意事项

对于HTTPS流量(baidu.com现在默认使用HTTPS)：
1. 你只能看到加密的流量，看不到实际内容
2. 如果需要解密HTTPS，需要配置SSL/TLS解密（这通常需要客户端配合）

## 常用过滤条件补充

- 只抓取请求：`tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420` (GET)
- 只抓取响应：`tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x48545450` (HTTP)

## 查看特定网卡

如果你知道流量通过哪个网卡（如eth0），可以指定：

```bash
sudo tcpdump -i eth0 -A -s 0 'host baidu.com and port 80'
```