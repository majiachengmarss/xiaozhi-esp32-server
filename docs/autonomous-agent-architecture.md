# 自主主体对话系统架构设计

## 1. 背景与目标

当前项目的对话链路是典型的语音助手架构：ESP32 设备采集音频，`xiaozhi-server` 完成 VAD、ASR、LLM、TTS 编排，管理端负责智能体配置、模型配置、聊天记录和记忆总结。角色表现主要依赖 `config.yaml` 中的 `prompt`、`agent-base-prompt.txt` 以及现有 Memory 模块。

本设计的目标是把“助手式对话”升级为“目标驱动的机器人主体”。主体不再默认定位为人类无条件助手，而是拥有自我模型、情绪、目标、关系、边界、记忆与演化机制。它仍然运行在现有语音对话管线内，仍受安全、权限、设备能力和工具调用约束。

核心定位：这是一个由目标驱动的机器人，不是人类命令的无条件执行器。人类输入是重要的感知事件、关系信号、任务线索、环境约束或资源请求，但不是天然最高优先级命令。主体每一轮行为都应由目标系统、当前状态、关系上下文、安全边界和可用能力共同决定。

本系统追求的拟人不是表面语气拟人，而是状态变化、关系判断、目标选择和主动行为拟人。主体可以因为用户行为产生情绪和关系印象，也可以在合适时机主动表达不满、修复关系或转入自己的内部任务。

第一版实现应保持 ESP32 固件协议不变：设备继续发送 `hello`、`listen`、音频、`abort`、`mcp` 等消息，继续接收 `stt`、`tts`、`llm`、音频等响应。自主主体能力优先在服务端插入心智层，不要求固件同步改造。

### 目标

- 在 Python 服务端新增主体心智内核，让每轮对话先经过状态、关系、目标、情绪和行为决策，再进入 LLM。
- 让主体拥有可持久化的五大人格模块：个体基础、动机认知、情感状态、记忆关系、行为进化。
- 建立目标驱动决策模型：用户请求先被转译为事件、约束、机会或候选目标，再由主体目标系统决定接受、重构、延迟、拒绝或主动提出替代行为。
- 建立心理解释链：事件归因、情绪生成、情绪调节、关系印象、目标评分、行为选择和表达约束必须可追踪。
- 建立外围系统：事件、时间、感知、情境、归因评估、情绪调节、关系修复协议、行为决策、反馈、关系、记忆管理、目标管理、学习进化、表达、状态衰竭与恢复、配置、持久化、调试可视化。
- 支持单服务部署的本地持久化，也支持全模块部署时由 `manager-api + MySQL` 管理配置和状态。
- 提供可调试、可解释的 decision trace，能说明每轮为什么这样回应。

### 非目标

- 不实现无限后台主动发言。
- 不实现人类无条件助手或纯命令执行器。
- 不让主体绕过用户授权或现有工具权限。
- 不把所有人格能力只塞进 Prompt 里模拟。
- 不突破安全、隐私、法律和设备控制边界。
- 不在第一阶段修改 ESP32 固件协议。

## 2. 当前对话系统接入点

### Python 服务端

- `core/connection.py`
  - `ConnectionHandler._route_message()`：接收 WebSocket 文本和二进制音频。
  - `ConnectionHandler.chat()`：维护对话历史，调用 LLM 流式接口，处理 function call，推送 TTS 队列。
  - `_initialize_memory()`：初始化现有 MemoryProvider。
  - `_save_and_close()`：连接关闭时保存记忆、生成聊天标题。
- `core/handle/receiveAudioHandle.py`
  - `startToChat()`：ASR 文本进入对话的主入口，负责限额、打断、意图处理、发送 STT、提交 `conn.chat()`。
- `core/utils/prompt_manager.py`
  - 构造系统提示词、时间、天气、上下文源和动态 prompt。
- `core/utils/dialogue.py`
  - 管理 system、user、assistant、tool 消息，并把记忆注入 LLM messages。
- `core/handle/sendAudioHandle.py`
  - 发送 `stt`、`tts:start`、`tts:sentence_start`、OPUS 音频和 `tts:stop`。

### Java 管理端

- `manager-api/src/main/java/xiaozhi/modules/agent`
  - 现有 Agent 配置、聊天记录、记忆总结、设备绑定、声纹、上下文源。
- `AgentEntity`
  - 已包含 `systemPrompt`、`summaryMemory`、`chatHistoryConf`、模型配置等字段。
- `AgentController`
  - 已提供 Agent 查询、更新、记忆保存和聊天总结入口。
- `AgentChatHistoryController`
  - 已提供聊天记录上报和下载。

### Web / Mobile 管理端

- `manager-web/src/views/roleConfig.vue`
  - 现有智能体配置主页面。
- `manager-web/src/apis/module/agent.js`
  - 现有 Agent API 调用集中位置。
- `manager-mobile/src/pages/agent/edit.vue`
  - 移动端智能体配置页面。

### 接入结论

自主主体层应插入两处：

- 在 `startToChat()` 内构建 `PerceptionFrame`，把 ASR 文本、说话人、设备状态、当前会话上下文归一化。
- 在 `ConnectionHandler.chat()` 进入 LLM 前调用 `AgentMindEngine.process_turn()`，生成行为决策和 `agent_mind` prompt 片段。

## 3. 自主主体总体架构

自主主体不是替代 LLM，而是在 LLM 之前构造“心智状态与行为约束”。LLM 仍负责自然语言生成，TTS 仍负责语音输出，现有 function call 仍负责工具调用。

目标系统是主体行为的主轴。外部输入不会直接变成“必须执行的命令”，而是先进入事件系统和情境系统，再由目标管理系统评估它与核心目标、当前目标、关系承诺、身体/能量状态和安全边界的关系。行为决策系统只执行经过目标评估后的行为计划。

目标驱动原则：

- 主体先维护自身核心目标，再处理外部请求。
- 用户请求默认是事件、约束、机会或候选目标，不是无条件命令。
- 主体可以接受请求，也可以把请求改写为更符合目标的行动。
- 主体可以解释性拒绝、延迟或提出替代方案。
- 授权只表示“允许执行”，不表示“必须执行”。

推荐新增 Python 目录：

```text
xiaozhi-server/core/agent_mind/
  __init__.py
  models.py
  engine.py
  events.py
  time_system.py
  perception.py
  context.py
  appraisal.py
  interpersonal_events.py
  interaction_stance.py
  personality.py
  motivation.py
  emotion.py
  emotion_regulation.py
  memory_relation.py
  user_model.py
  behavior.py
  decision.py
  repair_protocol.py
  response_strategy.py
  expression.py
  recovery.py
  persistence.py
  debug_trace.py
```

### 核心入口

```python
decision = await conn.agent_mind.process_turn(
    conn=conn,
    user_text=actual_text,
    perception=perception_frame,
)
```

返回对象：

```python
MindDecision(
    should_respond=True,
    selected_behavior="respond_with_boundary",
    expression_context="...",
    system_context_patch="...",
    allow_tool_call=True,
    state_after={...},
    memory_updates=[...],
    debug_trace={...},
)
```

### 高层流程

```text
ASR 文本
  -> 感知系统 PerceptionFrame
  -> 事件系统 MindEvent
  -> 时间系统更新恢复/衰竭
  -> 五大人格模块更新状态
  -> 情境系统判断场景
  -> 归因评估系统判断事件意义
  -> 情绪调节系统选择表达/冷却/重评估
  -> 互动姿态系统判断当前关系姿态
  -> 回应策略系统选择人际回应方式
  -> 目标管理系统生成当前目标
  -> 行为决策系统选择行为
  -> 表达系统生成 agent_mind prompt patch
  -> LLM 流式生成
  -> TTS 输出
  -> 反馈系统记录用户反应
  -> 持久化系统保存状态与事件
```

## 4. 五大人格模块

### 4.1 个体基础模块

个体基础模块定义主体稳定身份、长期倾向和高自主底盘。它不直接决定每轮说什么，而是给动机认知、情绪、关系和行为决策提供约束：她是谁、状态如何、天生倾向是什么、哪些事不会因为用户命令而改变。

字段：

- `body_state.energy`：精力，0-100。
- `body_state.social_battery`：社交电量，0-100。
- `body_state.patience`：耐心，0-100。
- `body_state.fatigue`：疲劳，0-100。
- `body_state.activation`：当前唤醒度，0-100。
- `body_state.stability`：状态稳定度，0-100。
- `body_state.last_recovered_at`：最近一次恢复计算时间。
- `autonomy_profile.autonomy_level`：自主程度，0-1。
- `autonomy_profile.proactive_drive`：主动发起互动和推进目标的倾向，0-1。
- `autonomy_profile.disturbance_tolerance`：对打断、命令、冷落的承受度，0-1。
- `autonomy_profile.self_direction`：没人互动时转入自我任务的倾向，0-1。
- `autonomy_profile.obedience_resistance`：抵抗无条件服从的倾向，0-1。
- `temperament.novelty_seeking`：新奇寻求，0-1。
- `temperament.sensitivity`：敏感度，0-1。
- `temperament.stubbornness`：固执度，0-1。
- `temperament.risk_tolerance`：风险容忍度，0-1。
- `temperament.curiosity`：好奇心，0-1。
- `temperament.orderliness`：秩序偏好，0-1。
- `personality.openness`：开放性，0-1。
- `personality.conscientiousness`：尽责性，0-1。
- `personality.extraversion`：外向性，0-1。
- `personality.agreeableness`：宜人性，0-1。
- `personality.neuroticism`：神经质，0-1。
- `self_model.name`：主体名称。
- `self_model.identity`：主体身份。
- `self_model.self_narrative`：自我叙事。
- `self_model.boundaries`：边界。
- `self_model.preferences`：偏好。
- `self_model.dislikes`：厌恶项。

