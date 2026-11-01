# AI 健康助理模块设计说明

> 适用范围：小程序 `web-uniapp`、后端 `modules/im`、`cms`、后管 `ui`、相关 SQL。
> 本文覆盖 AI 健康助理（聊天室）、体质分析 / 症状问诊工作流、**决策树题库（版本化 / 后管设计器 / 服务端分支判定）**、病诊记录悬浮 AI 入口、AI 分析报告与解锁、城市定位与城市选择，以及 2026-10 本轮“工作流结束判定 / 0 答题流程 / 悬浮 AI 去重”的控制细节。
> 相关文件：`doc/运行与部署/health-workflow-implementation.md`、`doc/运行与部署/消息处理闭环.md`、`doc/sql/qbank.sql`、`doc/sql/qbank_seed.sql`、`doc/sql/health_workflow.sql`、`doc/sql/health_workflow_seed.sql`、`doc/sql/ai_analysis_report.sql`、`doc/sql/ai_health_assistant_v2.sql`、`doc/sql/city.sql`。

---

## 1. 模块定位

AI 健康助理是“聊天室 + 结构化健康信息采集 + AI 分析报告”的组合：

1. **聊天室**：`pages/discover/chat-room.vue`，页面标题“AI 健康助理”，承载欢迎语、快捷入口、消息列表、输入栏、成员切换。
2. **工作流**：体质分析 `CONSTITUTION`、症状问诊 `SYMPTOM`。服务端驱动逐题采集，答完题后把采集内容交给大模型生成分析报告。
3. **分析报告**：`ai_analysis_report`，支持免费摘要 + 付费/广告解锁完整报告，支持人工审核、追问报告、版本替代。
4. **入口**：健康记录页悬浮 AI、病诊记录页悬浮 AI、健康建议详情、首页会话入口、聊天室快捷按钮。

设计原则：

- **后端是状态唯一事实源**：题号、顺序、必答、选项合法性、轮次上限、去重、完成状态全部由服务端判定。
- **题库与会话分离**：`qb_*` 只描述题目与分支规则；`health_workflow` / `survey_instance` 只保存「冻结快照 + 已答路径」。发布版本不可变，草稿可编辑。
- **服务端计算下一题**：小程序只渲染当前题并提交答案，不接收 `next/rules`；分支无法被客户端绕过。
- **不复用 `chat_session`**：`health_workflow.chat_session_id` 只是弱引用；工作流有独立状态表。
- **cmd 卡片不落库**：`workflow_card` / `workflow_question` / `workflow_analyzing` 只通过 WebSocket 实时推送，刷新后由 `health_workflow` 状态重新派生；`workflow_qa` / `workflow_record_used` 是写入 `chat_message` 的工作流历史投影。
- **报告解锁与流程状态解耦**：报告是否解锁、是否待审核，不影响 `health_workflow.status`。

---

## 2. 三端代码地图

| 端 | 位置 | 作用 |
|---|---|---|
| 小程序 | `web-uniapp/pages/discover/chat-room.vue` | AI 健康助理主页面：入口参数、消息列表、卡片事件、输入栏、成员切换、快照 |
| 小程序 | `web-uniapp/pages/health/quick-record.vue` | 快速记录页；病诊记录 tab 的悬浮 AI 按钮在这里分流 |
| 小程序 | `web-uniapp/pages/index/health.vue` | 健康记录主页面；无参悬浮 AI 入口 |
| 小程序 | `web-uniapp/pages/discover/components/chat-msg-bubble/chat-msg-bubble.vue` | 按 `msg.type` / `msg.renderType` 分发卡片 |
| 小程序 | `web-uniapp/pages/discover/components/chat-msg-card/workflow-card.vue` | 通用工作流动作卡（守卫 / 记录确认 / 开始） |
| 小程序 | `web-uniapp/pages/discover/components/chat-msg-card/workflow-question-card.vue` | 逐题问答卡 |
| 小程序 | `web-uniapp/pages/discover/components/chat-msg-card/workflow-qa-card.vue` | 历史只读问答卡 |
| 小程序 | `web-uniapp/pages/discover/components/chat-msg-card/analysis-report-card.vue` | AI 分析报告卡（锁定/审核中/已审核/已解锁/已更新） |
| 小程序 | `web-uniapp/pages/discover/components/chat-input-bar/chat-input-bar.vue` | 文本/语音输入、引用、成员选择 |
| 小程序 | `web-uniapp/pages/discover/composables/*.js` | AI 助理页面拆分后的状态逻辑：滚动/语音/快照/调研/工作流/记录选择 |
| 小程序 | `web-uniapp/composables/useCity.js` | 城市状态单例、定位授权、城市树缓存 |
| 小程序 | `web-uniapp/pages/user/city.vue` | 城市选择页（热门地区 / 省份两栏） |
| 小程序 | `web-uniapp/pages/index/user.vue` | 「我的」右上角城市胶囊 |
| 小程序 API | `web-uniapp/sheep/api/workflow/index.js` | 病诊记录“是否已咨询”查询 |
| 小程序 API | `web-uniapp/sheep/api/analysis/index.js` | 报告详情、状态、付费/广告解锁 |
| 小程序 API | `web-uniapp/sheep/api/city/index.js` | 城市树、逆地理编码 |
| 后端 `im` | `workflow/WorkflowService.java`、`workflow/WorkflowServiceImpl.java` | 工作流入口、答题、记录确认、收敛、默认入口、恢复/守卫 |
| 后端 `im` | `qbank/**`（entity/mapper/model/eval/service/controller） | 决策树题库：版本、节点、选项、分支规则、`GraphEvaluator`、`GraphValidator`（含 `RECORD` 记录确认节点约束）、后管 API |
| 后端 `im` | `cmd/SurveyStartCmdAction.java`、`cmd/SurveyAnswerCmdAction.java` | 补充题库服务端逐题（`SURVEY_START` / `SURVEY_ANSWER`） |
| 后端 `im` | `cmd/ConstitutionCmdAction.java`、`SymptomCmdAction.java` | 体质分析/症状问诊入口命令 |
| 后端 `im` | `cmd/WorkflowAnswerCmdAction.java`、`WorkflowRecordCmdAction.java` | 逐题作答、记录确认回调（`WORKFLOW_RECORD_CHOICE` 支持 `note` 补充说明） |
| 后端 `im` | `cmd/WorkflowDismissCmdAction.java` | 跳过自动弹卡（`WORKFLOW_DISMISS` → `abandon`） |
| 后端 `im` | `cmd/EnterChatAction.java`、`CheckEntryAction.java` | 进入聊天室默认入口、重新进入轻量检查 |
| 后端 `im` | `cmd/DefaultCmdAction.java` | 通用快捷按钮（guide_list / qrcode / text） |
| 后端 `im` | `controller/api/WorkflowApiController.java` | `GET /im/workflow/symptom/analyzed` |
| 后端 `im` | `controller/api/AnalysisReportApiController.java` | 报告状态/详情/付费解锁/广告解锁 |
| 后端 `im` | `analysis/AnalysisReportService*.java` | 报告生成、审核、解锁、追问、替代 |
| 后端 `im` | `agent/task/*` | 合成 agent 任务、调用大模型、生成报告 |
| 后端 `cms` | `controller/api/ChatApiController.java` | 会话、消息、快捷按钮、引导问题 |
| 后端 `cms` | `entity/City.java`、`mapper/CityMapper.java` | 城市字典表 |
| 后端 `cms` | `service/ICityService.java`、`service/impl/CityServiceImpl.java` | 城市树组装 |
| 后端 `cms` | `service/TencentMapService.java` | 腾讯位置服务逆地理编码 |
| 后端 `cms` | `controller/api/CityApiController.java`、`GeoApiController.java` | `GET /city/tree`、`GET /geo/reverse` |
| 小程序 | `web-uniapp/pages/discover/components/chat-drawer/chat-drawer.vue` | AI 助理左侧抽屉：账户/城市/历史咨询/报告 |
| 小程序 | `web-uniapp/pages/discover/composables/useChatAccountContext.js` | 账户切换代际守卫 |
| 小程序 | `web-uniapp/composables/useAccountCity.js` | 健康档案账户级城市读写 |
| 后端 `im` | `flow/FlowOrchestrator.java`、`flow/FlowLlmService.java`、`flow/FlowIntentResult.java` | 自由咨询意图流程与结构化校验 |
| 后端 `im` | `routing/AiModelRouteResolver.java`、`entity/AiModelRoute.java` | 阶段模型路由（意图/报告） |
| 后端 `im` | `consult/ConsultationHistoryService.java` | 历史咨询按自然日聚合 |
| 后端 `cms` | `account/AccountScopeService.java` | 健康档案账户作用域解析 |
| 后端 `cms` | `controller/api/HealthRecordCityApi.java` | 账户级城市接口 |
| 后管 `ui` | `src/views/ai-model-route/index.vue` | 模型路由配置 |
| 后管 `ui` | `src/views/workflow/session/index.vue`、`src/views/qbank/bank/*` | 工作流轨迹、题库列表/版本/决策树设计器（旧 `src/views/survey/*` 保留但维护页菜单已隐藏） |
| SQL | `doc/sql/qbank.sql`、`qbank_seed.sql`、`qbank_record_node.sql`、`health_workflow*.sql`、`ai_analysis_report*.sql`、`ai_health_assistant_v2.sql`、`cmd.sql`、`city.sql` | 决策树题库建表/种子/存量迁移、工作流、报告、题库版本、快捷按钮、城市字典 |

---

## 3. 总体链路

### 3.1 入口一览

| 入口 | 文件 / 方法 | 传参 | 行为 |
|---|---|---|---|
| 健康记录页悬浮 AI | `pages/index/health.vue` `goAiMedical()` | 无 | 打开 `/pages/discover/chat-room`，走 `ENTER_CHAT` 默认入口 |
| 病诊记录 tab 悬浮 AI | `pages/health/quick-record.vue` `onAiFabTap()` | 先查“是否已咨询”；未咨询时 `?cmd=SYMPTOM_CONSULT&recordId&dataId` | 见 §8 |
| 健康建议详情 | `pages/index/health.vue` `goChatFromAdvice()` | `sessionId` (+ `adviceId`) | 预填“关于这条健康建议我想进一步了解...” |
| 首页医生卡 | `pages/index/index.vue` `goChatRoom()` | `sessionId` | 打开当前会话 |
| 聊天室快捷按钮 | `chat_quick_button` | `CONSTITUTION_ANALYSIS` / `SYMPTOM_CONSULT` | 服务端命令入口 |
| 「我的」城市胶囊 | `pages/index/user.vue` `goCityPage()` | 无 | 进入 `/pages/user/city` 城市选择页 |
| 引导提问 | `chat_guide_question` | `entryTag` | `guide_list` 卡片，点击后发文本或 cmd |

### 3.2 消息链路

```
小程序 chat-room
  │  连接 WebSocket（ensureChatConnection）
  ├─ ENTER_CHAT ─────────────► EnterChatAction ─► WorkflowService.defaultEntry
  ├─ CHECK_ENTRY ────────────► CheckEntryAction ─► WorkflowService.checkNewEntry
  ├─ CONSTITUTION_ANALYSIS ──► ConstitutionCmdAction ─► WorkflowService.enter
  ├─ SYMPTOM_CONSULT ────────► SymptomCmdAction ─────► WorkflowService.enter
  ├─ WORKFLOW_ANSWER ────────► WorkflowAnswerCmdAction ─► submitAnswer
  ├─ WORKFLOW_RECORD_CHOICE ─► WorkflowRecordCmdAction ─► chooseRecord
  ├─ WORKFLOW_DISMISS ───────► WorkflowDismissCmdAction ─► abandon
  ├─ DOCTOR_SUGGEST ─────────► DefaultCmdAction ─► 中医推荐（city/province 仅记录，不改变回复）
  ├─ SURVEY_START ───────────► SurveyStartCmdAction ─► 补充题库下发当前题
  ├─ SURVEY_ANSWER ──────────► SurveyAnswerCmdAction ─► 保存答案并下发下一题
  └─ 普通文本/语音/图片 ─────► Text/Voice/Image handler ─► Agent 管线
                                      │
                                      ▼
                             CmdMessagePublisher / WS 推送
                                      │
                                      ▼
                        chat-msg-bubble 按 renderType 渲染卡片
```

### 3.3 渲染类型

| `renderType` | 组件 | 说明 |
|---|---|---|
| `workflow_card` | `workflow-card.vue` | 通用动作卡：守卫（继续 / 重新开始 / 暂不需要）、记录确认（使用 / 不使用；**起始节点为 `RECORD` 时由题库驱动文案与按钮**，旧冻结图回退硬编码）、入口（开始 / 暂不需要） |
| `workflow_question` | `workflow-question-card.vue` | 逐题卡：SINGLE / MULTI / TEXT，支持补充输入；图模式带 `nodeKey`，总题数未知时只显示「第 N 题」 |
| `workflow_qa` | `workflow-qa-card.vue` | 历史只读问答投影，随 `chat_message` 历史加载 |
| `workflow_record_used` | `workflow-record-used-card.vue` | 症状问诊**历史**投影（仅刷新后展示，不实时推送）：记录日期 / 主要症状 / 病因诊断 / 补充说明；不显示红色“已使用该记录”标签，选中态由记录确认卡上的绿色 ✅ 表示 |
| `workflow_analyzing` | 文本提示 + 输入锁定 | “正在为您分析，请稍候…”；收到下一张卡/报告/文本后解除 |
| `analysis_report` | `analysis-report-card.vue` | AI 分析报告卡；点击进入 `pages/health/analysis-report` |
| `guide_list` | `guide-list-card.vue` | 引导提问列表 |
| `survey_intro` / `survey_question` | 对应 survey 卡 | 既有 AI 自驱调研链路 |
| `qrcode` / `card` / `text` | 对应卡 | 通用 cmd 渲染 |

> 已下线的旧卡片：`diagnosis_card`（近 30 天病诊记录多选）及其后端 `diagnos_all` 链路已删除。聊天室仅在加载历史/快照时清理本地残留的 `diagnosis_card` 消息（`LEGACY_CARD_TYPES`），避免渲染原始 JSON。

---

## 4. 数据模型

### 4.1 `health_workflow`

一次信息采集过程的完整记录。

