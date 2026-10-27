---
title: Spring Boot 快速集成 Netty 教程（WebSocket 实时通信）
published: 2026-10-27
tags: [Netty, Spring Boot, WebSocket, 实时通信, 长连接, Redisson]
category: 学习笔记
draft: false
---

# Spring Boot 快速集成 Netty 教程（WebSocket 实时通信）

> 基于 easychat-java 项目的真实实现整理，所有代码均可直接复制粘贴。
> 效果：Spring Boot 启动后，Netty 在**独立端口**上开启一个 WebSocket 服务，前端连接后可实时收发消息、自动踢掉超时连接。

---

## 一、整体思路

整个集成分 6 步（第 7 章是进阶的集群部署方案）：

```
① pom.xml 引入依赖
② application.properties 配置 ws 端口
③ NettyWebSocketStarter —— Netty 服务端启动类（核心）
④ HandlerWebSocket / HandlerHeartBeat —— 两个消息处理器
⑤ ChannelContextUtils —— 管理所有在线用户的连接（最终执行推送）
⑥ InitRun —— Spring Boot 启动完成后拉起 Netty
⑦ MessageHandler + Redisson —— 可选进阶：跨节点广播，支持集群部署
```

数据流向：

```
前端 ws://ip:5051/ws?token=xxx
        │
        ▼
Netty(5051端口) 握手 → 校验token → 把 userId 和 Channel 绑定存入 Map
        │
        ▼
业务层(Service) 要推消息 → 从 Map 里找到对方的 Channel → writeAndFlush 发出
```

---

## 二、第 1 步：引入依赖（pom.xml）

在 `properties` 中声明版本，在 `dependencies` 中引入 `netty-all`（一个包包含 Netty 全部模块，最省事）：

```xml
<properties>
    <netty.version>4.1.50.Final</netty.version>
</properties>

<dependencies>
    <!--netty-->
    <dependency>
        <groupId>io.netty</groupId>
        <artifactId>netty-all</artifactId>
        <version>${netty.version}</version>
    </dependency>

    <!--redisson：集群部署时跨节点广播消息用（见第七章）-->
    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson</artifactId>
        <version>3.12.3</version>
    </dependency>
</dependencies>
```

---

## 三、第 2 步：编写配置文件

在 `application.properties` 中配置 WebSocket 端口（**注意：这个端口和 Spring Boot 的 server.port 不是一回事**，Netty 自己监听一个端口）：

```properties
# websocket 端口（Netty 使用）
ws.port=5051
```

再写一个配置类读取它（文件：`entity/config/AppConfig.java`）：

```java
package com.easychat.entity.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component("appConfig")
public class AppConfig {

    /**
     * websocket 端口
     */
    @Value("${ws.port:}")
    private Integer wsPort;

    public Integer getWsPort() {
        return wsPort;
    }
}
```

---

## 四、第 3 步：Netty 服务端启动类（核心）

文件：`websocket/netty/NettyWebSocketStarter.java`

关键概念一句话解释：
- **bossGroup**：负责"接电话"（接受新连接），1 个线程就够
- **workerGroup**：负责"干活"（处理读写消息），默认 CPU 核数×2
- **pipeline**：每个连接的处理流水线，消息按加入顺序流过一个个处理器

