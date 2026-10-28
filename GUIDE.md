# 视觉与导航仿真：操作与验收指南

本指南按一次比赛/考核的完整时间线组织：环境准备 → 资产核查 → 分链调试 → 赛前彩排 → 正式演示 → 赛后复盘。它不附带现场代码、脚本、模型和视频；标注“历史经验”的内容来自一次已有环境的实验，**不是在任意新环境验证通过的安装承诺**。执行会改变系统、工作区或进程的命令之前，先获得必要权限并确认影响范围。

WSL 安装流程（第 2 节）的命令转录自 ROS 2 与 Microsoft 官方文档，**未在本仓库维护者环境实测**；以官方文档当前版本为准。

## 1. 比赛流程总览

| 阶段 | 目标 | 关键动作 | 退出判据 |
|---|---|---|---|
| 0 环境准备 | 可用 ROS 工作环境 | 装 WSL/Ubuntu/ROS、装 pixi、验证 demo 节点 | talker/listener 通信正常 |
| 1 资产核查 | 上游与素材齐备且合规 | 拉源码、记 commit、核模型/地图/许可 | 每条链的输入清单核对完毕 |
| 2 分链调试 | 单链各自跑通 | 导航 → 离线视觉 → 算法回归 → 渲染闭环 | 每链按第 4-7 节判据逐项通过 |
| 3 赛前彩排 | 演示可一键复现 | 固化启动顺序、留证命令、演练故障预案 | 全流程计时跑通一次并留证 |
| 4 正式演示 | 分链展示、数据说话 | 按预定顺序演示、实时核对判据字段 | 每链当场拿到状态字段+位移/日志 |
| 5 赛后复盘 | 证据归档、口径诚实 | 归档日志、写结论、标注未验证项 | 记录完整、无夸大声明 |

**两条原则贯穿全程**：一条链通过不代表其他链通过；每一步的“成功”必须有状态字段、实测数据或用户目视支撑，进程存活与退出码 0 都不是充分条件。

**为什么链的顺序是“导航 → 离线视觉 → 算法回归 → 渲染闭环”**：依赖数量递增、失败排查半径递增。导航只依赖 ROS 本身（环境问题最先暴露）；离线视觉加编译链与推理库（ABI/模型问题）；算法回归加你自己的程序（逻辑问题）；渲染闭环加图形驱动、坐标变换、实时消息（域差与时序问题）。先跑通短链，环境问题就不会污染后面链的排查——如果直接从渲染闭环开始，一个报错可能是环境、编译、模型、坐标、QoS 五种原因，无从下手。

## 2. 环境准备：Windows + WSL2 安装 ROS 2 Humble

适用：只有 Windows 机器、需要 Ubuntu 22.04 + ROS 2 Humble 环境的情况。历史实验在原生 Ubuntu 22.04 上完成；WSL 适合导航与离线视觉链，**渲染仿真链（Vulkan/wgpu）在 WSL 下支持有限，建议原生 Linux 或双系统**。

### 2.1 Windows 侧准备

1. 确认系统为 Windows 10 22H2（内部版本 19045 及以上）或 Windows 11，并已启用虚拟化（任务管理器→性能→CPU→虚拟化：已启用）。
2. 以管理员身份打开 PowerShell，安装 WSL 与 Ubuntu 22.04：

```powershell
wsl --update
wsl --install -d Ubuntu-22.04
```

3. 首次启动 Ubuntu 设置用户名和密码（该密码用于 `sudo`）。

### 2.2 启用 systemd（ROS 依赖服务的标准方式）

WSL 版本需 0.67.6 及以上（`wsl --version` 可查，旧版本先 `wsl --update`）。在 Ubuntu 内执行：

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

然后在 PowerShell 中重启 WSL 使其生效：

```powershell
wsl --shutdown
```

重新打开 Ubuntu，用 `systemctl status` 确认 systemd 处于 running 状态。