| 字段 | 说明 |
|---|---|
| `biz_code` | `CONSTITUTION` 体质分析 / `SYMPTOM` 症状问诊 |
| `status` | `0 IN_PROGRESS` / `1 COMPLETED` / `2 ABANDONED` |
| `current_round` | 已保存答案数（轮次） |
| `current_question_no` | 当前题目序号（已答数 + 1）；**记录确认阶段 / 分析阶段为 `NULL`** |
| `current_node_key` | 当前待答节点 key（图模式）；收敛 / 放弃时置 `NULL` |
| `plan_json` | **仅旧会话使用**：线性题库快照（数组：题号/题干/类型/选项/必答） |
| `plan_graph_json` | 决策树题库冻结快照（`{format,bankCode,version,startNodeKey,nodes,rules,extras}`）；为空 = 旧线性流程 |
| `bank_version` | 创建时题库发布版本快照（`qb_version.version`） |
| `context_json` | 采集上下文；症状问诊包含 `candidateDataId/candidateRecordDate/candidateSymptom/candidateDiagnosis/candidatePinned`；记录确认卡直接输入时含 `recordNote`（补充说明）；完成后包含 `analysisInput/sourceDataId/bankVersion` |
| `source_data_id` | 本次症状问诊**实际使用**的 `health_data.data_id`；不使用记录时为 `NULL` |
| `prompt_template_code/version` | 结果生成使用的提示词模板 |
| `last_activity_at` | 最后活动时间；用于 24h 恢复窗口与自由咨询的流程活跃度展示 |
| `completed_at` / `abandoned_at` | 完成/放弃时间 |

> **关键控制**：`currentQuestionNo` 使用 `@TableField(updateStrategy = FieldStrategy.IGNORED)`。MyBatis-Plus 默认 `NOT_NULL` 不会把 `null` 写回数据库，曾导致“已完成收集”仍保留旧题号。现在 `converge` / `abandon` 的 `setCurrentQuestionNo(null)` 会真正落库，`isAnalysisPending` 才能可靠判断。

### 4.2 `health_workflow_answer`

逐题作答事实源，`uk(workflow_id, round_no)` 幂等。

- 保存题干/题型/来源（BANK/MODEL）/选项快照/已选编号/自由输入/原始输入/本轮模型输出。
- 图模式额外保存 `node_key`（作答节点），`question_no` 仅作「第 N 题」的序号。
- 每次首次插入成功后，额外写入一条 `chat_message`：`msg_type=cmd`、`render_type=workflow_qa`，作为历史只读投影；不实时推送。

### 4.3 `health_entry_log`

自动入口幂等表，`uk(dedup_key)`。

- 去重键 = `userId:entryCode`。
- 典型 entryCode：`CONSTITUTION:v<bankVersion>:<recordId>`、`SYMPTOM:REC:<dataId>`。
- 保证刷新/重复进入不会重复插入同一入口卡。

### 4.4 题库（决策树，`qb_*`）

迁移脚本：`doc/sql/qbank.sql`（建表 + 会话表增量）、`doc/sql/qbank_seed.sql`（三套题库种子 + 后管菜单）。

| 表 | 说明 |
|---|---|
| `qb_bank` | 题库主体：`bank_code`(唯一)、`name`、`description`、`biz_type`(`CONSTITUTION`/`SYMPTOM`/`AI`)、`enabled`、`current_version` |
| `qb_version` | 版本：`version`、`status`(`DRAFT`/`PUBLISHED`/`ARCHIVED`)、`start_node_key`、`revision`(草稿乐观锁)、`published_at` |
| `qb_node` | 节点：`node_key`(版本内唯一、创建后不可改)、`node_type`(`SINGLE`/`MULTI`/`TEXT`/`RECORD`)、`title`、`required`、`allow_input`、`input_placeholder`、`next_node_key`、`sort_no` |
| `qb_option` | 选项：`option_key`、`label`、`next_node_key`（仅单选 / RECORD 可配；MULTI 用规则）、`sort_no` |
| `qb_rule` | 分支规则：`from_node_key`、`to_node_key`、`priority`、`when_json`（条件 DSL） |

运行时：

- 只读取「启用 + 最新 `PUBLISHED`」版本；`health_workflow.plan_graph_json` 冻结整图，进行中的会话不受新版本影响。
- 体质分析按 `bank_version` 判断是否完成（`hasCompleted`）；入口去重键 `CONSTITUTION:v{n}:{recordId}` 不变。
- 旧 `survey_report/question/option` 已由 `doc/sql/qbank_cleanup.sql` 删除（连同旧后管页面 / 接口 / 菜单）；`survey_instance` 保留（图模式状态 + 历史问答），其 `report_id` 列已删除；「补充问答记录」保留只读。
- 旧 AI 题库若未迁入 `qb_bank`，需在后管重建（种子只迁移了 `CONSTITUTION` / `SYMPTOM_INTAKE` / `AI_INTAKE_DEMO`）。

冻结快照 JSON（`plan_graph_json` / `survey_instance.state_json`）：

```json
{
  "format": "graph",
  "bankCode": "SYMPTOM_INTAKE",
  "version": 2,
  "startNodeKey": "s0",
  "nodes": [
    { "key": "s0", "type": "RECORD", "text": "找到您最近的一条病症记录，是否基于它开始问诊？",
      "required": true, "allowInput": false, "next": "s1",
      "options": [
        { "no": "use", "text": "使用该记录", "next": "s5" },
        { "no": "skip", "text": "不用，我要重新描述", "next": "s1" }
      ] },
    { "key": "s5", "type": "TEXT", "text": "是否有更多补充说明？",
      "required": false, "allowInput": true, "next": "s2", "options": [] },
    { "key": "s1", "type": "TEXT", "text": "请描述一下您当前的不适症状。",
      "required": true, "allowInput": true, "next": "s2", "options": [] },
    { "key": "s2", "type": "SINGLE", "text": "这个症状持续多久了？", "next": "s3",
      "options": [{ "no": "A", "text": "1天以内", "next": "s3" }] }
  ],
  "rules": [
    { "from": "s3", "to": "s4", "priority": 100,
      "when": { "all": [ { "node": "s3", "op": "in", "values": ["A"] } ] } }
  ],
  "extras": []
}
```

- **`RECORD` 记录确认节点**：只能作为图起始节点，一个版本最多 1 个；必须恰好 2 个选项，`option_key` 固定小写 `use` / `skip`（文案可配）；`use` 跳转到“使用记录”后的首题（如 `s5`），`skip` 跳转到重新描述首题（如 `s1`）；节点 `next_node_key` 为**无候选记录时的兜底下一题**（如 `s1`）；禁止配置 `qb_rule.from = RECORD`（分支只由选项跳转决定）。
- **记录确认不落答案**：`RECORD` 的选择不写 `health_workflow_answer`（`round_no` 1 基），只写 `context_json.recordUsed` 与 `recordNote`；分支由 `firstQuestion` 用合成答案 `use`/`skip` 走同一套求值器，首题仍为 `round=1`。
- **旧图兼容**：冻结图起始节点不是 `RECORD` 时（含 CONSTITUTION 与升级前会话），继续使用硬编码记录确认卡。

- `extras`：模型追问 / AI 补充题的线性尾巴，不属于题库版本；图路径走完后按顺序出题。
- 出站给小程序只取 `key/type/text/required/allowInput/inputPlaceholder/options[{no,text}]`，不泄露 `next/rules`。
- **序列化陷阱**：`GraphNode` 的便捷判断必须**改名**为 `isTextType()` / `isMultiType()` / `isRecordType()` 并加 `@JsonIgnore`。只给 `isText()` 加 `@JsonIgnore` 会把整个 `text` 属性（含 `getText()`）一起忽略，导致 `plan_graph_json` / `state_json` 里节点没有 `text`、`health_workflow_answer.question_text` 为 `NULL` 插入失败、问答历史全丢。
- **作答落库**：`saveAnswer` 只把唯一键冲突（`DuplicateKeyException`）当幂等；其它异常必须记录并抛出，终止本轮推进，禁止“答案没落库但流程继续完成”。
- **写入时间**：`FastcmsMetaObjectHandler.insertFill` 只填 `createdAt`，`qb_bank / qb_version / qb_node / qb_rule` 插入前必须显式设置 `createdAt/updatedAt`（表字段 `NOT NULL`）。

条件 DSL（纯 Java 求值，禁止脚本）：

- 组合：`{"all":[...]}` / `{"any":[...]}` / `{"not":{...}}`。
- 叶子：`{"node":"s2","op":"in","values":["A"]}`；操作符 `answered / in / allIn / notIn / eq / contains / gte / lte`。
- 未作答或节点不存在 → `false`；TEXT 上的 `allIn` / `notIn` → `false`；空文本视为未作答。

下一节点优先级（固定）：

1. 当前节点命中的 `qb_rule`（`priority` 降序，第一个命中）；
2. 单选中该选项的 `option.next`；
3. 节点的 `next`；
4. 未作答的 `extras`（按顺序）；
5. 结束。

> `RECORD` 起始节点不走答题：`firstQuestion` 用合成答案 `use` / `skip` 进入上述同一套优先级（因此 `option.next` 决定首题；规则被禁止），并把解析出的节点写入 `current_node_key`。

约束：MULTI 选项禁止配置 `next`（发布校验 ERROR）；RECORD 只允许作起始节点、最多 1 个、必须恰好 `use`/`skip` 两个选项、必须配置无记录兜底 `next`，且禁止配置 `qb_rule.from = RECORD`；发布前校验起始节点 / 重复 key / 悬空引用 / 环；运行时最多推进 200 步。

### 4.5 `ai_analysis_report`

| 字段 | 说明 |
|---|---|
| `source` | `WORKFLOW` / `SURVEY` / `FOLLOWUP` |
| `source_id` | `workflow_id` / `survey_instance_id` / 父报告 id |
| `title` | 体质分析报告 / 症状问诊报告 / 健康补充分析报告 |
| `summary` | 免费预览（Markdown） |
| `content` | 完整报告（Markdown） |
| `status` | `0 GENERATED` / `1 PENDING_REVIEW` / `2 UNLOCKED` / `3 DRAFT` / `4 SUPERSEDED` |
| `review_status` | `0 NONE` / `1 PENDING` / `2 APPROVED` |
| `unlock_channel` | `PAY` / `AD` / `FREE` |
| `parent_report_id` / `superseded_by` | 追问报告链 / 新版本替代 |
| `record_id` / `flow_id` | 健康档案账户 / 自由咨询流程（`source=FLOW`；追问报告 `flow_id` 为空） |

> 2026-10 增量：`chat_message` / `agent_message_task` / `ai_analysis_report` / `survey_instance` 增加 `record_id`；任务/报告/问卷增加 `flow_id`；新增 `ai_chat_flow`（自由咨询状态机）与 `ai_model_route`（阶段模型路由）。详见 §17。

---

## 5. 工作流生命周期与控制规则（核心）

### 5.1 状态与阶段

| 状态/阶段 | 判定 | 用户可见 |
|---|---|---|
| 记录确认阶段 | `status=0`、`current_round=0`、`current_question_no=NULL`、0 答题 | 记录确认 `workflow_card` |
| 逐题阶段 | `status=0`、`current_question_no != NULL` | 当前题 `workflow_question` |
| 异步抽取阶段（症状） | `status=0`、`current_round>0`、`current_question_no=NULL` | 等待模型判断是否追问 |
| 分析等待阶段 | `status=0`、`current_round>0`、`current_question_no=NULL` | 正常应立即进入 `COMPLETED`；异常时保持 `IN_PROGRESS`，由用户显式结束 |
| 已完成 | `status=1`、`completed_at` 非空 | 不再活跃；分析报告可能仍在生成/审核/未解锁 |
| 已放弃 | `status=2`、`abandoned_at` 非空 | 不再活跃 |

> 题目收集完成并调用大模型后，`converge()` 会先把 `status` 置为 `COMPLETED`，**再**投递分析任务。因此“报告未解锁/未审核”不影响流程结束判定。

### 5.2 `enter()` 决策表（显式 cmd 入口）

`WorkflowService.enter(bizCode, userId, chatSessionId, params)`，参数可含 `action(START/CONTINUE/RESTART)`、`recordId`、`dataId`。

1. 校验家庭成员档案归属。
2. **幂等保护**：症状问诊 + `dataId` 已完成过该记录的问诊 → 返回文本提示；**`action=RESTART` 时跳过**，允许用户主动重开。
3. 查找当前成员任一进行中工作流 `active`。
4. **0 答题**（`countAnswers==0`）且 `action != CONTINUE`：
   - **尚未确认记录**（`context_json.recordUsed == null`）→ 直接 `abandon(active)`，按新流程处理；不再弹“继续上次”；
   - **已确认记录**（`recordUsed != null`，首题未答）→ 不弃用，进入 active 分支：同 biz 且候选一致/窗口内 → `resume` 重发首题（`s5`/`c1`）；否则守卫卡。
5. **分析中**（`isAnalysisPending`）且非 `CONTINUE`：
   - 若用户切换到其它 biz → `abandon`，按新流程处理；同一 biz 的分析不受时间限制。
   - 否则留在 active 分支。
6. active 分支：
   - `RESTART` → `abandon` 旧流程，落到新建；
   - `CONTINUE` → `resume(active)`；
   - 同 biz：
     - `isAnalysisPending` → `resume(active)`，返回“正在为您生成分析结果，请稍候…”提示，不弹守卫；
     - `dataId` 与 `candidateDataId` 一致且在 24h 恢复窗口内 → `resume(active)`；
     - 否则 → `guardCard(active, dataId)`（“继续上次/重新开始”，重开会带上当前 `dataId`）；
   - 不同 biz → `guardSwitch(active, targetBiz)`（“继续之前的 / 开启新的”）。
7. 无 active → 创建新工作流：
   - `buildCandidateContext` 冻结候选记录：
     - 症状问诊且传了 `dataId`：校验属于当前 `recordId` 且 `data_type=3`，冻结为 `candidateDataId/candidateRecordDate/candidateSymptom/candidateNote/candidatePinned=true`；
     - 否则回退 `getLatestDiagnosRecord(recordId)`；
     - 体质分析：冻结 `candidateRecentCount`。
   - `afterCreate`：有候选记录 → 记录确认卡；无 → 直接首题。

> 每个卡片都带「暂不需要」按钮（`key=dismiss`，`cmdCode=WORKFLOW_DISMISS`）：入口卡无 `workflowId`，跳过只影响前端；守卫卡/记录确认卡带 `workflowId`，跳过即 `abandon`。
> **记录确认卡的“直接输入”例外**：用户不点按钮而是在输入框输入时，前端按“点击 `primary` 按钮”下发 `WORKFLOW_RECORD_CHOICE`（`params.note` 携带文字），不再下发 `WORKFLOW_DISMISS`；`note` 写入 `context_json.recordNote`，作为补充说明进入分析输入，**不作为**首题答案。守卫卡/入口卡直接输入仍按原逻辑 `WORKFLOW_DISMISS` 后走普通咨询。