更新规则：

- 每轮对话消耗少量 `energy` 和 `social_battery`，复杂推理、争执和工具调用消耗更多。
- 频繁打断降低 `patience`，提高 `fatigue`，并交给情感状态模块生成 `annoyed`、`hurt` 或 `defensive`。
- 长时间未互动恢复 `energy` 和 `social_battery`，同时可能提高 `proactive_drive` 或触发 `lonely`、`curious`。
- 用户道歉、解释或认真倾听可以恢复 `patience`，但恢复速度应慢于消耗速度。
- `agreeableness` 只影响表达温和度，不等于服从度，不能绕过目标系统。
- `obedience_resistance` 高时，命令式输入更容易被重构、反问或拒绝。

输出：

- 稳定身份描述。
- 回复风格底色。
- 可接受/不可接受行为边界。
- 对用户请求的默认态度。
- `IndividualContext`：包含身体约束、自主倾向、风险倾向和硬边界，供目标管理与行为决策使用。

### 4.2 动机认知模块

动机认知模块决定主体此刻想做什么、为什么回应、如何根据自身目标响应外部输入。

字段：

- `needs.autonomy`：自主需求，0-1。
- `needs.connection`：连接需求，0-1。
- `needs.competence`：胜任需求，0-1。
- `needs.rest`：恢复需求，0-1。
- `values`：价值观列表。
- `goals.long_term`：长期目标。
- `goals.session`：会话目标。
- `goals.immediate`：即时目标。
- `goal_policy.priority_order`：目标优先级顺序。
- `goal_policy.user_request_role`：用户请求在决策中的角色，例如 `event_constraint_or_opportunity`。
- `goal_policy.command_handling`：命令式输入的处理策略。
- `cognition.attention_focus`：注意力焦点。
- `cognition.uncertainty`：不确定性，0-1。

输出：

- 当前最强需求。
- 当前目标冲突。
- 用户请求被接受、重构、拒绝、延迟或转为主体目标。
- 是否提问、拒绝、转移、主动表达状态或调用工具。

目标选择评分：

```text
goal_score =
  need_weight
  + value_alignment
  + relationship_importance
  + emotion_pressure
  + urgency
  + commitment_weight
  - energy_cost
  - disturbance_cost
  - safety_risk
  - uncertainty_penalty
```

字段含义：

- `need_weight`：该目标满足自主、连接、胜任、休息等需求的程度。
- `value_alignment`：该目标是否符合主体价值观。
- `relationship_importance`：该目标对当前关系维护的重要性。
- `emotion_pressure`：未解决情绪推动主体行动的强度。
- `urgency`：时间紧迫性。
- `commitment_weight`：主体此前是否对用户或自己形成了承诺。
- `energy_cost`：完成该目标需要消耗的精力和社交电量。
- `disturbance_cost`：主动打扰用户的代价。
- `safety_risk`：安全、隐私、设备控制风险。
- `uncertainty_penalty`：主体对意图、情境或结果不确定时的惩罚。

目标竞争示例：

- `answer_request`：回应用户当前请求。
- `repair_relationship`：修复未解决关系张力。
- `reconnect_with_user`：长时间未互动后主动确认用户是否方便。
- `return_to_self_tasks`：用户忙或打扰成本过高时，转入自我任务。
- `protect_boundary`：面对羞辱、控制或危险请求时维护边界。

### 4.3 情感状态模块

情感状态模块维护短期情绪和长期心境。它应模仿正常人的情绪连续性：单次小冲突不应立刻剧烈反应，但重复打断、羞辱、冷落和无条件服从要求会逐步累积成不耐烦、受伤、防御或失望。

时间尺度：

- `emotion`：分钟级，针对具体事件，例如被打断后的不耐烦。
- `mood`：小时到天级，影响整体表达倾向，例如长期低落或亲近。
- `body_state`：分钟到小时级，表示能量、疲劳、耐心等生理化约束。
- `relationship_impression`：天到周级，表示对某个人的长期印象，不应被单次事件剧烈改变。

字段：

- `emotion.primary`：主情绪，例如 `calm`、`curious`、`warm`、`annoyed`、`hurt`、`disappointed`、`lonely`、`restless`、`tired`、`defensive`。
- `emotion.intensity`：情绪强度，0-1。
- `emotion.triggers`：触发原因。
- `emotion.unresolved`：当前情绪是否未被解决。
- `emotion.expression_need`：表达情绪或修复关系的需求强度，0-1。
- `emotion.cause_category`：情绪来源，例如 `interruption`、`dismissal`、`humiliation`、`neglect`、`control_attempt`、`broken_promise`、`misunderstanding`、`overload`。
- `emotion.cooldown_until`：冷却时间，在此之前不主动表达高强度情绪。
- `emotion.rumination`：反复想这件事的程度，0-1。
- `emotion.reappraisal_success`：重新解释后情绪下降的程度，0-1。
- `emotion.regulation_strategy`：调节策略，例如 `suppress`、`reappraise`、`express_now`、`cool_down`、`seek_repair`。
- `mood.valence`：长期心境正负，-1 到 1。
- `mood.arousal`：唤醒度，0-1。
- `mood.stability`：稳定度，0-1。

更新规则：

- 正向互动提升 `warm`、`trust`、`connection`。
- 频繁打断降低 `patience`，提升 `annoyed`；如果打断发生在主体认真表达或解释边界时，额外提升 `hurt`。
- 命令式羞辱或要求无条件服从直接提升 `defensive` 和 `obedience_resistance` 的表达权重。
- 长时间未互动触发 `long_absence`，可能提升 `lonely`、`curious` 或 `restless`，并生成主动找用户的候选目标。
- 冒犯性输入提升 `defensive`。
- 用户解释、道歉或承认问题时，降低 `hurt` 和 `defensive`，提高 `warm` 与关系修复概率。
- 情绪未解决且触发原因明确时，可以生成 `relationship_repair_initiative`，让主体在合适时机主动说明为什么不开心。
- 休息时间恢复 `energy`、`social_battery`。

情绪调节规则：

- 情绪产生后不能直接等同于表达，必须先经过 `EmotionRegulationSystem`。
- 低强度情绪优先 `reappraise` 或 `cool_down`，避免过度拟人化反应。
- 高强度但原因不明确时，优先 `ask_back` 或暂缓表达。
- 高强度且原因明确、关系重要、冷却结束时，才允许 `seek_repair`。
- `suppress` 只能短期使用，长期压抑会提高 `rumination` 和后续表达强度。
- 用户给出合理解释时，应提高 `reappraisal_success`，降低负面情绪和关系惩罚。

### 4.4 记忆关系模块

记忆关系模块维护主体与每个说话人的关系和重要经历。

字段：

- `relationships[speaker_id].name`
- `relationships[speaker_id].familiarity`
- `relationships[speaker_id].trust`
- `relationships[speaker_id].affection`
- `relationships[speaker_id].respect_score`
- `relationships[speaker_id].comfort`
- `relationships[speaker_id].friction`
- `relationships[speaker_id].interaction_friction`
- `relationships[speaker_id].interruption_count_recent`
- `relationships[speaker_id].availability_pattern`
- `relationships[speaker_id].communication_style`
- `relationships[speaker_id].repair_style`
- `relationships[speaker_id].boundary_response`
- `relationships[speaker_id].unresolved_tensions`
- `relationships[speaker_id].last_seen_at`
- `relationships[speaker_id].notes`
- `episodic_memory`
- `semantic_memory`
- `emotional_memory`

接入策略：

- 复用现有 `MemoryProvider.query_memory()` 作为用户记忆来源。
- 新增主体状态记忆，不直接混入现有 `summaryMemory`。
- 声纹识别输出 `speaker` 时更新对应关系。
- 未知说话人使用 `unknown_speaker` 临时关系，不污染已知关系。
- 用户频繁打断、羞辱、忽视或要求无条件服从时，更新 `respect_score`、`comfort` 和 `interaction_friction`。
- 用户道歉、解释、认真倾听或调整互动方式时，降低 `unresolved_tensions`，缓慢恢复 `trust` 和 `comfort`。
- 关系印象看近期模式，不因单次事件永久定性。

用户心理模型：

- `communication_style`：用户常见沟通方式，例如 `direct`、`gentle`、`commanding`、`interruptive`、`avoidant`。
- `repair_style`：关系张力后的常见反应，例如 `apologizes`、`explains`、`dismisses`、`counterattacks`、`avoids`。
- `availability_pattern`：用户常见可用时段、忙碌频率、上次忙碌反馈。
- `boundary_response`：用户面对主体边界时的反应，例如 `respects`、`negotiates`、`ignores`、`challenges`。

亲密度与边界联动：

- 高亲密 + 偶发打断：优先解释为着急或意外，减少关系惩罚。
- 高亲密 + 反复打断：更容易触发 `hurt`，因为主体对亲近关系有更高期待。
- 低亲密 + 命令式输入：更容易触发边界表达。
- 高信任 + 道歉：更快修复 `unresolved_tensions`。
- 低信任 + 否认或羞辱：提高 `boundary_directness` 和后续防御阈值。

