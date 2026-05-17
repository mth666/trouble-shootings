# Server Slowness scenario standard Diagnostics and plan

scenario: Someone reported that the server is slow 

---

## common diagnosis commands

```
# cpu
```
top
ps aux --sort=-%cpu | head -10
```

# RAM/Memory
```
free -h
ps aux --sort=-%mem | head -10
```

# disk space
```
df -h
du -sh /*
```

# io wait
```
vmstat 1
iostat -x 1
```
# network
```
ping 8.8.8.8
traceroute 8.8.8.8
ss -tunlp
ss -tan | grep ESTABLISHED | wc -l
```
# load history list
```
uptime
w
```

# logs RH world
```
sudo tail -n 100 /var/log/messages
sudo grep -i "error" /var/log/messages
sudo systemctl --failed
```
### modern way: 
```
sudo journalctl -n 100 (reads from the binary journal)
```
### traditional way: 
```
sudo tail -n 100 /var/log/messages (reads from the text file
```
# logs Debian world (traditional text files) legacy logs (only exist if rsyslog/syslog is installed) 
```
sudo tail -n 100 /var/log/syslog
sudo grep -i "error" /var/log/syslog
sudo systemctl --failed
```
# logs modern Debian world way
### old way:
need to remember specific file paths (/var/log/syslog on Debian, /var/log/messages on RHEL) and use text tools like tail or grep.
### new way: 
no longer need to use path, use one unified engine (journalctl) across both operating systems, and it does the job.

# modern world (Both Red hat world & Debian using systemd)
```
sudo journalctl -n 100
sudo journalctl -p err
sudo systemctl --failed
```
# Journal
```
sudo journalctl -n 100
sudo journalctl -u nginx
```
## basic commands combos both Red hat world and Debian

| RH world / traditional commands (`/var/log/messages`) | modern Debian Equivalent (`journalctl`) | descriptions / contexts |
| :--- | :--- | :--- |
| `tail -f /var/log/messages` | `journalctl -f` | follows and displays new system logs in real time. |
| `cat /var/log/messages \| grep -i error` | `journalctl -p err` | filters the entire log stream by "Error" priority levels. |
| `less /var/log/messages` | `journalctl` | opens a scrollable, chronological viewer of all system logs. |
| `tail -n 100 /var/log/messages` | `journalctl -n 100` | views  the last 100 log lines. |
| grep "nginx" /var/log/messages | `journalctl -u nginx` |  display logs generated only by a specific service (exp: Nginx). |
| awk -v d="$(date -d '1 hour ago' +'%b %e %H:%M:%S')" '$0 > d' | `journalctl --since "1 hour ago"` | isolates logs from a specific timeframes. |
| sed -n '/Linux version/,$p' /var/log/messages | `journalctl -b` | hides previous logs and display only the current boot session. |

## first imp rule

always diagnose before you act. never restart things, never update packages, never change configs until understand what is actually wrong. a blind restart might mask the real problem or make things worse.

---

## 1st check CPU resources

ss something using all the cpu processing powers?
commands
```
top
```

```
htop
```

things to look for:
- process with high %CPU column
- Load average in the top line, on a 1 cpu system anything above 1.00 means queue is building up
- us = user processes eating CPU
- sy = kernel eating CPU
- wa = CPU waiting for IO 

quick snapshot overview instead of live view:

```
ps aux --sort=-%cpu | head -10
```

shows top 10 cpu consuming processes right now.

---

## 2nd Memory and Swap resource checking

is the server running out of memory?

```
free -h
```

things to look for:
- available column : real free memory, not just the free column
- swap usage : if swap is being used heavily the server is using disk as RAM which is extremely slow

```
free -m
```
-m human-friendly formats

if swap is heavily using, find what is eating RAM:

```
ps aux --sort=-%mem | head -10
```
retrieve top 10 memory using processes list.

---

## 3rd disk space resources check

is the disk full?

```
df -h
```

things to look for:
- any partition at 90% or above is a warning
- any partition at 100% is an emergency, services will start failing silently

find which folder is eating space:

```
du -sh /*
```

then find the fat folder:

```
du -sh /var/*
```
Note; a full disk causes all kinds of weird slowness, apps cant write logs, databases cant write data, tmp files cant be created. it breaks things in ways that don't obviously point to disk being the problems.

---

## 4th io Wait checking or disk read write

is the server waiting on disk reads and writes?

```
vmstat 1
```

note: 1 means run it with 1 second intervals and watch for a few seconds.

things to look for:

```
r  = processes waiting for CPU (should be in low)
b  = processes in uninterruptible sleep waiting for IO (should be zero)
wa = percentage CPU is waiting for IO (anything above 20% is concernings)
si = swap in (memory being read from swap disk)
so = swap out (memory being written to swap disk)
```

high wa means the disk is the bottleneck. everything else is waiting for reads and writes to finish.

more detailed IO stats:

```
iostat -x 1
```

note: displays per disk read and write speeds and utilization. this tool may need to install with:

```
sudo dnf install sysstat
```

---

## 5th running processes check
is something suspicious running that shouldnt be or not recognized?

```
ps aux
```

```
ps aux --sort=-%cpu | head -20
```

things to look for:
- processes that dont recognize or shady
- processes owned by wrong users
- multiple copies of the same process running when there should be one

check process tree to see parent and child relationships:

```
pstree
```

---

## 6th system load history log check

what has the load been doing over time?

```
uptime
```

display load averages for last 1, 5 and 15 minutes. if 15 minute average is high the problem has been going on a while. if only 1 minute is high it might be a spike.

```
w
```

display uptime, load averages and who is logged in. someone else might be running something heavy on the server concurrently.

---

## 7th network check

can the server reach the internet?

testing local network first:

```
ping gateway_IP
```

testing external connectivity:

```
ping 8.8.8.8
```

things to look for:
*packet loss percentage, anything above 1% is worth investigating
*response time consistency, big jumps between responses means delays
*complete failure means no route to host

trace the network path to find where packets are being drop:

```
traceroute 8.8.8.8
```

each line is one hop. need to looks reponse times suddenly jump or asterisks mean no responses. 

checking open connections and listening ports:

```
ss -tunlp
```

so many connections in certain states can indicate a network problem or even an attack.

count established connections:

```
ss -tan | grep ESTABLISHED | wc -l
```

if this number is unusually high the server might be getting hammered with too many connections.

---

## 8th logs checking

what is the system reported?

RHEL:
```
sudo tail -n 100 /var/log/messages
```

Debian:
```
sudo tail -n 100 /var/log/syslog
```

Watch live:
```
sudo tail -f /var/log/messages
```

looking for errors:
```
sudo grep -i "error" /var/log/messages
sudo grep -i "failed" /var/log/messages
sudo grep -i "warning" /var/log/messages
sudo grep -i "critical" /var/log/messages
```

checking authentication log for shady login attempts:

RHEL:
```
sudo tail -n 100 /var/log/secure
```

Debian:
```
sudo tail -n 100 /var/log/auth.log
```

checking nginx logs if web server is involved:
```
sudo tail -n 100 /var/log/nginx/error.log
sudo tail -n 100 /var/log/nginx/access.log
```

---

## 9th services checking

is all expected services actually running?

```
sudo systemctl list-units --type=service --state=running
```

checking a specific service:
```
sudo systemctl status nginx
sudo systemctl status sshd
sudo systemctl status firewalld
```

looks for failed services:
```
sudo systemctl --failed
```

note : this is quick way to see if anything was crashed.

---

## 10th system journal checking

note: modern RHEL uses systemd journal for logs. powerful for finding what happened and when.

display all recent logs:
```
sudo journalctl -n 100
```

watch live:
```
sudo journalctl -f
```

any logs since last boot only:
```
sudo journalctl -b
```
Logs for a specific service:
```
sudo journalctl -u nginx
sudo journalctl -u sshd
```

any logs from last hour:
```
sudo journalctl --since "1 hour ago"
```

---

## what does each findings mean

```

| system metrics / issues | system admin action |
| :--- | :--- |
| high CPU usage | find and investigate the hungry process. |
| **high RAM, swap in use** | find memory leak, plan about adding more RAM. |
| disk at 100% | find and clean fat folders immediately. |
| high IO wait | disk is bottleneck, check what is reading and writing heavily. |
| packet loss to 8.8.8.8 | network issues, report to network team. |
| errors in logs** | read them well, they usually tell exactly what was wrong. |
| failed services** |  `systemctl status` to find out why it crashed. |

```

---

## standard way to communicate with your team about above findings

report to manager:
* What you checked
* What looks healthy
* What looks shady
* What will be done next

a clear status update buys time and imply that the admin is in control even when there is no answers yet.
exp : "i have checked X, Y, Z and they look ok. i am now diving into more details other services A and B"

---

## common slowness causes in real work environments
```
1 runaway process eating 100% CPU
2 memory leak slowly consuming all RAM until swap kicks in
3 log files filling up the disk
4 many concurrent users overwhelming the server
5 external api the app depends on being slow or down
6 database queries running slow due to missing indexes
7 network congestion or packet loss
7 external attacks like DDoS attack overwhelming the network
8 cron job running at peak hours using system resources
```

---
