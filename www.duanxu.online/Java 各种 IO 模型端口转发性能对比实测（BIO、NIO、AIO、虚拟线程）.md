 
# Java 各种 IO 模型端口转发性能对比实测（BIO、NIO、AIO、虚拟线程）

Published: 2025-07-12 |  at  12:10

> 转载 cloudy491

## 背景[#](#背景)

Java 提供了多种 I/O 模型（BIO、NIO、AIO）以及不同的并发处理方式（线程池、虚拟线程）。

本文通过 **端口转发 + Apache Bench (ab)** 对比了不同 I/O 模型的吞吐表现，并给出实践建议。

## 测试方法与结果[#](#测试方法与结果)

测试采用 **三次采样取平均** 的方式，统计不同 I/O 模型的吞吐量（单位：Requests/sec，越高越好），结果如下：

| I/O 模型 | 结果1 | 结果2 | 结果3 | 平均  | 实现难度 |
| --- | --- | --- | --- | --- | --- |
| **原始（无转发）** | 90.11 | 90.20 | 90.05 | **90.12** | 简单  |
| **Socket + Thread（BIO）** | 85.63 | 85.60 | 85.10 | **85.44** | 简单  |
| **Socket + 线程池** | 88.81 | 89.00 | 88.95 | **88.92** | 简单  |
| **Socket + 虚拟线程** | 88.23 | 88.70 | 89.32 | **88.75** | 简单  |
| **AIO + 虚拟线程** | 89.57 | 89.46 | 89.38 | **89.47** | 简单  |
| **NIO（Selector）** | 85.66 | 85.62 | 85.97 | **85.75** | 困难  |
| **NIO + 虚拟线程** | 89.43 | 89.38 | 89.25 | **89.35** | 巨难  |
| **Windows 自带转发（netsh）** | 70.50 | 73.14 | 72.86 | **72.17** | 简单  |

补充：

| I/O 模型 | 结果1 | 结果2 | 结果3 | 平均  | 实现难度 |
| --- | --- | --- | --- | --- | --- |
| **原始（无转发）** | 89.80 | 89.76 | 89.71 | 89.76 | 简单  |
| **netty 4.2** | 89.48 | 89.47 | 89.53 | 89.49 | 困难  |

**Netty 的性能接近原始（无转发）场景，甚至略优于 Java AIO 虚拟线程实现**

## 分析与结论[#](#分析与结论)

### 1\. 性能对比[#](#1-性能对比)

*   🏆 **AIO + 虚拟线程** 最优（89.47），接近原始性能（90.12）。
*   🤖 **Socket + 虚拟线程** 并未明显优于线程池，可能受限于测试场景。
*   🤯 **NIO Selector** 表现平庸但实现复杂，不推荐用于简单场景。

### 2\. 开发难度分析[#](#2-开发难度分析)

| 模型  | 难度  | 特点  |
| --- | --- | --- |
| AIO + 虚拟线程 | ⭐   | 简洁高效，推荐 |
| NIO + 虚拟线程 | 🔥🔥🔥 | 性能尚可，但开发复杂度高，易出错, 易掉头发 |
| BIO + Thread/Pool | ⭐⭐  | 最易上手，适合初学者或低并发场景 |

### 3\. 推荐方案总结[#](#3-推荐方案总结)

> ✅ 推荐：**AIO + 虚拟线程**（高性能 + 易开发）
> 
> ✅ 保守可选：**Socket + 线程池**
> 
> 🚫 避免使用：**NIO（复杂难调试）**

## 测试环境信息[#](#测试环境信息)

| 项目  | 配置  |
| --- | --- |
| 操作系统 | Windows 11 |
| JDK | 21  |
| Node.js | v20.18.0 |
| Web服务 | Node.js HTTP Server |
| 模拟延迟 | 1秒延迟 |
| 压测工具 | Apache Bench (`ab`) |
| 压测命令 | `ab -n 1000 -c 100 http://127.0.0.1:9090/` |

## Node.js 延迟后端[#](#nodejs-延迟后端)

Node.js 后端代码（延迟模拟）

