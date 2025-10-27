

````markdown
# usb_cam 使用说明

此仓库 fork 自 [ros-drivers/usb_cam](https://github.com/ros-drivers/usb_cam)，用于在 ROS 2 架构下驱动大部分 V4L (Video for Linux) USB 摄像头。相似的驱动包还有 `v4l2_camera`。

> **重要提示**
> 使用前请保证已经跟随 Notion 的 "Pnx_Robomaster/算法组主页/Deploy Wiki/部署流程" 完成所有前置部署。

---

## 1. 安装流程

在你的工作空间下执行以下步骤：

````bash
# 1. 进入 src 目录
cd src

# 2. 克隆仓库
git clone https://github.com/PnX-HKUSTGZ/usb_cam.git

# 3. 返回工作空间根目录
cd ../

# 4. 安装依赖 (此操作非必须但推荐)
rosdep install --from-paths src --ignore-src -r -y

# 5. 编译 usb_cam 包
colcon build --packages-select usb_cam

# 6. source 工作区
source install/setup.bash
````

---

## 2. V4L USB Camera 基础测试

### 2.1 查找设备与格式

````bash
# 找到所有连接的 USB 相机设备 (通常是 /dev/video0, /dev/video1 等)
v4l2-ctl --list-devices

# 查找特定相机(/dev/video0)支持的全部格式、分辨率和帧率信息
v4l2-ctl --list-formats-ext -d /dev/video0
````

> **注意**：请务必关注相机支持的 `pixel_format`。例如，ROS 2 的另一个驱动包 `v4l2_camera` 不支持 "MJPG" 格式。

### 2.2 支持的像素格式

此 `usb_cam` 版本支持以下格式 (你也可以在源码 `usb_cam/include/usb_cam/formats` 中查看)：
- `rgb8`
- `yuyv`
- `yuyv2rgb`
- `uyvy`
- `uyvy2rgb`
- `mono8`
- `mono16`
- `y102mono8`
- `raw_mjpeg`
- `mjpeg2rgb` (用于 "MJPG" 格式的相机)
- `m4202rgb`

### 2.3 故障排查：设备被占用

有时，由于节点崩溃，设备没有被正确释放，导致无法打开。

````bash
# 检查是否存在其他进程正在占用 /dev/video0
fuser /dev/video0

# 尝试使用 ffmpeg 截取一帧图像，以验证相机本身是否可用
# (请根据 v4l2-ctl 的输出修改 -input_format 和 -video_size)
ffmpeg -f v4l2 -input_format mjpeg -video_size 1920x1080 -i /dev/video0 -pix_fmt rgb24 -vframes 1 test.bmp

# 强制杀死所有占用 /dev/video0 的进程
sudo fuser -k /dev/video0
````

---

## 3. Node 基础参数

你可以在 `usb_cam/src/usb_cam_node.cpp` 中找到驱动支持的所有参数。

### 3.1 启动示例 (Launch 文件)

#### ComposableNode 方式
````python
from launch_ros.actions import ComposableNode

wide_camera_node = ComposableNode(
    package='usb_cam',
    plugin='usb_cam::UsbCamNode',  # 使用组件插件名称
    name='wide_camera_node',
    parameters=[node_params],
    remappings=[
        ('/image_raw', '/wide_cam/image_raw'),
        ('/camera_info', '/wide_cam/camera_info')
    ],
    extra_arguments=[{'use_intra_process_comms': True}]
)
````

#### 普通 Node 方式
````python
from launch_ros.actions import Node

wide_camera_node = Node(
    package='usb_cam',
    executable='usb_cam_node_exe',
    name='wide_camera_node',
    parameters=[node_params],  
    remappings=[
        ('/image_raw', '/wide_cam/image_raw'),
        ('/camera_info', '/wide_cam/camera_info')
    ],
    output='screen'
)
````

### 3.2 参数文件示例 (`node_params.yaml`)

````yaml
/wide_camera_node:
  ros__parameters:
    video_device: "/dev/video0"
    framerate: 30.0
    camera_frame_id: "wide_cam_optical_frame"
    pixel_format: "mjpeg2rgb"  
    av_device_format: "YUV422P"
    image_width: 1920
    image_height: 1080
    camera_info_url: "package://usb_cam/config/camera_info.yaml"
````

---

## 4. 相机内参

大部分消费级 USB 相机不会提供详细的内参数据。你可以使用 ROS 2 官方的 `camera_calibration` 包或我们自己的相机标定程序进行标定。

---

## 5. 其他相机参数（曝光、亮度等）

> **注意！**
> `usb_cam` 作为一个通用驱动，其启动时设置的参数名不一定与相机硬件提供的参数名完全一致。

### 5.1 查看相机支持的控制参数

使用以下命令可以打印出相机硬件层面支持的所有可控参数及其当前值、范围和默认值。

````bash
v4l2-ctl --list-ctrls -d /dev/video0
````

请将此列表与 `usb_cam/src/usb_cam/usb_cam.hpp` 中定义的参数进行对比。

### 5.2 处理参数不匹配问题

如果 `usb_cam` 提供的参数无法满足你的相机需求，可以尝试以下方法：

1.  **直接在终端修改（临时方案）**
    这种方法简单粗暴，但每次重启设备或节点后都需要重新设置。可以考虑将其写入自启动脚本。
    ````bash
    v4l2-ctl -d /dev/video0 -c [parameter_name]=[value]
    # 示例: v4l2-ctl -d /dev/video0 -c exposure_auto=1
    ````

2.  **修改源码（推荐）**
    请修改此包的源代码以支持你的相机或修复参数映射，并提交 Pull Request。建议在提交时注明你使用的相机型号。

