# HONESTY.md

> Mandatory disclosure for the hackathon. This file lives at the root of the repository. Judges can cross-check it against the repository, README, dataset, and technical demo.
>
> **The goal of this file is transparency.** The robot dataset, model training, and rollouts were real. Some parts of the full intended system, such as automatic object classification and policy selection, were not fully implemented during the hackathon and are disclosed below.

---

## 1. Team — who did what

| Member | GitHub handle | Main contributions |
|---|---|---|
| Hugo Arsénio | `Hugo132645` / `@hugooaarsenio` | LeRobot setup, camera debugging, dataset verification, Hugging Face dataset upload, ACT training, Diffusion Policy training, SmolVLA experiment, rollout testing, README/HONESTY documentation |
| Batuhan Sakaoglu | `batusjr` | SO-101 leader/follower calibration, teleoperation support, physical robot setup, technical video |
| Austeja Lazaraviciute | `astlz` | Data collection support, idea design, human resources, business video and pitching expert |

---

## 2. What is fully working

Features that run end-to-end with real data, real hardware, and real logic:

- **SO-101 leader/follower teleoperation**
  - Input: human movement of the SO-101 leader arm.
  - Output: the SO-101 follower arm moves accordingly.
  - Status: working after calibration.

- **Camera setup with LeRobot**
  - Input: front camera and wrist camera streams.
  - Output: synchronized camera observations stored in the LeRobot dataset.
  - Status: working, although camera device names sometimes changed between `/dev/video2` and `/dev/video3`.

- **Real robot dataset recording**
  - Input: teleoperated demonstrations for two tasks.
  - Output: LeRobot dataset containing robot states, actions, camera observations, timestamps, episode metadata, and task labels.
  - Status: working.

- **Hugging Face dataset upload**
  - Dataset repo:
    ```text
    Hugo132645/OW_Dataset_20260606_230614
    ```
  - Status: uploaded and visualizable with the LeRobot dataset visualizer.

- **Dataset verification**
  - Input: Hugging Face LeRobot dataset.
  - Output: confirmed frame count, episode count, task labels, and feature structure.
  - Verified dataset:
    ```text
    60 total episodes
    17,530 frames
    30 chocolate bar episodes
    30 branch episodes
    ```

- **ACT training**
  - Input: full 60-episode dataset.
  - Output: trained ACT checkpoint.
  - Status: completed 20k training steps.
  - Result: policy loaded and controlled the robot, but did not grasp reliably.

- **Diffusion Policy training**
  - Input: wrist-camera observations and robot state/action data.
  - Output: trained Diffusion Policy checkpoints for the two tasks.
  - Status: completed.
  - Result: worked better than ACT for the final robot rollout.

- **Diffusion Policy rollout**
  - Input: wrist camera observation, robot state, and task-specific trained diffusion checkpoint.
  - Output: real SO-101 follower arm movement.
  - Status: working better than ACT during physical testing and grabbed the object in around 50% of the trials.

- **SmolVLA experiment**
  - Input: full dataset with task language instructions.
  - Output: trained SmolVLA checkpoint.
  - Status: trained and loaded, but did not reliably grasp the object during rollout.
  - Main reason: only around 5k training steps were possible within the available time.

---

## 3. What is mocked, stubbed, or hardcoded

Every shortcut is disclosed here.

| What is faked / simplified | Where | Why we simplified it | What the real version would do |
|---|---|---|---|
| Automatic object classification | Not implemented as a full working module in the final demo | The hackathon time was focused on real robot data collection, policy training, and rollout | Use YOLO-E, CLIP, or another vision-language/object detector to classify the visible object as chocolate/branch and select the correct policy |
| Policy selection | Manual or semi-manual during testing | Safer and more reliable for the physical demo | Automatically select `diffusion_chocolate_blue_wrist_30k` or `diffusion_branch_red_wrist_30k` based on object detection |
| Object/bin positions | Physical demo setup | Imitation learning with a small dataset is sensitive to object and bin placement | A robust system would generalize to varied object positions, orientations, lighting, and bin locations |
| Safety supervision | Manual human supervision | Real robot rollout can be unsafe if a policy outputs a bad action | A production system would include automatic collision checking, force limits, emergency stop logic, and safer motion constraints |
| Branch-only dataset creation | Created by editing the original dataset | Directly training with episodes `30–59` caused a dataset indexing issue | A cleaner pipeline would record branch-only data from the start or fix the dataset indexing/filtering issue |

No fake robot outputs or fake policy rollouts were used. The dataset recordings, ACT rollout, Diffusion Policy rollout, and SmolVLA rollout attempts were performed with the real robot setup.

---

## 4. External APIs, services & data sources

