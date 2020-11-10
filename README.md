# GNSS


## Npos220

Npos与IPC正常的连接是：

![Npos IPC connection](imgs/npos/connection_between_Npos_and_IPC.jpg "Npos IPC connection")

Npos220区分4G主站com1 和 从站com1。

其中，4G主站com1如下，

主站COM1之前已通过命令 "serialconfig com1 115200"配置成了115200的波特率。

![master com1](imgs/npos/4g_master_com1.jpg "master com1")

如果在主站com1输入log comconfig，（也就是将主站com1连至IPC的COM2，然后打开/dev/ttyS1）是没有任何相应的。

由于对主站com1进行了输出的配置，如下，

```bash
log com1 gpgga ontime 1
log com1 gprmc ontime 1
```

于是，gpgga 和 gprmc均是以1s的周期在发送，如下，

!["mastercom1 gpgga"](imgs/npos/gpgga_output_at_1sec_interval.PNG "mastercom1 gpgga")

!["mastercom1 gpgga"](imgs/npos/gprmc_output_at_1sec_interval.PNG "mastercom1 gprmc")

从站com1如下：

![rovercom1](imgs/npos/rovercom1.jpg "rovercom1")

同时还有主站com3（连接至IPC的com1口，所以会被映射成/dev/ttyS0），如下，

![master com3](imgs/npos/mastercom3.jpg "master com3")

在/dev/ttyS0，输入log loglist，其输出如下，

![master com3](imgs/npos/mastercom3_loglist.PNG "master com3")

### 问题1，inspvaxa消息里ins的状态无法到ins_solution_good 和 ins_rtkfixed

ins_solution一直处于 ins_aligning, motion_detect（这是车动起来的标识）, waiting_azimuth（因为双天线的heading始终是0，所以这里显示waiting_azimuth是可以理解的）的状态，就是到不了ins_solution_good的状态。

position_type则一直是"PSRDIFF"的状态。

先静态，等状态变成"INS_ALIGNMENT_COMPLETE"，然后再绕‘8’字收敛到INS_SOLUTION_GOOD。

有一点需要注意：由于当时双天线的heading始终是0，所以就不能用双天线初始化，需要用自动方式。也就是需要运行alignmentmode automatic。

### 问题2，双天线的heading始终为0

![heading3A is 0](imgs/npos/dualantenna_heading_is_always_0.png "heading3A is 0")

当遇到这个问题的时候，首先要用从站com1替换主站com1，连接到IPC的com2口，然后输以下3个命令来查看从站com1的两个方面的信息，

第一方面，查看授权码

当把从站com1接到工控机的COM2后，其会被映射成/dev/ttyS1，波特率为9600，需要对其授权方可以使用，可以通过命令 log authcodes来查看授权码，

当没有进行授权时，查看到的授权码如下，

![no authcodes for rovercom1](imgs/npos/no_authcodes_for_rovercom1.png "no authcodes for rovercom1")

将从站com1连至IPC的COM2后，使用sudo cutecom打开/dev/ttyS1，波特率选择9600，使用以下命令对从站com1进行授权，

```bash
AUTH ZHT63F,G4NT9R,FTFM93,HZ5RGN,NT8TKT,CDNLZN5NN
```

完成授权后，再次通过log authcodes查看授权码，

![auth finish for rovercom1](imgs/npos/rovercom1_auth_finish.png "auth finish for rovercom1")

成功给从站com1授权后，就有了双天线的heading信息了。此时需要把主站com1口再接回到IPC的COM2口，也就是恢复Npos和IPC的正常连接。

第二方面，通过命令 log comconfig查看com口的配置

首先，通过以下指令来设置主站(masterport) 和 从站(roverport)之间的align。

```bash
insalignconfig com2 com2 230400 1    ## 主站和从站之间的通信波特率是230400，同时主站与从站之间的ALIGN结果以1HZ频率输出
```

其次，通过命令 log comconfig查看com口的配置

![comconfig for rovercom1](imgs/npos/rovercom1_log_comconfig.png "comconfig for rovercom1")

由上图可知，COM2的波特率设置成了230400，发送以下指令

```bash
log com2 headingext2b onnew
saveconfig
```

上面这行指令将rtk从站的位置信息提供给了rtk主站，rtk主站拿到从站的信息后，可以产生ins和rtk之间的align结果，详细的解释如下，

![headingext2](imgs/npos/headingext2.PNG "headingext2")

