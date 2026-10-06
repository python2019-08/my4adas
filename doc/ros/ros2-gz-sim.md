# ch01.✅  ros-humble[gazebo_ros_pkgs] <=> ros-jazzy[ros_gz]  
 
> 
> Humble 里：
> 
> 
> - `gazebo_ros_pkgs` → Gazebo Classic (gazebo11)，插件：`gazebo_ros_diff_drive` / `gazebo_ros_laser` / `gazebo_ros_imu`
> 
> 
> Jazzy 里：
> 
> 
> - **`ros_gz`** = 替代 `gazebo_ros_pkgs` 的元包，配套 Gazebo Harmonic（`gz sim`）

## ros_gz 包含的子包（对应原来 gazebo_ros_pkgs 的能力）

| 包名                 | 作用 |
| ------------------- | --- |
| `ros_gz_sim`        | 提供启动gz sim的launch、工具，替代`gazebo_ros`的启动部分 |
| `ros_gz_bridge`     | ROS ↔ Gazebo Transport消息双向桥（最重要！） |
| `ros_gz_interfaces` | 消息/服务定义 |
| `ros_gz_image`      | 图像传输桥 |
| `ros_gz_sim_demos`  | 示例world、模型、launch |

## 安装命令（Jazzy）

```sh
# 一键安装全套 ros_gz
sudo apt install ros-jazzy-ros-gz
```

单独装核心组件：
```sh
sudo apt install ros-jazzy-ros-gz-sim ros-jazzy-ros-gz-bridge ros-jazzy-ros-gz-interfaces
```

## ⚠️ 重大变化（对你的差速底盘Xacro最关键）

1. **插件名字完全变了！**
 Humble Classic：
```xml
<plugin name="gazebo_ros_diff_drive" filename="libgazebo_ros_diff_drive.so"/>
```

 Jazzy + Gazebo Harmonic：**不再是 gazebo_ros 系列插件**
   - 差速驱动、激光、IMU 用 **Gazebo Sim System Plugin**，写在 `<plugin>`标签 
   - 常用：`gz::sim::systems::DiffDrive`、`gz::sim::systems::Lidar`、`gz::sim::systems::Imu`
   - 消息靠 `ros_gz_bridge` 把gz transport消息转发成ROS2 `/scan`、`/odom`、`/imu`
2. 启动命令：
   - Humble Classic：`ros2 launch gazebo_ros gazebo.launch.py`
   - Jazzy：`ros2 launch ros_gz_sim gz_sim.launch.py gz_args:=xxx.world`

## 额外：ros2_control 方案（推荐，更现代）

你做差速底盘，Jazzy 更推荐用 `gz_ros2_control`，在Gazebo里接入ros2_control控制器：

```sh
sudo apt install ros-jazzy-gz-ros2-control
```

这个方案可以直接用 `diff_drive_controller`，不用手动配置ros_gz_bridge转发odom。

## 迁移要点小结

- ❌ Jazzy 不要继续用 `gazebo_ros_diff_drive` / `gazebo_ros_laser`，**在Gazebo Harmonic里面不存在**
- ✅ 两种路线二选一：
  1. 原生Gazebo Sim系统插件 + ros_gz_bridge（轻量，适合简单机器人）
  2. gz_ros2_control + ros2_control控制器（工程首选，和真实硬件复用控制器代码）

要不要我给你一份适配Jazzy+Gazebo Harmonic的完整xacro片段，包含DiffDrive、Lidar、IMU的system插件写法？
=====================================

# ch02.SDF（SDFormat，Simulation Description Format）

**SDF 是基于XML的仿真描述文件，是 Gazebo Harmonic（Ignition Gazebo）原生格式**，用来完整描述整个仿真世界、机器人模型、物理引擎、传感器插件、光照、地形等。

> 
> 一句话区分：
> **URDF = ROS 描述机器人本体（运动学）；SDF = Gazebo 描述整个仿真世界（物理+传感器+场景）**

## 1. SDF能干什么（对你Autoware巡逻机器人仿真）

SDF文件可以写：

