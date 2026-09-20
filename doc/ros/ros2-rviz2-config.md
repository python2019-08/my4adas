# ch1.rviz2 配置文件（后缀 `.rviz`，本质 YAML）
一句话：**保存 RViz2 界面所有可视化配置，下次打开直接恢复一模一样的界面、话题、颜色、视角、工具**，不包含任何机器人算法，只是可视化界面参数。

## 五大核心模块
### 1. Global Options（全局选项，最关键）
```yaml
Global Options:
  Fixed Frame: map
  Frame Rate: 30
```
- `Fixed Frame`：**固定参考坐标系**（Nav2 仿真一般填 `map`，看模型/TF调试常用 `base_link`），RViz 所有数据都转换到这个坐标系渲染，**最容易踩坑的地方**
- Frame Rate：界面刷新帧率

### 2. Displays（显示项，你添加的所有可视化条目）
每一条对应 RViz 左侧 Add 的插件，包含：`Class`（插件类型）、`Name`、`Enabled`（是否启用）、插件专属参数。
常用插件示例（你的机器人仿真项目）：
```yaml
Displays:
  - Class: rviz_default_plugins/Grid          # 地面网格
    Name: Grid
    Enabled: true
  - Class: rviz_default_plugins/RobotModel    # 机器人URDF模型
    Name: RobotModel
    Enabled: true
    Description Topic: /robot_description
  - Class: rviz_default_plugins/TF            # TF坐标坐标轴
    Name: TF
    Enabled: true
  - Class: rviz_default_plugins/LaserScan     # 激光雷达 /scan
    Name: LaserScan
    Enabled: true
    Topic: /scan
    Size: 0.05
  - Class: rviz_default_plugins/Map           # 地图
    Name: Map
    Topic: /map
  - Class: rviz_default_plugins/Path          # Nav2规划路径
    Name: Planned Path
    Topic: /plan
  - Class: rviz_default_plugins/PoseArray     # 粒子滤波AMCL粒子
    Name: Particles
```
> 每个插件内部：话题名、颜色、透明度、点大小、阈值、可靠性策略全部存在这里。

### 3. Views（3D视图设置）
- 3D视角控制器类型（Orbit / FPS）
- 相机初始位置、俯仰角度、缩放
- 保存的多个预设视角（你保存的视图快照）

### 4. Panels（面板布局）
定义窗口上的各个面板：
- Displays（左侧显示列表面板）
- Time（时间面板，看sim_time）
- Views
- 导航面板 `rviz_navigation_plugins/NavigationPanel`（Navigate to Pose 那个按钮面板）
- 窗口分割、大小、位置布局

### 5. Tools（顶部工具栏工具）
- 2D Pose Estimate（AMCL初始位姿）
- 2D Nav Goal（Nav2目标点）
- 测量工具、选择工具等，保存工具参数

## 完整最简示例片段
```yaml
Visualization Manager:
  Class: ""
  Global Options:
    Fixed Frame: "map"
    Frame Rate: 30
  Displays:
    - Class: rviz_default_plugins/Grid
      Name: Grid
      Enabled: true
      Cell Size: 1.0
    - Class: rviz_default_plugins/RobotModel
      Name: RobotModel
      Enabled: true
      Description Topic: /robot_description
    - Class: rviz_default_plugins/LaserScan
      Name: LaserScan
      Enabled: true
      Topic: /scan
  Panels:
    - Class: rviz_common/Displays
      Name: Displays
    - Class: rviz_common/Time
      Name: Time
  Tools:
    - Class: rviz_default_plugins/Select
    - Class: rviz_navigation_plugins/SetNavGoal
  Views:
    Current:
      Class: rviz_default_plugins/Orbit
```

