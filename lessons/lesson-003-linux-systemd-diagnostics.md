# Lesson 003 — Diagnose a Linux systemd service safely

**Domain:** Linux operations / systemd / troubleshooting  
**Duration:** 30 minutes  
**Level:** Experienced DevOps / infrastructure engineer  
**Environment:** Local Debian 12/13 workstation or disposable Debian/Ubuntu VM; Bash  
**Version assumption:** Debian 12 Bookworm ships systemd 252; Debian 13 and newer Ubuntu releases may use newer versions. The commands in this lesson are standard systemd tooling.

> **Safety:** Use your local Debian workstation or a disposable VM, not your VPS or any production/shared server. This lesson is deliberately read-only with respect to running services. We will create and validate a unit file under your home directory, but we will not install, enable, start, stop, restart or reload any service.

## Objective

Practise a repeatable first-response workflow for a failed or unhealthy systemd service:

1. Identify its loaded and runtime state.
2. Read the correct journal entries for the current boot.
3. Extract useful structured properties such as `Result` and `ExecMainStatus`.
4. Validate a unit file offline before it is ever installed.

The key distinction: **a systemd unit being enabled is not the same as it currently being active**, and a service being active does not prove that the application behind it is healthy.

## 1. Prerequisites and environment check (4 minutes)

You need:
- A local Debian 12/13 or Ubuntu machine using systemd as its init system.
- A terminal and the `vim` editor (or another text editor).
- The commands `systemctl`, `journalctl` and `systemd-analyze`.

No additional package installation should be necessary on a normal Debian desktop/server installation.

Run:

```bash
cat /etc/os-release | head
ps -p 1 -o comm=
systemd-analyze --version
command -v systemctl journalctl systemd-analyze vim
```

Expected:
- `ps` reports `systemd` as PID 1.
- The three systemd commands and your editor have executable paths.

If PID 1 is not systemd, or these commands are unavailable, do not try to retrofit systemd into a container or your live VPS for this lesson. Use a disposable Debian/Ubuntu VM instead. Do not use WSL for this exercise unless you have deliberately configured it to boot systemd.

## 2. Create a disposable unit file (5 minutes)

We will keep every exercise file in:

`~/labs/tech-refresher/lesson-003-systemd/`

Create the directory and enter it:

```bash
mkdir -p ~/labs/tech-refresher/lesson-003-systemd
cd ~/labs/tech-refresher/lesson-003-systemd
pwd
```

The final line should show the directory above. This is your **lab directory**; it is not a systemd unit directory such as `/etc/systemd/system`.

Create a file named exactly `diag-demo.service` in this lab directory:

```bash
vim diag-demo.service
```

Paste the following complete content:

```ini
[Unit]
Description=Disposable systemd unit-file validation lab

[Service]
Type=oneshot
ExecStart=/usr/bin/true
User=nobody
Group=nogroup
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
```

Save and quit Vim with `:wq`.

What matters here:
- `[Unit]` describes the unit and its relationships.
- `[Service]` defines how a service process is run.
- `Type=oneshot` describes a command that runs and exits, rather than a long-running daemon.
- `ExecStart` is the executable systemd would run if this unit were started.
- The remaining directives demonstrate basic privilege and filesystem restrictions.

We will **not** start this unit. The executable `/usr/bin/true` is chosen because it is a harmless command that exits successfully.

Confirm the file exists and inspect it:

```bash
ls -l diag-demo.service
cat diag-demo.service
```

## 3. Validate the unit file offline (6 minutes)

Run:

```bash
systemd-analyze verify ./diag-demo.service
echo "exit=$?"
```

Expected: no diagnostic messages and an exit status of `0`.

This checks the unit-file syntax and references. It does not start the service, install the file, or prove that an application would behave correctly at runtime.

### Deliberately introduce an error

Make a copy so the known-good file stays untouched:

```bash
cp diag-demo.service diag-demo-broken.service
vim diag-demo-broken.service
```

In the copy only, change:

```ini
ExecStart=/usr/bin/true
```

to:

```ini
ExecStart=/usr/local/bin/not-a-real-command
```

Save and run:

```bash
systemd-analyze --recursive-errors=yes verify ./diag-demo-broken.service
echo "exit=$?"
```

Expected: systemd reports that the executable cannot be found or executed, and verification returns a non-zero status. Exact wording can vary by systemd release.

This is the value of offline validation: catch an obvious unit-file error before placing a file under `/etc/systemd/system` or asking the service manager to load it.

Remove only the intentionally broken copy:

```bash
rm diag-demo-broken.service
```

Keep `diag-demo.service` in your lab directory for reference.

## 4. Inspect a real service without changing it (9 minutes)

Still on your local Debian workstation or disposable VM, list failed units:

```bash
systemctl --failed --no-pager
```

Then list running services:

```bash
systemctl list-units --type=service --state=running --no-pager
```

