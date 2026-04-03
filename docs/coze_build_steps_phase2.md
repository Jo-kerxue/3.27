# Coze 微规则设计师 / 国策拆解助手 手把手搭建步骤（Phase 2）

> 本文档承接 `docs/coze_build_steps_phase1.md`。  
> 这一阶段的目标是：**把主工作流骨架真的搭出来，并确保状态初始化链跑通。**

---

## 1. 本阶段目标

这一阶段只做主工作流的最小闭环，不接大模型阶段机。

本阶段完成后，你的工作流必须能跑通：

```text
开始
→ 变量赋值 init_vars
→ 代码 gen_owner_key
→ 长期记忆检索 memory_query
→ 查询数据 state_latest
→ 选择器 has_state
   ├─ 无状态 → 新增数据 init_state
   └─ 有状态 → 继续
→ 意图识别 intent_cls
→ 代码 route_action
→ 选择器 intent_router
→ 输出
```

这一步跑通，说明以下关键基础都已经成立：

1. 用户输入能进入工作流  
2. `owner_key` 能算出来  
3. 数据库查询能按用户查当前状态  
4. 第一次进入时能自动初始化状态  
5. 意图识别结果能被后续代码理解  
6. 路由器能把对话分发到不同分支  

---

## 2. 开始前确认

正式搭节点前，先确认 4 件事：

### 2.1 你已经完成 Phase 1

也就是：

- 变量区已校准
- `mr_dialog_state` 已建好
- `mr_rule_nodes` 已建好
- `mr_event_log` 已建好（或至少准备好后续补）
- 长期记忆库可用
- 意图识别器可用

### 2.2 意图识别器名称

如果你现在还是用截图里的：

```text
mr_intent_v1
```

本阶段可以先继续用它。  
后面再统一升级为 `mr_intent_v2`。

### 2.3 当前阶段不处理的内容

这一阶段先**不做**：

- diagnose 大模型
- timeline 大模型
- rule_gen 大模型
- score_pick_rule
- plan 入树
- success/collapse/query 的完整逻辑

当前只把“骨架 + 初始化链 + 路由层”搭稳。

### 2.4 节点命名必须统一

为了后面排错方便，这一阶段请你把节点名直接改成下面这些：

| 顺序 | 节点类型 | 节点名称 |
|---:|---|---|
| N01 | 开始 | 开始 |
| N02 | 变量赋值 | init_vars |
| N03 | 代码 | gen_owner_key |
| N04 | 长期记忆检索 | memory_query |
| N05 | 查询数据 | state_latest |
| N06 | 选择器 | has_state |
| N07 | 新增数据 | init_state |
| N08 | 意图识别 | intent_cls |
| N09 | 代码 | route_action |
| N10 | 选择器 | intent_router |
| N11 | 变量聚合 | merge_output |
| N12 | 输出 | 输出 |

---

## 3. 工作流画布连线图

先按这个结构摆节点：

```text
N01 开始
  ↓
N02 init_vars
  ↓
N03 gen_owner_key
  ↓
N04 memory_query
  ↓
N05 state_latest
  ↓
N06 has_state
  ├─ 无状态 → N07 init_state ─┐
  └─ 有状态 ──────────────────┤
                              ↓
                          N08 intent_cls
                              ↓
                          N09 route_action
                              ↓
                          N10 intent_router
                 ├─ 设计分支 ───────────┐
                 ├─ 成功分支 ───────────┤
                 ├─ 崩溃分支 ───────────┤
                 ├─ 查询分支 ───────────┤
                 ├─ 暂停/恢复分支 ──────┤
                 └─ 结束分支 ───────────┘
                              ↓
                          N11 merge_output
                              ↓
                          N12 输出
```

> 说明：  
> 当前阶段虽然还没真正补全 6 个分支，但选择器先留好分叉位，后面不用重搭骨架。

---

## 4. N01【开始】配置

这一节点用系统默认开始节点即可。

### 4.1 你需要知道的唯一一件事

本轮用户输入会以系统变量形式进入工作流。  
在你的界面说明里已经写过：

```text
每次对话都会调用该对话流，用户“本轮对话输入”会作为对话流的输入参数 USER_INPUT 传入
```

所以后面在 `init_vars` 里，我们只做一件事：

```text
query = 开始.USER_INPUT
```

---

## 5. N02【变量赋值：init_vars】配置

### 5.1 作用

给后续所有节点准备统一的基础变量。

### 5.2 新增变量

在 `init_vars` 节点里配置以下变量：

