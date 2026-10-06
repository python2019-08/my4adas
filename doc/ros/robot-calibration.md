# ch1.机器人标定（ROS2 移动机器人，你的巡逻小车）
标定本质：**把模型理论值和真实物理机器人对齐**，消除误差，分大类：**底盘标定、传感器标定、TF/外参标定**，你现在做四轮/差分底盘+激光雷达，重点是下面几项。

> 适用：Jazzy / Humble，slam_toolbox、nav2、robot_localization 整套栈。

## 一、底盘标定（差分轮底盘，最关键，直接影响 odom）
> 你之前遇到 `/odom` 缺失、TF、建图漂移，**绝大多数根源是轮式里程计标定不准**
### 1. 轮距（track_width）标定
左右两轮中心距离，URDF / diff_drive_controller 参数。
- 错误：模型写0.5m，实际0.512m → 原地旋转时里程计角度漂移巨大
标定方法：
1. 原地旋转机器人 360°
2. 看里程计输出的旋转角度
3. 修正 track_width，直到转一圈 odom 角度刚好≈2π

### 2. 轮子半径（wheel_radius）标定
- 错误：轮子磨损、充气差异，理论半径和实际不一样 → 直线行走里程计距离不准
标定方法：
1. 让机器人直线走固定距离（例如5米）
2. 对比实际物理距离 和 odom输出距离
3. 修正 wheel_radius：`修正半径 = 实测距离 / odom读数 * 原半径`

> diff_drive_controller 这两个参数在 `ros2_control` yaml 里配置。

### 3. 电机编码器比例（encoder ticks per meter）
编码器脉冲数 → 位移换算系数。
很多单片机micro-ros小车这里很容易错，直接导致里程计完全不准。

## 二、激光雷达标定
### 1. 雷达外参标定（TF标定：laser_link 相对于 base_link）
就是雷达安装的 x,y,z,roll,pitch,yaw。
你之前的 `tf2_echo base_link laser_link` 就是看这个TF。
- 安装偏角yaw误差：地图整体倾斜、物体错位
- pitch/roll：地面点云抬高/压低

**标定工具：`camera_lidar_calibrator` / `lidar_calibration`；简易方法：**
1. 机器人正对一面平整白墙
2. 看RViz点云，墙的点云是否垂直，判断yaw/pitch偏移
3. 修改URDF里laser_joint的origin，直到对齐

> 重点：**外参标定完，写死在URDF/xacro，不要靠动态TF实时发布**。

### 2. 激光雷达内参（大部分成品雷达出厂已经标定，一般不用动）
测距偏移、角度补偿，买的商用雷达（RPLIDAR、SICK、禾赛等）出厂已标定。

## 三、IMU标定（如果你加IMU，用于robot_localization）
IMU标定是**必做**，否则融合里程计会飘得离谱：
1. **加速度计标定**：6面标定，把IMU6个面分别水平放置，消除bias
2. **陀螺仪标定**：静止采集，求零偏（gyro bias）
工具：`imu_calib` ROS2包
标定结果：bias参数填入 `robot_localization ekf.yaml`

> robot_localization EKF 融合 odom + imu，IMU零偏不准直接导致航向漂移。

## 四、相机标定（如果后面加视觉）
- 单目相机：内参（焦距、畸变系数），工具 `camera_calibration`
- 相机-激光联合标定：得到相机相对于雷达的外参

## 五、SLAM / EKF 相关标定（不是硬件标定，是参数调优）
1. robot_localization EKF：噪声参数 `process_noise_covariance`、测量噪声，属于参数调优，不是硬件标定
2. slam_toolbox：激光匹配阈值、搜索窗口，属于SLAM调参，不属于标定

## 标定顺序【推荐流程，照着一步步做】
1. ✅ 底盘：编码器 → wheel_radius → track_width（**优先做！里程计是根基**）
2. ✅ IMU标定（如果有IMU）
3. ✅ 激光雷达外参（TF，laser_link）
4. ✅ 测试：静止TF、直线行走、原地旋转，看odom是否符合物理运动
5. ✅ 再跑SLAM建图，此时漂移会大幅降低
6. ✅ 最后调 robot_localization EKF融合

## 常用ROS2标定工具清单
| 标定对象 | ROS2工具 |
|---|---|
| 差分轮底盘里程计 | 手工标定脚本 / `diff_drive_calibration` |
| IMU | `imu_calib` |
| 激光外参 | `lidar_calibrator` |
| 相机内参 | `camera_calibration` |
| 相机激光联合标定 | `camera_lidar_calibrator` |

## 常见踩坑（你这个巡逻小车场景）
1. 只改URDF模型数值，**没有物理标定**：仿真看着完美，真机建图疯狂漂移
2. 激光外参yaw差1~2度肉眼看不出，但长距离建图地图直接歪掉
3. IMU不标定直接进EKF：短时间航向就飘，城市道路巡检非常致命

---

你现在是**差分底盘 + 激光雷达**，要不要我直接给你一套**差分轮里程计标定完整步骤+采集数据的ros2命令**？
做完这个，你的 `/odom` 精度会明显提升，slam_toolbox建图漂移改善很大。