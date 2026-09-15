# 0. ros2命令列表

```sh
$ ros2 -h
usage: ros2 [-h] [--use-python-default-buffering] Call `ros2 <command> -h` for more detailed usage. ...

ros2 is an extensible command-line tool for ROS 2.

options:
  -h, --help            show this help message and exit
  --use-python-default-buffering
                        Do not force line buffering in stdout and instead use the python default buffering, which might be affected by PYTHONUNBUFFERED/-u and depends on whatever stdout is
                        interactive or not

Commands:
  action     Various action related sub-commands
  bag        Various rosbag related sub-commands
  component  Various component related sub-commands
  daemon     Various daemon related sub-commands
  doctor     Check ROS setup and other potential issues
  interface  Show information about ROS interfaces
  launch     Run a launch file
  lifecycle  Various lifecycle related sub-commands
  multicast  Various multicast related sub-commands
  node       Various node related sub-commands
  param      Various param related sub-commands
  pkg        Various package related sub-commands
  plugin     Various plugin related sub-commands
  run        Run a package specific executable
  security   Various security related sub-commands
  service    Various service related sub-commands
  topic      Various topic related sub-commands
  wtf        Use `wtf` as alias to `doctor`

  Call `ros2 <command> -h` for more detailed usage.

```
===============================================

# 1.

```sh
$ ros2 action -h
usage: ros2 action [-h] Call `ros2 action <command> -h` for more detailed usage. ...

Various action related sub-commands

options:
  -h, --help            show this help message and exit

Commands:
  info       Print information about an action
  list       Output a list of action names
  send_goal  Send an action goal
  type       Print a action's type
```


===============================================
# 1. ros2 bag 命令详解（对应你输出的帮助信息）
```sh
$ ros2 bag -h
usage: ros2 bag [-h] Call `ros2 bag <command> -h` for more detailed usage. ...
Various rosbag related sub-commands
options:
  -h, --help            show this help message and exit
Commands:
  burst    Burst data from a bag
  convert  Given an input bag, write out a new bag with different settings
  info     Print information about a bag to the screen
  list     Print information about available plugins to the screen
  play     Play back ROS data from a bag
  record   Record ROS data to a bag
  reindex  Reconstruct metadata file for a bag
```


> ROS2 的 bag 工具，默认格式 **MCAP**，也兼容旧 rosbag1 的 `.bag`

```bash
ros2 bag -h
```
## 子命令一览
|命令|作用|
|---|---|
|`burst`|高速爆发式播放bag，不按时间戳，快速把消息全部发出去|
|`convert`|格式转换/重编码bag（mcap↔bag，修改配置）|
|`info`|查看bag文件信息：话题、消息类型、时长、大小|
|`list`|列出可用的bag插件|
|`play`|回放bag文件|
|`record`|录制话题保存为bag|
|`reindex`|修复损坏bag的元数据索引（bag文件损坏时用）|

---

## 常用实操命令
### 1. 查看bag信息
```bash
ros2 bag info xxx.mcap
```
输出：文件时长、所有话题、消息数量、消息类型。

### 2. 录制 bag
```bash
# 录制指定话题
ros2 bag record -o my_bag /topic1 /topic2

# -o 指定输出文件夹，ros2 bag会自动生成文件夹，里面是mcap文件
# 录制全部话题
ros2 bag record -a
```
> ⚠️ ROS2 `ros2 bag record` **输出是文件夹，不是单个文件**，文件夹里面存放 `.mcap` 数据文件。

### 3. 回放 bag
```bash
# 正常回放
ros2 bag play my_bag

# 倍速回放 2倍速
ros2 bag play my_bag --rate 2.0

# 循环播放
ros2 bag play my_bag --loop

# 只回放部分话题
ros2 bag play my_bag --topics /scan /camera/image_raw
```

### 4. 格式转换 convert
把旧 ROS1 `.bag` 转 ROS2 mcap：
```bash
ros2 bag convert -i input.bag -o output.mcap
```

### 5. 修复损坏bag reindex
如果bag打不开、提示元数据损坏：
```bash
ros2 bag reindex my_bag
```

### 6. burst 爆发模式
不遵守时间戳，尽可能快把所有消息推送出去，适合做压力测试：
```bash
ros2 bag burst my_bag
```

---

## 小提示
1. ROS1：`rosbag record` 输出**单个 `.bag` 文件**
2. ROS2：`ros2 bag record` 输出**文件夹**，里面是 `.mcap` 文件
3. 查看帮助：每个子命令加 `-h`，例如
```bash
ros2 bag play -h
ros2 bag record -h
```

### 常见坑
- 回放时**时间不同步**：`ros2 bag play` 默认使用bag内部时间；如果你的节点用系统时间，需要加 `--clock` 参数，会发布 `/clock` 话题给ROS2时间系统。
```bash
ros2 bag play my_bag --clock
```

如果你需要，我可以给你一份完整 ros2 bag 常用命令速记。

===============================
===============================
# 2.ros2 component 命令详解
```sh
$ ros2 component  -h
usage: ros2 component [-h] Call `ros2 component <command> -h` for more detailed usage. ...
Various component related sub-commands
options:
  -h, --help            show this help message and exit
Commands:
  list        Output a list of running containers and components
  load        Load a component into a container node
  standalone  Run a component into its own standalone container node
  types       Output a list of components registered in the ament index
  unload      Unload a component from a container node
```

> **Component（组件）**：ROS2 把节点编译成共享库插件，可以**动态加载到容器进程**，多个组件跑在同一个进程内，开启**进程内通信 intra‑process**，避免DDS序列化拷贝，降低延迟、节省CPU内存，相当于ROS1的Nodelet。
> 容器：`component_container`，是宿主进程，里面跑`ComponentManager`服务，负责加载/卸载组件。

```bash
ros2 component -h
```

## 子命令列表
|命令|作用|
|---|---|
|`list`|列出正在运行的组件容器，以及容器内部已经加载的组件（组件ID、节点名）|
|`load`|把组件插件**动态加载**到已启动的容器中（运行时加载，不用重启）|
|`standalone`|直接运行组件，自动创建一个独立容器进程（等价于`ros2 run`）|
|`types`|列出系统中所有已注册的组件插件（包名::类名），来自ament索引|
|`unload`|根据组件ID，从容器卸载组件，停止该节点|

---

## 实操完整流程（示例）
### 1. 启动组件容器（第一个终端）
```bash
# 单线程容器（默认，适合控制环）
ros2 run rclcpp_components component_container
# 容器节点名字默认 /ComponentManager
```
> 另外两种容器：
> - `component_container_mt`：多线程executor，适合大量回调
> - `component_container_isolated`：每个组件独立executor，隔离调度

### 2. 查看容器状态
```bash
ros2 component list
```
输出示例：
```
/ComponentManager
```
此时容器是空的，没有加载组件。

### 3. 加载组件（第二个终端）
格式：`ros2 component load <容器节点名> <包名> <插件类名>`
```bash
# 加载demo里的talker组件
ros2 component load /ComponentManager composition composition::Talker
```
返回：
```
Loaded component 1 into '/ComponentManager' container node as '/talker'
```
> 组件会得到一个**唯一ID（1）**，卸载要用这个ID。

可以指定节点名、命名空间、参数：
```bash
ros2 component load /ComponentManager composition composition::Talker \
  --node-name my_talker \
  --node-namespace /ns \
  --param "my_param:=100"
```

### 4. 查看已加载组件
```bash
ros2 component list
```
看到容器内的组件ID和节点名称。

