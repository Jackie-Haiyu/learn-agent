# SOP 合规告警逻辑规则说明（V2）

> **负责人**：张海玉  
> **日期**：2026-04-14  
> **版本**：v2.0

---

## 1. 逻辑目标

本文用于约束 SOP 合规告警实现层逻辑，确保：

1. 归并口径唯一（不在多模块重复拼接）。
2. 事件乱序、重复、缺失可控处理。
3. 告警优先级和去重行为一致。
4. 代码逻辑简单，便于长期维护。

---

## 2. 事件处理优先级

同一 `instanceId` 下按如下优先级处理：

1. `instance_end`
2. `node_end`
3. `node_start`
4. `instance_start`
5. `sop_start` / `monitor_start` / `sop_end`

说明：

- 若事件乱序到达，以 `event_time` 回放排序，优先级仅用于同时间冲突决策。
- 同一幂等键重复事件直接丢弃。

---

## 3. 实例归并规则

### 3.1 节点状态来源

节点状态不做本地推演，直接使用事件中的 `node_end.status` 值。  
本服务只负责按 `instance_id + node_id` 归并最新节点快照。

### 3.2 实例状态规则

实例状态不做本地判定，直接使用 `instance_end.status`。  
若未收到 `instance_end`，实例保持未完成态，不自动写入 `error`。

---

## 4. 告警触发逻辑

### 4.1 节点告警触发

| nodeStatus | ruleCode | severity | 是否立即触发 |
| --- | --- | --- | --- |
| `failed` | `NODE_FAILED` | `P1` | 是 |
| `timeout` | `NODE_TIMEOUT` | `P1` | 是 |
| `skip` | `NODE_SKIP` | `P2` | 是 |
| `redo` | `NODE_REDO` | `P3` | 是 |
| `success` | 无 | 无 | 否 |

### 4.2 实例告警触发

| instanceStatus | ruleCode | severity |
| --- | --- | --- |
| `failed` | `INSTANCE_FAILED` | `P1` |
| `success` | 无 | 无 |

### 4.3 去重键与窗口

- 节点告警去重键：`instanceId + nodeId + ruleCode`
- 实例告警去重键：`instanceId + ruleCode`
- 默认去重窗口：`300s`

---

## 5. 乱序与缺失处理

### 5.1 乱序处理

1. 事件先写 `sop_event`。
2. 归并器按 `event_time` 增量归并快照。
3. 若旧事件到达且影响终态，触发快照重算并产出修正告警。

### 5.2 缺失 `instance_end`

1. `instance_start` 后进入进行中实例集合。
2. 若长期未收到 `instance_end`，仅记录“未完成实例”监控指标，不改写业务状态。

### 5.3 数据修复策略

- 支持按 `instanceId` 重放 `sop_event` 事件重建快照。
- 修复过程需幂等，不重复制造告警。

---

## 6. 推送过滤逻辑

### 6.1 过滤顺序

1. `tenantId`
2. `planId`
3. `skillId`
4. `sourceId`
5. `eventType`
6. `instanceStatus`
7. `nodeStatus`
8. `severity`

说明：任一条件不匹配即短路拒绝，减少推送开销。

### 6.2 推送内容

- `sop_event`：实例与节点摘要（用于实时列表刷新）
- `sop_alert`：告警摘要（用于告警列表插入）
- 详情统一走查询接口，不在推送包承载大 payload

---

## 7. 参考伪代码

```java
public Optional<AlertFact> decideNodeAlert(NodeSnapshot node) {
    switch (node.getNodeStatus()) {
        case "failed":
            return buildAlert("NODE_FAILED", "P1", node);
        case "timeout":
            return buildAlert("NODE_TIMEOUT", "P1", node);
        case "skip":
            return buildAlert("NODE_SKIP", "P2", node);
        case "redo":
            return buildAlert("NODE_REDO", "P3", node);
        default:
            return Optional.empty();
    }
}
```

---

## 8. 测试要点

1. 正常流程：全 `success`，实例终态为 `success`，无告警。
2. 跳步补做：先 `skip` 后 `redo`，告警两条且去重生效。
3. 节点超时：触发 `NODE_TIMEOUT`。
4. 缺失 `instance_end`：实例保持未完成态，并产生监控告警（非业务告警）。
5. 重复事件：不重复写快照，不重复推送告警。
6. 乱序事件：重放后快照与预期一致。

