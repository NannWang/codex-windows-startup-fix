# Fix Codex Desktop “Unable to load organization settings” on Windows (EPERM / `rename_staging`)

[**中文教程 / Chinese guide**](README.zh-CN.md)

> **Verified on one machine — October 23, 2026.** This is a community workaround, **not** an official OpenAI fix. The same dialog can have different causes. Check the logs before trying it.

## TL;DR — the fix that worked

Codex Desktop displayed **“Unable to load organization settings”** and blocked startup, even though ChatGPT worked in the browser. The desktop log showed:

```text
bundled_executable_relocation_failed
errorCode=EPERM
operation=rename_staging
executableName=codex.exe
```

After **fully closing Codex** and **renaming the local `bin` cache directory to a backup**, Codex launched normally without changing the proxy, reinstalling the app, or touching the `.codex` directory.

Run in **PowerShell**, as the normal Windows user, only if your logs show a similar error:

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

Then **relaunch Codex Desktop**. Keep the renamed backup until you've verified that your projects and sessions work.

**Important:** Do not delete `%USERPROFILE%\.codex`, and do not remove the newly generated `bin` directory. This procedure preserves the old `bin` directory by renaming it, but no workaround is guaranteed for every environment.

## Environment and symptoms

| Item | Observed |
| --- | --- |
| OS | Windows x64; exact OS build not recorded |
| App | Codex Desktop, installed via Microsoft Store / MSIX |
| Installed version during diagnosis | `26.1016.6573.0` |
| Proxy | Clash / Clash Verge; previously working settings were left unchanged |
| WSL | Not installed |
| Browser access | ChatGPT worked normally |
| Desktop startup | Blocked at “Unable to load organization settings”; Retry did not help |

The error appeared after the desktop client had previously worked. The message suggested checking the network, proxy, or firewall, but the evidence in this case pointed elsewhere.

## 1. Find the Codex Desktop logs

The directory that existed on this machine was:

```text
%LOCALAPPDATA%\Codex\Logs
```

To display the most recently modified log files:

```powershell
Get-ChildItem "$env:LOCALAPPDATA\Codex\Logs" -File -Recurse |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 10 LastWriteTime, Length, FullName |
    Format-List
```

To filter error-related lines from the most recent log:

```powershell
$f=Get-ChildItem "$env:LOCALAPPDATA\Codex\Logs" -File -Recurse | Sort-Object LastWriteTime -Descending | Select-Object -First 1; if($f){Select-String -Path $f.FullName -Pattern 'bundled_executable_relocation_failed|rename_staging|EPERM|source=missing|Desktop startup policy' | Select-Object -Last 30 | ForEach-Object {$_.Line}}
```

**Privacy:** logs may contain personal paths, emails, workspace metadata, or tokens. Redact them before sharing screenshots or excerpts publicly.

## 2. Relevant log evidence

These are the important fields from the affected machine's October 23 log; usernames, IDs, and stack traces are omitted:

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

The `rename` operation was trying to move a staged directory from this general pattern:

```text
%LOCALAPPDATA%\OpenAI\Codex\bin\.staging-<hash>-<random>
```

to:

```text
%LOCALAPPDATA%\OpenAI\Codex\bin\<hash>
```

A read-only check found that the expected hash-named target directory and `codex.exe` inside it were **both missing**. No matching staging directories remained at inspection time.

## 3. Checks performed before the successful workaround

| Check | Result |
| --- | --- |
| Browser access to ChatGPT | Worked |
| Renamed `%USERPROFILE%\.codex\config.toml` temporarily | **Did not fix startup** |
| Checked WSL | Not installed; no reason to install it for this incident |
| Checked MSIX app package | Package reported `Status: Ok` |
| Checked packaged runtime binaries | Original `codex.exe` (~342 MB) and companion executables existed in the installed application package |
| Checked expected local runtime directory | Missing |
| Created and renamed a temporary directory under the `bin` parent | Succeeded (`RENAME_OK`) |
| Renamed the existing `bin` cache directory to a timestamped backup | **Codex launched normally afterward** |

The ordinary rename test succeeding **does not** establish that runtime extraction and renaming cannot fail; an intermittent lock, security software, or a race during startup remains possible. The exact root cause was **not confirmed**.

## 4. Step-by-step recovery

1. **Fully close Codex Desktop.** In Task Manager, end remaining Codex processes if any. You can check PowerShell with `Get-Process *codex* -ErrorAction SilentlyContinue` (no output usually means none remain).
2. Open **PowerShell** under your normal Windows account.
3. Run the backup-and-rename command from the **TL;DR** section above.
4. Confirm it prints `BACKUP_OK: bin.backup-YYYYMMDD-HHMMSS`.
5. Relaunch Codex Desktop.
6. Verify the UI loads, projects/sessions are visible, and a simple non-destructive operation works.
7. **Keep the backup** until you're confident the application works normally; delete it only if you no longer need it.

**If the rename fails:** stop and inspect the specific access/lock error. Do not take ownership of `C:\Program Files\WindowsApps`, disable Windows security protections, or indiscriminately delete Codex data.

**If the same dialog remains:** inspect the **newest** desktop log after relaunch. The dialog is not specific to this error, and a different underlying issue may need a different fix.

## 5. Why this helped (and what we still don't know)

The packaged executable existed, but the app log recorded a failure during local executable relocation, followed by a missing-runtime message. Renaming the old user-local `bin` cache allowed startup to succeed on the next launch. This is consistent with a stale or conflicting local runtime cache, **not proof** of a specific Windows permission or antivirus bug.

The original error text suggested an organization-settings or network issue, so inspecting logs was essential to distinguish this case from actual login, policy, or connectivity failures.

## Related Codex reports

- [openai/codex#45858](https://github.com/openai/codex/issues/45858) — Windows MSIX startup/runtime relocation errors involving `rename_staging` and `EPERM`.
- [openai/codex#42484](https://github.com/openai/codex/issues/42484) — Similar staging-directory rename failures for bundled Node runtime (a **different component and symptom**).
- [openai/codex#51586](https://github.com/openai/codex/issues/51586) — The organization-settings dialog can also occur under different conditions, so **do not assume this workaround applies to every case**.

## Share your results

If this helped, consider starring this repository so other Windows users can find it. If it didn't, please include your **redacted** error keywords, Codex version, and result (without uploading complete unredacted logs).

---

*Personal troubleshooting record. Not affiliated with or endorsed by OpenAI.*