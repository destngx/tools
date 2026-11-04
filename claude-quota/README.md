# claude-quota

claude-quota checks the quota of all your Claude accounts in one go. It also hides a magic trick that anyone juggling several accounts will love. Abracadabra. 🪄

## What works where

claude-quota works on macOS only. It reads and writes Claude Code's login in the macOS Keychain through `/usr/bin/security`, and its background service and notices use launchd and macOS notifications.

It needs Claude Code installed and logged in with a Claude subscription account.

## Install

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

## License

MIT. See [LICENSE](LICENSE).