```
var http = require('http');

function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

http.createServer(async function (request, response) {
    response.setHeader('Access-Control-Allow-Origin', '*');
    response.setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
    response.setHeader('Access-Control-Allow-Headers', 'Content-Type');

    if (request.method === 'OPTIONS') {
        response.writeHead(204);
        response.end();
        return;
    }

    await sleep(1000); // 模拟业务延迟
    response.writeHead(200, { 'Content-Type': 'text/plain' });
    response.end('Hello World123\n');
}).listen(9090, '0.0.0.0', () => {
    console.log('Server running at http://127.0.0.1:9090/');
});
```

## 全部 Java 实现代码[#](#全部-java-实现代码)

> 所有代码已通过 JDK 21 验证，端口固定为 `9098 -> 9090`，可直接运行体验不同模型。

### 🔸 Socket + Thread（BIO）[#](#-socket--threadbio)

点击展开查看代码

```
package org.example;

import java.io.*;
import java.net.*;

public class PortForwarder1 {
    private final int localPort;
    private final String remoteHost;
    private final int remotePort;


    public PortForwarder1(int localPort, String remoteHost, int remotePort) {
        this.localPort = localPort;
        this.remoteHost = remoteHost;
        this.remotePort = remotePort;
    }

    public void start() throws IOException {
        try (ServerSocket serverSocket = new ServerSocket(localPort)) {
            System.out.printf("Forwarding %d -> %s:%d%n", localPort, remoteHost, remotePort);

            while (!Thread.interrupted()) {
                Socket clientSocket = serverSocket.accept();
                new Thread(() -> handleConnection(clientSocket)).start();
            }
        }
    }

    private void handleConnection(Socket clientSocket) {
        try (Socket remoteSocket = new Socket(remoteHost, remotePort);
             InputStream clientInput = clientSocket.getInputStream();
             OutputStream clientOutput = clientSocket.getOutputStream();
             InputStream remoteInput = remoteSocket.getInputStream();
             OutputStream remoteOutput = remoteSocket.getOutputStream()) {



            // 启动两个线程分别处理双向数据流
            Thread toRemote = new Thread(() -> forwardData(clientInput, remoteOutput));
            Thread toClient = new Thread(() -> forwardData(remoteInput, clientOutput));

            toRemote.start();
            toClient.start();

            // 等待转发线程结束
            toRemote.join();
            toClient.join();

        } catch (Exception e) {
            System.err.println("Connection error: " + e.getMessage());
        } finally {
            try {
                clientSocket.close();
            } catch (IOException e) {
                System.err.println("Failed to close client socket: " + e.getMessage());
            }
        }
    }

    private void forwardData(InputStream input, OutputStream output) {
        byte[] buffer = new byte[1024];
        int bytesRead;

        try (output) {
            while ((bytesRead = input.read(buffer)) != -1) {
                output.write(buffer, 0, bytesRead);
                output.flush();
            }
        } catch (IOException ignored) {

        }
    }

    public static void main(String[] args) throws IOException {
        new PortForwarder1(9098, "127.0.0.1", 9090).start();
    }
}
```

### 🔸 Socket + ThreadPool（线程池）[#](#-socket--threadpool线程池)

点击展开查看代码

