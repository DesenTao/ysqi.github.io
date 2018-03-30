
---
date: 2016-12-31T11:32:37+08:00
title: "GolanggRPC实践连载五拦截器Interceptor"
description: ""
disqus_identifier: 1485833557345156006
slug: "Golang-gRPCshi-jian--lian-zai-wu--lan-jie-qi--Interceptor"
source: "https://segmentfault.com/a/1190000007997759"
tags: 
- golang 
- grpc 
topics:
- 编程语言与开发
---

Interceptor
===========

grpc服务端提供了interceptor功能，可以在服务端接收到请求时优先对请求中的数据做一些处理后再转交给指定的服务处理并响应，功能类似middleware，很适合在这里处理验证、日志等流程。

在自定义Token认证的示例中，认证信息是由每个服务中的方法处理并认证的，如果有大量的接口方法，这种姿势就太蛋疼了，每个接口实现都要先处理认证信息。这个时候interceptor就站出来解决了这个问题，可以在请求被转到具体接口之前处理认证信息，一处认证，到处无忧，看代码吧，修改hello-token项目的服务端实现：

server/main.go

    package main

    import (
        "net"

        pb "git.vodjk.com/go-grpc/example/proto"

        "golang.org/x/net/context"
        "google.golang.org/grpc"
        "google.golang.org/grpc/codes"       // grpc 响应状态码
        "google.golang.org/grpc/credentials" // grpc认证包
        "google.golang.org/grpc/grpclog"
        "google.golang.org/grpc/metadata" // grpc metadata包
    )

    const (
        // Address gRPC服务地址
        Address = "127.0.0.1:50052"
    )

    // 定义helloService并实现约定的接口
    type helloService struct{}

    // HelloService ...
    var HelloService = helloService{}

    func (h helloService) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
        resp := new(pb.HelloReply)
        resp.Message = "Hello " + in.Name + "."

        return resp, nil
    }

    // auth 验证Token
    func auth(ctx context.Context) error {
        md, ok := metadata.FromContext(ctx)
        if !ok {
            return grpc.Errorf(codes.Unauthenticated, "无Token认证信息")
        }

        var (
            appid  string
            appkey string
        )

        if val, ok := md["appid"]; ok {
            appid = val[0]
        }

        if val, ok := md["appkey"]; ok {
            appkey = val[0]
        }

        if appid != "101010" || appkey != "i am key" {
            return grpc.Errorf(codes.Unauthenticated, "Token认证信息无效: appid=%s, appkey=%s", appid, appkey)
        }

        return nil
    }

    func main() {
        listen, err := net.Listen("tcp", Address)
        if err != nil {
            grpclog.Fatalf("Failed to listen: %v", err)
        }

        var opts []grpc.ServerOption

        // TLS认证
        creds, err := credentials.NewServerTLSFromFile("../../keys/server.pem", "../../keys/server.key")
        if err != nil {
            grpclog.Fatalf("Failed to generate credentials %v", err)
        }

        opts = append(opts, grpc.Creds(creds))

        // 注册interceptor
        var interceptor grpc.UnaryServerInterceptor
        interceptor = func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
            err = auth(ctx)
            if err != nil {
                return
            }
            // 继续处理请求
            return handler(ctx, req)
        }
        opts = append(opts, grpc.UnaryInterceptor(interceptor))

        // 实例化grpc Server
        s := grpc.NewServer(opts...)

        // 注册HelloService
        pb.RegisterHelloServer(s, HelloService)

        grpclog.Println("Listen on " + Address + " with TLS + Token + Interceptor")

        s.Serve(listen)
    }

运行：

    go run main.go

    Listen on 50052 with TLS + Token

运行客户端程序 client/main.go：

    go run main.go

    // 认证成功结果
    Hello gRPC
    Token info: appid=101010,appkey=i am key

    // 认证失败结果：
    rpc error: code = 16 desc = Token认证信息无效: appID=101010, appKey=i am not key

运行结果和hello-token项目一样，简单不，只需要在实例化server前注册需要的interceptor，就可以轻松解决那个蛋疼的问题，想注册几个就注册几个。

**项目推荐：**

[go-grpc-middleware](https://github.com/mwitkow/go-grpc-middleware)

这个项目对interceptor进行了封装，支持多个拦截器的链式组装，对于需要多种处理的地方使用起来会更方便些。

参考
----

### 本系列示例代码

-   [go-grpc-example](https://github.com/Jergoo/go-grpc-example)



