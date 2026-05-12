# SOP 研判 WebSocket 推送报文说明（前端联调）

本文描述 **SOP 研判** 专用 WebSocket 连接建立后，服务端可能下发的 **各类文本帧 JSON** 形态、字段含义及与 MQ 内层载荷的关系，便于与 **SSE `/api/v1/datapool/stream/sop`** 对齐实现（**业务帧 JSON 结构一致**，仅传输载体不同）。

**实现参考**：`SopJudgeWebSocketHandler`、`StreamServiceImpl#pushJudgeEvent`、`DataConverterService#convertJudgeEventToSSE`、`SopJudgeWebSocketHandler#handleTextMessage`（心跳应答）。

---

## 1. 连接地址与过滤参数

- **路径**：`ws(s)://{host}/api/v1/datapool/ws/stream/sop`
- **查询参数**（与 SSE `GET /api/v1/datapool/stream/sop` 一致，可多选逗号分隔）：
  - `sourceIds`：按 `meta_data.sourceId` / 载荷顶层同源字段匹配
  - `planIds`：`meta_data.planId`
  - `skillIds`：`meta_data.skillId`
  - `tenantIds`：`meta_data.tenantId`
- **全量订阅**：上述四个参数**均不传或均为空**时，该连接进入「广播」集合，**收到所有**研判推送（与概设中「无过滤条件全量」一致）。
- **鉴权**：若网关对 WebSocket 要求与 HTTP 一致的 OpenAPI 头（如 `cappkey`），由部署/网关约定；Handler 本身只解析上述 query。

---

## 2. 下行消息总览（服务端 → 前端）

| 场景 | 说明 |
|------|------|
| 连接成功 | 建立后**立即**下发一条 **connect 确认**（见 §3.1） |
| 周期保活 | 约 **每 30 秒** 下发 **heartbeat**（与 Workflow/SOP SSE 共用定时器，见 §3.2） |
| 业务事件 | 每消费一条研判 MQ 事件并过过滤后，下发一条 **sop_event 包装**（见 §3.3） |
| 客户端 ping | 前端发送文本 `ping` 或 `"ping"` 时，服务端回复 **pong**（见 §3.4） |

**解析建议**：每条消息均为 **单行 UTF-8 JSON**。先 `JSON.parse`，再按根字段 **`type`** 分支；业务帧另看 **`event`**（内层 SOP 事件名）。

---

## 3. 各类消息 JSON 形态

### 3.1 连接确认（connect）

连接注册成功后下发，**不属于研判业务**。

```json
{
  "type": "connect",
  "message": "ok"
}
```

---

### 3.2 心跳（heartbeat）

用于检测连接与中台存活；**无业务语义**，可仅更新「上次收到服务端时间」。

```json
{
  "type": "heartbeat",
  "timestamp": 1775737312000
}
```

- `timestamp`：服务端生成时的 **Unix 毫秒时间戳**（`long`）。

---

### 3.3 研判业务帧（sop_event）

与 **SSE 事件名 `judge` 的 data 字符串** 为**同一 JSON**（`StreamServiceImpl#pushJudgeEvent` 中 `judgeEventData.toJSONString()` 既用于 SSE 也用于 WebSocket）。

**外层包装（固定三字段）**：