```
package org.example;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class PortForwarder2 {
    private final int localPort;
    private final String remoteHost;
    private final int remotePort;
    private final ExecutorService executor = Executors.newCachedThreadPool();


    public PortForwarder2(int localPort, String remoteHost, int remotePort) {
        this.localPort = localPort;
        this.remoteHost = remoteHost;
        this.remotePort = remotePort;
    }

    public void start() throws IOException {
        try (ServerSocket serverSocket = new ServerSocket(localPort)) {
            System.out.printf("Forwarding %d -> %s:%d%n", localPort, remoteHost, remotePort);

            while (!Thread.interrupted()) {
                Socket clientSocket = serverSocket.accept();
                executor.submit(() -> handleConnection(clientSocket));
            }
        }
    }

    private void handleConnection(Socket clientSocket) {
        try (Socket remoteSocket = new Socket(remoteHost, remotePort);
             InputStream clientInput = clientSocket.getInputStream();
             OutputStream clientOutput = clientSocket.getOutputStream();
             InputStream remoteInput = remoteSocket.getInputStream();
             OutputStream remoteOutput = remoteSocket.getOutputStream()) {



            // 启动两个线程分别处理双向数据流
            Future<?> f1 = executor.submit(() -> forwardData(clientInput, remoteOutput));
            Future<?> f2 = executor.submit(() -> forwardData(remoteInput, clientOutput));

            f1.get();
            f2.get();

        } catch (Exception e) {
            System.err.println("Connection error: " + e.getMessage());
        } finally {
            try {
                clientSocket.close();
            } catch (IOException e) {
                System.err.println("Failed to close client socket: " + e.getMessage());
            }
        }
    }

    private void forwardData(InputStream input, OutputStream output) {
        byte[] buffer = new byte[1024];
        int bytesRead;

        try (output) {
            while ((bytesRead = input.read(buffer)) != -1) {
                output.write(buffer, 0, bytesRead);
                output.flush();
            }
        } catch (IOException ignored) {

        }
    }

    public static void main(String[] args) throws IOException {
        new PortForwarder2(9098, "127.0.0.1", 9090).start();
    }
}
```

### 🔸 Socket + VirtualThread（虚拟线线程）[#](#-socket--virtualthread虚拟线线程)

点击展开查看代码

```
package org.example;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class PortForwarder3 {
    private final int localPort;
    private final String remoteHost;
    private final int remotePort;
    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();


    public PortForwarder3(int localPort, String remoteHost, int remotePort) {
        this.localPort = localPort;
        this.remoteHost = remoteHost;
        this.remotePort = remotePort;
    }

    public void start() throws IOException {
        try (ServerSocket serverSocket = new ServerSocket(localPort)) {
            System.out.printf("Forwarding %d -> %s:%d%n", localPort, remoteHost, remotePort);

            while (!Thread.interrupted()) {
                Socket clientSocket = serverSocket.accept();
                executor.submit(() -> handleConnection(clientSocket));
            }
        }
    }

    private void handleConnection(Socket clientSocket) {
        try (Socket remoteSocket = new Socket(remoteHost, remotePort);
             InputStream clientInput = clientSocket.getInputStream();
             OutputStream clientOutput = clientSocket.getOutputStream();
             InputStream remoteInput = remoteSocket.getInputStream();
             OutputStream remoteOutput = remoteSocket.getOutputStream()) {


            // 启动两个线程分别处理双向数据流
            Future<?> f1 = executor.submit(() -> forwardData(clientInput, remoteOutput));
            Future<?> f2 = executor.submit(() -> forwardData(remoteInput, clientOutput));

            f1.get();
            f2.get();

        } catch (Exception e) {
            System.err.println("Connection error: " + e.getMessage());
        } finally {
            try {
                clientSocket.close();
            } catch (IOException e) {
                System.err.println("Failed to close client socket: " + e.getMessage());
            }
        }
    }

    private void forwardData(InputStream input, OutputStream output) {
        byte[] buffer = new byte[1024];
        int bytesRead;

        try (output) {
            while ((bytesRead = input.read(buffer)) != -1) {
                output.write(buffer, 0, bytesRead);
                output.flush();
            }
        } catch (IOException ignored) {

        }
    }

    public static void main(String[] args) throws IOException {
        new PortForwarder3(9098, "127.0.0.1", 9090).start();
    }
}
```

### 🔸 AIO + VirtualThread（虚拟线程）[#](#-aio--virtualthread虚拟线程)

点击展开查看代码

