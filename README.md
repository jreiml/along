# .along. — a set of helping hands

Two SO-101 arms you talk to. Say what you need ("hold this toothbrush", "turn it so I can put toothpaste on",
"press the soap") and Al does it, using three cameras and two grippers. Built at the Robotics & AI Hackathon, Berlin, 22 Nov 2026.

[![Demo: soap, toothbrush, toothpaste (2 min)](docs/along-demo.jpg)](https://github.com/jreiml/along/releases/download/demo/along-demo.mp4)
▶ [Watch the demo](https://www.youtube.com/watch?v=-Gc9LTbTNV4): press a soap dispenser, hold a toothbrush and toothpaste, turn the brush, hand it back. · [Slides](docs/along-slides.pdf)

## How it works

```
voice (GPT-Live) → router (gpt-6.1-sol, one call) → skill: hold · release · turn · tilt · press · open · park
                         └→ anything else: Al = a headless Codex turn driving `robot`
skills / Codex → so101.py (Python API) / robot CLI → robotd.py (only bus + camera owner, 50 Hz) → arms + cameras
```

- **robotd** owns the servos and cameras. Commands only change the target, so a new command overrides the running one. It ramps to targets, clamps to calibrated ranges, and survives replugging and power cuts.
- **Aiming**: the scene camera (C920) gives a pixel, `robot locate` turns it into xyz for each arm, and `kin.py` computes the IK.
- **Hand-over**: the arm reaches out with its jaws open, then a frame-difference check and a vision check confirm the object is between the jaws before it closes.

## Setup

```bash
./install.sh                     # uv sync, `robot` command, systemd user units, calibration links
systemctl --user start robotd    # logs: journalctl --user -u robotd -f
robot save rest                  # once, arms folded at rest
echo OPENAI_API_KEY=... > .env   # for along (GPT-Live + router)
systemctl --user start along
```

## Use

```bash
along on · along off             # voice on/off (also buttons on /along); while on, records cams + screen + audio
                                 # to ~/Videos/al-demo/<time>/ (ALONG_RECORD=0 = off)
along hold left toothbrush · along release left · along turn left 90 · along tilt right -20 · along park
along look "where is the toothbrush?" · along tell "hold this" · along attach   # vision · type to Al · Al's Codex TUI
robot step "goto right 0.28 0.03 0.06"   # move, wait, snap_all.jpg + state; `robot step` alone = look
robot recall ready right · robot locate 400 300 0.05 · robot ik right 0.3 0.05 0.06 · robot torque off left
robot stream                     # web view on :8101: cameras, /inspect (rollouts, 3D replay), /along (conversation, state)
robot help · along --help        # everything else
```

```python
from so101 import Robot
bot = Robot(); bot.step("move right_elbow_flex -5"); bot.right.goto(0.2, 0, 0.08); bot.locate(400, 300)
```

No hardware: `SO101_SIM=1` on robotd and clients, then `SO101_SIM=1 .venv/bin/python test_sim.py`.

## Files

| | |
|---|---|
| `along.py` · `along.html` · `codex_agent.py` · `llm.py` | voice helper, router, skills · its debug page · Al's Codex turns · model calls |
| `robotd.py` · `so101.py` · `robot.py` | daemon · Python API · CLI |
| `kin.py` · `so101.urdf` · `collide.py` · `planner.py` | FK/IK · URDF (TheRobotStudio/SO-ARM100) · table/arm collision · paths |
| `calib.py` · `scene.json` · `calibration/` · `poses.json` · `cams.json` | scene-camera calibration · table + camera pose · servo calibration · saved poses · camera settings |
| `screenrec.py` · `v4l.py` · `stream.html` · `inspect.html` | screen recording (desktop portal) · V4L2 controls · web pages |
| `AGENTS.md` · `prompts/along.md` | rules for coding agents on the arms (Al reads them too) · Al's brief |

## Hardware

- Left = white, right = orange; STS3215 ×6 each at 1 Mbaud, by-id serial paths in `robotd.py`. Units: degrees (0 = mid-range), gripper 0–100 % (0 = closed).
- Cameras: two wrist cams + Logitech C920 scene cam (also the mic). Plug the cameras into the laptop directly: through a hub they drop to USB 1.1.
- Leaving rest: raise shoulder_lift first, then elbow, then wrist. Never use LeRobot `max_relative_target`.
