# Codex Windows 客户端打不开，提示「无法加载组织设置」——我的排查和解决过程

[English version](README.md)

10 月 23 日，我的 Codex 客户端突然打不开了。之前一直能正常用，这次一启动就弹「无法加载组织设置」，点“重试”也没用。

弹窗让检查网络、代理或者防火墙，但我用的 Clash / Clash Verge 没改过配置，浏览器里的 ChatGPT 也正常。我一开始也在怀疑网络，后来翻了本地日志，才发现问题可能出在 Codex 自己的运行文件上。

**最后我用的办法很简单：完全退出 Codex，把本地 bin 缓存文件夹改名备份，再打开 Codex，就恢复正常了。**

把过程记下来，给遇到相同错误的人做个参考。

## 我是怎么解决的

先说一下：**建议确认日志里有 EPERM / rename_staging 再尝试**。光看「无法加载组织设置」这个弹窗，不能确定跟我的是同一种问题。

1. 先退出 Codex，在任务管理器里确认没有剩余的 Codex 进程。
2. 打开 PowerShell，不需要特意用管理员身份。
3. 执行下面的命令：

~~~powershell
$d = "$env:LOCALAPPDATA\OpenAI\Codex\bin"
if (Test-Path $d) {
    $backup = "bin.backup-$(Get-Date -Format 'yyyyMMdd-HHmmss')"
    Rename-Item -LiteralPath $d -NewName $backup -ErrorAction Stop
    Write-Output "BACKUP_OK: $backup"
} else {
    Write-Output "BIN_NOT_FOUND"
}
~~~

我当时的返回结果是：

~~~text
BACKUP_OK: bin.backup-20261023-163051
~~~

然后重新打开 Codex，就能进去了。没有改代理，也没有重装客户端。

这里的操作是**改名备份，不是直接删除**。旧的 bin 文件夹建议先留着，确认项目和会话都正常后再决定要不要清理。不要误删用户目录下的 **.codex**，也不要删新生成的 **bin** 文件夹。

## 日志里发现了什么

我电脑上的日志实际保存在：

~~~text
%LOCALAPPDATA%\Codex\Logs
~~~

PowerShell 可以这样查最新的日志：

~~~powershell
Get-ChildItem "$env:LOCALAPPDATA\Codex\Logs" -File -Recurse |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 10 LastWriteTime, Length, FullName
~~~

10 月 23 日的启动日志反复出现下面这些内容（我把个人路径和无关信息省略了）：

~~~text
warning bundled_executable_relocation_failed
errorCode=EPERM
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
~~~

我理解的是：Codex 启动时要把自带的可执行文件放到本地缓存目录，中间有一步重命名操作被 Windows 拒绝了。

目标位置在：

~~~text
%LOCALAPPDATA%\OpenAI\Codex\bin\
~~~

我检查的时候，预期的那个版本目录不存在，里面的 **codex.exe** 也没有，说明本地运行文件确实没有成功部署进去。

## 之前折腾过哪些地方

怕大家跟我一样绕弯路，把试过的也写一下：

- **网络：** ChatGPT 网页能用。Clash 还是原来的配置，我没有切全局模式。
- **配置文件：** 临时把用户目录里 **.codex** 下的 **config.toml** 改名隔离，没效果。
- **WSL：** 我的电脑根本没安装 WSL，这次也没有为了解决问题去装。
- **安装状态：** Windows 查询到 Codex 包状态为 **Ok**。
- **客户端版本：** 排查时已经是 **26.1016.6573.0**。安装包里能找到原始的 **codex.exe** 和几个配套程序。
- **文件夹权限：** 我在运行缓存目录下新建了测试文件夹，再重命名，返回 **RENAME_OK**。普通目录改名没问题，但 Codex 启动时还是报 EPERM。

所以我现在只能确定：**这次故障跟本地运行组件部署失败有关，重建 bin 缓存后恢复了。** 至于最开始为什么会出现 EPERM，是文件被占用、权限冲突还是别的原因，我没有查到可以确定的证据。

如果你按照这个方法操作后还是打不开，建议再看一次最新启动日志。相同弹窗不一定代表相同原因，不建议直接删整个 Codex 数据目录，也没必要为了这个错误去修改 WindowsApps 权限或者关闭系统防护。

另外，上传日志或截图到 GitHub 前记得把电脑用户名、邮箱、令牌等个人信息处理掉。

## 相关 Issue

- [openai/codex#45858](https://github.com/openai/codex/issues/45858)：Windows 上类似的 EPERM / rename_staging 故障。
- [openai/codex#51586](https://github.com/openai/codex/issues/51586)：也是「无法加载组织设置」，不过不一定跟我的原因一样。

以上就是我这次从报错到恢复的整个过程，希望能帮到碰上同样问题的人。

*个人排障记录，不是官方解决方案。*
