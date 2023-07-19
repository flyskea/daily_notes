## metadata 创建

### 🌲 使用New()：

```go
md := metadata.New(map[string]string{"key1": "value1","key2": "value2"})
```

### 🌲 使用Pairs()：

要注意如果有相同的 key 会自动合并

```go 
md := metadata.Pairs( "key1", "value1", "key1", "value1.2", // "key1" will have map value []string{"value1", "value1.2"} 
"key2", "value2", )
```


### 🌲 合并多个 metadata

```go 
md1 :=  metadata.Pairs("k1", "v1", "k2", "v2") 
md2 := metadata.New(map[string]string{"key1":"value1","key2":"value2"}) 
md := metadata.Join(md1, md2)
```

### 🌲 存储二进制数据

在 metadata 中，key 永远是 string 类型，但是 value 可以是 string 也可以是二进制数据。为了在 metadata 中存储二进制数据，我们仅仅需要在 key 的后面加上一个 - bin 后缀。具有 - bin 后缀的 key 所对应的 value 在创建 metadata 时会被编码（base64），收到的时候会被解码：

```go
md := metadata.Pairs( "key", "string value", "key-bin", string([]byte{96, 102}), )
```

metadata 结构本身也有一些操作方法，参考文档非常容易理解。这里不再赘述：[pkg.go.dev/google.gola…](https://link.juejin.cn?target=https%3A%2F%2Fpkg.go.dev%2Fgoogle.golang.org%2Fgrpc%40v1.44.0%2Fmetadata "https://pkg.go.dev/google.golang.org/grpc@v1.44.0/metadata")

## metadata 发送与接收

pb文件和生成出来的client与server端的接口

```protobuf
service OrderManagement { rpc getOrder(google.protobuf.StringValue) returns (Order); }
```

```go
type OrderManagementClient interface { GetOrder(ctx context.Context, in *wrapperspb.StringValue, opts ...grpc.CallOption) (*Order, error) }
```

```go
type OrderManagementServer interface { GetOrder(context.Context, *wrapperspb.StringValue) (*Order, error) mustEmbedUnimplementedOrderManagementServer() }
```


可以看到相比pb中的接口定义，生成出来的Go代码除了增加了`error`返回值，还多了`context.Context`

和错误处理类似，gRPC中的`context.Context` 也符合Go语言的使用习惯：通常情况下我们在函数首个参数放置`context.Context`用来传递一次RPC中有关的上下文，借助`context.WithValue()`或`ctx.Value()`往`context`添加变量或读取变量

`metadata`就是gRPC中可以传递的上下文信息之一，所以`metadata`的使用方式就是：`metadata`记录到`context`，从`context`读取`metadata`

![](picture/fdc272a40357b12808d1084caff23936_MD5.webp)

## Clinet发送Server接收

`client`发送`metadata`，那就是把`metadata`存储到`contex.Context`

`server`接收`metadata`，就是从`contex.Context`中读取`Metadata`

### Clinet 发送 Metadata

把`Metadata`放到`contex.Context`，有几种方式

#### 🌲 使用`NewOutgoingContext`

将新创建的`metadata`添加到`context`中，这样会 **覆盖** 掉原来已有的`metadata`

```go
// 将metadata添加到context中，获取新的context 
md := metadata.Pairs("k1", "v1", "k1", "v2", "k2", "v3") 
ctx := metadata.NewOutgoingContext(context.Background(), md) 
// unary RPC 
response, err := client.SomeRPC(ctx, someRequest) 
// streaming RPC 
stream, err := client.SomeStreamingRPC(ctx)
````

#### 🌲 使用`AppendToOutgoingContext`

可以直接将 key-value 对添加到已有的`context`中

- 如果`context`中没有`metadata`，那么就会 **创建** 一个
    
- 如果已有`metadata`，那么就将数据 **添加** 到原来的`metadata`

```go
// 如果对应的 context 没有 metadata，那么就会创建一个
ctx := metadata.AppendToOutgoingContext(ctx, "k1", "v1", "k1", "v2", "k2", "v3")

// 如果已有 metadata 了，那么就将数据添加到原来的 metadata  (例如在拦截器中)
ctx := metadata.AppendToOutgoingContext(ctx, "k3", "v4")

// 普通RPC（unary RPC）
response, err := client.SomeRPC(ctx, someRequest)

// 流式RPC（streaming RPC）
stream, err := client.SomeStreamingRPC(ctx)
```

### Server 接收 Metedata

普通RPC与流式RPC的区别不大，都是从`contex.Context`中读取`metadata`

#### 🌲 使用`FromIncomingContext`

**普通RPC（unary RPC）**

```go
//Unary Call
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    // do something with metadata
}
```

**流式RPC（streaming RPC）**

```go
//Streaming Call
func (s *server) SomeStreamingRPC(stream pb.Service_SomeStreamingRPCServer) error {
    md, ok := metadata.FromIncomingContext(stream.Context()) // get context from stream
    // do something with metadata
}
```

## Server发送Clinet接收

服务端发送的`metadata`被分成了`header`和 `trailer`两者，因而客户端也可以读取两者

### Server 发送 Metadata

对于**普通RPC（unary RPC）**server可以使用grpc包中提供的函数向client发送 `header` 和`trailer`

- `grpc.SendHeader()`
- `grpc.SetHeader()`
- `grpc.SetTrailer()`

对于**流式RPC（streaming RPC）server可以使用[ServerStream](https://link.juejin.cn?target=https%3A%2F%2Fgodoc.org%2Fgoogle.golang.org%2Fgrpc%23ServerStream "https://godoc.org/google.golang.org/grpc#ServerStream")接口中定义的函数向client发送`header`和 `trailer`

- `ServerStream.SendHeader()`
- `ServerStream.SetHeader()`
- `ServerStream.SetTrailer()`

#### 🌲 普通RPC（unary RPC）

使用 `grpc.SendHeader()` 和 `grpc.SetTrailer()` 方法 ，这两个函数将`context.Context`作为第一个参数

```go
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
  // 创建并发送header
  header := metadata.Pairs("header-key", "val")
  grpc.SendHeader(ctx, header)
  
  // 创建并发送trailer
  trailer := metadata.Pairs("trailer-key", "val")
  grpc.SetTrailer(ctx, trailer)
}
```

如果不想立即发送`header`，也可以使用`grpc.SetHeader()`。`grpc.SetHeader()`可以被多次调用，在如下时机会把多个`metadata`合并发送出去

- 调用`grpc.SendHeader()`
- 第一个响应被发送时
- RPC结束时（包含成功或失败）

```go
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
  // 创建header，在适当时机会被发送
  header := metadata.Pairs("header-key1", "val1")
  grpc.SetHeader(ctx, header)
    
  // 创建header，在适当时机会被发送
  header := metadata.Pairs("header-key2", "val2")
  grpc.SetHeader(ctx, header)
  
  // 创建并发送trailer
  trailer := metadata.Pairs("trailer-key", "val")
  grpc.SetTrailer(ctx, trailer)
}
```

#### 🌲 **流式RPC（streaming RPC）**

使用 `ServerStream.SendHeader()` 和 `ServerStream.SetTrailer()` 方法

```go
func (s *server) SomeStreamingRPC(stream pb.Service_SomeStreamingRPCServer) error {
  // create and send header
  header := metadata.Pairs("header-key", "val")
  stream.SendHeader(header)
  
  // create and set trailer
  trailer := metadata.Pairs("trailer-key", "val")
  stream.SetTrailer(trailer)
}
```

如果不想立即发送`header`，也可以使用`ServerStream.SetHeader()`。`ServerStream.SetHeader()`可以被多次调用，在如下时机会把多个`metadata`合并发送出去

- 调用`ServerStream.SendHeader()`
- 第一个响应被发送时
- RPC结束时（包含成功或失败）

```go
func (s *server) SomeStreamingRPC(stream pb.Service_SomeStreamingRPCServer) error {
  // create and send header
  header := metadata.Pairs("header-key", "val")
  stream.SetHeader(header)
  
  // create and set trailer
  trailer := metadata.Pairs("trailer-key", "val")
  stream.SetTrailer(trailer)
}
```

### Client 接收 Metadata

#### 🌲 **普通RPC（unary RPC）**

**普通RPC（unary RPC）**使用`grpc.Header()`和`grpc.Trailer()`方法来接收 Metadata

go



`// RPC using the context with new metadata. var header, trailer metadata.MD // Add Order order := pb.Order{Id: "101", Items: []string{"iPhone XS", "Mac Book Pro"}, Destination: "San Jose, CA", Price: 2300.00} res, err := client.AddOrder(ctx, &order, grpc.Header(&header), grpc.Trailer(&trailer)) if err != nil {   panic(err) }`

