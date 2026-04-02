# Coze 微规则设计师 / 国策拆解助手 锁定搭建文档 V1

> 这是当前**锁定版**方案。  
> 以后如果没有明确改版要求，全部按本文档执行，不再切换架构。

---

## 1. 目标

你要在 Coze 上搭建的，不是一个普通聊天机器人，而是一个可以长期运行的：

- 微规则设计师
- 国策拆解助手
- 规则树生成与迭代系统

它必须稳定支持：

1. 新目标接入  
2. 负面稳态诊断  
3. 时间轴回溯  
4. 干预节点定位  
5. 候选规则生成  
6. 候选规则筛选与确认  
7. 规则入库  
8. 成功反馈  
9. 崩溃复盘  
10. 查询当前规则  

---

## 2. 锁定架构

本方案固定采用：

```text
独立智能体 + 主工作流状态机 + 数据库主存储 + 长期记忆辅助层
```

### 2.1 架构分层

#### 智能体外壳
负责：
- 名称
- 开场白
- 推荐问题
- 绑定工作流
- 绑定数据表
- 绑定长期记忆

#### 主工作流
负责：
- 意图识别
- 路由
- 多轮阶段推进
- 数据查询和写入
- 规则生成
- 崩溃复盘

#### 数据库
负责：
- 当前对话状态
- 规则树
- 事件日志

#### 长期记忆
负责：
- 用户偏好
- 常见崩溃原因
- 高频触发场景
- 表达风格

### 2.2 不变原则

1. **数据库才是主事实来源**
2. **长期记忆不保存规则树真相**
3. **String 不作为关键索引**
4. **所有派生值都在流程里算好再写库**
5. **所有大模型节点强制纯 JSON 输出**

---

## 3. 先讲透：数据表里的值到底怎么赋值 / 修改

这是最关键的基础认知。

### 3.1 三种东西必须分清

#### A. 工作流变量
这是流程运行时临时存在的值，比如：

- `query`
- `now_ts`
- `owner_key`
- `classificationId`
- `selected_rule`

这些值不是数据库里的数据。

#### B. 数据表字段
这是表结构里的列，比如：

- `owner_key`
- `stage_code`
- `rule_text`
- `status_code`

#### C. 数据表记录
这是数据库里真正存下来的每一行数据。

---

### 3.2 正确的数据流

```text
开始输入
→ 变量赋值 / 代码 / 大模型节点得到工作流变量
→ 新增数据 / 更新数据 节点
→ 写入数据库
```

### 3.3 结论

#### “数据表字段怎么赋值？”
答案：

> 不是在表里直接赋值，而是在工作流里先准备变量，再通过「新增数据」节点写入。

#### “数据表字段怎么改值？”
答案：

> 先通过「查询数据」找到目标记录，再用「更新数据」节点把新值写回去。

---

### 3.4 最简单例子：新增一条状态记录

表：`mr_dialog_state`

字段映射：

| 表字段 | 值来源 |
|---|---|
| owner_key | 代码节点 `gen_owner_key.owner_key` |
| stage_code | 固定值 `10` |
| state_json | 固定值 `"{}"` |
| last_question | 固定值 `""` |
| turn_no | 固定值 `0` |
| updated_ts | 变量 `now_ts` |
| is_active | 固定值 `true` |

这就是“给数据表字段赋值”。

---

### 3.5 最简单例子：更新状态

表：`mr_dialog_state`

更新条件：

| 条件字段 | 值 |
|---|---|
| owner_key | 当前用户 owner_key |
| is_active | true |

更新字段：

| 表字段 | 新值 |
|---|---|
| stage_code | `20` |
| state_json | 当前阶段最新 JSON |
| last_question | 本轮模型生成的问题 |
| turn_no | 旧值 + 1 |
| updated_ts | `now_ts` |

---

### 3.6 “加 1” 怎么做？

Coze 数据表不会自动做 `success_days + 1`。

正确做法是：

```text
查询数据
→ 代码节点算出 new_success_days
→ 更新数据把 new_success_days 写回表
```

比如代码节点里：

```javascript
async function main({ params }) {
  const current = Number(params.current_success_days || 0);
  return {
    new_success_days: current + 1
  };
}
```

---

## 4. 最终需要的资源

### 4.1 智能体变量区

固定保留这 5 个变量：

