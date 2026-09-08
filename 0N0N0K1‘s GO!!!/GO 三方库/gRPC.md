# 1. 作用

[[RPC]]的一个实现框架（==基于HTTP/2 + protoBuf==）

客户端应用程序可以直接调用位于另一台机器上的服务器应用程序上的方法，就像调用本地对象一样，这使得创建分布式应用程序和服务变得更加容易。
![[Pasted image 20260908163538.png]]
grpc官方网站： https://grpc.org.cn/docs/
go grpc包文档： https://pkg.go.dev/google.golang.org/grpc
protobuf官方网站： https://protobuf.com.cn/
# 2. 导入

```sh
//安装go grpc插件
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

//导入包
go get -u google.golang.org/grpc
```
# 3. 使用示例

## 3.1创建 .proto 文件  
```protobuf 
// proto 版本  
syntax= "proto3" ;  
  
// .proto 自身的包名  
package  test;  
  
// 生成 go Stub 代码的  路径;包名  
option go_package="./pb;test";  
  
// 导入其他的 .proto 文件（可选）  
import "google/api/annotations.proto";  
  
// 定义消息结构，用作rpc调用函数的参数  
message testReq{  
  // <type> <var_name> = <unique index>  
  string name = 1;  
  uint32 num = 2;  
}  
  
message testRsp{  
  string rep=1;  
 uint32 num=2;  
}  
  
// 定义服务（RPC 接口集合）  
service test{  
  // 定义 rpc 方法 ，stream 关键词可以声明流式传输  
  // 一元调用  
  rpc onlyOnce (testReq)returns(testRsp);  
  // 客户端流式  
  rpc CliStream (stream testReq)returns (testRsp);  
  // 服务端流式  
  rpc  SerStream (testReq)returns (stream testRsp);  
  // 双向流式  
  rpc BothStream (stream testReq)returns(stream testRsp);  
}
```

## 3.2 命令行执行命令生成Stub代码
```shell
protoc -I./proto                         //生成所需 .proto 文件路径
  --go_out=./pb/test                          //--go_out 生成.proto里数据对象的.go文件的位置
  --go_opt=paths=source_relative         //输出路径相对于源.proto 文件的位置
  --go-grpc_out=./pb/test               //--go-grpc_out 生成grpc网络服务的.go文件的位置
  --go-grpc_opt=paths=source_relative 
    proto/*.proto                        //匹配 proto 目录下所有.proto文件
```


多个grpc项目建议根目录下结构如下
```
your-project/
|
├── others
|
|   
├── proto/    # 📁不同子目录下放不同功能的 .proto 文件
│   ├── user/
│   │    └── user.proto 
|   |     
│   ├── order/
│   │    └── order.proto
│   │      
│   └── common/
│        └── common.proto
│
└── pb/      # 📁不同子目录下放不同功能的生成代码
     ├── user/
     |    ├── user.pb.go        # protobuf 消息
     │    └── user_grpc.pb.go  # gRPC 客户端/服务端
     │             
     ├── order/
     │    ├── order.pb.go
     │    └── order_grpc.pb.go
     │      
     └── common/
         ├─ common.pb.go
         └─ order_grpc.pb.go
```

## 3.3 服务端

几点注意：
1. 实现服务方法结构体匿名嵌入stub的包下的 test.UnimplementedTestServer  结构体
2. 方法名、参数、返回值等格式严格按照stub中定义实现
3. .proto中非流式请求参数/响应结果在实现中不能调用stream.Send/Recv进行发送/接收
4. 实现的方法的 流式参数 不能用指针

