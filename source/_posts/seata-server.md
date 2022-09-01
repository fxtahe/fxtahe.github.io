---
title: seata-server
date: 2020-04-25 11:32:50
categories: 
- 中间件
---
> 版本:1.2.0

Seata主要包括三大组件:TC、TM和RM。TC（Transaction Coordinator）主要负责全局事务的提交和回滚，是seata的关键组件。对可用性及性能都有着较高的要求。

seata TC实现源码Server的各个包：

- coordinator:协调器核心模块
- event:事件管理模块
- lock:资源锁模块
- metrics: metrics指标模块
- session:session通信模块
- storage:事务信息存储模块，file和DB
- store:存储配置模块
- transaction:事务模块
<!--more-->
### Server启动流程

server启动的入口是`Server.main`方法

```java
    /**
     * The entry point of application.
     *
     * @param args the input arguments
     * @throws IOException the io exception
     */
    public static void main(String[] args) throws IOException {
        //初始化参数解析器
        ParameterParser parameterParser = new ParameterParser(args);

        //初始化度量指标metrics
        MetricsManager.get().init();
        //设置日志存储模式 file/Db
        System.setProperty(ConfigurationKeys.STORE_MODE, parameterParser.getStoreMode());
        //创建nettyserver
        RpcServer rpcServer = new RpcServer(WORKING_THREADS);
        //server port
        rpcServer.setListenPort(parameterParser.getPort());
        UUIDGenerator.init(parameterParser.getServerNode());
        //sessionholder 初始化
        SessionHolder.init(parameterParser.getStoreMode());
        //创建默认协调器
        DefaultCoordinator coordinator = new DefaultCoordinator(rpcServer);
        coordinator.init();
        rpcServer.setHandler(coordinator);
        // 注册关闭钩子
        ShutdownHook.getInstance().addDisposable(coordinator);
        ShutdownHook.getInstance().addDisposable(rpcServer);

        //127.0.0.1 and 0.0.0.0 are not valid here.
        if (NetUtil.isValidIp(parameterParser.getHost(), false)) {
            XID.setIpAddress(parameterParser.getHost());
        } else {
            XID.setIpAddress(NetUtil.getLocalIp());
        }
        XID.setPort(rpcServer.getListenPort());

        try {
            //启动
            rpcServer.init();
        } catch (Throwable e) {
            LOGGER.error("rpcServer init error:{}", e.getMessage(), e);
            System.exit(-1);
        }

        System.exit(0);
    }
```

主要流程：

- 1.解析启动参数
- 2.初始化度量
- 3.初始化netty server
- 4.初始化sessionholder
- 5.初始化默认协调器
- 6.启动netty server

#### 参数解析

> ParameterParser parameterParser = new ParameterParser(args);

```java
    public ParameterParser(String[] args) {
        this.init(args);
    }

    private void init(String[] args) {
        try {
            boolean inContainer = this.isRunningInContainer();
            //判断是否使用了容器Kubernetes/docker
            if (inContainer) {
                if (LOGGER.isInfoEnabled()) {
                    LOGGER.info("The server is running in container.");
                }
                //多配置环境
                this.seataEnv = StringUtils.trimToNull(System.getenv(ENV_SYSTEM_KEY));
                //host地址
                this.host = StringUtils.trimToNull(System.getenv(ENV_SEATA_IP_KEY));
                //server节点id
                this.serverNode = NumberUtils.toInt(System.getenv(ENV_SERVER_NODE_KEY), SERVER_DEFAULT_NODE);
                //端口号
                this.port = NumberUtils.toInt(System.getenv(ENV_SEATA_PORT_KEY), SERVER_DEFAULT_PORT);
                //存储模式
                this.storeMode = StringUtils.trimToNull(System.getenv(ENV_STORE_MODE_KEY));
            } else {
                //JCommander 解析参数
                JCommander jCommander = JCommander.newBuilder().addObject(this).build();
                jCommander.parse(args);
                if (help) {
                    jCommander.setProgramName(PROGRAM_NAME);
                    jCommander.usage();
                    System.exit(0);
                }
            }
            if (StringUtils.isNotBlank(seataEnv)) {
                System.setProperty(ENV_PROPERTY_KEY, seataEnv);
            }
            if (StringUtils.isBlank(storeMode)) {
                storeMode = ConfigurationFactory.getInstance().getConfig(ConfigurationKeys.STORE_MODE,
                    SERVER_DEFAULT_STORE_MODE);
            }
        } catch (ParameterException e) {
            printError(e);
        }

    }
```

