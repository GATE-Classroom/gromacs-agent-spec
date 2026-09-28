# 部署与设备解析

## 1. 三层标识分离

GROMACS 任务必须把以下概念分开：

| 标识 | 含义 | 是否稳定 |
|---|---|---|
| `server_id` | 物理或逻辑服务器的跨设备身份 | 稳定 |
| `deployment_profile` | 版本、路径、后端、激活方式和能力集合 | 可版本化 |
| `local_alias` | 当前设备 SSH config 的连接入口 | 设备相关 |

通用文档只写 `server_id` 和 profile 名称，不写固定 IP、用户、密码或 alias。任务 README 可以记录当次解析出的 alias 作为执行证据，但必须注明它只对当时设备有效。

## 2. Profile 内容

一个 deployment profile 至少描述：

- `server_id`；
- 远程项目根和任务目录规则；
- GROMACS 版本或版本发现命令；
- 激活方式和非交互 shell 要求；
- 执行后端（直接进程、调度器或其他）；
- 力场和共享数据的来源标识；
- 资源查询和日志位置规则；
- 远程路径是否可读写、是否需要 staging；
- known issues、适用边界和最近验证时间。

Profile 是配置来源，不应把当前一次任务的临时覆盖反写为默认值。版本、磁盘、运行进程和资源利用率必须在任务预检时动态确认。

## 3. 解析优先级

字段解析顺序为：

```text
用户当前任务明确指定
> 当前任务明确指定的 profile
> 工作区注册的默认 profile
> 询问用户补齐必要字段
```

解析结果要保存 `field | value | source`。如果覆盖了目标主机，不能继续继承原主机的路径、后端、GPU 或容器假设；必须从新 profile 或任务显式值重新解析。

不得从历史 README 猜测缺失配置，不得用 IP 临时代替 alias，也不得把 alias 当成 server_id 写入脚本。

## 4. 设备本地配置与 alias 查证

共享 deployment profile 位于 `config/gromacs-deployments.toml`，只保存稳定、非秘密字段。每台设备将 `config/gromacs-local.toml.example` 复制到：

```text
~/.config/zcode/gromacs-local.toml
```

也可用 `GROMACS_LOCAL_CONFIG` 指定其他本机路径。真实 local config 不进入同步工作区，且只能保存当前设备的 route/alias 选择和临时目录根，不保存密码、私钥或 token。

开始任务前，在当前设备的 SSH 配置中查找指向 `server_id` 的 alias。查证只读，不修改 SSH 配置，除非用户明确要求。

如果找到多个 alias，选择用户指定或 profile 允许的入口并记录来源；如果找不到、指向不一致或配置过期，停止远程操作并报告。不同 Windows 设备可以得到不同 alias，这不构成任务身份变化。

所有 SSH/SCP/上传/下载操作继续遵循 `ssh-skill`，不直接写 `ssh` 或 `scp` 命令。

## 5. 预检

远程运行前通过 `ssh-skill` 合并只读检查：

- hostname、时间和 `server_id` 是否一致；
- GROMACS 版本、构建和激活是否符合 profile；
- 目标远程路径可访问且空间足够；
- 调度、GPU/CPU 或其他资源状态；
- 任务、attempt、输入、输出和日志命名无冲突；
- 必要力场文件存在且来源可审计；
- profile 列出的 known issue 是否命中。

预检通过不等于资源已预约。无调度器时要记录预检和启动之间的竞态；不能把“看似空闲”描述为独占资源。

## 6. 版本证据

任务 README 至少保存实际版本输出、profile、远程路径、开始时间和关键输入/力场哈希。新版本或未知构建必须先做 pilot。

任务级覆盖只对当前 task/attempt 有效。不得在文档中复制服务器硬件教程或把一次测得的性能写成普适保证。

## 7. 未连接时的行为

如果用户要求只改文档或明确禁止连接服务器，只做本地配置和静态检查，不伪造预检、版本、资源或执行状态。README 应区分“已记录的历史事实”和“本次未验证”。
