FROM osrf/ros:humble-desktop-full

ENV ROS_DISTRO=humble

# 一次性安装全部需要的包
# 
# navigation2 是 meta-package（元包），它的依赖列表内置 nav2_map_server、
# nav2_amcl、nav2_planner、nav2_controller 全套 Nav2 组件。
# 
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