# protocol_buffers

.proto文件是简单易读的
.pb.h是很复杂的，第一感觉就是用工具生成的。

protocol buffer compiler

## Procotol buffers是啥，能干啥

protocol buffers是类似于xml的，用来序列化结构化数据(for serilizing structured data)。



## 如何写.proto文件？

当定义一个变量时，可以用optional, required, repeated修饰符，也可以不用这些修饰符。




## 关于pb编译器

protoc

在ThirdParty v1.0.1 cmake/Modules/ProtoBuf.cmake，我理解'${PROTOBUF_PROTOC_EXECUTABLE}'，这个宏所表示的就是protoc。

### pb编译器使用范例

protoc --proto_path=src --cpp_out=build/gen src/foo.proto src/bar/baz.proto

输入是src/foo.proto 和 src/bar/baz.proto，编译器会产生4个输出文件，
    a. build/gen/foo.pb.h and build/gen/foo.pb.cc
    b. build/gen/bar/baz.pb.h and build/gen/bar/baz.pb.cc

--cpp_out
  指定你要把你的C++输出放到哪个目录下。编译器产生一个header文件(.h)和一个具体实现文件(.cc)

最后产生文件的目录会由cpp_out指定的目录('build/gen')取代proto_path的目录('src')

src/foo.proto，用'build/gen'取代'src'，结果就是build/gen/foo.pb.h 和 build/gen/foo.pb.cc

以radar为例，radar_obstacle.proto 应该会产生 radar_obstacle.pb.h 和 radar_obstacle.pb.cc 这两个文件。

  - perception/radar/proto/radar_obstacle.proto

cmake .. && make 之后，才会产生.pb.h 和 .pb.cc文件。

mkdir build
cmake .. && make

  - build/perception/radar/proto/radar_obstacle.pb.h
  - build/perception/radar/proto/radar_obstacle.pb.cc








## 来个例子吧

### example1
```
message Person {
    required string name = 1;
    required int32 id = 2;
    optional string email = 3;

    enum PhoneType {
        MOBILE = 0;
        HOME = 1;
        WORK = 2;
    }

    message PhoneNumber {
        required string number = 1;
        optional PhoneType = 2 [default = HOME];
    }

    repeated PhoneNumber phone = 4;
}

很明显，name(1), id(2), email(3), phone(4)按顺序做了编号。
```