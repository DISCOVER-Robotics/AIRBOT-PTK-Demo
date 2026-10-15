# AIRBOT PTK 双臂 Demo

[English](README.md) | [简体中文](README.zh-CN.md)

本 Demo 展示 AIRBOT Play PTK 双臂数据采集、策略训练与本地推理流程。[叠衣服 Demo](https://github.com/DISCOVER-Robotics/AIRBOT-Play-PTK-Cloth-Folding-Demo) 是独立的任务仓库，不与本仓库混用。

## 真机复现前提

完整流程参考[模型复现教程](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/model-reproduction/pi0.5.html)，其中包含环境配置、微调训练、checkpoint 结构、同步推理及常见问题。

| 使用方式 | 操作顺序 |
| --- | --- |
| 已有训练好的 PTK 模型 | 按第 1、3、4 节准备环境、加载模型并执行真机推理，无需重新采集和训练 |
| 自行训练 PTK 模型 | 按第 1、2、3、4 节及官网教程完成环境安装、采集、训练和真机推理 |

运行已有模型需要完整 checkpoint、配套 `config.py` 和任务指令；本仓库未附带任务模型。自行训练时，使用教程流程生成的 checkpoint。相机和初始姿态按教程及实际模型配置核对即可。

## 资料入口

| 资料 | 链接 |
| --- | --- |
| 配套 OpenPI 和数据采集包 | [官网下载](https://docs.discover-robotics.com/document/assets/airbot-play/model-reproduction/pi0.5.zip) |
| PTK 采集配置 | [configs/config_PTK.yaml](configs/config_PTK.yaml) |
| OpenPI 训练示例配置 | [configs/config_ptk_example.py](configs/config_ptk_example.py) |
| 模型复现教程 | [官方文档](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/model-reproduction/pi0.5.html) |
| 数据采集教程 | [官方文档](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/data-collection.html) |
| 硬件依赖包 | [AIRBOT Play Hardware releases](https://github.com/DISCOVER-Robotics/AIRBOT-Play-Hardware/releases) |

下载上表中的“配套 OpenPI 和数据采集包”，解压后可获得 `openpi_v0.2.0.zip` 和 `airbot-data-5.1.6.7.zip`。以下训练、推理和数据采集步骤均使用这两个包中的代码。

## 硬件和软件

| 项目 | 基线 |
| --- | --- |
| 执行臂 | 2 台 AIRBOT Play |
| 采集时使用的示教臂 | 2 台 AIRBOT Replay |
| 相机 | 环境、左腕、右腕各一路 |
| 本地推理系统 | Ubuntu 24.04 |
| OpenPI 包 | openpi_v0.2.0 |
| 数据采集包 | airbot-data-5.1.6.7 |
| 设备配置包 | airbot-configure 5.1.6-1 |
| 硬件 wheel | 从发布页选择匹配 Python 和平台的包 |

采集与推理依赖不同的情况下，应使用独立环境。官方教程建议在 Ubuntu 24.04 上使用 Python 3.12 安装硬件驱动包。

## 安全注意事项

初始化、复位和推理均可能使真实机械臂运动。启动前确认 CAN 接口、相机顺序、关节限制和初始姿态，清空工作区并确保急停可用。不要将基础模型或其他任务的模型直接用于真实硬件。

## 1. 准备配套软件

下载并解压官网包，再解压其中两个子压缩包。进入 OpenPI 目录执行：

```bash
sudo apt install python3-venv clang ffmpeg libsvtav1-dev libturbojpeg gcc python3-dev v4l-utils
python3 -m pip install --user pipx
pipx install uv
GIT_LFS_SKIP_SMUDGE=1 uv sync
GIT_LFS_SKIP_SMUDGE=1 uv pip install -e .
```

将下列路径替换为真实路径，安装数据采集项目和匹配的硬件 wheel：

```bash
uv pip install -e "/path/to/airbot-data-5.1.6.7/data-collection[all]"
uv pip install /path/to/airbot_hardware_py-MATCHING_VERSION.whl
```

按官方硬件教程安装 `airbot-configure_5.1.6-1_all.deb`。依赖排查参考模型教程，包括 `tyro==0.9.22`、`linuxpy` 与 `pyturbojpeg==1.8.2`。

安装完成后，在 OpenPI 根目录检查关键导入，不连接机械臂：

```bash
.venv/bin/python -c 'from airbot_arm import AIRBOTArm; from airbot_data_collection.airbot.sensors.cameras.v4l2 import BsonV4L2Camera; import jax, tyro; print(jax.devices())'
```

必须能导入以上模块并识别到 GPU。仅安装名称相近的 wheel 不代表兼容；使用能提供 `airbot_arm.AIRBOTArm` 的匹配硬件包。后续使用 `uv run --no-sync` 保留手动安装的硬件依赖；需要更新依赖时显式执行安装并重新检查。

检查 FFmpeg 是否提供 SVT-AV1 编码器：

```bash
ffmpeg -encoders | grep svtav1
```

![FFmpeg 编码器检查结果](assets/ffmpeg-verification.png)

## 2. 配置数据采集

配置机械臂前，先检查 CAN 接口是否被识别。下图接口名仅为示例，请以实际设备对应关系为准。

```bash
ip link show
```

![CAN 接口识别结果](assets/can-interfaces.jpg)

结合官方采集流程使用 [config_PTK.yaml](configs/config_PTK.yaml)。按实际接线修改四个 CAN 接口和三个相机设备。每次采集前设置新的数据集 ID 和真实任务描述。

采集 YAML 使用 LeRobot Play 流程，OpenPI 训练配置使用 MCAP 话题。两种数据格式不能直接混用，应使用与代码包和数据集匹配的采集、转换流程。

## 3. 准备任务配置和模型

已有模型时，使用随模型提供的配置，直接进入下面的模型目录检查。自行训练时，在 OpenPI 根目录创建 `data/ptk_example/config.py`，根据数据格式选择配置：PI0.5 与 LeRobot 数据使用[模型复现教程](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/model-reproduction/pi0.5.html)“配置训练参数”中的配置；[config_ptk_example.py](configs/config_ptk_example.py) 是配套包中的 PI0 / MCAP 示例，不要直接套用到 PI0.5 / LeRobot 数据。

示例配置默认 `BASE_MODEL = "pi0"`。如使用 PI0.5，必须同步使用对应配置和权重，不能只修改模型目录名称。

**本仓库不包含 PTK 任务模型权重。** 推理前需要取得匹配的已训练模型及训练配置。`gs://openpi-assets/checkpoints/pi05_base/params` 是 PI0.5 训练基础权重，不是可直接执行 PTK 任务的策略。

已有模型时，应直接使用随模型交付的 `config.py`，不要用示例配置覆盖它。将其放到 `data/ptk_example/config.py`。`TASK_NAME`、`BASE_MODEL`、`USING_LORA`、相机映射和动作定义必须与训练一致。JAX checkpoint 应包含 `params/` 和 `assets/<TASK_NAME>/norm_stats.json`；推理路径指向包含这两者的具体训练步数目录，而非 `params/` 子目录或整个实验目录。该版本通过 `TASK_NAME` 定位归一化统计量。

自行训练时，先按[数据采集教程](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/data-collection.html)生成与训练配置匹配的数据集，再按模型复现教程完成“计算数据统计学信息”和“模型训练”。在配套 OpenPI 根目录执行：

```bash
uv run --no-sync examples/airbot/compute_norm_stats.py --config-path data/ptk_example/config.py
XLA_PYTHON_CLIENT_MEM_FRACTION=0.9 uv run --no-sync examples/airbot/airbot_train.py --config-path data/ptk_example/
```

训练需要真实数据集，本仓库不包含数据集。

模型目录示例如下。推理使用自行训练生成或已取得的完整 checkpoint。

![模型目录示例](assets/checkpoint-layout.png)

## 4. 双臂推理

修改配套包内 `examples/airbot/robot_config.py` 中的 `RobotAHConfig`，核对两个机械臂接口、相机编号及初始姿态。相机顺序必须是环境、左腕、右腕。

```bash
ip -details link show
v4l2-ctl --list-devices
```

确认两条执行臂 CAN 接口均已配置并启动，逐一预览相机以确认环境、左腕和右腕的对应关系。示例 `/dev/video2`、`/dev/video4`、`/dev/video6` 不是固定设备号；三个相机名称与编号必须一一对应。

| 配置项 | 要求 |
| --- | --- |
| `robot_groups` | `["left", "right"]`，左臂在前 |
| `robot_interface` | 两个执行臂实际 CAN 接口，顺序与机械臂一致 |
| `camera_names` | `["base_0_rgb", "left_wrist_0_rgb", "right_wrist_0_rgb"]` |
| `camera_index` | 与上述名称一一对应的三个实际视频设备编号 |
| `InferConfig.reset_action` | 与安装布局和任务匹配的 14 维复位动作，左臂 6 关节及夹爪、右臂 6 关节及夹爪 |
| `InferConfig.prompt` | 与模型训练任务一致的英文指令 |

实际复位动作在 `airbot_inference_sync_ah.py` 的 `InferConfig.reset_action` 中；只改 `RobotAHConfig.init_pos` 不会修改这里发送的复位动作。默认指令为 `place the wooden block into the bowl`，不代表任意模型都能执行该任务。

这一入口通过 `AIRBOTArm(interface=...)` 直接访问 CAN，不通过 gRPC。不要在相同接口上同时运行采集程序或其他机械臂控制服务。

在 OpenPI 根目录运行，将模型路径替换为匹配的任务模型：

```bash
uv run --no-sync examples/airbot/airbot_inference_sync_ah.py policy-config:local-policy-config \
  --policy-config.config-path data/ptk_example \
  --policy-config.checkpoint-dir /path/to/matching/ptk/checkpoint \
  --robot-config.robot-type play
```

启动前核对推理脚本中的 prompt 和复位姿态是否匹配训练任务。真机运行前，请在安全环境中验证配置和运动范围。

上述命令显式选择 `play`，并从 `RobotAHConfig` 读取其余硬件设置。省略该选项时，Tyro 也能根据字段默认值构建配置，但仍需确认默认值符合实际硬件。运行前先用相同命令追加 `--help` 检查参数；帮助检查不代表模型或硬件验证。

加载策略后，程序会先发送复位动作，再提示 `Press 'Enter' to continue or 'q' and 'Enter' to quit...`，因此不能等提示出现后才清空工作区。确认复位完成并准备好对应任务后按 Enter 开始推理。默认每轮最多 2000 步、发送频率 20 Hz，每次执行 35 个预测动作；应核对模型预测序列长度不少于 35。每轮结束会再次复位并等待输入。在等待提示时输入 `q` 并回车退出，正常退出会断开机械臂和相机。运动异常时使用急停；运行中不要把 `q` 当成实时停止键。

## 5. 复现验收与故障定位

| 检查 | 通过标准 / 处理方式 |
| --- | --- |
| 环境 | 关键模块导入成功，JAX 能看到 GPU |
| 模型 | 权重加载和归一化统计量读取成功；缺失统计量时补齐同次训练的 `assets`，不要用其他任务替代 |
| 设备 | 两臂连接成功，三路相机与训练视角对应；`Failed to configure` 时检查 CAN、视频设备权限与占用 |
| 复位 | 两臂到达经过确认的任务起始姿态，没有干涉 |
| 执行 | 日志出现 `Start inference...`，并完成该模型定义的目标任务 |
| 退出 | 等待提示时输入 `q`，连接正常释放 |

仅看到推理日志或机械臂运动不算任务复现成功；验收需记录模型版本、硬件包版本、相机布局、任务成功情况及退出结果。
