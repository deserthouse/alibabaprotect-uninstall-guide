# AlibabaProtect Uninstall Guide

**Windows 上清理 AlibabaProtect（`Alibaba PC Safe Service`）的实测指南**

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-0078D4.svg)](https://creativecommons.org/licenses/by/4.0/)
[![English summary](https://img.shields.io/badge/README-English_summary-0078D4.svg)](#english-summary)

原理与证据 → [alibabaprotect-forensics](https://github.com/deserthouse/alibabaprotect-forensics)

---

## 先说结论：这是什么、能不能删

- 你在任务管理器里看到的 `AlibabaProtect.exe`（服务名 `Alibaba PC Safe Service`），是阿里系软件（淘宝 / 旺旺 / 优酷 / UC / 1688 等）安装时**附带装入**的后台服务，属于阿里自带组件，不是病毒。
- **卸载那些阿里系软件后，它不会跟着卸载**，仍会在后台常驻运行并消耗 CPU。
- 如果你不依赖任何阿里系软件的功能，它可以安全删除；下面每一步都给出**预期输出**，照着核对即可。
- 暂时不想动系统？先做[第二节只读检查](#二先做只读检查不改动任何东西)，不改动任何东西，看完再决定。
- 看不懂命令行？把这份指南的链接交给你的 AI 助手，让它按文档逐步讲解或代你执行——每一步都有预期输出，照单核对即可。

---

## 这份指南适用于谁

✅ **适用于**：

- 你已经卸载了装它的客户端（如阿里旺旺 / 淘宝 / 优酷 / UC / 钉钉 / 1688 等），但它**仍然在后台运行**
- 你在任务管理器里看到 `AlibabaProtect.exe`，且占用可观的 CPU / 内存
- 你自己判断**不需要**这个组件的功能

❌ **不适用于**：

- 你正在使用依赖它的功能
- 你不确定它是什么、也不确定是否需要它

> 本指南描述的是**在你自己的设备上**、由**你本人以管理员权限**执行的清理步骤。请先读完一遍再动手。

---

## 一、它是什么（客观描述）

| 项目 | 值 |
|---|---|
| 服务名 | `AlibabaProtect` |
| **显示名**（`services.msc` 里看到的） | **`Alibaba PC Safe Service`** |
| 进程 | `AlibabaProtect.exe` |
| 安装路径 | `C:\Program Files (x86)\AlibabaProtect\<版本号>\`（如 `1.0.70.3194`） |
| 配套内核驱动 | 服务名 `AliPaladin`，文件 `C:\Windows\System32\drivers\AliPaladinEx64.sys` |

**为什么常规方法停不掉它** —— 对该驱动 `AliPaladinEx64.sys` 做静态分析可以看到它同时注册了：

| 注册的机制 | 效果 |
|---|---|
| `FltRegisterFilter`（**文件系统微过滤器**） | 保护自己的安装目录，`Remove-Item` 可能报 `Access denied` |
| `CmRegisterCallback`（**注册表回调**） | **"禁用后几秒 `Start` 自己变回 `auto`"的机制与该能力一致**（回滚动作与驱动注册的注册表回调相符，属推断；执行者未被逐一追踪，也可能是用户态进程轮询回写） |
| `PsSetCreateProcessNotifyRoutine`（**进程回调**） | 守护 `AlibabaProtect.exe` 本体 |

此外它还带 `RestartService.exe`（自我重启）、`AntiDebug.dll` / `AntiInject.dll`（反调试/反注入）。

> 完整的行为分析（枚举 API、上报通道、内嵌 SQLite、构建路径等）见取证仓。

---

## 二、先做只读检查（不改动任何东西）

以**管理员**身份打开 PowerShell，先看清楚现状：

```powershell
# 1) 服务与驱动服务是否存在、什么状态
sc.exe qc AlibabaProtect
sc.exe query AlibabaProtect
sc.exe qc AliPaladin

# 2) 进程是否在跑、占多少
Get-Process AlibabaProtect -ErrorAction SilentlyContinue |
    Select-Object Id, StartTime, CPU, @{n='WS_MB';e={[math]::Round($_.WorkingSet64/1MB,1)}}

# 3) 安装目录与驱动文件
Test-Path 'C:\Program Files (x86)\AlibabaProtect'
Test-Path 'C:\Windows\System32\drivers\AliPaladinEx64.sys'

# 4) 相关计划任务与启动项
Get-ScheduledTask | Where-Object { $_.TaskName -match 'Ali' } | Select-Object TaskName, State
```

---

## 三、三条已知的重新出现路径

清理前请先知道它可能怎么回来：

| 路径 | 机制 | 应对 |
|---|---|---|
| **客户端重装** | 阿里系客户端（旺旺/淘宝/优酷/1688 等）启动或更新时会检查并重新安装 | 见**第六节「防复发」** |
| **SCM 自动恢复** | 服务崩溃后，**服务控制管理器的恢复策略会在 60 秒后自动重启它**（系统日志事件 ID `7031`；注册表 `FailureActions` 里也写着同一策略：**3 次、每次 60000 毫秒**） | 删除服务（**而非仅禁用**）即可断掉 |
| **配置被回滚** | `Start` 改成 `Disabled` 后几秒自己变回 `Auto`（实测观察）；机制与配套驱动注册的注册表回调能力**相符（推断，未追踪执行者）** | 只能靠**删服务** + **重启**，改配置没用 |

> **三者独立**：删掉客户端不等于删掉服务；删掉服务才断掉 SCM 的自动恢复；而"改配置"这条路本身就走不通。

---

## 四、清理步骤

> 命令需在**管理员** PowerShell / CMD 中执行。如遇 `拒绝访问`，跳到第四节末尾的「备选路径」。

### 阶段 A：断掉自启与更新

```powershell
# 1) 先禁用相关计划任务（否则它会定期检查并装回来）
foreach ($t in @('AliProctectUpdate','AliUpdater')) {
    $task = Get-ScheduledTask -TaskName $t -ErrorAction SilentlyContinue
    if ($task) { Disable-ScheduledTask -TaskName $t }
}
```

> 📝 **`AliProctectUpdate` 的拼写不是笔误。** `Ali` + `Proctect`（把 `Protect` 写成了 `Proctect`）是**厂家的原始任务名**。照抄才能命中；改成"正确的" `AliProtectUpdate` 会找不到该任务。
>
> 顺带提醒：计划任务名与**进程名**不同 —— 进程叫 `AliProtectUpdate.exe`（拼写正确）。两者别混。

### 阶段 B：删除服务（断掉 SCM 自动恢复；进程由阶段 C 结束）

```cmd
sc.exe stop AlibabaProtect
sc.exe delete AlibabaProtect
```

> 预期：`stop` 可能返回 **1052（请求的控件对此服务无效）**——这是正常的，该服务没有实现停止逻辑。**`delete` 应当成功**（`[SC] DeleteService 成功`）。
>
> ⚠️ **`delete` 只是标记删除，不会终止正在运行的进程**——服务注册项删除后，SCM 不会再把它拉起来，但进程本体可能仍在运行，由下一阶段（阶段 C）显式结束。

### 阶段 C：结束残留进程

```powershell
Get-Process AlibabaProtect -ErrorAction SilentlyContinue | Stop-Process -Force
```

### 阶段 D：删除内核驱动服务与文件

```cmd
sc.exe stop AliPaladin
sc.exe delete AliPaladin
```

```powershell
Remove-Item 'C:\Windows\System32\drivers\AliPaladinEx64.sys' -Force
```

> 驱动**在线不可停止**（`sc stop` 会报 1062 或 1052）。删除服务注册项后，**重启即不再加载**；文件删除失败也不影响结论。

### 阶段 E：删除安装目录与残留数据

```powershell
Remove-Item 'C:\Program Files (x86)\AlibabaProtect' -Recurse -Force
Remove-Item 'C:\ProgramData\Alibaba\AlibabaProtectDT' -Recurse -Force
```

### 阶段 F：重启

**必须重启一次。** 驱动需要重启才会真正卸载；服务被删除后，SCM 也不会再把它拉起来。

### 备选路径：如果某一步报「拒绝访问」

说明自我保护机制仍在起作用（进程还活着）。改用**两阶段**方式：

1. **阶段 1（服务还在运行时）**：只做 `sc.exe config AlibabaProtect start= disabled`，然后重启
2. **重启**（此时它不会被拉起来，进程与文件占用都消失）
3. **阶段 2**：重启后执行上面的阶段 B–E（此时通常不会再受阻）

---

## 五、验证清单

重启后逐项核对：

| # | 检查 | 预期结果 |
|---|---|---|
| 1 | `sc.exe query AlibabaProtect` | **错误 1060**（服务未安装） |
| 2 | `tasklist /FI "IMAGENAME eq AlibabaProtect.exe"` | 无输出 |
| 3 | `Test-Path 'C:\Program Files (x86)\AlibabaProtect'` | `False` |
| 4 | `reg query "HKLM\SYSTEM\CurrentControlSet\Services\AlibabaProtect"` | 找不到项 |
| 5 | `Get-ScheduledTask \| Where-Object { $_.TaskName -match 'Ali' }` | 无 AlibabaProtect 相关条目 |
| 6 | `reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\AlibabaProtect.exe" /v Debugger` | **若你做了第六节的防复发设置，这里【应当有】条目** —— 见下方说明 |

五项（1–5）全部符合 = 清理完成。

> ⚠️ **第 6 项不要当残留删掉。** 如果你按第六节设置了 IFEO 拦截，`AlibabaProtect.exe` 等 **4 个**映像名在 `Image File Execution Options` 下**必然存在**键值 —— 那是**预期状态**，不是"没删干净"。删掉它等于关掉防复发。
>
> 想确认它是否在工作，用**功能验证**而不是看注册表：见 [取证仓的执行级取证章节](https://github.com/deserthouse/alibabaprotect-forensics/blob/main/docs/05-execution-forensics.md)。

**自动化核验**：不想逐条敲命令的话，取证仓提供了只读的核验脚本（含上述 1–5 项，并额外检查驱动与残留目录）：

```bash
python verify_clean.py
```

---

## 六、防复发

如果你不希望它被客户端重新装回来，有一种**不改动任何客户端文件**的做法：**映像执行选项（IFEO）**。

原理简述：Windows 在创建进程时会按**映像文件名**查询 `Image File Execution Options` 注册表位置；把一个名称的 `Debugger` 指向**不存在的路径**，那么任何以该名称启动的进程都会在 `CreateProcess` 阶段失败——**谁调用都一样，按名字全局生效**。

⇒ 完整原理、实测验证方法与局限，见取证仓：

| 内容 | 链接 |
|---|---|
| 防复发：方案对比、IFEO 原理、**写入被安全软件拦截的排查** | [docs/06-prevent-recurrence.md](https://github.com/deserthouse/alibabaprotect-forensics/blob/main/docs/06-prevent-recurrence.md) |
| 怎么**证明**拦截真的生效（探针实验 / Prefetch 指纹） | [docs/05-execution-forensics.md](https://github.com/deserthouse/alibabaprotect-forensics/blob/main/docs/05-execution-forensics.md) |
| 为什么删了还会回来（三条重新出现路径） | [docs/04-why-hard-to-remove.md](https://github.com/deserthouse/alibabaprotect-forensics/blob/main/docs/04-why-hard-to-remove.md) |

**一键执行**（脚本在取证仓 [`scripts/`](https://github.com/deserthouse/alibabaprotect-forensics/tree/main/scripts)，下载到本地后运行；可逆）：

```bash
python ifeo_block.py --status     # 查看状态（只读）
python ifeo_block.py --apply      # 建立 4 条拦截（需要管理员）
python ifeo_block.py --probe      # 功能性验证：真的拦住了吗
python ifeo_block.py --remove     # 一键还原
```

---

## 七、如何还原

如果你之后又需要它：

1. **删除 IFEO 拦截**（如果你做过防复发设置）
2. **重新安装**：安装任意阿里系客户端，或从官方渠道获取安装包

清理过程**删除了文件，但没有修改客户端**，所以还原不需要修复客户端。

---

## 八、常见问题（FAQ）

**Q：`sc.exe stop` 报 1052 怎么办？**
A：正常，该服务未实现停止逻辑。直接执行 `sc.exe delete` 即可。

**Q：`sc.exe delete` 之后服务还在 `services.msc` 里？**
A：删除标记需要重启后才完全生效。重启即可。

**Q：删文件报 `Access denied`？**
A：进程仍在运行（自我保护）。先确认 `AlibabaProtect.exe` 已结束；若仍不行，用第四节末尾的「备选路径」。

**Q：写注册表报 `拒绝访问`，但我明明是管理员？**
A：这**通常不是权限问题，而是安全软件（HIPS）的"注册表防护"**在拦截 —— 它专门拦 `Image File Execution Options` 的**新建子键**（IFEO 是常见的持久化手法，保护它是合理的）。

快速确认：**普通 HKLM 键能写、IFEO 子键不能写** 即可锁定是它。

处置：临时退出该安全软件 → 建立拦截 → **再把它开回来**（不要长期关闭）。详见[取证仓第 4.4 节](https://github.com/deserthouse/alibabaprotect-forensics/blob/main/docs/06-prevent-recurrence.md)。

**Q：`AliPaladin` 驱动删不掉文件？**
A：驱动在线无法卸载。删除服务注册项 + 重启，之后文件即可删除。

**Q：清理后 Ctrl+F 搜不到任何残留，但还是担心？**
A：执行第五节的五项验证清单。五项全部符合即为完成。

---

## 九、为什么不提供一键清理工具

本指南只提供文档和少量只读/可逆脚本，不提供"一键删除"工具，原因如下：

1. **清理动作不可逆，且因版本而异**。它的安装路径带版本号（实测一台机器上三个版本目录并存）、驱动有 8 个变体、不同机器加载的文件名不同（详见[取证仓 01](https://github.com/deserthouse/alibabaprotect-forensics/blob/main/docs/01-what-it-is.md)）。按固定路径写死的一键工具在别的机器上会**漏删或误删**。
2. **部分机器上会静默失败**。例如防复发步骤写入注册表的操作会被安全软件（HIPS）拦截（见第八节 FAQ），一键工具遇到拦截往往半途而废，反而留下"删不干净"的状态。
3. **文档让每一步可见**。本指南的每个命令都附预期输出，出错的当场就能发现并停下；工具执行失败时使用者通常毫无察觉。
4. 提供的边界：取证仓中的 `verify_clean.py`（只读核验）与 `ifeo_block.py`（可逆注册表脚本）**不含任何删除动作**，属于安全可逆的辅助；不可逆的删除操作一律由本指南文档承载、由使用者本人执行。

---

## 免责声明

- 本文档仅供**在你拥有管理员权限的自有设备上**、清理**你本人明确不需要**的软件时参考。
- 作者与文中提及的任何厂商均无关联。
- 操作**需要管理员权限**，且包含对系统服务、驱动与注册表的修改。**请在操作前创建系统还原点，并自行确认每一步的含义。**
- 请勿将本文用于他人设备或任何未经授权的场景。
- 本指南为纯文档，**不包含任何厂商软件的二进制或本体资源**；文中引用的日志/报错原文仅为说明所需的少量引用，全文为作者原创内容。
- 文中命令基于作者在实机上的验证记录整理；不同版本的文件路径/版本号可能不同，执行前请先用第二节的只读命令核对你自己的实际情况。

## AI 使用声明 / AI Usage Statement

- **中文**：本指南的调研、取证与撰写**深度参与使用了 AI 工具**（命令整理与文本组织）。全部**清理决策与最终验收由人类作者作出**；所有命令与结论均在作者本人设备上实测通过后发布。
- **English**: AI tools were **substantially involved** in drafting this guide (command organization and text). All **cleanup decisions and final acceptance were made by the human author**; every command and conclusion was verified on the author's own machine before publication.

## License

本文档以 **[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)** 发布。

---

## English summary

A hands-on guide to removing **AlibabaProtect (`Alibaba PC Safe Service`)** from Windows, for users who have already uninstalled the client that shipped it and no longer need the component.

**Why normal removal fails:** its companion kernel driver `AliPaladinEx64.sys` registers a **file-system minifilter** (`FltRegisterFilter`) to protect its install directory, a **registry callback** (`CmRegisterCallback`) consistent with the observed "start type reverts to *auto* seconds after being disabled" behavior (mechanism inference — the reverting agent was not traced), and a **process-creation callback** (`PsSetCreateProcessNotifyRoutine`) guarding the main process.

**Approach:** disable the updater scheduled tasks → `sc delete` the service (this cuts off SCM recovery; the running process is killed explicitly next) → kill any remaining process → `sc delete` the driver service → delete files → **reboot** → verify with a 5-item checklist (plus a 6th *informational* check that flags the expected IFEO entries if you applied the anti-recurrence step). Reboot is required so the driver unloads.

**Recurrence:** it can be reinstalled by Alibaba-family clients on launch/update, and the Service Control Manager's recovery policy restarts the service 60 s after a crash (event `7031`). The [forensics repo](https://github.com/deserthouse/alibabaprotect-forensics) documents a mechanism that blocks reinstallation without modifying any client file — and, importantly, how to **prove** it works.

> Note on scheduled task names: the task is literally named `AliProctectUpdate` (the vendor's own typo), while the *process* is `AliProtectUpdate.exe`. Don't "fix" the spelling or the command will miss.

If the command line is unfamiliar, hand this guide's link to your AI assistant and have it walk you through — or execute for you; every step carries expected output to verify against.

This document is **documentation only** — no binaries, no releases.

**Disclaimer (English):** This guide is intended **only for use on your own devices, by you, with administrator privileges**, to remove software you have decided you do not need. The author is not affiliated with any vendor mentioned. The operations require admin rights and modify system services, drivers, and the registry — **create a restore point first and understand each step**. Do not use this on other people's devices or in any unauthorized context. This guide is original text and contains **no vendor binaries or original resources**; quoted log/error lines are minimal excerpts for explanation only. Published under CC BY 4.0, without warranty of any kind.