```
package org.example;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class PortForwarder4 {
    private final int localPort;
    private final String remoteHost;
    private final int remotePort;
    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();


    public PortForwarder4(int localPort, String remoteHost, int remotePort) {
        this.localPort = localPort;
        this.remoteHost = remoteHost;
        this.remotePort = remotePort;
    }

    public void start() throws Exception {
        try (var assc = AsynchronousServerSocketChannel.open()) {
            System.out.printf("Forwarding %d -> %s:%d%n", localPort, remoteHost, remotePort);

            assc.bind(new java.net.InetSocketAddress(localPort));

            while (!Thread.interrupted()) {

                AsynchronousSocketChannel asc = assc.accept().get();
                executor.submit(() -> handleConnection(asc));
            }
        }
    }

    private void handleConnection(AsynchronousSocketChannel clientAsc) {

        try (var remoteAsc = AsynchronousSocketChannel.open()) {
            remoteAsc.connect(new java.net.InetSocketAddress(remoteHost, remotePort)).get();

            // 启动两个虚拟线程分别处理双向数据流
            Future<?> f1 = executor.submit(() -> forwardData(clientAsc, remoteAsc));
            Future<?> f2 = executor.submit(() -> forwardData(remoteAsc, clientAsc));

            f1.get();
            f2.get();

        } catch (Exception e) {
            System.err.println("Connection error: " + e.getMessage());
        } finally {
            try {
                clientAsc.close();
            } catch (IOException e) {
                System.err.println("Failed to close client socket: " + e.getMessage());
            }
        }
    }

    private void forwardData(AsynchronousSocketChannel input, AsynchronousSocketChannel output) {

        ByteBuffer buffer = ByteBuffer.allocateDirect(1024); // 使用直接缓冲区提高性能
        try {
            while (!Thread.interrupted()) {
                int bytesRead = input.read(buffer).get(); // 等待读取数据
                if (bytesRead == -1) {
                    output.shutdownOutput();// 通知对方“我不再写了”
                    break;
                }

                if(bytesRead>0){
                    buffer.flip();
                    while (buffer.hasRemaining()) {
                        output.write(buffer).get();
                    }
                    buffer.clear();
                }

            }

        } catch (Exception e) {

        }
    }

    public static void main(String[] args) throws Exception {
        new PortForwarder4(9098, "127.0.0.1", 9090).start();
    }
}
```

### 🔸 NIO[#](#-nio)

点击展开查看代码

