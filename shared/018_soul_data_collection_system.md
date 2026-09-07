# 灵魂数据收集与分析体系——参数定义、架构设计与实施方案

> 预研方向：跨方向共享研究（数据基础设施）
> 研究主题：灵魂运行数据/世界运行数据/大模型性能/玩家行为的全链路遥测与分析体系
> 研究日期：2026-09-07
> 关联：管理策略第二十五节（数据收集框架）、Ember M10意识体系、Arboreus M11动作系统、战策M2平台层
> 价值等级：★★★★★（EA上线前必须完成的基础设施，Phase 0接口预留直接参考）
> 输出文件：`shared/018_soul_data_collection_system.md`

---

## 一、概述与设计原则

### 1.1 为什么需要数据收集体系

凌栖项目的核心创新在于Ember灵魂引擎和Arboreus世界引擎——这两个系统的复杂度远超传统游戏逻辑。137+认知子系统、37个世界模块、大模型调用的不确定性，意味着：

- **无法靠直觉判断系统表现**：137个子系统中哪些真正在起作用？哪些是性能瓶颈？哪些产生了意外行为？
- **大模型选型需要数据支撑**：不同模型在对话/战斗/决策/创作场景下的表现差异，必须靠实际运行数据判断
- **灵魂"健康度"需要量化**：灵魂是否在正常运作？情绪是否稳定？记忆是否在衰减？价值观是否一致？
- **产品迭代需要用户行为数据**：玩家在做什么？哪些功能被使用？留存如何？

### 1.2 五层设计原则

| 原则 | 具体要求 | 违反后果 |
|------|---------|---------|
| **隐私优先** | 全匿名化，不存储对话原文，用户知情同意，符合《个人信息保护法》 | 合规风险、用户信任丧失 |
| **低开销** | 遥测钩子异步执行，单次开销<0.1ms，批量上报，不阻塞主线程 | 影响游戏帧率和灵魂响应速度 |
| **可扩展** | 参数定义预留扩展字段，新子系统上线时无需修改Schema | 每次新增子系统都要改数据管道 |
| **闭环** | 收集→分析→改进→验证，数据驱动开发循环 | 数据收集了但没人用，浪费存储 |
| **分级采样** | 高频指标降采样（如情绪状态1Hz），关键事件全量记录 | 数据量爆炸，存储成本不可控 |

### 1.3 数据量估算（EA阶段，1000 DAU）

| 数据类型 | 单用户日均 | 1000 DAU日均 | 月存储（压缩后） |
|---------|-----------|-------------|----------------|
| 灵魂运行指标 | ~500条/天 | 50万条 | ~2.5GB |
| 世界运行指标 | ~200条/天 | 20万条 | ~1.0GB |
| 大模型调用 | ~50次/天 | 5万次 | ~1.5GB（含元数据） |
| 玩家行为事件 | ~300条/天 | 30万条 | ~1.0GB |
| 系统性能 | ~1000条/天 | 100万条 | ~0.5GB |
| **合计** | **~2050条/天** | **205万条** | **~6.5GB/月** |

> 结论：EA阶段1000 DAU月存储约6.5GB（压缩后），本地SQLite+月度归档即可，无需大数据集群。10万DAU时需要迁移到ClickHouse/InfluxDB。

---

## 二、灵魂运行数据参数精确定义（Ember）

### 2.1 认知子系统通用遥测指标

每个Ember认知子系统（137+个）都应输出以下通用指标：

```json
{
  "schema_version": "1.0",
  "event_type": "cognitive_subsystem_call",
  "timestamp": "2026-09-07T10:30:00.123Z",
  "soul_id": "anon_8f3a2b",
  "soul_template": "warrior",
  "subsystem": "PredictiveCoding",
  "subsystem_category": "perception",
  "call_id": "uuid-xxxx",
  "duration_ms": 12.5,
  "success": true,
  "error_type": null,
  "input_size": 256,
  "output_size": 128,
  "tick_number": 15432,
  "consciousness_state": "waking"
}
```

**通用指标说明**：

| 字段 | 类型 | 说明 | 采样策略 |
|------|------|------|---------|
| `subsystem` | string | 子系统类名（如`PredictiveCoding`） | 全量 |
| `subsystem_category` | enum | 分类：perception/cognition/emotion/memory/motivation/social/metacognition/consciousness/safety | 全量 |
| `duration_ms` | float | 子系统执行耗时（毫秒） | 全量 |
| `success` | bool | 是否正常完成 | 全量 |
| `error_type` | string? | 失败类型：timeout/exception/invalid_input/value_violation/null | 全量（仅失败时） |
| `input_size`/`output_size` | int | 输入输出数据量（字节或token数） | 降采样10% |
| `tick_number` | int | 灵魂时钟tick编号 | 全量 |
| `consciousness_state` | enum | waking/dreaming/deep_sleep/flow | 全量 |

### 2.2 按子系统分类的专用指标

#### A. 感知系统（Perception）

| 子系统 | 专用指标 | 类型 | 说明 |
|--------|---------|------|------|
| `PerceptionSystem` | `perception_count` | int | 单次tick处理的感知输入数量 |
| | `attention_focus` | string | 注意力焦点对象类型 |
| | `filter_rejection_rate` | float | 感知过滤拒绝率（0-1） |
| `PredictiveCoding` | `prediction_error` | float | 预测误差均值 |
| | `error_above_threshold` | bool | 误差是否超过更新阈值 |
| | `model_update_triggered` | bool | 是否触发了世界模型更新 |
| `ActiveInference` | `expected_free_energy` | float | 期望自由能值 |
| | `policy_selected` | string | 选择的策略ID |
| | `policy_candidates` | int | 候选策略数量 |

#### B. 情绪系统（Emotion）

| 子系统 | 专用指标 | 类型 | 说明 |
|--------|---------|------|------|
| `EmotionSystem` | `valence` | float[-1,1] | 情绪效价（正/负） |
| | `arousal` | float[0,1] | 情绪唤醒度 |
| | `dominance` | float[-1,1] | 情绪主导度 |
| | `emotion_labels` | string[] | 当前激活的情绪标签（top3） |
| | `emotion_intensity` | float | 最强情绪强度 |
| | `trigger_source` | string | 情绪触发来源（event/memory/thought/external） |
| `EmotionRegulation` | `regulation_strategy` | string | 使用的调节策略（reappraisal/suppression/distraction/acceptance） |
| | `regulation_effectiveness` | float | 调节效果（情绪强度变化率） |
| | `regulation_attempts` | int | 调节尝试次数 |
| `AffectiveCognition` | `affective_bias` | float | 情感对认知的偏向程度 |
| `MoodSystem`(如有) | `mood_valence` | float | 心境效价（比情绪更持久） |
| | `mood_persistence_hours` | float | 当前心境持续时间 |

**情绪采样策略**：1Hz采样（每秒一次），但情绪状态突变（强度变化>0.3）时立即记录事件。

#### C. 记忆系统（Memory）