```java
package com.easychat.websocket.netty;

import com.easychat.entity.config.AppConfig;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;
import io.netty.handler.timeout.IdleStateHandler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.annotation.PreDestroy;
import javax.annotation.Resource;
import java.util.concurrent.TimeUnit;

@Component
public class NettyWebSocketStarter implements Runnable {

    private static final Logger logger = LoggerFactory.getLogger(NettyWebSocketStarter.class);

    @Resource
    private AppConfig appConfig;

    @Resource
    private HandlerWebSocket handlerWebSocket;

    /**
     * boss线程组，用于处理连接
     */
    private EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    /**
     * work线程组，用于处理消息
     */
    private EventLoopGroup workerGroup = new NioEventLoopGroup();

    /**
     * 资源关闭——在容器销毁时关闭
     */
    @PreDestroy
    public void close() {
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }

    @Override
    public void run() {
        try {
            //创建服务端启动助手
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(bossGroup, workerGroup);
            serverBootstrap.channel(NioServerSocketChannel.class)
                    .handler(new LoggingHandler(LogLevel.DEBUG))
                    .childHandler(new ChannelInitializer<Channel>() {
                        @Override
                        protected void initChannel(Channel channel) {
                            ChannelPipeline pipeline = channel.pipeline();
                            // http协议的编码器、解码器（websocket握手前是http请求）
                            pipeline.addLast(new HttpServerCodec());
                            // 聚合http消息，保证接收的http请求完整
                            pipeline.addLast(new HttpObjectAggregator(64 * 1024));
                            // 心跳：60秒内没读到客户端任何数据，就触发读空闲事件
                            pipeline.addLast(new IdleStateHandler(60, 0, 0, TimeUnit.SECONDS));
                            pipeline.addLast(new HandlerHeartBeat());
                            // 将http协议升级为ws协议，路径为 /ws
                            pipeline.addLast(new WebSocketServerProtocolHandler("/ws", null, true, 64 * 1024, true, true, 10000L));
                            pipeline.addLast(handlerWebSocket);
                        }
                    });
            //启动
            ChannelFuture channelFuture = serverBootstrap.bind(appConfig.getWsPort()).sync();
            logger.info("Netty服务端启动成功,端口:{}", appConfig.getWsPort());
            channelFuture.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

**pipeline 中的处理器顺序很重要**：请求先进 HTTP 解码 → 聚合 → 心跳检测 → 协议升级为 WebSocket → 最后交给业务处理器。

---

## 五、第 4 步：编写两个处理器

### 5.1 心跳处理器

文件：`websocket/netty/HandlerHeartBeat.java`
作用：配合 `IdleStateHandler`，60 秒没收到客户端消息就踢掉这个"僵尸连接"。

```java
package com.easychat.websocket.netty;

import io.netty.channel.ChannelDuplexHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.timeout.IdleState;
import io.netty.handler.timeout.IdleStateEvent;
import io.netty.util.Attribute;
import io.netty.util.AttributeKey;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class HandlerHeartBeat extends ChannelDuplexHandler {

    private static final Logger logger = LoggerFactory.getLogger(HandlerHeartBeat.class);

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent e = (IdleStateEvent) evt;
            if (e.state() == IdleState.READER_IDLE) {
                Attribute<String> attribute = ctx.channel().attr(AttributeKey.valueOf(ctx.channel().id().toString()));
                String userId = attribute.get();
                logger.info("用户{}没有发送心跳断开连接", userId);
                ctx.close();
            }
        }
    }
}
```

### 5.2 业务处理器（握手校验 + 收消息）

文件：`websocket/netty/HandlerWebSocket.java`
作用：
1. **握手完成时**（`userEventTriggered`）：从连接 URL `?token=xxx` 中取出 token，校验通过后把 userId 和 Channel 绑定；
2. **收到消息时**（`channelRead0`）：刷新该用户的心跳时间；
3. **连接断开时**（`channelInactive`）：清理该用户的连接。

```java
package com.easychat.websocket.netty;

import com.easychat.entity.dto.TokenUserInfoDto;
import com.easychat.redis.RedisComponet;
import com.easychat.utils.StringTools;
import com.easychat.websocket.ChannelContextUtils;
import io.netty.channel.Channel;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;
import io.netty.util.Attribute;
import io.netty.util.AttributeKey;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;

/**
 * 设置通道共享（所有连接共用这一个Handler实例，必须加@Sharable）
 */
@ChannelHandler.Sharable
@Component("handlerWebSocket")
public class HandlerWebSocket extends SimpleChannelInboundHandler<TextWebSocketFrame> {

    private static final Logger logger = LoggerFactory.getLogger(HandlerWebSocket.class);

    @Resource
    private ChannelContextUtils channelContextUtils;

    @Resource
    private RedisComponet redisComponet;