```
package org.example;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.*;

public class PortForwarder5 {

    private final int localPort;
    private final String remoteHost;
    private final int remotePort;

    public PortForwarder5(int localPort, String remoteHost, int remotePort) {
        this.localPort = localPort;
        this.remoteHost = remoteHost;
        this.remotePort = remotePort;
    }

    public void start() throws IOException {
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        serverChannel.bind(new InetSocketAddress(localPort));

        Selector selector = Selector.open();
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        System.out.printf("Forwarding %d -> %s:%d%n", localPort, remoteHost, remotePort);

        while (!Thread.interrupted()) {
            selector.select();

            Iterator<SelectionKey> keyIter = selector.selectedKeys().iterator();
            while (keyIter.hasNext()) {
                SelectionKey key = keyIter.next();
                keyIter.remove();

                if (!key.isValid()) {
                    closeKey(key);
                    continue;
                }

                if (key.isAcceptable()) {
                    acceptConnection(serverChannel, selector);
                } else if(key.isConnectable()){
                    finishConnect(key, selector);
                }
                else{
                    if (key.isValid() && key.isReadable()) {
                        readData(key, selector);
                    }
                    if (key.isValid() && key.isWritable()) {
                        writeData(key);
                    }
                }
            }
        }
    }

    private void acceptConnection(ServerSocketChannel serverChannel, Selector selector) {
        try {
            SocketChannel clientChannel = serverChannel.accept();
            clientChannel.configureBlocking(false);

            SocketChannel remoteChannel = SocketChannel.open();
            remoteChannel.configureBlocking(false);
            remoteChannel.connect(new InetSocketAddress(remoteHost, remotePort));

            ChannelAttachment channelAttachment = new ChannelAttachment();
            channelAttachment.dstChannel = clientChannel;
            remoteChannel.register(selector, SelectionKey.OP_CONNECT,channelAttachment );



        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void finishConnect(SelectionKey key, Selector selector) {
        SocketChannel remoteChannel = (SocketChannel) key.channel();
        ChannelAttachment channelAttachment = (ChannelAttachment)key.attachment();
        try {
            if (remoteChannel.finishConnect()) {
                key.interestOps(SelectionKey.OP_READ);
                SocketChannel clientChannel = channelAttachment.dstChannel;
                if (clientChannel != null && clientChannel.isOpen()) {
                    ChannelAttachment channelAttachment1 = new ChannelAttachment();
                    channelAttachment1.dstChannel = remoteChannel;
                    clientChannel.register(selector, SelectionKey.OP_READ, channelAttachment1);
                }
            }
        } catch (IOException e) {
            closeChannel(remoteChannel);
            SocketChannel peer = channelAttachment.dstChannel;
            if (peer != null) closeChannel(peer);
        }
    }

    private void readData(SelectionKey key, Selector selector) {
        SocketChannel channel = (SocketChannel) key.channel();
        ChannelAttachment channelAttachment = (ChannelAttachment)key.attachment();

        SocketChannel peerChannel = channelAttachment.dstChannel;

        if (peerChannel == null) {
            closeChannel(channel);
            return;
        }

        ByteBuffer readBuffer = ByteBuffer.allocateDirect(1024);
        int bytesRead;
        try {
            bytesRead = channel.read(readBuffer);
            if (bytesRead == -1) {
                closeChannel(channel);
                closeChannel(peerChannel);
            } else if (bytesRead > 0) {
                readBuffer.flip();

                SelectionKey selectionKey = peerChannel.keyFor(selector);
                ChannelAttachment attachment = (ChannelAttachment) selectionKey.attachment();

                peerChannel.write(readBuffer);
                if (readBuffer.hasRemaining()){
                    attachment.readQueue.offer(readBuffer);

                    if (selectionKey.isValid()) {
                        selectionKey.interestOps(selectionKey.interestOps() | SelectionKey.OP_WRITE);
                    }
                }

            }
        } catch (IOException e) {
            closeChannel(channel);
            closeChannel(peerChannel);
        }
    }

    private void writeData(SelectionKey key) {
        SocketChannel channel = (SocketChannel) key.channel();
        ChannelAttachment attachment = (ChannelAttachment) key.attachment();

        try {
            // 贪婪写入：循环处理队列中的所有缓冲区
            while (true) {
                ByteBuffer outBuffer = attachment.readQueue.peek();
                if (outBuffer == null) {
                    break; // 队列为空，没有更多数据可写
                }

                // 尝试写入数据
                channel.write(outBuffer);

                // 检查缓冲区是否已完全写入
                if (!outBuffer.hasRemaining()) {
                    attachment.readQueue.remove(); // 完全写入，移除缓冲区
                } else {
                    break; // 无法继续写入，退出循环
                }
            }

            // 根据是否有剩余数据更新interestOps
            if (attachment.readQueue.isEmpty()) {
                key.interestOps(key.interestOps() & (~SelectionKey.OP_WRITE));
            } else {
                key.interestOps(key.interestOps() | SelectionKey.OP_WRITE);
            }
        } catch (IOException e) {
            closeChannel(channel);
            SocketChannel peer = attachment.dstChannel;
            if (peer != null) {
                closeChannel(peer);
            }
        }
    }

    private void closeChannel(SocketChannel channel) {
        if (channel == null) return;
        try {
            channel.close();
        } catch (IOException ignored) {}
    }

    private void closeKey(SelectionKey key) {
        try {
            key.channel().close();
        } catch (IOException ignored) {}
        key.cancel();
    }

    private static class ChannelAttachment {
        Queue<ByteBuffer> readQueue = new ArrayDeque<>();
        SocketChannel dstChannel;
    }

    public static void main(String[] args) throws IOException {
        new PortForwarder5(9098, "127.0.0.1", 9090).start();
    }

}
```

### 🔸 NIO + VirtualThread（虚拟线程）[#](#-nio--virtualthread虚拟线程)

点击展开查看代码

