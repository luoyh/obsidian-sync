# 用 Java 实现 SOCKS5 代理，加速网络访问

Published: 2025-07-25 |  at  15:10

## 什么是代理[#](#什么是代理)

代理就是叫别人代替你做事情。一般是有些事情你做得不好或者做不到，或者隐藏自己，才使用代理叫别人去做。

## Socks5代理[#](#socks5代理)

SOCKS5 是一种通用的代理协议，基于 TCP（或 UDP）转发连接请求。代理服务器不关心请求内容，只负责将数据“搬运”到目标地址。

与 HTTP/HTTPS 代理不同，SOCKS5 更底层、灵活，不需要解析 HTTP 协议头，支持多种身份验证方式和地址类型（IPv4、IPv6、域名）。

## SOCKS5 协议流程[#](#socks5-协议流程)


```
1.客户端连接请求（客户端发送到服务器）：

    客户端发送的第一个数据包，用于协商版本和认证方式。
    数据包格式：
    +----+----------+----------+
    | VER| NMETHODS | METHODS  |
    +----+----------+----------+
    | 1  |    1     | 1 to 255 |
    +----+----------+----------+
    - VER：SOCKS版本，当前为 5。
    - NMETHODS：认证方法数量。
    - METHODS：认证方法列表，每个字节代表一种认证方法，可选值包括 0x00（无认证）和其他认证方法。

2.服务器回应版本协商（服务器发送到客户端）：

    服务器回应客户端的版本协商请求，选择一个认证方法。
    数据包格式：

    +----+--------+
    | VER| METHOD |
    +----+--------+
    | 1  |   1    |
    +----+--------+
    - VER：SOCKS版本，当前为 5。
    - METHOD：选择的认证方法，如果为 0xFF 表示无可接受的方法。
3.客户端向服务器发送认证信息（客户端发送到服务器）：

    客户端根据服务器选择的认证方法，向服务器发送认证信息。
    数据包格式：

    +----+------+----------+
    | VER| ULEN |  UNAME   |
    +----+------+----------+
    | 1  |  1   | 1 to 255 |
    +----+------+----------+
    - VER：SOCKS版本，当前为 5。
    - ULEN：用户名的长度。
    - UNAME：用户名，长度由 ULEN 指定。
4.服务器回应认证结果（服务器发送到客户端）：

    服务器根据客户端发送的认证信息进行验证，并回应认证结果。
    数据包格式：

    +----+--------+
    | VER| STATUS |
    +----+--------+
    | 1  |   1    |
    +----+--------+
    - VER：SOCKS版本，当前为 5。
    - STATUS：认证结果，0 表示成功，其他值表示失败。
    
5.建立连接请求（客户端发送到服务器）：

    客户端发送建立连接请求，请求与目标服务器建立连接。
    数据包格式：

    +----+-----+-------+------+----------+----------+
    | VER| CMD |  RSV  | ATYP | DST.ADDR | DST.PORT |
    +----+-----+-------+------+----------+----------+
    | 1  |  1  | X'00' |  1   | Variable |    2     |
    +----+-----+-------+------+----------+----------+
    - VER：SOCKS版本，当前为 5。
    - CMD：命令，1 表示连接，2 表示绑定。
    - RSV：保留字段，必须为 0x00。
    - ATYP：地址类型，1 表示 IPv4 地址，3 表示域名，4 表示 IPv6 地址。
    - DST.ADDR：目标地址，长度和类型由 ATYP 指定。
    - DST.PORT：目标端口，2 个字节表示。
    
6.服务器回应建立连接结果（服务器发送到客户端）：

    服务器根据客户端的连接请求，向客户端发送建立连接的结果。
    数据包格式：

    +----+-----+-------+------+----------+----------+
    | VER| REP |  RSV  | ATYP | BND.ADDR | BND.PORT |
    +----+-----+-------+------+----------+----------+
    | 1  |  1  | X'00' |  1   | Variable |    2     |
    +----+-----+-------+------+----------+----------+
    - VER：SOCKS版本，当前为 5。
    - REP：应答，0 表示成功，其他值表示失败。
    - RSV：保留字段，必须为 0x00。
    - ATYP：地址类型，1 表示 IPv4 地址，3 表示域名，4 表示 IPv6 地址。
    - BND.ADDR：绑定地址，长度和类型由 ATYP 指定。
    - BND.PORT：绑定端口，2 个字节表示。
```