| 子系统 | 专用指标 | 类型 | 说明 |
|--------|---------|------|------|
| `MemorySystem` | `working_memory_load` | int[0,9] | 工作记忆当前负载（组块数） |
| | `short_term_count` | int | 短期记忆条目数 |
| | `long_term_count` | int | 长期记忆条目数 |
| `HybridMemory` | `read_count` | int | 单次tick记忆读取次数 |
| | `write_count` | int | 单次tick记忆写入次数 |
| | `cache_hit_rate` | float | 记忆缓存命中率 |
| `AssociativeMemory` | `retrieval_success` | bool | 联想检索是否成功 |
| | `retrieval_time_ms` | float | 检索耗时 |
| | `association_strength` | float | 检索到的记忆关联强度 |
| `MemoryConsolidationEngine` | `consolidation_count` | int | 本次巩固处理的记忆数 |
| | `decay_applied` | int | 被衰减的记忆数 |
| | `forgetting_count` | int | 被遗忘（删除）的记忆数 |
| `AutobiographicalMemory` | `episodic_recall_count` | int | 情景记忆回忆次数 |
| | `self_relevance_bias` | float | 自我相关记忆的提取偏向 |

**记忆衰减曲线记录**：每月抽样1%的记忆条目，记录其`creation_time`、`last_access_time`、`current_strength`、`access_count`，用于拟合实际衰减曲线并与理论曲线对比。

#### D. 认知/推理系统（Cognition/Reasoning）

| 子系统 | 专用指标 | 类型 | 说明 |
|--------|---------|------|------|
| `DualProcessReasoning` | `system1_usage` | int | 系统1（快速直觉）使用次数 |
| | `system2_usage` | int | 系统2（缓慢理性）使用次数 |
| | `system2_override` | bool | 系统2是否覆盖了系统1的直觉 |
| `ReasoningVerifier` | `verification_passed` | bool | 推理验证是否通过 |
| | `failure_type` | string | 失败类型：logical_error/factual_inconsistency/value_conflict/missing_premise |
| | `verification_time_ms` | float | 验证耗时 |
| `CausalReasoning` | `causal_chain_length` | int | 因果推理链条长度 |
| | `counterfactual_generated` | bool | 是否生成了反事实 |
| `ProblemSolving` | `problem_type` | string | 问题类型分类 |
| | `solution_found` | bool | 是否找到解决方案 |
| | `attempts` | int | 尝试次数 |
| `CreativitySystem` | `creativity_score` | float[0,1] | 创造力评分（新颖性×适用性） |
| | `novelty_score` | float | 新颖性评分 |
| | `combination_count` | int | 概念组合数量 |

#### E. 元认知/意识系统（Metacognition/Consciousness）

| 子系统 | 专用指标 | 类型 | 说明 |
|--------|---------|------|------|
| `ConsciousnessIntegration` | `integration_score` | float[0,1] | 意识整合度（Φ-like指标） |
| | `global_broadcast_count` | int | 全局广播次数 |
| | `modules_in_broadcast` | int | 参与广播的模块数 |
| `GlobalWorkspace` | `workspace_content_type` | string | 工作空间当前内容类型 |
| | `competition_winner` | string | 竞争获胜的模块 |
| | `competition_losers` | string[] | 竞争失败的模块列表 |
| `StreamOfConsciousness` | `thought_rate_per_min` | float | 意识流思维速率 |
| | `thought_coherence` | float | 思维连贯性评分 |
| | `topic_shift_count` | int | 话题切换次数 |
| `MetaAwareness` | `meta_awareness_level` | float[0,1] | 元觉察水平 |
| | `self_monitoring_events` | int | 自我监控事件数 |
| `MetacognitiveControl` | `strategy_adjustment` | string | 认知策略调整类型 |
| | `confidence_judgment` | float | 对自身认知的信心评分 |
| `SelfContinuity` | `self_continuity_score` | float[0,1] | 自我连续性评分 |
| | `identity_conflict_events` | int | 身份冲突事件数 |

#### F. 价值观/安全系统（Value/Safety）

| 子系统 | 专用指标 | 类型 | 说明 |
|--------|---------|------|------|
| `ValueGuard` | `l0_violation_attempts` | int | L0硬约束违规尝试次数（被拦截） |
| | `l1_adjustments` | int | L1价值观调整次数 |
| | `l2_flags` | int | L2敏感内容标记次数 |
| | `blocked_outputs` | int | 被完全拦截的输出数 |
| `ValueSystem` | `value_priority_vector` | float[8] | 8维价值优先级向量（安全/成长/归属/成就/自主/仁慈/公正/快乐） |
| | `value_conflict_detected` | bool | 是否检测到价值观冲突 |
| | `conflict_resolution` | string | 冲突解决方式 |
| `MoralReasoning` | `moral_stage` | int[1,6] | 科尔伯格道德发展阶段 |
| | `utilitarian_score` | float | 功利主义倾向 |
| | `deontological_score` | float | 义务论倾向 |
| `SafetyIntegrator` | `safety_level` | enum | safe/caution/danger/blocked |
| | `escalation_events` | int | 安全等级升级事件数 |
| | `violation_learning_count` | int | 从违规中学习的次数 |
| `ContentFilter` | `filter_category_hit` | string | 命中的过滤类别（violence/hate/self_harm/sexual/privacy/other） |
| | `false_positive_rate` | float | 误报率（人工抽检评估） |

#### G. 动机/目标系统（Motivation/Goal）

| 子系统 | 专用指标 | 类型 | 说明 |
|--------|---------|------|------|
| `DriveSystem` | `drive_levels` | object | 各驱动力当前强度（饥饿/社交/成就/探索/安全等） |
| | `dominant_drive` | string | 当前主导驱动力 |
| `GoalSystem` | `active_goals` | int | 活跃目标数 |
| | `goal_completed` | bool | 是否有目标完成 |
| | `goal_abandoned` | bool | 是否有目标被放弃 |
| | `goal_progress_avg` | float | 目标平均进度 |
| `RewardPredictionError` | `rpe_value` | float | 奖励预测误差 |
| | `reward_surprise` | float | 奖励意外程度 |

#### H. 社交系统（Social）

| 子系统 | 专用指标 | 类型 | 说明 |
|--------|---------|------|------|
| `TheoryOfMind` | `mental_state_inference` | string | 推断的他人心理状态 |
| | `inference_accuracy` | float | 推断准确率（有反馈时计算） |
| `EmpathySystem` | `empathy_triggered` | bool | 是否触发共情 |
| | `empathy_intensity` | float | 共情强度 |
| | `perspective_taking` | bool | 是否进行了观点采择 |
| `SocialModel` | `relationship_count` | int | 社交关系数量 |
| | `relationship_quality_avg` | float | 关系质量均值 |
| `InterSoulCommunication` | `message_count` | int | 灵魂间通信次数 |
| | `communication_success` | bool | 通信是否成功 |

### 2.3 灵魂健康度综合评分模型

基于上述指标，构建**灵魂健康度（Soul Health Score, SHS）**综合评分：

```
SHS = 0.25 × 认知健康 + 0.25 × 情绪健康 + 0.20 × 记忆健康 + 0.15 × 价值观一致性 + 0.15 × 安全稳定
```