在日本public road项目里，由于没有从站com1口的授权码，所以在输入 log com2 headingext2b onnew指令之后，仍然没有双天线heading的数据。






# 坐标系

## PP7坐标系  /  ins坐标系

![PP7 coordinate](imgs/gps_ins/pp7_coordinates/pp7_coord.jpg "PP7 coordinate")

PP7坐标系已经画在设备上了，x向右，y向前，z向上。关于原点，其实原点肯定只有一个，在设备里面。上图中，其实也清晰地画出了imu x/y/z的原点。不管怎么安装pp7，x指的方向就是pp7 imu的x轴，y指的方向就是pp7 imu的y轴。不同的安装角度，影响的是外参，影响的是pp7和lidar坐标系的旋转角度，以及pp7和车体坐标系的旋转角度。

当在POD4上安装PP7 和 Lidar-32，用作地图车时，PP7的坐标系是: x向右，y向前，z向上。

当在Hunter上安装PP7 和 Lidar-32，用作地图车时，PP7的坐标系是: x向右，y向前，z向上。

## map坐标系

frame_id: /map

## Lidar坐标系

frame_id: /velodyne_32 或者 /velodyne_16_F

## 车体坐标系

frame_id: /base_link

## tf

### lidar-imu

lidar.cfg定义了Tx_vehicle_lidar，lidar坐标系在vehicle坐标系下的位姿。

config_imu.cfg定义了Tx_vehicle_imu，imu坐标系在vehicle坐标系下的位姿。

Tx_imu_lidar = [Tx_vehicle_imu]^T * Tx_vehicle_lidar.

# 标定

## 地图车标定  /  Lidar-32 + PP7D

### (ins 和 rtk 的标定)  /  GNSS/INS Calibration  /  NovAtel Calibration(ANT1, ALIGN, RBV)

#### 1. 双天线到ins杆臂值标定

需要登录到NovAtel，有以下两种方法都可以登录到NovAtel终端，

- telnet 192.168.8.60 3004
- 在浏览器中输入"192.168.8.60"，点击右上角的设置图标，打开浏览器版的终端

所有车辆的杆臂值采用手动测量。

```bash
# set level-arm and std value for dual-ant
# tx/ty/tz的单位是meter，std_tx/std_ty/std_tz为标准差，需要控制在0.03 ~ 0.05m。
setinstranslation ant1 tx ty tz std_tx std_ty std_tz （左天线杆臂值，默认为主天线）
setinstranslation ant2 tx ty tz std_tx std_ty std_tz  (右天线杆臂值)
saveconfig
这些数据会被写入到NovAtel存储器里，也就是NVM。

在<OEM7_Commands_Logs_Manual.pdf>，'4.7 INSCALIBRATE'，'SDThreshold'，有这样一句话"default Standard Deviation Threshold for lever arm calibration = 0.1m"。

比如，在POD4上，

setinstranslation ant1 -0.15 0.45 1.22 0.10 0.10 0.10  # 这条语句表明，主天线在PP7坐标系下的位置是：左边0.15m，前边0.45m，上方1.22m，x/y/z三个方向上允许的误差范围均是0.1m。

setinstranslation ant2 1.05 0.45 1.22 0.10 0.10 0.10  # 这条语句表明，从天线在PP7坐标系下的位置是：右边1.05m，前方0.45m，上方1.22m，x/y/z三个方向上允许的误差范围均是0.1m。

在Hunter上，

setinstranslation ant1 -0.26 -0.04 0.115 0.05 0.05 0.05  # 这条语句表明，Hunter小车上，主天线在PP7坐标系下的位置：-0.26m, -0.04m, 0.115m。x/y/z三个方向上允许的误差范围是0.05m。
setinstranslation ant2 -0.26 0.88 0.23 0.05 0.05 0.05  # 这条语句表明，Hunter小车上，从天线在PP7坐标系下的位置：-0.26m, 0.88m, 0.23m。x/y/z三个方向上允许的误差范围是0.05m。
```

#### 2. 设置输出原点(output origin)，也就是user输出为ins位置

```bash
setinstranslation user 0 0 0 0 0 0  # 这条语句应该会打印 OK message
setinsrotation user 0 0 0 0 0 0  # 这条语句应该会打印 OK message
saveconfig
```

#### 3. ALIGN标定（ins 和 rtk之间的标定）

ALIGN标定的结果是写在了 _config_gnss.cfg_

NovAtel操作key: 在开阔地段启动，在尽量开阔的地方绕行，能够缩短收敛时间。