## 代码[#](#代码)

测试socks5

```
curl --socks5-hostname 127.0.0.1:8888  https://huggingface.co/
```

代码主要用于学习。

简单的socks5代理

```
public class Main {

    static ExecutorService executorService = Executors.newCachedThreadPool();

    public static void main(String[] args) throws IOException {


        try (var serverSocket = new ServerSocket(8888)) {
            System.out.println("Socks5 代理服务器已启动，监听端口: 8888");
            while (!Thread.interrupted()) {
                var socket = serverSocket.accept();
                executorService.submit(() -> {
                    clientHandler(socket);
                });
            }
        }
    }

    private static void clientHandler(Socket socket) {

        try (socket) {

            negotiatedConnection(socket);

            connectServer(socket);

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void negotiatedConnection(Socket socket) throws IOException {
        //按需读取数据
        InputStream inputStream = socket.getInputStream();
        //协议判断
        if (inputStream.read() != 5) {
            throw new RuntimeException("只支持socks5协议!!");
        }
        //认证方式
        int authMethodCount = inputStream.read();
        boolean noAuth = false;
        for (int i = 0; i < authMethodCount; i++) {
            int data = inputStream.read();
            if (data == 0) {
                noAuth = true;
            }
        }
        if (!noAuth) {
            throw new RuntimeException("只支持无认证方式!!");
        }
        //响应无需验证
        socket.getOutputStream().write(new byte[]{5, 0});
    }

    private static void connectServer(Socket socket) throws IOException {

        InputStream inputStream = socket.getInputStream();

        int ver = inputStream.read();
        int cmd = inputStream.read();
        int rsv = inputStream.read();
        int atyp = inputStream.read();
        String host = "";

        //用于回复:成功
        var replyStream  = new ByteArrayOutputStream();
        replyStream.write(ver);
        replyStream.write(0);
        replyStream.write(rsv);
        replyStream.write(atyp);

        if (atyp == 1) {//IPv4 地址
            byte[] data = inputStream.readNBytes(4);
            host = InetAddress.getByAddress(data).getHostAddress();
            replyStream.write(data);
        } else if (atyp == 3) {//表示域名
            int len = inputStream.read();
            byte[] data = inputStream.readNBytes(len);
            host = new String(data);
            replyStream.write(len);
            replyStream.write(data);
        } else if (atyp == 4) {//IPv6 地址
            byte[] data = inputStream.readNBytes(16);
            host = InetAddress.getByAddress(data).getHostAddress();
            replyStream.write(data);
        } else {
            throw new RuntimeException("不支持的地址类型: " + atyp);
        }
        byte[] data = inputStream.readNBytes(2);
        int port = ((data[0] & 0xFF) << 8) | (data[1] & 0xFF);
        replyStream.write(data);
        System.out.println("请求连接: " + host + ":" + port);

        Socket targetSocket = new Socket(host, port);
        //响应连接成功
        socket.getOutputStream().write(replyStream.toByteArray());

        executorService.submit(() -> {
            //转发数据
            try (targetSocket;socket) {
                targetSocket.getInputStream().transferTo(socket.getOutputStream());
            } catch (Exception e) {
//                e.printStackTrace();
            }

        });
        //转发数据
        try (targetSocket){
            socket.getInputStream().transferTo(targetSocket.getOutputStream());
        }catch (Exception e) {
//            e.printStackTrace();
        }

    }

}
```

该版本使用异或（XOR）加密转发的数据，适合测试使用。

特点： 可设置为客户端或服务端；

```
使用 JVM 参数 -Dtype=client 启动客户端；

所有数据通过字符 'c' 进行异或加密。
```

