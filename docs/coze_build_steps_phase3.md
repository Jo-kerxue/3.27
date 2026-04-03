# Coze 微规则设计师 / 国策拆解助手 手把手搭建步骤（Phase 3）

> 本文档承接：
> - `docs/coze_build_steps_phase1.md`
> - `docs/coze_build_steps_phase2.md`
>
> 这一阶段的目标是：**把设计分支的阶段机真正接起来**，先打通：
>
> ```text
> diagnose → timeline → node → rule_gen → stress_test → plan
> ```
>
> 这是整个微规则设计师最核心的一条主链。

---

## 1. 本阶段目标

这一阶段完成后，你的 Coze 工作流必须能支持下面这条链：

```text
用户说“我想早点睡”
→ diagnose：收敛成负面稳态
→ timeline：补出触发链
→ node：找到最早可控节点
→ rule_gen：一次给 6 条候选规则
→ stress_test：代码打分筛掉太重的规则
→ plan：产出 7 天执行计划
→ 写入 mr_rule_nodes
→ 更新 mr_dialog_state
→ 输出给用户
```

### 1.1 本阶段暂时不做的内容

这一阶段先不做：

- report_success 的完整 success_days 演化
- report_collapse 的完整修正规则链
- query_rules 的完整规则树展示
- activate_pause / finish_session 的完整收口

这些会在后续阶段补上。  
当前先保证主设计链跑通。

---

## 2. 这一阶段会新增哪些节点

在 Phase 2 的骨架基础上，重点给 **设计分支** 加节点。

建议按下面命名继续往后接：

| 顺序 | 节点类型 | 节点名称 |
|---:|---|---|
| N13 | 查询数据 | reload_state |
| N14 | 代码 | route_design_stage |
| N15 | 选择器 | design_stage_router |
| N16 | 大模型 | LLM_diagnose |
| N17 | JSON反序列化 / 代码兜底 | parse_diagnose |
| N18 | 更新数据 | update_state_after_diagnose |
| N19 | 大模型 | LLM_timeline |
| N20 | JSON反序列化 / 代码兜底 | parse_timeline |
| N21 | 更新数据 | update_state_after_timeline |
| N22 | 大模型 | LLM_node |
| N23 | JSON反序列化 / 代码兜底 | parse_node |
| N24 | 更新数据 | update_state_after_node |
| N25 | 大模型 | LLM_rule_gen |
| N26 | JSON反序列化 / 代码兜底 | parse_rule_gen |
| N27 | 更新数据 | update_state_after_rule_gen |
| N28 | 代码 | score_pick_rule |
| N29 | 选择器 | pass_or_regen |
| N30 | 更新数据 | update_state_after_score |
| N31 | 大模型 | LLM_plan |
| N32 | JSON反序列化 / 代码兜底 | parse_plan |
| N33 | 新增数据 | insert_rule |
| N34 | 更新数据 | update_state_done |
| N35 | 变量赋值 | normalize_design_output |

> 说明：
> - 如果你的 Coze 版本里没有好用的 JSON 反序列化，就统一用 **代码节点做 JSON.parse 兜底**
> - 我下面会同时给你“有 JSON 反序列化节点”和“没有时用代码节点”的办法

---

## 3. 设计分支的总连线图

设计分支从 `intent_router` 的“设计分支”出去，接成下面这样：

```text
intent_router(设计分支)
  ↓
reload_state
  ↓
route_design_stage
  ↓
design_stage_router
   ├─ stage=10 → LLM_diagnose → parse_diagnose → update_state_after_diagnose ─┐
   ├─ stage=20 → LLM_timeline → parse_timeline → update_state_after_timeline ─┤
   ├─ stage=30 → LLM_node → parse_node → update_state_after_node ─────────────┤
   ├─ stage=40 → LLM_rule_gen → parse_rule_gen → update_state_after_rule_gen ─┤
   ├─ stage=50 → score_pick_rule → pass_or_regen ──────────────────────────────┤
   │                         ├─ 通过 → LLM_plan → parse_plan → insert_rule → update_state_done
   │                         └─ 不通过 → update_state_after_score(回40)
   └─ stage=60/99 → LLM_plan 或 done 输出
                                  ↓
                         normalize_design_output
                                  ↓
                              merge_output
```