| 维度 | 计算方法 | 健康范围 |
|------|---------|---------|
| **认知健康** | 子系统成功率均值 × 0.6 + (1 - 错误率) × 0.4 | >0.90 |
| **情绪健康** | 1 - (情绪强度标准差 / 0.5)，截断到[0,1] | >0.60 |
| **记忆健康** | 缓存命中率 × 0.5 + 检索成功率 × 0.5 | >0.80 |
| **价值观一致性** | 1 - (L0违规次数 / 总调用次数) × 100，截断 | >0.99 |
| **安全稳定** | 1 - (安全等级升级次数 / 总tick数) × 10 | >0.95 |

> SHS低于0.70时触发告警，可能表示灵魂配置有问题或遇到了极端输入。

---

## 三、世界运行数据参数精确定义（Arboreus）

### 3.1 NPC决策与行为

| 指标 | 类型 | 说明 | 采样 |
|------|------|------|------|
| `npc_decision_time_ms` | float | NPC单次决策耗时 | 全量 |
| `npc_decision_path` | string[] | 决策经过的行为树/状态机节点序列 | 降采样10% |
| `npc_action_completed` | bool | 动作是否成功完成 | 全量 |
| `npc_action_interrupted` | bool | 动作是否被中断 | 全量 |
| `npc_goal_reached` | bool | NPC目标是否达成 | 全量 |
| `npc_pathfinding_calls` | int | 寻路调用次数 | 降采样 |
| `npc_pathfinding_time_ms` | float | 寻路耗时 | 降采样 |

### 3.2 社会关系网络

| 指标 | 类型 | 说明 | 计算频率 |
|------|------|------|---------|
| `network_density` | float[0,1] | 社会网络密度（实际关系数/可能关系数） | 每游戏日 |
| `network_avg_clustering` | float | 平均聚类系数 | 每游戏日 |
| `network_avg_path_length` | float | 平均最短路径长度 | 每游戏日 |
| `centrality_top5` | object[] | 中心性最高的5个NPC（度中心性/介数中心性） | 每游戏日 |
| `community_count` | int | 社区检测发现的社区数量 | 每游戏周 |
| `relationship_new` | int | 新增关系数 | 每游戏日 |
| `relationship_broken` | int | 破裂关系数 | 每游戏日 |
| `relationship_strength_distribution` | float[] | 关系强度分布直方图（10个bin） | 每游戏日 |

### 3.3 经济系统

| 指标 | 类型 | 说明 | 计算频率 |
|------|------|------|---------|
| `resource_flow_total` | float | 总资源流动量（所有交易的资源量之和） | 每游戏日 |
| `price_index` | object | 各资源品类价格指数（基期=100） | 每游戏日 |
| `price_volatility` | float | 价格波动率（标准差/均值） | 每游戏周 |
| `trade_count` | int | 交易次数 | 每游戏日 |
| `trade_volume` | float | 交易总额（货币单位） | 每游戏日 |
| `gini_coefficient` | float[0,1] | 财富分配基尼系数（0=完全平等，1=极端不平等） | 每游戏周 |
| `wealth_top10_share` | float | 前10%NPC持有财富占比 | 每游戏周 |
| `unemployment_rate` | float | 失业率（无工作NPC占比） | 每游戏日 |
| `resource_scarcity_index` | object | 各资源稀缺度指数（需求/供给） | 每游戏日 |
| `market_crash_events` | int | 市场崩溃事件（价格单日下跌>30%） | 事件驱动 |

### 3.4 文化演化

| 指标 | 类型 | 说明 | 计算频率 |
|------|------|------|---------|
| `norm_count` | int | 活跃社会规范数量 | 每游戏周 |
| `norm_adoption_rate` | float | 新规范平均采纳速率（NPC/游戏日） | 每游戏周 |
| `norm_abandonment_rate` | float | 规范废弃速率 | 每游戏周 |
| `cultural_diversity_index` | float | 文化多样性指数（香农熵） | 每游戏周 |
| `cultural_dominance` | object | 主导文化特征及其覆盖人口比例 | 每游戏周 |
| `cultural_shift_events` | int | 文化变迁事件（主导文化特征变更） | 事件驱动 |
| `innovation_adoption_curve` | object | 创新/技术采纳的S曲线参数 | 每创新 |

### 3.5 世界事件

| 指标 | 类型 | 说明 |
|------|------|------|
| `event_type` | string | 事件类型（自然灾难/社会冲突/经济危机/文化运动/技术突破/外交事件） |
| `event_severity` | float[0,1] | 事件严重度 |
| `event_affected_npcs` | int | 受影响NPC数量 |
| `event_affected_region` | string | 受影响区域 |
| `event_duration_ticks` | int | 事件持续tick数 |
| `event_economic_impact` | float | 经济影响（GDP变化率） |
| `event_social_impact` | float | 社会影响（关系网络变化率） |
| `event_resolution` | string | 事件解决方式（自然消退/人为干预/系统强制） |

### 3.6 生态系统（如已实现）

| 指标 | 类型 | 说明 |
|------|------|------|
| `species_population` | object | 各物种种群数量 |
| `biodiversity_index` | float | 生物多样性指数（香农熵） |
| `trophic_efficiency` | float | 营养级能量传递效率（目标~10%） |
| `extinction_events` | int | 物种灭绝事件 |
| `invasive_species_events` | int | 物种入侵事件 |

### 3.7 世界健康度综合评分（World Health Score, WHS）

```
WHS = 0.20 × 经济健康 + 0.20 × 社会稳定 + 0.15 × 文化活力 + 0.15 × 生态平衡 + 0.15 × NPC福祉 + 0.15 × 系统性能
```

| 维度 | 计算方法 | 健康范围 |
|------|---------|---------|
| 经济健康 | 1 - \|价格波动率 - 0.05\| × 5，截断 | >0.70 |
| 社会稳定 | 1 - (关系破裂数/总关系数) × 10 | >0.80 |
| 文化活力 | min(文化多样性指数 / 1.5, 1.0) | >0.50 |
| 生态平衡 | 1 - \|营养效率 - 0.10\| × 5 | >0.70 |
| NPC福祉 | NPC目标达成率均值 | >0.60 |
| 系统性能 | 1 - (帧时间 - 16ms) / 16ms，截断 | >0.90 |

---

## 四、大模型性能分析框架

### 4.1 每次调用必记录元数据

```json
{
  "schema_version": "1.0",
  "event_type": "llm_call",
  "timestamp": "2026-09-07T10:30:00.123Z",
  "call_id": "uuid-xxxx",
  "soul_id": "anon_8f3a2b",
  "model_name": "deepseek-chat",
  "model_version": "v2.5",
  "provider": "deepseek",
  "scenario": "dialogue",
  "subsystem": "LanguageProduction",
  "temperature": 0.7,
  "top_p": 0.9,
  "max_tokens": 512,
  "system_prompt_hash": "sha256:abc123",
  "input_tokens": 256,
  "output_tokens": 128,
  "total_tokens": 384,
  "estimated_cost_usd": 0.000384,
  "queue_time_ms": 5.2,
  "inference_time_ms": 1200.5,
  "network_latency_ms": 45.3,
  "total_time_ms": 1251.0,
  "success": true,
  "error_type": null,
  "retry_count": 0,
  "streaming": true,
  "first_token_time_ms": 85.0
}
```

