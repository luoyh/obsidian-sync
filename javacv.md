
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

## push mp4 to rtmp

```java

        log.info("push from: {}", input);
        log.info("push to: {}", rtmp);
        // ffmepg日志级别
        avutil.av_log_set_level(avutil.AV_LOG_ERROR);
        FFmpegLogCallback.set();
        // 实例化帧抓取器对象，将文件路径传入
        // FFmpegFrameGrabber grabber = new
        // FFmpegFrameGrabber("C:\\Users\\x\\Downloads\\f4f8538c5de34dcd935ea67659c047f9.mp4");
        try (FFmpegFrameGrabber grabber = new FFmpegFrameGrabber(input)) {
            long startTime = System.currentTimeMillis();

            log.info("开始初始化帧抓取器");

            // 初始化帧抓取器，例如数据结构（时间戳、编码器上下文、帧对象等），
            // 如果入参等于true，还会调用avformat_find_stream_info方法获取流的信息，放入AVFormatContext类型的成员变量oc中
            grabber.start(true);
            // 设置推流位置
            grabber.setTimestamp(ss);

            log.info("帧抓取器初始化完成，耗时[{}]毫秒", System.currentTimeMillis() - startTime);

            // grabber.start方法中，初始化的解码器信息存在放在grabber的成员变量oc中
            AVFormatContext avFormatContext = grabber.getFormatContext();

            // 文件内有几个媒体流（一般是视频流+音频流）
            int streamNum = avFormatContext.nb_streams();

            // 没有媒体流就不用继续了
            if (streamNum < 1) {
                log.error("文件内不存在媒体流");
                grabber.close();
                return;
            }

            // 取得视频的帧率
            int frameRate = (int) grabber.getVideoFrameRate();

            log.info("视频帧率[{}]，视频时长[{}]秒，媒体流数量[{}]", frameRate, avFormatContext.duration() / 1000000,
                    avFormatContext.nb_streams());

            // 遍历每一个流，检查其类型
            for (int i = 0; i < streamNum; i++) {
                AVStream avStream = avFormatContext.streams(i);
                AVCodecParameters avCodecParameters = avStream.codecpar();
                log.info("流的索引[{}]，编码器类型[{}]，编码器ID[{}]", i, avCodecParameters.codec_type(),
                        avCodecParameters.codec_id());
            }

            // 视频宽度
            int frameWidth = grabber.getImageWidth();
            // 视频高度
            int frameHeight = grabber.getImageHeight();
            // 音频通道数量
            int audioChannels = grabber.getAudioChannels();

            log.info("视频宽度[{}]，视频高度[{}]，音频通道数[{}]", frameWidth, frameHeight, audioChannels);
            // 实例化FFmpegFrameRecorder，将SRS的推送地址传入
            try (FFmpegFrameRecorder recorder = new FFmpegFrameRecorder(
                    // "rtmp://219.153.49.165:1935/class/demo1",
                    rtmp, frameWidth, frameHeight, audioChannels)) {
                // 设置编码格式
                recorder.setVideoCodec(avcodec.AV_CODEC_ID_H264);

                // 设置封装格式
                recorder.setFormat("flv");

                // 一秒内的帧数
                recorder.setFrameRate(frameRate);

                // 两个关键帧之间的帧数
                recorder.setGopSize(frameRate);

                // 设置音频通道数，与视频源的通道数相等
                recorder.setAudioChannels(grabber.getAudioChannels());

//                // 降低编码延时
//                recorder.setVideoOption("tune", "zerolatency");
//                recorder.setMaxDelay(500);
//                recorder.setGopSize(10);
//                // 提升编码速度
//                recorder.setVideoOption("preset", "ultrafast");

                startTime = System.currentTimeMillis();
                log.info("开始初始化帧抓取器");

                // 初始化帧录制器，例如数据结构（音频流、视频流指针，编码器），
                // 调用av_guess_format方法，确定视频输出时的封装方式，
                // 媒体上下文对象的内存分配，
                // 编码器的各项参数设置
                recorder.start();

                log.info("帧录制初始化完成，耗时[{}]毫秒", System.currentTimeMillis() - startTime);

                Frame frame = null;

                log.info("开始推流");

                long videoTS = 0;

                int videoFrameNum = 0;
                int audioFrameNum = 0;
                int dataFrameNum = 0;

                // 假设一秒钟15帧，那么两帧间隔就是(1000/15)毫秒
                int interVal = 1000 / frameRate;
                // 发送完一帧后sleep的时间，不能完全等于(1000/frameRate)，不然会卡顿，
                // 要更小一些，这里取八分之一
                if (pushDelay <= 0) {
                    interVal = 0;
                    pushDelay = 1000;
                } else {
                    interVal /= pushDelay;
                }
                int ival = 1000 / frameRate;

                startTime = System.currentTimeMillis();
                // 持续从视频源取帧
                while (null != (frame = grabber.grab())) {
                    // videoTS = 1000 * (System.currentTimeMillis() - startTime);

                    // 时间戳
                    // recorder.setTimestamp(videoTS);

                    // 有图像，就把视频帧加一
//                    if (null != frame.image) {
//                        videoFrameNum++;
//                    }
//
//                    // 有声音，就把音频帧加一
//                    if (null != frame.samples) {
//                        audioFrameNum++;
//                    }
//
//                    // 有数据，就把数据帧加一
//                    if (null != frame.data) {
//                        dataFrameNum++;
//                    }

                    // 取出的每一帧，都推送到rtmp
                    recorder.record(frame);
                    
                    long end = System.currentTimeMillis();
                    // long delay = frame.timestamp / 1000 - (end - startTime);
                    long delay = Math.max(0, (ival - (end - startTime)) / pushDelay);
                    startTime = end;
                    if (delay > 0) {
//                        log.info("sleep: {}", delay);
                        Thread.sleep(delay);
                    }

                    // 停顿一下再推送
                    // Thread.sleep(interVal <= 0 ? 1 : interVal);
                    // }

                }

                log.info("推送完成，视频帧[{}]，音频帧[{}]，数据帧[{}]，耗时[{}]秒", videoFrameNum, audioFrameNum, dataFrameNum,
                        (System.currentTimeMillis() - startTime) / 1000);

                // 关闭帧录制器
//                recorder.close();
                // 关闭帧抓取器
//            grabber.close();
            }
        } finally {
            log.info("end push");
            finished.run();
        }
    
```