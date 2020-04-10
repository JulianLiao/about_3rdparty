
## 我关于PIMsgAdaptor的认知

PIMsgAdaptor用到了boost::signals2。

首先，用boost::signals2::signal 定义了一个变量 msg_signal_t，如下：
typedef boost::signals2::signal<pi_msg_callback_t> msg_signal_t;

之后又定义了如下的订阅接口，
boost::signals2::connection addSubscriberToSubMsg(const msg_signal_t::slot_type &subscriber);

调用订阅接口的手法很高明，直接把函数的实现放在参数里了。

```
这样的用法，是否眼熟呢？

radar的callback是啥，void(const int, const Radar77Data &)

在radar_77.h，首先用boost::signals2::signal定义了一个变量 radar_signal_t，如下

typedef boost::signals2::signal<void(const int, const Radar77Data &)> radar_signal_t;

之后，又定义了如下的订阅接口，
boost::signals2::connection SubscribeToRadar(const radar_signal_t::slot_type &subscriber);

回顾下，在radar_barrier_range_finder.cc是如何调用订阅接口的

can_obj_->GetRadar_77(0).SubscribeToRadar(boost::bind(&RadarBarrierRangeFinder::Impl::RadarUpdate, this, _1, -2));

RadarUpdate声明如下：
void RadarUpdate(const int chl, const chassis::Radar77Data &radar_objs);
```




首先，定义了一个回调函数

handler - 这条消息所对应的通道号，the channel id this message related to
p_env - 对所要发送消息的一个封装结构体
body - 消息内容
len - 消息内容长度
using pi_msg_callback_t = int(int handler, p_pi_msg_envelope_t p_env, const char *body, unsigned int len);

关于 p_pi_msg_envelope_t 定义如下，

```
要知道nanomsg 本身就是关于消息的。

正如其名字所言，pi_msg_envelope_t就是对打算发送的消息做了个封装。

typedef struct pi_msg_envelope_t_ {
    // used for message filtering
    char filter[16];        // filter肯定是用作过滤的，是对type字段作过滤吗
    // type of the package
    unsigned int type;        // type可能的取值都有啥，这个你需要了解下
    // counts of bytes in data block
    unsigned int length;        // 数据包含多少个字节
    // placeholder for the start address of actual body info.
    const char *data;
} pi_msg_envelope_t, *p_pi_msg_envelope_t;
```

关于 #pi_msg_envelope_t 里面的字段type的可能取值，如下

typedef enum pi_msg_type_t_ {
    // message from UI module
    PIMSG_UI_NULL = 128,
    PIMSG_UI_REQUEST_SET_TARGET,
    PIMSG_UI_REQUEST_LLOOP_START,
    PIMSG_UI_xx,

    // message from chassis&control module
    PIMSG_CPC_NULL = 256,
    PIMSG_CPC_PUBLISH_POS,
    PIMSG_CPC_PUBLISH_PLANPATH,
    PIMSG_CPC_PUBLISH_RADAR,
    PIMSG_CPC_xx,

    // message from Localization module
    PIMSG_LOCALIZATION_NULL = 384,
    PIMSG_LOCALIZATION_VIO_PUBLISH_POS,
    PIMSG_LOCALIZATION_VIO_PUBLISH_POS_DEBUG,
    PIMSG_LOCALIZATION_xx,

    // product publish msg
    PIMSG_SYS_CHASSIS_PUBLISH,
    PIMSG_SYS_LOCALIZATION_PUBLISH,
    PIMSG_SYS_PERCEPTION_PUBLISH,
    PIMSG_SYS_PLANNING_PUBLISH,
    PIMSG_SYS_ROUTE_PUBLISH,
    PIMSG_SYS_MAP_PUBLISH,

    // message from VIDAR module
    PIMSG_VIDAR_NULL = 512,
    PIMSG_VIDAR_RADAR_OBSTACLES_RAWDATA,

    // message from Sensor module
    PIMSG_SENSOR_STEREOIMAGERAWDATA = 700,
    PIMSG_SENSOR_IMURAWDATA = 701,
    PIMSG_SENSOR_GPSRAWDATA = 702
} pi_msg_type_t;

pi_msg_envelope_t是对所要发送消息的封装，是对消息的描述。

封装好的消息是要建立起通道，靠通道往外发送的。PIMsgChannelInfo则是对通道的描述。

struct PIMsgChannelInfo {
    // 通道类型，是sub（订阅通道）还是req（request/reply）
    // NN_SUB, 订阅通道
    int type;

    // 要么是tcp url，要么是ipc文件路径
    // tcp://127.0.0.1:6777
    std::string url;