| 变量名 | 类型 | 用途 |
|---|---|---|
| query | String | 本轮用户输入 |
| now_ts | String / Time | 当前时间 |
| limit | Integer | 查询上限 |
| default_stage_code | Integer | 默认阶段码 |
| root_parent_id | Integer | 根规则父节点固定值 |

固定值建议：

- `limit = 20`
- `default_stage_code = 10`
- `root_parent_id = 0`

---

### 4.2 数据表

最终使用 4 张表：

1. `mr_dialog_state`
2. `mr_rule_nodes`
3. `mr_event_log`
4. `mr_rule_groups`（增强版）

> 如果你现在已经有 `mr_collapse_log`，可以先保留；但正式版本统一以 `mr_event_log` 为日志主表。

---

## 5. 数据表结构

### 5.1 `mr_dialog_state`

| 字段名 | 类型 | 说明 |
|---|---|---|
| owner_key | Number | 用户数值键 |
| stage_code | Integer | 当前阶段 |
| state_json | String | 当前阶段完整 JSON |
| last_question | String | 上一轮问题 |
| turn_no | Integer | 轮次 |
| latest_intent_code | Integer | 最近意图码 |
| current_root_id | Integer | 当前根规则 id |
| current_node_id | Integer | 当前节点 id |
| pending_candidate_idx | Integer | 待确认候选规则序号 |
| updated_ts | Time | 更新时间 |
| is_active | Boolean / Integer | 是否当前活跃 |

---

### 5.2 `mr_rule_nodes`

| 字段名 | 类型 | 说明 |
|---|---|---|
| owner_key | Number | 用户键 |
| rule_id | Integer | 规则 id |
| parent_rule_id | Integer | 父规则 id |
| root_id | Integer | 根规则 id |
| depth | Integer | 层级 |
| priority_order | Integer | 同层排序 |
| title | String | 标题 |
| trigger_scene | String | 触发场景 |
| rule_text | String | 规则正文 |
| action_text | String | 动作 |
| failure_signal | String | 失败信号 |
| note_text | String | 备注 |
| rule_type_code | Integer | 规则类型 |
| status_code | Integer | 状态 |
| friction_score | Integer | 阻力评分 |
| reliability_score | Integer | 可靠性评分 |
| specificity_score | Integer | 具体性评分 |
| strength_level | Integer | 强化等级 |
| tolerance_quota | Integer | 容错额度 |
| success_days | Integer | 连续成功天数 |
| fail_count | Integer | 失败次数 |
| created_ts | Time | 创建时间 |
| updated_ts | Time | 更新时间 |

---

### 5.3 `mr_event_log`

| 字段名 | 类型 | 说明 |
|---|---|---|
| owner_key | Number | 用户键 |
| event_id | Integer | 事件 id |
| event_type_code | Integer | 事件类型 |
| rule_id | Integer | 关联规则 id |
| payload_json | String | 详情 JSON |
| created_ts | Time | 创建时间 |

---

### 5.4 `mr_rule_groups`

| 字段名 | 类型 | 说明 |
|---|---|---|
| owner_key | Number | 用户键 |
| group_id | Integer | 规则组 id |
| root_id | Integer | 所属根树 |
| group_name | String | 组名 |
| group_order | Integer | 顺序 |
| tolerance_quota | Integer | 容错额度 |
| active_flag | Boolean / Integer | 是否启用 |
| created_ts | Time | 创建时间 |

---

## 6. 固定编码

### 6.1 阶段码 `stage_code`

| 编码 | 含义 |
|---|---|
| 10 | diagnose |
| 20 | timeline |
| 30 | node |
| 40 | rule_gen |
| 50 | stress_test |
| 60 | plan |
| 70 | query_tree |
| 80 | collapse_review |
| 99 | done |

### 6.2 意图码 `intent_code`

| 编码 | 名称 |
|---|---|
| 1 | start_design |
| 2 | continue_design |
| 3 | confirm_rule |
| 4 | reject_rule |
| 5 | report_success |
| 6 | report_collapse |
| 7 | query_rules |
| 8 | activate_pause |
| 9 | finish_session |

### 6.3 规则类型码 `rule_type_code`

| 编码 | 含义 |
|---|---|
| 1 | 被动型 |
| 2 | 如果则型 |
| 3 | 机制型 |