    /**
     * 连接建立时调用
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        logger.info("有新的连接加入。。。");
    }

    /**
     * 连接断开时调用，做清理工作
     */
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        logger.info("有连接已经断开。。。");
        channelContextUtils.removeContext(ctx.channel());
    }

    /**
     * 读就绪事件：收到客户端发来的文本消息
     */
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame textWebSocketFrame) throws Exception {
        //收到消息即视为心跳，刷新用户的在线心跳时间
        Channel channel = ctx.channel();
        Attribute<String> attribute = channel.attr(AttributeKey.valueOf(channel.id().toString()));
        String userId = attribute.get();
        redisComponet.saveUserHeartBeat(userId);
    }

    /**
     * 用户事件触发：这里处理"握手完成"事件，做token校验和用户上线
     */
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof WebSocketServerProtocolHandler.HandshakeComplete) {
            WebSocketServerProtocolHandler.HandshakeComplete complete =
                    (WebSocketServerProtocolHandler.HandshakeComplete) evt;
            String url = complete.requestUri();
            String token = getToken(url);
            if (token == null) {
                ctx.channel().close();
                return;
            }
            TokenUserInfoDto tokenUserInfoDto = redisComponet.getTokenUserInfoDto(token);
            if (null == tokenUserInfoDto) {
                ctx.channel().close();
                return;
            }
            //校验通过，用户上线
            channelContextUtils.addContext(tokenUserInfoDto.getUserId(), ctx.channel());
        }
    }

    private String getToken(String url) {
        if (StringTools.isEmpty(url) || url.indexOf("?") == -1) {
            return null;
        }
        String[] queryParams = url.split("\\?");
        if (queryParams.length < 2) {
            return url;
        }
        String[] params = queryParams[1].split("=");
        if (params.length != 2) {
            return url;
        }
        return params[1];
    }
}
```

> 注意：Handler 里能直接 `@Resource` 注入 Spring Bean（如 RedisComponet），因为 Handler 本身就是 `@Component`。这是 Netty 与 Spring 业务层打通的关键。

---

## 六、第 5 步：连接管理 + 发送消息 API

文件：`websocket/ChannelContextUtils.java`

这是**业务层调用 Netty 发消息的统一入口**。核心数据结构是两个 Map：
- `USER_CONTEXT_MAP`：userId → Channel（单聊用）
- `GROUP_CONTEXT_MAP`：groupId → ChannelGroup（群聊用）

```java
package com.easychat.websocket;

import com.alibaba.fastjson.JSON;
import com.easychat.entity.dto.MessageSendDto;
import com.easychat.entity.enums.UserContactTypeEnum;
import com.easychat.utils.StringTools;
import io.netty.channel.Channel;
import io.netty.channel.group.ChannelGroup;
import io.netty.channel.group.DefaultChannelGroup;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import io.netty.util.Attribute;
import io.netty.util.AttributeKey;
import io.netty.util.concurrent.GlobalEventExecutor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

@Component("channelContextUtils")
public class ChannelContextUtils {

    private static final Logger logger = LoggerFactory.getLogger(ChannelContextUtils.class);

    /** 用户ID -> 通道 */
    public static final ConcurrentMap<String, Channel> USER_CONTEXT_MAP = new ConcurrentHashMap();

    /** 群组ID -> 通道组 */
    public static final ConcurrentMap<String, ChannelGroup> GROUP_CONTEXT_MAP = new ConcurrentHashMap();

    /**
     * 用户上线：绑定 userId 与 channel
     */
    public void addContext(String userId, Channel channel) {
        String channelId = channel.id().toString();
        AttributeKey attributeKey = null;
        if (!AttributeKey.exists(channelId)) {
            attributeKey = AttributeKey.newInstance(channel.id().toString());
        } else {
            attributeKey = AttributeKey.valueOf(channel.id().toString());
        }
        //把userId挂在channel的属性上，之后随时能取出来
        channel.attr(attributeKey).set(userId);

        USER_CONTEXT_MAP.put(userId, channel);
    }

    /**
     * 用户下线：解绑并清理
     */
    public void removeContext(Channel channel) {
        Attribute<String> attribute = channel.attr(AttributeKey.valueOf(channel.id().toString()));
        String userId = attribute.get();
        if (!StringTools.isEmpty(userId)) {
            USER_CONTEXT_MAP.remove(userId);
        }
    }

    /**
     * 业务层发消息的统一入口：根据contactId前缀自动判断发给个人还是群
     */
    public void sendMessage(MessageSendDto messageSendDto) {
        UserContactTypeEnum contactTypeEnum = UserContactTypeEnum.getByPrefix(messageSendDto.getContactId());
        switch (contactTypeEnum) {
            case USER:
                send2User(messageSendDto);
                break;
            case GROUP:
                sendMsg2Group(messageSendDto);
        }
    }

