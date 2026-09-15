# 1.Ubuntu24.04 + ROS2 Jazzy（默认版本）查找ROS2头文件

> 
> Ubuntu24 官方推荐ROS2版本是 **Jazzy Jalisco**，系统安装的ROS2包全部放在 `/opt/ros/jazzy/` 下面。
> 你前面要看的 `rviz_common/properties/ros_topic_property.hpp` 就在这套目录内。

## 一、系统安装的ROS2（apt安装，例如 rviz_common、rclcpp）

### 1. 基础总路径

```
/opt/ros/jazzy/include/
```

每个功能包单独一个子文件夹：

```
/opt/ros/jazzy/include/rviz_common/rviz_common/properties/ros_topic_property.hpp
/opt/ros/jazzy/include/rclcpp/rclcpp.hpp
```

### 2. 命令：查找某个包的安装前缀（最推荐）

```
# 先source ros环境，必须执行
source /opt/ros/jazzy/setup.bash

# 获取包的根目录
ros2 pkg prefix rviz_common
```

输出示例：`/opt/ros/jazzy`
👉 头文件就在 `${prefix}/include/包名/`

直接定位 `rviz_common` 的头文件目录：

```
# 拼接得到完整include目录
echo $(ros2 pkg prefix rviz_common)/include/rviz_common
```

### 3. find命令：直接搜索某个hpp头文件（快速定位文件）

```
# 全局搜索 ros_topic_property.hpp
find /opt/ros/jazzy -name "ros_topic_property.hpp"
```

输出结果：

```
/opt/ros/jazzy/include/rviz_common/rviz_common/properties/ros_topic_property.hpp
```

### 4. locate快速搜索（先更新数据库）

```
sudo updatedb
locate ros_topic_property.hpp
```

## 二、自己colcon工作空间编译出来的包（Autoware、你的机器人代码）

假设你的工作空间是 `~/colcon_ws`
编译后**安装后的头文件**位置：

```
~/colcon_ws/install/<package_name>/include/
```

同样可以用`ros2 pkg prefix`，source你的工作空间之后，自动优先找到工作空间内的包：

```
source ~/colcon_ws/install/setup.bash
ros2 pkg prefix autoware_adapi_adaptors
```

> 
> 源码位置（未编译的原始hpp）：`~/colcon_ws/src/xxx_package/include/`

## 三、查看环境变量（CMake读取的include路径）

```
# 查看ament的所有搜索前缀
echo $AMENT_PREFIX_PATH
```

`AMENT_PREFIX_PATH` 是ROS2的核心环境变量，ament_cmake会从这里自动查找所有包的include目录。

## 四、VSCode配置（解决红色波浪找不到头文件）

`.vscode/c_cpp_properties.json`

```
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**",
                "/opt/ros/jazzy/include/**"
            ],
            "compilerPath": "/usr/bin/g++",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "linux-gcc-x64"
        }
    ],
    "version":4
}
```

> 
> 注意：**必须在source过ROS环境的终端启动VSCode**，否则AMENT_PREFIX_PATH不会生效。

## 五、针对你正在看的 rviz_common/properties/ros_topic_property.hpp

```
source /opt/ros/jazzy/setup.bash
find /opt/ros/jazzy -name "ros_topic_property.hpp"
# 打开查看
cat $(find /opt/ros/jazzy -name "ros_topic_property.hpp")
```

## 六、区分两个概念

1. **源码文件（src目录）**：如果你是用源码编译安装的ros2/rviz，可以看到`.cpp`+`.hpp`原始源码；apt安装的二进制包**不带src源码，只有编译后的头文件和库**。
2. **安装后头文件（include目录）**：apt安装包自带，用来`#include`编译项目。

> 
> 如果想看rviz_common完整源码，需要单独下载源码：`git clone [https://github.com/ros2/rviz](https://github.com/ros2/rviz)`

## 七、常用命令速查表

| 功能 | 命令 |
| --- | --- |
| 获取包安装根目录 | `ros2 pkg prefix rviz_common` |
| 搜索头文件 | `find /opt/ros/jazzy -name "*.hpp" |
| 查看所有ROS包列表 | `ros2 pkg list` |
| 查看包信息 | `ros2 pkg xml rviz_common` |

如果你需要，我可以直接打开`ros_topic_property.hpp`的源码，给你逐行解析。