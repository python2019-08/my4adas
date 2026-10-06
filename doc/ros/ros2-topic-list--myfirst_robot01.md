# ch01.myfirst_robot01$ ros2 topic list
```sh
abner@abner-XPS-8940:~/ros/myfirst_robot01$ ros2 topic list
/clicked_point
/clock
/cmd_vel
/controller_selector
/downsampled_costmap
/downsampled_costmap_updates
/global_costmap/costmap
/global_costmap/costmap_updates
/global_costmap/voxel_marked_cloud
/goal_checker_selector
/imu
/initialpose
/joint_states
/local_costmap/costmap
/local_costmap/costmap_updates
/local_costmap/published_footprint
/local_costmap/voxel_marked_cloud
/local_plan
/map
/map_updates
/mobile_base/sensors/bumper_pointcloud
/odom
/parameter_events
/particle_cloud
/plan
/planner_selector
/progress_checker_selector
/robot_description
/rosout
/scan
/smoother_selector
/tf
/tf_static
/waypoints
```
==========================================
# ch02.ros2 topic info /downsampled_costmap -v

```sh
abner@abner-XPS-8940:~/ros/myfirst_robot01$ ros2 topic info /downsampled_costmap -v
    Type: nav_msgs/msg/OccupancyGrid

    Publisher count: 0

    Subscription count: 1

    Node name: rviz
    Node namespace: /
    Topic type: nav_msgs/msg/OccupancyGrid
    Topic type hash: RIHS01_8d348150c12913a31ee0ec170fbf25089e4745d17035792a1ba94d6f0bc0cfc7
    Endpoint type: SUBSCRIPTION
    GID: 01.10.a3.13.b3.4f.a5.5b.1a.d0.f9.15.00.00.5a.04
    QoS profile:
    Reliability: RELIABLE
    History (Depth): KEEP_LAST (1)
    Durability: TRANSIENT_LOCAL
    Lifespan: Infinite
    Deadline: Infinite
    Liveliness: AUTOMATIC
    Liveliness lease duration: Infinite
```

==========================================
# ch03.ros2 topic info /downsampled_costmap_updates   -v

```sh
$ ros2 topic info /downsampled_costmap_updates   -v
    Type: map_msgs/msg/OccupancyGridUpdate
    Publisher count: 0
    Subscription count: 1
    Node name: rviz
    Node namespace: /
    Topic type: map_msgs/msg/OccupancyGridUpdate
    Topic type hash: RIHS01_c4095b0be2e4363f8978a1570e0293ee093436f1e1c61d99b48502ac317025e8
    Endpoint type: SUBSCRIPTION
    GID: 01.10.a3.13.b3.4f.a5.5b.1a.d0.f9.15.00.00.5b.04
    QoS profile:
    Reliability: RELIABLE
    History (Depth): KEEP_LAST (5)
    Durability: VOLATILE
    Lifespan: Infinite
    Deadline: Infinite
    Liveliness: AUTOMATIC
    Liveliness lease duration: Infinite
```
## 解读这条输出
```
/downsampled_costmap_updates
Publisher count: 0
Subscription count: 1
Subscriber: rviz
```
含义：
> **RViz 在订阅这个话题，但是目前没有任何节点在发布 `/downsampled_costmap_updates`**
> 所以这个话题是空的，没有数据流。

## 这个话题是什么
`/downsampled_costmap_updates` 属于 Nav2 costmap 的**局部更新消息（map_msgs/OccupancyGridUpdate）**，是增量更新地图小块，不是完整 `/map`。
- 正常场景：Nav2 的 costmap_node / global_costmap / local_costmap 才会发布这个话题
- 你现在：Nav2 这部分节点**没启动 / 没开启 downsample 选项**，所以 Publisher=0
- RViz 只是“挂了一个订阅端口等着收数据”，收不到任何东西

## 快速排查命令
```bash
# 查看当前所有节点，确认 nav2 的 costmap 节点是否起来
ros2 node list

# 看 nav2 相关节点发布哪些topic，比如 global_costmap / local_costmap
ros2 node info /global_costmap_node
ros2 node info /local_costmap_node
```

## 两种情况区分
1. **你只是在 RViz 里勾选了这个话题**
RViz 只要在下拉框选中 `/downsampled_costmap_updates`，就会自动创建订阅，哪怕没人发数据，就会出现 `Publisher count:0`，属于**正常现象**，只是 RViz 空等消息。
👉 不影响建图（slam_toolbox输出的是 `/map`，不是这个增量话题）

2. **你想要 Nav2 输出这个 costmap 更新话题**
需要在 nav2 costmap yaml 配置里打开下采样发布选项（默认很多模板是关闭的）：
```yaml
# costmap 参数示例
publish_downsampled_costmap: true
downsampling_factor: 2
```

## 和你之前问题关联
你之前：
- `/map` 有publisher（slam_toolbox）
- `/scan` 正常
- 之前缺 `/odom`
> `/downsampled_costmap_updates` 是 Nav2 导航层的话题，**不属于SLAM建图**。
> 就算这个话题没人发，slam_toolbox建图、rviz显示/map依然可以正常工作。

---

### 快速验证建议
看RViz里你添加的显示类型：
- 如果是 `Map` 显示插件，订阅的是 `/map`，这个**才是SLAM输出的地图**
- 如果是 `MapUpdate` 插件，订阅的就是 `/downsampled_costmap_updates`，这个是costmap增量更新，**不是SLAM地图**

你可以执行这条看 Nav2 costmap 主话题：
```bash
ros2 topic info /global_costmap/costmap -v
```

要不要我帮你区分一下：RViz里应该用哪个插件看SLAM的地图，哪个看Nav2代价地图？

========================================

# ch04.ros2 topic info /global_costmap/voxel_marked_cloud -v

```sh
abner@abner-XPS-8940:~/ros/myfirst_robot01$ ros2 topic info /global_costmap/voxel_marked_cloud -v
Type: sensor_msgs/msg/PointCloud2

Publisher count: 0

Subscription count: 1

Node name: rviz
Node namespace: /
Topic type: sensor_msgs/msg/PointCloud2
Topic type hash: RIHS01_9198cabf7da3796ae6fe19c4cb3bdd3525492988c70522628af5daa124bae2b5
Endpoint type: SUBSCRIPTION
GID: 01.10.a3.13.b3.4f.a5.5b.1a.d0.f9.15.00.00.5f.04
QoS profile:
  Reliability: RELIABLE
  History (Depth): KEEP_LAST (5)
  Durability: VOLATILE
  Lifespan: Infinite
  Deadline: Infinite
  Liveliness: AUTOMATIC
  Liveliness lease duration: Infinite
```