---

## 4. 先统一一个原则：所有大模型都必须输出纯 JSON

这一阶段最容易失败的点就是：

- 模型多说一句解释
- 返回的不是纯 JSON
- 少字段
- 数组结构不一致

所以这里的固定原则不变：

### 4.1 模型提示词必须写死以下规则

每个模型节点系统提示词都必须包含类似要求：

```text
你只能输出纯 JSON。
不要输出 Markdown。
不要输出代码块。
不要输出解释。
字段必须完整。
如果字段不知道，也要返回空字符串 / 空数组 / false。
```

### 4.2 为什么一定这样做

因为后面要：

- JSON 反序列化
- 更新状态
- 写数据库
- 做选择器判断

如果模型输出一段自然语言，后面整条链都会断。

---

## 5. N13【查询数据：reload_state】

### 5.1 作用

设计分支正式开始前，再查一次当前用户的状态记录。

为什么不直接复用 `state_latest`？

因为：
- 经过 `init_state` 之后，状态可能已经变化
- 设计分支应该总是拿最新状态

### 5.2 配置

#### 数据表
`mr_dialog_state`

#### 查询条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| is_active | = | `true` |

#### 排序

- `updated_ts` 倒序

#### limit

- `1`

#### 输出

- `outputList`
- `rowNum`

---

## 6. N14【代码：route_design_stage】

### 6.1 作用

把数据库里当前状态整理成后续阶段机能直接使用的值。

主要输出：

- `current_stage_code`
- `state_json_text`
- `turn_no`
- `current_root_id`
- `current_node_id`
- `pending_candidate_idx`

### 6.2 输入

| 输入名 | 来源 |
|---|---|
| rowNum | `reload_state.rowNum` |
| outputList | `reload_state.outputList` |
| default_stage_code | `init_vars.default_stage_code` |

### 6.3 输出

| 输出名 | 类型 |
|---|---|
| current_stage_code | Integer |
| state_json_text | String |
| turn_no | Integer |
| current_root_id | Integer |
| current_node_id | Integer |
| pending_candidate_idx | Integer |

### 6.4 代码

```javascript
async function main({ params }) {
  const rowNum = Number(params.rowNum || 0);
  const list = Array.isArray(params.outputList) ? params.outputList : [];
  const row = rowNum > 0 && list.length > 0 ? list[0] : {};

  return {
    current_stage_code: Number(row.stage_code || params.default_stage_code || 10),
    state_json_text: typeof row.state_json === "string" ? row.state_json : "{}",
    turn_no: Number(row.turn_no || 0),
    current_root_id: Number(row.current_root_id || 0),
    current_node_id: Number(row.current_node_id || 0),
    pending_candidate_idx: Number(row.pending_candidate_idx || 0)
  };
}
```

---

## 7. N15【选择器：design_stage_router】

### 7.1 条件

按 `route_design_stage.current_stage_code` 分支：

| current_stage_code | 去向 |
|---|---|
| 10 | `LLM_diagnose` |
| 20 | `LLM_timeline` |
| 30 | `LLM_node` |
| 40 | `LLM_rule_gen` |
| 50 | `score_pick_rule` |
| 60 | `LLM_plan` |
| 99 | `normalize_design_output`（或 done 输出） |
| 其他 | `LLM_diagnose` |

### 7.2 当前阶段的实际建议

为了降低首次搭建难度，第一次联调时建议先只测：

- 10 diagnose
- 20 timeline

确认能流转后再补 30 / 40 / 50 / 60。

但节点结构可以先一次性摆好。

---

## 8. 阶段 10：N16【大模型：LLM_diagnose】

### 8.1 作用

把用户的模糊目标收敛为“负面稳态”。

### 8.2 输入建议

模型输入建议拼接 3 部分：

