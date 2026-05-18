# 🐶 Pupper V3 AprilTag Controller

Turn printed AprilTags into joystick-like commands for the Pupper V3 robot while keeping the existing Lab 5 reinforcement-learning walking policy intact. The robot sees a tag through its camera, estimates the tag distance, converts the tag ID into a movement command, beeps to acknowledge the command, waits one second, and then sends the command through the same ROS 2 control path that the PS5/PS-style joystick used.

---

## ✨ What This Project Does

This package adds a visual command layer on top of the trained neural locomotion controller:

1. 📷 **Camera perception** detects AprilTags from the robot camera.
2. 🏷️ **Tag decoding** reads the numeric AprilTag ID.
3. 🧠 **Command mapping** looks up the tag ID in `DefaultAprilTagCommands.yaml`.
4. 🧮 **Command resolution** chooses the closest tag, combines multiple tags in net-sum mode, or interprets tag roll angle in orientation mode.
5. 🔊 **Audio feedback** beeps when a command is accepted.
6. 🤖 **Robot execution** publishes synthetic `/joy` commands by default so the existing Lab 5 neural controller receives commands as if they came from a joystick.

The reinforcement-learning policy caller is treated as the walking engine. This package only replaces the human joystick input source with AprilTag commands.

---

## 🧩 Main Features

### ✅ Closest-Tag Mode

The default mode uses the closest valid AprilTag as the active command.

Example:

- Tag `1` = Forward 100%
- Tag `7` = Forward 50%
- Tag `5` = Turn Left 100%

If the robot sees several tags, it executes the nearest valid one.

---

### ➕ Instruction Net-Sum Mode

Enable this when multiple users hold different tags at the same time:

```bash
python3 DeployPolicyAprilTag.py --skip-policy-download InstructionNetSum=Enabled
```

Rules:

- Opposite commands cancel based on relative distance.
- The farthest tag becomes the denominator.
- The closer opposite command wins the direction.
- Non-conflicting commands are added directly.
- If two tags represent the same command, the farther duplicate is ignored.

Example:

- Forward 100% at 5 m
- Backward 100% at 3 m

Result:

```text
1 - 3 / 5 = 0.40
```

The robot moves backward at 40% power because the backward tag is closer.

---

### 🧭 Orientation Mode

Enable this when a user wants to steer using tag rotation:

```bash
python3 DeployPolicyAprilTag.py --skip-policy-download OrientationMode=Enabled
```

Orientation mode uses two special tag roles:

| Role | Default Tags | Behavior |
|---|---:|---|
| 🧭 Direction Wheel | `40`, `41` | Rotating the tag changes the direction of linear motion. Upright means forward. |
| 🔄 Rotation Wheel | `42`, `43` | Rotating the tag left/right creates yaw rotation. Upright/downward means no yaw. |

The two tags do not both need to be visible. If one is missing, that part of the command is treated as zero.

---

### 🔊 Beep Confirmation

The robot makes a short sound before executing accepted commands.

Default behavior:

- Closest-tag mode: 1 beep.
- Net-sum mode: 1 beep per accepted visible tag.
- Beeps are spaced by `0.3` seconds.
- Motion begins `1.0` second after the beep sequence.

These settings live in `DefaultAprilTagCommands.yaml`:

```yaml
AudioFeedback: Enabled
ExecutionDelayAfterNoiseSeconds: 1.0
NoiseBeepSpacingSeconds: 0.3
BeepDurationSeconds: 0.12
```

---

### 🛑 Safety Commands

Some AprilTags create synthetic joystick button presses for policy switching and emergency control.

| Tag ID | Command | Button Press |
|---:|---|---:|
| `20` | Switch default neural controller | `0` |
| `21` | Switch three-legged controller | `1` |
| `22` | Switch parkour controller | `2` |
| `23` | Switch test policy | `3` |
| `24` | Release emergency stop | `9` |
| `25` | Emergency stop | `12` |

Keep a real emergency-stop method available during testing.

---

## 📁 File Map

