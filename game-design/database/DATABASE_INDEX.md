# 游戏设计研究数据库索引

> 凌栖/Sojourn游戏设计预研数据库——历史、文化、哲学概念的结构化条目库，用于设计灵感和联想。

## 数据库结构

每个条目包含：
- **编号**：DB-XXX（按领域分类编号）
- **领域**：历史 / 文化 / 哲学 / 神话 / 象征 / 人物 / 事件
- **标题**：概念名称
- **核心定义**：一句话概括
- **详细阐述**：概念的内涵、起源、演变
- **跨文化关联**：与其他文明/概念的联系
- **设计灵感**：可用于游戏设计的方向（角色/场景/机制/叙事）
- **联想关键词**：用于跨条目关联的标签

## 编号规则

- DB-His-XXX：历史类
- DB-Cul-XXX：文化类
- DB-Phi-XXX：哲学类
- DB-Myt-XXX：神话类
- DB-Sym-XXX：象征类
- DB-Per-XXX：人物类
- DB-Eve-XXX：事件类

## 条目索引

### 灵魂与意识主题（第一批）

| 编号 | 标题 | 领域 | 文件 |
|------|------|------|------|
| DB-Phi-001 | 灵魂概念的跨文化谱系 | 哲学/文化 | 001_soul_concepts_crosscultural.md |
| DB-Phi-002 | 意识的难问题与心灵哲学 | 哲学 | 002_consciousness_hard_problem.md |
| DB-Myt-001 | 荣格原型与集体无意识 | 神话/心理学 | 003_jung_archetypes.md |
| DB-Cul-001 | 火与灵魂的象征关联 | 文化/象征 | 004_fire_soul_symbolism.md |
| DB-Phi-003 | 转生与灵魂迁移的哲学 | 哲学/宗教 | 005_reincarnation_soul_migration.md |

### 战争与战略主题（第二批）

| 编号 | 标题 | 领域 | 文件 |
|------|------|------|------|
| DB-His-001 | 孙子兵法战略框架（五事·七计·十二诡道） | 历史/军事哲学 | 006_sun_tzu_strategic_framework.md |
| DB-His-002 | 克劳塞维茨战争论核心概念（摩擦·三位一体·重心·迷雾） | 历史/军事哲学 | 007_clausewitz_war_theory.md |
| DB-Myt-002 | 战士原型与战争神话跨文化谱系 | 神话/文化 | 008_warrior_mythology_archetypes.md |
| DB-His-003 | 围城战与后勤战役史——持久战的艺术 | 历史/军事 | 009_siege_logistics_campaign.md |
| DB-His-004 | 阵法与战术编队演进史——从方阵到合成兵种 | 历史/军事组织 | 010_formation_tactics_evolution.md |

### 城市与文明主题（第三批）

| 编号 | 标题 | 领域 | 文件 |
|------|------|------|------|
| DB-His-005 | 城市起源与城市规划史——从泥砖到网格 | 历史/城市研究 | 011_city_origin_planning.md |
| DB-Phi-004 | 文明兴衰理论——从循环论到系统论 | 历史哲学/文明研究 | 012_civilization_cycle_theory.md |
| DB-Cul-002 | 建筑的宇宙象征——从通天塔到世界之轴 | 建筑史/宗教象征 | 013_architecture_cosmic_symbolism.md |
| DB-His-006 | 商路与文明交流——从丝绸之路到世界体系 | 历史/经济史 | 014_trade_route_network.md |
| DB-His-007 | 技术演进范式——技术革命与文明跃迁 | 技术史/创新理论 | 015_tech_evolution_paradigm.md |

## 联想图谱（持续构建）