### 6.4 状态码 `status_code`

| 编码 | 含义 |
|---|---|
| 1 | 激活 |
| 2 | 暂停 |
| 3 | 失败 |
| 4 | 归档 |

### 6.5 事件码 `event_type_code`

| 编码 | 含义 |
|---|---|
| 1 | 崩溃 |
| 2 | 成功 |
| 3 | 查询 |
| 4 | 系统 |

---

## 7. 意图识别器配置

### 7.1 分类器名称

建议固定为：

```text
mr_intent_v2
```

### 7.2 意图列表

必须包含：

- start_design
- continue_design
- confirm_rule
- reject_rule
- report_success
- report_collapse
- query_rules
- activate_pause
- finish_session

### 7.3 重要说明：你当前截图里的输出是 `classificationId`

根据你给的 Coze 截图，意图识别节点当前输出的是：

```text
classificationId (Integer)
```

这意味着下游不能直接拿到意图名字，而是拿到一个整数 id。

### 7.4 解决方式

固定做法：

1. 保持意图列表顺序固定  
2. 在代码节点里把 `classificationId` 映射回 `intent_code`

例如约定：

| 意图列表顺序 | 含义 |
|---:|---|
| 1 | start_design |
| 2 | continue_design |
| 3 | confirm_rule |
| 4 | reject_rule |
| 5 | report_success |
| 6 | report_collapse |
| 7 | query_rules |
| 8 | activate_pause |
| 9 | finish_session |

---

## 8. 主工作流总图

```text
开始
↓
变量赋值：init_vars
↓
代码：gen_owner_key
↓
长期记忆检索：memory_query
↓
查询数据：state_latest
↓
选择器：has_state
  ├─ 否 → 新增数据：init_state
  └─ 是 → 继续
↓
意图识别：intent_cls
↓
代码：route_action
↓
选择器：intent_router
  ├─ 设计分支
  ├─ 成功分支
  ├─ 崩溃分支
  ├─ 查询分支
  ├─ 暂停/恢复分支
  └─ 结束分支
↓
变量赋值：normalize_output
↓
代码：build_write_payload
↓
选择器：write_router
  ├─ 新增数据
  ├─ 更新数据
  ├─ 新增日志
  └─ 跳过
↓
长期记忆写入：memory_write
↓
变量聚合
↓
输出
```

---

## 9. 第一步实际搭建顺序

这部分是你后续在 Coze 里真正照着做的顺序。

### Step 1：智能体外壳

确认以下内容已经就位：

1. 智能体名称  
2. 开场白  
3. 推荐问题  
4. 绑定工作流入口  
5. 绑定数据库  
6. 绑定长期记忆  

---

### Step 2：变量区

在左侧变量区确认：

- `query`
- `now_ts`
- `limit`
- `default_stage_code`
- `root_parent_id`

---

### Step 3：表结构

先确保：

- `mr_dialog_state`
- `mr_rule_nodes`
- `mr_event_log`

都已经按第 5 节字段建好。

---

### Step 4：建工作流骨架

先只摆这些节点，不填复杂逻辑：

1. 开始  
2. 变量赋值 `init_vars`  
3. 代码 `gen_owner_key`  
4. 长期记忆检索 `memory_query`  
5. 查询数据 `state_latest`  
6. 选择器 `has_state`  
7. 新增数据 `init_state`  
8. 意图识别 `intent_cls`  
9. 代码 `route_action`  
10. 选择器 `intent_router`  
11. 变量聚合  
12. 输出  

---

## 10. 节点级配置：先打通最小闭环

先不要一下子接所有 LLM 分支。  
第一阶段只打通：

```text
开始 → 初始化变量 → 生成 owner_key → 查状态 → 无状态则建状态 → 意图识别 → 路由
```

这一步通了，后面再往里加设计分支。

---

## 11. 节点详细配置

### N01【开始】

使用系统默认对话输入。  
你的界面里会把本轮用户输入传成：

```text
USER_INPUT
```

---

### N02【变量赋值：init_vars】

新增这些变量：

| 变量名 | 值 |
|---|---|
| query | 引用 `开始.USER_INPUT` |
| now_ts | 当前时间 |
| limit | `20` |
| default_stage_code | `10` |
| root_parent_id | `0` |

---

### N03【代码：gen_owner_key】

