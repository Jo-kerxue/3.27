# Coze 微规则设计师 / 国策拆解助手 手把手搭建步骤（Phase 4）

> 本文档承接：
> - `docs/coze_build_steps_phase1.md`
> - `docs/coze_build_steps_phase2.md`
> - `docs/coze_build_steps_phase3.md`
>
> 这一阶段的目标是：**把剩余业务分支全部接上，并完成最终联调清单。**
>
> 也就是说，到这一阶段结束时，你的 Coze 智能体不再只是“能设计规则”，而是已经具备：
>
> - 规则设计
> - 成功反馈
> - 崩溃复盘
> - 规则查询
> - 暂停/恢复
> - 结束会话
> - 长期记忆写入
> - 最终统一输出

---

## 1. 本阶段目标

本阶段完成后，你的工作流要覆盖以下完整能力：

```text
start / continue / confirm / reject
→ 设计分支阶段机

report_success
→ 成功累计 + 日志

report_collapse
→ 崩溃复盘 + 修正规则 + 日志

query_rules
→ 查询当前规则与最近日志

activate_pause
→ 暂停 / 恢复规则

finish_session
→ 结束当前状态
```

并且要补齐：

1. `mr_event_log` 写入  
2. `success_days` 更新  
3. `fail_count` 更新  
4. `status_code` 切换  
5. 长期记忆写入  
6. `merge_output` 最终统一输出  
7. 端到端测试清单  

---

## 2. 本阶段新增节点总览

建议继续沿用 Phase 3 的编号往后加节点。

### 2.1 成功分支节点

| 顺序 | 节点类型 | 节点名称 |
|---:|---|---|
| N36 | 查询数据 | success_load_rule |
| N37 | 代码 | calc_success_stats |
| N38 | 更新数据 | update_rule_success |
| N39 | 新增数据 | log_success_event |
| N40 | 变量赋值 | normalize_success_output |

### 2.2 崩溃分支节点

| 顺序 | 节点类型 | 节点名称 |
|---:|---|---|
| N41 | 查询数据 | collapse_load_rule |
| N42 | 大模型 | LLM_collapse |
| N43 | JSON反序列化 / 代码兜底 | parse_collapse |
| N44 | 代码 | calc_collapse_stats |
| N45 | 更新数据 | mark_rule_failed |
| N46 | 新增数据 | insert_fix_rule |
| N47 | 新增数据 | log_collapse_event |
| N48 | 变量赋值 | normalize_collapse_output |

### 2.3 查询分支节点

| 顺序 | 节点类型 | 节点名称 |
|---:|---|---|
| N49 | 查询数据 | query_active_rules |
| N50 | 查询数据 | query_recent_events |
| N51 | 大模型 / 文本处理 | summarize_rules |
| N52 | 变量赋值 | normalize_query_output |

### 2.4 暂停 / 恢复分支节点

| 顺序 | 节点类型 | 节点名称 |
|---:|---|---|
| N53 | 查询数据 | load_rule_for_toggle |
| N54 | 代码 | decide_toggle_status |
| N55 | 更新数据 | update_rule_toggle |
| N56 | 新增数据 | log_toggle_event |
| N57 | 变量赋值 | normalize_toggle_output |

### 2.5 结束分支节点

| 顺序 | 节点类型 | 节点名称 |
|---:|---|---|
| N58 | 更新数据 | finish_dialog_state |
| N59 | 新增数据 | log_finish_event |
| N60 | 变量赋值 | normalize_finish_output |

### 2.6 统一写入与长期记忆节点（如果你想严格按总图补全）

| 顺序 | 节点类型 | 节点名称 |
|---:|---|---|
| N61 | 代码 | build_write_payload |
| N62 | 选择器 | write_router |
| N63 | 长期记忆写入 | memory_write |

> 说明：  
> 这三个节点不是必须现在就完美做完，因为前面很多节点已经直接写库了。  
> 但为了和锁定版总图一致，这一阶段建议把它们补上，至少留结构。