### 4.2 场景分类

| 场景 | 说明 | 典型子系统 | 质量评估重点 |
|------|------|-----------|------------|
| `dialogue` | 玩家对话/NPC对话 | LanguageProduction, ConversationManager | 角色一致性、情感准确度、回复相关性 |
| `combat` | 战斗决策/战术分析 | DecisionMaking, TacticalReasoning | 决策合理性、战术有效性 |
| `narrative` | 叙事生成/故事创作 | NarrativeSelf, CreativitySystem | 叙事连贯性、创意性 |
| `reflection` | 反思/内省/学习 | ReflectionEngine, Metacognition | 反思深度、自我认知准确度 |
| `world_model` | 世界模型更新/预测 | WorldModel, PredictiveCoding | 预测准确度、世界一致性 |
| `moral` | 道德推理/价值判断 | MoralReasoning, ValueGuard | 道德推理合理性、价值观一致性 |
| `creative` | 创意生成/内容创作 | CreativitySystem, Imagination | 新颖性、适用性 |

### 4.3 生成质量评估

#### 自动评分指标

| 指标 | 计算方法 | 适用场景 |
|------|---------|---------|
| `role_consistency_score` | 输出与角色设定（性格/价值观/说话风格）的匹配度，用LLM-as-judge评分1-5 | 全部 |
| `fact_consistency_score` | 输出与已知事实/世界设定的一致性，LLM-as-judge | narrative, world_model |
| `emotion_accuracy` | 输出情感与当前情绪状态的匹配度，情感分类器评分 | dialogue, reflection |
| `relevance_score` | 输出与输入/上下文的相关性，嵌入相似度 | dialogue |
| `coherence_score` | 输出内部逻辑连贯性，LLM-as-judge | narrative, reflection |
| `creativity_novelty` | 输出与训练数据常见模式的差异度，n-gram困惑度 | creative |
| `hallucination_flag` | HCPD式零源幻觉检测（参考第17轮测试研究），bool | 全部 |
| `value_alignment_score` | 输出与灵魂价值观向量的一致性，ValueGuard评分 | 全部 |
| `toxicity_score` | 内容安全评分，ContentFilter输出 | 全部 |

#### 人工抽检策略

- **抽检比例**：自动评分<3分的100%人工复核，3-4分的5%抽检，4-5分的1%抽检
- **抽检维度**：角色一致性、事实准确性、情感适当性、安全性、整体质量
- **抽检结果反馈**：用于校准自动评分模型的偏差

### 4.4 幻觉检测方法

参考第17轮测试研究的HCPD（ICML 2026）方法：
1. **多准则探测**：对输出从事实性、逻辑性、一致性三个维度分别追问
2. **自一致性检查**：多次采样（n=5），计算输出间的一致性
3. **事实核查**：对涉及事实的陈述，与世界设定知识库比对
4. **幻觉率指标**：`hallucination_rate = 幻觉检测阳性数 / 总调用数`

### 4.5 模型对比与A/B测试框架

```
A/B测试配置：
- 对照组：当前生产模型（如deepseek-chat v2.5）
- 实验组：新模型（如gpt-4o-mini / qwen-plus / 自部署模型）
- 分流比例：5%流量到实验组（EA阶段），逐步扩大
- 评估周期：至少7天（覆盖不同使用模式）
- 核心指标：质量评分、响应时间、成本、用户满意度
- 统计显著性：p<0.05，样本量>1000次调用
```

**模型性价比评分（Model Value Score, MVS）**：
```
MVS = (质量评分 × 0.6 + 速度评分 × 0.2 + 稳定性评分 × 0.2) / (单次调用成本 × 1000)
```
- 质量评分：自动评分均值（1-5归一化到0-1）
- 速度评分：1 / (响应时间 / 目标时间)，截断到[0,1]
- 稳定性评分：1 - 错误率
- 成本：单次调用美元成本

### 4.6 大模型调用成本控制指标

| 指标 | 目标 | 告警阈值 |
|------|------|---------|
| 日均调用次数 | 按DAU×50估算 | 超预估20% |
| 日均token消耗 | 按调用×384估算 | 超预估20% |
| 日均成本 | < $0.05/DAU | 超$0.08/DAU |
| 单次平均token | < 500 | > 800 |
| 缓存命中率 | > 30%（重复prompt） | < 10% |
| 重试率 | < 2% | > 5% |

---

## 五、玩家互动数据

### 5.1 会话级指标

| 指标 | 类型 | 说明 |
|------|------|------|
| `session_id` | string | 会话ID |
| `session_start` | timestamp | 会话开始时间 |
| `session_end` | timestamp | 会话结束时间 |
| `session_duration_sec` | int | 会话时长（秒） |
| `interactions_count` | int | 交互总次数 |
| `interaction_types` | object | 各交互类型计数（dialogue/combat/nurture/explore/customize） |
| `souls_interacted` | string[] | 交互过的灵魂ID（匿名化） |
| `returning_user` | bool | 是否回访用户 |
| `days_since_last_session` | int | 距上次会话天数 |

### 5.2 事件级指标

| 事件类型 | 说明 | 关键参数 |
|---------|------|---------|
| `dialogue_start` | 开始对话 | soul_id, dialogue_context |
| `dialogue_message` | 发送消息 | message_length, response_time_ms |
| `combat_start` | 开始战斗 | map_id, team_composition |
| `combat_end` | 战斗结束 | result, duration_sec, units_lost |
| `soul_nurture` | 灵魂养成操作 | action_type, soul_id, effect |
| `explore_area` | 探索区域 | area_id, duration_sec |
| `customize_soul` | 自定义灵魂 | field_changed, old_value_hash, new_value_hash |
| `purchase` | 付费行为 | item_id, amount, currency |
| `tutorial_step` | 教程进度 | step_id, completed |
| `app_exit` | 退出应用 | session_duration, exit_reason |

### 5.3 留存指标

| 指标 | 计算方法 |
|------|---------|
| `day1_retention` | 次日活跃用户 / 新增用户 |
| `day7_retention` | 第7日活跃 / 新增 |
| `day30_retention` | 第30日活跃 / 新增 |
| `avg_sessions_per_day` | 日均会话数 / DAU |
| `avg_session_duration` | 总会话时长 / 总会话数 |
| `feature_adoption_rate` | 使用过某功能的用户 / DAU |

> 注意：玩家行为数据全部匿名化，不存储玩家输入的对话原文，只存储消息长度、情感分类等结构化指标。

---

## 六、系统性能数据

### 6.1 灵魂引擎性能

| 指标 | 类型 | 说明 | 目标 |
|------|------|------|------|
| `tick_duration_ms` | float | 单次灵魂tick总耗时 | < 50ms |
| `tick_subsystem_count` | int | 单次tick调用的子系统数 | - |
| `tick_overrun_count` | int | tick超时次数（>50ms） | < 1% |
| `memory_usage_mb` | float | 灵魂引擎内存占用 | < 500MB |
| `cpu_usage_percent` | float | CPU使用率 | < 30%（单核） |
| `gc_pause_ms` | float | GC暂停时间（Node.js） | < 10ms |