## 解读输出
```
/global_costmap/voxel_marked_cloud
Type: sensor_msgs/msg/PointCloud2
Publisher count: 0
Subscription count: 1
Subscriber: rviz
```
**RViz订阅了这个点云话题，但没有任何节点向外发布数据。**

## 这个话题是什么
`/global_costmap/voxel_marked_cloud` 是 Nav2 **Voxel Layer（体素层）** 的可视化话题：
- 只有当 costmap 启用 `voxel_layer` 才会生成这个话题
- 作用：发布被标记占用的3D体素点云，在RViz里看3D障碍物
- **默认不开启**，你配置里用的是 `obstacle_layer`（2D激光层）的话，这个话题根本不会被创建

> 简单区分：
> - `obstacle_layer`：2D激光，生成2D代价地图，**没有点云输出**
> - `voxel_layer`：3D体素，会把激光投影到3D栅格，可以输出 `voxel_marked_cloud` 点云用于可视化

## 为什么 Publisher=0
两个必要条件缺一不可：
1. global_costmap 配置加载了 `voxel_layer`
2. voxel_layer 参数开启 `publish_voxel_cloud: true`

示例yaml片段：
```yaml
global_costmap:
  plugins:
    - static_layer
    - obstacle_layer
    - voxel_layer
  voxel_layer:
    plugin: "nav2_costmap_2d::VoxelLayer"
    publish_voxel_cloud: true   # 必须打开，才会发布 /global_costmap/voxel_marked_cloud
    ...
```
你现在大概率：
- 没加voxel_layer，或者
- 加了但`publish_voxel_cloud`是false

👉 **这个话题仅仅是可视化调试用，不开启完全不影响导航功能。**

## 和你当前现象总结
你现在RViz里，连续多个Nav2调试话题都出现：
`Publisher count:0, Subscriber count:1`
原因统一：**RViz添加了对应的显示插件，订阅了话题，但是Nav2 costmap没有开启对应发布参数，没有节点输出数据**
- `/downsampled_costmap` / `/downsampled_costmap_updates` → publish_downsampled_costmap 未打开
- `/global_costmap/voxel_marked_cloud` → voxel_layer 未启用 / publish_voxel_cloud=false

> 重点：这些都是**Nav2代价地图的可视化附属话题**，**和 slam_toolbox 的 /map 建图无关**。
> 你只需要保证：
> 1. `/map` 正常发布（slam_toolbox）
> 2. `/global_costmap/costmap` 正常发布（Nav2）
> 其余这些调试可视化话题可按需开启，不开也没关系。

## 快速验证命令
```bash
# 查看真正在用的全局代价地图话题（这个是导航核心）
ros2 topic info /global_costmap/costmap -v
```

如果你想，我可以把 `global_costmap` 完整yaml给你，带上voxel_layer和所有可视化开关，直接用于你的巡逻小车。

=================================================

# ch05.ros2 topic info /global_costmap/costmap -v

```sh
a@a:~/ros/myfirst_robot01$ ros2 topic info /global_costmap/costmap -v
Type: nav_msgs/msg/OccupancyGrid

Publisher count: 0

Subscription count: 1

Node name: rviz
Node namespace: /
Topic type: nav_msgs/msg/OccupancyGrid
Topic type hash: RIHS01_8d348150c12913a31ee0ec170fbf25089e4745d17035792a1ba94d6f0bc0cfc7
Endpoint type: SUBSCRIPTION
GID: 01.10.a3.13.b3.4f.a5.5b.1a.d0.f9.15.00.00.5c.04
QoS profile:
  Reliability: RELIABLE
  History (Depth): KEEP_LAST (1)
  Durability: TRANSIENT_LOCAL
  Lifespan: Infinite
  Deadline: Infinite
  Liveliness: AUTOMATIC
  Liveliness lease duration: Infinite
```

## 核心结论

`/global_costmap/costmap` **Publisher count:0，Subscriber count:1（rviz）**
👉 RViz订阅了全局代价地图，**但是 Nav2 的 global_costmap 节点根本没有在运行，没有任何节点发布这个话题**。
> 这个和前面几个可视化话题性质不一样：`/global_costmap/costmap` 是 Nav2 导航**核心话题**，它没数据=Nav2代价地图模块没跑起来，无法做路径规划。

## 排查步骤
### 1. 先确认 nav2 costmap 节点是否启动
```bash
ros2 node list
```
正常跑起来应该看到类似：
- `/global_costmap`
- `/local_costmap`

如果列表里**没有这两个节点**：
- 你的 nav2 bringup launch 文件**没有启动 costmap_2d 节点**
- 或者 launch 启动报错，节点crash退出了

### 2. 检查 nav2 bringup 启动日志
重新启动 nav2 bringup，看终端报错，常见失败原因：
1. **缺少 /odom 话题**（你之前踩过这个坑！costmap 依赖 odom TF + odom 消息）
2. TF链条断裂：`base_link` ↔ `odom` 不存在，costmap 启动直接退出
3. costmap yaml配置写错（插件名、参数缩进错误，yaml对缩进非常敏感）
4. 没有收到 `/map`（global_costmap 默认依赖slam输出的/map静态层）

### 3. 检查 TF 和 odom（优先做）
```bash
# 查看odom话题是否存在
ros2 topic info /odom

# 查看 odom -> base_link 变换
ros2 run tf2_ros tf2_echo odom base_link
```
> global_costmap 启动的硬性前提：
> ✅ 存在 `odom` 话题
> ✅ TF 可以查询 `odom → base_link`
> ✅ 能拿到 `/map` 消息（static_layer）

### 4. 区分两个costmap话题
- `/global_costmap/costmap`：全局代价地图，用于全局路径规划，**必须要**
- `/local_costmap/costmap`：局部代价地图，用于局部避障

执行这条查看局部代价图状态：
```bash
ros2 topic info /local_costmap/costmap -v
```

## 快速定位思路
1. 如果你只启动了 slam_toolbox，**没有启动 nav2 bringup**：那 global_costmap 节点自然不存在，publisher=0 是正常现象。slam_toolbox只管输出/map，**不负责代价地图**。
2. 如果你**已经启动nav2 bringup**，但看不到costmap节点：说明nav2启动失败，看终端日志找报错。

> 一句话：**SLAM输出/map，Nav2的costmap节点读取/map+激光雷达，生成代价地图。Nav2节点不启动，代价地图话题就不会存在。**

你现在有没有启动 nav2 bringup？还是只跑了slam_toolbox？