---

## 3. 成功分支（report_success）

---

### 3.1 目标

当用户说：

- 今天做到了
- 执行成功了
- 这条规则有效

系统要：

1. 找到当前激活规则  
2. `success_days + 1`  
3. 写成功事件日志  
4. 给出下一步建议  

---

### 3.2 N36【查询数据：success_load_rule】

#### 数据表
`mr_rule_nodes`

#### 查询条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| status_code | = | `1` |

#### 排序

- `updated_ts` 倒序

#### limit

- `1`

#### 输出

- `outputList`
- `rowNum`

---

### 3.3 N37【代码：calc_success_stats】

#### 作用

把当前规则的旧值取出来，算出：

- `new_success_days`
- `next_hint_level`
- 是否建议加子规则

#### 输入

| 输入名 | 来源 |
|---|---|
| rowNum | `success_load_rule.rowNum` |
| outputList | `success_load_rule.outputList` |

#### 输出

| 输出名 | 类型 |
|---|---|
| target_rule_id | Integer |
| current_success_days | Integer |
| new_success_days | Integer |
| suggest_child_rule | Boolean |
| suggest_upgrade | Boolean |
| speak | String |

#### 代码

```javascript
async function main({ params }) {
  const rowNum = Number(params.rowNum || 0);
  const list = Array.isArray(params.outputList) ? params.outputList : [];

  if (rowNum <= 0 || list.length === 0) {
    return {
      target_rule_id: 0,
      current_success_days: 0,
      new_success_days: 0,
      suggest_child_rule: false,
      suggest_upgrade: false,
      speak: "我还没找到当前激活规则，先去完成一次规则设计吧。"
    };
  }

  const row = list[0] || {};
  const current_success_days = Number(row.success_days || 0);
  const new_success_days = current_success_days + 1;
  const target_rule_id = Number(row.rule_id || 0);

  const suggest_child_rule = new_success_days >= 3 && new_success_days < 7;
  const suggest_upgrade = new_success_days >= 7;

  let speak = `已记录成功，当前连续完成 ${new_success_days} 天。`;
  if (suggest_child_rule) {
    speak += " 这条规则已经开始稳定了，可以考虑加一条更轻的子规则。";
  } else if (suggest_upgrade) {
    speak += " 这条规则已经比较稳了，可以考虑升级难度或扩展规则树。";
  } else {
    speak += " 先继续保持，不要急着加难度。";
  }

  return {
    target_rule_id,
    current_success_days,
    new_success_days,
    suggest_child_rule,
    suggest_upgrade,
    speak
  };
}
```

---

### 3.4 N38【更新数据：update_rule_success】

#### 数据表
`mr_rule_nodes`

#### 更新条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| rule_id | = | `calc_success_stats.target_rule_id` |

#### 选择并设置字段

| 字段 | 值 |
|---|---|
| success_days | `calc_success_stats.new_success_days` |
| updated_ts | `init_vars.now_ts` |

---

### 3.5 N39【新增数据：log_success_event】

#### 数据表
`mr_event_log`

#### 字段映射

| 字段 | 值 |
|---|---|
| owner_key | `gen_owner_key.owner_key` |
| event_id | 用时间戳整数 / 或代码节点生成 |
| event_type_code | `2` |
| rule_id | `calc_success_stats.target_rule_id` |
| payload_json | 手动拼一个 JSON 字符串 |
| created_ts | `init_vars.now_ts` |

#### `payload_json` 建议格式

```json
{"type":"success","new_success_days":3}
```

> 如果 Coze 里不好手写 JSON 拼接，可以先简单写成：
>
> ```text
> success_days=3
> ```
>
> 后面再规范成 JSON。

---

### 3.6 N40【变量赋值：normalize_success_output】

统一输出变量：

| 变量名 | 值 |
|---|---|
| final_speak | `calc_success_stats.speak` |
| final_stage_code | `99` |
| final_intent | `"report_success"` |
| final_data | 可先写成简单文本 / 或 JSON 字符串 |