### 5. 卸载组件（用组件ID）
```bash
ros2 component unload /ComponentManager 1
```

### 6. 直接独立运行组件（standalone）
不用手动启动容器，自动新建一个进程运行组件：
```bash
ros2 component standalone composition composition::Talker
```
等价于 `ros2 run composition talker`，但底层是组件机制。

### 7. 查看系统所有可用组件插件
```bash
ros2 component types
```
输出：`package_name::ClassName`，所有编译注册过的组件。

---

## 关键概念对比
1. **普通节点（ros2 run）**：每个节点是独立进程，节点之间通信走DDS，需要序列化拷贝。
2. **Component组件**：编译为.so共享库，加载进同一个容器进程；同容器内组件可以开启`use_intra_process_comms`，直接传内存指针，**零拷贝**，大幅降低图像、点云这类大消息延迟。

> 注意：组件**只支持C++**；Python没有组件机制。

## 常见坑
1. 加载组件时，容器必须先运行，否则报错。
2. `unload`使用**组件ID**，不是节点名字。
3. 组件要开启进程内通信，需要在load时加参数：
```bash
ros2 component load /ComponentManager composition composition::Talker --extra-args use_intra_process_comms:=true
```
4. 组件写代码时，构造函数参数必须是`const rclcpp::NodeOptions & options`，不能是普通构造函数，还要注册插件宏`RCLCPP_COMPONENTS_REGISTER_NODE`。

## 配套launch用法
launch文件中使用`ComposableNodeContainer`+`ComposableNode`，在启动时直接把多个组件加载进容器，是工程中最常用的方式，不用命令行load/unload。

如果你需要，我可以给你一份极简C++组件示例代码。

===============================
===============================
# 3.ros2 daemon 命令详解
```sh
$ ros2  daemon -h
usage: ros2 daemon [-h] Call `ros2 daemon <command> -h` for more detailed usage. ...

Various daemon related sub-commands

options:
  -h, --help            show this help message and exit

Commands:
  start   Start the daemon if it isn't running
  status  Output the status of the daemon
  stop    Stop the daemon if it is running

  Call `ros2 daemon <command> -h` for more detailed usage.
```


> **ros2 daemon**：ROS2 的后台守护进程（`ros2-daemon`），负责：
> - 缓存节点、话题、服务、动作的发现信息
> - 加速 `ros2 topic list`、`ros2 node list` 等命令查询速度
> - 管理DDS发现，减少重复的网络扫描
>
> 只要执行任意`ros2`命令，daemon会自动后台启动；但有时候会卡死、缓存异常，就需要手动启停。

```bash
ros2 daemon -h
```

## 子命令
|命令|作用|
|---|---|
|`start`|启动daemon，如果已经运行则无操作|
|`status`|查看daemon运行状态（运行/未运行）|
|`stop`|停止daemon后台进程|

### 常用实操
#### 1. 查看状态
```bash
ros2 daemon status
```
输出示例：
```
daemon is running
```
或者 `daemon is not running`

#### 2. 停止守护进程
```bash
ros2 daemon stop
```
> ⚠️ 停止后，再执行`ros2 topic list`这类命令，会**重新自动拉起daemon**。

#### 3. 重启daemon（遇到发现问题最常用）
当出现：
- `ros2 node list`看不到节点
- 话题找不到、服务发现异常
- 命令执行很慢

执行重启：
```bash
ros2 daemon stop
ros2 daemon start
```

> 小技巧：**彻底清缓存**，停止daemon之后删除缓存目录：
```bash
rm -rf ~/.ros/daemon
```

---

## 关键知识点
1. **daemon是可选的吗？**
不强制。不运行daemon，ros2命令依然可以工作，但是会每次重新做DDS发现，命令会变得很慢。

2. 什么时候要重启daemon？
- 跨机器通信发现不到节点
- 重启节点后，`ros2 topic list`还显示旧话题
- 命令卡住、超时
> 很多ROS2诡异的发现问题，重启daemon就能解决。

3. 进程名：
```bash
ps aux | grep ros2_daemon
```

4. 注意：**daemon只作用于命令行工具**，不会影响节点之间DDS通信本身。节点之间互相发现不受daemon控制；daemon只是给`ros2 cli`工具做缓存。

---

## 快速排错模板
遇到ROS2命令行发现异常：
```bash
# 1. 停止daemon
ros2 daemon stop
# 2. 清除缓存
rm -rf ~/.ros/daemon
# 3. 重新启动
ros2 daemon start
# 4. 验证
ros2 node list
```

> 补充：ROS1没有ros2 daemon这套机制。

如果你需要，我可以整理一份ROS2常用命令速查表（ros2 topic/node/service/daemon/component/bag/lifecycle）汇总。

=====================
# 4.ros2 doctor 详解
```sh
$ ros2 doctor -h
usage: ros2 doctor [-h] [--report | --report-failed] [--exclude-packages] [--include-warnings] Call `ros2 doctor <command> -h` for more detailed usage. ...

Check ROS setup and other potential issues

options:
  -h, --help            show this help message and exit
  --report, -r          Print all reports.
  --report-failed, -rf  Print reports of failed checks only.
  --exclude-packages, -ep
                        Exclude package checks or report.
  --include-warnings, -iw
                        Include warnings as failed checks. Warnings are ignored by default.

Commands:
  hello  Check network connectivity between multiple hosts

  Call `ros2 doctor <command> -h` for more detailed usage.
```


> **ros2 doctor**（别名 `ros2 wtf`）：ROS2 系统诊断工具，对标 ROS1 的 `roswtf`。
> 自动扫描：网络、环境变量、RMW中间件、ROS版本、包依赖、QoS兼容性、运行时节点话题状态，找出配置隐患、多机通信问题、QoS不匹配等问题。

```bash
ros2 doctor -h
```

## 参数选项
| 参数 | 简写 | 作用 |
|---|---|---|
| `--report` | `-r` | 输出**全部完整诊断报告**（网络、平台、RMW、ROS信息、话题列表） |
| `--report-failed` | `-rf` | **只输出失败/警告项**，过滤掉正常通过的检查，方便排错 |
| `--exclude-packages` | `-ep` | 跳过包相关检查，不扫描软件包状态 |
| `--include-warnings` | `-iw` | 将警告当作失败项处理；默认警告不会标记为FAIL |

## 子命令
`hello`：**多机连通性测试**，用来验证不同机器之间ROS2通信是否通，同时测试DDS话题通信 + UDP组播底层网络。
```bash
ros2 doctor hello -h
```

---

## 常用实操命令
### 1. 快速做基础检查（只输出问题，不输出全部详情）
```bash
ros2 doctor
```
> 输出警告/错误，正常则提示 `All checks passed!`

### 2. 输出完整详细报告（排查问题首选，提交issue时复制这份）
```bash
ros2 doctor --report
```
报告分为几大块：
- NETWORK CONFIGURATION：网卡、IP、组播、防火墙
- PLATFORM INFORMATION：系统版本
- RMW MIDDLEWARE：当前DDS实现（CycloneDDS / Fast‑DDS）
- ROS 2 INFORMATION：版本、环境变量`ROS_DOMAIN_ID`等
- TOPIC LIST：运行中的话题，检测QoS不兼容、无订阅的发布者等问题

### 3. 只看失败项，精简输出
```bash
ros2 doctor --report-failed
# 同时把警告也当成失败
ros2 doctor --report-failed --include-warnings
```

### 4. 跳过包检查，加快运行
```bash
ros2 doctor --report --exclude-packages
```