=================================================
# ch06.ros2 topic info /global_costmap/costmap_updates -v
```sh
a@a:~/ros/myfirst_robot01$ ros2 topic info /global_costmap/costmap_updates -v
Type: map_msgs/msg/OccupancyGridUpdate

Publisher count: 0

Subscription count: 1

Node name: rviz
Node namespace: /
Topic type: map_msgs/msg/OccupancyGridUpdate
Topic type hash: RIHS01_c4095b0be2e4363f8978a1570e0293ee093436f1e1c61d99b48502ac317025e8
Endpoint type: SUBSCRIPTION
GID: 01.10.a3.13.b3.4f.a5.5b.1a.d0.f9.15.00.00.5d.04
QoS profile:
  Reliability: RELIABLE
  History (Depth): KEEP_LAST (5)
  Durability: VOLATILE
  Lifespan: Infinite
  Deadline: Infinite
  Liveliness: AUTOMATIC
  Liveliness lease duration: Infinite
```
## 解读
`/global_costmap/costmap_updates`
- 消息类型：`map_msgs/msg/OccupancyGridUpdate`，**增量小块更新**（不是完整整张地图）
- Publisher count:0，Subscriber count:1（rviz）

> 和 `/global_costmap/costmap` 成对：
> - `/global_costmap/costmap`：完整全局代价地图（OccupancyGrid）
> - `/global_costmap/costmap_updates`：局部增量更新块，用来高效刷新，不用每次下发整张大图
>
> **前提：Nav2 global_costmap节点成功运行，并且参数 `publish_costmap_updates: true`（默认开启），才会发布这两个话题。**
> 现在 publisher=0，本质还是同一个根源：**global_costmap 节点根本没跑起来**。节点都没启动，这一对话题都不会产生任何数据。

## 汇总你现在观察到的现象
所有 nav2 costmap 相关话题：
- `/global_costmap/costmap`
- `/global_costmap/costmap_updates`
- `/downsampled_costmap` / `/downsampled_costmap_updates`
- `/global_costmap/voxel_marked_cloud`

全部：`Publisher=0，Subscriber=rviz`
👉 说明：**RViz这边预先订阅了这些话题，但是Nav2的costmap节点没有启动或者启动后崩溃退出。**

## 优先做2条诊断命令
```bash
# 看当前所有运行节点，确认有没有 /global_costmap /local_costmap
ros2 node list

# 看有没有odom（Nav2 costmap最依赖的条件，你之前踩过这个坑）
ros2 topic info /odom
```

### Nav2 costmap 启动必要条件（重点）
1. TF：`odom → base_link` 可以正常查询
2. 存在 `/odom` 话题
3. 能接收 `/map`（来自slam_toolbox）
4. costmap yaml配置无缩进错误、插件名称正确
5. 启动文件正确加载 nav2 bringup + costmap 参数

> 区分概念：
> slam_toolbox 只负责建图输出 `/map`；
> **代价地图（costmap）完全是Nav2的模块，不在slam_toolbox里面。**
> 如果你只启动slam_toolbox，不启动nav2 bringup，那么所有`/global_costmap/*`话题全部没有发布者，属于正常现象。

## 下一步选择
1. 如果你**只是调试SLAM建图**：不需要启动Nav2，这些costmap话题为空完全没问题，不用管。RViz只添加 `/map` 显示即可。
2. 如果你**准备做导航**：需要启动nav2 bringup，并且先保证 `/odom` 和 `odom→base_link` TF正常，否则costmap节点直接无法启动。

你可以把 `ros2 node list` 的输出贴出来，我们确认 nav2 节点是否已经成功拉起。


## /global_costmap/costmap 与 /global_costmap/costmap_updates 两者关系
 
1. **/global_costmap/costmap**
消息类型：`nav_msgs/msg/OccupancyGrid`
- 完整整张全局代价地图
- QoS 默认 `TRANSIENT_LOCAL`
- 新订阅者连接时，节点会一次性推送完整地图
- 数据量大，低频发布

2. **/global_costmap/costmap_updates**
消息类型：`map_msgs/msg/OccupancyGridUpdate`
- **增量局部更新**，只发送发生变化的栅格小块（x,y,宽高+栅格数组）
- QoS 默认 `VOLATILE`，没有历史消息留存
- 数据体积小、高频，用来局部刷新地图，不用每次下发整张大图

> Nav2 默认同时开启这两个输出，参数 `publish_costmap_updates: true`（默认true）。
> RViz 的 Map 插件优先接收 `/global_costmap/costmap`；
> MapUpdate 插件专门接收 `/global_costmap/costmap_updates` 做增量渲染。
   
## 为什么global_costmap节点起不来（你巡逻小车高频坑）
1. ❌ **没有 /odom 话题**（你之前遇到过！costmap必须odom+odom→base_link TF）
2. ❌ TF无法查询 `odom -> base_link`
3. ❌ 拿不到 `/map`（slam_toolbox没跑、或者/map frame_id不对）
4. ❌ costmap yaml缩进错误、插件名字写错（yaml极其敏感）
5. ❌ launch文件没有加载costmap参数，或者没启动nav2的costmap_2d节点

## 快速诊断命令
```bash
# 1. 查看运行节点，确认有没有 /global_costmap
ros2 node list

# 2. 检查odom话题
ros2 topic info /odom

# 3. 测试odom到base_link的TF
ros2 run tf2_ros tf2_echo odom base_link
```

## 重要区分
- `/map`：slam_toolbox输出的**静态栅格地图（环境地图）**
- `/global_costmap/costmap`：Nav2基于`/map`+激光雷达实时生成的**代价地图（带障碍物膨胀、禁行区域）**

> 👉 只做SLAM建图：不用启动Nav2，这两个话题为空完全正常，忽略即可，RViz只看`/map`。
> 👉 要做路径规划、自主导航：必须把`/odom`、TF、slam的/map全部调通，再启动nav2 bringup，global_costmap节点才会正常发布这两个话题。

## 可选：global_costmap关键yaml片段
```yaml
global_costmap:
  global_frame: map
  robot_base_frame: base_link
  publish_costmap_updates: true # 默认开启，输出costmap_updates增量消息
  plugins:
    - static_layer
    - obstacle_layer
  static_layer:
    plugin: "nav2_costmap_2d::StaticLayer"
    map_topic: /map
  obstacle_layer:
    plugin: "nav2_costmap_2d::ObstacleLayer"
    observation_sources: scan
    scan:
      topic: /scan
      data_type: LaserScan
      marking: true
      clearing: true
```

你现在是只想调SLAM建图，还是准备跑Nav2导航？

========================================

# ch07.ros2 topic info /local_costmap/costmap  -v