#### 🌲 **流式RPC（streaming RPC）**

**流式RPC（streaming RPC）**通过调用返回的 `ClientStream`接口的`Header()`和 `Trailer()`方法接收 `metadata`

```go
// RPC using the context with new metadata.
var header, trailer metadata.MD

// Add Order
order := pb.Order{Id: "101", Items: []string{"iPhone XS", "Mac Book Pro"}, Destination: "San Jose, CA", Price: 2300.00}
res, err := client.AddOrder(ctx, &order, grpc.Header(&header), grpc.Trailer(&trailer))
if err != nil {
  panic(err)
}
```

### `Header`和`Trailer`区别

根本区别：发送的时机不同！

✨ `headers`会在下面三种场景下被发送

- `SendHeader()` 被调用时（包含`grpc.SendHeader`和`stream.SendHeader`)
- 第一个响应被发送时
- RPC结束时（包含成功或失败）

✨ `trailer`会在rpc返回的时候，即这个请求结束的时候被发送

差异在流式RPC（streaming RPC）中比较明显：

因为`trailer`是在服务端发送完请求之后才发送的，所以client获取`trailer`的时候需要在`stream.CloseAndRecv`或者`stream.Recv` 返回非nil错误 (包含 io.EOF)之后

如果`stream.CloseAndRecv`之前调用`stream.Trailer()`获取的是空

