---
layout: post
title:  "GRPC基本使用"
date:   2018-08-08 19:19:00
categories: 应用组件
tags: rpc java
---

* content
{:toc}

RPC相比HTTP提供了一种更高效的访问方式，这里主要记录下GRPC的基本使用

![](http://incdn1.b0.upaiyun.com/2016/10/57f7d50f506df745766064d6b9ec5143.png)





## proto文件
```java
// Copyright 2015 The gRPC Authors
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
syntax = "proto3";

option java_multiple_files = true;
option java_package = "cn.datatech.grpc.proxy";
option java_outer_classname = "ServiceRequestProxyProto";
option objc_class_prefix = "SRP";

package proxy;

service ServiceRequestProxy {
  rpc request(ServiceRequest) returns (ResponseModel) {}
}

message ServiceRequest{
	string service = 1; // 服务名称
	string method = 2; // 请求方法
	string body = 3; // 请求体
}

// 响应结果集
message ResponseModel {
  int32 code = 1;//状态码
  string message = 2; //消息
  string data = 3; //数据
}
```

## maven编译

```java
<plugin>
				<groupId>org.xolstice.maven.plugins</groupId>
				<artifactId>protobuf-maven-plugin</artifactId>
				<version>${protobuf.plugin.version}</version>
				<configuration>
					<protocArtifact>com.google.protobuf:protoc:${protoc.version}:exe:${os.detected.classifier}</protocArtifact>
					<pluginId>grpc-java</pluginId>
					<pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
					<protoSourceRoot>src/main/resources/proto</protoSourceRoot>
					<outputDirectory>src/main/java/</outputDirectory>
					<clearOutputDirectory>false</clearOutputDirectory>
				</configuration>
				<executions>
					<execution>
						<goals>
							<goal>compile</goal>
							<goal>compile-custom</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
```

## serve端构建

> 1.实现xxxxGrpc.xxxxxImplBase service类

> 2.构建Server = ServerBuilder.forPort(9999).addServcie(service).build();

> 3.启动server.start()

> 4.关闭监听 Runtime.getRuntime().addShutdownHook

> 4.等待主线程终止server.awaitTermination()

> 5.关闭 server.shutdown()

## client端构建

> 1.构建ManagedChannel = ManagedChannelBuilder.forAddress(host, port).usePlaintext()

> 2.建构xxxxBlockingStub = xxxxGrpc.newBlockingStub(ManagedChannel)

> 3.异步xxxxxStub = xxxxGrpc.newStub(ManagedChannel)

> 4.参数xxxxRequest.newBuilder().set.build

> 5.channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);


## example

### server

```java
public class HelloWordServer {

    public static void main(String[] args) {
        Server server = ServerBuilder.forPort(8989).addService(new GreeterService()).build();
        try {
            System.out.println("listing:" + 8989);
            server.start();
            server.awaitTermination();
        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        }
    }

    static class GreeterService extends GreeterGrpc.GreeterImplBase {
        @Override
        public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
            HelloReply reply = HelloReply.newBuilder().setMessage("Hello " + req.getName()).build();
            System.out.println(reply.getMessage());
            responseObserver.onNext(reply);
            responseObserver.onCompleted();
        }
    }
}
```

### client

```java
public class HelloWordClient {

    public static void main(String[] args) {
        while (true) {
            ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", 8989).usePlaintext().build();
            GreeterBlockingStub blockingStub = GreeterGrpc.newBlockingStub(channel);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            HelloRequest request = HelloRequest.newBuilder().setName("你好啊").build();
            blockingStub.sayHello(request);
        }
    }
}
```

`springboot启动checkParameter错误注意端口释放`