1. `<world>`：仿真世界，地面、墙壁、障碍物、光照、物理引擎参数（重力、步长）
2. `<model>`：机器人模型/静态物体（底盘、轮子）
3. `<link>`：刚体，包含碰撞体、可视化模型、惯性参数
4. `<joint>`：关节（轮子旋转关节）
5. `<sensor>`：**LiDAR、IMU、相机**（Gazebo Harmonic的传感器插件全部写在SDF里）
6. `<plugin>`：Gazebo插件，实现传感器数据输出、ros_gz桥接、底盘控制

> 
> SDF不止描述机器人，还能描述**整个仿真场景**，这是URDF做不到的。URDF只能描述单个机器人的连杆关节，无法定义世界、地形、光照。

## 2. SDF vs URDF（ROS2 Jazzy + Gazebo Harmonic重点）

|  | URDF | SDF |
| --- | --- | --- |
| 用途 | ROS标准，描述机器人运动学，RViz可视化 | Gazebo原生，描述仿真世界+物理+传感器插件 |
| 范围 | **只能单个机器人** | 整个仿真世界 + 多个模型 |
| 物理/传感器 | 物理参数弱，传感器插件在Gazebo Classic | 原生支持LiDAR/IMU、摩擦、物理引擎、插件 |
| Jazzy+Harmonic | 可以加载，但URDF里的`gazebo_ros`插件失效，要改用`ros_gz` | 推荐原生格式，Harmonic原生支持 |
| 文件后缀 | `.urdf` / `.xacro` | `.sdf` |

> 
> 你在Jazzy+Gazebo Harmonic里依然可以继续用URDF写机器人，`ros_gz`会自动内部转成SDF；但传感器、物理插件推荐放到SDF里。

## 3. SDF 最简结构示例（snippet）

```
<?xml version="1.0"?>
<sdf version="1.11">
  <world name="patrol_world">
    <!-- 物理引擎 -->
    <physics name="default_physics" type="ode">
      <max_step_size>0.001</max_step_size>
    </physics>
    <!-- 地面 -->
    <model name="ground_plane">
      <static>true</static>
      <link name="ground">
        <collision name="collision">
          <geometry><plane><size>100 100</size></plane></geometry>
        </collision>
      </link>
    </model>
    <!-- 巡逻机器人模型 -->
    <model name="patrol_robot">
      <link name="base_link">
        <pose>0 0 0.2 0 0 0</pose>
        <!-- 激光雷达传感器 -->
        <sensor name="lidar" type="lidar">
          <plugin filename="libgz-sim-lidar-system.so" name="gz::sim::systems::Lidar"/>
        </sensor>
      </link>
    </model>
  </world>
</sdf>
```

## 4. 和你Autoware Jazzy仿真的关联

1. Autoware Jazzy仿真环境用 **Gazebo Harmonic**，底层读取SDF；
2. 场景、建筑物、静态障碍物一般写在`.sdf world`文件；
3. 机器人本体可以二选一：
   - 方案A：URDF + ros_gz插件（老习惯，适合快速迁移旧巡逻机器人模型）
   - 方案B：纯SDF模型（Harmonic原生，传感器配置更稳定）
4. `ros_gz_bridge`负责把Gazebo里SDF定义的LiDAR、IMU数据转发成ROS2话题，送入Autoware定位模块。

## 5. 常见误区

- ❌ SDF不是URDF替代品：**URDF依然是ROS2机器人运动学标准，RViz读URDF**；Gazebo仿真器读SDF；两者各司其职。
- ❌ Gazebo Harmonic不再认老版`gazebo_ros` URDF插件，插件语法全部改成`ros_gz`，这就是之前说Classic迁移坑点。
- ✅ 可以用`sdformat`库工具，在命令行做URDF ↔ SDF转换。

如果你需要，我可以直接给一份**可运行的巡逻机器人SDF完整world文件**，包含差速底盘+16线激光雷达+IMU，直接用`gz sim`启动，对接Autoware定位。

要不要？

