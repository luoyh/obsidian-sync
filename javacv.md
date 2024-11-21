
```bash

# ffmpeg push from mp4 to rtmp
ffmpeg  -i "http://tuqiaoxing-minio:31954/public/f4f8538c5de34dcd935ea67659c047f9.mp4?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=xvkuFtojM5vdGP1qp0Im%2F20241120%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20241120T084839Z&X-Amz-Expires=86400&X-Amz-SignedHeaders=host&X-Amz-Signature=eafb07cdb8b0d82f2fdb4025ca2b9ae5ba0f9dff77d4f3b9d853fd1c259998f0" -vcodec copy -acodec copy -f flv rtmp://localhost:18082/class/demo1

#  rtmp://219.153.49.165:1935/class/demo1



```