    /**
     * 发送消息给指定用户
     */
    private void send2User(MessageSendDto messageSendDto) {
        String contactId = messageSendDto.getContactId();
        sendMsg(messageSendDto, contactId);
    }

    /**
     * 发送消息到整个群（群内所有在线成员）
     */
    private void sendMsg2Group(MessageSendDto messageSendDto) {
        if (messageSendDto.getContactId() == null) {
            return;
        }
        ChannelGroup group = GROUP_CONTEXT_MAP.get(messageSendDto.getContactId());
        if (group == null) {
            return;
        }
        group.writeAndFlush(new TextWebSocketFrame(JSON.toJSONString(messageSendDto)));
    }

    /**
     * 发送消息核心方法：找到对方的Channel，把DTO转成JSON用TextWebSocketFrame发出
     */
    private static void sendMsg(MessageSendDto messageSendDto, String reciveId) {
        if (reciveId == null) {
            return;
        }
        Channel sendChannel = USER_CONTEXT_MAP.get(reciveId);
        if (sendChannel == null) {
            //对方不在线，直接返回（也可在此做离线消息逻辑）
            return;
        }
        sendChannel.writeAndFlush(new TextWebSocketFrame(JSON.toJSONString(messageSendDto)));
    }

    /**
     * 把channel加入群组
     */
    private void add2Group(String groupId, Channel context) {
        ChannelGroup group = GROUP_CONTEXT_MAP.get(groupId);
        if (group == null) {
            group = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
            GROUP_CONTEXT_MAP.put(groupId, group);
        }
        if (context == null) {
            return;
        }
        group.add(context);
    }

    /**
     * 用户加入群聊时，把他的channel挂到群里
     */
    public void addUser2Group(String userId, String groupId) {
        Channel channel = USER_CONTEXT_MAP.get(userId);
        add2Group(groupId, channel);
    }
}
```

### 业务层怎么调用？（发送消息 API）

任何 Service / Controller 注入后一行代码即可推送消息。项目里的真实用法（如 `UserContactServiceImpl`）：

```java
@Resource
private ChannelContextUtils channelContextUtils;

public void pushSomething() {
    MessageSendDto messageSendDto = new MessageSendDto();
    messageSendDto.setMessageType(1);          //消息类型（自己定义的枚举）
    messageSendDto.setContactId("U123456");    //接收人ID（单聊）或群ID（群聊）
    messageSendDto.setSendUserId("U888888");   //发送人ID
    messageSendDto.setMessageContent("你好");   //内容
    channelContextUtils.sendMessage(messageSendDto);   //← 就是这一行
}
```

前端收到的就是一条 JSON 字符串（`TextWebSocketFrame` 包裹的 `MessageSendDto`）。

---

## 七、进阶：集成 Redisson，实现可集群部署（MessageHandler）

### 7.1 为什么要这一步？

第六章的 `ChannelContextUtils` 里，`USER_CONTEXT_MAP` / `GROUP_CONTEXT_MAP` 都是 **JVM 内存中的 Map**。单机部署没问题，但一旦部署两个节点（集群），问题就来了：

```
用户A连在节点1，用户B连在节点2
A给B发消息 → 请求落在节点1 → 节点1的Map里没有B的Channel → 消息丢失！
```

解决方案：借助 **Redisson 的 RTopic（发布/订阅）**。每个节点都往 Redis 的同一个频道发消息、同时也监听这个频道——任何一个节点收到业务消息，都先广播到频道，所有节点收到后各自在自己的本地 Map 里找目标 Channel 推送。找不到就说明用户不连在本节点，自然忽略。

```
                 ┌──────────── Redis 频道 message.topic ────────────┐
                 │                    (广播)                        │
节点1(MessageHandler订阅) ◄──────────┐             ┌──────► 节点2(MessageHandler订阅)
    │  本地Map找到A的Channel → 推给A │             │   本地Map没有B → 忽略
    └──────────────────────────────┘             └── 本地Map找到B的Channel → 推给B
```

### 7.2 第 1 步：引入 Redisson 依赖（pom.xml）

见第一章，`netty-all` 下面加一个即可（本项目用的版本）：

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.12.3</version>
</dependency>
```

### 7.3 第 2 步：创建 RedissonClient Bean