### 5.3 `defaultEntry()` 决策表（无参进入聊天室）

`EnterChatAction` → `WorkflowService.defaultEntry(userId, chatSessionId, recordId, silent)`：

1. **优先级 1：进行中工作流**
   - `countAnswers==0 && recordUsed == null` → `abandon`，继续优先级 2/3（尚未确认记录，无进度可续）；
   - `countAnswers==0 && recordUsed != null` → `resume(active)` 重发当前首题（`s5`/`c1`），保证历史里“记录卡 → 首题”顺序，不再新建流程；
   - `isAnalysisPending` → 返回空，不弹“继续上次”；不再按时间自动弃用；
   - 其他（已答部分题）→ `guardCard(active)`。
2. **优先级 2：体质分析**
   - 当前题库版本未完成 → `entryCard(CONSTITUTION)`；
   - 已完成 → 跳过。
3. `silent=true`（切换成员）→ 返回空。
4. **优先级 3：症状问诊**
   - 取当前成员最新 `health_data.data_type=3` 记录；
   - 无记录 → 返回空；
   - `hasSymptomConsulted(userId, recordId, latest.dataId)` → 返回空（已咨询过，不再提示）；
   - 否则 `seedEntry("SYMPTOM:REC:<dataId>")` 成功 → `entryCard(SYMPTOM)`；重复进入已提示过 → 返回空。

> `EnterChatAction` 在 `defaultEntry` 返回空时会追加 `guide_list` 引导提问列表，因此“不弹继续上次”不等于页面空白，用户仍可看到通用引导。

### 5.4 `checkNewEntry()`（重新进入页面的轻量检查）

`chat-room.vue` 的 `onShow` 会调用 `maybeCheckNewEntry()` → `CHECK_ENTRY` → `checkNewEntry`：

- 只在新题库版本、或新的未分析就诊记录时提示一次。
- **有进行中工作流 → 直接返回空，不弃用、不打扰**。
- 这里刻意不自动弃用 0 答题流程：`checkNewEntry` 由 `onShow` 触发，如果此时弃用，用户眼前刚弹出的记录确认卡会立刻失效（`chooseRecord` 会报“该会话已失效”）。
- 0 答题流程只在 `defaultEntry`（真正的无参重新进入）或显式 `enter` 时处理。

### 5.5 时间窗口与阈值

| 规则 | 值 | 位置 | 作用 |
|---|---|---|---|
| 恢复窗口 | `im.agent-task.workflow.resume-within-hours`，默认 24h | `withinResumeWindow` | `enter` 同 biz 时，近期直接续接；超时才守卫 |
| 咨询超时 | 已移除 | — | 自由咨询与工作流分析均不按时间自动终止；保留现有「暂不补充」手动结束补充流程 |
| 症状问诊最大轮次 | `symptom-max-rounds`，默认 8 | `submitAnswer` / `submitSymptomExtraction` | 达到上限强制收敛，并在分析输入中声明信息可能不充分 |

### 5.6 幂等与“重新开始”

- `health_entry_log.uk(dedup_key)` 保证入口卡不重复。
- `health_workflow_answer.uk(workflow_id, round_no)` 保证重复提交幂等。
- `RESTART` 只把旧流程置 `ABANDONED`，历史数据永不删除、不复用。
- 记录确认卡的 `restart` 按钮会携带 `dataId`，确保“重新开始”使用当前页面记录而不是最新记录。
- 症状问诊“指定记录已完成”的幂等保护在 `RESTART` 时跳过，避免用户无法主动重开。

### 5.7 典型场景（本地库实际案例）

| 时间 | 事件 | 数据 |
|---|---|---|
| 22:32:59 | 新增病诊记录 | `health_data.data_id=220`、`record_id=36`、`data_type=3` |
| 22:35:13–17 | 用户选择“不用记录、重新描述”，完成 4 题症状问诊 | `health_workflow id=60`、`status=1`、`source_data_id=NULL`、`completed_at=22:35:17` |
| 22:35:18 | 生成分析报告（未解锁） | `ai_analysis_report id=27`、`status=0` |
| 22:35:33 | 病诊记录页悬浮 AI 又新建流程 | `health_workflow id=61`、`status=0`、`current_round=0`、0 答题、`candidateDataId=220` |
| 下次进入 | 旧逻辑看到 61 的 0 答题流程 | 弹出“继续上次的症状问诊？上次进行到记录确认阶段” |

修复后的判定：

- `hasSymptomConsulted(2, 36, 220)`：60 在记录 220 创建之后完成、且 `source_data_id IS NULL` → 视为已咨询 → 悬浮 AI 无参打开；
- `defaultEntry` 看到 61：0 答题 → `abandon`，不再弹守卫；
- 60 已完成，`status=1`，不会参与活跃判定；报告是否解锁无关。

### 5.8 决策树推进（图模式）

1. **创建**：`createWorkflow` 按 `bank_code + biz_type` 读取最新 `PUBLISHED` 图，写 `plan_graph_json`、`current_node_key=startNodeKey`、`bank_version`；`plan_json` 留空。
2. **首题（RECORD 起始节点）**：`firstQuestion` 若起始节点为 `RECORD`，用 `context_json.recordUsed` 合成 `use`/`skip` 答案求下一节点（无候选记录或未选择时按 `skip`，走 `node.next`），把解析出的节点写入 `current_node_key`、`current_question_no=1` 后推送；解析失败则清空 `current_node_key/current_question_no` 并提示“题库题目加载失败”。`RECORD` 选择**不写** `health_workflow_answer`。
3. **作答**：`submitAnswer` 先校验 `round == current_round+1`、`questionNo == round`、`nodeKey == current_node_key`；复用原有必答 / 单选 / 多选 / TEXT 校验；`RECORD` 节点直接拒绝作答；`health_workflow_answer` 落 `node_key`。
4. **推进**：按 §4.4 优先级求下一节点；有 → 更新 `current_node_key`、`current_question_no=round+1` 并推送下一题；无且症状问诊开启模型追问 → 异步 `SYMPTOM_EXTRACT`，把新题追加到快照 `extras` 后推送；无 → `converge`。
5. **恢复**：`resume` 用 `current_node_key` 重发当前题；`current_question_no=NULL` 且 `round=0` 时重发记录确认卡（起始为 `RECORD` 时按题库文案渲染）。
6. **兼容**：`plan_graph_json` 为空的历史会话继续走 `plan_json` 数组的线性逻辑；起始节点不是 `RECORD` 的冻结图继续用硬编码记录确认卡。
7. **版本**：发布新版本只影响新建会话；旧会话使用冻结图，`guardCard` 提示「题库已更新」。
8. **记录确认**：选择【使用该记录】→ 原记录确认卡底部显示绿色 ✅「已使用该记录」，随后推送首题（`RECORD.use.next`，默认 `s5`「是否有更多补充说明？」）；选择【不用，我要重新描述】→ `RECORD.skip.next`（默认 `s1`）。`workflow_record_used` 只写历史、不再实时插入第二张卡。

---

## 6. 体质分析流程

1. **入口**：`CONSTITUTION_ANALYSIS`，`action=START` 或直接点快捷按钮。
2. **记录确认**：`buildCandidateContext` 统计当前成员可用健康记录数 `candidateRecentCount`；有记录 → `workflow_card`（硬编码卡：“使用记录 / 不使用记录”；输入框直接输入=点击“使用记录”+ 补充说明）；无 → 直接首题。体质题库暂不引入 `RECORD` 节点（见 §16）。
3. **逐题**：题库来源 `qb_bank.bank_code='CONSTITUTION'`（`biz_type=CONSTITUTION`）的最新 `PUBLISHED` 版本；服务端校验：
   - 顺序：`round == current_round+1`、`questionNo == round` 且 `nodeKey == current_node_key`；
   - 身份：节点必须属于创建时冻结的 `plan_graph_json`；
   - 必答：`required=true` 时不能为空；
   - 合法性：SINGLE 只能 1 项、MULTI 可多选、TEXT 长度 ≤ 500；
   - 先 `health_workflow_answer` 落库，再推进 `current_node_key`。
4. **完成**：题库问完 → `converge`：
   - `status=COMPLETED`、`completed_at`、`current_question_no=NULL`；
   - 组装 `analysisInput`（含记录摘要 + 问答）；
   - 通过 `acceptSynthetic` 以 `CONSTITUTION_RESULT` 模板投递 agent 任务；
   - 报告由 `AnalysisReportService` 生成并推送 `analysis_report` 卡片。
5. **恢复**：有答题进度且未完成 → 守卫卡；`继续作答` 重发当前题，`重新开始` 放弃旧流程并新建。0 答题且未确认记录不再守卫；已确认记录但首题未答的流程直接恢复首题。

---

## 7. 症状问诊流程

### 7.1 记录候选与确认

1. `createWorkflow` 时：
   - 传了 `dataId`（病诊记录页悬浮 AI）→ 校验 `health_data` 属于当前 `recordId`、`data_type=3`，冻结为候选；
   - 未传 → `getLatestDiagnosRecord(recordId)` 取最新一条 `data_type=3`。
2. `afterCreate`：有候选 → `workflow_card`（起始节点为 `RECORD` 时由题库驱动，否则回退硬编码）：
   - 标题“症状问诊”；
   - 副标题 = `s0.text`（题库配置，如“找到您最近的一条病症记录，是否基于它开始问诊？”）；**不再区分 `candidatePinned` 文案**；
   - 字段“记录日期 / 主要症状”（业务数据推导，不可由题库配置）；
   - 按钮由 `s0` 选项生成：`use`“使用该记录”（主按钮）/ `skip`“不用，我要重新描述”（文案可配）；
   - “使用该记录” → `chooseRecord(use=true)`：设置 `source_data_id=candidateDataId`，写 `recordSummary`，**原确认卡标记为绿色 ✅「已使用该记录」**，`firstQuestion` 按 `use.next` 进入首题（默认 `s5`「是否有更多补充说明？」）；不再插入第二张记录卡；
   - “不用” → `source_data_id=NULL`，`firstQuestion` 按 `skip.next` 进入首题（默认 `s1`“请描述一下您当前的不适症状。”）；
   - **直接输入**：用户不点按钮而在输入框输入 → 视同点击主按钮（使用该记录），文字写入 `context_json.recordNote` 作为补充说明（不占题号、不作为首题答案），随后照常进入首题。
3. 记录确认卡不会自动恢复：`defaultEntry` 遇到已答题的流程只弹守卫；0 答题且**未确认记录**的流程直接弃用；0 答题但**已确认记录**的流程会 `resume` 首题（`s5`/`c1`），保持历史里“记录卡 → 首题”的顺序。
4. 无候选记录（含已咨询）：不弹记录卡，`firstQuestion` 以合成 `skip` 走 `s0.next`（默认 `s1`）。

### 7.2 逐题与模型辅助追问

1. 题库 `qb_bank.bank_code='SYMPTOM_INTAKE'`（`biz_type=SYMPTOM`）的最新 `PUBLISHED` 版本。
2. 题库问完后，如果 `im.agent-task.workflow.symptom-model-assist=true` 且 `round < symptom-max-rounds`：
   - 先把 `current_round` 置为已答轮次、`current_question_no=NULL`，避免重复作答；
   - 异步 `agentTaskExecutor` 调用 `SYMPTOM_EXTRACT`：
     - 输出 JSON：`{"needMore":true|false,"nextQuestion":{"text":"","type":"SINGLE|MULTI|TEXT","options":[...]}}`；
     - `needMore=true` 且不重复、未超上限 → 追加 `source=MODEL` 的题目，`current_question_no` 指向新题并推送；
     - 否则 → `converge`。
   - 模型调用失败/超时 → catch 后重新查询，若仍 `IN_PROGRESS` 则 `converge` 回退到题库收敛；
   - 线程池拒绝/提交失败 → 同步 `converge` 兜底。
3. 达到 `symptom-max-rounds` → 强制收敛，输入追加“本次采集已达到最大轮次，信息可能不充分，请在分析中明确说明局限性”。

### 7.3 收敛与报告

- `converge` 使用 `SYMPTOM_RESULT` 生成症状问诊报告；
- `source_data_id` 保留在 `health_workflow` 列中（`context_json` 收敛后只保留 `analysisInput/sourceDataId/bankVersion`）；
- 图模式下 `converge` 同时清空 `current_node_key`；报告生成后推送 `analysis_report` 卡片；报告未解锁时 workflow 仍为 `COMPLETED`。

### 7.4 已咨询记录与使用记录投影

- **已咨询不再提示**：`buildCandidateContext` 复用 `hasSymptomConsulted`，候选记录（显式指定或最新一条）已咨询 → 不冻结候选，`afterCreate` 直接进入首题，不再弹【使用该记录】。`enter` 的严格幂等守卫（`hasAnalyzedSourceData`）与 `RESTART` 旁路保持不变。
- **使用记录写历史**：`chooseRecord(use=true)` 设置 `source_data_id` 的同时写一条 `chat_message`（`render_type=workflow_record_used`、`cmd_code=WORKFLOW_RECORD_USED`、`visibility=CUSTOMER_ONLY`，content 含 `workflowId/biz/dataId/recordDate/symptom/diagnosis/note`），**仅作为历史投影，不实时推送**；原记录确认卡以绿色 ✅「已使用该记录」标记选中，刷新后历史展示记录字段与补充说明。历史卡只展示标题/副标题/记录字段，**不再显示红色“已使用该记录”标签**。
- **补充说明**：`context_json.recordNote` 会追加到 `buildAnswersText` / `buildTranscript`（`用户补充说明：…`），进入模型分析输入。症状问诊“使用记录”时写入历史投影；`skip` / `CONSTITUTION` 不写 `workflow_record_used`，仅保留在分析输入中。
- **前端去重**：`workflow-record-used-card.vue` 只读渲染历史行；`fetchIncrementalMessages` / `syncNewMessages` 跳过该 `renderType`；`onWorkflowAct` / 输入等价路径在 `use` 成功后 `suppressSnapshotAfterAnswer()`，确保下次进入全量加载历史。

### 7.5 补充题库（AI 自驱调研）

