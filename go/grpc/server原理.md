## 1. 服务端

### 1.1 核心数据结构

![](picture/65573d7a47c0564f7ebca31f9da161a9_MD5.webp)

  

在 grpc 服务端领域，自上而下有着三个层次分明的结构：server->service->method

- 最高级别是 server，是对整个 grpc 服务端的抽象
- 一个 server 下可以注册挂载多个业务服务 service
- 一个 service 下存在多个业务处理方法 method

**（1）server**

```go
type Server struct {
    // 配置项
    opts serverOptions
    // 互斥锁保证并发安全
    mu  sync.Mutex 
    // tcp 端口监听器池
    lis map[net.Listener]bool
    // ...
    // 连接池
    conns    map[string]map[transport.ServerTransport]bool
    serve    bool
    cv       *sync.Cond          
    // 业务服务映射管理  
    services map[string]*serviceInfo // service name -> service info
    // ...
    serveWG            sync.WaitGroup // counts active Serve goroutines for GracefulStop
    // ...
}
```

Server 类是对 grpc 服务端的代码实现，其中通过一个名为 services 的 map，记录了由服务名到具体业务服务模块的映射关系.

**（2）serviceInfo**

```go
type serviceInfo struct {
    // 业务服务类
    serviceImpl interface{
    // 业务方法映射管理  
    methods     map[string]*MethodDesc
    // ...
}
```

serviceInfo 是某一个具体的业务服务模块，其中通过一个名为 methods 的 map 记录了由方法名到具体方法的映射关系.

**（3）MethodDesc**

```go
type MethodDesc struct {
    MethodName string
    Handler    methodHandler
}
```

MethodDesc 是对方法的封装，其中的字段 Handler 是真正的业务处理方法.

**（4）methodHandler**

```go
type methodHandler func(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor UnaryServerInterceptor) (interface{}, error)
```

methodsHandler 是业务处理方法的类型，其中几个关键入参的含义分别是：

- srv：业务处理方法从属的业务服务模块
- dec：进行入参 req 反序列化的闭包函数
- interceptor：业务处理方法外部包裹的拦截器方法

### 1.2 创建 server

```go
func NewServer(opt ...ServerOption) *Server {
    opts := defaultServerOptions
    for _, o := range extraServerOptions {
        o.apply(&opts)
    }
    for _, o := range opt {
        o.apply(&opts)
    }
    s := &Server{
        lis:      make(map[net.Listener]bool),
        opts:     opts,
        conns:    make(map[string]map[transport.ServerTransport]bool),
        services: make(map[string]*serviceInfo),
        quit:     grpcsync.NewEvent(),
        done:     grpcsync.NewEvent(),
        czData:   new(channelzData),
    }
    chainUnaryServerInterceptors(s)
    //...
    s.cv = sync.NewCond(&s.mu)
    // ...   
    return s
}
```

grpc.NewServer 方法中会创建 server 实例，并调用 chainUnaryServerInterceptors 方法，将一系列拦截器 interceptor 成链，并注入到 ServerOption 当中. 

```go
func chainUnaryServerInterceptors(s *Server) {
    interceptors := s.opts.chainUnaryInts
    if s.opts.unaryInt != nil {
        interceptors = append([]UnaryServerInterceptor{s.opts.unaryInt}, s.opts.chainUnaryInts...)
    }


    var chainedInt UnaryServerInterceptor
    if len(interceptors) == 0 {
        chainedInt = nil
    } else if len(interceptors) == 1 {
        chainedInt = interceptors[0]
    } else {
        chainedInt = chainUnaryInterceptors(interceptors)
    }
    
    s.opts.unaryInt = chainedInt
}

func chainUnaryInterceptors(interceptors []UnaryServerInterceptor) UnaryServerInterceptor {

    return func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (interface{}, error) {
        return interceptors[0](ctx, req, info, getChainUnaryHandler(interceptors, 0, info, handler))
    }
}

func getChainUnaryHandler(interceptors []UnaryServerInterceptor, curr int, info *UnaryServerInfo, finalHandler UnaryHandler) UnaryHandler {

    if curr == len(interceptors)-1 {
        return finalHandler
    }

    return func(ctx context.Context, req interface{}) (interface{}, error) {
        return interceptors[curr+1](ctx, req, info, getChainUnaryHandler(interceptors, curr+1, info, finalHandler))
    }
}
```

### 1.3 注册 service

![](picture/a83d112850aacd38bd475d8f98070a1a_MD5.png)

  
  创建好 grpc server 后，接下来通过使用桩代码中预生成好的 RegisterXXXServer 方法，业务处理服务 service 模块注入到 server 当中.

```go
func RegisterHelloServiceServer(s grpc.ServiceRegistrar, srv HelloServiceServer) {
    s.RegisterService(&HelloService_ServiceDesc, srv)
}
```

```go
func (s *Server) RegisterService(sd *ServiceDesc, ss interface{}) {
    // ...
    s.register(sd, ss)
}
```