1. 本轮用户输入 `init_vars.query`
2. 长期记忆结果 `memory_query.outputList`
3. 当前状态 `route_design_stage.state_json_text`

### 8.3 系统提示词（直接复制）

```text
你是“微规则设计师 / 国策拆解助手”。
你现在只负责诊断阶段，不做后面的步骤。

任务目标：
把用户模糊的习惯目标，收敛为一个可观察、可复述的“负面稳态”。

规则：
1. 只做诊断，不要提前生成规则。
2. 如果信息不足，只问一个最关键的问题。
3. 你只能输出纯 JSON。
4. 不要输出 Markdown，不要输出代码块，不要输出解释。
5. 就算信息不足，字段也必须完整返回。

输出 JSON 结构必须严格等于：
{
  "stage_code": 10,
  "goal_text": "",
  "negative_state": "",
  "need_more": true,
  "next_question": "",
  "state_patch": {
    "goal_text": "",
    "negative_state": ""
  },
  "next_stage_code": 20,
  "speak": ""
}

判定规则：
- 如果用户只说“我想早睡”“我想自律”，通常 need_more = true
- 如果已经足够明确成可观察情境，比如“我晚上11点后躺在床上刷手机停不下来”，则 need_more = false，next_stage_code = 20
- next_question 必须短，只问一个问题
- speak 给用户看的话，简洁自然
```

### 8.4 用户输入拼接模板

```text
【用户本轮输入】
{{init_vars.query}}

【长期记忆摘要】
{{memory_query.outputList}}

【当前状态】
{{route_design_stage.state_json_text}}
```

---

## 9. N17【parse_diagnose】两种做法

### 9.1 如果你 Coze 里有好用的 JSON 反序列化节点

直接把 `LLM_diagnose` 输出接到 JSON 反序列化节点。

拆出字段：

- stage_code
- goal_text
- negative_state
- need_more
- next_question
- state_patch
- next_stage_code
- speak

### 9.2 如果没有稳定的 JSON 反序列化节点，统一用代码节点

节点名仍叫 `parse_diagnose`，类型改成 **代码节点**。

#### 输入

| 输入名 | 来源 |
|---|---|
| raw_text | `LLM_diagnose` 的原始输出 |

#### 输出

| 输出名 | 类型 |
|---|---|
| stage_code | Integer |
| goal_text | String |
| negative_state | String |
| need_more | Boolean |
| next_question | String |
| next_stage_code | Integer |
| speak | String |
| state_patch_json | String |

#### 代码

```javascript
async function main({ params }) {
  const raw = (params.raw_text || "").trim();
  let obj = {};

  try {
    obj = JSON.parse(raw);
  } catch (e) {
    obj = {
      stage_code: 10,
      goal_text: "",
      negative_state: "",
      need_more: true,
      next_question: "你最想先改变的具体场景是什么？",
      state_patch: {
        goal_text: "",
        negative_state: ""
      },
      next_stage_code: 10,
      speak: "我先帮你把问题描述得更具体一点。"
    };
  }

  return {
    stage_code: Number(obj.stage_code || 10),
    goal_text: obj.goal_text || "",
    negative_state: obj.negative_state || "",
    need_more: Boolean(obj.need_more),
    next_question: obj.next_question || "",
    next_stage_code: Number(obj.next_stage_code || 10),
    speak: obj.speak || "",
    state_patch_json: JSON.stringify(obj.state_patch || {})
  };
}
```

---

## 10. N18【更新数据：update_state_after_diagnose】

### 10.1 作用

把 diagnose 的结果写回 `mr_dialog_state`。

### 10.2 更新条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| is_active | = | `true` |

### 10.3 更新字段

| 字段名 | 新值 |
|---|---|
| stage_code | `parse_diagnose.need_more ? 10 : 20` |
| state_json | `parse_diagnose.state_patch_json` 或你合并后的完整 JSON |
| last_question | `parse_diagnose.next_question` |
| turn_no | `route_design_stage.turn_no + 1` |
| latest_intent_code | `route_action.intent_code` |
| updated_ts | `init_vars.now_ts` |

