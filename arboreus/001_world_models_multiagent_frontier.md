# 世界模型与多智能体模拟前沿追踪（2026年中）

> 预研方向：建木 Arboreus 世界引擎
> 追踪时间：2026-09-06
> 关注焦点：多人世界模型（Multiplayer World Models）、多智能体生成式模拟、权威共享状态架构

---

## 一、领域概览：世界模型正在从"单智能体视频生成"走向"多人共享世界"

2026 年是世界模型（World Models）的关键转折点。此前的工作（GameNGen、Genie 系列、Waypoint 等）主要解决**单智能体**条件下的视频帧预测——给定一个动作流，生成下一帧画面。但真实游戏和模拟世界本质上是**多智能体共享环境**：多个玩家、NPC、机器人同时行动，彼此的行为通过同一个世界状态耦合。

2026 年上半年集中出现了一批直接瞄准"多人共享世界"的工作，核心矛盾是：**如何在多个观察者之间保持世界状态的一致性，同时不让计算量随玩家数爆炸**。两条技术路线正在收敛：

| 路线 | 代表工作 | 核心思想 |
|------|----------|----------|
| 权威状态解耦 | MASS (2026.08) | 学习一个 Logic Engine 推进全局类型化状态，Rendering Engine 按需渲染各视角 |
| 智能体对称编码 | Gamma-World (2026.05) | 用单纯形旋转编码表示智能体身份，稀疏 Hub 注意力降低跨智能体通信成本 |

这两条路线并非互斥——MASS 解决"状态一致性"，Gamma-World 解决"智能体可扩展性"，二者结合是自然方向。

---

## 二、MASS：多人世界模型的权威共享状态架构

### 2.1 核心洞察

MASS（Multiplayer world models with Authoritative Shared State）的出发点极其直接：**多人在线游戏已经解决了这个问题**。传统网游架构中，权威服务器推进唯一的规范游戏状态，客户端接收版本化快照、在更新间隙做本地预测、渲染各自摄像机。世界只模拟一次，与渲染它的客户端数量无关。

MASS 把这个架构契约搬到了学习式世界模型中。

### 2.2 架构拆解

```
┌─────────────────────────────────────────────────┐
│              Authoritative Server               │
│  ┌──────────┐   joint actions   ┌───────────┐  │
│  │  Schema  │ ────────────────► │Logic Engine│  │
│  │ (typed   │                   │(Transformer│  │
│  │  entities)│                   │  advances  │  │
│  └──────────┘                   │ typed state)│  │
│       ▲                         └─────┬─────┘  │
│       │                               │        │
│       │                    versioned snapshot  │
│       │                               │        │
└───────┼───────────────────────────────┼────────┘
        │                               │
        │              ┌────────────────▼───────────────┐
        │              │         Client (per player)     │
        │              │  ┌──────────────────────────┐   │
        │              │  │ Rendering Engine         │   │
        │              │  │ (camera-conditioned      │   │
        │              │  │  frame synthesis)        │   │
        │              │  └──────────────────────────┘   │
        │              │  Local prediction during stall  │
        │              └─────────────────────────────────┘
        │
   declarative game definition
```

**三个核心组件：**

1. **World State Tokenizer + Schema**
   - 每个游戏通过一个声明式 Schema 文件定义实体类型、字段、取值范围、实例数量
   - 示例（Snake N=1024）：1024 条 snake 记录 + 4096 个食物空间桶
   - 状态被序列化为固定宽度的类型化记录序列，可直接检查、可版本化、可作为同步消息
   - **关键：不需要手写任何转移函数**，模型从轨迹中学习

2. **Logic Engine**
   - 唯一推进共享状态的学习组件，decoder-only Transformer
   - 记录级因子化分解：每条记录独立预测，自注意力限制在记录内部
   - 跨实体交互通过"邻域窗口"（neighborhood window）注入——从共享状态计算每条记录周围的上下文
   - Schema 派生的掩码限制每个输出位置的合法取值，确定性选择器强制执行跨记录约束（排序、唯一性）
   - 支持 1024 实体世界因为记录间无注意力，可自由批处理

3. **Rendering Engine**
   - 从预测状态 + 客户端摄像机生成画面
   - 所有视图条件于同一个权威状态，一致性由构造保证
   - 渲染按需执行，模拟工作量与观众数解耦

### 2.3 关键实验结果

- **状态恢复率**：MASS 0.76 vs 最强视频基线 0.128（匹配的多人 Snake benchmark）
- **规模**：1024 并发玩家实体，10000 个循环步（recurrent ticks），结构有效
- **跨视图一致性**：LPIPS 0.098，显著优于 MultiWorld (0.277) 和 per-view 基线 (0.397)
- **客户端预测**：网络中断时，客户端用同一个 Logic Engine 本地推进最新权威状态，不会产生本地玩家位移

### 2.4 与战策/Arboreus 的架构同构性

MASS 的设计与战策当前的**服务端权威架构**几乎是同构的：