## 工程使用要点（适配你的 robot_nav_bringup）
1. 保存方式：RViz2 界面 `File -> Save Config As`，存到 `config/nav.rviz`
2. launch.py 启动加载：
```python
Node(
    package='rviz2',
    executable='rviz2',
    arguments=['-d', rviz_config_path],
    parameters=[{'use_sim_time': True}]
)
```
3. 注意：
- `.rviz` 只是界面配置，**不包含URDF、不启动任何ROS节点**
- 修改话题名、Fixed Frame，直接改yaml或者在RViz界面改后保存
- 提交到代码仓库时，把 `.rviz` 文件放进包的 `config` 目录

## 常见坑
- Fixed Frame 写错，模型/激光雷达看不见，全是红色报错
- Topic名字和实际ROS话题不一致，面板无数据
- 仿真环境必须开启 `use_sim_time:=true`，否则时间戳报错

要不要我直接给你一份适配你差速底盘+Nav2仿真的完整 `.rviz` 模板文件？

================================================

# ch2.rviz2 场景中, 建图前 的灰绿色地图背景  如何才能显示

```sh
terminal 1中:
(base) abner@abner-XPS-8940:~$  ros2 launch slam_toolbox online_async_launch.py use_sim_time:=True
[INFO] [launch]: All log files can be found below /home/abner/.ros/log/2026-09-20-07-52-01-845810-abner-XPS-8940-9783
[INFO] [launch]: Default logging verbosity is set to INFO
[INFO] [async_slam_toolbox_node-1]: process started with pid [9798]
[async_slam_toolbox_node-1] [INFO] [1789861922.126158337] [slam_toolbox]: Node using stack size 40000000
[async_slam_toolbox_node-1] [INFO] [1789861922.319896287] [slam_toolbox]: Configuring
[async_slam_toolbox_node-1] [INFO] [1789861922.348830682] [slam_toolbox]: Using solver plugin solver_plugins::CeresSolver
[async_slam_toolbox_node-1] [INFO] [1789861922.349139753] [slam_toolbox]: CeresSolver: Using SCHUR_JACOBI preconditioner.
[INFO] [launch.user]: [LifecycleLaunch] Slamtoolbox node is activating.
[async_slam_toolbox_node-1] [INFO] [1789861922.372484860] [slam_toolbox]: Activating
 
terminal 2中:
(base) abner@abner-XPS-8940:~$ conda deactivate 
abner@abner-XPS-8940:~$ ros2 run slam_toolbox async_slam_toolbox_node --ros-args -p use_sim_time:=True

terminal 3中:
abner@abner-XPS-8940:~$ rviz2
[INFO] [1789862264.215318861] [rviz2]: Stereo is NOT SUPPORTED
[INFO] [1789862264.215373928] [rviz2]: OpenGl version: 4.6 (GLSL 4.6)
[INFO] [1789862264.309384511] [rviz2]: Stereo is NOT SUPPORTED

rviz2中:No map received
rviz2 场景中, 建图前 的灰绿色地图背景  如何才能显示
```

你说的**灰绿色（未知区域）画布**，是 `Map` 插件渲染 `nav_msgs/msg/OccupancyGrid` 的结果：
> 栅格值 `-1` = 未知区域 → RViz Map插件默认渲染成灰绿色；
> 0 = 空闲（白色）；100 = 障碍物（黑色）
 
✅ 你的现象：**slam_toolbox节点成功启动，但它不会一上电就直接发布 /map（灰绿未知地图）**。
slam_toolbox 必须**收到有效的 `/scan`激光雷达数据 + `/odom`里程计 + 完整TF**之后，才会开始处理并发布 `/map`。
> 没有雷达扫描数据 → 不会发布 OccupancyGrid → RViz Map插件：`No map received`

> ⚠️ 你现在两个终端同时启动了两份slam_toolbox（终端1 launch、终端2又run），冲突了！**关掉其中一个**，只保留一个slam_toolbox。

