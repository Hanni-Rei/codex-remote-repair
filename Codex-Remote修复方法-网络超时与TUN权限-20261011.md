# Codex Remote 修复方法：网络超时与 TUN 权限

案例与整理日期：2026-10-11（Asia/Shanghai）
环境：Windows、Codex 桌面版、已安装的 Clash Verge / Mihomo
验收结果：电脑端连接恢复，用户确认手机已能连接并正常操作。

> 本文记录一次实际修复，不是所有 Remote 故障的通用结论。本次原因与[重复后端冲突案例](Codex-Remote修复方法-重复后端冲突-20260915.md)不同；使用前应重新检查当前进程、配置与日志。

## 1. 适用条件与故障证据

本次桌面端提示“无法更新远程控制状态”，以下现场证据共同支持网络路径排查：

- 只有一个桌面主应用及其对应的 `codex.exe app-server`，没有同一主应用下重复后端的证据。
- 本地后端初始化成功，握手耗时约 222 毫秒。
- 远程控制 WebSocket 反复失败，底层错误为 `os error 10060`。
- 同一地址直连超时，经现有本地代理能够收到 HTTP 响应。
- Clash 系统代理开启、TUN 关闭；尝试开启 TUN 后日志明确报权限不足。

桌面安装包版本为 `26.1007.2314.0`，日志中的桌面客户端版本为 `26.1007.21434`。两者来自不同字段，仅用于标记本案例，不是其他电脑的固定版本要求。

## 2. 故障链与修复结果

**远程控制连接直连超时 → 仅有系统代理时仍不能连通 → TUN 因权限不足未启动 → 以足够权限启动现有代理并开启 TUN → Remote 连接成功。**

| 检查项 | 修复前 | 修复后 |
|---|---|---|
| 桌面关联后端 | 一个，初始化成功 | 保持原有后端，未重启 Codex |
| 远程控制地址 | 直连约 8 秒超时 | 无显式代理的测试约 2.32 秒收到 HTTP 426 |
| 显式使用本地代理 | 约 1.69 秒收到 HTTP 426 | 保留现有系统代理设置 |
| TUN | 关闭；启动尝试失败 | 保存为开启，`Meta Tunnel` 网卡为 `Up` |
| 权限 | `configure tun interface: Access is denied.` | 管理员重启后，观察到代理服务运行 |
| Remote 状态 | 反复报 `10060` | 10:22:01 记录 `Connected` |
| 设置页 | 远程控制不可用 | 允许连接开启；设备列表请求成功、`errorCode=null`，显示手机刚刚连接 |
| 手机实测 | 不可用 | 用户确认“已能连接并正常操作” |

恢复连接后至本次核对时，相关远程控制日志未再出现警告或错误。这是观察窗口内的结果，不是长期稳定性保证。

已确认的是本次网络路径与权限修复有效。不能据此断言所有版本都忽略系统代理、所有 `10060` 都由 TUN 关闭导致，或所有 Remote 故障都需要改代理。

## 3. 核对进程与日志

以下 PowerShell 命令只读取进程信息：

```powershell
Get-CimInstance Win32_Process |
  Where-Object { $_.Name -match '^(codex|ChatGPT)\.exe$' } |
  Select-Object ProcessId, ParentProcessId, Name, ExecutablePath,
                CommandLine, CreationDate |
  Format-List
```

按父子关系和完整命令行识别桌面主应用、`app-server`、`exec-server`、渲染器及其他工具进程。独立诊断工具也可能启动 `app-server --listen stdio://`，不能仅按 `codex.exe` 数量判定重复实例。

当前版本的后端可位于用户目录下的运行时文件夹，桌面程序则位于 WindowsApps 安装包；不能仅按相同安装目录判断归属。权限不足导致路径不可见时，应写明“暂未确认”。只有发现同一桌面主应用下重复后端，且日志支持异常重建后残留，才转到重复后端指南。

优先查看故障时间附近的日志，区分以下层级：

| 层级 | 本次使用的日志证据 |
|---|---|
| 本地初始化 | `initialize_handshake_result`、`outcome=success` |
| 远程连接失败 | `failed to connect to app-server remote control websocket`、`os error 10060` |
| 远程连接恢复 | `next_status=Connected`、`connected to app-server remote control websocket` |
| TUN 权限不足 | `Start TUN listening error`、`configure tun interface: Access is denied.` |

本次桌面日志位于安装包用户数据中的 `LocalCache\Local\Codex\Logs`，部分后端事件保存在 Codex 用户数据的 `logs_2.sqlite`。其他版本可能采用不同路径或数据库结构；读取数据库应使用只读连接，不要删除锁文件、账号文件或数据库。

`remoteControl/status/read` 或 `remoteControl/enable` 返回 `errorCode=null`，只代表对应请求未报错，不能单凭它证明网络连接已建立。应核对实际 Remote 状态及手机操作。