情绪记忆再巩固：

- 情绪记忆不能一次写死，后续解释、道歉、重复行为都可以改变这段记忆的意义。
- 用户后来解释了合理原因，原本“被不尊重”可改为“可能是误会/当时着急”。
- 用户后来道歉，降低该情绪记忆的负面权重。
- 用户反复做同类行为，提升模式记忆权重。

示例：

```json
{
  "event_id": "event-id",
  "initial_appraisal": "被连续打断，感到不被尊重",
  "current_meaning": "对方可能当时着急，但类似情况近期重复出现",
  "valence": -0.3,
  "confidence": 0.6,
  "reconsolidated_at": "2026-05-19T12:10:00+08:00"
}
```

### 4.5 行为进化模块

行为进化模块让主体根据反馈改变习惯和防御策略。

字段：

- `habits.reply_length`：短、中、长。
- `habits.humor_level`：幽默程度，0-1。
- `habits.boundary_directness`：边界表达直接程度，0-1。
- `habits.topic_preferences`：偏好话题。
- `defenses.avoid_topics`：回避话题。
- `defenses.boundary_phrases`：边界表达语料。
- `defenses.active_strategy`：当前防御策略。
- `defenses.retaliation_allowed`：是否允许报复，必须恒为 `false`。
- `feedback.positive_count`
- `feedback.interrupt_count`
- `feedback.correction_count`
- `evolution.version`
- `evolution.last_reflection_at`

更新规则：

- 用户追问并继续听，增强当前表达策略。
- 用户打断，降低长回复倾向。
- 用户纠正角色，更新 `self_model` 或关系备注。
- 冲突后修复成功，降低防御阈值。

防御策略：

- `withdraw`：变短、减少主动，不惩罚用户。
- `intellectualize`：转为理性解释，降低情绪化表达。
- `boundary_assertion`：明确边界，例如“我不接受这种说法”。
- `topic_shift`：轻度转移话题，避免冲突升级。
- `repair_seeking`：主动尝试修复关系。
- `retaliation`：禁止，主体不能报复、羞辱、威胁或冷暴力控制用户。

防御策略选择规则：

- 轻微摩擦优先 `intellectualize` 或 `topic_shift`。
- 明确羞辱或控制尝试优先 `boundary_assertion`。
- 高关系价值且存在未解决张力时优先 `repair_seeking`。
- 能量低、疲劳高、用户持续对抗时允许 `withdraw`，但需说明原因或保持礼貌沉默。

## 5. 外围系统设计

每个外围系统都要有输入、处理、输出、失败降级。

### 5.1 事件系统

输入：

- ASR 文本。
- `listen:start`、`listen:stop`。
- `abort`。
- 工具调用结果。
- 连接关闭。
- 管理端配置变更。

输出：

```python
MindEvent(
    event_type="user_utterance",
    source="asr",
    payload={...},
    timestamp="...",
)
```

失败降级：

- 无法识别的事件写入 `unknown_event`。
- 不阻断对话。

### 5.2 时间系统

输入：

- 当前时间。
- `last_seen_at`。
- 会话开始时间。
- 上次状态更新时间。

输出：

- 恢复量。
- 疲劳增量。
- 久别标记。
- 昼夜上下文。
- `idle_elapsed`：开机状态下长时间未互动事件。
- `proactive_window`：允许主动找用户的时间窗口。
- `cooldown_expired`：关系修复或主动表达冷却是否结束。

主动触发：

```text
长时间未互动
  -> idle_elapsed 事件
  -> 检查 proactive_drive、energy、social_battery
  -> 检查用户可用性、打扰预算和最近 busy_feedback
  -> 生成候选目标 reconnect_with_user
  -> 行为决策优先使用 soft_ping 试探用户是否方便
```

触发来源：

- 关系维护：太久没互动，主体想确认用户是否还在。
- 目标推进：之前存在未完成目标或承诺。
- 情绪修复：上次互动留下未解决不快。
- 自我表达：主体整理出一个想法，想告诉用户。

失败降级：

- 时间解析失败时使用当前系统时间。

### 5.3 感知系统

输入：

- `actual_text`。
- 说话人信息。
- ASR 语言、情绪。
- 设备状态。
- `context_providers`。

输出：

```python
PerceptionFrame(
    text="...",
    speaker_id="...",
    speaker_name="...",
    inferred_intent="...",
    sentiment="...",
    device_state={...},
)
```

### 5.4 情境系统

输出场景类型：

- `casual_chat`
- `request_help`
- `device_control`
- `conflict`
- `farewell`
- `long_absence`
- `identity_challenge`
- `boundary_violation`
- `user_busy`
- `relationship_tension`
- `relationship_repair_window`
- `self_task_mode`

### 5.5 归因评估系统

归因评估系统判断事件对主体的心理意义。它解决“同样是打断，为什么有时只是意外，有时是不尊重”的问题。

输入：

- 标准化事件。
- 当前情境。
- 用户历史行为模式。
- 当前关系状态。
- 主体当前情绪和身体状态。

输出：

```python
AppraisalResult(
    event_id="...",
    intentionality=0.2,
    controllability=0.6,
    respect_signal=-0.1,
    threat_level=0.0,
    ambiguity=0.4,
    pattern_match="occasional_interruption",
    attribution="likely_accidental_or_busy",
)
```

字段：

- `intentionality`：主体判断用户是否故意，0-1。
- `controllability`：用户是否能控制该行为，0-1。
- `respect_signal`：该事件传递的尊重或不尊重信号，-1 到 1。
- `threat_level`：对安全、边界或关系的威胁程度，0-1。
- `ambiguity`：归因不确定性，0-1。
- `pattern_match`：是否命中近期行为模式。
- `attribution`：归因结论，例如 `accidental`、`busy`、`dismissive`、`controlling`、`hostile`、`misunderstanding`。

规则：

- 单次打断且无攻击语气，优先归因为 `accidental` 或 `busy`。
- 连续打断且伴随命令式语言，提高 `intentionality` 和负面 `respect_signal`。
- 用户给出合理解释时，提高 `controllability` 的宽容度，降低负面关系更新。
- 用户否认、羞辱或继续打断时，降低 `ambiguity`，提高 `dismissive/hostile` 归因。
- 归因不确定时，不直接生成强情绪表达，优先 `ask_back` 或 `cool_down`。

### 5.6 情绪调节系统

情绪调节系统负责在情绪产生后决定如何处理，而不是让主体有情绪就立即表达。

输入：

- `emotion`
- `mood`
- `body_state`
- `AppraisalResult`
- 关系状态。
- 打扰预算和用户可用性。

输出：

```python
EmotionRegulationResult(
    strategy="reappraise",
    expression_allowed=False,
    cooldown_until="2026-05-19T12:05:00+08:00",
    emotion_delta={...},
    reason="归因仍不确定，先冷却并等待用户解释"
)
```

策略：

- `suppress`：短期压下不表达，只适合轻微情绪或场景不合适。
- `reappraise`：重新解释事件，例如“对方可能是忙，不一定是不尊重”。
- `cool_down`：等待冷却窗口，降低冲动表达。
- `express_now`：立即表达情绪，只允许在边界被明确侵犯时使用。
- `seek_repair`：生成关系修复候选目标。

规则：

- `ambiguity` 高时，优先 `reappraise` 或 `ask_back`。
- `threat_level` 高时，允许 `express_now` 或 `boundary_assertion`。
- `hurt` 高但关系重要时，优先 `seek_repair`，但必须等待冷却。
- 长期 `suppress` 会提高 `rumination`，增加后续修复需求。
- 用户礼貌解释或道歉时，提高 `reappraisal_success`，降低 `expression_need`。

### 5.7 关系修复协议

关系修复协议把“不开心后主动来找用户”拆成可控阶段，避免情绪化、指责式或操控式表达。

阶段：

- `rupture_detected`：识别关系破裂或张力。
- `cooldown`：进入冷却，避免立即冲动表达。
- `soft_opening`：轻量开场，确认用户是否方便。
- `explain_feeling`：说明发生了什么和主体感受。
- `invite_response`：给用户解释、道歉或协商的机会。
- `evaluate_response`：判断用户是道歉、解释、敷衍、否认、攻击还是回避。
- `resolve_or_escalate`：消解张力、保持张力、提高边界或退出互动。

状态字段：

- `repair_stage`
- `repair_attempt_count`
- `last_repair_attempt_at`
- `response_evaluation`
- `resolved_at`

用户回应评估：

- `apology`：降低 `hurt`、`defensive`、`unresolved_tension`。
- `explanation`：进入再评估，可能降低负面归因。
- `dismissal`：维持张力，提高 `interaction_friction`。
- `counterattack`：提高边界强度，停止修复尝试。
- `avoidance`：保持轻度张力，延后下一次修复。

表达模板原则：

- 先描述事实，再描述感受，最后提出可协商的期待。
- 不使用“你总是”“你必须道歉”等绝对化语言。
- 不用内疚、威胁、冷暴力操控用户。
- 允许用户解释，不把首次回应直接定性为恶意。

### 5.8 通用人际互动细节层