### 5. 多机连通测试 `ros2 doctor hello`
> 两台机器都要执行，测试互相能否听见对方。
```bash
ros2 doctor hello
```
- 发布`/canyouhearme`话题，同时发送UDP组播包
- 分别统计：**DDS话题层能看到哪些主机**、**底层UDP组播能看到哪些主机**
- 如果UDP组播通，但ROS话题看不到：多半是`ROS_DOMAIN_ID`、RMW、防火墙问题
- 如果UDP组播都不通：是底层网络/交换机/防火墙问题

> 可加超时参数：`ros2 doctor hello --timeout 15`

---

## 常见会报的问题
1. `ROS_DOMAIN_ID` 未设置（多机务必统一）
2. 多网卡，DDS绑定网卡异常
3. QoS不兼容：publisher是best‑effort，subscriber是reliable，消息收不到，doctor会报出对应话题
4. 发布者没有订阅者（话题只有发布没有订阅）
5. 防火墙拦截DDS组播端口
6. RMW中间件环境变量异常

## 排错工作流（ROS2通信异常）
1. 先运行 `ros2 doctor --report-failed`，看环境/网络警告
2. 多机问题：`ros2 doctor hello` 区分是网络层还是DDS配置问题
3. 怀疑daemon缓存：`ros2 daemon stop && ros2 daemon start`
4. 查看QoS不匹配：`ros2 doctor --report` 看QOS COMPATIBILITY LIST

> 小提示：`ros2 wtf` 等价于 `ros2 doctor`，是别名，很多人习惯敲wtf快速查错。

如果你需要，我可以把前面所有ros2 cli（topic/node/service/daemon/component/bag/lifecycle/doctor）整理一份完整速查表。

========================================
========================================

# 5.ros2 launch 命令详解
```sh
$ ros2 launch -h
usage: ros2 launch [-h] [-n] [-d] [-p | -s] [-a]
                   [--launch-prefix LAUNCH_PREFIX]
                   [--launch-prefix-filter LAUNCH_PREFIX_FILTER]
                   package_name [launch_file_name]
                   [launch_arguments ...]

Run a launch file

positional arguments:
  package_name          Name of the ROS package which contains the
                        launch file
  launch_file_name      Name of the launch file
  launch_arguments      Arguments to the launch file; '<name>:=<value>'
                        (for duplicates, last one wins)

options:
  -h, --help            show this help message and exit
  -n, --noninteractive  Run the launch system non-interactively, with no
                        terminal associated
  -d, --debug           Put the launch system in debug mode, provides
                        more verbose output.
  -p, --print, --print-description
                        Print the launch description to the console
                        without launching it.
  -s, --show-args, --show-arguments
                        Show arguments that may be given to the launch
                        file.
  -a, --show-all-subprocesses-output
                        Show all launched subprocesses' output by
                        overriding their output configuration using the
                        OVERRIDE_LAUNCH_PROCESS_OUTPUT envvar.
  --launch-prefix LAUNCH_PREFIX
                        Prefix command, which should go before all
                        executables. Command must be wrapped in quotes
                        if it contains spaces (e.g. --launch-prefix
                        'xterm -e gdb -ex run --args').
  --launch-prefix-filter LAUNCH_PREFIX_FILTER
                        Regex pattern for filtering which executables
                        the --launch-prefix is applied to by matching
                        the executable name.
```

> `ros2 launch` 用来启动 launch 文件（`.launch.py` / `.launch.xml` / `.launch.yaml`），可以批量启动节点、组件、设置参数、启动容器、配置环境，是ROS2工程最常用启动方式。

```bash
ros2 launch -h
```

## 位置参数
| 参数 | 说明 |
|---|---|
| `package_name` | launch文件所在的ROS包名 |
| `launch_file_name` | launch文件名（`xxx.launch.py`） |
| `launch_arguments` | 传入launch的参数，格式 `key:=value`，重复参数取最后一个 |

## 选项参数
| 参数 | 简写 | 作用 |
|---|---|---|
| `-n, --noninteractive` | | 非交互模式，没有终端交互，适合后台运行 |
| `-d, --debug` | | debug调试模式，输出大量详细日志，排查launch报错 |
| `-p, --print` | | **只打印launch描述，不实际启动**，看会启动哪些节点、参数 |
| `-s, --show-args` | | 列出该launch文件支持哪些输入参数 |
| `-a, --show-all-subprocesses-output` | | 强制把所有子进程的打印全部输出到终端，覆盖launch内部的输出配置 |
| `--launch-prefix` | | 给所有可执行程序加前缀命令，例如gdb、valgrind、xterm；带空格要用引号包裹 |
| `--launch-prefix-filter` | | 正则过滤，只对匹配名字的程序应用上面的prefix |

---

## 常用实操示例
### 1. 启动launch文件
```bash
# 格式：ros2 launch 包名 launch文件
ros2 launch demo_nodes_cpp talker_listener.launch.py
```

### 2. 传入launch参数
```bash
ros2 launch demo_nodes_cpp talker_listener.launch.py chatter:=hello_world
```

### 3. 查看launch文件支持哪些参数
```bash
ros2 launch demo_nodes_cpp talker_listener.launch.py -s
```

### 4. 预览launch会启动什么，不真正运行
```bash
ros2 launch demo_nodes_cpp talker_listener.launch.py -p
```

### 5. debug模式，打印详细日志，定位launch报错
```bash
ros2 launch demo_nodes_cpp talker_listener.launch.py -d
```

### 6. 所有子进程输出全部打印到终端（调试节点打印）
```bash
ros2 launch demo_nodes_cpp talker_listener.launch.py -a
```

### 7. 用gdb调试launch里的节点（launch‑prefix）
> 给所有程序套gdb调试
```bash
ros2 launch demo_nodes_cpp talker_listener.launch.py --launch-prefix 'gdb -ex run --args'
```

> 只对`talker`这个可执行文件使用gdb，其他节点不受影响：
```bash
ros2 launch demo_nodes_cpp talker_listener.launch.py \
  --launch-prefix 'gdb -ex run --args' \
  --launch-prefix-filter '^talker$'
```

### 8. 直接运行本地launch文件（不通过包）
> 不需要package，直接给文件路径
```bash
ros2 launch ./my_launch.launch.py
```

---

## 重要注意点
1. **launch文件后缀**：ROS2主流是 `.launch.py`（python），也支持xml/yaml；ROS1的`.launch`不再支持。
2. 传参格式固定 `key:=value`，**冒号等于号**，不要写成 `key=value`。
3. `-a` 参数很有用：很多launch文件内部设置了`output="screen"/"log"`，默认不打印；加`-a`强制全部输出。
4. `--launch‑prefix` 是非常强大的调试手段，可以套gdb、valgrind、perf、xterm等工具。
5. 按 `Ctrl+C` 停止launch，会自动关闭所有启动的节点进程。

## 常见坑
1. 找不到launch文件：确认包已经`colcon build`，并且`source install/setup.bash`。
2. 参数不生效：参数格式写错，必须 `:=`。
3. 节点打印看不到：加 `-a` 参数。
4. launch内部报错，看不到细节：加 `-d` debug模式。

---

### 补充：launch文件里常见对象
- `Node`：启动普通节点
- `ComposableNodeContainer`：组件容器
- `ComposableNode`：加载组件到容器
- `Parameter`：设置参数
- `LaunchConfiguration`：接收外部传入参数

如果你需要，我可以把前面全部 ros2 cli 汇总一份完整速查清单。


