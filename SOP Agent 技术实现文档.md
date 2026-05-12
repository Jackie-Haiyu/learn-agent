# SOP Agent 技术实现文档

---

## 0. 执行流程示例（洗手任务）

以下以**七步洗手流程**为例，展示SOP Agent的完整执行过程。

### 0.1 SOP配置定义

```yaml
# hand_wash.yaml - 洗手流程配置（多摄像头，SOP仅定义逻辑摄像头）
description: "七步洗手流程，用于监测洗手操作是否符合标准流程"
name: "七步洗手流程"
version: "1.1"

# 全局参数
timeout: 300           # instance整体超时（秒）
window:
  back_window: 2       # 回看窗口：检测前几个环节是否有补做
  forward_window: 2    # 向前窗口：检测后几个环节是否有跳步

# 逻辑摄像头定义（仅保留稳定标识，不写死真实URL）
cameras:
  - id: "sink_cam_01"           # 技术唯一ID，节点绑定用
    role: "sink_area"           # 业务语义，便于部署自动映射
  - id: "soap_cam_01"
    role: "soap_dispenser_area"
  - id: "dryer_cam_01"
    role: "hand_dryer_area"

# 节点定义
nodes:
  - id: 1
    name: "打湿手"
    prev_node: null              # 上一个节点名字，null表示起始节点
    camera_id: "sink_cam_01"     # 节点绑定摄像头（引用cameras[].id）
    model: "qwen3-vl"      # 模型
    instruct: "判断是否在打湿手" # 描述这个节点要干啥
    tools:                      # 工具列表（目前只有抽帧工具）
      - "frame_capture"         # 抽帧工具
    duration: null              # 持续时长（秒），null表示不限制
    timeout: null              # 超时（秒），null表示不限制

  - id: 2
    name: "挤洗手液"
    prev_node: "打湿手"
    camera_id: "soap_cam_01"
    model: "qwen3-vl"
    instruct: "判断是否在挤洗手液"
    tools:
      - "frame_capture"
    duration: null
    timeout: 10                 # 前置环节完成后10秒内必须开始

  - id: 3
    name: "搓手"
    prev_node: "挤洗手液"
    camera_id: "sink_cam_01"
    model: "qwen3-vl"
    instruct: "判断是否有搓手动作"
    tools:
      - "frame_capture"
    duration: 20                # 必须持续20秒
    timeout: null

  - id: 4
    name: "冲洗"
    prev_node: "搓手"
    camera_id: "sink_cam_01"
    model: "qwen3-vl"
    instruct: "判断是否在冲洗"
    tools:
      - "frame_capture"
    duration: null
    timeout: 10

  - id: 5
    name: "烘干"
    prev_node: "冲洗"
    camera_id: "dryer_cam_01"
    model: "qwen3-vl"
    instruct: "判断是否在烘干"
    tools:
      - "frame_capture"
    duration: 5                 # 必须持续5秒
    timeout: null
```

### 0.2 完整执行流程图（SOP Navigator主导）

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                           洗手流程 SOP Agent 执行流程                                        │
└─────────────────────────────────────────────────────────────────────────────────────────────┘

输入: video_url="rtsp://camera/washroom", sop_config={...}

                                    ┌─────────────────────┐
                                    │  1. 初始化           │
                                    │  - 加载SOP配置       │
                                    │  - 初始化状态机      │
                                    │  - SOP Navigator就绪 │
                                    └──────────┬──────────┘
                                               │
                                               ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                           Agent循环 (SOP Navigator编排 + LLM决策 + Tool执行 + 状态更新)        │
├──────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐     │
│  │  SOP Navigator编排                                                                  │      │
│  │  - 事件触发: SOP自动监测视频流，检测是否有人开始洗手                                   │      │
│  │  - 获取当前环节: 根据当前状态确定需要检测的环节                                        │      │
│  │  - 构建上下文: 当前环节+窗口上下文+instruct                                           │      │
│  └─────────────────────────────────────────────────────────────────────────────────────┘     │
│          │                                                                                   │
│          ▼                                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐     │
│  │  Tool执行: 智能抽帧                                                               │      │
│  │  - 按语义条件获取关键帧序列                                                       │      │
│  │  - 返回: [frame1, frame2, frame3] + 时间戳                                        │      │
│  └─────────────────────────────────────────────────────────────────────────────────────┘      │
│          │                                                                                  │
│          ▼                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐      │
│  │  LLM决策  （这里要考虑两个事件的过渡片段）                                             │      │
│  │  输入:                                                                              │      │
│  │    - 图片序列 [frame1, frame2, frame3]                                              │      │
│  │    - SOP上下文                                                                      │      │
│  │    - instruct: "判断是否在挤洗手液"                                                  │      │
│  │  思考: 这些帧显示正在挤洗手液，动作持续时间足够                                        │      │
│  │  决策: 输出动作分析结果                                                              │      │
│  └─────────────────────────────────────────────────────────────────────────────────────┘      │
│          │                                                                                  │
│          ▼                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐      │
│  │  状态机更新 + 滑动窗口检测                                                        │      │
│  │  - 当前环节C=2, 检测到环节2动作                                                    │      │
│  │  - 检查窗口[C-back_window, C+forward_window]=[1,3]内的其他触发                                           │      │
│  │  - 环节2: running → success                                                       │      │
│  │  - 更新当前环节C=3                                                                │      │
│  │  - SSE输出: {"event":"node_end","node_id":2,"status":"success","actual_duration":2.3}             │      │
│  └─────────────────────────────────────────────────────────────────────────────────────┘      │
│          │                                                                                  │
│          ▼                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐      │
│  │  持续时长检测 (duration > 0的环节)                                                │      │
│  │  - 环节3"搓手"需要持续20秒                                                        │      │
│  │  - 每轮检测累计持续时间                                                           │      │
│  │  - 中断则重置计时器                                                               │      │
│  │  - 累计满20秒 → 标记"success"                                                     │      │
│  └─────────────────────────────────────────────────────────────────────────────────────┘      │
│                                                                                              │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
                                     ┌─────────────────────┐
                                     │  实例结束           │
                                     │  - 最后环节5完成    │
                                     │  - 或全局超时       │
                                     │  - 输出最终报告     │
                                     │  - 回到监控首节点   │
                                     │    (继续循环)       │
                                     └──────────┬──────────┘
                                                │
                                                ▼
                                     ┌─────────────────────┐
                                     │  持续循环监控       │
                                     │  等待下一个实例     │
                                     └─────────────────────┘