标定过程可在车体静止下完成，也可以将车体绕8字完成。建议先将车体绕8字绕个1分钟左右，之后静止直至达到收敛条件。

做ALIGN标定的前提条件： /novatel_data/inspvax的"position_type" = 56(RTK_FIXED)，同时"ins_status" = 3(SOLUTION_GOOD)，通常通过将车绕8字来达到此状态。

登录到NovAtel终端，telnet 192.168.8.60 3004 或者登录网页版，启动ALIGN标定，

```bash
# std是用来设置收敛条件的，单位是degree，建议设为0.1 degree，更小的std较难收敛。
inscalibrate align new 0.1
log inscalstatus onchanged  # 查看标定状态更新
saveconfig  # 将标定结果写入NVM存储器
```

当发现"9. Source Status"由 CALIBRATING 变成 CALIBRATED，就表明align的标定过程已经完成。

Source Status
----
ASCII  |  Description
----|----
CALIBRATING  |  offset values是在标定过程中给出的
CALIBRATED  |  offset values是在标定过程完成后给出的
INS_CONVERGING  |  offset values就是初始的输入值。此时，calibration过程还没开始，ins solution还在收敛的过程中，直到ins solution是converged，calibration过程才真正开始


![align calibrate](imgs/gps_ins/align_calibrate_complete.png "align calibrate")

做"inscalibrate align new 0.1"之前的insconfig如下：

![insconfig before align calib](imgs/gps_ins/insconfig_before_align_calibrate.png "insconfig before align calib")

完成"inscalibrate align new 0.1"之后的insconfig如下：

![insconfig after align calib](imgs/gps_ins/insconfig_after_align_calibrate.png "insconfig after align calib")

可以看到，ALIGN IMUBODY的结果由"0 0 -90 9.2356 0 6.9952 FROM_DUAL_ANT" 变成了 "0 0 -73.7647 45 45 0.0999 CALIBRATED"

运行inscalibrate align new 0.1，依次达到CALIBRATED时输出，

- （第一次）0  0  -73.7647 45 45 0.0999 CALIBRATED
- （第二次）0  0  -73.6808 45 45 0.2789 CALIBRATED
- （第三次）0  0  -82.4248 45 45 0.1614 CALIBRATED
- （第四次）0 -0  -90.2700 45 45 0.0897 CALIBRATED
- （第五次）0 -0  -90.4366 45 45 0.0927 CALIBRATED

到第5次时，就认为已经完成了inscalibrate align new 0.1这个标定过程。

如果很难收敛到设定角度，达到一定角度时，可以敲命令 inscalibrate align stop，此时屏幕打印的align标定的状态会由Calibrating变为Calibrated，如果满足要求的话，此时同样可以认为完成了inscalibrate align。

**注意**：屏幕打印由Calibrating变成Calibrated，只能说明完成了ALIGN标定这个过程，但是标定结果是否能用，可以从以下两个方面来考量。

其一，观察绕z轴的旋转角度，对于POD4，目前看到的这个角度应该是接近-90deg;而对于HUNTER，这个角度应该是接近0deg。对于POD4，如果由Calibrating变为Calibrated时，这个角度是-73.7647deg，与-90deg参考值相差较大时，则说明这一次Align标定虽然完成了，但结果并不理想，还需要继续运行 inscalibrate align new 0.1直至绕z轴的旋转角度接近-90deg为止。

![z rotation](imgs/gps_ins/z_rotation_on_POD4_and_HUNTER.jpg "z rotation")

其二，查看dualantenna heading与inspvax的azimuth是否一致，如果相差较大，同样说明结果不满足要求，还需要继续执行 inscalibrate align new 0.1直至这两个角度比较一致为止。此外，还有一种查看角度值的方法，就是先把车摆向正北方向（用手机指南针做参考），查看此时dualantenna heading与inspvax的azimuth是否是接近于0deg或者360deg。

关于如何查看dualantenna heading，可以在NovAtel terminal里，输入log dualantennaheading once来查看，也可以在IPC上通过rostopic echo /novatel_data/dualantennaheading来查看。

### RBV标定（ins 和 vehicle之间的标定）

RBV标定结果是写在了 _config_imu.cfg_

RBV标定其实是在标定imu的y轴与车体前进方向之间的夹角, for IMU angle offset from vehicle body

做RBV标定的前提条件： /novatel_data/inspvax的"position_type" = 56(RTK_FIXED)，同时"ins_status" = 3(SOLUTION_GOOD)，通常通过将车绕8字来达到此状态。

