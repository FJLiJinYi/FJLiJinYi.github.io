> 因为公司项目需要，实现跟客户设备通信方式相同的自己设备（md,不就是没有备用方案，临时拉我上来做的吗？啥都不清楚，跟客户设备对接的人都也是一脸懵逼，我要只能靠自己了。-----小声哔哔）

**在网上找到了相关的demo跟jar包，开启自己的使用Netty填坑之路。**

Jar 包下载路径：
		[下载地址](https://download.csdn.net/download/binbinxiaoz/10488657)
当然这个分数有点高，这个是我网上找的这个。要是没有分数那就留言或者发邮件给我（fflijinyi@foxmail.com）

关于使用以及遇到的问题：
1、==注意编解码问题。注意编解码问题，一定要注意编解码问题。==
> 注意点，关于数据传输编解码问题，一定要根据服务器来实现。不然很容出现能收到服务器的数据，但是发出去的数据服务器收到不到，或者是服务器能收到客户端发出去的数据，但是客户端收不到任何消息。

2、下面的代码我是参考网上一个小哥的代码上去做的。因为没有保存网址，不能附上链接，万分抱歉。

### 实现部分
NettyListener 用于实现抽象接口已经常量的定义：

```java
package com.example.lijinyi.*.*.api.socketApi;

/**
 * author : lijinyi
 * e-mail : fflijinyi@foxmail.com
 * date   : 2019/9/2516:11
 * desc   :
 * version: 1.0
 */
public interface NettyListener {
    byte STATUS_CONNECT_SUCCESS = 1;//连接成功

    byte STATUS_CONNECT_CLOSED = 0;//关闭连接

    byte STATUS_CONNECT_ERROR = 0;//连接失败

    int STATUS_SUCCESS = 0;

    int STATUS_ERROR = 1;

    //人像库操作
    int ADD_LIB = 1;
    int CHG_LIB = 2;
    int DEL_LIB = 3;

    //人像操作
    int ADD_FACE = 4;
    int CHG_FACE = 5;
    int DEL_FACE = 6;

    /**
     * 当接收到系统消息
     */
    void onMessageResponse(int statusCode,Object msg);

    /**
     * 当连接状态发生变化时调用
     */
    void onServiceStatusConnectChanged(int statusCode);

}
```

NettyClient 这个部分代码实现了断线重连跟心跳

```java
package com.example.lijinyi.*.*.manager;
import com.example.lijinyi.*.*.api.socketApi.NettyListener;



/**
 * author : lijinyi
 * e-mail : fflijinyi@foxmail.com
 * date   : 2019/9/2516:12
 * desc   :
 * version: 1.0
 */
public class NettyClient {

    private static final String TAG = "NettyClient";

    private EventLoopGroup group;//Bootstrap参数

    private NettyListener listener;//写的接口用来接收服务端返回的值

    private Channel channel;//通过对象发送数据到服务端

    private boolean isConnect = false;//判断是否连接了

    private static int reconnectNum = Integer.MAX_VALUE;//定义的重连到时候用
    private boolean isNeedReconnect = true;//是否需要重连
    private boolean isConnecting = false;//是否正在连接
    private long reconnectIntervalTime = 5000;//重连的时间

    public String host;//ip
    public int tcp_port;//端口

    /*
   构造 传入 ip和端口
    */
    public NettyClient(String host, int tcp_port) {
        this.host = host;
        this.tcp_port = tcp_port;
    }

    /*
   连接方法
    */
    public void connect() {

        if (isConnecting) {
            return;
        }
        //起个线程
        Thread clientThread = new Thread("client-Netty") {
            @Override
            public void run() {
                super.run();
                isNeedReconnect = true;
                reconnectNum = Integer.MAX_VALUE;
                connectServer();
            }
        };
        clientThread.start();
    }

    //连接时的具体参数设置
    private void connectServer() {
        synchronized (NettyClient.this) {
            ChannelFuture channelFuture = null;//连接管理对象
            if (!isConnect) {
                isConnecting = true;
                group = new NioEventLoopGroup();//设置的连接group
                Bootstrap bootstrap = new Bootstrap().group(group)//设置的一系列连接参数操作等
                        .option(ChannelOption.TCP_NODELAY, true)//屏蔽Nagle算法试图
                        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
                        .channel(NioSocketChannel.class)
                        .handler(new ChannelInitializer<SocketChannel>() { // 5
                            @Override
                            public void initChannel(SocketChannel ch) throws Exception {
									
								//这部分编解码根据自己的实际需要使用。
			
                                ch.pipeline().addLast(new StringDecoder());//进行字符串的编解码设置
                                ch.pipeline().addLast(new StringEncoder());
                                ch.pipeline().addLast(new ReadTimeoutHandler(60));//设置超时时间
                                ch.pipeline().addLast(new HttpRequestEncoder());// 客户端对发送的httpRequest进行编码
                                ch.pipeline().addLast(new HttpResponseDecoder());// 客户端需要对服务端返回的httpresopnse解码
                                ch.pipeline().addLast(new HttpRequestDecoder());
                                ch.pipeline().addLast(new HttpResponseEncoder());

                                ch.pipeline().addLast(new NettyClientHandler(listener));//需要的handlerAdapter
                            }
                        });

                try {
                    //连接监听
                    channelFuture = bootstrap.connect(host, tcp_port).addListener(new ChannelFutureListener() {
                        @Override
                        public void operationComplete(ChannelFuture channelFuture) throws Exception {
                            if (channelFuture.isSuccess()) {
                                Log.e(TAG, "连接成功");
                                isConnect = true;
                                channel = channelFuture.channel();
                            } else {
                                Log.e(TAG, "连接失败");
                                isConnect = false;
                            }
                            isConnecting = false;
                        }
                    }).sync();

                    // 等待连接关闭
                    channelFuture.channel().closeFuture().sync();
                    Log.e(TAG, " 断开连接");
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    isConnect = false;
                    listener.onServiceStatusConnectChanged(NettyListener.STATUS_CONNECT_CLOSED);//STATUS_CONNECT_CLOSED 这我自己定义的 接口标识
                    if (null != channelFuture) {
                        if (channelFuture.channel() != null && channelFuture.channel().isOpen()) {
                            channelFuture.channel().close();
                        }
                    }
                    group.shutdownGracefully();
                    reconnect();//重新连接
                }
            }
        }
    }

    //断开连接
    public void disconnect() {
        Log.e(TAG, "disconnect");
        isNeedReconnect = false;
        group.shutdownGracefully();
    }

    //重新连接
    public void reconnect() {
        Log.e(TAG, "reconnect");
        if (isNeedReconnect && reconnectNum > 0 && !isConnect) {
            reconnectNum--;
            SystemClock.sleep(reconnectIntervalTime);
            if (isNeedReconnect && reconnectNum > 0 && !isConnect) {
                Log.e(TAG, "重新连接");
                connectServer();
            }
        }
    }

    //发送消息到服务端。
    public boolean sendMsgHeartToServer(String data, String sUri, ChannelFutureListener listener) {
        boolean flag = channel != null && isConnect;
        try {
            if (flag) {

                URI uri = new URI(sUri);
                ByteBuf byteBuf = Unpooled.wrappedBuffer(data.getBytes(StandardCharsets.UTF_8));

                FullHttpRequest request = new DefaultFullHttpRequest(HttpVersion.HTTP_1_1, HttpMethod.POST, uri.toASCIIString(), byteBuf);
                request.headers().set(HttpHeaderNames.CONNECTION, HttpHeaderValues.KEEP_ALIVE);
                request.headers().set(HttpHeaderNames.CONTENT_LENGTH, request.content().readableBytes());
                request.headers().set(HttpHeaderNames.CONTENT_TYPE, HttpHeaderValues.APPLICATION_JSON);

                channel.writeAndFlush(request).addListener(listener).sync();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        return flag;
    }

  

    //重连时间
    public void setReconnectNum(int reconnectNum) {
        NettyClient.reconnectNum = reconnectNum;
    }

    public void setReconnectIntervalTime(long reconnectIntervalTime) {
        this.reconnectIntervalTime = reconnectIntervalTime;
    }

    //现在连接的状态
    public boolean getConnectStatus() {
        return isConnect;
    }

    public boolean isConnecting() {
        return isConnecting;
    }

    public void setConnectStatus(boolean status) {
        this.isConnect = status;
    }

    public void setListener(NettyListener listener) {
        this.listener = listener;
    }

}
```
这个部分代码实际是通过启动一个线程来实现socket运行，通过心跳发送的来判断是否发送成功来设置标志位，从而实现断线重连的效果。

NettyClientHandler 相关代码

```java
package com.example.lijinyi.*.*.manager;

/**
 * author : lijinyi
 * e-mail : fflijinyi@foxmail.com
 * date   : 2019/9/2516:23
 * desc   :
 * version: 1.0
 */
public class NettyClientHandler extends ChannelInboundHandlerAdapter {
    private static final String TAG = "NettyClientHandler";
    private NettyListener listener;

    //人像库处理
    private LibInfoDao libManagerDao = null;
    private FaceInfoDao faceManagerDao = null;
    private FaceEnvironment environment = null;
    private JSONObject object;
    private int statuCode = -1;
    private String strStatu = "";

    public NettyClientHandler(NettyListener listener) {
        this.listener = listener;
    }

    //每次给服务器发送的东西， 让服务器知道我们在连接 ---心跳部分
	//这个可以定义成我们想要发送心跳包的格式
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            if (event.state() == IdleState.WRITER_IDLE) {
                ctx.channel().writeAndFlush("Heartbeat" + System.getProperty("line.separator"));
            }
        }
    }

    /**
     * 连接成功
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Log.e(TAG, "channelActive");
        super.channelActive(ctx);
        listener.onServiceStatusConnectChanged(NettyListener.STATUS_CONNECT_SUCCESS);

    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        super.channelInactive(ctx);
        Log.e(TAG, "channelInactive");
    }

    //接收消息的地方， 接口调用返回到activity了
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //这里是接收从服务器发送过来的数据。
        System.out.println("客户端开始读取服务端过来的信息" + " " + msg.toString());
        
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // 当引发异常时关闭连接。
        Log.e(TAG, "exceptionCaught");
        listener.onServiceStatusConnectChanged(NettyListener.STATUS_CONNECT_ERROR);
        cause.printStackTrace();
        ctx.close();
    }

}

```

整个Netty 实现客户端主要注意点就是编解码问题，channelRead() 接收数据部分，writeAndFlush()发送数据部分。其他也没有什么需要注意的地方了。

那么就这个样了，加油~

# ヾ(◍°∇°◍)ﾉﾞ
