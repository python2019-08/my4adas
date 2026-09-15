# ch1.✅ ROS2 Jazzy **Gazebo Harmonic**

ROS2 Jazzy：官方配套是 **Gazebo Harmonic（新版Gazebo，旧名叫Ignition Gazebo），不支持 Gazebo Classic（Gazebo11）**
> Ubuntu 24.04（Jazzy的系统）官方源已经不带Gazebo Classic 11，Jazzy 没有 `gazebo_ros_pkgs`，改用 `ros_gz` 系列包。

## 一、安装命令（Jazzy）
```bash
# 完整安装 ros_gz + Harmonic
sudo apt install ros-jazzy-ros-gz
```
核心包：
- `ros_gz_sim`：仿真启动
- `ros_gz_bridge`：ROS2 ↔ Gazebo 话题桥接（非常关键）
- `gz_ros2_control`：替代旧版 `gazebo_ros2_control`

启动命令不再是 `gazebo`，而是：
```bash
gz sim
```

## 二、Classic vs Harmonic 关键差异（对你Autoware巡逻机器人仿真很重要）
|项目|Gazebo Classic(Gazebo11)|Gazebo Harmonic|
|---|---|---|
|ROS2版本|Humble及更早|Jazzy、Kilted|
|ROS集成包|`gazebo_ros_pkgs`|`ros_gz`|
|启动命令|`gazebo`|`gz sim`|
|URDF插件|`gazebo_ros`|`ros_gz`|
|消息|`gazebo_msgs`|`ros_gz_interfaces`|
|SDF|SDF1.6左右|SDF1.1X，支持USD|

> ⚠️ 老的 Classic 的URDF插件不能直接拿到Harmonic里跑，需要改URDF，替换插件标签。

## 三、Autoware Jazzy仿真注意点
Autoware Jazzy 仿真环境，默认就是基于 **Gazebo Harmonic + ros_gz**，不再使用老Gazebo Classic。
- 仿真里LiDAR、IMU、底盘驱动全部用 `ros_gz` 的插件；
- 如果你有旧巡逻机器人的Classic模型，需要迁移URDF/SDF。

## 四、验证当前安装的Gazebo版本
```bash
gz --version
```

如果你需要，我可以给你一份**巡逻机器人URDF从Gazebo Classic迁移到Harmonic最小示例**（LiDAR+IMU+差速底盘）。

=====================================

# ch2.SDF（SDFormat，Simulation Description Format）

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
# ch3.`gz sim -h`
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
# ch4. 机器人建图流程

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
# ch5. 两条命令对比：`map_saver_cli` vs slam_toolbox save_map service
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
# torque vs effort
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
 