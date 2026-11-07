# Session 09: nginx install + debugging

- **Date:** 2026-11-06 (part 1) and 2026-11-07 (part 2) — one session, split by a VM pause
- **Phase:** 2 (Linux basics, services) + Phase 1 (Git, alongside)
- **Topic:** Install nginx on Ubuntu, serve on port 80, debug it as a Linux service
- **Session size:** Full (two long blocks)
- **Repo location:** `1.3-nginx-install-debug/`
- **Branch:** `nginx-homework`

---

## Outcome of the day

- **Mentor homework bullet 1 — nginx on port 80, minimal config: DONE.**
  - **Proof:** `elinks http://localhost` shows **"Welcome to nginx!"**, and `curl -I http://localhost` returns `HTTP/1.1 200 OK`, `Server: nginx/1.28.3 (Ubuntu)`, `Content-Length: 615`.
- **Mentor homework bullet 2 — 5 debugging tasks: DONE (lived, not simulated).**
  - Four real incidents happened on their own, plus one deliberate sabotage. See the Debugging section.
- **Config toolkit learned:** `nginx -t`, `systemctl reload`, `ss`, `curl -I`, access log reading.

---

## Red thread (the hotel metaphor)

- **nginx** = the receptionist. **Port 80** = the door she stands at. Only one person can hold a door.
- **`/var/www/html`** = the shelf of brochures she hands out. **`index ...`** = the priority order of brochures.
- **The config file** = the job description pinned above the desk. She only re-reads it when told (`reload`).
- **`nginx -t`** = the proofreader who checks the new job description *before* it is pinned.
- **Logs** = the receptionist's diary. It records who called, when, and what she handed over.
- **Core lesson:** `curl` tells you what the server **sends**. A browser tells you what it **decided to show you**. Trust the server first.

---

## Timeline — installation