```

### 0.3 实例生命周期

```
持续循环: 监控首节点 → 创建Instance → Agent循环执行 → Instance结束 → 回到监控首节点
│                      │                            │
│                      │                            └──────────────┘
│                      │                                      │
│                      │    ┌─────────────────────────────┐
│                      └───→│  环节1 → 环节2 → ... → 环节N │←┘
└─────────────────────────────┘     (循环监控)
```

1. 实例创建时机:
   - 首节点(prev_node=null)作为隐式触发节点
   - 持续监控首节点，当检测到首节点动作时创建instance_id
   - 例如：持续监控"打湿手"动作，检测到时创建sop_inst_001

2. 实例结束时机:
   - 最后环节完成 → 回到监控首节点，继续等待下一个实例
   - 全局超时 → 回到监控首节点，继续等待下一个实例

3. 循环监督:
   - 一个实例结束后，自动回到初始状态
   - 继续监控首节点，等待下一个实例开始


### 0.4 异常场景示例

#### 场景A: 跳步检测

```
SOP配置: 环节3"搓手"(需持续20秒), forward_window=2
实际执行: 环节1 → 环节2 → 环节4（跳过环节3）

检测逻辑:
  - 当前环节C=3（pending）
  - LLM判断：当前动作是"冲洗"
  - 滑动窗口[C-back_window, C+forward_window]=[1,4]，检测到环节4动作
  - 环节3标记为"skip"，记录异常日志
  - 当前环节更新为C=5，继续监测
  - 全局状态标记为"异常"

SSE输出:
  {"event":"node_end","node_id":3,"status":"skip","detail":"在环节2未完成时检测到环节4"}
```

#### 场景B: 补做检测

```
SOP配置: back_window=2
实际执行: 环节1 → 环节2 → 环节3(未完成) → 环节4 → 环节3(补做)

检测逻辑:
  - 当前环节C=4
  - LLM判断：当前动作是"搓手"
  - 滑动窗口[C-back_window, C+forward_window]=[2,5]，检测到环节3动作
  - 环节3原状态为"skip"，更新为"redo"
  - 记录补做时间
  - 当前环节保持C=4，继续监测

SSE输出:
  {"event":"node_end","node_id":3,"status":"redo","detail":"环节3在环节4阶段被补做"}
```

#### 场景C: 超时检测

```
配置: 环节2"挤洗手液"的timeout=10秒

检测逻辑:
  - 环节1完成后，记录开始时间 T1
  - 等待检测环节2，但10秒内未检测到
  - 环节2标记为"timeout"，记录异常日志
  - 继续等待环节2或后续环节执行

SSE输出:
  {"event":"node_end","node_id":2,"status":"timeout","detail":"环节2等待超时(10s)"}