| 变量名 | 类型 | 值 |
|---|---|---|
| query | String | 引用 `开始.USER_INPUT` |
| now_ts | String / Time | 当前时间 |
| limit | Integer | 固定填 `20` |
| default_stage_code | Integer | 固定填 `10` |
| root_parent_id | Integer | 固定填 `0` |
| debug_user_id | String | 固定填 `"demo_user"` |

> `debug_user_id` 是这一阶段特意增加的调试变量。  
> 如果你的 Coze 暂时不好拿系统真实用户 id，就先用这个值让流程能跑通。

### 5.3 这一步具体怎么点

1. 点开 `变量赋值` 节点  
2. 一条一条新增变量  
3. `query` 的值，选择：  
   - 系统变量 / 上游节点变量
   - 引用 `开始.USER_INPUT`
4. `limit` 直接填数字 `20`
5. `default_stage_code` 直接填 `10`
6. `root_parent_id` 直接填 `0`
7. `debug_user_id` 直接填字符串 `demo_user`

### 5.4 验收标准

这一步完成后，右侧变量输出区里你能看到这 6 个变量名。

---

## 6. N03【代码：gen_owner_key】配置

### 6.1 作用

把用户标识转成 Number 型 `owner_key`。  
因为 String 不能做关键索引，所以数据库查询必须尽量用数值键。

### 6.2 输入配置

你在代码节点里先定义输入参数：

| 输入名 | 类型 | 值来源 |
|---|---|---|
| raw_uid | String | 优先先引用 `init_vars.debug_user_id` |

> 当前阶段先不要纠结系统真实 uid。  
> 我们先让全链路跑通，后面再把 `raw_uid` 换成系统真实用户标识。

### 6.3 输出配置

在代码节点输出区定义：

| 输出名 | 类型 |
|---|---|
| owner_key | Number |

### 6.4 代码内容（JavaScript）

直接复制下面这段：

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

### 6.5 为什么这样做

因为：

- Coze 数据表里 String 不适合做关键索引
- 用户原始 uuid 往往是字符串
- 所以先哈希成 Number，再统一查库

### 6.6 验收标准

这一步完成后：

- 代码节点保存无报错
- 输出里有 `owner_key`

---

## 7. N04【长期记忆检索：memory_query】配置

### 7.1 作用

把用户相关偏好、失败模式等辅助信息先取出来。  
虽然这一阶段还没深度用到，但先把接口接上，后面不用改骨架。

### 7.2 记忆库

选你当前已经可用的长期记忆库。  
你截图里显示的是：

```text
try1
```

如果你要沿用，就选它。

### 7.3 输入配置

| 输入名 | 值 |
|---|---|
| query | `init_vars.query` |
| limit | `init_vars.limit` |

### 7.4 输出

按你截图当前格式，会输出：

- `outputList`
  - `output`
  - `date`

### 7.5 当前阶段怎么理解这个输出

现在先不用深挖，只要知道：

- 下游如果要给模型补充个性化信息，可以引用 `memory_query.outputList`

### 7.6 验收标准

节点能保存，无报错即可。

---

## 8. N05【查询数据：state_latest】配置

### 8.1 作用

查询这个用户当前是否已经存在一条活跃状态记录。

### 8.2 目标表

选择：

```text
mr_dialog_state
```

### 8.3 查询条件

你要加两个条件：

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| is_active | = | `true` 或 `1` |

> 如果你表里的 `is_active` 是 Boolean，就填 `true`  
> 如果你表里的 `is_active` 是 Integer，就填 `1`

### 8.4 排序方式

排序字段：

```text
updated_ts
```

排序：

```text
倒序
```

### 8.5 查询上限

固定填：

```text
1
```

### 8.6 输出设置

保留默认输出：

- `outputList`
- `rowNum`

### 8.7 这一步的目的

我们不是想拿很多历史数据，  
只是想知道：

> “这个用户现在有没有一条还在进行中的状态记录？”

### 8.8 验收标准

节点保存后无报错。

---

## 9. N06【选择器：has_state】配置

### 9.1 作用

根据 `state_latest.rowNum` 判断：

- 有没有历史状态

### 9.2 条件配置

配置两个分支：

#### 分支 A：有状态

条件：

```text
state_latest.rowNum > 0
```

#### 分支 B：无状态

否则分支。

### 9.3 连线规则

- “无状态”分支 → `N07 init_state`
- “有状态”分支 → 直接连 `N08 intent_cls`
- `N07 init_state` 的输出再连到 `N08 intent_cls`

### 9.4 这一步的意义

第一次使用这个智能体的人，没有状态记录。  
必须先创建一条默认状态，否则后面所有更新都没法对准对象。