```sh
a@a:~/ros/myfirst_robot01$ ros2 topic info /local_costmap/costmap  -v
Type: nav_msgs/msg/OccupancyGrid

Publisher count: 0

Subscription count: 1

Node name: rviz
Node namespace: /
Topic type: nav_msgs/msg/OccupancyGrid
Topic type hash: RIHS01_8d348150c12913a31ee0ec170fbf25089e4745d17035792a1ba94d6f0bc0cfc7
Endpoint type: SUBSCRIPTION
GID: 01.10.a3.13.b3.4f.a5.5b.1a.d0.f9.15.00.00.60.04
QoS profile:
  Reliability: RELIABLE
  History (Depth): KEEP_LAST (1)
  Durability: TRANSIENT_LOCAL
  Lifespan: Infinite
  Deadline: Infinite
  Liveliness: AUTOMATIC
  Liveliness lease duration: Infinite
```
## 解读 /local_costmap/costmap
```
Type: nav_msgs/msg/OccupancyGrid
Publisher count: 0
Subscription count: 1 (rviz)
```
**局部代价地图，同样没有发布节点，只有RViz在空等数据。**

> `/local_costmap/costmap` 是Nav2局部代价图，用于**局部避障、DWB控制器**，跟随机器人实时移动，只保留机器人附近一小片区域。
> - 局部代价图的参考坐标系：`odom`
> - 全局代价图参考坐标系：`map`

## 现在汇总全部现象
- `/global_costmap/costmap` & `/global_costmap/costmap_updates` → Publisher=0
- `/local_costmap/costmap` → Publisher=0
- 所有 costmap 系列话题，订阅者都是RViz

✅ 结论：**Nav2的costmap节点（global_costmap、local_costmap）根本没有成功运行。**
只要 nav2 bringup 没有正常拉起，这一组话题全部不会有发布者。

## local_costmap 启动硬性条件（和global不一样）
1. TF：`odom → base_link` 正常
2. `/odom` 话题正常输出里程
3. 激光`/scan`正常（你这边scan是正常的）
4. local_costmap yaml配置正确，observation_sources绑定`/scan`
> local_costmap **不需要 /map**，它只依赖激光+odom，实时感知机器人周边障碍物

## 现在两条路
### 场景A：你目前只调试SLAM建图（slam_toolbox）
👉 **完全不用管local/global costmap！**
这些是导航模块，和建图无关。RViz删掉所有costmap相关显示插件，只保留 `/map` 的Map插件即可。

### 场景B：你要启动Nav2做自主导航
必须：
1. 先把 `/odom` 调通（你之前遇到odom缺失，这是最大卡点）
2. TF `odom <-> base_link` 稳定
3. 启动完整nav2 bringup launch文件
4. 检查启动日志，看costmap节点报错（大概率卡在odom/TF）

## 执行这条，看nav节点是否存在
```bash
ros2 node list | grep costmap
```
- 有输出：节点启动了，但内部报错，看终端日志
- **无任何输出：costmap节点根本没被launch启动**

## 补充local_costmap最简yaml参考
```yaml
local_costmap:
  global_frame: odom
  robot_base_frame: base_link
  update_frequency: 10.0
  publish_frequency: 5.0
  plugins:
    - obstacle_layer
  obstacle_layer:
    plugin: "nav2_costmap_2d::ObstacleLayer"
    observation_sources: scan
    scan:
      topic: /scan
      data_type: LaserScan
      marking: true
      clearing: true
```

你执行 `ros2 node list | grep costmap` 之后把结果发我，我们确认costmap节点有没有被拉起。

=============================================

# ch08.# /local_costmap/published_footprint 与 /local_costmap/voxel_marked_cloud
## 一句话总结
二者**完全独立**，都是 local_costmap 的**可视化调试话题，不参与导航计算**，来源是 Nav2 costmap2d 模块，默认都不发布，需要手动开启参数。

## 1. /local_costmap/published_footprint
- 消息类型：`geometry_msgs/msg/PolygonStamped`
- 作用：**机器人轮廓（footprint）多边形**
  把你在 costmap 配置里写的机器人外形多边形，实时发布出来给RViz可视化。
  显示的是机器人本体的占用区域，用于看代价地图膨胀层是否按照机器人轮廓做障碍物膨胀。
- 开启参数：
```yaml
local_costmap:
  publish_footprint: true
```
- 坐标系：`odom`（local_costmap的global_frame）

## 2. /local_costmap/voxel_marked_cloud
- 消息类型：`sensor_msgs/msg/PointCloud2`
- 作用：**3D体素占用点云**
  只有启用 `voxel_layer` 才生效，输出被标记为占用的三维体素点云。
  用于在RViz看3D障碍物（区分地面、低空、高空障碍物）。
- 开启参数：
```yaml
local_costmap:
  plugins:
    - obstacle_layer
    - voxel_layer
  voxel_layer:
    plugin: "nav2_costmap_2d::VoxelLayer"
    publish_voxel_cloud: true
```
> 如果你只用 `obstacle_layer`（2D激光），这个话题永远不会发布。

## 对比表
|话题|消息类型|用途|依赖|
| ---- | ---- | ---- | ---- |
|`/local_costmap/published_footprint`|PolygonStamped|绘制机器人外形轮廓|costmap正常运行 + `publish_footprint: true`，不需要voxel层|
|`/local_costmap/voxel_marked_cloud`|PointCloud2|3D体素障碍物点云|必须启用voxel_layer + `publish_voxel_cloud: true`|

## 联系与区别
✅ **联系**
- 发布者都是 `/local_costmap` 节点
- 都只用于RViz可视化调试，**不参与路径规划、避障核心计算**，不开也不影响导航运行。

❌ **区别**
1. footprint 是**机器人自身轮廓（2D多边形）**；voxel_marked_cloud 是**环境障碍物3D点云**
2. footprint 不需要体素层，obstacle_layer就可以用；voxel点云必须打开voxel_layer
3. footprint 用来检查**机器人膨胀禁区**；voxel点云用来看**高低障碍物，滤除地面**

## 和你当前现状关联
你现在：
`/local_costmap/costmap` publisher=0，说明 local_costmap 节点**没有启动**。
👉 只要节点没跑，`published_footprint`、`voxel_marked_cloud` 这两个话题Publisher也必然是0，RViz只会单方面订阅等待数据。

> 小提示：
> `footprint` 是机器人本体；`obstacle/voxel` 是环境障碍物，不要混淆。

需要我给一份完整 local_costmap yaml，同时开启 `publish_footprint` 和 voxel_layer 吗？

================================================

# ch09.# 三个话题：`/imu`、`/initialpose`、`/joint_states`

  三个话题详解 + 重点：**发布方不一定是硬件！可以是仿真/URDF/代码模拟出来**

## 1. /imu
消息类型：`sensor_msgs/msg/Imu`
包含：角速度、线加速度、姿态（四元数）、协方差。
**作用**
- 输出惯性测量单元数据，用于姿态估计、里程计融合（`robot_localization`）、姿态修正
- 很多底盘/IMU驱动会把IMU的重力方向、倾斜信息喂给EKF，修正odom漂移