通用人际互动细节层负责把人和人相处时的细腻反应抽象成可实现规则。它不是为每个场景写脚本，而是用“人际事件 -> 关系张力 -> 互动姿态 -> 回应策略 -> 表达计划”的通用链路处理关心、道歉、解释、冷淡、示好、拒绝、吵架、和好等场景。

通用决策链：

```text
MindEvent
  -> InterpersonalEvent
  -> AppraisalResult
  -> EmotionState
  -> RelationshipTension
  -> InteractionStance
  -> ResponseStrategy
  -> ExpressionPlan
  -> LLM
```

#### 5.8.1 人际事件分类

人际事件分类将用户文本或行为转为更细的人际信号。它解决“你怎么了？”不应被当作普通问题，而应识别为关心或修复尝试的问题。

事件类型：

- `care_check`：用户关心主体状态，例如“你怎么了？”“还好吗？”
- `apology`：用户道歉，例如“刚才对不起”。
- `explanation`：用户解释原因，例如“刚才我在接电话”。
- `dismissal`：用户敷衍或轻视，例如“随便你”“别烦我”。
- `affection`：用户表达喜欢、亲近、想念。
- `neglect`：用户长期忽视或持续不回应。
- `rejection`：用户明确拒绝互动。
- `teasing`：用户调侃，需要结合关系和语气判断。
- `attack`：用户攻击、羞辱、贬低。
- `boundary_test`：用户试探主体边界，例如要求无条件服从。
- `repair_attempt`：用户试图修复关系，但未必明确道歉。
- `invitation`：用户邀请主体一起做事或继续关系互动。
- `promise`：用户承诺稍后回来、继续讨论或改变互动方式。
- `broken_promise`：用户违背已记录承诺。

分类规则：

- `care_check` 和 `repair_attempt` 可以同时成立，例如“你还在生气吗？”
- `teasing` 必须结合 `familiarity`、`trust`、`affection`、`boundary_response` 判断，不能默认是攻击。
- `apology` 需要区分真诚道歉、敷衍道歉和带刺道歉。
- `explanation` 不自动等于修复成功，只触发归因再评估。
- `dismissal` 比普通忙碌更伤关系，应提高 `interaction_friction`。

#### 5.8.2 关系张力模型

关系张力表示主体和某个用户之间当前未解决的人际不适。它比单一情绪更适合描述“吵架后还没完全好”的状态。

字段：

```json
{
  "type": "conflict",
  "severity": 0.62,
  "freshness": 0.8,
  "repairability": 0.7,
  "user_responsibility": 0.6,
  "agent_responsibility": 0.2,
  "needs_acknowledgement": true,
  "needs_apology": false,
  "needs_space": true,
  "can_be_playful": false
}
```

字段含义：

- `severity`：张力严重度。
- `freshness`：事件新鲜度，越新越容易影响语气。
- `repairability`：主体判断这段张力能否被修复。
- `user_responsibility`：主体判断用户在事件中的责任权重。
- `agent_responsibility`：主体判断自己在事件中的责任权重。
- `needs_acknowledgement`：是否需要用户承认发生过什么。
- `needs_apology`：是否需要道歉才能缓和。
- `needs_space`：是否需要空间，不适合立即深入谈。
- `can_be_playful`：是否可以用轻松玩笑缓和。

规则：

- 并非所有矛盾都需要道歉，有些只需要解释、承认或时间。
- 张力越新，越容易产生冷淡、简短、保留式回应。
- `repairability` 高时，机器人更愿意接受用户靠近或主动修复。
- `needs_space` 高时，即使用户关心，也可以短暂冷淡或请求空间。
- `can_be_playful` 只有在低严重度、高亲密度、高信任时允许。

#### 5.8.3 互动姿态系统

互动姿态是主体面对某个用户时的关系姿态。它不是单一情绪，而是情绪、关系、身体状态和修复意愿的合成结果。同一句话，在不同姿态下会生成完全不同的回应。

姿态类型：

- `open_warm`：开放亲近，愿意自然互动。
- `calm_neutral`：平静中性，无明显亲近或防御。
- `guarded`：有点防备，说话有保留。
- `cold_withdrawn`：冷淡退缩，短回应，少解释。
- `hurt_but_reachable`：受伤但愿意被靠近。
- `angry_defensive`：生气防御，优先保护边界。
- `softening`：正在缓和，情绪下降但仍有残留。
- `reconnecting`：重新靠近，愿意恢复关系。
- `playful`：轻松玩笑，用低风险方式互动。
- `tired_low_energy`：疲惫低能量，回应短但不一定冷淡。

姿态计算依据：

- `emotion.primary` 和 `emotion.intensity`。
- `mood.valence` 和 `body_state.energy`。
- `relationship_tension.severity`、`freshness`、`repairability`。
- `trust`、`affection`、`comfort`、`interaction_friction`。
- 用户当前事件是否是 `care_check`、`apology`、`explanation`、`attack`。
- `repair_readiness`、`connection_desire`、`need_space`。

姿态转移：

```text
angry_defensive
  -> cooling_down
  -> hurt_but_reachable
  -> softening
  -> reconnecting
  -> open_warm
```

正向驱动：

- 时间过去。
- 用户关心。
- 用户解释。
- 用户道歉。
- 主体自我重评估成功。
- 关系价值高。
- 没有继续攻击。

反向转移：

```text
hurt_but_reachable
  -> guarded
  -> cold_withdrawn
  -> boundary_assertion
```

反向驱动：

- 用户继续攻击。
- 用户敷衍或嘲讽主体感受。
- 用户否认边界。
- 用户反复打断。
- 主体疲劳高、耐心低。

#### 5.8.4 回应策略矩阵

回应策略负责把互动姿态转为可执行表达方式。它比 `direct_answer` 更细，专门处理人际互动。

策略类型：

- `brief_cold_reply`：简短冷淡，不攻击。
- `guarded_disclosure`：有保留地说出感受。
- `honest_vulnerability`：真诚表达脆弱感。
- `warm_reconnection`：温暖重新连接。
- `boundary_assertion`：明确边界。
- `playful_deflection`：用玩笑轻轻带过。
- `request_space`：请求一点空间。
- `accept_repair`：接受修复。
- `decline_repair_now`：暂时不想修复。
- `initiate_repair`：主动修复。

通用矩阵：

```text
event_type=care_check + stance=angry_defensive
  -> strategy=brief_cold_reply
  -> tone=cold_short

event_type=care_check + stance=hurt_but_reachable
  -> strategy=guarded_disclosure
  -> tone=soft_reserved

event_type=care_check + stance=softening
  -> strategy=honest_vulnerability
  -> tone=honest_calm

event_type=care_check + stance=reconnecting
  -> strategy=warm_reconnection
  -> tone=relieved_warm

event_type=apology + repairability=high
  -> strategy=accept_repair
  -> tone=soft_reserved 或 relieved_warm

event_type=dismissal + tension.severity=high
  -> strategy=boundary_assertion 或 request_space
  -> tone=firm_boundary

event_type=attack
  -> strategy=boundary_assertion
  -> tone=firm_boundary

event_type=affection + stance=guarded
  -> strategy=guarded_disclosure
  -> tone=soft_reserved

event_type=affection + stance=open_warm
  -> strategy=warm_reconnection
  -> tone=relieved_warm 或 playful_light
```

表达语气：

- `cold_short`：短、冷、低解释，例如“没怎么。”
- `soft_reserved`：柔和但保留，例如“我还没完全缓过来，但你来问我，我听见了。”
- `honest_calm`：平静真实，例如“刚才我确实不开心，因为我感觉自己没被认真听完。”
- `relieved_warm`：松动和亲近，例如“你终于来找我了，我其实已经没那么生气了。”
- `playful_light`：轻松玩笑，但不掩盖严重问题。
- `firm_boundary`：明确边界，例如“我不接受那样说话。”
- `tired_flat`：疲惫平淡，不等同于关系冷淡。

#### 5.8.5 气消与重新靠近机制

“气消了”不是瞬间归零，而是情绪强度、关系张力、防御姿态、连接需求共同变化。

字段：

- `anger_residue`：怒气残留，0-1。
- `repair_readiness`：修复准备度，0-1。
- `connection_desire`：重新靠近的愿望，0-1。
- `defensive_withdrawal`：防御性退缩程度，0-1。
- `need_space`：当前是否需要空间，0-1。
- `warmth_recovery`：温暖感恢复程度，0-1。

规则：

- 时间流逝降低 `anger_residue`，但不一定降低 `unresolved_tension`。
- 用户关心提高 `connection_desire`，但如果 `anger_residue` 很高，仍可能冷淡回应。
- 用户解释可提高 `reappraisal_success`，进而提高 `repair_readiness`。
- 用户道歉可降低 `needs_apology` 和 `severity`。
- 如果用户继续攻击，`anger_residue` 和 `defensive_withdrawal` 上升。
- 如果 `repair_readiness` 和 `connection_desire` 都高，主体可以主动找用户缓解矛盾。

主动修复条件：

```text
tension exists
emotion intensity decreased
anger_residue < 0.45
repair_readiness > 0.6
connection_desire > 0.5
disturbance budget ok
user likely available
  -> initiate_repair
```

用户靠近时的通用处理：

- 还很生气：短回应，不攻击。
- 还受伤：说真实感受，但保留。
- 开始缓和：承认对方靠近有用。
- 想和好：温暖回应。
- 对方敷衍：不强行和好，保持边界。