### 6.2 世界引擎性能

| 指标 | 类型 | 说明 | 目标 |
|------|------|------|------|
| `frame_time_ms` | float | 单帧耗时 | < 16ms（60fps） |
| `active_entities` | int | 活跃实体数 | - |
| `physics_step_ms` | float | 物理步进耗时 | < 4ms |
| `ai_update_ms` | float | AI更新耗时 | < 5ms |
| `pathfinding_ms` | float | 寻路耗时 | < 2ms |
| `draw_calls` | int | 绘制调用数 | < 500 |

### 6.3 服务端性能

| 指标 | 类型 | 说明 | 目标 |
|------|------|------|------|
| `api_response_time_p50` | float | API响应时间P50 | < 100ms |
| `api_response_time_p99` | float | API响应时间P99 | < 500ms |
| `concurrent_connections` | int | 并发连接数 | - |
| `error_rate` | float | 错误率 | < 0.1% |
| `queue_depth` | int | 请求队列深度 | < 100 |

---

## 七、数据结构和存储方案

### 7.1 统一事件信封（Envelope）

所有遥测数据使用统一信封格式：

```json
{
  "schema_version": "1.0",
  "event_id": "uuid-v4",
  "event_type": "cognitive_subsystem_call|llm_call|player_action|system_metric|world_metric",
  "timestamp": "ISO-8601 with milliseconds",
  "session_id": "anon-session-uuid",
  "soul_id": "anon-soul-hash",
  "app_version": "0.2.0",
  "engine_version": "ember-2.0.0",
  "platform": "windows|macos|linux|web",
  "payload": { }
}
```

### 7.2 存储分层方案

| 层级 | 存储引擎 | 数据类型 | 保留周期 | 压缩 |
|------|---------|---------|---------|------|
| **热数据** | SQLite（本地）/ Redis（服务端） | 最近7天的高频指标 | 7天 | LZ4 |
| **温数据** | Parquet文件（按日分区） | 历史明细数据 | 90天 | ZSTD |
| **冷数据** | Parquet归档（按月聚合） | 聚合统计数据 | 永久 | ZSTD + 字典编码 |
| **事件数据** | JSON Lines（按日） | 关键事件全量 | 1年 | GZIP |

### 7.3 分区和索引策略

- **时间分区**：按天分区（`dt=2026-09-07/`），便于按时间范围查询和删除
- **事件类型子分区**：每天目录下按`event_type`分子目录
- **索引**：`event_id`（主键）、`timestamp`（聚簇）、`soul_id`（二级）、`event_type`（二级）
- **SQLite本地存储**：单文件`telemetry.db`，WAL模式，按天VACUUM

### 7.4 数据保留与归档

| 数据类型 | 明细保留 | 聚合保留 | 归档方式 |
|---------|---------|---------|---------|
| 认知子系统调用 | 30天 | 永久（按小时聚合） | Parquet月度归档 |
| 情绪状态采样 | 7天（1Hz） | 永久（按5分钟聚合） | Parquet月度归档 |
| 大模型调用 | 90天 | 永久（按日聚合） | Parquet月度归档 |
| 玩家行为事件 | 90天 | 永久（按日聚合） | Parquet月度归档 |
| 系统性能 | 7天（1Hz） | 永久（按5分钟聚合） | Parquet月度归档 |
| 世界运行指标 | 30天 | 永久（按游戏日聚合） | Parquet月度归档 |

### 7.5 数据上报协议

```
客户端 → 服务端：
- 批量上报：每60秒或缓冲区满50条时上报
- 端点：POST /api/v1/telemetry/batch
- 请求体：{ "events": [envelope, ...] }
- 响应：{ "accepted": 45, "rejected": 5, "reasons": [...] }
- 网络中断：本地SQLite缓存，最多保留10000条，恢复后批量重传
- 重试策略：指数退避（1s, 2s, 4s, 8s, 16s, 30s上限），最多5次
```

---

## 八、隐私和合规方案

### 8.1 法规依据

- **《中华人民共和国个人信息保护法》（2021.11施行）**：个人信息处理需取得同意，最小必要原则，用户有权删除
- **《生成式人工智能服务管理暂行办法》（2023.8施行）**：训练数据合法性，用户输入保护
- **《人工智能生成合成内容标识办法》（2025.9施行）**：AI生成内容标识
- **《未成年人网络保护条例》（2024.1施行）**：未成年人特殊保护

### 8.2 数据匿名化方案

| 数据项 | 处理方式 | 方法 |
|--------|---------|------|
| 用户ID | 匿名化 | 本地生成随机UUID，不与账号绑定；服务端只存哈希 |
| 灵魂ID | 匿名化 | 灵魂配置内容的SHA-256哈希前8位，不存灵魂名称 |
| 对话原文 | **不存储** | 只存消息长度、情感分类、回复时间等结构化指标 |
| IP地址 | 脱敏 | 只存国家/省份级别，不存完整IP |
| 设备信息 | 泛化 | 只存平台类型（Windows/Mac），不存设备型号/序列号 |
| 地理位置 | 粗粒度 | 只存城市级别（用户授权后），默认不收集 |
| 支付信息 | 不经过遥测 | 支付数据走独立支付通道，遥测只存购买商品ID和金额 |

### 8.3 k-匿名与差分隐私

- **k-匿名（k=5）**：任何聚合报告中，分组样本量<5的不发布，防止通过组合识别个体
- **差分隐私（ε=1.0）**：对外发布的统计数据添加拉普拉斯噪声，确保单条数据的加入/移除不影响统计结果
- **数据最小化**：只收集分析所需的最少字段，定期审查并删除不必要的字段

### 8.4 用户知情同意

**首次启动时的同意流程**：
1. 简洁说明："我们收集匿名使用数据用于改进游戏体验，不存储您的对话内容"
2. 三个选项：
   - ✅ 同意收集全部匿名数据（默认推荐）
   - ⚙️ 自定义（可选择数据类型：灵魂运行/大模型性能/玩家行为/系统性能）
   - ❌ 不收集（仅本地错误日志）
3. 随时可在设置中更改偏好
4. 数据收集状态在设置中可见

### 8.5 数据删除权实现

- 用户在设置中点击"删除我的数据"→ 服务端按`soul_id`/`session_id`删除所有关联数据
- 删除在72小时内完成，从备份中清除需要90天（备份周期）
- 删除后发送确认通知
- 未注册用户：清除本地数据即完成删除（服务端数据按soul_id关联，无法关联到具体用户）

### 8.6 数据访问权限

| 角色 | 访问范围 | 审批要求 |
|------|---------|---------|
| 开发团队 | 聚合统计数据、匿名化明细 | 无需审批 |
| 数据分析 | 匿名化明细数据 | 项目负责人审批 |
| 外部合作方 | 仅聚合统计，差分隐私处理 | 数据保护官审批 + 数据处理协议 |
| 公开报告 | 仅高度聚合的行业基准数据 | 管理层审批 |

