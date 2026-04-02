# Coze 微规则设计师 / 国策拆解助手 手把手搭建步骤（Phase 1）

> 本文档是基于当前已经锁定的方案写的**第一阶段实操文档**。  
> 目标不是一次把全部功能做完，而是先把**基础资源 + 初始化链 + 意图路由骨架**稳定搭起来。  
> 只要这一阶段搭对，后面接大模型阶段机会非常稳。

---

## 1. 本阶段目标

这一阶段只完成下面 4 件事：

1. 校准智能体变量区
2. 校准 3 张核心表
3. 校准意图识别器
4. 搭出主工作流最小骨架

本阶段结束时，你的工作流至少要能跑通：

```text
开始
→ 变量赋值
→ 代码生成 owner_key
→ 长期记忆检索
→ 查询当前状态
→ 无状态则初始化状态
→ 意图识别
→ 路由分支
→ 输出
```

---

## 2. 先建立正确认知：数据表字段不是变量

在 Coze 里必须把 3 种东西分清：

### 2.1 工作流变量

例如：

- `query`
- `now_ts`
- `limit`
- `owner_key`
- `classificationId`

这些值只在本次工作流运行时存在。

### 2.2 数据表字段

例如 `mr_dialog_state` 表里的：

- `owner_key`
- `stage_code`
- `state_json`

这些是表结构里的列，不会自己变化。

### 2.3 数据表记录

例如数据库里真实的一行：

- owner_key = 123456
- stage_code = 10
- state_json = "{}"

这才是最终被保存的数据。

### 2.4 正确数据流

```text
开始输入
→ 变量赋值 / 代码节点 / 大模型节点得到中间变量
→ 新增数据 / 更新数据节点
→ 数据表记录被写入或被修改
```

### 2.5 两个关键结论

#### 给数据表“赋值”

不是直接在数据表里赋值，而是：

```text
准备变量 → 新增数据节点写入数据库
```

#### 修改数据表里的值

不是在表里手动改，而是：

```text
查询旧记录 → 算新值 → 更新数据节点写回数据库
```

---

## 3. 智能体外壳配置

你现在已经有一个智能体外壳，继续沿用。

建议名称固定为：

```text
微规则拆解师
```

### 3.1 开场白建议

```text
你好，我是能帮你拆解复杂规则的智能体，让规则变得清晰可执行。
```

### 3.2 推荐问题建议

- 请拆解这条规则为微规则
- 如何把复杂规则拆分成最小单元
- 你能帮我分析规则的执行步骤吗

---

## 4. 变量区配置

在智能体左侧变量区中，固定保留这 5 个变量。

| 变量名 | 类型 | 固定值 / 说明 |
|---|---|---|
| query | String | 本轮用户输入 |
| now_ts | String 或 Time | 当前时间 |
| limit | Integer | 固定填 `20` |
| default_stage_code | Integer | 固定填 `10` |
| root_parent_id | Integer | 固定填 `0` |

### 4.1 配置动作

如果变量区里没有，就新增。  
如果名字已经有类似变量，但命名不一致，就统一改成上面这 5 个名字。

### 4.2 为什么这样配

- `query`：后面给意图识别、大模型、长期记忆检索复用
- `now_ts`：后面写数据库时统一作为时间戳来源
- `limit`：查询节点统一复用，不要每个节点手动写
- `default_stage_code`：初始化状态时用
- `root_parent_id`：后面建根规则时复用

---

## 5. 数据表配置

本阶段先只校准 3 张表：

1. `mr_dialog_state`
2. `mr_rule_nodes`
3. `mr_event_log`

> 如果你现在已有 `mr_collapse_log`，可以保留，但后续统一主日志表用 `mr_event_log`。

---

## 6. 表一：mr_dialog_state

### 6.1 用途

保存“当前这轮会话做到哪一步”。

### 6.2 字段

| 字段名 | 类型 | 说明 |
|---|---|---|
| owner_key | Number | 用户数值键 |
| stage_code | Integer | 当前阶段 |
| state_json | String | 当前阶段完整 JSON 文本 |
| last_question | String | 上一轮问题 |
| turn_no | Integer | 轮次 |
| latest_intent_code | Integer | 最近意图码 |
| current_root_id | Integer | 当前根规则 id |
| current_node_id | Integer | 当前节点 id |
| pending_candidate_idx | Integer | 待确认候选序号 |
| updated_ts | Time | 更新时间 |
| is_active | Boolean / Integer | 当前是否激活 |