========================================
========================================
# 6.ros2 lifecycle 命令详解
```sh
$ ros2 lifecycle -h
usage: ros2 lifecycle [-h] Call `ros2 lifecycle <command> -h` for more detailed usage. ...

Various lifecycle related sub-commands

options:
  -h, --help            show this help message and exit

Commands:
  get    Get lifecycle state for one or more nodes
  list   Output a list of available transitions
  nodes  Output a list of nodes with lifecycle
  set    Trigger lifecycle state transition

  Call `ros2 lifecycle <command> -h` for more detailed usage.
```


> 生命周期节点（Lifecycle Node）：继承`rclcpp_lifecycle::LifecycleNode`，拥有状态机，不会启动就直接工作；需要手动执行状态切换，适合机器人设备管理、硬件驱动节点。
```bash
ros2 lifecycle -h
```

## 子命令列表
|命令|作用|
|---|---|
|`get`|获取一个/多个生命周期节点当前状态|
|`list`|列出全部合法的状态转换（transition）|
|`nodes`|列出系统中所有**生命周期节点**（普通节点不会出现）|
|`set`|触发状态切换，执行状态转换指令|

---

## 实操示例
### 1. 列出所有生命周期节点
```bash
# abner: 列出所有 已经run 的生命周期节点
ros2 lifecycle nodes
```
输出示例：
```
/lc_talker
```
> 普通`rclcpp::Node`不会出现在这里，只有LifecycleNode才会被识别。

### 2. 获取节点当前生命周期状态
```bash
ros2 lifecycle get /lc_talker
```
输出会显示当前状态名称：`unconfigured` / `inactive` / `active` 等。

### 3. 查看全部允许的状态转换
```bash
ros2 lifecycle list
```
打印完整状态机：
- unconfigured → configuring
- configuring → inactive
- inactive → activating
- activating → active
- active → deactivating
- deactivating → inactive
- inactive → cleaningup
- cleaningup → unconfigured

### 4. 手动切换生命周期状态（set）
格式：`ros2 lifecycle set <节点名> <转换指令>`
```bash
# 配置：unconfigured → configuring → inactive
ros2 lifecycle set /lc_talker configure

# 激活：inactive → activating → active（节点开始发布话题）
ros2 lifecycle set /lc_talker activate

# 去激活：active → deactivating → inactive（停止发布）
ros2 lifecycle set /lc_talker deactivate

# 清理：inactive → cleaningup → unconfigured
ros2 lifecycle set /lc_talker cleanup
```

> 注意：`set`后面写的是**转换动作名**，不是目标状态。
> - `configure` 触发配置流程
> - `activate` 触发激活流程

---

## 生命周期状态简要说明
1. **unconfigured**：刚启动，还没有做任何初始化
2. **configuring**：正在配置（加载参数、打开硬件）
3. **inactive**：配置完成，但**不工作、不发布消息**
4. **activating**：正在激活
5. **active**：正常运行，发布话题、执行业务逻辑
6. **deactivating**：正在去激活
7. **cleaningup**：清理资源，释放硬件

> 只有进入`active`状态，Lifecycle节点才会开始发布消息。

## 配套话题
- `/lc_talker/state`：`lifecycle_msgs/msg/State`，持续发布当前状态
- `/lc_talker/transition_event`：`lifecycle_msgs/msg/TransitionEvent`，发布每次状态切换事件

```bash
# 监听状态变化事件
ros2 topic echo /lc_talker/transition_event
```

## 常见坑
1. 节点启动后，**默认是unconfigured**，不会发布任何话题；必须`configure`+`activate`。
2. 执行`set`命令只是发送请求，**不是立刻完成**，需要等待节点内部回调执行完毕。
3. 普通节点没有生命周期，`ros2 lifecycle nodes`看不到它们。
4. launch文件中可以使用`LifecycleNode`动作，自动完成configure/activate，不用手动敲命令。

---

## 快速工作流
```bash
# 1. 看到有哪些生命周期节点
ros2 lifecycle nodes

# 2. 查看当前状态
ros2 lifecycle get /lc_talker

# 3. 配置 + 激活
ros2 lifecycle set /lc_talker configure
ros2 lifecycle set /lc_talker activate

# 4. 查看状态事件
ros2 topic echo /lc_talker/transition_event
```

> 如果你需要，我可以把前面所有 ros2 cli（topic/node/service/daemon/component/bag/lifecycle/doctor/launch）汇总一份完整速查清单。

===================================

# 7.ros2 multicast 命令详解
```sh
$ ros2 multicast -h
usage: ros2 multicast [-h] Call `ros2 multicast <command> -h` for more detailed usage. ...

Various multicast related sub-commands

options:
  -h, --help            show this help message and exit

Commands:
  receive  Receive a single UDP multicast packet
  send     Send a single UDP multicast packet

  Call `ros2 multicast <command> -h` for more detailed usage.
```


> **ros2 multicast**：ROS2 自带UDP组播测试工具，用来验证**网络层UDP组播是否通**。
> ROS2 DDS发现机制依赖UDP组播；多机看不到节点，优先用这个命令排查底层网络，区分：是网络组播问题，还是DDS/ROS配置问题。

```bash
ros2 multicast -h
```

## 子命令
|命令|作用|
|---|---|
|`receive`|监听UDP组播数据包，等待接收消息|
|`send`|发送一条UDP组播测试包|

> 会使用 `ROS_DOMAIN_ID` 对应的组播地址；**两台机器必须相同ROS_DOMAIN_ID**才能互相收到。

---

## 实操测试（多机排错标准流程）
### 场景1：同一台机器两个终端测试
终端A（接收）
```bash
ros2 multicast receive
```
终端B（发送）
```bash
ros2 multicast send
```
✅成功输出：
```
Received from 127.0.0.1:xxxx: 'Hello World!'
```
❌收不到：本机网卡组播异常、防火墙拦截。

### 场景2：两台不同机器测试
机器A（接收）
```bash
export ROS_DOMAIN_ID=0
ros2 multicast receive
```
机器B（发送）
```bash
export ROS_DOMAIN_ID=0
ros2 multicast send
```

- ✅机器A收到消息：**底层UDP组播网络正常**，ROS2发现问题属于DDS/配置层面（domain_id、rmw、防火墙、daemon）。
- ❌收不到：**网络层组播不通**，优先排查：防火墙、网卡是否开启MULTICAST、交换机/路由器是否禁用组播、是否跨子网、WSL/NAT环境等。

> 查看网卡是否开启组播标志：
```bash
ip addr
# 看网卡flags里是否有 MULTICAST
```

### 防火墙放行组播（Ubuntu ufw）
```bash
sudo ufw allow in proto udp to 224.0.0.0/4
sudo ufw allow out proto udp from 224.0.0.0/4
```

---

## 排错逻辑（非常关键）
1. `ping` 可以通，但是 `ros2 multicast receive` 收不到包
→ **UDP组播被网络/防火墙拦截**，DDS发现必然失效，即使ROS配置全部正确也看不到节点。

2. `ros2 multicast` 收发正常，但ros2仍然看不到别的机器节点
→ 网络层没问题，问题在ROS侧：
   - `ROS_DOMAIN_ID` 不一致
   - 两边RMW中间件不一样（CycloneDDS ↔ Fast‑DDS）
   - daemon缓存异常（重启daemon）
   - 多网卡DDS绑定到错误网卡
   - 可以使用 `ros2 doctor hello` 进一步验证DDS发现层。

---