| Time | Step | Command | Result |
|---|---|---|---|
| 11-06 ~13:05 | Refresh the package catalog | `sudo apt update` | ❌ `Release file ... is not valid yet` |
| 11-06 13:10 | Inspect the clock | `timedatectl` | VM said 31 Oct, reality was 6 Nov. `synchronized: no` |
| 11-06 13:12 | Find the time service | `systemctl list-units --type=service \| grep -E 'chrony\|ntp\|timesync'` | `chrony.service` (not `systemd-timesyncd`) |
| 11-06 13:15 | Measure the drift | `chronyc tracking` | `System time: 527646 seconds slow` ≈ 6.1 days |
| 11-06 13:18 | Jump the clock | `sudo chronyc makestep` | `200 OK`, clock → 6 Nov ✅ |
| 11-06 13:20 | Retry the catalog | `sudo apt update` | ✅ All `Hit:`, no errors |
| 11-06 13:34 | Install nginx | `sudo apt install nginx` | ❌ `bind() to 0.0.0.0:80 failed (98: Address already in use)` |
| 11-06 13:36 | Who holds door 80? | `sudo ss -tlnp \| grep :80` | `apache2`, pid 128720 — a rival web server |
| 11-06 13:40 | Evict Apache permanently | `sudo systemctl disable --now apache2` | Stopped now **and** at boot |
| 11-06 13:41 | Start nginx | `sudo systemctl start nginx` | ✅ `active (running)` |
| 11-06 13:42 | Install the text browser | `sudo apt install elinks` | Already present |
| 11-06 15:42 | First call | `elinks http://localhost` | ❌ **Apache2 Default Page** |
| 11-06 15:44 | Cross-check with curl | `curl -I http://localhost` | `Server: nginx` but `Content-Length: 10672` |
| 11-06 15:46 | Confirm the door | `sudo ss -tlnp \| grep :80` | Only `nginx` — Apache really gone |
| 11-06 15:48 | Why `127.0.0.1`? | `grep localhost /etc/hosts` | `127.0.0.1 localhost` / `::1 ip6-localhost` |
| 11-06 15:50 | Inspect the shelf | `ls -l /var/www/html` | `index.html` = 10672 B (Apache's), `index.nginx-debian.html` = 615 B |
| 11-06 15:52 | Rename, don't delete | `sudo mv .../index.html .../index.html.apache-backup` | Reversible fix ✅ |
| 11-06 15:53 | Verify at the source | `curl -I http://localhost` | `Content-Length: 615` ✅ |
| 11-06 15:57 | Fresh browser instance | `elinks -no-connect http://localhost` | 🏆 **"Welcome to nginx!"** |

### The three nested bugs of part 1

1. **Clock skew** — apt rejected a catalog that looked like it came from the future.
2. **Port conflict** — Apache already owned door 80, so nginx could not bind.
3. **Document-root collision** — nginx served Apache's leftover `index.html`, because `index.html` is listed *before* `index.nginx-debian.html` in the config. The last stubborn screen was an elinks instance still alive in the background.

---

## Timeline — configuration and debugging (11-07)

| Time | Step | Command | Result |
|---|---|---|---|
| 08:32 | Re-sync after VM pause | `sudo chronyc makestep` + `date` | `Sat Nov 7 08:32:39 UTC 2026` ✅ |
| 09:30 | Read the live config | `grep -vE '^\s*#\|^\s*$' /etc/nginx/sites-enabled/default` | 10 live lines, comments stripped |
| 09:35 | Prove where nginx reads from | `grep -rn include /etc/nginx/nginx.conf` | `include /etc/nginx/sites-enabled/*;` |
| 09:40 | Validate config | `sudo nginx -t` | ✅ `test is successful` |
| 09:44 | *(backup placed inside `sites-enabled/`)* | `sudo nginx -t` | ❌ `[emerg] a duplicate default server for 0.0.0.0:80 ... :22` |
| 09:46 | Apply it anyway | `systemctl reload nginx` | ❌ `Job for nginx.service failed` |
| 09:50 | **Is the site actually down?** | `curl -I http://localhost` | ✅ `200 OK` — **site never went down** |
| 09:52 | Remove the duplicate | `sudo rm /etc/nginx/sites-enabled/default.backup-ok` | ✅ `test is successful` |
| 09:55 | Backup, done right | `sudo cp .../default /tmp/default.backup-ok` | 2412 B, outside nginx's reach |
| 10:05 | **Sabotage:** remove one `;` | `sudo sed -i '...' /etc/nginx/sites-enabled/default` | Silent |
| 10:08 | Diagnose | `sudo nginx -t` | ❌ `invalid parameter "listen" ... default:23` |
| 10:10 | Look at the named line | `sed -n '22,24p' /etc/nginx/sites-enabled/default` | Line 22 missing `;`, line 23 has it |
| 10:15 | Repair | `sudo nano /etc/nginx/sites-enabled/default` | Added the `;` back |
| 10:17 | Re-test, then apply | `sudo nginx -t` → `sudo systemctl reload nginx` | ✅ Successful |
| 10:18 | Final proof | `curl -I http://localhost` | ✅ `200 OK`, `Content-Length: 615` |

### Break-and-fix → real-life mapping

| Simulated break | Real-world scenario it represents | What it taught |
|---|---|---|
| Backup file left in `sites-enabled/` | Engineer copies a config "just in case", service scans the whole folder | Never leave backups where a service reads. Use `/tmp`, `/root`, or Git |
| `duplicate default server` | Two teams both deploy a `default_server` vhost on the same port | `nginx -t` catches it before any downtime |
| `reload` on a broken config | The 9am "I pushed config and now it's down" ticket | `reload` **refuses** and keeps the old config live. `restart` would have taken the site down |
| Missing semicolon | A one-character typo during a late-night fix | Config syntax errors are the #1 cause of failed deploys |
| Error reported at line 23, typo on line 22 | Any parser error in any language | The error points to where the parser *gave up*, often one line after the real mistake |

---

## Command cheat-sheet

### Service control

| Command | Question it answers | Brief explanation | Metaphor |
|---|---|---|---|
| `systemctl status nginx` | Is it running? | Shows active/failed, recent log lines | Peek into the lobby: is anyone at the desk? |
| `sudo systemctl start nginx` | — | Starts the service now | Receptionist clocks in |
| `sudo systemctl reload nginx` | — | Re-reads config **without dropping connections** | She reads the updated rules; desk stays open |
| `sudo systemctl restart nginx` | — | Stop then start. **Brief downtime** | She goes home, a new one arrives |
| `sudo systemctl disable --now apache2` | — | Stops it now **and** prevents it at boot | The rival is sent home for good |

### Diagnostics

| Command | Question it answers | Brief explanation | Metaphor |
|---|---|---|---|
| `sudo nginx -t` | Is my config valid? | Tests syntax, changes nothing | A proofreader checks the rules before they're pinned |
| `sudo ss -tlnp \| grep :80` | Who holds port 80? | `-t` TCP, `-l` listening, `-n` numeric, `-p` process | Asking the doorman who stands behind door 80 |
| `curl -I http://localhost` | What does the server **really** send? | `-I` = headers only, no cache | Asking the hotel directly, ignoring rumours |
| `tail -5 /var/log/nginx/access.log` | Who visited, when, what did they get? | Last 5 lines of the visitor log | The receptionist's diary |
| `tail /var/log/nginx/error.log` | What went wrong? | Last lines of the problem log | The complaints page of the diary |
| `timedatectl` / `chronyc tracking` | Is my clock right? | Shows drift and sync status | Comparing your watch to the bell tower |
| `sudo chronyc makestep` | — | Jumps the clock to correct time immediately | Setting the watch by force |

### Files and text

| Command | Question it answers | Brief explanation | Metaphor |
|---|---|---|---|
| `cat <file>` | What's in this file? | Prints the whole file | Reading the page aloud |
| `grep -vE '^\s*#\|^\s*$' <file>` | What's actually *active*? | Hides comments and blank lines | Removing the pencil scribbles |
| `grep -rn include <file>` | What else does it load? | `-r` recursive, `-n` line numbers | Following the cross-references |
| `sed -n '22,24p' <file>` | What's on line 22-24? | Prints a line range | Flipping to one page |
| `ls -l <dir>` | What's on the shelf, how big, how old? | Long listing with size and date | Inventory of the shelf |
| `sudo mv <old> <new>` | — | Rename or move. **Reversible** | Relabelling a brochure |
| `sudo rm <file>` | — | Delete. **No undo, no recycle bin** | Burning it |
| `sudo cp <src> <dst>` | — | Copy | A photocopy for the safe |
| `sudo nano <file>` | — | Simple editor. `Ctrl+O` Enter = save, `Ctrl+X` = exit | Pen and paper |

### Terminal survival

| Keys | What it does |
|---|---|
| `Ctrl+C` | Cancel the current line / escape a stuck prompt |
| `Tab` | Auto-complete a path. If it completes, it exists |
| `↑` | Recall the previous command |
| `q` then `y` | Quit elinks |
| `^` in nano's bar | Means `Ctrl`. `M-` means `Alt` |

---

## Config anatomy: `/etc/nginx/sites-enabled/default`

```nginx
server {
        listen 80 default_server;                                  # the main door (IPv4)
        listen [::]:80 default_server;                             # the side door (IPv6)
        root /var/www/html;                                        # which shelf
        index index.html index.htm index.nginx-debian.html;        # brochure priority order
        server_name _;                                             # answer to any hostname
        location / {
                try_files $uri $uri/ =404;                         # file, then folder, else 404
        }
}
```

**Two syntax rules that break everything:**
1. Every instruction ends with a **semicolon** `;`
2. Blocks are wrapped in balanced **braces** `{ }`

**Where nginx reads from (proven, not assumed):** `nginx -t` names `/etc/nginx/nginx.conf`, which contains `include /etc/nginx/sites-enabled/*;` — the `*` means **every file in that folder**.

---

## HTTP status codes

| Family | Meaning | Metaphor |
|---|---|---|
| 1xx | Still working | "One moment please" |
| 2xx | Success | "Here you go" |
| 3xx | Go elsewhere | "That department moved" |
| **4xx** | **You** made a mistake | "Wrong room number, sir" |
| **5xx** | **The server** made a mistake | "Sorry, our system is down" |

Common ones: `200` OK · `301`/`302` redirect · `304` not modified · `401` who are you? · `403` I know you, and no · `404` not found · `500` server crashed · `502`/`504` nginx's backend misbehaved · `503` overloaded or in maintenance.

**The 4/5 split is the one to memorise** — it tells you instantly which side to investigate.

---

## Reading an access log line

```
127.0.0.1 - - [06/Nov/2026:15:42:15 +0000] "GET / HTTP/1.1" 200 10672 "-" "ELinks/0.19.1..."
    │                    │                      │            │    │              │
   who                 when                  request      status size        user-agent
```

The log proved the whole mystery: `10672` bytes at 15:42 (Apache's file), `615` bytes at 15:57 (nginx's own). nginx was answering correctly the entire time.

---

## Good practice used

- ✅ `nginx -t` **before** every reload — the single most important habit here
- ✅ `reload` over `restart` — no downtime when the config is valid, no outage when it isn't
- ✅ **Rename, don't delete** — `mv` is reversible, `rm` is not
- ✅ Backups go **outside** the folder the service scans
- ✅ **Least privilege** — dropped `sudo` where it wasn't needed (reading the access log)
- ✅ Never install from random websites — only official Ubuntu repositories
- ✅ `Tab` completion to avoid typos before pressing Enter
- ✅ Verify with a **second tool** when two sources disagree

---

## What worked, what failed

- **Worked:** every diagnosis came from reading the actual error text, never from guessing. The missing-semicolon fix was done solo, start to finish.
- **Failed usefully:** three separate installs/starts failed before nginx served correctly. All three were ordinary, all three had a clear error message pointing at the cause.
- **Surprise:** the VM clock drifts whenever Hyper-V pauses or saves the VM. Fix after every resume: `sudo chronyc makestep`.
- **Biggest realisation:** a broken config on disk does **not** mean a broken site. nginx keeps the last working config in memory.

---

## Homework / next outcome

- **Mentor homework:** both bullets complete. ✅
- **Next session, Outcome:** replace the default page with a custom `index.html` served by nginx | **Done when:** `curl http://localhost` returns your own HTML, confirmed in `elinks`.
- **Carry-over habit:** run `sudo chronyc makestep` after resuming the VM.

---

*Committing this file is itself Git practice.*
*Branch: `nginx-homework` → folder: `1.3-nginx-install-debug/`*
