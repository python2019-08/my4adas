# 1.ROS2 plugin vs component 区别
一句话总结：
> **Component 是一种特殊的 Plugin（pluginlib 插件）；Plugin 是通用动态库加载框架，Component 只是插件在节点层面的一个应用。**

## 1. 基础概念
### Plugin（pluginlib）
`pluginlib`：ROS2通用**C++插件框架**。
- 原理：编译成独立`.so`共享库，运行时动态加载，不需要编译链接到主程序。
- 任意C++类都可以做成插件：规划器、代价地图、传感器驱动、滤波器、控制器等。
- 需要：`plugin.xml` + `ament_register_plugins` + 宏注册插件类。
- 基类**可以是任意自定义基类**，不限于节点。

### Component（组件）
Component = **基于pluginlib实现的、专门用来封装ROS2节点的插件**
- 基类固定：`rclcpp_components::NodeFactory`
- 插件内部封装一个继承`rclcpp::Node`的节点类。
- 组件不能单独作为独立可执行文件运行；**必须加载到组件容器进程（component container）**。
- 容器可以是：
  - `rclcpp_components::ComponentContainer`（launch里的容器）
  - `ros2 component standalone`（自动创建容器）
  - `ros2 component load` 加载到正在运行的容器

> ✅ 所有Component都是Plugin；但Plugin不一定是Component。

## 2. 对比表
|对比项|Plugin（pluginlib通用插件）|Component（节点组件插件）|
|---|---|---|
|用途|任意C++可扩展类：算法、驱动、规划器|封装ROS节点（publisher/subscriber/service等）|
|基类|用户自定义基类|固定：`rclcpp_components::NodeFactory`|
|运行载体|宿主程序（自己写的主程序dlopen加载so）|Component容器进程|
|是否独立进程|否，加载进宿主进程|否，多个组件可以共享同一个容器进程（共享内存，减少进程开销）|
|注册宏|pluginlib宏`PLUGINLIB_EXPORT_CLASS`|`RCLCPP_COMPONENTS_REGISTER_NODE`（本质封装pluginlib）|
|查看命令|`ros2 plugin list`|`ros2 plugin list --base-class rclcpp_components::NodeFactory` / `ros2 component list`|
|Python支持|❌ Python不支持pluginlib插件|❌ Python组件是另一套机制，不是pluginlib|

## 3. 组件（Component）核心优势
1. **多节点跑在同一个进程**：减少进程间DDS通信开销、降低内存占用。
2. **动态加载/卸载**：运行时往容器里新增/移除节点，不用重启程序。
```bash
# 示例：容器启动后动态加载组件
ros2 component load /my_container my_pkg my_pkg::MyNode
ros2 component unload /my_container 1
```

## 4. 代码层面简单看关系
组件注册宏底层等价于：
```cpp
// RCLCPP_COMPONENTS_REGISTER_NODE(MyNode)
// 本质就是用pluginlib导出一个NodeFactory工厂类，工厂负责new出你的节点
PLUGINLIB_EXPORT_CLASS(..., rclcpp_components::NodeFactory)
```
所以在`plugin.xml`里，Component插件声明的基类就是`rclcpp_components::NodeFactory`。

## 5. 使用场景区分
### 用通用 Plugin（pluginlib）
- 导航2的插件：规划器、代价地图层、控制器
- 点云滤波器、图像算法插件
- 硬件驱动抽象，运行时选择不同驱动实现
> 宿主程序自己调用`pluginlib::ClassLoader`加载插件类。

### 用 Component
- 多个轻量ROS节点，希望**同进程运行**降低开销
- 需要**动态启停节点**，不重启整个程序
> 典型场景：感知链路，多个小节点打包进同一个容器。

## 6. 配套CLI区分
```bash
# 查看全部插件（包含组件插件和其他算法插件）
ros2 plugin list

# 只查看组件插件（NodeFactory类型）
ros2 plugin list --base-class rclcpp_components::NodeFactory

# 查看正在运行容器里加载了哪些组件实例
ros2 component list
```

## 7. 容易踩的坑
1. 很多人混淆：**Component ≠ 独立可执行程序**。
   - `ros2 run`运行的是独立节点（单独进程）
   - Component是so库，必须加载进容器。
2. Python组件**不是pluginlib**，不能用`ros2 plugin list`查到。
3. 通用pluginlib插件不能直接用`ros2 component load`加载；只有NodeFactory类型插件才能作为组件加载。

---

### 一句话总结再浓缩
- **pluginlib：通用动态C++类加载框架，万能扩展机制。**
- **Component：pluginlib的一个特例，专门用来封装ROS节点，支持加载到共享容器进程。**

如果你想，我可以写极简最小demo对比：一个普通pluginlib插件 + 一个Component组件插件。