### 6.3 字段填写建议

如果 `is_active` 不方便用 Boolean，就用 Integer：

- 1 = 激活
- 0 = 关闭

---

## 7. 表二：mr_rule_nodes

### 7.1 用途

保存规则树节点。

### 7.2 字段

| 字段名 | 类型 | 说明 |
|---|---|---|
| owner_key | Number | 用户键 |
| rule_id | Integer | 规则 id |
| parent_rule_id | Integer | 父节点 id |
| root_id | Integer | 根规则 id |
| depth | Integer | 层级 |
| priority_order | Integer | 同层顺序 |
| title | String | 标题 |
| trigger_scene | String | 触发场景 |
| rule_text | String | 规则正文 |
| action_text | String | 触发动作 |
| failure_signal | String | 失败信号 |
| note_text | String | 备注 |
| rule_type_code | Integer | 规则类型码 |
| status_code | Integer | 状态码 |
| friction_score | Integer | 阻力分 |
| reliability_score | Integer | 可靠性分 |
| specificity_score | Integer | 具体性分 |
| strength_level | Integer | 强化等级 |
| tolerance_quota | Integer | 容错额度 |
| success_days | Integer | 连续成功天数 |
| fail_count | Integer | 崩溃次数 |
| created_ts | Time | 创建时间 |
| updated_ts | Time | 更新时间 |

---

## 8. 表三：mr_event_log

### 8.1 用途

统一记录：

- 成功
- 崩溃
- 查询
- 系统动作

### 8.2 字段

| 字段名 | 类型 | 说明 |
|---|---|---|
| owner_key | Number | 用户键 |
| event_id | Integer | 事件 id |
| event_type_code | Integer | 事件类型 |
| rule_id | Integer | 关联规则 |
| payload_json | String | 事件详情 |
| created_ts | Time | 创建时间 |

---

## 9. 固定编码

### 9.1 阶段码 stage_code

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

### 9.2 意图码 intent_code

| 编码 | 含义 |
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

---

## 10. 意图识别器配置

分类器名建议固定：

```text
mr_intent_v2
```

如果你当前已经有 `mr_intent_v1`，Phase 1 也可以先继续用。

### 10.1 当前阶段必须配置的意图

本阶段建议先把 9 类都配好：

- start_design
- continue_design
- confirm_rule
- reject_rule
- report_success
- report_collapse
- query_rules
- activate_pause
- finish_session

### 10.2 当前截图确认到的一个关键点

你给的 Coze 界面里，意图识别节点输出的是：

```text
classificationId
```

不是意图名字。

所以后面必须加一个代码节点，把 `classificationId` 映射成我们自己的 `intent_code`。

---

## 11. 主工作流 Phase 1 骨架

本阶段只摆这些节点：

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

## 12. 节点手把手配置

---

### N01【开始】

使用系统默认开始节点即可。

你后面在变量赋值里引用它的：

```text
USER_INPUT
```

---

### N02【变量赋值：init_vars】

新增并赋值：

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

把用户标识转成 Number 类型，解决 String 不能做索引的问题。

#### 输入

| 输入名 | 值 |
|---|---|
| raw_uid | 优先使用系统用户 id；如果当前界面一时拿不到，先填 `"demo_user"` 调试 |

#### 输出

| 输出名 | 类型 |
|---|---|
| owner_key | Number |

#### JavaScript 代码

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

#### 记忆库

选择你当前已经绑定的长期记忆库。

#### 输入

| 输入名 | 值 |
|---|---|
| query | `init_vars.query` |
| limit | `init_vars.limit` |

#### 输出

按你的截图，当前可用输出是：

- `outputList`
  - `output`
  - `date`

---

### N05【查询数据：state_latest】

#### 目标表

`mr_dialog_state`

#### 查询条件