### 核心主题网络
```
灵魂(Soul) ── 意识(Consciousness) ── 自我(Self)
   │                │                    │
   ├── 火(Fire)     ├── 难问题(Hard Problem)  ├── 原型(Archetype)
   ├── 转生(Reincarnation)  ├── 具身(Embodiment)  ├── 阴影(Shadow)
   └── 寄居(Sojourn)        └── 感知(Perception)   └── 自性(Selfhood)

战争(War) ── 战略(Strategy) ── 战术(Tactics)
   │              │                  │
   ├── 诡道(Deception)  ├── 摩擦(Friction)  ├── 方阵(Phalanx)
   ├── 后勤(Logistics)  ├── 重心(Center of Gravity)  ├── 合成兵种(Combined Arms)
   ├── 围城(Siege)      ├── 战争迷雾(Fog of War)      └── 砧锤战术(Anvil-Hammer)
   └── 战士原型(Warrior Archetype)
        ├── 阿喀琉斯(Achilles) ── 荣誉(Honor)
        ├── 狂战士(Berserker) ── 命运(Wyrd)
        ├── 武士(Samurai) ── 武士道(Bushido)
        ├── 刹帝利(Kshatriya) ── 达摩(Dharma)
        └── 侠与将 ── 忠义(Loyalty)

城市(City) ── 文明(Civilization) ── 技术(Technology)
   │                │                    │
   ├── 网格(Grid)    ├── 凝聚力(Asabiyyah)  ├── 范式跃迁(Paradigm Shift)
   ├── 宇宙轴(Axis Mundi) ├── 挑战-回应(Challenge-Response) ├── S曲线(S-Curve)
   ├── 神庙(Temple)  ├── 复杂性递减(Diminishing Returns) ├── 路径依赖(Path Dependence)
   ├── 里坊(Ward)    ├── 包容vs榨取(Inclusive vs Extractive) └── 技术集群(Tech Cluster)
   └── 商路(Trade Route)
        ├── 丝绸之路(Silk Road) ── 技术传播
        ├── 印度洋贸易(Indian Ocean) ── 文化融合
        ├── 哥伦布大交换(Columbian Exchange) ── 生物交换
        └── 琥珀之路(Amber Road) ── 早期文明接触
```

### 主题间关联
- **灵魂 ↔ 战争**：战士原型中的灵魂概念（武器有灵魂、战死后灵魂进入英灵殿）→ 灵魂迁移（GD-005）
- **意识 ↔ 战争迷雾**：战争中的信息不确定性（GD-007）与意识的感知局限性（GD-002）同构
- **原型 ↔ 战士**：荣格英雄原型（GD-003）在战争领域的具体化 → 战士原型（GD-008）
- **火 ↔ 战争**：火的毁灭与净化象征（GD-004）与战争的毁灭与重生对应
- **孙子 ↔ 克劳塞维茨**：东方诡道 vs 西方理性，互补的战略思想体系（GD-006 + GD-007）
- **阵法 ↔ 后勤**：编队战术（GD-010）依赖后勤补给（GD-009），二者共同决定战役胜负
- **城市 ↔ 文明兴衰**：城市是文明的载体，城市规划模式（GD-011）反映文明的组织原则，文明兴衰理论（GD-012）解释城市的命运
- **建筑象征 ↔ 世界之轴**：神圣建筑（GD-013）是宇宙轴的物化，与Arboreus（建木=世界树）的命名直接呼应
- **商路 ↔ 技术传播**：商路（GD-014）是技术传播的通道，技术演进（GD-015）的速度取决于商路的畅通
- **技术 ↔ 文明兴衰**：技术范式跃迁（GD-015）是文明兴衰（GD-012）的重要驱动力
- **凝聚力 ↔ 战争**：伊本·赫勒敦的Asabiyyah（GD-012）与孙子的"道"（GD-006）本质相同——凝聚力是战斗力的源泉
- **火 ↔ 技术**：火是人类第一项技术（GD-015），也是灵魂的象征（GD-004）——技术与意识的共同起源

### 与项目的关联
- **凌栖/Sojourn**：灵魂寄居、迁移、暂住 → 转生/灵魂迁移主题
- **灵火/Ember**：灵魂之火 → 火与灵魂象征
- **建木/Arboreus**：世界树、连接天地 → 宇宙轴/世界树神话
- **战策/Battleplan**：灵魂对战 → 战争与战略主题（第二批全部条目直接服务于战策RTS设计）
  - 孙子兵法（GD-006）→ 战术欺骗与心理战机制
  - 克劳塞维茨（GD-007）→ 摩擦系统与战争迷雾深化
  - 战士原型（GD-008）→ 阵营文化差异化与英雄设计
  - 围城后勤（GD-009）→ 补给线系统与持久战模式
  - 阵法演进（GD-010）→ 编队系统与兵种协同
- **建木/Arboreus**：世界树、连接天地 → 城市与文明主题（第三批直接服务于Arboreus世界模拟）
  - 城市规划（GD-011）→ 聚落/城市生成系统的文化差异化
  - 文明兴衰（GD-012）→ 文明模拟的凝聚力/制度/复杂性机制
  - 建筑象征（GD-013）→ 神圣建筑系统与世界之轴主题
  - 商路交流（GD-014）→ 贸易系统与文明交流机制
  - 技术演进（GD-015）→ 多线科技树与范式跃迁设计