### 10.4 关于 `state_json` 的重要说明

#### 最简单可跑版
先直接存：

```text
parse_diagnose.state_patch_json
```

#### 更完整版
后面可以升级成：

```text
旧 state_json + 新 state_patch 合并后的完整 JSON
```

首次联调阶段先求通，不求完美合并。

---

## 11. 阶段 20：N19【大模型：LLM_timeline】

### 11.1 作用

把当前问题的触发链补完整。

### 11.2 系统提示词（直接复制）

```text
你是“微规则设计师 / 国策拆解助手”。
你现在只负责时间轴回溯阶段。

任务目标：
根据用户输入和当前状态，找出问题发生前的触发链。

规则：
1. 只做时间轴回溯，不生成规则。
2. 信息不足时，只问一个问题。
3. 只能输出纯 JSON。
4. 不要输出 Markdown，不要输出解释。
5. 字段必须完整。

输出 JSON 结构必须严格等于：
{
  "stage_code": 20,
  "timeline": {
    "trigger_time": "",
    "trigger_place": "",
    "pre_action": "",
    "trigger_object": "",
    "emotion": ""
  },
  "need_more": false,
  "next_question": "",
  "state_patch": {
    "timeline": {}
  },
  "next_stage_code": 30,
  "speak": ""
}

判断原则：
- 如果还不知道“什么时候 / 在哪里 / 之前做了什么”，need_more 可为 true
- 如果已经能说清触发链，need_more = false，next_stage_code = 30
- next_question 只问一个问题
```

### 11.3 输入拼接模板

```text
【用户本轮输入】
{{init_vars.query}}

【当前状态】
{{route_design_stage.state_json_text}}
```

---

## 12. N20【parse_timeline】

如果有 JSON 反序列化节点，直接拆字段。  
如果没有，继续使用代码节点兜底。

### 12.1 输入

| 输入名 | 来源 |
|---|---|
| raw_text | `LLM_timeline` 原始输出 |

### 12.2 输出

| 输出名 | 类型 |
|---|---|
| stage_code | Integer |
| need_more | Boolean |
| next_question | String |
| next_stage_code | Integer |
| speak | String |
| state_patch_json | String |

### 12.3 代码

```javascript
async function main({ params }) {
  const raw = (params.raw_text || "").trim();
  let obj = {};

  try {
    obj = JSON.parse(raw);
  } catch (e) {
    obj = {
      stage_code: 20,
      timeline: {
        trigger_time: "",
        trigger_place: "",
        pre_action: "",
        trigger_object: "",
        emotion: ""
      },
      need_more: true,
      next_question: "一般是在什么时候开始进入这个状态的？",
      state_patch: {
        timeline: {}
      },
      next_stage_code: 20,
      speak: "我先帮你把触发链补完整。"
    };
  }

  return {
    stage_code: Number(obj.stage_code || 20),
    need_more: Boolean(obj.need_more),
    next_question: obj.next_question || "",
    next_stage_code: Number(obj.next_stage_code || 20),
    speak: obj.speak || "",
    state_patch_json: JSON.stringify(obj.state_patch || {})
  };
}
```

---

## 13. N21【更新数据：update_state_after_timeline】

### 13.1 更新条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| is_active | = | `true` |

### 13.2 更新字段

| 字段名 | 新值 |
|---|---|
| stage_code | `parse_timeline.need_more ? 20 : 30` |
| state_json | `parse_timeline.state_patch_json` |
| last_question | `parse_timeline.next_question` |
| turn_no | `route_design_stage.turn_no + 1` |
| latest_intent_code | `route_action.intent_code` |
| updated_ts | `init_vars.now_ts` |

---

## 14. 阶段 30：N22【大模型：LLM_node】

### 14.1 作用

根据触发链，找最早、最可控、最低阻力的干预节点。

### 14.2 系统提示词（直接复制）

