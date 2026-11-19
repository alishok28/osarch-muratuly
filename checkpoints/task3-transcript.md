# Task 3 — Diagnose: `office-srv`


## Transcript

| # | Evidence file (command) | What I expected | What I found | What I concluded |
|---|---|---|---|---|
| 1 | 01 (hostnamectl), 02 (uptime) | An ordinary Ubuntu server; get context before looking at the fault | Ubuntu 24.04.1 LTS, x86-64, a VirtualBox VM. `up 6 days, 23:51`, load average 0.00 | The machine is up and idle. It was last rebooted about 7 days ago, on Nov 9, which matches "last week". Context only. |
| 2 | 03 (df -h) | A full disk is the classic reason a log stops growing | `/` is 12G, 48% used, 5.8G free. `/boot` 12%. No filesystem is full | **Dead end:** not a disk-space problem. |
| 3 | 04 (free -h) | Memory exhaustion is another cause of silently dying jobs | 1.9Gi total, 1.5Gi available, swap 0B used | **Dead end:** not a memory problem. |
| 4 | 05 (systemctl --failed) | `nightly-report` failing | Two failed units: `legacy-sync.service` and `nightly-report.service` | Two leads. Check which one relates to the ticket. |
| 5 | 10 (systemctl status legacy-sync) | It might be the cause of the missing report | Description says "old office, no longer used". `ExecStart=/bin/false` (exits 1 by design). Enabled, failed on Nov 9 09:51 | It is built to fail. It has nothing to do with the report. |
| 6 | 11 (journalctl -u legacy-sync) | Failures starting around the script update | Failures on Sep 14, Oct 02, Oct 19 and Nov 09, once per boot | **Dead end:** it has been failing since September, long before the ticket. Not the cause. Noted as unrelated clutter. |
| 7 | 06 (systemctl list-timers) | Maybe the timer is dead, so the job never starts | `nightly-report.timer` is active: last run 12s ago, next in 1min 47s. All other timers normal | **Dead end:** the scheduler works. The accountant is right that the job fires every two minutes. The fault is in what the timer starts. |
| 8 | 08 (systemctl cat service and timer) | A wrong path or a typo in the unit | `Type=oneshot`, `ExecStart=/usr/local/bin/nightly-report.sh`. Timer: `OnBootSec=1min`, `OnUnitActiveSec=2min`, described "while being tested" | The unit is fine, and its path matches the file name. The 2-minute interval is a test setting (follow-up, not the fault). |
| 9 | 07 (systemctl status nightly-report) | Probably "Permission denied" | `status=203/EXEC`: `Failed to execute /usr/local/bin/nightly-report.sh: No such file or directory`, CPU 2ms | systemd could not even start the program. Not a permission message, and it says the file is missing. Check the file. |
| 10 | 13 (ls -l /usr/local/bin) | File missing, as the error says | The file **exists**: `-rwxr-xr-x root root 91 bytes`, mtime `Nov 9 14:20` | Contradiction: the file is there and executable. So "No such file" must refer to something the file points to. The mtime 14:20 is a lead. |
| 11 | 09 (journalctl -u nightly-report) | Failures starting with the Nov 16 run | Success every 2 min up to `14:18:01` on Nov 9 (`Deactivated successfully`). First failure `14:20:01`, then every run since | It broke between 14:18 and 14:20 on Nov 9, the minute the script file was changed. This matches the ticket. |
| 12 | 21 (apt history) | The Nov 9 upgrade of bash/systemd/openssh broke it | `apt full-upgrade` 09:38–09:41 upgraded kernel, systemd, bash, openssh | **Dead end:** tempting, but the report ran fine at 14:12–14:18, hours after the upgrade and reboot (09:50). Not the cause. |
| 13 | 12 (journalctl -p err -b) | Some deeper error behind the exec failure | `vboxguest` and `vmwgfx` kernel lines at 09:50–09:51, the legacy-sync failure, and the same exec error for nightly-report | **Dead end:** the VirtualBox/graphics lines are VM noise, happen once at boot, nothing ties them to the report. No new clue. |
| 14 | 14 (ls -l /var/log) | A log not owned or writable by root | `nightly-report.log` is `-rw-r--r-- root`, mtime `Nov 9 14:18`, 86412 bytes. Other logs updated Nov 16 | Log permissions are fine. It stopped growing exactly at the last good run. |
| 15 | 15 (tail nightly-report.log) | Recent entries | Last entry `Mon Nov 9 14:18:01 UTC 2026`, then nothing | Confirms the ticket: no writes since 14:18 on Nov 9. |
| 16 | 16 (cat nightly-report.sh) | A syntax error or wrong path | Three normal lines: `#!/bin/bash`, a comment, `{ date; df -h /; free -h; } >> /var/log/nightly-report.log` | **Dead end:** content looks perfect. Something invisible must be wrong. |
| 17 | 19 (run by hand) | The script's own error message | `bash: ... cannot execute: required file not found`; with sudo: `unable to execute ...: No such file or directory` | Same failure as root, so not permissions. "Required file not found" is how bash reports a **missing interpreter** from the `#!` line. |
| 18 | 20 (which bash; ls -l /bin/bash) | Bash was removed or replaced in the upgrade | `/usr/bin/bash` and `/bin/bash` exist, `-rwxr-xr-x`, 1446024 bytes | The interpreter exists. So the shebang must name something other than `/bin/bash`. |
| 19 | 18 (head -1 \| od -c) | `#!/bin/bash\n` | `# ! / b i n / b a s h \r \n`: the first line ends in `\r\n` | The kernel looks for an interpreter literally named `/bin/bash\r`. That does not exist, so it returns "No such file". |
| 20 | 17 (cat -A) | The same carriage return on other lines | Every line ends with `^M$`. Plain `cat` (row 16) hid it | The whole file has **Windows (CRLF) line endings**. |
| 21 | 22 (bash history, aigerim) | How the file got there | After the upgrade and reboot: `sudo cp ~/nightly-report.sh /usr/local/bin/nightly-report.sh`, `chmod +x`, `start`, `status`, `ls -l`, `chmod 755`, `start`, `status`, `exit` | Explains the change at 14:20: the script was copied from her home directory. She tried `chmod` twice, chasing the misleading "No such file" message as a permissions problem. It did not help (**dead end**), because the mode was already `755`. |
| 22 | not in pack: `file /usr/local/bin/nightly-report.sh` | Would show `... ASCII text executable, with CRLF line terminators` | Not collected | Would confirm row 20 in one line. |
| 23 | not in pack: `cat -A ~/nightly-report.sh` | Would show the source copy has `^M$` too | Not collected (`ls -l ~` output is also missing) | The original in `/home/aigerim` is very likely CRLF too. It must be fixed as well, or a re-copy brings the fault back. |