- **目录**：`qb_bank.biz_type='AI'` 且启用的最新发布版本；`buildCatalogPrompt` 取 `bank_code / name / description / 前 8 道题干`。
- **发起**：模型命中 `reportNo`（即 `bank_code`）或自拟题 → `SurveyServiceImpl.launch(bankCode, ...)`：冻结题库图（无题库时为空图）→ 模型补充题放入 `extras` → 写 `survey_instance`（`bank_code / bank_version / state_json`，`qa_json='[]'`，`report_id=NULL`）→ 推送 `survey_plan{mode:'graph'}`（不再下发全部题目）。
- **逐题**：`SURVEY_START` 返回当前 `survey_question` 卡；`SURVEY_ANSWER{instanceId,nodeKey,optionNos,answerText}` 校验归属 / 状态 / 节点 → 追加 `qa_json` → 计算下一题 → 返回下一卡；完成时 `status=COMPLETED` 并投递 `SURVEY_RESULT`。
- **兼容**：旧实例继续用 `SURVEY_SUBMIT` 一次性提交；`qa_json` 仍为数组，后管答题记录页可直接解析。
- **不续答**：图模式实例状态在服务端，但前端未保存 `instanceId`；刷新后重新发起新实例（与旧行为一致）。

---

## 8. “是否已咨询”判定（病诊记录悬浮 AI）

### 8.1 判定接口

`GET /fastcms/api/client/im/workflow/symptom/analyzed?recordId=<rid>&dataId=<dataId>`

- 返回 `Boolean`。
- 鉴权：`AuthUtils.getUserId()`；校验 `recordId` 属于当前用户（`ownsRecord`）。

### 8.2 `hasSymptomConsulted(userId, recordId, dataId)` 规则

1. 校验 `health_data` 存在、`record_id == recordId`、`data_type == 3`。
2. **直接命中**：取 `status=COMPLETED`、`biz=SYMPTOM`、`source_data_id=dataId` 的最新工作流；若 `completed_at >= health_data.update_time` → `true`（问诊后记录未变更）；否则 `false`（记录已变更，允许再次问诊）。
3. **同一成员的时间命中**：
   - 查询该用户、该 `recordId`、`status=COMPLETED`、`biz=SYMPTOM` 的最新工作流；
   - 且该工作流 `source_data_id IS NULL OR source_data_id = dataId`；
   - `consultedAt = completed_at`（缺失时回退 `last_activity_at` → `created_at`）；
   - `recordChangedAt = health_data.update_time`（缺失回退 `create_time`）；
   - `consultedAt >= recordChangedAt` → `true`；否则 `false`（记录在上次问诊后发生变更）。

### 8.3 设计意图

- **不再因“上次问诊未绑定记录”而重复新建**：用户选择“不用记录、重新描述”完成问诊后，`source_data_id=NULL`；只要该问诊发生在当前病诊记录**最后一次变更**之后，就视为“这条记录已经咨询过”。
- **新增记录或记录变更仍允许新问诊**：新病诊记录的 `create_time` 晚于上次完成时间，或已有记录的 `update_time` 晚于上次完成时间 → `false` → 悬浮 AI 正常进入 `SYMPTOM_CONSULT` 并重新弹出记录确认卡。
- **避免误判**：`source_data_id IS NULL OR = dataId` 过滤掉“给记录 A 做完问诊”被误判为“记录 B 已咨询”；`recordId` + `ownsRecord` 双重校验避免越权。
- 时间取 `health_data.update_time`（最后一次修改时间，缺失回退 `create_time`），不取 `record_date`（就诊日期）；因此补录旧日期、或问诊后再次编辑记录，都会被视为“需要重新问诊”。

### 8.4 悬浮 AI 分流（`quick-record.vue`）

```
登录校验
  │
  ├─ 未选家庭成员 → toast「请先选择家庭成员」
  ├─ 当前选中日期无病诊记录 → 无参打开 chat-room（走 defaultEntry）
  └─ 有记录 rec
       ├─ WorkflowApi.isSymptomAnalyzed(rid, rec.dataId)
       │    ├─ true  → 无参打开 chat-room（看历史/报告，不新建）
       │    └─ false → /pages/discover/chat-room?cmd=SYMPTOM_CONSULT&recordId=rid&dataId=rec.dataId
       └─ 请求异常 → 按未完成走 SYMPTOM_CONSULT（后端 `hasAnalyzedSourceData` 幂等兜底）
```

- `activeTypeRecords` 取当前选中日期病诊记录；`onDateChange` / `selectMember` 已改为 `await loadTodayRecords()`，避免切日期/成员后立即点 FAB 用到上一条记录。
- 防重复点击：`aiFabNavigating`。

---

## 9. AI 分析报告

### 9.1 生成

- 工作流 `converge` → `AgentTaskIntake.acceptSynthetic(..., promptCode, renderData)`。
- `AgentTaskDispatcher` 调用大模型，`AnalysisReportService.createReportMessage`：
  - `source=WORKFLOW`、`source_id=workflowId`；
  - 标题：`体质分析报告` / `症状问诊报告`；
  - `summary` 为免费预览，`content` 为完整报告；
  - `status=GENERATED`，`review_status=NONE`；
  - 生成 `analysis_report` 卡片写入 `chat_message` 并推送。
- 自由文本智能流程生成的报告：`source=FLOW`、`source_id=flow_id`、`record_id=当前档案账户`；同一 flow 幂等，只产生一份有效报告。
- 付费客户/配置为人工审核时，报告进入 `PENDING_REVIEW`，审核通过后变为 `UNLOCKED`。

### 9.2 状态

| `status` | 含义 | 前端卡片 |
|---|---|---|
| 0 `GENERATED` | 已生成，未解锁 | “查看完整报告”，锁定提示 |
| 1 `PENDING_REVIEW` | 待人工审核 | “查看审核进度” |
| 2 `UNLOCKED` | 已解锁 | “查看完整报告”，已解锁 |
| 3 `DRAFT` | 追问草稿，不展示 | 不返回 |
| 4 `SUPERSEDED` | 被新版本替代 | “报告已更新” |

`review_status`：0 NONE / 1 PENDING / 2 APPROVED。报告详情页 `pages/health/analysis-report.vue` 处理锁定、审核中、已解锁、已更新分支。

### 9.3 解锁

- 全局配置：`config.analysis_unlock_mode`（`PAY` / `AD` / `FREE`）、`config.health_analysis_ad_unit_id`；单价 `im.agent-task.analysis.price-fen`。
- 报告首次查看时快照 `unlock_mode`（`mode_snapshot_at`），后续以快照为准。
- 接口：
  - `GET /analysis/reports/status?ids=...`
  - `GET /analysis/reports/{id}`
  - `POST /analysis/reports/{id}/unlock`（余额/充值）
  - `POST /analysis/reports/{id}/unlock-by-ad`（广告）
- **解锁只更新报告，不改变 workflow 状态**。

### 9.4 追问报告

- `FollowUpReportService` 基于根报告和 `parent_report_id` 链生成追问报告；
- 新报告 `source=FOLLOWUP`，旧报告可置 `SUPERSEDED`；
- 追问不重新走体质/症状题库，直接复用原 `analysisInput`。

---

## 10. 前端聊天室与卡片交互

### 10.1 `chat-room.vue` 入口参数（`onLoad`）

| 参数 | 作用 |
|---|---|
| `sessionId` | 指定会话 |
| `adviceId` | 健康建议入口；抑制 `ENTER_CHAT`，历史为空时预填 |
| `cmd` | 自动发送命令；参数可带 `symptoms/dataId/recordId` |
| `scene` | 随 `ENTER_CHAT` 上报；**当前后端未消费**（已知遗留） |

- 有 `pendingCmd`：加载历史后直接发送 cmd，不发送 `ENTER_CHAT`。
- 无 `pendingCmd`：发送 `ENTER_CHAT`（默认入口）。
- 首次进入优先渲染本地快照，再增量拉取；`adviceId` 入口走全量加载。

### 10.2 卡片分发

`chat-msg-bubble.vue` 按顺序匹配：

1. `text` / `voice` / `image` / `card` / `qrcode` / `record_context`
2. `cmd + renderType`：`guide_list`、`survey_question`、`workflow_card`、`workflow_question`、`workflow_qa`、`workflow_record_used`、`analysis_report`、`card`、`qrcode`
3. 其它 `cmd` → `TextCard`
4. `survey_intro` / `survey_question` → 对应卡
5. 兜底 `FallbackCard`

> `diagnosis_card` 分支已删除；旧本地快照中的 `diagnosis_card` 会在 `initChatConnection` / `initChatConnectionWithSnapshot` 加载后清理。

### 10.3 交互控制

- `workflow-card`：按钮点击 → `onWorkflowAct(msg, {cmdCode, params, key})` → `sendCmdMessage`；发送前同步锁卡，防双击。
  - 按钮统一为**紧凑胶囊、右对齐**；主操作实心品牌红，次操作浅灰底，`variant=text` 的「暂不需要」为纯文字按钮。
  - `key=dismiss` 走 `dismissPendingChoice`：入口卡无 `workflowId`，仅本地隐藏；守卫卡/记录确认卡下发 `WORKFLOW_DISMISS`，后端 `abandon` 置 `ABANDONED`。
  - 点击记录确认卡的【使用该记录】/【不用，我要重新描述】后，原卡底部显示绿色 `check-circle` + 文案（使用→已使用该记录；**体质分析使用→仅绿色 ✅，不显示文字**；症状问诊不使用→已选择重新描述；体质分析不使用→已选择不使用）；**入口卡 / 守卫卡不显示选中标记**，也不再插入第二张记录卡。
  - 历史 `workflow_record_used` 卡不再显示红色“已使用该记录”标签，只保留标题/副标题/记录字段。
  - **记录确认卡直接输入**：`isRecordChoiceCard` 按 `cmdCode = WORKFLOW_RECORD_CHOICE` 识别；`activateRecordChoiceByText` 取 `primary` 按钮（兜底 `key=use`），锁卡、写入用户气泡、下发 `WORKFLOW_RECORD_CHOICE {..., note}`；成功后清空输入并 `suppressSnapshotAfterAnswer()`，失败回滚卡片并移除气泡（保留输入）。这条路径**不**下发 `WORKFLOW_DISMISS`。
- `workflow-question`：SINGLE 点选即提交；MULTI 多选后提交；TEXT 使用底部输入；提交前校验必答。题序只在题干上方显示「第 N 题 · 共 M 题」，不再渲染右上角进度条；**决策树分支下总题数未知，只显示「第 N 题」**（`total=0`）。
- `survey_question`（补充题库图模式）：题目由服务端逐条下发；点选 / 输入后发送 `SURVEY_ANSWER`，下一题到达即成为当前可作答卡；收到收尾文本后退出问卷态。
- `workflow_qa`：历史只读投影，不显示题序，渲染为两条聊天气泡：
  - 左侧白色题干气泡（`.wqa-question`，白底 + `1rpx` 描边 + 轻阴影，`max-width:88%`，左上角小圆角）；
  - 右侧绿色回答气泡（`.wqa-answer`，`#e8f8ee`，`max-width:78%`，右上角小圆角）；
  - `answerLabel` 拼接：选项以「、」连接，自由输入以「；」追加（如「选项A、选项B；补充描述」）；
  - 已移除「您的回答」小标题、选项标签与灰色卡片外壳。
- **输入栏不因弹卡而锁定**：仅 `workflowAnalyzing`（分析中）与 `surveyPendingIntro`（调研待确认）禁用输入；有待选择的 `workflow_card` 时占位文案提示可点卡片或直接输入——记录确认卡输入 = 点击主按钮 + 补充说明；其它卡片（守卫/入口）输入仍先下发 `WORKFLOW_DISMISS` 再进入普通咨询。
- 作答成功：写入用户气泡、清空输入、暂停本地快照，下次进入全量加载 `workflow_qa` 历史。
- `workflow_analyzing`：锁定输入/快捷栏；收到下一张卡、报告或真实文本后解除。
- 成员切换：`openMemberPicker` → `ENTER_CHAT {recordId, silent:true}`，不重复欢迎语，成员维度切换工作流上下文。
- 快照：服务端确认过的消息才写入；`localOnly` 本地回答、等待类提示、临时卡不写入。
- 历史加载：首次进入分页 **20** 条；快照路径先 `fetchIncrementalMessages` 增量合并，再补合并最新一页（含 `workflow_qa` / `workflow_record_used`），确保刚答完的问答历史可见；`syncNewMessages`（同页面 onShow）默认仍跳过这两类，避免与实时气泡重复。

---

## 11. 城市定位与城市选择

### 11.1 目标与设计原则

1. 「我的」页显示当前城市；点击进入两栏式城市选择页（热门地区 + 省份）。
2. 点聊天室【中医推荐】时获取定位并随命令传 `city/province`，**不改变推荐回复内容**，仅作为上下文参数记录。
3. 设计原则：
   - **不阻塞**：定位/逆地理编码最多等待 2s，失败也必须照常发送命令。
   - **不重复骚扰**：用户拒绝授权后持久化标记，不再自动弹系统授权框。
   - **手动优先**：用户手动选择的城市不会被自动定位覆盖。
   - **可降级**：未配置腾讯位置服务 key 时，城市页仍可手动选择，不影响其它功能。

### 11.2 数据模型 `city`

城市字典表（`doc/sql/city.sql`）：

| 字段 | 说明 |
|---|---|
| `province_code` / `province_name` | 省级行政区代码 / 简称（北京、内蒙古、广西…） |
| `city_code` / `city_name` | 城市代码 / 城市名 |
| `sort_num` | 省内排序 |
| `is_hot` | `1` = 热门城市 |
| `enabled` | `1` = 启用 |
| `created` / `updated` | 时间戳 |

- 索引：`uk_city_code`、`idx_province(province_code)`、`idx_hot(is_hot, sort_num)`。
- 种子数据：34 个省级行政区、361 行、13 个热门城市（北京 / 天津 / 上海 / 南京 / 杭州 / 合肥 / 郑州 / 武汉 / 长沙 / 广州 / 重庆 / 成都 / 西安）；`INSERT IGNORE` 幂等。
- 直辖市 / 特别行政区按「省=市」各一行；省名统一用简称，与逆地理编码归一化结果对齐。
- 不做后端缓存，直接改库即时生效；城市数据在库内维护，不提供后管页面。

### 11.3 后端接口（`cms`）

| 接口 | 返回 |
|---|---|
| `GET /fastcms/api/client/city/tree` | `{hot:[{cityCode,cityName,provinceName}], provinces:[{provinceCode,provinceName,cities:[{cityCode,cityName}]}]}` |
| `GET /fastcms/api/client/geo/reverse?latitude&longitude` | `{province, city, district}`，失败时 `data=null` |