```go
package main  
  
import (  
    "context"  
    "fmt"  
    "gRPC_Study/pb/test"    
    "google.golang.org/grpc"    
    "io"    
    "log"    
    "net"
)  
  
// 实现服务方法结构体
type testserver struct {  
    test.UnimplementedTestServer  
}  
  
// OnlyOnce 一元调用  
func (t *testserver) OnlyOnce(ctx context.Context, in *test.TestReq) (out *test.TestRsp, err error) {  
  
    return &test.TestRsp{  
       Rep: fmt.Sprintf("response:  recv:    name: %s     num: %d ", in.Name, in.Num),  
       Num: 0,  
    }, nil  
  
}  
  
// CliStream 客户端流式调用 test.Test_CliStreamServer 不能用指针  
func (t *testserver) CliStream(in test.Test_CliStreamServer) error {  
    var res string  
    for {  
       recv, err := in.Recv()  
       if err == io.EOF {  
          break  
       }  
       if err != nil {  
          return err  
       }  
       res = res + fmt.Sprintf("response:  recv:    name: %s     num: %d ", recv.Name, recv.Num)  
    }  
    err := in.SendAndClose(&test.TestRsp{  
       Rep: res,  
       Num: 0,  
    })  
    if err != nil {  
       return err  
    }  
    return nil  
}  
  
// SerStream 服务端流式调用 test.Test_SerStreamServer 不能用指针  
func (t *testserver) SerStream(req *test.TestReq, in test.Test_SerStreamServer) error {  
    var res test.TestRsp  
    for i := 0; i < 5; i++ {  
       res.Rep = fmt.Sprintf("response:  recv:    name: %s     num: %d ", req.Name, req.Num)  
       res.Num = uint32(i)  
       err := in.Send(&res)  
       if err != nil {  
          log.Println(err)  
          break  
       }  
    }  
    return nil  
}  
  
// BothStream 双端流式调用 test.Test_BothStreamServer 不能用指针  
func (t *testserver) BothStream(in test.Test_BothStreamServer) error {  
    var req test.TestReq  
    var res test.TestRsp  
    var i uint32  
    for {  
       err := in.RecvMsg(&req)  
       if err != nil {  
          log.Println(err)  
          return err  
       }  
       res.Rep = fmt.Sprintf("response:  recv:    name: %s     num: %d ", req.Name, req.Num)  
       res.Num = i  
       err = in.Send(&res)  
       if err != nil {  
          log.Println(err)  
          break  
       }  
    }  
    return nil  
}  
  
// 服务端启动代码  
func main() {  
    // 获得tcp监听器  
    ls, _ := net.Listen("tcp", "127.0.0.1:7979")  
  
    // 构建grpc server 实例  
    server := grpc.NewServer()  
    
    // 注入实现的服务端接口到grpc server中
    test.RegisterTestServer(server, &test{})  
  
    // 在监听端口上启动服务 阻塞  
    err := server.Serve(ls)  
    if err != nil {  
       return  
    }  
  
}
```
## 3.4 客户端
```go
package main  
  
import (  
    "context"  
    "gRPC_Study/pb/test"    
    "google.golang.org/grpc"    
    "google.golang.org/grpc/credentials/insecure"    
    "io"    
    "log"    
    "time"
)  
  
var client test.TestClient  
  
func main() {  
    // 获得与服务端的连接句柄  
    conn, err := grpc.NewClient("127.0.0.1:7979", grpc.WithTransportCredentials(insecure.NewCredentials()))  
    if err != nil {  
       log.Fatal(err)  
    }  
    defer conn.Close()  
  
    // 注入连接 返回客户端接口实例
    client = test.NewTestClient(conn)  
  
    // 使用客户端接口调用方法————如本地函数一样调用
    //在  client.OnlyOnce 或其他调用函数时会阻塞
    onlyOnce()  
    cliStream()  
    serStream()  
    bothStream()  
}  
  
// 一元调用  
func onlyOnce() {  
    req, err := client.OnlyOnce(context.TODO(), &test.TestReq{  
       Name: "0N0N0K1",  
       Num:  0,  
    })  
    if err != nil {  
       log.Fatal(err)  
    }  
    log.Println(req)  
}  
  
// 客户端流式调用  
func cliStream() {  
    stream, err := client.CliStream(context.TODO())  
    if err != nil {  
       log.Fatal(err)  
    }  
    for i := 0; i < 5; i++ {  
       err = stream.Send(&test.TestReq{  
          Name: "0N0N0K1",  
          Num:  uint32(i),  
       })  
       if err != nil {  
          log.Println(err)  
          break  
       }  
    }  
    recv, err := stream.CloseAndRecv()  
    if err != nil {  
       log.Fatal(err)  
    }  
    log.Println(recv)  
}  
  
// 客户端流式调用  
func serStream() {  
    stream, err := client.SerStream(context.TODO(), &test.TestReq{  
       Name: "0N0N0K1",  
       Num:  0,  
    })  
    if err != nil {  
       log.Fatal(err)  
    }  
    var recv test.TestRsp  
    for {  
       err = stream.RecvMsg(&recv)  
       if err == io.EOF {  
          return  
       }  
       if err != nil {  
          log.Println(err)  
          break  
       }  
       log.Println(recv.Rep, recv.Num)  
    }  
}  
  
// 双端流式调用  
func bothStream() {  
    stream, err := client.BothStream(context.TODO())  
    if err != nil {  
       log.Fatal(err)  
    }  
    for i := 0; i < 5; i++ {  
       err = stream.Send(&test.TestReq{  
          Name: "0N0N0K1",  
          Num:  uint32(i),  
       })  
       if err != nil {  
          log.Println(err)  
          break  
       }  
       recv, err := stream.Recv()  
       if err != nil {  
          log.Println(err)  
          break  
       }  
       log.Println(recv)  
       time.Sleep(time.Second * 3)  
    }  
    err = stream.CloseSend()  
    if err != nil {  
       log.Println(err)  
    }  
}
```
客户端得到的响应
```
//一元响应
2026/09/08 15:21:42 rep:"response:  recv:    name: 0N0N0K1     num: 0 "

// client流式响应
2026/09/08 15:21:42 rep:"response:  recv:    name: 0N0N0K1     num: 0 response:  recv:    name: 0N0N0K1     num: 1 response:  recv:    name: 0N0N0K1     num: 2 response:  recv:    name: 0N0N0K1     num: 3 response:  recv:    name: 0N0N0K1     num: 4 "

// server流式响应
2026/09/08 15:21:42 response:  recv:    name: 0N0N0K1     num: 0  0
2026/09/08 15:21:42 response:  recv:    name: 0N0N0K1     num: 0  1
2026/09/08 15:21:42 response:  recv:    name: 0N0N0K1     num: 0  2
2026/09/08 15:21:42 response:  recv:    name: 0N0N0K1     num: 0  3
2026/09/08 15:21:42 response:  recv:    name: 0N0N0K1     num: 0  4

// 双向流式响应
2026/09/08 15:21:42 rep:"response:  recv:    name: 0N0N0K1     num: 0 "
2026/09/08 15:21:45 rep:"response:  recv:    name: 0N0N0K1     num: 1 "
2026/09/08 15:21:48 rep:"response:  recv:    name: 0N0N0K1     num: 2 "
2026/09/08 15:21:51 rep:"response:  recv:    name: 0N0N0K1     num: 3 "
2026/09/08 15:21:54 rep:"response:  recv:    name: 0N0N0K1     num: 4 "
```
# 4. 生成的代码
## 4.1 rpc接口及方法的实现
```go
package test  
  
import (  
    context "context"  
    grpc "google.golang.org/grpc"  
    codes "google.golang.org/grpc/codes"  
    status "google.golang.org/grpc/status"  
)  
  
const _ = grpc.SupportPackageIsVersion9  
//方法名常量，格式:  "/{package}.{service}/{method}"
//这个字符串是 gRPC 通信的**唯一标识**，客户端和服务端通过它来匹配是哪个方法
const (  
    Test_OnlyOnce_FullMethodName   = "/test.test/onlyOnce"  
    Test_CliStream_FullMethodName  = "/test.test/CliStream"  
    Test_SerStream_FullMethodName  = "/test.test/SerStream"  
    Test_BothStream_FullMethodName = "/test.test/BothStream"  
)  
  
---------------------------------------------------------------------------------------  客户端Stub提供的接口与可调用方法

// TestClient is the client API for Test service.//  
// 定义服务（RPC 接口集合）  
type TestClient interface {  
    // 定义 rpc 方法 ，stream 关键词可以声明流式传输  
    // 一元调用  
    OnlyOnce(ctx context.Context, in *TestReq, opts ...grpc.CallOption) (*TestRsp, error)  
    // 客户端流式  
    CliStream(ctx context.Context, opts ...grpc.CallOption)(grpc.ClientStreamingClient[TestReq, TestRsp], error)  
    // 服务端流式  
    SerStream(ctx context.Context, in *TestReq, opts ...grpc.CallOption) (grpc.ServerStreamingClient[TestRsp], error)  
    // 双向流式  
    BothStream(ctx context.Context, opts ...grpc.CallOption) (grpc.BidiStreamingClient[TestReq, TestRsp], error)  
}  
---------------------------------------------------------------------------------------  客户端调用方法后具体通信细节

//实现接口的结构体对象，封装了实际的网络连接
type testClient struct {  
    cc grpc.ClientConnInterface  // 实际的网络连接
}  
// 传入连接对象获得客户端
func NewTestClient(cc grpc.ClientConnInterface) TestClient {  
    return &testClient{cc}  
}  
/*调用 OnlyOnce()
    ↓
c.cc.Invoke()  ← 实际发起网络请求，传入：方法名 /test.test/onlyOnce、请求参数 in
    ↓
接收：响应 out
    ↓
返回给调用者*/
func (c *testClient) OnlyOnce(ctx context.Context, in *TestReq, opts ...grpc.CallOption) (*TestRsp, error) {  
    cOpts := append([]grpc.CallOption{grpc.StaticMethod()}, opts...)  
    out := new(TestRsp)  
    err := c.cc.Invoke(ctx, Test_OnlyOnce_FullMethodName, in, out, cOpts...)  
    if err != nil {  
       return nil, err  
    }  
    return out, nil  
}  
  
func (c *testClient) CliStream(ctx context.Context, opts ...grpc.CallOption) (grpc.ClientStreamingClient[TestReq, TestRsp], error) {  
    cOpts := append([]grpc.CallOption{grpc.StaticMethod()}, opts...)  
    stream, err := c.cc.NewStream(ctx, &Test_ServiceDesc.Streams[0], Test_CliStream_FullMethodName, cOpts...)  
    if err != nil {  
       return nil, err  
    }  
    x := &grpc.GenericClientStream[TestReq, TestRsp]{ClientStream: stream}  
    return x, nil  
}  
  
func (c *testClient) SerStream(ctx context.Context, in *TestReq, opts ...grpc.CallOption) (grpc.ServerStreamingClient[TestRsp], error) {  
    cOpts := append([]grpc.CallOption{grpc.StaticMethod()}, opts...)  
    stream, err := c.cc.NewStream(ctx, &Test_ServiceDesc.Streams[1], Test_SerStream_FullMethodName, cOpts...)  
    if err != nil {  
       return nil, err  
    }  
    x := &grpc.GenericClientStream[TestReq, TestRsp]{ClientStream: stream}  
    if err := x.ClientStream.SendMsg(in); err != nil {  
       return nil, err  
    }  
    if err := x.ClientStream.CloseSend(); err != nil {  
       return nil, err  
    }  
    return x, nil  
}  
  
func (c *testClient) BothStream(ctx context.Context, opts ...grpc.CallOption) (grpc.BidiStreamingClient[TestReq, TestRsp], error) {  
    cOpts := append([]grpc.CallOption{grpc.StaticMethod()}, opts...)  
    stream, err := c.cc.NewStream(ctx, &Test_ServiceDesc.Streams[2], Test_BothStream_FullMethodName, cOpts...)  
    if err != nil {  
       return nil, err  
    }  
    x := &grpc.GenericClientStream[TestReq, TestRsp]{ClientStream: stream}  
    return x, nil  
}  
  

  
----------------------------------------------------------------------------------------
服务端Stub提供接口与需实现方法

// TestServer is the server API for Test service.
// All implementations must embed UnimplementedTestServer  
// 定义服务（RPC 接口集合）  
type TestServer interface {  
    // 定义 rpc 方法 ，stream 关键词可以声明流式传输  
    // 一元调用  
    OnlyOnce(context.Context, *TestReq) (*TestRsp, error)  
    // 客户端流式  
    CliStream(grpc.ClientStreamingServer[TestReq, TestRsp]) error  
    // 服务端流式  
    SerStream(*TestReq, grpc.ServerStreamingServer[TestRsp]) error  
    // 双向流式  
    BothStream(grpc.BidiStreamingServer[TestReq, TestRsp]) error  
    mustEmbedUnimplementedTestServer()  
}  
---------------------------------------------------------------------------------------  UnimplementedTestServer must be embedded 当未实现时默认返回"method xxx not implemented"
// UnimplementedTestServer must be embedded to have
// forward compatible implementations.  
//  
// NOTE: this should be embedded by value instead of pointer to avoid a nil  
// pointer dereference when methods are called.  
type UnimplementedTestServer struct{}  
  
func (UnimplementedTestServer) OnlyOnce(context.Context, *TestReq) (*TestRsp, error) {  
    return nil, status.Error(codes.Unimplemented, "method OnlyOnce not implemented")  
}  
func (UnimplementedTestServer) CliStream(grpc.ClientStreamingServer[TestReq, TestRsp]) error {  
    return status.Error(codes.Unimplemented, "method CliStream not implemented")  
}  
func (UnimplementedTestServer) SerStream(*TestReq, grpc.ServerStreamingServer[TestRsp]) error {  
    return status.Error(codes.Unimplemented, "method SerStream not implemented")  
}  
func (UnimplementedTestServer) BothStream(grpc.BidiStreamingServer[TestReq, TestRsp]) error {  
    return status.Error(codes.Unimplemented, "method BothStream not implemented")  
}  
func (UnimplementedTestServer) mustEmbedUnimplementedTestServer() {}  
func (UnimplementedTestServer) testEmbeddedByValue()              {}  
 
// UnsafeTestServer may be embedded to opt out of forward compatibility for this service.
// Use of this interface is not recommended, as added methods to TestServer will
// result in compilation errors.  
type UnsafeTestServer interface {  
    mustEmbedUnimplementedTestServer()  
}  

---------------------------------------------------------------------------------------
服务注册函数

func RegisterTestServer(s grpc.ServiceRegistrar, srv TestServer) {  
    // If the following call panics, it indicates UnimplementedTestServer was  
    // embedded by pointer and is nil.  This will cause panics if an    // unimplemented method is ever invoked, so we test this at initialization    // time to prevent it from happening at runtime later due to I/O.    if t, ok := srv.(interface{ testEmbeddedByValue() }); ok {  
       t.testEmbeddedByValue()  
    }  
    s.RegisterService(&Test_ServiceDesc, srv)  
}  
  
--------------------------------------------------------------------------------------- 
根据路由调用方法处理函数处理 RPC 请求

func _Test_OnlyOnce_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {  
    in := new(TestReq)  
    //dec(in) 解码二进制数据 → TestReq 对象
    if err := dec(in); err != nil {  
       return nil, err  
    }  
    // 无拦截器，直接调用服务端实现的业务逻辑函数
    if interceptor == nil {  
       return srv.(TestServer).OnlyOnce(ctx, in)  
    }  
    
    info := &grpc.UnaryServerInfo{  
       Server:     srv,  
       FullMethod: Test_OnlyOnce_FullMethodName,  
    }  
    handler := func(ctx context.Context, req interface{}) (interface{}, error) {  
       return srv.(TestServer).OnlyOnce(ctx, req.(*TestReq))  
    }  
    // 先执行拦截器
    return interceptor(ctx, in, info, handler)  
}  
  
  
func _Test_CliStream_Handler(srv interface{}, stream grpc.ServerStream) error {  
     //调用服务端实现的业务逻辑函数
    return srv.(TestServer).CliStream(&grpc.GenericServerStream[TestReq, TestRsp]{ServerStream: stream})  
}  
  
  

func _Test_SerStream_Handler(srv interface{}, stream grpc.ServerStream) error {  
    //先读取一次请求
    m := new(TestReq)  
    if err := stream.RecvMsg(m); err != nil {  
       return err  
    }  
    //调用服务端实现的业务逻辑函数
    return srv.(TestServer).SerStream(m, &grpc.GenericServerStream[TestReq, TestRsp]{ServerStream: stream})  
}  
  

  
func _Test_BothStream_Handler(srv interface{}, stream grpc.ServerStream) error {
    //调用服务端实现的业务逻辑函数  
    return srv.(TestServer).BothStream(&grpc.GenericServerStream[TestReq, TestRsp]{ServerStream: stream})  
}  
  

---------------------------------------------------------------------------------------  请求到达先通过服务描述符进行路由

// Test_ServiceDesc is the grpc.ServiceDesc for Test service.
// It's only intended for direct use with grpc.RegisterService,  
// and not to be introspected or modified (even as a copy)  
var Test_ServiceDesc = grpc.ServiceDesc{  
    ServiceName: "test.test",  
    HandlerType: (*TestServer)(nil),  
    Methods: []grpc.MethodDesc{  
       {  
          MethodName: "onlyOnce",  
          Handler:    _Test_OnlyOnce_Handler,  
       },  
    },  
    Streams: []grpc.StreamDesc{  // 流式方法列表
       {  
          StreamName:    "CliStream",  
          Handler:       _Test_CliStream_Handler,  
          ClientStreams: true,  
       },  
       {  
          StreamName:    "SerStream",  
          Handler:       _Test_SerStream_Handler,  
          ServerStreams: true,  
       },  
       {  
          StreamName:    "BothStream",  
          Handler:       _Test_BothStream_Handler,  
          ServerStreams: true,  
          ClientStreams: true,  
       },  
    },  
    Metadata: "test.proto",  
}
```