主要解析参数中的存储模式、环境配置，ip端口。

#### 初始化server

>RpcServer rpcServer = new RpcServer(WORKING_THREADS);

构造器初始化父类构造器

```java
    public AbstractRpcRemotingServer(final ThreadPoolExecutor messageExecutor, NettyServerConfig nettyServerConfig) {
        super(messageExecutor);
        //初始化RpcServer
        serverBootstrap = new RpcServerBootstrap(nettyServerConfig);
    }
```

根据配置判断初始化EventLoopGroup

```java
    public RpcServerBootstrap(NettyServerConfig nettyServerConfig) {

        this.nettyServerConfig = nettyServerConfig;
        //根据配置判断初始化根据配置判断初始化EventLoopGroup
        if (NettyServerConfig.enableEpoll()) {
            this.eventLoopGroupBoss = new EpollEventLoopGroup(nettyServerConfig.getBossThreadSize(),
                new NamedThreadFactory(nettyServerConfig.getBossThreadPrefix(), nettyServerConfig.getBossThreadSize()));
            this.eventLoopGroupWorker = new EpollEventLoopGroup(nettyServerConfig.getServerWorkerThreads(),
                new NamedThreadFactory(nettyServerConfig.getWorkerThreadPrefix(),
                    nettyServerConfig.getServerWorkerThreads()));
        } else {
            this.eventLoopGroupBoss = new NioEventLoopGroup(nettyServerConfig.getBossThreadSize(),
                new NamedThreadFactory(nettyServerConfig.getBossThreadPrefix(), nettyServerConfig.getBossThreadSize()));
            this.eventLoopGroupWorker = new NioEventLoopGroup(nettyServerConfig.getServerWorkerThreads(),
                new NamedThreadFactory(nettyServerConfig.getWorkerThreadPrefix(),
                    nettyServerConfig.getServerWorkerThreads()));
        }

        // 构造设置端口，防止端口为空
        setListenPort(nettyServerConfig.getDefaultListenPort());
    }
```

#### SessionHolder初始化

> SessionHolder.init(parameterParser.getStoreMode());

根据持久化配置file/db,去初始化sessionManager 进行session管理和持久化，主要包括下面四种

- `ROOT_SESSION_MANAGER` root session manager
- `ASYNC_COMMITTING_SESSION_MANAGER`异步提交session manager
- `RETRY_COMMITTING_SESSION_MANAGER`重试提交session manager
- `RETRY_ROLLBACKING_SESSION_MANAGER`重试回滚session manager

```java
    public static void init(String mode) throws IOException {
        if (StringUtils.isBlank(mode)) {
            mode = CONFIG.getConfig(ConfigurationKeys.STORE_MODE);
        }
        StoreMode storeMode = StoreMode.get(mode);
        //DB 方式
        if (StoreMode.DB.equals(storeMode)) {
            //初始化四种sessionmanager
            ...
        // File 方式
        } else if (StoreMode.FILE.equals(storeMode)) {
            //初始化四种sessionmanager
            ...
        } else {
            throw new IllegalArgumentException("unknown store mode:" + mode);
        }
        //reload是否需要进行回滚或提交
        //处理未完成的事务
        reload();
    }
```

#### TC协调器初始化

> DefaultCoordinator coordinator = new DefaultCoordinator(rpcServer);  
  coordinator.init();

创建默认的TC协调器，并将server与其组合，主要看`init`方法