```go
stream, err := client.SomeStreamingRPC(ctx)

// retrieve header
header, err := stream.Header()

// retrieve trailer 
// `trailer`会在rpc返回的时候，即这个请求结束的时候被发送
// 因此此时调用`stream.Trailer()`获取的是空
trailer := stream.Trailer()

stream.CloseAndRecv()

// retrieve trailer 
// `trailer`会在rpc返回的时候，即这个请求结束的时候被发送
// 因此此时调用`stream.Trailer()`才可以获取到值
trailer := stream.Trailer()
```

## 使用场景

既然我们把`metadata`类比成`HTTP Header`，那么`metadata`的使用场景也可以借鉴`HTTP`的`Header`。如传递用户`token`进行用户认证，传递`trace`进行链路追踪等

### 拦截器中的metadata

在拦截器中，我们不但可以获取或修改**接收**到的`metadata`，甚至还可以截取并修改要**发送**出去的`metadata`

还记得拦截器如何实现么？如果已经忘了快快回顾一下吧：

🌰 举个例子：

我们在客户端拦截器中从要发送给服务端的`metadata`中读取一个时间戳字段，如果没有则补充这个时间戳字段

注意这里用到了一个上文没有提到的`FromOutgoingContext(ctx)`函数

```go
func orderUnaryClientInterceptor(ctx context.Context, method string, req, reply interface{},
	cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {

	var s string

	// 获取要发送给服务端的`metadata`
	md, ok := metadata.FromOutgoingContext(ctx)
	if ok && len(md.Get("time")) > 0 {
		s = md.Get("time")[0]
	} else {
        // 如果没有则补充这个时间戳字段
		s = "inter" + strconv.FormatInt(time.Now().UnixNano(), 10)
		ctx = metadata.AppendToOutgoingContext(ctx, "time", s)
	}

	log.Printf("call timestamp: %s", s)

	// Invoking the remote method
	err := invoker(ctx, method, req, reply, cc, opts...)

	return err
}

func main() {
	conn, err := grpc.Dial("127.0.0.1:8009",
		grpc.WithInsecure(),
		grpc.WithChainUnaryInterceptor(
			orderUnaryClientInterceptor,
		),
	)
	if err != nil {
		panic(err)
	}
    
    c := pb.NewOrderManagementClient(conn)

	ctx = metadata.AppendToOutgoingContext(context.Background(), "time",
		"raw"+strconv.FormatInt(time.Now().UnixNano(), 10))

	// RPC using the context with new metadata.
	var header, trailer metadata.MD

	// Add Order
	order := pb.Order{
		Id:          "101",
		Items:       []string{"iPhone XS", "Mac Book Pro"},
		Destination: "San Jose, CA",
		Price:       2300.00,
	}
	res, err := c.AddOrder(ctx, &order)
	if err != nil {
		panic(err)
	}
}
```

以上的思路在server同样适用。基于以上原理我们可以实现链路追踪、用户认证等功能

### 错误信息

还记得[错误处理](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzIyMTI4OTY3Mw%3D%3D%26mid%3D2247484115%26idx%3D1%26sn%3D681de56485e73f4b2ee330901e3cca68%26chksm%3De83e4f75df49c66361751531b35aed0dbe59b1c173fce790034361dd39d58a6d91d09209d8e4%23rd "https://mp.weixin.qq.com/s?__biz=MzIyMTI4OTY3Mw==&mid=2247484115&idx=1&sn=681de56485e73f4b2ee330901e3cca68&chksm=e83e4f75df49c66361751531b35aed0dbe59b1c173fce790034361dd39d58a6d91d09209d8e4#rd")一文中留下的问题么：gRPC 中如何传递错误消息`Status`的呢？没错！也是使用的`metadata`或者说`http2.0` 的`header`。`Status`的三种信息分别使用了三个`header`头

- `Grpc-Status`: 传递`Status`的`code`
- `Grpc-Message`: 传递`Status`的`message`
- `Grpc-Status-Details-Bin`: 传递`Status`的`details`

```go
func (ht *serverHandlerTransport) WriteStatus(s *Stream, st *status.Status) error {
	// ...
		h := ht.rw.Header()
		h.Set("Grpc-Status", fmt.Sprintf("%d", st.Code()))
		if m := st.Message(); m != "" {
			h.Set("Grpc-Message", encodeGrpcMessage(m))
		}

		if p := st.Proto(); p != nil && len(p.Details) > 0 {
			stBytes, err := proto.Marshal(p)
			if err != nil {
				// TODO: return error instead, when callers are able to handle it.
				panic(err)
			}

			h.Set("Grpc-Status-Details-Bin", encodeBinHeader(stBytes))
		}
    // ...
}
```