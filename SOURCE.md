# Source and Version Notes

- Documentation checkout: DISCOVER Robotics documentation, inspected 2026-10-08.
- Collection configuration: `docs/assets/airbot-play/config_PTK.yaml`.
- Training configuration: `openpi_v0.2.0/examples/airbot/config_ptk_example.py` from the local documentation package `docs/assets/airbot-play/model-reproduction/pi0.5.zip`.
- Companion archives: `openpi_v0.2.0.zip`, `airbot-data-5.1.6.7.zip`.
- The OpenPI archive includes an Apache-2.0 license, retained as `LICENSE` for the copied training example. This does not grant rights to separately distributed hardware packages, datasets, or model weights; consult their own terms.
- No nested Git metadata or internal Git history is redistributed here.
- The original training example defaults to PI0 and MCAP topics. The newer documentation describes PI0.5 and LeRobot data as well; use the appropriate configuration rather than mixing the two baselines.
- No task-specific PTK weights were present in the inspected package. The cloth-folding policy belongs to the separate cloth-folding workflow and is not claimed as this demo's model.
