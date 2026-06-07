# Organic Waste Sorting with LeRobot

Robotic organic/non-organic waste sorting using a real SO-101 leader/follower arm setup, Hugging Face LeRobot, imitation learning, ACT, Diffusion Policy, and SmolVLA experiments.

The goal of this project is to teach a robot arm to sort objects into the correct waste bin:

- **Chocolate bar** → **blue non-organic bin**
- **Branch** → **red organic bin**

This repository documents the full process from hardware setup to policy rollout:

```text
setup → calibration → teleoperation → camera integration → data collection → Hugging Face dataset upload → ACT training → Diffusion Policy training → real robot rollout
```

---

## Final Demonstration

### Diffusion Policy: Branch → Red Organic Bin

The branch policy was trained with Diffusion Policy using wrist-camera observations. It produced smoother behavior than ACT and performed better during physical rollout.

<video src="media/diffusion_branch_rollout.mp4" controls width="720"></video>

[Watch branch diffusion rollout](media/diffusion_branch_rollout.mp4)

---

### Diffusion Policy: Chocolate Bar → Blue Non-Organic Bin

The chocolate policy was trained with Diffusion Policy using wrist-camera observations.

<video src="media/diffusion_chocolate_rollout.mp4" controls width="720"></video>

[Watch chocolate diffusion rollout](media/diffusion_chocolate_rollout.mp4)

---

## Dataset Demonstration: Episode 30

Episode 30 shows the branch sorting task:

> Place branch on the red organic bin.

The episode was recorded in LeRobot format with synchronized camera observations, robot states, actions, and task labels.

### Front Camera

![Episode 30 front camera](media/episode30_front.gif)

### Wrist Camera

![Episode 30 wrist camera](media/episode30_wrist.gif)

### 3D View

![Episode 30 3D view](media/episode30_3d_view.gif)

### Joint State / Action Graphs

![Episode 30 joint graph](media/episode30_graph.gif)

---

## Project Summary

| Component | Status |
|---|---|
| SO-101 follower setup | Complete |
| SO-101 leader setup | Complete |
| Follower calibration | Complete |
| Leader calibration | Complete |
| Teleoperation | Working |
| Teleoperation with cameras | Working |
| Dataset recording | Complete |
| Hugging Face dataset upload | Complete |
| ACT baseline training | Complete |
| ACT rollout | Working, but grasp unreliable |
| Diffusion Policy training | Complete |
| Diffusion rollout | Best-performing policy |
| SmolVLA | Experimental |
| YOLO-E object selector | Planned / optional |

---

## Hardware Setup

The project used:

- SO-101 follower robotic arm
- SO-101 leader teleoperation arm
- Front USB camera
- Wrist USB camera
- Ubuntu/Linux workstation
- NVIDIA GPU for training
- Hugging Face LeRobot

Camera setup:

```text
front camera:
  device: /dev/video0
  resolution: 1920x1080
  fps: 5

wrist camera:
  device: /dev/video2 or /dev/video3 depending on Linux camera mapping
  resolution: 640x480
  fps during recording: 30
  fps during rollout: 15
```

Camera device numbers changed between runs, so we used:

```bash
lerobot-find-cameras opencv
```

and also checked stable Linux camera paths with:

```bash
v4l2-ctl --list-devices
ls -l /dev/v4l/by-id/
```

Using `/dev/v4l/by-id/...` is more stable than relying only on `/dev/video2` or `/dev/video3`.

---

## Environment Setup

Clone LeRobot:

```bash
git clone https://github.com/huggingface/lerobot.git
cd lerobot
```

Install LeRobot:

```bash
pip install -e .
```

For SmolVLA experiments:

```bash
pip install -e ".[smolvla]"
```

Login to Hugging Face:

```bash
hf auth login
```

Set the project variables:

```bash
export HF_USER=Hugo132645
export DATASET_REPO_ID=Hugo132645/OW_Dataset_20260606_230614
export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True
```

---

## Robot Port Discovery

We used:

```bash
lerobot-find-port
```

Typical ports:

```text
follower: /dev/ttyACM0 or /dev/ttyACM1
leader:   /dev/ttyACM0 or /dev/ttyACM1
```

The exact port changed depending on USB order.

---

## Follower Calibration

The follower arm was calibrated with:

```bash
lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=FOLLOWER \
  --robot.calibration_dir=configs/calibration/robots
```

If the follower appeared on the other port:

```bash
--robot.port=/dev/ttyACM1
```