## 4.2 消息类型的实现
```go

package test  
  
import (  
    protoreflect "google.golang.org/protobuf/reflect/protoreflect"  
    protoimpl "google.golang.org/protobuf/runtime/protoimpl"  
    reflect "reflect"  
    sync "sync"  
    unsafe "unsafe"  
)  
  
const (  
    // Verify that this generated code is sufficiently up-to-date.  
    _ = protoimpl.EnforceVersion(20 - protoimpl.MinVersion)  
    // Verify that runtime/protoimpl is sufficiently up-to-date.  
    _ = protoimpl.EnforceVersion(protoimpl.MaxVersion - 20)  
)  
  
// 定义消息结构，用作rpc调用函数的参数  
type TestReq struct {  
    state protoimpl.MessageState `protogen:"open.v1"`  
    // <type> <var_name> = <unique index>  
    Name          string `protobuf:"bytes,1,opt,name=name,proto3" json:"name,omitempty"`  
    Num           uint32 `protobuf:"varint,2,opt,name=num,proto3" json:"num,omitempty"`  
    unknownFields protoimpl.UnknownFields  
    sizeCache     protoimpl.SizeCache  
}  
  
func (x *TestReq) Reset() {  
    *x = TestReq{}  
    mi := &file_test_proto_msgTypes[0]  
    ms := protoimpl.X.MessageStateOf(protoimpl.Pointer(x))  
    ms.StoreMessageInfo(mi)  
}  
  
func (x *TestReq) String() string {  
    return protoimpl.X.MessageStringOf(x)  
}  
  
func (*TestReq) ProtoMessage() {}  
  
func (x *TestReq) ProtoReflect() protoreflect.Message {  
    mi := &file_test_proto_msgTypes[0]  
    if x != nil {  
       ms := protoimpl.X.MessageStateOf(protoimpl.Pointer(x))  
       if ms.LoadMessageInfo() == nil {  
          ms.StoreMessageInfo(mi)  
       }  
       return ms  
    }  
    return mi.MessageOf(x)  
}  
  
// Deprecated: Use TestReq.ProtoReflect.Descriptor instead.  
func (*TestReq) Descriptor() ([]byte, []int) {  
    return file_test_proto_rawDescGZIP(), []int{0}  
}  
  
func (x *TestReq) GetName() string {  
    if x != nil {  
       return x.Name  
    }  
    return ""  
}  
  
func (x *TestReq) GetNum() uint32 {  
    if x != nil {  
       return x.Num  
    }  
    return 0  
}  
  
type TestRsp struct {  
    state         protoimpl.MessageState `protogen:"open.v1"`  
    Rep           string                 `protobuf:"bytes,1,opt,name=rep,proto3" json:"rep,omitempty"`  
    Num           uint32                 `protobuf:"varint,2,opt,name=num,proto3" json:"num,omitempty"`  
    unknownFields protoimpl.UnknownFields  
    sizeCache     protoimpl.SizeCache  
}  
  
func (x *TestRsp) Reset() {  
    *x = TestRsp{}  
    mi := &file_test_proto_msgTypes[1]  
    ms := protoimpl.X.MessageStateOf(protoimpl.Pointer(x))  
    ms.StoreMessageInfo(mi)  
}  
  
func (x *TestRsp) String() string {  
    return protoimpl.X.MessageStringOf(x)  
}  
  
func (*TestRsp) ProtoMessage() {}  
  
func (x *TestRsp) ProtoReflect() protoreflect.Message {  
    mi := &file_test_proto_msgTypes[1]  
    if x != nil {  
       ms := protoimpl.X.MessageStateOf(protoimpl.Pointer(x))  
       if ms.LoadMessageInfo() == nil {  
          ms.StoreMessageInfo(mi)  
       }  
       return ms  
    }  
    return mi.MessageOf(x)  
}  
  
// Deprecated: Use TestRsp.ProtoReflect.Descriptor instead.  
func (*TestRsp) Descriptor() ([]byte, []int) {  
    return file_test_proto_rawDescGZIP(), []int{1}  
}  
  
func (x *TestRsp) GetRep() string {  
    if x != nil {  
       return x.Rep  
    }  
    return ""  
}  
  
func (x *TestRsp) GetNum() uint32 {  
    if x != nil {  
       return x.Num  
    }  
    return 0  
}  
  
var File_test_proto protoreflect.FileDescriptor  
  
const file_test_proto_rawDesc = "" +  
    "\n" +  
    "\n" +  
    "test.proto\x12\x04test\"/\n" +  
    "\atestReq\x12\x12\n" +  
    "\x04name\x18\x01 \x01(\tR\x04name\x12\x10\n" +  
    "\x03num\x18\x02 \x01(\rR\x03num\"-\n" +  
    "\atestRsp\x12\x10\n" +  
    "\x03rep\x18\x01 \x01(\tR\x03rep\x12\x10\n" +  
    "\x03num\x18\x02 \x01(\rR\x03num2\xba\x01\n" +  
    "\x04test\x12(\n" +  
    "\bonlyOnce\x12\r.test.testReq\x1a\r.test.testRsp\x12+\n" +  
    "\tCliStream\x12\r.test.testReq\x1a\r.test.testRsp(\x01\x12+\n" +  
    "\tSerStream\x12\r.test.testReq\x1a\r.test.testRsp0\x01\x12.\n" +  
    "\n" +  
    "BothStream\x12\r.test.testReq\x1a\r.test.testRsp(\x010\x01B\vZ\t./pb;testb\x06proto3"  
  
var (  
    file_test_proto_rawDescOnce sync.Once  
    file_test_proto_rawDescData []byte  
)  
  
func file_test_proto_rawDescGZIP() []byte {  
    file_test_proto_rawDescOnce.Do(func() {  
       file_test_proto_rawDescData = protoimpl.X.CompressGZIP(unsafe.Slice(unsafe.StringData(file_test_proto_rawDesc), len(file_test_proto_rawDesc)))  
    })  
    return file_test_proto_rawDescData  
}  
  
var file_test_proto_msgTypes = make([]protoimpl.MessageInfo, 2)  
var file_test_proto_goTypes = []any{  
    (*TestReq)(nil), // 0: test.testReq  
    (*TestRsp)(nil), // 1: test.testRsp  
}  
var file_test_proto_depIdxs = []int32{  
    0, // 0: test.test.onlyOnce:input_type -> test.testReq  
    0, // 1: test.test.CliStream:input_type -> test.testReq  
    0, // 2: test.test.SerStream:input_type -> test.testReq  
    0, // 3: test.test.BothStream:input_type -> test.testReq  
    1, // 4: test.test.onlyOnce:output_type -> test.testRsp  
    1, // 5: test.test.CliStream:output_type -> test.testRsp  
    1, // 6: test.test.SerStream:output_type -> test.testRsp  
    1, // 7: test.test.BothStream:output_type -> test.testRsp  
    4, // [4:8] is the sub-list for method output_type  
    0, // [0:4] is the sub-list for method input_type  
    0, // [0:0] is the sub-list for extension type_name  
    0, // [0:0] is the sub-list for extension extendee  
    0, // [0:0] is the sub-list for field type_name  
}  
  
func init() { file_test_proto_init() }  
func file_test_proto_init() {  
    if File_test_proto != nil {  
       return  
    }  
    type x struct{}  
    out := protoimpl.TypeBuilder{  
       File: protoimpl.DescBuilder{  
          GoPackagePath: reflect.TypeOf(x{}).PkgPath(),  
          RawDescriptor: unsafe.Slice(unsafe.StringData(file_test_proto_rawDesc), len(file_test_proto_rawDesc)),  
          NumEnums:      0,  
          NumMessages:   2,  
          NumExtensions: 0,  
          NumServices:   1,  
       },  
       GoTypes:           file_test_proto_goTypes,  
       DependencyIndexes: file_test_proto_depIdxs,  
       MessageInfos:      file_test_proto_msgTypes,  
    }.Build()  
    File_test_proto = out.File  
    file_test_proto_goTypes = nil  
    file_test_proto_depIdxs = nil  
}
```

