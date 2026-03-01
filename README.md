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
│ penelope menu        │ Session 1 (PTY)     │ Session 2 (PTY)     │
│                      │ root@target1 #      │ www-data@target2 $  │
│ [+] Got shell...     │                     │                     │
│ [+] Got shell...     │                     │                     │
│ ➤ Main Menu          │                     │                     │
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

## Windows Improvements

- **Upload** is shell-aware (`psh`/`cmd`) and avoids `certutil`/`mshta`:
  - Small payloads: inline base64 + `Expand-Archive`
  - Large payloads: built-in `FileServer` + `Net.WebClient.DownloadFile()` + `Expand-Archive`
- **Download** is robust for repeated runs:
  - Builds archives via PowerShell `-EncodedCommand`
  - Uses begin/end markers for deterministic base64 extraction
  - Resolves relative paths against current remote CWD before archiving
- **`write_access()`** temp-file probe uses proper f-string interpolation.

## Security Hardening

Seven security vulnerabilities identified and fixed via code audit:

| ID | Vulnerability | Fix |
|----|--------------|-----|
| S1 | **Tar path traversal** — `tar.extractall()` allowed writing outside destination | `safe_tar_extractall()` validates each member via `os.path.realpath()`, rejects external symlinks |
| S2 | **Zip path traversal** — `zipdata.extract()` allowed writing outside destination | Realpath validation before extraction, skips malicious entries |
| S3 | **Arbitrary code exec via `eval()`** in `do_SET` command | Replaced with `ast.literal_eval()` |
| S4 | **RC file race condition** — `exec()` ran before permission check | `stat()` + `chmod` before `exec()`, secure `touch(mode=0o600)` on creation |
| S5 | **Shell injection** — `subprocess.run(shell=True)` in meterpreter module | Converted to argv list with `shell=False`, env dict, file redirect |
| S6 | **`os.system()` calls** — command injection surface via `reset`/`clear` | Replaced with `subprocess.run(["reset"])` / `subprocess.run(["clear"])` |
| S7 | **Port forwarding injection** — unvalidated rhost/rport interpolated into code | Hostname regex validation + port range (1-65535) enforcement |

## Bug Fixes

Nine bugs identified and fixed:

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
|Spawn shells on multiple tabs and/or hosts|✅|✅|❌|
|Maintain X amount of active shells per host no matter what|✅|✅|❌|
|Auto-split tmux panes on new sessions (-A)|✅|✅|✅|

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