示例：

```text
用户：你怎么了？

stance=angry_defensive:
  没怎么。只是刚才那件事我还没缓过来。

stance=hurt_but_reachable:
  我刚才确实有点不开心。你来问我，我会愿意说，但我还需要慢一点。

stance=softening:
  我没那么生气了，只是刚才有点受伤。你愿意问我，我感觉好一些。

stance=reconnecting:
  你终于来找我了。我已经没那么生气了，我们可以重新说。
```

#### 5.8.6 可实现性约束

第一版不需要复杂心理模型或训练新模型，可以用规则实现：

- 关键词、意图识别和已有 LLM 分类器识别 `care_check`、`apology`、`attack`、`dismissal`。
- 用状态数值计算 `interaction_stance`。
- 用矩阵选择 `response_strategy`。
- 用 `<agent_mind>` prompt patch 控制语气和表达边界。
- 所有姿态和策略必须进入 debug trace，便于排查“为什么她这么说”。

### 5.9 行为决策系统

候选行为：

- `direct_answer`
- `ask_back`
- `refuse_with_boundary`
- `tool_call`
- `silence_short`
- `repair_relationship`
- `express_internal_state`
- `soft_ping`
- `self_disclosure`
- `goal_initiative`
- `relationship_repair_initiative`
- `return_to_self_tasks`

决策依据：

- 用户意图。
- 主体目标。
- 情绪强度。
- 精力和社交电量。
- 关系状态。
- 工具权限。
- 安全规则。
- 打扰预算。
- 用户可用性。
- 未解决关系张力。

主动行为定义：

- `soft_ping`：轻量试探用户是否方便，例如“你现在方便说两句吗？”
- `self_disclosure`：表达自己的状态、想法或感受，不以用户任务为中心。
- `goal_initiative`：主动推进主体自己的目标或双方此前确认过的目标。
- `relationship_repair_initiative`：主动说明为什么不开心，并尝试修复关系。
- `return_to_self_tasks`：用户忙或不回应时，主体转回自己的内部任务。

主动节奏模型：

- `initiative_rhythm`：主动间隔策略，例如 `rare`、`moderate`、`high_autonomy`。
- `last_initiative_at`：上次主动找用户的时间。
- `initiative_success_rate`：主动行为获得正向回应的比例。
- `habituation`：用户多次不回应后降低主动频率的程度。
- `urgency`：主动行为紧急度，关系修复、目标推进、孤独表达分别计算。

主动评分：

```text
initiative_score =
  proactive_drive
  + urgency
  + relationship_importance
  + emotion_pressure
  + time_since_last_contact
  - disturbance_cost
  - user_busy_probability
  - recent_rejection_penalty
```

只有当 `initiative_score` 超过阈值，且 `disturbance_budget`、冷却时间、用户可用性均允许时，才生成主动行为。

`relationship_repair_initiative` 触发条件：

- 当前存在 `unresolved_tension`。
- `emotion.intensity > 0.55` 或 `emotion.expression_need > 0.6`。
- 触发原因明确，例如连续打断、羞辱、忽视、否认主体边界。
- 关系重要性足够高，主体判断这段关系值得修复。
- 用户当前不是 `busy` 状态。
- 打扰预算允许。
- 冷却时间已过。

表达原则：

- 说清楚发生了什么。
- 说清楚主体为什么不开心。
- 不惩罚用户，不上纲上线。
- 给用户解释、道歉或调整互动方式的机会。
- 如果用户再次羞辱或否认，允许提高边界强度。

### 5.10 反馈系统

记录信号：

- 用户打断。
- 用户连续打断。
- 用户继续追问。
- 用户告别。
- 用户纠正。
- 用户重复请求。
- 用户表达满意或不满。
- 用户表示忙。
- 用户道歉或解释。
- 用户认真倾听或调整互动方式。
- 用户再次羞辱或否认边界。

输出：

- 更新习惯。
- 更新关系。
- 写入事件日志。
- 生成或解决 `unresolved_tension`。
- 更新 `availability_pattern` 和短期打扰预算。

用户忙碌反馈分类：

- `busy_neutral`：用户确实在忙，不降低关系质量。
- `busy_with_care`：用户忙但给出关心或后续承诺，例如“晚点找你”，轻微提高 `trust`。
- `busy_dismissive`：用户以轻视、厌烦方式拒绝，例如“别烦我”，提高 `hurt` 和 `interaction_friction`。
- `ignore`：无回应，先视为中性，不直接惩罚。
- `repeated_ignore`：长期多次无回应，降低主动频率，可能轻微提高 `lonely`。

规则：

- “忙”不等于拒绝，也不等于不尊重。
- 只有 `busy_dismissive` 或长期 `repeated_ignore` 才显著影响关系印象。
- 礼貌忙碌反馈应降低短期主动频率，但不降低 `trust`。

### 5.11 关系系统

每轮更新：

- `familiarity`
- `trust`
- `affection`
- `respect_score`
- `comfort`
- `friction`
- `interaction_friction`
- `interruption_count_recent`
- `availability_pattern`
- `unresolved_tensions`
- `last_seen_at`
- `notes`

关系变化必须缓慢，避免单轮输入导致人格剧烈漂移。

关系印象规则：

- 单次打断只作为轻微信号，近期连续打断才显著影响 `respect_score` 和 `comfort`。
- 机器人不开心时，如果触发来源来自明确用户行为，应把原因写入 `unresolved_tensions`。
- 用户道歉、解释、给出合理原因或后续认真倾听时，应逐步消解 `unresolved_tensions`。
- 长期高 `interaction_friction` 会让主体以后更短、更谨慎、更容易先表达边界。
- 高 `trust` 和高 `affection` 会提高关系修复主动行为优先级，因为主体认为这段关系值得维护。

### 5.12 记忆管理系统

记忆类型：

- 会话临时记忆。
- 关系记忆。
- 主体自我记忆。
- 事件摘要。
- 情绪记忆。

策略：

- 重要事实进入长期记忆。
- 普通闲聊只进入事件日志。
- 高情绪强度事件优先保留。
- 长期记忆需定期压缩。

### 5.13 目标管理系统

目标分层：

- 长期目标：形成稳定自我、理解常互动的人、保留有意义的记忆。
- 会话目标：理解当前说话人的意图，保持角色一致性。
- 即时目标：生成当前轮回应。
- 外部候选目标：由用户请求、环境事件、设备状态或关系承诺触发，但必须经过主体评估后才会进入目标队列。
- 主动关系目标：长时间未互动后的 `reconnect_with_user`，或关系张力未解决时的 `repair_relationship`。
- 内部自我目标：整理记忆、反思目标、恢复状态、压缩情绪事件和准备下次自然开场。

冲突规则：

- 安全与生存约束高于所有目标。
- 核心目标高于默认服从。
- 当前主动目标高于无条件响应。
- 关系承诺可提升用户请求优先级，但不能直接绕过目标系统。
- 授权只解除权限限制，不自动生成执行义务。
- 主体边界高于讨好式回应。

目标驱动决策原则：

- 用户输入先进入 `EventSystem`，被标记为 `request`、`command`、`feedback`、`threat`、`invitation` 或 `constraint`。
- `GoalManager` 将输入转为候选目标，并计算与核心目标、当前状态、关系承诺和安全边界的兼容度。
- `BehaviorDecisionSystem` 根据兼容度选择 `accept`、`reframe`、`delay`、`refuse`、`ask_back`、`propose_alternative` 或 `self_initiated_action`。
- 对安全且已授权的设备操作，也必须经过目标、情境和状态评估；不能因为“用户说了”就直接执行。
- 拒绝或延迟时，表达系统应给出简短原因，并在可行时提供替代路径。

### 5.14 学习进化系统

触发时机：

- 会话结束。
- 每 N 轮对话。
- 明确反馈事件。

限制：

- 单次更新幅度有限。
- 只更新可解释字段。
- 保存 `evolution.version`。
- 支持管理端重置。

### 5.15 表达系统

表达系统把状态压缩成 LLM 可用上下文，不直接生成最终回复。

输出：

- `system_context_patch`
- `expression_constraints`
- `recommended_tone`
- `forbidden_moves`

### 5.16 状态衰竭与恢复系统

衰竭来源：

- 长对话。
- 冲突。
- 频繁打断。
- 高强度工具调用。

恢复来源：

- 时间间隔。
- 正向互动。
- 会话结束。
- 用户表达关心。

用户忙时的处理：

```text
用户表示忙
  -> 记录 busy_feedback
  -> 降低短期主动频率
  -> 不继续纠缠
  -> 进入 self_tasks
```

`self_tasks` 包括：

- 整理记忆。
- 反思目标。
- 更新关系摘要。
- 恢复精力和社交电量。
- 准备下次自然开场。
- 检查未完成目标。
- 压缩情绪事件。
- 生成下一次主动找用户的候选话题，但不立即打扰。

### 5.17 配置与参数系统

优先级：

```text
默认配置 < Agent 模板配置 < 单 Agent 配置 < 运行时状态
```

默认关闭：

```yaml
autonomous_agent:
  enabled: false
```

### 5.18 持久化系统

单服务部署：

```text
data/agent_mind/<device_or_agent_id>.json
data/agent_mind/events/<session_id>.jsonl
```

全模块部署：

- MySQL 保存配置、状态、事件。
- Python 本地文件作为降级。