```
package org.example;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class PortForwarder6 {

    private final int localPort;
    private final String remoteHost;
    private final int remotePort;
    private final int workerCount = Runtime.getRuntime().availableProcessors();
    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();


    public PortForwarder6(int localPort, String remoteHost, int remotePort) {
        this.localPort = localPort;
        this.remoteHost = remoteHost;
        this.remotePort = remotePort;
    }

    public void start() throws Exception {
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        serverChannel.bind(new InetSocketAddress(localPort));
        System.out.printf("Forwarding %d -> %s:%d%n", localPort, remoteHost, remotePort);



        //读写，连接
        List<WorkerSelect> readAndWriteList = new ArrayList<>();
        for (int i = 0; i < workerCount; i++) {
            WorkerSelect workerSelect = new WorkerSelect();
            executor.submit(workerSelect);
            readAndWriteList.add(workerSelect);
        }

        //接受
        WorkerSelect workerSelect = new WorkerSelect();
        Future<?> submit = executor.submit(workerSelect);
        workerSelect.addWorker(readAndWriteList);

        workerSelect.registerChannel(serverChannel, SelectionKey.OP_ACCEPT, null);

        submit.get();

    }



    class WorkerSelect implements Runnable{

        private final AtomicInteger index = new AtomicInteger(0);


        volatile List<WorkerSelect> workerList = Collections.synchronizedList(new ArrayList<>());

        ConcurrentLinkedQueue<Object[]> queue = new ConcurrentLinkedQueue<>();

        private Selector selector;

        @Override
        public void run() {
            try {

                selector = Selector.open();

                while (!Thread.interrupted()) {
                    selector.select(100);

                    Iterator<SelectionKey> keyIter = selector.selectedKeys().iterator();
                    while (keyIter.hasNext()) {
                        SelectionKey key = keyIter.next();
                        keyIter.remove();

                        if (!key.isValid()) {
                            closeKey(key);
                            continue;
                        }

                        if (key.isAcceptable()) {
                            acceptConnection(key, selector);
                        } else if (key.isConnectable()) {
                            finishConnect(key, selector);
                        } else {
                            if (key.isValid() && key.isReadable()) {
                                readData(key, selector);
                            }
                            if (key.isValid() && key.isWritable()) {
                                writeData(key);
                            }
                        }
                    }
                    while (true) {
                        Object[] task = queue.poll();
                        if (task == null) break;
                        SelectableChannel channel = (SelectableChannel)task[0];
                        int key = (Integer) task[1];
                        Object attm = task[2];
                        if(channel.isOpen()){
                            channel.register(selector, key, attm);
                        }
                    }
                }

            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        public void registerChannel(SelectableChannel channel, int key, Object attachment) {
            queue.offer(new Object[]{channel, key, attachment});
            if(selector!=null){
                selector.wakeup();
            }
        }

        public void addWorker(WorkerSelect worker) {
            workerList.add(worker);
        }

        public void addWorker(List<WorkerSelect> list) {
            workerList.addAll(list);
        }

        private void acceptConnection(SelectionKey key, Selector selector) {
            try {
                ServerSocketChannel serverChannel = (ServerSocketChannel)key.channel();
                SocketChannel clientChannel = serverChannel.accept();
                clientChannel.configureBlocking(false);

                SocketChannel remoteChannel = SocketChannel.open();
                remoteChannel.configureBlocking(false);
                remoteChannel.connect(new InetSocketAddress(remoteHost, remotePort));

                ChannelAttachment channelAttachment = new ChannelAttachment();
                channelAttachment.dstChannel = clientChannel;
                if(!workerList.isEmpty()) {
                    WorkerSelect worker = workerList.get(
                            Math.abs(index.getAndIncrement() % workerList.size())
                    );
                    channelAttachment.workerSelect = worker;
                    worker.registerChannel(remoteChannel, SelectionKey.OP_CONNECT, channelAttachment);

                } else {
                    remoteChannel.register(selector, SelectionKey.OP_CONNECT, channelAttachment);
                }



            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        private void finishConnect(SelectionKey key, Selector selector) {
            SocketChannel remoteChannel = (SocketChannel) key.channel();
            ChannelAttachment channelAttachment = (ChannelAttachment)key.attachment();
            channelAttachment.selector = selector;
            try {
                if (remoteChannel.finishConnect()) {
                    key.interestOps(SelectionKey.OP_READ);
                    SocketChannel clientChannel = channelAttachment.dstChannel;
                    if (clientChannel != null && clientChannel.isOpen()) {
                        ChannelAttachment channelAttachment1 = new ChannelAttachment();
                        channelAttachment1.dstChannel = remoteChannel;
                        channelAttachment1.selector = selector;


                        clientChannel.register(selector, SelectionKey.OP_READ, channelAttachment1);

                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
                closeChannel(remoteChannel);
                SocketChannel peer = channelAttachment.dstChannel;
                if (peer != null) closeChannel(peer);
            }
        }

        private void readData(SelectionKey key, Selector selector) {
            SocketChannel channel = (SocketChannel) key.channel();
            ChannelAttachment channelAttachment = (ChannelAttachment)key.attachment();

            SocketChannel peerChannel = channelAttachment.dstChannel;

            if (peerChannel == null) {
                closeChannel(channel);
                return;
            }

            ByteBuffer readBuffer = ByteBuffer.allocateDirect(1024);
            int bytesRead;
            try {
                bytesRead = channel.read(readBuffer);
                if (bytesRead == -1) {
                    closeChannel(channel);
                    closeChannel(peerChannel);
                } else if (bytesRead > 0) {
                    readBuffer.flip();
                    SelectionKey selectionKey = peerChannel.keyFor(selector);
                    ChannelAttachment attachment = (ChannelAttachment) selectionKey.attachment();

                    peerChannel.write(readBuffer);
                    if (readBuffer.hasRemaining()){
                        attachment.readQueue.offer(readBuffer);
                        if (selectionKey.isValid()) {
                            selectionKey.interestOps(selectionKey.interestOps() | SelectionKey.OP_WRITE);
                        }
                    }

                }
            } catch (IOException e) {
                closeChannel(channel);
                closeChannel(peerChannel);
            }
        }

        private void writeData(SelectionKey key) {
            SocketChannel channel = (SocketChannel) key.channel();
            ChannelAttachment attachment = (ChannelAttachment) key.attachment();

            try {
                // 贪婪写入：循环处理队列中的所有缓冲区
                while (true) {
                    ByteBuffer outBuffer = attachment.readQueue.peek();
                    if (outBuffer == null) {
                        break; // 队列为空，没有更多数据可写
                    }

                    // 尝试写入数据
                    channel.write(outBuffer);

                    // 检查缓冲区是否已完全写入
                    if (!outBuffer.hasRemaining()) {
                        attachment.readQueue.remove(); // 完全写入，移除缓冲区
                    } else {
                        break; // 无法继续写入，退出循环
                    }
                }

                // 根据是否有剩余数据更新interestOps
                if (attachment.readQueue.isEmpty()) {
                    key.interestOps(key.interestOps() & (~SelectionKey.OP_WRITE));
                } else {
                    key.interestOps(key.interestOps() | SelectionKey.OP_WRITE);
                }
            } catch (IOException e) {
                closeChannel(channel);
                SocketChannel peer = attachment.dstChannel;
                if (peer != null) {
                    closeChannel(peer);
                }
            }
        }

        private void closeChannel(SocketChannel channel) {
            if (channel == null) return;
            try {
                channel.close();
            } catch (IOException ignored) {}
        }

        private void closeKey(SelectionKey key) {
            try {
                key.channel().close();
            } catch (IOException ignored) {}
            key.cancel();
        }



        private static class ChannelAttachment {
            Queue<ByteBuffer> readQueue = new LinkedBlockingQueue<>();
            SocketChannel dstChannel;
            WorkerSelect workerSelect;
            Selector selector;
        }
    }



    public static void main(String[] args) throws Exception {
        new PortForwarder6(9098, "127.0.0.1", 9090).start();
    }

}
```