---

## 4. 崩溃分支（report_collapse）

---

### 4.1 目标

当用户说：

- 我又崩了
- 今天没做到
- 失败了

系统要：

1. 找到当前激活规则  
2. 分析崩溃原因  
3. 旧规则标记失败  
4. `fail_count + 1`  
5. 新建一条更轻的修正规则  
6. 写崩溃事件日志  

---

### 4.2 N41【查询数据：collapse_load_rule】

配置和成功分支类似。

#### 数据表
`mr_rule_nodes`

#### 查询条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| status_code | = | `1` |

#### 排序

- `updated_ts` 倒序

#### limit

- `1`

---

### 4.3 N42【大模型：LLM_collapse】

#### 作用

分析这条规则为什么崩，并输出修正版。

#### 系统提示词（直接复制）

```text
你是“微规则设计师”的崩溃复盘节点。

你的任务只有一个：
根据用户这次描述的失败情况，对当前激活规则进行非评判式复盘，并给出更轻的修正规则。

严格要求：
1. 只能输出纯 JSON
2. 不要输出 Markdown
3. 不要输出代码块
4. 不要解释
5. 字段必须完整
6. 如果信息不足，也必须返回完整 JSON，并尽量给保守修正规则

reason_code 说明：
1 = friction（阻力太高）
2 = forgot（容易忘）
3 = emotion（情绪干扰）
4 = context（场景不稳定）

输出结构必须严格为：
{
  "reason_code": 1,
  "reason_text": "",
  "fix_rule": {
    "rule_text": "",
    "rule_type_code": 2,
    "friction_score": 2,
    "reliability_score": 8,
    "specificity_score": 8,
    "title": "",
    "trigger_scene": "",
    "action_text": "",
    "failure_signal": "",
    "note_text": ""
  },
  "speak": ""
}
```

#### 输入拼接建议

把下面三段拼进用户输入：

1. 本轮用户文本 `init_vars.query`
2. 当前激活规则文本
3. 当前规则评分字段

---

### 4.4 N43【JSON反序列化 / 代码兜底：parse_collapse】

如果你有 JSON 反序列化节点，就直接用。  
如果没有，就用代码节点：

```javascript
async function main({ params }) {
  const raw = (params.raw_text || "").trim();
  try {
    const data = JSON.parse(raw);
    return {
      reason_code: Number(data.reason_code || 1),
      reason_text: data.reason_text || "",
      fix_rule: data.fix_rule || {},
      speak: data.speak || ""
    };
  } catch (e) {
    return {
      reason_code: 1,
      reason_text: "模型输出解析失败，默认按阻力过高处理",
      fix_rule: {
        rule_text: "把当前规则缩短为更轻的一步",
        rule_type_code: 2,
        friction_score: 2,
        reliability_score: 6,
        specificity_score: 6,
        title: "修正规则",
        trigger_scene: "",
        action_text: "",
        failure_signal: "",
        note_text: "JSON 解析失败后的保底修正规则"
      },
      speak: "我先把这条规则降门槛，改成更容易活下来的版本。"
    };
  }
}
```

---

### 4.5 N44【代码：calc_collapse_stats】

#### 作用

从旧规则记录里算出：

- 旧规则 `fail_count + 1`
- 新规则的 id
- 修正规则是否作为根规则平替

#### 输入

| 输入名 | 来源 |
|---|---|
| outputList | `collapse_load_rule.outputList` |
| rowNum | `collapse_load_rule.rowNum` |
| fix_rule | `parse_collapse.fix_rule` |

#### 输出

| 输出名 | 类型 |
|---|---|
| old_rule_id | Integer |
| old_fail_count | Integer |
| new_fail_count | Integer |
| new_rule_id | Integer |
| root_id | Integer |
| parent_rule_id | Integer |
| depth | Integer |

#### 代码

