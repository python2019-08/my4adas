# ch1.Ubuntu24.04（Noble）宿主机，Docker内部跑 ROS2 Humble 
> 本质：宿主机只是运行docker引擎；**ROS环境全部隔离在容器内，不受主机系统版本影响**，彻底规避Jazzy/Humble版本、头文件差异、conda/python冲突问题。

### 优点（刚好解决你现在遇到的问题）
1. 固定Humble完整环境，和教材/教程一致，**不再有Jazzy头文件改名（planner_exceptions.hpp这类坑）**
2. 容器内python纯净，不会和宿主机miniconda串环境，再也不会报`catkin_pkg`缺失
3. 工作空间代码放在宿主机目录，VSCode在宿主机编辑代码；**编译、colcon、ros2 run全部在容器内部执行**
4. 镜像可备份、可迁移，换电脑直接复用整套环境

### 缺点
1. RViz2 / Gazebo图形界面需要额外配置X11转发
2. 真机调试：串口、USB雷达、相机，需要挂载`/dev`，部分权限需要`--privileged`
3. DDS通信：推荐`--network=host`，保证ros2节点发现正常（仿真/本机开发首选）

## sec.1 安装docker（Ubuntu24.04）
```bash
sudo apt update
sudo apt install docker.io
# 将当前用户加入docker组，免sudo
sudo usermod -aG docker $USER
newgrp docker
# 验证
docker run hello-world
```
 
## sec.2 Dockerfile（基于 osrf/ros:humble-desktop-full，预装所需包）

如果用下面的命令拉取官方Humble桌面完整版镜像'osrf/ros:humble-desktop-full',
会得到一个 自带ros-humble/rviz/colcon的镜像,但是其中不包括gazebo/nav2/slam-toolbox等.
每次重启docker 容器,都要重新安装这些依赖库,太麻烦了
```bash
docker pull osrf/ros:humble-desktop-full
```
### sec.2.1 Dockerfile
所以 用下面的 Dockerfile 创建镜像
```dockerfile
FROM osrf/ros:humble-desktop-full

ENV ROS_DISTRO=humble

# 一次性安装全部需要的包
RUN sed -i 's/archive.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list && \
    apt update && apt install -y \
    ros-$ROS_DISTRO-navigation2 \
    ros-$ROS_DISTRO-nav2-bringup \
    ros-$ROS_DISTRO-slam-toolbox \
    ros-$ROS_DISTRO-robot-localization \
    ros-$ROS_DISTRO-robot-state-publisher \
    ros-$ROS_DISTRO-joint-state-publisher \
    ros-$ROS_DISTRO-xacro \
    ros-$ROS_DISTRO-rviz2 \
    ros-$ROS_DISTRO-teleop-twist-keyboard \
    ros-$ROS_DISTRO-tf2-tools \
    ros-$ROS_DISTRO-gazebo-ros-pkgs \
    # === 新增下面3行 ===
    ros-$ROS_DISTRO-ros2-control \
    ros-$ROS_DISTRO-ros2-controllers \
    ros-$ROS_DISTRO-gazebo-ros2-control \
    python3-colcon-common-extensions \
    python3-rosdep \
    && rm -rf /var/lib/apt/lists/*
 
# rosdep初始化
# RUN rosdep init || true && rosdep update

# rosdep 清华镜像正确配置
RUN mkdir -p /etc/ros/rosdep/sources.list.d && \
    curl -o /etc/ros/rosdep/sources.list.d/20-default.list -L https://mirrors.tuna.tsinghua.edu.cn/github-raw/ros/rosdistro/master/rosdep/sources.list.d/20-default.list && \
    export ROSDISTRO_INDEX_URL=https://mirrors.tuna.tsinghua.edu.cn/rosdistro/index-v4.yaml && \
    rosdep fix-permissions && \
    rosdep update




# 交互式bash自动source ros环境
RUN echo "source /opt/ros/humble/setup.bash" >> /etc/bash.bashrc

CMD ["/bin/bash"]
```

重要说明
1. `echo "source ... >> /etc/bash.bashrc` 只对**交互式bash**生效（`-it` 启动的容器）。
   - 如果是非交互式脚本执行（`docker run 直接跑命令`），不会读取bashrc，需要手动在脚本开头写 `source /opt/ros/humble/setup.bash`。