=====================================
# ch03.`gz sim -h`
```sh
$ gz sim -h
Run and manage Gazebo simulations.                                              
                                                                                
  gz sim [options] [file]                                                       
                                                                                
                                                                                
Available Options:                                                              
  -g                           Run only the GUI.                                

  --initial-sim-time [arg]     Initial simulation time, in seconds.             

  --iterations [arg]           Number of iterations to execute.                 

  --levels                     Use the level system. The default is false,      
                               which loads all models. It's always true         
                               with --network-role.                             

  --network-role [arg]         Participant role used in a distributed           
                               simulation environment. Role is one of           
                               [primary, secondary]. It implies --levels.       

  --network-secondaries [arg]  Number of secondary participants expected        
                               to join a distributed simulation                 
                               environment. (Primary only).                     

  --record                     Use logging system to record states and          
                               console messages to the default location,        
                               in ~/.gz/sim/log.                       

  --record-path [arg]          Implicitly invokes --record, and specifies       
                               custom path to put recorded files. Argument      
                               is path to record states and console             
                               messages. Specifying this argument will          
                               enable console logging to a console.log          
                               file in the specified path.                      

  --record-resources           Implicitly invokes --record, and records         
                               meshes and material files, in addition to        
                               states and console messages.                     

  --record-topic [arg]         Specify the name of an additional topic to       
                               record. Implicitly invokes --record.             
                               Zero or more topics can be specified by          
                               using multiple --record-topic options.           
                               Regular expressions can be used, which           
                               likely requires quotes. A default set of         
                               topics are also recorded, which support          
                               simulation state playback. Enable debug          
                               console output with the -v 4 option              
                               and look for 'Recording default topic' in        
                               order to determine the default set of            
                               topics.                                          
                               Examples:                                        
                                 1. Record all topics.                          
                                     --record-topic ".*"                      
                                 2. Record only the /stats topic.               
                                     --record-topic /stats                      
                                 3. Record the /stats and /clock topics.        
                                     --record-topic /stats                     
                                     --record-topic /clock                      

  --record-period [arg]        Specify the time period (seconds) between        
                               state recording.                                 

  --log-overwrite              When recording, overwrite existing files.        
                               Only valid if recording is enabled.              

  --log-compress               When recording, compress final log files.        
                               Only valid if recording is enabled.              

  --seed [arg]                 Pass a custom seed value to the random           
                               number generator.                                

  --playback [arg]             Use logging system to play back states.          
                               Argument is path to recorded states.             

  --headless-rendering         Run rendering in headless mode                   

  -r                           Run simulation on start.                         

  -s                           Run only the server (headless mode). This        
                               overrides -g, if it is also present.             

  -v [ --verbose ] [arg]       Adjust the level of console output (0~4).        
                               The default verbosity is 1, use -v without       
                               arguments for level 3.                           

  --gui-config [arg]           Gazebo GUI configuration file to load.           
                               If no config is given, the configuration in      
                               the SDF file is used. And if that's not          
                               provided, the default installed config is        
                               used.                                            

  --physics-engine [arg]       Gazebo Physics engine plugin to load.            
                               Gazebo will use DART by default.                 
                               (gz-physics-dartsim-plugin)                
                               Make sure custom plugins are in                  
                               GZ_SIM_PHYSICS_ENGINE_PATH.                      

  --render-engine [arg]        Gazebo Rendering engine plugin to load for       
                               both the server and the GUI. Gazebo will use     
                               OGRE2 by default. (ogre2)                        
                               Make sure custom plugins are in                  
                               GZ_SIM_RENDER_ENGINE_PATH.                       

  --render-engine-api-backend [arg]                                             
                               API to use for both the Server & GUI.            
                               Possible values for ogre2:                       
                                 - opengl (default)                             
                                 - vulkan (beta)                                
                                 - metal (Apple only, default for Apple)        
                               Note: If using Vulkan in the GUI and gz-gui      
                               was built against Qt < 5.15.2, it may be very    
                               slow.                                            

  --render-engine-gui [arg]    Gazebo Rendering engine plugin to load for       
                               the GUI. Gazebo will use OGRE2 by default.       
                               (ogre2)                                          
                               Make sure custom plugins are in                  
                               GZ_SIM_RENDER_ENGINE_PATH.                       

  --render-engine-gui-api-backend [arg]                                         
                               Same as --render-engine-api-backend but only     
                               for the GUI.                                     

  --render-engine-server [arg] Gazebo Rendering engine plugin to load for       
                               the server. Gazebo will use OGRE2 by default.    
                               (ogre2)                                          
                               Make sure custom plugins are in                  
                               GZ_SIM_RENDER_ENGINE_PATH.                       

  --render-engine-server-api-backend [arg]                                      
                               Same as --render-engine-api-backend but only     
                               for the server.                                  

  --version                    Print Gazebo version information.                

  -z [arg]                     Update rate in Hertz.                            

  -h [--help]                Print this help message.
                                                    
  --force-version <VERSION>  Use a specific library version.
                                                    
  --versions                 Show the available versions.

Environment variables:                                                          
  GZ_SIM_RESOURCE_PATH         Colon separated paths used to locate             
 resources such as worlds and models.                                         

  GZ_SIM_SYSTEM_PLUGIN_PATH    Colon separated paths used to                    
 locate system plugins.                                                       

  GZ_SIM_SERVER_CONFIG_PATH    Path to server configuration file.             

  GZ_GUI_PLUGIN_PATH           Colon separated paths used to locate GUI         
 plugins.                                                                       
  GZ_GUI_RESOURCE_PATH    Colon separated paths used to locate GUI              
 resources such as configuration files.                                        
```