#### 作用
把用户标识转成 Number 型 `owner_key`。

#### 输入

| 输入名 | 说明 |
|---|---|
| raw_uid | 系统用户 id；如果一时拿不到，先用调试字符串 |

> 如果你当前 Coze 界面一时取不到系统用户 id，可以先传固定值 `"demo_user"` 做联调。

#### 输出

| 输出名 | 类型 |
|---|---|
| owner_key | Number |

#### 代码（JavaScript）

```javascript
async function main({ params }) {
  const s = (params.raw_uid || "demo_user").toString();
  let h = 2166136261;
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i);
    h = (h * 16777619) >>> 0;
  }
  return {
    owner_key: Number(h)
  };
}
```

---

### N04【长期记忆检索：memory_query】

#### 输入

| 输入名 | 值 |
|---|---|
| query | `init_vars.query` |
| limit | `init_vars.limit` |

#### 输出
按你当前截图，输出为：

- `outputList`
  - `output`
  - `date`

---

### N05【查询数据：state_latest】

#### 数据表
`mr_dialog_state`

#### 查询条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| is_active | = | `true` |

#### 排序

- `updated_ts` 倒序

#### 查询上限

- `1`

#### 输出

- `outputList`
- `rowNum`

---

### N06【选择器：has_state】

条件：

- 如果 `state_latest.rowNum > 0` → 有状态  
- 否则 → 无状态

---

### N07【新增数据：init_state】

只有在“无状态”分支执行。

#### 数据表
`mr_dialog_state`

#### 字段映射

| 字段名 | 值 |
|---|---|
| owner_key | `gen_owner_key.owner_key` |
| stage_code | `10` |
| state_json | `"{}"` |
| last_question | `""` |
| turn_no | `0` |
| latest_intent_code | `0` |
| current_root_id | `0` |
| current_node_id | `0` |
| pending_candidate_idx | `0` |
| updated_ts | `init_vars.now_ts` |
| is_active | `true` |

---

### N08【意图识别：intent_cls】

#### 输入

| 输入名 | 值 |
|---|---|
| query | `init_vars.query` |

#### 输出

按你当前界面，输出：

| 输出名 | 类型 |
|---|---|
| classificationId | Integer |

---

### N09【代码：route_action】

#### 作用

1. 把 `classificationId` 映射成 `intent_code`  
2. 读取当前 `stage_code`  
3. 决定本轮走哪条分支  

#### 输入建议

| 输入名 | 来源 |
|---|---|
| classificationId | `intent_cls.classificationId` |
| rowNum | `state_latest.rowNum` |
| current_stage_code | 如果有状态，就取 `state_latest.outputList[0].stage_code`，否则传 `10` |

> 如果你的 Coze 当前不方便直接从 `outputList[0]` 取值，就先额外加一个变量赋值节点做中转。

#### 输出建议

| 输出名 | 类型 | 含义 |
|---|---|---|
| intent_code | Integer | 当前意图码 |
| stage_code | Integer | 当前阶段 |
| branch_code | Integer | 分支码 |
| need_init_state | Boolean | 是否刚初始化状态 |

#### 代码（JavaScript）

```javascript
async function main({ params }) {
  const classificationId = Number(params.classificationId || 2);
  const stage_code = Number(params.current_stage_code || 10);
  let branch_code = 1;

  // branch_code:
  // 1 = design
  // 2 = success
  // 3 = collapse
  // 4 = query
  // 5 = activate_pause
  // 6 = finish

  if (classificationId === 5) branch_code = 2;
  else if (classificationId === 6) branch_code = 3;
  else if (classificationId === 7) branch_code = 4;
  else if (classificationId === 8) branch_code = 5;
  else if (classificationId === 9) branch_code = 6;
  else branch_code = 1;

  return {
    intent_code: classificationId,
    stage_code,
    branch_code,
    need_init_state: false
  };
}
```

---

### N10【选择器：intent_router】

按 `branch_code` 分支：

| branch_code | 分支 |
|---|---|
| 1 | 设计分支 |
| 2 | 成功分支 |
| 3 | 崩溃分支 |
| 4 | 查询分支 |
| 5 | 暂停/恢复分支 |
| 6 | 结束分支 |

---

## 12. 设计分支的最终阶段机

### 阶段 10：diagnose
把模糊目标变成负面稳态。