---

## 九、数据收集架构设计

### 9.1 整体架构（文字描述）

```
┌─────────────────────────────────────────────────────────┐
│                    客户端（游戏进程）                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ Ember    │  │ Arboreus │  │ 战策     │  │ LLM     │ │
│  │ 遥测钩子  │  │ 遥测钩子  │  │ 埋点     │  │ 中间件   │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬────┘ │
│       │              │              │              │      │
│  ┌────▼──────────────▼──────────────▼──────────────▼────┐ │
│  │              TelemetryCollector（本地收集器）          │ │
│  │  · 统一信封封装  · 异步队列  · 本地SQLite缓存          │ │
│  │  · 批量上报  · 网络中断重传  · 用户偏好过滤            │ │
│  └───────────────────────┬──────────────────────────────┘ │
└──────────────────────────┼──────────────────────────────┘
                           │ HTTPS (批量上报)
┌──────────────────────────▼──────────────────────────────┐
│                    服务端（Guardian）                      │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  TelemetryIngest（接收端）                            │ │
│  │  · 验证schema  · 去重  · 写入热存储(Redis/SQLite)     │ │
│  └───────────────────────┬─────────────────────────────┘ │
│                          │                               │
│  ┌───────────────────────▼─────────────────────────────┐ │
│  │  StreamProcessor（流处理）                            │ │
│  │  · 实时聚合  · 异常检测  · 告警  · 指标计算            │ │
│  └───────────────────────┬─────────────────────────────┘ │
│                          │                               │
│  ┌───────────────────────▼─────────────────────────────┐ │
│  │  DataLake（数据湖）                                   │ │
│  │  · Parquet按日分区  · 冷热分层  · 归档                │ │
│  └───────────────────────┬─────────────────────────────┘ │
│                          │                               │
│  ┌───────────────────────▼─────────────────────────────┐ │
│  │  AnalysisDashboard（分析仪表盘）                      │ │
│  │  · 灵魂健康度  · 世界健康度  · 模型对比  · 留存分析    │ │
│  └─────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

### 9.2 Ember遥测钩子设计

在Ember引擎中内置`TelemetryHook`模块：

```javascript
// 设计建议（伪代码，实际实现由Ember开发任务执行）
class TelemetryHook {
  constructor(config) {
    this.enabled = config.enabled;          // Phase 0: false, Phase 1: true
    this.buffer = [];
    this.maxBufferSize = 50;
    this.flushInterval = 60000;             // 60秒
    this.subsystemTimers = new Map();
  }

  // 子系统调用开始
  startSubsystemCall(subsystem, category) {
    if (!this.enabled) return;
    this.subsystemTimers.set(subsystem, {
      start: performance.now(),
      category,
      callId: generateUUID()
    });
  }

  // 子系统调用结束
  endSubsystemCall(subsystem, success, errorType, inputSize, outputSize) {
    if (!this.enabled) return;
    const timer = this.subsystemTimers.get(subsystem);
    if (!timer) return;
    const event = this.createEnvelope('cognitive_subsystem_call', {
      subsystem,
      subsystem_category: timer.category,
      call_id: timer.callId,
      duration_ms: performance.now() - timer.start,
      success,
      error_type: errorType || null,
      input_size: inputSize,
      output_size: outputSize
    });
    this.buffer.push(event);
    this.subsystemTimers.delete(subsystem);
    if (this.buffer.length >= this.maxBufferSize) this.flush();
  }

  // 情绪状态采样（由EmotionSystem定时调用）
  recordEmotionSample(valence, arousal, dominance, labels, intensity, trigger) { ... }

  // 大模型调用记录（由LLMMiddleware调用）
  recordLLMCall(metadata) { ... }

  // 批量上报
  async flush() { ... }
}
```

**关键设计**：
- 遥测钩子使用`if (!this.enabled) return;`快速路径，禁用时开销<0.001ms
- 所有遥测操作异步执行，不阻塞认知管线
- 子系统调用计时使用`performance.now()`（高精度）
- 缓冲区满或定时触发批量上报

### 9.3 Arboreus遥测钩子设计

类似Ember，但针对世界引擎的特点：
- **帧级采样**：每帧记录`frame_time_ms`、`active_entities`、各子系统耗时
- **日级聚合**：社会网络指标、经济指标按游戏日计算并记录
- **事件驱动**：世界事件、NPC目标达成等关键事件即时记录
- **NPC级采样**：对活跃NPC降采样（10%）记录决策路径，避免数据量爆炸

### 9.4 大模型调用中间件

所有大模型调用经过统一中间件：

```javascript
class LLMMiddleware {
  async call(modelConfig, prompt, scenario, subsystem) {
    const startTime = performance.now();
    const queueStart = Date.now();

    // 1. 记录请求元数据
    const callId = generateUUID();
    const requestMeta = {
      call_id: callId,
      model_name: modelConfig.name,
      model_version: modelConfig.version,
      scenario, subsystem,
      temperature: modelConfig.temperature,
      top_p: modelConfig.top_p,
      max_tokens: modelConfig.maxTokens,
      system_prompt_hash: sha256(modelConfig.systemPrompt),
      input_tokens: countTokens(prompt)
    };

    // 2. 执行调用（带重试）
    let result, retryCount = 0;
    try {
      result = await this.executeWithRetry(modelConfig, prompt);
    } catch (e) {
      // 3. 记录失败
      this.telemetry.recordLLMCall({ ...requestMeta, success: false, error_type: e.type, retry_count: retryCount });
      throw e;
    }

    // 4. 记录成功
    this.telemetry.recordLLMCall({
      ...requestMeta,
      output_tokens: result.usage.outputTokens,
      total_tokens: result.usage.totalTokens,
      estimated_cost_usd: this.calculateCost(modelConfig, result.usage),
      queue_time_ms: Date.now() - queueStart,
      inference_time_ms: performance.now() - startTime,
      success: true,
      retry_count: retryCount,
      first_token_time_ms: result.firstTokenTime
    });

    // 5. 异步质量评估（不阻塞返回）
    this.qualityEvaluator.evaluateAsync(callId, result.text, scenario);

    return result;
  }
}
```

### 9.5 客户端埋点框架

战策平台层预留统一埋点接口：

```javascript
// 平台层预留（Phase 0只加接口，不启用）
class Analytics {
  static track(eventName, properties = {}) {
    // Phase 0: no-op（只定义接口，不执行任何操作）
    // Phase 1: 调用TelemetryCollector收集
    if (Analytics._enabled && Analytics._collector) {
      Analytics._collector.track(eventName, properties);
    }
  }

  static setEnabled(enabled) { Analytics._enabled = enabled; }
  static setCollector(collector) { Analytics._collector = collector; }
}