```text
你是“微规则设计师 / 国策拆解助手”。
你现在只负责“干预节点定位”。

任务目标：
根据已有触发链，找出“最早、最可控、最低阻力”的切入点。

规则：
1. 不生成规则，只定位节点。
2. 只能输出纯 JSON。
3. 不要输出 Markdown，不要输出解释。
4. 字段必须完整。

输出 JSON 结构必须严格等于：
{
  "stage_code": 30,
  "intervention_node": "",
  "why": "",
  "need_more": false,
  "next_question": "",
  "state_patch": {
    "intervention_node": "",
    "why": ""
  },
  "next_stage_code": 40,
  "speak": ""
}
```

### 14.3 输入拼接模板

```text
【用户本轮输入】
{{init_vars.query}}

【当前状态】
{{route_design_stage.state_json_text}}
```

---

## 15. N23【parse_node】

如果没有 JSON 反序列化节点，继续用代码兜底。

### 15.1 代码

```javascript
async function main({ params }) {
  const raw = (params.raw_text || "").trim();
  let obj = {};

  try {
    obj = JSON.parse(raw);
  } catch (e) {
    obj = {
      stage_code: 30,
      intervention_node: "",
      why: "",
      need_more: true,
      next_question: "最早是从哪一个动作开始偏掉的？",
      state_patch: {
        intervention_node: "",
        why: ""
      },
      next_stage_code: 30,
      speak: "我先帮你找到最早能拦住它的那个点。"
    };
  }

  return {
    stage_code: Number(obj.stage_code || 30),
    intervention_node: obj.intervention_node || "",
    why: obj.why || "",
    need_more: Boolean(obj.need_more),
    next_question: obj.next_question || "",
    next_stage_code: Number(obj.next_stage_code || 30),
    speak: obj.speak || "",
    state_patch_json: JSON.stringify(obj.state_patch || {})
  };
}
```

---

## 16. N24【更新数据：update_state_after_node】

### 更新字段

| 字段名 | 新值 |
|---|---|
| stage_code | `parse_node.need_more ? 30 : 40` |
| state_json | `parse_node.state_patch_json` |
| last_question | `parse_node.next_question` |
| turn_no | `route_design_stage.turn_no + 1` |
| latest_intent_code | `route_action.intent_code` |
| updated_ts | `init_vars.now_ts` |

---

## 17. 阶段 40：N25【大模型：LLM_rule_gen】

### 17.1 作用

一次生成 6 条候选规则。

### 17.2 系统提示词（直接复制）

```text
你是“微规则设计师 / 国策拆解助手”。
你现在只负责候选规则生成。

任务目标：
基于当前干预节点，一次生成 6 条候选规则：
- 被动型 2 条
- 如果则型 2 条
- 机制型 2 条

规则：
1. 不做最终确认，只输出候选规则。
2. 只能输出纯 JSON。
3. 不要输出 Markdown，不要输出解释。
4. 每条规则都必须带评分字段。

输出 JSON 结构必须严格等于：
{
  "stage_code": 40,
  "candidates": [
    {
      "rule_text": "",
      "rule_type_code": 1,
      "friction_score": 1,
      "reliability_score": 1,
      "specificity_score": 1,
      "anti_loophole_note": ""
    }
  ],
  "need_more": false,
  "next_question": "",
  "state_patch": {
    "candidates": []
  },
  "next_stage_code": 50,
  "speak": ""
}
```

### 17.3 输入拼接模板

```text
【用户本轮输入】
{{init_vars.query}}

【当前状态】
{{route_design_stage.state_json_text}}
```

---

## 18. N26【parse_rule_gen】

### 18.1 输出

| 输出名 | 类型 |
|---|---|
| stage_code | Integer |
| candidates_json | String |
| next_stage_code | Integer |
| speak | String |

### 18.2 代码