```java
    public void init() {
        retryRollbacking.scheduleAtFixedRate(() -> {
            try {
                handleRetryRollbacking();
            } catch (Exception e) {
                LOGGER.info("Exception retry rollbacking ... ", e);
            }
        }, 0, ROLLBACKING_RETRY_PERIOD, TimeUnit.MILLISECONDS);

        retryCommitting.scheduleAtFixedRate(() -> {
            try {
                handleRetryCommitting();
            } catch (Exception e) {
                LOGGER.info("Exception retry committing ... ", e);
            }
        }, 0, COMMITTING_RETRY_PERIOD, TimeUnit.MILLISECONDS);

        asyncCommitting.scheduleAtFixedRate(() -> {
            try {
                handleAsyncCommitting();
            } catch (Exception e) {
                LOGGER.info("Exception async committing ... ", e);
            }
        }, 0, ASYNC_COMMITTING_RETRY_PERIOD, TimeUnit.MILLISECONDS);

        timeoutCheck.scheduleAtFixedRate(() -> {
            try {
                timeoutCheck();
            } catch (Exception e) {
                LOGGER.info("Exception timeout checking ... ", e);
            }
        }, 0, TIMEOUT_RETRY_PERIOD, TimeUnit.MILLISECONDS);

        undoLogDelete.scheduleAtFixedRate(() -> {
            try {
                undoLogDelete();
            } catch (Exception e) {
                LOGGER.info("Exception undoLog deleting ... ", e);
            }
        }, UNDO_LOG_DELAY_DELETE_PERIOD, UNDO_LOG_DELETE_PERIOD, TimeUnit.MILLISECONDS);
    }
```

`init`方法主要是添加定时任务，处理提交回滚删除日志等操作

#### 启动server

> rpcServer.init();

```java
    public void init() {
        //实例化默认的服务消息监听器，包括事务消息、RM注册消息、TM注册消息、检查消息
        DefaultServerMessageListenerImpl defaultServerMessageListenerImpl =
            new DefaultServerMessageListenerImpl(getTransactionMessageHandler());
        //初始化日志处理线程池
        defaultServerMessageListenerImpl.init();
        defaultServerMessageListenerImpl.setServerMessageSender(this);
        super.setServerMessageListener(defaultServerMessageListenerImpl);
        //添加channelhandler
        super.setChannelHandlers(new ServerHandler());
        //核心方法
        super.init();
    }
```

进入`init`方法,`AbstractRpcRemotingServer.init()`

```java
    public void init() {
        //启动超时消息定时任务
        super.init();
        //启动服务
        serverBootstrap.start();
    }
```

跟入`serverBootstrap.start();`,这里就是服务的初始化

```java
    public void start() {
        //设置channel参数
        this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupWorker)
            .channel(nettyServerConfig.SERVER_CHANNEL_CLAZZ)
            .option(ChannelOption.SO_BACKLOG, nettyServerConfig.getSoBackLogSize())
            .option(ChannelOption.SO_REUSEADDR, true)
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            .childOption(ChannelOption.TCP_NODELAY, true)
            .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSendBufSize())
            .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketResvBufSize())
            .childOption(ChannelOption.WRITE_BUFFER_WATER_MARK,
                new WriteBufferWaterMark(nettyServerConfig.getWriteBufferLowWaterMark(),
                    nettyServerConfig.getWriteBufferHighWaterMark()))
            .localAddress(new InetSocketAddress(listenPort))
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) {
                    ch.pipeline().addLast(new IdleStateHandler(nettyServerConfig.getChannelMaxReadIdleSeconds(), 0, 0))
                        .addLast(new ProtocolV1Decoder())
                        .addLast(new ProtocolV1Encoder());
                    if (null != channelHandlers) {
                        addChannelPipelineLast(ch, channelHandlers);
                    }

                }
            });

        try {
            ChannelFuture future = this.serverBootstrap.bind(listenPort).sync();
            LOGGER.info("Server started ... ");
            //高可用注册
            RegistryFactory.getInstance().register(new InetSocketAddress(XID.getIpAddress(), XID.getPort()));
            initialized.set(true);
            future.channel().closeFuture().sync();
        } catch (Exception exx) {
            throw new RuntimeException(exx);
        }

    }
```

主要是netty参数的配置并启动，到这里也就完成了SeataServer的初始化及启动。通过seata-server的源码看出，其内部使用netty作为服务器，并且用到大量线程池和定时任务去提高性能。