### 2.3 安装 ROS 2 Humble（官方 deb 方式）

**官方警告**：全新安装的系统必须先更新系统包（systemd/udev 相关），否则安装 ROS 2 可能触发关键系统包被移除。先做：

```bash
sudo apt update && sudo apt upgrade -y
```

再按 [ROS 2 官方安装文档](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html) 配置 apt 源：

```bash
sudo apt update && sudo apt install curl -y
export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F '"' '{print $4}')
curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo ${UBUNTU_CODENAME:-${VERSION_CODENAME}})_all.deb"
sudo dpkg -i /tmp/ros2-apt-source.deb
```

安装桌面版（含 RViz 与演示）并配置环境：

```bash
sudo apt update
sudo apt install ros-humble-desktop -y
echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
exec bash
```

### 2.4 安装 rosdep 与常用工具

```bash
sudo apt install python3-rosdep python3-colcon-common-extensions git -y
sudo rosdep init || true   # 已初始化时报错可忽略
rosdep update
```

rosdep 的用途：拿到上游 ROS 包源码后，`rosdep install --from-paths src -y` 按包声明自动装依赖，避免逐个手装漏项。

### 2.5 验证安装

```bash
ros2 --help >/dev/null && echo ROS_CLI_OK
```

开两个 Ubuntu 终端，分别运行：

```bash
ros2 run demo_nodes_cpp talker
```

```bash
ros2 run demo_nodes_cpp listener
```

listener 持续打印 `I heard: [Hello World: N]` 即通信正常，Ctrl+C 结束。图形验证：WSLg（Windows 11 内置）下直接运行 `rviz2` 能出窗口即可。

### 2.6 WSL 特有注意事项

| 事项 | 说明 |
|---|---|
| 文件位置 | 源码和构建一律放在 Linux 文件系统内（如 `~/`），不要在 `/mnt/c` 下编译（IO 慢一个数量级以上） |
| 渲染仿真 | WSLg 对 Vulkan/wgpu 支持有限，Bevy 类渲染仿真建议原生 Linux；导航 RViz 一般可用 |
| 资源限制 | 大工作区（历史导航环境约 6GB+）注意 `C:\Users\<你>\.wslconfig` 限制的内存与磁盘余量 |
| 备份 | 赛前用 `wsl --export Ubuntu-22.04 备份.tar` 做整盘快照 |
| 时钟漂移 | WSL 长时间挂起后时钟可能漂移，影响 ROS 时间戳；重要演示前重启 WSL |
| 实车控制 | 不要用 WSL 跑实车控制或低延迟串口链路 |

## 3. 资产核查（比赛准备期）

### 3.1 资源地图：去哪找什么

| 需要什么 | 首选渠道 | 具体入口 | 说明 |
|---|---|---|---|
| ROS 2 文档与教程 | 官方文档 | https://docs.ros.org/en/humble | 安装、概念、TF/QoS 都有专章 |
| Nav2 导航框架文档 | 官方文档 | https://docs.nav2.org | 参数调优、生命周期、故障排查 |
| RM 官方开源组织 | GitHub | https://github.com/robomaster-oss | rmoss 系列（Gazebo 插件、相机接口）等官方开源 |
| RM 各队开源 | GitHub 搜索 | 搜 `robomaster vision`、`robomaster simulator`，按 star/更新时间排序 | 各队赛季开源，看 README 的赛季与平台标注 |
| RM 社区文章 | RoboMaster 论坛 | https://bbs.robomaster.com | 战队开源专栏；历史实验的导航上游即出自论坛文章 1882897 的开源链 |
| 导航仿真（历史实验用） | GitHub | https://github.com/qza36/COD_loopback_sim | Humble 适配 loopback sim；许可证状态需自行核实 |
| 视觉框架（历史实验用） | GitHub | https://github.com/TongjiSuperPower/sp_vision_25 | 仓库元数据为 MIT；模型/视频等资产许可须逐项核对 |
| 渲染仿真（历史实验用） | GitHub | https://github.com/HNUYueLuRM/bevy_robomaster_simulator | Bevy 引擎 RM 战场，AGPL-3.0 |
| pixi 包管理器 | 官网 | https://pixi.sh | 安装、文档；用锁文件复现环境 |
| OpenVINO | 官方 | https://docs.openvino.ai | 版本、ABI 支持、各安装渠道对照 |
| 地图素材 | 上游自带 / 自制 | 仿真仓库 maps/ 目录；自制用 PGM+YAML | 自制注意分辨率、原点与障碍膨胀 |