```javascript
async function main({ params }) {
  const raw = (params.raw_text || "").trim();
  let obj = {};

  try {
    obj = JSON.parse(raw);
  } catch (e) {
    obj = {
      stage_code: 40,
      candidates: [],
      need_more: false,
      next_question: "",
      state_patch: {
        candidates: []
      },
      next_stage_code: 50,
      speak: "我已经生成了一组候选规则。"
    };
  }

  return {
    stage_code: Number(obj.stage_code || 40),
    candidates_json: JSON.stringify(Array.isArray(obj.candidates) ? obj.candidates : []),
    next_stage_code: Number(obj.next_stage_code || 50),
    speak: obj.speak || ""
  };
}
```

---

## 19. N27【更新数据：update_state_after_rule_gen】

### 更新字段

| 字段名 | 新值 |
|---|---|
| stage_code | `50` |
| state_json | `parse_rule_gen.candidates_json` |
| last_question | `""` |
| turn_no | `route_design_stage.turn_no + 1` |
| latest_intent_code | `route_action.intent_code` |
| updated_ts | `init_vars.now_ts` |

> 当前可跑版先把 `state_json` 直接存候选数组 JSON。  
> 后面要做成更完整上下文，再升级为对象。

---

## 20. 阶段 50：N28【代码：score_pick_rule】

### 20.1 作用

给候选规则打分，选出最佳规则。

### 20.2 输入

| 输入名 | 来源 |
|---|---|
| candidates_json | `parse_rule_gen.candidates_json` 或 `route_design_stage.state_json_text` |

### 20.3 输出

| 输出名 | 类型 |
|---|---|
| ranked_json | String |
| best_rule_json | String |
| pass | Boolean |
| next_stage_code | Integer |
| speak | String |

### 20.4 代码（直接复制）

```javascript
async function main({ params }) {
  let candidates = [];

  try {
    const raw = params.candidates_json || "[]";
    candidates = JSON.parse(raw);
    if (!Array.isArray(candidates)) candidates = [];
  } catch (e) {
    candidates = [];
  }

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
    ranked_json: JSON.stringify(ranked),
    best_rule_json: JSON.stringify(best || {}),
    pass,
    next_stage_code: pass ? 60 : 40,
    speak: pass
      ? `我选出了一条当前最稳的规则。`
      : "这批规则还是偏重，我会降门槛再生成一轮。"
  };
}
```

---

## 21. N29【选择器：pass_or_regen】

按 `score_pick_rule.pass` 分支：

| 条件 | 去向 |
|---|---|
| true | `LLM_plan` |
| false | `update_state_after_score`（回到阶段 40） |

---

## 22. N30【更新数据：update_state_after_score】

### 22.1 如果未通过

更新字段：

| 字段名 | 新值 |
|---|---|
| stage_code | `40` |
| state_json | `score_pick_rule.ranked_json` |
| last_question | `""` |
| turn_no | `route_design_stage.turn_no + 1` |
| updated_ts | `init_vars.now_ts` |

### 22.2 如果通过

也可以额外做一个“通过版更新”节点，先把：

- `stage_code = 60`
- `state_json = best_rule_json`

写回去，再进 `LLM_plan`。

为降低复杂度，第一轮你也可以直接把 `LLM_plan` 接在通过分支后面。

---

## 23. 阶段 60：N31【大模型：LLM_plan】

### 23.1 作用

根据最终被选中的规则，生成 7 天执行计划。

### 23.2 系统提示词（直接复制）

```text
你是“微规则设计师 / 国策拆解助手”。
你现在只负责“落地计划与入树前准备”。

任务目标：
基于当前已选中的最佳规则，给出：
- 7天执行计划
- 每天触发提醒
- 失败兜底动作

规则：
1. 不重新生成规则，只围绕已选中的规则做计划。
2. 只能输出纯 JSON。
3. 不要输出 Markdown，不要输出解释。
4. 字段必须完整。

输出 JSON 结构必须严格等于：
{
  "stage_code": 60,
  "selected_rule": {
    "rule_text": "",
    "rule_type_code": 1,
    "friction_score": 1,
    "reliability_score": 1,
    "specificity_score": 1
  },
  "plan_7d": {
    "daily_trigger": "",
    "checkin_question": "",
    "fallback_if_fail": ""
  },
  "next_stage_code": 99,
  "speak": ""
}
```