The calibration was saved in:

```text
configs/calibration/robots
```

---

## Leader Calibration

The leader arm was calibrated with:

```bash
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=LEADER \
  --teleop.calibration_dir=configs/calibration/teleoperators
```

If the leader appeared on the other port:

```bash
--teleop.port=/dev/ttyACM0
```

The calibration was saved in:

```text
configs/calibration/teleoperators
```

---

## Basic Teleoperation

After calibration, we tested leader/follower teleoperation without cameras:

```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=FOLLOWER \
  --robot.calibration_dir=configs/calibration/robots \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=LEADER \
  --teleop.calibration_dir=configs/calibration/teleoperators \
  --display_data=false
```

This confirmed that leader-arm motion controlled the follower arm correctly.

---

## Camera Discovery and Testing

We detected cameras with:

```bash
lerobot-find-cameras opencv
```

We also tested cameras manually using OpenCV:

```python
import cv2

cam = "/dev/video3"

cap = cv2.VideoCapture(cam, cv2.CAP_V4L2)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
cap.set(cv2.CAP_PROP_FPS, 30)

print("opened:", cap.isOpened())

ret, frame = cap.read()
print("read:", ret)

if ret:
    print("frame shape:", frame.shape)

cap.release()
```

This helped confirm which `/dev/videoX` path corresponded to the usable camera stream.

---

## Teleoperation with Cameras

We then tested teleoperation with both the front and wrist cameras:

```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=FOLLOWER \
  --robot.calibration_dir=configs/calibration/robots \
  --robot.cameras='{
    front: {type: opencv, index_or_path: /dev/video0, width: 1920, height: 1080, fps: 5, warmup_s: 3},
    wrist: {type: opencv, index_or_path: /dev/video3, width: 640, height: 480, fps: 30, warmup_s: 3}
  }' \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=LEADER \
  --teleop.calibration_dir=configs/calibration/teleoperators \
  --display_data=true
```

This confirmed that both cameras could be used during robot teleoperation.

---

## Dataset Recording

The dataset was recorded with real teleoperated demonstrations.

Each episode contains:

- robot actions
- robot joint states
- front camera video
- wrist camera video
- task language instruction
- episode metadata

Final Hugging Face dataset:

```text
Hugo132645/OW_Dataset_20260606_230614
```

### Task 1: Chocolate Bar → Blue Non-Organic Bin

```bash
python src/lerobot/scripts/lerobot_record.py \
  --resume=true \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=FOLLOWER \
  --robot.calibration_dir=configs/calibration/robots \
  --robot.cameras='{
    front: {type: opencv, index_or_path: /dev/video0, width: 1920, height: 1080, fps: 5, warmup_s: 3},
    wrist: {type: opencv, index_or_path: /dev/video3, width: 640, height: 480, fps: 30, warmup_s: 3}
  }' \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=LEADER \
  --teleop.calibration_dir=configs/calibration/teleoperators \
  --dataset.repo_id=Hugo132645/OW_Dataset_20260606_230614 \
  --dataset.root=data \
  --dataset.num_episodes=30 \
  --dataset.single_task="Place chocolate bar on the blue non-organic bin." \
  --dataset.push_to_hub=true \
  --display_data=false
```

### Task 2: Branch → Red Organic Bin

```bash
python src/lerobot/scripts/lerobot_record.py \
  --resume=true \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=FOLLOWER \
  --robot.calibration_dir=configs/calibration/robots \
  --robot.cameras='{
    front: {type: opencv, index_or_path: /dev/video0, width: 1920, height: 1080, fps: 5, warmup_s: 3},
    wrist: {type: opencv, index_or_path: /dev/video3, width: 640, height: 480, fps: 30, warmup_s: 3}
  }' \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=LEADER \
  --teleop.calibration_dir=configs/calibration/teleoperators \
  --dataset.repo_id=Hugo132645/OW_Dataset_20260606_230614 \
  --dataset.root=data \
  --dataset.num_episodes=30 \
  --dataset.single_task="Place branch on the red organic bin." \
  --dataset.push_to_hub=true \
  --display_data=false
```

---

## Dataset Verification

We verified the dataset with:

```bash
python - <<'PY'
from lerobot.datasets.lerobot_dataset import LeRobotDataset
from collections import Counter

repo_id = "Hugo132645/OW_Dataset_20260606_230614"
ds = LeRobotDataset(repo_id)

print("Dataset:", repo_id)
print("Frames:", ds.num_frames)
print("Episodes:", ds.num_episodes)

print("\nTasks:")
print(ds.meta.tasks)

counts = Counter()
for ep in ds.meta.episodes:
    tasks = ep["tasks"]
    if isinstance(tasks, list):
        for task in tasks:
            counts[task] += 1
    else:
        counts[str(tasks)] += 1

print("\nTask counts:")
for task, count in counts.items():
    print(count, "episodes:", task)
PY
```

Final dataset statistics:

```text
Frames: 17,530
Episodes: 60

30 episodes: Place chocolate bar on the blue non-organic bin.
30 episodes: Place branch on the red organic bin.
```

Dataset features:

```text
action: 6 robot joints
observation.state: 6 robot joints
observation.images.front: 1920x1080 RGB video
observation.images.wrist: 640x480 RGB video
timestamp
frame_index
episode_index
task_index
```

---

## Hugging Face Dataset Visualization

The dataset was uploaded to Hugging Face:

```text
Hugo132645/OW_Dataset_20260606_230614
```

It can be visualized with the LeRobot dataset visualizer:

```text
https://huggingface.co/spaces/lerobot/visualize_dataset
```

Local dataset visualization:

```bash
lerobot-dataset-viz \
  --dataset.repo_id=Hugo132645/OW_Dataset_20260606_230614
```

---

## ACT Baseline

ACT was trained first as a baseline imitation learning policy.

```bash
lerobot-train \
  --dataset.repo_id=${DATASET_REPO_ID} \
  --policy.type=act \
  --output_dir=outputs/train/act_food_sorting_baseline_20k \
  --job_name=act_food_sorting_baseline_20k \
  --policy.device=cuda \
  --wandb.enable=false \
  --steps=20000 \
  --batch_size=1 \
  --num_workers=0 \
  --log_freq=200 \
  --save_checkpoint=true \
  --save_freq=5000 \
  --policy.repo_id=${HF_USER}/act_food_sorting_baseline_20k \
  --policy.push_to_hub=false \
  --policy.use_amp=true \
  --policy.use_vae=true \
  --policy.chunk_size=50 \
  --policy.n_action_steps=50
```

Final ACT checkpoint:

```text
outputs/train/act_food_sorting_baseline_20k/checkpoints/last/pretrained_model
```

### ACT Rollout

```bash
lerobot-rollout \
  --policy.path=outputs/train/act_food_sorting_baseline_20k/checkpoints/last/pretrained_model \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=FOLLOWER \
  --robot.calibration_dir=configs/calibration/robots \
  --robot.cameras='{
    front: {type: opencv, index_or_path: /dev/video0, width: 1920, height: 1080, fps: 5, warmup_s: 3},
    wrist: {type: opencv, index_or_path: /dev/video3, width: 640, height: 480, fps: 15, warmup_s: 3}
  }' \
  --task="Place chocolate bar on the blue non-organic bin." \
  --fps=5 \
  --display_data=false
```

ACT loaded and controlled the robot, but it was not reliable enough for the final demonstration. The arm trembled and did not consistently grasp the object.

---

## Diffusion Policy

Diffusion Policy was trained after ACT because it produced smoother action trajectories.

In our LeRobot version, Diffusion Policy required image observations to have matching shapes. The full dataset used different camera resolutions:

```text
front: 1920x1080
wrist: 640x480
```

Because of this, we trained wrist-camera-only diffusion policies.

### Chocolate Bar → Blue Bin

```bash
lerobot-train \
  --dataset.repo_id=${DATASET_REPO_ID} \
  --dataset.episodes='[0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29]' \
  --policy.type=diffusion \
  --output_dir=outputs/train/diffusion_chocolate_blue_wrist_30k \
  --job_name=diffusion_chocolate_blue_wrist_30k \
  --policy.device=cuda \
  --wandb.enable=false \
  --steps=30000 \
  --batch_size=1 \
  --num_workers=0 \
  --log_freq=200 \
  --save_checkpoint=true \
  --save_freq=5000 \
  --policy.repo_id=${HF_USER}/diffusion_chocolate_blue_wrist_30k \
  --policy.push_to_hub=false \
  --policy.use_amp=true \
  --policy.horizon=16 \
  --policy.n_action_steps=8 \
  --policy.num_inference_steps=10 \
  --policy.resize_shape='[240,320]' \
  --policy.input_features='{
    observation.state: {type: STATE, shape: [6]},
    observation.images.wrist: {type: VISUAL, shape: [3, 480, 640]}
  }' \
  --policy.output_features='{
    action: {type: ACTION, shape: [6]}
  }'
```

