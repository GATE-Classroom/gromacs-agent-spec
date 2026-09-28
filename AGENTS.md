# AGENTS.md — GROMACS 工作区稳定工程规约

## 1. 适用范围

本工作区管理 GROMACS 分子动力学任务的输入、远程执行、审计、分析结果与交付物。
本文件只保存长期稳定的工程边界；具体协议、性能参数和版本经验写入任务记录或 `docs/`。
所有交流、任务记录和最终汇报使用中文；命令、字段名和标准术语可保留英文。

## 2. 不可违反的边界

- 不得把服务器 IP、SSH 用户、密码、密钥或设备专属 SSH alias 写入通用文档、脚本或配置模板。
- 所有远程查询、传输和执行必须使用 `ssh-skill`；禁止直接调用 `ssh`、`scp` 或远程 `rsync`。
- 未经用户明确授权，不修改远程部署、共享软件、公共力场或其他任务产物。
- 未经用户明确授权，不删除本地或远程文件；失败证据和旧 attempt 必须保留。
- 不得改写任务历史 README 中已经发生的执行事实；更正只能追加并说明依据。
- 不把退出码为 0、轨迹存在或图形正常直接等同于科学验收通过。

## 3. 数据驻留原则

本工作区本地长期区不保留轨迹，包括 `.xtc`、`.trr`、`.dcd` 等时间序列文件。
本地任务目录只保留输入、参数、命令、manifest、日志摘要、QC、派生小文件和交付物。
本地临时轨迹与分析缓存只能位于：

```text
D:/Temp/GROMACS/<task>/<analysis>/
```

临时目录必须可删除、可由远程必要轨迹和 replay 记录重建，不得成为唯一数据来源。
远程端保留复算、续跑、审计和已批准分析所必要的轨迹、检查点、日志、拓扑与最终结构。
远程保留集合必须在 manifest 中列出；空间不足时先形成归档计划，不得先删后补记录。
既有本地历史数据不在本规约中自动删除；迁移或清理必须由用户单独批准。

### 3.1 归档保留区（孤儿轨迹）

存在一类轨迹：**远程端已无从取得**（计算侧已清理、平台不再提供、或本就只产生于平台侧）。
若本地也不保留，该数据即永久丢失。此类数据不得进 `D:/Temp`（临时区可删除，会使其成为孤本），
只能归档到：

```text
Y:/Data/MD_Trajectories/<task_id>/
```

归档区与临时区的性质不同，不得混用：

| | `D:/Temp/GROMACS/` | `Y:/Data/MD_Trajectories/` |
|---|---|---|
| 性质 | 临时分析缓存 | 归档保留区 |
| 是否唯一副本 | 否，必须可由远程重建 | **是**，不得随意删除 |
| 可删除性 | 可随时删除 | 仅在确认有第二副本后 |
| 保护 | 无 | manifest + SHA-256 |

约束：

- 归档前必须逐文件校验（`size` + SHA-256），结果写入目标 `manifest.csv` 并回写 `tasks/<task_id>/manifests/`。
- manifest 中状态标 `local-authoritative`，即声明该数据当前仅此一份。
- 归档目录内部保留源相对路径，不得扁平化，以免丢失来源信息。
- `Y:` 是网络挂载（UNC 路径，具体映射以本机 `net use` 为准），可用性依赖 NAS 在线；这是明确的已知风险，不因此豁免归档前的校验。
- 一旦该数据在别处出现可靠副本，须更新 manifest 状态，不得沿用 `local-authoritative`。

详见 `docs/remote-artifact-and-temp-analysis.md`。

## 4. 单任务逻辑根

每个任务只有一个稳定 `task_id` 和一个逻辑根；同一任务不得按设备、服务器或分析阶段另建平行任务。
推荐本地逻辑根：

```text
tasks/<task_id>/
```

远程逻辑根由 deployment profile 解析，末级目录使用相同 `task_id`。
本地临时根固定映射到 `D:/Temp/GROMACS/<task_id>/`。
远程无副本的孤儿轨迹归档到 `Y:/Data/MD_Trajectories/<task_id>/`（见 §3.1）。
逻辑根下用 `attempts/<attempt_id>/` 区分运行尝试，用 `analysis/<analysis_id>/` 区分可重放分析。
历史任务可保留原结构；新工作不得为了套模板而破坏历史路径或事实。
详见 `docs/task-layout-and-replay.md`。

## 5. 部署与设备解析

必须分离三种标识：

1. `server_id`：跨设备稳定的逻辑服务器标识，不含连接凭据；
2. `deployment_profile`：GROMACS 版本、激活方式、远程根、调度后端和能力约束；
3. `local_alias`：当前设备 SSH config 中可用的连接别名，只在运行时解析。

任务记录保存 `server_id` 和 `deployment_profile`；`local_alias` 仅作为当次执行证据，可注明设备相关。
不得从 `server_id` 猜测 alias，不得从旧任务静默继承 profile，不得用 IP 代替任一标识。
profile 缺失、冲突或跨主机覆盖时，停止并向用户确认；不设置私人服务器 fallback。
执行前动态发现实际 GROMACS 版本、构建选项、力场来源、磁盘、调度和资源状态。
详见 `docs/deployment-and-device-resolution.md`。

## 6. Python 与依赖

本地 Python 必须通过工作区 `uv` 环境运行：

```bash
uv run python ...
```