## 完整多机故障排查链路
```
1. ping 机器IP → 基础IP连通
2. ros2 multicast send / receive → UDP组播网络层是否通
3. ros2 doctor hello → DDS发现层连通性
4. ros2 doctor --report-failed → 检查domain_id、rmw、QoS
5. ros2 daemon stop && ros2 daemon start → 清理cli缓存
```

> 注意：`ros2 multicast` 只测**UDP组播**，不代表DDS业务消息一定通；只是定位网络底层是否正常。

---

## 补充说明
- 这个工具属于 `ros2multicast` 包；部分精简安装会缺少，需要安装：
```bash
sudo apt install ros-humble-ros2multicast
```
- 部分网络环境（WSL2、VPN、云服务器）默认不支持组播，这时可以改用 **Discovery Server（发现服务器）** 模式，不需要组播。

如果你需要，我可以把前面全部 ros2 cli（topic/node/service/daemon/component/bag/lifecycle/doctor/launch/multicast）整理一份完整速查清单。


=================================
=================================
# 8.ros2 node 命令详解
```sh
$ ros2 node -h
usage: ros2 node [-h] Call `ros2 node <command> -h` for more detailed usage. ...

Various node related sub-commands

options:
  -h, --help            show this help message and exit

Commands:
  info  Output information about a node
  list  Output a list of available nodes

  Call `ros2 node <command> -h` for more detailed usage.
```


> `ros2 node`：查看ROS2节点信息，节点是ROS2的基本执行单元。
```bash
ros2 node -h
```

## 子命令
|命令|作用|
|---|---|
|`list`|列出当前系统所有运行的节点名称|
|`info`|查看单个节点详细信息：订阅、发布、服务、动作、参数|

---

## 实操示例
### 1. 列出全部节点
```bash
ros2 node list
```
输出示例：
```
/talker
/listener
/lc_talker
```
> 注意：**显示的是节点全名（含命名空间）**。
> 加上 `-t` 参数，可以同时显示节点对应的进程PID：
```bash
ros2 node list -t
```

### 2. 查看节点详细信息 `ros2 node info`
```bash
ros2 node info /talker
```
输出包含：
- 该节点 **发布的话题（Publishers）**
- 该节点 **订阅的话题（Subscribers）**
- 提供的服务（Services）
- 客户端服务（Service Clients）
- 动作服务端（Action Servers）
- 动作客户端（Action Clients）

示例输出片段：
```
/talker
  Subscribers:
  Publishers:
    /chatter: std_msgs/msg/String
  Service Servers:
  Service Clients:
  Action Servers:
  Action Clients:
```

> 这个命令非常适合排查：节点有没有发布话题、有没有订阅，用来定位“消息收不到”问题。

### 3. 过滤节点（使用grep）
```bash
# 只看带 lc 的节点
ros2 node list | grep lc
```

---

## 常见坑
1. 节点看不到：
   - 检查 `ROS_DOMAIN_ID` 是否一致
   - 重启 daemon：`ros2 daemon stop && ros2 daemon start`
   - 多机通信：`ros2 multicast` 验证组播网络

2. `ros2 node list` 看不到，但节点实际在跑：
> daemon缓存异常，重启daemon即可；**节点之间DDS通信不受daemon影响，daemon只是给CLI工具做缓存**。

3. 生命周期节点：`ros2 node list` 可以看到，但是它进入`unconfigured/inactive`状态时，**不会发布话题**，需要`ros2 lifecycle set`激活。

---

## 配套常用组合命令
```bash
# 查看所有节点
ros2 node list

# 看某个节点的发布订阅关系
ros2 node info /talker

# 查看节点PID
ros2 node list -t

# 结合topic，看话题对应哪些节点
ros2 topic list
ros2 topic info /chatter
```

> 补充：
> - `ros2 node list` 只能看**已经被发现到的节点**；
> - 节点之间的通信是DDS直接完成，不经过ros2 cli工具。

---

### 补充对比 ROS1
ROS1：`rosnode list` / `rosnode info`
ROS2：`ros2 node list` / `ros2 node info`

> 如果你需要，我把前面全部 ros2 cli 汇总一份完整速查表。

==================================
==================================

# 9.ros2 param 命令详解
```sh
$ ros2 param -h
usage: ros2 param [-h] Call `ros2 param <command> -h` for more detailed usage. ...

Various param related sub-commands

options:
  -h, --help            show this help message and exit

Commands:
  delete    Delete parameter
  describe  Show descriptive information about declared parameters
  dump      Show all of the parameters of a node in a YAML file format
  get       Get parameter
  list      Output a list of available parameters
  load      Load parameter file for a node
  set       Set parameter
``` 


> `ros2 param`：ROS2 参数操作工具，用来查询、修改、删除、加载节点参数；参数存储在节点内部，每个节点拥有独立参数集合。
```bash
ros2 param -h
```

## 子命令列表
|命令|作用|
|---|---|
|`delete`|删除节点的一个参数|
|`describe`|查看参数声明信息（类型、描述、是否只读）|
|`dump`|把节点全部参数导出为YAML格式输出|
|`get`|读取单个参数的值|
|`list`|列出节点所有参数名称|
|`load`|从yaml文件加载参数到指定节点|
|`set`|设置节点参数的值|

---

## 实操示例
### 1. 列出节点全部参数
```bash
ros2 param list /talker
```
输出该节点所有参数名。

### 2. 获取单个参数的值
```bash
ros2 param get /talker my_param
```

### 3. 设置参数（运行时修改）
```bash
# 整型
ros2 param set /talker count 10
# 字符串
ros2 param set /talker msg "hello"
# bool
ros2 param set /talker enable true
# 数组
ros2 param set /talker arr [1,2,3]
```
> ⚠️ 只有节点**声明允许动态修改**的参数，set才会生效；只读参数无法修改。

### 4. 删除参数
```bash
ros2 param delete /talker my_param
```

### 5. 查看参数声明描述
```bash
ros2 param describe /talker my_param
```
会显示：参数类型、描述文本、是否只读。

### 6. 导出节点全部参数为YAML
```bash
ros2 param dump /talker
```
直接打印到终端；保存到文件：
```bash
ros2 param dump /talker > talker_params.yaml
```

### 7. 从yaml文件加载参数到节点
```bash
ros2 param load /talker talker_params.yaml
```

---

## 关键知识点
1. **参数属于节点**：参数不是全局的，必须指定节点名；不同节点可以有同名参数，互不干扰。
2. 参数类型：`bool / int / double / string / 数组`。
3. **声明参数**：代码里必须用 `declare_parameter()` 声明，否则 `ros2 param list` 看不到，也不能get/set。
4. 动态修改：
   - 声明时设置 `read_only=false` 才允许运行时 `ros2 param set` 修改。
   - 如果参数是`read_only=true`，只能启动时传入，运行时不能修改。

### 启动节点时传入参数
```bash
ros2 run demo_nodes_cpp talker --ros-args -p my_param:=123
```
launch文件中也可以通过`Parameter`对象给节点传参。

---

## 常见坑
1. `ros2 param get` 报 `Parameter not found`
   - 节点没有声明该参数；
   - 节点名写错；
   - 生命周期节点：`unconfigured`状态下部分参数还未声明。

2. `ros2 param set` 提示无法设置
   - 参数被声明为`read_only`，只能启动时赋值。

3. 加载yaml文件：yaml文件格式必须符合ROS2参数规范。

---

## 常用组合工作流
```bash
# 1. 查看节点有哪些参数
ros2 param list /talker

# 2. 读取参数
ros2 param get /talker my_param

# 3. 修改参数
ros2 param set /talker my_param 200

# 4. 导出参数备份
ros2 param dump /talker > params.yaml

# 5. 从文件回写参数
ros2 param load /talker params.yaml
```