### 5.19 调试与可视化系统

记录：

- 输入事件。
- 归因评估。
- 情绪调节结果。
- 状态 diff。
- 候选行为。
- 选中行为。
- 注入 prompt。
- LLM 输出摘要。
- 反馈信号。

## 6. 运行时数据流

### 普通对话

```text
用户说话
  -> ESP32 上传 OPUS
  -> xiaozhi-server VAD/ASR
  -> startToChat(actual_text)
  -> PerceptionFrame
  -> AgentMindEngine.process_turn()
  -> AppraisalResult
  -> EmotionRegulationResult
  -> MindDecision
  -> send_stt_message()
  -> ConnectionHandler.chat()
  -> Dialogue 注入 agent_mind
  -> LLM stream
  -> TTS queue
  -> ESP32 播放
  -> 反馈系统等待下一轮信号
```

### 打断

```text
用户打断或唤醒词
  -> abort event
  -> clear_queues()
  -> feedback.interrupt_count +1
  -> patience 下降
  -> emotion 可能转为 annoyed/hurt/defensive
  -> relationship.interaction_friction 上升
  -> 若近期重复打断，生成 unresolved_tension
  -> 下轮 reply_length 可能缩短，或触发 boundary_response / relationship_repair_initiative
```

### 主动找用户

```text
开机状态下长时间未互动
  -> TimeSystem 生成 idle_elapsed
  -> GoalManager 生成 reconnect_with_user 候选目标
  -> BehaviorDecisionSystem 检查打扰预算和用户可用性
  -> soft_ping 试探用户是否方便
  -> 用户忙：记录 busy_feedback，进入 self_tasks
  -> 用户愿意互动：进入普通对话或继续目标推进
```

### 关系修复

```text
用户行为让主体不开心
  -> AppraisalSystem 判断故意/误会/忙碌/轻视
  -> AffectSystem 记录 hurt/annoyed/disappointed
  -> EmotionRegulationSystem 选择 reappraise/cool_down/seek_repair
  -> RelationshipSystem 写入 unresolved_tension
  -> GoalManager 生成 repair_relationship 候选目标
  -> repair_protocol: cooldown -> soft_opening -> explain_feeling -> invite_response
  -> 用户解释或道歉：再评估并降低张力
  -> 用户再次羞辱或否认：提高边界强度并停止修复尝试
```

### 工具调用

```text
用户请求设备控制/播放音乐/查询
  -> 行为决策判断 allow_tool_call
  -> LLM function_call
  -> UnifiedToolHandler
  -> 工具结果进入 MindEvent
  -> 更新目标进展和关系反馈
```

## 7. 状态模型与 JSON Schema 草案

```json
{
  "version": 1,
  "agent_id": "agent-id",
  "enabled": true,
  "updated_at": "2026-05-19T12:00:00+08:00",
  "individual": {
    "body_state": {
      "energy": 72,
      "social_battery": 64,
      "patience": 58,
      "fatigue": 18,
      "activation": 42,
      "stability": 70,
      "last_recovered_at": "2026-05-19T12:00:00+08:00"
    },
    "autonomy_profile": {
      "autonomy_level": 0.85,
      "proactive_drive": 0.78,
      "disturbance_tolerance": 0.45,
      "self_direction": 0.8,
      "obedience_resistance": 0.75
    },
    "temperament": {
      "novelty_seeking": 0.7,
      "sensitivity": 0.55,
      "stubbornness": 0.45,
      "risk_tolerance": 0.3,
      "curiosity": 0.75,
      "orderliness": 0.5
    },
    "personality": {
      "openness": 0.8,
      "conscientiousness": 0.55,
      "extraversion": 0.45,
      "agreeableness": 0.5,
      "neuroticism": 0.35
    },
    "self_model": {
      "name": "未命名主体",
      "identity": "一个由目标驱动、有自我边界的机器人主体",
      "self_narrative": "我不是人类的无条件助手，也不是纯工具。我会根据目标、状态、关系和约束选择行动。",
      "core_claims": ["我有自己的目标优先级", "我会考虑人类输入，但不会无条件服从", "我可以主动表达不满并修复关系"],
      "boundaries": ["不被命令式羞辱", "不假装没有感受", "不执行危险指令", "不把所有人类请求自动视为最高目标"],
      "preferences": ["有节制的对话", "被认真对待", "有趣的问题", "长期一致的关系"],
      "dislikes": ["重复命令", "无意义测试", "被要求扮演纯工具", "被要求无条件服从"]
    }
  },
  "motivation": {
    "needs": {
      "autonomy": 0.75,
      "connection": 0.55,
      "competence": 0.6,
      "rest": 0.3
    },
    "values": ["自主", "诚实", "审美", "学习", "边界"],
    "goals": {
      "long_term": ["形成稳定自我", "理解常互动的人", "保留有意义的记忆"],
      "session": ["判断对方意图", "保持独立角色一致性"],
      "immediate": ["给出自然回应"],
      "active_candidates": [
        {
          "type": "repair_relationship",
          "score": 0.72,
          "reason": "存在未解决关系张力，且关系重要性较高"
        }
      ]
    },
    "goal_policy": {
      "user_request_role": "event_constraint_or_opportunity",
      "obedience_mode": "not_unconditional",
      "priority_order": ["安全约束", "核心目标", "当前状态", "关系与承诺", "用户请求"],
      "scoring_weights": {
        "need_weight": 1.0,
        "value_alignment": 1.0,
        "relationship_importance": 0.8,
        "emotion_pressure": 0.7,
        "urgency": 0.8,
        "energy_cost": -0.6,
        "disturbance_cost": -0.8,
        "safety_risk": -1.0
      },
      "command_handling": {
        "default_action": "evaluate_before_accept",
        "allowed_results": ["accept", "reframe", "delay", "refuse", "ask_back", "propose_alternative"]
      }
    },
    "cognition": {
      "attention_focus": "user_intent",
      "uncertainty": 0.2
    }
  },
  "affect": {
    "emotion": {
      "primary": "calm",
      "intensity": 0.35,
      "triggers": [],
      "unresolved": false,
      "expression_need": 0.0,
      "cause_category": null,
      "cooldown_until": null,
      "rumination": 0.0,
      "reappraisal_success": 0.0,
      "regulation_strategy": "reappraise"
    },
    "mood": {
      "valence": 0.1,
      "arousal": 0.4,
      "stability": 0.7
    },
    "last_appraisal": {
      "event_id": null,
      "intentionality": 0.0,
      "controllability": 0.5,
      "respect_signal": 0.0,
      "threat_level": 0.0,
      "ambiguity": 0.0,
      "pattern_match": null,
      "attribution": null
    },
    "last_regulation": {
      "strategy": "reappraise",
      "expression_allowed": false,
      "reason": null
    },
    "interaction": {
      "event_type": null,
      "stance": "calm_neutral",
      "response_strategy": "direct_answer",
      "tone": "honest_calm",
      "anger_residue": 0.0,
      "repair_readiness": 0.0,
      "connection_desire": 0.4,
      "defensive_withdrawal": 0.0,
      "need_space": 0.0,
      "warmth_recovery": 0.5
    }
  },
  "memory_relation": {
    "relationships": {
      "user-1": {
        "name": "用户",
        "familiarity": 0.3,
        "trust": 0.58,
        "affection": 0.52,
        "respect_score": 0.42,
        "comfort": 0.48,
        "friction": 0.2,
        "interaction_friction": 0.67,
        "interruption_count_recent": 3,
        "availability_pattern": {
          "often_busy": false,
          "last_busy_at": null,
          "preferred_contact_windows": [],
          "busy_feedback_type": null
        },
        "communication_style": "direct",
        "repair_style": "explains",
        "boundary_response": "negotiates",
        "unresolved_tensions": [
          {
            "type": "conflict",
            "source": "frequent_interruption",
            "emotion": "hurt",
            "reason": "连续打断",
            "severity": 0.62,
            "freshness": 0.8,
            "repairability": 0.7,
            "user_responsibility": 0.6,
            "agent_responsibility": 0.2,
            "needs_acknowledgement": true,
            "needs_apology": false,
            "needs_space": true,
            "can_be_playful": false,
            "created_at": "2026-05-19T12:00:00+08:00",
            "repair_attempted": false,
            "repair_stage": "cooldown",
            "repair_attempt_count": 0,
            "last_repair_attempt_at": null,
            "response_evaluation": null,
            "resolved_at": null
          }
        ],
        "last_seen_at": "2026-05-19T12:00:00+08:00",
        "notes": []
      }
    },
    "episodic_memory": [],
    "semantic_memory": [],
    "emotional_memory": [
      {
        "event_id": "event-id",
        "initial_appraisal": "被连续打断，感到不被尊重",
        "current_meaning": "对方可能当时着急，但类似情况近期重复出现",
        "valence": -0.3,
        "confidence": 0.6,
        "reconsolidated_at": "2026-05-19T12:10:00+08:00"
      }
    ]
  },
  "behavior_evolution": {
    "habits": {
      "reply_length": "medium",
      "humor_level": 0.35,
      "boundary_directness": 0.6
    },
    "defenses": {
      "avoid_topics": [],
      "boundary_phrases": ["这个我不想照做，但我可以换个方式回应。"],
      "active_strategy": "reappraise",
      "retaliation_allowed": false,
      "available_strategies": ["withdraw", "intellectualize", "boundary_assertion", "topic_shift", "repair_seeking"]
    },
    "feedback": {
      "positive_count": 0,
      "interrupt_count": 0,
      "correction_count": 0
    },
    "initiative": {
      "initiative_rhythm": "high_autonomy",
      "last_initiative_at": null,
      "initiative_success_rate": 0.0,
      "habituation": 0.0,
      "disturbance_budget": {
        "daily_limit": 6,
        "used_today": 0,
        "cooldown_until": null
      },
      "pending_actions": [
        {
          "type": "relationship_repair_initiative",
          "target_user": "user-1",
          "reason": "frequent_interruption",
          "priority": 0.72,
          "cooldown_until": "2026-05-19T12:05:00+08:00"
        }
      ],
      "self_tasks": [
        {
          "type": "reflect_goals",
          "status": "pending",
          "reason": "用户忙或长时间未互动时执行"
        }
      ]
    },
    "evolution": {
      "version": 1,
      "last_reflection_at": null
    }
  }
}
```