**发布方**
✅ 硬件：真实IMU驱动（mpu6050、 bno055、 vectornav等）
✅ 非硬件：Gazebo仿真、代码手动模拟、micro-ros仿真、rosbag回放

> 注意：IMU的姿态是**相对于IMU自身坐标系**，需要TF把imu_link转到base_link。

## 2. /initialpose
消息类型：`geometry_msgs/msg/PoseWithCovarianceStamped`
**作用**
👉 **给Nav2/SLAM设置机器人在地图中的初始位置**
在RViz里用「2D Pose Estimate」点击地图，就会往 `/initialpose` 发一条消息：告诉SLAM/导航“机器人现在在地图这个坐标、朝向”。
slam_toolbox、 nav2 amcl 都会监听这个话题。

**发布方**
✅ **几乎从来不是硬件！**
- RViz（最常用，鼠标点一下发布）
- 程序自动发布（自动初始化定位）
- rosbag / 脚本一键设置初始位姿
硬件不会发布这个话题。它是**人为/软件下发的定位指令**，不是传感器数据。

## 3. /joint_states
消息类型：`sensor_msgs/msg/JointState`
字段：`name[], position[], velocity[], effort[]`
**作用**: 告诉 `robot_state_publisher` 各个活动关节的角度/位置，然后`robot_state_publisher`根据URDF，**计算并发布TF**（比如左右车轮关节、转向关节）。
> 你之前遇到的 `right_wheel_link` TF缺失，根源就是 joint_states 没有车轮关节数据。

**发布方**
✅ 硬件：真实电机驱动（ros2_control / diff_drive_controller 读取编码器，发布joint_states）
✅ 非硬件：
- Gazebo仿真自动发布
- `joint_state_publisher_gui`（可视化滑块拖动，手动模拟关节）
- 自己写python脚本模拟车轮角度
- 无动力被动关节（固定关节不需要）

---

## 汇总表格
|话题            |消息类型                   |核心用途              |是否只能硬件发布  |典型发布源|
|---------------|--------------------------|---------------------|------------|-----------------------|
|`/imu`         |sensor_msgs/Imu           |惯性传感器数据，EKF融合 |❌ 不是      |IMU驱动 / Gazebo仿真 / 脚本|
|`/initialpose` |PoseWithCovarianceStamped |设置机器人地图初始位姿  |❌ 绝对不是硬件|RViz、python脚本           |
|`/joint_states`|sensor_msgs/JointState    |输出关节位置，供robot_state_publisher生成TF |❌ 不是 |ros2_control控制器、仿真、joint_state_publisher_gui|

## 结合你的巡逻小车项目重点提醒
1. `/joint_states` 是你之前TF问题的关键点：
diff_drive_controller读取轮子编码器 → 发布`/joint_states` → robot_state_publisher 生成 base_link ↔ wheel_link 的TF。
> 如果控制器没跑，`/joint_states`为空，车轮link的TF就消失，也就是你之前报错：`No transform from [right_wheel_link] to [base_footprint]`

2. `/imu`：你后面上`robot_localization`（EKF融合odom+imu）的时候会用到，减少里程计漂移。

3. `/initialpose`：建图完成后，每次启动导航，需要发布这个话题，让AMCL定位。

## 快速查看命令
```bash
# 查看谁在发布joint_states
ros2 topic info /joint_states -v

# 查看imu
ros2 topic info /imu -v

# 监听initialpose（鼠标在rviz点2D Pose Estimate就会输出）
ros2 topic echo /initialpose
```

要不要我顺便把这三个话题的**常见故障排查清单**整理给你？

===============================================

# ch10./map 与 /map_updates
## 1. /map
- **消息类型**：`nav_msgs/msg/OccupancyGrid`（完整栅格地图）
- **发布方**：**slam_toolbox（软件节点）**，**不是硬件**
- 作用：整张静态栅格地图。包含分辨率、原点、宽高、全部栅格占用信息。
  - QoS：`TRANSIENT_LOCAL`，新订阅者（Nav2、RViz）连上时会一次性收到完整地图。
  - 建图阶段：随机器人移动不断更新整张地图；
  - 建图完成保存地图后，`map_server` 也可以读取地图文件来发布 `/map`。

## 2. /map_updates
- **消息类型**：`map_msgs/msg/OccupancyGridUpdate`（**增量地图小块更新**）
- **发布方**：**slam_toolbox（软件节点）**，**不是硬件**
> ⚠️ 注意：`/map_updates` 是 slam_toolbox 的增量话题，**不是Nav2 costmap的话题**（Nav2的是 `/global_costmap/costmap_updates`，不要搞混！）
- 作用：只推送**发生改动的一小块栅格**，不发送整张大图。
  - 优势：带宽低，RViz只需要刷新局部改动区域，不用每次接收完整大地图。
  - 开关：slam_toolbox 参数 `publish_map_updates: true`，默认开启。

## 回答你的核心问题：发布方是硬件吗？
**不是硬件！**
- 硬件（激光雷达、IMU、轮式编码器）只负责输出原始传感器数据：`/scan`、`/imu`、`/odom`
- 硬件**不会生成栅格地图**；
- 地图是**软件SLAM算法（slam_toolbox）**接收传感器原始数据，实时运算生成，然后发布 `/map` + `/map_updates`。

> 数据流链条：
> 雷达硬件 → /scan（原始激光点）
> odom硬件/驱动 → /odom（里程）
> ↓
> slam_toolbox（软件）接收 /scan + TF + /odom
> ↓
> 计算生成栅格地图 → 发布 `/map`（完整图）、`/map_updates`（增量小块）

## 和Nav2 costmap话题做区分（非常关键，你前面一直在看costmap）
| Topic | 发布者 | 是什么 |
|---|---|---|
| `/map` | slam_toolbox / map_server | SLAM构建的**环境静态地图** |
| `/map_updates` | slam_toolbox | SLAM地图增量更新 |
| `/global_costmap/costmap` | nav2 global_costmap节点 | Nav2基于/map+激光生成的**代价地图（带障碍物膨胀）** |
| `/global_costmap/costmap_updates` | nav2 global_costmap节点 | Nav2代价图增量更新 |

## 简单理解
- `/map`：**环境地图（墙、道路）**，SLAM建出来的，描述世界长啥样
- costmap：**导航代价地图**，Nav2在/map基础上再加工，给机器人规划路径用，会把障碍物向外膨胀一圈，预留机器人安全距离