| File | Purpose |
|---|---|
| `DeployPolicyAprilTag.py` | Launches the existing neural controller and then starts the AprilTag controller. |
| `AprilTagPupperController.py` | Robot-side ROS 2 node for camera input, AprilTag detection, command publishing, logging, and beeps. |
| `AprilTagCommandMath.py` | Pure command logic for closest-tag mode, net-sum mode, and orientation mode. |
| `DefaultAprilTagCommands.yaml` | Main configuration file for tag IDs, powers, camera settings, controller settings, and joystick mapping. |
| `GenerateAprilTagCommands.py` | User-side script that creates printable AprilTag command pages from the YAML file. |
| `VerifyAprilTagController.py` | Offline verification script for tag generation, tag recognition, command math, and controller edge cases. |
| `setup_robot_dependencies.sh` | Installs robot-side dependencies on the Raspberry Pi. |
| `requirements_robot.txt` | Robot-side Python dependency list. |
| `requirements_user.txt` | Computer/user-side Python dependency list for tag generation. |
| `GeneratedAprilTags/CommandTags.pdf` | Printable default command tags. |
| `GeneratedAprilTags/CommandTagSummary.csv` | Tag ID summary table generated with the tag PDF. |
| `docs/Pupper_AprilTag_Controller_Instructions.pdf` | Full step-by-step setup and verification guide. |
| `policy/test_policy.json` | Uploaded trained policy file included for deployment/testing. |
| `policy/config.yaml` | Uploaded policy configuration file. |
| `policy/estop_controller.cpp` | Reference emergency-stop and controller-switch logic. |

---

## 🛠️ Robot-Side Installation

Copy the project folder to the robot. A good location is:

```bash
~/pupperv3-monorepo/ros2_ws/src/pupper_apriltag_project
```

Then install dependencies:

```bash
cd ~/pupperv3-monorepo/ros2_ws/src/pupper_apriltag_project
bash setup_robot_dependencies.sh
```

Verify that OpenCV, YAML, and NumPy load correctly:

```bash
python3 - <<'PY'
import cv2
import cv2.aruco
import yaml
import numpy
print("✅ Dependencies loaded")
print("OpenCV version:", cv2.__version__)
print("AprilTag dictionary exists:", hasattr(cv2.aruco, "DICT_APRILTAG_36h11"))
PY
```

---

## 🖨️ Generate Printable AprilTags

On your computer or on the robot:

```bash
python3 GenerateAprilTagCommands.py \
  --config DefaultAprilTagCommands.yaml \
  --output_dir GeneratedAprilTags
```

Generated outputs:

```text
GeneratedAprilTags/CommandTags.pdf
GeneratedAprilTags/CommandTagSummary.csv
```

Print `CommandTags.pdf`. Keep the physical printed size consistent with the YAML value:

```yaml
PrintedTagSizeMillimeters: 160.0
TagSizeMeters: 0.160
```

This matters because the robot estimates tag distance from apparent tag size.

---

## 🚀 Run the Robot

### 1. Confirm the normal Lab 5 controller works

Before using AprilTags, make sure the normal neural controller can walk using the joystick.

### 2. Start AprilTag control

```bash
cd ~/pupperv3-monorepo/ros2_ws/src/pupper_apriltag_project
python3 DeployPolicyAprilTag.py --skip-policy-download
```

### 3. Run with net-sum mode

```bash
python3 DeployPolicyAprilTag.py --skip-policy-download InstructionNetSum=Enabled
```

### 4. Run with orientation mode

```bash
python3 DeployPolicyAprilTag.py --skip-policy-download OrientationMode=Enabled
```

### 5. Run with direct velocity output instead of joystick output

Use this only if you intentionally want to bypass synthetic joystick publishing:

```bash
python3 DeployPolicyAprilTag.py --skip-policy-download CommandOutputMode=CmdVel teleop:=False
```

Default mode is safer:

```yaml
CommandOutputMode: Joy
```

---

## 🎮 Default Joystick Mapping

The controller publishes synthetic `/joy` messages by default.

