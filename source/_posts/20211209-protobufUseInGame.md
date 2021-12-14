---
title: 浏览器使用websocket通过protobuf协议与netty服务端通信
date: 2021-12-09 10:39:30
tags:
- Protobuf
- Netty
- websocket
- java
---

## 概要
本文主要介绍如何通过浏览器WEB使用websocket、protobuf协议与netty服务端通信。
主要涉及到的技术点：js、websocket、java、netty、protobuf，下面会详细介绍如何使用。

## 前端
主要涉及到四个文件，看截图，完成这些，客户端这边就OK了。
![](/image/protoClientDemo.png)

### ptoto文件
1. SubscribeReq.proto，用于发送到服务端：
```
syntax="proto3";
package req;

message SubscribeReq {
  int32 subReqId = 1;
  string userName = 2;
  string productName = 3;
  string address = 4;
}
```
2. SubscribeResp.proto，用于解析服务端发送过来的消息：
```
syntax="proto3";
package res;

message SubscribeResp{
  int32 subReqId = 1;
  int32 respCode = 2;
  string desc = 3;
}
```
注：package对应客户端中的代码root.lookupType(className)，className:#{package}.#{message}

### protobuf.min.js
[protobuf.min.js](https://github.com/protobufjs/protobuf.js/releases)下载地址
1. 下载
   选择最近版本zip下载
2. 解压，获取目标文件
   protobuf.min.js文件在解压后dist目录下，如图：
![](/image/protoJsFile.png)
3. 放到html同级目录，并在html中引用

### html客户端
TestProtobuf.html：
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <script src="protobuf.min.js"></script>
    <title>Title</title>
</head>
<body>
<button onclick="sendMsg()">手动发送</button>
</body>
<script>
    let buffer;

    /**
     * 手动发送消息
     */
    function sendMsg(){
        let payload = {subReqId: 1, userName: "cjf", productName: "hello server 我是客户端   111 !!", address:"地址"};
        let message = reqMessage.create(payload); // or use .fromObject if conversion is necessary
        buffer = reqMessage.encode(message).finish();
        webSocket.send(buffer);
    }
    // 客户端请求对象
    let reqMessage;
    // 服务端返回对象
    let resMessage;

    /**
     * 初始化 reqMessage
     * @param fileName
     * @param className
     * @param type 1:request,2:response
     */
    function initCusMsg(fileName, className, type) {

        return protobuf.load(fileName)
            .then((root) => {

                if (type === 1) {
                    reqMessage = root.lookupType(className);
                    return reqMessage;
                } else if (type === 2) {
                    resMessage = root.lookupType(className);
                    return resMessage;
                }
            });
    }

    const address = "ws://127.0.0.1:9999/ws";

    let webSocket = new WebSocket(address);

    webSocket.onopen = function () {


        console.log("webSocket连接建立成功... " + "服务端address: " + address);
        //连接成功 发送消息

        let reqMsgPromise = initCusMsg("SubscribeReq.proto", "req.SubscribeReq", 1);

        reqMsgPromise.then((cusMsg) => {

            //参考 https://github.com/protobufjs/protobuf.js#using-proto-files

            // Exemplary payload
            let cuTime = new Date().getTime();
            let payload = {
                subReqId: 1, userName: "cjf", productName: "hello server 我是客户端 2222!!", address:"地址"};
            // Verify the payload if necessary (i.e. when possibly incomplete or invalid)
            let errMsg = cusMsg.verify(payload);
            if (errMsg) {

                throw Error(errMsg);
            }
            // Create a new message
            let message = cusMsg.create(payload); // or use .fromObject if conversion is necessary
            // Encode a message to an Uint8Array (browser) or Buffer (node)
            buffer = cusMsg.encode(message).finish();

            webSocket.send(buffer);
        });
    };

    //接收消息
    let reader = new FileReader();
    //监听服务端响应消息
    webSocket.onmessage = function (event) {
        let resMsgPromise = initCusMsg("SubscribeResp.proto", "res.SubscribeResp", 2);

        resMsgPromise.then((cusMsg) => {
            reader.readAsArrayBuffer(event.data);
            reader.onload = () => {

                let arrayBuffer = reader.result;
                let buffer = new Uint8Array(arrayBuffer);

                let resObject = resMessage.decode(buffer);
                let errMsg = cusMsg.verify(resObject);
                if (errMsg) {

                    throw Error(errMsg);
                }
                console.log(resObject)
            };
        });
    }
</script>
</body>
</html>
```

## 后端
主要涉及到几个文件，看截图，完成这些，服务端这边就OK了。
![](/image/protoServerDemo.png)

### ptoto文件
1. SubscribeReq.proto：定义解析客户端发过来的数据格式
```
syntax="proto3";
option java_package = "com.gate.protobuf";
option java_outer_classname = "SubscribeReqProto";

message SubscribeReq {
  int32 subReqId = 1;
  string userName = 2;
  string productName = 3;
  string address = 4;
}

```
2. SubscribeResp.proto：定义发送到客户端的数据格式
```
syntax="proto3";
option java_package = "com.gate.protobuf";
option java_outer_classname="SubscribeRespProto";

message SubscribeResp{
  required int32 subReqId = 1;
  required int32 respCode = 2;
  required string desc = 3;
}

```

3. 下载[protobuf-java-3.19.1.zip](https://github.com/protocolbuffers/protobuf/releases)，解压后可以配置下全局命令执行。
4. 生成java文件
执行：
```
protoc --java_out=./ SubscribeReq.proto
protoc --java_out=./ SubscribeResp.proto
```
生成SubscribeReqProto.java、SubscribeRespProto.java


注：java_package路径对应生成的java包路径， java_outer_classname对应生成的java文件名

### 依赖包引用
```
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.19.1</version>
</dependency>

```

注：protobuf-java的版本要对应protoc的版本

### netty服务端
Server.java代码:
```
public class Server {
    /**
     * 自定义ChannelInitializer
     */
    public void start() {
        short port = 9999;
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        NioEventLoopGroup boos = new NioEventLoopGroup();
        NioEventLoopGroup worker = new NioEventLoopGroup();
        serverBootstrap.group(boos, worker);
        serverBootstrap.channel(NioServerSocketChannel.class);
        serverBootstrap.childHandler(new CustomChannelInitializer());
        try {
            Channel channel = serverBootstrap.bind(port).sync().channel();
            System.out.println("服务端启动 端口:" + port);
            channel.closeFuture().await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        new Server().start();
    }
}
```

CustomChannelInitializer.java代码：
```
public class CustomChannelInitializer  extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel socketChannel) {
        ChannelPipeline pipeline = socketChannel.pipeline();

        pipeline.addLast(new HttpServerCodec());
        pipeline.addLast(new ChunkedWriteHandler());
        pipeline.addLast(new HttpObjectAggregator(1024 * 64));

        pipeline.addLast(new WebSocketServerProtocolHandler("/ws"));

        //将WebSocketFrame转为ByteBuf 以便后面的 ProtobufDecoder 解码
        pipeline.addLast(new MessageToMessageDecoder<WebSocketFrame>() {

            @Override
            protected void decode(ChannelHandlerContext ctx, WebSocketFrame frame, List<Object> out) throws Exception {

                ByteBuf byteBuf = frame.content();
                byteBuf.retain();
                out.add(byteBuf);
            }
        });

        pipeline.addLast(new ProtobufDecoder(SubscribeReqProto.SubscribeReq.getDefaultInstance()));
        //自定义入站处理
        pipeline.addLast(new TestProtoBufInboundHandler());

        //出站处理 将protoBuf实例转为WebSocketFrame
        pipeline.addLast(new ProtobufEncoder() {

            @Override
            protected void encode(ChannelHandlerContext ctx, MessageLiteOrBuilder msg, List<Object> out) throws Exception {

                SubscribeRespProto.SubscribeResp mpMsg = (SubscribeRespProto.SubscribeResp) msg;
                WebSocketFrame frame = new BinaryWebSocketFrame(Unpooled.wrappedBuffer(mpMsg.toByteArray()));
                out.add(frame);
            }
        });
    }
}
```

TestProtoBufInboundHandler.java代码:
```
public class TestProtoBufInboundHandler extends SimpleChannelInboundHandler<SubscribeReqProto.SubscribeReq> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, SubscribeReqProto.SubscribeReq msg) throws Exception {
        Gson gson = new Gson();
        System.out.printf("服务端收到请求数据：%s", gson.toJson(msg));

        //响应消息
        SubscribeRespProto.SubscribeResp.Builder builder = SubscribeRespProto.SubscribeResp.newBuilder();
        builder.setRespCode(200);
        builder.setSubReqId(1);
        builder.setDesc("success receive!");
        SubscribeRespProto.SubscribeResp build = builder.build();

        ctx.channel().writeAndFlush(build);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();//将消息从发送缓冲区中写入socketchannel中
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        ctx.close();
    }
}
```

## 启动调试
完成以上操作、配置后，就可以先启动Server.java，再打开TestProtobuf.html，进行调试了。