做RBV标定的要求：
- 标定过程需要在一条300 - 500米平坦道路进行，行驶过程中避免方向变化
- 最好道路可以逆行，从一端开始，驶向另一端，如果到终点时已经收敛到目标角度那最好了，但是如果到终点时还没有收敛到目标角度，那么应该先运行inscalibrate rbv stop手动终止，掉头后，运行inscalibrate rbv add继续标定

登录到NovAtel终端，telnet 192.168.8.60 3004 或者登录网页版，启动RBV标定，

```bash
# std是用来设置收敛条件的，单位是degree，建议设为0.05 degree，更小的std较难收敛。
inscalibrate rbv new 0.05
log inscalstatus onchanged  # 查看标定状态更新
inscalibrate rbv stop  # 手动停止标定，此时屏幕打印的标定状态应该由Calibrating 变成Calibrated。
inscalibrate rbv add  # 继续标定
saveconfig  # 将标定结果写入存储器
在<OEM7_Commands_Logs_Manual.pdf>，'4.7 INSCALIBRATE'，'SDThreshold'，有这样一句话"default Standard Deviation Threshold for RBV calibration = 0.5degrees"。
```

当输入 inscalibrate rbv stop时，屏幕上的打印由

RBV 2.5550 0.6705 -0.3149 0.3036 2.1844 0.2195 CALIBRATING 1

RBV 2.5550 0.6705 -0.3149 0.3036 2.1844 0.2195 INSUFFICIENT_SPEED 1

变成了

RBV 2.5550 0.6705 -0.3149 0.3036 2.1844 0.2195 CALIBRATED 1

既输入 inscalibrate rbv stop后，输入 inscalibrate rbv add，继续标定

再次输入 inscalibrate rbv stop，屏幕打印由

RBV 2.3906 1.0029 -0.2068 0.4063 2.6315 0.3284 CALIBRATING 2

RBV 2.3906 1.0029 -0.2068 0.4063 2.6315 0.3284 INSUFFICIENT_SPEED 2

变成了

RBV 2.4728 0.8367 -0.2609 0.2536 1.7100 0.1975 CALIBRATED 2

再次输入 inscalibrate rbv add，继续标定，

第三次输入inscalibrate rbv stop，屏幕打印由

RBV 2.4727 0.3667 -0.0015 0.4285 1.9880 0.3114 CALIBRATING 3

RBV 2.4727 0.3667 -0.0015 0.4285 1.9880 0.3114 INSUFFICIENT_SPEED 3

变成了

RBV 2.4728 0.6800 -0.1744 0.2213 1.3186 0.1677 CALIBRATED 3

第三次输入 inscalibrate rbv add，继续标定，同时

RBV 2.4753 0.5785 -0.2113 0.1724 1.0369 0.1292 CALIBRATED 4

此时已经可以认为完成了rbv校准，前后的insconfig对比如下，

执行rbv calibrate之前，

![before rbv](imgs/gps_ins/insconfig_before_rbv_calibrate.png "before rbv")

执行rbv calibrate之后，

![after rbv](imgs/gps_ins/insconfig_after_rbv_calibrate1.png "after rbv")

可以看到rbv结果已经由

RBV IMUBODY 0 0 0 0 0 0 FROM_NVM

变成了

RBV IMUBODY 2.4753 0.5785 -0.2113 0.1724 1.0369 0.1292 CALIBRATED

### Lidar和IMU的外参标定 / Lidar - INS

注意事项：
- 直接把那些config文件都放在/opt/allride/data/localization/config下，不要有P001或者H001这样的子文件夹.

错误的放置方法：
![No subdirectory](imgs/ins-vehicle/no_need_subdirectory_Hunter001.png "No subdirectory")

正确的放置方法：
![under config directly](imgs/ins-vehicle/put_carconfig_under_config_directly.png "under config directly")

遇到的问题有：

#### 1. lidar odom没有运行，result目录 和 lidar_odom.log都是空的

![odom not running](imgs/ins-vehicle/lidar_odom_not_running.png "odom not running")

通过tail -f lidar_odom.log深入进去看，发现了一下错误：

"Unable to contact my own server at [http://192.168.8.100:37161]. This usually means that the network is not configured properly. A common cause is that the machine cannot ping itself."

![odom network err](imgs/ins-vehicle/lidar_odom_network_err.png "odom network err")

此时，ifconfig查看发现根本就不存在 192.168.8.100这样一个ip，如下：

![check ifconfig](imgs/ins-vehicle/check_ifconfig.png "check ifconfig")