2. 清理`/var/lib/apt/lists/*`是为**减小镜像体积**，推荐保留。

### 构建命令
```bash
docker build -t ros-humble-nav2-full .

(base) abner@abner-XPS-8940:~$ docker images
                                                                                                                 
IMAGE                                                     ID             DISK USAGE   CONTENT SIZE   EXTRA
ghcr.io/autowarefoundation/autoware:universe-cuda-jazzy   65fe7dd55a6c       16.8GB         5.02GB        
hello-world:latest                                        7f4da0fc94bc       25.9kB         9.49kB    U   
osrf/ros:humble-desktop-full                              1db1e4e941d4       5.41GB          1.2GB    U   
ros-humble-nav2-full:latest                               f1c1ea992884       6.29GB         1.42GB     
```

<!-- 
### 运行容器
```bash 
# 如果你需要GUI（RViz2/Gazebo），运行容器时要加x11转发参数，示例：
docker run -it --rm \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix:/tmp/.X11-unix \
    ros-humble-nav2-full
```
> 进入容器后，bash会自动加载 `/opt/ros/humble/setup.bash`，直接可以用 `ros2`、`rviz2`、`nav2` 相关命令。
> `docker-compose.yml` 可以一键启动这个带Nav2+Gazebo的容器  
 
### 验证包是否安装成功（进容器后执行）
```bash
ros2 pkg list | grep -E "navigation2|nav2_bringup|slam_toolbox"
```
-->

 
## sec.3 启动容器（带X11 + NVIDIA GPU透传，直接复制）

### sec.3.1 配置 X11 转发 (宿主机侧)
完整 X11 转发配置（Ubuntu24.04宿主机 + Docker ROS Humble，支持 RViz2 / Gazebo）
> 原理：宿主机X Server接收GUI渲染，容器把绘图指令转发给宿主机显示器。你是NVIDIA显卡，后面还要注意NVIDIA GPU透传。
 
#### ① 安装依赖（宿主机Ubuntu24.04）
```bash
# 宿主机安装 `x11-xserver-utils` → 提供`xhost`
sudo apt install x11-xserver-utils
``` 

#### ② 开放本地X访问权限 
```bash
# 确认宿主机DISPLAY环境变量
# 正常输出类似 `:0` 或者 `:1`。
echo $DISPLAY

conda deactivate

# ⚠️ 这个命令重启终端会失效，每次**启动容器前执行一次**。
# 含义：开放本地X访问权限,允许docker容器连接本机X11服务,用于RViz2窗口弹出（宿主机终端执行）
# ⚠️ `xhost +` 不推荐生产使用，开放权限有安全风险，开发调试临时用。
xhost +local:docker
```

### sec.3.2 启动容器命令
```bash 
# RUN 官方镜像 osrf/ros:humble-desktop-full
docker run -it \
  --name humble_dev \
  --network=host \
  --gpus all \
  --env DISPLAY=$DISPLAY \
  --env QT_X11_NO_MITSHM=1 \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /mnt/a2disk/a2/zdev/nv/adas-01/ros/:/root/ros \
  --shm-size=2g \
  osrf/ros:humble-desktop-full

# ------------------------------
# RUN 自定义镜像 ros-humble-nav2-full
docker run -it  --rm \
  --name humble_nav2 \
  --network=host \
  --gpus all \
  --env DISPLAY=$DISPLAY \
  --env QT_X11_NO_MITSHM=1 \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /mnt/a2disk/a2/zdev/nv/adas-01/ros/:/root/ros \
  --shm-size=2g \
  ros-humble-nav2-full  
# ---------------------------------
docker run -it   \
  --name humble_nav2 \
  --network=host \
  --gpus all \
  --env DISPLAY=$DISPLAY \
  --env QT_X11_NO_MITSHM=1 \
  --env http_proxy=http://192.168.1.110:7897 \
  --env https_proxy=http://192.168.1.110:7897 \
  --env all_proxy=socks5://192.168.1.110:7897 \
  --env HTTP_PROXY=http://192.168.1.110:7897 \
  --env HTTPS_PROXY=http://192.168.1.110:7897 \
  --env ALL_PROXY=socks5://192.168.1.110:7897 \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /home/abner/a2/zdev/nv/adas-01/ros/:/root/ros \
  --shm-size=2g \
  ros-humble-nav2-full
```