| 战策 Battleplan | MASS |
|-----------------|------|
| 服务端权威状态 | Logic Engine 推进的 typed state |
| 客户端快照同步 | versioned typed snapshot |
| 客户端预测/回滚 | stall 期间本地 Logic Engine 预测 |
| 各玩家视角渲染 | Rendering Engine 按需渲染 |
| 游戏逻辑代码 | 学习到的转移（无手写规则） |

**核心差异**：战策的状态转移是手写的游戏逻辑代码，MASS 的转移是学习到的。这不是替代关系——对于竞技 RTS，确定性和可平衡性要求手写逻辑；但 MASS 提示了一个方向：**非核心玩法的环境模拟（天气、生态、平民行为、叙事事件）可以用学习式状态推进来降低开发成本**。

---

## 三、Gamma-World：多智能体世界模型的对称编码与高效注意力

### 3.1 核心问题

单智能体世界模型扩展到多智能体时，两个工程问题突出：
1. **智能体身份表示**：slot-based 方法需要固定的智能体顺序和每个 slot 的学习参数，无法泛化到训练时未见的玩家数
2. **跨智能体注意力**：dense attention 是 O(n²)，玩家多时不可接受

### 3.2 两个关键创新

**Simplex Rotary Agent Encoding（单纯形旋转智能体编码）**
- 3D RoPE 的无参数扩展，将智能体表示为旋转角空间中正单纯形的顶点
- 每个智能体获得不同的相位，但所有智能体**置换等价**（permutation-equivalent）
- 不需要学习每个 slot 的身份参数，不需要固定顺序
- 效果：训练时只用 2 玩家，零样本泛化到 4 玩家

**Sparse Hub Attention（稀疏 Hub 注意力）**
- 可学习的 hub token 作为智能体间通信的中介
- 跨智能体注意力成本从 O(n²) 降到 O(n)
- 每个智能体只与 hub token 交互，hub token 聚合全局信息后再分发

### 3.3 训练与推理

- 双向多智能体教师 → 块因果学生的蒸馏
- 最终因果模型可用 KV caching 做流式推理
- 达到 **24 FPS** 实时动作响应滚动

### 3.4 对 Arboreus 的启示

Arboreus 的 SoulPerceptionSystem 每 tick 为每个灵魂生成 PerceptionFrame，当前是基于规则的感知过滤（FOV 锥、声音衰减、注意力权重）。Gamma-World 提示：

- **智能体身份的对称表示**可以用于灵魂感知系统中的"他者表征"——不同灵魂不应有固定的 slot 顺序，而应通过相对关系编码
- **Hub 注意力机制**可以作为大规模灵魂社会模拟的通信骨架——当世界中有数百个灵魂时，全连接社交计算不可行，hub 结构（如聚落、派系、信息中介）是自然的稀疏化方式

---

## 四、其他值得关注的工作

### 4.1 Solaris（2026.02）：第一个多人世界模型

- Checkpointed Self Forcing——内存高效的 self-forcing 变体
- Minecraft 中多智能体一致同步视图
- 证明了多人世界模型的可行性，但视图一致性仍依赖共享视觉潜变量

### 4.2 Waypoint-1（2026.01）：实时交互式世界生成

- Self-forcing + DMD 蒸馏
- 从 10K 小时游戏数据从头训练
- 实时交互生成，证明了世界模型可以达到游戏级帧率

### 4.3 Emergence World（2026.06）：长时程多智能体自治评估

- 持续运行的多智能体模拟平台
- 关注数周到数月尺度上才显现的动态：行为漂移、不同环境下的治理、不同模型家族智能体间的交叉影响
- **对凌栖的启示**：灵魂系统的长期行为漂移、跨世界适应、灵魂间的长期社会影响，需要长时程评估框架

### 4.4 Qwen-AgentWorld（2026.06）：语言世界模型

- 预测"环境对智能体动作的响应"，而非决定"采取什么动作"
- 35B 和 397B 两个开放权重模型
- 证明了 LLM 可以作为世界状态预测器，而非仅作为决策者
- **对 Ember+Arboreus 桥接的启示**：灵魂引擎（决策）与世界引擎（状态预测）的分离，与 Qwen-AgentWorld 的"模拟先于行动"范式一致

### 4.5 ReactiveGWM（2026）：反应式游戏世界模型

- 显式解耦玩家控制与 NPC 行为
- 合成玩家与 NPC 之间的动态交互
- 为策略丰富的 NPC 交互铺路

### 4.6 Aivilization（2025.11，港科大）：大规模社会模拟

- 数万个 AI 智能体在虚拟社会中生活、学习、交易、社交
- 玩家通过 MBTI 定制智能体，通过对话框下达指令
- 自然涌现社会治理结构、经济系统、文化规范
- **对 Arboreus 社会模拟模块的直接参考**

---

## 五、与 Arboreus 当前系统的关系分析

### 5.1 Arboreus M10 感知系统的定位

Arboreus 刚完成 M10 感知系统（SDK v2.6.0），包含：
- 视觉锥 FOV 感知
- 声音距离衰减感知
- 感知过滤与注意力系统
- SoulPerceptionSystem 多模态整合

这是**世界→灵魂**的单向通道。世界模型前沿正在解决的是**世界状态本身的推进与预测**，这是感知系统的上游。

### 5.2 差距与机会