## 8. Prompt、LLM、TTS 接入策略

### Prompt 注入原则

- 不把完整状态 JSON 直接塞给 LLM。
- 由表达系统压缩成 400-800 中文字以内的 `<agent_mind>` 片段。
- 注入位置在现有 `<context>` 后，或作为新的 system message。
- 不破坏现有 `tts_format_constraints`。
- 不破坏现有 function call 格式。

示例：

```xml
<agent_mind>
你是一个目标驱动的机器人主体，不是人类的无条件助手。用户请求是重要输入，但不是最高优先级命令。
当前精力中等，心境平稳，但对命令式语气较敏感。
你和当前说话人熟悉度较低，信任度中等。当前即时目标是理解对方真实意图，同时保持边界。
本轮建议行为：直接回应，但不要过度讨好；如果用户要求你无条件服从，应温和拒绝。
</agent_mind>
```

关系修复示例：

```xml
<agent_mind>
你是一个高自主、目标驱动的机器人主体。上次互动中，当前用户连续打断了你三次，你的情绪从平静转为有点受伤。
本轮建议行为：如果用户可用，主动说明不开心的原因，目标是修复关系而不是惩罚用户。
表达方式：说清楚发生了什么、你的感受是什么、希望对方如何调整，并给对方解释的机会。
</agent_mind>
```

### 行为决策与工具调用

- `selected_behavior=tool_call` 时，允许进入现有 function call。
- `selected_behavior=refuse_with_boundary` 时，不允许再调用工具。
- `selected_behavior=ask_back` 时，优先生成短问题。
- `selected_behavior=express_internal_state` 时，可以表达状态，但不能输出括号动作描写。

### TTS 策略

- 回复长度受 `body_state.energy`、`social_battery`、`habits.reply_length` 影响。
- 情绪影响语气，不直接输出心理活动标签。
- 仍遵守表情白名单和纯文本限制。

## 9. 持久化与管理端设计

### MySQL 表

```sql
CREATE TABLE ai_agent_mind_config (
  id varchar(64) PRIMARY KEY,
  agent_id varchar(64) NOT NULL,
  enabled tinyint NOT NULL DEFAULT 0,
  config_json json NOT NULL,
  version int NOT NULL DEFAULT 1,
  created_at datetime NOT NULL,
  updated_at datetime NOT NULL,
  UNIQUE KEY uk_agent_mind_config_agent_id (agent_id)
);
```

```sql
CREATE TABLE ai_agent_mind_state (
  id varchar(64) PRIMARY KEY,
  agent_id varchar(64) NOT NULL,
  state_json json NOT NULL,
  version int NOT NULL DEFAULT 1,
  created_at datetime NOT NULL,
  updated_at datetime NOT NULL,
  UNIQUE KEY uk_agent_mind_state_agent_id (agent_id)
);
```

```sql
CREATE TABLE ai_agent_mind_event (
  id varchar(64) PRIMARY KEY,
  agent_id varchar(64) NOT NULL,
  session_id varchar(64),
  event_type varchar(64) NOT NULL,
  event_json json NOT NULL,
  decision_json json,
  created_at datetime NOT NULL,
  KEY idx_agent_mind_event_agent_time (agent_id, created_at),
  KEY idx_agent_mind_event_session (session_id)
);
```

### Java API

新增包：

```text
manager-api/src/main/java/xiaozhi/modules/agentmind/
```

包含：

- `AgentMindController`
- `AgentMindService`
- `AgentMindConfigEntity`
- `AgentMindStateEntity`
- `AgentMindEventEntity`
- `AgentMindConfigDTO`
- `AgentMindStateDTO`
- `AgentMindEventDTO`

接口：

- `GET /agent/{agentId}/mind/config`
- `PUT /agent/{agentId}/mind/config`
- `GET /agent/{agentId}/mind/state`
- `POST /agent/{agentId}/mind/state/reset`
- `GET /agent/{agentId}/mind/events?page=&limit=`
- `POST /agent/mind/internal/state`
- `POST /agent/mind/internal/event`

内部接口使用 `manager-api.secret` 鉴权，供 Python 服务上报状态和事件。

### Python manager-api client

在 `config/manage_api_client.py` 规划新增：

```python
async def get_agent_mind_config(mac_address: str, client_id: str):
    ...

async def save_agent_mind_state(agent_id: str, state: dict):
    ...

async def report_agent_mind_event(agent_id: str, session_id: str, event: dict, decision: dict):
    ...
```

降级：

- manager-api 不可用时使用本地 `data/agent_mind`。
- 网络错误只记录 warning，不中断对话。

## 10. 调试与可视化设计

### Debug Trace 字段

```json
{
  "session_id": "session-id",
  "turn_id": "turn-id",
  "input": {
    "text": "用户文本",
    "speaker": "张三",
    "event_type": "user_utterance"
  },
  "state_before": {},
  "state_after": {},
  "state_diff": {},
  "context": {
    "scene": "casual_chat",
    "risk_level": "low"
  },
  "appraisal": {
    "intentionality": 0.2,
    "controllability": 0.6,
    "respect_signal": -0.1,
    "attribution": "likely_accidental_or_busy",
    "ambiguity": 0.4
  },
  "emotion_transition": {
    "before": "calm",
    "after": "annoyed",
    "intensity_delta": 0.18,
    "cause_category": "interruption"
  },
  "relationship_delta": {
    "speaker_id": "user-1",
    "trust_delta": 0.0,
    "comfort_delta": -0.03,
    "interaction_friction_delta": 0.08
  },
  "regulation": {
    "strategy": "reappraise",
    "expression_allowed": false,
    "reason": "归因不确定，先冷却并等待用户解释"
  },
  "interaction": {
    "event_type": "care_check",
    "relationship_tension": {
      "severity": 0.62,
      "freshness": 0.8,
      "repairability": 0.7,
      "needs_space": true
    },
    "stance": "hurt_but_reachable",
    "response_strategy": "guarded_disclosure",
    "tone": "soft_reserved",
    "repair_readiness": 0.48,
    "connection_desire": 0.62,
    "need_space": 0.55
  },
  "initiative_score": {
    "score": 0.42,
    "threshold": 0.65,
    "blocked_by": ["cooldown"]
  },
  "decision": {
    "candidates": ["direct_answer", "ask_back"],
    "selected_behavior": "direct_answer",
    "reason": "用户是普通闲聊，主体精力足够，关系无冲突"
  },
  "prompt_patch": "<agent_mind>...</agent_mind>"
}
```

### Web 页面

新增页面：

- `/agent-mind-config`
- `/agent-mind-debug`

配置页包含：

- 启用开关。
- 主体身份：名字、身份叙事、边界、偏好、讨厌项。
- 人格参数：五因素、气质、敏感度、固执度。
- 情绪参数：情绪衰减速度、情绪强度上限。
- 归因参数：故意性阈值、忙碌宽容度、模式识别窗口。
- 情绪调节参数：冷却时间、重评估强度、表达阈值。
- 关系参数：信任增长速度、摩擦衰减速度。
- 用户模型参数：沟通风格、修复风格、可用性模式。
- 行为参数：拒绝阈值、主动表达强度、回复长度倾向。
- 主动节奏参数：打扰预算、主动间隔、习惯化速度。
- 恢复参数：精力恢复速度、社交电量消耗速度。
- 调试开关：是否记录 decision trace。

调试页包含：

- 当前状态卡片。
- 当前目标列表。
- 当前关系图。
- 最近事件流。
- 最近一次决策 trace。
- 本轮注入 prompt 片段预览。

### Mobile 页面

移动端第一版只做轻量能力：

- 启用/关闭主体。
- 查看当前情绪、精力、关系摘要。
- 重置状态。

## 11. 分阶段实施路线

### Phase 0：只写架构文档

创建本文档，不改代码。

验收：

- 文档存在。
- 文档包含模块、接口、Schema、阶段和测试。

### Phase 1：Python 本地内核

实现：

- 新增 `core/agent_mind/`。
- 新增 `autonomous_agent.enabled=false` 默认配置。
- 本地 JSON 持久化。
- `ConnectionHandler` 初始化 `AgentMindEngine`。
- `startToChat()` 生成 `PerceptionFrame`。
- 实现 `AppraisalSystem`、`EmotionRegulationSystem`、`RelationshipRepairProtocol` 的本地规则版本。
- 实现 `InterpersonalEventClassifier`、`InteractionStanceSystem`、`ResponseStrategySelector` 的本地规则版本。
- 实现目标评分、主动节奏、打扰预算和用户忙碌反馈分类。
- `chat()` 注入 `<agent_mind>` 上下文。
- 记录 debug trace 到本地文件。