| Robot Command | Joystick Axis | Default Axis ID |
|---|---|---:|
| Forward/backward | `AxisLinearX` | `1` |
| Left/right strafe | `AxisLinearY` | `0` |
| Turn left/right | `AxisAngularYaw` | `3` |

Configured in:

```yaml
JoyMapping:
  AxisLinearX: 1
  AxisLinearY: 0
  AxisAngularYaw: 3
  AxesCount: 8
  ButtonsCount: 16
```

---

## 🏷️ Default AprilTag Commands

| Tag ID | Command |
|---:|---|
| `0` | Stop |
| `1` | Forward 100% |
| `2` | Backward 100% |
| `3` | Left 100% |
| `4` | Right 100% |
| `5` | Turn Left 100% |
| `6` | Turn Right 100% |
| `7` | Forward 50% |
| `8` | Backward 50% |
| `9` | Left 50% |
| `10` | Right 50% |
| `11` | Turn Left 50% |
| `12` | Turn Right 50% |
| `13` | Forward Left 100% |
| `14` | Forward Right 100% |
| `15` | Backward Left 100% |
| `16` | Backward Right 100% |
| `20`-`25` | Controller switch and emergency commands |
| `26`-`31` | 25% movement commands |
| `40`-`43` | Orientation-mode tags |

---

## 🧬 How Command Encoding Works

AprilTags do not store arbitrary text like QR codes. They store a numeric ID inside a tag family.

This project encodes command information like this:

```text
Printed AprilTag ID -> YAML lookup -> command vector -> joystick or velocity output
```

Example YAML entry:

```yaml
1: {Name: Forward100, Command: Forward, Power: 1.0, LinearX: 1.0, LinearY: 0.0, AngularZ: 0.0}
```

Meaning:

- Tag ID `1`
- Command name `Forward100`
- Power `1.0`
- Forward axis `+1.0`
- No sideways movement
- No yaw rotation

To create a new command:

1. Pick an unused tag ID.
2. Add it to `DefaultAprilTagCommands.yaml`.
3. Regenerate the tag PDF.
4. Print the new tag.
5. Copy the updated YAML file to the robot.
6. Restart `DeployPolicyAprilTag.py`.

---

## 📷 Camera Setup

The default camera backend subscribes to ROS image topics:

```yaml
RosImageTopics:
  - /camera/image_raw
  - /image_raw
  - /camera_node/image_raw
```

Find your actual camera topic:

```bash
ros2 topic list | grep image
```

Launch with a custom camera topic:

```bash
python3 DeployPolicyAprilTag.py --skip-policy-download CameraTopic=/your/image/topic
```

If no image arrives, the controller prints a warning and tells you to check the camera topic.

---

## 🧪 Verification

Run the offline verifier:

```bash
python3 VerifyAprilTagController.py
```

This checks:

- ✅ YAML loading
- ✅ AprilTag image generation
- ✅ AprilTag recognition
- ✅ closest-tag command selection
- ✅ net-sum command math
- ✅ same-command duplicate filtering
- ✅ diagonal non-conflicting command combination
- ✅ orientation mode
- ✅ edge cases such as missing tags and invalid ranges

Expected result:

```text
All verification tests passed.
```

---

## 🧯 First-Test Safety Checklist

Before allowing the robot to walk freely:

- 🧍 Keep the robot on a stand or hold it with feet lightly unloaded.
- 🛑 Keep the real joystick/emergency-stop method nearby.
- 🏷️ Test tag detection with `VerboseEveryFrame: true` first.
- 🔉 Confirm the robot beeps before motion.
- 🐢 Start with 25% or 50% power tags.
- 📏 Stay within the configured range limits.
- 🔋 Make sure the battery is charged.
- 🧱 Keep the floor clear.

---

## 🪵 Runtime Logs

The terminal logs every accepted command and its distance.

Typical log line:

```text
Accepted tag 1 Forward100 at 2.37 m -> LinearX=1.00 LinearY=0.00 AngularZ=0.00
```

In net-sum mode, the log lists all tags used in the combined command.

---

## ⚙️ Important Configuration Values