---

## 10. N07【新增数据：init_state】配置

### 10.1 作用

当用户第一次进入时，往 `mr_dialog_state` 插入一条默认状态记录。

### 10.2 目标表

选择：

```text
mr_dialog_state
```

### 10.3 字段映射

在“选择并设置字段”里逐条设置：

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
| is_active | `true` 或 `1` |

### 10.4 注意点

#### `state_json`
这里先直接填：

```text
"{}"
```

意思是：当前还没有积累任何阶段上下文。

#### `latest_intent_code`
先填 `0`，表示还没真正进入业务意图处理。

### 10.5 验收标准

保存无报错。  
这一步在实际运行时，第一次进入应能成功写一条记录。

---

## 11. N08【意图识别：intent_cls】配置

### 11.1 作用

识别用户这轮输入属于哪一类意图。

### 11.2 输入配置

| 输入名 | 值 |
|---|---|
| query | `init_vars.query` |

### 11.3 分类器

你当前截图里已经有一个意图识别器，名称是：

```text
mr_intent_v1
```

这一阶段先沿用它。

### 11.4 当前输出形式

根据你截图，现在输出是：

```text
classificationId
```

类型是：

```text
Integer
```

### 11.5 这意味着什么

说明意图识别节点当前不会直接给你：

- `start_design`
- `continue_design`

这种文字名，而是给你一个整数编号。

所以：

> 后面必须用代码节点把 `classificationId` 映射成业务意图。

---

## 12. N09【代码：route_action】配置

### 12.1 作用

这是当前骨架里最重要的代码节点。  
它要做 3 件事：

1. 把 `classificationId` 转成业务意图码  
2. 读取当前阶段  
3. 告诉后面的选择器，本轮该走哪一个分支

### 12.2 输入配置

定义以下输入：

| 输入名 | 类型 | 值来源 |
|---|---|---|
| classificationId | Integer | `intent_cls.classificationId` |
| rowNum | Integer | `state_latest.rowNum` |
| current_stage_code | Integer | 先手动传 `10` 或后续中转变量 |

> 这里有一个现实问题：  
> 你的 Coze 查询节点输出 `outputList` 是数组对象，很多平台直接在代码节点里取 `outputList[0].stage_code` 会有点麻烦。  
> 所以本阶段先简化：
>
> - 如果当前你不方便从 `outputList[0]` 取 `stage_code`
> - 就先把 `current_stage_code` 直接传成 `10`
>
> 这样不影响骨架联调。  
> 下一阶段再补“读取已有状态并恢复阶段”。

### 12.3 输出配置

定义以下输出：

| 输出名 | 类型 | 含义 |
|---|---|---|
| intent_code | Integer | 当前意图码 |
| stage_code | Integer | 当前阶段码 |
| branch_code | Integer | 分支码 |
| need_init_state | Boolean | 是否刚初始化 |

### 12.4 代码内容（JavaScript）

直接复制下面这段：

```javascript
async function main({ params }) {
  const classificationId = Number(params.classificationId || 2);
  const stage_code = Number(params.current_stage_code || 10);
  const rowNum = Number(params.rowNum || 0);

  let branch_code = 1;

  // branch_code
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
    need_init_state: rowNum === 0
  };
}
```

### 12.5 当前这段代码的前提

它默认你意图识别器分类顺序最终会是：

| classificationId | 对应意图 |
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

> 如果你现在的 `mr_intent_v1` 只有 5 类，  
> 那么这一阶段你也可以先只用前 5 类，代码照样跑，只是后面两个分支暂时不会命中。  
> 等你后续升级到 `mr_intent_v2` 再完整覆盖。

---

## 13. N10【选择器：intent_router】配置

### 13.1 作用

根据 `route_action.branch_code` 做分流。

### 13.2 分支定义

至少先配 6 个分支口：

| branch_code | 分支含义 |
|---:|---|
| 1 | 设计分支 |
| 2 | 成功分支 |
| 3 | 崩溃分支 |
| 4 | 查询分支 |
| 5 | 暂停/恢复分支 |
| 6 | 结束分支 |

### 13.3 当前阶段怎么连

因为这一阶段还没真正补完这些分支逻辑，  
所以你可以先把所有分支都连到同一个 `merge_output`，但保留分支口。

例如：

- branch 1 → `merge_output`
- branch 2 → `merge_output`
- branch 3 → `merge_output`
- branch 4 → `merge_output`
- branch 5 → `merge_output`
- branch 6 → `merge_output`

这样骨架先通，后面分支一个个替换进去。

