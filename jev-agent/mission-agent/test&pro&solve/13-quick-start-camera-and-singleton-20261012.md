# Quick Start 相机分辨率与重复启动保护

更新：2026-10-12。

## 相机模式

| 选项 | Arm RGB | D455 RGB / Depth | 默认 |
| --- | --- | --- | --- |
| 标准 | 640x480 | 640x480 | 是 |
| 低分辨率 | 320x240 | 320x240 | 否，用户勾选后保存 |

这里调整渲染/发布分辨率，不改变 ROS raw 图像编码，也不改 JPEG 压缩策略。640x480 来自当前 real 相机配置基准；没有声称是传感器可支持的最大分辨率。低分辨率保持同样4:3比例。

- Quick Start 的 Isaac 区新增低分辨率复选框，Full/Arm-only 都适用。
- 新 Isaac 进程环境、四个独立复制按钮、命令预览使用相同 profile。
- 四个前/后 Play wrapper 都在运行子脚本之前应用 profile，避免从 Arm-only/低分辨率切回 Full/标准时保留旧分辨率。
- Arm-only 不再自动降分辨率。D455 独立脚本默认从640x360改为640x480；Arm 独立脚本保持640x480。
- 统一维护入口：`src/robot_sim/robot_sim/camera_profile.py`。`ISAAC_CAMERA_LOW_RESOLUTION=1` 选低分辨率，0选标准。
- wrapper 按统一 profile 覆盖旧 per-camera resolution 变量；直接运行独立 camera 脚本时仍可使用该脚本原有的分辨率环境覆盖。
- 相机 CameraInfo 来自对应 render product；原有重新执行时 detach/recreate publisher 的机制保留。
- 本轮不改变帧率。用户主动选低分辨率时，sim/real相机规格差异告警仍可能出现，不应伪装成与real完全一致。

生效方法：重新打开 Quick Start，选定分辨率，Stop 状态运行对应 Play 前代码，Play 后运行对应 Play 后代码。当前正在运行的相机不会因为编辑源码自动改变。本轮没有重启当前 Isaac 或运动机器人。

## 启动链路审查

一次 Quick Start 调用的结构为：ROS sim/Nav2/Arm 启动脚本 + 独立 Dashboard 脚本。两个 sim ROS 脚本没有另外启动 agent_node、Dashboard、Health Monitor、Recovery Agent；agent_dashboard.launch.py 对每个各定义一次。

风险来自：
1. 用户重复点击或打开多个 Quick Start 窗口。
2. 旧 Dashboard/Agent 窗口和手动启动的 Python 进程残留。
3. 原先的“扫描进程 -> 启动”不是原子操作，两个启动请求可能同时看到没有节点。
4. ROS cleanup 主要清理 sim/Nav2/Arm，并不等价于结束 Agent 服务。

本轮保护：
- GUI 启动前扫描同工作区、同 ROS domain 的四种可执行脚本及尚在启动的 ros2 launch，明确提示 PID；不会静默复用不同配置的旧 Agent。
- 重入时检查 Start 按钮状态。
- `agent_stack_guard.py --run ...` 先获取工作区/domain专属 flock，再检查现有进程；锁通过exec继承，由ros2 launch在整个生命周期持有。
- sim/real Dashboard shell入口都使用该 guard。旧 launch 意外退出留下孤儿节点时，下一次仍由进程扫描拦截。
- 只拒绝重复启动，不自动 kill，不取消现有 Mission；复用当前服务时可取消勾选启动 Dashboard。

边界：绕过 shell入口直接执行 ros2 launch/ros2 run，其他机器或工作区，仍不能据本锁保证全局单例。不同 ROS domain 也不能据此保证同一个HTTP端口可重复绑定；此处锁用于同工作区/domain执行服务互斥。

## 验证

- 17 项测试通过：标准/低分辨率、四个wrapper、旧分辨率清除、clipboard生成、Python孤儿/launch父进程识别、锁互斥/释放/继承。
- Xvfb下创建真实GTK窗口，实际点击四个复制按钮，验证320x240 -> 640x480切换以及预览；模拟重复进程时，确认未调用启动终端。
- 增加滚动容器，避免设置窗口高度超过屏幕后 Start 区域不可访问。
- 现有进程下运行guard确认拒绝第二次启动（退出1），不触碰运行中的服务。
- 语法编译和shell语法检查通过。尚未在Isaac内重新运行publisher并测量新图像尺寸/RTF；由用户下一次选定配置后验证。