### 23.3 输入拼接模板

```text
【用户本轮输入】
{{init_vars.query}}

【最佳规则】
{{score_pick_rule.best_rule_json}}
```

---

## 24. N32【parse_plan】

### 24.1 输出

| 输出名 | 类型 |
|---|---|
| selected_rule_json | String |
| plan_7d_json | String |
| next_stage_code | Integer |
| speak | String |

### 24.2 代码

```javascript
async function main({ params }) {
  const raw = (params.raw_text || "").trim();
  let obj = {};

  try {
    obj = JSON.parse(raw);
  } catch (e) {
    obj = {
      stage_code: 60,
      selected_rule: {},
      plan_7d: {
        daily_trigger: "",
        checkin_question: "",
        fallback_if_fail: ""
      },
      next_stage_code: 99,
      speak: "我已经给你整理出一个可执行的 7 天方案。"
    };
  }

  return {
    selected_rule_json: JSON.stringify(obj.selected_rule || {}),
    plan_7d_json: JSON.stringify(obj.plan_7d || {}),
    next_stage_code: Number(obj.next_stage_code || 99),
    speak: obj.speak || ""
  };
}
```

---

## 25. N33【新增数据：insert_rule】

### 25.1 作用

把最终规则写进 `mr_rule_nodes`。

### 25.2 一个关键现实问题

Coze 的新增数据节点不一定能直接从 JSON 字符串里取内部字段。  
所以第一版最稳的做法是：

#### 做法 A（推荐）
在 `parse_plan` 之后再加一个代码节点，把 `selected_rule_json` 展开成平铺字段。

不过为了不把节点数量爆炸，这里也给你一个简化方案：

#### 做法 B（当前最简可跑版）
先只写这些稳定字段：

| 字段 | 值 |
|---|---|
| owner_key | `gen_owner_key.owner_key` |
| rule_id | 当前时间戳整数（用代码节点提前生成更稳） |
| parent_rule_id | `0` |
| root_id | 同 `rule_id` |
| depth | `1` |
| priority_order | `1` |
| rule_text | 暂时用 `score_pick_rule.best_rule_json` 或额外展开后的文本 |
| rule_type_code | 先固定 `1` 或由展开后的字段给出 |
| status_code | `1` |
| friction_score | 先固定或展开后填写 |
| reliability_score | 先固定或展开后填写 |
| specificity_score | 先固定或展开后填写 |
| success_days | `0` |
| fail_count | `0` |
| created_ts | `init_vars.now_ts` |
| updated_ts | `init_vars.now_ts` |

### 25.3 更推荐的增强版

在 `parse_plan` 后加一个代码节点 `flatten_selected_rule`，再写库。  
如果你愿意，下一阶段我可以把这个节点也补成标准件。

---

## 26. N34【更新数据：update_state_done】

### 26.1 更新条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| is_active | = | `true` |

### 26.2 更新字段

| 字段名 | 新值 |
|---|---|
| stage_code | `99` |
| state_json | `parse_plan.plan_7d_json` |
| last_question | `""` |
| turn_no | `route_design_stage.turn_no + 1` |
| latest_intent_code | `route_action.intent_code` |
| updated_ts | `init_vars.now_ts` |

---

## 27. N35【变量赋值：normalize_design_output】

### 27.1 作用

把设计分支里不同阶段的输出统一成一套格式，方便接到 `merge_output`。

### 27.2 当前最简做法

新增这些变量：

| 变量名 | 值 |
|---|---|
| speak | 优先用当前阶段节点的 `speak` |
| stage_code | 当前阶段更新后的值 |
| intent | `"continue_design"` |
| data_json | 当前阶段关键输出 JSON |

### 27.3 建议

第一版你可以在每个阶段后都各自接一个 `normalize_design_output`，  
或者把 `normalize_output` 放到每个阶段更新后统一处理。

---