```go
func (s *Server) register(sd *ServiceDesc, ss interface{}) {
    s.mu.Lock()
    defer s.mu.Unlock()
    // ...
    info := &serviceInfo{
        serviceImpl: ss,
        methods:     make(map[string]*MethodDesc),
        streams:     make(map[string]*StreamDesc),
        mdata:       sd.Metadata,
    }
    for i := range sd.Methods {
        d := &sd.Methods[i]
        info.methods[d.MethodName] = d
    }
    // ...
    s.services[sd.ServiceName] = info
}
```

注册过程会经历 RegisterHelloServiceServer->Server.RegisterService -> Server.register 的调用链路，把 service 的所有方法注册到 serviceInfo 的 methods map 当中，然后将 service 封装到 serviceInfo 实例中，注册到 server 的 services map 当中

### 1.4 运行 server

```go
func (s *Server) Serve(lis net.Listener) error {
    // ...
    var tempDelay time.Duration // how long to sleep on accept failure
    for {
        rawConn, err := lis.Accept()
        if err != nil {
            // ...
        }
        // ...
        s.serveWG.Add(1)
        go func() {
            s.handleRawConn(lis.Addr().String(), rawConn)
            s.serveWG.Done()
        }()
    }
}
```

![](picture/71c9e917628491565641f0eed689acb1_MD5.webp)

grpc server 运行的流程，核心是基于 for 循环实现的主动轮询模型，每轮会通过调用 net.Listener.Accept 方法，基于 IO 多路复用 epoll 方式，阻塞等待 grpc 请求的到达.

每当有新的连接到达后，服务端会开启一个 goroutine，调用对应的 Server.handleRawConn 方法对请求进行处理.

### 1.5 处理请求

```go
func (s *Server) handleRawConn(lisAddr string, rawConn net.Conn) {
    // ...
    st := s.newHTTP2Transport(rawConn)
    // ...
    go func() {
        s.serveStreams(st)
        s.removeConn(lisAddr, st)
    }()
}
```

![](picture/a78761aec3d5ac4d263adcbc33f52904_MD5.webp)

在 Server.handleRawConn 方法中，会基于原始的 net.Conn 封装生成一个 HTTP2Transport，然后开启 goroutine 调用 Server.serveStream 方法处理请求.

```go
func (s *Server) serveStreams(st transport.ServerTransport) {
    var wg sync.WaitGroup
    var roundRobinCounter uint32
    st.HandleStreams(func(stream *transport.Stream) {
        go func() {
            defer wg.Done()
            s.handleStream(st, stream, s.traceInfo(st, stream))
        }()
    }, func(ctx context.Context, method string) context.Context {
        // ...
    })
    wg.Wait()
}
```

```go
// HandleStreams receives incoming streams using the given handler. This is
// typically run in a separate goroutine.
// traceCtx attaches trace to ctx and returns the new context.
func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {
    defer close(t.readerDone)
    for {
            if err == io.EOF || err == io.ErrUnexpectedEOF {
                t.Close(err)
                return
            }
            t.Close(err)
            return
        }
        switch frame := frame.(type) {
        case *http2.MetaHeadersFrame:
            if err := t.operateHeaders(frame, handle, traceCtx); err != nil {
                t.Close(err)
                break
            }
        case *http2.DataFrame:
            t.handleData(frame)
        case *http2.RSTStreamFrame:
            t.handleRSTStream(frame)
        case *http2.SettingsFrame:
            t.handleSettings(frame)
        case *http2.PingFrame:
            t.handlePing(frame)
        case *http2.WindowUpdateFrame:
            t.handleWindowUpdate(frame)
        case *http2.GoAwayFrame:
            // TODO: Handle GoAway from the client appropriately.
        default:
            if t.logger.V(logLevel) {
                t.logger.Infof("Received unsupported frame type %T", frame)
            }
        }
    }
}
```

如果 frame 类型是 Header 则调用传入的匿名函数，调用到 handleStream。如果 frame 类型是 Data，则将数据读入 stream 自身 buf 中，供 handler 函数使用。

```go
func (s *Server) handleStream(t transport.ServerTransport, stream *transport.Stream, trInfo *traceInfo) {
    sm := stream.Method()
    // ...
    pos := strings.LastIndex(sm, "/")
    service := sm[:pos]
    method := sm[pos+1:]

    srv, knownService := s.services[service]
    if knownService {
        if md, ok := srv.methods[method]; ok {
            s.processUnaryRPC(t, stream, srv, md, trInfo)
            return
        }
        if sd, ok := srv.streams[method]; ok {
            s.processStreamingRPC(t, stream, srv, sd, trInfo)
            return
        }
    }
    // ...
}
```

```go
func (s *Server) processUnaryRPC(t transport.ServerTransport, stream *transport.Stream, info *serviceInfo, md *MethodDesc, trInfo *traceInfo) (err error) {
    // ...
    d, err := recvAndDecompress(&parser{r: stream}, stream, dc, s.opts.maxReceiveMessageSize, payInfo, decomp)
    // ...
    df := func(v interface{}) error {
        if err := s.getCodec(stream.ContentSubtype()).Unmarshal(d, v); err != nil {
           // ...
        }
        // ...
    }
    ctx := NewContextWithServerTransportStream(stream.Context(), stream)
    reply, appErr := md.Handler(info.serviceImpl, ctx, df, s.opts.unaryInt)
    // ...

    if err := s.sendResponse(t, stream, reply, cp, opts, comp); err != nil {
        // ...
    }
    // ...
}
```

