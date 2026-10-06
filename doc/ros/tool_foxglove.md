# ch1. Foxglove（Foxglove Studio）
机器人/ROS生态里的现代化可视化、数据观测平台，对标 RViz / RViz2，非常适合你的四轮巡逻机器人项目调试。
> 单词本身 foxglove 原意是**毛地黄**（有毒植物），工具取名来自这个植物名。

## ✅ 核心定位
Foxglove = 机器人数据观测平台，支持**实时在线看机器人** + **离线回放bag/MCAP数据包**，支持 ROS1 / ROS2，有网页版和桌面客户端（Linux/Windows/macOS）。

## 🧩 关键组件
1. **Foxglove Studio**
    桌面端可视化软件（早期开源 1.x，新版本转为商业产品，社区仍可用旧开源版）。
    模块化面板：3D场景、图像、点云、TF、地图、话题曲线、消息查看、ROS参数、Topic Graph，支持自定义JS脚本做数据转换，也能开发自定义扩展面板。
2. **foxglove_bridge**
    C++高性能WebSocket桥接包，部署在机器人端，替代 rosbridge，低延迟，支持大流量点云/图像，把ROS2数据推给Foxglove客户端，是ROS2项目标配。
3. **MCAP**
    Foxglove自研的机器人日志格式，比rosbag性能更好，压缩率高，现在很多自动驾驶、机器人项目都用MCAP替代rosbag。

## 📌 和 RViz2 的对比（对你巡逻机器人项目）
| Foxglove | RViz2 |
|---|---|
| 跨平台，Windows也能连机器人看数据 | 原生Linux优先，Windows支持差 |
| 时间轴同步回放，多传感器时间对齐很强，适合事后分析bag | 实时可视化强，回放功能弱 |
| 网页端可用，远程浏览器查看机器人 | 只能本地桌面 |
| 面板自由拖拽布局，可保存工程配置 | 布局能力弱一些 |
| 新商业版收费，旧Studio 1.x开源 | 完全开源免费 |
| 支持MCAP + rosbag | 只支持rosbag |

## 🚀 最简使用流程（ROS2）
1. 机器人端安装：`foxglove_bridge`
2. 启动bridge节点
3. 本地电脑打开 Foxglove Studio，连接 `ws://机器人IP:8765`
4. 添加3D面板、Image、Plot，查看点云、相机、Odometry、TF、路径

> 离线场景：直接拖入 `.mcap` / `.bag` 文件回放，不需要启动ROS。

## ⚠️ 补充
Foxglove 不是ROS官方工具，是独立创业公司产品；**1.87之后版本不再开源**，很多开发者继续使用 v1.87 开源版。

## 适合你的场景
巡逻机器人：激光雷达、相机、IMU、定位、规划路径，**在线调试 + 事故数据包离线复盘**，远程网页查看机器人状态非常方便。

要不要我给你一段 foxglove_bridge 的 ROS2 launch 文件，直接放到你的巡逻机器人工程里用？