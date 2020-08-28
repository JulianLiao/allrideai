# allrideai

## 坐标系

### PP7坐标系

PP7坐标系已经画在设备上了，一般是x向右，y向前。不管怎么安装pp7，x指的方向就是pp7 imu的x轴，y指的方向就是pp7 imu的y轴。不同的安装角度，影响的是pp7和lidar坐标系的旋转角度，以及pp7和车体坐标系的旋转角度。


## 标定

当在POD4上安装PP7和Lidar-32，用作地图车时，PP7的坐标系：x向右，y向前，z向上。

setinstranslation ant1 -0.15 0.45 1.22 0.10 0.10 0.10  # 这条语句表明，主天线在PP7坐标系下的位置是：左边0.15m，前边0.45m，上方1.22m，x/y/z三个方向上允许的误差范围均是0.1m。

setinstranslation ant2 1.05 0.45 1.22 0.10 0.10 0.10  # 这条语句表明，从天线在PP7坐标系下的位置是：右边1.05m，前方0.45m，上方1.22m，x/y/z三个方向上允许的误差范围均是0.1m。

1. 运行 inscalibrate align new 0.1

敲命令 log inscalstatus onchanged

当发现"9. Source Status"由 CALIBRATING 变成 CALIBRATED，就表明align的标定过程已经完成。

Source Status
----
ASCII  |  Description
----|----|----
CALIBRATING  |  offset values是在标定过程中给出的
CALIBRATED  |  offset values是在标定过程完成后给出的
INS_CONVERGING  |  offset values就是初始的输入值。此时，calibration过程还没开始，直到ins solution是converged，calibration过程才真正开始


![align calibrate](imgs/gps_ins/align_calibrate_complete.png "align calibrate")

做"inscalibrate align new 0.1"之前的insconfig如下：

![insconfig before align calib](imgs/gps_ins/insconfig_before_align_calibrate.png "insconfig before align calib")

完成"inscalibrate align new 0.1"之后的insconfig如下：

![insconfig after align calib](imgs/gps_ins/insconfig_after_align_calibrate.png "insconfig after align calib")

可以看到，ALIGN IMUBODY的结果由"0 0 -90 9.2356 0 6.9952 FROM_DUAL_ANT" 变成了 "0 0 -73.7647 45 45 0.0999 CALIBRATED"

运行inscalibrate align new 0.1，依次达到CALIBRATED时输出，

1. （第一次）0  0  -73.7647 45 45 0.0999 CALIBRATED
2. （第二次）0  0  -73.6808 45 45 0.2789 CALIBRATED
3. （第三次）0  0  -82.4248 45 45 0.1614 CALIBRATED
4. （第四次）0 -0  -90.2700 45 45 0.0897 CALIBRATED
5. （第五次）0 -0  -90.4366 45 45 0.0927 CALIBRATED


## 制图



## ros topics


topic | 备注
-----|-----
/novatel_data/inspvas  |  未知
/novatel_data/inspvax  |  未知
/rosout  |  未知
/velodyne_32_points  |  未知
  |  未知
  |  未知
  |  未知
  |  未知
  |  未知
  |  未知
  |  未知
  |  未知
  |  未知
  |  未知


- docker exec -it drv_node bash
- source devel/setup.bash


### [rostopic echo /novatel_data/inspvax]

输出示例：

![inspvax](imgs/gps_ins/inspvax_sample_output.png "inspvax")

重点关注ins_status和position_type

参见文件novatel_msgs/INSPVAX.msg，可以看到
## ros topics
ins_status
----
value | ASCII | ins_status | definition | description
----|----|----|----|----
3  |  INS_SOLUTION_GOOD  |  SOLUTION_GOOD  |  uint32 INS_STATUS_SOLUTION_GOOD=3  |  The INS filter is in navigation mode and the INS solution is good.

position_type
----
value | ASCII | position_type | definition | description
----|----|----|----|---
54  |  INS_PSRDIFF  |  PSEUDORANGE_DIFFERENTIAL  |  uint32 POSITION_TYPE_PSEUDORANGE_DIFFERENTIAL=54  |  INS pseudorange differential solution
56  |  INS_RTKFIXED  |  RTK_FIXED  |  uint32 POSITION_TYPE_RTK_FIXED=56  |  INS RTK fixed ambiguities solution


## commands on pp7

通过以下指令登录到pp7，
- telnet 192.168.8.60 3004

### log insconfig

输出示例：

![log insconfig](imgs/gps_ins/log_insconfig_output.png "log insconfig")

平移量类型  |  含义
----|----
ANT1  |  从IMU中心到主天线相位中心的平移量
ANT2  |  从IMU中心到从天线相位中心的平移量

### setinstranslation

1. setinstranslation user 0 0 0 0 0 0        # 把ins输出设置到ins中心点




第二个问题：在roslaunch-XXX.log里，都出现了哪些"Added node of type"？

node | 备注
-----|-----
localization/gnss_ntrip_client_node  |  会有对应的gnss_ntrip_client_node.INFO log文件
localization/online_gnss_rtk_node  |  会有对应的online_gnss_rtk_node.INFO log文件
localization/sensor_fusion_node  |  会有对应的sensor_fusion_node.INFO log文件
localization/map_registration_node  |  会有对应的map_registration_node.INFO log文件



第三个问题：docker是啥？如何使用？



第四个问题：ros里面的tf到底指的是啥？