## GZ_SIM_RESOURCE_PATH
```sh
export GZ_SIM_RESOURCE_PATH=~/a2/zdev/nv/adas-01/ros/_models:~/a2/zdev/nv/adas-01/ros/_tmp
```

##  https://gazebosim.org
 https://gazebosim.org/docs/harmonic/building_robot/

## Entiy tree 

==========================================
# ch04. 机器人建图流程

## sec.1 使用的命令 
```bash
# 命令1：启动自己编写的Nav2导航栈launch文件
# 包名：myfirst_robot；launch脚本：my_robot_nav2_launchv3_hourse.py
# 作用：拉起整套Nav2生命周期节点（map_server、amcl、planner、controller等导航模块）
# 一般还会启动RViz2，用于可视化机器人、地图、规划路径
ros2 launch  myfirst_robot  my_robot_nav2_launchv3_hourse.py
```
 
```bash
# 命令2：启动 slam_toolbox 在线异步SLAM建图launch文件
# 包名：slam_toolbox；脚本：online_async_launch.py（异步在线SLAM，适合边跑边建图）
# 参数 use_sim_time:=True：使用仿真时间，和Gazebo仿真时钟同步，仿真环境必须开启这个参数
# 作用：订阅激光雷达+TF数据，实时构建二维栅格地图，提供/slam_toolbox/save_map保存地图服务
ros2 launch  slam_toolbox  online_async_launch.py  use_sim_time:=True
```

```bash
# 命令3：调用slam_toolbox的保存地图服务，将当前建好的栅格地图写入磁盘
# 服务名：/slam_toolbox/save_map
# 服务接口类型：slam_toolbox/srv/SaveMap
# 请求体：name字段是std_msgs/String类型，嵌套data，传入【地图文件前缀】
# 最终生成：/home/abner/a2/zdev/nv/adas-01/ros/_tmp/mymap.pgm 和 mymap.yaml
ros2 service call /slam_toolbox/save_map slam_toolbox/srv/SaveMap "{name: {data: '/home/abner/a2/zdev/nv/adas-01/ros/_tmp/mymap'}}"
```

## 整套流程逻辑注释（整体串联）
1. 启动Nav2导航框架（规划、控制、定位相关节点）
2. 启动SlamToolbox在线异步SLAM，开始接收激光雷达数据**实时建图**，`use_sim_time`适配Gazebo仿真时间，保证时间戳对齐
3. 机器人在仿真环境内移动，扫描环境构建地图，地图在RViz实时预览
4. 环境扫描完成后，调用service保存地图，生成pgm栅格图片 + yaml地图描述文件