> 补充：ROS1对应命令：`rosparam`。

---

### 完整ROS2 CLI汇总（已学全部）
- `ros2 topic`：话题
- `ros2 node`：节点
- `ros2 service`：服务
- `ros2 action`：动作
- `ros2 param`：参数
- `ros2 bag`：录制回放数据
- `ros2 component`：组件
- `ros2 lifecycle`：生命周期节点
- `ros2 daemon`：守护进程
- `ros2 doctor`：系统诊断
- `ros2 launch`：启动launch文件
- `ros2 multicast`：UDP组播网络测试

如果你需要，我可以把全部命令整理一份一页速查文档。

=====================================
# 10.ros2 pkg 命令详解
```sh
$ ros2 pkg -h
usage: ros2 pkg [-h] Call `ros2 pkg <command> -h` for more detailed usage. ...

Various package related sub-commands

options:
  -h, --help            show this help message and exit

Commands:
  create       Create a new ROS 2 package
  executables  Output a list of package specific executables
  list         Output a list of available packages
  prefix       Output the prefix path of a package
  xml          Output the XML of the package manifest or a specific tag
```


> `ros2 pkg`：ROS2 包管理工具，用于查看、创建包、查看可执行文件、查看包路径、查看`package.xml`。
```bash
ros2 pkg -h
```

## 子命令列表
|命令|作用|
|---|---|
|`create`|新建一个ROS2软件包|
|`executables`|列出该包内所有可执行程序（节点）|
|`list`|列出系统中所有可用的ROS2包|
|`prefix`|输出包的安装路径（install目录下）|
|`xml`|打印包的`package.xml`完整内容，也可以只打印指定标签|

---

## 实操示例
### 1. 列出所有ROS2包
```bash
ros2 pkg list
```
> 过滤查找某个包：
```bash
ros2 pkg list | grep demo
```

### 2. 查看包的安装前缀路径
```bash
ros2 pkg prefix demo_nodes_cpp
```
输出示例：
```
/opt/ros/humble
```
> 自己工作空间的包会输出 `~/ws/install` 下路径。

### 3. 查看包里面所有可执行文件（节点程序）
```bash
ros2 pkg executables demo_nodes_cpp
```
输出示例：
```
demo_nodes_cpp talker
demo_nodes_cpp listener
```
> 格式：`包名 可执行文件名`，对应 `ros2 run 包名 可执行名`。

### 4. 查看 package.xml 完整内容
```bash
ros2 pkg xml demo_nodes_cpp
```
只看某个标签，例如`buildtool_depend`：
```bash
ros2 pkg xml demo_nodes_cpp -t buildtool_depend
```

### 5. 创建新包 `ros2 pkg create`
语法：
```bash
ros2 pkg create <包名> --build-type <编译类型> --deps <依赖包1> <依赖包2>
```

示例：创建C++包，依赖rclcpp、std_msgs
```bash
ros2 pkg create my_pkg --build-type ament_cmake --deps rclcpp std_msgs
```

- `--build-type`：
  - `ament_cmake`：C++包
  - `ament_python`：Python包
- `--deps`：声明运行依赖。

> ⚠️ 新建包需要放在 `src` 目录下，之后执行 `colcon build`。

---

## 常用工作流
```bash
# 1. 查找包
ros2 pkg list | grep xxx

# 2. 看包安装位置
ros2 pkg prefix my_pkg

# 3. 看包有哪些节点
ros2 pkg executables my_pkg

# 4. 查看package.xml
ros2 pkg xml my_pkg

# 5. 创建C++包
ros2 pkg create my_pkg --build-type ament_cmake --deps rclcpp std_msgs
```

## 常见坑
1. `ros2 pkg list`看不到自己写的包
> 没有执行`colcon build`，或者没有source `install/setup.bash`。

2. `ros2 pkg executables`看不到节点
> `CMakeLists.txt`没有安装可执行文件；python包没有设置entry_point。

3. `ros2 pkg create`创建的包直接在工作空间根目录，**建议进入src目录再执行**。

 
===========================================

# 11.ros2 plugin 命令详解

```sh
$ ros2 plugin -h
usage: ros2 plugin [-h] Call `ros2 plugin <command> -h` for more detailed usage. ...

Various plugin related sub-commands

options:
  -h, --help            show this help message and exit

Commands:
  list  Output a list of plugins

```


> **ros2 plugin**：ROS2插件工具，用于查询**插件（plugin）**，基于`pluginlib`机制。
> 插件是动态加载的共享库，典型场景：`component组件`、导航插件、传感器驱动、自定义算法插件。
> 插件通过`pluginlib`注册，在`plugin.xml`中声明，ament索引扫描识别。

```bash
ros2 plugin -h
```

## 子命令
|命令|作用|
|---|---|
|`list`|列出系统中所有已注册的插件，输出：插件库、插件类名、基类类型|

### 语法
```bash
ros2 plugin list
```

### 可选过滤
```bash
# 只看某个基类的插件
ros2 plugin list --base-class rclcpp_components::NodeFactory
```

输出示例：
```
Package: composition
  Plugin: composition::Talker
  Base Class: rclcpp_components::NodeFactory
```
> 这个就是我们之前`ros2 component load`加载的组件插件。

---

## 概念回顾
1. **pluginlib**：ROS2插件框架，C++实现；
2. 每个插件包里面要有`plugin.xml`，声明插件类、基类、库文件；
3. `ament_index` 会扫描所有`plugin.xml`，`ros2 plugin list`就是读取这个索引；
4. **Component组件本质就是一种pluginlib插件**，基类为`rclcpp_components::NodeFactory`。

### 查看指定包的插件
```bash
ros2 plugin list | grep composition
```

## 常见使用场景
1. 调试组件：确认组件插件是否注册成功
```bash
ros2 plugin list --base-class rclcpp_components::NodeFactory
```
> 如果这里看不到你的组件，说明：
> - `plugin.xml`没有写
> - 没有`ament_register_plugins` 注册
> - 没有`colcon build`，没有source install

2. 导航、SLAM插件排查：查看可用的规划器、代价地图插件。

## 完整工作流（开发组件插件）
1. 写C++代码，实现组件，宏 `RCLCPP_COMPONENTS_REGISTER_NODE`
2. 编写`plugin.xml`，注册插件
3. CMakeLists.txt 使用 `ament_register_plugins()`
4. `colcon build` && `source install/setup.bash`
5. `ros2 plugin list` 验证插件是否被识别
6. `ros2 component load` 加载插件

---

## 常见坑
1. `ros2 plugin list`看不到插件
- 没有执行`ament_register_plugins`；
- `plugin.xml`路径没有安装；
- 没有source install；
- 基类名称写错。

2. 插件只能C++，Python不支持pluginlib插件机制。
 
==============================================

# 12.ros2 run 命令详解
```sh
$ ros2 run -h
usage: ros2 run [-h] [--prefix PREFIX] package_name executable_name ...

Run a package specific executable

positional arguments:
  package_name     Name of the ROS package
  executable_name  Name of the executable
  argv             Pass arbitrary arguments to the executable

options:
  -h, --help       show this help message and exit
  --prefix PREFIX  Prefix command, which should go before the executable. Command must
                   be wrapped in quotes if it contains spaces (e.g. --prefix 'gdb -ex
                   run --args').
```

> `ros2 run`：直接运行ROS包里面的可执行程序（节点），是最基础启动节点的命令。
```bash
ros2 run -h
```

