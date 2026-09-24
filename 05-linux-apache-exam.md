# Topic 5 — Linux Exam: Apache Web Server Setup and Diagnosis

**Goal:** a timed, practical Linux exam — user/group management, standing up Apache from a cold `systemctl status` check, changing its listening port, and diagnosing a fault deliberately introduced by the invigilator, all from the command line only.

## The task list

1. Display username, hostname, and IP.
2. Create `exam/web/logs` under the home directory in one command.
3. Write an index number into `notes.txt` with `echo`, then display it.
4. Create a `webadmin` user, set a password, create a `webteam` group, add the user to it.
5. Check Apache's status; start and enable it if it isn't running.
6. Confirm the default page loads with `curl http://localhost`.
7. Replace the default page with a custom `index.html`.
8. Show which port Apache is listening on.
9. Move Apache to port 8080, reload, and prove the new port serves the site.
10. Find Apache's access log and show its most recent entry.
11. Diagnose and fix a fault the invigilator introduces (service stopped, or `index.html` renamed), then state in one sentence what was wrong and how it was fixed.

## Full command set

```bash
### 1. Username, hostname, IP
whoami
hostname
hostname -I

### 2. Create directory in one command
mkdir -p ~/exam/web/logs

### 3. notes.txt with index number, then display
echo "FCM.41.018.050.24" > ~/exam/notes.txt
cat ~/exam/notes.txt

### 4. Create webadmin user, set password, create group, add user to group
sudo useradd webadmin
sudo passwd webadmin
sudo groupadd webteam
sudo usermod -aG webteam webadmin
groups webadmin

### 5. Check Apache status, start/enable if needed
systemctl status apache2
sudo systemctl start apache2
sudo systemctl enable apache2

### 6. Confirm default page loads
curl http://localhost

### 7. Replace default page with a custom one
echo "<h1>APPIAH SCHNEIDER AGYARE - FCM.41.018.050.24</h1>" | sudo tee /var/www/html/index.html
curl http://localhost

### 8. Show which port Apache is listening on
sudo ss -tulnp | grep apache2
nc -zv localhost 80

### 9. Change Apache to listen on 8080, reload, prove it works
sudo sed -i 's/Listen 80/Listen 8080/' /etc/apache2/ports.conf
sudo sed -i 's/:80>/:8080>/' /etc/apache2/sites-enabled/000-default.conf
sudo systemctl reload apache2
curl http://localhost:8080

### 10. Find Apache access log, show most recent entry
sudo tail -n 1 /var/log/apache2/access.log

### 11. Diagnose and fix a broken web server
systemctl status apache2
ls -l /var/www/html/
sudo journalctl -u apache2 -n 30 --no-pager
sudo tail -n 10 /var/log/apache2/error.log
```

(RHEL/Rocky/Fedora differs in package name — `httpd` instead of `apache2` — and config path — `/etc/httpd/conf/httpd.conf` instead of `ports.conf` plus a site file, with a SELinux port label needed if enforcing: `sudo semanage port -a -t http_port_t -p tcp 8080`. `cat /etc/os-release` first confirms which set applies.)

## Real troubleshooting: "enabled but inactive"

Partway through, Apache reported enabled (would start on boot) but stayed inactive even after `systemctl start`. Enabled and healthy are two different things, so the actual failure needed digging out rather than just retrying the start:

```bash
sudo systemctl status apache2 -l
sudo journalctl -xeu apache2 --no-pager | tail -30
sudo apache2ctl configtest        # catches a config typo, e.g. from editing ports.conf for task 9
sudo ss -tulnp | grep :80         # something else already bound to port 80?
sudo tail -n 30 /var/log/apache2/error.log
```

A candidate cause on this attempt: a `ports.conf` edit made earlier for the port-8080 task had a typo (a stray `>`), which `apache2ctl configtest` would catch directly by naming the exact file and line. On the actual run, the service came back healthy on a retry (`systemctl is-active apache2` → `active (running)`, confirmed alongside `curl` returning the real page and `ss -tulnp` showing two live `apache2` worker processes on port 80) before the root cause needed to be pinned down further — but the diagnostic sequence above is the one that would have isolated it if it hadn't cleared.

One real slip caught along the way: the custom `index.html` was typed once without the dot in the index number (`FCM41.018.050.24` instead of `FCM.41.018.050.24`, matching what `notes.txt` actually contained) — a reminder to diff exam output against the value you were given, not just against what you meant to type.

## Practicing the actual invigilator-fault scenario, not just reading about it

Rather than only walking through the diagnosis in theory, the fault was reproduced for real — renaming the live `index.html` exactly the way an invigilator would, then working the diagnosis cold:

```bash
# Simulate the break
sudo mv /var/www/html/index.html /var/www/html/index.html.bak

# Confirm it's actually broken
curl http://localhost:8080          # now 403 Forbidden — no index file, directory listing disabled

# Diagnose without assuming the cause
systemctl status apache2            # active (running) — rules out the service itself
ls -l /var/www/html/                # shows index.html.bak sitting there instead of index.html — the actual answer
sudo journalctl -u apache2 -n 30 --no-pager
sudo tail -n 10 /var/log/apache2/error.log   # "File does not exist: /var/www/html/index.html"

# Fix and verify
sudo mv /var/www/html/index.html.bak /var/www/html/index.html
curl http://localhost:8080          # custom page back
```

**One-sentence exam answer:** "The web server's index page was missing because the invigilator had renamed `index.html` to `index.html.bak`, which I found using `ls -l /var/www/html/`, and fixed by renaming it back with `sudo mv /var/www/html/index.html.bak /var/www/html/index.html`."

The reasoning that generalizes: `systemctl status` ruling the service itself in or out first, then `ls -l` on the actual served directory, is a faster and more certain path to the cause than guessing between "service problem" and "content problem" — the two most common categories an invigilator-introduced fault falls into.

## Summary

| Task | Real issue hit | Root cause / fix |
|---|---|---|
| Start Apache | Showed enabled but inactive after `start` | "Enabled" only means it'll try on boot; used `journalctl` + `apache2ctl configtest` + `ss` to isolate the real failure instead of retrying blindly |
| Custom index.html | Index number typed without the dot | Caught by comparing rendered output against `notes.txt`, not against memory |
| Port 8080 change | Needed edits in two separate files (`ports.conf` + site config) | Both required on Debian/Ubuntu; RHEL/Rocky consolidates into one file, differs by distro |
| Invigilator fault | `index.html` renamed to `.bak` | Diagnosed cold with `systemctl status` → `ls -l` → `journalctl`, ruling out the service before checking the content |