### 阶段 20：timeline
提取时间、地点、前置动作、触发物、情绪。

### 阶段 30：node
找最早、最可控、最低阻力的切入点。

### 阶段 40：rule_gen
一次输出 6 条候选规则。

### 阶段 50：stress_test
代码评分，低分回退再生成。

### 阶段 60：plan
输出 7 天计划并入库。

### 阶段 70：query_tree
从数据库读取规则树。

### 阶段 80：collapse_review
对失败规则做非评判复盘并生成修正版。

### 阶段 99：done
本轮完成。

---

## 13. 候选规则评分节点

### 作用
从 6 条候选规则里选出最稳的一条。

### 代码

```javascript
async function main({ params }) {
  const candidates = Array.isArray(params.candidates) ? params.candidates : [];

  function score(c) {
    const r = Number(c.reliability_score || 0);
    const f = Number(c.friction_score || 10);
    const s = Number(c.specificity_score || 0);
    return 0.45 * r + 0.35 * (10 - f) + 0.20 * s;
  }

  const ranked = candidates
    .map(c => ({
      ...c,
      total_score: Number(score(c).toFixed(2))
    }))
    .sort((a, b) => b.total_score - a.total_score);

  const best = ranked[0] || null;
  const pass = !!best && best.total_score >= 6.5;

  return {
    ranked,
    best,
    pass,
    next_stage_code: pass ? 60 : 40
  };
}
```

---

## 14. 成功 / 崩溃 / 查询分支

### 14.1 成功分支

流程：

```text
查询 active 规则
→ 代码节点算 success_days + 1
→ 更新规则
→ 写成功日志
→ 输出建议
```

---

### 14.2 崩溃分支

流程：

```text
查询 active 规则
→ 大模型归因
→ 旧规则 status_code = 3
→ fail_count + 1
→ 新增修正规则
→ 写崩溃日志
→ 输出修复建议
```

---

### 14.3 查询分支

流程：

```text
查询 active / paused 规则
→ 查询最近日志
→ 输出当前规则视图
```

---

## 15. 长期记忆的写入内容

固定只写这 4 类：

1. 高频触发场景  
2. 更适合的规则类型  
3. 常见崩溃原因  
4. 偏好的说话风格  

不要把以下内容写进长期记忆：

- 当前阶段码
- 当前激活规则结构
- parent_rule_id
- root_id
- status_code

这些都应存数据库。

---

## 16. 统一输出协议

所有分支最后统一输出：

```json
{
  "speak": "给用户看的自然语言",
  "stage_code": 20,
  "intent": "continue_design",
  "data": {
    "selected_rule": {},
    "plan_7d": {},
    "next_question": "",
    "rules": [],
    "collapse_fix": {}
  }
}
```

---

## 17. 第一轮联调必须通过的检查项

### 最小闭环检查

1. 用户输入一段话，`query` 能拿到  
2. `owner_key` 能稳定生成  
3. `mr_dialog_state` 能查到 / 查不到  
4. 无状态时能自动写入初始化记录  
5. 意图识别能输出 `classificationId`  
6. `route_action` 能分支  

### 设计分支检查

7. diagnose 阶段能输出 JSON  
8. timeline 阶段能输出 JSON  
9. rule_gen 阶段能输出候选数组  
10. 评分节点能正常选出规则  
11. plan 阶段能写 `mr_rule_nodes`

### 反馈分支检查

12. report_success 能更新 success_days  
13. report_collapse 能生成修正版  
14. query_rules 能查出当前规则  

---

## 18. 后续实际教学顺序（固定）

后续继续手把手时，一律按这个顺序讲：

1. 先校准变量区  
2. 再校准数据表  
3. 再校准意图识别器  
4. 再搭工作流骨架  
5. 先跑通初始化链  
6. 再接设计分支  
7. 再接成功 / 崩溃 / 查询分支  
8. 最后统一输出与测试  

---

## 19. 当前锁定结论

当前最终锁定方案为：

> **微规则设计师 / 国策拆解助手 Coze 搭建方案 V1**
>
> 核心结构：
> **独立智能体 + 工作流状态机 + 数据库主存储 + 长期记忆辅助 + 纯 JSON 协议 + 代码评分筛选**

从这一版开始：

- 架构不再变化
- 只继续细化具体节点配置
- 所有后续步骤都以这份文档为准