```
public class ClientOrServer {

    static ExecutorService executorService = Executors.newCachedThreadPool();

    static String type = ""; // 可以设置为 "server" 或 "client"，根据需要切换

    public static void main(String[] args) throws IOException {

        // 启动命令示例：java -Dtype=server org.example.ClientOrServer
        // 读取系统属性
        type = System.getProperty("type","client");

        try (var serverSocket = new ServerSocket(8888)) {
            System.out.println("Socks5 代理服务器已启动，监听端口: 8888");
            while (!Thread.interrupted()) {
                var socket = serverSocket.accept();
                executorService.submit(() -> {
                    clientHandler(socket);
                });
            }
        }
    }

    private static void clientHandler(Socket socket) {

        try (socket) {

            negotiatedConnection(socket);

            connectServer(socket);

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void negotiatedConnection(Socket socket) throws IOException {
        //按需读取数据
        InputStream inputStream = socket.getInputStream();
        //协议判断
        if (inputStream.read() != 5) {
            throw new RuntimeException("只支持socks5协议!!");
        }
        //认证方式
        int authMethodCount = inputStream.read();
        boolean noAuth = false;
        for (int i = 0; i < authMethodCount; i++) {
            int data = inputStream.read();
            if (data == 0) {
                noAuth = true;
            }
        }
        if (!noAuth) {
            throw new RuntimeException("只支持无认证方式!!");
        }
        //响应无需验证
        socket.getOutputStream().write(new byte[]{5, 0});
    }

    private static void connectServer(Socket socket) throws IOException {

        InputStream inputStream = socket.getInputStream();

        int ver = inputStream.read();
        int cmd = inputStream.read();
        int rsv = inputStream.read();
        int atyp = inputStream.read();
        String host = "";

        //用于回复:成功
        var replyStream  = new ByteArrayOutputStream();
        replyStream.write(ver);
        replyStream.write(0);
        replyStream.write(rsv);
        replyStream.write(atyp);

        if (atyp == 1) {//IPv4 地址
            byte[] data = inputStream.readNBytes(4);
            host = InetAddress.getByAddress(data).getHostAddress();
            replyStream.write(data);
        } else if (atyp == 3) {//表示域名
            int len = inputStream.read();
            byte[] data = inputStream.readNBytes(len);
            host = new String(data);
            replyStream.write(len);
            replyStream.write(data);
        } else if (atyp == 4) {//IPv6 地址
            byte[] data = inputStream.readNBytes(16);
            host = InetAddress.getByAddress(data).getHostAddress();
            replyStream.write(data);
        } else {
            throw new RuntimeException("不支持的地址类型: " + atyp);
        }
        byte[] data = inputStream.readNBytes(2);
        int port = ((data[0] & 0xFF) << 8) | (data[1] & 0xFF);
        replyStream.write(data);
        System.out.println("请求连接: " + host + ":" + port);


        Proxy proxy = Proxy.NO_PROXY;
        host = xorEncrypt(host, 'c'); // 使用异或加密
        System.out.println("位运算后的主机: " + host);


        if(type.equals("client")) {
            proxy = new Proxy(Proxy.Type.SOCKS, new InetSocketAddress("154.201.78.157", 8888));
        }

        Socket targetSocket = new Socket(proxy);
        targetSocket.connect(new InetSocketAddress(host,port));

        //响应连接成功
        socket.getOutputStream().write(replyStream.toByteArray());

        executorService.submit(() -> {
            //转发数据
            try (socket) {
                transferTo(targetSocket.getInputStream(), socket.getOutputStream());
            } catch (Exception e) {
//                e.printStackTrace();
            }

        });
        //转发数据
        try (targetSocket){
            transferTo(socket.getInputStream(), targetSocket.getOutputStream());
        }catch (Exception e) {
//            e.printStackTrace();
        }

    }

    private static void transferTo(InputStream in,OutputStream out) throws IOException {
        byte[] buffer = new byte[1024];
        int read;
        while ((read = in.read(buffer, 0, buffer.length)) >= 0) {

            // 对每个字节进行异或加密
            for (int i = 0; i < read; i++) {
                buffer[i] = (byte) (buffer[i] ^ 'c'); // 异或运算加密
            }
            out.write(buffer, 0, read);
        }
    }

    public static String xorEncrypt(String input, char key) {
        char[] chars = input.toCharArray();
        for (int i = 0; i < chars.length; i++) {
            chars[i] = (char) (chars[i] ^ key); // 使用异或运算
        }
        return new String(chars);
    }

}
```

- - -

*    [Java](/tags/java/)
