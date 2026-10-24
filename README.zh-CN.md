# Windows Codex 客户端提示「无法加载组织设置」：EPERM / rename_staging 修复记录

[English README](README.md)

> **实际修复成功的个人案例（2026 年 10 月 23 日）**。这不是 OpenAI 官方修复方案。相同的界面报错可能由不同原因引起，请先看日志再操作。

## 一句话解决方案

Windows 版 Codex Desktop 原来可以使用，后来打开时始终提示「无法加载组织设置」。ChatGPT 网页可以正常访问，Clash 代理配置也没有改动。

检查日志，发现：

```text
bundled_executable_relocation_failed
errorCode=EPERM
operation=rename_staging
executableName=codex.exe
```

**最后有效的操作：完全退出 Codex，把本地 `bin` 运行缓存目录改名备份，重新启动 Codex。** 实际测试后，客户端恢复了正常启动。没有修改代理、重装软件或安装 WSL。

在普通 Windows PowerShell 中执行：

```powershell
$d = "$env:LOCALAPPDATA\OpenAI\Codex\bin"
if (Test-Path $d) {
    $backup = "bin.backup-$(Get-Date -Format 'yyyyMMdd-HHmmss')"
    Rename-Item -LiteralPath $d -NewName $backup -ErrorAction Stop
    Write-Output "BACKUP_OK: $backup"
} else {
    Write-Output "BIN_NOT_FOUND"
}
```

看到 `BACKUP_OK: bin.backup-<时间戳>` 后，再打开 Codex。

**注意：** 只重命名旧的 `bin` 缓存，不删除 `%USERPROFILE%\.codex`。保留原备份，先确认工作区、历史会话和功能都正常。

## 1. 故障现象和环境

| 项目 | 本次实际情况 |
| --- | --- |
| 系统 | Windows x64，具体系统构建号未记录 |
| 客户端 | Microsoft Store 安装的 Codex Desktop（MSIX） |
| 排查时版本 | `26.1016.6573.0` |
| 弹窗 | 「无法加载组织设置」，点击重试无效 |
| 网页 | ChatGPT 浏览器端正常 |
| 代理 | Clash / Clash Verge，沿用之前正常使用时的配置 |
| WSL | 未安装 |

报错文案提到网络、代理、防火墙，但**这个案例的关键异常不在弹窗文案中，而在本地日志中**。

## 2. 找到真正的错误日志

实际存在的日志目录：

```text
%LOCALAPPDATA%\Codex\Logs
```

在 PowerShell 中查看最近的日志文件：

```powershell
Get-ChildItem "$env:LOCALAPPDATA\Codex\Logs" -File -Recurse |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 10 LastWriteTime, Length, FullName |
    Format-List
```

从最新日志里筛选关键异常：

```powershell
$f=Get-ChildItem "$env:LOCALAPPDATA\Codex\Logs" -File -Recurse | Sort-Object LastWriteTime -Descending | Select-Object -First 1; if($f){Select-String -Path $f.FullName -Pattern 'bundled_executable_relocation_failed|rename_staging|EPERM|source=missing|Desktop startup policy' | Select-Object -Last 30 | ForEach-Object {$_.Line}}
```

**注意隐私：** 日志可能包含用户名、本地目录、邮箱、令牌等信息，公开之前要脱敏，不要上传完整的原始日志。

### 实际发现的关键日志

```text
warning bundled_executable_relocation_failed
errorCode=EPERM
errorName=BundledExecutableRelocationError
operation=rename_staging
executableName=codex.exe
originalError={"errno":-4048,"code":"EPERM","syscall":"rename",...}

info windows_core_runtime_component_resolved
component=codex
executablePath=null
source=missing

warning Desktop startup policy read failed
attempt=1
phase=preparation
reason=unavailable
retrying=false
```

这意味着：Codex 在尝试把自带的运行程序从暂存目录移动到最终缓存目录时，遇到了 Windows `EPERM` 错误。随后又提示本地运行程序未找到。

预期的目录结构为：

```text
%LOCALAPPDATA%\OpenAI\Codex\bin\.staging-<哈希>-<随机后缀>
                        ↓
%LOCALAPPDATA%\OpenAI\Codex\bin\<哈希>
```

在本次排查中，预期的目标目录不存在，其中的 `codex.exe` 也不存在；检查时没有残留对应的暂存目录。

## 3. 我已经尝试过的排查

| 操作 | 结果 |
| --- | --- |
| 浏览器访问 ChatGPT | 正常 |
| 临时隔离 `%USERPROFILE%\.codex\config.toml` | 无效 |
| 查看 WSL | 未安装，不需要为此强行安装 |
| 查询 MSIX 安装状态 | `Ok` |
| 检查安装包 | 原始 `codex.exe` 和多个配套运行文件存在 |
| 检查本地运行目录 | `codex.exe` 缺失 |
| 测试普通目录的创建和重命名 | `RENAME_OK` |
| **改名备份旧 `bin` 目录后重启** | **恢复正常启动** |

注意：普通目录可以重命名，不代表运行时的暂存目录一定不会被进程占用或拦截。最初触发 `EPERM` 的具体原因仍未确认。

## 4. 具体操作步骤

**第 1 步：完全退出 Codex。** 打开任务管理器，结束确认属于 Codex 的进程。可运行 `Get-Process *codex* -ErrorAction SilentlyContinue` 检查是否仍有相关进程。

**第 2 步：打开 Windows PowerShell。** 使用日常账号执行，不必额外提升管理员权限。

**第 3 步：运行前文的备份命令。** 命令只将 `%LOCALAPPDATA%\OpenAI\Codex\bin` 重命名为带时间戳的 `bin.backup-...`，**不会直接删除旧目录**。

**第 4 步：确认输出。** 我当时返回的形式是：

```text
BACKUP_OK: bin.backup-20261023-163051
```

**第 5 步：重新打开 Codex。** 本案例中，客户端成功进入主界面。

**第 6 步：检查项目和历史会话。** 建议先尝试一个不会修改文件的简单操作。旧 `bin.backup-...` 保留几天，确认一切正常后再考虑清理。

如果命令报拒绝访问或重命名失败，应先查文件占用/权限；不要强行修改 `C:\Program Files\WindowsApps` 权限，也不建议关闭系统防护。

如果改名缓存后仍然出现同样弹窗，请重新读取**最新一次启动**的日志。这个弹窗还有其他可能原因，不能保证此方法对所有人都有效。

## 5. 技术上的理解

本次安装包里有 `codex.exe`，但日志显示本地部署阶段执行目录重命名失败，随后运行组件状态为 `source=missing`。改名备份本地 `bin` 缓存后，下一次启动恢复成功，因此**旧缓存或组件部署流程很可能与问题有关**。

但没有证据能进一步认定一定是杀毒软件、文件系统权限或客户端更新竞态，因此不应把这些猜测当成已证实的根因。

## 参考讨论

- [openai/codex#45858](https://github.com/openai/codex/issues/45858)：Windows 版本遇到类似的 `rename_staging` / `EPERM` 运行组件部署异常。
- [openai/codex#42484](https://github.com/openai/codex/issues/42484)：捆绑 Node 组件有相似的目录重命名问题，但组件和表现不完全相同。
- [openai/codex#51586](https://github.com/openai/codex/issues/51586)：相同的组织配置弹窗在其他环境中也可能由不同问题引起。

如果这篇记录帮你解决了问题，欢迎给仓库点个 **Star**，让其他遇到相同故障的人更容易找到。

---

*社区个人排障记录，与 OpenAI 官方无关联。*