    // 如果指定了该filter，只有以该filter起头的消息还会被接收。
    // "system"，只接收以system起头的消息
    std::string filter;

    int handler;  // handler为负数代表无效通道
}






我的疑问：
1. #pi_msg_envelope_t里面的data字段 和 #pi_msg_callback_t里面的body参数 之间的区别是啥？
    要知道，他们都是 const char *数据类型。


## PIMsgAdapter重点函数分析

// 成功的话，会返回一个非负数的channel id
int registerOneMessageChannel(PIMsgChannelInfo chl);



std::shared_ptr<PIAUTO::msg::PIMsgAdapter> vehicle_status_adp_;    // 从命名来看，这个adapter和车体状态有关。
vehicle_status_adp_ = std::make_shared<PIAUTO::msg::PIMsgAdapter>();

std::string sub_status_addr = "tcp://127.0.0.1:6777";
int sub_status_handler_ = vehicle_status_adp_->registerOneMessageChannel({NN_SUB, "tcp://127.0.0.1:6777", "system", -1});

// 回顾下，回调函数的定义是这样的
// using pi_msg_callback_t = int(int handler, p_pi_msg_envelope_t p_env, const char *body, unsigned int len);
vehicle_status_adp_->addSubscriberToSubMsg([&](int handler, p_pi_msg_envelope_t p_env, const char *body, unsigned int len) {
  if (PIMSG_SYS_PLANNING_PUBLISH == p_env->type) {
      LOG(INFO) << "Planning vehicle status is received";

      piauto::planning::Planning planning_decision;
      planning_decision.ParseFromArray(body, len);

      std::unique_lock<std::shared_timed_mutex> lck(vehicle_status_mutex_);

      piauto::planning::MainDecision *main_decision = planning_decision.mutable_decision()->mutable_main_decision();

      // MainStop is triggered
      if (main_decision->has_stop()) {
          LOG(INFO) << "Planning vehicle status: main stop";

          piauto::planning::MainStop *main_stop = main_decision->mutable_stop();
          piatuo::planning::StopReasonCode stop_reason = main_stop->reason_code();  // 造成车体停车有很多种原因，而我们只关心探测到障碍物而停车的情况
          if (piauto::planning::STOP_REASON_OBSTACLE == stop_reason || piauto::planning::STOP_REASON_PEDESTRIAN == stop_reason || piauto::planning::STOP_REASON_HEAD_VEHICLE == stop_reason) {
              LOG(INFO) << "Planning vehicle status: main stop, reason: obstacle";
              if (true == save_full_rawdata_) {
                  // 之前已经收到了PIMSG_SYS_PLANNING_PUBLISH并且车体因障碍物停车的消息，没必要创建新的vehicle_stop_timestamp.txt文件
                  // 此种情况，无需做任何修改
              } else {
                  // 存文件的标记还没被置过，此时需要做2个动作： 1. 置存文件标记 2. 创建一个新的vehicle_stop_timestamp.xx
                  // 这种情况是发生在收到PIMSG_SYS_PLANNING_PUBLISH前提下，首次收到stop_reason为OBSTACLE消息
                  save_full_rawdata_ = true;
                  std::string file_name = "tmp/vehicle_stop_timestamp.txt";
                  raw_data_file_.open(file_name, std::ios::out);
              }
          }
      } else if (!main_decision->has_stop() && !main_decision->has_estop()) {
          // 这个case是在收到了planning发过来的PIMSG_SYS_PLANNING_PUBLISH的前提下，同时车体既没有停车，也没有紧急停车
          // vehicle doesn't stop, it's moving
          // 进入该case，我采取了2个动作：1. 把存文件标记置为false 2. 关闭文件
          LOG(INFO) << "Planning vehicle status: moving or waiting";

          save_full_rawdata_ = false;
          if (raw_data_file_.is_open()) {
              raw_data_file_.close();
          }
      }
      return 1;
  }
});

就像我一开始怀疑的那样，RadarBarrierRangeFinder::Impl只是注册一个回调函数，这个回调函数什么时候被调用，是依赖于msg_signal_t调用的。

构造函数只负责置flag和创建文件，而不负责写具体的源数据。负责写具体的源数据是发生在 #DataProcessLoop这个线程里的，并不是发生在 GetObstacles这个接口调用，注意，写文件并不依赖于GetObstacle接口的调用，但确实依赖于planning 通过msg_signal_t发消息过来（打开flag并且创建文件）。

radar_barrier_range_finder.cc 用到了 planning.pb.h，但它用到了 radar_obstacle.pb.h了吗？


SendProtoLoop() 这个线程干的事情，是把radar源数据发送给感知的进程，视觉和radar Associate用的。

当实际运行1/2个例子后，就会理解pi_msg_envelope_t含义。