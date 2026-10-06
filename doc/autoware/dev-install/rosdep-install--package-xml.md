# ch01.执行rosdep install时会扫描package.xml并自动安装其中的依赖
 
**执行 `rosdep install` 时，会扫描 package.xml，自动安装里面声明的依赖，但有几个硬性限制**。

> 
> rosdep 不是万能的，它只安装**rosdistro索引里登记的依赖**，底层调用 `apt`，**不会自动编译源码库**（比如PCL源码编译版不行）

## 1、package.xml里哪些标签会被rosdep识别

```
<build_depend>xxx</build_depend>        # 编译依赖
<exec_depend>xxx</exec_depend>          # 运行依赖
<depend>xxx</depend>                    # 同时包含build+exec
<buildtool_depend>xxx</buildtool_depend> # 编译工具，如catkin
```

只要写在上面标签里，`rosdep install` 就会读取。

## 2、关键限制（非常重要，踩坑点）

1. **依赖名字必须是 rosdep 的key，不是C++库名**
比如你在package.xml写：

```
<depend>libpcl-dev</depend>
```

✅ 可以，rosdistro里有映射，rosdep会调用`apt install libpcl-dev`
但如果你**自己源码编译的PCL**，rosdep不认，它只管apt包。

> 
> ❌ 如果你写的库不在rosdistro数据库内 → rosdep找不到，不会自动装。

2. **`--ignore-src` 参数含义（工作空间常用命令）**

```
rosdep install --from-paths src --ignore-src -y
```

`--ignore-src`：**如果这个依赖包本身就在你本地src源码文件夹里，就不去安装它**，只装系统依赖。

> 
> 也就是：工作区内自己写的ROS源码包，rosdep不会去下载安装，只处理系统库/已经打包好的ROS deb包。

3. **只安装缺失的，已经装好的会跳过**
rosdep会先检查是否存在，存在就不重复安装。
4. **rosdep不会编译你的ROS包！**
rosdep只管**安装依赖**；编译包仍然要靠 `catkin_make / colcon build`

## 3、实操流程（标准）

```
# 首次初始化（只跑一次）
sudo rosdep init
rosdep update

# 一键安装src下所有包的依赖
rosdep install --from-paths src --ignore-src -y
```

## 4、你关心的PCL例子

package.xml写：

```
<depend>libpcl-dev</depend>
```

运行rosdep install，**会自动apt安装libpcl-dev**（就是我们前面说的apt版PCL）。
但如果你需要**源码编译的PCL**，rosdep无能为力，需要自己手动编译。

## 5、调试命令（看rosdep会执行什么，不真安装）

```
rosdep install --from-paths src --ignore-src -s
```

加 `-s` 模拟，打印出将要执行的apt命令，方便排查。

## 快速总结

✅ package.xml写对依赖 + rosdistro数据库有映射 → rosdep自动apt安装
❌ 自定义源码库、不在rosdistro里的库 → rosdep不会管，需要手动装

要不要我给你一段完整可直接复制的package.xml示例（包含PCL）？