### 🔸 Windows 自带转发（netsh）[#](#-windows-自带转发netsh)

```
netsh interface portproxy add v4tov4 listenport=9099 listenaddress=0.0.0.0 connectport=9090 connectaddress=127.0.0.1
```

### 🔸 netty 4.2 (补充)[#](#-netty-42-补充)

点击展开查看代码

```
package org.example;

import io.netty.bootstrap.Bootstrap;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.Unpooled;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;


public class PortForwarder7 {
    static final int LOCAL_PORT = Integer.parseInt(System.getProperty("localPort", "9098"));
    static final String REMOTE_HOST = System.getProperty("remoteHost", "127.0.0.1");
    static final int REMOTE_PORT = Integer.parseInt(System.getProperty("remotePort", "9090"));

    public static void main(String[] args) throws Exception {
        System.err.println("Proxying *:" + LOCAL_PORT + " to " + REMOTE_HOST + ':' + REMOTE_PORT + " ...");

        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) {
                            ch.pipeline().addLast(
                                    new HexDumpProxyFrontendHandler(REMOTE_HOST, REMOTE_PORT));
                        }
                    })
                    .childOption(ChannelOption.AUTO_READ, false)
                    .bind(LOCAL_PORT).sync().channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    static class HexDumpProxyFrontendHandler extends ChannelInboundHandlerAdapter {
        private final String remoteHost;
        private final int remotePort;
        private Channel outboundChannel;

        HexDumpProxyFrontendHandler(String remoteHost, int remotePort) {
            this.remoteHost = remoteHost;
            this.remotePort = remotePort;
        }

        @Override
        public void channelActive(ChannelHandlerContext ctx) {
            final Channel inboundChannel = ctx.channel();

            Bootstrap b = new Bootstrap();
            b.group(inboundChannel.eventLoop())
                    .channel(ctx.channel().getClass())
                    .handler(new HexDumpProxyBackendHandler(inboundChannel))
                    .option(ChannelOption.AUTO_READ, false);
            ChannelFuture f = b.connect(remoteHost, remotePort);
            outboundChannel = f.channel();
            f.addListener((ChannelFuture future) -> {
                if (future.isSuccess()) {
                    inboundChannel.read();
                } else {
                    inboundChannel.close();
                }
            });
        }

        @Override
        public void channelRead(final ChannelHandlerContext ctx, Object msg) {
            if (outboundChannel.isActive()) {
                outboundChannel.writeAndFlush(msg).addListener((ChannelFuture future) -> {
                    if (future.isSuccess()) {
                        ctx.channel().read();
                    } else {
                        future.channel().close();
                    }
                });
            }
        }

        @Override
        public void channelInactive(ChannelHandlerContext ctx) {
            if (outboundChannel != null) {
                closeOnFlush(outboundChannel);
            }
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
            cause.printStackTrace();
            closeOnFlush(ctx.channel());
        }

        static void closeOnFlush(Channel ch) {
            if (ch.isActive()) {
                ch.writeAndFlush(Unpooled.EMPTY_BUFFER).addListener(ChannelFutureListener.CLOSE);
            }
        }
    }

    static class HexDumpProxyBackendHandler extends ChannelInboundHandlerAdapter {
        private final Channel inboundChannel;

        HexDumpProxyBackendHandler(Channel inboundChannel) {
            this.inboundChannel = inboundChannel;
        }

        @Override
        public void channelActive(ChannelHandlerContext ctx) {
            if (!inboundChannel.isActive()) {
                HexDumpProxyFrontendHandler.closeOnFlush(ctx.channel());
            } else {
                ctx.read();
            }
        }

        @Override
        public void channelRead(final ChannelHandlerContext ctx, Object msg) {
            inboundChannel.writeAndFlush(msg).addListener((ChannelFuture future) -> {
                if (future.isSuccess()) {
                    ctx.channel().read();
                } else {
                    future.channel().close();
                }
            });
        }

        @Override
        public void channelInactive(ChannelHandlerContext ctx) {
            HexDumpProxyFrontendHandler.closeOnFlush(inboundChannel);
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
            cause.printStackTrace();
            HexDumpProxyFrontendHandler.closeOnFlush(ctx.channel());
        }
    }
}
```

- - -

*    [Java](/tags/java/)
*    [IO模型](/tags/io模型/)
*    [虚拟线程](/tags/虚拟线程/)
*    [性能测试](/tags/性能测试/)
*    [网络编程](/tags/网络编程/)