### 补充重要备注
> 1. 两个launch需要**分开两个终端**并行运行，不能在同一个终端先后执行（launch会阻塞终端）
> 2. `online_async` = 异步在线SLAM，优点：计算压力分散，机器人移动不会卡顿；适合你的巡逻机器人场景
> 3. 保存路径 `/home/abner/a2/zdev/nv/adas-01/ros/_tmp/` 目录必须**预先存在**，否则保存失败
> 4. 保存地图成功（返回`result=0`）之后，后续可以关掉slam_toolbox，用map_server加载静态地图，切换到AMCL静态定位模式，不再建图，只做导航。

如果你需要，我可以给你**静态地图加载版本**的启动命令（建好图之后，不再跑SLAM，直接加载mymap.yaml + AMCL定位）。



=======================================
# ch05. 两条命令对比：`map_saver_cli` vs slam_toolbox save_map service
## 一句话核心区别
- `map_saver_cli`：**Nav2 自带独立命令行工具**，订阅 `/map` 话题直接保存地图，**不依赖SLAM工具**，只要有`/map`话题就能存。
- `/slam_toolbox/save_map`：**slam_toolbox内部提供的服务**，调用SLAM算法后端直接导出地图，**只能在slam_toolbox节点运行时使用**。

## 逐条解析
### 1. ros2 run nav2_map_server map_saver_cli  -f room
```bash
# -f ：地图文件前缀（prefix）
ros2 run nav2_map_server map_saver_cli  -f room
```
- 原理：启动一个临时节点，订阅ROS话题 `/map`，拿到最新栅格地图，写到磁盘。
- 输出文件：`room.pgm` + `room.yaml`，保存在**当前终端所在目录**。
- 依赖条件：
  ✅ 系统中正在发布 `/map` 话题（可以来自slam_toolbox、cartographer、或静态map_server）
  ❌ **不需要slam_toolbox运行**。只要任何节点在发布`/map`都能用。
- 优点：
  - 通用性极强，Nav2生态标准工具，不管用哪种SLAM算法都可以保存地图
  - 简单，不需要记YAML嵌套语法，不容易写错
- 缺点：
  - 取的是**发布到ROS话题上的地图副本**，存在少量延迟。如果SLAM内部地图还没发布到`/map`，保存的会是旧地图。

> 额外参数：`--ros-args -p map_topic:=/custom_map` 可以指定非默认地图话题。

### 2. ros2 service call /slam_toolbox/save_map slam_toolbox/srv/SaveMap "{name: {data: '/home/abner/a2/zdev/nv/adas-01/ros/_tmp/mymap'}}"
```bash
# 调用slam_toolbox内置服务，直接从SLAM后端数据库导出地图
ros2 service call /slam_toolbox/srv/SaveMap "{name: {data: '/home/abner/a2/zdev/nv/adas-01/ros/_tmp/mymap'}}"
```
- 原理：通过ROS服务，**直接读取slam_toolbox后端的完整图数据库**生成地图，不是读`/map`话题。
- 输出：`mymap.pgm` + `mymap.yaml`，路径由你写的完整路径前缀决定。
- 依赖条件：
  ✅ slam_toolbox节点**必须正在运行**（online_async/online_sync）
  ❌ 关闭slam_toolbox之后，这个服务直接消失，无法调用。
- 优点：
  - 拿到SLAM后端**最新原始地图**，不受`/map`话题发布频率的延迟影响
  - 可以保存SLAM的位姿图，部分模式下还可以保存slam数据库文件（`.serial`）用于后续继续建图
- 缺点：
  - 绑定slam_toolbox，换Cartographer就不能用这个服务
  - 命令行需要写嵌套YAML，容易语法报错（你之前踩过这个坑）

## 对比表
|项目|map_saver_cli|slam_toolbox save_map service|
|---|---|---|
|来源|nav2_map_server（Nav2自带）|slam_toolbox内部服务|
|数据源|订阅`/map`话题|直接读取slam_toolbox后端地图数据库|
|是否依赖slam_toolbox|❌ 不依赖，只要有/map话题|✅ 必须运行slam_toolbox|
|路径控制|默认保存在**当前工作目录**；加`--ros-args -p save_map_dir:=/xxx`可以指定目录|直接在data字段写完整路径前缀，更灵活|
|适用场景|通用，任何SLAM、静态地图都能保存|只用于slam_toolbox建图时导出地图|
|语法难度|简单，命令行参数|YAML嵌套，容易写错|

