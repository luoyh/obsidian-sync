
```bash

# ffmpeg push from mp4 to rtmp
> ffmpeg  -i "http://tuqiaoxing-minio:31954/public/f4f8538c5de34dcd935ea67659c047f9.mp4?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=xvkuFtojM5vdGP1qp0Im%2F20241120%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20241120T084839Z&X-Amz-Expires=86400&X-Amz-SignedHeaders=host&X-Amz-Signature=eafb07cdb8b0d82f2fdb4025ca2b9ae5ba0f9dff77d4f3b9d853fd1c259998f0" -vcodec copy -acodec copy -f flv rtmp://localhost:18082/class/demo1

#  rtmp://219.153.49.165:1935/class/demo1


# 生成30秒黑色背景视频
> ffmpeg -ss 0 -t 30 -f lavfi -i color=0x000000:s=1920x1080:r=240 -vcodec libx264 b_240fps.mp4

# 视频每帧加上时间戳
> ffmpeg -re -i b_240fps.mp4 -vf "drawtext=fontfile=C\\:/Windows/fonts/consola.ttf:text='%{pts\:hms}':x=100:y=100:fontsize=200:fontcolor=red" -c:v libx264 -an -f mp4 black_pts.mp4

> ffmpeg -i b_240fps.mp4 -vf "drawtext=fontfile=C\\:/Windows/fonts/consola.ttf:text='%{pts\:hms}':x=100:y=100:fontsize=200:fontcolor=red" -c:v libx264 -an -f mp4 black_pts.mp4

# 没有毫秒
> ffmpeg -re -ss 20 -i b_240fps.mp4 -vf "drawtext=fontfile=C\\:/Windows/fonts/consola.ttf:text='Timestamp\: %{pts\:gmtime\:0\:%M\\\:%S}.':x=100:y=100:fontsize=100:fontcolor=red" -c:v libx264 -an -f mp4 black_pts2.mp4

# MP4 to m3u8
# 默认的每片长度为 2 秒，m3u8 文件中默认只保存最新的 5 条片的信息，导致最后播放的时候只能播最后的一小部分（直播的时候特别注意）。  
# -hls_time n 设置每片的长度，默认值为 2，单位为秒。  
# -hls_list_size n 设置播放列表保存的最多条目，设置为 0 会保存有所片信息，默认值为5。  
# -hls_wrap n 设置多少片之后开始覆盖，如果设置为0则不会覆盖，默认值为0。这个选项能够避免在磁盘上存储过多的 片，而且能够限制写入磁盘的最多的片的数量。  
# -hls_start_number n 设置播放列表中 sequence number 的值为 number，默认值为 0。  
# 注意：播放列表的 sequence number 对每个 segment 来说都必须是唯一的，而且它不能和片的文件名（当使用 wrap 选项时，文件名有可能会重复使用）混淆。
# 设置关键帧间隔，设置间隔为 2 秒的参数如下：`-force_key_frames "expr:gte(t,n_forced*2)`“
> ffmpeg -i test.mp4 -force_key_frames "expr:gte(t,n_forced*2)" -strict -2 -c:a aac -c:v libx264 -hls_time 2 -hls_list_size 0 -f hls index.m3u8

> ffmpeg -i small.mp4 -g 60 -hls_time 2 -hls_list_size 0 -hls_segment_size 500000 output.m3u8

```


### docker image for javacv

```bash
FROM ubuntu:latest
# 安装Java
RUN apt-get update && apt-get install -y openjdk-17-jdk
# 安装FFmpeg依赖
RUN apt-get install -y ffmpeg libavcodec-extra

# 复制FFmpeg库文件
# COPY /path/to/ffmpeg/libs /usr/local/lib

# 复制Java程序
WORKDIR /opt
COPY app.jar app.jar

EXPOSE 80

# 配置容器启动命令
CMD java -Dserver.port=80 $JAVA_OPTS -jar app.jar

```