```

### 0.5 SSE事件输出序列

```
data: {"event":"agent_message","answer":"{\"event\":\"sop_start\",\"description\":\"七步洗手流程\"}"}
data: {"event":"agent_message","answer":"{\"event\":\"monitor_start\",\"description\":\"七步洗手流程\"}"}
data: {"event":"agent_message","answer":"{\"event\":\"instance_start\",\"instance_id\":\"sop_inst_001\"}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_start\",\"instance_id\":\"sop_inst_001\",\"node_id\":1,\"name\":\"打湿手\"}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_end\",\"instance_id\":\"sop_inst_001\",\"node_id\":1,\"name\":\"打湿手\",\"camera_id\":\"sink_cam_01\",\"camera_info\":\"sink_area\",\"status\":\"success\",\"actual_duration\":2.3}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_start\",\"instance_id\":\"sop_inst_001\",\"node_id\":2,\"name\":\"挤洗手液\"}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_end\",\"instance_id\":\"sop_inst_001\",\"node_id\":2,\"name\":\"挤洗手液\",\"camera_id\":\"soap_cam_01\",\"camera_info\":\"soap_area\",\"status\":\"success\",\"actual_duration\":3.1}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_start\",\"instance_id\":\"sop_inst_001\",\"node_id\":3,\"name\":\"搓手\"}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_end\",\"instance_id\":\"sop_inst_001\",\"node_id\":3,\"name\":\"搓手\",\"camera_id\":\"sink_cam_01\",\"camera_info\":\"sink_area\",\"status\":\"success\",\"actual_duration\":20}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_start\",\"instance_id\":\"sop_inst_001\",\"node_id\":4,\"name\":\"冲洗\"}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_end\",\"instance_id\":\"sop_inst_001\",\"node_id\":4,\"name\":\"冲洗\",\"camera_id\":\"sink_cam_01\",\"camera_info\":\"sink_area\",\"status\":\"success\",\"actual_duration\":8.5}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_start\",\"instance_id\":\"sop_inst_001\",\"node_id\":5,\"name\":\"烘干\"}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_end\",\"instance_id\":\"sop_inst_001\",\"node_id\":5,\"name\":\"烘干\",\"camera_id\":\"dryer_cam_01\",\"camera_info\":\"dryer_area\",\"status\":\"success\",\"actual_duration\":5}"}
data: {"event":"agent_message","answer":"{\"event\":\"instance_end\",\"instance_id\":\"sop_inst_001\",\"status\":\"success\"}"}
data: {"event":"agent_message","answer":"{\"event\":\"sop_end\",\"status\":\"success\"}"}
data: {"event":"message_end",...}
```

> 说明：`sop_start`/`monitor_start`/`sop_end` 根级不带 `instance_id`；`node_end` 带 `camera_id`/`camera_info`；完整 MQ 字段示例见同目录 `sop_judge_data_demos.txt`。

### 0.6 最终报告输出

```json
{
  "instance_id": "sop_inst_001",
  "description": "七步洗手流程",
  "status": "success",
  "start_time": "2026-01-11T10:05:20",
  "end_time": "2026-01-11T10:05:58.9",
  "total_duration": 38.9,
  "timeline": [
    {
      "node_id": 1,
      "name": "打湿手",
      "status": "success",
      "duration": null,
      "timeout": null,
      "actual_duration": 2.3,
      "actual_timeout": null,
      "start_time": "2026-01-11T10:05:20",
      "end_time": "2026-01-11T10:05:22.3",
      "detail": ""
    },
    {
      "node_id": 2,
      "name": "挤洗手液",
      "status": "success",
      "duration": null,
      "timeout": 10,
      "actual_duration": 3.1,
      "actual_timeout": null,
      "start_time": "2026-01-11T10:05:22.3",
      "end_time": "2026-01-11T10:05:25.4",
      "detail": ""
    },
    {
      "node_id": 3,
      "name": "搓手",
      "status": "success",
      "duration": 20,
      "timeout": null,
      "actual_duration": 20.0,
      "actual_timeout": null,
      "start_time": "2026-01-11T10:05:25.4",
      "end_time": "2026-01-11T10:05:45.4",
      "detail": "持续20秒检测通过"
    },
    {
      "node_id": 4,
      "name": "冲洗",
      "status": "success",
      "duration": null,
      "timeout": 10,
      "actual_duration": 8.5,
      "actual_timeout": null,
      "start_time": "2026-01-11T10:05:45.4",
      "end_time": "2026-01-11T10:05:53.9",
      "detail": ""
    },
    {
      "node_id": 5,
      "name": "烘干",
      "status": "success",
      "duration": 5,
      "timeout": null,
      "actual_duration": 5.0,
      "actual_timeout": null,
      "start_time": "2026-01-11T10:05:53.9",
      "end_time": "2026-01-11T10:05:58.9",
      "detail": ""
    }
  ],
  "anomalies": []
}
```
---

## 一、需求分析

### 1.1 业务场景

SOP（Standard Operating Procedure，标准操作流程）用于对视频流中的多目标进行实时行为监测。典型场景包括：

- 洗手流程监测
- 作业流程合规监测

### 1.2 核心需求

| 需求 | 说明 |
|------|------|
| **Agent驱动** | SOP Navigator 主导整个流程，LLM负责判断动作类型 |
| **节点配置** | 用户通过YAML配置SOP，支持串行/并行节点 |
| **时序约束** | 支持持续时长(duration)、超时(timeout)配置 |
| **滑动窗口** | 支持配置回看窗口(back_window)、前向窗口(forward_window)，检测跳步/补做 |
| **乱序执行不中断** | 允许在规定窗口[back_window, forward_window]内乱序，超出窗口判定违规 |
| **逻辑违规记录** | 实时记录跳步、补做、超时等违规状态 |
| **流程循环** | 实例结束后自动回到初始状态，继续监控首节点，等待下一个实例 |
| **流式输出** | 通过SSE实时输出各环节状态 |
| **事件触发** | 首节点(prev_node=null)作为隐式触发节点，持续监控首节点动作，检测到时创建Instance |
| **图片序列** | 每次处理的是连续帧序列，而非单帧 |
| **多实例** | 支持多实例并行执行，每个实例独立运行, 但该版本暂只支持单实例情况 |

---

## 二、架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                           SOP Agent (LLM驱动的Agent)                                         │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│    ┌──────────────┐      ┌──────────────┐      ┌──────────────┐      ┌──────────────┐     │
│    │SOP Navigator│ ──→ │   LLM决策    │ ──→ │  Tool执行    │ ──→ │  状态机更新  │     │
│    │  (编排)      │      │              │      │              │      │              │     │
│    │- 事件触发   │      │- 判断动作类型│      │- 智能抽帧   │      │- 状态转移   │     │
│    │- 获取当前环节│      │              │      │              │      │- 跳步/补做  │     │
│    │- 构建上下文  │      │              │      │              │      │- 时序约束   │     │
│    └──────────────┘      └──────────────┘      └──────────────┘      └──────────────┘     │
│                                                                              ↓             │
│                                                                   ┌──────────────┐          │
│                                                                   │  SSE输出    │          │
│                                                                   │  - 事件流   │          │
│                                                                   │  - 实时状态 │          │
│                                                                   └──────────────┘          │
│                                 Agent循环 (直到流程结束)                                     │
│                                                                                             │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                           MCP Tools (工具层)                                         │  │
│  │   ┌─────────────┐                                                                   │  │
│  │   │frame_capture│   # 智能抽帧工具                                                  │  │
│  │   │  智能抽帧   │                                                                   │  │
│  │   └─────────────┘                                                                   │  │
│  │   注意：动作类型判断由大模型(LLM)根据图片序列完成                                     │  │
│  └───────────────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 模块职责

| 模块 | 职责 |
|------|------|
| **SOP Navigator** | 编排执行流程，确定当前环节，构建上下文（省去一次LLM调用） |
| **LLM决策器** | 根据图片序列判断动作类型 |
| **Tool执行器** | 调用MCP工具（抽帧） |
| **状态机** | 管理环节状态、处理跳步/补做/超时逻辑 |
| **滑动窗口管理器** | 维护窗口上下文 [back_window, forward_window] |
| **SSE输出器** | 实时推送事件流 |
| **Instance管理器** | 管理多实例（创建、销毁），当前版本一次只运行一个实例 |

---

## 三、SOP配置文件格式（YAML）

### 3.1 完整配置示例

```yaml
# hand_wash.yaml（多摄像头）
description: "七步洗手流程，用于监测洗手操作是否符合标准流程"
name: "七步洗手流程"
version: "1.1"