Choose a **non-critical** service shown in that list. Do not choose a service whose output could expose sensitive application details. For the commands below, replace the example value with the actual unit name you selected:

```bash
UNIT=systemd-logind.service
```

If that example is not present on your machine, use a service that is listed. Now inspect it:

```bash
systemctl status "$UNIT" --no-pager -l
```

Read the important fields:
- **Loaded:** whether systemd found and loaded the unit file; the line also shows its file path and often whether it is enabled.
- **Active:** current runtime state, such as `active (running)`, `inactive (dead)` or `failed`.
- **Main PID:** the main process ID, if one exists.
- **Recent log lines:** useful initial context, but not necessarily the full history.

Next, ask systemd for specific properties:

```bash
systemctl show "$UNIT" \
  -p ActiveState \
  -p SubState \
  -p Result \
  -p ExecMainCode \
  -p ExecMainStatus \
  -p MainPID \
  -p Restart \
  -p NRestarts
```

This is better for scripts and structured diagnostics than parsing the human-oriented output of `systemctl status`.

Now inspect the unit's journal entries for this boot:

```bash
sudo journalctl -u "$UNIT" -b --no-pager -n 30 -o short-iso-precise
```

The `sudo` is for access to the system journal; it does not modify the service. If your account already has journal-reading permission, it may not be necessary.

Useful variations:

```bash
sudo journalctl -u "$UNIT" --since "30 minutes ago" --no-pager
sudo journalctl -u "$UNIT" -b -1 --no-pager -n 30
```

The first filters by time. The second looks at the previous boot, if those journal records are available. Some systems keep only volatile journal data, so previous-boot history may not exist.

## 5. Read the evidence in the right order (4 minutes)

Use this sequence during an incident:

1. `systemctl status UNIT` — get the current/most recent runtime state and a quick log sample.
2. `systemctl show UNIT -p Result -p ExecMainStatus ...` — extract structured failure information.
3. `journalctl -u UNIT -b` — inspect the service's complete relevant log history for this boot.
4. `systemctl cat UNIT` — inspect the unit definition and any drop-in overrides, without editing them.
5. `systemctl list-dependencies UNIT` — inspect units it depends on if the evidence points to a dependency problem.

Interpretation examples:
- `ActiveState=failed` with `Result=exit-code`: the service process exited unsuccessfully; inspect its journal and exit status.
- `ActiveState=inactive`: not automatically an error. A oneshot task may have completed normally.
- `enabled` in the Loaded line: configured to start through the relevant boot target; it does **not** mean it is currently running.
- `active (running)`: systemd sees the process as running; it does **not** establish that the application endpoint, database connection or business function is healthy.

Do not jump straight to `restart`. First establish what failed, when it failed, whether it is still failing, and whether a dependency or configuration change explains it.

## 6. Self-check and checkpoint (2 minutes)

Answer these without looking back:

1. What is the difference between `enabled` and `active`?
2. Which command shows human-readable service state and recent logs?
3. Which command filters journal history for one unit?
4. What does `ExecMainStatus` tell you?
5. What does `systemd-analyze verify` prove, and what does it not prove?

**Report this checkpoint when finished:**
- OS and systemd version;
- whether the valid unit passed verification;
- whether the deliberately broken copy was detected;
- the selected service's `ActiveState`, `SubState` and `Result` (omit the service name if you prefer);
- one useful diagnostic distinction you refreshed;
- confidence from 1–5.

Do not paste full logs, hostnames, private paths or sensitive service configuration. Do not mark the exercise complete until you have run the checks.

## Production / interview relevance

A disciplined first response separates **unit-manager state**, **process exit evidence**, **journal history** and **application health**. This avoids unnecessary restarts and gives incident responders concrete evidence for escalation or rollback. In an interview, explain why `systemctl status`, `systemctl show` and `journalctl` answer different questions.

## Official documentation

- [Debian 12 (Bookworm) — systemd-analyze(1)](https://manpages.debian.org/bookworm/systemd/systemd-analyze.1.en.html) — especially `verify` and `security`.
- [Debian 12 (Bookworm) — journalctl(1)](https://manpages.debian.org/bookworm/systemd/journalctl.1.en.html) — unit, boot, time and output filters.
- [Debian 12 (Bookworm) — systemctl(1)](https://manpages.debian.org/bookworm/systemd/systemctl.1.en.html) — service state and unit inspection.

## 30-minute plan

| Time | Activity |
|---:|---|
| 0–4 min | Check OS, PID 1 and systemd tools |
| 4–9 min | Create the lab directory and valid unit file |
| 9–15 min | Verify the good unit, introduce and detect one error |
| 15–24 min | Read-only inspection of a non-critical local service and its journal |
| 24–28 min | Interpret state, result and exit-status fields |
| 28–30 min | Answer the self-check and record the checkpoint |
| **Total** | **30 minutes** |