Everything the project used or depended on:

| Service / API / dataset | Used for | Real call or mocked? | Auth |
|---|---|---|---|
| Hugging Face Hub | Storing and downloading the LeRobot dataset | Real | Hugging Face account/token |
| Hugging Face LeRobot | Robot calibration, teleoperation, dataset recording, training, rollout, dataset visualization | Real | None for local use; Hugging Face token for upload |
| LeRobot Dataset Visualizer | Viewing recorded episodes, camera streams, robot states, actions, and graphs | Real | None / Hugging Face web access |
| SO-101 leader arm | Human teleoperation input | Real hardware | Local serial/USB |
| SO-101 follower arm | Physical robot policy rollout | Real hardware | Local serial/USB |
| OpenCV cameras | Front and wrist visual observations | Real hardware input | None |
| ACT policy implementation | Baseline imitation learning policy | Real implementation from LeRobot | None |
| Diffusion Policy implementation | Main final manipulation policy | Real implementation from LeRobot | None |
| SmolVLA / SmolVLM weights | Experimental vision-language-action training | Real pretrained model download | Hugging Face model access |
| ChatGPT | Debugging help, github template | AI assistance only; not part of runtime robot system | User prompts |

---

## 5. Pre-existing code

Anything not written from scratch during the hackathon:

| Item | Source | Roughly how much | License |
|---|---|---|---|
| Hugging Face LeRobot | `https://github.com/huggingface/lerobot` | Core robotics framework for calibration, teleoperation, data collection, training, dataset handling, and rollout | LeRobot license applies |
| ACT policy implementation | Included in LeRobot | Used through `lerobot-train` and `lerobot-rollout` | LeRobot license applies |
| Diffusion Policy implementation | Included in LeRobot | Used through `lerobot-train` and `lerobot-rollout` | LeRobot license applies |
| SmolVLA implementation | Included in LeRobot / Hugging Face model ecosystem | Used experimentally through `lerobot-train` | Relevant model and LeRobot licenses apply |
| SmolVLM2 pretrained weights | Hugging Face model: `HuggingFaceTB/SmolVLM2-500M-Video-Instruct` | Used as part of SmolVLA experiment | Model license applies |
| EuroTech LeRobot tutorial | `https://github.com/dariusss04/eurotech_hackathon_lerobot_tutorial` | Used as guidance for the workflow and command structure | Tutorial repository license applies |

The contribution of this repository is not a new implementation of ACT, Diffusion Policy, SmolVLA, or LeRobot. The contribution is the complete real-world robot-learning pipeline: hardware setup, data collection, dataset upload, training configuration, debugging, policy comparison, rollout testing, and documentation.

---

## 6. Known limitations & next steps

- **ACT did not reliably grasp the object.**
  - ACT learned the general trajectory but produced trembling/unstable behavior during the fine grasping phase.

- **Diffusion Policy worked better, but still depends on controlled setup.**
  - The robot performs best when the object and bins are placed similarly to the demonstrations.

- **The dataset is small.**
  - The final dataset has 60 total episodes, which is useful for a hackathon demonstration but limited for robust generalization.

- **The original dataset had mismatched camera resolutions.**
  - Front camera: `1920x1080`
  - Wrist camera: `640x480`
  - Because of this, Diffusion Policy was trained using only the wrist camera.

- **Camera device names were unstable.**
  - The wrist camera sometimes appeared as `/dev/video2` and sometimes as `/dev/video3`.
  - A more robust setup should use `/dev/v4l/by-id/...` paths.

- **The front camera ran at low FPS.**
  - The front camera was recorded at 5 FPS, which limited rollout speed and caused control loop warnings.

- **SmolVLA was not trained long enough.**
  - SmolVLA trained and loaded, but did not reliably grasp during rollout.
  - It was only trained for around 5k steps due to time and compute limits.

- **The final system does not yet include automatic object classification.**
  - The current demo uses manual or semi-manual task/policy selection.
  - A future version should use YOLO-E, CLIP, or another vision-language/object detection model to select the correct policy automatically.

- **No production-grade safety system is implemented.**
  - Rollouts were manually supervised.
  - A real deployment would need collision checking, force limits, emergency stop integration, and safer motion constraints.

- **Future improvement: train more episodes.**
  - More demonstrations with varied object positions, lighting, orientations, and bin locations would improve robustness.

---

## Final disclosure summary

This project demonstrates a real end-to-end robot learning workflow:

```text
real teleoperation → real LeRobot dataset → Hugging Face upload → ACT baseline → Diffusion Policy training → real robot rollout
```

The dataset, training, and robot rollouts are real. The main incomplete parts are fully automatic object classification, robust generalization, and production-grade safety.
