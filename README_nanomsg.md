
## 关于PIMsgAdaptor，我的认知

1. PIMsgAdaptor用到了boost::signals2。


## PIMsgAdapter重点变量及其类型分析

### 消息

pi_msg_envelope_t是对所要发送消息的封装，是对消息的描述。

```
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

疑问1. #pi_msg_envelope_t里面的data字段 和 #pi_msg_callback_t里面的body参数 之间的区别是啥？

    要知道，他们都是 const char *数据类型。无论是接数据，还是发数据，很少用到#pi_msg_envelope_t的data字段，大多数时候都用的是#pi_msg_callback_t里面的body参数。

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
```

### 消息通道MsgChannel

封装好的消息是要建立起通道，靠通道往外发送的。PIMsgChannelInfo则是对通道的描述。

struct PIMsgChannelInfo {
    // 通道类型，是sub（订阅通道）还是req（request/reply）
    // NN_SUB, 订阅通道
    // NN_PUB
    // NN_REQ
    // NN_REP, 订阅通道
    // NN_RESPONDENT
    // NN_SURVEYOR
    int type;

    // 要么是tcp url，要么是ipc文件路径
    // tcp://127.0.0.1:6777
    std::string url;

    // 如果指定了该filter，只有以该filter起头的消息还会被接收。
    // "system"，只接收以system起头的消息
    std::string filter;

    int handler;  // handler为负数代表无效通道
}
--------------------------------
// 它的key只有3个取值

//    NN_SUB    -    NN_SUB这种通道类型可以有很多个通道，channel1, channel2, channel3...，所以，对应的value会是一个std::vector类型

//    NN_REP

//    NN_RESPONDENT

typedef std::unordered_map<int, std::vector<PIMsgChannelInfo>> channels_map_t;

### 回调函数callback

要有订阅接口，除了定义订阅接口外，还得定义其对应的callback。

boost::signals2::connection addSubscriberToSubMsg(const msg_signal_t::slot_type &subscriber);

关于msg_signal_t的定义如下：typedef boost::signals2::signal<pi_msg_callback_t> msg_signal_t;

handler
    - 这条消息是哪个通道发过来的
p_env
    - #handler通道来的消息的封装结构体
body
    - 这个通道发过来的具体消息内容
len
    - 这个通道发过来的具体消息内容长度

关于回调函数的定义如下：using pi_msg_callback_t = int(int handler, p_pi_msg_envelope_t p_env, const char *body, unsigned int len);

调用订阅接口的手法很高明，直接把函数的实现放在参数里了。

```
回想一下，当时在radar_77.h，是如何定义订阅接口及其对应的callback

boost::signals2::connection SubscribeToRadar(const radar_signal_t::slot_type &subscriber);

    关于radar_sig_t定义如下：typedef boost::signals2::signal<void(const int, const Radar77Data &)> radar_signal_t;

在radar_barrier_range_finder.cc，是如何调用订阅接口的？

can_obj_->GetRadar_77(0).SubscribeToRadar(boost::bind(&RadarBarrierRangeFinder::Impl::RadarUpdate, this, _1, _2));

其中，RadarUpdate声明如下：

void RadarUpdate(const int id, const Radar77Data &radar_objs);
```

## PIMsgAdapter重点函数分析

### 注册消息通道
// 成功的话，会返回一个非负数的channel id
int registerOneMessageChannel(PIMsgChannelInfo chl);

注册了消息通道，就会有进程往这个消息通道发送消息

不管是发送消息，还是接收消息，都要注册一个消息通道


### 注册回调函数

// 监听从不同通道过来的消息
boost::signals2::connection addSubscriberToSubMsg(const msg_signal_t::slot_type &subscriber);


### 往指定通道发消息

// 返回值是0，发送成功，不是0，发送失败
int send(int handler, p_pi_msg_envelope_t p_env, const char *body, unsigned int len, std::function<pi_msg_callback_t> callback);

std::string pub_addr = "tcp://0.0.0.0:4050";
std::shared_ptr<PIAUTO::msg::PIMsgAdaptor> obsmsg_adp_ = std::make_shared<PIAUTO::msg::PIMsgAdaptor>();
int pub_obs_handler_ = obsmag_adp_->registerOneMessageChannel({NN_SUB, "tcp://0.0.0.0:4050", "radar_obs", -1});

    size_t size = radar_obstacles.ByteSizeLong();  // radar_obstacles这个message所包含的字节数
    char buff[size];
    radar_obstacles.SerializeToArray(buff, size);

pi_msg_envelope_t env;
memcpy(env.filter, "radar_obs", sizeof("radar_obs"));
env.type = PIMSG_VIDAR_RADAR_OBSTACLES_RAWDATA;
env.length = size;

int rc = obsmsg_adp_->send(pub_obs_handler_, &env, buff, size, nullptr);

### 启动/关闭接收线程

void startRecvLoop();
  如果只注册了NN_SUB消息通道的话，那么只会启动一个线程 #subReceiveLoop

  在 #subReceiveLoop线程里，做了2件事情：
      a. 收数据
          收数据是看注册了哪些消息通道，他就从这些消息通道里一个一个去收消息
      b. 发数据
          发数据是看谁注册了回调函数，谁注册了就发给谁

void stopRecvLoop();

注意：startRecvLoop()不是一定要调用的，如果只是要往外发消息，就没必要启动接收线程。如果需要从外面接收消息，那就需要调用startRecvLoop()。

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



## 关于DataCollection

所有运营数据都会被收集，这些数据包括：

- 运营时间
- 干预次数
- 不同模式下的里程数

- chassis数据
  * 车速
  * 转角
  * 刹车
- planning数据
  * 信号(signal)
  * 决策(decision)
- perception数据
  * 视觉obstacles
  * radar obstacles
  * sonar obstacles
- localization数据
  * pose
  * heading
- 路线(route)数据
  * 地图


### chassis 与 DataCollection交互

    Step1: ./chassis_test

    Step2: ./data_collection --ID=LiaoMeng --chassis_addr=tcp://0.0.0.0:7001

      运行Step2后，就创建了一个目录，2020_04_10_20_53_00，里面会出现 chassis.pb.dat，这个文件里面保存了chassis发过来的数据。