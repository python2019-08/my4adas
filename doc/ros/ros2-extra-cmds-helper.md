#  ch1.问题2：`ros2 control -h` 报错

```
ros2: error: argument Call `ros2 <command> -h` for more detailed usage.: invalid choice: 'control'
```

✅ 原因：**`ros2 control` 不是内置主命令，它是 ros2_control 提供的扩展插件，包没安装就没有这个子命令**

### 安装 ros2_control 全套（机器人底盘控制必需）

```sh
sudo apt install ros-jazzy-ros2-control ros-jazzy-ros2-controllers ros-jazzy-gz-ros2-control
```

安装完成后，**重新 source Jazzy 环境**（关键！不source不会加载新插件）

```sh
source /opt/ros/jazzy/setup.bash
# 再测试
ros2 control -h
```

> 
> 💡 补充：`ros2 control` 常用子命令
> 
> 
> - `ros2 control list_controllers`
> - `ros2 control list_hardware_interfaces`
> - `ros2 control switch_controllers`

---

# 快速自检清单（Jazzy + Harmonic + ros2_control）

```
# 1. 加载环境
source /opt/ros/jazzy/setup.bash

# 2. 检查gz仿真
gz --version

# 3. 检查ros2 control插件
ros2 control -h

# 4. 查看ros_gz包是否存在
ros2 pkg list | grep ros_gz
```

---

# 额外提醒（适配你的巡逻机器人+Autoware）

`ros-jazzy-gz-ros2-control` 就是 **Gazebo Harmonic 里的 ros2_control 硬件接口**，用来在仿真里驱动差速底盘，替代老 classic 的 `gazebo_ros2_control`。
SDF模型里插件标签示例：

```
<plugin filename="libgz_ros2_control-system.so" name="gz::sim::systems::Ros2ControlSystem">
```

你要的话我可以给一份 **差速轮机器人SDF完整示例，自带gz_ros2_control配置**，直接跑仿真底盘。