## 小坑提醒
如果你启动 `map_server` 加载已经建好的地图文件：
- map_server 只会发布 `/map`，**不会发布 `/map_updates`**，`/map_updates` 只有slam在线建图阶段才有。

需要我给你 slam_toolbox 参数片段，控制是否开启 `publish_map_updates` 吗？

===============================================
# ch11./local_plan |  /plan | /planner_selector  这些 topic 分别是做什么的？发布方 是硬件吗
 
**这三个话题全部都是 Nav2 软件节点输出，和硬件完全无关！**
硬件只输出传感器原始数据（scan、imu、odom）；规划路径是软件算法计算出来的。

---
## 1. /plan
- 消息类型：`nav_msgs/msg/Path`
- **发布方：Global Planner（全局规划器，如SmacPlanner）**
- 含义：**全局路径**
从机器人当前位置 → 目标点的**整条长距离参考路径**，一次性规划出来。
> 比如：从A巡逻点到B巡逻点，穿过整条道路的大路线。
> 只在目标点变更、或者路径被障碍物阻断时重新规划。
- RViz里添加`Path`插件订阅 `/plan`，就能看到蓝色全局参考线。

## 2. /local_plan
- 消息类型：`nav_msgs/msg/Path`
- **发布方：Local Controller（局部控制器，如DWB Controller）**
- 含义：**局部短路径**
在全局大路径的约束下，机器人**接下来一小段时间（比如2~3秒）要走的轨迹**。
> 作用：实时避障。根据最新激光雷达障碍物，动态微调局部轨迹。
> 频率很高，不断重新生成。
> RViz里显示的另一根短线就是 `/local_plan`。

> 简单对比
> `/plan`：长途的**导航路线（大方向）**
> `/local_plan`：紧跟大路线，实时躲避障碍物的**短期运动轨迹**

## 3. /planner_selector
- 消息类型：`nav2_msgs/msg/PlannerSelector`
- **发布方：planner_server（规划服务节点）**
- 含义：**规划器选择器**
Nav2支持多规划器切换（例如Smac2D / SmacLattice）。
这个话题用来**广播当前正在使用哪一个全局规划器**，也可以通过服务动态切换规划器。
> 日常巡逻导航时，这个话题多用于调试、监控；一般不用于RViz可视化。

---
## 完整数据流（帮你串联整个Nav2）
```
slam_toolbox → /map（环境地图）
传感器 → /scan、/odom
        ↓
global_costmap + planner_server
        ↓
输出 /plan 【全局路径】
        ↓
local_costmap + controller_server（DWB）
        ↓
输出 /local_plan【局部轨迹】 → 输出速度指令 /cmd_vel 给底盘驱动
```

## 重点区分记忆
| Topic              | 发布节点                          | 作用                    |
|--------------------|----------------------------------|------------------------|
| `/plan`            | planner_server（全局规划器）        | 全局长路径，从起点到目标点 |
| `/local_plan`       | controller_server（局部控制器DWB） | 局部短期轨迹，实时避障     |
| `/planner_selector` | planner_server                   | 广播当前激活的全局规划器名称 |

## 常见坑
1. 没有 `/plan`：大概率是全局规划失败，检查：`/map`、TF、costmap、目标点设置
2. 有 `/plan`，但是没有 `/local_plan`：局部控制器无法跟踪全局路径，检查局部costmap、速度限制、机器人footprint
3. 这三个话题**都不是硬件输出**，硬件底盘只接收 `/cmd_vel` 速度指令。

要不要顺带把 `/cmd_vel`、`/controller_selector` 也一起解释，把Nav2整套导航话题串齐？

===============================================
# ch12./mobile_base/sensors/bumper_pointcloud  这  topic  是做什么的？发布方 是硬件吗
 
## /mobile_base/sensors/bumper_pointcloud基本信息
- 消息类型：`sensor_msgs/msg/PointCloud2`
- **发布方：不一定是硬件，分两种情况**
  1. **真实物理机器人**：底盘驱动软件节点（不是硬件本身）读取碰撞开关（bumper/防撞条）硬件信号，把碰撞点转换成点云，发布这个topic
  2. **Gazebo仿真**：Gazebo插件仿真碰撞，由仿真插件节点发布，完全软件模拟

> 硬件（防撞条/微动开关）本身**不会直接发布ROS话题**。硬件只输出高低电平信号，由驱动程序读取、转换成ROS消息。

## 作用
把**保险杠碰撞位置**转为点云，送入Nav2 costmap的`obstacle_layer`。
- 当碰撞条被挤压触发，就会在碰撞位置生成一个点；
- costmap收到这个点云，把该位置标记为障碍物；
- 用于兜底避障：激光雷达有盲区（低矮物体、透明玻璃），激光看不到，撞到保险杠后，立刻标记障碍物，紧急停车。

## Nav2配置示例（把bumper点云加入代价地图）
```yaml
obstacle_layer:
  plugin: "nav2_costmap_2d::ObstacleLayer"
  observation_sources: scan bumper
  scan:
    topic: /scan
    data_type: LaserScan
    marking: true
    clearing: true
  bumper:
    topic: /mobile_base/sensors/bumper_pointcloud
    data_type: PointCloud2
    marking: true
    clearing: false # 碰撞点一般不清空，直到机器人离开
```
- `marking=true`：收到点云，标记栅格为占用
- `clearing=false`：不自动清除这个障碍物（碰撞属于硬接触，保守处理）

## 关键点
1. 这个topic**不是激光雷达**，是碰撞传感器的点云表达；
2. 仅作为**安全兜底**，不能作为主感知；
3. 很多自研底盘默认**不启用这个话题**，如果没装物理防撞条，这个话题Publisher=0；
4. 坐标系一般是`base_link`。

## 简单数据流
物理防撞条硬件 → 底盘驱动 → 生成PointCloud2 → `/mobile_base/sensors/bumper_pointcloud` → costmap obstacle_layer → 标记障碍物

如果你想，我可以把Nav2里常用的**所有感知类topic**整理一张汇总表（scan、imu、bumper_pointcloud、pointcloud2）。

===============================================

# ch13. /odom、/parameter_events、/scan 详解 
> 核心结论：**只有硬件相关的原始传感器数据（/odom、/scan）可以来自硬件驱动，但也可以由仿真/软件算法生成；/parameter_events 完全是ROS2系统内置话题，和硬件无关。**

## 1. /odom
消息类型：`nav_msgs/msg/Odometry`
**作用**：里程计消息。包含机器人的位姿(x,y,yaw)、线速度、角速度。
- 坐标系：`odom`
- 用途：
  1. SLAM（slam_toolbox）需要odom作为运动初值
  2. Nav2 costmap、控制器依赖odom和`odom→base_link`的TF变换