In `DefaultAprilTagCommands.yaml`:

```yaml
Detection:
  MaxRangeMeters: 6.0
  MinRangeMeters: 0.15

Controller:
  CommandOutputMode: Joy
  InstructionNetSum: Disabled
  OrientationMode: Disabled
  AudioFeedback: Enabled
  ExecutionDelayAfterNoiseSeconds: 1.0
  SafetyStopWhenNoTagSeconds: 0.25
  MaxLinearX: 0.75
  MaxLinearY: 0.5
  MaxAngularZ: 2.0
```

Meaning:

- `MaxRangeMeters` rejects tags that are too far away.
- `MinRangeMeters` rejects tags unrealistically close to the camera.
- `SafetyStopWhenNoTagSeconds` stops command output shortly after tags disappear.
- `MaxLinearX`, `MaxLinearY`, and `MaxAngularZ` scale normalized tag commands into robot velocity limits.

---

## 🧯 Troubleshooting

### ❌ No camera frames

Run:

```bash
ros2 topic list | grep image
```

Then relaunch with:

```bash
python3 DeployPolicyAprilTag.py --skip-policy-download CameraTopic=/correct/topic
```

---

### ❌ AprilTags are detected but distance is wrong

Check that the printed tag size matches:

```yaml
TagSizeMeters: 0.160
PrintedTagSizeMillimeters: 160.0
```

If the print size changed, update both values and regenerate the tags.

---

### ❌ Robot detects tags but does not move

Check these in order:

1. The Lab 5 neural controller works with the normal joystick.
2. `/joy` messages are being published.
3. The emergency stop is released.
4. The active neural controller is switched on.
5. The tag command has nonzero `Power` and nonzero vector fields.

Useful commands:

```bash
ros2 topic echo /joy
ros2 topic list
```

---

### ❌ Robot moves in the wrong direction

Invert the affected joystick sign in YAML:

```yaml
AxisLinearXSign: -1.0
AxisLinearYSign: -1.0
AxisAngularYawSign: -1.0
```

Only change the axis that is wrong.

---

## 🧠 Basic Robot Behavior Summary

The robot behaves like a visually commanded joystick-controlled Pupper: it watches for AprilTags, turns the nearest or combined set of valid tags into normalized forward, sideways, and turning commands, beeps to confirm what it received, waits one second, and then sends those commands to the existing reinforcement-learning locomotion controller. In normal mode it follows the closest tag; in net-sum mode it combines multiple users' tags by distance-weighted cancellation and non-conflicting vector addition; in orientation mode it treats a tag like a steering wheel so rotating the printed marker changes movement direction or yaw. The policy still performs the actual walking, while AprilTags replace the PS-style joystick as the high-level command source.

---

## 📦 Minimal Run Sequence

```bash
# 1. Enter project folder
cd ~/pupperv3-monorepo/ros2_ws/src/pupper_apriltag_project

# 2. Install robot dependencies
bash setup_robot_dependencies.sh

# 3. Generate/check tags
python3 GenerateAprilTagCommands.py --config DefaultAprilTagCommands.yaml --output_dir GeneratedAprilTags

# 4. Run offline verification
python3 VerifyAprilTagController.py

# 5. Launch neural controller + AprilTag controller
python3 DeployPolicyAprilTag.py --skip-policy-download
```

---

## ✅ Recommended Demo Order

1. 🏷️ Show Tag `0` Stop.
2. 🐢 Show Tag `26` Forward 25%.
3. 🚶 Show Tag `7` Forward 50%.
4. 🏃 Show Tag `1` Forward 100%.
5. ↩️ Show Tag `5` Turn Left 100%.
6. ➕ Enable net-sum mode and show opposite tags at different distances.
7. 🧭 Enable orientation mode and steer with Tag `40`.
8. 🛑 Show Tag `25` Emergency Stop.

---

## 🐾 Final Note

Use AprilTags as the high-level command interface, not as the walking controller itself. The AprilTag layer decides **what velocity command the robot should follow**; the trained reinforcement-learning policy decides **how the legs move to follow that command**.
