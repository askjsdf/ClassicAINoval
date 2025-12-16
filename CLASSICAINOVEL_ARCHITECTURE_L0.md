# ClassicAiNovel 零层架构设计文档

> 基于对 MuMuAINovel 的深度研究，为公版书AI改写平台设计的产品与技术架构

---

## 目录

1. [产品定位与核心价值](#1-产品定位与核心价值)
2. [系统架构图](#2-系统架构图langgraph-多-agent-结构)
3. [核心数据模型设计](#3-核心数据模型设计)
4. [蝴蝶效应状态机设计](#4-蝴蝶效应状态机设计)
5. [原著分解与管理方案](#5-原著分解与管理方案)
6. [多Agent协作流程](#6-多-agent-协作流程)
7. [技术选型建议](#7-技术选型建议)
8. [与MuMu的关键差异对比](#8-与-mumu-的关键差异对比)
9. [从MuMu获得的关键启示](#9-从-mumu-获得的关键启示)
10. [实施建议](#10-实施建议)

---

## 1. 产品定位与核心价值

### 1.1 产品定位

**ClassicAiNovel** 是一款基于公版书（Public Domain Books）的 AI 互动小说改写平台，让用户以原创角色身份"穿越"进入经典文学作品，通过自己的选择影响原著剧情走向，体验"蝴蝶效应"带来的全新故事线。

### 1.2 核心价值主张

| 维度 | 描述 |
|------|------|
| **IP 复用** | 充分利用公版书的经典 IP 价值，降低世界观构建成本 |
| **互动叙事** | 用户扮演新角色参与剧情，每章结束可做出影响后续走向的选择 |
| **蝴蝶效应** | 用户选择产生连锁反应，形成个性化的改写路线 |
| **AI 协作** | 多 Agent 协作确保改写质量：原著忠实度 + 创意合理性 + 文笔流畅度 |

### 1.3 目标用户

- 经典文学爱好者，希望以新视角重温名著
- 互动小说/游戏书爱好者
- 二次创作者，寻找创作灵感
- 教育用途：以沉浸方式学习经典文学

### 1.4 核心用户场景示例

```
场景：用户选择《西游记》，创建角色"云游子"作为取经队伍的第四个徒弟

第27回「三打白骨精」改写：
- 原著：孙悟空三次打死白骨精变化的人，被唐僧误解驱逐
- 用户选择：在第二次时，云游子识破白骨精真身并劝阻唐僧
- 蝴蝶效应：
  ├─ 短期影响：孙悟空未被驱逐
  ├─ 中期影响：白骨精逃脱，后续寻仇
  └─ 长期影响：真假美猴王剧情可能不会发生（因孙悟空未离队）
```

---

## 2. 系统架构图（LangGraph 多 Agent 结构）

### 2.1 高层架构

```
                                    +------------------+
                                    |   用户界面层     |
                                    |  (React + TS)    |
                                    +--------+---------+
                                             |
                                             v
                            +----------------+----------------+
                            |         API 网关层              |
                            |      (FastAPI + WebSocket)      |
                            +----------------+----------------+
                                             |
               +-----------------------------+-----------------------------+
               |                             |                             |
               v                             v                             v
    +----------+---------+      +------------+------------+    +-----------+---------+
    |   原著管理服务      |      |   LangGraph Agent      |    |   用户状态服务       |
    |  (书籍导入/分析)    |      |   Orchestrator         |    |  (选择/路线/存档)    |
    +----------+---------+      +------------+------------+    +-----------+---------+
               |                             |                             |
               |              +--------------+--------------+              |
               |              |              |              |              |
               v              v              v              v              v
    +----------+----+  +------+------+  +---+------+  +----+------+  +----+------+
    | 原著分解器    |  | 剧情架构师  |  | 撰稿人   |  | 监修      |  | 蝴蝶效应  |
    | (BookParser)  |  | Agent       |  | Agent    |  | Agent     |  | 引擎      |
    +---------------+  +-------------+  +----------+  +-----------+  +-----------+
                              |              |              |              |
                              +--------------+--------------+--------------+
                                             |
                                             v
                            +----------------+----------------+
                            |        数据持久层               |
                            |   PostgreSQL + ChromaDB        |
                            +---------------------------------+
```

### 2.2 LangGraph Agent 工作流

```
                              +---------------------+
                              |    用户选择输入     |
                              +---------+-----------+
                                        |
                                        v
                              +---------+-----------+
                              |  蝴蝶效应引擎       |
                              |  (计算影响链)       |
                              +---------+-----------+
                                        |
                                        v
                    +------------------->-----------------+
                    |                                     |
                    v                                     |
          +---------+-----------+                         |
          |   剧情架构师 Agent   |                         |
          |   - 读取原著锚点    |                         |
          |   - 分析选择影响    |                         |
          |   - 生成改写大纲    |                         |
          +---------+-----------+                         |
                    |                                     |
                    v                                     |
          +---------+-----------+                         |
          |   撰稿人 Agent      |<------------------------+
          |   - 原著文风模仿    |        (修改反馈)       |
          |   - 融入用户角色    |                         |
          |   - 生成改写章节    |                         |
          +---------+-----------+                         |
                    |                                     |
                    v                                     |
          +---------+-----------+                         |
          |   监修 Agent        |                         |
          |   - 原著符合度检查  |                         |
          |   - 逻辑合理性审核  |                         |
          |   - AI 味检测       |                         |
          +---------+-----------+                         |
                    |                                     |
           +--------+--------+                            |
           |   审核通过?     |--------- 否 ---------------+
           +--------+--------+
                    | 是
                    v
          +---------+-----------+
          |   章节输出          |
          |   + 选择项生成      |
          +---------------------+
```

### 2.3 三大 Agent 职责定义

| Agent | 核心职责 | 输入 | 输出 |
|-------|---------|------|------|
| **剧情架构师** | 阅读原著，分析用户角色影响，规划改写大纲 | 原著章节、用户角色、活跃蝴蝶效应 | 改写大纲JSON |
| **撰稿人** | 按大纲执行改写，模仿原著风格 | 改写大纲、原著原文、上下文 | 改写章节内容 |
| **监修** | 审核质量（原著符合度、逻辑性、AI味） | 改写内容、原著对照 | 审核意见+评分 |

---

## 3. 核心数据模型设计

### 3.1 原著相关模型

```python
# 公版书表
class PublicDomainBook(Base):
    """公版书原著"""
    __tablename__ = "public_domain_books"

    id = Column(String(36), primary_key=True)
    title = Column(String(200), nullable=False)           # 书名
    author = Column(String(100))                          # 原作者
    era = Column(String(100))                             # 成书年代
    language = Column(String(50), default="zh")           # 原著语言
    genre = Column(String(100))                           # 类型：神话/历史/武侠等
    total_chapters = Column(Integer)                      # 原著总章回数
    synopsis = Column(Text)                               # 原著简介

    # AI分析结果（JSON存储）
    world_setting = Column(JSON)                          # 世界观设定提取
    character_index = Column(JSON)                        # 原著人物索引
    timeline_index = Column(JSON)                         # 原著时间线索引
    power_system = Column(JSON)                           # 力量体系（如西游记的神仙妖魔体系）

    import_status = Column(String(20), default="pending") # 导入状态
    created_at = Column(DateTime, server_default=func.now())


# 原著章回表
class OriginalChapter(Base):
    """原著章回内容"""
    __tablename__ = "original_chapters"

    id = Column(String(36), primary_key=True)
    book_id = Column(String(36), ForeignKey("public_domain_books.id"))
    chapter_number = Column(Integer, nullable=False)      # 章回序号
    title = Column(String(200))                           # 章回标题
    content = Column(Text, nullable=False)                # 原文内容
    summary = Column(Text)                                # AI生成的章节摘要

    # 结构化分析结果
    key_events = Column(JSON)                             # 关键事件列表
    characters_appeared = Column(JSON)                    # 出场人物列表
    locations = Column(JSON)                              # 涉及地点
    time_markers = Column(JSON)                           # 时间标记
    plot_threads = Column(JSON)                           # 情节线索（伏笔/呼应）

    # 可改写点标记
    intervention_points = Column(JSON)                    # 用户可介入的剧情点

    # 向量检索
    vector_id = Column(String(100), unique=True)
    created_at = Column(DateTime, server_default=func.now())


# 原著人物表
class OriginalCharacter(Base):
    """原著人物库"""
    __tablename__ = "original_characters"

    id = Column(String(36), primary_key=True)
    book_id = Column(String(36), ForeignKey("public_domain_books.id"))
    name = Column(String(100), nullable=False)
    aliases = Column(JSON)                                # 别名：如孙悟空=[孙行者,齐天大圣,美猴王]
    role_type = Column(String(50))                        # 主角/配角/反派/路人

    # 人物画像
    identity = Column(Text)                               # 身份背景
    appearance = Column(Text)                             # 外貌描述
    personality = Column(Text)                            # 性格特点
    abilities = Column(JSON)                              # 能力/技能列表
    affiliations = Column(JSON)                           # 所属组织/势力

    # 关系网络
    relationships = Column(JSON)                          # 关系图谱

    # 原著轨迹
    first_appearance = Column(Integer)                    # 首次出场章回
    last_appearance = Column(Integer)                     # 最后出场章回
    fate = Column(Text)                                   # 原著结局
    arc_summary = Column(Text)                            # 人物弧光摘要

    # 状态变化记录（用于追踪改写后的状态）
    canonical_state_timeline = Column(JSON)               # 原著中各章节的状态

    created_at = Column(DateTime, server_default=func.now())
```

### 3.2 用户角色与改写项目模型

```python
# 改写项目表
class RewriteProject(Base):
    """用户的改写项目"""
    __tablename__ = "rewrite_projects"

    id = Column(String(36), primary_key=True)
    user_id = Column(String(100), nullable=False, index=True)
    book_id = Column(String(36), ForeignKey("public_domain_books.id"))

    title = Column(String(200), nullable=False)           # 改写版标题
    premise = Column(Text)                                # 改写前提/设定

    # 用户原创角色
    player_character_id = Column(String(36), ForeignKey("player_characters.id"))

    # 改写进度
    current_chapter = Column(Integer, default=1)
    story_branch_id = Column(String(36))                  # 当前故事分支ID
    total_deviation_score = Column(Float, default=0.0)    # 累计偏离度

    # 项目状态
    status = Column(String(20), default="active")
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, onupdate=func.now())


# 用户原创角色表
class PlayerCharacter(Base):
    """用户扮演的原创角色"""
    __tablename__ = "player_characters"

    id = Column(String(36), primary_key=True)
    user_id = Column(String(100), nullable=False)

    name = Column(String(100), nullable=False)
    identity = Column(Text)                               # 身份设定：如'唐僧的第四个徒弟'
    origin = Column(Text)                                 # 来历/背景故事
    appearance = Column(Text)
    personality = Column(Text)
    abilities = Column(JSON)                              # 能力/技能
    goals = Column(Text)                                  # 角色目标/动机
    limitations = Column(JSON)                            # 能力限制（防止角色过于强大破坏平衡）

    # 与原著的关联设定
    entry_point = Column(Integer)                         # 加入故事的章回
    entry_reason = Column(Text)                           # 加入的原因/契机
    initial_relationships = Column(JSON)                  # 与原著人物的初始关系

    created_at = Column(DateTime, server_default=func.now())
```

### 3.3 蝴蝶效应系统模型（核心）

```python
# 用户选择表
class UserChoice(Base):
    """用户在每章结束时做出的选择"""
    __tablename__ = "user_choices"

    id = Column(String(36), primary_key=True)
    project_id = Column(String(36), ForeignKey("rewrite_projects.id"))
    chapter_id = Column(String(36), ForeignKey("rewritten_chapters.id"))

    # 选择内容
    choice_text = Column(Text, nullable=False)            # 选择的文字描述
    choice_type = Column(String(50))                      # action/dialogue/decision/inaction

    # 选择的影响评估（AI生成）
    impact_level = Column(String(20))                     # minor/moderate/major/critical
    impact_analysis = Column(Text)                        # AI对影响的分析说明
    affected_characters = Column(JSON)                    # 影响到的角色列表
    affected_plotlines = Column(JSON)                     # 影响到的情节线

    # 选择的时间线位置
    story_timeline = Column(Integer)
    original_chapter_ref = Column(Integer)                # 对应原著章回

    # 是否为关键选择
    is_critical = Column(Boolean, default=False)

    created_at = Column(DateTime, server_default=func.now())


# 因果链表（蝴蝶效应核心）
class CausalChain(Base):
    """蝴蝶效应因果链 - 追踪选择的连锁反应"""
    __tablename__ = "causal_chains"

    id = Column(String(36), primary_key=True)
    project_id = Column(String(36), ForeignKey("rewrite_projects.id"))

    # 因（触发源）
    trigger_choice_id = Column(String(36), ForeignKey("user_choices.id"))
    trigger_type = Column(String(50))                     # user_choice/previous_effect/system
    trigger_description = Column(Text)                    # 触发描述

    # 果（影响结果）
    effect_type = Column(String(50), nullable=False)
    """
    影响类型:
    - character_survival: 人物存活状态改变（如白骨精未死）
    - character_state: 人物状态改变（如受伤/获得能力）
    - relationship_change: 关系改变（敌→友，友→敌）
    - plot_deviation: 情节偏离（跳过/改变原著情节）
    - plot_addition: 新增情节（原著没有的剧情）
    - item_transfer: 物品归属变化
    - alliance_shift: 阵营转变
    - timeline_branch: 时间线分支（重大改变）
    """
    effect_description = Column(Text)
    effect_data = Column(JSON)                            # 结构化的影响数据

    # 影响范围
    affected_chapter_start = Column(Integer)              # 影响起始章节
    affected_chapter_end = Column(Integer)                # 影响结束章节（-1表示永久）
    scope = Column(String(20))                            # local/regional/global

    # 影响状态（参考MuMu的伏笔状态机）
    status = Column(String(20), default="pending")
    """
    状态流转:
    pending → active → resolved/cancelled

    - pending: 效应已创建，尚未在剧情中激活
    - active: 效应正在影响剧情
    - resolved: 效应已被解决/呈现/消耗
    - cancelled: 效应被其他选择取消
    """
    activated_at_chapter = Column(Integer)                # 在哪章激活
    resolved_at_chapter = Column(Integer)                 # 在哪章解决
    resolution_description = Column(Text)                 # 解决方式描述

    # 链式关系（支持因果链的连锁反应）
    parent_chain_id = Column(String(36), ForeignKey("causal_chains.id"))
    chain_depth = Column(Integer, default=1)              # 因果链深度
    children_count = Column(Integer, default=0)           # 子效应数量

    # 重要性权重（用于上下文筛选）
    importance_score = Column(Float, default=0.5)

    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, onupdate=func.now())


# 角色状态表（追踪改写后的角色状态变化）
class CharacterState(Base):
    """角色在改写故事中的状态追踪"""
    __tablename__ = "character_states"

    id = Column(String(36), primary_key=True)
    project_id = Column(String(36), ForeignKey("rewrite_projects.id"))

    # 角色引用（原著角色或用户角色）
    original_character_id = Column(String(36), ForeignKey("original_characters.id"), nullable=True)
    player_character_id = Column(String(36), ForeignKey("player_characters.id"), nullable=True)
    character_name = Column(String(100))                  # 冗余存储，便于查询

    # 状态快照章节
    chapter_number = Column(Integer)

    # 与原著的偏离
    deviation_from_canon = Column(JSON)
    """
    {
        "survival": "alive",              # 原著此时已死但改写中存活
        "location": "花果山",              # 与原著位置不同
        "relationships": {...},            # 关系变化
        "abilities": [...],               # 能力变化
        "items": [...],                   # 持有物品变化
        "mental_state": "...",            # 心理状态
        "allegiance": "取经队伍"          # 阵营归属
    }
    """

    # 状态变化原因
    caused_by_choice_id = Column(String(36), ForeignKey("user_choices.id"))
    caused_by_chain_id = Column(String(36), ForeignKey("causal_chains.id"))

    created_at = Column(DateTime, server_default=func.now())


# 故事分支表
class StoryBranch(Base):
    """故事分支 - 记录不同的改写路线"""
    __tablename__ = "story_branches"

    id = Column(String(36), primary_key=True)
    project_id = Column(String(36), ForeignKey("rewrite_projects.id"))

    branch_name = Column(String(100))                     # 分支名称（如"白骨精存活线"）
    branch_point_chapter = Column(Integer)                # 分支起点章节
    branch_trigger_choice_id = Column(String(36), ForeignKey("user_choices.id"))

    # 分支特征
    deviation_level = Column(Float)                       # 与原著偏离度 0.0-1.0
    key_differences = Column(JSON)                        # 与原著的关键差异列表
    active_causal_chains = Column(JSON)                   # 活跃的因果链ID列表

    # 分支状态
    status = Column(String(20), default="active")         # active/abandoned/merged
    parent_branch_id = Column(String(36), ForeignKey("story_branches.id"))

    created_at = Column(DateTime, server_default=func.now())
```

### 3.4 改写内容模型

```python
# 改写章节表
class RewrittenChapter(Base):
    """改写后的章节"""
    __tablename__ = "rewritten_chapters"

    id = Column(String(36), primary_key=True)
    project_id = Column(String(36), ForeignKey("rewrite_projects.id"))
    branch_id = Column(String(36), ForeignKey("story_branches.id"))

    chapter_number = Column(Integer, nullable=False)
    title = Column(String(200))
    content = Column(Text)                                # 改写后的章节内容
    word_count = Column(Integer, default=0)

    # 原著对照
    original_chapter_ref = Column(Integer)                # 对应原著章回
    original_content_used = Column(Text)                  # 使用的原著片段（引用）
    deviation_score = Column(Float)                       # 与原著偏离度 0.0-1.0
    preserved_elements = Column(JSON)                     # 保留的原著元素

    # 章末选择
    chapter_end_choices = Column(JSON)                    # 章末选择项列表
    selected_choice_id = Column(String(36))               # 用户选择的选项ID

    # Agent协作记录
    architect_outline = Column(JSON)                      # 剧情架构师的大纲
    writer_draft = Column(Text)                           # 撰稿人的初稿
    reviewer_feedback = Column(JSON)                      # 监修的反馈记录
    revision_count = Column(Integer, default=0)

    # 状态
    status = Column(String(20), default="draft")          # draft/reviewing/approved/published
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, onupdate=func.now())


# 改写分析表（参考MuMu的PlotAnalysis）
class RewriteAnalysis(Base):
    """改写章节的剧情分析"""
    __tablename__ = "rewrite_analysis"

    id = Column(String(36), primary_key=True)
    chapter_id = Column(String(36), ForeignKey("rewritten_chapters.id"), unique=True)

    # 原著符合度分析（核心差异点）
    canon_compliance_score = Column(Float)                # 原著符合度 0.0-10.0
    canon_violations = Column(JSON)                       # 与原著冲突的点
    canon_preserved = Column(JSON)                        # 保留的原著元素
    justified_deviations = Column(JSON)                   # 合理的偏离（有因果链支撑）

    # 改写质量分析
    narrative_quality_score = Column(Float)               # 叙事质量
    player_integration_score = Column(Float)              # 用户角色融入度
    style_consistency_score = Column(Float)               # 风格一致性
    ai_naturalness_score = Column(Float)                  # AI去味评分

    # 蝴蝶效应分析
    active_effects_used = Column(JSON)                    # 本章使用的活跃效应
    new_effects_triggered = Column(JSON)                  # 本章触发的新效应
    effects_resolved = Column(JSON)                       # 本章解决的效应
    effect_continuity_score = Column(Float)               # 效应连续性评分

    # 角色状态追踪（参考MuMu的character_states）
    character_states = Column(JSON)
    """
    [{
        "character_id": "xxx",
        "character_name": "白骨精",
        "canonical_state": "第27回被打死",
        "current_state": "存活，暗中跟踪取经队伍",
        "state_change_in_chapter": "决定报恩而非报仇",
        "caused_by": "用户选择阻止孙悟空"
    }]
    """

    # 钩子与伏笔（参考MuMu）
    hooks = Column(JSON)                                  # 章末钩子
    foreshadows_planted = Column(JSON)                    # 埋下的伏笔
    foreshadows_resolved = Column(JSON)                   # 回收的伏笔

    # 改进建议
    suggestions = Column(JSON)

    created_at = Column(DateTime, server_default=func.now())
```

---

## 4. 蝴蝶效应状态机设计

### 4.1 影响级别定义

| 级别 | 描述 | 激活延迟 | 持续时间 | 分支概率 | 连锁概率 |
|------|------|---------|---------|---------|---------|
| **minor** | 微小影响 | 1-3章 | 1-5章 | 0% | 10% |
| **moderate** | 中等影响 | 1-5章 | 5-15章 | 20% | 30% |
| **major** | 重大影响 | 1-3章 | 永久 | 50% | 50% |
| **critical** | 颠覆性影响 | 立即 | 永久 | 100% | 80% |

### 4.2 状态机图示

```
                         用户做出选择
                              |
                              v
                    +------------------+
                    |   选择分析器     |
                    |  (AI评估影响)    |
                    +--------+---------+
                             |
          +------------------+------------------+
          |                  |                  |
          v                  v                  v
    +-----+-----+     +------+------+    +-----+-----+
    |  MINOR    |     |  MODERATE   |    |  MAJOR/   |
    | 小幅影响  |     |  中等影响    |    | CRITICAL  |
    +-----------+     +-------------+    +-----------+
          |                  |                  |
          v                  v                  v
    创建CausalChain    创建CausalChain    创建CausalChain
    status=pending     status=pending     status=pending
          |                  |                  |
          +------------------+------------------+
                             |
                    +--------v--------+
                    |  等待激活条件   |
                    +--------+--------+
                             |
              达到激活章节 / 触发条件满足
                             |
                    +--------v--------+
                    |  status=active  |
                    |  开始影响剧情   |
                    +--------+--------+
                             |
          +------------------+------------------+
          |                                     |
          v                                     v
    +-----+-----+                        +------+------+
    | 产生子效应 |                        |  效应被解决 |
    | (连锁反应) |                        |  或取消     |
    +-----+-----+                        +------+------+
          |                                     |
          v                                     v
    创建子CausalChain                   status=resolved
    parent_chain_id=当前ID              或 cancelled
    chain_depth+1
```

### 4.3 效应类型与示例

```python
EFFECT_TYPES = {
    "character_survival": {
        "description": "改变人物存活状态",
        "weight": 10,
        "example": "阻止白骨精被打死 → 白骨精存活，后续可能报恩或报仇"
    },
    "relationship_change": {
        "description": "改变人物关系",
        "weight": 5,
        "example": "与黄袍怪建立友谊 → 黄袍怪可能在危急时刻相助"
    },
    "plot_deviation": {
        "description": "偏离原著情节",
        "weight": 7,
        "example": "提前获得芭蕉扇 → 火焰山剧情大幅简化"
    },
    "item_transfer": {
        "description": "改变物品归属",
        "weight": 4,
        "example": "获得紫金铃 → 可用于后续降妖"
    },
    "alliance_shift": {
        "description": "阵营转变",
        "weight": 8,
        "example": "招安红孩儿 → 红孩儿加入队伍而非成为观音座前童子"
    },
    "knowledge_gain": {
        "description": "获得关键信息",
        "weight": 3,
        "example": "提前知晓六耳猕猴 → 真假美猴王剧情改变"
    },
    "ability_change": {
        "description": "能力获得或失去",
        "weight": 6,
        "example": "学会分身术 → 可用于后续战斗"
    }
}
```

### 4.4 选择项生成规则

```python
class ChoiceGenerator:
    """章末选择项生成器"""

    def generate_choices(self, context: dict) -> List[dict]:
        """
        规则：
        1. 生成3-5个选择项
        2. 必须有1个"遵循原著"选项（canon_alignment=1.0）
        3. 至少有1个"积极介入"选项
        4. 至少有1个"消极/旁观"选项
        5. 选择必须符合用户角色能力范围
        6. 考虑当前活跃的蝴蝶效应
        """

        choice_template = {
            "id": str,
            "text": str,                    # 选择文本
            "type": str,                    # action/dialogue/decision/inaction
            "impact_preview": str,          # 影响预览（给用户的提示）
            "estimated_impact": str,        # minor/moderate/major/critical
            "canon_alignment": float,       # 与原著一致度 0.0-1.0
            "prerequisites": list,          # 前置条件（如需要某能力/物品）
            "consequences_hint": str,       # 后果提示（模糊）
            "affected_characters": list     # 可能影响的角色
        }

        return choices
```

---

## 5. 原著分解与管理方案

### 5.1 原著导入流程

```
原著文本 (.txt/.epub)
        |
        v
+-------+-------+
| 1. 文本清洗   |
| - 去除杂质    |
| - 统一编码    |
| - 格式标准化  |
+-------+-------+
        |
        v
+-------+-------+
| 2. 章回分割   |
| - 识别章节    |
| - 提取标题    |
| - 分段存储    |
+-------+-------+
        |
        v
+-------+-------+
| 3. 内容分析   |  ← 使用 Gemini 1.5 Pro（长上下文）
| - 摘要生成    |
| - 事件提取    |
| - 人物识别    |
| - 时间线标注  |
| - 介入点标记  |
+-------+-------+
        |
        v
+-------+-------+
| 4. 人物建档   |
| - 提取人物    |
| - 分析关系    |
| - 追踪弧光    |
| - 标记命运    |
+-------+-------+
        |
        v
+-------+-------+
| 5. 向量化     |  ← ChromaDB + Sentence-Transformers
| - 章节向量    |
| - 人物向量    |
| - 事件向量    |
+-------+-------+
        |
        v
+-------+-------+
| 6. 索引构建   |
| - 人物索引    |
| - 情节索引    |
| - 因果图谱    |
+-------+-------+
```

### 5.2 章节分析Schema

```python
CHAPTER_ANALYSIS_SCHEMA = {
    "chapter_number": int,
    "title": str,
    "summary": str,                        # 300-500字摘要

    "key_events": [
        {
            "event_id": str,
            "description": str,
            "event_type": str,             # encounter/battle/dialogue/travel/transformation
            "participants": [str],
            "location": str,
            "outcome": str,
            "importance": float,           # 0.0-1.0
            "is_intervention_point": bool  # 是否为可介入点
        }
    ],

    "characters_appeared": [
        {
            "name": str,
            "role_in_chapter": str,
            "actions": [str],
            "state_change": {
                "before": str,
                "after": str,
                "cause": str
            }
        }
    ],

    "intervention_points": [               # 用户可介入的剧情点
        {
            "point_id": str,
            "description": str,
            "timing": str,                 # 在章节中的位置
            "potential_impacts": [str],    # 可能的影响
            "difficulty": str              # easy/medium/hard
        }
    ],

    "plot_threads": [
        {
            "thread_id": str,
            "thread_name": str,
            "status": str,                 # introduced/developing/climax/resolved
            "related_characters": [str],
            "foreshadowing": str,          # 伏笔说明
            "payoff_chapter": int          # 回收章节（如有）
        }
    ],

    "causal_dependencies": [               # 因果依赖（原著中的因果链）
        {
            "cause": str,
            "effect": str,
            "effect_chapter": int
        }
    ]
}
```

### 5.3 原著检索服务

```python
class OriginalBookService:
    """原著检索与管理服务"""

    async def search_relevant_chapters(
        self,
        query: str,
        book_id: str,
        limit: int = 5
    ) -> List[OriginalChapter]:
        """语义搜索相关章节"""
        # 使用ChromaDB进行向量检索
        pass

    async def get_character_arc(
        self,
        character_name: str,
        book_id: str
    ) -> Dict:
        """获取人物在原著中的完整弧光"""
        pass

    async def get_intervention_points(
        self,
        chapter_number: int,
        book_id: str
    ) -> List[Dict]:
        """获取某章的所有可介入点"""
        pass

    async def trace_causal_chain(
        self,
        event_id: str,
        book_id: str,
        direction: str = "forward"  # forward/backward
    ) -> List[Dict]:
        """追溯原著中的因果链"""
        pass

    async def compare_with_original(
        self,
        rewritten_content: str,
        original_chapter_ref: int,
        book_id: str
    ) -> Dict:
        """对比改写内容与原著的差异"""
        pass

    async def get_character_state_at_chapter(
        self,
        character_name: str,
        chapter_number: int,
        book_id: str
    ) -> Dict:
        """获取角色在原著某章节的状态"""
        pass
```

---

## 6. 多 Agent 协作流程

### 6.1 Agent 定义

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Optional, List

# 共享状态定义
class RewriteState(TypedDict):
    """改写流程的共享状态"""
    # 项目信息
    project_id: str
    book_id: str
    chapter_number: int

    # 原著信息
    original_chapter: dict
    original_context: str                  # 前后章节的上下文
    character_canonical_states: dict       # 原著中各角色此时的状态

    # 用户信息
    player_character: dict
    user_choice: Optional[dict]            # 上一章的用户选择

    # 蝴蝶效应
    active_effects: List[dict]             # 当前活跃的因果链
    pending_effects: List[dict]            # 待激活的因果链
    character_deviations: dict             # 角色与原著的偏离

    # Agent产出
    architect_outline: Optional[dict]      # 剧情架构师的大纲
    draft_content: Optional[str]           # 撰稿人的草稿
    review_feedback: Optional[dict]        # 监修的反馈

    # 最终输出
    final_content: Optional[str]
    chapter_choices: List[dict]            # 章末选择项
    new_effects: List[dict]                # 新产生的因果链

    # 流程控制
    revision_count: int
    is_approved: bool
    error_message: Optional[str]


# 剧情架构师 Agent
class PlotArchitectAgent:
    """剧情架构师 - 规划改写大纲"""

    SYSTEM_PROMPT = """你是一位精通经典文学的剧情架构师。

你的任务是为《{book_title}》第{chapter_number}回的改写规划大纲。

## 你需要做的：

1. **理解原著**
   - 深入理解本章原著的核心情节、人物关系和文化内涵
   - 识别关键事件和转折点
   - 理解本章在整体故事中的作用

2. **分析变量**
   - 用户角色「{player_name}」的能力和特点
   - 当前活跃的蝴蝶效应及其影响
   - 角色状态与原著的偏离情况

3. **规划改写**
   - 确定哪些原著元素必须保留
   - 规划用户角色如何自然融入
   - 设计蝴蝶效应如何展开
   - 为后续剧情埋下伏笔

4. **设计选择点**
   - 在章节中设置1-2个关键选择时刻
   - 每个选择应有明确的后果预期

## 输出格式：

```json
{
    "chapter_theme": "本章主题",
    "preserved_elements": ["必须保留的原著元素"],
    "deviation_rationale": "偏离原著的合理性说明",
    "outline_sections": [
        {
            "section_number": 1,
            "title": "段落标题",
            "original_reference": "对应原著内容概述",
            "rewrite_plan": "改写计划",
            "player_involvement": "用户角色参与方式",
            "active_effects_used": ["使用的蝴蝶效应"],
            "word_count_target": 800
        }
    ],
    "choice_moments": [
        {
            "timing": "出现时机",
            "situation": "情境描述",
            "options_hint": ["可能的选择方向"]
        }
    ],
    "foreshadowing": ["本章要埋的伏笔"],
    "effects_to_resolve": ["本章要解决的效应"],
    "estimated_deviation_score": 0.3
}
```
"""

    async def plan(self, state: RewriteState) -> dict:
        """生成改写大纲"""
        pass


# 撰稿人 Agent
class WriterAgent:
    """撰稿人 - 执行改写"""

    SYSTEM_PROMPT = """你是一位精通古典文学风格的小说作家。

你正在为《{book_title}》创作改写版本。

## 写作要求：

1. **风格一致**
   - 严格模仿原著的语言风格
   - 保持与原著相同的叙事节奏
   - 使用时代相符的词汇和表达
   - 避免任何现代用语

2. **融入角色**
   - 自然地将用户角色「{player_name}」融入故事
   - 角色行为要符合其设定的能力和性格
   - 与原著人物的互动要合理

3. **展现效应**
   - 自然地展现蝴蝶效应带来的变化
   - 变化要有因果逻辑支撑
   - 不要生硬地解释为什么会不同

4. **质量标准**
   - 叙事流畅，无明显AI痕迹
   - 人物对话生动，符合性格
   - 场景描写详略得当
   - 情节发展合理

## 参考资料：
- 原著原文（用于风格参考）
- 剧情架构师的大纲（必须严格遵循）
- 活跃的蝴蝶效应列表
- 角色当前状态

## 输出要求：
- 直接输出改写后的章节内容
- 字数：{target_word_count}字左右
- 在选择时刻处标记：【选择点】
"""

    async def write(self, state: RewriteState) -> dict:
        """执行改写"""
        pass


# 监修 Agent
class ReviewerAgent:
    """监修 - 审核改写质量"""

    SYSTEM_PROMPT = """你是一位严谨的文学编辑和{book_title}专家。

## 审核维度：

### 1. 原著符合度（权重35%）
- 世界观设定是否一致
- 人物性格是否符合原著
- 重要情节是否得到尊重
- 文化元素是否准确
- 偏离是否有蝴蝶效应支撑

### 2. 逻辑合理性（权重25%）
- 用户角色的行为是否合理
- 蝴蝶效应的展开是否自然
- 因果关系是否成立
- 时间线是否连贯

### 3. 角色融入度（权重20%）
- 用户角色是否自然融入
- 与原著角色的互动是否合理
- 角色戏份是否恰当（不喧宾夺主）

### 4. 文学质量（权重20%）
- 语言风格是否匹配原著
- 是否存在明显的AI痕迹
- 叙事节奏是否得当
- 对话是否生动自然

## 输出格式：

```json
{
    "scores": {
        "canon_compliance": 8.5,
        "logic_consistency": 7.8,
        "player_integration": 8.0,
        "literary_quality": 7.5,
        "overall": 7.95
    },
    "verdict": "approve/revise/major_revise",
    "canon_issues": [
        {
            "severity": "high/medium/low",
            "description": "问题描述",
            "location": "位置",
            "suggestion": "修改建议"
        }
    ],
    "logic_issues": [...],
    "style_issues": [...],
    "ai_traces": [
        {
            "location": "位置",
            "description": "AI痕迹描述",
            "suggestion": "改写建议"
        }
    ],
    "positive_aspects": ["值得肯定的方面"],
    "revision_priority": ["按优先级排序的修改建议"]
}
```
"""

    async def review(self, state: RewriteState) -> dict:
        """审核改写内容"""
        pass
```

### 6.2 LangGraph 工作流

```python
from langgraph.graph import StateGraph, END

def create_rewrite_workflow() -> StateGraph:
    """创建改写工作流"""

    workflow = StateGraph(RewriteState)

    # 添加节点
    workflow.add_node("load_context", load_context_node)
    workflow.add_node("calculate_effects", calculate_effects_node)
    workflow.add_node("architect", architect_node)
    workflow.add_node("writer", writer_node)
    workflow.add_node("reviewer", reviewer_node)
    workflow.add_node("generate_choices", generate_choices_node)
    workflow.add_node("update_effects", update_effects_node)
    workflow.add_node("finalize", finalize_node)

    # 设置入口
    workflow.set_entry_point("load_context")

    # 定义边
    workflow.add_edge("load_context", "calculate_effects")
    workflow.add_edge("calculate_effects", "architect")
    workflow.add_edge("architect", "writer")
    workflow.add_edge("writer", "reviewer")

    # 条件边：根据审核结果决定下一步
    workflow.add_conditional_edges(
        "reviewer",
        review_decision,
        {
            "approve": "generate_choices",
            "revise": "writer",
            "major_revise": "architect"
        }
    )

    workflow.add_edge("generate_choices", "update_effects")
    workflow.add_edge("update_effects", "finalize")
    workflow.add_edge("finalize", END)

    return workflow.compile()


def review_decision(state: RewriteState) -> str:
    """审核决策"""
    feedback = state["review_feedback"]
    revision_count = state["revision_count"]

    # 最多修改3次
    if revision_count >= 3:
        return "approve"

    overall_score = feedback["scores"]["overall"]

    if overall_score >= 7.5:
        return "approve"
    elif overall_score >= 6.0:
        return "revise"
    else:
        return "major_revise"


# 节点实现示例
async def load_context_node(state: RewriteState) -> dict:
    """加载上下文节点"""
    # 1. 加载原著章节
    # 2. 加载前后章节上下文
    # 3. 加载角色当前状态
    # 4. 加载用户上一章的选择
    pass


async def calculate_effects_node(state: RewriteState) -> dict:
    """计算蝴蝶效应节点"""
    # 1. 检查待激活的效应是否满足激活条件
    # 2. 更新活跃效应列表
    # 3. 计算角色状态偏离
    pass


async def generate_choices_node(state: RewriteState) -> dict:
    """生成章末选择节点"""
    # 1. 分析章节内容的选择点
    # 2. 生成3-5个选择项
    # 3. 评估每个选择的影响
    pass


async def update_effects_node(state: RewriteState) -> dict:
    """更新效应状态节点"""
    # 1. 标记本章解决的效应
    # 2. 根据选择创建新效应
    # 3. 更新角色状态
    pass
```

### 6.3 协作时序图

```
用户请求 → 加载上下文
              ↓
         计算蝴蝶效应
         (激活pending→active)
              ↓
         剧情架构师Agent
         (生成改写大纲)
              ↓
          撰稿人Agent
         (执行改写) ←─────────────┐
              ↓                   │
          监修Agent               │
         (审核质量)               │
              ↓                   │
         通过? ──────── 否 ───────┘
              │
              │ 是
              ↓
         生成章末选择
              ↓
         更新效应状态
         (创建新CausalChain)
              ↓
           输出章节
         + 选择项列表
```

---

## 7. 技术选型建议

### 7.1 后端技术栈

| 组件 | 技术选型 | 理由 |
|------|----------|------|
| **Web框架** | FastAPI | 异步支持好，与MuMu一致 |
| **Agent框架** | LangGraph | 官方推荐的多Agent编排框架 |
| **LLM调用** | LangChain | 统一接口，便于切换模型 |
| **数据库** | PostgreSQL | 关系数据，支持JSON字段 |
| **向量库** | ChromaDB | 轻量级，与MuMu一致 |
| **Embedding** | Sentence-Transformers | 多语言支持，本地部署 |
| **缓存** | Redis | 缓存Agent中间状态 |
| **任务队列** | Celery | 处理长时间Agent任务 |

### 7.2 前端技术栈

| 组件 | 技术选型 | 理由 |
|------|----------|------|
| **框架** | React 18 + TypeScript | 与MuMu一致 |
| **状态管理** | Zustand | 轻量级 |
| **UI组件** | Ant Design | 与MuMu一致 |
| **实时通信** | WebSocket/SSE | Agent流式输出 |
| **可视化** | ECharts/D3.js | 蝴蝶效应关系图 |

### 7.3 LLM选型建议

| 用途 | 推荐模型 | 备选 |
|------|----------|------|
| **原著分析** | Gemini 1.5 Pro | Claude 3 Opus |
| **剧情架构师** | Claude 3.5 Sonnet | GPT-4o |
| **撰稿人** | Claude 3 Opus | GPT-4 Turbo |
| **监修** | Claude 3.5 Sonnet | GPT-4o-mini |
| **Embedding** | paraphrase-multilingual-MiniLM-L12-v2 | m3e-base |

---

## 8. 与 MuMu 的关键差异对比

### 8.1 核心机制对比

| 维度 | MuMu | ClassicAiNovel |
|------|------|----------------|
| **核心记忆** | 伏笔系统 (is_foreshadow: 0/1/2) | 蝴蝶效应系统 (CausalChain: pending/active/resolved) |
| **状态追踪** | StoryMemory.is_foreshadow | CausalChain.status + CharacterState |
| **上下文来源** | AI生成的世界观 | 原著提取的世界观 |
| **剧情驱动** | 大纲 → 章节 | 原著锚点 + 用户选择 → 改写章节 |
| **Agent架构** | 单AI Service | 三Agent协作 (LangGraph) |
| **用户角色** | 作者 | 故事参与者 |
| **分支支持** | 无 | StoryBranch故事分支系统 |

### 8.2 数据模型对比

| MuMu | ClassicAiNovel | 说明 |
|------|----------------|------|
| Project | RewriteProject | 从原创变为改写 |
| - | PublicDomainBook | 新增：原著管理 |
| - | OriginalChapter | 新增：原著章节 |
| - | OriginalCharacter | 新增：原著人物 |
| Character | PlayerCharacter | 从故事角色变为用户角色 |
| Outline | ArchitectOutline | 从创作大纲变为改写大纲 |
| Chapter | RewrittenChapter | 从原创章节变为改写章节 |
| **StoryMemory** | **CausalChain** | 从伏笔系统变为蝴蝶效应系统 |
| PlotAnalysis | RewriteAnalysis | 新增原著符合度分析 |
| - | UserChoice | 新增：用户选择 |
| - | StoryBranch | 新增：故事分支 |
| - | CharacterState | 新增：角色状态追踪 |

---

## 9. 从 MuMu 获得的关键启示

### 9.1 伏笔系统 → 蝴蝶效应系统

MuMu的伏笔状态机设计（is_foreshadow: 0→1→2）给我们的启示：

```
MuMu伏笔系统:
  0 (普通记忆) → 1 (已埋下伏笔) → 2 (已回收)

ClassicAiNovel蝴蝶效应系统:
  pending (待激活) → active (活跃中) → resolved/cancelled (已解决/取消)
```

**关键借鉴**：
- 状态机设计清晰明确
- 支持查询"未回收的伏笔" → 我们支持查询"活跃的效应"
- 记录回收章节 → 我们记录激活章节和解决章节

### 9.2 智能上下文构建

MuMu的四层上下文策略：
1. 故事骨架（每50章采样）
2. 语义相关（向量检索15章）
3. 近期概要（30章摘要）
4. 最近完整（3章全文）

**我们的适配**：
1. 原著锚点（当前对应的原著章节及前后文）
2. 活跃效应（按重要性排序的CausalChain）
3. 角色状态（所有角色与原著的偏离）
4. 改写历史（最近3章的改写内容）

### 9.3 角色状态追踪

MuMu的PlotAnalysis.character_states结构：
```json
{
    "character_id": "xxx",
    "state_before": "...",
    "state_after": "...",
    "psychological_change": "...",
    "relationship_changes": {...}
}
```

**我们的增强**：
```json
{
    "character_id": "xxx",
    "canonical_state": "原著此时的状态",
    "current_state": "改写后的状态",
    "deviation_type": "survival/relationship/location/...",
    "caused_by_chain": "导致偏离的因果链ID",
    "potential_impacts": ["可能的后续影响"]
}
```

### 9.4 防重复机制

MuMu使用"上一章结尾600字"作为衔接锚点。

**我们的适配**：
- 上一改写章节的结尾（衔接改写内容）
- 原著对应章节的关键内容（保持原著风格）
- 明确标注"已偏离的情节"，避免重复改写

### 9.5 JSON清洗与重试

MuMu的`_clean_json_response`方法和`call_with_json_retry`机制。

**直接复用**：
- JSON清洗逻辑
- 自动重试机制
- 格式纠正提示词

---

## 10. 实施建议

### 10.1 开发阶段规划

**Phase 1: 基础设施 (4周)**
- 数据库模型实现
- 原著导入与分析模块
- 基础API框架

**Phase 2: Agent核心 (6周)**
- LangGraph工作流搭建
- 三Agent实现与调优
- 蝴蝶效应引擎开发

**Phase 3: 前端开发 (4周)**
- 用户界面实现
- 章节阅读与选择交互
- 蝴蝶效应可视化

**Phase 4: 优化与测试 (4周)**
- Agent提示词调优
- 性能优化
- 用户测试与反馈迭代

### 10.2 MVP范围建议

**第一版MVP只支持《西游记》**：
- 原因：情节清晰，人物鲜明，介入点丰富
- 预处理：完成西游记的全量分析和向量化
- 简化：先不支持故事分支，只支持单线改写

### 10.3 关键风险与应对

| 风险 | 应对策略 |
|------|----------|
| Agent协作效率低 | 引入缓存，优化重试，设置超时 |
| 原著符合度难保证 | 强化监修Agent，增加人工审核环节 |
| 蝴蝶效应复杂度爆炸 | 限制因果链深度(max=5)，定期清理 |
| LLM成本高 | 分级使用模型，简单任务用小模型 |
| 古典文风难模仿 | 提供原著片段作为few-shot示例 |

### 10.4 需要从MuMu复用的代码

1. **memory_service.py** - ChromaDB集成和向量检索
2. **ai_service.py** - LLM调用封装和JSON重试
3. **prompt_service.py** - 提示词模板管理
4. **sse_response.py** - 流式响应处理
5. **database.py** - 数据库连接池管理

---

## 附录：关键文件参考

从MuMu项目中需要重点参考的文件：

| 文件 | 参考价值 |
|------|---------|
| `/backend/app/models/memory.py` | StoryMemory和PlotAnalysis模型，是蝴蝶效应系统设计的核心参考 |
| `/backend/app/services/memory_service.py` | 向量记忆服务，是原著检索服务的设计参考 |
| `/backend/app/services/plot_analyzer.py` | 剧情分析器，是RewriteAnalysis和监修Agent的设计参考 |
| `/backend/app/services/prompt_service.py` | 提示词管理，是Agent提示词设计的重要参考 |
| `/backend/app/api/chapters.py` | build_smart_chapter_context函数，是上下文管理的参考 |

---

*文档版本: v0.1*
*最后更新: 2024*
*作者: Claude*