参数解释（图形相关部分）
- `--env DISPLAY=$DISPLAY`：把宿主机的显示器编号传入容器
- `--env QT_X11_NO_MITSHM=1`：**解决Qt程序（RViz）白屏/崩溃经典bug**
- `-v /tmp/.X11-unix:/tmp/.X11-unix`：挂载X11套接字，GUI通信核心
- `--gpus all`：NVIDIA显卡透传，RViz/Gazebo硬件加速，**必须装nvidia-container-toolkit才能生效**

参数解释（其他部分）
- `--rm`: 容器退出后自动删除，用完即清，不会残留停止的容器.
- `--network=host`：共享宿主机网络，ros2 dds节点互相发现，**仿真开发强烈推荐**
- `-v 宿主机路径:/root/fishbot_nav`：**挂载你的fishbot工作空间**，代码在宿主机保存，容器内编译
- `--shm-size=2g`：增大共享内存，Gazebo仿真必加，防止崩溃


> ⚠️ 如果你还没装nvidia容器工具包，`--gpus all` 会报错，下面给安装命令：
```bash
# Ubuntu24.04 安装nvidia-container-toolkit
curl -fsSL [https://nvidia.github.io/libnvidia-container/gpgkey](https://nvidia.github.io/libnvidia-container/gpgkey) | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
&& curl -s -L [https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list](https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list) | \
sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### sec.3.3 进入容器后
```bash
# 容器内加载ros环境
source /opt/ros/humble/setup.bash

# 测试1：基础图形测试（弹出xclock窗口即X转发成功）
xclock
# 测试2：直接启动rviz2
rviz2
# 测试3：gazebo
gazebo

# 进入你的工作空间
cd /root/fishbot_nav
# 编译
colcon build
source install/setup.bash
```
✅ 能弹出窗口=X11转发正常。


## sec.4. 持久化容器：后续重新进入容器流程
```bash
# 宿主机每次启动容器前，先授权X
xhost +local:docker

# 启动已创建的容器
# docker start humble_dev
docker start humble_nav2
# 进入终端
# docker exec -it humble_dev bash
docker exec -it humble_nav2 bash 

# 假设宿主机代理地址：127.0.0.1:7897 
docker exec -it \
  -e HTTP_PROXY=http://127.0.0.1:7897 \
  -e HTTPS_PROXY=http://127.0.0.1:7897 \
  -e NO_PROXY=localhost,127.0.0.1 \
humble_nav2 bash


#  进入容器后，需要手动 source ROS2 Humble 环境： 
source /opt/ros/humble/setup.bash
cd ~/ros/ros2humble_nav_ws
```

### sec.5.1 可选优化：永久xhost配置（不用每次敲xhost）
> 不推荐生产，仅开发机用：
编辑 `~/.bashrc`，末尾添加
```bash
export XHOST_DOCKER="xhost +local:docker"
```
缺点：图形权限放宽，仅本地开发环境使用。

## sec.6、工作流（推荐，非常适合你）
1. 宿主机Ubuntu24.04：VSCode打开 `/mnt/a2disk/a2/zdev/nv/adas-01/ros/` 写代码、改CMake/package.xml
2. 进入docker容器：执行colcon build、ros2 launch、rviz2
> 代码存在宿主机硬盘，容器删掉后代码也不会丢
 

## sec.7 常见坑

### 坑1：RViz 白屏 / 黑屏
1. 确认加了环境变量 `QT_X11_NO_MITSHM=1`
2. 确认 `--gpus all` 生效，容器内执行 `nvidia-smi` 能看到显卡信息
3. 容器内不要装额外QT库，避免版本冲突

### 坑2：`xhost: command not found`
宿主机缺少 `x11-xserver-utils`，执行 `sudo apt install x11-xserver-utils`

### 坑3：权限报错 `Cannot connect to display :0`
大概率忘记执行 `xhost +local:docker`，**每次宿主机重启/新开终端启动容器前必须运行**

### 坑4：Gazebo闪退
必须加 `--shm-size=2g`，Gazebo对共享内存要求高。

## sec.8 备选：docker-compose.yml 版本（推荐长期使用）
```yaml
version: "3.8"
services:
  ros_humble:
    image: osrf/ros:humble-desktop-full
    container_name: humble_dev
    network: host
    runtime: nvidia
    environment:
      - DISPLAY=${DISPLAY}
      - QT_X11_NO_MITSHM=1
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix
      - /mnt/a2disk/a2/zdev/nv/adas-01/ros/fishbot_nav:/root/fishbot_nav
    shm_size: 2g
    tty: true
    stdin_open: true
