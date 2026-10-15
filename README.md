# AIRBOT PTK Dual-Arm Demo

[English](README.md) | [简体中文](README.zh-CN.md)

Dual-arm data collection, policy training, and local inference with AIRBOT Play PTK. The task-specific [cloth-folding demo](https://github.com/DISCOVER-Robotics/AIRBOT-Play-PTK-Cloth-Folding-Demo) is maintained separately.

## Physical Robot Reproduction Requirements

Follow the [model reproduction guide](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/model-reproduction/pi0.5.html) for environment setup, fine-tuning, checkpoint structure, synchronous inference, and troubleshooting.

| Workflow | Steps |
| --- | --- |
| Run an existing trained PTK policy | Follow sections 1, 3, and 4 to prepare the environment, load the policy, and run on hardware; collection and retraining are unnecessary |
| Train a PTK policy | Follow sections 1, 2, 3, and 4 together with the official guides for setup, collection, training, and hardware inference |

An existing policy requires its complete checkpoint, matching `config.py`, and task instruction; task weights are not bundled in this repository. When training your own policy, use the checkpoint produced by the tutorial workflow. Check camera settings and the initial pose against the guide and your policy configuration.

The instructions below have been checked against the companion package code. GPU model loading and dual-arm hardware execution of this repository's workflow have not yet been validated.

## Resources

| Resource | Location |
| --- | --- |
| Compatible OpenPI and data-collection packages | [Official package download](https://docs.discover-robotics.com/document/assets/airbot-play/model-reproduction/pi0.5.zip) |
| PTK collection configuration | [configs/config_PTK.yaml](configs/config_PTK.yaml) |
| OpenPI training example configuration | [configs/config_ptk_example.py](configs/config_ptk_example.py) |
| Model reproduction guide | [Official documentation](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/model-reproduction/pi0.5.html) |
| Data collection guide | [Official documentation](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/data-collection.html) |
| Hardware packages | [AIRBOT Play Hardware releases](https://github.com/DISCOVER-Robotics/AIRBOT-Play-Hardware/releases) |

Download the OpenPI and data collection package linked in the table above and extract it to obtain `openpi_v0.2.0.zip` and `airbot-data-5.1.6.7.zip`. The training, inference, and data collection steps below use the code in these two packages.

## Hardware and Software

| Component | Baseline |
| --- | --- |
| Follower arms | 2 AIRBOT Play arms |
| Teaching arms, for collection | 2 AIRBOT Replay arms |
| Cameras | Environment, left wrist, and right wrist |
| Local inference OS | Ubuntu 24.04 |
| OpenPI archive | openpi_v0.2.0 |
| Data-collection archive | airbot-data-5.1.6.7 |
| Device configuration package | airbot-configure 5.1.6-1 |
| Hardware wheel | Select the matching Python/platform wheel from the hardware releases |

Keep collection and inference environments separate when their dependency requirements differ. The official guide recommends Python 3.12 for hardware-driver installation on Ubuntu 24.04.

## Safety

These workflows can move physical robots, including during initialization and reset. Verify CAN interfaces, camera order, joint limits, and initial poses before launching. Keep the workspace clear and the emergency stop accessible. Never run a base model or a checkpoint trained for another task directly on hardware.

## 1. Prepare the Compatible Packages

Download and extract the official package, then extract its two nested archives. Enter the extracted OpenPI directory. Run installation commands there:

```bash
sudo apt install python3-venv clang ffmpeg libsvtav1-dev libturbojpeg gcc python3-dev v4l-utils
python3 -m pip install --user pipx
pipx install uv
GIT_LFS_SKIP_SMUDGE=1 uv sync
GIT_LFS_SKIP_SMUDGE=1 uv pip install -e .
```

Install the extracted data-collection project and your matching hardware wheel, replacing the paths below with actual local paths:

```bash
uv pip install -e "/path/to/airbot-data-5.1.6.7/data-collection[all]"
uv pip install /path/to/airbot_hardware_py-MATCHING_VERSION.whl
```

Install `airbot-configure_5.1.6-1_all.deb` according to the official hardware guide. Refer to the model reproduction guide for dependency troubleshooting, including `tyro==0.9.22`, `linuxpy`, and `pyturbojpeg==1.8.2`.

Check the required imports from the OpenPI root without connecting the robot:

```bash
.venv/bin/python -c 'from airbot_arm import AIRBOTArm; from airbot_data_collection.airbot.sensors.cameras.v4l2 import BsonV4L2Camera; import jax, tyro; print(jax.devices())'
```

All imports must succeed and JAX must detect a GPU. Use a compatible hardware package that provides `airbot_arm.AIRBOTArm`; a similarly named wheel is not sufficient. Use `uv run --no-sync` afterward to preserve manually installed hardware dependencies. Install dependency updates explicitly and repeat the check.

Verify that FFmpeg provides the SVT-AV1 encoder:

```bash
ffmpeg -encoders | grep svtav1
```

![FFmpeg encoder verification](assets/ffmpeg-verification.png)

## 2. Configure Data Collection

Check that the CAN interfaces are recognized before configuring the arms. The interface names below are examples; use your actual device assignments.

```bash
ip link show
```

![Recognized CAN interfaces](assets/can-interfaces.jpg)

Use [config_PTK.yaml](configs/config_PTK.yaml) with the collection workflow documented above. Edit all four CAN interfaces and the three camera devices for your hardware. Set a new dataset ID and the actual task description before every collection run.

The collection YAML uses the LeRobot Play workflow. The OpenPI training configuration uses MCAP topics; these formats are not interchangeable. Use a collection and conversion workflow that matches your package and dataset.

## 3. Prepare the Task Configuration and Model

For an existing policy, use its delivered configuration and proceed to the checkpoint checks below. For training, create `data/ptk_example/config.py` from the OpenPI root and select the configuration for your data format: for PI0.5 with LeRobot data, use the training configuration in the [model reproduction guide](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/model-reproduction/pi0.5.html). The bundled [config_ptk_example.py](configs/config_ptk_example.py) is a PI0 / MCAP example; do not apply it unchanged to PI0.5 / LeRobot data.

The example defaults to `BASE_MODEL = "pi0"`. PI0.5 requires the corresponding configuration and weights; do not switch models merely by renaming a checkpoint directory.

**This repository does not include task-specific PTK model weights.** Obtain a matching trained checkpoint and its training configuration before inference. The PI0.5 base weights at `gs://openpi-assets/checkpoints/pi05_base/params` are a training starting point, not a ready-to-run PTK task policy.

For an existing policy, place the delivered training configuration at `data/ptk_example/config.py`; do not overwrite it with the example. Keep `TASK_NAME`, `BASE_MODEL`, `USING_LORA`, camera mappings, and action definitions consistent with training. A JAX checkpoint must include `params/` and `assets/<TASK_NAME>/norm_stats.json`. Point inference to the specific training-step directory containing both, rather than its `params/` subdirectory or the entire experiment directory. This version uses `TASK_NAME` to locate normalization statistics.

For training, first produce a dataset matching the training configuration using the [data collection guide](https://docs.discover-robotics.com/document/airbot-play/hardware-driver/tutorials/data-collection.html). Follow the normalization-statistics and model-training sections of the model reproduction guide. Run from the compatible OpenPI root:

```bash
uv run --no-sync examples/airbot/compute_norm_stats.py --config-path data/ptk_example/config.py
XLA_PYTHON_CLIENT_MEM_FRACTION=0.9 uv run --no-sync examples/airbot/airbot_train.py --config-path data/ptk_example/
```

Training requires a real dataset; this repository does not include one.

Example checkpoint directory. Use a complete checkpoint from your training run or an existing policy for inference.

![Example checkpoint directory](assets/checkpoint-layout.png)

## 4. Run Dual-Arm Inference

Edit `examples/airbot/robot_config.py` in the extracted package. For its `RobotAHConfig`, check both arm interfaces, camera indices, and initial positions. Camera order must be environment, left wrist, right wrist.

```bash
ip -details link show
v4l2-ctl --list-devices
```

Both follower CAN interfaces must be configured and up. Preview each camera to identify the environment, left wrist, and right wrist views. Example devices `/dev/video2`, `/dev/video4`, and `/dev/video6` are not fixed assignments; camera names and indices must match one to one.

| Setting | Requirement |
| --- | --- |
| `robot_groups` | `["left", "right"]`, left arm first |
| `robot_interface` | Actual follower CAN interfaces in the same arm order |
| `camera_names` | `["base_0_rgb", "left_wrist_0_rgb", "right_wrist_0_rgb"]` |
| `camera_index` | Three actual video indices in the same camera order |
| `InferConfig.reset_action` | A verified 14-element reset action for this installation and task: left arm's six joints and gripper, then right arm's six joints and gripper |
| `InferConfig.prompt` | The English instruction matching policy training |

The reset action actually sent is `InferConfig.reset_action` in `airbot_inference_sync_ah.py`. Changing only `RobotAHConfig.init_pos` does not change that action. The default prompt is `place the wooden block into the bowl`; it does not establish what an arbitrary checkpoint can do.

This entry point accesses CAN directly through `AIRBOTArm(interface=...)`, without gRPC. Do not run collection programs or another arm control service on the same interfaces concurrently.

Run from the OpenPI root, replacing the checkpoint path with the matching trained checkpoint:

```bash
uv run --no-sync examples/airbot/airbot_inference_sync_ah.py policy-config:local-policy-config \
  --policy-config.config-path data/ptk_example \
  --policy-config.checkpoint-dir /path/to/matching/ptk/checkpoint \
  --robot-config.robot-type play
```

Check the inference script's prompt and reset pose against your training task before starting. Validate the configuration and motion range in a safe environment before operating on hardware.

The command explicitly selects `play` and reads the remaining hardware settings from `RobotAHConfig`. Tyro can also construct this configuration from its field defaults when the option is omitted; those defaults still need to match your hardware. Append `--help` to the same command to check argument parsing before execution; this does not validate the model or hardware.

After loading the policy, the program sends the reset action **before** displaying `Press 'Enter' to continue or 'q' and 'Enter' to quit...`. Clear the workspace before launch. Once reset is complete and the task is ready, press Enter to start inference. Defaults are 2000 steps per episode, a 20 Hz action rate, and 35 executed actions per prediction; verify that the model predicts at least 35 actions. Each episode is followed by another reset and input prompt. At that prompt, enter `q` to disconnect the arms and cameras normally. Use the emergency stop for abnormal motion; `q` is not a live stop key during execution.

## 5. Acceptance and Troubleshooting

| Check | Pass condition / corrective action |
| --- | --- |
| Environment | Required imports succeed and JAX detects a GPU |
| Model | Weights and normalization statistics load; restore missing `assets` from the same training run, never another task |
| Devices | Both arms connect and all three views match training; for `Failed to configure`, check CAN, video permissions, and competing processes |
| Reset | Both arms reach the verified task starting pose without interference |
| Execution | Logs show `Start inference...` and the policy completes its defined task |
| Exit | Entering `q` at the input prompt releases connections |

Inference logs or arm motion alone do not establish successful reproduction. Record the checkpoint version, hardware package version, camera layout, task outcome, and shutdown result.