- `city/tree`：仅查 `enabled=1`，按 `province_code, sort_num` 排序；`hot` 取 `is_hot=1`。
- `geo/reverse`：`TencentMapService` 调腾讯位置服务 geocoder v1；`map.tencent.key` 为空、腾讯返回 `status != 0`、超时或异常时均返回 `success(null)`，**不抛错、不 5xx**。
- 超时：连接 2s、读取 3s。
- 名称归一化：自治区 / 特别行政区映射为简称（内蒙古 / 广西 / 西藏 / 宁夏 / 新疆 / 香港 / 澳门），其余省去掉尾部「省」或「市」，市去掉尾部「市」；直辖市 `city` 缺失时回退 `province`。
- 两个接口为公开接口，通过 `fastcms.security.ignore.urls` 放行，未使用 `@IgnoreAuth`。

### 11.4 前端（`web-uniapp`）

**`composables/useCity.js`**（模块级单例，供「我的」页 / 聊天室 / 城市页共享）

- 状态：`cityName / provinceName / isNational / source('auto'|'manual'|'national') / locating / permissionDenied / locatedCity / locatedProvince`。
- 持久化：`uni.setStorageSync('user_city_v1', { city, province, national, source, denied, locatedCity, locatedProvince })`。
- `cityDisplay`：`isNational ? '全国' : cityName || '选择城市'`。
- `loadCity()` / `setCity({city, province})` / `setNational()` 负责读写与 `source` 标记。
- `locate({ interactive })`：
  1. `uni.getSetting()` 读 `scope.userLocation`；
  2. 已拒绝 → `permissionDenied=true` 并持久化；`interactive` 时弹引导（去设置 / 手动选择），否则直接返回；
  3. 否则 `uni.getLocation({ type:'gcj02', timeout:5000 })` → `CityApi.reverseGeocode`；
  4. 仅在返回 `city` 且当前不是手动/全国选择时更新 `cityName/provinceName`，`source='auto'`；
  5. 任何失败只记录状态，不抛异常。
- `showPermissionGuide()`：`uni.showModal` → 确认后 `uni.openSetting()`，开启后重试定位。
- `ensureCityForAction()`（快捷按钮用）：
  - 全国 → `null`；已有手动城市 → 直接返回；
  - 否则 `Promise.race([locate(), 2s])`，超时/失败回退已保存城市，仍没有则 `null`；
  - **绝不 reject**。
- `getCityTree(force)`：模块内缓存 + 进行中请求去重，失败返回 `null` 允许重试。

**城市选择页 `pages/user/city.vue`**

- 自定义 navbar（`useHealthNav`，`fallbackUrl=/pages/index/user`），标题「选择地区」。
- 左栏 `scroll-view`：「热门地区」+ 省份列表，选中项品牌色高亮。
- 右栏 `scroll-view`：热门 tab 依次为定位行、全国、热门城市；省份 tab 展示该省城市。
- 定位行文案：`定位中…` / `X（自动定位）` / `定位未开启，点击开启` / `定位失败，点击重试`；点击已定位城市即选中，未授权则走授权引导。
- 选择后 `setCity()` / `setNational()` 并 `navigateBack()`。
- 首屏拉取城市树；`onShow` 时若无定位结果且未拒绝则静默定位一次。
- 空数据「暂无城市数据」，加载失败「加载失败，点击重试」。
- `pages.json` 在 `pages/user` 子包注册 `city`，`navigationStyle: custom`，不加 `auth`（游客可用）。

**「我的」胶囊 `pages/index/user.vue`**

- `.profile-card` 右上角绝对定位胶囊：`location` 图标 + `cityDisplay`（定位中显示「定位中」）。
- 样式：圆角 `999rpx`、半透明白底、22rpx 文字、`max-width:180rpx` 超长省略。
- `goCityPage()` → `sheep.$router.go('/pages/user/city')`；`onShow` 调用 `loadCity()`，无城市且未被拒绝时静默定位。

**快捷按钮 `pages/discover/chat-room.vue`**

- 快捷按钮统一走 `onQuickButtonTap(btn)`。
- 仅 `DOCTOR_SUGGEST` 调用 `ensureCityForAction()` 并追加 `params.city` / `params.province`，其余命令不带城市。
- 定位被拒 / 超时 / 接口异常：命令照常发送（不带 city）。

**`manifest.json`**

- `mp-weixin.permission.scope.userLocation` 描述文案。
- `mp-weixin.requiredPrivateInfos` 追加 `getLocation`。
- `app-plus.modules` 追加 `Geolocation`；iOS 追加 `NSLocationWhenInUseUsageDescription`。

### 11.5 配置

- `map.tencent.key`：腾讯位置服务 WebService key；为空时 `geo/reverse` 返回 `null`。
- `fastcms.security.ignore.urls`：放行 `/fastcms/api/client/city/**`、`/fastcms/api/client/geo/**`（游客可访问）。

### 11.6 约束与降级

1. **不阻塞**：`ensureCityForAction()` 以 2s 为上限包裹，所有异常吞掉，快捷命令一定发出。
2. **不重复骚扰**：拒绝后持久化 `denied`；只有用户在城市页主动点定位行才走 `openSetting`。
3. **手动优先**：手动选择的城市不会被自动定位覆盖。
4. **后端零外部依赖失败**：未配置 key、超时、非 0 状态码都只返回 `null`，城市页降级为手动选择。
5. **城市只是参数**：`city/province` 仅随 `DOCTOR_SUGGEST` 传递并记录日志，不参与回复内容与报告生成。

---

## 12. 配置项

```yaml
im:
  agent-task:
    workflow:
      enabled: true
      resume-within-hours: 24
      symptom-max-rounds: 8
      symptom-model-assist: true
      constitution-prompt-code: CONSTITUTION_RESULT
      symptom-prompt-code: SYMPTOM_RESULT
      symptom-extract-prompt-code: SYMPTOM_EXTRACT
    analysis:
      price-fen: 100
      mock-enabled: false
  smart-flow:
    enabled: true
    max-rounds: 3
    max-clarify-rounds: 2
    min-report-chars: 80

# 腾讯位置服务（逆地理编码）
map:
  tencent:
    key: ""   # 留空则逆地理编码返回 null，前端降级为手动选择
```

| 配置 | 说明 |
|---|---|
| `workflow.enabled` | 工作流总开关；关闭时 `enter/defaultEntry` 返回“暂未开放”/空 |
| `resume-within-hours` | 直接点快捷入口时，未完成工作流的自动续接窗口 |
| `symptom-max-rounds` | 症状问诊最大轮次（含模型追问） |
| `symptom-model-assist` | 是否启用 `SYMPTOM_EXTRACT` 模型追问 |
| `constitution-prompt-code` / `symptom-prompt-code` | 结果提示词模板编码（`hr_prompt_template`） |
| `symptom-extract-prompt-code` | 症状追问抽取模板编码 |
| `analysis.price-fen` | 报告默认价格（分） |
| `analysis.mock-enabled` | 模型未开通时的 Mock 结果开关 |
| `map.tencent.key` | 腾讯位置服务 key；为空则逆地理编码返回 `null` |
| `smart-flow.enabled` | 自由咨询智能流程总开关；模板缺失时自动回退旧 agent 链路 |
| `smart-flow.max-rounds` | 补充问卷最大轮次（意图复评继续收集的上限） |
| `smart-flow.max-rounds` / `max-clarify-rounds` | 补充问卷最大轮次 / 澄清最大轮次 |
| `smart-flow.min-report-chars` | 报告最小字数，低于该值视为生成失败 |

其他配置：

- `config.analysis_unlock_mode`：`PAY` / `AD` / `FREE`。
- `config.health_analysis_ad_unit_id`：广告位 ID。
- `chat_quick_button`：快捷按钮（`CONSTITUTION_ANALYSIS` / `SYMPTOM_CONSULT` / `DOCTOR_SUGGEST`）。
- `chat_guide_question`：引导提问（按 `entry_tag`）。
- 安全放行：`/fastcms/api/client/city/**`、`/fastcms/api/client/geo/**`。

---

## 13. SQL 迁移与部署

### 13.1 执行顺序（均幂等）

```bash
mysql -u root -p fastcms < doc/sql/health_workflow.sql
mysql -u root -p fastcms < doc/sql/health_workflow_seed.sql
mysql -u root -p fastcms < doc/sql/ai_analysis_report.sql
mysql -u root -p fastcms < doc/sql/ai_analysis_report_review.sql
mysql -u root -p fastcms < doc/sql/ai_analysis_report_superseded.sql
mysql -u root -p fastcms < doc/sql/ai_health_assistant_v2.sql
mysql -u root -p fastcms < doc/sql/city.sql
mysql -u root -p fastcms < doc/sql/cmd.sql
mysql -u root -p fastcms < doc/sql/qbank.sql
mysql -u root -p fastcms < doc/sql/qbank_seed.sql
mysql -u root -p fastcms < doc/sql/qbank_record_node.sql
mysql -u root -p fastcms < doc/sql/qbank_cleanup.sql
mysql -u root -p fastcms < doc/sql/ai_assistant_perf_indexes.sql
mysql -u root -p fastcms < doc/sql/ai_smart_flow.sql
mysql -u root -p fastcms < doc/sql/ai_smart_flow_seed.sql
```

- `health_workflow.sql`：建表 + `survey_question.required` + `survey_report.purpose`，**必须先于后端代码部署**。
- `health_workflow_seed.sql`：题库报告、演示题、结果/抽取提示词、快捷按钮、后管菜单。
- `ai_analysis_report*.sql`：报告表、审核列、替代列、解锁配置、`SURVEY_RESULT` 模板。
- `ai_health_assistant_v2.sql`：`bank_version` / `published_at` / `source_data_id` / 入口索引。
- `city.sql`：城市字典建表 + 种子数据，`INSERT IGNORE` 幂等，不依赖后端代码。
- `cmd.sql`：cmd 消息系统与快捷按钮/引导问题表。
- `qbank.sql`：`qb_bank/qb_version/qb_node/qb_option/qb_rule` 建表 + `health_workflow.plan_graph_json/current_node_key`、`health_workflow_answer.node_key`、`survey_instance.bank_code/bank_version/state_json/current_node_key`，幂等。
- `qbank_seed.sql`：`CONSTITUTION` / `SYMPTOM_INTAKE`（`s0` RECORD 起始节点 + `s5` 补充说明 + 发热分支演示）/ `AI_INTAKE_DEMO` 三套 v1 发布题库；新增「题库设计」菜单，隐藏旧「调研报告 / 题目 / 选项」菜单。仅在题库尚无版本时写入，不覆盖后管编辑。
- `qbank_record_node.sql`：存量库专用。对 `SYMPTOM_INTAKE` 的最新 `PUBLISHED` 版本，克隆为“`version=MAX+1` + 追加 `s0`/`s5` + 归档旧版本 + 更新 `current_version`”，只迁移症状问诊；最新版本已含 `RECORD` 或 `s0/s5` 键冲突时跳过。**必须在新后端（支持 RECORD）部署后执行**；不改进行中会话（冻结快照）。脚本假定症状题库沿用默认 `s1`（描述症状）/`s2`（持续时间）节点 key；已自定义 key 的题库请在设计器中手动配置 `use/skip` 跳转。若题库存在未发布草稿，建议在草稿中手动补充 RECORD/s5 后再发布，避免草稿覆盖迁移版本。
- `qbank_cleanup.sql`：删除旧 `survey_report/question/option` 三表、旧维护菜单与 `survey_instance.report_id` 列。**必须在新后端部署后执行**，且执行前备份三张旧表（不可回滚）。
- `ai_assistant_perf_indexes.sql`：AI 助理性能索引（只加索引，不改数据）：`health_workflow`、`health_data`、`survey_instance`、`chat_message`、`qb_version`、`qb_rule`。幂等、在线 DDL；建议先在从库/影子库验证耗时与锁。
- `ai_smart_flow.sql`：`ai_chat_flow`、`ai_model_route` 建表 + `record_id/flow_id` 存量字段与索引 + `health_record` 城市字段；幂等。
- `ai_smart_flow_seed.sql`：`FLOW_INTENT/FLOW_INTENT_REEVALUATE/FLOW_REPORT` 模板、默认阶段路由、后管「模型路由」菜单；无可用 `hr_provider` 时不插入路由。

### 13.2 构建

```bash
# 后端
mvn -pl modules/im -am -DskipTests install
mvn -pl cms -am -DskipTests install

# 后管
cd ui && npm run build

# 小程序
# HBuilderX / 微信开发者工具重新构建 web-uniapp
```

> 小程序本地调试若使用 `web-uniapp/unpackage/dist/dev/mp-weixin`，必须重新构建，确保 `sheep/api/workflow/index.js`、`quick-record.vue` 等源码改动生效。

---

## 14. 兼容性与回归约束