```
启动：
```bash
# 宿主机先执行
xhost +local:docker
docker compose up -d
docker compose exec ros_humble bash
```

> 注意：使用docker-compose的nvidia runtime，同样需要前面装好`nvidia-container-toolkit`。

## x.8 小提醒
- Wayland桌面（Ubuntu24.04默认）：X11转发**仍然能用**，但部分场景会不稳定。如果rviz一直无法弹出，可以把宿主机桌面切回Xorg（登录界面右下角选择）。

要不要我给你写一段容器内自动source humble环境的脚本，每次进容器不用手动敲`source /opt/ros/humble/setup.bash`？

## 五、两个关键坑提醒
1. **文件权限**：容器内创建的build/install目录，属主是root，宿主机修改文件会有权限问题。解决：启动容器时指定UID/GID，或者用完后宿主机`sudo chown -R $USER:USER fishbot_nav`
2. 不要在宿主机同时装Jazzy ROS2！宿主机完全干净，**所有ROS操作全部在容器内**，彻底杜绝环境混淆。

## 六、对比你的两个选择
|方案|优点|缺点|
|---|---|---|
|Docker Humble（推荐）|环境纯净，和教程完全一致，无Jazzy头文件兼容坑，隔离conda|图形、USB设备需要配置|
|原生Ubuntu24.04 + Jazzy|原生，显卡/串口简单|Nav2接口、头文件大量改动，教程Humble代码大量需要移植（你现在踩的坑）|

---

如果你需要，我可以直接给你一份**docker-compose.yml**，一键启动，不用每次敲一长串run命令，并且自动配置UID避免文件权限问题。
==================================
==================================
# ch2.`failed to fetch anonymous token: ... EOF`
## x.1报错原因
`failed to fetch anonymous token: ... EOF`
Docker 在国内直连 Docker Hub 认证服务器网络中断，**必须配置国内镜像加速器**才能拉取。

### 步骤1：配置docker国内镜像加速器（直接复制整条执行）
```bash
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "[https://docker.mirrors.sjtug.sjtu.edu.cn](https://docker.mirrors.sjtug.sjtu.edu.cn)",
    "[https://docker.mirrors.ustc.edu.cn](https://docker.mirrors.ustc.edu.cn)",
    "[https://docker.nju.edu.cn](https://docker.nju.edu.cn)"
  ]
}
EOF
```

### 步骤2：重载并重启docker
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 步骤3：验证镜像源是否生效
```bash
docker info
```
看输出里的 `Registry Mirrors`，出现上面三个地址就成功。

### 步骤4：重新拉取humble镜像
```bash
docker pull osrf/ros:humble-desktop-full
```

> 补充：如果还是报错，先测试最简镜像确认docker网络：
```bash
docker pull hello-world
```
hello-world能拉成功，说明加速器没问题，再去拉ros镜像。

## x.2 备选方案：如果镜像源依然拉不动（备选镜像地址拉osrf）
有些国内镜像站做了osrf镜像，可以直接换地址拉：
```bash
docker pull docker.xuanyuan.run/osrf/ros:humble-desktop-full
```

## x.3 额外提醒
1. 你现在终端是 `(base)` conda环境，**conda不影响docker**，docker是系统级程序，不用退出conda执行docker命令。
2. 拉取完镜像，后面启动容器命令还是我上一条给你的那一大串，挂载你的fishbot工作空间。
3. 镜像很大（几个GB），耐心等待。

### 备选B方案（不想折腾docker的Plan B）
如果你docker反复网络拉不下来，还有另一个方案：**在Ubuntu24.04用chroot容器装Ubuntu22.04 + ROS2 Humble**，不用docker，也能隔离环境，也可以图形RViz。 
=======================================

# ch3.分步检查 nvidia-container-toolkit
（按顺序执行，定位你 `CDI no known GPU vendor` 报错）
## 1. 检查包是否安装
```bash
dpkg -l | grep nvidia-container-toolkit
```
- 有输出代表包装好了；无输出 → 没安装，重新执行 `sudo apt install nvidia-container-toolkit`

## 2. 检查工具二进制可执行文件
```bash
nvidia-ctk --version
```
✅ 正常：输出版本号；
❌ command not found：包安装失败/环境异常。

## 3. 检查宿主机NVIDIA驱动（前提！）
```bash
nvidia-smi
```
> 你是3060Ti，这里必须正常输出显卡信息。如果`nvidia-smi`报错，宿主机驱动本身异常，docker GPU透传必然失败。

## 4. 检查docker是否识别nvidia runtime
```bash
docker info | grep -i runtime
```
✅ 正常输出会包含：`nvidia` runtime。
> 原理：`sudo nvidia-ctk runtime configure --runtime=docker` 会写入 `/etc/docker/daemon.json`
```bash
cat /etc/docker/daemon.json
```
文件里面应该包含 `runtimes` 字段，有nvidia配置。

## 5. 【重点：你的报错根源】CDI设备检查（`failed to discover GPU vendor from CDI`）
> CDI（Container Device Interface）是新版nvidia-ctk用来暴露GPU给容器的机制，你的报错就是CDI找不到GPU设备。
```bash
nvidia-ctk cdi list
```
✅ 正常：会列出 `nvidia.com/gpu=0`
❌ 空输出/报错 → CDI文件缺失，执行下面命令手动生成CDI：
```bash
sudo rm -f /etc/cdi/*.yaml
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
# 再次查看
nvidia-ctk cdi list
```
生成完CDI后，重启docker
```bash
sudo systemctl restart docker
```

## 6. 最终验证：容器内跑nvidia-smi（黄金测试）
```bash
docker run --rm --gpus all nvidia/cuda:12.2-base nvidia-smi
```
✅ 成功：容器内部打印你的3060Ti显卡信息；
❌ 仍然报CDI错误：就是刚才CDI没有生成成功。

=============================================
# ch4.`failed to discover GPU vendor from CDI: no known GPU vendor found`
这个错误是**CDI设备定义缺失**，在Ubuntu24.04 + 新版nvidia-container-toolkit很常见，修复命令：
```bash
sudo rm -rf /etc/cdi
sudo mkdir /etc/cdi
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
sudo systemctl restart docker
```
执行完成后，再测试：
```bash
nvidia-ctk cdi list
```

## 备选方案：绕过CDI（临时方案，优先跑ROS Humble）
如果CDI怎么修都不行，**先放弃GPU透传**，用不带`--gpus all`的容器启动命令，RViz用软件渲染，先解决你的Humble编译问题，GPU硬件加速后面再调。

```bash
docker run -it  --rm \
  --name humble_nav2 \
  --network=host \
  --env DISPLAY=$DISPLAY \
  --env QT_X11_NO_MITSHM=1 \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /mnt/a2disk/a2/zdev/nv/adas-01/ros/:/root/ros \
  --shm-size=2g \
  ros-humble-nav2-full bash 
```

## 操作建议
1. 先执行 `nvidia-smi` 确认宿主机显卡驱动正常
2. 然后执行 `nvidia-ctk cdi list`，看是否识别GPU
3. 如果list为空，手动生成CDI yaml，重启docker
4. 再跑 `docker run --rm --gpus all nvidia/cuda:12.2-base nvidia-smi` 验证

你把 `nvidia-ctk cdi list` 的输出贴给我，我帮你直接定位CDI问题。

> 补充：ROS开发场景，**GPU透传只是RViz/Gazebo硬件加速**，代码编译、Nav2算法开发完全不需要GPU。就算GPU透传失败，容器内Humble依然可以正常开发。

要不要我先直接启动不带GPU的humble容器，你先进去把fishbot的代码编译跑通，后续有空再调试nvidia CDI？


=====================================================