### 13.4 临时输出策略

当前阶段统一先输出一个调试话术，例如：

- 设计分支：`已进入设计分支`
- 成功分支：`已进入成功分支`
- 崩溃分支：`已进入崩溃分支`

如果你懒得分别做，也可以先统一由后面变量赋值做一版简单输出。

---

## 14. N11【变量聚合：merge_output】配置

### 14.1 作用

把多个分支的输出合并成一个统一输出，供最后输出节点使用。

### 14.2 当前阶段的简化做法

由于现在还没真正接入复杂分支，所以你可以先聚合一个字符串输出：

例如定义一个聚合字段：

| 聚合字段名 | 类型 |
|---|---|
| final_text | String |

每个上游分支先给一个字符串。

### 14.3 过渡方案

如果你暂时还没在各个分支产出字符串变量，  
也可以先不加 `merge_output`，直接把 `intent_router` 某个统一分支接到 `输出`。  
但长期看，保留 `merge_output` 更稳。

---

## 15. N12【输出】配置

### 15.1 当前阶段目标

先能看到工作流确实跑到了终点，并且知道进入了哪个分支。

### 15.2 临时输出内容建议

当前阶段你可以让输出至少包含：

- 用户输入 query
- owner_key
- branch_code
- 一个简短的调试文本

如果你的输出节点只方便输出一个字符串，那就先输出：

```text
初始化链已跑通，branch_code = X
```

等下阶段再升级成统一 JSON 协议输出。

---

## 16. 本阶段的最小联调方法

现在开始测试最小链路。

### 16.1 测试输入 1

```text
我想改掉晚睡
```

期望结果：

1. `init_vars.query` 有值  
2. `gen_owner_key.owner_key` 有数值  
3. `state_latest.rowNum` 在第一次应为 `0`  
4. `init_state` 被执行  
5. `intent_cls.classificationId` 有整数输出  
6. `route_action.branch_code` 应走设计分支  
7. 最终能输出一段调试信息  

### 16.2 测试输入 2

```text
看看我的规则
```

期望结果：

1. 第二次进入时，`state_latest.rowNum` 应大于 0  
2. 不再执行 `init_state`  
3. 路由应走查询分支  

### 16.3 测试输入 3

```text
今天做到了
```

期望结果：

1. 路由应走成功分支  
2. 当前虽然还没补成功分支逻辑，但至少 branch_code 应正确  

---

## 17. 如果你现在遇到问题，优先按这个顺序排查

### 问题 1：查询数据查不到

先检查：

1. `owner_key` 有没有成功算出来  
2. `mr_dialog_state` 里是否真的有记录  
3. `is_active` 用的是 `true` 还是 `1`，要和表类型一致  

### 问题 2：新增数据报错

先检查：

1. 字段名有没有写错  
2. `updated_ts` 类型是否匹配  
3. `owner_key` 输出是不是 Number  

### 问题 3：意图识别输出看不懂

先别急着追求文字名，  
当前阶段只要能拿到 `classificationId` 就够。

### 问题 4：代码节点取不到数组里的 stage_code

这个是正常的。  
本阶段我们已经故意规避了：

- 先固定 `current_stage_code = 10`

下一阶段再补从 `outputList[0]` 中取字段。

---

## 18. 本阶段完成后的验收标准

你完成这一阶段后，必须满足：

### 18.1 结构验收

工作流画布上至少有：

- 开始
- init_vars
- gen_owner_key
- memory_query
- state_latest
- has_state
- init_state
- intent_cls
- route_action
- intent_router
- 输出

### 18.2 运行验收

至少 3 个测试输入能跑通，不报错：

1. 我想改掉晚睡  
2. 看看我的规则  
3. 今天做到了  

### 18.3 状态验收

第一次运行后，`mr_dialog_state` 至少新增一条记录，字段值合理：

- owner_key = 数字
- stage_code = 10
- state_json = "{}"
- is_active = true / 1

---

## 19. 你完成这一阶段后，下一阶段做什么

下一阶段就是：

# Phase 3：正式接入设计分支阶段机

会开始接这些节点：

1. `LLM_diagnose`
2. `LLM_timeline`
3. `LLM_node`
4. `LLM_rule_gen`
5. `score_pick_rule`
6. `LLM_plan`

并且会开始把输出从“调试字符串”升级成统一 JSON 协议。

---

## 20. 当前阶段最终结论

本阶段你只需要记住一句话：

> **先把“初始化状态 + 意图识别 + 路由骨架”搭通，再接复杂业务节点。**

这一步稳了，后面就会顺很多。