## 第一步：快速自检命令（按顺序执行）
```bash
#1 看话题列表，检查有没有 /scan /odom /map
ros2 topic list

#2 看/map话题有没有发布者（重点！）
ros2 topic info /map

#3 查看激光雷达数据（必须有输出，卡住不动=没有雷达数据）
ros2 topic echo /scan --once

#4 查看里程计
ros2 topic echo /odom --once

#5 查看TF树，必须有 odom → base_link → laser_link
ros2 run tf2_ros tf2_monitor
```

### 现象判断
1. `ros2 topic info /map` → Publisher count:0 → **slam_toolbox没有发布map，根源是收不到/scan**
2. `ros2 topic echo /scan` 卡住无输出：Gazebo仿真机器人没启动 / 雷达插件没正常输出LaserScan
3. 有`/scan`但是TF断裂（tf2_monitor报错）：雷达link到base_link的TF缺失，slam无法使用雷达数据

## 第二步：为什么之前我说启动slam_toolbox就有灰绿底图？
> 旧认知容易踩坑：
> slam_toolbox**不是启动就直接输出一张全-1空白栅格**（这点和cartographer+occupancy_grid_node不一样！）
> slam_toolbox逻辑：
> 1. 等待收到第一帧有效的激光scan
> 2. 匹配里程计、TF，初始化机器人位姿
> 3. 初始化完成之后，才开始发布/map（此时整张地图全部是-1未知区域，就是你想要的灰绿色画布）
>
> 👉 **没有收到任何scan，永远不会发布/map**

## 第三步：你的仿真环境要补齐的组件（Jazzy+gz sim）
你前面做的Xacro机器人模型，必须同时全部启动：
1. `gz sim office.world` 启动gazebo仿真世界
2. `robot_state_publisher` 加载你的机器人xacro，发布TF
3. 底盘差速控制器(gz_ros2_control)，发布 `/odom`
4. 雷达插件(gazebo_ros_ray)，发布 `/scan`
5. slam_toolbox（online_async），订阅/scan、/odom → 发布/map

> 你现在大概率**没启动Gazebo仿真+机器人模型**，没有`/scan`雷达数据 → slam_toolbox一直空等，不输出/map

## 第四步 RViz侧确认
1. RViz中Map插件：Topic=`/map`
2. Global Options → Fixed Frame = `map`
3. Color Scheme = `map`

## 临时快速验证方案（不用启动Gazebo，直接看到灰绿地图）
如果你只是想**立刻在rviz看到灰绿色背景**，不管slam，直接运行前面给你的空白map发布节点：
```bash
python3 empty_map_pub.py
```
运行后，`ros2 topic info /map`就会看到publisher，RViz立刻出现灰绿色未知地图。

## 推荐启动顺序（仿真建图完整顺序）
终端A：启动Gazebo + robot_state_publisher + gz ros2 control 加载你的机器人
```bash
ros2 launch robot_nav_bringup gazebo_robot.launch.py
```
终端B：启动slam_toolbox
```bash
ros2 launch slam_toolbox online_async_launch.py use_sim_time:=True
```
终端C：rviz2

启动后，Gazebo里让小车稍微移动一点点，产生scan+odom，slam_toolbox收到第一帧雷达，立刻发布/map，RViz就出现灰绿底图。

## 常见坑
1. `use_sim_time:=True`：Gazebo仿真环境，**所有节点（robot_state_publisher、slam、rviz）都必须开启use_sim_time**，时间戳不一致会导致slam丢弃雷达数据
2. 话题名不匹配：雷达如果发布`/laser_scan`而不是`/scan`，slam_toolbox默认订阅`/scan`，就收不到数据，需要修改slam参数`scan_topic: /laser_scan`
3. 坐标系名字不匹配，TF断链：laser的frame_id和URDF的link名称不一致

你可以先执行`ros2 topic list`，把输出贴给我，我一眼就能判断缺少哪个话题。

需要我给你修改slam_toolbox参数文件，自定义雷达话题名、frame_id吗？
