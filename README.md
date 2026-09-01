# Self-Healing Watchdog

A script that watches your important services (nginx, MySQL, Redis,
PHP-FPM, etc.) and Supervisor jobs, and automatically restarts anything
that crashes — then tells you (Slack / Google Chat / Email) if it
couldn't fix it on its own.

Works on **Debian/Ubuntu** and **RHEL/CentOS/Fedora/AlmaLinux/Rocky**.

---

## 1. What it actually does (in plain terms)

Think of it as a guard that checks on your services every 30 seconds:

- **Service is fine** → does nothing, just logs `OK`.
- **Service is down** → tries restarting it, up to 3 times, 30 seconds
  apart, checking health after each try.
- **Still down after 3 tries** → gives up retrying (so it doesn't loop
  forever) and sends you an alert saying "this needs a human."
- **Service comes back healthy later** (on its own, or because you fixed
  it) → automatically notices, resets everything, and sends you a
  "recovered" alert. It's ready to try again from scratch next time it
  breaks.

It also keeps a running log of everything it does, rotates that log so
it doesn't fill your disk, and shows a quick health summary every time
someone logs into the server.

---

## 2. Requirements

- Run as **root** (`sudo`).
- `systemctl` and `flock` must already be on the system (standard on
  any modern Linux) — the script stops immediately if either is
  missing.
- `curl` is auto-installed if missing (needed for Slack/Google Chat and
  HTTP health checks).
- If you want email alerts, the script installs a mail sender for you
  (`mailutils` on Debian/Ubuntu, `mailx` on RHEL-based).

---

## 3. Installing it

**Easiest way — just run it and answer the prompts:**

```bash
sudo ./setup-self-healing.sh
```

It'll ask you things like "any extra services to watch?", "Slack
webhook?", "how many restart attempts?" — press Enter on any question
to accept the sensible default shown.

**Or skip the questions** by passing options directly (useful for
automation/repeat installs):

```bash
sudo ./setup-self-healing.sh \
  --googlechat-webhook "https://chat.googleapis.com/v1/spaces/.../messages?key=...&token=..." \
  --max-attempts 3 --retry-delay 30 --check-interval 30 --cooldown 1800
```

**You can re-run this anytime** — it's safe. It merges new settings
into what's already configured instead of wiping it out. Use this to
add a service later, change your webhook, or adjust timing.

---

## 4. All the options