需要将ROS_HOSTNAME=192.168.8.100改成ROS_HOSTNAME=localhost。

继续运行，碰到报错，"ProtoIO: failed to open file(RD): /opt/allride/data/localization/config/map_layer_dictionary.cfg"，该问题就是由于将config文件放到了Hunter001这样的子文件夹下了，而不是直接放到了/opt/allride/data/localization/config目录下造成的。

解决了上面2个问题，就可以顺利地走完./start_calibration.sh的pipeline，当脚本运行完成后，会在/opt/allride/data/calibration/result目录下生成以下4个文件，

![Lidar-ins result](imgs/ins-lidar/calibration_result.png "Lidar-ins result")

我们需要用代表标定结果的这4个文件替换原来的/opt/allride/data/calibration/config下同样名字的4个文件。至此，地图车的标定结束。

对于所有/opt/allride/data/calibration/config目录下的文件，应给一个版本号，例如0.0.438，将其上传到 oss://allride-release/carconfig相关的目录下。

H001的config目录： oss://allride-release/carconfig/demo/hunter/0.0.438/H001

P001的config目录： oss://allride-release/carconfig/demo/perceptin_setup/0.0.388/P001

P12的config目录： oss://allride-release/carconfig/demo/perceptin_setup/0.0.388/P12

看下H001 IPC上的product/version.ini，如下，

![IPC map version](imgs/mapping_pipeline/version.ini.png "IPC map version")

carconfig_branch=demo_hunter 和 carconfig_version=0.0.438，在结合/etc/profile "export CAR_NAME=H001"，把上面3个条件组合到一起，就表明H001这台车用的carconfig在oss服务器上的路径是 oss://allride-release/carconfig/demo/hunter/0.0.438/H001。


## 普通车标定  /  Lidar-16 + oem718d

因为没有ins，可以基于地图来做标定，会更加的简单。


# 制图

遇到的问题有：

## 1. 提示"REQUIRED process [multi_sensor_odometry_node-2] has died!", "Initiating shutdown"

如下图所示，当运行./start_ndt_mapping.sh后，在处理玩mapping configures后，就会去执行"Processing lidar odometry..."，当要执行完成"lidar odometry"时，通过tail -f /opt/allride/data/mapping/data/loc_mapping_2020-10-24-12-28-12/lidar_odometry.log可以看到如下的错误提示（红色部分），

![running lidar odom](imgs/mapping_pipeline/mapping_pipeline_processing_lidar_odom.png "running lidar odom")

![finish lidar odom](imgs/mapping_pipeline/finish_lidar_odom_tips.png "finish lidar odom")

备注："REQUIRED process [multi_sensor_odometry_node-2] has died!", "Initiating shutdown"是正常退出lidar odom的提示。

对于IPC而言，地图和config一样都可以从oss上pull下来。

对于跑完mapping pipeline生成的两个*.mdb(data.mdb 和 lock.mdb)，应该给一个版本号并将其上传到 oss://allride-release/localization目录下。(Generally we tag map data with a version code and upload it to oss://allride-release/localization/, for version control and automatically setup).

![localcation mdb put](imgs/mapping_pipeline/location_mdb_files_put.png "localcation mdb put")

如下图所示，2.0.x代表深圳的地图，具体2.0.3就是大学城益田假日里背面停车场于2020-10-24采集的地图bag所建的地图。

![oss map version](imgs/mapping_pipeline/map_version.png "oss map version")

看下H001 IPC上的product/version.ini，如下，

![IPC map version](imgs/mapping_pipeline/version.ini.png "IPC map version")

上图中version.ini中的loc_version=2.0.3用的就是oss://allride-release/localization/2.0.3。


# ros topics


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

## [rostopic echo /novatel_data/bestpos]

输出示例：

![bestpos](imgs/gps_ins/bestpos_sample_output.png "bestpos")

参见文件novatel_msgs/BESTPOS.msg，

solution_status
----
value | ASCII | solution_status | definition | description
----|----|----|----|----
0  |  SOL_COMPUTED  |  SOL_COMPUTED  |  uint32 SOLUTION_STATUS_SOLUTION_COMPUTED=0  |  Solution computed