禁止直接调用 `python`、`python3` 或 `pip install`。
可复用依赖使用 `uv add` 固化；环境同步使用 `uv sync --locked`。
不得手工修改 `.venv/`，不得归档或上传 `.venv/`。
一次性依赖注入仅限明确不会复用的工具，并须在执行记录中说明。

## 7. 四层状态

每个 attempt 和任务汇总必须分别记录四层状态：

1. `execution_state`：进程是否按预期结束；
2. `artifact_state`：预期产物是否存在、完整并通过哈希；
3. `validation_state`：数值、物理、化学和方法学门禁是否通过；
4. `release_state`：是否已批准进入下游、交付或归档。

各层只允许使用文档定义的枚举，不得用一个“完成”覆盖四层含义。
任何上层成功都不能自动推导下层成功；未知必须写 `UNKNOWN`，不得猜测。
详见 `docs/state-attempt-and-immutability.md`。

## 8. Attempt 与不可覆盖

正式运行、pilot、重试、恢复和参数变更都必须使用唯一 `attempt_id`。
推荐格式为 `attempt-001`、`attempt-002`；编号只递增，不复用。
冻结输入、原始日志、检查点、原始结果、manifest 和已发布交付物默认不可变。
发现同名目标时默认拒绝写入；重跑创建新 attempt，并记录父 attempt、原因和差异。
续跑必须保留 checkpoint 来源和追加语义；不能证明安全 append 时创建新 attempt。
失败 attempt 不删除、不改名冒充成功，也不以新日志覆盖旧日志。
详见 `docs/state-attempt-and-immutability.md`。

## 9. Pilot 门禁

新体系、新协议、新力场、新部署、新 GROMACS 版本或新分析链路必须先运行最小代表性 pilot。
pilot 应覆盖输入解析、拓扑、短运行、续跑或产物读取、分析和完整性检查。
pilot 通过后才能扩展批量或长生产；扩大规模的批准、证据和边界写入 README。
pilot 失败时保留证据并回到相应阶段，不得用 `-maxwarn` 或降低门禁掩盖问题。
pilot 成功只证明所测试范围，不自动证明长时间采样收敛或所有体系适用。

## 10. Replay 与 Manifest

每个正式 attempt 必须可由冻结输入和 replay 记录重建。
replay 至少记录命令、工作目录、环境激活、版本、输入哈希、参数、随机种子、资源和依赖 attempt。
命令中不得嵌入凭据或设备专属连接信息；连接目标以 `server_id/profile` 表达。
manifest 至少关联：

```text
task_id | attempt_id | stage | input | output | log | version | state | sha256
```

重要输入、运行参数、日志、最终结构、分析表和交付物必须生成 SHA-256。
大轨迹可用逐文件哈希或分块/归档清单；采用何种范围必须显式记录。
传输归档只是载体，解包并逐文件校验后不作为第二份权威副本长期保留。
详见 `docs/integrity-and-archive-policy.md`。

## 11. 版本与 Known Issues

GROMACS 二进制、构建选项、力场内容、分析库和文件格式读取器都视为版本化依赖。
执行前必须把实际版本与本工作区 `docs/maturity-and-todos.md` 的 known issues 对照（该文件是工作区本地运行记录，不在规约仓库中分发）。
任务 README 记录命中的 issue、适用判断、缓解措施和验证证据。
版本标签或模块名不是不可变身份；必要时记录二进制路径、版本输出和关键文件 SHA-256。
升级后不得沿用旧版本的 parser、性能或数值假设，须重新跑相关 pilot。
未经验证的方法按 A/B/C/TODO 成熟度标记，不得提升为稳定规则。

## 12. 分析与临时目录

分析必须先创建唯一 `analysis_id`，并在其临时目录保存输入映射、命令、参数和中间缓存。
分析不得直接改写远程原始轨迹；PBC、抽帧、拟合和拼接均生成派生产物。
长期保留的是可重放记录、小型结果、图表、选择清单和哈希，不是本地轨迹副本。
清理临时目录前验证结果已写回任务逻辑根，且 replay 能定位远程源数据。
分析失败或中断时如实记录，不能用部分帧结果冒充完整分析。

## 13. AlphaDynamics 路由

涉及 AlphaDynamics、Alpha Dynamics、Logic MD、LogicMD、MHPC512、思朗集群或 MaPU 的 MD 输入准备、转换、校验或打包时，必须使用：

```text
.agents/skills/alpha-dynamics-input-prep/SKILL.md
```

该 skill 依赖平台授权手册，**不在规约仓库中分发**，由工作区自行维护。
本文件不复制平台格式、转换步骤或验收细节；以该 Skill 和权威手册为准。
若请求只是非 MD 的芯片或指令集讨论，则不触发此路由。
远程操作、attempt、manifest、SHA-256、状态和不可覆盖规则仍完全适用。

## 14. 任务 README 与最终汇报

任务 README 至少记录目的、输入真值、`task_id`、profile、attempt、pilot、版本、命令、四层状态和路径。
还应记录 manifest/SHA-256、异常、known issue、未执行步骤、科学边界和下一步。
批量任务必须由 manifest 驱动，每个成员可唯一追溯到输入、attempt、输出和状态。
最终汇报必须区分做了什么、执行结果、产物完整性、科学验收和发布状态。
不得把经验性性能、单条轨迹平台、最低瞬时能量或视觉外观单独作为放行依据。
环肽 ensemble 的专门工作流见 `docs/cyclic-peptide-ensemble-workflow.md`。
工程成熟度与待办见工作区本地的 `docs/maturity-and-todos.md`（不在规约仓库中分发）。