| Option | What it does | Default |
|---|---|---|
| `--services "svc1,svc2"` | Watch extra services by name, even if they're not running right now. | — |
| `--http-check "svc=URL"` | Check a service by hitting a URL instead of the default check. | — |
| `--tcp-check "svc=PORT"` | Check a service by testing a TCP port instead of the default check. | — |
| `--slack-webhook URL` | Send alerts to a Slack channel. | — |
| `--googlechat-webhook URL` | Send alerts to a Google Chat space. | — |
| `--email-to ADDR` | Send alerts to this email address. | — |
| `--email-from ADDR` | "From" address on alert emails. | `self-healing@<hostname>` |
| `--max-attempts N` | How many restarts to try before giving up. | `3` |
| `--retry-delay SECONDS` | How long to wait between restart attempts. | `30` |
| `--check-interval SECONDS` | How often to check services when everything's healthy. | `30` |
| `--cooldown SECONDS` | Minimum gap between repeat "still down" alerts for the same service (see [notification cooldown](#6-notifications)). | `1800` (30 min) |
| `--dry-run` | Show what it *would* do, without changing anything. | — |
| `--uninstall` | Remove everything this script installed. | — |
| `-h`, `--help` | Show the built-in help text. | — |

---

## 5. Which services get watched, and how

Every time you run the script, it looks for a standard list of common
services (nginx, apache2, MySQL/MariaDB, Redis, PHP-FPM, Meilisearch,
cron, Supervisor) and only picks the ones that are **actually installed
and currently running** — plus anything you added with `--services`.

If you run it interactively, it asks you to confirm each one it found
(`Y/n`) before adding it.

**How it checks each service is healthy** — set per service in
`/etc/self-healing/services.conf`:

| Check type | Confirms |
|---|---|
| `none` | The process is running. That's it. |
| `tcp` | The process is running **and** a specific port accepts connections. |
| `http` | The process is running **and** a URL responds successfully. |
| `supervisor` | A Supervisor job shows as `RUNNING`. |

The script guesses reasonable default ports for well-known services
(nginx/apache2 → 80, MySQL/MariaDB → 3306, Redis → 6379, Meilisearch →
7700). **These are just guesses** — if your service actually runs on a
different port (e.g. an HTTPS-only nginx that only listens on 443), the
check will fail even though the service is perfectly fine. Fix it by
editing the line for that service in `/etc/self-healing/services.conf`:

```
nginx:tcp:443
```

then apply the change:

```bash
sudo systemctl restart self-healing-watchdog.service
```

To add a totally custom app, add a line the same way:

```
myapp:http:http://127.0.0.1:8080/health
```

---

## 6. Notifications

When a service gives up after 3 failed restarts — or recovers after
having given up — you get an alert like this:

```
❌ CRITICAL: *nginx* on *rhel.faveodemo.com*
Status: *DOWN*
Restart attempts: 3/3
Time: 2026-09-01 10:56:44 EDT
Detail: substate=failed result=exit-code recent_log: ...
```

```
✅ RESOLVED: *nginx* on *rhel.faveodemo.com*
Status: *RECOVERED*
Restart attempts: 0/3
Time: 2026-09-01 11:05:02 EDT
Detail: Service is healthy again ...
```

- ✅ = recovered, ❌ = down — easy to scan at a glance.
- Text wrapped in `*asterisks*` shows up **bold** in Slack and Google
  Chat. In plain email it just shows the asterisks as-is.
- The email subject line also has the emoji, e.g.
  `[self-healing] ❌ nginx on rhel.faveodemo.com — DOWN`, so you can see
  it's urgent without opening the email.

**Why you might not get an alert every single time a service flaps:**
if a service keeps going down and recovering repeatedly, you'd get
spammed with "still down" alerts. So repeat DOWN alerts for the *same*
service are limited to once every `--cooldown` seconds (default 30
min). **Recovery alerts are never limited** — you'll always hear about
a recovery, and that's actually what resets the cooldown so the *next*
real outage always alerts you too.

If an alert gets skipped for this reason, it's still recorded in the
log:
```
[2026-09-01 13:55:27] Notification for nginx (DOWN) suppressed (cooldown active).
```

---

## 7. Where everything lives

| Path | What it is |
|---|---|
| `/etc/self-healing/config.conf` | Your webhooks/email + timing settings. |
| `/etc/self-healing/services.conf` | Which services are watched and how (`name:checktype:target`). Edit by hand to add/adjust services. |
| `/usr/local/bin/self-healing-watchdog.sh` | The daemon that does the checking/restarting. |
| `/usr/local/bin/self-healing-notify.sh` | Sends the actual Slack/Google Chat/Email alerts. |
| `/etc/systemd/system/self-healing-watchdog.service` | The systemd service definition (auto-restarts itself if it ever dies). |
| `/etc/logrotate.d/self-healing` | Log rotation rules — rotates **daily**, keeps the last **7 days**, compressed. |
| `/etc/profile.d/99-server-health.sh` | The health summary shown on login. |
| `/var/lib/self-healing/state/` | Tracks restart attempts / give-up status per service. |
| `/var/lib/self-healing/heartbeat` | Timestamp updated every check cycle — proves the watchdog is alive. |
| `/var/log/self-healing/events.log` | The full event log. |

---

## 8. Checking on it

```bash
systemctl status self-healing-watchdog.service   # is it running?
tail -f /var/log/self-healing/events.log         # watch it live
cat /etc/self-healing/services.conf              # what's being watched
```

You'll also see a health summary automatically every time you (or
anyone) logs into the server — it shows CPU/memory/disk, and flags any
service that's currently down or has a stale/dead watchdog.

---

## 9. Log cleanup

Handled automatically — no cron job to set up yourself. Every day, the
system's built-in `logrotate` checks `/var/log/self-healing/events.log`
against this config and rotates it, keeping the last 7 days compressed
and deleting anything older.

To check it's working or force a rotation now:
```bash
sudo logrotate -d /etc/logrotate.d/self-healing   # dry-run, shows what would happen
sudo logrotate -f /etc/logrotate.d/self-healing   # force it now
```

---

## 10. Removing it

```bash
sudo ./setup-self-healing.sh --uninstall
```

This stops the watchdog and removes the scripts, systemd service, login
banner, and log rotation rule.

**It deliberately leaves your config, state, and logs behind** —
`/etc/self-healing/`, `/var/lib/self-healing/`, `/var/log/self-healing/`
— in case you want to look back at them or reinstall later. Delete
those folders yourself if you want a completely clean system.

---

## 11. Troubleshooting

**"It asked me a bunch of questions when I didn't want it to"**
That only happens when you run it with zero options from a real
terminal. Pass any flag (even `--dry-run`) to skip straight to
non-interactive mode.

**"I'm getting a warning about some unrelated `.service` file during
install"**
Harmless. The install step double-checks your systemd unit is valid
using a tool that also happens to scan every other unit file on the
system while it's at it. It's not a sign anything about self-healing
broke — check `systemctl status self-healing-watchdog.service` to
confirm the watchdog itself is fine.

**"A service keeps failing the check even though `systemctl status`
says it's running fine"**
This means the *health check itself* is testing the wrong thing — most
commonly, a TCP port the service doesn't actually listen on. This
happened with an HTTPS-only nginx that only serves on 443: the default
check assumed port 80, which nginx was never using. Confirm with:
```bash
ss -tlnp | grep ':<port>'
curl -v http://127.0.0.1:<port>/
```
If nothing's listening there, fix the check in
`/etc/self-healing/services.conf` to point at the port the service
really uses, then `sudo systemctl restart self-healing-watchdog.service`.

**"A flapping service isn't sending me every alert"**
Check the log for `suppressed (cooldown active)` — that's the DOWN
cooldown working as designed, not a bug. Recovery alerts are never
suppressed (see [Notifications](#6-notifications)).