```json
{
  "type": "sop_event",
  "event": "<内层 SOP 事件类型，如 instance_end>",
  "data": { }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | 固定 `"sop_event"` | 与 SSE 的「事件名 `judge`」区分；WebSocket 统一用 `type` 识别帧种类 |
| `event` | string | **内层** `JudgeSopEvents` 的 `event` 字段，如 `sop_start`、`monitor_start`、`instance_start`、`node_start`、`node_end`、`instance_end`、`sop_end` |
| `data` | object | **规范载荷**：与落库 `normalized_payload`、MQ 解信封后 **Jackson 再序列化** 的内层 JSON **同源**（Fastjson `JSONObject`） |

**前端处理**：`switch (parsed.event)` 或 `switch (parsed.data?.event)`（注意根级已有 `event`，一般 **`parsed.event` 即可**）；`parsed.data` 为业务字段主体。

#### 3.3.1 `data` 各事件类型结构与示例

> 以下只展示 `data` 内层；实际 WebSocket 文本帧外层固定为 `{"type":"sop_event","event":"...","data":{...}}`。

- `sop_start`

```json
{
  "event": "sop_start",
  "meta_data": {
    "planId": "plan_003",
    "planName": "视频分析计划003",
    "skillId": "skill_003",
    "skillName": "人员检测",
    "sourceType": "file",
    "sourceId": "file_001",
    "sourceName": "20260303_120000.mp4",
    "taskId": "task_001",
    "planType": "manual",
    "addType": "manual",
    "plan_create_type": "manual",
    "tenantId": "tenant_demo"
  },
  "description": "七步洗手流程，用于监测洗手操作是否符合标准流程",
  "timestamp": 1775737257000
}
```

- `monitor_start`

```json
{
  "event": "monitor_start",
  "meta_data": {
    "planId": "plan_003",
    "planName": "视频分析计划003",
    "skillId": "skill_003",
    "skillName": "人员检测",
    "sourceType": "file",
    "sourceId": "file_001",
    "sourceName": "20260303_120000.mp4",
    "taskId": "task_001",
    "planType": "manual",
    "addType": "manual",
    "plan_create_type": "manual",
    "tenantId": "tenant_demo"
  },
  "description": "七步洗手流程，用于监测洗手操作是否符合标准流程",
  "timestamp": 1775737257000
}
```

- `instance_start`（开始出现 `instance_id`）

```json
{
  "event": "instance_start",
  "meta_data": {
    "planId": "plan_003",
    "planName": "视频分析计划003",
    "skillId": "skill_003",
    "skillName": "人员检测",
    "sourceType": "file",
    "sourceId": "file_001",
    "sourceName": "20260303_120000.mp4",
    "taskId": "task_001",
    "planType": "manual",
    "addType": "manual",
    "plan_create_type": "manual",
    "tenantId": "tenant_demo"
  },
  "instance_id": "sop_inst_8f04fdb7",
  "timestamp": 1775737276000
}
```

- `node_start`（无节点级 `start_time`）

```json
{
  "event": "node_start",
  "meta_data": {
    "planId": "plan_003",
    "planName": "视频分析计划003",
    "skillId": "skill_003",
    "skillName": "人员检测",
    "sourceType": "file",
    "sourceId": "file_001",
    "sourceName": "20260303_120000.mp4",
    "taskId": "task_001",
    "planType": "manual",
    "addType": "manual",
    "plan_create_type": "manual",
    "tenantId": "tenant_demo"
  },
  "instance_id": "sop_inst_8f04fdb7",
  "node_id": 1,
  "name": "打湿手",
  "timestamp": 1775737276000
}
```

- `node_end`（含 `camera_id` / `camera_info`，无节点级 `start_time`/`end_time`）

```json
{
  "event": "node_end",
  "meta_data": {
    "planId": "plan_003",
    "planName": "视频分析计划003",
    "skillId": "skill_003",
    "skillName": "人员检测",
    "sourceType": "file",
    "sourceId": "file_001",
    "sourceName": "20260303_120000.mp4",
    "taskId": "task_001",
    "planType": "manual",
    "addType": "manual",
    "plan_create_type": "manual",
    "tenantId": "tenant_demo"
  },
  "instance_id": "sop_inst_8f04fdb7",
  "node_id": 1,
  "name": "打湿手",
  "status": "success",
  "actual_duration": 1.2,
  "duration": null,
  "timeout": null,
  "actual_timeout": null,
  "camera_id": "sink_cam_01",
  "camera_info": "sink_area",
  "detail": "",
  "frame_urls": [],
  "timestamp": 1775737276000
}
```

- `instance_end`（顶层无 `start_time`/`end_time`；`timeline[*].start_time/end_time` 为毫秒）

```json
{
  "event": "instance_end",
  "meta_data": {
    "planId": "plan_003",
    "planName": "视频分析计划003",
    "skillId": "skill_003",
    "skillName": "人员检测",
    "sourceType": "file",
    "sourceId": "file_001",
    "sourceName": "20260303_120000.mp4",
    "taskId": "task_001",
    "planType": "manual",
    "addType": "manual",
    "plan_create_type": "manual",
    "tenantId": "tenant_demo"
  },
  "instance_id": "sop_inst_8f04fdb7",
  "status": "success",
  "total_duration": 35.721116,
  "timeline": [
    {
      "node_id": 1,
      "name": "打湿手",
      "status": "success",
      "duration": null,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": 1775737276000,
      "end_time": 1775737277200,
      "detail": ""
    }
  ],
  "anomalies": [],
  "timestamp": 1775737312000
}
```

- `sop_end`（不带 `instance_id`）

```json
{
  "event": "sop_end",
  "meta_data": {
    "planId": "plan_003",
    "planName": "视频分析计划003",
    "skillId": "skill_003",
    "skillName": "人员检测",
    "sourceType": "file",
    "sourceId": "file_001",
    "sourceName": "20260303_120000.mp4",
    "taskId": "task_001",
    "planType": "manual",
    "addType": "manual",
    "plan_create_type": "manual",
    "tenantId": "tenant_demo"
  },
  "timestamp": 1775737312000
}
```

---

### 3.4 客户端 ping → 服务端 pong

前端可发**纯文本**（非 JSON 也可被识别）：

- `ping` 或 `Ping`（大小写不敏感）
- 或 JSON 字符串 `"ping"`

服务端回复：

```json
{
  "type": "pong",
  "timestamp": 1775737312000
}
```

---

## 4. `data` 内层字段约定（与 MQ / 脚本对齐）

以下与 **`JudgeSopEvents` + `JudgeSopEventMetaData`** 及联调示例 **`sop_judge_data_demos.txt`** 一致，便于与 RocketMQ 内层对照。

### 4.1 通用

- **`meta_data`**：对象，键名以 **camelCase** 为主（如 `planId`、`tenantId`），与 `JudgeSopEventMetaData` 一致。
- **`timestamp`**：业务时间，**Unix 毫秒**数字（若上游未传，以服务端解析结果为准）。
- **`instance_id`**：**仅**出现在 `instance_start`、`instance_end`、`node_start`、`node_end`；`sop_start` / `monitor_start` / `sop_end` 根级**不应带** `instance_id`（反序列化后可能为 `null`，序列化时可能省略该键）。

### 4.2 按 `event` 的 `data` 要点

| `event` | `data` 中前端应关注 |
|-----------|---------------------|
| `sop_start` | `description`（若有）、`meta_data`、`timestamp` |
| `monitor_start` | 同上 |
| `instance_start` | `instance_id`、`meta_data`、`timestamp` |
| `node_start` | `instance_id`、`node_id`、`name`、`meta_data`、`timestamp`（**无**节点 `start_time` 约定） |
| `node_end` | `instance_id`、`node_id`、`name`、`status`、`actual_duration`、`duration`、`timeout`、`actual_timeout`、`camera_id`、`camera_info`、`detail`、`frame_urls`、`timestamp`；**无**节点级 `start_time`/`end_time` 约定 |
| `instance_end` | `instance_id`、`description`、`status`、`total_duration`、`timeline`、`anomalies`、`meta_data`、`timestamp`；**无**顶层 `start_time`/`end_time` 约定；`timeline[]` 内 `start_time`/`end_time` 为 **Unix 毫秒** |
| `sop_end` | `meta_data`、`timestamp`（**无**根级 `instance_id` 约定；上游若携带 `status`，当前强类型规范化后默认不保留） |

### 4.3 `node_end` 与 `frame_urls`

消费链路中，服务端可能对 `node_end` 的 **`frame_urls`** 做图片上传，再将 `frame_urls` **替换为对象存储 key 列表**后再序列化推送（与入库一致）。前端展示时需与 **HTTP 详情接口**（预签名 URL）策略对齐，**勿假定** WebSocket 里永远是 HTTP 直链。

### 4.4 未知事件类型

若将来出现未在 `JudgeSopEvents` 注册的事件，解析器可能走 **`Unknown`** 分支，此时 `data` 为保留的原始内层 JSON 对象，字段以实际上游为准；`event` 根级仍为内层 `event` 字符串。

---

## 5. 订阅过滤如何生效

服务端从 **`data` 与 `meta_data`** 中解析 `sourceId`、`planId`、`skillId`、`tenantId`（见 `JudgePayloadFieldConstants` 与 `firstFromPayloadOrMeta`），与连接上的 filter 做交集匹配：

- 连接带过滤：仅当消息中对应维度**命中**其一即推送（各维度为 AND：plan 列表须命中 plan，且 source 须命中 source，等）。
- 连接为广播：所有研判消息均推送。

**注意**：`sop_start` / `monitor_start` / `sop_end` **无** `instance_id` 时，仍可通过 **`meta_data`** 中的 `planId` 等参与过滤；若过滤条件过窄，可能导致「有 MQ 无 WebSocket」的表象，联调时需核对 meta 是否带齐维度。

---

## 6. 与 SSE 的差异（给前端的结论）

| 项目 | SSE `/stream/sop` | WebSocket `/ws/stream/sop` |
|------|-------------------|-----------------------------|
| 业务内容 | `event:judge`，`data` 为 **同一 JSON 字符串** | 文本帧即该 **整段 JSON 字符串**（内含 `type":"sop_event"`） |
| 心跳 | `event:heartbeat`，`data` 为 heartbeat JSON | 文本帧直接为 heartbeat JSON |
| connect | 无单独帧 | 首帧 **connect** |
| 客户端保活 | 一般依赖浏览器/EventSource | 可发 **ping** 换 **pong** |

---

## 7. 最小前端伪代码

```javascript
const ws = new WebSocket(`${wssBase}/api/v1/datapool/ws/stream/sop?planIds=${planId}`);
ws.onmessage = (ev) => {
  const msg = JSON.parse(ev.data);
  if (msg.type === "connect") return;
  if (msg.type === "heartbeat") return;
  if (msg.type === "pong") return;
  if (msg.type === "sop_event") {
    const { event, data } = msg;
    // 按 event 渲染：data 为 snake_case 主契约 + meta_data camelCase
    onSopJudgeEvent(event, data);
  }
};
// 可选：ws.send("ping");
```

---

## 8. 相关文件

- MQ 内层示例行：`docs/PreliminaryDesign/alert-message-center/sop_judge_data_demos.txt`
- 强类型定义：`JudgeSopEvents`、`JudgeSopEventMetaData`
- 推送组装：`DataConverterService#convertJudgeEventToSSE`、`StreamServiceImpl#pushJudgeEvent`