Rollout:

```bash
lerobot-rollout \
  --policy.path=outputs/train/diffusion_chocolate_blue_wrist_30k/checkpoints/last/pretrained_model \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=FOLLOWER \
  --robot.calibration_dir=configs/calibration/robots \
  --robot.max_relative_target=5.0 \
  --robot.cameras='{
    wrist: {type: opencv, index_or_path: /dev/video3, width: 640, height: 480, fps: 15, warmup_s: 3}
  }' \
  --task="Place chocolate bar on the blue non-organic bin." \
  --fps=5 \
  --display_data=false
```

### Branch → Red Bin

A direct training attempt with episodes `30–59` caused an indexing issue, so we created a branch-only dataset by deleting episodes `0–29`.

```bash
export BRANCH_DATASET=${HF_USER}/OW_Dataset_branch_red_only
```

```bash
lerobot-edit-dataset \
  --repo_id=${DATASET_REPO_ID} \
  --new_repo_id=${BRANCH_DATASET} \
  --operation.type=delete_episodes \
  --operation.episode_indices='[0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29]' \
  --push_to_hub=true
```

Train branch diffusion policy:

```bash
lerobot-train \
  --dataset.repo_id=${BRANCH_DATASET} \
  --policy.type=diffusion \
  --output_dir=outputs/train/diffusion_branch_red_wrist_30k \
  --job_name=diffusion_branch_red_wrist_30k \
  --policy.device=cuda \
  --wandb.enable=false \
  --steps=30000 \
  --batch_size=1 \
  --num_workers=0 \
  --log_freq=200 \
  --save_checkpoint=true \
  --save_freq=5000 \
  --policy.repo_id=${HF_USER}/diffusion_branch_red_wrist_30k \
  --policy.push_to_hub=false \
  --policy.use_amp=true \
  --policy.horizon=16 \
  --policy.n_action_steps=8 \
  --policy.num_inference_steps=10 \
  --policy.resize_shape='[240,320]' \
  --policy.input_features='{
    observation.state: {type: STATE, shape: [6]},
    observation.images.wrist: {type: VISUAL, shape: [3, 480, 640]}
  }' \
  --policy.output_features='{
    action: {type: ACTION, shape: [6]}
  }'
```

Rollout:

```bash
lerobot-rollout \
  --policy.path=outputs/train/diffusion_branch_red_wrist_30k/checkpoints/last/pretrained_model \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=FOLLOWER \
  --robot.calibration_dir=configs/calibration/robots \
  --robot.max_relative_target=5.0 \
  --robot.cameras='{
    wrist: {type: opencv, index_or_path: /dev/video3, width: 640, height: 480, fps: 15, warmup_s: 3}
  }' \
  --task="Place branch on the red organic bin." \
  --fps=5 \
  --display_data=false
```

## Diffusion Result

Diffusion Policy worked better than ACT for the final robot behavior. It produced smoother movement and better grasping behavior.

### Branch → Red Organic Bin

<video src="media/diffusion_branch_rollout.mp4" controls width="720"></video>

[Watch branch diffusion rollout](media/diffusion_branch_rollout.mp4)

### Chocolate Bar → Blue Non-Organic Bin

<video src="media/diffusion_chocolate_rollout.mp4" controls width="720"></video>

[Watch chocolate diffusion rollout](media/diffusion_chocolate_rollout.mp4)

---

## SmolVLA Experiments

SmolVLA was tested as a language-conditioned vision-language-action policy.

Initial debug command:

```bash
lerobot-train \
  --policy.path=lerobot/smolvla_base \
  --dataset.repo_id=${DATASET_REPO_ID} \
  --output_dir=outputs/train/debug_smolvla_food_sorting \
  --job_name=debug_smolvla_food_sorting \
  --policy.device=cuda \
  --wandb.enable=false \
  --steps=100 \
  --batch_size=1 \
  --num_workers=0 \
  --log_freq=20 \
  --save_checkpoint=true \
  --save_freq=100 \
  --policy.repo_id=${HF_USER}/debug_smolvla_food_sorting \
  --policy.push_to_hub=false
```

The pretrained SmolVLA policy expected camera names:

```text
observation.images.camera1
observation.images.camera2
observation.images.camera3
```

Our dataset used:

```text
observation.images.front
observation.images.wrist
```

The fix was to use `--rename_map`:

```bash
lerobot-train \
  --policy.path=lerobot/smolvla_base \
  --dataset.repo_id=${DATASET_REPO_ID} \
  --rename_map='{
    "observation.images.front": "observation.images.camera1",
    "observation.images.wrist": "observation.images.camera2"
  }' \
  --policy.empty_cameras=1 \
  --policy.num_vlm_layers=8 \
  --policy.train_expert_only=true \
  --policy.freeze_vision_encoder=true \
  --output_dir=outputs/train/debug_smolvla_food_sorting \
  --job_name=debug_smolvla_food_sorting \
  --policy.device=cuda \
  --wandb.enable=false \
  --steps=100 \
  --batch_size=1 \
  --num_workers=0 \
  --log_freq=20 \
  --save_checkpoint=true \
  --save_freq=100 \
  --policy.repo_id=${HF_USER}/debug_smolvla_food_sorting \
  --policy.push_to_hub=false
```

SmolVLA was kept as experimental because ACT and Diffusion Policy were more practical for the available time and hardware.

---

## High-Level Object Selection

For a fully autonomous version, the system needs a high-level object selector.

Planned architecture:

```text
camera frame
    ↓
YOLO-E / VLM / classifier
    ↓
detected object class
    ↓
select correct LeRobot policy
    ↓
run rollout
```

Example:

```text
chocolate bar detected → run diffusion_chocolate_blue_wrist_30k
branch detected        → run diffusion_branch_red_wrist_30k
```

For the hackathon version, task selection can be manual or semi-automatic. The manipulation policies and robot rollouts are real.

---

## Media Conversion

The LeRobot dataset visualizer recordings were saved as `.webm` files and converted to GIFs:

```bash
ffmpeg -i media/raw/episode30_front.webm \
  -vf "fps=8,scale=720:-1" \
  media/episode30_front.gif

ffmpeg -i media/raw/episode30_wrist.webm \
  -vf "fps=8,scale=720:-1" \
  media/episode30_wrist.gif

ffmpeg -i media/raw/episode30_3d_view.webm \
  -vf "fps=8,scale=720:-1" \
  media/episode30_3d_view.gif

ffmpeg -i media/raw/episode30_graph.webm \
  -vf "fps=8,scale=720:-1" \
  media/episode30_graph.gif
```

The rollout demonstrations were kept as compressed MP4 videos:

```bash
ffmpeg -i media/raw/diffusion_branch_raw.mp4 \
  -vf "scale=720:-1" \
  -c:v libx264 \
  -crf 28 \
  -preset fast \
  -pix_fmt yuv420p \
  -movflags +faststart \
  media/diffusion_branch_rollout.mp4

ffmpeg -i media/raw/diffusion_chocolate_raw.mp4 \
  -vf "scale=720:-1" \
  -c:v libx264 \
  -crf 28 \
  -preset fast \
  -pix_fmt yuv420p \
  -movflags +faststart \
  media/diffusion_chocolate_rollout.mp4
```

---

## Repository Structure

```text
.
├── README.md
├── HONESTY.md
├── LICENSE
├── .gitignore
├── commands/
├── media/
│   ├── episode30_front.gif
│   ├── episode30_wrist.gif
│   ├── episode30_3d_view.gif
│   ├── episode30_graph.gif
│   ├── diffusion_branch_rollout.mp4
│   └── diffusion_chocolate_rollout.mp4
└── notes/
```

---

## Known Limitations

- ACT learned the general trajectory but did not grasp reliably.
- Diffusion Policy required wrist-only training because the original two cameras had different resolutions.
- The system currently uses manual or semi-automatic task selection.
- YOLO-E or another classifier can be added to choose the correct policy automatically.
- Camera device names changed between `/dev/video2` and `/dev/video3`.
- The front camera ran at 5 FPS, which constrained rollout speed.
- The dataset is small, with 60 total episodes.
- The robot works best when the object and bins are placed similarly to the demonstrations.
- Rollouts were manually supervised for safety.

---

## License

This repository is released under the MIT License.

External dependencies such as Hugging Face LeRobot, ACT, Diffusion Policy, SmolVLA, and YOLO-E retain their original licenses.

---

## Acknowledgements

This project uses:

- Hugging Face LeRobot
- SO-101 leader/follower robot arms
- Hugging Face Hub
- LeRobot Dataset Visualizer
- ACT
- Diffusion Policy
- SmolVLA
- Ultralytics YOLO-E for planned object selection
