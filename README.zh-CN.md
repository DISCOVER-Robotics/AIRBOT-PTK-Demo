# AIRBOT PTK 双臂 Demo

[English](README.md) | [简体中文](README.zh-CN.md)

本仓库整理 DISCOVER Robotics 文档中 AIRBOT Play PTK 的双臂数据采集、策略训练与本地推理流程。[叠衣服 Demo](https://github.com/DISCOVER-Robotics/AIRBOT-Play-PTK-Cloth-Folding-Demo) 是独立的任务仓库，不与本仓库混用。

## 资料入口

| 资料 | 链接 |
| --- | --- |
| 配套 OpenPI 和数据采集包 | [官网下载](https://docs.discover-robotics.com/document/assets/airbot-play/model-reproduction/pi0.5.zip) |
| PTK 采集配置 | [configs/config_PTK.yaml](configs/config_PTK.yaml) |
| 配套包原始训练配置 | [configs/config_ptk_example.py](configs/config_ptk_example.py) |
| 模型复现教程 | [官方文档](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/model-reproduction/pi0.5.html) |
| 数据采集教程 | [官方文档](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/data-collection.html) |
| 硬件依赖包 | [AIRBOT Play Hardware releases](https://github.com/DISCOVER-Robotics/AIRBOT-Play-Hardware/releases) |

实现源码位于配套下载包，不在本仓库重复上传。包内包含 `openpi_v0.2.0.zip` 与 `airbot-data-5.1.6.7.zip`。请使用包含 AIRBOT 集成的配套代码，不要假定上游 OpenPI 具有相同脚本。

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
| 硬件 wheel | 从发布页选择匹配 Python 和平台的包；本仓库不虚构具体版本 |

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

## 2. 配置数据采集

结合官方采集流程使用 [config_PTK.yaml](configs/config_PTK.yaml)。按实际接线修改四个 CAN 接口和三个相机设备。每次采集前设置新的数据集 ID 和真实任务描述。

此 YAML 对应当前文档的 LeRobot Play 采集流程；配套包原始 OpenPI 训练配置使用 MCAP 话题。两种数据格式不能直接混用，应使用与代码包和数据集匹配的采集、转换流程。

## 3. 准备任务配置和模型

在 OpenPI 根目录创建 `data/ptk_example/config.py`，以本仓库提供的原始 `config_ptk_example.py` 为起点，根据真实数据修改任务名、数据目录、相机话题、模型类型和实验名。

原始包配置默认 `BASE_MODEL = "pi0"`，本仓库保留原值。新版文档也介绍 PI0.5；更换模型必须同步使用对应配置和权重，不能只修改模型目录名称。

**在已检查的本地配套包中没有找到独立的 PTK 任务模型权重。** 本仓库不包含权重。推理前需要取得匹配的已训练模型及训练配置。文档中的 `gs://openpi-assets/checkpoints/pi05_base/params` 是 PI0.5 训练基础权重，不是可直接执行 PTK 任务的策略。

如需自行训练，在配套 OpenPI 根目录执行：

```bash
uv run examples/airbot/compute_norm_stats.py --config-path data/ptk_example/config.py
XLA_PYTHON_CLIENT_MEM_FRACTION=0.9 uv run examples/airbot/airbot_train.py --config-path data/ptk_example/
```

训练需要真实数据集，本仓库不包含数据集。

## 4. 双臂推理

修改配套包内 `examples/airbot/robot_config.py` 中的 `RobotAHConfig`，核对两个机械臂接口、相机编号及初始姿态。相机顺序必须是环境、左腕、右腕。

在 OpenPI 根目录运行，将模型路径替换为匹配的任务模型：

```bash
uv run examples/airbot/airbot_inference_sync_ah.py policy-config:local-policy-config \
  --policy-config.config-path data/ptk_example \
  --policy-config.checkpoint-dir /path/to/matching/ptk/checkpoint
```

启动前核对推理脚本中的 prompt 和复位姿态是否匹配训练任务。此命令是流程参考，本次没有完成真机部署验证。

## 来源说明

采集 YAML 来自 `docs/assets/airbot-play/config_PTK.yaml`；训练配置原样取自已下载的 `openpi_v0.2.0.zip`。版本和许可说明见 [SOURCE.md](SOURCE.md)。不上传内部 Git 历史、无关产品资料、模型权重或数据集。