![](picture/afad1f7e1206a03b4cec3c5a06faaf8a_MD5.webp)

接下来一连建立了 Server.serveStreams -> http2Server.HandleStreams -> http2Server.operateHeaders -> http2Server.handleStream -> Server.processUnaryRPC 的方法调用链：

- 在 Server.handleStream 方法中，会拆解来自客户端的请求路径 ${service}/${method}，通过"/" 前段得到 service 名称，通过 "/" 后端得到 method 名称，并分别映射到对应的业务服务和业务方法
- 在 Server.processUnaryRPC 方法中，会通过 recvAndDecompress 读取到请求内容字节流，然后通过闭包函数 df 封装好反序列请求参数的逻辑，继而调用 md.Handler 方法处理请求，最终通过 Server.sendResponse 方法将响应结果进行返回

```go
func _HelloService_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
    in := new(HelloReq)
    if err := dec(in); err != nil {
        return nil, err
    }
    if interceptor == nil {
        return srv.(HelloServiceServer).SayHello(ctx, in)
    }
    info := &grpc.UnaryServerInfo{
        Server:     srv,
        FullMethod: "/pb.HelloService/SayHello",
    }
    handler := func(ctx context.Context, req interface{}) (interface{}, error) {
        return srv.(HelloServiceServer).SayHello(ctx, req.(*HelloReq))
    }
    return interceptor(ctx, in, info, handler)
}
```

以本文介绍的 helloService 为例，客户端调用 SayHello 方法后，服务端对应的 md.Handler 正是 .proto 文件生成的位于 .grpc.pb.go 文件中的桩方法 _HelloService_SayHello_Handler.

在该桩方法内部中，包含的执行步骤如下：

- 调用闭包函数 dec，将请求内容反序列化到请求入参 in 当中
- 将业务处理方法 HelloServiceServer.SayHello 闭包封装到一个 UnaryHandler 当中
- 调用 intercetor 方法，分别执行拦截器和 handler 的处理逻辑

## 2 拦截器

有关 grpc 中拦截器 interceptor 部分的内容理解起来比较费脑，我们单开一章来展开聊聊.

### 2.1 原理介绍

拦截器的作用，是在执行核心业务方法的前后，创造出一个统一的切片，来执行所有业务方法锁共有的通用逻辑. 此外，我们还能够通过这部分通用逻辑的执行结果，来判断是否需要熔断当前的执行链路，以起到所谓的”拦截“效果.

有关 grpc 拦截器的内容，其实和 gin 框架中的 handlersChain 是异曲同工的. 在我之前分享的文章 ”解析 Gin 框架底层原理“ 的第 5 章内容中有作详细介绍，大家不妨引用对比，以此来触类旁通，加深理解.

下面我们看看 grpc 中对于一个拦截器函数的具体定义：

```go
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

其中几个入参的含义分别为：

- req：业务处理方法的请求参数
- info：当前所属的业务服务 service
- handler：真正的业务处理方法

因此一个拦截器函数的使用模式应该是：

```go
var myInterceptor1 = func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
    // 前处理校验
    if err := preLogicCheck();err != nil{
       // 前处理校验不通过，则拦截，不调用业务方法直接返回
       return nil,err 
    }
    
     // 前处理校验通过，正常调用业务方法
     resp, err = handle(ctx,req)
     if err != nil{
         return nil,err 
     }
     
      // 后置处理校验
      if err := postLogicCheck();err != nil{
         // 后置处理校验不通过，则拦截结果，包装错误返回
         return nil,err 
      }
      
      // 正常返回结果
      return resp,nil 
}
```

### 2.2 拦截器链

![](picture/cb185af475428e0e09873ebafdbacafc_MD5.webp)

  

```go
func chainUnaryInterceptors(interceptors []UnaryServerInterceptor) UnaryServerInterceptor {
    return func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (interface{}, error) {
        return interceptors[0](ctx, req, info, getChainUnaryHandler(interceptors, 0, info, handler))
    }
}


func getChainUnaryHandler(interceptors []UnaryServerInterceptor, curr int, info *UnaryServerInfo, finalHandler UnaryHandler) UnaryHandler {
    if curr == len(interceptors)-1 {
        return finalHandler
    }
    return func(ctx context.Context, req interface{}) (interface{}, error) {
        return interceptors[curr+1](ctx, req, info, getChainUnaryHandler(interceptors, curr+1, info, finalHandler))
    }
}
```

首先，chainUnaryInterceptors 方法会将一系列拦截器 interceptor 成链，并返回首枚interceptor 供 ServerOption 接收设置.

其中，拦截器成链的关键在于 getChainUnaryHandler 方法中，其中会闭包调用拦截器数组的首枚拦截器函数，接下来依次用下一枚拦截器对业务方法 handler 进行包裹，封装成一个新的 ”handler“ 供当前拦截器使用.

![](picture/ede457e5f82204a3cb36bb549e8cca1a_MD5.webp)