**拿到任何上游仓库后先做四件事（查找方法论）**：

1. 读 README 的“支持平台 / ROS 版本 / 依赖”三行——版本不匹配是最常见的死因（历史经验：一个 Gazebo 系方案被排除的原因是 ROS 版本要求不兼容，而不是“太重”）。
2. 看 LICENSE（仓库根或 GitHub 侧栏）——决定能否再分发、修改后是否须开源。
3. 拿错误关键字搜仓库 issues——编译/运行报错大概率有人先踩过。
4. `git log --oneline -5` 看最近提交——长期无维护的仓库遇到版本漂移只能自己扛。

### 3.2 四条链各自的资产清单

| 链 | 必需资产 | 从哪来 | 缺了会怎样 |
|---|---|---|---|
| 导航 | 仿真源码+pixi 锁文件、地图 YAML+PGM、初始位姿与目标点 | COD_loopback_sim 或同类；地图上游自带或自制 | 起不来或没有障碍回波 |
| 离线视觉 | 视觉框架源码、OpenCV/OpenVINO 依赖、模型权重、示例视频 | sp_vision_25 或同类；模型与视频看上游 assets/ 与其许可 | 编译失败或运行无检测 |
| 算法回归 | 自研在环程序、场景配置、上游解算/跟踪库 | 自研 + 上游库 | 无法对照 |
| 渲染闭环 | 渲染仿真、相机内参标定、检测模型、消息桥 | Bevy 上游 + 标定自做 + 桥自研 | 图像有了但对不上控制 |

历史实验环境为 x86_64、Ubuntu 22.04、ROS 2 Humble；导航使用 pixi；视觉实验使用 OpenVINO 和 CPU。具体版本以所选上游的官方说明、锁文件和目标机器实际情况为准，切勿假设其他架构二进制兼容。

### 3.3 只读环境核查

```bash
uname -m
. /etc/os-release; printf '%s\n' "$PRETTY_NAME"
command -v git; git --version
command -v pixi && pixi --version
command -v ros2 && ros2 --help >/dev/null
```

ROS 终端可能需要先 `source /opt/ros/humble/setup.bash`；pixi 可能安装在用户目录但不在非交互 SSH 的 `PATH` 中（历史路径为 `~/.pixi/bin/pixi`）。缺命令时先查本机和上游安装文档，不要把“未在 PATH”当作“未安装”。

把上游版本（commit）、自己的改动、配置和模型哈希写进实验记录；不要把别人的权重、视频和修改代码当作本仓库 MIT 内容。

## 4. 分链调试 · 导航

**为什么第一条链是导航**：它只依赖 ROS 2 与一个仿真上游，能把“环境装没装对、话题通不通、TF 建没建”这些基础问题全部暴露；这些问题不先解决，后面三条链的任何报错都要多猜一层。

### 4.1 构建与启动：逐步操作

