
```bash

# https://www.wireshark.org/docs/man-pages/tshark.html

> tshark.exe -r d:\.work\S1message.pcap -c80 -t ad -T pdml > d:\.work\pcap.xml
> tshark.exe -r d:\.work\S1message.pcap -c80 -t ad -T json > d:\.work\pcap.json

# 解析从ip 1.2.3.4发送的所有报文并保存为16进制字符串文本
> tshark -r tmp/6.pcap -Y "ip.src == 1.2.3.4" -T fields -e data > tmp/6.bin
> 

# 抓取本地19978端口的报文
> tcpdump -i any tcp port 19978 -s 0 -w 1.pcap 

# 抓取指定ip与端口的报文
> tcpdump -i any host 192.168.3.210 and tcp port 19933 -s 0 -w 2.pcap

```