position_type
----
value | ASCII | position_type | definition | description
----|----|----|----|----
49  |  WIDE_INT  |  WIDE_INT  |  uint32 POSITION_TYPE_WIDE_INT=49  |  Multi-frequency RTK solution with carrier phase ambiguities resolved to wide-lane integers
50  |  NARROW_INT  |  NARROW_INT  |  uint32 POSITION_TYPE_NARROW_INT=50  |  Multi-frequency RTK solution with carrier phase ambiguities resolved to narrow-lane integers，RTK位置精度是在厘米级别，实际例子见bestpos_positiontype50实际例子1，latitude_std: 0.011618m，ongitude_std: 0.013968m，厘米级别

bestpos_positiontype50实际例子1：

![bestpos_positiontype50](imgs/commands/bestpos_positiontype50_sample.jpg "bestpos_positiontype50")


## [rostopic echo /novatel_data/inspvax]

输出示例：

![inspvax](imgs/gps_ins/inspvax_sample_output.png "inspvax")

重点关注ins_status和position_type

参见文件novatel_msgs/INSPVAX.msg，可以看到

ins_status
----
value | ASCII | ins_status | definition | description
----|----|----|----|----
3  |  INS_SOLUTION_GOOD  |  SOLUTION_GOOD  |  uint32 INS_STATUS_SOLUTION_GOOD=3  |  The INS filter is in navigation mode and the INS solution is good.
7  |  INS_ALIGNMENT_COMPLETE  |  ALIGNMENT_COMPLETE  |  uint32 INS_STATUS_ALIGNMENT_COMPLETE = 7  |  The INS filter is in navigation mode, but not enough vehicle dynamics have been experienced for the system to be within the specifications.        20201023测试        上午在众冠时代广场楼下完成"inscalibrate align new 0.1"后，下午再去在众冠时代广场楼下打算做"inscalibrate rbv new 0.5"，就一直出现ins_status = 7的状态，之后将车开出广场外的人行道（两边有树遮挡），ins_status一直是7，当开到小河边后ins_status变成了3，因不被允许在小河边测试，后又开回到人行道上，在人行道上空旷区域绕八字后可以进入ins_status=3的状态。


position_type
----
value | ASCII | position_type | definition | description
----|----|----|----|---
53  |  INS_PSRSP  |  PSEUDORANGE_SINGLE_POINT  |  uint32 POSITION_TYPE_PSEUDORANGE_SINGLE_POINT=53  |  single point
54  |  INS_PSRDIFF  |  PSEUDORANGE_DIFFERENTIAL  |  uint32 POSITION_TYPE_PSEUDORANGE_DIFFERENTIAL=54  |  INS pseudorange differential solution（伪距差分），实际例子见inspvax_position_type54实际例子1，latitude_std: 0.323787m，分米级别
55  |  INS_RTKFLOAT  |  RTK_FLOAT  |  uint32 POSITION_TYPE_RTK_FLOAT=55  |  floating ambiguity RTK solution，可能是L1_FLOAT或者NARROW_FLOAT solution，RTK std是在分米级
56  |  INS_RTKFIXED  |  RTK_FIXED  |  uint32 POSITION_TYPE_RTK_FIXED=56  |  INS RTK fixed ambiguities solution，可能是L1_INT，WIDE_INT或者NARROW_INT solution，RTK std是在厘米级，RTK位置精度在厘米级

inspvax_position_type54实际例子1：

![inspvax_positiontype54](imgs/commands/inspvax_positiontype54_sample.jpg "inspvax_positiontype54")

# commands on pp7

通过以下指令登录到pp7，
- telnet 192.168.8.60 3004

## log insconfig

输出示例：

![log insconfig](imgs/gps_ins/log_insconfig_output.png "log insconfig")

平移量类型  |  含义
----|----
ANT1  |  从IMU中心到主天线相位中心的平移量
ANT2  |  从IMU中心到从天线相位中心的平移量

## setinstranslation

1. setinstranslation user 0 0 0 0 0 0        # 把ins输出设置到ins中心点




第二个问题：在roslaunch-XXX.log里，都出现了哪些"Added node of type"？

node | 备注
-----|-----
localization/gnss_ntrip_client_node  |  会有对应的gnss_ntrip_client_node.INFO log文件
localization/online_gnss_rtk_node  |  会有对应的online_gnss_rtk_node.INFO log文件
localization/sensor_fusion_node  |  会有对应的sensor_fusion_node.INFO log文件
localization/map_registration_node  |  会有对应的map_registration_node.INFO log文件


## insalignconfig masterport [roverport] [baudrate] [outputrate]

实际例子：

insalignconfig com2 com2 230400 1



第三个问题：docker是啥？如何使用？



第四个问题：ros里面的tf到底指的是啥？