## 参数说明
| 参数 | 说明 |
|---|---|
| `package_name` | ROS包名 |
| `executable_name` | 可执行文件（节点名） |
| `argv` | 传给程序本身的命令行参数 |
| `--prefix PREFIX` | 给程序加前置命令，用于gdb、valgrind、xterm调试；带空格要用引号包裹 |

## 基础用法
```bash
# 格式：ros2 run 包名 可执行程序
ros2 run demo_nodes_cpp talker
```

> 查看包里面有哪些可执行程序：
```bash
ros2 pkg executables demo_nodes_cpp
```

## 给节点传入ROS参数（`--ros‑args`）
> 注意：`--ros‑args` 是ROS专用参数，**不是程序原生argv**，用来传参数、重映射话题。
```bash
# 启动节点并设置参数
ros2 run demo_nodes_cpp talker --ros-args -p count:=100

# 话题重映射，把 /chatter 改成 /my_chatter
ros2 run demo_nodes_cpp talker --ros-args -r chatter:=/my_chatter
```

### `--ros‑args`常用标记
- `-p key:=value`：设置参数
- `-r old_topic:=new_topic`：话题重映射
- `-n node_name`：修改节点名称
- `--namespace /ns`：设置节点命名空间

## --prefix 调试用法
### 1. gdb调试节点
```bash
ros2 run demo_nodes_cpp talker --prefix 'gdb -ex run --args'
```

### 2. 在新终端xterm中运行节点
```bash
ros2 run demo_nodes_cpp talker --prefix 'xterm -e'
```

### 3. valgrind内存检测
```bash
ros2 run demo_nodes_cpp talker --prefix 'valgrind'
```

## 传入普通程序参数（非ros参数）
> 直接写在可执行文件名后面，不会被ROS解析，直接传给二进制程序
```bash
ros2 run my_pkg my_node --my-arg 123
```

## 对比：ros2 run vs ros2 component standalone
- `ros2 run pkg exe`：**普通独立进程节点**，每个节点一个进程
- `ros2 component standalone pkg::Class`：底层是组件，自动新建一个容器进程运行组件

## 常见坑
1. `ros2 run` 找不到可执行文件
    - 没有`colcon build`
    - 没有`source install/setup.bash`
    - CMakeLists.txt没有`install(TARGETS ...)`安装可执行文件
2. `--ros‑args`必须写在**可执行文件名之后**，不能写在前面
3. `--prefix`内部带空格，必须用单引号包裹。

## 示例组合
```bash
# 启动节点，设置参数，重映射话题
ros2 run demo_nodes_cpp talker --ros-args -p count:=50 -r chatter:=/test_chatter

# gdb调试节点
ros2 run demo_nodes_cpp talker --prefix 'gdb -ex run --args'
```
 
==============================================

# 13.ros2 security 命令详解
```sh
$ ros2 security -h
usage: ros2 security [-h] Call `ros2 security <command> -h` for more detailed usage. ...

Various security related sub-commands

options:
  -h, --help            show this help message and exit

Commands:
  create_enclave      Create enclave
  create_key          DEPRECATED: Create enclave. Use create_enclave instead
  create_keystore     Create keystore
  create_permission   Create permission
  generate_artifacts  Generate keys and permission files from a list of identities and policy files
  generate_policy     Generate XML policy file from ROS graph data
  list_enclaves       List enclaves in keystore
  list_keys           DEPRECATED: List enclaves in keystore. Use list_enclaves instead
```

> `ros2 security` 是 **SROS2（ROS2安全）** 的命令行工具，基于 **DDS‑Security**，实现节点通信加密、身份认证、访问权限控制（谁可以发布/订阅哪些话题）。
> 依赖包：`sros2`，默认不一定安装，需要手动安装：
```bash
sudo apt install ros-humble-sros2
```

```bash
ros2 security -h
```

## 子命令列表
|命令|作用|
|---|---|
|`create_enclave`|创建安全飞地（enclave），生成节点证书私钥；**替代废弃的create_key**|
|`create_key`|**已废弃**，改用`create_enclave`|
|`create_keystore`|创建密钥仓库（keystore），根CA证书存放目录，整个安全系统的根目录|
|`create_permission`|根据XML策略文件生成权限签名文件（permissions.p7s），控制节点发布订阅权限|
|`generate_artifacts`|批量生成身份证书+权限文件，输入身份列表+策略文件|
|`generate_policy`|从当前运行的ROS图，自动导出安全策略XML文件|
|`list_enclaves`|列出keystore内所有已创建的enclave；**替代废弃的list_keys**|

> 废弃提示：`create_key` / `list_keys` 只是兼容旧版本，不要使用。

## 核心概念
1. **keystore（密钥仓库）**：存放CA根证书、所有enclave的证书、权限文件，是整个安全系统的根目录。
2. **enclave（安全飞地）**：一个安全隔离单元，**一个enclave对应一组节点**；每个enclave拥有自己证书、私钥、权限策略；节点启动时指定`--enclave`使用该安全配置。
3. **policy.xml**：权限策略文件，定义：哪些节点允许发布/订阅哪些话题、服务。
4. **DDS‑Security**：底层，实现加密、认证、访问控制。

## 完整实操流程（最简示例）
### 1. 创建keystore密钥仓库
```bash
export ROS_SECURITY_KEYSTORE=~/my_keystore
ros2 security create_keystore ${ROS_SECURITY_KEYSTORE}
```
> 生成CA根证书（private/ca.key.pem，务必妥善保管）、public证书。

### 2. 为节点创建enclave（飞地）
> enclave路径使用**完整节点名称**
```bash
ros2 security create_enclave ${ROS_SECURITY_KEYSTORE} /talker
ros2 security create_enclave ${ROS_SECURITY_KEYSTORE} /listener
```

### 3. 查看keystore里全部enclave
```bash
ros2 security list_enclaves ${ROS_SECURITY_KEYSTORE}
```

### 4. 根据策略xml生成权限文件
```bash
ros2 security create_permission ${ROS_SECURITY_KEYSTORE} /talker ./policy.xml
ros2 security create_permission ${ROS_SECURITY_KEYSTORE} /listener ./policy.xml
```

### 5. 从当前运行ROS图自动生成策略xml
```bash
ros2 security generate_policy -o my_policy.xml
```

### 6. 启动节点启用安全
需要设置两个环境变量，并且指定enclave：
```bash
export ROS_SECURITY_ENABLE=true
export ROS_SECURITY_KEYSTORE=~/my_keystore

# 启动talker，指定enclave为/talker
ros2 run demo_nodes_cpp talker --ros-args --enclave="/talker"
```

> 注意：**必须两端都启用安全**，否则节点之间无法通信。

## 环境变量
- `ROS_SECURITY_ENABLE`：`true/false`，开关安全功能
- `ROS_SECURITY_KEYSTORE`：keystore目录路径

## 常见坑
1. 命令找不到：没装`sros2`包。
2. 启用安全后节点互相看不见：
   - 两端都必须开启`ROS_SECURITY_ENABLE=true`
   - enclave权限策略xml配置错误，没有允许话题通信
   - 中间件（rmw）必须支持DDS‑Security：Fast‑DDS、ConnextDDS；**CycloneDDS不支持DDS‑Security**。
3. `create_enclave` 传入的是**完整节点名**，不是包名。
4. 私钥`ca.key.pem`不能泄露，这是整个安全体系的根。




==============================================

# 14.ros2 service 命令详解