```javascript
async function main({ params }) {
  const rowNum = Number(params.rowNum || 0);
  const list = Array.isArray(params.outputList) ? params.outputList : [];
  const row = rowNum > 0 && list.length > 0 ? list[0] : {};

  const old_rule_id = Number(row.rule_id || 0);
  const old_fail_count = Number(row.fail_count || 0);
  const new_fail_count = old_fail_count + 1;

  const now = Date.now();
  const new_rule_id = Number(String(now).slice(-10));

  return {
    old_rule_id,
    old_fail_count,
    new_fail_count,
    new_rule_id,
    root_id: Number(row.root_id || old_rule_id || new_rule_id),
    parent_rule_id: Number(row.parent_rule_id || 0),
    depth: Number(row.depth || 1)
  };
}
```

---

### 4.6 N45【更新数据：mark_rule_failed】

#### 数据表
`mr_rule_nodes`

#### 更新条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| rule_id | = | `calc_collapse_stats.old_rule_id` |

#### 选择并设置字段

| 字段 | 值 |
|---|---|
| status_code | `3` |
| fail_count | `calc_collapse_stats.new_fail_count` |
| updated_ts | `init_vars.now_ts` |

---

### 4.7 N46【新增数据：insert_fix_rule】

#### 数据表
`mr_rule_nodes`

#### 字段映射

| 字段 | 值 |
|---|---|
| owner_key | `gen_owner_key.owner_key` |
| rule_id | `calc_collapse_stats.new_rule_id` |
| parent_rule_id | `calc_collapse_stats.parent_rule_id` |
| root_id | `calc_collapse_stats.root_id` |
| depth | `calc_collapse_stats.depth` |
| priority_order | `1` |
| title | `parse_collapse.fix_rule.title` |
| trigger_scene | `parse_collapse.fix_rule.trigger_scene` |
| rule_text | `parse_collapse.fix_rule.rule_text` |
| action_text | `parse_collapse.fix_rule.action_text` |
| failure_signal | `parse_collapse.fix_rule.failure_signal` |
| note_text | `parse_collapse.fix_rule.note_text` |
| rule_type_code | `parse_collapse.fix_rule.rule_type_code` |
| status_code | `1` |
| friction_score | `parse_collapse.fix_rule.friction_score` |
| reliability_score | `parse_collapse.fix_rule.reliability_score` |
| specificity_score | `parse_collapse.fix_rule.specificity_score` |
| strength_level | `1` |
| tolerance_quota | `0` |
| success_days | `0` |
| fail_count | `0` |
| created_ts | `init_vars.now_ts` |
| updated_ts | `init_vars.now_ts` |

---

### 4.8 N47【新增数据：log_collapse_event】

#### 数据表
`mr_event_log`

#### 字段映射

| 字段 | 值 |
|---|---|
| owner_key | `gen_owner_key.owner_key` |
| event_id | 用时间戳整数 |
| event_type_code | `1` |
| rule_id | `calc_collapse_stats.old_rule_id` |
| payload_json | 建议写 JSON 字符串 |
| created_ts | `init_vars.now_ts` |

#### `payload_json` 参考

```json
{"type":"collapse","reason_code":1,"new_rule_id":123456}
```

---

### 4.9 N48【变量赋值：normalize_collapse_output】

| 变量名 | 值 |
|---|---|
| final_speak | `parse_collapse.speak` |
| final_stage_code | `80` |
| final_intent | `"report_collapse"` |
| final_data | 可写成修正规则摘要 |

---

## 5. 查询分支（query_rules）

---

### 5.1 目标

用户说：

- 我现在有哪些规则
- 看看我的规则树
- 当前激活规则

系统要从数据库读，而不是让模型凭空编。

---

### 5.2 N49【查询数据：query_active_rules】

#### 数据表
`mr_rule_nodes`

#### 查询条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |

> 如果 Coze 查询节点支持多条件并带“或”，建议查：
>
> - status_code = 1
> - 或 status_code = 2
>
> 如果不方便，第一版先只查 `status_code = 1`

#### 排序

- `updated_ts` 倒序