## 你的项目实操建议
1. **正在跑 slam_toolbox 在线建图，想要获取最新SLAM原始地图** → 优先用 `slam_toolbox/save_map` 服务
2. **已经建好地图，slam_toolbox已经关闭，map_server在发布静态/map** → 只能用 `map_saver_cli`
3. **日常快速保存、不想写复杂YAML** → `map_saver_cli` 更省心

### map_saver_cli 指定保存目录（常用扩展）
如果你想直接保存到`_tmp`目录，避免文件落在当前目录：
```bash
ros2 run nav2_map_server map_saver_cli -f mymap --ros-args -p save_map_dir:=/home/abner/a2/zdev/nv/adas-01/ros/_tmp
```

## 补充重要坑
1. `map_saver_cli` 启动后会**等待一小段时间**获取地图，如果`/map`只发布一次，有可能订阅失败拿不到地图。
2. slam_toolbox的save_map保存的yaml里，`image`字段是**相对路径**，移动yaml/pgm文件到别的文件夹时，容易出现找不到pgm图片的问题，需要手动修改yaml。
3. 两个工具生成的yaml格式**完全一致**，map_server加载时无差别。

---
### 工作流推荐（你现在的slam_toolbox建图流程）
```bash
# 建图完成后，方案A（推荐，拿SLAM后端最新地图）
ros2 service call /slam_toolbox/save_map slam_toolbox/srv/SaveMap "{name: {data: '/home/abner/a2/zdev/nv/adas-01/ros/_tmp/mymap'}}"

# 或者方案B（简单，读/map话题）
ros2 run nav2_map_server map_saver_cli -f mymap --ros-args -p save_map_dir:=/home/abner/a2/zdev/nv/adas-01/ros/_tmp
```

要不要顺便讲：保存地图后，**静态加载地图+AMCL定位**的launch写法，替代slam_toolbox在线建图？

=================================
# ch06.torque vs effort
**英文：torque = 力矩（物理名词）；effort = ROS消息字段名**

1. **物理概念：Torque**
> 物理上，旋转的力矩，标准术语就是 **torque**，单位 N·m（牛顿·米）。

2. **ROS消息字段：effort**
`sensor_msgs/JointState` 里面字段叫 `effort`，**它存的就是 torque（力矩）**。
> 历史原因：ROS早期设计，把关节的“输出作用力/力矩”统一命名为 `effort`。
> - 旋转关节（revolute）：effort = **torque 力矩 N·m**
> - 移动关节（prismatic，直线伸缩）：effort = **force 力 N（牛顿）**

> 所以：
> - 旋转关节：`effort` 等价于 `torque`
> - 滑动关节：`effort` 等价于 `force`
 
## 为什么不直接叫 torque？
ROS1 设计的时候，为了**一个字段同时兼容旋转关节和直线关节**，就用了通用词 `effort`（作用力/出力），而不是区分 torque / force。
这是ROS历史遗留命名，**不是翻译错误**。
 
===============================================

# ch07.gz topic 命令解析

```
gz topic -t "/cmd_vel" -m gz.msgs.Twist -p "linear: {x: 0.5}, angular: {z: 0.05}"
```

**作用：向Gazebo Transport话题 `/cmd_vel` 一次性发送一条Gazebo原生Twist速度指令，驱动SDF里的 `gz::sim::systems::DiffDrive` 差速底盘**

> 
> ⚠️ 重点：**这是 gz transport 的话题，不是 ROS2 的 /cmd_vel！**
> 消息类型是 `gz.msgs.Twist`，不是ROS2的 `geometry_msgs/msg/Twist`。

## 参数拆解

| 参数 | 含义 |
| --- | --- |
| `gz topic` | Gazebo Sim 的命令行话题工具（类似 ros2 topic） |
| `-t "/cmd_vel"` | topic名称：`/cmd_vel`（要和DiffDrive插件里`<topic>cmd_vel</topic>`名字匹配） |
| `-m gz.msgs.Twist` | 消息类型：Gazebo原生Twist速度消息 |
| `-p "linear: {x: 0.5}, angular: {z: 0.05}"` | 消息内容，YAML格式 |