文件：`redis/RedisConfig.java`
直接复用 `application.properties` 里已有的 `spring.redis.host/port` 配置，不用新增配置项：

```java
package com.easychat.redis;

import io.lettuce.core.RedisConnectionException;
import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RedisConfig<V> {

    private static final Logger logger = LoggerFactory.getLogger(RedisConfig.class);

    @Value("${spring.redis.host:}")
    private String redisHost;

    @Value("${spring.redis.port:}")
    private Integer redisPort;

    @Bean(name = "redissonClient", destroyMethod = "shutdown")
    public RedissonClient redissonClient() {
        try {
            // 创建配置，指定redis地址及节点信息
            Config config = new Config();
            config.useSingleServer().setAddress("redis://" + redisHost + ":" + redisPort);
            // 根据config创建出RedissonClient实例
            RedissonClient redissonClient = Redisson.create(config);
            return redissonClient;
        } catch (RedisConnectionException e) {
            logger.error("redis配置错误，请检查redis配置");
        }
        return null;
    }
}
```

> 如果 Redis 是哨兵/集群模式，把 `useSingleServer()` 换成 `useSentinelServers()` / `useClusterServers()` 即可，其余代码不变。

### 7.4 第 3 步：编写 MessageHandler（集群广播核心）

文件：`websocket/MessageHandler.java`
整个类只有两个方法，逻辑非常简单：

- **`lisMessage()`**（`@PostConstruct`，Bean 创建后自动执行）：订阅 Redis 频道 `message.topic`，收到广播就调 `channelContextUtils.sendMessage()` 推给**本节点**上的连接；
- **`sendMessage()`**：不再直接找 Channel，而是把消息 `publish` 到频道——所有节点（包括自己）都会收到并各自推送。

```java
package com.easychat.websocket;

import com.alibaba.fastjson.JSON;
import com.easychat.entity.dto.MessageSendDto;
import org.redisson.api.RTopic;
import org.redisson.api.RedissonClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import javax.annotation.Resource;

@Component("messageHandler")
public class MessageHandler<T> {

    private static final Logger logger = LoggerFactory.getLogger(MessageHandler.class);

    /** Redis频道的名字，所有节点必须一致 */
    private static final String MESSAGE_TOPIC = "message.topic";

    @Resource
    private RedissonClient redissonClient;

    @Resource
    private ChannelContextUtils channelContextUtils;

    /**
     * 容器启动后自动订阅频道：收到广播消息 → 推给本节点的连接
     */
    @PostConstruct
    public void lisMessage() {
        RTopic rTopic = redissonClient.getTopic(MESSAGE_TOPIC);
        rTopic.addListener(MessageSendDto.class, (MessageSendDto, sendDto) -> {
            logger.info("收到广播消息:{}", JSON.toJSONString(sendDto));
            channelContextUtils.sendMessage(sendDto);
        });
    }

    /**
     * 业务层调用：把消息发布到Redis频道，所有节点都会收到并推送
     */
    public void sendMessage(MessageSendDto sendDto) {
        RTopic rTopic = redissonClient.getTopic(MESSAGE_TOPIC);
        rTopic.publish(sendDto);
    }
}
```

### 7.5 第 4 步：业务层切换调用入口

**单机版**直接调 `channelContextUtils.sendMessage()`；**集群版**改成注入 `MessageHandler`，一行换成一行即可（本项目业务层如 `UserContactServiceImpl`、`GroupInfoServiceImpl` 都是这么调的）：

```java
@Resource
private MessageHandler messageHandler;

// 原来是 channelContextUtils.sendMessage(messageSendDto);
messageHandler.sendMessage(messageSendDto);   // ← 改成走Redis广播
```

不用改 Controller、不用改前端、不用改 Netty 启动类——`ChannelContextUtils` 原封不动，它依然是"最终执行推送"的那一层。

### 7.6 集群部署要点