| 维度 | Arboreus 当前 | 世界模型前沿 | 机会 |
|------|--------------|-------------|------|
| 状态推进 | 手写规则系统 | 学习式 Logic Engine | 非核心系统可探索学习式推进 |
| 多智能体一致性 | 服务端权威（战策） | MASS 权威共享状态 | 架构同构，可借鉴状态序列化方案 |
| 智能体扩展性 | 基于规则的社交计算 | Gamma-World 对称编码+Hub注意力 | 大规模灵魂社会模拟的通信骨架 |
| 长时程行为 | 未系统评估 | Emergence World 长时程平台 | 需要建立灵魂长期行为漂移评估 |
| 感知→预测 | 只有感知（现在） | 世界模型预测未来状态 | 灵魂可以"想象"未来世界状态 |

### 5.3 最有价值的交叉点：灵魂的"世界模型"能力

Ember 灵魂引擎的元认知体系（M9 完成）包含 LearningRegulation 和 CognitiveTransfer。如果灵魂不仅能**感知**当前世界状态，还能**预测**世界状态的演化（一个微型世界模型），则：
- 灵魂可以进行反事实推理（"如果我走这条路会怎样"）
- 灵魂可以规划更长的行动序列
- 灵魂之间可以有"心智理论"（预测其他灵魂的行为）
- 这与预测编码/主动推理（第1轮 Ember 预研）形成闭环：灵魂用内部世界模型预测感知，预测误差驱动学习

---

## 六、实施建议（下一阶段）

### 建议1：建立 Arboreus 状态序列化规范（借鉴 MASS Schema）

- 为 Arboreus 的核心实体（灵魂、建筑、资源、生态区、领土）定义类型化状态 Schema
- 状态可序列化为固定宽度记录，支持版本化快照
- 这是后续任何学习式状态推进、客户端预测、状态同步的基础
- **优先级：高**，因为这是架构基础设施，不依赖 AI 模型

### 建议2：探索 Hub 结构的大规模灵魂社交模拟（借鉴 Gamma-World）

- 在 Arboreus social 模块中引入"信息枢纽"实体（聚落中心、消息传递者、派系领袖）
- 灵魂间的社交计算通过 hub 中继，降低全连接复杂度
- 支持数百灵魂规模的社会模拟
- **优先级：中**，可在 M11 或 M12 探索

### 建议3：建立灵魂长时程行为评估框架（借鉴 Emergence World）

- 设计持续运行的灵魂模拟沙盒，追踪数周尺度的行为漂移
- 评估指标：人格稳定性、社会关系演化、跨环境适应、学习迁移
- **优先级：中**，与 Ember 元认知体系的验证直接相关

### 建议4：研究"灵魂内部世界模型"作为 Ember-Arboreus 桥接方向

- 每个灵魂维护一个对世界状态的压缩预测模型
- 预测误差作为感知显著性信号（与预测编码闭环）
- 这是理论方向，需要更多预研后再实施
- **优先级：低（理论预研阶段）**

---

## 七、风险与挑战

1. **学习式状态推进的确定性问题**：竞技游戏需要确定性，学习模型有随机性。MASS 用 Schema 掩码和确定性选择器部分解决，但仍不适合核心战斗逻辑。应限定在环境/叙事/非核心模拟。
2. **计算成本**：世界模型推理需要 GPU，Arboreus 当前是 CPU 型 Node.js/TS 引擎。引入学习式组件需要异构计算架构。
3. **数据需求**：训练 Logic Engine 需要大量游戏轨迹数据，凌栖项目目前没有这个规模的数据。
4. **可解释性**：学习式状态转移难以调试，对于需要精确 bug 定位的开发流程是挑战。

---

## 八、信息来源

- MASS: Multiplayer World Models with Authoritative Shared State, arXiv:2608.06257v2, 2026-08-10, Alaya Lab / Peking University / Institute of Science Tokyo
- Gamma-World: Generative Multi-Agent World Modeling Beyond Two Players, arXiv:2605.28816, 2026-05, NVIDIA / Tsinghua / U Toronto
- Solaris: Checkpointed Self Forcing for Multiplayer World Models, 2026-02
- Waypoint-1: Real-time Interactive World Generation, 2026-01
- Emergence World: A Platform for Evaluating Long-Horizon Multi-Agent Autonomy, arXiv:2606.08367, 2026-06
- Qwen-AgentWorld: Language World Models for Agent Environments, 2026-06
- Aivilization: HKUST large-scale multi-agent social simulation, 2025-11
- From Masks to Worlds: A Hitchhiker's Guide to World Models, arXiv:2510.20668, 2025
- Awesome World Models for Video, Games & 3D (GitHub evolution tracker)

---

## 九、下一步研究方向

1. 深入研究 MASS 的 Schema 定义语言，为 Arboreus 实体设计状态序列化方案
2. 调研世界模型在非游戏场景（机器人、自动驾驶）的状态表示方法
3. 研究"预测编码 + 世界模型"在灵魂认知架构中的整合方案（与 Ember 预研第1轮衔接）
4. 追踪 Genie-3、Mineworld 等 3D 世界模型的最新进展
