# vcs2l 安装（Autoware / ROS2 源码编译必备，vcstool 的维护版替代品）
> vcs2l 是 vcstool 的活跃维护 fork，**命令行依旧是 `vcs`**，完全兼容旧的 `.repos` 文件，Autoware 现在推荐使用它替代老旧的 vcstool。

## 方式1：Ubuntu/Debian（推荐，apt安装）
> 如果你之前装过 `python3‑vcstool`，先卸载避免冲突
```bash
sudo apt purge python3-vcstool -y
```

### 配置ROS2 apt源（必须）
```bash
sudo apt update && sudo apt install curl -y

export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F'"' '{print $4}')

curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo ${UBUNTU_CODENAME:-${VERSION_CODENAME}})_all.deb"

sudo dpkg -i /tmp/ros2-apt-source.deb

sudo apt update
```

### 安装 vcs2l
```bash
sudo apt install python3-vcs2l -y
```

### 开启bash自动补全（可选）
```bash
echo 'source /usr/share/vcs2l-completion/vcs.bash' >> ~/.bashrc
source ~/.bashrc
```

## 方式2：pip安装（非Ubuntu系统，或不想加apt源）
```bash
pip3 install vcs2l
```

## 方式3：源码编译安装（开发调试）
```bash
git clone https://github.com/ros-infrastructure/vcs2l.git
cd vcs2l
pip3 install --user -e .
```

## ✅验证是否安装成功
```bash
vcs --version
# 输出类似 vcs2l 1.1.4 就成功
```

## Autoware 开发提示
Autoware 编译源码时，`vcs import`、`vcs pull`、`vcs export` 命令**完全和旧vcstool一样**，不需要修改任何脚本，直接使用即可。

> 如果你之前用 `ros-dev-tools`，建议替换为：
```bash
sudo apt purge ros-dev-tools -y
sudo apt install ros-build-essential python3-bloom python3-colcon-common-extensions python3-colcon-mixin python3-rosdep python3-vcs2l wget
```

需要我顺带给你一份 autoware 源码编译的完整简要步骤吗？