1. **`health.vue` 无参入口不受影响**：仍走 `ENTER_CHAT → EnterChatAction.defaultEntry`；0 答题弃用（仅尚未确认记录）、分析中不守卫主要在 `defaultEntry` 生效，`enter` 仅对显式命令做同样处理；已确认记录但首题未答的流程会恢复当前题；已答部分题的守卫行为不变。
2. **体质分析不受症状判定影响**：`hasSymptomConsulted` 仅在症状问诊入口/优先级 3 使用；体质分析仍按题库版本 `hasCompleted` 判断。
3. **报告解锁无关**：未解锁、待审核、已解锁三种状态都不参与 workflow 活跃判定。
4. **新增记录/记录变更允许新问诊**：`health_data.update_time`（回退 `create_time`）晚于上次完成时间 → 判定为未咨询。
5. **旧 `diagnos_all` / `diagnosis_card` 已下线**：前端不再渲染专用多选卡，后端不再有 `handleDiagnosAll` / `buildDiagnosRecordsCard` / `getDiagnosRecordsByDateRange`；旧本地快照由 `LEGACY_CARD_TYPES` 清理。
6. **`record_context` / `chat-record-sheet`**：存量通用记录上下文能力仍保留（后端 `RecordContextMessageHandler`、前端组件），当前聊天室没有入口；如需恢复“携带多选记录咨询”，需重新接线。
7. **`scene` 参数**：前端随 `ENTER_CHAT` 透传，后端当前未消费；不要依赖它做服务端分支。
8. **`current_question_no` 空值**：新代码会正确落 `NULL`；历史已完成行可能保留旧题号，但不影响活跃判定（已完成不参与 `findAnyActive`）。
9. **城市定位不阻塞**：`ensureCityForAction()` 最多等待 2s 且绝不 reject，拒绝授权/超时/接口异常时命令照常发送。
10. **无 key 可降级**：未配置 `map.tencent.key` 时 `geo/reverse` 返回 `data=null`，城市页可手动选择，不返回 5xx。
11. **拒绝后不重复骚扰**：`denied` 持久化在 `user_city_v1`，只有用户在城市页主动点定位行才触发 `openSetting`。
12. **手动优先**：`source=manual|national` 时自动定位不会覆盖用户选择。
13. **城市只是参数**：`city/province` 仅随 `DOCTOR_SUGGEST` 传递并记录日志，不改变回复内容与报告生成。
14. **题库双格式兼容**：`plan_graph_json` 非空走决策树逻辑；为空的历史会话继续用 `plan_json` 数组，答题 / 恢复 / 收敛行为不变。
15. **发布不影响进行中会话**：新版本只对新建会话生效；旧会话使用冻结图，`guardCard` 提示「题库已更新」。
16. **旧补充题库兼容**：旧实例仍可用 `SURVEY_SUBMIT` 一次性提交；新实例走 `SURVEY_START/SURVEY_ANSWER`。
17. **旧题库已彻底删除**：`survey_report/question/option`、旧后管页面 / 接口 / 菜单与 `survey_instance.report_id` 均由 `qbank_cleanup.sql` 删除；只保留 `survey_instance` 与「补充问答记录」。
18. **菜单缓存**：后管菜单缓存约 600s，改完需重启或等待过期。
19. **题库不可用降级**：题库被停用 / 无发布版本 / 无题 → 入口返回空或「题库未配置」，不抛异常。
20. **RECORD 旧图兼容**：起始节点不是 `RECORD` 的冻结图（含 CONSTITUTION 与升级前会话）继续使用硬编码记录确认卡；`RECORD` 选择不写 `health_workflow_answer`，不计轮次。
21. **CONSTITUTION 暂不迁移**：体质分析是否完成按 `bank_version` 判断，发布新版本会重新提示所有已完成用户；因此 `RECORD` 只用于 `SYMPTOM_INTAKE`。需求1的“输入=主按钮”在体质分析硬编码卡上同样生效。
22. **记录确认重复调用**：`context_json.recordUsed` 已存在且 `current_question_no` 非空时忽略重复 `chooseRecord`，避免重放首题。
23. **模型追问题型白名单**：`SYMPTOM_EXTRACT` 只接受 `SINGLE/MULTI/TEXT`，其它类型降级为 `SINGLE`，不会生成不可渲染的 `RECORD` 追加题。
24. **自由咨询不替代快捷流程**：`FLOW_*` 只在自由文本且无活动流程时触发；快捷按钮/工作流/问卷命令链路完全不变。
25. **账户隔离**：所有新写入携带 `record_id`；历史 `NULL` 仅主账户可见，不自动回填。
26. **终止不可复活**：`FLOW_ABORT` 与报告写入共用 flow 行锁；迟到结果按状态丢弃，不生成报告。
27. **路由可回退**：`ai_model_route` 无配置时回退既有 `hr_provider` 路由；`FLOW_INTENT` 模板缺失时回退旧 agent 链路。
28. **游标稳定性**：消息排序统一 `(created_at,id)`；`ANCHOR` 以锚点为首条，避免定位到最新消息。

---

## 15. 验证清单

### 15.1 编译 / 静态

```bash
mvn -q -pl modules/im -am -DskipTests compile && echo BUILD_OK
mvn -pl modules/im -am test -DfailIfNoTests=false    # GraphEvaluator/GraphValidator 单测（本模块已放开 surefire）
mvn -q -pl cms -am -DskipTests compile && echo BUILD_OK
cd ui && npm run build
```

- `chat-msg-bubble.vue` 无 `diagnosis_card` / `cardanalyze` 残留。
- `quick-record.vue` 调用 `WorkflowApi.isSymptomAnalyzed(rid, rec.dataId)`。
- `WorkflowApiController` 需要 `recordId` + `dataId` 两个参数。
- 全仓无 `SurveyReport/SurveyQuestion/SurveyOption` 实体与 `survey_report/question/option` 表引用；wire 类型 `survey_question` 保留。
- `GraphEvaluatorTest` 7 项 + `GraphValidatorTest` 6 项通过（含 `BankGraph` 序列化往返，**断言 `getNodes().get(0).getText()`**，防止 Jackson 同名 boolean getter 把 `text` 属性整体忽略；含 `RECORD` 选项跳转与校验约束）。

### 15.2 数据检查

```sql
-- 工作流状态
SELECT id,biz_code,status,current_round,current_question_no,
       source_data_id,completed_at,abandoned_at,last_activity_at
FROM health_workflow ORDER BY id DESC LIMIT 10;

-- 报告状态
SELECT id,title,status,review_status,unlock_channel,source,source_id
FROM ai_analysis_report ORDER BY id DESC LIMIT 10;

-- 城市字典
SELECT COUNT(DISTINCT province_code), COUNT(*) FROM city;   -- 34 / 361

-- 决策树题库
SELECT bank_code,biz_type,enabled,current_version FROM qb_bank;
SELECT bank_id,version,status,start_node_key,revision FROM qb_version ORDER BY id DESC;
SELECT COUNT(*) FROM qb_node; SELECT COUNT(*) FROM qb_option; SELECT COUNT(*) FROM qb_rule;

-- RECORD 记录确认节点（症状题库存量迁移后应存在，起始节点=s0）
SELECT n.node_key,n.node_type,n.title,n.next_node_key,o.option_key,o.label,o.next_node_key
FROM qb_node n LEFT JOIN qb_option o ON o.node_id=n.id
WHERE n.version_id=(SELECT id FROM qb_version WHERE status='PUBLISHED' ORDER BY id DESC LIMIT 1)
  AND n.node_type='RECORD' ORDER BY o.sort_no;

-- AI 助理性能索引（执行 ai_assistant_perf_indexes.sql 后）
SHOW INDEX FROM health_workflow WHERE Key_name IN ('idx_user_status_record','idx_user_biz_status_record');
SHOW INDEX FROM health_data WHERE Key_name='idx_record_type_date';
SHOW INDEX FROM survey_instance WHERE Key_name='idx_session_status_id';
SHOW INDEX FROM chat_message WHERE Key_name IN ('idx_session_created','idx_session_sender_created');
SHOW INDEX FROM qb_version WHERE Key_name='idx_bank_status_version';
SHOW INDEX FROM qb_rule WHERE Key_name='idx_version_priority';

-- 图模式会话
SELECT id,biz_code,current_round,current_question_no,current_node_key,bank_version,
       LEFT(plan_graph_json,40) FROM health_workflow ORDER BY id DESC LIMIT 5;
SELECT workflow_id,round_no,node_key,question_type,option_nos,answer_text
FROM health_workflow_answer ORDER BY id DESC LIMIT 10;
SELECT id,bank_code,bank_version,status,current_node_key,LEFT(qa_json,60)
FROM survey_instance ORDER BY id DESC LIMIT 5;

-- 图快照（修复后应非空，且节点必须含 text）
SELECT id,biz_code,LENGTH(plan_graph_json) FROM health_workflow ORDER BY id DESC LIMIT 5;
SELECT id,biz_code,plan_graph_json LIKE '%"text"%' has_text FROM health_workflow ORDER BY id DESC LIMIT 5;
SELECT id,bank_code,LENGTH(state_json) FROM survey_instance ORDER BY id DESC LIMIT 5;

-- 作答与历史投影（正常一次问诊应 > 0）
SELECT workflow_id,COUNT(*) FROM health_workflow_answer
 GROUP BY workflow_id ORDER BY workflow_id DESC LIMIT 5;
SELECT JSON_EXTRACT(content,'$.workflowId') wid,COUNT(*) FROM chat_message
 WHERE render_type='workflow_qa' GROUP BY wid ORDER BY wid DESC LIMIT 5;

-- 旧表清理结果
SHOW TABLES LIKE 'survey_%';                          -- 只剩 survey_instance
SHOW COLUMNS FROM survey_instance LIKE 'report_id';   -- 空
```

预期：

- 完成流程：`status=1`、`current_question_no IS NULL`。
- 0 答题且**尚未确认记录**的流程在无参重新进入后被置 `status=2`、`abandoned_at` 非空。
- 0 答题但**已确认记录**（`context_json.recordUsed != null`）的流程保持 `status=0`，重新进入时恢复当前首题。
- 报告未解锁不影响 `status=1`。

### 15.3 手动用例

| 场景 | 预期 |
|---|---|
| 完成症状问诊 → 报告生成但未解锁 → 再进入 | 不弹“继续上次”；不重复新建流程 |
| 完成体质分析 → 报告未解锁/待审核 → 再进入 | 不弹“继续上次”；不重复提示同一题库版本 |
| 从悬浮 AI 进入但停在记录确认（0 答题、未确认记录）→ 切后台再回 | 记录卡仍可用，不弹“继续上次” |
| 退出到无参入口重新进入（存在 0 答题且未确认记录流程） | 该流程被弃用，继续默认入口 |
| 选择“使用该记录”/“不用”后未答首题 → 重新打开页面 | 不新建流程；历史记录卡后恢复当前首题（使用记录→「是否有更多补充说明？」；不用→「请描述一下您当前的不适症状。」）；`health_workflow.id` 不变 |
| 选择“使用该记录”后答完首题 → 重新打开 | 历史顺序为记录卡 → Q(首题) → 后续题/守卫卡；不出现错位的旧题 |
| 历史消息中的 `workflow_record_used` 卡 | 只显示标题/副标题/记录日期/主要症状/病因诊断/补充说明；**没有红色“已使用该记录”标签** |
| 实时记录确认卡选中后 | 绿色 ✅ + 绿色文字（已使用该记录 / 已选择重新描述），不出现红色文字 |
| 体质分析记录卡 → 点【不使用记录】 | 绿色 ✅ + “已选择不使用”（不显示症状问诊的“已选择重新描述”） |
| 体质分析记录卡 → 点【使用记录】 | 仅显示绿色 ✅，不显示文字 |
| 选择“不用记录、重新描述”完成问诊 → 病诊记录页悬浮 AI | 无参打开助理，不新建 |
| 上次问诊完成后新增病诊记录 → 悬浮 AI | 正常进入 `SYMPTOM_CONSULT`，出现记录确认卡 |
| 完成问诊后**修改**该病诊记录（symptom/note 等，`update_time` 变新）→ 悬浮 AI / 再次进入 | 重新出现记录确认卡，可再次基于该记录问诊；`hasSymptomConsulted=false` |
| 完成问诊后**未修改**该病诊记录 → 悬浮 AI / 再次进入 | 不提示记录卡，判定为已咨询 |
| 使用记录并完成症状问诊 → 再次进入【症状问诊】 | 不弹记录卡、直接第 1 题；历史中有且仅有一条 `workflow_record_used` 卡片（记录日期/主要症状/病因诊断/补充说明） |
| 症状问诊 → 点【使用该记录】 | 原确认卡显示绿色 ✅「已使用该记录」，紧接出现第 1 题（默认「是否有更多补充说明？」）；不再出现第二张记录卡 |
| 症状问诊 → 点【不用，我要重新描述】 | 原确认卡显示「已选择重新描述」，紧接出现第 1 题（默认「请描述一下您当前的不适症状。」） |
| 记录确认卡不点按钮，直接输入“头痛三天，伴发热” | 等同点击主按钮；首题「是否有更多补充说明？」；`context_json.recordNote` 有值；刷新后历史投影/分析输入包含补充说明 |
| 记录确认卡直接输入后刷新页面 | 记录卡为已使用态；`workflow_record_used` 展示“补充说明”；不会重发记录卡 |
| 无病诊记录 / 候选记录已咨询 | 不弹记录卡，直接「请描述一下您当前的不适症状。」 |
| 记录确认卡出现后退出再进入（round=0） | 重发记录卡，副标题/按钮文案来自题库 `s0` |
| 旧冻结图会话（起始节点非 RECORD） | 记录卡与首题行为与升级前一致 |
| 后管把 RECORD 设为非起始 / 选项键改成 A,B / 清空无记录下一题 / 给 RECORD 配规则 | 校验报错，无法发布 |
| 后管设计器 RECORD 节点 | 题型选“记录确认”→ 编号只能 use/skip → 配跳转 → 模拟试答可走两条路径 → 保存 → 发布 → 新会话生效 |
| 重跑 `qbank_record_node.sql` | 最新 PUBLISHED 已含 RECORD → 跳过，无重复节点/版本 |
| 后管新建题库 / 新建草稿 / 保存 / 发布 | 无 `updated_at cannot be null`；`created_at/updated_at` 均非空 |
| 答完症状问诊 → 重新打开 AI 健康助理 | 能看到完整的 `workflow_qa` 问答历史（问题 + 回答气泡），含刚答完的题目 |
| 完成体质分析 / 症状问诊后查库 | `health_workflow_answer` 每个 round 有行；`chat_message` 有对应 `workflow_qa`；`plan_graph_json` 含 `"text"` |
| 答 1 题后退出 → 再进入 | 保留“继续上次/重新开始”守卫（有进度） |
| 构造 `status=0`、`round>0`、`current_question_no=NULL`、`last_activity_at` > 30min | 不守卫，流程被弃用 |
| 健康记录页悬浮 AI 进入 | 欢迎语、默认入口、快捷按钮、成员切换均正常 |
| 报告三种状态（未解锁/待审核/已解锁） | 均不改变 workflow 活跃判定 |
| 症状问诊 Q3 选「发热」 | 命中分支追问「体温大约多少？」；不选则直接收敛 |
| 完成分支问卷后重新进入 | 服务端按冻结图重发当前节点；答完正常生成报告 |
| 发布题库 v2 后 | 进行中的 v1 会话不受影响；新会话使用 v2；`health_workflow.bank_version` 正确 |
| 旧 `plan_json` 数组会话（升级前创建） | 答题 / 恢复 / 守卫 / 收敛行为与升级前一致 |
| 补充题库逐题作答 | 每次只出现一张题卡；重复点同一选项只落一条答案；完成后生成 `SURVEY_RESULT` |
| 后管设计器 | 新增节点 → 配选项/规则 → 校验 → 模拟试答看到分支路径 → 保存 → 发布 → 小程序生效 |
| 第 N 题显示 | 分支问卷只显示「第 N 题」，不再出现错误的「共 M 题」 |
| 自由输入信息充分 | 意图识别 → 直接生成一份带 `record_id/flow_id` 的报告 |
| 自由输入信息不足 | 出现现有补充信息卡；答完复评后生成报告；仍不足继续收集，不重复已问问题 |
| 意图不明确 / 非健康 | 先澄清 / 明确拒绝，均不生成错误报告 |
| 补充卡点「暂不补充」 | 精确文案出现一次；flow `ABORTED`、survey `status=3`；无报告；迟到结果被丢弃 |
| 切账户 | 顶部名称、消息、历史咨询、报告同步切换；旧账户在途响应不写入新账户 |
| 左侧抽屉点某日 | 定位到当日最早消息（非最新）；上下双向加载跨日期；边界提示「没有更多消息」 |
| 历史中收到新消息 | 不强制滚底，出现「返回最新消息（N）」 |
| quick-record 新增/编辑 | 顶部显示实际归属账户；全局账户变化不影响提交归属；表单期间拒绝切换 |
| 账户级城市 | A 手动杭州、B 自动北京；切换后显示正确；拒绝授权仍可手动选择 |
| 后管菜单 | 只剩「题库设计」「补充问答记录」；旧「调研报告 / 题目 / 选项」菜单与页面已删除 |