以下步骤以 [COD_loopback_sim 上游](https://github.com/qza36/COD_loopback_sim)为参照；命令以该上游当前文档为准，此处给的是流程骨架与每步的判断方法。

**第 1 步 · 获取源码并记录版本**

```bash
mkdir -p ~/rm_ws/src && cd ~/rm_ws/src
git clone --depth 1 https://github.com/qza36/COD_loopback_sim
cd COD_loopback_sim && git rev-parse HEAD | tee ../COD_commit.txt
```

为什么记录 commit：后续任何“跑不起来”的排查、以及向别人求助，第一句话都要能答“我用的是哪个版本”。`--depth 1` 只取最新提交，够用且快。

**第 2 步 · 核对平台支持并准备 pixi 环境**

先看 `pixi.toml`（或 `pyproject.toml`）的 `platforms` 字段是否包含 `linux-64`（WSL 里是 x86_64）。历史经验：该上游最初只写了 `osx-arm64`，Linux 直接 `pixi install` 会报 no candidates，需要按 pixi 文档先 `pixi workspace platform add linux-64` 再安装：

```bash
grep -n "platforms" pixi.toml
df -h ~                       # 先确认磁盘：历史环境装完约 6GB
pixi install                  # 失败提示平台缺失时，先加平台再重试
```

**第 3 步 · 构建工作区**

```bash
pixi run build    # 以上游 README 的构建命令为准；上游曾警告禁用 --symlink-install
```

判断成功：构建日志无 error，且 `install/` 目录下生成 `setup.bash`。失败时先看第一条 error 而不是最后一条；把第一条错误信息拿去搜上游 issues。

**第 4 步 · 首次启动（headless）**

```bash
pixi run bash -c 'source install/setup.bash; ros2 launch nav2_loopback_sim tb3_loopback_simulation_launch.py use_rviz:=false'
```

为什么要先 headless：先验证数据链（地图话题、TF、激光），再引入图形界面这个新变量。启动后**另开一个终端**做三项检查：

```bash
source /opt/ros/humble/setup.bash
ros2 topic list                 # 应看到 /map /scan /odom /tf 等
ros2 topic echo /map --once | head -20    # width/height 非零 = 地图真的加载了
ros2 run tf2_ros tf2_echo map base_footprint   # 持续输出 = map→odom 已建立
```

三项全过 = 仿真数据链健康。任何一项不过，先停在这里排查（见 4.3 与第 11 节），不要急着发导航目标。

**第 5 步 · 落初始位姿**

```bash
ros2 topic pub --once /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
  '{header: {frame_id: map}, pose: {pose: {position: {x: 0.0, y: 0.0}, orientation: {w: 1.0}}}}'
```

为什么必须这步（历史实测的机制）：该回环仿真器**只在收到 `/initialpose` 之后才发布 map→odom 变换**。不发初始位姿，RViz 看不到车、Nav2 报 frame 不存在——这是“地图有了车不见”的第一根因。若 `ros2 topic pub` 丢消息收不到（历史环境 FastDDS 共享内存问题），改用小段 rclpy 脚本以 1Hz 重发直到 TF 就绪。

**第 6 步 · 发一个短距目标并实测位移**

用 RViz 的 "2D Goal Pose" 或 Nav2 的 NavigateToPose（如 RViz 工具不方便，写一小段 rclpy action 客户端）发一个 0.5-1m 的目标。判定三点：目标被 accepted、最终状态 SUCCEEDED、odom 起终位移 ≈ 目标距离。**accepted 不等于走完**，退出码 0 也不等于成功——只认状态字段加实测位移。

**第 7 步 · 图形界面检查（可选）**

`use_rviz:=true` 重启后再看两个历史坑：官方 RViz 配置里 RobotModel 默认 Enabled:false（车永远不画，需手动勾选或换自建配置）；正交相机 Scale 过大时小场地只显示局部。

### 4.2 验收清单

| 顺序 | 检查 | 合格依据 |
|---|---|---|
| 地图 | 地图话题与加载日志 | 尺寸、原点和图像与预期一致 |
| 坐标 | map、odom 与机器人坐标变换 | 初始位姿后变换持续可查 |
| 行动 | NavigateToPose 反馈与最终状态 | 最终 SUCCEEDED；超时或取消不算成功 |
| 位移 | 起终坐标和目标距离 | 与目标距离和容差一致，不能只看接受消息 |
| 可视化 | 实际显示窗口 | 观察连续运动，而非仅看进程或静态画面 |
| 激光 | `/scan` 与地图障碍 | 有合理障碍回波；发布端 BEST_EFFORT 时订阅端必须匹配，否则静默无数据 |

### 4.3 停止与清场（为什么不能直接关终端）

`ros2 launch` 被 Ctrl+C 或杀掉后，map_server、robot_state_publisher、lifecycle_manager 等会变成挂在 init 下的孤儿进程；下次启动时 DDS 里同名节点打架，典型症状是 map_server 永远停在 unconfigured、拿不到地图。正确做法是写清场脚本：按**仿真环境安装路径前缀**精确匹配（不是按宽泛的进程名），先 TERM 等 3 秒再 KILL，最后用 `pgrep -f <环境路径> | wc -l` 校验零残留。宁可多花十秒清场，不要留给下一轮一个“地图玄学坏了”的现场。

## 5. 分链调试 · 离线视觉

**为什么第二链用离线视频**：把“算法栈能不能跑”与“图像从哪来”解耦。检测、解算、跟踪链路的编译与推理问题，在没有实时图像源时排查半径最小；这条链通了，渲染闭环出问题时你才知道嫌疑只剩图像源和坐标。

1. 从 [sp_vision_25 上游](https://github.com/TongjiSuperPower/sp_vision_25)取得源码，按对应版本的官方构建说明准备 OpenCV、OpenVINO、CMake 等依赖；构建命令以上游文档为准（历史环境为 CMake + 指定 OpenVINO 路径）。
2. 核对配置所指模型和视频的路径、格式和许可；验证文件存在不等于验证模型能识别目标。
3. 按上游说明运行离线示例，记录输入视频、帧数、错误、退出状态和推理耗时分布。无图形会话时可按应用要求配置虚拟显示（如 `xvfb-run`），但应先确认它与本机环境兼容。
4. 对照实际画面检查检测框和跟踪行为；只报告运行成功时可观察到的结果，不把它称作仿真图像或真实机器人测试。

**ABI 排障经验（这条链最常见的编译死因）**：若链接报大量涉及 `std::string` 的 undefined reference，这是两个 C++ ABI 版本混链的指纹，不是代码写错。定位三步：看 `build/CMakeFiles/*.dir/flags.make` 里有没有被某个依赖的 CMake 配置注入 `-D_GLIBCXX_USE_CXX11_ABI=0`；对比报错符号的 mangled 名（`RKSs` 结尾=旧 ABI，`__cxx11`=新 ABI）；找出哪个依赖是 ABI=0 构建的（历史案例：pip 安装的 OpenVINO wheel 是旧 ABI，与系统库冲突，换 conda 主包后解决）。修复原则：**全链统一 ABI**，缺哪个版本就换哪个渠道的包，不要在编译参数上碰运气，更不要直接覆盖系统共享库。

## 6. 分链调试 · 算法在环与计分

**为什么做真值注入**：把装甲板角点直接从仿真真值喂给解算器，绕过检测环节。目的不是省事，而是把“后半链（PnP 解算→EKF 跟踪→预测→决策→弹道）”的改进效果从检测噪声里隔离出来量化。代价是明确且必须声明的：它不包含漏检、误检、域差和检测延迟，所以成绩只能用于同一模拟器内的算法回归对照。

**最佳方法**：

1. 固定场景、距离、运动模式、算法版本、配置与随机种子，并记录原始输入。为什么固定种子：不固定则两次跑分差异分不清是改动还是随机。
2. 每次只修改一个变量，在相同考卷上跑修改前后；保留失败、超时和未检出的样本。为什么单变量：双变量同改的“提升”无法归因。
3. 以总命中数除以总发射数统计微平均命中率；对未成功样本明确计入失败惩罚（如按上限时间计），不只平均成功案例。为什么微平均：逐场景百分比再平均会给 5 发的小场景与 500 发的大场景同等权重。
4. 在独立验证命中判定之前，几何近似结果只能用于同一模拟器内部回归，不可当作实车命中率。
5. 记录场景版本、模型哈希、命令、日志和结果。人工接管的片段不计自主成绩。

历史实验曾有 16 个理想化场景及其统计记录（近中距击杀占优、7m 高速旋转目标为短板），但本仓库不发布原始数据和现场程序，读者不能仅凭该数字声称复现成功。

## 7. 分链调试 · 渲染图像接入

**为什么这条链放最后**：它叠加图形驱动、实时话题、坐标变换、控制回路四层不确定性，任何一层出问题都会表现成“检测不稳”。前置链全部通过后，这里的故障面被压缩到图像域差与实时性。

1. 从 [Bevy 模拟器上游](https://github.com/HNUYueLuRM/bevy_robomaster_simulator)获取代码和资源，核对 ROS 版本、图形驱动、消息定义、坐标系及许可证（上游为 AGPL-3.0，分发义务按实际使用方式核实）。
2. 以实际源码确认图像、相机信息、云台反馈和命令接口。历史实验发现 README 描述与源码接口曾不同（实际订阅的不是 README 写的话题）：**名称相似不等于消息类型、单位和语义兼容**；核对方法是直接读消息定义源码（`msg/` 目录或类型包），确认字段、单位（度/弧度）与坐标系约定。
3. 建立相机内参、外参和坐标转换；先验证图像发布与接收，再接检测模型。为什么先内参后检测：内参错则 PnP 解算系统性偏差，检测再准也白搭；先用棋盘格或已知尺寸物体标定，再验证重投影误差。
4. 顺序验收：图像连续 → 真实检出 → 身份稳定跟踪 → 云台反馈 → 偏心回正 → 遮挡恢复 → 多目标切换。为什么不能跳级：每级是下一级的前提——跟踪不稳时测回正毫无意义；跳级得到的“通过”在上一级破坏时会整体塌掉。
5. 单独测试命中、偏离和遮挡三种碰撞情形（对准必中、明确偏离必不中、遮挡不穿墙），通过后再启用正式开火和成绩统计。选择固定初态、相同种子、自动重置并记录全量失败。

**不能跳级**：话题能连、画面能显示或命令已发出，都不能证明闭环稳定或命中真实有效。历史接入尚未完成全部上述验收。

## 8. 赛前彩排：一键化与取证

彩排的目标是让正式演示不现场调试。按以下清单固化：

1. **启动顺序文档化**：每条链写清“从干净状态到可演示”的确切命令序列，包括环境变量、工作目录、预期日志关键字和等待判据。
2. **清场脚本化**：停止仿真用精确匹配环境路径的脚本（两轮 TERM→KILL 后校验零残留），演示前先清场再启动，避免孤儿节点抢占。
3. **取证命令固化**：每条链准备一条“演示即取证”命令，输出直接落到带时间戳的日志文件；判据字段（状态码/位移/帧数/CSV 行数）当场可读。
4. **计时全流程**：从开机到全部链演示完成计时跑一遍；历史经验表明换图、首次编译、模型加载的耗时容易被低估。
5. **故障预案**：对每个链准备一条“演示失败时的降级话术+证据”——诚实报告该链当前状态，比现场修环境更可信。

## 9. 正式演示：顺序与判据

| 顺序 | 演示内容 | 当场判据 | 说明 |
|---|---|---|---|
| 1 | 导航运动 | 动作状态 SUCCEEDED + 实测位移 ≈ 目标距离 + 用户目视连续运动 | 先演示最稳定链 |
| 2 | 离线视觉 | 帧数、退出码 0、推理耗时分布 | 明确声明输入是离线视频 |
| 3 | 算法在环刷分 | 全场景 CSV 完整、统计口径正确 | 明确声明检测为真值注入 |
| 4 | 渲染闭环（如具备） | 分级验收通过的部分如实展示 | 未过验收的门禁如实说明 |

演示话术口径：主动声明每条链的输入源与边界，不混淆“导航仿真/离线录像/理想化角点/渲染图像”四种输入。

## 10. 赛后复盘与归档

1. **证据归档**：原始日志、CSV、配置哈希、上游 commit、模型版本入库；大文件（GB 级环境包）只存校验和与说明。
2. **结论回填**：把实测数据回填文档并附时间戳；跨机复跑须重记校验和。
3. **口径自查**：对照第 9 节口径检查最终报告——每条结论是否与输入源匹配、是否有夸大。
4. **贡献认定**：算法改进（同考卷前后对照）、基建改进（自动化/稳定性）分开记录，不互相冒充。
5. **遗留问题**：未过验收的门禁、已知短板、下赛季计划如实列出。

## 11. 常见故障

| 现象 | 优先检查 | 修复方向 |
|---|---|---|
| WSL 中 `wsl --version` 报错 | WSL 版本过旧 | `wsl --update` 后重试 |
| WSL 中 systemd 服务不可用 | `/etc/wsl.conf` 是否含 `[boot] systemd=true` | 写入后 PowerShell 执行 `wsl --shutdown` 重启 |
| apt 装 ROS 时提示移除系统包 | 是否跳过了系统升级 | 先 `sudo apt update && sudo apt upgrade` 再装 |
| pixi install 报 no candidates | `pixi.toml` 的 platforms 是否含本机平台 | 按文档添加平台后重装 |
| 地图有、车不可见 | 初始位姿是否已发、TF、RViz RobotModel Enabled | 先发 `/initialpose` 再查变换；勾选 RobotModel |
| 二次启动 map_server 卡 unconfigured | 上次启动的孤儿节点 | 按环境路径精确清场后重启 |
| 激光无障碍或订阅不到 | 地图获取与 `/scan` QoS | 检查地图是否就绪、QoS 是否匹配（BEST_EFFORT） |
| 接受目标但车不移动 | 导航状态、控制输出、位移 | 区分接受、执行、到达与超时 |
| C++ 链接报字符串符号错误 | 依赖库的 ABI 与编译标志 | 按 4.1/第 5 节 ABI 三步定位，统一 ABI，不覆盖系统库 |
| 渲染图像有但检测不稳定 | 模型、内参、坐标、光照和目标域差 | 单变量对照，检查识别而非注入真值 |
| 云台指令存在但不跟随 | 消息类型、单位、控制模式、姿态反馈 | 核对实际接口及执行反馈 |

## 12. 通用调试方法论与记录模板

**排障通用顺序**（适用于任何一条链）：

1. 先定位层：报错的第一行（不是最后一行）；它是编译期、启动期还是运行期。
2. 再隔离层：最小复现——能不能只用官方 demo（无你的代码）跑通？能，则问题在你的集成；不能，则问题在环境/依赖。
3. 再证据层：`ros2 topic hz`（频率）、`ros2 topic echo`（内容）、`tf2_echo`（坐标）、日志文件（时间戳）。用数据说话，不用“感觉”。
4. 最后才改代码层：改前记录版本，改后同条件复测；连续三次“小修”无效就停下重新定位，不要在错误方向上堆补丁。

**求助前自查清单**（给别人看之前自己先答）：用的什么系统/ROS 版本、上游哪个 commit、完整报错的第一行、你已试过什么、官方 demo 能否复现。

**记录模板**：每次实验至少记录：日期、硬件和系统、源码 commit、模型与配置哈希、图像/角点输入来源、场景、命令、动作结果、原始日志、通过与失败项目、尚未验证的部分。公开记录前先检查个人信息、内部地址、素材权属和第三方许可。