#### limit

- `20`

---

### 5.3 N50【查询数据：query_recent_events】

#### 数据表
`mr_event_log`

#### 查询条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |

#### 排序

- `created_ts` 倒序

#### limit

- `10`

---

### 5.4 N51【大模型 / 文本处理：summarize_rules】

#### 目标

把数据库查出来的规则和日志整理成人能看懂的话。

#### 推荐方式

如果你只想求稳，可以不用模型，直接文本处理拼接。  
如果你希望语言自然一点，就用大模型。

#### 系统提示词（模型版）

```text
你是“微规则设计师”的规则查询展示节点。

你的任务是：
根据数据库查询结果，把当前规则情况总结成用户能看懂的简短说明。

严格要求：
1. 不要编造数据库里没有的规则
2. 只基于输入内容总结
3. 语言简短，最多 200 字
4. 输出纯 JSON

输出格式：
{
  "speak": "",
  "rules_summary": "",
  "next_suggestion": ""
}
```

#### 输入建议

拼进：

- `query_active_rules.outputList`
- `query_recent_events.outputList`

---

### 5.5 N52【变量赋值：normalize_query_output】

| 变量名 | 值 |
|---|---|
| final_speak | 优先用 `summarize_rules.speak` |
| final_stage_code | `70` |
| final_intent | `"query_rules"` |
| final_data | 规则摘要 |

---

## 6. 暂停 / 恢复分支（activate_pause）

---

### 6.1 目标

用户说：

- 暂停这条规则
- 先停一下
- 恢复这条规则
- 重新启用

系统要能够把规则状态在 `1` 和 `2` 之间切换。

---

### 6.2 N53【查询数据：load_rule_for_toggle】

#### 数据表
`mr_rule_nodes`

#### 查询条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |

#### 排序

- `updated_ts` 倒序

#### limit

- `1`

> 第一版先默认切换“最近活跃的那条规则”

---

### 6.3 N54【代码：decide_toggle_status】

#### 作用

根据用户文本决定是：

- 暂停（status_code = 2）
- 恢复（status_code = 1）

#### 输入

| 输入名 | 来源 |
|---|---|
| query | `init_vars.query` |
| outputList | `load_rule_for_toggle.outputList` |

#### 输出

| 输出名 | 类型 |
|---|---|
| target_rule_id | Integer |
| next_status_code | Integer |
| speak | String |

#### 代码

```javascript
async function main({ params }) {
  const query = (params.query || "").toString();
  const list = Array.isArray(params.outputList) ? params.outputList : [];
  const row = list[0] || {};
  const target_rule_id = Number(row.rule_id || 0);

  const isPause = query.includes("暂停") || query.includes("停用") || query.includes("先停");
  const next_status_code = isPause ? 2 : 1;
  const speak = isPause ? "已暂停这条规则。" : "已恢复这条规则。";

  return {
    target_rule_id,
    next_status_code,
    speak
  };
}
```

---

### 6.4 N55【更新数据：update_rule_toggle】

#### 数据表
`mr_rule_nodes`

#### 更新条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| rule_id | = | `decide_toggle_status.target_rule_id` |

#### 选择并设置字段

| 字段 | 值 |
|---|---|
| status_code | `decide_toggle_status.next_status_code` |
| updated_ts | `init_vars.now_ts` |

---

### 6.5 N56【新增数据：log_toggle_event】

#### 数据表
`mr_event_log`

#### 字段映射

| 字段 | 值 |
|---|---|
| owner_key | `gen_owner_key.owner_key` |
| event_id | 时间戳整数 |
| event_type_code | `4` |
| rule_id | `decide_toggle_status.target_rule_id` |
| payload_json | `"toggle_status"` 或 JSON |
| created_ts | `init_vars.now_ts` |

---

### 6.6 N57【变量赋值：normalize_toggle_output】

| 变量名 | 值 |
|---|---|
| final_speak | `decide_toggle_status.speak` |
| final_stage_code | `99` |
| final_intent | `"activate_pause"` |
| final_data | `"rule toggled"` |

