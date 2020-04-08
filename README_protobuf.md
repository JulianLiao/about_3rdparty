# protocol_buffers

.proto文件是简单易读的
.pb.h是很复杂的，第一感觉就是用工具生成的。

protocol buffer compiler

## Procotol buffers是啥，能干啥

protocol buffers是类似于xml的，用来序列化结构化数据(for serilizing structured data)。



## 如何写.proto文件？

当定义一个变量时，可以用optional, required, repeated修饰符，也可以不用这些修饰符。

```
在ThirdParty/pi_proto/planning.proto，有如下语句：
  package piauto.planning;

  此外，还有如下语句：
  import "decision.proto"

  定义了如下的message

  message Planning {
    common.Header header = 1;
    ControlState control_state = 2;
    VehicleState state = 3;
    // decision of the vehicle, lane follow, stop by obstacle and etc..
    DecisionResult decision = 4;
    repeated common.Point3D trajectory_point = 5;  // predict trajectory
    ADCSignals adc_signals = 6;
    bool autonomous_mode = 7;
  }

在ThirdParty/pi_proto/decision.proto，有如下语句：
  package piauto.planning;

  在planing.proto中所引用的DecisionResult定义如下：
  message DecisionResult {
    MainDecision main_decision = 1;  // decisions based on task and motion planning
    ObjectDecisions object_decision = 2;  // decisions based on each object
  }

  其中，MainDecision这个message的定义如下：
  message MainDecision {
    oneof task {
      MainLaneKeeping lane_keeping = 1;
      MainStop stop = 2;
      MainEmergencyStop estop = 3;
      MainMissionComplete mission_compelete = 4;
      MainNotReady not_ready = 5;
      MainParking parking = 6;
    }
  }

  挑出其中的MainStop看下，

  message MainStop {
    StopReasonCode reason_code = 1;
    string reason = 2；
    // When stopped, the front center of vehicle should be at this point.
    common.Point3D stop_point = 3;
    // When stopped, the heading of the vehicle should be stop_heading.
    double stop_heading = 4;
  }

  可以想象得到，StopReasonCode也会是一个message，其实不然，StopReasonCode是一个enum，它代表了停车类型的枚举。

  enum StopReasonCode {
    STOP_REASON_HEAD_VEHICLE = 0;
    STOP_REASON_DESTINATION = 1;
    STOP_REASON_PEDESTRIAN = 2;
    STOP_REASON_OBSTACLE = 3;
    STOP_REASON_PREPARKING = 4;
    STOP_REASON_SIGNAL = 5;
    STOP_REASON_STOP_SIGN = 6;
    STOP_REASON_YIELD_SIGN = 7;
    STOP_REASON_CLEAR_ZONE = 8;
    STOP_REASON_CROSSWALK = 9;
    STOP_REASON_CREEPER = 10;
    STOP_REASON_REFERENCE_END = 11;  // end of the reference_line
    STOP_REASON_YELLOW_SIGNAL = 12;  // yellow signal
    STOP_REASON_LANE_CHANGE_URGENCY = 13;
  }


## 如何使用.proto文件？


第一步：定义了一个package piauto.planning && message Planning对象
    piatuo::planning::Planning planning_decision;
    const char *body;
    unsigned int len;
    planning_decision.ParseFromArray(body, len);  ## 我理解，这句话是给planning_decision字段做了赋值。
    // Planning确实有一个字段是'decision'，#mutable_decision我理解是解决了多线程问题，另外，调用mutable_decision()返回的是一个指针。
    // DecisionResult确实有一个字段'main_decision'，#mutable_main_decision我理解是解决了多线程问题，另外，调用mutable_main_decision()返回的是一个指针。
    // 我理解前面加上mutable_可能和多线程没有关系，仅仅是返回值是一个指针
    // 可以确定在proto buff里，获取字段前面加上mutable，返回的是指针，不加这个mutable返回的不是指针。问题是，mutable是否和多线程有关，这个还得确认下。
    planning_decision.mutable_decision()->mutable_main_decision();
    piauto::planning::MainDecision *main_decision = planning_decision.mutable_decision()->mutable_main_decision();
    // MainDecision确实有个字段是stop
    if (main_decision->has_stop()) {

    }
    // MainDecision确实有字段stop 和 estop
    if (!main_decision->has_stop() && !main_decision->has_estop()) {

    }


```



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