## Root cause

`/usr/local/bin/nightly-report.sh` was replaced on Nov 9 at 14:20 with a copy that has Windows (CRLF) line endings, so its shebang is `#!/bin/bash\r`. The kernel cannot find an interpreter called `/bin/bash\r` and systemd fails the unit with `203/EXEC` every two minutes. Proof: files 18 (`\r\n` in the first line) and 17 (`^M$` on every line), the failing exec in 07/09 that starts at 14:20:01 (the same minute as the file's mtime in 13), and the working script before that (09, 15).

## Fix

On the server, in this order:

```bash
# 1. keep a copy of the broken file
sudo cp -p /usr/local/bin/nightly-report.sh /root/nightly-report.sh.crlf.bak

# 2. strip the carriage returns (sed -i keeps owner and mode 755)
sudo sed -i 's/\r$//' /usr/local/bin/nightly-report.sh

# 3. fix the source copy too, so it is not re-installed broken (run as aigerim)
sed -i 's/\r$//' ~/nightly-report.sh

# 4. clear the failed state and run the job now
sudo systemctl reset-failed nightly-report.service
sudo systemctl start nightly-report.service
```

No `chmod` is needed: the mode was already `755`.

## Verification

**1. The file is clean**

```bash
cat -A /usr/local/bin/nightly-report.sh
head -1 /usr/local/bin/nightly-report.sh | od -c
ls -l /usr/local/bin/nightly-report.sh
```

Expected: lines end with `$` only (no `^M`). `od -c` shows `# ! / b i n / b a s h \n`, with the end offset `0000014` instead of `0000015`. `ls -l` shows `-rwxr-xr-x root root 88` (it was 91; 3 lines lose one `\r` each).

**2. Run by hand**

```bash
sudo /usr/local/bin/nightly-report.sh; echo "exit code: $?"
tail -n 8 /var/log/nightly-report.log
```

Expected: `exit code: 0` and a new block with the current date, the `df -h /` table and the `free -h` output.

**3. Through systemd**

```bash
systemctl status nightly-report.service
systemctl --failed
```

Expected: `Active: inactive (dead)`, `ExecStart=... (code=exited, status=0/SUCCESS)`, `Finished nightly-report.service`. `systemctl --failed` lists only `legacy-sync.service` (unrelated, row 6).

**4. The scheduler keeps writing every two minutes** (wait about 5 minutes)

```bash
journalctl -u nightly-report.service --since "10 min ago" --no-pager
tail -n 12 /var/log/nightly-report.log
```

Expected: a `Starting / Deactivated successfully / Finished` triple every 2 minutes (like the 14:12–14:18 lines in file 09) and new timestamps in the log.

**5. It still works after a reboot**

```bash
sudo reboot
# log in again after about 2 minutes
uptime                                              # up 1-2 min
systemctl is-enabled nightly-report.timer           # enabled
systemctl list-timers nightly-report.timer          # LAST is a few minutes ago, NEXT within 2 min
tail -n 12 /var/log/nightly-report.log              # timestamps later than `uptime -s`
head -1 /usr/local/bin/nightly-report.sh | od -c    # still ends in \n, no \r
```

Expected: the timer starts by itself 1 minute after boot (`OnBootSec=1min`), new log blocks appear after the boot time, and the file on disk stays clean. The timer was already running after the Nov 9 reboot (file 09), so it is enabled.

## Follow-ups (not part of the fix)

- `legacy-sync.service` is "no longer used" and fails at every boot. If the office agrees, `sudo systemctl disable legacy-sync.service`. I did not touch it, because it is not part of this ticket.
- The timer is a test setting (every 2 minutes, file 08), and the log will grow quickly. Once the report is confirmed, change it to a real nightly schedule with `OnCalendar=`.
- Prevention: save scripts with LF endings (editor setting or `dos2unix`), and check new scripts with `cat -A` or `file` before installing them.