## 4. 比较直连与现有代理

先确认本机代理地址和端口。下面的 `7897` 是本案例端口，应替换为现场实际值。

```powershell
# 禁用 curl 的显式代理；若 TUN 已启动，此请求仍可能被 TUN 接管。
curl.exe --noproxy '*' --connect-timeout 8 --max-time 12 `
  -sS -o NUL -w 'status=%{http_code} connect=%{time_connect} total=%{time_total}\n' `
  'https://chatgpt.com/backend-api/wham/remote/control/server'

# 明确使用已存在的本地代理。
curl.exe --proxy 'http://127.0.0.1:7897' --connect-timeout 8 --max-time 12 `
  -sS -o NUL -w 'status=%{http_code} connect=%{time_connect} total=%{time_total}\n' `
  'https://chatgpt.com/backend-api/wham/remote/control/server'
```

这些请求不携带账号令牌。HTTP 426 表明收到了要求协议升级的响应，说明该测试路径可达；它不代表认证成功或完整 WebSocket 握手通过。TUN 开启后的无显式代理测试不能再称为纯直连。

若两条路径都失败，继续检查节点、DNS、路由和服务状态。若两条路径都可达，继续检查带认证的实际连接日志，不要直接套用本案例结论。

## 5. 修复 TUN 权限

1. **保存基线。** 备份代理配置，记录系统代理、TUN、节点和规则状态，核对当前代理及核心的实际路径与归属。
2. **明确中断范围。** 代理重启会短暂中断网络，确认正在运行任务的中断范围后再执行。
3. **以足够权限启动。** 退出已核实的当前 Clash 实例并确认其核心退出，从本机已安装的有效入口以管理员权限启动；Windows 授权提示由用户确认。
4. **开启并保存 TUN。** 也可按当前 Clash 版本提供的服务模式获得所需权限，以实际状态为准。
5. **检查真正生效。** 查看代理日志和网卡，确认没有权限失败，而不只是开关变为开启。

本次保留了带时间戳的备份，配置比较显示 `enable_tun_mode` 从 `false` 改为 `true`；没有更换节点或修改规则。重启后观察到 `clash_verge_service` 正在运行、启动类型为 `Automatic`，以及 `Meta Tunnel` 网卡为 `Up`。其他安装方式、服务名和网卡名可能不同。

```powershell
Get-NetAdapter | Select-Object Name, InterfaceDescription, Status
Get-Service |
  Where-Object { $_.Name -match 'clash|verge' } |
  Select-Object Name, Status, StartType
```

**出现 `Access is denied` 时，TUN 没有成功启动。** 控制接口返回成功或开关亮起不能代替网卡与日志检查。

不要按名称批量结束所有代理或 Codex 进程，不要使用历史 PID、运行时目录或控制管道作为固定值。本仓库不附带绑定特定机器的一键提权或强制结束进程脚本。

## 6. 完整验收

- [ ] TUN 网卡实际启动，日志没有权限失败。
- [ ] 原先超时的网络路径能够收到响应。
- [ ] Remote 后端状态为 `Connected`，观察期间没有再次超时。
- [ ] 设置页“允许连接”开启，刷新设备列表成功。
- [ ] 手机 Remote 能连接主机、打开已有对话并正常操作。
- [ ] 已记录配置变化与备份位置，明确尚未验证项目。

本案例以上项目均已完成，最终状态为**已恢复**。没有重启 Codex 或 Windows，没有修改账号、工作区、配对和本地数据库。

## 7. 回退与复发处理

- 不再需要本次改动时，可通过当前 Clash 界面关闭 TUN；必要时对照备份恢复对应配置。关闭后原先的直连超时可能重新出现，应重新验证 Remote。
- 复发时先检查最新状态与日志：TUN 是否启动、权限或服务是否变化、节点与路由是否可用，以及是否出现新的错误类型。
- 若出现重复桌面后端，转到重复后端指南，避免反复重启代理掩盖其他原因。
- 服务正在运行不等于已做过系统重启后的自动恢复测试。本案例未进行 Windows 重启验收。

## 8. 复用记录模板

```text
检查时间与时区：
Codex / 代理版本及安装方式：
提示原文与故障时间：
桌面主应用及关联后端数量：
本地初始化结果：
远程错误与日志时段：
直连测试 / 显式代理测试：
TUN、系统代理、权限与服务状态：
现有任务与允许中断范围：
配置备份位置（仅本地保留）：
实际修改：
网卡、网络路径、Connected 和设备列表验证：
手机实际操作结果：
尚未完成的验证：
最终状态：已恢复 / 待验证 / 未恢复
```

公开记录只保留必要的脱敏错误片段与结果，不上传认证文件、代理订阅、完整私有配置、账号或设备唯一标识。
