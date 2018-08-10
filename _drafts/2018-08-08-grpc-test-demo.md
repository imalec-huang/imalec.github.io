xxxx-grpc
serve端构建
1.实现xxxxGrpc.xxxxxImplBase service类
2.构建Server = ServerBuilder.forPort(9999).addServcie(service).build();
3.启动server.start()
4.关闭监听 Runtime.getRuntime().addShutdownHook
4.等待主线程终止server.awaitTermination()
5.关闭 server.shutdown()

client端构建
1.构建ManagedChannel = ManagedChannelBuilder.forAddress(host, port).usePlaintext()
2.建构xxxxBlockingStub = xxxxGrpc.newBlockingStub(ManagedChannel)
3.异步xxxxxStub = xxxxGrpc.newStub(ManagedChannel)
4.参数xxxx..newBuilder().set.build
5.channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);

问题
springboot启动checkParameter错误注意端口释放



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

修改官方源为豆瓣源

编辑配置文件, 如果没有, 新建一份:

vi ~/.pip/pip.conf

添加内容如下:
[global]
index-url = http://pypi.douban.com/simple
trusted-host = pypi.douban.com