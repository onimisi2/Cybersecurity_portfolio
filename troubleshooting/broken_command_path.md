# Troubleshooting a Broken Command Path on Kali Linux: Sn1per Case Study

## Overview

After installing Sn1per (a reconnaissance and vulnerability scanning framework) on Kali Linux, the tool appeared to install successfully but could not be run from the terminal. This writeup documents the diagnostic process used to identify the root cause and the steps taken to permanently resolve it.

## Environment

- Operating System: Kali Linux
- Shell: Zsh
- Tool: Sn1per version 9.2

## Symptom

After installation completed without errors, running the tool produced:

```
sniper
sniper: command not found
```

## Diagnostic Process

### Step 1: Confirm the tool actually installed

Rather than assuming the installation failed, the first step was to search the file system for the actual program files.

```bash
locate sniper 2>/dev/null | grep bin
```

This returned only helper scripts inside `/usr/share/sniper/bin/`, not the main executable itself. A closer directory listing was needed.

```bash
ls -la /usr/share/sniper/
```

This confirmed the main executable was present at:

```
/usr/share/sniper/sniper
```

Conclusion: the installation had completed successfully. The problem was not a failed install.

### Step 2: Create a symbolic link into a standard binary directory

Most command line tools are expected to live in a folder that the shell automatically checks, such as `/usr/local/bin/`. A symbolic link (a shortcut) was created to point there:

```bash
sudo ln -sf /usr/share/sniper/sniper /usr/local/bin/sniper
sudo chmod +x /usr/local/bin/sniper
```

Running `sniper` still returned "command not found," which ruled out a missing symlink as the sole cause.

### Step 3: Inspect the shell's Path variable

The Path environment variable defines which folders the terminal searches when looking for a command. It was checked with:

```bash
echo $PATH
```

Result:

```
/usr/sbin:/usr/bin:/sbin:/bin
```

This was missing `/usr/local/bin` and `/usr/local/sbin` entirely, which explained why the symlink was never found even though it existed.

### Step 4: Trace where the Path variable is normally set

On a standard Kali installation, the Path variable should include `/usr/local/bin` by default. To find out why it did not, the relevant configuration files were checked in order of precedence:

```bash
cat /etc/environment
cat /etc/zsh/zshenv
cat /etc/zsh/zprofile
cat ~/.zshenv
cat ~/.zprofile
cat ~/.zshrc
```

The file `/etc/environment` contained the correct full path definition, but this file is only loaded automatically during certain login methods and was not being applied in this session.

The file `/etc/zsh/zshenv` contained a conditional repair rule intended to fix a broken Path variable automatically:

```bash
if [[ -z "$PATH" || "$PATH" == "/bin:/usr/bin" ]]
then
        export PATH="/usr/local/bin:/usr/bin:/bin:/usr/games"
fi
```

This rule only triggers if the Path variable is completely empty or exactly equal to `/bin:/usr/bin`. The actual Path variable in this environment was `/usr/sbin:/usr/bin:/sbin:/bin`, which matched neither condition, so the repair rule never activated. This was the root cause: a narrow conditional check that did not account for every possible broken state of the Path variable.

### Step 5: Apply a permanent fix

Rather than relying on the existing conditional logic, an unconditional line was appended to the same file so that the correct Path is always set for every shell session, regardless of what the Path variable looked like beforehand:

```bash
echo 'export PATH="/usr/local/sbin:/usr/local/bin:$PATH"' | sudo tee -a /etc/zsh/zshenv
```

A brand new terminal window was opened (not just a new command prompt in the same window) to ensure the shell configuration file was read fresh from the start.

```bash
echo $PATH
sniper --help
```

The corrected Path variable was confirmed, and the tool's help menu displayed correctly.

### Step 6: Resolve a secondary issue with elevated privileges

Sn1per requires root privileges to run scans. Running it with `sudo` still failed:

```bash
sudo sniper --help
sudo: sniper: command not found
```

This is because `sudo` uses its own separate, restricted path definition (called a secure path) that ignores the currently logged in user's personal Path variable. The fix was to switch to a full root shell instead of prefixing individual commands with `sudo`:

```bash
sudo -i
sniper --help
```

This loads the corrected Path variable properly for the root user and resolved the issue permanently.

## Root Cause Summary

| Layer | What was wrong | Why it mattered |
|---|---|---|
| Installation | Nothing — install completed fully | Ruled out reinstalling |
| Symbolic link | Missing initially | Needed for the shell to find the executable by name |
| Shell Path variable | Missing `/usr/local/bin` and `/usr/local/sbin` | The shell did not search the folder where the symlink lived |
| System configuration file | A conditional repair rule with too narrow a matching condition | The automatic fix never triggered for this specific broken Path state |
| Elevated privileges | `sudo` uses a separate, fixed path definition | The personal Path fix did not apply when using `sudo` directly |

## General Troubleshooting Steps for "Command Not Found" After Installation

1. Confirm the program files actually exist on disk before assuming installation failed.
   ```bash
   find / -iname "<tool name>" -type f 2>/dev/null
   ```
2. Check the current Path variable.
   ```bash
   echo $PATH
   ```
3. Create a symbolic link if the executable exists but is not linked into a standard binary directory.
   ```bash
   sudo ln -sf /full/path/to/real/file /usr/local/bin/<tool name>
   sudo chmod +x /usr/local/bin/<tool name>
   ```
4. If standard folders are missing from the Path variable, add them permanently at the system level.
   ```bash
   echo 'export PATH="/usr/local/sbin:/usr/local/bin:$PATH"' | sudo tee -a /etc/zsh/zshenv
   ```
5. Open a completely new terminal window to confirm the fix loads correctly on a fresh session.
6. If the tool requires elevated privileges, use a full root shell rather than prefixing commands with `sudo`, since `sudo` uses its own separate path definition.
   ```bash
   sudo -i
   <tool name> --help
   ```

## Key Takeaway

An installation failing to run is not always a sign that the installation itself failed. In this case, every file was present and correct — the actual fault was in how the shell located executable commands. Verifying file existence before troubleshooting the installer itself saved significant time and pointed directly to the real issue.
