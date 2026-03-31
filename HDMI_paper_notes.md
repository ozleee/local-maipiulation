# HDMI: Learning Interactive Humanoid Whole-Body Control from Human Videos

- **arXiv**: [2509.16757](https://arxiv.org/abs/2509.16757)
- **项目主页**: https://hdmi-humanoid.github.io/
- **代码仓库**: https://github.com/LeCAR-Lab/HDMI
- **作者**: Haoyang Weng, Yitang Li, Nikhil Sobanbabu, Zihan Wang, Zhengyi Luo, Tairan He, Deva Ramanan, Guanya Shi（CMU Robotics Institute）

---

## 一、研究背景与动机

全身交互控制（Whole-Body Interactive Control）要求人形机器人用全身（包括手臂、腿、躯干）同时完成运动与操作任务（Loco-Manipulation）。现有方法面临以下挑战：

1. **数据稀缺**：高质量动作捕捉（MoCap）数据成本高昂，难以规模化获取。
2. **接触复杂**：人机-物体接触丰富，力学关系难以建模。
3. **泛化性差**：大多数方法针对特定任务设计，难以推广。

HDMI 的核心思想：**直接从普通单目 RGB 视频中学习全身交互技能**，无需专用动作捕捉设备，实现仿真训练后零样本（Zero-Shot）迁移到真实机器人。

---

## 二、整体框架（Pipeline）

```
普通人类视频 (Monocular RGB Video)
        │
        ▼
【阶段 1】数据提取与重定向（Data Extraction & Retargeting）
  ├── 人体姿态估计（GVHMR 等）
  ├── 物体轨迹提取
  └── 运动重定向 → 结构化机器人运动数据集
        │
        ▼
【阶段 2】强化学习策略训练（RL Policy Training in Simulation）
  ├── 统一物体表征（Unified Object Representation）
  ├── 残差动作空间（Residual Action Space）
  └── 通用交互奖励（General Interaction Reward）
        │
        ▼
【阶段 3】零样本仿真到真实迁移（Zero-Shot Sim-to-Real Transfer）
  └── 部署到真实人形机器人（如 Unitree G1）
```

---

## 三、各模块详解

### 3.1 数据提取与重定向（Stage 1）

**目标**：将无约束人类视频转化为机器人可用的结构化参考轨迹。

- **人体姿态估计**：使用 GVHMR 等方法从视频逐帧估计人体的全身姿态（位置、朝向、关节角度）。
- **物体轨迹提取**：同步提取与人交互的物体的位置和旋转轨迹。
- **运动重定向（Retargeting）**：将人体运动映射到机器人的关节空间，解决人机形体差异。
- **数据格式**：
  - Body States: `[T, B_robot + B_object, feature_dims]`（T = 时间步，B = body 数量）
  - 包含位置、朝向、线速度、角速度及关节角度/速度
  - 元信息存储于 `meta.json`（记录 body/joint 顺序）

---

### 3.2 强化学习策略训练（Stage 2）

**目标**：训练能同时跟踪机器人与物体状态的 RL 策略。

训练算法：**PPO（Proximal Policy Optimization）**，在 Isaac Sim 仿真器中训练。

#### 3.2.1 统一物体表征（Unified Object Representation）
- 无论物体类型（门、箱子、球等）或配置，均以统一的方式建模。
- 物体状态（位置、旋转、速度）直接拼接到机器人 Body 状态中，形成统一观测向量。
- 优势：**同一策略网络可泛化到多种交互任务**，无需为每类任务单独设计。

#### 3.2.2 残差动作空间（Residual Action Space）
- 策略不直接输出绝对关节角度命令，而是输出**相对于参考轨迹的残差动作**。
- 公式：`a_final = a_ref + Δa_policy`
- 优势：
  - 视频提取的参考轨迹不完美时，策略可自适应修正。
  - 减小策略的探索空间，提升样本效率和训练稳定性。

#### 3.2.3 通用交互奖励（General Interaction Reward）
- 奖励由两部分组成：
  1. **运动跟踪奖励**：惩罚机器人姿态与参考轨迹的偏差（位置、速度、关节角度）。
  2. **交互质量奖励**：奖励合理的接触行为（如手与物体的接触），惩罚不期望的碰撞。
- 无需为每个任务手工设计奖励函数，具有很强的通用性。

#### 3.2.4 策略网络输入/输出
| 输入（Observation） | 说明 |
|---|---|
| 本体感知状态 | 关节角度、速度、IMU 数据 |
| 物体状态 | 物体在机器人局部坐标系中的位置和朝向 |
| 相位变量 | 用于捕捉周期性运动（如步态） |

| 输出（Action） | 说明 |
|---|---|
| 残差关节角度增量 | 加到参考轨迹关节角度上，形成最终控制命令 |

---

### 3.3 仿真到真实迁移（Stage 3）

- **零样本部署**：训练好的策略无需任何真实数据微调，直接部署到真实机器人（Unitree G1）。
- 关键设计保证迁移性：
  - 局部坐标系观测（减少域偏移）
  - 统一的物体表征（泛化到真实物体）
  - 残差动作空间（提升对噪声的鲁棒性）

---

## 四、实验结果

| 任务类型 | 结果 |
|---|---|
| 穿越门（Door Traversal） | 连续成功 **67 次** |
| 仿真中任务数 | **14 种**不同的 Loco-Manipulation 任务 |
| 真实机器人任务数 | **6 种**任务成功部署 |

主要对比基线：Motion Imitation、task-specific RL 方法。HDMI 在泛化性和成功率上均优于基线。

---

## 五、主要创新点总结

| 创新点 | 描述 |
|---|---|
| 视频驱动的数据获取 | 无需 MoCap，从普通视频提取运动数据 |
| 统一物体表征 | 单一策略处理多类物体交互 |
| 残差动作空间 | 提升参考跟踪的鲁棒性和样本效率 |
| 通用交互奖励 | 无需任务级奖励工程 |
| 零样本 Sim-to-Real | 仿真训练直接迁移真实机器人 |

---

## 六、相关工作关联

| 方向 | 代表工作 |
|---|---|
| 人形运动模仿 | PHC, AMP, ASE |
| 人-物交互学习 | OMNIH2O, HumanPlus |
| Sim-to-Real | Domain Randomization, RMA |
| 视频运动提取 | GVHMR, SLAHMR |

---

## 七、局限性与未来方向

- 依赖视频中物体轨迹提取的精度
- 当前物体表征对形状变化泛化能力有限
- 未来可结合语言/视觉条件实现更灵活的任务指定