| 事项 | 说明 |
|------|------|
| 每个节点配置 | 各节点的 `ws.port` 可以一样（不同机器），Redis 必须是**同一个** |
| 消息对象 | 广播的对象 `MessageSendDto` 必须可序列化（Redisson 默认用 Jackson/json 编解码），字段保持简单 |
| 频道名 | 所有节点 `MESSAGE_TOPIC` 必须一致，否则收不到彼此的广播 |
| 心跳/在线状态 | 本项目把心跳时间、联系人列表放 Redis（`RedisComponet`），各节点共享，天然支持集群 |
| 收到自己的广播 | 本节点也会收到自己发的消息，`channelContextUtils` 找到本机 Channel 就推、找不到就忽略，不会重复推送（一个用户只连一个节点） |
| 性能 | RTopic 走 Redis 原生 pub/sub，毫秒级；消息量大时可按业务拆多个 topic 或改用 Redis Stream |

---

## 八、第 6 步：Spring Boot 启动后拉起 Netty

文件：`InitRun.java`（实现 `ApplicationRunner`，在 Spring 容器启动完成后执行）

**关键点：`new Thread(nettyWebSocketStarter).start()`** —— Netty 的 `bind().sync()` 是阻塞的，必须开新线程跑，否则会卡死 Spring Boot 启动。

```java
package com.easychat;

import com.easychat.websocket.netty.NettyWebSocketStarter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;

@Component("initRun")
public class InitRun implements ApplicationRunner {

    private static final Logger logger = LoggerFactory.getLogger(InitRun.class);

    @Resource
    private NettyWebSocketStarter nettyWebSocketStarter;

    @Override
    public void run(ApplicationArguments args) {
        try {
            new Thread(nettyWebSocketStarter).start();
            logger.error("服务启动成功，可以开始愉快的开发了");
        } catch (Exception e) {
            logger.error("服务启动失败", e);
        }
    }
}
```

---

## 九、测试验证

项目跑起来后，控制台出现：

```
Netty服务端启动成功,端口:5051
服务启动成功，可以开始愉快的开发了
```

浏览器控制台（F12）连接测试：

```javascript
const ws = new WebSocket("ws://localhost:5051/ws?token=你的token");
ws.onopen = () => console.log("连接成功");
ws.onmessage = (e) => console.log("收到消息：", e.data);
ws.onclose = () => console.log("连接关闭");
```

或用 [Postman / 在线 WebSocket 测试工具] 连接 `ws://localhost:5051/ws?token=xxx`。

---

## 十、常见坑位总结

| 问题 | 原因与解法 |
|------|-----------|
| Spring Boot 启动卡住不动 | Netty `sync()` 是阻塞的，必须像本项目一样 `new Thread(...).start()` 单独跑 |
| `Handler is not a @Sharable` | Handler 是单例 `@Component` 且被多个连接共用时，类上必须加 `@ChannelHandler.Sharable` |
| 收不到 WebSocket 消息 | pipeline 里忘了加 `HttpServerCodec` + `HttpObjectAggregator` + `WebSocketServerProtocolHandler`，或路径 `/ws` 和前端连的对不上 |
| 业务 Bean 注入为 null | Handler 必须用 `@Component` 交给 Spring 管理，不要自己 `new` |
| 60 秒后连接被踢 | 这不是 bug，是 `IdleStateHandler` + 心跳处理器在踢掉不发消息的连接；前端需定时发心跳（本项目里客户端任意发消息都算心跳） |
| 端口冲突 | `ws.port`（Netty）和 `server.port`（Spring Boot）要用不同端口，如 5051 和 5050 |

---

## 十一、文件清单（一图速览）

```
src/main/java/com/easychat/
├── InitRun.java                          # ApplicationRunner，启动后拉起Netty
├── entity/config/AppConfig.java          # 读取 ws.port 配置
├── redis/
│   └── RedisConfig.java                  # 创建RedissonClient（集群广播用，见第七章）
└── websocket/
    ├── netty/
    │   ├── NettyWebSocketStarter.java    # Netty服务端启动类（boss/worker/pipeline）
    │   ├── HandlerWebSocket.java         # 业务处理器：握手校验、收消息、断开清理
    │   └── HandlerHeartBeat.java         # 心跳处理器：踢掉超时连接
    ├── MessageHandler.java               # Redisson RTopic广播：集群部署的消息入口
    └── ChannelContextUtils.java          # 连接管理 + 最终执行推送
```

照着以上 6 步从上到下复制，10 分钟即可让一个 Spring Boot 项目具备 WebSocket 实时推送能力；需要集群部署时，再按第七章加上 MessageHandler + Redisson 即可平滑扩展。