- **发布方不一定是硬件**
  ✅ 真实机器人：**底盘驱动程序**读取轮式编码器硬件，计算里程，发布`/odom`。硬件编码器只输出脉冲，**硬件本身不会发ROS消息**。
  ✅ Gazebo仿真：Gazebo差分驱动插件直接软件仿真生成`/odom`。
  ✅ 软件算法：`robot_localization`可以融合IMU+轮速，输出融合后的`/odom`。

> 一句话：硬件提供原始脉冲，**软件驱动计算后发布/odom**；仿真环境完全软件生成。

## 2. /scan
消息类型：`sensor_msgs/msg/LaserScan`
**作用**：激光雷达的二维扫描数据，包含每个角度的测距、强度。SLAM、Nav2代价地图的核心感知输入。
- **发布方不一定是硬件**
  ✅ 真实机器人：激光雷达驱动节点读取雷达硬件，解析测距数据，打包成LaserScan发布。雷达硬件只输出串口/网口二进制数据，**硬件不直接发布ROS话题**。
  ✅ Gazebo仿真：Gazebo激光插件，虚拟射线检测，软件直接生成`/scan`。

## 3. /parameter_events
消息类型：`rcl_interfaces/msg/ParameterEvent`
**作用**：ROS2系统自带话题。**监控所有节点参数的变更**。
节点修改参数（yaml加载、运行时动态set_param），就会向这个话题广播事件：参数新增、修改、删除。
- **发布方：ROS2系统底层rcl，和硬件完全无关**
- 典型用途：调试工具（rqt_reconfigure）监听参数变化；极少业务代码直接使用。

## 汇总对比表
| Topic | 消息类型 | 功能 | 是否只能硬件发布 |
|---|---|---|---|
| `/odom` | nav_msgs/msg/Odometry | 里程位姿+速度 | ❌ 底盘驱动/仿真/robot_localization都能发 |
| `/scan` | sensor_msgs/msg/LaserScan | 二维激光测距数据 | ❌ 雷达驱动 / Gazebo仿真插件 |
| `/parameter_events` | rcl_interfaces/msg/ParameterEvent | 参数变更事件 | ❌ ROS2底层系统话题，和硬件无关 |

## 重要概念（帮你理清，ROS2通用）
> **所有ROS话题，永远是软件节点发布，硬件不能直接发布ROS话题**
硬件（编码器、激光雷达、防撞条、IMU）输出的是：脉冲、串口、以太网二进制流。
必须有**驱动软件节点**读取硬件原始信号，封装成ROS消息，才会发布topic。

两种来源：
1. 物理硬件 + 驱动节点
2. 仿真器/纯软件算法，虚拟生成传感器数据（无硬件）

## 数据流示例
1. 轮编码器硬件 → 底盘驱动节点 → `/odom`
2. 激光雷达硬件 → 雷达驱动节点 → `/scan`
3. 修改slam_toolbox参数 → ROS底层rcl → `/parameter_events`

## 你项目的关联提醒
你之前Nav2 costmap起不来，**/odom + /scan**是两个最核心输入。
- `/scan`：你这边已经正常有数据
- `/odom`：你之前看到topic不存在，这个是卡点。

要不要把前面所有我们聊过的Nav2+SLAM常用话题，合并成一张完整总表？

===================================================

# ch14./particle_cloud  /waypoints这些 topic 分别是做什么的？发布方是硬件还是仿真软件
 
> 一句话结论：**两个话题都属于ROS2软件算法输出，不是硬件，也不是仿真软件原生输出。仿真只是提供传感器数据，这两个话题由定位/导航算法节点发布。**

## 1. /particle_cloud
- 消息类型：`sensor_msgs/msg/PointCloud2`
- **发布方：AMCL（自适应蒙特卡洛定位节点 `amcl`）**
> 如果你用 slam_toolbox 自带的定位模式，slam_toolbox也可以输出粒子云。
- 含义：**粒子滤波器的粒子集合**
AMCL 用大量粒子代表机器人在地图里所有可能的位姿假设。
每一个粒子 = 一个假设的机器人坐标(x,y,yaw) + 权重（置信度）。
`/particle_cloud` 把全部粒子渲染成点云，RViz里可以直接可视化：
- 粒子聚集在一起：定位收敛，位置可信
- 粒子四散散开：定位丢失，位姿不确定

✅ 用途：**调试定位状态**，只用于可视化，不参与导航规划计算。
> 硬件/仿真只输出 scan/odom，AMCL算法基于这些数据运算生成粒子云。

> 注意：slam_toolbox 在线建图模式**默认不输出particle_cloud**；只有在**定位模式（localization）**启用粒子滤波才会发布。

## 2. /waypoints
- 消息类型：`nav2_msgs/msg/WaypointArray`
- **发布方：Waypoint Follower（nav2_waypoint_follower节点）**
- 含义：**航路点/巡检点序列**
存储一组带编号的目标点位姿（x,y,yaw），用于多点巡逻任务。
- 你可以加载一组巡检waypoint列表，让机器人依次访问每个点；
- 到达一个waypoint后，自动接收下一个，同时把全部航路点发布在 `/waypoints`，供RViz可视化。

> 区分两个容易混淆的：
> - `/waypoints`：**全部任务巡检点列表（waypoint follower输出）**
> - `/plan`：**单次全局规划的路径（planner_server输出）**

## 汇总对比
| Topic | 发布节点 | 消息类型 | 作用 | 是否硬件输出 |
|---|---|---|---|---|
| `/particle_cloud` | amcl / slam_toolbox(定位模式) | PointCloud2 | 定位粒子可视化，看定位收敛情况 | ❌ 纯算法输出 |
| `/waypoints` | nav2_waypoint_follower | WaypointArray | 巡逻任务的一系列航路点 | ❌ 纯导航任务节点输出 |

## 数据流简要
1. `/particle_cloud`
`/scan` + `/odom` + `/map` → AMCL粒子滤波算法 → `/particle_cloud`
2. `/waypoints`
你加载预设巡检点位 → waypoint_follower节点 → 发布全部点位 `/waypoints`，同时依次发送目标点给Nav2规划器

## 小坑提醒
1. `/particle_cloud` Publisher=0：大概率没启动AMCL，或者slam_toolbox没有开启粒子定位模块（在线建图模式默认不开）
2. `/waypoints` Publisher=0：没有启动waypoint_follower，或者没有加载巡检任务点。单纯单点导航不会发布这个话题。

如果你需要，我可以把前面所有聊过的Nav2/SLAM话题合并成一张完整总表，方便你写方案文档。

===================================================

# ch15. /robot_description 和 /rosout

> 两个**都不是硬件直接发布**；仿真/真机环境都可以存在，全部是ROS2软件节点输出。

