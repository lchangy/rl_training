# rl_training 项目代码逻辑与运行流程（新手向）

这份文档用中文、按“从入口脚本到训练循环”的顺序讲清楚：项目依赖了什么、代码如何层层拼起来、训练/回放是怎么跑起来的。

## 1) 项目在做什么（一句话）

这是一个基于 **Isaac Lab** 的机器人强化学习训练项目：给机器人下达“速度指令”，训练一个策略让它在不同地形上稳定行走/运动。

核心流程可以记成：  
`训练脚本 -> 任务注册 -> 环境配置 -> 物理仿真 + 传感器 + 奖励 -> RL算法更新策略`

## 2) 主要依赖（谁负责什么）

- **Isaac Sim**：物理仿真和渲染（机器人、地形、碰撞等）。
- **Isaac Lab**：环境和任务框架（ManagerBased 环境、观测/奖励/事件/终止等“管理器”结构）。
- **rsl_rl**：训练循环（采样、优势估计、策略更新等）。
- **PyTorch**：神经网络训练引擎（rsl_rl 的底层计算）。

这些依赖不是在本仓库里实现的，而是你安装在 Python 环境里的库。

## 3) 目录结构与职责（新手最容易迷路的部分）

- `source/rl_training/rl_training/`  
  本项目的 Python 包根目录。

- `source/rl_training/rl_training/tasks/`  
  所有任务定义与注册的位置。

- `source/rl_training/rl_training/tasks/manager_based/locomotion/velocity/`  
  “速度跟踪”任务的通用配置。  
  这里定义“场景、命令、动作、观测、奖励、终止、事件、课程”这些模块化部件。

- `source/rl_training/rl_training/tasks/manager_based/locomotion/velocity/config/`  
  机器人/地形的具体配置：比如 Lite3（四足）或 M20（轮式）。  
  这些文件会继承通用配置，然后覆盖具体参数。

- `scripts/reinforcement_learning/rsl_rl/`  
  训练和回放脚本入口（train.py / play.py）。

- `logs/`、`outputs/`  
  训练日志、模型权重、导出的模型文件。

## 4) 任务是如何注册的（为什么 `--task` 能生效）

**第一步：导入任务包时自动注册。**  
`source/rl_training/rl_training/tasks/__init__.py` 中使用 `import_packages(...)` 自动扫描并导入子包。  
这一步会触发各个任务的 `__init__.py` 执行注册逻辑。

**第二步：具体任务注册发生在 config 的 __init__.py 里。**  
例如 Lite3 的注册在：  
`source/rl_training/rl_training/tasks/manager_based/locomotion/velocity/config/quadruped/deeprobotics_lite3/__init__.py`  
其中 `gym.register(...)` 把任务 ID（如 `Rough-Deeprobotics-Lite3-v0`）注册到 Gymnasium。

**第三步：训练脚本会 import 任务包。**  
`scripts/reinforcement_learning/rsl_rl/train.py` 里有：
```python
import rl_training.tasks
```
这行就是触发“自动注册”的关键。

## 5) 配置体系的层级关系（最重要）

这套环境配置是“层层继承 + 覆盖”的结构：

1) **通用任务配置（基类）**  
`source/rl_training/rl_training/tasks/manager_based/locomotion/velocity/velocity_env_cfg.py`  
这里定义了完整任务的骨架：  
场景、传感器、命令、观测、动作、奖励、终止条件、随机化事件、课程学习等。

2) **机器人专用配置（子类）**  
比如 Lite3 粗糙地形配置：  
`source/rl_training/rl_training/tasks/manager_based/locomotion/velocity/config/quadruped/deeprobotics_lite3/rough_env_cfg.py`  
它继承 `LocomotionVelocityRoughEnvCfg`，然后在 `__post_init__` 里做“具体化”：
   - 替换机器人资产（DEEPROBOTICS_LITE3_CFG）
   - 指定关节/连杆命名
   - 调整观测缩放、动作缩放
   - 设置奖励权重和参数
   - 开关部分随机化/终止/课程

3) **算法配置（PPO 之类）**  
在注册处也会指向 RSL-RL 的配置入口，比如：
```python
"rsl_rl_cfg_entry_point": "....rsl_rl_ppo_cfg:DeeproboticsLite3RoughPPORunnerCfg"
```
这决定了训练算法、网络结构、学习率等超参数。

## 6) 训练流程（train.py 的真实运行顺序）

当你运行：
```bash
python scripts/reinforcement_learning/rsl_rl/train.py --task=Rough-Deeprobotics-Lite3-v0 --headless
```
实际发生的步骤是：

1) **AppLauncher 启动 Isaac Sim**  
`AppLauncher` 负责起仿真进程。

2) **Hydra 根据 task ID 找到配置**  
`@hydra_task_config(args_cli.task, args_cli.agent)` 会解析任务 ID，拿到：
   - 环境配置类（env_cfg_entry_point）
   - 算法配置类（rsl_rl_cfg_entry_point）

3) **创建 Gym 环境**  
`gym.make(args_cli.task, cfg=env_cfg)`  
这一步会真正实例化 Isaac Lab 的环境，并加载地形/机器人/传感器等。

4) **环境包装（RSL-RL 需要）**  
`RslRlVecEnvWrapper` 让环境适配 rsl_rl 的训练接口。

5) **创建训练器并开始学习**  
`OnPolicyRunner(...).learn(...)`  
从环境采样 -> 计算优势 -> 更新策略网络 -> 反复循环。

6) **日志与模型保存**  
训练过程会在 `logs/rsl_rl/...` 下写入参数、checkpoint、视频等。

## 7) 回放流程（play.py 的运行顺序）

运行：
```bash
python scripts/reinforcement_learning/rsl_rl/play.py --task=Rough-Deeprobotics-Lite3-v0 --num_envs=10
```
主要流程是：

1) 启动 Isaac Sim  
2) 加载环境配置（与训练一样）  
3) 找到 checkpoint（或预训练模型）  
4) 用 `OnPolicyRunner` 加载模型并拿到推理策略  
5) 循环：`policy(obs) -> env.step(actions)`  
6) 可选导出 `policy.onnx` / `policy.pt` 方便部署

## 8) 新手最常改的地方（给你一个方向）

- **想改奖励**：  
通用奖励在 `velocity_env_cfg.py`，具体权重通常在 `rough_env_cfg.py` 里覆盖。

- **想改观测**：  
`ObservationsCfg.PolicyCfg` 决定了策略看什么。  
例如删除高度扫描、调整噪声范围。

- **想改动作幅度**：  
`ActionsCfg` 里有 `scale` 和 `clip`。

- **想改地形**：  
在 `velocity_env_cfg.py` 的 `TerrainImporterCfg`，或在机器人子类里改 `terrain_generator` 的比例。

- **想改任务 ID / 注册**：  
去对应机器人 config 里的 `__init__.py` 看 `gym.register`。

## 9) 最简单的心智模型（记住这几句就够了）

- 环境配置 = 游戏规则  
- 机器人模型 = 角色  
- 观测 = 角色能看到的东西  
- 动作 = 角色能做的事  
- 奖励 = 游戏得分  
- RL 算法 = 学会得高分的方法

---

如果你想从某个文件开始深入，我可以一行一行带你读（比如 `velocity_env_cfg.py` 或 `train.py`）。  
