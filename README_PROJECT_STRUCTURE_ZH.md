# 四足强化学习项目代码结构与抽象关系（详细版）

本文面向初学者，解释本项目在 **Isaac Lab** 框架下的代码结构、模块职责、以及“抽象层级是怎么一层层组合起来的”。

> 适用范围：当前仓库 `rl_training`（四足为主，也包含轮式）。示例以 Lite3 四足为主。

---

## 1. 总体结构（从大到小）

项目整体可以理解为三层：

1. **仿真与任务框架层（Isaac Sim / Isaac Lab）**
   - 提供仿真、物理、传感器、环境管理框架。
2. **任务抽象层（本项目的任务模板）**
   - 定义“通用的四足速度跟踪任务”。
3. **机器人具体实现层（某个具体机器人配置）**
   - 例如 Lite3，替换机器人资产、调整观测/奖励/动作参数。

简化为：

```
Isaac Lab（框架）
   └── 通用任务配置（velocity_env_cfg.py）
         └── 具体机器人配置（rough_env_cfg.py / flat_env_cfg.py）
               └── 训练脚本（train.py）
```

---

## 2. 主要目录与职责

### 2.1 本项目源码

- `source/rl_training/rl_training/`
  - Python 包根目录。

- `source/rl_training/rl_training/tasks/`
  - 所有任务的注册与实现。

- `source/rl_training/rl_training/tasks/manager_based/locomotion/velocity/`
  - “速度跟踪”任务的通用配置。
  - **核心配置文件**：
    - `velocity_env_cfg.py`
  - **局部 MDP 组件（观测/奖励/事件等）**：
    - `mdp/observations.py`
    - `mdp/rewards.py`
    - `mdp/events.py`
    - `mdp/terminations.py`

- `source/rl_training/rl_training/tasks/manager_based/locomotion/velocity/config/`
  - 某个具体机器人/场景的配置，比如 Lite3、M20。

- `scripts/reinforcement_learning/rsl_rl/`
  - 训练与回放脚本：`train.py` / `play.py`。

### 2.2 外部依赖（不在本仓库）

- **Isaac Sim**：物理仿真。
- **Isaac Lab**：环境管理框架（ManagerBasedRLEnv 等）。
- **rsl_rl**：训练算法框架（PPO 等）。

---

## 3. 抽象关系：从“框架”到“具体机器人”

### 3.1 框架层（Isaac Lab）

核心类：
- `ManagerBasedRLEnvCfg`：环境配置基类。
- `ManagerBasedRLEnv`：运行时环境实例。

这层提供统一的“环境管理接口”：
- Scene（场景）
- Observations（观测）
- Actions（动作）
- Rewards（奖励）
- Terminations（终止）
- Events（事件随机化）
- Curriculum（课程学习）

### 3.2 通用任务层（本项目的模板）

文件：
- `source/rl_training/rl_training/tasks/manager_based/locomotion/velocity/velocity_env_cfg.py`

这个文件的作用：
- 定义一个“通用的速度跟踪任务”。
- 只关心任务逻辑，不关心“具体机器人是谁”。

在这里完成的内容：
- 定义场景：地形、传感器、光源。
- 定义动作：关节位置动作。
- 定义观测：速度、姿态、关节信息等。
- 定义奖励：跟踪速度、能耗、稳定性等。
- 定义终止条件：摔倒、碰撞等。

### 3.3 具体机器人层（Lite3 / M20）

例如：
- `.../config/quadruped/deeprobotics_lite3/rough_env_cfg.py`

这个文件的作用：
- 继承通用任务配置。
- 把通用配置变成“某个具体机器人可用”的版本。

典型操作：
- 替换机器人资产（URDF/USD）。
- 修改关节名称、脚名称。
- 调整动作缩放、奖励权重。
- 关闭/打开特定随机化事件。

---

## 4. MDP 组件的抽象

MDP（马尔可夫决策过程）里最重要的组件就是“观测/动作/奖励/终止”。

项目的抽象方式：

1. **通用 MDP 定义**：
   - `velocity_env_cfg.py` 里定义结构。

2. **具体计算函数在 mdp 子模块**：
   - 例如 `mdp/observations.py` 定义 `base_lin_vel` 具体怎么算。

3. **机器人配置会替换或覆盖**：
   - 在 `rough_env_cfg.py` 中修改某些 `func` 或参数。

通俗理解：
- **配置类负责“装配”**，
- **mdp 函数负责“计算”**。

---

## 5. 任务注册机制（为什么 `--task` 能用）

- `source/rl_training/rl_training/tasks/__init__.py` 会自动导入子包。
- 子包的 `__init__.py` 里调用 `gym.register` 注册任务 ID。

例如 Lite3：
- 任务 ID：`Rough-Deeprobotics-Lite3-v0`
- 对应 entry point：
  - 环境配置：`rough_env_cfg:DeeproboticsLite3RoughEnvCfg`
  - 算法配置：`rsl_rl_ppo_cfg:DeeproboticsLite3RoughPPORunnerCfg`

---

## 6. 训练流程（从命令到学习）

当你运行：
```
python scripts/reinforcement_learning/rsl_rl/train.py --task=Rough-Deeprobotics-Lite3-v0
```
执行流程：

1. **启动 Isaac Sim**
2. **Hydra 根据 task 找到 env_cfg / agent_cfg**
3. **创建环境（gym.make）**
4. **包装为 rsl_rl 可用格式**
5. **OnPolicyRunner 开始训练**
6. **日志和模型保存到 logs/**

---

## 7. 回放流程（play.py）

回放不训练，只加载模型：

1. 启动 Isaac Sim。
2. 创建环境。
3. 加载 checkpoint。
4. 循环 `policy(obs)` 生成动作。

---

## 8. 新手最关心的“改哪里”

- **改奖励权重**：`rough_env_cfg.py` 里的 `self.rewards.xxx.weight`
- **改观测内容**：`velocity_env_cfg.py` 或 `mdp/observations.py`
- **改机器人模型**：`rough_env_cfg.py` 中的 `self.scene.robot`
- **改动作幅度**：`self.actions.joint_pos.scale`

---

## 9. 总结：抽象关系一句话

> Isaac Lab 提供框架，本项目提供通用任务模板，再由具体机器人配置把模板“填完整”，最终由训练脚本驱动 RL 训练。

---

如果你希望更细的文档（比如逐行解释 `velocity_env_cfg.py`），告诉我你想从哪部分开始。