---

## 7. 结束分支（finish_session）

---

### 7.1 目标

用户说：

- 先这样
- 结束
- 今天就到这里

系统要把当前会话状态关闭。

---

### 7.2 N58【更新数据：finish_dialog_state】

#### 数据表
`mr_dialog_state`

#### 更新条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| is_active | = | `true` |

#### 选择并设置字段

| 字段 | 值 |
|---|---|
| is_active | `false` 或 `0` |
| stage_code | `99` |
| updated_ts | `init_vars.now_ts` |

---

### 7.3 N59【新增数据：log_finish_event】

#### 数据表
`mr_event_log`

#### 字段映射

| 字段 | 值 |
|---|---|
| owner_key | `gen_owner_key.owner_key` |
| event_id | 时间戳整数 |
| event_type_code | `4` |
| rule_id | `0` |
| payload_json | `"finish_session"` |
| created_ts | `init_vars.now_ts` |

---

### 7.4 N60【变量赋值：normalize_finish_output】

| 变量名 | 值 |
|---|---|
| final_speak | `"本轮对话已结束。下次你继续来，我会从新一轮开始。"` |
| final_stage_code | `99` |
| final_intent | `"finish_session"` |
| final_data | `"session finished"` |

---

## 8. build_write_payload / write_router / memory_write（总图补全版）

这一部分是为了和锁定版总图完全一致。

如果你前面已经在每个分支里直接写库，这一节可以先做“简化占位版”。

---

### 8.1 N61【代码：build_write_payload】

#### 作用

把各分支已经算好的结果，标准化成统一输出对象，便于：

- 写库
- 写长期记忆
- 输出给前端

#### 输入建议

把各个 `normalize_xxx_output.final_*` 都接进来。

#### 输出建议

| 输出名 | 类型 |
|---|---|
| final_speak | String |
| final_stage_code | Integer |
| final_intent | String |
| final_data_json | String |
| should_write_memory | Boolean |
| memory_user_content | String |
| memory_assistant_content | String |

#### 代码（简化版）

```javascript
async function main({ params }) {
  return {
    final_speak: params.final_speak || "",
    final_stage_code: Number(params.final_stage_code || 99),
    final_intent: params.final_intent || "",
    final_data_json: params.final_data || "",
    should_write_memory: ["report_success", "report_collapse", "query_rules"].includes(params.final_intent || ""),
    memory_user_content: params.memory_user_content || "",
    memory_assistant_content: params.final_speak || ""
  };
}
```

---

### 8.2 N62【选择器：write_router】

第一版可以非常简单：

- 如果 `should_write_memory = true` → 去 `memory_write`
- 否则 → 直接去 `merge_output`

---

### 8.3 N63【长期记忆写入：memory_write】

#### 记忆库

选你当前绑定的长期记忆库。

#### `messageList` 建议写法

可以写成两条：

```json
[
  {
    "role": "user",
    "content": "用户刚才的行为或失败描述"
  },
  {
    "role": "assistant",
    "content": "系统总结出的触发场景 / 规则偏好 / 崩溃原因"
  }
]
```

#### 第一版建议什么时候写入

只在这三类分支写入：

- `report_success`
- `report_collapse`
- `query_rules`

---

## 9. merge_output 与 输出节点的最终统一

前面各分支已经统一产出了：

- `final_speak`
- `final_stage_code`
- `final_intent`
- `final_data`

所以现在 `merge_output` 节点只做一件事：

### 9.1 Group1

把所有分支的 `final_speak` 聚到一个输出变量上。

### 9.2 如果你的变量聚合支持多个字段

建议同时聚合：

- final_speak
- final_stage_code
- final_intent
- final_data

### 9.3 输出节点

第一版最简单做法：

输出 `final_speak` 给对话界面。

如果你的 Coze 输出节点支持结构化输出，可以进一步输出：