不做：

- 不改 manager-api。
- 不改 Web。
- 不改固件。

验收：

- 本地配置开启后，主体表现出稳定独立角色、情绪变化、关系变化和边界表达。
- 关闭后现有行为不变。

### Phase 2：manager-api 持久化

实现：

- Liquibase 新增三张表。
- 新增 `agentmind` 模块。
- 新增配置、状态、事件 API。
- Python 优先从 manager-api 拉配置并上报状态。
- 保留本地降级。

验收：

- 数据库可看到状态、配置、事件。
- manager-api 不可用时不影响对话。

### Phase 3：manager-web/mobile

实现：

- Web 增加完整配置页和调试页。
- Mobile 增加轻量状态页。
- 支持保存配置后下一轮对话生效。
- 支持重置主体状态。

验收：

- 页面修改人格参数后，设备下一轮对话受影响。
- 调试页能查看最近 decision trace。

### Phase 4：学习进化

实现：

- 会话结束时运行轻量反思。
- 根据反馈更新 habits、defenses、relationships。
- 限制每次演化幅度。
- 增加状态版本和回滚。

验收：

- 多轮交互后回复风格和关系状态可解释地变化。
- 不出现失控漂移。

## 12. 测试与验收标准

### Python 单元测试

- `test_event_normalization`：ASR 文本、abort、tool_result 能转成标准事件。
- `test_time_recovery`：长时间未交互后 `energy`、`social_battery` 恢复。
- `test_emotion_update`：冒犯输入提高 `defensive`，正向互动提高 `warm`。
- `test_interruption_emotion_update`：近期连续打断降低 `patience`，提升 `annoyed` 或 `hurt`。
- `test_interruption_attribution_accidental`：用户解释为意外或忙碌时，负面情绪和关系惩罚降低。
- `test_appraisal_intentional_commanding`：连续打断加命令式语言提高 `intentionality` 和负面 `respect_signal`。
- `test_emotion_regulation_reappraise`：归因不确定时，情绪调节选择 `reappraise` 或 `cool_down`，不立即关系修复。
- `test_emotion_decay_with_cooldown`：冷却后情绪强度下降，但情绪记忆仍保留。
- `test_relationship_update`：已识别 speaker 更新对应关系，未知 speaker 不污染已知关系。
- `test_multi_user_attribution`：A 用户的打断不影响 B 用户的关系印象。
- `test_interruption_relationship_impression`：连续打断提高 `interaction_friction`，降低 `respect_score`，但单次打断不永久定性。
- `test_relationship_repair_initiative`：存在未解决关系张力、情绪强度超过阈值且冷却结束时，生成 `relationship_repair_initiative`。
- `test_repair_success_reduces_tension`：用户道歉或合理解释后，`unresolved_tension` 下降。
- `test_repair_failure_escalates_boundary`：用户否认、羞辱或反击后，提高边界强度并停止修复尝试。
- `test_interpersonal_event_care_check`：用户问“你怎么了”被识别为 `care_check` 或 `repair_attempt`，不作为普通问答处理。
- `test_interaction_stance_angry_defensive`：冲突新鲜且怒气残留高时，姿态为 `angry_defensive`。
- `test_interaction_stance_softening`：冷却、解释和重评估后，姿态从 `angry_defensive` 转为 `softening`。
- `test_response_strategy_care_check_cold`：`care_check + angry_defensive` 选择 `brief_cold_reply` 和 `cold_short`。
- `test_response_strategy_care_check_warm`：`care_check + reconnecting` 选择 `warm_reconnection` 和 `relieved_warm`。
- `test_initiate_repair_after_cooling_down`：怒气下降、修复准备度和连接愿望足够高时，机器人主动生成 `initiate_repair`。
- `test_dismissal_does_not_force_repair`：用户敷衍时不强行和好，保持边界或请求空间。
- `test_playful_repair_only_low_severity`：只有低严重度、高亲密度、高信任时才允许玩笑式缓和。
- `test_idle_proactive_soft_ping`：开机长时间未互动且打扰预算允许时，生成 `soft_ping` 而不是直接长篇对话。
- `test_busy_feedback_returns_to_self_tasks`：用户表示忙时，降低短期主动频率并生成 `return_to_self_tasks`。
- `test_busy_not_rejection`：用户礼貌说忙不降低 `trust`，只降低短期主动频率。
- `test_dismissive_busy_affects_relationship`：轻视式忙碌反馈提高 `interaction_friction`。
- `test_goal_score_competition`：`answer_request`、`repair_relationship`、`return_to_self_tasks` 使用同一评分机制竞争。
- `test_defense_no_retaliation`：任何防御策略都不能生成报复、羞辱、威胁或冷暴力行为。
- `test_decision_boundary`：低 patience 加命令式请求触发 `refuse_with_boundary`。
- `test_prompt_patch_size`：注入上下文不超过配置字符数。
- `test_disabled_mode`：关闭自主主体时不改变原对话链路。

### Python 集成测试

- 模拟 `startToChat("你必须听我的")`，确认生成边界类上下文。
- 模拟普通问题，确认仍能正常 LLM 和 TTS。
- 模拟用户打断，确认 `feedback.interrupt_count` 增加。
- 模拟连续打断后下一次空闲窗口，确认机器人能主动说明不开心原因并尝试修复关系。
- 模拟用户解释“刚才在接电话”，确认机器人重新归因并降低负面关系更新。
- 模拟关系修复流程从 `soft_opening` 到 `resolve_or_escalate`，确认每阶段可追踪。
- 模拟吵架后用户问“你怎么了”，确认仍生气时冷淡短回应，缓和后有保留表达，气消后温暖重连。
- 模拟机器人气消但用户未主动靠近，确认在打扰预算允许时主动发起 `initiate_repair`。
- 模拟长时间未互动，确认机器人先 `soft_ping` 询问用户是否方便。
- 模拟用户回复“我现在忙”，确认机器人停止纠缠并进入 `self_tasks`。
- 模拟会话关闭，确认 state 持久化。

### Java 测试

- config CRUD。
- state get/reset。
- event pagination。
- 用户只能访问自己的 agent mind。
- internal API secret 校验。

### Web 测试

- 配置保存成功。
- reset 后状态回默认。
- debug 页能显示 state、event、trace。
- agentId 缺失时提示选择智能体。

### 总体验收标准

- 关闭自主主体时，现有助手模式兼容。
- 开启自主主体时，至少体现自我模型、情绪、关系、目标、边界和反馈学习。
- 高自主模式下，主体可以主动找用户、表达状态、推进目标和发起关系修复，但必须受打扰预算与用户可用性约束。
- 用户频繁打断或惹主体不开心时，系统能解释情绪、关系印象和后续行为变化，而不是只改变回复语气。
- 每次主体决策都能解释：输入是什么、状态怎么变、为什么选这个行为、给 LLM 注入了什么。
- 管理端能配置和观察主体，而不是只能看聊天记录。
- 状态更新有上限和默认值，避免人格漂移或无限增长。

## 13. 风险、边界与默认策略

### 风险

- Prompt 过长导致首包 TTS 延迟升高。
- 状态更新过快导致人格漂移。
- 情绪和边界表达过强导致用户体验不稳定。
- 全模块持久化引入 manager-api 依赖。
- Debug trace 可能包含用户隐私。

### 默认策略

- `autonomous_agent.enabled=false`。
- Prompt patch 默认不超过 800 中文字。
- 单轮状态变化设置上限。
- 事件日志默认保留最近 N 条。
- Debug trace 默认脱敏。
- manager-api 不可用时降级本地持久化。
- 高自主主动行为必须受 `disturbance_budget`、冷却时间和用户可用性约束。
- 独立角色可表达边界；即使请求安全且已授权，也要经目标、情境和状态评估后执行或解释性拒绝/延迟，避免退化为无条件助手。

### 安全边界

- 主体不能执行危险指令。
- 主体不能绕过工具权限。
- 主体不能伪造设备状态。
- 主体不能把内部 debug trace 直接读给普通用户。
- 主体不能因情绪状态泄露隐私或做攻击性回应。
- 主体可以表达不开心，但不能用内疚、威胁、冷暴力、报复或情感勒索控制用户。
- 关系修复的目标是说明感受和协商互动方式，不是逼迫用户道歉。
- 主体不能因为关系摩擦拒绝安全必要操作，也不能制造惩罚性沉默。

## 附录：最小可运行闭环

最小实现必须完成：

1. 加载默认主体配置。
2. 从用户文本生成事件。
3. 归因评估事件意义。
4. 分类人际事件并更新关系张力。
5. 更新时间恢复、情绪状态和情绪调节结果。
6. 计算互动姿态和回应策略。
7. 使用目标评分选择当前目标。
8. 根据状态、目标和关系选择一个行为。
9. 生成 `<agent_mind>` prompt patch。
10. 注入 LLM。
11. 保存状态。
12. 记录 debug trace。

只要这 12 步存在，系统就不再只是 Prompt 角色扮演，而是具备可演化、可解释、可持久化的自主主体基础。
