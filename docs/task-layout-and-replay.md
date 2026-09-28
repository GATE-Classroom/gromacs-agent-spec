# 任务布局与 Replay

## 目的

本文件定义新 GROMACS 任务的单任务逻辑根、阶段目录和可重放记录。它不要求改造既有历史任务；历史事实优先保留。

## 1. 唯一身份

每个研究任务使用唯一 `task_id`，推荐格式：

```text
YYYYMMDD_<project-or-target>_<short-purpose>
```

`task_id` 表示研究问题，不表示某次运行、服务器或分析。所有本地记录、远程逻辑根、manifest 和交付物都通过它关联。

每次实际执行使用新的 `attempt_id`，例如 `attempt-001`。每次派生分析使用新的 `analysis_id`，例如 `analysis-001`。同一任务可以有多个 attempt 和 analysis，但不能有多个逻辑根。

## 2. 推荐目录

```text
tasks/<task_id>/
├── README.md
├── input/                 # 冻结输入或小型输入清单
├── params/                # mdp、选择规则、协议快照
├── attempts/
│   └── <attempt_id>/      # 命令、状态、manifest、日志摘要和证据
├── analysis/              # 小型结果、报告和可重放定义
├── exports/               # 已批准下游交付物
└── audit/                 # SHA-256、版本和审计记录
```

轨迹、检查点和大型运行文件不放入本地任务根。它们保留在远程任务根的对应 attempt 下；分析临时文件只放 `D:/Temp/GROMACS/<task_id>/<analysis_id>/`。

远程目录由 deployment profile 提供根路径；其末级目录为 `<task_id>`，attempt 和 stage 使用任务约定命名。不要建立按本地设备或服务器 alias 分裂的复制任务。

## 3. attempt 内容

每个 attempt 至少需要：

```text
attempt.yaml 或 attempt.tsv
replay.sh / replay.txt       # 实际命令和顺序记录，不要求可执行
manifest.tsv
status.tsv
logs/                         # 远程日志的索引或摘要
```

实际文件名可按现有工具调整，但字段语义不能缺失。`attempt` 记录父 attempt、启动原因、输入清单、参数版本、资源、退出码和四层状态。

## 4. Replay 最小要求

replay 必须回答：

- 从哪个 `task_id/attempt_id` 开始；
- 使用哪些冻结输入和 SHA-256；
- 运行了哪些阶段、命令、工作目录和环境激活；
- 使用哪个 `server_id` 与 deployment profile；
- 当时解析出的 local alias（仅作证据，不作稳定配置）；
- GROMACS、力场和分析工具的实际版本；
- 随机种子、时间步、步数、输出间隔和选择组；
- 每个阶段产生哪些输出、日志和校验值；
- 失败、重试、恢复或人工批准发生在哪里。

Replay 不能包含密码、密钥、IP 或不可追溯的“运行同样命令”描述。可用变量引用 profile 和任务 manifest。

## 5. 分析 replay

分析命令不得隐式读取某个用户目录或旧缓存。应把远程源文件、时间范围、抽帧规则、PBC 处理、选择表达式、算法参数、软件版本和输出路径写入 analysis replay。

临时分析完成后，将报告、表格、图和必要的小型样本复制到任务逻辑根，并记录来源轨迹的远程路径与 SHA-256。不要把整个轨迹复制进 `tasks/`。

## 6. 批量任务

批量输入由 manifest 驱动。每一行必须能唯一解析到成员、输入、attempt、远程输出前缀和状态。批量中的单项失败不能被“整体完成”掩盖；保留已完成项并为失败项创建新 attempt。

## 7. 迁移和历史兼容

旧任务可能直接在任务根保存 mdp、日志或小型结果，也可能在旧目录中有轨迹。新规约不追溯改写这些事实。若需要迁移，先复制清单和哈希，再在独立变更中记录迁移前后路径、未迁移项和批准人。
