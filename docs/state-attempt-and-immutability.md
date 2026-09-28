# 状态、Attempt 与不可变性

## 1. 四层状态模型

任务和每个 attempt 都要分别记录四层状态：

| 层 | 字段 | 建议枚举 | 判断内容 |
|---|---|---|---|
| 执行 | `execution_state` | `NOT_STARTED/RUNNING/SUCCEEDED/FAILED/STOPPED/UNKNOWN` | 命令或作业是否按预期结束 |
| 产物 | `artifact_state` | `EXPECTED/MISSING/PARTIAL/COMPLETE/VERIFIED/UNKNOWN` | 文件、数量、结构和哈希 |
| 验收 | `validation_state` | `NOT_RUN/PASS/FAIL/PARTIAL/UNKNOWN` | 数值、物理、化学和方法学门禁 |
| 发布 | `release_state` | `HOLD/READY/RELEASED/REJECTED/UNKNOWN` | 是否允许下游使用或交付 |

状态应附证据路径和时间。`SUCCEEDED` 不自动意味着 `VERIFIED`，`COMPLETE` 不自动意味着 `PASS`，`PASS` 也不自动意味着 `RELEASED`。

## 2. Attempt 生命周期

推荐生命周期：

```text
PLANNED → PRECHECKED → RUNNING → SUCCEEDED/FAILED/STOPPED
                                  ↓
                         ARTIFACT_VERIFIED
                                  ↓
                         VALIDATION_PASS
                                  ↓
                         RELEASED
```

跳过的阶段写明原因。状态回退时保留旧状态和变更原因，不删除历史事件。长任务至少有独立日志、PID/作业标识和状态文件。

## 3. Attempt 命名

输入冻结后的第一次正式运行是 `attempt-001`。pilot、调试运行、恢复和重试都要有明确身份；若它们不属于正式结果，也应记录为独立 attempt 或 pilot attempt。

失败后不得删除目录再用原名重跑。新 attempt 必须记录：

```text
parent_attempt | reason | parameter_diff | new_input_manifest | started_at
```

安全续跑只有在 checkpoint、TPR、日志和 append 语义均通过检查时允许；否则新 attempt 从已验证输入起始。

## 4. 不可覆盖对象

默认不可覆盖：冻结输入、原始日志、原始轨迹、checkpoint、TPR、manifest、SHA-256 清单、已发布结构和验收报告。

派生分析也不应静默覆盖。相同参数和相同输入若结果一致可记为 no-op；否则创建新的 analysis_id。用户明确要求刷新时，记录覆盖范围、原因和新哈希。

任何“force”操作都必须限定到当前任务和派生产物，不能触碰原始输入、远程正式产物或其他任务。

## 5. 停止与失败

停止前确认 `task_id ↔ attempt_id ↔ job/PID ↔ input ↔ output ↔ log` 对应关系。禁止宽泛 `pkill`、模糊进程匹配或批量删除。

停止或失败后保留部分产物、原始日志、退出码、命令和原因。先诊断再重试；新 attempt 的 README 要链接旧证据。

## 6. 任务汇总

任务级状态是各 attempt 和分析的汇总，不覆盖成员细节。批量任务不得因为多数成员成功就把失败成员标为成功。汇总应显示计数、失败对象和 release 阻断项。

## 7. 审计格式

每次状态变更至少记录：

```text
time | actor | object | old_state | new_state | evidence | reason
```

状态未知时写 `UNKNOWN`，并列出下一步只读检查。不要用“应该没问题”“文件看起来存在”等推断代替证据。
