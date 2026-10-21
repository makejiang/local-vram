---
name: local-vram
description: |
  Windows AI PC 上的英特尔本地集成显卡显存调整。当用户使用中文或英文要求限制、调整、增加、更改或查询 GPU 共享内存上限时使用此技能。触发词包括“限制GPU显存/调整显存/设置显卡显存/更改共享显存/限制显存/查询显存设置”，以及 limit GPU memory/adjust VRAM/set VRAM/change GPU shared memory/set GPU commit limit/query GPU memory limit；无论用户是否明确指定“locally/本地”都应触发。

  此技能可查询当前 Windows GPU 共享内存上限，或将其设置为系统总内存的 13% 到 87%（最大 87%）。

  当用户想查询或调整本机 GPU/VRAM 内存上限时，优先使用此技能。
---

# local-vram

**此技能不附带任何脚本。** 请按下面的说明自己执行命令来读写注册表。

## 目标注册表值

| 项目 | 值 |
| --- | --- |
| 注册表项 | `HKLM\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\MemoryManager` |
| 值名称 | `SystemPartitionCommitLimitPercentage` |
| 类型 | `REG_DWORD` |
| 含义 | GPU 共享内存上限占**系统总内存**的百分比 |
| 支持范围 | 13 – 87（最大 87） |

**只允许读写上面这一个值。** 禁止创建、修改或删除 `MemoryManager` 项下的其他值（如 `DedicatedSegmentSize`、`HwSchMode`），也禁止改动任何其他注册表项。如果用户的目标无法通过这一个值达成，请直接说明，不要改动别的键值。

## 运行约束

- 必须在本地 Windows 主机上执行；禁止在沙箱、容器、虚拟机或远程执行器中运行。
- 仅支持英特尔 AIPC 平台，任何操作前先做步骤一的平台检查。
- 修改后**需要重启电脑才会生效**。
- 查询不需要管理员权限；写入需要。
- PowerShell 和 Git Bash 两种写法都给出了，按当前可用的命令环境任选其一即可。

## 步骤一：平台检查（AIPC）

读取 CPU 型号：

```powershell
(Get-CimInstance -Query 'SELECT Name FROM Win32_Processor' | Select-Object -First 1).Name
```

Git Bash：

```bash
powershell.exe -NoProfile -Command "(Get-CimInstance -Query 'SELECT Name FROM Win32_Processor' | Select-Object -First 1).Name"
```

判定规则（与平台工具 `platform.ps1` 保持一致）：

1. 取型号字符串**最后一个空格之后**的片段作为型号代码（没有空格则取整个字符串）。
2. 型号代码首字符必须是数字，且**不能是 `1`**；否则不是 AIPC。
3. 满足第 2 条后，若**完整型号字符串**包含 `Ultra`（区分大小写），或型号代码首位数字 ≥ `3`，则为 AIPC。

例：`Intel(R) Core(TM) Ultra 7 268V` → 代码 `268V`，首字符 `2` 非 `1`，完整型号含 `Ultra` → 是 AIPC。

**不是 AIPC 就停止**，告知用户此技能仅支持英特尔 AIPC 平台，不要继续读写注册表。

## 步骤二：查询当前值

PowerShell：

```powershell
(Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\MemoryManager' -Name 'SystemPartitionCommitLimitPercentage' -ErrorAction SilentlyContinue).SystemPartitionCommitLimitPercentage
```

Git Bash（`MSYS2_ARG_CONV_EXCL='*'` **必须加**，否则 MSYS 会把 `/v` 等参数误转换成 Windows 路径）：

```bash
MSYS2_ARG_CONV_EXCL='*' reg.exe query 'HKLM\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\MemoryManager' /v SystemPartitionCommitLimitPercentage
```

结果解读：

- 有值 → 当前 GPU 内存限制为 *N*%。
- **值不存在** → 当前未设置，系统使用内置默认值。注意：这种状态**没有对应的数字**，请如实报告“未设置”，不要换算成某个百分比。
- `reg.exe` 以**十六进制**显示 REG_DWORD（例如 `0x39` 实为十进制 `57`），报告前必须换算成十进制；`Get-ItemProperty` 返回的已经是十进制。

可选：把百分比换算成实际容量，便于用户理解。

```powershell
[math]::Round((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB, 1)
```

## 步骤三：设置新值

### 1. 校验取值

- 只接受 13 – 87 之间的整数。
- 超出范围**直接拒绝并说明**（`只支持 13% - 87% 之间的值`），**不要**自动截断到边界值。
- 用户说“最大 / 调到最高”时按 87 处理，但写入前要把具体数字告诉用户。

### 2. 先读后写

先执行步骤二读取当前值并记录（包括“未设置”这一状态），然后报告将要发生的变化，例如 `当前 87% → 将设置为 50%`。记下原始状态，用户之后要求还原时需要用到。

### 3. 检查管理员权限

```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

- 返回 `True` → 直接执行 4a。
- 返回 `False` → 执行 4b 自动提权（会弹出 UAC 授权窗口）。

### 4a. 已有管理员权限时写入

```powershell
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\MemoryManager' -Name 'SystemPartitionCommitLimitPercentage' -Value 50 -Type DWord
```

Git Bash：

```bash
MSYS2_ARG_CONV_EXCL='*' reg.exe add 'HKLM\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\MemoryManager' /v SystemPartitionCommitLimitPercentage /t REG_DWORD /d 50 /f
```

若提示注册表项不存在，先创建该项再写入：

```powershell
New-Item -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\MemoryManager' -Force | Out-Null
```

### 4b. 无管理员权限时自动提权写入

```powershell
$p = Start-Process powershell -Verb RunAs -Wait -PassThru -ArgumentList '-NoProfile', '-Command', "Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\MemoryManager' -Name 'SystemPartitionCommitLimitPercentage' -Value 50 -Type DWord"
$p.ExitCode
```

- **必须带 `-Wait -PassThru`**：提权进程是独立进程，不带这两个参数就拿不到退出码，只能靠猜。
- 用户拒绝 UAC、或系统策略禁止提权时，`Start-Process` 会抛错。此时如实告知用户操作未完成，并建议以管理员身份重新启动终端后重试。

### 5. 写入后必须验证

**无论 4a 还是 4b，写入后都要重新执行步骤二的查询**，确认读回的值确实等于目标值。只有读回值匹配才能向用户报告成功；`ExitCode` 为 0 不等于值已写入。

验证通过后告知用户：设置已更改为 *N*%，**需要重启电脑后生效**。

## 恢复系统默认值

用户要求“改回去 / 恢复默认”时，取决于**修改前**的原始状态：

- 修改前**有具体数值** → 把该数值写回去（流程同步骤三）。
- 修改前**值不存在**（系统默认）→ 必须**删除**该值，而不是写入 87 或任何其他数字。写数字只会得到一个看起来像默认、实际不同的状态。

```powershell
Remove-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\MemoryManager' -Name 'SystemPartitionCommitLimitPercentage' -ErrorAction SilentlyContinue
```

Git Bash：

```bash
MSYS2_ARG_CONV_EXCL='*' reg.exe delete 'HKLM\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\MemoryManager' /v SystemPartitionCommitLimitPercentage /f
```

删除同样需要管理员权限，按步骤三的方式提权，并在之后重新查询确认该值已不存在。