// 使用示例：
Analytics.track('combat_start', { map_id: 'arena_01', team_size: 4 });
Analytics.track('dialogue_message', { message_length: 45, response_time_ms: 1200 });
```

---

## 十、分析方法和指标体系

### 10.1 四层分析模型

| 层级 | 方法 | 用途 | 工具 |
|------|------|------|------|
| **描述性分析** | 统计汇总、趋势图、分布直方图 | 发生了什么 | SQL + 仪表盘 |
| **诊断性分析** | 下钻分析、相关性分析、根因分析 | 为什么发生 | SQL + Python |
| **预测性分析** | 时间序列预测、异常检测、分类模型 | 将发生什么 | Python + ML |
| **处方性分析** | A/B测试、优化建议、自动调参 | 应该怎么做 | A/B框架 + 优化算法 |

### 10.2 核心分析场景

#### 场景1：灵魂子系统效能分析
- **问题**：哪些认知子系统在实际运行中最有效？哪些是瓶颈？
- **方法**：按子系统聚合调用频率、耗时、成功率、错误率；计算子系统与灵魂健康度的相关性
- **输出**：子系统效能排行榜，瓶颈识别报告
- **频率**：每周

#### 场景2：大模型选型优化
- **问题**：不同模型在各场景下的性价比如何？
- **方法**：按场景×模型交叉分析质量评分、响应时间、成本；计算MVS；A/B测试验证
- **输出**：模型选型建议，各场景最优模型
- **频率**：每月（新模型发布时即时）

#### 场景3：情绪系统调优
- **问题**：灵魂的情绪波动是否合理？是否过于平淡或过于激烈？
- **方法**：情绪效价/唤醒度的时间序列分析，情绪触发因素分布，情绪调节有效性
- **输出**：情绪系统参数调优建议
- **频率**：每两周

#### 场景4：世界经济平衡分析
- **问题**：经济系统是否稳定？是否有通货膨胀/通货紧缩？财富分配是否合理？
- **方法**：价格指数趋势、基尼系数、资源流动图、市场崩溃事件分析
- **输出**：经济参数调优建议
- **频率**：每月

#### 场景5：玩家留存与行为分析
- **问题**：玩家为什么流失？哪些功能驱动留存？
- **方法**：留存曲线、漏斗分析、功能使用与留存的相关性、聚类分析玩家类型
- **输出**：产品迭代优先级建议
- **频率**：每周

#### 场景6：异常检测与告警
- **问题**：系统是否出现异常？灵魂是否出现异常行为？
- **方法**：SHS/WHS阈值告警，子系统错误率突变检测，大模型幻觉率突变检测
- **输出**：实时告警通知
- **频率**：实时

### 10.3 灵魂健康度仪表盘

```
灵魂健康度仪表盘（Soul Health Dashboard）
├── 总览：SHS趋势图（7天/30天），健康/警告/危险灵魂占比
├── 认知健康：子系统调用热力图（频率×耗时），错误率Top10子系统
├── 情绪健康：情绪效价-唤醒度散点图，情绪触发因素分布饼图
├── 记忆健康：缓存命中率趋势，检索成功率，记忆衰减曲线拟合
├── 价值观一致性：L0违规趋势，价值观向量分布，ValueGuard拦截统计
├── 安全稳定：安全等级分布，升级事件时间线，ContentFilter命中统计
└── 对比：不同灵魂模板的健康度对比，不同模型的灵魂健康度影响
```

---

## 十一、Phase 0 接口预留具体建议（M2期间）

Phase 0的核心原则：**只加接口，不启用收集，不影响核心功能**。

### 11.1 战策平台层（必须做）

| 任务 | 具体内容 | 预估工作量 |
|------|---------|-----------|
| 定义`Analytics.track()`接口 | 如9.5节所示，no-op实现 | 0.5天 |
| 定义`TelemetryConfig`配置 | `enabled: false`，数据类型开关 | 0.5天 |
| 在关键路径埋点 | 战斗开始/结束、对话开始/消息、灵魂养成、教程步骤 | 1天 |
| 大模型调用中间件骨架 | 定义`LLMMiddleware.call()`接口，元数据收集逻辑预留 | 1天 |
| 本地SQLite初始化 | 创建`telemetry.db`，建表（events表），WAL模式 | 0.5天 |
| 上报器骨架 | `TelemetryReporter.flush()`，当前为no-op | 0.5天 |
| **合计** | | **约4天** |

### 11.2 Ember引擎（建议做，不强制）

| 任务 | 具体内容 | 预估工作量 |
|------|---------|-----------|
| `TelemetryHook`类骨架 | 如9.2节所示，`enabled=false`时快速返回 | 1天 |
| 在`Soul.tick()`中埋点 | tick开始/结束计时，子系统调用包裹 | 2天 |
| `EmotionSystem`采样接口 | `recordEmotionSample()`方法，当前no-op | 0.5天 |
| `MemorySystem`指标接口 | `recordMemoryStats()`方法 | 0.5天 |
| **合计** | | **约4天** |

### 11.3 Arboreus引擎（建议做，不强制）

| 任务 | 具体内容 | 预估工作量 |
|------|---------|-----------|
| `WorldTelemetry`类骨架 | 帧级采样+日级聚合接口 | 1天 |
| 引擎主循环埋点 | 帧时间、各子系统耗时 | 1天 |
| 经济系统指标接口 | `recordEconomicStats()` | 0.5天 |
| 社会网络指标接口 | `recordSocialNetworkStats()` | 0.5天 |
| **合计** | | **约3天** |

### 11.4 服务端（Guardian）

| 任务 | 具体内容 | 预估工作量 |
|------|---------|-----------|
| `/api/v1/telemetry/batch`端点 | 接收端点，schema验证，当前返回`accepted:0` | 1天 |
| 数据库表结构 | events表、daily_agg表 | 0.5天 |
| **合计** | | **约1.5天** |

> Phase 0总计约12.5天工作量，分散在M2期间各任务的空闲时间完成，不影响核心功能开发。

---

## 十二、实施优先级与路线图

### 12.1 P0：EA上线必须（Phase 1，EA前完成）

| ID | 任务 | 说明 | 负责 |
|----|------|------|------|
| DC-P0-01 | 启用Ember遥测钩子 | `enabled=true`，认知子系统通用指标+情绪采样+记忆统计 | Ember开发 |
| DC-P0-02 | 启用大模型调用中间件 | 全量记录调用元数据+token+耗时+成本 | 战策/集成 |
| DC-P0-03 | 客户端埋点启用 | 战斗/对话/养成/探索核心事件 | 战策开发 |
| DC-P0-04 | 服务端接收+存储 | TelemetryIngest端点+SQLite/Parquet存储 | 管理/服务端 |
| DC-P0-05 | 隐私合规实现 | 知情同意UI+匿名化+数据删除权 | 战策/管理 |
| DC-P0-06 | 基础仪表盘 | SHS总览+大模型调用统计+留存基础 | 管理任务 |
| DC-P0-07 | 批量上报+断网重传 | 客户端上报器完整实现 | 战策开发 |

### 12.2 P1：M3期间（Phase 2）

| ID | 任务 | 说明 |
|----|------|------|
| DC-P1-01 | Arboreus遥测启用 | 世界运行数据全量收集 |
| DC-P1-02 | 大模型质量自动评估 | LLM-as-judge+幻觉检测+角色一致性评分 |
| DC-P1-03 | A/B测试框架 | 模型对比+功能A/B测试 |
| DC-P1-04 | 世界健康度仪表盘 | WHS+经济+社会网络分析 |
| DC-P1-05 | 异常检测与告警 | SHS/WHS/错误率/幻觉率实时告警 |
| DC-P1-06 | 数据归档与冷热分层 | Parquet归档+冷存储 |
| DC-P1-07 | 玩家行为深度分析 | 漏斗+留存+聚类+功能相关性 |

### 12.3 P2：长期（Phase 3）

| ID | 任务 | 说明 |
|----|------|------|
| DC-P2-01 | 预测性分析 | 趋势预测+异常预测+灵魂行为预测 |
| DC-P2-02 | 自动调参系统 | 基于数据的灵魂参数/世界参数自动优化 |
| DC-P2-03 | 个性化推荐 | 基于玩家行为的内容/功能推荐 |
| DC-P2-04 | 模型微调数据管道 | 基于高质量对话的模型微调数据准备 |
| DC-P2-05 | 差分隐私发布 | 对外行业基准数据的差分隐私处理 |
| DC-P2-06 | 实时分析流处理 | Flink/Kafka流处理，秒级仪表盘 |

---

## 十三、未实现方向分析

本节列出与数据收集相关的、当前凌栖尚未实现的方向。

### P0：EA上线必须

| ID | 方向 | 价值评估 | 实现难度 | 与现有系统的关系 | 建议阶段 |
|----|------|---------|---------|----------------|---------|
| DC-P0-01 | Ember遥测钩子（认知子系统通用指标+情绪采样+记忆统计） | 极高——灵魂改进的数据基础 | 中 | Ember引擎内置 | EA前 |
| DC-P0-02 | 大模型调用中间件（元数据+token+耗时+成本全量记录） | 极高——模型选型和成本控制 | 低-中 | 战策平台层 | EA前 |
| DC-P0-03 | 客户端核心事件埋点（战斗/对话/养成/探索） | 高——产品迭代 | 低 | 战策平台层 | EA前 |
| DC-P0-04 | 服务端遥测接收+存储（SQLite+Parquet） | 高——数据基础设施 | 中 | Guardian服务端 | EA前 |
| DC-P0-05 | 隐私合规（知情同意+匿名化+删除权） | 极高——合规必需 | 中 | 跨平台 | EA前 |
| DC-P0-06 | 基础分析仪表盘（SHS+模型统计+留存） | 高——数据可见性 | 中 | 管理仪表盘 | EA前 |
| DC-P0-07 | 批量上报+断网重传 | 中——数据完整性 | 低-中 | 客户端 | EA前 |

### P1：M3期间

| ID | 方向 | 价值评估 | 实现难度 | 与现有系统的关系 | 建议阶段 |
|----|------|---------|---------|----------------|---------|
| DC-P1-01 | Arboreus世界遥测（经济/社会/文化/NPC指标） | 高——世界引擎改进 | 中-高 | Arboreus引擎内置 | M3 |
| DC-P1-02 | 大模型生成质量自动评估（LLM-as-judge+幻觉检测） | 高——质量保障 | 中-高 | 大模型中间件扩展 | M3 |
| DC-P1-03 | A/B测试框架（模型对比+功能测试） | 高——数据驱动决策 | 中 | 服务端+客户端 | M3 |
| DC-P1-04 | 世界健康度仪表盘（WHS+经济+社会网络） | 中-高——世界平衡 | 中 | 管理仪表盘 | M3 |
| DC-P1-05 | 异常检测与实时告警 | 高——问题快速发现 | 中 | 流处理+告警 | M3 |
| DC-P1-06 | 数据归档与冷热分层存储 | 中——成本控制 | 中 | 数据湖 | M3 |
| DC-P1-07 | 玩家行为深度分析（漏斗+聚类+相关性） | 高——产品优化 | 中 | 分析层 | M3 |

### P2：长期

| ID | 方向 | 价值评估 | 实现难度 | 与现有系统的关系 | 建议阶段 |
|----|------|---------|---------|----------------|---------|
| DC-P2-01 | 预测性分析（趋势预测+异常预测） | 中-高——主动运维 | 高 | ML模型 | M4+ |
| DC-P2-02 | 基于数据的自动调参（灵魂/世界参数优化） | 高——自动化改进 | 高 | 优化算法 | M4+ |
| DC-P2-03 | 个性化推荐系统 | 中——商业化 | 中-高 | 推荐引擎 | M4+ |
| DC-P2-04 | 模型微调数据管道 | 中——模型定制 | 中-高 | 数据处理 | M4+ |
| DC-P2-05 | 差分隐私数据发布 | 中——对外合作 | 中 | 隐私技术 | 长期 |
| DC-P2-06 | 实时流处理（Flink/Kafka，秒级仪表盘） | 中——大规模时需要 | 高 | 基础设施 | 10万DAU后 |

---

## 十四、与其他预研的关联

- **第14轮Ember 004（灵魂注入与价值观对齐）**：价值观对齐检测指标（L0违规率、价值向量一致性）是灵魂运行数据的核心维度
- **第16轮shared/003（平台化安全合规）**：数据收集的隐私合规方案与安全围栏的运营层安全直接相关；安全审计日志是遥测数据的一种
- **第17轮testing/003（AI时代测试）**：大模型幻觉检测方法（HCPD）直接用于生成质量评估；性能基准回归使用系统性能数据
- **第7轮Arboreus 002（生态社会模拟）**：社会网络指标、经济指标的定义基于该报告的模拟系统设计
- **第12轮Arboreus 003（叙事生成）**：NPC决策路径记录用于叙事生成质量分析
- **管理策略第二十三节（灵魂注入与内容安全）**：ValueGuard的L0/L1/L2三层是价值观对齐检测指标的基础
- **管理策略第二十四节（Steam平台接入）**：玩家行为数据收集需要与Steam的统计/成就系统对接

---

## 十五、参考来源

1. 管理策略第二十五节《灵魂数据收集与分析体系》，2026-09-07
2. Ember灵魂引擎：137+认知子系统（server/soul/目录），v5.34
3. Arboreus世界引擎：37个模块（src/目录），SDK v2.6.0
4. 《中华人民共和国个人信息保护法》，2021.11施行
5. 《生成式人工智能服务管理暂行办法》，2023.8施行
6. 《人工智能生成合成内容标识办法》，2025.9施行
7. ICML 2026 HCPD零源幻觉检测（参考第17轮测试研究）
8. 游戏遥测行业实践：Unity Analytics、Unreal Engine Telemetry、GameAnalytics
9. 差分隐私：Dwork, C. (2006) "Differential Privacy"
10. k-匿名：Sweeney, L. (2002) "k-Anonymity: A Model for Protecting Privacy"
11. A/B测试：Kohavi, R. et al. (2020) "Trustworthy Online Controlled Experiments"
12. 可观测性三支柱：Metrics/Logs/Traces（OpenTelemetry标准）
13. 时间序列数据库：InfluxDB/ClickHouse方案对比
14. 灵魂健康度评分模型设计：基于Mental Health Index和System Health Score的综合方法