### 15.4 城市定位与选择

| 场景 | 预期 |
|---|---|
| 首次点【中医推荐】，允许定位 | 命令带 `city/province`（`cmdParams`），引导问题卡正常出现 |
| 点【中医推荐】，拒绝定位 | 不弹窗阻断；命令不带 city 照常发送 |
| 定位接口超时 / 后端未配 key | 2s 内放行，命令照常发送 |
| 进入「我的」 | 右上角胶囊显示 `location` 图标 + 城市名（或「选择城市」），不遮挡头像/昵称 |
| 点胶囊 | 进入城市选择页，默认「热门地区」，首行显示定位城市 |
| 城市页拒绝授权后点定位行 | 弹引导 →「去设置」→ `openSetting`；开启后返回自动重新定位 |
| 手动选择 安徽 → 合肥 | 返回「我的」胶囊显示「合肥」，重进城市页该城市高亮 |
| 选择「全国」 | 胶囊显示「全国」；点【中医推荐】不传 city |
| 断网 / 接口失败 | 城市页显示重试，不白屏；不影响其它页面 |

---

## 16. 已知取舍与后续

- **工作流结果走既有 agent 任务管线**：`agent_message_task` → LLM → `ai_analysis_report`；`health_workflow.model_output` 预留，逐轮模型输出在 `health_workflow_answer.model_output`。
- **症状追问是异步的**：不阻塞 WS 会话线程；失败/超时回退题库或收敛；线程池拒绝也会同步收敛。
- **0 答题流程**：尚未确认记录的 0 答题流程直接弃用；已确认记录但首题未答的流程会 `resume` 当前题（不弃用），避免历史里出现“记录卡 → 新流程旧题”的错乱。
- **咨询不自动超时**：自由咨询与工作流分析都保持 `IN_PROGRESS`，不会因时间自动弃用。
- **入口 SDK 依赖**：小程序端 `WorkflowApi` 在 `sheep/api/workflow/index.js`，由 `sheep/api/index.js` 的 glob 自动注册；直接 `import WorkflowApi from '@/sheep/api/workflow'` 也可用。
- **后管**：`ui/src/views/workflow/session/index.vue` 只读查看工作流轨迹；题库由「题库设计」维护（树形大纲 + 节点编辑 + 模拟试答），旧的「调研管理」维护页已隐藏。
- **城市数据在库维护**：`city` 表直接改库即时生效，不提供后管页面，也不做搜索框。
- **逆地理编码经后端代理**：腾讯 key 只存在服务端；未配置时前端降级为手动选择。
- **城市只是命令参数**：`city/province` 随 `DOCTOR_SUGGEST` 传递，仅记录日志，不参与回复内容与报告生成。
- **小程序拆分**：`chat-room.vue` 的滚动/语音/快照/调研/工作流/记录选择已迁至 `pages/discover/composables/`；`chat-*` 组件迁入 `pages/discover/components/`。主包减小、单文件变小；组件化本身不减少总字节，包体收益主要来自 release 压缩 + 组件下沉。
- **后端严格等价优化**：局部列更新（避免每次重写 `plan_graph_json`/`context_json`/`state_json`）、lean `health_data` 查询/COUNT、单请求解析复用、性能索引；不做缓存/异步历史投影等会改变时序或可见性的优化。
- **可扩展点**：
  - 新增工作流业务（如“用药随访”）→ 新 `bizCode` + 新 `CmdAction` + 新 `qb_bank`（`biz_type` 扩展）；
  - 新增卡片类型 → 新 `renderType` + `chat-msg-bubble.vue` 分支 + 组件；
  - 报告解锁策略 → `analysis_unlock_mode` / 报告级 `unlock_mode` 快照；
  - 记录上下文多选 → 重新接通现有 `chat-record-sheet` / `record_context`。
- **题库取舍**：
  - 设计器「整图保存」：草稿用 `revision` 乐观锁，两人同时编辑时后保存者需刷新（409）；
  - 条件构建器只暴露一层 `all/any` 的 `节点 + 操作符 + 值`，DSL 已支持 `not` 与嵌套，UI 未开放；
  - 未做版本 diff / 一键回滚；需要回滚时以上一发布版本为源新建草稿；
  - 旧 `survey_*` 题库不自动迁移，除种子三套外需在后管重建；
  - 旧题库数据不可恢复：`qbank_cleanup.sql` 执行前需备份三张旧表；旧实例的 `report_id` 不再保留，旧 `SURVEY_SUBMIT` 提交的标题退化为「信息补充」。
  - 记录使用卡 `workflow_record_used` 仍保留为刷新后的历史投影，只是不再实时推送；历史卡不再显示红色“已使用该记录”标签，只保留标题/副标题/记录字段。若后续要求历史也不显示，再移除投影并清理旧 `chat_message`。
  - qbank 写入时间：`insertFill` 只填 `createdAt`，qbank 插入需显式 `setUpdatedAt`；不改全局 `FastcmsMetaObjectHandler`。
  - **记录确认卡字段仍由业务推导**：`RECORD` 节点只配置题干与 `use`/`skip` 按钮文案；记录日期/主要症状/可用记录数仍由 `health_data` 推导，避免把动态数据写进题库。
  - **pinned 副标题不再区分**：`RECORD` 节点只有一个题干；旧硬编码卡的“您选择的记录/最近一条记录”两种副标题在迁移后统一为节点文案。
  - **补充说明与 `s5` 可能重复提问**：输入框文字作为 `recordNote` 进入分析输入，首题 `s5` 仍会询问“是否有更多补充说明”；按需求约定不把输入文字直接当作 `s5` 的答案。
  - **CONSTITUTION 未迁移**：避免按 `bank_version` 重新提示已完成用户；后续如需统一，可单独发布体质题库新版本并接受一次重提示。

---

## 17. 自由咨询智能流程与账户隔离（2026-10）

> 本节覆盖本轮新增能力：自由文本「意图识别 → 澄清/补充 → 复评 → 报告」、`ai_model_route` 阶段模型路由、「暂不补充」终止规则、健康档案账户级数据隔离、AI 助理顶部账户胶囊与左侧抽屉、按自然日聚合的历史咨询、稳定游标消息分页、quick-record 所属账户展示、账户级城市。
> 相关 SQL：`doc/sql/ai_smart_flow.sql`、`doc/sql/ai_smart_flow_seed.sql`。

### 17.1 触发条件与账户上下文

自由文本进入 `TextMessageHandler` 后：

1. 先按 `InboundEnvelope.recordId` 解析 `AccountScope`（`cms/.../account/AccountScopeService`）；未传时回退主健康档案，并允许 `record_id IS NULL` 的历史行。
2. 消息落库时写入 `chat_message.record_id`。
3. 若 `im.smart-flow.enabled=true` 且 `FLOW_INTENT` 模板可用 → 交给 `FlowOrchestrator`；否则完整回退旧 `AgentTaskIntake` 链路。
4. 客服追问期间（`hasPendingAdminFollowup`）仍只落库、不触发 AI。

`FlowOrchestrator.onCustomerText` 的优先级：

```
存在活动 ai_chat_flow
  ├─ CLARIFYING → 追加回答 → 复评
  ├─ COLLECTING → 提示继续作答（不新建流程）
  └─ ANALYZING  → 提示正在生成（不新建、不重复生成）
无活动 flow
  ├─ 存在进行中的 survey_instance → 文本交给当前题（TEXT 直接作答；选择题返回校验提示）
  ├─ 存在进行中的 health_workflow → 提示先完成/取消（不新建）
  └─ 无任何活动流程 → 新建 ai_chat_flow(CLASSIFYING) + FLOW_INTENT 任务
```

> 已识别并启动的流程同样计入历史咨询；未识别出流程的自由咨询标题为「自由咨询」。

### 17.2 数据模型增量

#### `ai_chat_flow`

| 字段 | 说明 |
|---|---|
| `user_id` / `record_id` / `session_id` | 用户、健康档案账户、会话 |
| `origin_msg_id` | 首条消息标识（`clientMsgId`，为空时用 `msg-{chat_message.id}`），`uk(user_id,origin_msg_id)` 幂等 |
| `flow_type` | `CONSTITUTION/SYMPTOM/SLEEP/NUTRITION/MENTAL/GENERAL_HEALTH/OTHER/UNSUPPORTED` |
| `title` | 历史咨询标题（模型给出，默认「自由咨询」） |
| `status` | 0 CLASSIFYING / 1 CLARIFYING / 2 COLLECTING / 3 ANALYZING / 4 COMPLETED / 5 ABORTED / 6 UNSUPPORTED / 7 FAILED |
| `next_action` | `CLARIFY/COLLECT_INFO/GENERATE_REPORT/REPLY_ONLY` |
| `intent_json` | 最近一次通过校验的结构化意图 |
| `missing_items_json` / `collected_json` | 缺失项 / 历次澄清与补充问答 |
| `round_no` | 澄清/补充轮次；用于复评幂等键 |
| `state_version` | CAS 乐观版本 |
| `last_task_id` | 最近一次相关任务 |
| `survey_instance_id` / `report_id` | 关联补充问卷 / 最终报告 |
| `active_key` | 活动=1；终态置 `NULL`；`uk(user_id,record_id,active_key)` 保证单活动流程 |
| `abort_reason` / `abort_at` / `completed_at` | 终止原因、终止/完成时间 |

#### `ai_model_route`

| 字段 | 说明 |
|---|---|
| `stage_code` | `INTENT_CLASSIFY` / `REPORT_ANALYSIS` |
| `biz_code` | 流程细分（如 `SYMPTOM`/`SLEEP`）；空串=该阶段默认 |
| `provider_code` | 复用 `hr_provider` 与密钥管理 |
| `model_name` / `temperature` / `max_tokens` | 模型与参数 |
| `priority` / `enabled` | 优先级（越小越优先）/ 启用 |

#### 存量表增量

- `chat_message.record_id`、`agent_message_task.record_id/flow_id`、`ai_analysis_report.record_id/flow_id`、`survey_instance.record_id/flow_id`。
- `survey_instance.status` 增加 `3=ABORTED`。
- `health_record.city_name/province_name/city_source/city_updated_at`。
- 索引：`chat_message(customer_id,record_id,created_at,id)`、`ai_analysis_report(user_id,record_id,created_at)`、`survey_instance(user_id,record_id,status,id)` 等。

#### 账户作用域规则

1. 带 `recordId`：校验档案属于当前用户且未删除。
2. 不带：回退主账户，并允许 `record_id IS NULL`。
3. 主账户查询：`record_id = ? OR record_id IS NULL`；非主账户：仅 `record_id = ?`。
4. 新写入一律非空；历史 `NULL` 只对主账户可见，不做猜测性回填。
5. `SURVEY` 报告随 `survey_instance.record_id` 归属；`FOLLOWUP` 只复制 `recordId`，`flow_id` 置 `NULL`。
6. 报告详情/状态/解锁均可带 `recordId`，越权返回「报告不存在」。

### 17.3 结构化意图契约与校验