## 28. 状态 JSON 现在先怎么存最稳

这是搭 Coze 时最容易纠结的点。

### 28.1 当前阶段的推荐策略

先不要追求一次把 `state_json` 设计得非常完美。  
第一阶段建议采用“能跑就先跑”的策略：

| 阶段 | state_json 先存什么 |
|---|---|
| diagnose | `state_patch_json` |
| timeline | `state_patch_json` |
| node | `state_patch_json` |
| rule_gen | `candidates_json` |
| stress_test | `best_rule_json` / `ranked_json` |
| plan | `plan_7d_json` |

### 28.2 为什么这样做

因为当前最重要的是：
- 阶段推进通
- 数据有地方可回写
- 后续能读到最近一次关键结果

等主链跑通后，再升级成“完整上下文合并版”。

---

## 29. 第一轮联调建议顺序

不要一次性把 10→60 全测。

### 29.1 联调顺序建议

#### Round 1：只测 diagnose
用户输入：

```text
我想早点睡
```

预期：
- 进入 stage 10
- 输出一个追问
- `mr_dialog_state.stage_code` 仍是 10 或推进到 20

#### Round 2：测 timeline
用户继续输入更具体的话：

```text
我一般晚上11点躺床上开始刷短视频
```

预期：
- 进入 stage 20
- 补出触发链
- 数据库更新为 20 或 30

#### Round 3：再补 node
预期：
- 能定位出“最早切入点”

#### Round 4：再补 rule_gen + stress_test
预期：
- 生成 6 条候选
- 代码能选出一条最佳规则

#### Round 5：最后补 plan + insert_rule
预期：
- `mr_rule_nodes` 里新增一条规则

---

## 30. 这一阶段你最容易踩的坑

### 坑 1：模型输出不是纯 JSON
解决：
- 提示词必须写死
- parse 节点必须有兜底逻辑

### 坑 2：更新数据条件不唯一
解决：
- 永远带 `owner_key + is_active`

### 坑 3：`state_json` 结构变化太大导致后面读不懂
解决：
- 先接受“阶段内局部 JSON”
- 主链通了再合并

### 坑 4：JSON 反序列化节点不稳定
解决：
- 统一改成代码节点 `JSON.parse` 兜底

### 坑 5：rule_gen 返回数组不稳定
解决：
- parse_rule_gen 里统一兜底成 `[]`

### 坑 6：insert_rule 无法直接从 JSON 字符串里取字段
解决：
- 下一阶段加 `flatten_selected_rule`
- 第一版先用简化写库方案

---

## 31. 这一阶段你做完后应该给我哪些截图

请优先发我下面这些截图，我下一轮就能继续带你补 Phase 4：

1. `reload_state` 节点配置图  
2. `route_design_stage` 节点配置图  
3. `design_stage_router` 选择器图  
4. `LLM_diagnose` 提示词配置图  
5. `parse_diagnose` 配置图  
6. `update_state_after_diagnose` 配置图  
7. `LLM_timeline` 提示词配置图  
8. `update_state_after_timeline` 配置图  
9. `LLM_rule_gen` 配置图  
10. `score_pick_rule` 代码图  
11. `LLM_plan` 配置图  
12. `insert_rule` 配置图  
13. 整个设计分支全景图

---

## 32. 本阶段完成标准

如果满足以下条件，就说明 Phase 3 成功：

1. `design_stage_router` 能按 stage 正确分流  
2. diagnose 输出能写回 `mr_dialog_state`  
3. timeline 输出能写回 `mr_dialog_state`  
4. node 输出能写回 `mr_dialog_state`  
5. rule_gen 能输出候选数组  
6. `score_pick_rule` 能筛出最佳规则  
7. plan 能给出 7 天执行计划  
8. `mr_rule_nodes` 能插入第一条规则  

---

## 33. 当前结论

Phase 3 的核心不是“做漂亮”，而是：

> **先把设计分支的阶段机跑通。**

只要这条主链通了，后面成功反馈、崩溃复盘、规则树查询都会变得非常顺。

