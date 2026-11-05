# claude-quota

claude-quota checks the quota of all your Claude accounts in one go. It also hides a magic trick that anyone juggling several accounts will love. Abracadabra. 🪄

## What works where

claude-quota works on macOS and on Windows 11. The commands are the same on both.

- On macOS it reads and writes Claude Code's login in the Keychain through `/usr/bin/security`. The background service uses launchd and notices are macOS notifications.
- On Windows it reads and writes Claude Code's login in `.credentials.json`. The background service is a Task Scheduler task and notices are Windows toasts.

It needs Claude Code logged in with a Claude subscription account. On Windows that can be the Claude Code command-line tool, or only the Claude extension for VS Code. The macOS steps are below. The Windows steps are in [Install on Windows](#install-on-windows).

## Install on macOS

1. Download the binary for your Mac from the [releases page](https://github.com/destngx/tools/releases), and `SHA256SUMS` from the same release. Use `darwin-arm64` for Apple silicon and `darwin-amd64` for Intel. Only binaries are published, not the source.
2. Check the download. Run this in the folder that holds both files. It prints `OK` for each file it finds, and the "No such file" lines for the other platforms are expected.

   ```sh
   shasum -a 256 -c SHA256SUMS
   ```

3. Make it executable and move it to `~/.local/bin` under the name `claude-quota`. Use the file name you downloaded.

   ```sh
   chmod +x claude-quota-VERSION-darwin-arm64
   mkdir -p ~/.local/bin
   mv claude-quota-VERSION-darwin-arm64 ~/.local/bin/claude-quota
   ```

4. Put `~/.local/bin` on your `PATH`. For zsh, add this line to `~/.zshrc` and open a new terminal.

   ```sh
   export PATH="$HOME/.local/bin:$PATH"
   ```

macOS may block a binary downloaded from the internet. If it does, run `xattr -d com.apple.quarantine ~/.local/bin/claude-quota` once. The hooks and the service call `~/.local/bin/claude-quota`, so keep the binary there.

## First run

claude-quota works on Claude Code's default setup: the `~/.claude` folder, `~/.claude.json` and the Keychain entry `Claude Code-credentials`. You don't need to set anything. If you keep Claude Code's files in another folder through `CLAUDE_CONFIG_DIR`, claude-quota follows that variable instead. In that case it must be set in every shell that runs claude-quota, and claude-quota refuses to run when its value changes. If you add or remove the variable later, run `claude-quota reconfigure` to move to the new setting.

1. Check the setup. `doctor` tells you what to fix. Warnings about the hooks and the service are normal before you do the steps below.

   ```sh
   claude-quota doctor
   ```

2. Add each account. Log in to the first account in Claude Code with `/login`, then save it. Repeat for every other account: run `/login` as that account, then `add` again. `claude-quota login` explains the steps and checks that the active account is saved.

   ```sh
   claude-quota add
   ```

3. Look at what you have.

   ```sh
   claude-quota list         # every saved account and its quota
   claude-quota switch       # switch to the account with the most quota left
   ```

4. Switch automatically when a session hits its limit.

   ```sh
   claude-quota hooks install
   ```

   This adds a `StopFailure` hook to `~/.claude/settings.json` and one line to your statusline script. The statusline must be a script file. Running the command twice changes nothing.

Run `claude-quota` with no arguments for the full command list.

## Background service

The service is optional. The `StopFailure` hook that `hooks install` adds switches accounts on its own. The service also checks quota in the background, so it can switch before the active account runs out.

To run it under launchd, save this as `~/Library/LaunchAgents/com.destngx.claude-quota.plist`. Replace `/Users/you` with your home directory.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.destngx.claude-quota</string>
  <key>ProgramArguments</key>
  <array>
    <string>/Users/you/.local/bin/claude-quota</string>
    <string>service</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <dict>
    <key>SuccessfulExit</key>
    <false/>
  </dict>
  <key>ThrottleInterval</key>
  <integer>60</integer>
  <key>ProcessType</key>
  <string>Background</string>
  <key>StandardOutPath</key>
  <string>/Users/you/Library/Logs/claude-quota.log</string>
  <key>StandardErrorPath</key>
  <string>/Users/you/Library/Logs/claude-quota.log</string>
</dict>
</plist>
```

There is no `CLAUDE_CONFIG_DIR` block because you use the default setup. If you set `CLAUDE_CONFIG_DIR` in your shell, add this inside the outer `<dict>`, with the same value as in your shell:

```xml
  <key>EnvironmentVariables</key>
  <dict>
    <key>CLAUDE_CONFIG_DIR</key>
    <string>/Users/you/path/to/folder</string>
  </dict>
```

Load it:

```sh
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.destngx.claude-quota.plist
```

Run `claude-quota doctor` again. It reports whether the service `com.destngx.claude-quota` is loaded.

### Read the log

The service writes to the file named in the plist. It prints a line when a pass fails, and nothing when all is well.

```sh
tail -f ~/Library/Logs/claude-quota.log
```

Notices such as a switch, or a saved login that is about to expire, are shown as macOS notifications and also appended to `~/Library/Application Support/claude-quota/notices.log`.

### Stop and remove

Stop the service and delete its plist:

```sh
launchctl bootout gui/$(id -u)/com.destngx.claude-quota
rm ~/Library/LaunchAgents/com.destngx.claude-quota.plist
```

Remove the hooks. This takes out only the lines `hooks install` added.

```sh
claude-quota hooks uninstall
```

To forget the accounts, remove each one. This deletes its saved login from the Keychain. Run `claude-quota list` to see the emails.

```sh
claude-quota remove EMAIL
```

Then delete the binary with `rm ~/.local/bin/claude-quota` and the folder `~/Library/Application Support/claude-quota`. Claude Code keeps its own login and keeps working.

## Install on Windows

Run these in PowerShell, one line at a time. Replace `VERSION` with the release you want, such as `v0.1.0`. The links below use `windows-amd64`, which is right for Intel and AMD processors. For an Arm laptop, use `windows-arm64` in the first and third lines.

1. Download the program and `SHA256SUMS` from the [releases page](https://github.com/destngx/tools/releases) into the current folder. Only binaries are published, not the source.

   ```powershell
   Invoke-WebRequest -Uri https://github.com/destngx/tools/releases/download/claude-quota/VERSION/claude-quota-VERSION-windows-amd64.exe -OutFile claude-quota-VERSION-windows-amd64.exe
   ```

   ```powershell
   Invoke-WebRequest -Uri https://github.com/destngx/tools/releases/download/claude-quota/VERSION/SHA256SUMS -OutFile SHA256SUMS
   ```

2. Check the download. This prints `OK` when the hash of the file matches the one in `SHA256SUMS`.

   ```powershell
   $f = 'claude-quota-VERSION-windows-amd64.exe'; if ((Get-FileHash $f -Algorithm SHA256).Hash -eq ((Select-String -Path SHA256SUMS -Pattern ([regex]::Escape($f))).Line -split '\s+')[0]) { 'OK' } else { 'MISMATCH, do not run it' }
   ```

3. Move it to `.local\bin` in your user folder under the name `claude-quota.exe`, and remove the download mark so Windows doesn't block it. The hooks and the service call `%USERPROFILE%\.local\bin\claude-quota.exe`, so keep it there.

   ```powershell
   New-Item -ItemType Directory -Force "$env:USERPROFILE\.local\bin" | Out-Null; Move-Item -Force claude-quota-VERSION-windows-amd64.exe "$env:USERPROFILE\.local\bin\claude-quota.exe"; Unblock-File "$env:USERPROFILE\.local\bin\claude-quota.exe"
   ```

4. Add that folder to your user `PATH`, then open a new PowerShell window.

   ```powershell
   $p = [Environment]::GetEnvironmentVariable('Path', 'User'); if (($p -split ';') -notcontains "$env:USERPROFILE\.local\bin") { [Environment]::SetEnvironmentVariable('Path', ("$p;$env:USERPROFILE\.local\bin").TrimStart(';'), 'User') }
   ```

### First run on Windows

claude-quota works on Claude Code's default setup, the `.claude` folder and `.claude.json` in your user folder. You don't need to set anything. If you keep Claude Code's files in another folder through `CLAUDE_CONFIG_DIR`, set it as a user variable so the background service sees it too, and claude-quota refuses to run when its value changes. Run `claude-quota reconfigure` after you add or remove the variable.

1. Check the setup. `doctor` tells you what to fix. Warnings about the hooks and the service are normal before the steps below.

   ```powershell
   claude-quota doctor
   ```

   With only VS Code and the Claude extension installed, there is no `claude` command. `doctor` says so, says it found the extension, and counts that as fine.

2. Add each account. In Claude Code or in the Claude extension in VS Code, run `/login` and sign in to the first account, then save it. Repeat for every other account: `/login` as that account, then `add` again. `claude-quota login` explains the steps and checks that the active account is saved.

   ```powershell
   claude-quota add
   ```

3. Look at what you have. After a switch, `/usage` and `/status` show the new email, and a running VS Code window follows without a reload.

   ```powershell
   claude-quota list
   ```

   ```powershell
   claude-quota switch
   ```

4. Switch automatically when a session hits its limit. This adds a `StopFailure` hook to `settings.json` in the `.claude` folder, and one line to your statusline script when you have one. With no statusline, only the hook is added, and that is enough. Running it twice changes nothing.

   ```powershell
   claude-quota hooks install
   ```

`check-version` and `upstream` decide whether a Claude Code update is safe. They need the Claude Code command-line tool, and say so when it isn't installed.

### Background service on Windows

The service is optional. The `StopFailure` hook switches accounts on its own. The service also checks quota in the background, so it can switch before the active account runs out.

This creates a Task Scheduler task named `claude-quota`. It starts at logon, runs in a hidden window and appends its output to `%LOCALAPPDATA%\claude-quota\service.log`. It runs as you and needs no administrator rights.

```powershell
$cmd = "& '$env:USERPROFILE\.local\bin\claude-quota.exe' service 2>&1 | Out-File -Append -Encoding utf8 '$env:LOCALAPPDATA\claude-quota\service.log'"; Register-ScheduledTask -TaskName claude-quota -Description 'claude-quota background service' -Action (New-ScheduledTaskAction -Execute powershell.exe -Argument "-NoProfile -WindowStyle Hidden -Command `"$cmd`"") -Trigger (New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME) -Settings (New-ScheduledTaskSettingsSet -ExecutionTimeLimit ([TimeSpan]::Zero) -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 1) -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries) -Force
```

Start it now without logging out, then run `claude-quota doctor` again. It reports whether the task is there and whether `CLAUDE_CONFIG_DIR` matches.

```powershell
Start-ScheduledTask -TaskName claude-quota
```

Read the log. The service prints a line when a pass fails, and nothing when all is well.

```powershell
Get-Content "$env:LOCALAPPDATA\claude-quota\service.log" -Wait
```

Notices such as a switch, or a saved login about to expire, show as Windows notifications and are also appended to `%LOCALAPPDATA%\claude-quota\notices.log`.

### Stop and remove on Windows

Stop the service and delete its task:

```powershell
Unregister-ScheduledTask -TaskName claude-quota -Confirm:$false
```

Remove the hooks. This takes out only what `hooks install` added.

```powershell
claude-quota hooks uninstall
```

To forget the accounts, remove each one. This deletes its saved login. Run `claude-quota list` to see the emails.

```powershell
claude-quota remove EMAIL
```

Then delete the program and the state folder. Claude Code keeps its own login and keeps working.

```powershell
Remove-Item "$env:USERPROFILE\.local\bin\claude-quota.exe", "$env:LOCALAPPDATA\claude-quota" -Recurse -Force
```

## License

MIT. See [LICENSE](LICENSE).