## 1. /robot_description
- 消息类型：`std_msgs/msg/String`
- **发布方：robot_state_publisher (RSP) 节点**
- 内容：**完整URDF/Xacro机器人模型文本字符串**
- 作用：
  1. RSP加载你的xacro文件，解析成完整urdf，把一整个xml字符串放到这个话题
  2. RViz订阅它，用来渲染机器人3D模型（轮子、雷达、连杆）
  3. RSP内部基于joint_states，计算并持续发布整套TF坐标变换（`base_link`、`laser_link`、车轮link等）

> 数据流：
> xacro文件 → robot_state_publisher → 发布字符串到 `/robot_description`
> joint_states话题 → RSP → 计算TF树

✅ 真机、Gazebo仿真**都要用这个话题**，属于机器人模型描述，和硬件传感器无关。
> 注意：**硬件不发布这个话题**，是软件加载模型文件后发布。

## 2. /rosout
- 消息类型：`rcl_interfaces/msg/Log`
- **发布方：所有ROS2节点（任何节点打印日志都会发消息到这里）**
- 作用：ROS2的**日志流话题**
  - 节点调用 `RCLCPP_INFO / RCLCPP_WARN / RCLCPP_ERROR` 打印日志时，会自动把日志消息发布到 `/rosout`
  - rqt_console 订阅 `/rosout`，图形化看所有节点的日志、报错、警告
  - 终端里`ros2 topic echo /rosout`，可以抓取全部节点输出的日志

> 特点：
> - 仿真环境、真机环境都存在
> - 不是某一个单独节点，**所有运行中的节点都可以作为Publisher**
> - 不参与机器人导航、建图控制，仅用于调试查看日志

---

## 汇总对比表
| Topic | 消息类型 | 发布方 | 硬件/仿真？ | 核心用途 |
|---|---|---|---|---|
| `/robot_description` | std_msgs/msg/String | robot_state_publisher | 软件节点（真机/仿真通用） | 传递机器人URDF模型，RViz显示模型、生成TF |
| `/rosout` | rcl_interfaces/msg/Log | 全部ROS2节点 | 软件日志系统（真机/仿真通用） | 收集所有节点打印的日志信息，调试用 |

## 补充你之前整套话题体系快速归类
1. **传感器原始数据话题**（硬件驱动节点输出：/scan、/imu、/odom、bumper_pointcloud）
2. **模型与TF相关**：`/robot_description`、`/joint_states` → robot_state_publisher
3. **SLAM建图输出**：`/map`、`/map_updates` → slam_toolbox
4. **Nav2导航代价地图**：`/global_costmap/*`、`/local_costmap/*` → costmap_2d节点
5. **Nav2规划路径**：`/plan`、`/local_plan` → planner_server / controller_server
6. **日志调试类**：`/rosout`

如果你需要，我可以把上面全部聊过的话题合并成一张**总清单**，方便你做项目文档。

===================================================

# ch16. /smoother_selector  /tf    /tf_static这些 topic 分别是做什么的？发布方是硬件还是仿真软件
 
> 全部**不是硬件直接输出**，由ROS2软件节点发布；仿真环境下就是仿真内的节点发布。

## 1. /smoother_selector
- 消息类型：`nav2_msgs/msg/SmootherSelector`
- **发布方：smoother_server（Nav2平滑器服务节点）**
- 作用：Nav2支持**多路径平滑器**（比如SimpleSmoother、SavitzkyGolaySmoother）。 这个话题广播当前正在使用哪一个路径平滑器。
- 用途：调试、监控；可以通过Nav2服务动态切换平滑算法。
- 什么时候才有数据：启动 `smoother_server` 才会发布，不启动则 Publisher=0。

> 补充：全局规划器输出的 `/plan` 原始路径可能拐点生硬、不符合机器人运动约束；**smoother 对全局路径做平滑**，输出平滑后的路径给控制器。

## 2. /tf
- 消息类型：`tf2_msgs/msg/TFMessage`
- **发布方：各个需要发布坐标变换的软件节点**（不是硬件，不是仿真本体，是ROS节点）
- 作用：**动态坐标变换**（随时间不断变化的坐标系关系）
典型例子：
`odom → base_link` 机器人底盘里程计变换，机器人一动，这个TF就持续更新。
还有激光雷达、IMU等动态外参也会放这里。
- 特点：高频持续发布，带时间戳；旧TF会过期丢弃。
- 来源举例：
  - diff_drive_controller / ekf/robot_localization：发布 odom → base_link
  - 雷达驱动：雷达如果有动态安装角度，会发布动态TF（极少）

## 3. /tf_static
- 消息类型：`tf2_msgs/msg/TFMessage`
- **发布方：static_transform_publisher / robot_state_publisher**
- 作用：**静态坐标变换**（固定不变，安装位置不会变）
例如：
`base_link → laser_link`、`base_link → imu_link`、`base_link → left_wheel_link`
雷达、IMU相对于机器人本体的安装位置，一旦装好就不变。
- 特点：只发一次，QoS `TRANSIENT_LOCAL`；新节点订阅时自动收到全部静态变换。
> ⚠️ URDF/Xacro 由 `robot_state_publisher` 解析，自动发布连杆之间的静态TF到 `/tf_static`。

## 一句话区分 tf / tf_static
- `/tf`：**动态变换**（会变：odom→base_link，机器人移动）
- `/tf_static`：**静态不变**（雷达、IMU相对于机身的安装位置）

## 发布者归属总结
| Topic | 发布者 | 硬件/仿真说明 |
|---|---|---|
| `/smoother_selector` | Nav2 smoother_server | 纯Nav2软件，和硬件无关；仿真/真机都可以启动 |
| `/tf` | 里程计融合/ros2_control等节点 | 节点读取传感器原始数据后**计算生成**TF；硬件只输出原始传感器数据，不会直接发TF |
| `/tf_static` | robot_state_publisher / static_transform_publisher | 来自URDF模型，**完全软件定义**，和硬件无关，真机、仿真共用同一套URDF |

## 串联你整套机器人数据流
1. URDF → robot_state_publisher → `/tf_static`（雷达、IMU固定安装位置）
2. 轮式编码器/IMU → 里程融合节点 → 计算 odom→base_link → `/tf`
3. planner_server生成原始全局路径 → smoother_server平滑路径；同时发布 `/smoother_selector` 告知当前使用哪个平滑器

## 你之前踩坑回顾
`No transform from [right_wheel_link] to [base_footprint]`
👉 本质就是 **URDF的joint/link定义问题，robot_state_publisher没有把对应变换发布到 `/tf_static`**。

要不要我把Nav2+TF+SLAM整套核心话题整理成一张总表，包含发布方、作用，方便你做调试文档？

===================================================