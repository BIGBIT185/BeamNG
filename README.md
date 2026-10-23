# BeamNG 部署与数据接收

Windows 运行 BeamNG 并采集点云、图像，Linux 通过普通 SSH 接收并发布 ROS 话题。

云盘下载链接：

## 一、Windows 部署

**1. 解压运行包**

将完整 BeamNG_files 压缩包解压到 D:\BeamNG_files（可换成其他路径）。保留包内 game、BeamNGRL、python_runtime 和 windows_recorder/.venv。

**2. 配置路径（首次解压后执行一次）**

在 Windows PowerShell 中执行：

~~~powershell
Set-Location D:\BeamNG_files
$env:PATH = "$PWD\python_runtime;$PWD\python_runtime\Library\bin;$env:PATH"
& .\python_runtime\python.exe -I -B -m venv --without-pip --upgrade .\windows_recorder\.venv
& .\windows_recorder\.venv\Scripts\python.exe -X utf8 -I -B .\configure_paths.py
~~~

这会将包内 Python 环境和采集配置指向当前解压目录，保留已有依赖。

**3. 启动游戏和采集**

在 Windows 桌面双击 start_game.cmd。游戏打开后，在同一 PowerShell 中执行：

~~~powershell
& .\windows_recorder\.venv\Scripts\python.exe -X utf8 -I -B .\windows_recorder\recover_scenario.py
& .\windows_recorder\.venv\Scripts\python.exe -X utf8 -B .\windows_recorder\bridge_control.py start
~~~

看到 streaming 表示采集已启动。默认发送点云和图像，不录包、不启用导航。

## 二、Linux 接收数据

前提：Linux 已有 Docker 和 SSH 客户端，可以通过 SSH 登录 Windows。

**1. 准备接收文件**

从运行包中取出以下两个文件，复制到 Linux，保持目录结构：

~~~text
~/BeamNG_files/
├─ scripts/full_bridge_ros.py
└─ windows_recorder/bridge_transport.py
~~~

**2. 配置 ROS 接收环境（首次执行一次）**

~~~bash
cd ~/BeamNG_files
docker build -t beamng-ros-receiver:noetic - <<'DOCKERFILE'
FROM ros:noetic-ros-base-focal
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3-numpy ros-noetic-grid-map-msgs ros-noetic-tf2-msgs \
    && rm -rf /var/lib/apt/lists/*
DOCKERFILE

docker run -d --name beamng_ros --network host \
  -v "$PWD:/workspace:ro" \
  -e ROS_MASTER_URI=http://127.0.0.1:11413 -e ROS_IP=127.0.0.1 \
  beamng-ros-receiver:noetic roscore -p 11413
~~~

**3. 建立 SSH 转发**

在 Linux 终端一执行，将用户名和 IP 换成实际 Windows 信息，保持此终端运行：

~~~bash
ssh -NT -o ExitOnForwardFailure=yes \
  -L 127.0.0.1:18765:127.0.0.1:8765 windows_user@192.168.1.20
~~~

**4. 启动接收器**

在 Linux 终端二执行：

~~~bash
docker start beamng_ros
docker exec -it beamng_ros bash -lc 'source /opt/ros/noetic/setup.bash && rosparam set /use_sim_time true && /usr/bin/python3 /workspace/scripts/full_bridge_ros.py --port 18765'
~~~

**5. 检查点云和图像**

在 Linux 终端三依次执行；每条频率命令观察后按 Ctrl+C，再执行下一条：

~~~bash
docker exec -it beamng_ros bash -lc 'source /opt/ros/noetic/setup.bash && rostopic hz --wall-time /lidar/points'
docker exec -it beamng_ros bash -lc 'source /opt/ros/noetic/setup.bash && rostopic hz --wall-time /camera/color/image_raw'
~~~

持续显示频率即表示对应话题正在接收。以后运行时跳过首次配置，只需启动 Windows 游戏和采集，再建立 SSH 转发、启动接收器。