# 5. 实际业务扩展
## 5.1 超时控制

为了避免客户端（调用服务方法时会阻塞）可能会无限期地等待响应，应该在客户端rpc请求中明确设置一个合理的截止时间
如果服务端在处理请求时超过了截止时间，客户端将放弃等待，并以  DEADLINE_EXCEEDED 状态使 RPC 调用失败

==**加入超时后**==：

![[Pasted image 20260908221204.png]]

==**客户端**==
当客户端设置的上下文超时，且该超时在服务端业务逻辑完成之前触发时，调用方法会停止阻塞返回状态码 codes.DeadlineExceeded
```go

// 一元为例，其它同理
func onlyOnce(t time.Duration,req *test.TestReq){
   // 创建一个t秒超时关闭的ctx
    ctx, cancel := context.WithTimeout(context.Background(), t)  
    // 调用成功关闭ctx，避免资源浪费
    defer cancel() 
    // 调用方法，传入ctx，req,进入阻塞等待
    //当客户端设置的上下文超时，且该超时在服务端业务逻辑完成之前触发时
    //client.OnlyOnce会返回状态码 codes.DeadlineExceeded
    req, err := client.OnlyOnce(ctx, req)  
    if err != nil {  
       log.Println(err)  
       return
    }  
    log.Println(req) 
  }
    
    
func main() {  
    // 获得与服务端的连接句柄  
    conn, err := grpc.NewClient("127.0.0.1:7979",                                           grpc.WithTransportCredentials(insecure.NewCredentials()))  
    if err != nil {  
       log.Fatal(err)  
    }  
    defer conn.Close()  
    // 注入连接 返回客户端接口实例
    client = test.NewTestClient(conn)  

    onlyOnce(4，myReq)
}  
```

