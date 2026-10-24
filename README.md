# Codex Desktop won't open on Windows: “Unable to load organization settings” / EPERM

[中文记录](README.zh-CN.md)

I ran into this on October 23, 2026. Codex Desktop had been working on my Windows PC, then one day it wouldn't get past the **“Unable to load organization settings”** dialog. Clicking Retry did nothing.

The message suggested a network or proxy problem. I use Clash / Clash Verge, but hadn't changed my settings, and ChatGPT still worked in the browser. After going through a few checks, I found something more useful in the local logs.

**What fixed it for me:** close Codex completely, rename its local runtime cache folder, then open Codex again. I'm documenting the details because the error dialog really doesn't point you toward this.

## The fix

**Only try this if your log shows the EPERM / rename_staging error below.** The same organization-settings popup can have other causes.

1. Close Codex. Check Task Manager and end any remaining Codex processes.
2. Open PowerShell as your normal Windows user.
3. Run:

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

I got:

~~~text
BACKUP_OK: bin.backup-20261023-163051
~~~

4. Start Codex again.

It opened normally on the next attempt. I didn't have to change my proxy, install WSL, or reinstall Codex.

The command **renames** the old cache instead of deleting it. Keep that backup until you've checked that your projects and sessions still work. Don't delete your user profile's **.codex** folder or the newly created **bin** folder.

## How I found the issue

On my machine, the Codex Desktop logs were in:

~~~text
%LOCALAPPDATA%\Codex\Logs
~~~

This shows the latest files:

~~~powershell
Get-ChildItem "$env:LOCALAPPDATA\Codex\Logs" -File -Recurse |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 10 LastWriteTime, Length, FullName
~~~

The log from the failed startup contained these lines (paths and IDs shortened):

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

Codex was trying to rename a staging directory into a final folder under:

~~~text
%LOCALAPPDATA%\OpenAI\Codex\bin\
~~~

That rename failed. When I checked the expected destination, both the hash-named directory and its **codex.exe** were missing.

## Other things I checked

For reference, here's what I tried before renaming the cache:

- ChatGPT worked in my browser. I left my existing Clash settings alone.
- Temporarily moving my **config.toml** out of the **.codex** folder didn't help.
- WSL wasn't installed. I didn't install it just to troubleshoot Codex.
- Windows reported the app package status as **Ok**.
- The installed version was **26.1016.6573.0**. The original packaged **codex.exe** and related executables were present.
- Creating and renaming a test directory under the runtime **bin** path worked (PowerShell returned **RENAME_OK**). So normal folder rename permission wasn't generally broken.
- But the executable was still missing from the local runtime cache after the failed launches.

This makes the cache/deployment step the likely trouble spot. I **don't know** exactly why Windows returned EPERM. It could have been a temporary file lock or another interference during startup; I haven't proved which one.

If the folder rename doesn't help, look at the *newest* log after the next failed launch rather than repeating this fix. Also, don't change permissions on the WindowsApps installation directory or turn off security protection just to force it through.


## Related reports

- [openai/codex#45858](https://github.com/openai/codex/issues/45858) — Windows runtime relocation failures with EPERM / rename_staging.
- [openai/codex#51586](https://github.com/openai/codex/issues/51586) — A similar organization-settings dialog, but not necessarily the same underlying cause.

That's everything I did in this case. Hopefully it saves someone else the time I spent checking the wrong things first.

*My own troubleshooting notes, not an official Codex fix.*
