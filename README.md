# g1-blind-locomotion

Blind (proprioception-only) rough-terrain walking policy for the Unitree G1 (29-DoF), trained in Isaac Lab on top of [unitreerobotics/unitree_rl_lab](https://github.com/unitreerobotics/unitree_rl_lab) and deployed on the real robot.

This repository contains **only the files I changed or added**, at the same paths as in unitree_rl_lab. It is not a standalone project: copy these files over an upstream checkout to use them.

## Demo
- Real robot: [TBD]

## What I changed (v2)
Starting from a flat-terrain velocity policy, training was resumed on rough terrain with:

| Change | Setting | Where |
|---|---|---|
| Rough terrain generator | flat 20%, random rough 30%, slopes up/down 15% each (0–0.3), stairs up/down 10% each (step height 0–0.08 m) | `BLIND_ROUGH_TERRAINS_CFG` |
| Survival terrain curriculum | promote only if the robot walked more than half the tile **and did not fall**; demote if it walked less than half the commanded distance **or fell** | `terrain_levels_survival` |
| Action delay randomization | joint position targets applied 0–2 physics steps late (sim dt 0.005 s → 0–10 ms), resampled per episode | `DelayedJointPositionAction` |
| PD gain randomization | stiffness and damping scaled by U(0.9, 1.1) per joint at reset | `RoughEventCfg.actuator_gains` |
| Command ranges (training) | lin_vel_x (-0.3, 0.6), lin_vel_y (-0.3, 0.3), ang_vel_z (-0.5, 0.5) | `RobotRoughEnvCfg` |
| New task | `Unitree-G1-29dof-Velocity-Rough` (+ play config) | `29dof/__init__.py` |

The policy observes only base angular velocity, projected gravity, velocity command, joint positions/velocities and last action. There is no height scan in the policy observations; the observation/action dimensions are the same as the flat policy, so training can resume from a flat checkpoint.

Smaller changes to the base config, carried over from my earlier flat-terrain setup:
- `velocity_env_cfg.py`: initial/limit command ranges, waist deviation penalty weight (-1 → -0.1), `ang_vel_cmd_levels` curriculum enabled, flat `terrain_levels_vel` curriculum disabled.
- `mdp/curriculums.py`: `ang_vel_cmd_levels` promotion threshold 0.8 → 0.5.

`docs/upstream.diff` shows the exact diff of these files against upstream commit `4960b84`.

## Files
| Path | Purpose |
|---|---|
| `source/.../robots/g1/29dof/velocity_env_cfg.py` | Rough-terrain env config: terrain, survival curriculum, action delay, PD gain randomization |
| `source/.../robots/g1/29dof/__init__.py` | Registers `Unitree-G1-29dof-Velocity-Rough` |
| `source/.../mdp/curriculums.py` | Angular velocity command curriculum threshold |
| `deploy/robots/g1_29dof/config/policy/velocity/v2/params/deploy.yaml` | Deploy config written by upstream `train.py` for the v2 run (joint order, gains, obs/action scales) |
| `docs/upstream.diff` | Diff against upstream |

## Training
1. Set up Isaac Lab and unitree_rl_lab following the [upstream instructions](https://github.com/unitreerobotics/unitree_rl_lab#installation), at commit `4960b84`.
2. Copy the `source/` files from this repository over the upstream tree.
3. Train a flat policy first (`Unitree-G1-29dof-Velocity`), then resume it on rough terrain:
   ```bash
   python scripts/rsl_rl/train.py --task Unitree-G1-29dof-Velocity-Rough --headless \
       --resume --load_run <flat_run_dir> --checkpoint model_<N>.pt \
       agent.algorithm.entropy_coef=0.001 agent.algorithm.desired_kl=0.005
   ```
   PPO settings used for v2 (from the run's `agent.yaml`): learning rate 1e-3, entropy coefficient 0.001 (upstream default 0.01), desired KL 0.005 (upstream default 0.01), 24 steps per env, seed 42. v2 resumed from flat checkpoint iteration 7100.

Note: my local run used rsl-rl 5.0.1, which needed a small runner-config compatibility change in `train.py`. That change is not included here; upstream `train.py` is used as is.

## Training results (from the training log)
At iteration 13192 of the v2 run:

| Logged metric | Value |
|---|---|
| Mean episode length (steps; episode limit 20 s = 1000 steps) | 995.63 |
| `Episode_Termination/time_out` | 0.9791 |
| `Curriculum/terrain_levels` (mean level, 10 rows) | 3.1524 |
| `Episode_Reward/track_lin_vel_xy` | 0.8618 |
| `Episode_Reward/track_ang_vel_z` | 0.4018 |

Real-robot results: [TBD]

## Play, export and deploy
```bash
# play in Isaac Lab (upstream play.py also exports exported/policy.onnx next to the checkpoint)
python scripts/rsl_rl/play.py --task Unitree-G1-29dof-Velocity-Rough --load_run <run_dir>
```
Copy the exported policy next to the deploy config (`params/deploy.yaml` in the run folder; the v2 one is included here):
```
deploy/robots/g1_29dof/config/policy/velocity/v2/
├── exported/policy.onnx      # from logs/rsl_rl/<experiment>/<run>/exported/ (not included)
└── params/deploy.yaml        # included
```
Then build and run the upstream controller (`deploy/robots/g1_29dof`) as described in the upstream README: sim2sim in unitree_mujoco first, then sim2real with `./g1_ctrl --network <net-if>`. The controller loads the last version folder under `config/policy/velocity/` that contains `exported/`.

Checkpoints and ONNX files are not included.

## License and attribution
- Upstream: [unitreerobotics/unitree_rl_lab](https://github.com/unitreerobotics/unitree_rl_lab), Apache License 2.0.
- The files in this repository are modified versions of upstream files and are distributed under the same license. See [`LICENSE`](./LICENSE) and [`NOTICE`](./NOTICE).