| 字段 | 条件 | 值 |
|---|---|---|
| owner_key | = | `gen_owner_key.owner_key` |
| is_active | = | `true` 或 `1`（按你的字段类型） |

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

- `state_latest.rowNum > 0` → 有状态
- 否则 → 无状态

---

### N07【新增数据：init_state】

只接在 “无状态” 分支后。

#### 目标表

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
| is_active | `true` 或 `1` |

#### 这里你要理解的点

这一步就是“给数据表赋值”的标准例子。

不是去表里手工改，而是：

```text
左边选字段
右边填固定值 / 上游变量
```

---

### N08【意图识别：intent_cls】

#### 输入

| 输入名 | 值 |
|---|---|
| query | `init_vars.query` |

#### 输出

按你的截图，当前拿到：

| 输出名 | 类型 |
|---|---|
| classificationId | Integer |

---

### N09【代码：route_action】

#### 作用

1. 把 `classificationId` 转成 `intent_code`
2. 读取当前 `stage_code`
3. 判断本轮走哪个分支

#### 输入

| 输入名 | 值 |
|---|---|
| classificationId | `intent_cls.classificationId` |
| current_stage_code | 有历史状态就传当前 stage，没有就传 `10` |

> 如果你当前 Coze 不方便直接从 `outputList[0]` 里取字段，先加一个中间变量赋值节点做展开也可以。

#### 输出

| 输出名 | 类型 |
|---|---|
| intent_code | Integer |
| stage_code | Integer |
| branch_code | Integer |

#### branch_code 约定

| branch_code | 含义 |
|---|---|
| 1 | 设计分支 |
| 2 | 成功分支 |
| 3 | 崩溃分支 |
| 4 | 查询分支 |
| 5 | 暂停/恢复分支 |
| 6 | 结束分支 |

#### JavaScript 代码

```javascript
async function main({ params }) {
  const classificationId = Number(params.classificationId || 2);
  const stage_code = Number(params.current_stage_code || 10);
  let branch_code = 1;

  if (classificationId === 5) branch_code = 2;
  else if (classificationId === 6) branch_code = 3;
  else if (classificationId === 7) branch_code = 4;
  else if (classificationId === 8) branch_code = 5;
  else if (classificationId === 9) branch_code = 6;
  else branch_code = 1;

  return {
    intent_code: classificationId,
    stage_code,
    branch_code
  };
}
```

---

### N10【选择器：intent_router】

按 `route_action.branch_code` 分流：

| branch_code | 去向 |
|---|---|
| 1 | 设计分支 |
| 2 | 成功分支 |
| 3 | 崩溃分支 |
| 4 | 查询分支 |
| 5 | 暂停/恢复分支 |
| 6 | 结束分支 |

> 本阶段暂时只要把骨架分出来，不用马上把所有分支内部都搭完。

---

## 13. Phase 1 验收标准

本阶段完成后，你必须能验证：

1. 用户输入能进入 `query`
2. `owner_key` 能成功输出 Number
3. `mr_dialog_state` 能按 owner_key 查询
4. 如果没有状态，能自动新增一条初始状态记录
5. 意图识别能产出 `classificationId`
6. `route_action` 能输出 `branch_code`
7. `intent_router` 能正常分流

---

## 14. 这一阶段做完后应该截图什么

做完 Phase 1 后，建议固定拍这几类截图：

1. 变量区截图
2. `mr_dialog_state` 字段页
3. `mr_rule_nodes` 字段页
4. `mr_event_log` 字段页
5. `gen_owner_key` 代码节点配置页
6. `state_latest` 查询数据节点配置页
7. `init_state` 新增数据节点配置页
8. `intent_cls` 意图识别节点配置页
9. `route_action` 代码节点配置页
10. 整个工作流骨架全景图

---

## 15. Phase 2 预告（下一阶段做什么）

Phase 1 完成后，下一阶段再接：

- diagnose
- timeline
- node
- rule_gen
- stress_test
- plan

也就是正式把“设计分支”搭起来。

---

## 16. 当前结论

到这里，你已经有了：

- 明确的数据表赋值思路
- 明确的数据表更新思路
- 可执行的初始化链
- 可执行的意图路由骨架

这一步完成后，后面再接大模型阶段机就会稳定很多。