```json
{
  "speak": "...",
  "stage_code": 99,
  "intent": "report_success",
  "data": "..."
}
```

---

## 10. 最终联调顺序

到这一阶段，必须开始联调。

按下面顺序测，不要乱。

---

### 10.1 初始化链测试

输入：

```text
我想早点睡
```

检查：

1. `query` 是否拿到
2. `owner_key` 是否输出
3. `state_latest` 是否能查空
4. `init_state` 是否自动写入一条状态

---

### 10.2 设计分支测试

输入顺序建议：

1. 我想早点睡
2. 主要是躺床刷短视频
3. 一般23:00洗漱后开始
4. 好
5. 继续
6. 可以

检查：

1. diagnose 是否进入 10
2. timeline 是否进入 20
3. node 是否进入 30
4. rule_gen 是否产出候选数组
5. score_pick_rule 是否选出 best
6. insert_rule 是否写入 `mr_rule_nodes`
7. update_state_done 是否更新 `mr_dialog_state`

---

### 10.3 成功分支测试

输入：

```text
今天做到了
```

检查：

1. 是否查到当前 active 规则
2. `success_days` 是否 +1
3. 是否写成功日志

---

### 10.4 崩溃分支测试

输入：

```text
又失败了
```

检查：

1. 是否查到当前 active 规则
2. 旧规则是否变 `status_code = 3`
3. `fail_count` 是否 +1
4. 是否生成新规则
5. 是否写崩溃日志

---

### 10.5 查询分支测试

输入：

```text
看看我的规则
```

检查：

1. 是否能查出当前规则
2. 是否能查出最近日志
3. 输出是否不是模型乱编

---

### 10.6 暂停/恢复分支测试

输入：

```text
暂停这条规则
```

再输入：

```text
重新启用
```

检查：

1. `status_code` 是否从 1 → 2 → 1
2. 是否写系统日志

---

### 10.7 结束分支测试

输入：

```text
今天就到这里
```

检查：

1. `mr_dialog_state.is_active` 是否关闭
2. `stage_code` 是否变 99
3. 是否写结束日志

---

## 11. 必拍截图清单

你每完成一段，发我这些截图，我就能帮你验收。

---

### 11.1 成功分支

请拍：

1. `success_load_rule`
2. `calc_success_stats`
3. `update_rule_success`
4. `log_success_event`

---

### 11.2 崩溃分支

请拍：

1. `collapse_load_rule`
2. `LLM_collapse`
3. `parse_collapse`
4. `mark_rule_failed`
5. `insert_fix_rule`
6. `log_collapse_event`

---

### 11.3 查询分支

请拍：

1. `query_active_rules`
2. `query_recent_events`
3. `summarize_rules`

---

### 11.4 暂停/恢复分支

请拍：

1. `load_rule_for_toggle`
2. `decide_toggle_status`
3. `update_rule_toggle`

---

### 11.5 结束分支

请拍：

1. `finish_dialog_state`
2. `log_finish_event`

---

### 11.6 总体

还要拍：

1. 工作流全景图
2. `merge_output`
3. 输出节点

---

## 12. 当前阶段完成标准

如果以下 8 条都通过，就说明你整套 V1 已经真正成型：

1. 可以开始设计规则  
2. 可以完成 diagnose → plan 主链  
3. 可以把规则写进数据库  
4. 可以成功累计 success_days  
5. 可以崩溃复盘并生成修正规则  
6. 可以查询当前规则  
7. 可以暂停/恢复规则  
8. 可以结束当前会话  

---

## 13. 当前锁定结论

到 Phase 4 结束时，你的 Coze 智能体已经具备：

> **规则设计 + 规则入库 + 成功反馈 + 崩溃修复 + 查询展示 + 状态切换 + 会话结束 + 长期记忆辅助**

这已经是锁定版 V1 的完整闭环。

从这里开始，后续再做的事情不再是“补架构”，而是：

1. 联调排错  
2. 细化查询展示  
3. 优化长期记忆写入  
4. 做更漂亮的规则树展示  