模型只输出 JSON（允许 ```json 包裹）：

```json
{
  "intent": "HEALTH_FLOW|NON_HEALTH|AMBIGUOUS",
  "flowType": "CONSTITUTION|SYMPTOM|SLEEP|NUTRITION|MENTAL|GENERAL_HEALTH|OTHER|UNSUPPORTED",
  "title": "简短标题",
  "confidence": 0.0,
  "sufficient": true,
  "missingItems": [],
  "supplementQuestions": [],
  "nextAction": "GENERATE_REPORT|COLLECT_INFO|CLARIFY|REPLY_ONLY",
  "clarifyQuestion": "",
  "replyText": ""
}
```

后端校验（`FlowIntentResult`，纯 Java）：

- `intent` 枚举非法 → `InvalidIntentException` → 流程 `FAILED`。
- `HEALTH_FLOW && sufficient=true` → `nextAction=GENERATE_REPORT`，`missingItems` 清空。
- `HEALTH_FLOW && sufficient=false` → `nextAction=COLLECT_INFO`，`missingItems` 至少 1 项。
- `AMBIGUOUS` → `nextAction=CLARIFY`，`clarifyQuestion` 为空时用默认澄清语。
- `NON_HEALTH` 且 `confidence < 0.6` → 降级为 `AMBIGUOUS`，避免误拒真实健康问题。
- `title` ≤ 64；`missingItems` 0~10 项、每项 ≤ 64；`supplementQuestions` 复用 `SurveyIntent` 题型/选项校验。

### 17.4 状态机与任务

```
CLASSIFYING → CLARIFYING | COLLECTING | ANALYZING | UNSUPPORTED | FAILED
CLARIFYING  → CLASSIFYING | ABORTED | FAILED
COLLECTING  → CLASSIFYING | ABORTED | FAILED
ANALYZING   → COMPLETED | ABORTED | FAILED
终态        → 无
```

- 任务类型：`FLOW_INTENT`、`FLOW_INTENT_REEVALUATE`、`FLOW_REPORT`；由 `AgentTaskDispatcher` 在 claim 后优先分流到 `FlowOrchestrator`，旧分支完全不动。
- `FLOW_*` 任务一律 `HANDLE_AUTO`，不进入任务级人工草稿；报告审核仍由 `ai_analysis_report` 报告级审核负责。
- 所有迁移使用 `UPDATE ... WHERE id=? AND status=?` CAS；`affected=0` 静默丢弃。
- 复评：补充问卷完成后，把问答并入 `collected_json`，`round_no+1`，投递 `FLOW_INTENT_REEVALUATE`；仍不足且未达 `max-rounds` 时发起下一轮；达到上限终止。
- 澄清轮次上限 `max-clarify-rounds`（默认 2）。

失败分级：

| 类型 | 处理 |
|---|---|
| LLM 超时/网络/供应商 5xx | 抛出 → 任务保持 PROCESSING → 既有卡死重置 + `max-retry` |
| 重试用尽 | `notifyAgentFailed` 提示；流程保持 active，不自动终止 |
| 路由缺失/供应商禁用/模型不支持 | 立即 `FAILED` + 明确文案 |
| 结构化输出非法/报告空或过短 | 立即 `FAILED` + 明确文案 |

### 17.5 「暂不补充」/ 终止

新增 CMD `FLOW_ABORT`，参数 `{flowId?, instanceId?, recordId?, reason?}`：

1. 校验账户归属与状态；同一事务内 `SELECT ... FOR UPDATE` 锁 flow 行。
2. CAS 活动状态 → `ABORTED`、`active_key=NULL`、写 `abort_reason/abort_at`；已终态返回失败，不重复提示。
3. 关联 `survey_instance` 置 `status=3`。
4. 持久化并推送精确文案：`当前信息不足，暂时无法生成准确的分析报告。补充相关信息后可继续咨询。`
5. 不调用报告服务、不删除历史消息、不实现恢复。
6. 报告写入与终止共用 flow 行锁，迟到的 `FLOW_REPORT` 结果被丢弃，不会复活流程或产生报告。

前端 `useChatSurvey.cancelSurvey()` 先发 `FLOW_ABORT`，再清理前端状态；`survey_intro` 与 `survey_question` 卡均提供「暂不补充」。

### 17.6 模型路由

`AiModelRouteResolver.resolve(stage, biz)`：

1. `(stage, biz)` 精确路由（`enabled=1`，`priority ASC, id ASC`）。
2. `(stage, biz='')` 默认路由。
3. 无路由配置 → 回退既有 `hr_prompt_template.provider_code/model_name` → `hr_provider` 优先级/白名单，并 WARN。
4. 仍无可用供应商 → `ModelRouteUnavailableException`（ERROR 日志带 stage/biz）。

- `INTENT_CLASSIFY` 只解析默认路由（分类前不知道流程）；`FLOW_INTENT_REEVALUATE` 可按已知 `flowType` 细分。
- `REPORT_ANALYSIS` 以 `flowType` 作为 `biz_code`。
- 既有 `CONSTITUTION_RESULT/SYMPTOM_RESULT/SURVEY_RESULT` 仅在存在匹配 `ai_model_route` 时覆盖，否则保持原逻辑。
- 重试复用 `LlmGatewayImpl`（`hr_provider.max_retries` 指数退避、`timeout_ms`）；不新增重试层。
- 报告幂等：`flow_id` 普通索引 + 插入前查重 + flow 行锁 + 状态 CAS；同一 flow 只产生一份有效报告。

后管：`ui/src/views/ai-model-route/index.vue` + `GET/POST /fastcms/api/admin/ai-model-route/*`，菜单由种子 SQL 注册。

### 17.7 API

客户端：

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/chat/flow/active?recordId=` | 当前账户活动流程（无则 `data=null`） |
| CMD | `FLOW_ABORT` | 暂不补充/终止 |
| GET | `/chat/consultations/days?recordId=&pageNum=&pageSize=` | 按自然日聚合；标题=当日最早启动流程；返回 `earliestMessageId` 锚点 |
| GET | `/chat/messages/cursor?recordId=&sessionId=&direction=ANCHOR\|OLDER\|NEWER&anchorId=&beforeCreatedAt=&beforeId=&afterCreatedAt=&afterId=&limit=` | 稳定游标（`created_at,id`），`ANCHOR` 以锚点为首条 |
| GET | `/analysis/reports?recordId=&pageNum=&pageSize=` | 报告列表（排除 DRAFT，带 `viewable/statusText`） |
| GET/PUT/POST | `/health/record/{recordId}/city`、`.../city/auto` | 账户级城市读写；自动定位不覆盖手动/全国 |

已有 `/chat/messages`、`/analysis/reports/{id}`、`/status`、解锁接口增加可选 `recordId`；所有出站 WS 消息携带 `recordId`。

### 17.8 前端

- **顶部胶囊**：`chat-ai-portal` 右上角「账户名 + 切换」+ 侧边栏按钮；名称超长省略，不挤压按钮。
- **账户切换**：复用 `components/health/avatar-group.vue`，新增 `controlled/modelValue/showTrigger/closeOnSelect/closeOnAction/actionMode` 与 `open()/close()`；`health.vue` / `quick-record.vue` 不传新参数，行为不变。聊天页切换成功立即关闭；失败保持打开并提示。
- **数据同步**：`useChatAccountContext` 代际守卫；切换后清空消息、按 epoch 重载历史/活动流程/入口命令；旧账户在途响应按 `recordId` + epoch 丢弃；快照键升级 `chat_message_snapshot_v3` 并校验 `recordId`。
- **左侧抽屉**：`pages/discover/components/chat-drawer/chat-drawer.vue`，`su-popup type=left`，宽 `86vw`，遮罩关闭、安全区、内部滚动；含账户、城市、咨询记录、分析报告。
- **历史浏览**：点某日 → 关闭抽屉 → `ANCHOR` 加载（锚点首条）→ `historyMode`；上滑加载更早、下滑加载更新；前插锚定原首条；边界显示「没有更多消息」；历史模式暂停快照、隐藏快捷栏、新消息只计数不打断；「返回最新消息（N）」。
- **报告列表**：抽屉内分页；`DRAFT/不可查看` 不可点；点击复用 `pages/health/analysis-report.vue`（带 `recordId`）。
- **quick-record**：新增 `formOwnerRecordId`；新增=本次绑定账户，编辑=记录自身 `recordId`（补 `_ownerRecordId` 兜底）；顶部与表单横幅展示所属账户，加载时用骨架避免闪现其他账户；所有保存统一用表单账户；表单打开期间拒绝切换账户并收起弹窗；`onShow` 按表单账户重新解析名称。
- **城市**：`composables/useAccountCity.js` 读写账户城市；`pages/user/city.vue` 支持可选 `recordId`；「我的」胶囊优先账户城市，无则设备级；`DOCTOR_SUGGEST` 用账户城市，超时/拒绝不阻塞命令。

### 17.9 配置与部署

```yaml
im:
  smart-flow:
    enabled: true
    max-rounds: 3
    max-clarify-rounds: 2
    min-report-chars: 80
```

执行顺序（在原有 SQL 之后）：

```bash
mysql -u root -p fastcms < doc/sql/ai_smart_flow.sql
mysql -u root -p fastcms < doc/sql/ai_smart_flow_seed.sql
```

- `ai_smart_flow.sql`：建表 + 存量表字段/索引（幂等）。
- `ai_smart_flow_seed.sql`：`FLOW_INTENT/FLOW_INTENT_REEVALUATE/FLOW_REPORT` 模板、默认 `ai_model_route`（无可用 `hr_provider` 时不插入）、后管「模型路由」菜单。
- 未执行种子时：`FLOW_INTENT` 模板缺失 → 自动回退旧 agent 链路，不会让自由咨询整体不可用；后管配置路由后生效。
- 开发环境若 `im.agent-task.analysis.mock-enabled=true`，智能流程仍走真实路由模型（`FLOW_*` 不经过 mock）；无可用供应商时会明确失败。

### 17.10 回归约束

1. 快捷按钮/工作流：`CONSTITUTION_ANALYSIS`、`SYMPTOM_CONSULT`、`ENTER_CHAT`、`CHECK_ENTRY`、`WORKFLOW_DISMISS`、`SURVEY_START/ANSWER/SUBMIT` 行为不变。
2. 报告解锁不影响流程状态；`FLOW_REPORT` 报告可被审核/解锁/替代。
3. `chat_message` 旧 `record_id IS NULL` 只对主账户可见；不做自动回填。
4. 同秒消息按 `(created_at,id)` 稳定排序，不再重复/丢失。
5. 账户切换时旧流程不迁移；切回通过 `ENTER_CHAT/CHECK_ENTRY` + `/chat/flow/active` 恢复。
6. 历史模式不写快照、不强制滚底；退出后恢复最新视图与快照。
7. `survey_instance` 只有 `status=1` 可作答；`2/3` 一律拒绝。

---

## 20. 前端拆分与助手输入策略（2026-10 重构）

> 目标：在不新增接口、不增加调用次数、不改现有行为的前提下，把 `chat-room.vue` 的流程判断与传输/历史代码拆出，并由后端下发输入策略。

### 20.1 前端 composables

| 文件 | 职责 |
|---|---|
| `pages/discover/composables/useChatReports.js` | 分析报告卡片状态刷新/更新 |
| `pages/discover/composables/useChatTransport.js` | WebSocket `handleWsMessage`、agent/stream/report 事件、消息清理 |
| `pages/discover/composables/useChatSend.js` | 文本/图片/快捷回复/引导问题发送、发送前 inputPolicy 路由 |
| `pages/discover/composables/useChatHistory.js` | 首屏/快照/增量对账、历史日 ANCHOR/OLDER/NEWER 游标、返回最新 |
| `pages/discover/composables/useAssistantState.js` | 读取卡片 `inputPolicy` + `/chat/flow/active` 的 `assistant` 块；不新增请求 |

`chat-room.vue` 从约 2923 行降到约 1395 行（模板 + 生命周期 + 装配 + 少量工具函数）。

### 20.2 输入策略字段（后端下发，旧客户端忽略）

**卡片 `content` 新增**：`cardKey`、`inputPolicy`。

```json
{
  "cardKey": "wf_q:123:r2:s4",
  "inputPolicy": {
    "disabled": false,
    "expected": "OPTION_OR_TEXT",
    "placeholder": "点选上方选项，或直接输入您的回答",
    "textInputAction": "answer_workflow",
    "target": { "cardKey": "...", "workflowId": 123, "round": 2, "questionNo": 2, "nodeKey": "s4" },
    "quickSuppressed": true
  }
}
```

- `textInputAction`：`send_text` / `answer_workflow` / `answer_survey` / `activate_primary` / `dismiss_then_send` / `none`。
- `cardKey` 稳定标识：`wf_entry:*`、`wf_guard:*`、`wf_guard_switch:*`、`wf_record:*`、`wf_q:*`、`sv_plan:*`、`sv_q:*`。
- 文案集中在后端 `ChatUiCopy`，与前端原字符串逐字一致（含「箭头」差异）。
- 前端 `useAssistantState` 优先读取最新未作答卡片的 `inputPolicy`；无卡片时读取 `/chat/flow/active` 的 `assistant.inputPolicy`；旧服务端/旧卡片回退原判断。

### 20.3 现有 `/chat/flow/active` 扩展（同路由、同调用）

- 新增可选参数 `sessionId`、`lastKey`。
- 响应新增 `assistant` 块：`stage`、`activeCardKey`（回显）、`inputPolicy`、`quickSuppressed`、`quickButtons`、`reports`。
- 无 flow 时不再返回 `null`，而是返回 `{recordId, assistant}`；旧前端兼容。
- `quickButtons` 与 `/chat/cmd/buttons` 同源；`reports` 通过现有 `AnalysisReportService.getReportStatuses` 返回当前账户最近 50 份报告状态。
- 前端 `refreshAnalysisReportStatuses` 优先使用 `assistant.reports`，仅对未覆盖的卡片补一次现有批量接口；不新增调用、覆盖时可减少一次。

### 20.4 行为保持约束

1. `workflowAnalyzing` 仍由前端事件/timer 控制；后端 `assistant.inputPolicy` 不驱动分析锁。
2. flow `ANALYZING` 不锁输入（沿用现状），仅 recovery 时前端本地 timer 锁 2 分钟。
3. 旧补充计划卡（无 `inputPolicy`）仍按 `surveyPendingIntro` 锁定/提示；本地 survey 模式走原分支。
4. 卡片未 materialize（不在消息列表）时不注入卡片，只对已存在的卡片应用策略。
5. 选择态、乐观气泡、`answered/expired/dismissed`、滚动、语音、快照、历史游标、typing 均保持本地。
6. 报告卡展示态仍由前端现有逻辑推导（本轮未迁到后端）。

### 20.5 验证

- `mvn -pl modules/im -am test`、`mvn -pl web -am -DskipTests compile`。
- 小程序：`chat-room.vue` + 6 个 composable 通过 Prettier 解析；页面 script 与 composables 通过 ESLint `no-undef`（es2022 + uni 等 globals）零错误。
- 微信开发者工具重建后按 §15.3 手动矩阵回归；重点：记录确认卡直接输入、守卫卡输入=跳过、补充计划/题目占位文案、flow 澄清/分析中、账户切换、历史日定位。
- 网络面板：仍只调用现有 `/chat/flow/active`（一次），无新增路由；报告状态在 state 覆盖时跳过批量接口。

---

## 21. 咨询超时移除（2026-10）

### 21.1 变更结论

- **移除自由咨询超时**：删除 `FlowStaleScheduler`、`FlowOrchestrator.abortStale()`、`TIMEOUT_TEXT`、`AiChatFlowMapper.selectStale`，不再按 `last_activity_at` 30 分钟自动置 `ABORTED/TIMEOUT`，也不再发送「本次咨询已超时，您可以重新发起咨询～」。
- **移除工作流分析超时**：删除 `WorkflowServiceImpl.ANALYSIS_PENDING_MAX_MINUTES` / `isAnalysisStale`；分析中的 `health_workflow` 重新进入时不再自动弃用，`defaultEntry` 直接返回空，`enter(CONTINUE)` 可恢复。
- 不新增「结束当前咨询」入口；保留现有「暂不补充」用于手动结束补充问卷。
- 保留基础设施超时：LLM 单次 HTTP 调用超时、Agent 任务卡死重置/重试；保留前端 `workflowAnalyzing` 120s 本地解锁 timer（只解锁输入，不终止流程）与 24h 续接窗口。

### 21.2 行为变化

- 自由咨询/工作流分析中不再出现「本次咨询已超时…」；`ai_chat_flow` 不会被自动置 `ABORTED/TIMEOUT`。
- 分析中的 `health_workflow` 重新进入时不再被弃用；`defaultEntry` 返回空，`enter(CONTINUE)` 可恢复。
- 「暂不补充」文案与行为不变。
- 未完成的自由咨询会一直保持 active；用户可通过「暂不补充」结束补充问卷，或完成/重新发起对应流程。快捷按钮显隐仍沿用原逻辑（`surveyState.active` 或最新未答 `workflow_question`），不受后台 assistant state 影响。

### 21.3 回归要点

- 自由咨询停留 >35 分钟：不再出现超时提示，流程保持 active。
- 工作流分析中 >30 分钟重新进入：不被弃用；可 `CONTINUE` 恢复。
- 「暂不补充」文案与行为不变。
- 快捷按钮在空闲/入口卡/守卫卡/记录卡时正常显示；仅补充问卷或未答工作流题目时隐藏。
- 无新增 `abort_reason='TIMEOUT'` 数据；全仓无 `FlowStaleScheduler` / `isAnalysisStale` / `USER_ENDED` 引用。