- `linear.x = 0.5`：**底盘前进线速度 0.5 m/s**
- `angular.z = 0.05`：**绕Z轴角速度 0.05 rad/s**

👉 效果：小车一边以0.5m/s向前走，一边缓慢左转，走一个大圆弧。

## 关键坑点

1. **一次性单发消息**
这条命令**只发1次**，DiffDrive插件需要持续收到速度指令。执行一次后，很快小车就会停下。
想要持续跑，写循环脚本，或者用`gz topic -r`持续发布（部分版本支持）。

> 
> 很多人疑惑：执行一次，小车动一下马上停，就是这个原因。
2. **话题名字很容易带model前缀**
官网demo里很多时候话题全名是 `/model/机器人名字/cmd_vel`，不是简单 `/cmd_vel`。

```
# 示例，机器人名叫robot1
gz topic -t "/model/robot1/cmd_vel" -m gz.msgs.Twist -p "linear: {x:0.5}, angular:{z:0.05}"
```

用下面命令查看当前gz里所有话题，确认真实topic名称：

```
gz topic -l
```

3. 和ROS2话题的桥梁：`ros_gz_bridge`
如果你想用ROS2命令（`ros2 topic pub`）控制这个gz sim小车，就需要桥接：

```
ros2 run ros_gz_bridge parameter_bridge /cmd_vel@geometry_msgs/msg/Twist@gz.msgs.Twist
```

桥接之后：

- ROS2侧发布 `geometry_msgs/msg/Twist` → ros_gz_bridge → 转发给gz transport `/cmd_vel`
- 这时你就可以用 `ros2 topic pub /cmd_vel geometry_msgs/msg/Twist "{linear: {x:0.5}, angular:{z:0.05}}"` 控制小车

## 配套查看gz侧里程计命令

```
gz topic -e -t /model/robot/odometry
```

查看Gazebo原生里程计输出（gz.msgs.Odometry），同样需要ros_gz_bridge才能转成ROS2 `/odom`。

## 两条路线对比（回顾你前面的选型）

- 路线A（当前这条，原生gz DiffDrive System Plugin）：
gz sim 内部DiffDrive插件，接收 **gz transport /cmd_vel**，必须用`gz topic`或者ros_gz_bridge。
- 路线B（gz_ros2_control）：
控制器直接订阅 **ROS2原生 /cmd_vel**，不需要ros_gz_bridge做cmd_vel桥接，直接`ros2 topic pub`就能控制，真机代码一致。

## 停止小车命令

给0速度即可：

```
gz topic -t "/cmd_vel" -m gz.msgs.Twist -p "linear: {x: 0.0}, angular: {z: 0.0}"
```

如果你需要，我可以写一个**bash循环脚本**，持续发布速度，小车一直转圈。

=========================================================
# ch08.ROS2 Jazzy中的<plugin> 和 <sensor>
<!-- google AI -->
在 ROS 2 Jazzy 所使用的 Gazebo Sim（原名 Ignition Gazebo，SDF 版本通常为 1.10+）中，<plugin> 和 <sensor> 扮演着完全不同的角色，它们的分工更加模块化和清晰。
一句话核心区别：<sensor> 负责“生成/产生”数据（如雷达、相机），而 <plugin> 负责“控制”或“传输”数据（如让车子动起来、把传感器数据打包发布给 ROS 2）。
它们在结构、功能和典型应用上有明显的对比：

-  🧱 核心对比