# 全局参数
timeout: 300              # instance整体超时（秒）
window:
  back_window: 2          # 回看窗口：检测前几个环节是否有补做
  forward_window: 2     # 向前窗口：检测后几个环节是否有跳步

# 逻辑摄像头定义（SOP层不写真实URL）
cameras:
  - id: "sink_cam_01"
    role: "sink_area"
  - id: "soap_cam_01"
    role: "soap_dispenser_area"
  - id: "dryer_cam_01"
    role: "hand_dryer_area"

# 节点定义
nodes:
  - id: 1
    name: "打湿手"
    prev_node: null              # 上一个节点名字，null表示起始节点
    camera_id: "sink_cam_01"
    model: "qwen3-vl"      # 模型
    instruct: "判断是否在打湿手" # 描述这个节点要干啥
    tools:                      # 工具列表（目前只有抽帧工具）
      - "frame_capture"         # 抽帧工具
    duration: null              # 持续时长（秒），null表示不限制
    timeout: null              # 超时（秒），null表示不限制

  - id: 2
    name: "挤洗手液"
    prev_node: "打湿手"
    camera_id: "soap_cam_01"
    model: "qwen3-vl"
    instruct: "判断是否在挤洗手液"
    tools:
      - "frame_capture"
    duration: null
    timeout: 10

  - id: 3
    name: "搓手"
    prev_node: "挤洗手液"
    camera_id: "sink_cam_01"
    model: "qwen3-vl"
    instruct: "判断是否有搓手动作"
    tools:
      - "frame_capture"
    duration: 20
    timeout: null

  - id: 4
    name: "冲洗"
    prev_node: "搓手"
    camera_id: "sink_cam_01"
    model: "qwen3-vl"
    instruct: "判断是否在冲洗"
    tools:
      - "frame_capture"
    duration: null
    timeout: 10

  - id: 5
    name: "烘干"
    prev_node: "冲洗"
    camera_id: "dryer_cam_01"
    model: "qwen3-vl"
    instruct: "判断是否在烘干"
    tools:
      - "frame_capture"
    duration: 5
    timeout: null
```

### 3.1.1 并行节点示例

```yaml
# 并行节点配置
cameras:
  - id: "cam_a_01"
    role: "area_a"
  - id: "cam_b_01"
    role: "area_b"

nodes:
  - id: 1
    name: "动作A"
    prev_node: null
    camera_id: "cam_a_01"
    ...

  - id: 2
    name: "动作B"
    prev_node: null          # 与节点1并行
    camera_id: "cam_b_01"
    ...

  - id: 3
    name: "动作C"
    prev_node: "动作A"       # 串行依赖
    camera_id: "cam_a_01"
    ...

  - id: 4
    name: "动作D"
    prev_node: "动作A"       # 与节点3并行，都依赖节点1
    camera_id: "cam_b_01"
    ...
```

### 3.2 配置Schema定义

```yaml
# sop_schema.yaml
description: string              # SOP描述，整体是干啥的
name: string                    # SOP名称
version: string                 # 版本号

timeout: integer            # instance整体超时（秒），默认300

window:
  back_window: integer      # 回看窗口，默认2
  forward_window: integer  # 向前窗口，默认2

cameras:
  - id: string              # 逻辑摄像头唯一ID（稳定主键）
    role: string            # 业务语义（如 sink_area / dryer_area）

nodes:
  - id: integer            # 节点ID（从1开始）
    name: string           # 节点名称
    prev_node: string      # 上一个节点名字，null表示起始节点
    camera_id: string      # 节点绑定的逻辑摄像头ID（引用cameras[].id）
    model: string          # 算法模型名称（用于大模型判断动作类型）
    instruct: string       # 节点描述
    tools: [string]        # 工具列表（目前只有抽帧工具：frame_capture）
    duration: integer      # 持续时长（秒），null表示不限制
    timeout: integer       # 超时（秒），null表示不限制
