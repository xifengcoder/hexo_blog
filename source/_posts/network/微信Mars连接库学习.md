---
title: 微信Mars连接库学习
urlname: network_mars
date: 2021-12-26 17:52:11
tags: TCP
categories: Network
description: 微信Mars长连接库的学习与理解...
---

#### 主要目标

在尽量不影响用户收消息**及时性**的前提下，根据网络类型**自适应**的找出保活信令TCP连接的**尽可能大的心跳间隔**，从而达到减少因心跳引起的空中信道资源消耗，减少心跳Server的负载，以及减少部分因心跳引起的耗电。

#### 自适应心跳间隔优化：

##### 影响TCP连接寿命的因素

1. NAT超时
2. DHCP的租期
3. 网络状态变化

##### 心跳范围选择

1. ###### 前后台区分处理

   为了保证微信收消息及时性的体验，当微信处于前台活跃状态时，使用固定心跳；

   微信进入后台时，先用几次最小心跳维持长链接。然后进入后台自适应心跳计算。这样做的目的是尽量选择用户不活跃的时间段，来减少心跳计算可能产生的消息不及时收取影响。

2. ###### 后台自适应心跳选择区间

   **2.1 自适应心跳算法描述：**

   **按网络类型区分计算：**

   **a）变量说明**

   [MinHeart，MaxHeart]——心跳可选区间，具体是[4.5min,  9min50s]；
   NetHeartbeatInfo.cur_heart_ ——当前心跳间隔
   NetHeartbeatInfo.success_curr_heart_count_——当前心跳间隔下的成功次数。当心跳间隔变化时，该值清零。
   NetHeartbeatInfo.fail_heart_count_——当前心跳间隔下的失败次数。当心跳间隔变化时，该值清零。
   
```cpp
// Heartbeart Range
#define MinHeartInterval (4  * 60 * 1000 + 30 * 1000)   //最小心跳间隔（4.5min）
#define MaxHeartInterval (9 * 60 * 1000 + 50 * 1000)   //最大心跳间隔（9min50s）

#define HeartStep (60 * 1000)  //心跳增加步长
#define SuccessStep  (20 * 1000) //稳定期后的探测步长

#define MaxHeartFailCount (3)    //最大心跳失败次数阈值
#define BaseSuccCount (3) //当前心跳间隔下的成功次数阈值
#define NetStableTestCount (3)  //使用最小心跳值达到3次时，开始探测动态心跳。

```

长链接建立时，先用最小心跳（MinHeartInterval），当成功次数达到NetStableTestCount(3)时，开始心跳探测。
当前心跳间隔（current_net_heart_info_.cur_heart_）下：
1. 如果成功次数达到阈值时，在当前心跳间隔的基础上增加HeartStep(60s)，最大不高于MaxHeartInterval（9min50s）；
当心跳值达到阈值时，设置心跳间隔为MaxHeartInterval - SuccessStep，并设为稳定态，同时成功和失败次数清零。
2. 如果失败次数达到阈值时，注意这里面根据是否是稳定状态（is_stable_）分开处理：
稳定状态：心跳值直接用最小值，并翻转为非稳定状态，同时成功和失败次数清零；
非稳定状态：在当前心跳间隔的基础上减小（HeartStep + SuccessStep），最小不低于MinHeartInterval（4.5mins），并翻转为稳定状态。同时成功和失败次数清零。

### 动态超时机制
```cpp
const static unsigned int kWifiTaskDelay = 1500; //Wifi下延迟时间(ms)
const static unsigned int kGPRSTaskDelay = 3000; //GPRS下延迟时间(ms)


const static unsigned int kGPRSMinRate = 4 * 1024; //GPRS下最低网速（bit/s）
const static unsigned int kWifiMinRate = 12 * 1024; // Wifi下最低网速（bit/s）


const static unsigned int kDynTimeFirstPackageWifiTimeout = 7 * 1000; //wifi下动态传输时间（网络状况佳）
const static unsigned int kDynTimeFirstPackageGPRSTimeout = 10 * 1000; //mobile动态首包传输时间（网络状况佳）

//基准首包传输时间
const static unsigned int kBaseFirstPackageWifiTimeout = 12 * 1000; //wifi下基准首包传输时间
const static unsigned int kBaseFirstPackageGPRSTimeout = 15 * 1000; //GPRS下基准首包传输时间


//最大超时时间阈值
const static unsigned int kMaxFirstPackageWifiTimeout = 22 * 1000; //wifi下首包最大超时时间（阈值）
const static unsigned int kMaxFirstPackageGPRSTimeout = 30 * 1000; //GPRS下手包最大超时时间（阈值）

const static unsigned int kMaxRecvLen = 64 * 1024; //最大接收长度
```

#### 首包超时时间： 

1. 网络状态Excellent、且没有设置服务器的处理耗时时间，则首包超时时间(ms)为：

   动态首包传输时间 + 排队个数 * 任务延迟时间

2. 如果设置了服务器的处理耗时（StnLogic.Task的serverProcessCost字段），则首包超时时间(ms)：

   传入的服务器处理耗时 + 1000 *  发包大小 / 最低网速 + 排队个数 * 任务延迟时间;

3. 否则的话，首包超时时间(ms)为：

   基准首包传输时间 + 1000 * 发包大小 / 最低网速+ 排队个数 * 任务延迟时间

#### 读写超时时间：

首包超时时间 + 1000 * 最大接收长度 / 最小传输速率