| 特性 | <sensor>（传感器） | <plugin>（插件） |
|---|---|---|
| 主要功能 | 定义物理世界中的硬件观测设备（如激光、摄像头、IMU），计算光线投射或物理碰撞并生成原始数据。 | 赋予模型动态行为、控制逻辑，或者充当 Gazebo 与 ROS2 之间的通信桥梁。 |
| 放置位置 | 必须嵌套在 <link> 标签内部（因为传感器需要绑定在某个具体的物理刚体上）。 | 可以放在 <model>（控制整车）、<world>（环境插件）或 <link>/<joint> 内部。 |
| ROS2 通信 | 自带传输接口，但默认发布在 Gazebo 的内部总线（Gz Transport）上。 | 用于监听 ROS2 的控制指令（如 cmd_vel），或将 Gz 数据桥接到 ROS2（如 ros_gz_bridge）。 |

----
## 1. 📂 <sensor> 详解（传感器）
<sensor> 用于在仿真中模拟真实的硬件。它决定了传感器的采样频率、分辨率、噪声以及探测范围。
💡 关键点： 在 ROS 2 Jazzy 中，单纯声明 <sensor> 只会让 Gazebo 内部产生数据。要让 ROS 2 节点（如 Rviz2）看到这些数据，通常需要配合 <plugin> 或在终端运行 ros_gz_bridge 来进行转发。

📝 代码示例（在 <link> 内部）：
```xml
<link name="lidar_link">
  <sensor name="gpu_lidar" type="gpu_lidar">
    <pose>0 0 0 0 0 0</pose>
    <update_rate>10</update_rate> <!-- 10Hz 刷新率 -->
    <lidar>
      <scan>
        <horizontal>
          <samples>640</samples>
          <resolution>1</resolution>
          <min_angle>-3.1415926</min_angle>
          <max_angle>3.1415926</max_angle>
        </horizontal>
      </scan>
      <range>
        <min>0.08</min>
        <max>10.0</max>
      </range>
    </lidar>
    <!-- 重点：新版 Gazebo 允许直接在 sensor 里加 plugin，以便让它自动发布到 Gz Transport -->
    <plugin filename="gz-sim-sensors-system" name="gz::sim::systems::Sensors">
    </plugin>
  </sensor>
</link>
```
----
## 2. ⚙️ <plugin> 详解（插件）

<plugin> 是 C++ 编写的动态链接库（.so 文件），用于扩展 Gazebo 的功能。在 ROS 2 Jazzy 中，你最常用到的是 Gazebo 官方系统插件 和 ROS-Gz 桥接插件。

💡 关键点： 比如你想让你之前定义的 vehicle_blue 两轮车能够接收 /cmd_vel 话题并走动，你就必须在 <model> 级别添加一个差速驱动的 <plugin>。
📝 代码示例（在 <model> 内部）：

<model name="vehicle_blue">
  <!-- ... 之前的 link 和 joint ... -->

  <!-- 差速驱动插件：赋予车辆移动物理特性的能力 -->
  <plugin
    filename="gz-sim-diff-drive-system"
    name="gz::sim::systems::DiffDrive">
    <left_joint>left_wheel_joint</left_joint>
    <right_joint>right_wheel_joint</right_joint>
    <wheel_separation>0.4</wheel_separation>
    <wheel_radius>0.133</wheel_radius>
    <odom_publish_frequency>30</odom_publish_frequency>
    <topic>cmd_vel</topic> <!-- 监听的 Gazebo 话题 -->
  </plugin>
</model>

----
## 🚀 在 ROS 2 Jazzy (Gazebo Sim) 中的黄金搭档
在旧版 Gazebo（Classic）中，我们习惯在 <sensor> 里面塞一个 libgazebo_ros_ray_sensor.so 插件直接把数据吐给 ROS。
但在 ROS 2 Jazzy 中，标准的架构推荐做法是：

   1. 用 <sensor> 声明激光雷达或相机。
   2. 用 gz-sim-sensors-system 插件（通常写在 world 文件中或直接附在传感器上）让 Gazebo 把数据生成出来。
   3. 在 ROS 2 启动文件（Launch file）中，使用 ros_gz_bridge 节点，将 Gazebo 的数据线（Gz Topic）和 ROS 2 的数据线（ROS Topic）连接起来。

您目前是正在为您这辆蓝色的车子（vehicle_blue）添加特定的传感器（如激光雷达/相机），还是准备编写 Launch 文件来让它在 ROS 2 Jazzy 中跑起来？我可以为您提供对应的完整配置代码！