==**服务端**==
服务端应用程序有责任停止其为服务该 RPC 而产生的任何活动
如果你的应用程序正在运行一个长周期进程
你应该定期检查触发该进程的 RPC 是否已被取消，如果是，则应停止处理
```go
func (t *testserver) OnlyOnce(ctx context.Context, in *test.TestReq) (out *test.TestRsp, err error) {  
    // 进入服务前先检查网络传输过程是否导致超时
    select {  
    case <-ctx.Done():  
       return nil, status.Errorf(codes.DeadlineExceeded, "request cancelled: %v", ctx.Err())  
    default:  
    }  
    
    // 模拟工作耗时  
    time.Sleep(2 * time.Second)  
    var res = test.TestRsp{  
       Rep: fmt.Sprintf("response:  recv:    name: %s     num: %d ", in.Name, in.Num),  
       Num: 0,  
    }  
    
    // 工作完成后检查是否超时，决定返回值以减少网络消耗
    select {  
    case <-ctx.Done():  
       return nil, status.Errorf(codes.DeadlineExceeded, "request cancelled: %v",  ctx.Err())  
    default:  
       return &res,nil  
    }  
    
    
    
func main() {  
    // 获得tcp监听器  
    ls, _ := net.Listen("tcp", "127.0.0.1:7979")  
    // 构建grpc server 实例  
    server := grpc.NewServer()  
    // 注册server 与 实现调用的对象  
    test.RegisterTestServer(server, &testserver{})  
    // 启动服务 阻塞  
    err := server.Serve(ls)  
    if err != nil {  
       return  
    }  
  
}
}
```

## 5.2 基于JWT的认证
==由于JWT未加密约等于裸奔，所以这一步必需建立在TSL加密信道基础上==



通过拦截器（Interceptor）在 RPC 调用前后处理认证逻。客户端将 JWT 放入 gRPC 的 metadata (元数据) 中发送，服务端从 metadata 中提取并验证 JWT，验证通过后将用户信息存入 context 供后续业务逻辑使用。


> [!fail]   想通过context既传递token，又传递超时消息是错误的
>    1. context只在本进程中建立父子关系以 .Value 方法向上寻值
>    2. grpc 没有定义如何传输这类关系
>    3. 其他语言不一定支持context