```

运行时摄像头映射（部署层，不在SOP文件内）：

```yaml
# camera_runtime_map.yaml
sink_cam_01: "rtsp://10.0.0.11/live"
soap_cam_01: "rtsp://10.0.0.12/live"
dryer_cam_01: "rtsp://10.0.0.13/live"
```

---

## 四、接口设计

### 4.1 输入接口（ChatMessages）

```json
{
  "query": '{"task_id":"xxxx", "sop": "$sop yaml str"}',
  "response_mode": "streaming",
  "messages": [],
  "files": [
    {
      "type": "video",
      "transfer_method": "remote_url",
      "url": "rtsp://10.0.0.11/live",
      "upload_file_id": "string"
    },
    {
      "type": "video",
      "transfer_method": "remote_url",
      "url": "rtsp://10.0.0.12/live",
      "upload_file_id": "string"
    },
    {
      "type": "video",
      "transfer_method": "remote_url",
      "url": "rtsp://10.0.0.13/live",
      "upload_file_id": "string"
    }
  ]
}
```

说明：

- 多摄像头绑定约定：`files` 顺序与 `sop.cameras` 顺序一一对应（`files[i] -> cameras[i].id`）。
- 节点通过 `nodes[].camera_id` 引用 `cameras[].id` 完成路由。

### 4.2 输出接口（SSE事件流）

基于现有ToolCallAgent协议，SOP输出复用了`agent_message`事件类型，所有内容以JSON字符串形式存放在`answer`字段中。

#### 4.2.1 事件序列

约束：
- 每条 `agent_message` 的 `answer` 都必须透传 `task_id`。
- 每个步骤事件（`node_start` / `node_end`）必须包含 `timestamp`（Unix 毫秒时间戳数字）。

| 序号 | 事件 | SSE类型 | answer内容（JSON字符串） |
|------|------|---------|------------------------|
| 1 | SOP开始执行 | `agent_message` | `'{"event":"sop_start","task_id":"xxxx","description":"七步洗手流程","timestamp":1778100320000}'` |
| 2 | 开始监控 | `agent_message` | `'{"event":"monitor_start","task_id":"xxxx","description":"七步洗手流程","timestamp":1778100321000}'` |
| 3 | 实例开始 | `agent_message` | `'{"event":"instance_start","task_id":"xxxx","instance_id":"sop_inst_001","timestamp":1778100322000}'` |
| 4 | 节点开始 | `agent_message` | `'{"event":"node_start","task_id":"xxxx","instance_id":"sop_inst_001","node_id":1,"name":"打湿手","timestamp":1778100323000}'` |
| 5 | 节点结束 | `agent_message` | `'{"event":"node_end","task_id":"xxxx","instance_id":"sop_inst_001","node_id":1,"name":"打湿手","status":"success","actual_duration":2.3,"timestamp":1778100326000}'` |
| 6 | 下一节点开始 | `agent_message` | `'{"event":"node_start","task_id":"xxxx","instance_id":"sop_inst_001","node_id":2,"name":"挤洗手液","timestamp":1778100327000}'` |
| 7 | 节点结束 | `agent_message` | `'{"event":"node_end","task_id":"xxxx","instance_id":"sop_inst_001","node_id":2,"name":"挤洗手液","status":"success","actual_duration":3.1,"timestamp":1778100330000}'` |
| ... | ... | ... | ... |
| N | 实例结束执行结果 | `agent_message` | `'{"event":"instance_end","task_id":"xxxx","instance_id":"sop_inst_001","status":"success","timeline":[...],"anomalies":[],"timestamp":1778100362000}'` |
| N+1 | SOP执行结束 | `message_end` | `{metadata: {...}}` |

#### 4.2.2 事件类型说明

公共字段约束：
- `task_id`：所有 `agent_message` 必填，值等于输入中的 `task_id`。
- `timestamp`：建议所有事件都带；`node_start` / `node_end` 必填。

| event字段 | 说明 | 
|-----------|------|
| `instance_start` | 实例开始（创建新实例） | 
| `instance_end` | 实例结束 | 
| `sop_start` | SOP开始执行（第一条消息，整个SOP大维度） | 
| `monitor_start` | 开始监控（等待首节点触发） |
| `node_start` | 节点开始 | 
| `node_end` | 节点结束（包含status：success/skip/redo/timeout/failed） | 
| `sop_end` | SOP执行结果（仅触发异常时，如断流、中断） | 

#### 4.2.3 示例输出

```
data: {"event":"agent_message","answer":"{\"event\":\"sop_start\",\"task_id\":\"xxxx\",\"instance_id\":\"sop_inst_001\",\"description\":\"七步洗手流程\",\"timestamp\":1778100320000}"}
data: {"event":"agent_message","answer":"{\"event\":\"monitor_start\",\"task_id\":\"xxxx\",\"description\":\"七步洗手流程\",\"timestamp\":1778100321000}"}
data: {"event":"agent_message","answer":"{\"event\":\"instance_start\",\"task_id\":\"xxxx\",\"instance_id\":\"sop_inst_001\",\"timestamp\":1778100322000}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_start\",\"task_id\":\"xxxx\",\"instance_id\":\"sop_inst_001\",\"node_id\":1,\"name\":\"打湿手\",\"timestamp\":1778100323000}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_end\",\"task_id\":\"xxxx\",\"instance_id\":\"sop_inst_001\",\"node_id\":1,\"name\":\"打湿手\",\"status\":\"success\",\"actual_duration\":2.3,\"timestamp\":1778100326000}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_start\",\"task_id\":\"xxxx\",\"instance_id\":\"sop_inst_001\",\"node_id\":2,\"name\":\"挤洗手液\",\"timestamp\":1778100327000}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_end\",\"task_id\":\"xxxx\",\"instance_id\":\"sop_inst_001\",\"node_id\":2,\"name\":\"挤洗手液\",\"status\":\"success\",\"actual_duration\":3.1,\"timestamp\":1778100330000}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_start\",\"task_id\":\"xxxx\",\"instance_id\":\"sop_inst_001\",\"node_id\":3,\"name\":\"搓手\",\"timestamp\":1778100331000}"}
data: {"event":"agent_message","answer":"{\"event\":\"node_end\",\"task_id\":\"xxxx\",\"instance_id\":\"sop_inst_001\",\"node_id\":3,\"name\":\"搓手\",\"status\":\"success\",\"actual_duration\":20,\"timestamp\":1778100351000}"}
...
data: {"event":"agent_message","answer":"{\"event\":\"instance_end\",\"task_id\":\"xxxx\",\"instance_id\":\"sop_inst_001\",\"status\":\"success\",\"timestamp\":1778100361000}"}
data: {"event":"agent_message","answer":{"event":"instance_end","task_id":"xxxx","instance_id":"sop_inst_001","status":"success","timeline":[...],"anomalies":[],"timestamp":1778100362000}'"}
data: {"event":"message_end",...}
```

客户端收到后统一`JSON.parse(answer)`解析。

### 4.3 最终报告输出

<details>
<summary>跳步</summary>

```json
{
  "instance_id": "sop_inst_001",
  "description": "七步洗手流程",
  "status": "failed", # success, failed, timeout
  "start_time": "2026-01-11T10:05:20",
  "end_time": "2026-01-11T10:10:30",
  "timeline": [
    {
      "node_id": 1,
      "name": "打湿手",
      "status": "success",
      "duration": null,
      "timeout": null,
      "actual_duration": 5,
      "actual_timeout": null,
      "start_time": "2026-01-11T10:05:20",
      "end_time": "2026-01-11T10:05:25",
      "detail": ""
    },
    {
      "node_id": 2,
      "name": "挤洗手液",
      "status": "success",
      "duration": null,
      "timeout": 10,
      "actual_duration": 45,
      "actual_timeout": null,
      "start_time": "2026-01-11T10:05:25",
      "end_time": "2026-01-11T10:06:10",
      "detail": ""
    },
    {
      "node_id": 3,
      "name": "搓手",
      "status": "skip",
      "duration": 20,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": null,
      "end_time": null,
      "detail": "在环节3未完成时检测到环节4"
    },
    {
      "node_id": 4,
      "name": "冲洗",
      "status": "success",
      "duration": 20,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": null,
      "end_time": null,
      "detail": ""
    },
    {
      "node_id": 5,
      "name": "烘干",
      "status": "success",
      "duration": 20,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": null,
      "end_time": null,
      "detail": ""
    }
  ],
  "anomalies": [
    {
      "type": "skip",
      "node_id": 3,
      "timestamp": 1778100375000,
      "detail": "在环节3未完成时检测到环节4"
    }
  ]
}

```
</details>

<details>
<summary>补做</summary>

```json
{
  "instance_id": "sop_inst_001",
  "description": "七步洗手流程",
  "status": "failed", # success, failed, timeout
  "start_time": "2026-01-11T10:05:20",
  "end_time": "2026-01-11T10:10:30",
  "timeline": [
    {
      "node_id": 1,
      "name": "打湿手",
      "status": "success",
      "duration": null,
      "timeout": null,
      "actual_duration": 5,
      "actual_timeout": null,
      "start_time": "2026-01-11T10:05:20",
      "end_time": "2026-01-11T10:05:25",
      "detail": ""
    },
    {
      "node_id": 2,
      "name": "挤洗手液",
      "status": "success",
      "duration": null,
      "timeout": 10,
      "actual_duration": 45,
      "actual_timeout": null,
      "start_time": "2026-01-11T10:05:25",
      "end_time": "2026-01-11T10:06:10",
      "detail": ""
    },
    {
      "node_id": 3,
      "name": "搓手",
      "status": "skip",
      "duration": 20,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": null,
      "end_time": null,
      "detail": "在环节3未完成时检测到环节4"
    },
    {
      "node_id": 4,
      "name": "冲洗",
      "status": "success",
      "duration": 20,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": null,
      "end_time": null,
      "detail": ""
    },
    {
      "node_id": 3,
      "name": "搓手",
      "status": "redo",
      "duration": 20,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": null,
      "end_time": null,
      "detail": "在环节4未完成后补做环节3"
    },
    {
      "node_id": 4,
      "name": "冲洗",
      "status": "success",
      "duration": 20,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": null,
      "end_time": null,
      "detail": ""
    },
    {
      "node_id": 5,
      "name": "烘干",
      "status": "success",
      "duration": 20,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": null,
      "end_time": null,
      "detail": ""
    }
  ],
  "anomalies": [
    {
      "type": "redo",
      "node_id": 3,
      "timestamp": 1778100375000,
      "detail": "在环节3未完成时检测到环节4"
    }
  ]
}

```
</details>

<details>
<summary>正常</summary>

```json
{
  "instance_id": "sop_inst_001",
  "description": "七步洗手流程",
  "status": "success", # success, failed, timeout
  "start_time": "2026-01-11T10:05:20",
  "end_time": "2026-01-11T10:10:30",
  "timeline": [
    {
      "node_id": 1,
      "name": "打湿手",
      "status": "success",
      "duration": null,
      "timeout": null,
      "actual_duration": 5,
      "actual_timeout": null,
      "start_time": "2026-01-11T10:05:20",
      "end_time": "2026-01-11T10:05:25",
      "detail": ""
    },
    {
      "node_id": 2,
      "name": "挤洗手液",
      "status": "success",
      "duration": null,
      "timeout": 10,
      "actual_duration": 45,
      "actual_timeout": null,
      "start_time": "2026-01-11T10:05:25",
      "end_time": "2026-01-11T10:06:10",
      "detail": ""
    },
    {
      "node_id": 3,
      "name": "搓手",
      "status": "success",
      "duration": 20,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": null,
      "end_time": null,
      "detail": ""
    },
    {
      "node_id": 4,
      "name": "冲洗",
      "status": "success",
      "duration": 20,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": null,
      "end_time": null,
      "detail": ""
    },
    {
      "node_id": 5,
      "name": "烘干",
      "status": "success",
      "duration": 20,
      "timeout": null,
      "actual_duration": null,
      "actual_timeout": null,
      "start_time": null,
      "end_time": null,
      "detail": ""
    }
  ],
  "anomalies": [
  ]
}

```
</details>

---

## 五、数据模型

### 5.1 核心类定义

```python
# models.py
from enum import Enum
from datetime import datetime
from typing import Optional, List, Dict, Any

class StepStatus(str, Enum):
    PENDING = "pending"
    SUCCESS = "success"
    SKIP = "skip"
    REDO = "redo"
    TIMEOUT = "timeout"
    FAILED = "failed"

class FlowStatus(str, Enum):
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    TIMEOUT = "timeout"

class SOPConfig:
    description: str              # SOP描述，整体是干啥的
    name: str                    # SOP名称
    version: str
    timeout: int                    # instance整体超时
    back_window: int              # 回看窗口
    forward_window: int          # 向前窗口
    nodes: List['SOPNode']

class SOPNode:
    id: int
    name: str
    prev_node: str                   # 上一个节点名字，null表示起始节点
    model: str                      # 模型名称（用于大模型判断动作类型）
    instruct: str                   # 节点描述
    tools: List[str]                # 工具列表（目前只有抽帧工具：frame_capture）
    duration: Optional[int]          # 持续时长（秒）
    timeout: Optional[int]          # 超时（秒）

class SOPInstance:
    instance_id: str                # 自动生成，事件触发时创建
    description: str                # SOP描述
    name: str                      # SOP名称
    video_url: str
    status: FlowStatus
    current_step: int              # 当前环节
    start_time: datetime
    end_time: Optional[datetime]
    step_status: Dict[int, StepStatus]  # node_id -> status
    step_records: Dict[int, 'StepRecord']
    anomaly_records: List['AnomalyRecord']
    duration_timers: Dict[int, float]   # node_id -> 持续时长累计
    timeout_timers: Dict[int, float]    # node_id -> 超时开始时间

class StepRecord:
    node_id: int
    name: str
    status: StepStatus
    duration: Optional[int]           # 配置的duration
    timeout: Optional[int]            # 配置的timeout
    actual_duration: Optional[float] # 实际持续时长
    actual_timeout: Optional[float]  # 实际超时时间
    start_time: Optional[datetime]
    end_time: Optional[datetime]
    detail: str
    evidence: List['FrameEvidence']

class FrameEvidence:
    frame_time: datetime
    frame_index: int
    confidence: float
    detection_results: Dict[str, Any]

class AnomalyRecord:
    anomaly_type: str  # skip, redo, timeout
    node_id: int
    timestamp: datetime
    detail: str
```

---

## 六、执行流程

### 6.1 Agent循环流程

```
1. 输入ChatMessages
   |
   ├─── 解析sop_config
   │    ├── sop_yaml → SOPParser → SOPConfig
   │    └── video_url
   │
   ▼
2. 初始化状态机
   ├── 加载SOP配置
   ├── 初始化窗口 [back_window, forward_window]
   └── 初始化SOP Navigator
   │
   ▼
3. Agent循环 (SOP Navigator编排 → LLM决策 → Tool执行 → 状态更新)
   │
   ├── 3.1 SOP Navigator编排（省去一次LLM调用）
   │    ├── 事件触发：自动监测视频流，检测是否有人开始执行SOP（通过轮询方式持续检测首节点）
   │    ├── 获取当前环节：根据当前状态确定需要检测的环节
   │    ├── 构建上下文：当前环节 + 窗口上下文 + instruct
   │    └── 决策：调用抽帧工具
   │
   ├── 3.2 Tool执行
   │    ├── 智能抽帧：按语义条件获取关键帧序列
   │    └── 返回：{frames: [frame1, frame2, frame3], timestamps: [...]}
   │
   ├── 3.3 LLM决策
   │    ├── 输入：图片序列 + 上下文（SOP描述+环节定义+instruct）
   │    ├── 判断动作类型
   │    └── 输出：{action, target_id, confidence}
   │
   ├── 3.4 状态机更新
   │    ├── 检测到当前环节C执行 → success, C=C+1
   │    ├── 检测到C+k执行(k≤M) → skip, C=C+k+1
   │    ├── 检测到C-k执行(k≤N)且状态为skip → redo
   │    ├── timeout超时 → timeout
   │    └── duration持续时长满足 → success
   │
   ├── 3.5 滑动窗口处理
   │    ├── 窗口范围：[back_window, forward_window]
   │    ├── 并行检测窗口内所有可能触发的事件
   │    └── 一次性更新窗口内所有环节状态
   │
   ├── 3.5 SSE输出事件
   │    ├── agent_message (node_start/node_end)
   │    └── message_end (最终报告后)
   │
   └── 3.6 实例结束处理
        ├── 最后步骤完成 → 输出报告 → 回到监控首节点
        ├── 全局超时 → 输出报告 → 回到监控首节点
        └── 继续循环等待下一个实例
   │
   ▼
4. 输出最终报告 + 回到监控状态
```

### 6.2 状态机流转

```
┌──────────────────────────────────────────────────────────────────┐
│                           状态机                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐    检测到C执行     ┌──────────┐                   │
│  │  步骤C    │ ───────────────→│ 步骤C    │                   │
│  │  pending │                   │ success │                   │
│  └──────────┘                   └──────────┘                   │
│                                      ↓                           │
│  ┌──────────┐    检测到C+k执行   ┌──────────┐                   │
│  │  步骤C    │ ───────────────→│ 步骤C+k  │                   │
│  │  pending │  (k≤M, 跳步违规) │ success │                   │
│  └──────────┘                   └──────────┘                   │
│                                      ↓                           │
│  ┌──────────┐    检测到C-k执行  ┌──────────┐                   │
│  │  步骤C-k  │ 且状态为"skip"    │ 步骤C-k  │                   │
│  │  skip    │ ───────────────→│ redo     │                   │
│  └──────────┘                   └──────────┘                   │
│                                      ↓                           │
│  ┌──────────┐    等待超时       ┌──────────┐                   │
│  │  步骤C    │ ───────────────→│ 步骤C    │                   │
│  │  pending │  (配置timeout)   │ timeout │                   │
│  └──────────┘                   └──────────┘                   │
│                                      ↓                           │
│  ┌──────────┐    最后步骤完成   ┌──────────┐                   │
│  │  流程    │ ───────────────→│ 流程     │                   │
│  │  running │                   │ done     │                   │
│  └──────────┘                   └──────────┘                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 七、实现计划

### 7.1 目录结构

```
magent/agent_family/sop_agent/
├── config/
│   ├── __init__.py
│   ├── schema.yaml              # 配置Schema定义
│   ├── hand_wash.yaml          # 洗手流程示例
│   └── sample_parallel.yaml    # 并行节点示例
├── __init__.py
├── models.py                    # 数据模型
├── sop_parser.py                # YAML解析器
├── sop_navigator.py             # 核心：编排 + 窗口管理 + 状态机（合并）
├── serving_sop_agent.py         # 主入口（继承ToolCallAgent）
├── prompts.py                   # 提示词模板
└── utils.py                     # 工具函数
```

### 7.2 实现顺序

| 阶段 | 模块 | 任务 |
|------|------|------|
| **Phase 1** | models.py | 定义所有数据模型、枚举类 |
| **Phase 2** | sop_parser.py | YAML解析、配置校验 |
| **Phase 3** | sop_navigator.py | 核心逻辑：事件触发、窗口管理、状态机、状态转移、上下文构建 |
| **Phase 4** | serving_sop_agent.py | 继承ToolCallAgent，整合LLM决策循环、SSE输出 |
| **Phase 5** | config/*.yaml | 示例配置文件 |

### 7.3 关键技术点

1. **SOP Navigator主导**: 编排执行流程，确定当前环节，构建上下文
2. **窗口管理**: 计算窗口范围，构建窗口上下文作为LLM输入
3. **状态机**: 管理环节状态、处理跳步/补做/超时逻辑
4. **事件触发**: 持续监控首节点，检测到时创建Instance
5. **图片序列**: 每次处理的是连续帧序列，通过智能抽帧获取
6. **SSE流式输出**: 通过queue.put()实时推送事件

---

## 八、LLM决策提示词设计

### 8.1 系统提示词

```
你是一个SOP执行助手，负责根据当前视频帧序列和SOP配置，判断当前动作属于哪个环节。

## 你的职责
1. 根据抽帧获取的图片序列，判断当前动作类型
2. 结合SOP配置和当前状态，做出决策
3. 调用相应的Tool执行具体操作

## SOP配置
- 描述: {description}  # SOP整体描述
- 环节定义: {nodes}
- 当前环节: {current_step}
- 窗口范围: [{back_window}, {forward_window}]

## 当前状态
- 实例状态: {instance_status}
- 环节状态: {step_status}
- 异常记录: {anomaly_history}

## 决策规则
- 如果当前动作与当前环节匹配，输出当前环节完成
- 如果检测到跳步（跳过当前环节），标记跳步并继续
- 如果检测到补做（回头做之前的环节），记录补做
- 如果持续时长不足，继续等待
```

### 8.2 工具定义

```python
# Tool: get_frame_sequence
# 按语义条件获取关键帧序列
{
    "name": "get_frame_sequence",
    "description": "按语义条件从视频流中获取关键帧序列",
    "parameters": {
        "video_url": "视频URL",
        "condition": "语义条件，如'湿手动作'、'搓手动作'",
        "frame_count": 3  # 获取帧数
    }
}

# Tool: analyze_action
# 大模型分析动作类型
{
    "name": "analyze_action",
    "description": "分析图片序列，判断动作类型",
    "parameters": {
        "frames": ["frame1_url", "frame2_url", "frame3_url"],
        "instruct": "判断是否有搓手动作",
        "sop_config": {...}
    }
}
```

---

## 九、待确认问题

### 接口相关的改动


- [x] IR和前端接口，看起来不兼容，需再拉齐
- [x] 需要增加ROI参数输入（可暂不支持）
- [x] yaml 业务存数据库是文本，可放到query 里
- [x] step_id 改成 node_id
- [ ] 为方便前端展示，node finish 时，需要把抽帧的所有url返回
- [x] 增加业务 task_id 传入，输出也要透传出去
- [x] step 里有多个 node （期望给到yaml的是摊平后的）
- [x] 抽帧需要cache（由抽帧工具来cache）
- [x] 最后输出报告的status判断 （redo/skip 算不算faild？算）
- [ ] 智能体异常处理， mcp-client 连 mcp-server 若断了，sop 当前任务销毁重置\
- [x] 每个步骤的数据结构增加时间戳信息
- [x] 环节状态增加“未执行”(sop 没有未执行的状态)
- [x] 考虑多路流的情况。
- [ ] 首节点(prev_node=null)作为隐式触发节点，改为首窗口作为隐式触发节点。（加个开关，放到yaml）
- [ ] 全局status失败，成功，失败，异常.
- [x] 增加sop_end，表示视频分析结束。
- [ ] 跳步是被跳过的节点。
- [ ] 补做时，执行节点不回滚。
- [ ] SOP暂不考虑并行。
- [ ] task_id 改成 meta_data, json串传入。

0326
- [ ] 节点返回camera_id
- [ ] 没有sop_end , 停止由客户端抉择

0327
- [ ] sop报错信息的event类型值确定一下，用sop_err,透传过来错误的信息


可PR的
* 多路流协作，跨事件对齐
* 跨时间对齐
* Navigator 结构


新增待讨论
- [ ] 需要透传一些业务信息：包括哪一个任务、哪一个设备、哪一个技能？