```sh
$ ros2 service -h
usage: ros2 service [-h] [--include-hidden-services] Call `ros2 service <command> -h` for more detailed usage. ...

Various service related sub-commands

options:
  -h, --help            show this help message and exit
  --include-hidden-services
                        Consider hidden services as well

Commands:
  call  Call a service
  echo  Echo a service
  find  Output a list of available services of a given type
  info  Print information about a service
  list  Output a list of available services
  type  Output a service's type 
```


> **Service（服务）**：ROS2同步请求‑应答模型，客户端发请求，服务端立刻返回应答。
```bash
ros2 service -h
```

## 选项参数
|参数|说明|
|---|---|
|`--include-hidden-services`|包含隐藏服务（以下划线开头的内部服务）|

## 子命令列表
|命令|作用|
|---|---|
|`call`|调用服务，发送请求，接收返回应答|
|`echo`|监听服务的请求与应答（观察服务调用）|
|`find`|按服务类型查找匹配的服务名称|
|`info`|查看服务详情：类型、服务端节点|
|`list`|列出系统中所有可用服务|
|`type`|查询某个服务对应的服务接口类型|

---

## 实操示例
### 1. 列出全部服务
```bash
ros2 service list
# 包含隐藏服务
ros2 service list --include-hidden-services
```

### 2. 查询服务的接口类型
```bash
ros2 service type /add_two_ints
```
输出示例：
```
example_interfaces/srv/AddTwoInts
```

### 3. 查看服务详细信息
```bash
ros2 service info /add_two_ints
```
输出：服务类型、提供该服务的节点名称。

### 4. 根据服务类型查找服务
```bash
ros2 service find example_interfaces/srv/AddTwoInts
```

### 5. 调用服务 `call`
格式：`ros2 service call <服务名> <服务类型> <请求参数>`
```bash
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 10, b: 20}"
```
返回应答：
```
response:
  sum: 30
```

> 注意参数是**YAML格式字符串**，大括号要包裹。

### 6. 监听服务调用（echo）
> 可以看到每一次的请求和应答，调试用
```bash
ros2 service echo /add_two_ints
```

---

## 配套接口查看
查看服务接口定义（请求、应答字段）
```bash
ros2 interface show example_interfaces/srv/AddTwoInts
```
输出：
```
int64 a
int64 b
---
int64 sum
```
> `---` 分隔：上面是**request请求**，下面是**response应答**。

---

## 常见坑
1. `ros2 service call` 报错找不到服务
    - 服务端节点没有运行；
    - `ROS_DOMAIN_ID`不一致；
    - daemon缓存异常，重启 `ros2 daemon stop && ros2 daemon start`。
2. 参数格式错误：必须YAML，`{key: value}`，引号注意。
3. 服务是**同步阻塞**，客户端等待服务端返回结果；如果需要异步，使用Action。

## 对比：Service vs Action
- **Service**：简单同步请求应答，适合快速小任务，不能反馈进度。
- **Action**：异步，支持目标、反馈进度、结果、取消。

---

## 快速工作流
```bash
# 1. 列出服务
ros2 service list

# 2. 看服务类型
ros2 service type /add_two_ints

# 3. 看服务接口定义
ros2 interface show example_interfaces/srv/AddTwoInts

# 4. 调用服务
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a:5, b:3}"

# 5. 监听服务调用
ros2 service echo /add_two_ints
```
 

===============================================

# 15.ros2 topic 命令详解
```sh
$ ros2 topic -h
usage: ros2 topic [-h] [--include-hidden-topics] Call `ros2 topic <command> -h` for more detailed usage. ...

Various topic related sub-commands

options:
  -h, --help            show this help message and exit
  --include-hidden-topics
                        Consider hidden topics as well

Commands:
  bw     Display bandwidth used by topic
  delay  Display delay of topic from timestamp in header
  echo   Output messages from a topic
  find   Output a list of available topics of a given type
  hz     Print the average receiving rate to screen
  info   Print information about a topic
  list   Output a list of available topics
  pub    Publish a message to a topic
  type   Print a topic's type
```


> **Topic（话题）**：ROS2 发布‑订阅模型，异步单向通信，节点发布消息，其他节点订阅接收。
```bash
ros2 topic -h
```

## 全局选项
| 参数 | 说明 |
|---|---|
| `--include-hidden-topics` | 包含以下划线开头的内部隐藏话题 |

## 子命令列表
|命令|作用|
|---|---|
|`bw`|统计话题带宽占用|
|`delay`|根据消息header时间戳计算消息延迟|
|`echo`|订阅话题，打印收到的消息|
|`find`|按消息类型查找对应话题|
|`hz`|统计话题消息发布频率（Hz）|
|`info`|查看话题详情：发布者、订阅者、QoS配置|
|`list`|列出全部话题|
|`pub`|命令行发布消息到话题|
|`type`|查询话题的消息类型|

---

## 实操示例
### 1. 列出话题
```bash
ros2 topic list
# 显示隐藏话题
ros2 topic list --include-hidden-topics
```

### 2. 查询话题消息类型
```bash
ros2 topic type /chatter
```
输出：`std_msgs/msg/String`

### 3. 查看话题详细信息（发布者、订阅者、QoS）
```bash
ros2 topic info /chatter
```

### 4. 按消息类型查找话题
```bash
ros2 topic find std_msgs/msg/String
```

### 5. 订阅话题，打印消息（echo）
```bash
ros2 topic echo /chatter
```

### 6. 命令行发布消息（pub）
格式：`ros2 topic pub <话题> <消息类型> <消息内容>`
```bash
ros2 topic pub /chatter std_msgs/msg/String "{data: 'hello ros2'}"
```
> 加上 `--once` 只发布一次就退出
```bash
ros2 topic pub --once /chatter std_msgs/msg/String "{data: 'one time'}"
```

### 7. 统计发布频率 hz
```bash
ros2 topic hz /chatter
```

### 8. 统计话题带宽 bw
```bash
ros2 topic bw /chatter
```

### 9. 计算消息延迟 delay
> 需要消息自带 `header` 时间戳
```bash
ros2 topic delay /odom
```

---

## 配套查看消息接口
```bash
ros2 interface show std_msgs/msg/String
```

## 常见坑
1. `ros2 topic echo` 收不到消息
    - 没有发布者；
    - QoS不匹配；
    - `ROS_DOMAIN_ID`不一致；
    - daemon缓存异常，重启 `ros2 daemon stop && ros2 daemon start`。
2. pub参数是 **YAML格式**，`{key: value}`。
3. `hz/bw/delay` 需要持续接收消息，没有消息就没有输出。

## 快速工作流
```bash
# 列出话题
ros2 topic list
# 看话题类型
ros2 topic type /chatter
# 看发布订阅和QoS
ros2 topic info /chatter
# 监听消息
ros2 topic echo /chatter
# 发布消息
ros2 topic pub /chatter std_msgs/msg/String "{data:'test'}"
# 看发布频率
ros2 topic hz /chatter
```

---

### ROS2 CLI 完整清单
1. `ros2 topic` 话题
2. `ros2 node` 节点
3. `ros2 service` 服务
4. `ros2 action` 动作
5. `ros2 param` 参数
6. `ros2 bag` 录制回放
7. `ros2 component` 组件
8. `ros2 lifecycle` 生命周期
9. `ros2 daemon` 守护进程
10. `ros2 doctor` 系统诊断
11. `ros2 launch` 启动launch
12. `ros2 multicast` UDP组播测试
13. `ros2 pkg` 包管理
14. `ros2 plugin` 插件查询
15. `ros2 run` 运行节点
16. `ros2 security` 安全SROS2

接下来可以看 `ros2 action`。