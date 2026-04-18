Process / Thread / Service

Process = running program with isolated memory and PID

Thread = execution unit inside process sharing process memory

Service = long-running background app managed by systemd/init

Slow Linux server

Check CPU, load, memory, swap, disk, inodes, I/O wait, process hotspots, logs, sockets:

top, uptime, free -m, vmstat 1, iostat -xz 1, df -h, df -i, ps, journalctl



Disk 40% full but no new files

Likely inode exhaustion, permissions, read-only FS, quota, corruption



chmod 755

Owner rwx, group r-x, others r-x



SIGTERM vs SIGKILL

SIGTERM graceful, catchable; SIGKILL immediate, uncatchable



Pod can’t reach google.com

Check pod → exec → DNS → curl → network policy → CoreDNS → CNI → node → route/NAT/SG/NACL



DNS / TCP / HTTP / TLS

DNS = name resolution

TCP = reliable transport

HTTP = app protocol

TLS = encryption/authentication layer



CIDR 10.0.1.0/24

24 network bits, 256 addresses, subnet mask 255.255.255.0



SG vs NACL

SG = stateful, instance-level, allow only

NACL = stateless, subnet-level, allow/deny



502

Bad upstream/backend issue, unhealthy targets, protocol mismatch, timeout, bad response



Image vs Container

Image = immutable package

Container = running instance of image



Container exits

Main process ends/crashes/fails startup/OOM/wrong command



CMD vs ENTRYPOINT

ENTRYPOINT = executable

CMD = default args/default command



Minimal images

Smaller, faster, safer, fewer CVEs



Deployment / StatefulSet / DaemonSet / Job

Stateless replicas / stable identity stateful / one per node / run to completion



Pending pod causes

Insufficient CPU/mem, taints, node selector, affinity, PVC issue, node not ready, quota, scheduler issue, no matching nodes



Service not routing

Selector, labels, endpoints, readiness, ports, namespace, app listen port, network policy



Readiness vs Liveness

Readiness = can receive traffic

Liveness = should be restarted?



CrashLoopBackOff

Container repeatedly crashes and restart backoff occurs



K8s service discovery

Service selector → endpoints → DNS via CoreDNS → traffic routing via kube-proxy/dataplane



Subnet / route table / IGW / NAT

IP range / traffic rules / internet access for public / outbound internet for private



Private subnets

Reduce direct internet exposure



ALB vs NLB

ALB Layer 7 HTTP/HTTPS; NLB Layer 4 TCP/UDP/TLS



IAM role

Assumable identity with temporary credentials; safer than hardcoded secrets



Terraform state

Infra mapping store; corruption/misuse can break management



Drift

Real infra differs from Terraform state/config



CI vs CD

CI validates/builds changes

CD releases/deploys validated changes



Pipeline passed but prod broke

Tests/checks incomplete or env/runtime/config mismatch



Metrics / Logs / Traces

Numeric trends / event detail / request path across services



Useless alert

Noisy, unactionable, contextless, irrelevant, ownerless



\---------------------------------------------------------------

## **Day 1: Linux**

**5-4-26**

#### Lesson 1: Processes, PIDs, parent/child, open files

A process is a running program. Every process has:



a PID (process ID)

a PPID (parent process ID)

an owner/user

memory

open file descriptors

possibly child processes

one or more threads



##### Identify the process -

ps -p 2001 -f

ps -p 2001 -o pid,ppid,user,%cpu,%mem,cmd

readlink -f /proc/2001/exe

cat /proc/2001/cmdline | tr '\\0' ' '

ps -o pid,ppid,user,cmd -p 2001

pstree -sp 2001



##### Check child processes

pgrep -P 2001

pstree -p 2001



##### Check open files/sockets

lsof -p 2001

ls -l /proc/2001/fd

ss -tulpn



PID 1			First userspace process, ancestor of all processes, adopts orphans

Orphan Process		Child whose parent died — gets re-parented to PID 1

Zombie Process		Child that exited but parent hasn't called wait() — stuck in process table

Zombie Reaping		The act of the parent calling wait() to clean up dead children

Why it matters in containers	Your app is PID 1, must handle signals and reap zombies properly



Zombies don't consume CPU or memory but they occupy slots in the process table. The limit is defined by:

cat /proc/sys/kernel/pid\_max

Once that's exhausted, no new processes can be created. Your system can't fork anything. SSH stops working. Cron jobs fail. Services can't restart. Complete meltdown.

##### You CANNOT kill a zombie process.

A zombie is ALREADY DEAD. It's a corpse sitting in the process table. You're trying to kill a dead body. kill -9 sends SIGKILL to a process — but there's no process to kill. It already exited. The entry is just waiting to be cleaned up.



So How Do You Actually Get Rid of Them?

You kill the PARENT.



\# Find the zombie

ps aux | awk '\\$8 == "Z"'



\# Find its parent PID

ps -eo pid,ppid,stat,cmd | grep Z



\# Kill the PARENT process

kill -SIGCHLD <parent\_pid>    # First try: nudge the parent to reap

kill <parent\_pid>              # Second try: terminate the parent

kill -9 <parent\_pid>           # Last resort: force kill the parent



When the parent dies:



Zombie gets re-parented to PID 1

PID 1 calls wait()

Zombie gets reaped

Process table entry freed





And in Containers?

If your containerized app IS the parent AND is PID 1 AND isn't reaping zombies — you have two choices:



Fix the application to properly handle child processes

Use tini or dumb-init as PID 1 instead



Key Takeaway:

###### ***You don't kill zombies. You kill their negligent parents.***



\------------------------------------------------

##### CPU usage

How much CPU is actively being used right now.



Example:



user CPU

system CPU

idle

steal

iowait



top

mpstat

vmstat 1

\-------------------------------------------

##### Load average

This is not CPU percent.



Load average is roughly the number of tasks that are:



running on CPU

or waiting in uninterruptible state, often waiting on disk I/O

Shown as:



1 min

5 min

15 min averages

Example:



on a 4-core machine, load of 4.0 means roughly full utilization

load of 10 on 4 cores means serious contention

\---------------------------------------------------------------

##### Memory usage

How much RAM is used/free/available, including cache/buffers.



free -m

cat /proc/meminfo



Linux uses memory for caching. “Used” memory is not automatically bad.



Focus on:



available memory

swap usage

OOM events



\----------------------------------------------------

##### I/O wait

CPU time spent waiting for disk/network block I/O to complete.



If iowait is high:



CPU may appear underused

but system still feels slow

because processes are blocked waiting on storage



vmstat 1

iostat -xz 1

top

\------------------------------------------------

##### Inodes

A filesystem stores:



actual file data blocks

metadata structures called inodes

An inode stores metadata like:



owner

permissions

timestamps

file type

pointers to data blocks

What it does not store:



the filename directly in the inode in the usual user sense; directory entries map names to inode numbers

Why you care

You can run out of inodes even if disk space remains.



Example:



millions of tiny files

df -h shows free space

but df -i shows 100% inode use

result: cannot create new files

This is a very real production issue.

\---------------------------------------------

##### systemd and services

What is systemd?

systemd is the init and service manager on most modern Linux distros.



It:



starts services during boot

manages daemons

restarts services if configured

keeps logs via journald

tracks service states and dependencies

\------------------------------------------------

##### Why users cannot connect to port 8080 even if service listens there



Layered troubleshooting model

If service listens on 8080 but users can’t connect, causes may be:



###### Application layer

app is bound only to 127.0.0.1, not 0.0.0.0

app is unhealthy

app crashes after startup

wrong protocol expectation



###### Host layer

local firewall rules (iptables, nftables, ufw, firewalld)

SELinux/AppArmor restrictions

port occupied by wrong process

###### 

###### OS/network layer

routing issue

wrong interface

NIC issue

###### 

###### Cloud/network perimeter

security groups

NACLs

route table

load balancer config

target group health check failure

###### 

###### Name/proxy layer

DNS points elsewhere

reverse proxy misconfigured

ingress misrouted

\------------------------------------------

##### Flow of a browser request to https://example.com



High-level path:



User enters URL

https://example.com



Browser parses URL

scheme: https

host: example.com

default port: 443



DNS resolution

browser/OS checks cache

if not cached, resolver queries DNS to get IP



Routing decision

your machine decides how to reach that IP using routing table/default gateway



TCP handshake

browser opens TCP connection to server IP on port 443



TLS handshake

negotiate encryption

server presents certificate

client validates cert

session keys established



HTTP request sent

e.g. GET / HTTP/1.1

host header included



Load balancer / CDN / reverse proxy may receive it first

and forward upstream



Application responds

server returns status code, headers, body



Browser renders

may request CSS/JS/images

each may trigger more requests

\----------------------------------------------------



##### TLS handshake

High-level only, not cryptography PhD nonsense.



TLS handshake does:



authenticate server identity

negotiate cipher suite/protocol

establish shared session keys

enable encrypted communication



Typical flow:

client says supported TLS versions/ciphers

server replies with chosen parameters and certificate



client validates certificate:

trusted CA?

hostname matches?

not expired?

key exchange happens

encrypted session begins



Common TLS failure causes

expired cert

hostname mismatch

untrusted CA

protocol mismatch

bad intermediate cert chain

clock skew

\-----------------------------------------

##### VM resolves DNS but cannot reach external websites — possible causes

This is an interview favorite because it tests whether you think in layers.



If DNS works but websites don’t load, causes may include:



1. no internet route/default gateway
2. security group blocks egress
3. NACL blocks outbound or return traffic
4. firewall on VM blocks outbound
5. proxy misconfiguration
6. NAT gateway missing/broken for private subnet
7. internet gateway missing for public subnet
8. public IP missing on supposedly public VM
9. application using wrong port/protocol
10. TLS/certificate failure
11. remote site blocks traffic
12. MTU issue / packet fragmentation issue
13. outbound port 80/443 blocked by corporate network
14. time skew causing TLS failures
15. local routing issue
16. service mesh/proxy sidecar issue if inside platform
17. bad iptables/nftables rules
18. network interface down
19. upstream ISP/network issue
20. destination reachable by DNS but app endpoint down

\--------------------------------------------------------------

### Lesson 2: Signals — The Language Processes Speak



Signal		Number		Meaning						Can Process Ignore It?		Default Action

SIGHUP		1		Terminal closed / "Hang up"			✅ Yes				Terminate

SIGINT		2		Interrupt — what happens when you press Ctrl+C	✅ Yes				Terminate

SIGQUIT		3		Quit + dump core — Ctrl+\\			✅ Yes				Terminate + core dump

SIGKILL		9		Force kill — immediate death			❌ CANNOT be caught or ignored	Terminate

SIGTERM		15		Graceful termination request			✅ Yes				Terminate

SIGSTOP		19		Pause/freeze process				❌ CANNOT be caught or ignored	Stop process

SIGCONT		18		Resume a stopped process			✅ Yes				Continue

SIGCHLD		17		Child process died or stopped			✅ Yes				Ignore

SIGUSR1/2	10/12		User-defined — app decides what to do		✅ Yes				Terminate



##### Real-World: The Kubernetes Pod Termination Lifecycle



1\. Pod state set to "Terminating"

2\. preStop hook executes (if defined)

3\. SIGTERM sent to PID 1 in the container

4\. Pod is removed from Service endpoints (stops receiving traffic)

5\. terminationGracePeriodSeconds countdown begins (default: 30s)

6\. Application is expected to:

&#x20;  - Stop accepting new requests

&#x20;  - Finish processing in-flight requests

&#x20;  - Close database connections

&#x20;  - Flush caches/buffers

&#x20;  - Exit cleanly

7\. If process hasn't exited after grace period → SIGKILL





⚠️ WHERE THIS GOES CATASTROPHICALLY WRONG:

Scenario 1: Application doesn't handle SIGTERM



Your Node.js app doesn't have a SIGTERM handler. Kubernetes sends SIGTERM → app ignores it → 30 seconds of doing nothing → SIGKILL. Users experience 30 seconds of requests going to a pod that's supposed to be dead. During rolling deployments, this means dropped requests and timeouts.



Scenario 2: Cleanup takes longer than 30 seconds



Your app handles SIGTERM but needs 60 seconds to drain connections. SIGKILL arrives at 30 seconds. Connections severed mid-transaction. Database writes half-completed. Data corruption.



**spec:**

&#x20; **terminationGracePeriodSeconds: 90  # Give it enough time**

&#x20; **containers:**

&#x20; **- name: my-app**

&#x20;   **lifecycle:**

&#x20;     **preStop:**

&#x20;       **exec:**

&#x20;         **command: \["/bin/sh", "-c", "sleep 5"]**

&#x20;         **# Why sleep 5? Because step 4 (endpoint removal)**

&#x20;         **# is ASYNCHRONOUS. This gives kube-proxy time to**

&#x20;         **# update iptables rules so new traffic stops**

&#x20;         **# arriving BEFORE your app starts shutting down.**



Scenario 3: App traps SIGTERM but doesn't exit



Your app catches SIGTERM, runs cleanup, but has a bug — it never calls exit(). Process hangs. 30 seconds later, SIGKILL. Same problem as Scenario 1 but sneakier.



💡 SIGHUP in the Real World:

Many daemons use SIGHUP to reload configuration without restarting:

\# Reload Nginx config without downtime

kill -SIGHUP $(cat /var/run/nginx.pid)



\# Same as

nginx -s reload



This is how you do zero-downtime config changes on traditional servers. Prometheus, HAProxy, Nginx — they all support this.





💡 SIGUSR1/SIGUSR2 in the Real World:

These are custom signals. Applications define their own behavior:

\# Tell dd to print progress

kill -USR1 $(pgrep dd)



\# Tell Golang apps to dump goroutine stack traces

kill -USR1 <go-app-pid>





The kill Command — It's Not Just For Killing:

Despite the name, kill is really send\_signal:



kill <pid>          # Sends SIGTERM (15) by default — GRACEFUL

kill -9 <pid>       # Sends SIGKILL — NUCLEAR, no cleanup

kill -HUP <pid>     # Sends SIGHUP — reload config

kill -STOP <pid>    # Freezes process

kill -CONT <pid>    # Resumes frozen process

kill -0 <pid>       # Sends nothing — just checks if process exists





**Process States — What ps Shows You:**



**D — Uninterruptible sleep (waiting on I/O, CANNOT be killed even with SIGKILL)**

**R — Running**

**S — Sleeping (interruptible)**

**T — Stopped (received SIGSTOP or Ctrl+Z)**

**Z — Zombie (dead but not reaped)**



**D state is the scariest. A process in D state is waiting on I/O (usually disk). Even SIGKILL won't touch it. If your disk is dying and processes are stuck in D state — nothing short of a reboot fixes it.**



**# Check what the process is waiting on**

**cat /proc/<pid>/wchan**



**# Check for disk errors**

**dmesg -T | grep -i "error\\|fail\\|hung"**



**# Check if it's an NFS issue**

**mount | grep nfs**



**# Check disk health**

**smartctl -a /dev/sda**





**Question 1:**

**You're rolling out a new deployment. Users are reporting 502 errors for about 5 seconds during each rolling update, even though the new pods come up healthy. Old pods are being terminated.**



**Based on what you just learned about the termination lifecycle, what is the most likely cause and how do you fix it?**



1\. Kubernetes says "terminate old pod"

2\. SIGTERM sent to the app → app starts shutting down

3\. BUT kube-proxy hasn't updated iptables yet

4\. For \~3-5 seconds, traffic is STILL being routed to the dying pod

5\. Dying pod is either:

&#x20;  - Already closed its listener → CONNECTION REFUSED → 502

&#x20;  - Processing shutdown → SLOW/TIMEOUT → 502

6\. kube-proxy finally updates → traffic stops flowing to old pod

&#x20;  But damage is done.



The Fix — preStop Hook:



lifecycle:

&#x20; preStop:

&#x20;   exec:

&#x20;     command: \["/bin/sh", "-c", "sleep 5"]



1\. Kubernetes says "terminate old pod"

2\. preStop hook runs → sleep 5

3\. During those 5 seconds, kube-proxy removes pod from endpoints

4\. Traffic stops flowing to old pod

5\. preStop finishes → SIGTERM sent

6\. App shuts down gracefully with ZERO in-flight requests

7\. No 502s. No dropped connections. Clean.



Kubernetes components (API server, kubelet, kube-proxy, endpoints controller) are eventually consistent, not instantly consistent. There are always race conditions between them.



The Problem: Shell Form vs. Exec Form



The issue is how the CMD is written in the Dockerfile.



The developer used the shell form: CMD node server.js.



When you use the shell form, Docker does not run your application as the primary process (PID 1). Instead, it wraps your command in a shell:

/bin/sh -c "node server.js"



In this scenario, /bin/sh is PID 1, and your Node.js application is a child process.



Why exactly 30 seconds?



The Signal Gap: When Kubernetes terminates a pod, it sends a SIGTERM signal to PID 1 to request a graceful shutdown.



The Shell Barrier: By default, a shell (/bin/sh) does not forward signals to its child processes. Therefore, the SIGTERM hits the shell, but it is never passed down to the Node.js process.



The Wait: Because the Node.js app never receives the signal, it continues to run as if nothing happened. Kubernetes waits for the pod to exit on its own.



The Kill: Kubernetes has a default terminationGracePeriodSeconds of 30 seconds. Once this timer expires, Kubernetes gives up on the "graceful" part and sends a SIGKILL.



The Result: SIGKILL cannot be ignored or blocked; it forces the process to terminate immediately. This is why the pod always takes exactly 30 seconds to die.



Two Ways to Fix It

Fix 1: Use the "Exec Form" (Recommended)



Change the CMD syntax to the JSON array format. This tells Docker to run the executable directly without wrapping it in a shell. Node.js will become PID 1 and will receive the SIGTERM directly.



Change this:

CMD node server.js



To this:



code

Dockerfile

download

content\_copy

expand\_less

CMD \["node", "server.js"]

Fix 2: Use an Init Process (e.g., Tini)



In some complex scenarios, you might actually need a shell or have multiple processes. In those cases, use a lightweight init system like tini. Tini is designed to act as PID 1, reap zombie processes, and properly forward signals to children.



Updated Dockerfile:



code

Dockerfile

download

content\_copy

expand\_less

FROM node:18



\# Install tini

RUN apt-get update \&\& apt-get install -y tini

WORKDIR /app

COPY . .

RUN npm install



\# Use tini as the entrypoint to manage signals

ENTRYPOINT \["/usr/bin/tini", "--"]

CMD \["node", "server.js"]



\-------------------------------------------------------

### Lesson 3: File Descriptors, I/O Redirection \& The /proc Filesystem

File Descriptors (FDs)

Every process in Linux interacts with files, pipes, sockets, and devices through file descriptors — small integer handles that the kernel assigns.



FD	Name	Points To	Symbol

0	stdin	Standard Input (keyboard by default)	<

1	stdout	Standard Output (terminal by default)	> or 1>

2	stderr	Standard Error (terminal by default)	2>



After that, any file, socket, or pipe the process opens gets FD 3, 4, 5, and so on.





Why This Matters:

When your app writes logs, opens database connections, opens files — each one consumes a file descriptor. Every TCP connection is a file descriptor. Every open file is a file descriptor.



\# How many FDs is a process using?

ls /proc/<pid>/fd | wc -l



\# What is each FD pointing to?

ls -la /proc/<pid>/fd

\# You'll see symlinks like:

\# 0 -> /dev/pts/0 (terminal)

\# 1 -> /dev/pts/0 (terminal)

\# 2 -> /dev/pts/0 (terminal)

\# 3 -> socket:\[12345] (a TCP connection!)

\# 4 -> /var/log/app.log (a log file)





###### **FD Limits — The Silent Production Killer:**

Every system has limits on how many FDs a process can open:



\# Soft limit (what's enforced by default)

ulimit -Sn

\# Usually 1024



\# Hard limit (maximum the soft limit can be raised to)

ulimit -Hn

\# Usually 65536



\# System-wide maximum

cat /proc/sys/fs/file-max





###### **Real-world disaster scenario:**

Your Nginx reverse proxy handles 10,000 concurrent connections. Each connection = 2 FDs (one for client, one for upstream). That's 20,000 FDs minimum. Default soft limit is 1024.

What happens? Nginx starts rejecting connections. Logs show:



socket() failed (24: Too many open files)



**Error code 24 = EMFILE = file descriptor limit reached.**



**# Temporary (current session)**

**ulimit -n 65536**



**# Permanent — edit /etc/security/limits.conf**

**nginx    soft    nofile    65536**

**nginx    hard    nofile    65536**



**# For systemd services — edit the unit file**

**\[Service]**

**LimitNOFILE=65536**



In Kubernetes, this matters too:



\# You can set this in your pod spec

securityContext:

&#x20; ulimits:

&#x20; - name: nofile

&#x20;   soft: 65536

&#x20;   hard: 65536



###### **I/O Redirection — The Full Picture**

You probably know > and >>. But do you know ALL of these?



\# stdout to file (overwrite)

command > file.txt



\# stdout to file (append)

command >> file.txt



\# stderr to file

command 2> errors.txt



\# stdout AND stderr to same file

command > all.txt 2>\&1

\# OR (modern bash)

command \&> all.txt



\# stderr to stdout (merge streams)

command 2>\&1



\# Discard all output (the black hole)

command > /dev/null 2>\&1



\# Send stdout to one file, stderr to another

command > output.txt 2> errors.txt



\# Pipe only stderr

command 2>\&1 1>/dev/null | grep "error"



###### The 2>\&1 Order Matters:

\# WRONG — stderr still goes to terminal

command 2>\&1 > file.txt

\# This says: "redirect stderr to where stdout currently points (terminal),

\# THEN redirect stdout to file." Stderr still hits terminal.



\# RIGHT — both go to file

command > file.txt 2>\&1

\# This says: "redirect stdout to file, THEN redirect stderr to

\# where stdout currently points (file)." Both go to file.





###### **The /proc Filesystem — Your X-Ray Vision**

/proc is a virtual filesystem. Nothing on disk. The kernel generates it on the fly. It's a window into every running process and the kernel's state.



Per-Process Information (/proc/<pid>/):



\# Command that started this process

cat /proc/<pid>/cmdline | tr '\\0' ' '



\# Environment variables of the process

cat /proc/<pid>/environ | tr '\\0' '\\n'



\# Current working directory

ls -la /proc/<pid>/cwd



\# What binary is running

ls -la /proc/<pid>/exe



\# Open file descriptors (we covered this)

ls -la /proc/<pid>/fd



\# Memory map

cat /proc/<pid>/maps



\# Process status (state, memory usage, threads)

cat /proc/<pid>/status



\# Network connections this process has

cat /proc/<pid>/net/tcp



System-Wide Information:

\# CPU info

cat /proc/cpuinfo



\# Memory info

cat /proc/meminfo



\# Kernel version

cat /proc/version



\# Mount points

cat /proc/mounts



\# Network statistics

cat /proc/net/dev



\# Current load average

cat /proc/loadavg



\# System uptime in seconds

cat /proc/uptime



###### **Real-World Uses:**

Scenario 1: An app is misbehaving but you can't find its config file. Where is it reading config from?



\# Check what files it has open

ls -la /proc/<pid>/fd | grep -i config



\# Check its environment variables for config paths

cat /proc/<pid>/environ | tr '\\0' '\\n' | grep -i config



\# Check its command line arguments

cat /proc/<pid>/cmdline | tr '\\0' ' '



Scenario 2: You suspect a process has been secretly modified (security incident). Is the running binary the same as what's on disk?



\# What binary is the process actually running?

ls -la /proc/<pid>/exe

\# If this points to "(deleted)" — someone replaced

\# the binary on disk while it was running. RED FLAG. 🚩



Scenario 3: You need to know what environment variables a running Java process has — but you can't restart it to add debugging:



cat /proc/<pid>/environ | tr '\\0' '\\n'

\# Now you can see every env var including secrets,

\# DB connection strings, API keys — everything the

\# process was given at startup.

\# (This is also why /proc permissions matter for security)



Concept	Where You'll Use It

**FD limits	Nginx, HAProxy, any high-connection service. EKS/K8s pod tuning.**

**/proc/<pid>/fd	Debugging "too many open files" errors, connection leaks**

**/proc/<pid>/environ	Debugging misconfigured containers without restarting them**

**I/O redirection	Every shell script, every cron job, every pipeline**

**/proc/meminfo	Understanding OOM kills, memory pressure in nodes**

**/proc/loadavg	Node health monitoring, auto-scaling decisions**

**2>\&1 ordering	Getting logging right in CI/CD pipelines, systemd services**



**-----------------------------------------------------------------------------**

### **Lesson 4: Filesystem, Inodes, Storage \& Disk I/O**



**The Linux Filesystem Hierarchy — What Lives Where:**



**/           → Root of everything**

**├── /bin    → Essential user binaries (ls, cp, cat)**

**├── /sbin   → System binaries (iptables, fdisk, mount)**

**├── /etc    → Configuration files (nginx.conf, fstab, hosts)**

**├── /var    → Variable data (logs, databases, spool, cache)**

**│   ├── /var/log    → System and application logs**

**│   ├── /var/lib    → Application state data (docker, mysql)**

**│   └── /var/cache  → Cached application data**

**├── /tmp    → Temporary files (cleared on reboot)**

**├── /home   → User home directories**

**├── /opt    → Optional/third-party software**

**├── /proc   → Virtual filesystem — kernel/process info (we covered this)**

**├── /sys    → Virtual filesystem — hardware/device info**

**├── /dev    → Device files (disks, terminals, null, random)**

**└── /mnt, /media → Mount points**



Why this matters: When a disk fills up, WHERE it fills up determines the impact:



/var/log full → logging stops, debugging becomes impossible

/var/lib/docker full → containers can't start, images can't pull

/tmp full → applications that use temp files crash

/ full → system-wide catastrophe, SSH may stop working

/etc full → can't save configuration changes



###### Inodes — The Thing Nobody Checks Until It's Too Late

Everyone runs df -h. Almost nobody runs df -i. This kills production systems.



What is an inode?

Every file and directory on a Linux filesystem has an inode — a data structure that stores metadata:



\- File type (regular, directory, symlink, socket)

\- Permissions

\- Owner/Group

\- Size

\- Timestamps (access, modify, change)

\- Pointers to data blocks on disk

\- Hard link count



The inode does NOT store the filename. The filename is stored in the directory entry which maps name → inode number.



\# See inode number of a file

ls -i file.txt

\# 1234567 file.txt



\# See inode usage

df -i

\# Filesystem    Inodes   IUsed   IFree  IUse%  Mounted on

\# /dev/sda1    6553600  6553600      0   100%  /





##### The Inode Exhaustion Disaster:

A filesystem has a fixed number of inodes set at creation time. You can have plenty of disk space but zero inodes. When inodes run out:

Can't create new files

Can't create new directories

Applications crash with "No space left on device" — but df -h shows free space

Logs stop writing

Deployments fail



The #1 cause: Millions of tiny files. Common culprits:



\- Session files (/tmp/sess\_\*)

\- Mail queue files

\- Container layers (old Docker images)

\- Log rotation creating millions of small compressed logs

\- Package manager cache

\- Kubernetes pods creating temp files that never get cleaned up



###### How to Investigate:

\# Check inode usage

df -i



\# Find which directory has the most files

find / -xdev -printf '%h\\n' | sort | uniq -c | sort -rn | head -20



\# Count files in a specific directory

find /var/log -type f | wc -l



\# Find and delete old files

find /tmp -type f -mtime +7 -delete



##### Disk I/O — The Performance Dimension

Disk space and inodes are about capacity. Disk I/O is about speed. A disk can have 90% free space but be 100% saturated with I/O operations.



**Key Metrics:**

**Metric			What It Means				Tool**

**IOPS			Input/Output Operations Per Second	iostat**

**Throughput		MB/s read/written			iostat**

**Latency			How long each I/O operation takes	iostat**

**I/O Wait (%iowait)	CPU time spent waiting for disk		top, vmstat**

**Queue Depth (avgqu-sz)	How many I/O requests are waiting	iostat**

**Utilization (%util)	How busy the disk is			iostat**



Reading iostat:

iostat -xz 1

\# -x: extended stats

\# -z: skip idle devices

\# 1: refresh every 1 second



\# Output:

\# Device  r/s    w/s    rkB/s   wkB/s  rrqm/s  wrqm/s  %util  await  avgqu-sz

\# sda     150    300    2400    4800    10      50      98.5   45.2   12.3



Reading the output:



%util = 98.5% → Disk is almost fully saturated. BAD.

await = 45.2ms → Each I/O operation takes 45ms. For SSD should be <1ms, for HDD <10ms. This is SLOW.

avgqu-sz = 12.3 → 12 operations waiting in queue. Requests are piling up.

r/s + w/s = 450 → 450 IOPS. Compare against disk capability.



##### I/O Wait — The CPU Metric That's Really a Disk Problem:



top

\# %Cpu(s): 5.2 us, 2.1 sy, 0.0 ni, 30.5 id, 62.0 wa, 0.0 hi, 0.2 si

\#                                                 ^^^^^^

\#                                                 62% iowait!



62% iowait means the CPU is spending 62% of its time doing NOTHING — just waiting for disk I/O to complete. This is why your server feels "slow" even though CPU usage looks low. The CPU isn't the bottleneck — the disk is.



Finding the Guilty Process:

**# Which process is doing the most I/O?**

**iotop -oP**

**# -o: only show processes doing I/O**

**# -P: show per-process, not per-thread**



**# Alternative if iotop isn't installed**

**pidstat -d 1**



##### Hard Links vs Soft Links:



\# Hard link — same inode, different name

ln original.txt hardlink.txt



\# Both point to the SAME inode

\# Deleting original.txt doesn't affect hardlink.txt

\# Cannot cross filesystem boundaries

\# Cannot link directories



\# Soft link (symlink) — pointer to a path

ln -s original.txt symlink.txt



\# Separate inode, points to the PATH of original

\# Deleting original.txt BREAKS symlink (dangling link)

\# Can cross filesystem boundaries

\# Can link directories



Real-world use: When you see /usr/bin/python3 -> /usr/bin/python3.11 — that's a symlink. Package managers use them extensively. Kubernetes uses symlinks inside /proc and for secret/configmap volume mounts.



###### Kubernetes Secret Volumes — The Symlink Trick:

When Kubernetes mounts a Secret as a volume:



/etc/secrets/

├── ..data -> ..2024\_01\_15\_12\_00\_00.123456789   (symlink)

├── ..2024\_01\_15\_12\_00\_00.123456789/            (actual directory)

│   └── db-password

└── db-password -> ..data/db-password            (symlink)



When the Secret is updated, Kubernetes:



Creates a new timestamped directory with new values

Atomically swaps the ..data symlink to point to the new directory

Deletes the old directory

This is an atomic update — applications either see the old value or the new value, never a half-written state. Symlinks make this possible.



##### Mount Points \& fstab:



\# See all mounted filesystems

mount



\# Or cleaner

findmnt



\# /etc/fstab — filesystems to mount at boot

\# <device>        <mount point>  <type>  <options>        <dump>  <pass>

/dev/sda1         /              ext4    defaults          0       1

/dev/sdb1         /var/lib/docker ext4   defaults,noatime  0       2

UUID=abc-123      /data          xfs     defaults          0       2



Why this matters: In production, you often mount separate disks for:



/var/log — so logs filling up don't kill root

/var/lib/docker — so Docker storage is isolated

/data — application data on high-performance storage

This is called filesystem isolation and it's a fundamental production best practice.



Filesystem Types You Should Know:

Type		Used For			Notes

ext4		General purpose Linux		Most common, reliable, battle-tested

xfs		Large files, high throughput	Default on RHEL/CentOS, better for large scale

tmpfs		In-memory filesystem		Used for /tmp, Kubernetes secrets/emptyDir with medium: Memory

overlay2	Docker storage driver		Layered filesystem for container images

nfs		Network-attached storage	Shared across machines, can cause D-state processes if server dies



###### **CheatSheet:**



CAPACITY CHECKS:

&#x20; df -h          → Disk space

&#x20; df -i          → Inode usage (THE ONE EVERYONE FORGETS)

&#x20; du -sh /path   → Directory size

&#x20; ncdu /         → Interactive disk usage explorer



PERFORMANCE CHECKS:

&#x20; iostat -xz 1   → Disk I/O metrics

&#x20; iotop -oP      → Per-process I/O

&#x20; pidstat -d 1   → Per-process I/O (alternative)

&#x20; top → %iowait  → CPU waiting on disk



INVESTIGATION:

&#x20; lsof +D /path  → What processes have files open in this directory

&#x20; lsof -p <pid>  → All files a process has open

&#x20; find / -xdev -printf '%h\\n' | sort | uniq -c | sort -rn | head

&#x20;                → Find directories with most files (inode investigation)



KUBERNETES:

&#x20; Secrets use symlinks for atomic updates

&#x20; emptyDir with medium: Memory uses tmpfs

&#x20; PV/PVC abstract storage from pods





##### \# Symlinks Across the Entire DevOps/SRE Landscape



\### 1. Log Rotation (logrotate)



```bash

/var/log/nginx/access.log        → current log

/var/log/nginx/access.log.1      → yesterday's log

/var/log/nginx/access.log.2.gz   → day before, compressed

```



Some setups use symlinks to always point to the "current" log:



```bash

/var/log/app/current → /var/log/app/app.2024-01-15.log

```



When logs rotate, the symlink is updated. Monitoring tools always read from `current` without caring about the date-stamped filename.



\---



\### 2. Software Version Management



This is one of the MOST common uses in production:



```bash

Python

/usr/bin/python → /usr/bin/python3

/usr/bin/python3 → /usr/bin/python3.11



Java

/usr/bin/java → /etc/alternatives/java → /usr/lib/jvm/java-17/bin/java



Node.js (nvm)

 \~/.nvm/current →  \~/.nvm/versions/node/v18.17.0



Terraform (tfenv)

 \~/.tfenv/bin/terraform →  \~/.tfenv/versions/1.6.0/terraform



Custom application deployments

/opt/myapp/current → /opt/myapp/releases/v2.3.1

```



\*\*The `alternatives` system on Debian/RHEL uses symlinks entirely:\*\*



```bash

update-alternatives --config java

Switches the symlink chain to point to a different JDK

```



\*\*Why this matters:\*\* You can switch versions instantly by changing a symlink. No reinstallation. No path changes. Rollback is just pointing the symlink back.



\---



\### 3. Blue-Green Deployments on Bare Metal / VMs



```bash

/opt/myapp/current → /opt/myapp/releases/v2.3.1   # LIVE

/opt/myapp/releases/v2.3.0                          # Previous

/opt/myapp/releases/v2.3.1                          # Current

/opt/myapp/releases/v2.4.0                          # Staged



Deploy new version:

ln -sfn /opt/myapp/releases/v2.4.0 /opt/myapp/current

systemctl reload myapp



Rollback:

ln -sfn /opt/myapp/releases/v2.3.1 /opt/myapp/current

systemctl reload myapp

```



Capistrano, Ansible, and many deployment tools use exactly this pattern. The application always reads from `/opt/myapp/current`. Deployments and rollbacks are \*\*atomic symlink swaps\*\*.



\---



\### 4. SSL/TLS Certificate Management



```bash

/etc/ssl/certs/ca-certificates.crt → bundled CA store



Let's Encrypt / Certbot

/etc/letsencrypt/live/example.com/fullchain.pem → ../../archive/example.com/fullchain1.pem

/etc/letsencrypt/live/example.com/privkey.pem   → ../../archive/example.com/privkey1.pem



On renewal, certbot updates the symlinks to point 

to the new cert files without changing the path 

that Nginx/Apache is configured to read from

```



\*\*Why this matters:\*\* Nginx config says `ssl \_certificate /etc/letsencrypt/live/example.com/fullchain.pem`. When the cert renews, the symlink silently points to the new cert. Nginx reload picks it up. Zero config changes.



\---



\### 5. Docker \& Container Runtime



```bash

Docker data root

/var/lib/docker   # Default location



Some teams symlink this to a larger/faster disk

/var/lib/docker → /mnt/fast-ssd/docker



Container layers use overlay2 which relies on symlinks internally

```



\---



\### 6. Ansible Roles \& Inventory



```bash

Shared roles across projects

/etc/ansible/roles/common → /opt/shared-ansible-roles/common



Environment-specific inventory

/etc/ansible/inventory/current → /etc/ansible/inventory/production

```



\---



\### 7. Systemd Service Management



```bash

Enabling a service creates a symlink

systemctl enable nginx

Creates: /etc/systemd/system/multi-user.target.wants/nginx.service 

→ /lib/systemd/system/nginx.service



Disabling removes the symlink

systemctl disable nginx



The entire systemd enable/disable mechanism IS symlinks

```



\---



\### 8. Terraform \& IaC Workspace Management



```bash

.terraform directory symlinks to provider binaries in plugin cache

 \~/.terraform.d/plugin-cache/registry.terraform.io/hashicorp/aws/5.0.0/linux \_amd64/

Terraform creates symlinks from .terraform/providers → cache

Saves disk space and download time across multiple projects

```



\---



\### 9. Cron Jobs \& Script Management



```bash

Instead of editing crontab directly

/etc/cron.daily/backup → /opt/scripts/backup.sh

/etc/cron.hourly/healthcheck → /opt/scripts/healthcheck.sh

Centralize scripts in one place, symlink into cron directories

```



\---



\### 10. Kubernetes (Yes, It Belongs Here Too — But As ONE Item, Not The Only One)



```

- Secret/ConfigMap volume mounts use symlinks for atomic updates

- hostPath mounts may follow symlinks (security consideration)

- Container filesystem layers use symlinks extensively

```



\---



\## Key Principle:



>  \* \*Symlinks are the universal mechanism for indirection in Linux. \* \* Anywhere you need to decouple "where something points" from "the path that consumers use" — symlinks are the answer. Version switching, deployments, certificate rotation, service management, log management — all symlinks.



\---



\# Now — The Troubleshooting Concepts I Should Have TAUGHT, Not Asked



\## Concept: Deleted Files Still Holding Disk Space



When you `rm` a file, Linux removes the \*\*directory entry\*\* (the name → inode mapping). But the inode and data blocks are NOT freed until \*\*every process that has the file open closes it\*\*.



```bash

You delete an 80GB log file

rm /var/log/nginx/access.log



 But Nginx still has a file descriptor open to it

 The kernel sees: "inode still has open FD references, 

 can't free the blocks yet"



 Space is NOT reclaimed

df -h  # Still shows 95%

```



\### How to Find These Ghost Files:



```bash

lsof +L1

 Lists all open files that have been "unlinked" (deleted)

 +L1 means "link count less than 1" = deleted but still open



 Output:

 COMMAND  PID  USER  FD  TYPE  SIZE      NLINK  NAME

 nginx    1234 root  4w  REG   80G       0      /var/log/nginx/access.log (deleted)

```



\### How to Actually Reclaim the Space:



\*\*Option 1:\*\* Restart the process holding the file open



```bash

systemctl restart nginx

 Process closes all FDs → kernel frees the blocks → space reclaimed

```



\*\*Option 2:\*\* Truncate instead of delete (THE SMART WAY)



```bash

 Instead of rm, truncate the file to zero bytes

truncate -s 0 /var/log/nginx/access.log

 OR

> /var/log/nginx/access.log



 File still exists, FD still valid, but size is now 0

 Space is reclaimed IMMEDIATELY

 Nginx keeps writing to the same file — no restart needed

```



\*\*Option 3:\*\* If you already deleted it, empty the FD directly



```bash

 Find the FD number from lsof output (say FD 4)

 Write nothing to it via /proc

: > /proc/1234/fd/4

 This truncates the deleted file through the still-open FD

```



\### Prevention:



```bash

 1. Use logrotate with copytruncate

/var/log/nginx/access.log {

   daily

   rotate 7

   compress

   copytruncate    # Truncates original instead of moving it

                   # No need to restart nginx

}



 2. Monitor for deleted-but-open files

lsof +L1 | awk '{sum += $7} END {print sum/1024/1024/1024 " GB held by deleted files"}'



 3. Separate /var/log onto its own mount point

 So log bloat never fills root filesystem

```



\---



\## Concept: Inode Exhaustion on Servers



When inodes hit 100%, no new files can be created even with plenty of disk space.



\### Common Culprits by Environment:



| Environment | Usual Culprit |

|-------------|---------------|

| \*\*Web Servers\*\* | PHP session files in `/tmp`, mail queue in `/var/spool` |

| \*\*CI/CD Agents\*\* | Build artifacts, workspace temp files never cleaned |

| \*\*Docker Hosts\*\* | Overlay2 layers from old images, stopped containers |

| \*\*Any Server\*\* | Runaway log rotation creating millions of `.gz` files |

| \*\*Package Managers\*\* | Cached packages in `/var/cache/apt` or `/var/cache/yum` |



\### Immediate Fix:



```bash

 Find directories with the most files

find / -xdev -printf '%hn' | sort | uniq -c | sort -rn | head -20



 Once identified, delete the offenders

find /tmp -type f -mtime +7 -delete

find /var/spool/mail -type f -delete

docker system prune -af   # On Docker hosts

```



\### Long-Term Prevention:



```bash

 1. Cron job to clean temp files

0 2  \*  \*  \* find /tmp -type f -mtime +3 -delete



 2. Monitor inode usage in Prometheus

 node \_exporter exposes: node \_filesystem \_files \_free



 3. Create filesystem with more inodes if needed

mkfs.ext4 -N 10000000 /dev/sdb1

 -N sets the number of inodes at creation time



 4. Use XFS for large-scale systems

 XFS allocates inodes dynamically — doesn't have fixed inode count

```



\---



\# 🔥 NOW — Fair Troubleshooting Questions



Everything below is based \*\*solely\*\* on concepts I've just taught you.



\### Question 1:



You truncated a large log file using `> /var/log/app/debug.log`. Space was reclaimed. But 10 minutes later, the file is \*\*50GB again\*\* and growing fast.



1\. \*\*What's happening?\*\*

2\. \*\*What's your fix — both immediate and permanent?\*\*



\### Question 2:



An engineer says: \*"I upgraded our app from Java 11 to Java 17 using `update-alternatives`. But when I run `java -version`, it still shows Java 11."\*



Based on what you learned about symlinks, \*\*what's the most likely cause and how do you debug it?\*\*





##### **Strace — Your Last Resort Debugging Weapon**

When nothing else tells you why a process is misbehaving, strace shows you every system call the process makes. It's an X-ray into what the process is actually doing at the kernel level.



bash

\# Trace a running process

strace -p <pid>



\# Trace with timestamps

strace -tt -p <pid>



\# Trace only specific system calls

strace -e open,read,write -p <pid>    # File operations

strace -e connect,accept,sendto -p <pid>  # Network operations

strace -e openat -p <pid>             # What files is it opening?



\# Trace a new command from start

strace -f -o /tmp/trace.log command args

\# -f: follow child processes (forks)

\# -o: write to file instead of stderr



\# Count system calls (summary)

strace -c -p <pid>

\# Shows a table of which syscalls were called how many times

\# and how much time was spent in each

\# INCREDIBLE for finding bottlenecks

Real-World Uses:

"App won't start but gives no useful error message":



bash

strace -f ./myapp 2>\&1 | tail -50

\# Look for the LAST system call before it dies

\# Often you'll see:

\# openat(AT\_FDCWD, "/etc/myapp/config.yml", O\_RDONLY) = -1 ENOENT

\# (No such file or directory)

\# AH HA — it's looking for a config file that doesn't exist

"App is slow but CPU and memory are fine":



bash

strace -c -p <pid>

\# % time     seconds  usecs/call     calls    syscall

\# ------ ----------- ----------- --------- ---------

\#  89.42    4.234567        4234      1000    futex

\#   5.21    0.246789         246      1000    read

\#

\# 89% of time in futex = lock contention

\# The app is spending most of its time waiting on locks

"App can't connect to the database":



bash

strace -e connect -p <pid>

\# connect(5, {sa\_family=AF\_INET, sin\_port=htons(5432),

\#   sin\_addr=inet\_addr("10.0.0.5")}, 16) = -1 ETIMEDOUT

\#

\# Now you know the EXACT IP and port it's trying to reach

\# And that it's timing out — not connection refused

\# This tells you it's a NETWORK/FIREWALL issue, not a service-down issue

⚠️ WARNING: strace adds significant overhead to the traced process. It can slow it down 10-100x. Never leave strace attached to a production process longer than necessary. Attach, get your data, detach.





##### **Core Dumps Filling Disk**

When a process crashes with a segfault or abort, Linux can write a core dump — a snapshot of the process's memory at the time of the crash. These can be HUGE.



bash

\# Check if core dumps are enabled

ulimit -c

\# 0 = disabled

\# unlimited = enabled with no size limit ← DANGEROUS



\# Where do core dumps go?

cat /proc/sys/kernel/core\_pattern

\# Possible values:

\# core                          → in the process's working directory

\# /var/crash/core.%e.%p.%t     → centralized with metadata

\# |/usr/lib/systemd/systemd-coredump  → systemd manages them

The Production Disaster:

An app keeps segfaulting in a loop. Each crash writes a 2GB core dump. Restart policy restarts it. Crashes again. Another 2GB dump. Within an hour: disk full.



The Fix:

bash

\# Option 1: Disable core dumps entirely (common in production)

echo 0 > /proc/sys/kernel/core\_pattern

\# Or in limits.conf:

\* hard core 0



\# Option 2: Let systemd manage them with size limits

\# /etc/systemd/coredump.conf

\[Coredump]

Storage=external

MaxUse=1G           # Max total storage for core dumps

ProcessSizeMax=500M # Max size per dump

\-----------------------------------------------------------------------

### Special Permissions — setuid, setgid, sticky bit





##### setuid (Set User ID) — The Dangerous One

When set on an executable file, it runs as the file's owner instead of the user who launched it.



bash

ls -la /usr/bin/passwd

\# -rwsr-xr-x 1 root root 68208 /usr/bin/passwd

\#    ^

\#    s = setuid bit



\# When ANY user runs 'passwd', it executes as ROOT

\# This is how regular users can change their own password

\# (because /etc/shadow is only writable by root)

The Security Danger:



If an attacker can place a setuid-root binary on your system, they have instant root access:



bash

\# Finding all setuid binaries (security audit)

find / -perm -4000 -type f 2>/dev/null

\# Review this list. Any unexpected setuid binaries = compromised system



\# Remove setuid from a binary

chmod u-s /path/to/binary

In production/containers:



yaml

\# Kubernetes — drop ALL capabilities and prevent privilege escalation

securityContext:

&#x20; allowPrivilegeEscalation: false  # Prevents setuid from working

&#x20; runAsNonRoot: true

&#x20; capabilities:

&#x20;   drop:

&#x20;     - ALL



##### setgid (Set Group ID)

On files: Runs as the file's group instead of the user's group.



On directories: This is where it's actually useful. Any file created inside the directory inherits the directory's group instead of the creator's primary group:



bash

\# Shared project directory

mkdir /opt/project

chgrp developers /opt/project

chmod 2775 /opt/project

\#     ^

\#     2 = setgid bit



\# Now when anyone in the developers group creates a file:

touch /opt/project/newfile.txt

ls -la /opt/project/newfile.txt

\# -rw-rw-r-- 1 deploy developers ...

\#                     ^^^^^^^^^^

\#                     Inherited from directory, not user's primary group

This is essential for shared directories where multiple users need to read/write each other's files.



##### Sticky Bit

On directories: Only the file's owner (or root) can delete or rename files within the directory, even if others have write permission.



bash

ls -ld /tmp

\# drwxrwxrwt 15 root root 4096 Jan 15 03:00 /tmp

\#          ^

\#          t = sticky bit



\# Everyone can write to /tmp

\# But user A can't delete user B's files in /tmp

\# Only the file owner or root can delete



\# Set sticky bit

chmod 1777 /tmp

\#     ^

\#     1 = sticky bit

Numeric Special Permission Bits:

text

4000 = setuid

2000 = setgid

1000 = sticky bit



\# Combined with regular permissions:

chmod 4755 binary      # setuid + rwxr-xr-x

chmod 2775 directory   # setgid + rwxrwxr-x

chmod 1777 /tmp        # sticky + rwxrwxrwx

umask — Default Permission Control

When a new file or directory is created, its permissions are determined by the umask:



bash

umask

\# 0022



\# For files:   666 - 022 = 644 (rw-r--r--)

\# For directories: 777 - 022 = 755 (rwxr-xr-x)



\# More restrictive umask

umask 077

\# Files: 666 - 077 = 600 (rw-------)

\# Dirs:  777 - 077 = 700 (rwx------)



\# Set in /etc/profile or \~/.bashrc for persistence

Why base is 666 for files and not 777: Linux deliberately doesn't give execute permission by default. You must explicitly chmod +x. This prevents accidentally creating executable files.



sudo — Privilege Escalation Done Right

bash

\# The sudo config file

visudo   # ALWAYS use visudo to edit — it validates syntax

&#x20;        # A syntax error in sudoers = NOBODY can sudo = locked out



\# /etc/sudoers format:

\# user   HOST=(RUNAS\_USER:RUNAS\_GROUP)  COMMANDS

deploy   ALL=(ALL:ALL)                  ALL           # Full sudo access

nginx    ALL=(ALL)                      NOPASSWD: /bin/systemctl reload nginx

\#                                       ^^^^^^^^

\#                                       No password required for this specific command



\# Group-based sudo (recommended)

%sudo    ALL=(ALL:ALL) ALL

%devops  ALL=(ALL) NOPASSWD: ALL

\# % means group

sudo Best Practices in Production:

bash

\# 1. NEVER give blanket NOPASSWD: ALL to individual users

\# Use specific commands:

deploy ALL=(ALL) NOPASSWD: /bin/systemctl restart myapp, /bin/journalctl



\# 2. Use groups, not individual users

\# /etc/sudoers.d/devops

%devops ALL=(ALL) NOPASSWD: ALL



\# 3. Drop files in /etc/sudoers.d/ instead of editing /etc/sudoers directly

\# Easier to manage with Ansible/Puppet/Chef



\# 4. Log all sudo commands (usually enabled by default)

\# Check: /var/log/auth.log or /var/log/secure

grep sudo /var/log/auth.log



\# 5. Restrict root login

\# /etc/ssh/sshd\_config

PermitRootLogin no

\# Force everyone to use their own account + sudo

\# This gives you an AUDIT TRAIL of who did what

Production Scenarios — Permission Issues

Scenario 1: Application Can't Write Logs After Deployment

bash

\# Ansible deploys new code as root

\# Files now owned by root:root

\# Application runs as 'appuser'

\# Application can't write to /var/log/app/



\# The fix — in your Ansible playbook:

\- name: Set correct ownership

&#x20; file:

&#x20;   path: /var/log/app/

&#x20;   owner: appuser

&#x20;   group: appgroup

&#x20;   mode: '0755'

&#x20;   recurse: yes



\# Or better — use logrotate with create directive:

/var/log/app/\*.log {

&#x20;   create 0644 appuser appgroup

&#x20;   # New log files created with correct ownership

}

Scenario 2: Docker Volume Mount Permission Denied

bash

\# Container runs as UID 1000 (appuser inside container)

\# Host directory /data owned by root:root with 755

\# Container tries to write to /data → Permission denied



\# Fix 1: Match UIDs

chown 1000:1000 /data



\# Fix 2: Run container as specific user

docker run -u 1000:1000 -v /data:/data myapp



\# Fix 3: Use init container in Kubernetes to fix permissions

initContainers:

\- name: fix-permissions

&#x20; image: busybox

&#x20; command: \['sh', '-c', 'chown -R 1000:1000 /data']

&#x20; volumeMounts:

&#x20; - name: data

&#x20;   mountPath: /data

Scenario 3: SSH Key Authentication Fails Despite Correct Key

bash

\# SSH is EXTREMELY strict about permissions

\# If any of these are wrong, key auth silently fails:



chmod 700 \~/.ssh

chmod 600 \~/.ssh/authorized\_keys

chmod 600 \~/.ssh/id\_rsa          # Private key

chmod 644 \~/.ssh/id\_rsa.pub      # Public key

chown -R user:user \~/.ssh



\# Debug SSH auth failures:

ssh -vvv user@host   # Verbose output shows exactly where auth fails



\# Server-side logs:

tail -f /var/log/auth.log

\# "Authentication refused: bad ownership or modes for directory /home/user/.ssh"

Scenario 4: Cron Job Works Manually But Fails in Crontab

bash

\# Common causes:



\# 1. PATH is different in cron environment

\# Cron has minimal PATH: /usr/bin:/bin

\# Your script uses /usr/local/bin/terraform → not found

\# Fix: Use full paths in cron scripts

0 2 \* \* \* /usr/local/bin/terraform apply -auto-approve



\# 2. Script not executable

chmod +x /opt/scripts/backup.sh



\# 3. Permission to write output

\# Cron tries to email output but sendmail isn't configured

\# Redirect output explicitly:

0 2 \* \* \* /opt/scripts/backup.sh >> /var/log/backup.log 2>\&1



\# 4. Environment variables not available

\# Cron doesn't load .bashrc or .profile

\# Define needed vars in the crontab:

SHELL=/bin/bash

PATH=/usr/local/bin:/usr/bin:/bin

AWS\_DEFAULT\_REGION=us-east-1

0 2 \* \* \* /opt/scripts/backup.sh

Scenario 5: "Operation not permitted" Even as Root

bash

\# You're root. You try to delete a file. Permission denied.

rm -f /etc/important.conf

\# rm: cannot remove '/etc/important.conf': Operation not permitted



\# What? You're ROOT. How?



\# Answer: File has IMMUTABLE attribute set

lsattr /etc/important.conf

\# ----i--------e-- /etc/important.conf

\#     ^

\#     i = immutable — even root can't modify or delete



\# Remove immutable attribute

chattr -i /etc/important.conf



\# Now you can delete/modify it



\# Set immutable (useful for protecting critical configs)

chattr +i /etc/important.conf



\# Other useful attributes:

chattr +a /var/log/audit.log    # Append-only — can add but not delete/modify

&#x20;                                 # Perfect for audit logs

Summary — Permission Debugging Checklist:

text

When "Permission denied" hits, check IN THIS ORDER:



1\. File/directory ownership:     ls -la <path>

2\. File/directory permissions:   ls -la <path>

3\. Parent directory permissions: ls -la <parent>  (need x to traverse)

4\. Special attributes:           lsattr <path>    (immutable? append-only?)

5\. SELinux/AppArmor:             getenforce        (is MAC enforcing?)

&#x20;                                 ls -laZ <path>   (SELinux context)

6\. ACLs:                         getfacl <path>    (extended ACLs overriding?)

7\. Mount options:                mount | grep <fs>  (mounted read-only? noexec?)

8\. Capabilities:                 getcap <binary>    (Linux capabilities)

Speaking of which — I mentioned SELinux and ACLs. Let me cover those.



SELinux / AppArmor — Mandatory Access Control (MAC)

Regular permissions (rwx) are Discretionary Access Control (DAC) — the file owner decides who gets access. SELinux and AppArmor are Mandatory Access Control — the SYSTEM decides, and even root can be restricted.



SELinux (Red Hat, CentOS, Fedora, Amazon Linux):

bash

\# Check status

getenforce

\# Enforcing = actively blocking violations

\# Permissive = logging violations but not blocking

\# Disabled = off completely



\# View SELinux context of files

ls -laZ /var/www/html/

\# -rw-r--r--. root root unconfined\_u:object\_r:httpd\_sys\_content\_t:s0 index.html

\#                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

\#                        SELinux context



\# View SELinux context of processes

ps auxZ | grep httpd

\# system\_u:system\_r:httpd\_t:s0  root  1234 ... /usr/sbin/httpd

\#                   ^^^^^^^^

\#                   httpd runs in httpd\_t domain

\#                   It can ONLY access files labeled httpd\_sys\_content\_t

The Classic SELinux Production Disaster:



bash

\# You deploy a new app that writes to /opt/myapp/data/

\# App runs fine on your dev box (SELinux disabled)

\# Deploy to production (SELinux enforcing) → Permission denied

\# rwx permissions look correct

\# Ownership looks correct

\# You spend 3 hours going insane



\# Check the audit log:

ausearch -m AVC -ts recent

\# type=AVC msg=audit: avc:  denied  { write } for pid=1234

\# comm="myapp" name="data" scontext=system\_u:system\_r:myapp\_t

\# tcontext=system\_u:object\_r:default\_t



\# The directory has default\_t context — wrong!

\# Fix:

semanage fcontext -a -t myapp\_data\_t "/opt/myapp/data(/.\*)?"

restorecon -Rv /opt/myapp/data/



\# Or if you don't have a custom policy:

chcon -R -t httpd\_sys\_rw\_content\_t /opt/myapp/data/



\# QUICK DIAGNOSTIC: Temporarily set permissive to confirm SELinux is the issue

setenforce 0    # Permissive (logs but doesn't block)

\# Test your app — if it works now, SELinux was the problem

setenforce 1    # Back to enforcing

\# Then fix the contexts properly

AppArmor (Ubuntu, Debian, SUSE):

bash

\# Check status

aa-status



\# Profiles can be in:

\# enforce — blocking violations

\# complain — logging violations (like SELinux permissive)



\# View loaded profiles

cat /sys/kernel/security/apparmor/profiles



\# If AppArmor blocks your app:

\# Check logs

journalctl | grep apparmor



\# Temporarily disable profile for a binary

aa-complain /usr/sbin/nginx



\# Create/modify profile

aa-genprof /usr/sbin/myapp

ACLs — Access Control Lists (Fine-Grained Permissions)

When standard owner/group/others isn't enough, ACLs let you set permissions for specific users or groups:



bash

\# View ACLs

getfacl /opt/project/



\# Set ACL — give 'deploy' user read+write, without changing ownership

setfacl -m u:deploy:rw /opt/project/data.txt



\# Set ACL — give 'devops' group full access

setfacl -m g:devops:rwx /opt/project/



\# Set default ACL (inherited by new files created in directory)

setfacl -d -m g:devops:rwx /opt/project/



\# Remove ACL

setfacl -x u:deploy /opt/project/data.txt



\# Remove ALL ACLs

setfacl -b /opt/project/data.txt



\# A + sign in ls -la indicates ACLs are set:

ls -la /opt/project/

\# drwxrwx---+ 2 root root 4096 ...

\#           ^

\#           + means ACLs exist — check with getfacl

Real-world use: Shared CI/CD build directories where multiple service accounts need different levels of access but you don't want to create a group for every combination.



Linux Capabilities — Granular Root Powers

Instead of giving a process full root access, Linux capabilities let you grant specific root privileges:



bash

\# View capabilities of a binary

getcap /usr/bin/ping

\# /usr/bin/ping = cap\_net\_raw+ep

\# This is why regular users can run ping without root

\# It only has the NET\_RAW capability, not full root



\# Common capabilities:

\# CAP\_NET\_BIND\_SERVICE — bind to ports below 1024

\# CAP\_NET\_RAW — raw sockets (ping, tcpdump)

\# CAP\_SYS\_ADMIN — various admin operations (almost as powerful as root)

\# CAP\_CHOWN — change file ownership

\# CAP\_DAC\_OVERRIDE — bypass file permission checks



\# Set capability on a binary

setcap cap\_net\_bind\_service+ep /usr/local/bin/myapp

\# Now myapp can bind to port 80 without running as root



\# In Kubernetes:

securityContext:

&#x20; capabilities:

&#x20;   add:

&#x20;     - NET\_BIND\_SERVICE    # Only this specific privilege

&#x20;   drop:

&#x20;     - ALL                 # Drop everything else

Why this matters: Running containers as root is a massive security risk. Capabilities let you give containers ONLY the specific privileges they need. In FAANG-level security reviews, drop: ALL + selective add is mandatory.





USERS \& GROUPS:

&#x20; whoami / id                              → Current user info

&#x20; useradd -m -s /bin/bash <user>          → Create user

&#x20; usermod -aG <group> <user>              → Add to group (ALWAYS use -a!)

&#x20; /etc/passwd                              → User accounts

&#x20; /etc/shadow                              → Password hashes

&#x20; /etc/group                               → Group memberships



PERMISSIONS:

&#x20; chmod 755 file        → rwxr-xr-x

&#x20; chmod 600 file        → rw-------

&#x20; chown user:group file → Change ownership

&#x20; r=4, w=2, x=1        → Octal math



SPECIAL BITS:

&#x20; 4000 = setuid         → Runs as file owner (DANGEROUS)

&#x20; 2000 = setgid         → Inherits directory group

&#x20; 1000 = sticky bit     → Only owner can delete (e.g., /tmp)



UMASK:

&#x20; umask 022             → Files: 644, Dirs: 755

&#x20; umask 077             → Files: 600, Dirs: 700



SUDO:

&#x20; visudo                → ALWAYS use this to edit sudoers

&#x20; usermod -aG sudo user → Grant sudo via group

&#x20; /etc/sudoers.d/       → Drop-in files (Ansible-friendly)



SPECIAL ATTRIBUTES:

&#x20; lsattr / chattr       → Immutable (+i), append-only (+a)

&#x20; chattr +i file        → Even root can't delete



SELINUX:

&#x20; getenforce            → Check status

&#x20; setenforce 0          → Permissive (test mode)

&#x20; ls -Z                 → View SELinux contexts

&#x20; restorecon -Rv /path  → Fix contexts

&#x20; ausearch -m AVC       → Check denial logs



APPARMOR:

&#x20; aa-status             → Check status

&#x20; aa-complain <profile> → Switch to complain mode



ACLS:

&#x20; getfacl / setfacl     → Fine-grained per-user/group permissions

&#x20; setfacl -m u:user:rwx → Grant specific user access

&#x20; + in ls -la           → Indicates ACLs present



CAPABILITIES:

&#x20; getcap / setcap       → View/set granular root privileges

&#x20; cap\_net\_bind\_service  → Bind to port <1024 without root



KUBERNETES SECURITY:

&#x20; runAsNonRoot: true

&#x20; runAsUser: 1000

&#x20; allowPrivilegeEscalation: false

&#x20; readOnlyRootFilesystem: true

&#x20; capabilities: drop: \[ALL]



PERMISSION DEBUGGING ORDER:

&#x20; 1. ls -la             → ownership \& permissions

&#x20; 2. ls -la parent dir  → traversal (x bit)

&#x20; 3. lsattr             → immutable/append-only

&#x20; 4. getenforce / ls -Z → SELinux

&#x20; 5. getfacl             → ACLs

&#x20; 6. mount               → read-only mount?

&#x20; 7. getcap              → capabilities

================================================================

#### Lesson 6: cgroups \& namespaces — The Foundation of Containers



\### The Big Picture



A container is NOT a virtual machine. There's no separate kernel. There's no hypervisor. A container is just a \*\*regular Linux process\*\* with two restrictions:



1\. \*\*Namespaces\*\* — restrict what the process can \*\*SEE\*\*

2\. \*\*cgroups\*\* — restrict what the process can \*\*USE\*\*



That's it. That's the entire foundation. Everything else is tooling on top.



\---



###### \### Part 1: Linux Namespaces — Isolation of Visibility



Namespaces partition kernel resources so that one set of processes sees one set of resources, and another set sees a different set.



\#### The 8 Namespace Types:



| Namespace | Flag | What It Isolates | Container Impact |

|-----------|------|-----------------|------------------|

| \*\*PID\*\* | CLONE\_NEWPID | Process IDs | Container sees only its own processes. PID 1 inside = your app |

| \*\*NET\*\* | CLONE\_NEWNET | Network stack | Container gets its own IP, ports, routing table, iptables |

| \*\*MNT\*\* | CLONE\_NEWNS | Mount points | Container sees its own filesystem, not the host's |

| \*\*UTS\*\* | CLONE\_NEWUTS | Hostname \& domain | Container can have its own hostname |

| \*\*IPC\*\* | CLONE\_NEWIPC | Inter-process communication | Shared memory, semaphores isolated between containers |

| \*\*USER\*\* | CLONE\_NEWUSER | User/Group IDs | UID 0 inside container can map to UID 1000 on host (rootless containers) |

| \*\*CGROUP\*\* | CLONE\_NEWCGROUP | Cgroup root view | Container sees only its own cgroup hierarchy |

| \*\*TIME\*\* | CLONE\_NEWTIME | System clocks | Container can have different boot/monotonic clocks (Linux 5.6+) |



\#### How to See Namespaces:



```bash

 Every process has namespace references in /proc

ls -la /proc/self/ns/

 lrwxrwxrwx 1 root root 0 Jan 15 cgroup -> 'cgroup: \[4026531835]'

 lrwxrwxrwx 1 root root 0 Jan 15 ipc -> 'ipc: \[4026531839]'

 lrwxrwxrwx 1 root root 0 Jan 15 mnt -> 'mnt: \[4026531840]'

 lrwxrwxrwx 1 root root 0 Jan 15 net -> 'net: \[4026531992]'

 lrwxrwxrwx 1 root root 0 Jan 15 pid -> 'pid: \[4026531836]'

 lrwxrwxrwx 1 root root 0 Jan 15 user -> 'user: \[4026531837]'

 lrwxrwxrwx 1 root root 0 Jan 15 uts -> 'uts: \[4026531838]'



 Each number in brackets is a unique namespace ID

 Processes in the same namespace share the same ID

 Processes in different namespaces have different IDs



 Compare host process vs container process:

ls -la /proc/1/ns/net        # Host's PID 1 network namespace

ls -la /proc/<container-pid>/ns/net  # Container's network namespace

 Different IDs = isolated network stacks

```



\#### PID Namespace — Why Containers See PID 1:



```bash

 On the HOST, your container's main process might be PID 28456

ps aux | grep myapp

 root  28456 ... myapp



 But INSIDE the container, the same process is PID 1

docker exec mycontainer ps aux

 PID  USER  COMMAND

 1    root  myapp



 Same process, different PID namespace

 The container can't see any host processes

 The host CAN see the container's process (host namespace encompasses all)

```



\#### NET Namespace — How Container Networking Actually Works:



```bash

 Each container gets its own network namespace with:

 - Its own eth0 interface

 - Its own IP address

 - Its own routing table

 - Its own iptables rules

 - Its own port space (container A and B can both use port 80)



 Docker creates a veth pair (virtual ethernet cable):

 One end inside the container namespace (eth0)

 Other end on the host, attached to docker0 bridge



 View network namespaces

ip netns list



 Execute a command in a specific network namespace

ip netns exec <namespace> ip addr show



 This is exactly what 'docker exec' does under the hood

 for network operations

```



\#### MNT Namespace — How Containers Have Their Own Filesystem:



```bash

 The container sees a different mount table than the host

 Docker uses overlay2 filesystem to layer:

   Base image layers (read-only)

   + Container layer (read-write)

   = Union filesystem presented to the container



 Inside container:

mount

 overlay on / type overlay (lowerdir=...,upperdir=...,workdir=...)



 Host can see the overlay layers:

docker inspect <container> | grep -A 5 "GraphDriver"

 "LowerDir": "/var/lib/docker/overlay2/<id>/diff",

 "UpperDir": "/var/lib/docker/overlay2/<id>/diff",

 "WorkDir":  "/var/lib/docker/overlay2/<id>/work"

```



\#### USER Namespace — Rootless Containers:



```bash

 Without user namespace:

 UID 0 (root) inside container = UID 0 (root) on host

 If container breaks out → ROOT ON HOST → game over



 With user namespace (rootless mode):

 UID 0 inside container = UID 100000 on host (unprivileged)

 If container breaks out → unprivileged user on host → limited damage



 Check if Docker is running rootless

docker info | grep -i rootless



 Enable rootless Docker

dockerd-rootless-setuptool.sh install



 Kubernetes + rootless:

 Kubernetes 1.22+ supports rootless mode via containerd

 This is a FAANG-level security requirement

```



\#### Entering a Container's Namespace — nsenter:



```bash

 docker exec is convenient, but nsenter gives you MORE control

 and works even if the container has no shell



 Find the container's PID on the host

docker inspect --format '{{.State.Pid}}' <container>

 28456



 Enter ALL namespaces of the container

nsenter -t 28456 -m -u -i -n -p -- /bin/sh

 -t = target PID

 -m = mount namespace

 -u = UTS namespace

 -i = IPC namespace

 -n = network namespace

 -p = PID namespace



 Enter ONLY the network namespace (for network debugging)

nsenter -t 28456 -n -- ip addr show

nsenter -t 28456 -n -- ss -tlnp

nsenter -t 28456 -n -- curl http://10.0.0.5:8080



 This is INCREDIBLY powerful for debugging:

 Container has no curl? No ss? No ip?

 Use nsenter to run HOST tools in the container's namespace

 You get the host's full toolkit with the container's network view

```



\*\*This is a top 1% debugging technique.\*\* When a container's image is minimal (distroless, scratch) and has no debugging tools, `nsenter` lets you use host tools inside the container's isolated environment.



\---



###### \### Part 2: cgroups — Resource Control



cgroups (Control Groups) limit HOW MUCH of the system's resources a process can consume. Without cgroups, a single container could eat all CPU, all memory, all I/O, and starve everything else.



\#### cgroup v1 vs v2:



```bash

 Check which version your system uses

stat -fc %T /sys/fs/cgroup/

 tmpfs = cgroup v1

 cgroup2fs = cgroup v2



 Or

mount | grep cgroup

 cgroup2 on /sys/fs/cgroup type cgroup2 → v2

 cgroup on /sys/fs/cgroup/memory type cgroup → v1

```



| Feature | cgroup v1 | cgroup v2 |

|---------|-----------|-----------|

| Structure | Multiple hierarchies (one per controller) | Single unified hierarchy |

| Adoption | Legacy, still common | Modern, default in newer kernels |

| Kubernetes | Supported | Required for some features (MemoryQoS, swap) |

| Location | `/sys/fs/cgroup/<controller>/` | `/sys/fs/cgroup/` |



\#### What cgroups Control:



| Controller | What It Limits | Why It Matters |

|-----------|---------------|----------------|

| \*\*memory\*\* | RAM usage, swap | Prevents OOM from noisy neighbors |

| \*\*cpu\*\* | CPU time/shares | Fair CPU allocation |

| \*\*cpuset\*\* | Which CPU cores | Pin workloads to specific cores |

| \*\*blkio / io\*\* | Disk I/O bandwidth | Prevent one container from saturating disk |

| \*\*pids\*\* | Number of processes | Prevents fork bombs |

| \*\*devices\*\* | Device access | Security — block access to /dev/sda etc |

| \*\*freezer\*\* | Pause/resume processes | Used by checkpoint/restore |



\#### Viewing cgroup Limits and Usage:



```bash

 For a Docker container:

 Find its cgroup path

docker inspect --format '{{.HostConfig.CgroupParent}}' <container>



 cgroup v2 — everything under one hierarchy

cat /sys/fs/cgroup/system.slice/docker-<container-id>.scope/memory.max

 536870912 (512MB in bytes)



cat /sys/fs/cgroup/system.slice/docker-<container-id>.scope/memory.current

 234567890 (current usage in bytes)



 cgroup v1 — separate directories per controller

cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit                                      \_in                                      \_bytes

cat /sys/fs/cgroup/memory/docker/<container-id>/memory.usage                                      \_in                                      \_bytes

cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.shares



 For Kubernetes pods, the path includes the QoS class:

 /sys/fs/cgroup/kubepods/burstable/pod<uid>/<container-id>/

 /sys/fs/cgroup/kubepods/guaranteed/pod<uid>/<container-id>/

 /sys/fs/cgroup/kubepods/besteffort/pod<uid>/<container-id>/

```



\#### How Docker Maps Resource Flags to cgroups:



```bash

docker run --memory=512m --cpus=1.5 --pids-limit=100 myapp



 This creates cgroup entries:

 memory.max = 536870912 (512MB)

 cpu.max = 150000 100000 (150ms of every 100ms period = 1.5 CPUs)

 pids.max = 100

```



\#### How Kubernetes Maps Resources to cgroups:



```yaml

resources:

 requests:

   memory: "256Mi"   # Used for SCHEDULING (which node has room?)

   cpu: "250m"       # 250 millicores = 0.25 CPU

 limits:

   memory: "512Mi"   # Hard limit → OOM kill if exceeded

   cpu: "1000m"      # 1 full CPU → throttled if exceeded

```



```bash

 What Kubernetes sets in cgroups:



 Memory limit → memory.max = 536870912

 Memory request → used by scheduler, not directly in cgroup



 CPU limit → cpu.max = 100000 100000 (100ms per 100ms period = 1 CPU)

 CPU request → cpu.weight (proportional sharing)



 CRITICAL DIFFERENCE:

 Memory limit exceeded → process KILLED (OOM)

 CPU limit exceeded → process THROTTLED (slowed down, not killed)

```



\#### CPU Throttling — The Silent Performance Killer:



```bash

 Your pod has:

 cpu limit: 500m (half a CPU)

 Your app occasionally needs a burst of CPU for 200ms



 What happens:

 The cgroup says: "you get 50ms of CPU per 100ms period"

 Your app uses its 50ms in the first 50ms of the period

 For the remaining 50ms, the kernel PAUSES your process

 This is called THROTTLING



 Check if a container is being throttled:

cat /sys/fs/cgroup/.../cpu.stat

 usage \_usec 123456789

 user \_usec 100000000

 system \_usec 23456789

 nr \_periods 50000

 nr \_throttled 12000    ← THROTTLED 12,000 times!

 throttled \_usec 600000000  ← 600 seconds total throttle time



 In Kubernetes, this shows up as:

 - Increased latency (P99 spikes)

 - Slow response times

 - App appears "laggy" but CPU usage looks fine



 Prometheus metric:

 container \_cpu \_cfs \_throttled \_periods \_total

 container \_cpu \_cfs \_throttled \_seconds \_total



 THE CONTROVERSIAL FIX:

 Many FAANG companies REMOVE CPU limits entirely

 and only set CPU requests

 Rationale: CPU limits cause throttling which causes latency spikes

 CPU requests still guarantee fair sharing via cpu.weight

 Google's Borg paper advocates this approach

 But it requires careful capacity planning



 How to do it:

resources:

 requests:

   cpu: "500m"      # Guaranteed share

 limits:

   memory: "512Mi"  # Keep memory limits (OOM is worse than throttle)

   # No CPU limit!  # Process can burst above 500m when capacity available

```



\#### Memory cgroup — How OOM Kill Actually Works at the Kernel Level:



```bash

 When a container hits its memory.max:



 1. Kernel tries to reclaim memory (page cache, inactive pages)

 2. If reclaim fails → kernel invokes cgroup OOM killer

 3. OOM killer picks a process within the cgroup (usually the main one)

 4. Process receives SIGKILL

 5. Container runtime detects the kill

 6. Kubernetes records: "Reason: OOMKilled, Exit Code: 137"

    (137 = 128 + 9, where 9 = SIGKILL)



 Exit code 137 ALWAYS means SIGKILL

 In context of a container, usually means OOM killed



 Other important exit codes:

 0   = clean exit

 1   = application error

 126 = command not executable (permission issue)

 127 = command not found (binary missing)

 130 = SIGINT (128 + 2) — killed by Ctrl+C

 137 = SIGKILL (128 + 9) — usually OOM

 143 = SIGTERM (128 + 15) — graceful shutdown signal



 Check if OOM killed:

kubectl describe pod <pod> | grep -A3 "Last State"

 Last State: Terminated

   Reason: OOMKilled

   Exit Code: 137



 Or check kernel messages on the node:

dmesg -T | grep -i "oom|killed"

```



\#### PID cgroup — Fork Bomb Protection:



```bash

 Docker

docker run --pids-limit=200 myapp



 Kubernetes (requires support in container runtime)

 Not directly in pod spec, but configurable at:

 - Kubelet flag: --pod-max-pids=200

 - Container runtime config



 Check current PID limit for a cgroup:

cat /sys/fs/cgroup/.../pids.max

 200



 Check current PID usage:

cat /sys/fs/cgroup/.../pids.current

 15

```



\---



\### Part 3: How Docker Uses Both Together



When you run `docker run -it --memory=512m --cpus=1 ubuntu bash`, Docker:



1\. Creates new \*\*namespaces\*\* (PID, NET, MNT, UTS, IPC) → isolation

2\. Creates a new \*\*cgroup\*\* with limits (memory=512m, cpu=1) → resource control

3\. Sets up an \*\*overlay filesystem\*\* → container image layers

4\. Creates a \*\*veth pair\*\* and attaches to docker0 bridge → networking

5\. Runs your process inside all of the above



That's a container. Not a VM. Not magic. Just namespaces + cgroups + filesystem layering.



```bash

 You can create a "container" yourself without Docker:

 1. Create a cgroup

mkdir /sys/fs/cgroup/mycontainer

echo 536870912 > /sys/fs/cgroup/mycontainer/memory.max

echo $$ > /sys/fs/cgroup/mycontainer/cgroup.procs



 2. Enter new namespaces

unshare --mount --uts --ipc --net --pid --fork /bin/bash



 3. Change root filesystem

pivot \_root /path/to/new/root /path/to/put/old/root



 Congratulations, you just built a container from scratch

 Docker just automates this with a nice CLI and image management

```



\---



\### Production Scenarios:



\#### Scenario 1: Debugging Container Resource Issues Without kubectl top



```bash

 kubectl top pod shows high-level metrics

 But sometimes you need RAW cgroup data



 SSH to the node

 Find the container's cgroup path:

crictl ps | grep myapp

crictl inspect <container-id> | grep cgroupsPath



 Read raw memory stats:

cat /sys/fs/cgroup/<path>/memory.current    # Current usage

cat /sys/fs/cgroup/<path>/memory.max        # Limit

cat /sys/fs/cgroup/<path>/memory.stat       # Detailed breakdown

 Shows: anon, file, kernel, slab, sock, shmem, etc.

 "anon" = application heap memory

 "file" = page cache (not necessarily a problem)



 Read raw CPU stats:

cat /sys/fs/cgroup/<path>/cpu.stat

 nr \_throttled → how many times CPU was throttled

 throttled \_usec → total throttle time

```



\#### Scenario 2: Container Using Way More Memory Than Expected



```bash

 kubectl top pod myapp

 NAME    CPU   MEMORY

 myapp   50m   1.8Gi    ← Expected  \~500Mi!



 Check detailed memory breakdown:

cat /sys/fs/cgroup/<path>/memory.stat

 anon 524288000          ← 500MB — app's actual heap (expected)

 file 1400000000         ← 1.3GB — PAGE CACHE!

 

 Page cache = files read from disk that kernel keeps in memory

 This is NORMAL and HARMLESS

 Kernel will release page cache under memory pressure

 

 But kubectl top reports anon + file = 1.8GB

 This misleads people into thinking there's a memory leak

 

 THE REAL METRIC TO WATCH:

 memory.current vs memory.max (are you approaching the limit?)

 NOT the absolute number

```



\#### Scenario 3: nsenter for Network Debugging in Distroless Containers



```bash

 Your production container uses Google's distroless image

 There's no shell, no curl, no ping, no ss, no ip

 Pod can't reach a downstream service

 kubectl exec doesn't work (no shell!)



 SSH to the node

 Find container PID:

crictl ps | grep myapp

crictl inspect <container-id> | grep pid

 "pid": 28456



 Use nsenter to enter ONLY the network namespace:

nsenter -t 28456 -n -- curl http://downstream-service:8080

nsenter -t 28456 -n -- ss -tlnp

nsenter -t 28456 -n -- ip addr show

nsenter -t 28456 -n -- ping 10.0.0.5

nsenter -t 28456 -n -- nslookup downstream-service



 You're running HOST tools in the CONTAINER'S network view

 Best debugging technique for minimal/distroless containers

```



\#### Scenario 4: Process Inside Container Can't See All CPU Cores



```bash

 Container has cpu limit of 2 cores

 Java app sees 96 cores (the host has 96)

 Java creates 96 threads → massive context switching → terrible performance



 Why?

 CPU LIMITS don't restrict visibility of /proc/cpuinfo

 They restrict TIME, not CORES

 Java (and many languages) read /proc/cpuinfo to determine thread count

 They see ALL host cores



 Fix for Java:

 JVM 10+ respects cgroup limits automatically:

java -XX:+UseContainerSupport -jar app.jar  # Default in modern JVMs



 For older JVMs:

java -XX:ActiveProcessorCount=2 -jar app.jar



 Fix for other languages:

 Read from cgroup instead of /proc/cpuinfo:

 Python: os.sched \_getaffinity(0)

 Go: runtime.GOMAXPROCS() — automatically reads cgroup since Go 1.19

 Node.js: doesn't auto-detect, use UV \_THREADPOOL \_SIZE env var

```



\---



\# 📋 LESSON 6 QUICK REFERENCE — cgroups \& Namespaces



```

NAMESPACES (What processes can SEE):

 PID    → Isolated process tree (PID 1 inside container)

 NET    → Own IP, ports, routing, iptables

 MNT    → Own filesystem mounts (overlay2)

 UTS    → Own hostname

 IPC    → Own shared memory/semaphores

 USER   → UID mapping (rootless containers)

 CGROUP → Own cgroup view

 TIME   → Own clocks (Linux 5.6+)



 View:       ls -la /proc/<pid>/ns/

 Enter:      nsenter -t <pid> -m -u -i -n -p -- /bin/sh

 Net only:   nsenter -t <pid> -n -- curl http://service:8080

 Create:     unshare --mount --uts --ipc --net --pid --fork /bin/bash



CGROUPS (What processes can USE):

 memory  → RAM limits (exceed = OOM KILL)

 cpu     → CPU time (exceed = THROTTLE, not kill)

 cpuset  → Pin to specific cores

 blkio   → Disk I/O bandwidth

 pids    → Max process count (fork bomb protection)



 cgroup v1: /sys/fs/cgroup/<controller>/<path>/

 cgroup v2: /sys/fs/cgroup/<path>/



 Key files:

   memory.max / memory.current    → limit and usage

   memory.stat                     → detailed breakdown (anon, file, etc.)

   cpu.stat                        → nr \_throttled, throttled \_usec

   pids.max / pids.current        → process limit and count



DOCKER → CGROUPS MAPPING:

 --memory=512m    → memory.max = 536870912

 --cpus=1.5       → cpu.max = 150000 100000

 --pids-limit=100 → pids.max = 100



KUBERNETES → CGROUPS MAPPING:

 limits.memory    → memory.max (exceed = OOM KILL, exit code 137)

 limits.cpu       → cpu.max (exceed = THROTTLE)

 requests.memory  → scheduling only

 requests.cpu     → cpu.weight (proportional sharing)



EXIT CODES:

 0   = clean exit

 1   = app error

 126 = not executable

 127 = not found

 130 = SIGINT (Ctrl+C)

 137 = SIGKILL (OOM)

 143 = SIGTERM (graceful)



CPU THROTTLING:

 Check: cat cpu.stat → nr \_throttled

 Monitor: container \_cpu \_cfs \_throttled \_periods \_total

 FAANG approach: remove CPU limits, keep only requests

 Keep memory limits ALWAYS



PAGE CACHE vs REAL MEMORY:

 memory.stat → anon = real app memory

 memory.stat → file = page cache (kernel managed, not a leak)

 kubectl top shows both combined → misleading



CONTAINER = PROCESS + NAMESPACES + CGROUPS + OVERLAY FS

Not a VM. Same kernel. Just isolated and resource-limited.



NSENTER (top 1% debugging):

 crictl inspect <id> | grep pid

 nsenter -t <pid> -n -- curl/ss/ip/ping

 Works on distroless/minimal containers with no shell

```



\---



\# 📝 Retention Questions — Lesson 6 ONLY



\*\*Q1:\*\* A developer says \*"my container only has 2 CPU cores allocated but it's creating 64 threads and performance is terrible."\* Explain why the container sees 64 cores despite having a 2-core limit, and how you fix it for a Java application.



\*\*Q2:\*\* A pod shows `Exit Code: 137` in `kubectl describe pod`. What signal killed it, what's the most likely cause, and where do you look to confirm?



\*\*Q3:\*\* You need to debug network connectivity from a production pod that uses a distroless image (no shell, no curl, no tools). How do you run network diagnostic commands from that pod's network perspective?



\*\*Q4:\*\* An SRE notices that their latency P99 spikes every few seconds even though CPU usage averages only 30%. What cgroup-related issue could cause this, how do you confirm it, and what's the controversial FAANG fix?



\*\*Go.\*\* 🎯



====================================================================================

#### \## Lesson 7: systemd — The Service Manager That Controls Everything



systemd is PID 1 on virtually every modern Linux distribution. It boots your system, manages services, handles logging, controls timers, manages mounts, and more. If you're managing servers, deploying applications, writing Ansible playbooks, or debugging why a service won't start — you're dealing with systemd.



\---



\### What Is systemd?



systemd is an \*\*init system and service manager\*\*. It replaced older init systems (SysVinit, Upstart) because:



\- \*\*Parallel startup\*\* — starts services simultaneously, not sequentially

\- \*\*Dependency management\*\* — "start nginx AFTER network is up"

\- \*\*Process supervision\*\* — auto-restarts crashed services

\- \*\*Unified logging\*\* — journald captures all service output

\- \*\*Socket/timer activation\*\* — start services on demand

\- \*\*cgroup integration\*\* — each service gets its own cgroup automatically



When your Linux system boots:

```

BIOS/UEFI → Bootloader (GRUB) → Kernel → systemd (PID 1)

                                            ↓

                                    Reads unit files

                                            ↓

                                    Starts services in parallel

                                    based on dependency graph

                                            ↓

                                    System is ready

```



\---



\### Unit Files — The Heart of systemd



Everything systemd manages is a \*\*unit\*\*. Unit files define what to run, when, and how.



\#### Unit Types:



| Type | Extension | Purpose | Example |

|------|-----------|---------|---------|

| \*\*Service\*\* | `.service` | Daemons and processes | nginx.service, docker.service |

| \*\*Socket\*\* | `.socket` | Network/IPC socket activation | sshd.socket |

| \*\*Timer\*\* | `.timer` | Scheduled tasks (replaces cron) | certbot.timer |

| \*\*Mount\*\* | `.mount` | Filesystem mounts | var-lib-docker.mount |

| \*\*Target\*\* | `.target` | Groups of units (like runlevels) | multi-user.target |

| \*\*Path\*\* | `.path` | Watch filesystem for changes | myapp-config.path |

| \*\*Slice\*\* | `.slice` | cgroup hierarchy for resource control | user.slice |

| \*\*Scope\*\* | `.scope` | Externally created processes | session-1.scope |

| \*\*Device\*\* | `.device` | Kernel device management | dev-sda1.device |

| \*\*Swap\*\* | `.swap` | Swap space management | dev-sdb1.swap |



\#### Where Unit Files Live:



```bash

Priority order (highest to lowest):



/etc/systemd/system/          # Admin overrides — HIGHEST PRIORITY

                             # This is where YOU put custom units

                             # and where 'systemctl edit' writes



/run/systemd/system/          # Runtime units (transient, lost on reboot)



/usr/lib/systemd/system/      # Package-installed units — LOWEST PRIORITY

/lib/systemd/system/          # (same, symlinked on some distros)

                             # NEVER edit these directly

                              # Your changes get overwritten on package update

```



\*\*Golden rule:\*\* NEVER edit files in `/usr/lib/systemd/system/`. Always override using `/etc/systemd/system/` or `systemctl edit`.



\---



\### Anatomy of a Service Unit File



```ini

/etc/systemd/system/myapp.service



 \[Unit]

Description=My Application Server

Documentation=https://docs.myapp.com

After=network-online.target postgresql.service

Wants=network-online.target

Requires=postgresql.service



 \[Service]

Type=simple

User=appuser

Group=appgroup

WorkingDirectory=/opt/myapp

EnvironmentFile=/etc/myapp/env

ExecStartPre=/opt/myapp/pre-start.sh

ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yml

ExecStartPost=/opt/myapp/post-start.sh

ExecReload=/bin/kill -HUP $MAINPID

ExecStop=/opt/myapp/graceful-shutdown.sh

Restart=on-failure

RestartSec=5

StartLimitIntervalSec=300

StartLimitBurst=5

TimeoutStartSec=30

TimeoutStopSec=60

KillMode=mixed

KillSignal=SIGTERM

FinalKillSignal=SIGKILL



Resource limits

LimitNOFILE=65535

LimitNPROC=4096

MemoryMax=2G

CPUQuota=150%



Security hardening

ProtectSystem=strict

ProtectHome=yes

NoNewPrivileges=yes

PrivateTmp=yes

ReadWritePaths=/opt/myapp/data /var/log/myapp



 \[Install]

WantedBy=multi-user.target

```



Let me break down every section and directive:



\---



##### \### \[Unit] Section — Metadata and Dependencies



```ini

 \[Unit]

Description=My Application Server

Human-readable description shown in systemctl status



Documentation=https://docs.myapp.com

Link to docs — shows in systemctl status



After=network-online.target postgresql.service

Start this unit AFTER these are started

This is ORDERING only — doesn't mean they must be running



Wants=network-online.target

"Soft" dependency — start network-online.target if it exists

If it fails, myapp STILL starts



Requires=postgresql.service

"Hard" dependency — if postgresql.service fails or is stopped,

myapp.service is ALSO stopped

Use carefully — creates tight coupling



BindsTo=docker.service

Even stronger than Requires

If docker.service is stopped for ANY reason (even manual stop),

this unit is also stopped

Rarely used, but critical for container-dependent services



Conflicts=apache2.service

Cannot run simultaneously with apache2

Starting this unit stops apache2 and vice versa

Real use: nginx and apache both want port 80



Condition \*= and Assert \*=

Conditional startup — unit only starts if condition is met:

ConditionPathExists=/opt/myapp/config.yml

Don't start if config doesn't exist — prevents crash loop



AssertPathExists=/opt/myapp/bin/server

If path doesn't exist, FAIL HARD (vs Condition which silently skips)

```



\### Dependency Directives Cheat Sheet:



| Directive | What It Does | On Failure |

|-----------|-------------|------------|

| \*\*After\*\* | Ordering only — start after X | No effect if X fails |

| \*\*Before\*\* | Ordering only — start before X | No effect if X fails |

| \*\*Wants\*\* | Soft dependency — try to start X too | myapp starts anyway |

| \*\*Requires\*\* | Hard dependency — X must succeed | myapp stops if X fails |

| \*\*BindsTo\*\* | Strongest — lifecycle bound to X | myapp stops if X stops for ANY reason |

| \*\*Conflicts\*\* | Mutual exclusion | Starting one stops the other |



\*\*Common mistake:\*\* Using `Requires` when you mean `After`.



```ini

WRONG — This has a race condition:

Requires=postgresql.service

postgresql starts simultaneously with myapp

myapp might start BEFORE postgres is ready



RIGHT — Ordering + dependency:

Requires=postgresql.service

After=postgresql.service

postgres starts first, THEN myapp starts after postgres is up

```



\---



##### \### \[Service] Section — How to Run the Process



\#### Service Types:



```ini

Type=simple

DEFAULT. systemd considers the service started immediately 

after ExecStart forks. The process specified in ExecStart 

IS the main process. Most modern applications use this.

Example: node server.js, python app.py, go binaries



Type=forking

For traditional daemons that fork a child and the parent exits.

systemd waits for the parent to exit, then tracks the child.

Often requires PIDFile= directive.

Example: traditional Apache, older MySQL

You'll see this in legacy services.



Type=oneshot

For scripts that run once and exit.

systemd waits for ExecStart to finish before marking as "active".

Example: database migration script, system initialization

Often used with RemainAfterExit=yes



Type=notify

Service tells systemd when it's ready via sd \_notify().

Most reliable for complex services with initialization phases.

Example: PostgreSQL, systemd-aware services

Requires the application to call sd \_notify("READY=1")



Type=exec

Like simple, but systemd waits for the binary to be 

successfully executed (exec() call succeeds) before 

considering it started. Catches "binary not found" faster.



Type=idle

Like simple, but delays start until all jobs are dispatched.

Used for console output ordering. Rarely used in practice.

```



\*\*Which type to use in practice:\*\*



```

Modern app (Node, Python, Go, Java) → Type=simple

Legacy daemon that forks           → Type=forking + PIDFile=

One-time script                    → Type=oneshot + RemainAfterExit=yes

App with slow initialization       → Type=notify (if app supports sd \_notify)

```



\#### Exec Directives:



```ini

ExecStartPre=/opt/myapp/pre-start.sh

Runs BEFORE main process starts

Use for: migrations, config validation, directory creation

If this fails, service doesn't start



ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yml

The main process. ONLY ONE ExecStart allowed (for Type=simple).

For Type=oneshot, multiple ExecStart lines are allowed.



ExecStartPost=/opt/myapp/post-start.sh

Runs AFTER main process starts

Use for: registration with service discovery, notifications



ExecReload=/bin/kill -HUP $MAINPID

Runs when you do 'systemctl reload myapp'

Typically sends SIGHUP for config reload

$MAINPID is a special variable — systemd tracks the main PID



ExecStop=/opt/myapp/graceful-shutdown.sh

Custom stop command. If not specified, systemd sends KillSignal.

Use for: custom drain logic, deregistration from load balancer



ExecStopPost=/opt/myapp/cleanup.sh

Runs AFTER service has stopped

Use for: cleanup temp files, send notification, remove PID file

```



##### \#### Restart Policies:



```ini

Restart=on-failure

Restart only if the process exits with non-zero exit code,

killed by signal (except SIGHUP, SIGINT, SIGTERM, SIGPIPE),

timeout, or watchdog timeout



Options:

Restart=no              # Never restart (default)

Restart=always          # Restart regardless of exit status

Restart=on-success      # Restart only if clean exit (code 0)

Restart=on-failure      # Restart on non-zero exit, signal, timeout

Restart=on-abnormal     # Restart on signal, timeout, watchdog only

Restart=on-abort        # Restart on signal only

Restart=on-watchdog     # Restart on watchdog timeout only



RestartSec=5

Wait 5 seconds between restart attempts

Prevents rapid crash loops from overwhelming the system



Rate limiting restarts:

StartLimitIntervalSec=300

StartLimitBurst=5

Allow maximum 5 restarts within 300 seconds (5 minutes)

After that, unit enters "failed" state and stops restarting

This prevents infinite restart loops from burning resources

To reset: systemctl reset-failed myapp

```



\*\*Production recommendation:\*\*

```ini

  For a critical production service:

Restart=on-failure

RestartSec=5

StartLimitIntervalSec=600

StartLimitBurst=10

  Allows 10 restarts in 10 minutes before giving up

  Gives enough room for transient failures without infinite loops

```



\#### Kill Behavior:



```ini

KillMode=mixed

  How systemd kills the service when stopping:

  control-group (default) — sends signal to ALL processes in cgroup

  mixed — sends KillSignal to main process, SIGKILL to remaining after timeout

  process — sends signal only to main process

  none — doesn't kill anything (dangerous)



KillSignal=SIGTERM

  Signal sent first (default SIGTERM)



FinalKillSignal=SIGKILL

  Signal sent after TimeoutStopSec expires



TimeoutStopSec=60

  How long to wait for graceful shutdown before SIGKILL

  Default is 90 seconds — adjust based on your app's drain time

  Sound familiar? Same concept as K8s terminationGracePeriodSeconds



SendSIGKILL=yes

  Whether to send SIGKILL after timeout. Default yes.

  Set to no if you NEVER want forceful kill (risky)

```



\#### Environment Variables:



```ini

  Inline environment variables

Environment=NODE \_ENV=production

Environment=PORT=8080 DB \_HOST=localhost



  From file (preferred — keeps secrets out of unit file)

EnvironmentFile=/etc/myapp/env

  The file format:

  NODE \_ENV=production

  DB \_HOST=postgres.internal

  DB \_PASSWORD=supersecret



  Multiple env files (loaded in order, later overrides earlier)

EnvironmentFile=/etc/myapp/defaults.env

EnvironmentFile=-/etc/myapp/overrides.env

  The - prefix means "don't fail if file doesn't exist"

```



\#### Resource Limits:



```ini

  File descriptor limit

LimitNOFILE=65535



  Process limit

LimitNPROC=4096



  Core dump size (0 = disabled)

LimitCORE=0



  Memory limit (uses cgroup)

MemoryMax=2G

MemoryHigh=1.5G    # Soft limit — kernel will try to reclaim before hitting max



  CPU limit (uses cgroup)

CPUQuota=150%       # 1.5 CPU cores (150% of one core)

CPUWeight=100       # Relative weight (default 100, range 1-10000)



  I/O limits

IOWeight=100

IOReadBandwidthMax=/dev/sda 100M

IOWriteBandwidthMax=/dev/sda 50M

```



\#### Security Hardening:



```ini

  Run as unprivileged user

User=appuser

Group=appgroup



  Filesystem protection

ProtectSystem=strict     # Mount / as read-only (except /dev, /proc, /sys)

ProtectHome=yes          # Make /home, /root, /run/user inaccessible

PrivateTmp=yes           # Give service its own /tmp (not shared)

ReadWritePaths=/opt/myapp/data /var/log/myapp

  Only these paths are writable



  Capability restrictions

NoNewPrivileges=yes      # Process can't gain new privileges (setuid won't work)

CapabilityBoundingSet=CAP \_NET \_BIND \_SERVICE

AmbientCapabilities=CAP \_NET \_BIND \_SERVICE

  Only allow binding to ports below 1024



  Network restrictions

RestrictAddressFamilies=AF \_INET AF \_INET6 AF \_UNIX

  Only allow IPv4, IPv6, and Unix sockets



  System call filtering (like seccomp)

SystemCallFilter=@system-service

  Only allow syscalls typical for a service process



  Private network namespace (complete network isolation)

PrivateNetwork=yes

  Use for services that don't need network (batch processors, etc.)

```



\---



\### \[Install] Section — When to Start



```ini

 \[Install]

WantedBy=multi-user.target

  This service should start when multi-user.target is reached

  (which is the normal boot target for servers — like runlevel 3)



  What 'systemctl enable' actually does:

  Creates a symlink:

  /etc/systemd/system/multi-user.target.wants/myapp.service

    → /etc/systemd/system/myapp.service

  Remember our symlinks lesson? systemd enable/disable IS symlinks.



  Other targets:

WantedBy=graphical.target    # For desktop systems

WantedBy=timers.target       # For timer units



  Aliases:

Alias=my-application.service  # Alternative name for the service

```



\#### Targets — The Modern Runlevels:



```bash

  Targets group units together. Think of them as "system states."

  Traditional runlevels mapped to targets:



  Runlevel 0 → poweroff.target     (shutdown)

  Runlevel 1 → rescue.target       (single-user, root shell)

  Runlevel 3 → multi-user.target   (multi-user, no GUI — servers)

  Runlevel 5 → graphical.target    (multi-user + GUI — desktops)

  Runlevel 6 → reboot.target       (reboot)



  Check current target

systemctl get-default

  multi-user.target



  Set default target

systemctl set-default multi-user.target



  Switch target immediately

systemctl isolate rescue.target    # Drop to rescue mode

```



\---



\### Essential systemctl Commands



```bash

  SERVICE LIFECYCLE:

systemctl start myapp           # Start now

systemctl stop myapp            # Stop now

systemctl restart myapp         # Stop then start

systemctl reload myapp          # Reload config (runs ExecReload)

systemctl reload-or-restart myapp # Reload if supported, else restart

systemctl kill myapp            # Send signal to service processes

systemctl kill -s SIGUSR1 myapp # Send specific signal



  ENABLE/DISABLE (persist across reboots):

systemctl enable myapp          # Start on boot (creates symlink)

systemctl disable myapp         # Don't start on boot (removes symlink)

systemctl enable --now myapp    # Enable AND start immediately

systemctl is-enabled myapp      # Check if enabled



  STATUS                      \& INSPECTION:

systemctl status myapp          # Status, PID, recent logs, memory

systemctl is-active myapp       # Returns "active" or "inactive"

systemctl is-failed myapp       # Returns "failed" or not

systemctl show myapp            # ALL properties (detailed)

systemctl show myapp -p MainPID # Specific property

systemctl cat myapp             # Show the unit file contents

systemctl list-dependencies myapp # Show dependency tree



  UNIT FILE MANAGEMENT:

systemctl daemon-reload         # REQUIRED after editing unit files

                                 # systemd re-reads all unit files

                                 # Forgetting this is the #1 mistake

systemctl edit myapp            # Create override file (drop-in)

systemctl edit --full myapp     # Edit full unit file copy

systemctl revert myapp          # Remove all overrides, revert to package default



  LISTING:

systemctl list-units            # All loaded units

systemctl list-units --failed   # Failed units only

systemctl list-unit-files       # All installed unit files + state

systemctl list-timers           # All timers and next run time



  SYSTEM:

systemctl reboot                # Reboot

systemctl poweroff              # Shutdown

systemctl suspend               # Suspend to RAM

systemctl rescue                # Drop to rescue mode



  RESET:

systemctl reset-failed          # Clear all failed states

systemctl reset-failed myapp    # Clear specific service's failed state

```



\---



\### Overriding Unit Files — The Drop-In Method



Never edit package-installed unit files directly. Use drop-in overrides:



```bash

  Method 1: systemctl edit (recommended)

systemctl edit myapp

  Opens editor for /etc/systemd/system/myapp.service.d/override.conf

  Add ONLY the directives you want to change



  Example override — increase memory limit and file descriptors:

 \[Service]

LimitNOFILE=131072

MemoryMax=4G



  Method 2: Manual drop-in file

mkdir -p /etc/systemd/system/myapp.service.d/

cat > /etc/systemd/system/myapp.service.d/override.conf << 'EOF'

 \[Service]

LimitNOFILE=131072

MemoryMax=4G

EOF

systemctl daemon-reload



  Method 3: Full override (copies entire file)

systemctl edit --full myapp

  Creates a full copy in /etc/systemd/system/ that takes priority

  Over the package version in /usr/lib/systemd/system/



  View effective configuration (merged original + overrides):

systemctl cat myapp



  Revert all overrides:

systemctl revert myapp

```



\*\*⚠️ CRITICAL GOTCHA with drop-in overrides:\*\*



```ini

  Some directives are ADDITIVE (like Environment=)

  Some directives REPLACE (like ExecStart=)



  If you want to override ExecStart, you must FIRST clear it:

 \[Service]

ExecStart=                               # Clear the original

ExecStart=/opt/myapp/bin/server --debug   # Set the new one



  Without the empty ExecStart= line, you'll get an error:

  "Service has more than one ExecStart= setting"

```



\---



\### journald — systemd's Logging System



Every service managed by systemd has its stdout/stderr captured by \*\*journald\*\*. This is the centralized log system for all systemd services.



```bash

  VIEW LOGS:

journalctl -u myapp              # All logs for myapp service

journalctl -u myapp -f           # Follow (live tail) — like tail -f

journalctl -u myapp --since "1 hour ago"

journalctl -u myapp --since "2024-01-15 03:00:00"

journalctl -u myapp --since today

journalctl -u myapp -n 100      # Last 100 lines

journalctl -u myapp -p err      # Only errors and above

journalctl -u myapp -o json     # JSON output (for parsing)

journalctl -u myapp -o json-pretty # Pretty JSON

journalctl -u myapp --no-pager  # Don't use less, dump to stdout



  PRIORITY LEVELS:

  0 = emerg, 1 = alert, 2 = crit, 3 = err, 

  4 = warning, 5 = notice, 6 = info, 7 = debug

journalctl -p 3                  # Show err and above for ALL services

journalctl -p warning..err       # Show only warning through err



  CROSS-SERVICE:

journalctl -u nginx -u myapp    # Logs from multiple services

journalctl \_PID=1234            # Logs from specific PID

journalctl \_UID=1000            # Logs from specific user



  KERNEL MESSAGES:

journalctl -k                    # Kernel ring buffer (like dmesg but persistent)

journalctl -k --since "1 hour ago"



  BOOT LOGS:

journalctl -b                    # Current boot

journalctl -b -1                 # Previous boot

journalctl --list-boots          # List all recorded boots



  DISK USAGE:

journalctl --disk-usage

journalctl --vacuum-size=500M    # Reduce to 500MB

journalctl --vacuum-time=7d      # Keep only last 7 days



  EXPORT:

journalctl -u myapp --since today > /tmp/myapp-logs.txt

journalctl -u myapp -o json --since today | jq '.MESSAGE' > /tmp/messages.txt

```



\#### journald Configuration:



```bash

  /etc/systemd/journald.conf

 \[Journal]

Storage=persistent          # Write to /var/log/journal/ (survives reboot)

                            # Default on some distros is "auto" or "volatile"

SystemMaxUse=1G             # Max total journal size

SystemMaxFileSize=100M      # Max single file size

MaxRetentionSec=30day       # Max retention

MaxFileSec=1day             # Rotate daily

Compress=yes                # Compress journal files

ForwardToSyslog=yes         # Also send to traditional syslog

                            # Useful for shipping to ELK/Splunk

RateLimitIntervalSec=30s    # Rate limiting window

RateLimitBurst=10000        # Max messages per window per service



  Apply changes:

systemctl restart systemd-journald

```



\*\*⚠️ Rate Limiting Gotcha:\*\*



```bash

  journald rate-limits by default: 10000 messages per 30 seconds per service

  If your service logs faster than this, messages are SILENTLY DROPPED



  In the journal you'll see:

  "Suppressed 4523 messages from myapp.service"



  For high-volume services, increase or disable:

  In the service unit file:

 \[Service]

LogRateLimitIntervalSec=0    # Disable rate limiting for this service



  Or system-wide in journald.conf:

RateLimitIntervalSec=0       # Disable (not recommended for all services)

```



\---



\### systemd Timers — The Modern Cron



Timers are the systemd replacement for cron. They offer: logging via journald, dependency management, randomized delays, and persistent tracking (runs missed executions).



```bash

  A timer has TWO unit files:

  1. The timer unit (WHEN to run)

  2. The service unit (WHAT to run)



  /etc/systemd/system/backup.timer

 \[Unit]

Description=Run backup every night



 \[Timer]

OnCalendar= \*- \*- \* 02:00:00     # Daily at 2 AM

  Other examples:

  OnCalendar=hourly             # Every hour

  OnCalendar=weekly             # Weekly

  OnCalendar=Mon  \*- \*- \* 09:00:00 # Mondays at 9 AM

  OnCalendar= \*- \*- \*  \*: \*:00       # Every minute



RandomizedDelaySec=900          # Add random 0-15min delay

                                 # Prevents thundering herd

                                 # (all servers running backup at exactly 2:00)



Persistent=true                  # If system was off at 2 AM, run when it boots

                                 # Cron would just skip it



AccuracySec=1min                 # How precise the timing needs to be



 \[Install]

WantedBy=timers.target



  /etc/systemd/system/backup.service

 \[Unit]

Description=Backup Script



 \[Service]

Type=oneshot

User=backup

ExecStart=/opt/scripts/backup.sh

  No  \[Install] section needed — the timer activates it

```



```bash

  Enable and start the timer:

systemctl enable --now backup.timer



  View all timers and when they'll next fire:

systemctl list-timers

  NEXT                         LEFT          LAST                         PASSED

  Tue 2024-01-16 02:07:23 UTC  8h left       Mon 2024-01-15 02:12:45 UTC  15h ago

  backup.timer                               backup.service



  Check timer status:

systemctl status backup.timer



  Check if the last run succeeded:

systemctl status backup.service

journalctl -u backup.service --since today



  Run the service manually (bypass timer):

systemctl start backup.service



  Monotonic timers (relative to boot/activation):

OnBootSec=5min           # 5 minutes after boot

OnActiveSec=30s          # 30 seconds after timer is activated

OnUnitActiveSec=1h       # 1 hour after the SERVICE last started

OnUnitInactiveSec=30min  # 30 minutes after the SERVICE last finished

```



\*\*Timer vs Cron — Why Timers Win:\*\*



| Feature | Cron | systemd Timer |

|---------|------|---------------|

| Logging | Must configure manually | Automatic via journald |

| Missed runs | Skipped if system was off | `Persistent=true` catches up |

| Dependencies | None | Full systemd dependency graph |

| Resource limits | None | cgroup limits via service unit |

| Random delay | None | `RandomizedDelaySec` |

| Output handling | Tries to email (usually fails) | journald captures everything |

| Status | No easy way to check | `systemctl status`, `list-timers` |

| Security | Runs with user's full privileges | Full systemd sandboxing |



\---



##### \### Socket Activation — Start on Demand



Socket activation starts a service only when a connection arrives. Saves resources and enables zero-downtime deployments.



```ini

  /etc/systemd/system/myapp.socket

 \[Unit]

Description=My App Socket



 \[Socket]

ListenStream=8080

  systemd opens port 8080 and holds it

  When a connection arrives, systemd starts myapp.service

  and hands over the socket



Accept=no

  no = one service instance handles all connections (typical)

  yes = new service instance per connection (inetd-style)



 \[Install]

WantedBy=sockets.target



  /etc/systemd/system/myapp.service

 \[Unit]

Description=My Application

Requires=myapp.socket



 \[Service]

Type=simple

ExecStart=/opt/myapp/bin/server

  The application must be written to accept the socket from systemd

  via file descriptor inheritance (sd \_listen \_fds)

```



\*\*Real-world use cases:\*\*

\- \*\*SSH:\*\* sshd.socket starts sshd only when someone connects

\- \*\*Docker:\*\* docker.socket starts dockerd on first `docker` command

\- \*\*Zero-downtime deployment:\*\* Socket holds connections while service restarts



```bash

  Check: is Docker using socket activation?

systemctl status docker.socket

  Active: active (listening)



  This is why Docker starts on first use even if dockerd isn't running

```



\---



##### \### Production Scenarios:



###### \#### Scenario 1: Service Starts But Immediately Fails — Crash Loop



```bash

systemctl status myapp

  Active: activating (auto-restart)

  Status: start-limit-hit



  Translation: Service crashed too many times within StartLimitIntervalSec

  systemd gave up restarting it



  Debug steps:

  1. Check logs

journalctl -u myapp -n 50 --no-pager

  Look for the error message



  2. Check exit code

systemctl show myapp -p ExecMainStatus

  ExecMainStatus=217

  Google "systemd exit code 217" → usually means User= doesn't exist



  Common systemd-specific exit codes:

  200 = generic — check journal

  201 = failed to exec (binary not found or not executable)

  203 = failed to open stdin/stdout/stderr

  205 = failed to set cgroup

  207 = failed to set user/group (User= or Group= invalid)

  209 = failed to set namespace

  214 = failed to set AppArmor profile

  217 = failed to change user (User= doesn't exist on system)

  226 = failed to set mount namespace (ProtectSystem= issue)



  3. Try starting manually as the service user

sudo -u appuser /opt/myapp/bin/server --config /etc/myapp/config.yml

  This often reveals the actual application error



  4. After fixing, reset the failed state:

systemctl reset-failed myapp

systemctl start myapp

```



###### \#### Scenario 2: "daemon-reload" Forgotten — Silent Configuration Mismatch



```bash

  Engineer edits /etc/systemd/system/myapp.service

  Changes MemoryMax from 2G to 4G

  Runs: systemctl restart myapp

  But FORGOT: systemctl daemon-reload



  What happens?

  systemd is still using the OLD unit file from memory

  The restart uses OLD configuration

  Memory limit is still 2G despite the file saying 4G

  Service gets OOM killed at 2G

  Engineer is confused because "I changed the limit!"



  ALWAYS:

systemctl daemon-reload    # Re-read unit files from disk

systemctl restart myapp    # Now restart with new config



  Or use the one-liner in Ansible:

- name: Restart myapp

  systemd:

    name: myapp

    state: restarted

    daemon \_reload: yes

```



###### \#### Scenario 3: Slow Shutdown Causing Deployment Delays



```bash

  Rolling deployment tool waits for service to stop

  Service takes 90 seconds to stop (default TimeoutStopSec)

  10 servers × 90 seconds = 15 minutes for a rolling deploy



  Investigation:

systemctl stop myapp

  Hangs for 90 seconds...

  Finally killed by SIGKILL



  The app has a graceful shutdown handler but it hangs:

  Maybe waiting for a connection that never closes

  Maybe waiting for a lock that's held by a dead thread



  Fix 1: Reduce timeout

 \[Service]

TimeoutStopSec=30    # 30 seconds is usually enough



  Fix 2: Fix the application's shutdown handler

  Add timeouts to connection draining

  Add cleanup deadlines



  Fix 3: Use KillMode=mixed

KillMode=mixed

  Sends SIGTERM to main process

  After TimeoutStopSec, sends SIGKILL to ALL remaining processes in cgroup

  This catches orphaned child processes that ignore SIGTERM

```



###### \#### Scenario 4: Service Works Via CLI But Fails Under systemd



```bash

  You test: /opt/myapp/bin/server → works perfectly

  You start: systemctl start myapp → fails immediately



  Common causes:



  1. PATH is different

  CLI has: /usr/local/bin:/usr/bin:/bin:...

  systemd has minimal PATH: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin

  Fix: Use full paths in ExecStart



  2. Environment variables missing

  CLI inherits your shell's env (JAVA \_HOME, AWS \_REGION, etc.)

  systemd has almost no env vars

  Fix: Add EnvironmentFile= or Environment= directives



  3. Working directory different

  CLI runs from wherever you are

  systemd starts from / by default

  Fix: Add WorkingDirectory=/opt/myapp



  4. User context different

  CLI runs as root or your user

  Service runs as User= specified in unit file

  Different home dir, different shell, different permissions

  Fix: Check permissions for the service user



  5. TTY/terminal not available

  Some apps expect a terminal (stdin/stdout)

  systemd doesn't provide one

  Fix: Ensure app doesn't require TTY, or add StandardInput=tty



  Debug technique — see EXACT environment systemd provides:

systemd-run --scope -p User=appuser env

  Shows every env var the service user would have

```



###### \#### Scenario 5: Emergency — System Boots to Rescue Mode



```bash

  System won't boot normally

  Drops to rescue shell or emergency shell



  rescue.target vs emergency.target:

  rescue = basic system services loaded, root filesystem mounted rw

  emergency = almost nothing loaded, root filesystem might be read-only



  Common causes:

  1. Bad /etc/fstab entry (trying to mount nonexistent disk)

  2. Corrupt filesystem

  3. Failed service with FailureAction=reboot → boot loop

  4. Full root filesystem → systemd can't create cgroups/temp files



  Fix from emergency shell:

  1. Check fstab

mount -o remount,rw /   # Make root writable

cat /etc/fstab          # Check for bad entries

  Comment out the bad entry



  2. Check failed services

systemctl list-units --failed

systemctl status <failed-unit>

systemctl disable <problematic-unit>



  3. Check disk space

df -h

df -i

  Clean if full



  4. Check kernel messages

dmesg -T | tail -100

journalctl -b -p err



  5. Reboot

systemctl default   # Try to boot to default target

  OR

reboot

```



###### \#### Scenario 6: Service Has Memory Leak — Automated Restart



```bash

  Instead of letting the OOM killer decide, 

  use systemd to proactively manage it:



 \[Service]

MemoryMax=2G            # Hard limit — OOM killed if exceeded

MemoryHigh=1.5G         # Soft limit — kernel starts reclaiming

WatchdogSec=30          # Process must notify systemd every 30s

                        # If it stops notifying (hung), systemd restarts it



  For apps that don't support watchdog/sd \_notify:

  Use a simple restart schedule as a workaround

RuntimeMaxSec=86400     # Restart service every 24 hours regardless

                        # "Restart therapy" — crude but effective for memory leaks

                        # while the team works on the actual fix

```



\---



\### Writing Unit Files for Real Production Services — Complete Examples



\#### Example 1: Node.js Application



```ini

  /etc/systemd/system/nodeapp.service

 \[Unit]

Description=Node.js API Server

Documentation=https://github.com/company/nodeapp

After=network-online.target

Wants=network-online.target

StartLimitIntervalSec=600

StartLimitBurst=10



 \[Service]

Type=simple

User=nodeapp

Group=nodeapp

WorkingDirectory=/opt/nodeapp



  Environment

EnvironmentFile=/etc/nodeapp/env

Environment=NODE \_ENV=production

Environment=NODE \_OPTIONS="--max-old-space-size=1536"



  Process

ExecStart=/usr/bin/node /opt/nodeapp/dist/server.js

ExecReload=/bin/kill -HUP $MAINPID



  Restart policy

Restart=on-failure

RestartSec=5



  Timeouts

TimeoutStartSec=30

TimeoutStopSec=30

KillMode=mixed

KillSignal=SIGTERM



  Resource limits

LimitNOFILE=65535

MemoryMax=2G

CPUQuota=200%



  Security

NoNewPrivileges=yes

ProtectSystem=strict

ProtectHome=yes

PrivateTmp=yes

ReadWritePaths=/opt/nodeapp/uploads /var/log/nodeapp



  Logging

StandardOutput=journal

StandardError=journal

SyslogIdentifier=nodeapp



 \[Install]

WantedBy=multi-user.target

```



\#### Example 2: Java Spring Boot Application



```ini

  /etc/systemd/system/springapp.service

 \[Unit]

Description=Spring Boot Application

After=network-online.target postgresql.service

Wants=network-online.target

Requires=postgresql.service



 \[Service]

Type=simple

User=springapp

Group=springapp

WorkingDirectory=/opt/springapp



EnvironmentFile=/etc/springapp/env

Environment=JAVA \_HOME=/usr/lib/jvm/java-17

Environment=JAVA \_OPTS="-Xms512m -Xmx1536m -XX:+UseG1GC -XX:+UseContainerSupport"



ExecStart=/usr/lib/jvm/java-17/bin/java 

    $JAVA \_OPTS  \\

    -jar /opt/springapp/app.jar  \\

    --spring.profiles.active=production  \\

    --server.port=8080



ExecStop=/bin/kill -TERM $MAINPID

Restart=on-failure

RestartSec=10



TimeoutStartSec=120     # Java apps take longer to start

TimeoutStopSec=60



LimitNOFILE=65535

MemoryMax=2G



SuccessExitStatus=143   # SIGTERM causes exit 143 — that's expected, not failure



NoNewPrivileges=yes

ProtectSystem=strict

ReadWritePaths=/opt/springapp/logs /opt/springapp/data



 \[Install]

WantedBy=multi-user.target

```



\*\*Key detail:\*\* `SuccessExitStatus=143` — Java exits with 143 on SIGTERM (128+15). Without this, systemd treats every stop as a "failure" and triggers Restart=on-failure. This is a very common gotcha with Java services on systemd.







\# 📝 Retention Questions — Lesson 7 ONLY



\*\*Q1:\*\* You edit a systemd unit file to increase `MemoryMax` from 2G to 4G. You run `systemctl restart myapp`. The service gets OOM killed at 2G. What did you forget?



\*\*Q2:\*\* A Java Spring Boot service is configured with `Restart=on-failure`. Every time you run `systemctl stop springapp`, the service immediately restarts. Why is this happening, and how do you fix it?



\*\*Q3:\*\* You need a backup script to run every night at 2 AM with a random delay of up to 15 minutes. You also need it to run if the server was off at 2 AM. Write the complete timer and service unit files.



\*\*Q4:\*\* A service works perfectly when you run the binary manually as root, but fails immediately under systemd. `journalctl -u myapp` shows exit code 217. What's wrong?



\*\*Q5:\*\* Your CI/CD pipeline restarts a service during deployment. The `systemctl restart` command takes 90 seconds to complete, making deployments slow. The application should shut down in under 5 seconds. What's causing the delay and how do you fix it?



\*\*Go.\*\* 🎯





==========================================================================



##### \## Lesson 8: Kernel Tuning (sysctl) — Making Linux Production-Ready



The Linux kernel ships with defaults optimized for \*\*general purpose desktop use\*\* — not for running high-traffic web servers, databases, or container orchestrators. Out-of-the-box Linux will choke under production workloads unless you tune it.



`sysctl` is the interface to the kernel's runtime configuration. Every parameter lives under `/proc/sys/` and controls how the kernel handles networking, memory, filesystems, and security.



\---



\### How sysctl Works



```bash

 View a kernel parameter

sysctl net.ipv4.tcp \_tw \_reuse

 net.ipv4.tcp \_tw \_reuse = 0



 Same thing via /proc

cat /proc/sys/net/ipv4/tcp \_tw \_reuse

 0



 The mapping:

 sysctl dots = filesystem slashes

 net.ipv4.tcp \_tw \_reuse → /proc/sys/net/ipv4/tcp \_tw \_reuse


 Set a parameter (temporary — lost on reboot)

sysctl -w net.ipv4.tcp \_tw \_reuse=1

 OR

echo 1 > /proc/sys/net/ipv4/tcp \_tw \_reuse



 Set parameters permanently

 /etc/sysctl.conf (traditional)

 /etc/sysctl.d/99-custom.conf (modern, drop-in — recommended)

 Files in /etc/sysctl.d/ are loaded in alphabetical order

 Higher numbers = loaded later = override earlier ones



 Apply all sysctl configs

sysctl -p                          # Load /etc/sysctl.conf

sysctl --system                    # Load ALL files from /etc/sysctl.d/



 View ALL current parameters

sysctl -a                          # Thousands of parameters

sysctl -a | wc -l                  # Usually 1000+



 Search for specific parameters

sysctl -a | grep tcp

sysctl -a | grep vm.

```



\*\*File naming convention in production:\*\*



```bash

/etc/sysctl.d/10-network.conf      # Network tuning

/etc/sysctl.d/20-memory.conf       # Memory tuning

/etc/sysctl.d/30-security.conf     # Security hardening

/etc/sysctl.d/40-kubernetes.conf   # K8s-specific tuning

```



\---



###### \### Category 1: Network Stack Tuning



This is where 70% of production sysctl changes happen. The default Linux network stack is designed for a laptop with 10 connections, not a server handling 100,000.



\#### Connection Handling



```bash

 ── BACKLOG AND CONNECTION QUEUE ──



net.core.somaxconn = 65535

 Maximum number of connections waiting in the accept queue

 DEFAULT: 128 ← ABSURDLY LOW for any server

 When this is full, new connections get DROPPED

 Nginx, HAProxy, any server behind a load balancer needs this high

 Symptom of too low: "connection refused" under load despite server being healthy



net.core.netdev \_max \_backlog = 65535

 Maximum packets queued on the INPUT side when interface receives

 packets faster than the kernel can process them

 DEFAULT: 1000

 On 10Gbps+ NICs, packets arrive faster than 1000/burst

 Symptom of too low: packet drops visible in `ifconfig` or `ip -s link`



net.ipv4.tcp \_max \_syn \_backlog = 65535

 Maximum number of half-open connections (SYN \_RECV state)

 DEFAULT: 128 or 1024 depending on distro

 During SYN flood attacks or traffic spikes, this fills up

 Legitimate connections get dropped

 Symptom: "connection timeout" for new users during traffic spikes



net.ipv4.tcp \_max \_tw \_buckets = 2000000

 Maximum number of TIME \_WAIT sockets

 DEFAULT: varies (often 32768)

 We covered TIME \_WAIT exhaustion in Lesson 4

 If you hit this limit, kernel DESTROYS TIME \_WAIT sockets prematurely

 That's actually dangerous — can cause connection confusion

 Set high enough that you never hit it, then fix the real problem

 (connection pooling, keep-alive)

```



\#### TIME\_WAIT and Connection Reuse



```bash

net.ipv4.tcp \_tw \_reuse = 1

 Allow reusing TIME \_WAIT sockets for new OUTBOUND connections

 DEFAULT: 0 (disabled)

 SAFE for client-side connections (your app connecting to DB, APIs)

 The kernel checks TCP timestamps to ensure safety

 MUST HAVE for reverse proxies, microservices, anything making many outbound calls



 NOTE: tcp \_tw \_recycle was REMOVED from Linux 4.12+

 It was dangerous behind NAT (broke connections from same source IP)

 If you see old guides recommending it → IGNORE THEM

```



\#### Keep-Alive



```bash

net.ipv4.tcp \_keepalive \_time = 300

 How many SECONDS before sending the first keepalive probe

 DEFAULT: 7200 (2 hours!) ← Way too long

 If a connection is idle for 2 hours before detecting it's dead,

 you're holding resources for a ghost connection

 Set to 300 (5 minutes) or lower for services behind load balancers



net.ipv4.tcp \_keepalive \_intvl = 30

 Seconds between subsequent keepalive probes

 DEFAULT: 75



net.ipv4.tcp \_keepalive \_probes = 5

 Number of failed probes before declaring connection dead

 DEFAULT: 9

 With these settings: dead connection detected in 300 + (30 × 5) = 450 seconds

 vs default: 7200 + (75 × 9) = 7875 seconds (over 2 hours!)



 Why this matters:

 Load balancers (AWS ALB/NLB) have their own idle timeouts

 ALB default: 60 seconds

 If your keepalive \_time (7200s) > ALB idle timeout (60s):

   ALB silently drops the connection

   Server doesn't know it's dead

   Next request on that connection → RST → error

   This causes intermittent "connection reset" errors

 FIX: Set keepalive \_time LOWER than your load balancer's idle timeout

```



\#### TCP Buffer Sizes



```bash

 ── TCP MEMORY AND BUFFER SIZES ──



net.core.rmem \_max = 16777216

 Maximum receive buffer size (16MB)

 DEFAULT: 212992 (208KB)



net.core.wmem \_max = 16777216

 Maximum send buffer size (16MB)

 DEFAULT: 212992 (208KB)



net.core.rmem \_default = 1048576

net.core.wmem \_default = 1048576

 Default buffer sizes (1MB)



net.ipv4.tcp \_rmem = 4096 1048576 16777216

 TCP receive buffer: min  default  max

 Kernel auto-tunes between min and max

 DEFAULT: 4096 131072 6291456



net.ipv4.tcp \_wmem = 4096 1048576 16777216

 TCP send buffer: min  default  max

 DEFAULT: 4096 16384 4194304



net.ipv4.tcp \_mem = 786432 1048576 1572864

 Total TCP memory in PAGES (not bytes): low  pressure  high

 low = below this, TCP doesn't worry about memory

 pressure = TCP starts being careful, reduces buffers

 high = TCP refuses new allocations

 Each page = 4KB, so: 3GB  4GB  6GB

 Adjust based on total system RAM



 Why this matters:

 For high-throughput applications (file transfers, streaming, databases):

 - Small buffers = TCP can't fill the pipe = low throughput

 - On a 10Gbps link with 50ms RTT, you need  \~62MB window to saturate

 - Default 208KB buffer can only push  \~33Mbps on that link

 This is called the Bandwidth-Delay Product (BDP)

 BDP = bandwidth × RTT = 10Gbps × 0.05s = 62.5MB

```



\#### Ephemeral Ports



```bash

net.ipv4.ip \_local \_port \_range = 1024 65535

 Range of ports used for outbound connections

 DEFAULT: 32768 60999 ( \~28,000 ports)

 Expanded: 1024 65535 ( \~64,000 ports)

 Critical for reverse proxies and microservices making many outbound calls

 We covered this in TIME \_WAIT lesson — doubling ports = doubling headroom

```



###### \#### TCP Performance Features



```bash

net.ipv4.tcp \_window \_scaling = 1

 Enable TCP window scaling (RFC 1323)

 DEFAULT: 1 (enabled)

 Required for windows larger than 64KB

 NEVER disable this



net.ipv4.tcp \_timestamps = 1

 Enable TCP timestamps (RFC 1323)

 DEFAULT: 1

 Required for tcp \_tw \_reuse to work safely

 Also helps with RTT measurement and PAWS protection

 NEVER disable this



net.ipv4.tcp \_sack = 1

 Selective Acknowledgment

 DEFAULT: 1

 Allows receiver to tell sender EXACTLY which packets were lost

 Without SACK: entire window must be retransmitted

 With SACK: only lost packets are retransmitted

 MASSIVE performance improvement on lossy networks

 NEVER disable this



net.ipv4.tcp \_fastopen = 3

 TCP Fast Open — send data in the SYN packet

 DEFAULT: 1 (client only)

 3 = enable for both client and server

 Saves one RTT on connection establishment

 Particularly useful for HTTP connections

 Supported by modern browsers and servers



net.ipv4.tcp \_slow \_start \_after \_idle = 0

 DEFAULT: 1

 When 1: TCP resets congestion window after idle period

 After the idle timeout, TCP acts like a brand new connection

 (starts with slow start, ramps up slowly)

 For persistent connections (HTTP keep-alive, database pools),

 this means every burst after idle gets artificially throttled

 Set to 0: maintain congestion window across idle periods

 Critical for services with bursty traffic patterns



net.ipv4.tcp \_mtu \_probing = 1

 Enable Path MTU discovery

 DEFAULT: 0

 Discovers the maximum packet size along the path

 Prevents fragmentation issues, especially in VPNs/tunnels/cloud networks

 AWS VPC has MTU of 9001 (jumbo frames) within VPC but 1500 across VPN

 Without MTU probing, you get mysterious packet loss on cross-VPC traffic

```



\#### SYN Flood Protection



```bash

net.ipv4.tcp \_syncookies = 1

 Enable SYN cookies

 DEFAULT: 1 (usually enabled)

 When SYN backlog is full, instead of dropping connections,

 kernel encodes state in the SYN-ACK sequence number

 Allows legitimate connections to complete even during SYN flood

 This is your first line of defense against SYN flood DDoS



net.ipv4.tcp \_synack \_retries = 2

 Number of times to retry SYN-ACK

 DEFAULT: 5

 During SYN flood, retrying 5 times wastes resources on fake connections

 Reduce to 2 — legitimate clients retry on their own

 Frees up resources faster



net.ipv4.tcp \_syn \_retries = 3

 Number of times to retry SYN for outbound connections

 DEFAULT: 6 (which can mean  \~127 seconds before giving up!)

 Reduce to 3 for faster failure detection

 Prefer fast failure + application retry over slow kernel retry



net.ipv4.tcp \_abort \_on  \_overflow = 0

 DEFAULT: 0

 When 0: If accept queue is full, kernel silently drops the ACK

 Client will retry (TCP retransmission)

 When 1: Kernel sends RST immediately → client gets "connection refused"

 For most production servers: keep at 0

 Silent drop + retry is better than hard rejection

 Exception: set to 1 if you want FAST failure for debugging

```



\---



###### \### Category 2: Memory Management



```bash

 ── VIRTUAL MEMORY ──



vm.swappiness = 10

 How aggressively kernel swaps memory to disk

 Range: 0-100

 DEFAULT: 60

 0 = only swap when absolutely necessary (avoid for databases)

 10 = light swapping (good for most servers)

 60 = moderate swapping (desktop default — too aggressive for servers)

 100 = swap aggressively

  \\\\#

 For databases (MySQL, PostgreSQL, Redis): use 1 or even 0

 Swapping database memory pages to disk = catastrophic latency

 For Kubernetes nodes: use 10

 For Redis servers: use 0 (Redis is PURE in-memory, swapping kills it)



vm.overcommit \_memory = 0

 Memory overcommit policy

 0 = heuristic (kernel guesses if allocation will succeed) — DEFAULT

 1 = always allow (malloc never fails — DANGEROUS, OOM killer becomes only protection)

 2 = strict accounting (never overcommit beyond swap + ratio)

  \\\\#

 For Redis: USE 1 (Redis documentation explicitly requires this)

   Redis forks for background saves (RDB snapshots, AOF rewrite)

   Fork needs to "copy" entire memory space (copy-on-write)

   With overcommit=0, kernel might deny the fork if it appears

   to need more memory than available → Redis background save fails

   With overcommit=1, fork succeeds, COW means actual usage is minimal

  \\\\#

 For most production servers: keep 0 (default)

 For high-security environments: use 2



vm.overcommit \_ratio = 80

 Only used when overcommit \_memory = 2

 System allows: swap + (physical \_RAM × ratio/100) total memory

 With 32GB RAM, 8GB swap, ratio=80:

 Max = 8GB + (32GB × 0.80) = 33.6GB total allocatable



vm.dirty \_ratio = 15

 Percentage of total memory that can be dirty (waiting to be written to disk)

 before the PROCESS is forced to start writing

 DEFAULT: 20

 When a process writes data, it goes to page cache (dirty pages)

 Kernel eventually flushes to disk

 If dirty pages exceed this ratio, the WRITING PROCESS is blocked

 until dirty pages are flushed — causes application latency spikes



vm.dirty \_background \_ratio = 5


 Percentage of total memory that can be dirty before the kernel

 starts flushing IN THE BACKGROUND

 DEFAULT: 10

 Background flushing doesn't block the application

 Set lower than dirty \_ratio to start background flush early

 Prevents hitting dirty \_ratio (which blocks the app)



 For databases and write-heavy workloads:

 Lower both ratios to reduce flush storms:

vm.dirty \_ratio = 10

vm.dirty \_backgroun \_ratio = 3

 This ensures smaller, more frequent flushes instead of

 large, infrequent flushes that cause latency spikes



vm.min \_free \_kbytes = 524288

 Minimum free memory the kernel maintains (in KB)

 DEFAULT: varies based on RAM (usually very low)

 524288 = 512MB

 Kernel reserves this much free memory for critical allocations

 On busy servers, if free memory drops too low, memory allocation

 can stall waiting for reclaim → latency spike

 Set to 512MB-1GB on production servers with 32GB+ RAM

 Too high = wasted memory; too low = allocation stalls



vm.panic \_on \_oom = 0

 DEFAULT: 0

 0 = OOM killer picks a process to kill

 1 = kernel panics (reboots if panic \_on \_oom \_timeout is set)

 Use 1 for Kubernetes nodes where you'd rather reboot than

 have a half-dead node with random processes killed

 The node comes back clean, kubelet reschedules pods elsewhere

 Some FAANG companies use this on K8s nodes



vm.vfs \_cache \_pressure = 50

 How aggressively kernel reclaims inode/dentry cache

 DEFAULT: 100

 Lower = kernel holds onto filesystem metadata cache longer

 Higher = kernel reclaims cache more aggressively

 50 = bias toward keeping cache (good for file-heavy workloads)

 For jump servers, CI agents: keep 100

 For web servers, databases: 50

```



\---



###### \### Category 3: Filesystem



```bash

fs.file-max = 2097152

 Maximum number of file descriptors system-wide

 DEFAULT: varies (usually  \~100K-200K based on RAM)

 Set to 2 million for production servers

 This is the SYSTEM limit — individual process limits (ulimit) still apply

 If you set ulimit -n 65535 but fs.file-max is lower, you're capped



fs.inotify.max \_user \_watches = 524288

 Maximum inotify file watches per user

 DEFAULT: 8192

 inotify = kernel facility to watch files/directories for changes

 Used by: file sync tools, IDEs, webpack, log collectors, Prometheus node \_exporter

 8192 is laughably low for any development or monitoring server

 Kubernetes nodes with many pods need this high

 Symptom of too low: "no space left on device" (misleading error from inotify)

 or "ENOSPC: System limit for file watchers reached"



fs.inotify.max \_user \_instances = 8192

 Maximum inotify instances per user

 DEFAULT: 128

 Each monitoring tool, log collector, etc. creates an instance

 On K8s nodes with many pods, each pod may create inotify instances



fs.aio-max-nr = 1048576

 Maximum number of asynchronous I/O requests

 DEFAULT: 65536

 Databases (especially Oracle, MySQL InnoDB) use AIO heavily

 Too low = database can't submit enough parallel I/O requests = slow queries

```



\---



###### \### Category 4: Kubernetes-Specific Tuning



```bash

 ── REQUIRED FOR KUBERNETES ──



net.bridge.bridge-nf-call-iptables = 1

 Allow iptables to process bridged traffic

 REQUIRED for Kubernetes networking to work

 Pod-to-pod traffic crosses a Linux bridge — this ensures

 iptables rules (which kube-proxy manages) are applied



net.bridge.bridge-nf-call-ip6tables = 1

 Same for IPv6



net.ipv4.ip \_forward = 1

 Enable IP forwarding (routing)

 DEFAULT: 0

 REQUIRED for any machine that routes packets (routers, K8s nodes, VPN gateways)

 Without this, packets arriving for other IPs are DROPPED

 Kubernetes nodes MUST forward traffic between pods on different nodes



net.ipv4.conf.all.forwarding = 1

 Enable forwarding on all interfaces



net.ipv6.conf.all.forwarding = 1

 Same for IPv6



 ── CONNTRACK (Connection Tracking) ──



net.netfilter.nf \_conntrack \_max = 1048576

 Maximum entries in the connection tracking table

 DEFAULT: 65536 or 131072

 Every connection through iptables/kube-proxy creates a conntrack entry

 On busy K8s nodes with many services, this fills up FAST

 Symptom: "nf \_conntrack: table full, dropping packet"

 This causes RANDOM packet drops — one of the hardest issues to debug

 Monitor: cat /proc/sys/net/netfilter/nf \_conntrack \_count



net.netfilter.nf \_conntrack \_tcp \_timeout \_established = 86400

 How long to keep conntrack entries for established connections

 DEFAULT: 432000 (5 days!)

 Stale entries waste conntrack table space

 86400 (1 day) is sufficient for most workloads



net.netfilter.nf \_conntrack \_tcp \_timeout \_time \_wait = 30


 Conntrack timeout for TIME \_WAIT connections

 DEFAULT: 120

 These are already closing — don't need 2 minutes in conntrack



 ── KUBERNETES INOTIFY ──

 (covered above in filesystem section)

 K8s nodes with 100+ pods need high inotify limits

 Each pod's file watchers, ConfigMap/Secret mounts use inotify

```



\#### The Conntrack Disaster — Real Production Incident:



```bash

 Incident at NovaMart (simulation preview):

 

 Symptoms:

 - Random 5xx errors across multiple services

 - Errors appear and disappear unpredictably

 - Happens more during peak traffic

 - Individual service health checks pass

 - Network team says "no issues"

  \\\\#

 Investigation:

dmesg -T | grep conntrack

  \[Jan 15 14:23:45] nf \_conntrack: table full, dropping packet

  \[Jan 15 14:23:45] nf \_conntrack: table full, dropping packet

  \[Jan 15 14:23:46] nf \_conntrack: table full, dropping packet

  \\\\#

 Check current usage:

cat /proc/sys/net/netfilter/nf \_conntrack \_count

 65432

cat /proc/sys/net/netfilter/nf \_conntrack \_max

 65536

 99.8% full! Packets being dropped!

  \\\\#

 Immediate fix:

sysctl -w net.netfilter.nf \_conntrack \_max=1048576

  \\\\#

 Permanent fix:

echo "net.netfilter.nf \_conntrack \_max = 1048576" >> /etc/sysctl.d/40-kubernetes.conf

sysctl --system

  \\\\#

 Monitor in Prometheus:

 node \_nf \_conntrack \_entries / node \_nf \_conntrack \_entries \_limit

 Alert when > 75%

```



\---



###### \### Category 5: Security Hardening



```bash

 ── NETWORK SECURITY ──



net.ipv4.conf.all.rp \_filter = 1

 Reverse Path Filtering — anti-spoofing

 DEFAULT: varies

 1 = strict mode — drop packets if return path uses different interface

 Prevents IP address spoofing attacks

 ENABLE on all production servers



net.ipv4.conf.default.rp \_filter = 1

 Same for new interfaces created after boot



net.ipv4.conf.all.accept \_redirects = 0

net.ipv6.conf.all.accept \_redirects = 0

 Don't accept ICMP redirects

 Redirects can be used to reroute traffic through attacker's machine

 On servers with static routing, you don't need redirects



net.ipv4.conf.all.send \_redirects = 0

 Don't send ICMP redirects

 Only routers should send redirects

 Your server is not a router (even if ip \_forward is enabled for K8s)



net.ipv4.conf.all.accept \_source \_route = 0

net.ipv6.conf.all.accept \_source \_route = 0

 Reject source-routed packets

 Source routing lets the SENDER specify the route — security risk



net.ipv4.icmp \_echo \_ignore \_broadcasts = 1

 Ignore broadcast ICMP echo (ping)

 Prevents Smurf attacks



net.ipv4.icmp \_ignore \_bogus \_error \_responses = 1

 Ignore bogus ICMP error messages

 Prevents log flooding from malformed ICMP



net.ipv4.conf.all.log \_martians = 1

 Log packets with impossible source addresses

 Helps detect spoofing attempts

 Check logs: dmesg | grep martian



 ── KERNEL SECURITY ──



kernel.randomize \_va \_space = 2

 Address Space Layout Randomization (ASLR)

 DEFAULT: 2

 0 = disabled (NEVER do this in production)

 1 = randomize stack, mmap, VDSO

 2 = full randomization (stack, mmap, VDSO, heap)

 Makes buffer overflow exploits much harder



kernel.dmesg \_restrict = 1

 Restrict dmesg to root only

 DEFAULT: 0

 Prevents unprivileged users from reading kernel messages

 Kernel messages can contain sensitive info (memory addresses, hardware)



kernel.kptr \_restrict = 2

 Hide kernel pointers in /proc/kallsyms

 DEFAULT: 0

 Prevents leaking kernel memory layout (used in exploits)

 1 = hide from non-root

 2 = hide from everyone including root



kernel.yama.ptrace \_scope = 2

 Restrict ptrace (used by debuggers, strace)

 DEFAULT: 1

 0 = any process can ptrace any other (DANGEROUS)

 1 = only parent can ptrace child

 2 = only root with CAP \_SYS \_PTRACE can ptrace

 3 = no process can ptrace (maximum security)

 2 is good balance — allows root debugging, prevents user attacks



kernel.sysrq = 0

 Disable Magic SysRq key

 DEFAULT: varies (often enabled)

 SysRq allows keyboard shortcuts to perform kernel actions

 (reboot, kill all processes, sync filesystems)

 On servers (especially cloud), disable for security

 Exception: some admins keep value 176 (allows sync+reboot only)

```



\---



###### \### Category 6: Performance Tuning for Specific Workloads



###### \#### High-Traffic Web Server (Nginx/HAProxy):



```bash

 /etc/sysctl.d/10-webserver.conf

net.core.somaxconn = 65535

net.core.netdev \_max \_backlog = 65535

net.ipv4.tcp \_max \_syn \_backlog = 65535

net.ipv4.tcp \_tw \_reuse = 1

net.ipv4.ip \_local \_port \_range = 1024 65535

net.ipv4.tcp \_keepalive \_time = 60

net.ipv4.tcp \_keepalive \_intvl = 10

net.ipv4.tcp \_keepalive \_probes = 6

net.ipv4.tcp \_slow \_start \_after \_idle = 0

net.ipv4.tcp \_fastopen = 3

net.ipv4.tcp \_syncookies = 1

net.core.rmem \_max = 16777216

net.core.wmem \_max = 16777216

net.ipv4.tcp \_rmem = 4096 1048576 16777216

net.ipv4.tcp \_wmem = 4096 1048576 16777216

fs.file-max = 2097152

```



###### \#### Database Server (PostgreSQL/MySQL):



```bash

 /etc/sysctl.d/10-database.conf

vm.swappiness = 1

vm.dirty \_ratio = 10

vm.dirty \_background \_ratio = 3

vm.overcommit \_memory = 2

vm.overcommit \_ratio = 90

net.core.somaxconn = 65535

net.ipv4.tcp \_keepalive \_time = 300

net.ipv4.tcp \_keepalive \_intvl = 30

net.ipv4.tcp \_keepalive \_probes = 5

fs.file-max = 2097152

fs.aio-max-nr = 1048576

 Also: set huge pages for large shared \_buffers

vm.nr \_hugepages = 1024    # 1024 × 2MB = 2GB huge pages

 Huge pages = locked in RAM, never swapped, lower TLB pressure

 Critical for databases with large buffer pools

```



###### \#### Redis Server:



```bash

 /etc/sysctl.d/10-redis.conf

vm.swappiness = 0              # NEVER swap Redis memory

vm.overcommit \_memory = 1       # REQUIRED by Redis for fork/COW

net.core.somaxconn = 65535     # Redis accept queue

net.ipv4.tcp \_max \_syn \_backlog = 65535

 Also in Redis config:

 tcp-backlog 65535            # Must match somaxconn

```



###### \#### Kubernetes Node:



```bash

 /etc/sysctl.d/40-kubernetes.conf

 Network

net.bridge.bridge-nf-call-iptables = 1

net.bridge.bridge-nf-call-ip6tables = 1

net.ipv4.ip \_forward = 1

net.ipv4.conf.all.forwarding = 1

net.ipv6.conf.all.forwarding = 1

net.ipv4.tcp \_tw \_reuse = 1

net.ipv4.ip \_local \_port \_range = 1024 65535

net.core.somaxconn = 65535

net.ipv4.tcp \_keepalive \_time = 60



 Conntrack

net.netfilter.nf \_conntrack \_max = 1048576

net.netfilter.nf \_conntrack \_tcp \_timeout \_established = 86400

net.netfilter.nf \_conntrack \_tcp \_timeout \_time \_wait = 30



 Memory

vm.swappiness = 10

vm.panic \_on \_oom = 0

vm.min \_free \_kbytes = 524288



 Filesystem

fs.file-max = 2097152

fs.inotify.max \_user \_watches = 524288

fs.inotify.max \_user \_instances = 8192



 Security

net.ipv4.conf.all.rp \_filter = 1

net.ipv4.conf.all.accept \_redirects = 0

net.ipv4.conf.all.send \_redirects = 0

kernel.panic = 10    # Auto-reboot 10 seconds after kernel panic

                    # On K8s nodes, fast reboot is better than staying dead

```



\---



###### \### Setting sysctl in Kubernetes Pods



Some sysctls can be set per-pod (those in namespaced kernel parameters):



```yaml

apiVersion: v1

kind: Pod

metadata:

 name: myapp

spec:

 securityContext:

   sysctls:

   - name: net.core.somaxconn

     value: "65535"

   - name: net.ipv4.tcp \_keepalive \_time

     value: "60"

 containers:

 - name: myapp

   image: myapp:latest

```



\*\*But there's a catch:\*\*



```bash

 Kubernetes classifies sysctls as:

 SAFE — namespaced, doesn't affect other pods

   net.ipv4.ping \_group \_range

   net.ipv4.ip \_unprivileged \_port \_start

  \\\\#

 UNSAFE — could affect the node or other pods

   net.core.somaxconn

   net.ipv4.tcp \_keepalive \_time

   (most useful ones are "unsafe")



 To allow unsafe sysctls, kubelet must be configured:

 --allowed-unsafe-sysctls="net.core.somaxconn,net.ipv4.tcp \_keepalive \_time"



 Or in kubelet config:

apiVersion: kubelet.config.k8s.io/v1beta1

kind: KubeletConfiguration

allowedUnsafeSysctls:

 - "net.core.somaxconn"

 - "net.ipv4.tcp \_keepalive \_time"

 - "net.ipv4.tcp \_tw \_reuse"



 For node-level sysctls that CAN'T be namespaced:

 Use a DaemonSet with init containers or a startup script

 Or configure via Ansible/Terraform in the node AMI

```



\---



\### How to Apply sysctl via Ansible (Production Method):



```yaml

 roles/base/tasks/sysctl.yml

  \\\\- name: Apply kernel tuning parameters

 ansible.posix.sysctl:

   name: "{{ item.name }}"

   value: "{{ item.value }}"

   sysctl \_file: /etc/sysctl.d/10-production.conf

   reload: yes

   state: present

 loop:

   - { name: 'net.core.somaxconn', value: '65535' }

   - { name: 'net.ipv4.tcp \_tw \_reuse', value: '1' }

   - { name: 'net.ipv4.ip \_local \_port \_range', value: '1024 65535' }

   - { name: 'vm.swappiness', value: '10' }

   - { name: 'fs.file-max', value: '2097152' }

   # ... etc

```



\### How to Apply sysctl via Terraform (EKS Node Groups):



```hcl

 In the EKS node group launch template user \_data:

resource "aws \_launch \_template" "eks \_nodes" {

 user \_data = base64encode(<<-USERDATA

   #!/bin/bash

   cat >> /etc/sysctl.d/40-kubernetes.conf << 'EOF'

   net.bridge.bridge-nf-call-iptables = 1

   net.ipv4.ip \_forward = 1

   net.netfilter.nf \_conntrack \_max = 1048576

   vm.swappiness = 10

   fs.file-max = 2097152

   fs.inotify.max \_user \_watches = 524288

   EOF

   sysctl --system

 USERDATA

 )

}

```



\---



###### \### Production Scenarios



###### \#### Scenario 1: Intermittent Connection Failures on High-Traffic Service



```bash

 Symptoms:

 - API gateway returns 502 errors randomly

 - More frequent during peak hours (Black Friday)

 - Network team says "no packet loss on switches"

 - Individual services are healthy



 Investigation:

netstat -s | grep -i "listen"

 23456 times the listen queue of a socket overflowed

 ^^^^^^ THIS NUMBER IS GROWING



ss -ltn

 State   Recv-Q  Send-Q  Local Address:Port

 LISTEN  129     128          \*:8080

         ^^^ Recv-Q > Send-Q = backlog is FULL



 Recv-Q = current connections waiting in accept queue

 Send-Q = maximum accept queue size (somaxconn or listen backlog)

 When Recv-Q >= Send-Q → connections are being DROPPED



 Root cause: somaxconn = 128 (default)

 Fix:

sysctl -w net.core.somaxconn=65535



 Also need to set in application:

 Nginx: listen 80 backlog=65535;

 Java: server.tomcat.accept-count=65535

 Node.js: server.listen(8080, {backlog: 65535})

 The kernel somaxconn CAPS the application's backlog setting

 Both must be set high

```



###### \#### Scenario 2: Database Latency Spikes Every Few Minutes



```bash

 Symptoms:

 - PostgreSQL query latency spikes to 500ms every 2-3 minutes

 - Normal latency is 2ms

 - Spikes last 5-10 seconds

 - Disk I/O shows burst of writes during spikes



 Investigation:

cat /proc/sys/vm/dirty \_ratio

 20

cat /proc/sys/vm/dirty \_background \_ratio

 10



 What's happening:

 Application writes dirty pages to page cache

 When dirty pages reach 10% of RAM → background flush starts

 On a 64GB server: 10% = 6.4GB of dirty data!

 The flush writes 6.4GB to disk all at once

 This overwhelms the disk → everything waiting on I/O → latency spike

 Then cache is clean again → normal for 2-3 minutes → repeat



 Fix:

sysctl -w vm.dirty \_ratio=10

sysctl -w vm.dirty \_background \_ratio=3

 Now: background flush starts at 3% (1.9GB) — much smaller flushes

 Hard limit at 10% (6.4GB) — processes blocked only if flush can't keep up

 Flushes are smaller and more frequent → no spikes

```



###### \#### Scenario 3: Kubernetes Node Goes NotReady After Running for Weeks



```bash

 Symptoms:

 - Node was healthy for 3 weeks

 - Suddenly goes NotReady

 - Pods on the node start failing

 - kubectl describe node shows: "KubeletNotReady"

 - SSH to node still works



 Investigation:

dmesg -T | tail -50

 \[Jan 15 03:42:11] nf \_conntrack: table full, dropping packet

 \[Jan 15 03:42:11] nf \_conntrack: table full, dropping packet



cat /proc/sys/net/netfilter/nf \_conntrack \_count

 131072

cat /proc/sys/net/netfilter/nf \_conntrack \_max

 131072

 FULL!



 Why did it take 3 weeks?

 Conntrack entries accumulate slowly

 Long-lived connections (database pools, gRPC streams) don't close

 Default timeout for established connections: 5 DAYS

 After 3 weeks: enough accumulated connections + stale entries = full



 Kubelet uses the network to report status to API server

 Dropped packets → kubelet can't heartbeat → node goes NotReady



 Immediate fix:

sysctl -w net.netfilter.nf \_conntrack \_max=1048576

sysctl -w net.netfilter.nf \_conntrack \_tcp \_timeout \_established=86400

 Reduce timeout from 5 days to 1 day → stale entries expire faster



 Permanent fix in node AMI/launch template + monitoring

```



###### \#### Scenario 4: Redis Background Save Fails on Large Datasets



```bash

 Symptoms:

 Redis logs: "Can't save in background: fork: Cannot allocate memory"

 Redis has 20GB of data, server has 32GB RAM

 free -m shows 8GB available



 What's happening:

 Redis forks to create RDB snapshots (background save)

 Fork creates a copy of the process (copy-on-write)

 Kernel sees: "process wants 20GB, I only have 8GB free"

 With overcommit \_memory=0: kernel denies the fork

 Even though COW means it would only use a fraction of that



 Fix:

sysctl -w vm.overcommit \_memory=1

 Now kernel allows the fork

 COW means actual memory usage is minimal during save

 Redis documentation REQUIRES this setting



 Also ensure:

 - Swap is configured (small amount, 1-2GB) as safety net

 - vm.swappiness=0 (never actually use swap, just have it)

 - Monitor memory usage and alert at 70% of physical RAM

```



\---



\### Monitoring sysctl Parameters



```bash

 Prometheus node \_exporter exposes many of these automatically:



 Conntrack:

 node \_nf \_conntrack \_entries        → current count

 node \_nf \_conntrack \_entries \_limit  → max limit

 Alert: entries/limit > 0.75



 Network:

 node \_sockstat \_TCP \_tw             → TIME \_WAIT socket count

 node \_netstat \_Tcp \_ListenOverflows → listen queue overflows

 node \_netstat \_Tcp \_ListenDrops     → listen queue drops



 Memory:

 node \_memory \_Dirty \_bytes          → current dirty pages

 node \_vmstat \_pgmajfault           → major page faults (reading from disk)



 File descriptors:

 node \_filefd \_allocated            → current system-wide FDs

 node \_filefd \_maximum              → system-wide FD limit

 Alert: allocated/maximum > 0.75



 Custom check script (for parameters not in node \_exporter):

  \\\\#!/bin/bash

echo "conntrack \_usage $(cat /proc/sys/net/netfilter/nf \_conntrack \_count)"

echo "conntrack \_max $(cat /proc/sys/net/netfilter/nf \_conntrack \_max)"

echo "somaxconn $(cat /proc/sys/net/core/somaxconn)"

echo "file \_max $(cat /proc/sys/fs/file-max)"

echo "tcp \_tw \_reuse $(cat /proc/sys/net/ipv4/tcp \_tw \_reuse)"

```



\---



\### Validating Tuning Is Applied



```bash

 After applying sysctl changes, VERIFY:



 Method 1: Direct check

sysctl net.core.somaxconn

 Should show new value



 Method 2: Check from /proc

cat /proc/sys/net/core/somaxconn



 Method 3: Check running config vs file

sysctl -a > /tmp/running.txt

diff <(sort /etc/sysctl.d/40-kubernetes.conf | grep -v '^#' | grep -v '^$')      \\

    <(sysctl -a | sort | grep -F -f <(grep -v '^#' /etc/sysctl.d/40-kubernetes.conf | grep -v '^$' | cut -d= -f1 | tr -d ' '))



 Method 4: Ansible verification

  \\\\- name: Verify sysctl settings

 command: sysctl {{ item.name }}

 register: result

 failed \_when: "item.value not in result.stdout"

 loop:

   - { name: 'net.core.somaxconn', value: '65535' }

   # ... etc

```



\---



\### ⚠️ Dangerous sysctl Mistakes



```bash

 MISTAKE 1: Setting vm.swappiness=0 on a server without monitoring

 If RAM fills up completely, OOM killer activates immediately

 No swap = no buffer = instant process death

 Always have SOME swap (even 1GB) as a safety net unless you know exactly why not



 MISTAKE 2: Setting net.ipv4.tcp \_tw \_recycle=1

 REMOVED from kernel 4.12+ because it breaks connections behind NAT

 Many old blog posts still recommend it. IGNORE THEM.



 MISTAKE 3: Setting kernel.shmmax too low before installing PostgreSQL

 PostgreSQL will fail to start: "could not create shared memory segment"

 Check: sysctl kernel.shmmax

 Set: sysctl -w kernel.shmmax=17179869184  (16GB)



 MISTAKE 4: Enabling ip \_forward without firewall rules

 You just turned your server into a router

 Without iptables rules, traffic flows through unrestricted

 Potential security breach — anyone who can reach your server can route through it



 MISTAKE 5: Not persisting changes

 sysctl -w changes are TEMPORARY — lost on reboot

 ALWAYS write to /etc/sysctl.d/ AND run sysctl -w

 Or just write the file and run sysctl --system



 MISTAKE 6: Copy-pasting sysctl configs from the internet without understanding

 Every parameter is workload-specific

 Redis needs overcommit \_memory=1, but a general server shouldn't

 Database needs swappiness=1, but a CI runner might want 60

 UNDERSTAND what each parameter does before setting it

```





One addition I should make:



```bash

 TRANSPARENT HUGE PAGES (THP) — Database Killer

 Linux enables THP by default — dynamically allocates 2MB pages

 Sounds good but causes MASSIVE latency spikes for databases

 Kernel pauses processes to defragment memory into huge pages



 Check status:

cat /sys/kernel/mm/transparent \_hugepage/enabled

      \[always] madvise never



 Disable for database servers (Redis, MongoDB, PostgreSQL):

echo never > /sys/kernel/mm/transparent \_hugepage/enabled

echo never > /sys/kernel/mm/transparent \_hugepage/defrag



 Make persistent via systemd unit or rc.local

 Redis and MongoDB documentation explicitly say: DISABLE THP

```





\# 📋 LESSON 8 QUICK REFERENCE — Kernel Tuning (sysctl)



```

HOW SYSCTL WORKS:

 sysctl -w key=value              → Set temporarily

 /etc/sysctl.d/XX-name.conf      → Set permanently (XX = priority)

 sysctl --system                  → Apply all config files

 sysctl -a                        → View all parameters



NETWORK TUNING:

 net.core.somaxconn = 65535                    → Accept queue size

 net.core.netdev \_max \_backlog = 65535          → NIC input queue

 net.ipv4.tcp \_max \_syn \_backlog = 65535         → Half-open connection queue

 net.ipv4.tcp \_tw \_reuse = 1                    → Reuse TIME \_WAIT sockets

 net.ipv4.ip \_local \_port \_range = 1024 65535    → Ephemeral port range

 net.ipv4.tcp \_keepalive \_time = 300            → First keepalive probe

 net.ipv4.tcp \_slow \_start \_after \_idle = 0       → Maintain window after idle

 net.ipv4.tcp \_fastopen = 3                    → TCP Fast Open (client+server)

 net.ipv4.tcp \_syncookies = 1                  → SYN flood protection

 net.core.rmem \_max/wmem \_max = 16777216        → Max buffer 16MB

 net.ipv4.tcp \_mtu \_probing = 1                 → Path MTU discovery



MEMORY TUNING:

 vm.swappiness = 10 (server), 1 (database), 0 (Redis)

 vm.overcommit \_memory = 0 (default), 1 (Redis), 2 (strict)

 vm.dirty \_ratio = 10-15              → Block process above this %

 vm.dirty \_background \_ratio = 3-5     → Background flush above this %

 vm.min \_free \_kbytes = 524288         → 512MB reserved free memory

 vm.panic \_on \_oom = 0 or 1            → Reboot vs OOM kill

 Disable THP for databases: echo never > .../transparent \_hugepage/enabled



FILESYSTEM:

 fs.file-max = 2097152               → System-wide FD limit

 fs.inotify.max \_user \_watches = 524288 → File watchers (K8s needs this)

 fs.inotify.max \_user \_instances = 8192

 fs.aio-max-nr = 1048576             → Async I/O (databases)



KUBERNETES REQUIRED:

 net.bridge.bridge-nf-call-iptables = 1

 net.ipv4.ip \_forward = 1

 net.netfilter.nf \_conntrack \_max = 1048576

 kernel.panic = 10                    → Auto-reboot after panic



SECURITY:

 net.ipv4.conf.all.rp \_filter = 1     → Anti-spoofing

 net.ipv4.conf.all.accept \_redirects = 0

 net.ipv4.conf.all.send \_redirects = 0

 kernel.randomize \_va \_space = 2        → ASLR

 kernel.dmesg \_restrict = 1            → Root-only dmesg

 kernel.yama.ptrace \_scope = 2         → Restrict ptrace



CONNTRACK MONITORING:

 /proc/sys/net/netfilter/nf \_conntrack \_count  → Current

 /proc/sys/net/netfilter/nf \_conntrack \_max    → Limit

 Alert when count/max > 75%

 "table full, dropping packet" = IMMEDIATE action needed



COMMON MISTAKES:

 - tcp \_tw \_recycle removed in kernel 4.12+ → NEVER use

 - swappiness=0 without swap = instant OOM kill

 - ip \_forward=1 without firewall = open router

 - sysctl -w without writing config file = lost on reboot

 - Copy-pasting configs without understanding workload needs

```





\# 📝 Retention Questions — Lesson 8



\*\*Q1:\*\* Your Nginx reverse proxy starts dropping connections during a traffic spike. `ss -ltn` shows `Recv-Q: 129, Send-Q: 128` for port 80. What's the exact problem and what TWO things do you need to change (one kernel, one application)?



\*\*Q2:\*\* Your PostgreSQL database has periodic latency spikes lasting 5-10 seconds every few minutes. `iostat` shows burst writes during spikes. What kernel parameter is likely misconfigured, what's happening at the kernel level, and what values would you set?



\*\*Q3:\*\* A Redis server with 20GB of data fails background save with "Cannot allocate memory" even though `free -m` shows 8GB available. What kernel parameter needs to change and WHY does the default behavior cause this failure? Explain the fork + copy-on-write mechanism.



\*\*Q4:\*\* A Kubernetes node running 80+ pods suddenly starts dropping random packets. `dmesg` shows "nf\_conntrack: table full, dropping packet." What is conntrack, why did it fill up, what's your immediate fix, and how do you monitor this going forward?



\*\*Q5:\*\* An engineer applies sysctl tuning via `sysctl -w` commands during an incident and everything improves. Two weeks later the server reboots for a kernel update and all the problems return. What went wrong and what's the production-grade way to manage sysctl across a fleet of 200 servers?



\*\*Go.\*\* 🎯



===========================================================================================



=====================================================================================

\---



### \# 🌐 PHASE 1 — NETWORKING FUNDAMENTALS



You scored \*\*0/10\*\* on networking in the entrance exam. You said "dont know" when I asked what happens when you `curl google.com`. That ends today.



Networking is the \*\*single most important skill\*\* for a DevOps/SRE engineer. Every outage, every architecture decision, every security rule, every service mesh, every load balancer, every DNS resolution — it's all networking. If you don't understand networking, everything else you build is a house of cards.



\---



##### \## Lesson 1: The Network Models — OSI \& TCP/IP



Before we dive into protocols, you need the mental framework for how network communication is structured. These models describe \*\*layers of abstraction\*\* — each layer has a specific job and talks to the layers above and below it.



\---



\### The OSI Model (7 Layers) — The Reference Model



```

Layer 7: APPLICATION    → HTTP, HTTPS, DNS, FTP, SMTP, gRPC, WebSocket

Layer 6: PRESENTATION   → SSL/TLS encryption, data formatting, compression

Layer 5: SESSION        → Session management, authentication tokens

Layer 4: TRANSPORT      → TCP, UDP — port numbers, reliability

Layer 3: NETWORK        → IP addressing, routing — source/destination IPs

Layer 2: DATA LINK      → MAC addresses, Ethernet frames, switches, ARP

Layer 1: PHYSICAL       → Cables, radio waves, electrical signals, NICs

```



\*\*Memory aid:\*\* "\*\*P\*\*lease \*\*D\*\*o \*\*N\*\*ot \*\*T\*\*hrow \*\*S\*\*ausage \*\*P\*\*izza \*\*A\*\*way" (bottom to top)



\### The TCP/IP Model (4 Layers) — What's Actually Used



The OSI model is theoretical. The TCP/IP model is what the internet actually runs on:



```

TCP/IP Layer        OSI Equivalent      Protocols

─────────────       ──────────────      ─────────

APPLICATION         Layers 5-7          HTTP, DNS, TLS, SSH, gRPC

TRANSPORT           Layer 4             TCP, UDP

INTERNET            Layer 3             IP, ICMP, ARP

NETWORK ACCESS      Layers 1-2         Ethernet, WiFi, MAC

```



\### Why Layers Matter for DevOps:



When you're troubleshooting, layers help you \*\*isolate where the problem is\*\*:



```

"Can't reach the website"



Layer 1: Is the cable plugged in? Is the NIC up?

 → ip link show (is the interface UP?)



Layer 2: Can we reach the local network?

 → arping <gateway> (is ARP resolving?)



Layer 3: Can we reach the remote IP?

 → ping <ip> (is IP routing working?)

 → traceroute <ip> (where does the path break?)



Layer 4: Can we reach the remote port?

 → telnet <ip> <port> or nc -zv <ip> <port>

 → ss -tlnp (is anything listening?)



Layer 7: Is the application responding correctly?

 → curl -vvv http://<ip>:<port>/health

 → Is it returning 200? 502? Timeout?

```



\*\*This layered approach to debugging is how the top 1% troubleshoots.\*\* They don't randomly try things. They systematically eliminate layers from bottom to top.



\---



\### Encapsulation — How Data Travels Through Layers



When you send data, each layer wraps the data in its own header:



```

APPLICATION LAYER:

 \[HTTP Request: GET /index.html]



TRANSPORT LAYER adds TCP header:

 \[TCP Header: SrcPort=45678, DstPort=80, SeQ=1, ACK=0, Flags=SYN]

 \[HTTP Request: GET /index.html]



NETWORK LAYER adds IP header:

 \[IP Header: Src=192.168.1.10, Dst=93.184.216.34, TTL=64, Protocol=TCP]

 \[TCP Header: SrcPort=45678, DstPort=80]

 \[HTTP Request: GET /index.html]



DATA LINK LAYER adds Ethernet frame:

 \[Ethernet: SrcMAC=aa:bb:cc:dd:ee:ff, DstMAC=11:22:33:44:55:66, Type=IPv4]

 \[IP Header]

 \[TCP Header]

 \[HTTP Request]

 \[Ethernet FCS (checksum)]

```



At each hop (router, switch), the outer layers are stripped and rebuilt. The Ethernet frame changes at every hop (new MAC addresses), but the IP header stays the same end-to-end (source and destination IPs don't change).



\*\*Why this matters:\*\*

\- \*\*Load balancers\*\* operate at different layers — Layer 4 (TCP) vs Layer 7 (HTTP) — and this determines what they can see and do

\- \*\*Firewalls/Security Groups\*\* filter at Layer 3/4 (IP + Port)

\- \*\*WAFs\*\* filter at Layer 7 (HTTP content)

\- \*\*tcpdump\*\* captures at Layer 2 — you see everything

\- \*\*Wireshark\*\* decodes all layers for analysis



\---



\### MTU — Maximum Transmission Unit



```bash

 MTU = maximum size of a single packet at Layer 2

 Standard Ethernet MTU: 1500 bytes

 AWS VPC within same region: 9001 bytes (jumbo frames)

 AWS across regions/VPN: 1500 bytes



 Check MTU of an interface:

ip link show eth0

 ... mtu 9001 ...



 Why MTU matters:

 If your packet is larger than the path MTU:

 Option 1: Fragmentation — packet split into pieces (slow, unreliable)

 Option 2: Drop + ICMP "Fragmentation Needed" (if DF bit set)

 Option 3: Black hole — packet dropped, no notification (PMTUD failure)



 Real-world disaster:

 VPN tunnel has MTU 1400 (overhead from encryption headers)

 Application sends 1500-byte packets

 DF (Don't Fragment) bit is set

 Packets are too big for the tunnel

 Packets get silently dropped

 Symptom: small requests work, large responses fail

 SSH works, SCP hangs after transferring a few KB

 curl works for small responses, hangs for large ones



 Debug:

ping -M do -s 1472 <destination>

 -M do = set DF bit (don't fragment)

 -s 1472 = payload size (1472 + 20 IP + 8 ICMP = 1500 total)

 Reduce -s until ping succeeds = that's your path MTU



 Fix:

 Option 1: Reduce MTU on the interface

ip link set dev eth0 mtu 1400



 Option 2: Enable TCP MSS clamping (for VPNs/tunnels)

iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN  \\

 -j TCPMSS --clamp-mss-to-pmtu



 Option 3: Enable MTU path discovery (we covered this in sysctl)

sysctl -w net.ipv4.tcp \_mtu \_probing=1

```



\---



##### \## Lesson 2: IP Addressing, Subnets \& CIDR



\### IPv4 Address Structure



```

IP Address: 192.168.1.100

Binary:     11000000.10101000.00000001.01100100



4 octets, each 0-255

Total: 32 bits =  \~4.3 billion addresses (not enough for the world)

```



\### Private vs Public IP Ranges



```

PRIVATE (not routable on the internet):

 10.0.0.0/8       → 10.0.0.0 - 10.255.255.255     (16.7M addresses)

 172.16.0.0/12    → 172.16.0.0 - 172.31.255.255    (1M addresses)

 192.168.0.0/16   → 192.168.0.0 - 192.168.255.255  (65K addresses)



SPECIAL:

 127.0.0.0/8      → Loopback (localhost)

 169.254.0.0/16   → Link-local (APIPA — when DHCP fails)

 0.0.0.0          → "All interfaces" or "default route"

 255.255.255.255  → Broadcast



PUBLIC:

 Everything else — assigned by IANA/RIRs to ISPs and organizations

```



\*\*AWS VPCs use private ranges.\*\* Your EC2 instances get private IPs (e.g., 10.0.1.50). Public access goes through NAT Gateways, Elastic IPs, or Load Balancers that translate between public and private.



\### CIDR Notation — The Subnet Language



CIDR (Classless Inter-Domain Routing) defines network ranges:



```

Format: IP/prefix \_length

The prefix = number of bits that define the NETWORK portion

Remaining bits = HOST portion (individual addresses)



192.168.1.0/24

 Network: 192.168.1     (first 24 bits are fixed)

 Hosts:   .0 - .255     (last 8 bits vary)

 Usable:  .1 - .254     (256 - 2 = 254 hosts)

 .0 = network address   (identifies the network itself)

 .255 = broadcast        (reaches all hosts on network)



CIDR CHEAT SHEET:

 /32 = 1 IP        (single host — used in security group rules)

 /31 = 2 IPs       (point-to-point links)

 /30 = 4 IPs       (2 usable)

 /28 = 16 IPs      (14 usable — smallest AWS subnet)

 /27 = 32 IPs      (30 usable)

 /26 = 64 IPs      (62 usable)

 /25 = 128 IPs     (126 usable)

 /24 = 256 IPs     (254 usable — the classic "Class C")

 /20 = 4,096 IPs   (4,094 usable)

 /16 = 65,536 IPs  (65,534 usable — the classic "Class B")

 /8  = 16.7M IPs   (the classic "Class A")



QUICK MATH:

 Number of IPs = 2^(32 - prefix)

 /24 = 2^(32-24) = 2^8 = 256

 /20 = 2^(32-20) = 2^12 = 4096

 /16 = 2^(32-16) = 2^16 = 65536

```



\### Subnet Masks — The Other Way to Write It



```

/24 = 255.255.255.0

/16 = 255.255.0.0

/8  = 255.0.0.0

/20 = 255.255.240.0

/27 = 255.255.255.224



 The mask tells you which bits are NETWORK (1s) and which are HOST (0s)

 /24 = 11111111.11111111.11111111.00000000 = 255.255.255.0

```



\### AWS VPC Subnetting — Real Architecture



```

VPC CIDR: 10.0.0.0/16 (65,536 IPs)



 Public Subnets (internet-facing):

   10.0.1.0/24   → us-east-1a  (254 IPs)

   10.0.2.0/24   → us-east-1b  (254 IPs)

   10.0.3.0/24   → us-east-1c  (254 IPs)



 Private Subnets (application tier):

   10.0.10.0/24  → us-east-1a  (254 IPs)

   10.0.20.0/24  → us-east-1b  (254 IPs)

   10.0.30.0/24  → us-east-1c  (254 IPs)



 Database Subnets (data tier):

   10.0.100.0/24 → us-east-1a  (254 IPs)

   10.0.200.0/24 → us-east-1b  (254 IPs)



 ⚠️ AWS reserves 5 IPs per subnet:

   .0   = Network address

   .1   = VPC router

   .2   = DNS server

   .3   = Reserved for future use

   .255 = Broadcast (not supported in VPC but reserved)

   

 So a /24 in AWS gives you 251 usable IPs, not 254

 A /28 (smallest allowed) gives you 11 usable IPs, not 14

```



\### Kubernetes Networking CIDR:



```

 K8s has THREE separate CIDR ranges:



Node Network:    10.0.0.0/16    → IP addresses of the nodes themselves

Pod Network:     172.16.0.0/16  → IP addresses assigned to pods

Service Network: 10.96.0.0/12   → Virtual IPs for Kubernetes Services



 These MUST NOT overlap

 Each node gets a subnet from the pod CIDR:

   Node 1: pods get IPs from 172.16.0.0/24

   Node 2: pods get IPs from 172.16.1.0/24

   Node 3: pods get IPs from 172.16.2.0/24



 This is managed by the CNI plugin (Calico, Cilium, AWS VPC CNI)



 AWS VPC CNI is special:

 Pods get REAL VPC IPs (not overlay network IPs)

 This means pods are directly routable in the VPC

 But it EATS VPC IPs fast — each pod consumes a real IP

 With 200 pods per node, you need large subnets

 This is why EKS clusters often use /18 or /16 subnets

```



\---



##### \## Lesson 3: DNS — The Internet's Phone Book



DNS is involved in \*\*literally every network request your application makes\*\*. Database connections, API calls, service discovery, email — all start with DNS. Misunderstanding DNS causes some of the most confusing production outages.



\### How DNS Resolution Works — Step by Step



When your app does `curl api.example.com`:



```

Step 1: Check LOCAL CACHE

 → Browser/OS/application has a DNS cache

 → If cached and TTL hasn't expired → use cached IP → DONE



Step 2: Check /etc/hosts

 → Local static mappings file

 → 127.0.0.1 localhost

 → If found → use this IP → DONE



Step 3: Check /etc/resolv.conf

 → Which DNS server to ask?

 → nameserver 10.0.0.2 (AWS VPC default)

 → nameserver 8.8.8.8 (Google Public DNS)



Step 4: Query RECURSIVE RESOLVER (your configured DNS server)

 → Client asks: "What is api.example.com?"

 → Resolver checks ITS cache

 → If cached → return answer → DONE

 → If not → resolver starts the recursive lookup:



Step 5: Resolver queries ROOT SERVERS

 → "Who handles .com?"

 → Root server: "Ask the .com TLD servers at a.gtld-servers.net"

 → (There are 13 root server clusters globally)



Step 6: Resolver queries TLD SERVER (.com)

 → "Who handles example.com?"

 → TLD server: "Ask ns1.example.com at 198.51.100.1"



Step 7: Resolver queries AUTHORITATIVE SERVER (ns1.example.com)

 → "What is api.example.com?"

 → Authoritative: "api.example.com = 93.184.216.34, TTL=300"



Step 8: Resolver CACHES the answer (for TTL seconds) and returns to client



Step 9: Client connects to 93.184.216.34

```



\### DNS Record Types You MUST Know:



```

A Record:

 api.example.com → 93.184.216.34

 Maps hostname to IPv4 address

 The most common record type



AAAA Record:

 api.example.com → 2606:2800:220:1:248:1893:25c8:1946

 Maps hostname to IPv6 address



CNAME Record:

 www.example.com → example.com

 Alias — points one name to another name

 The target must be resolved again

 CANNOT coexist with other records on the same name

 CANNOT be used for the zone apex (example.com itself)

 Common use: www subdomain, CDN aliases



ALIAS / ANAME Record:

 example.com → d1234.cloudfront.net

 Like CNAME but CAN be used at zone apex

 AWS Route53 calls this an "Alias record"

 Resolves to IPs at query time



MX Record:

 example.com → 10 mail.example.com

 Mail exchange — where to deliver email

 Number = priority (lower = preferred)



TXT Record:

 example.com → "v=spf1 include: \_spf.google.com      \~all"

 Arbitrary text — used for:

   SPF (email authentication)

   DKIM (email signing)

   DMARC (email policy)

   Domain verification (Google, AWS ACM, Let's Encrypt)



NS Record:

 example.com → ns1.example.com, ns2.example.com

 Declares the authoritative nameservers for a domain

 Delegation — "these servers are responsible for this zone"



SOA Record:

 Start of Authority — metadata about the zone

 Serial number, refresh intervals, admin email

 Every DNS zone has exactly one SOA record



SRV Record:

  \_http. \_tcp.example.com → 10 60 8080 web1.example.com

 Service location — port and host for a service

 Format: priority weight port target

 Used by: Kubernetes DNS, SIP, LDAP, some service discovery



PTR Record:

 34.216.184.93.in-addr.arpa → api.example.com

 Reverse DNS — IP to hostname

 Used for: email server verification, logging, security audits

```



\### DNS in Kubernetes — CoreDNS



```bash

 Every Kubernetes cluster runs CoreDNS (or kube-dns)

 It provides service discovery via DNS



 Service DNS:

 <service-name>.<namespace>.svc.cluster.local

 my-service.default.svc.cluster.local → 10.96.0.15 (ClusterIP)



 Pod DNS:

 <pod-ip-dashed>.<namespace>.pod.cluster.local

 172-16-0-5.default.pod.cluster.local → 172.16.0.5



 Headless Service DNS (ClusterIP: None):

 Returns individual pod IPs instead of a single ClusterIP

 Used for StatefulSets (databases, Kafka, etc.)

 my-db.default.svc.cluster.local → 172.16.0.5, 172.16.0.6, 172.16.0.7



 StatefulSet DNS:

 <pod-name>.<service-name>.<namespace>.svc.cluster.local

 my-db-0.my-db.default.svc.cluster.local → 172.16.0.5

 my-db-1.my-db.default.svc.cluster.local → 172.16.0.6

 Each pod gets a STABLE DNS name — critical for databases



 How pods resolve DNS — /etc/resolv.conf inside a pod:

cat /etc/resolv.conf

 nameserver 10.96.0.10         ← CoreDNS ClusterIP

 search default.svc.cluster.local svc.cluster.local cluster.local

 ndots:5



 The "search" line is why you can just say "my-service"

 instead of "my-service.default.svc.cluster.local"

 Kubernetes tries appending each search domain:

   my-service.default.svc.cluster.local ← found!

```



\### ndots:5 — The Hidden Performance Killer



```bash

 ndots:5 means: if the name has fewer than 5 dots,

 try ALL search domains BEFORE querying as absolute



 When your app calls: api.external-service.com (2 dots, < 5)

 Kubernetes DNS tries:

   1. api.external-service.com.default.svc.cluster.local → NXDOMAIN

   2. api.external-service.com.svc.cluster.local → NXDOMAIN

   3. api.external-service.com.cluster.local → NXDOMAIN

   4. api.external-service.com → RESOLVED!

 

 That's 4 DNS queries for ONE lookup!

 At 1000 requests/second: 4000 unnecessary DNS queries/second

 CoreDNS overloaded, DNS latency increases, everything slows down



 FIX 1: Add trailing dot to external domains in your app config

 "api.external-service.com." ← the trailing dot means "absolute name"

 Skips all search domain appending



 FIX 2: Lower ndots in pod spec

spec:

 dnsConfig:

   options:

   - name: ndots

     value: "2"

 # Now only names with < 2 dots get search domains appended

 # "my-service" (0 dots) → still searches cluster domains

 # "api.external.com" (2 dots) → queried directly



 FIX 3: Use NodeLocal DNSCache DaemonSet

 Runs a DNS cache on every node

 Reduces CoreDNS load by 80%+

 Google recommends this for all GKE clusters

 AWS EKS has equivalent

```



\### DNS TTL — The Caching Trap



```bash

 TTL = Time To Live = how long DNS answers are cached



 Low TTL (30-60 seconds):

 + Fast propagation of changes (failover, blue-green deploys)

 - More DNS queries (higher load on DNS servers)

 Use for: load balancer endpoints, services that need fast failover



 High TTL (3600+ seconds):

 + Fewer DNS queries, faster resolution from cache

 - Slow propagation — changes take hours to reach all clients

 Use for: static infrastructure, MX records, NS records



 THE TTL TRAP:

 You set TTL=300 (5 minutes) on api.example.com

 You change the IP (migration, failover)

 You expect everyone to get the new IP in 5 minutes

 But SOME clients still hit the old IP after 30 minutes!

 

 Why?

 1. Some DNS resolvers IGNORE your TTL and cache longer

 2. Java's DNS cache (InetAddress) caches FOREVER by default

 3. Some ISP resolvers have minimum TTL of 300s regardless

 4. Operating system DNS cache has its own TTL



 Java DNS caching fix:

 In jvm options or code:

java -Dsun.net.inetaddress.ttl=60 -jar app.jar

 Or in $JAVA \_HOME/conf/security/java.security:

networkaddress.cache.ttl=60

networkaddress.cache.negative.ttl=10



 AWS Route53 Alias records:

 TTL is automatically set to match the target resource

 ALB alias: TTL=60 (managed by AWS)

 You CANNOT set custom TTL on Alias records

```



\### dig and nslookup — DNS Debugging Tools



```bash

 dig — the superior DNS debugging tool

dig api.example.com

 Shows: query, answer, authority, additional sections

 Answer section shows the actual record + TTL



dig api.example.com +short

 Just the IP: 93.184.216.34



dig api.example.com +trace

 Shows the ENTIRE resolution chain:

 Root → TLD → Authoritative

 Invaluable for debugging delegation issues



dig @8.8.8.8 api.example.com

 Query a specific DNS server (bypass local resolver)



dig api.example.com MX

 Query specific record type



dig -x 93.184.216.34

 Reverse DNS lookup (PTR record)



dig api.example.com +norecurse @ns1.example.com

 Query authoritative server directly without recursion

 Shows what the source of truth says



 nslookup — simpler but less powerful

nslookup api.example.com

nslookup api.example.com 8.8.8.8



 host — simplest

host api.example.com



 DEBUGGING DNS INSIDE KUBERNETES:

kubectl run debug --rm -it --image=nicolaka/netshoot -- bash

dig my-service.default.svc.cluster.local

dig kubernetes.default.svc.cluster.local

 This tells you if CoreDNS is working



 Check CoreDNS pods:

kubectl get pods -n kube-system -l k8s-app=kube-dns

kubectl logs -n kube-system -l k8s-app=kube-dns



 Check CoreDNS config:

kubectl get configmap coredns -n kube-system -o yaml

```



\### AWS Route53 — DNS at Scale



```bash

 Route53 is AWS's managed DNS service

 Supports:

   Public hosted zones (internet-facing)

   Private hosted zones (VPC-internal resolution)



 Routing Policies:

 Simple        → Single IP or random from multiple

 Weighted      → 70% to server A, 30% to server B (canary deployments!)

 Latency       → Route to lowest-latency region

 Failover      → Primary/secondary with health checks

 Geolocation   → Route based on user's country/region

 Geoproximity  → Route based on geographic distance

 Multivalue    → Return up to 8 healthy IPs (simple load balancing)



 Health Checks:

 Route53 can health-check endpoints every 10 or 30 seconds

 If endpoint fails health check → removed from DNS responses

 Combines with Failover routing for automatic disaster recovery



 Example: Multi-region failover

 api.example.com → Failover routing

   Primary: us-east-1 ALB (health check: /health on port 443)

   Secondary: eu-west-1 ALB (health check: /health on port 443)

 If us-east-1 fails health check → Route53 automatically

 returns eu-west-1 ALB IP → traffic fails over to EU



 Split-horizon DNS (Private + Public):

 Same domain name resolves differently inside vs outside VPC

 Public:  api.example.com → 54.x.x.x (public ALB IP)

 Private: api.example.com → 10.0.1.50 (internal ALB IP)

 Internal services talk to each other via private IPs

 External users hit the public endpoint

 Both use the same domain name — cleaner configuration

```



\### Production Scenarios:



\#### Scenario 1: "DNS Resolution is Slow" in Kubernetes



```bash

 Symptoms:

 Application response times increased by 200ms

 All external API calls are slow

 Internal service-to-service calls are fine

 CoreDNS pods CPU usage is 90%



 Investigation:

kubectl top pods -n kube-system -l k8s-app=kube-dns

 NAME          CPU    MEMORY

 coredns-xxx   980m   256Mi   ← CPU maxed out

 coredns-yyy   950m   248Mi



 Check CoreDNS logs:

kubectl logs -n kube-system coredns-xxx | tail -50

 Massive volume of queries



 Root cause: ndots:5

 Every external domain query generates 4 wasted lookups

 500 pods × 100 external calls/min × 4 extra queries = 200,000 wasted queries/min



 Fix:

 1. Deploy NodeLocal DNSCache

 2. Set ndots:2 on pods making heavy external calls

 3. Scale CoreDNS (add more replicas)

 4. Add trailing dots to external domain configs



 CoreDNS autoscaling:

 Deploy dns-autoscaler in kube-system

 Scales CoreDNS replicas based on node/core count

```



\#### Scenario 2: "Site is down" After DNS Migration



```bash

 Symptoms:

 Migrated DNS from GoDaddy to Route53

 Updated NS records at the registrar

 Some users can reach the site, some can't

 Has been 6 hours since the change



 What's happening:

 NS record changes propagate SLOWLY

 TTL on NS records is often 48 hours

 Some DNS resolvers have cached the OLD NS records

 They're still asking GoDaddy's nameservers (which no longer have your records)

 → NXDOMAIN → site appears down



 Debug:

dig example.com NS +trace

 See which nameservers are being returned at each level

 If TLD servers return old NS → propagation hasn't reached there yet



dig @8.8.8.8 example.com        # Google's resolver

dig @1.1.1.1 example.com        # Cloudflare's resolver  

dig @ns1.route53.aws.com example.com  # Your new authoritative server

 Compare results — if Route53 returns correct IP but Google doesn't,

 Google is still using cached old NS records



 Mitigation:

 1. BEFORE migration: lower TTL on all records to 60 seconds

 2. Wait for old TTL to expire

 3. THEN change NS records

 4. Keep old DNS provider active for 48-72 hours

 5. Verify propagation with multiple resolvers



 Check global propagation:

 https://www.whatsmydns.net/

 Shows DNS resolution from 20+ global locations

```



\#### Scenario 3: Java Microservice Caches DNS Forever



```bash

 Symptoms:

 Database failover completed successfully

 All services except the Java order-service still connect to OLD database IP

 Order-service restart fixes it



 Root cause:

 Java's InetAddress caches DNS FOREVER by default (TTL = -1)

 The service resolved the DB hostname at startup

 Database failed over → DNS updated → new IP

 Java never re-resolved → keeps connecting to dead IP



 Fix:

 In java.security file:

networkaddress.cache.ttl=60          # Cache for 60 seconds

networkaddress.cache.negative.ttl=10 # Cache NXDOMAIN for 10 seconds



 Or via JVM flag:

java -Dsun.net.inetaddress.ttl=60 -jar app.jar



 Or in code:

java.security.Security.setProperty("networkaddress.cache.ttl", "60");



 For Spring Boot with AWS RDS failover:

 Use the RDS Proxy or Aurora endpoint

 These handle failover at the connection level

 But STILL set the TTL — defense in depth

```



\#### Scenario 4: DNS-Based Service Discovery Failure



```bash

 Symptoms:

 Kubernetes headless service used for StatefulSet (Kafka)

 Kafka brokers can't find each other

 "UnknownHostException: kafka-0.kafka-headless.default.svc.cluster.local"



 Investigation:

kubectl get svc kafka-headless

 NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)

 kafka-headless    ClusterIP   None         <none>        9092/TCP

                               ^^^^ Good — headless



kubectl get endpoints kafka-headless

 NAME              ENDPOINTS

 kafka-headless    <none>

 ^^^ EMPTY! No endpoints!



 Same problem we saw in Phase 0:

 Service selector labels don't match pod labels

kubectl describe svc kafka-headless | grep Selector

 Selector: app=kafka



kubectl get pods --show-labels | grep kafka

 kafka-0   Running   app=kafka-broker   ← MISMATCH!



 Fix:

kubectl edit svc kafka-headless

 Change selector to: app=kafka-broker

 Or fix the StatefulSet labels



 After fix:

kubectl get endpoints kafka-headless

 kafka-headless   172.16.0.5:9092,172.16.0.6:9092,172.16.0.7:9092



dig kafka-0.kafka-headless.default.svc.cluster.local

 172.16.0.5 ← Individual pod resolved!

```



\---



\### /etc/resolv.conf and /etc/nsswitch.conf — The Resolution Chain



```bash

 /etc/resolv.conf — DNS client configuration

nameserver 10.0.0.2       # Primary DNS server

nameserver 8.8.8.8         # Fallback DNS server

search example.com         # Default search domain

options timeout:2 attempts:3 rotate

 timeout = seconds before retry

 attempts = number of retries

 rotate = round-robin between nameservers (load balancing)



 /etc/nsswitch.conf — Resolution ORDER

 This file controls what Linux checks FIRST

hosts: files dns myhostname

 files = /etc/hosts (checked FIRST)

 dns = /etc/resolv.conf (checked SECOND)

 myhostname = systemd fallback



 Why this matters:

 If someone puts a wrong entry in /etc/hosts,

 it OVERRIDES DNS completely

 "But DNS is correct!" — doesn't matter, /etc/hosts wins

 Always check /etc/hosts first when debugging DNS issues



 In containers:

 Kubernetes manages /etc/resolv.conf via dnsPolicy:

 dnsPolicy: ClusterFirst (default) — use CoreDNS

 dnsPolicy: Default — use node's DNS

 dnsPolicy: None — you define everything manually via dnsConfig

```





\# 📋 LESSONS 1-3 QUICK REFERENCE — Networking Foundations



```

OSI MODEL (bottom to top):

 Physical → Data Link → Network → Transport → Session → Presentation → Application

 Troubleshoot bottom-up: cable → ARP → ping → port → HTTP



TCP/IP MODEL:

 Network Access → Internet → Transport → Application



ENCAPSULATION:

 Data → \\\[TCP Header + Data] → \\\[IP Header + TCP + Data] → \\\[Ethernet + IP + TCP + Data]

 Ethernet frame changes at each hop, IP header stays end-to-end



MTU:

 Standard: 1500 bytes, AWS VPC: 9001, VPN tunnels: \\\~1400

 Debug: ping -M do -s 1472 <dest> (reduce until works)

 Fix: tcp \_mtu \_probing=1 or MSS clamping



IP ADDRESSING:

 Private: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16

 CIDR: /prefix = network bits, remaining = host bits

 IPs = 2^(32-prefix), AWS reserves 5 per subnet



K8S CIDRS (must not overlap):

 Node network, Pod network, Service network

 AWS VPC CNI: pods get real VPC IPs (IP exhaustion risk)



DNS RESOLUTION ORDER:

 Local cache → /etc/hosts → /etc/resolv.conf → Recursive resolver →

 Root → TLD → Authoritative → Cache answer for TTL



DNS RECORDS:

 A=IPv4, AAAA=IPv6, CNAME=alias, ALIAS=apex alias,

 MX=mail, TXT=verification/SPF, NS=nameservers,

 SRV=service discovery, PTR=reverse lookup



K8S DNS:

 <svc>.<ns>.svc.cluster.local → ClusterIP

 <pod>.<svc>.<ns>.svc.cluster.local → StatefulSet pod

 Headless (ClusterIP:None) → returns individual pod IPs

 ndots:5 causes 4 extra queries per external lookup → fix with ndots:2 or trailing dot



DNS DEBUGGING:

 dig <domain> +short / +trace / @<server>

 nslookup <domain>

 In K8s: kubectl run debug --rm -it --image=netshoot -- dig <domain>



ROUTE53:

 Simple, Weighted, Latency, Failover, Geo routing

 Alias records for apex domains (no custom TTL)

 Health checks for automatic failover

 Split-horizon: same domain, different IPs inside vs outside VPC



GOTCHAS:

 Java caches DNS forever → set networkaddress.cache.ttl=60

 NS record changes take up to 48h → lower TTL before migration

 /etc/hosts overrides DNS → check nsswitch.conf order

 CoreDNS overload from ndots:5 → deploy NodeLocal DNSCache

 Empty endpoints = label selector mismatch (ALWAYS)

```



\---



\# 📝 Retention Questions — Networking Lessons 1-3



\*\*Q1:\*\* A developer says external API calls from their Kubernetes pod take 200ms longer than expected. You check CoreDNS and it's at 95% CPU. Based on what you learned, what's the most likely cause, and give me three different ways to fix it.



\*\*Q2:\*\* After a database failover, all services recover within 60 seconds except one Java service that keeps connecting to the old IP until it's restarted. What's happening and how do you fix it permanently?



\*\*Q3:\*\* You're designing an AWS VPC for NovaMart. You need: 3 AZs, public subnets, private app subnets, private database subnets, and room for an EKS cluster with 200 pods per node. What VPC CIDR would you choose and why? How would you handle EKS pod IP exhaustion with AWS VPC CNI?



\*\*Q4:\*\* A user reports your website is down. You run `dig example.com` and get `NXDOMAIN`. But `dig @ns1.route53.aws.com example.com` returns the correct IP. What layer of the DNS chain is broken, and what's your troubleshooting sequence?



\*\*Go.\*\* 🎯

====================================================================================



#### \# 🌐 PHASE 1 — NETWORKING



##### \## Lesson 4: TCP Deep Dive — The Backbone of Reliable Communication



TCP (Transmission Control Protocol) is the transport layer protocol that powers HTTP, HTTPS, SSH, database connections, gRPC — virtually everything in your production stack. Understanding TCP at a deep level is non-negotiable.



\### TCP vs UDP — When and Why



```

TCP (Transmission Control Protocol):

 ✅ Reliable — guarantees delivery

 ✅ Ordered — packets arrive in sequence

 ✅ Connection-oriented — handshake before data

 ✅ Flow control — sender doesn't overwhelm receiver

 ✅ Congestion control — adapts to network capacity

 ❌ Slower — overhead from all the above

 

 Used for: HTTP/S, SSH, databases, email, file transfer,

         gRPC, WebSockets — anything requiring reliability



UDP (User Datagram Protocol):

 ✅ Fast — minimal overhead

 ✅ Connectionless — no handshake, just send

 ❌ Unreliable — no delivery guarantee

 ❌ Unordered — packets can arrive out of sequence

 ❌ No flow/congestion control

 

 Used for: DNS queries, video streaming, gaming, VoIP,

         metrics collection (StatsD), health checks,

         QUIC/HTTP3, NTP, SNMP, container runtime metrics

```



\*\*Why this matters for DevOps:\*\*

\- DNS uses UDP port 53 (but falls back to TCP for large responses >512 bytes)

\- Prometheus scraping uses TCP (HTTP)

\- StatsD metrics use UDP (fire and forget)

\- Health check probes can be TCP or HTTP

\- Load balancers operate differently on TCP vs UDP

\- Network Policies in K8s must specify protocol (TCP/UDP)



\---



\### The TCP 3-Way Handshake — Connection Establishment



Every TCP connection begins with this:



```

CLIENT                          SERVER

 |                                |

 |  ──── SYN (seq=100) ────►     |  Step 1: Client says "I want to connect"

 |                                |           Client moves to SYN\_SENT state

 |                                |

 |  ◄── SYN-ACK (seq=300,        |  Step 2: Server says "OK, I acknowledge"

 |        ack=101) ────           |           Server moves to SYN\_RECV state

 |                                |

 |  ──── ACK (seq=101,           |  Step 3: Client says "Confirmed"

 |        ack=301) ────►          |           Both move to ESTABLISHED state

 |                                |

 |  ═══ CONNECTION ESTABLISHED ═══|

 |        Data can now flow       |

```



\*\*Why seq and ack numbers matter:\*\*

\- `seq=100` → "My starting sequence number is 100"

\- `ack=101` → "I've received everything up to 100, send me 101 next"

\- These numbers track EVERY byte sent and received

\- If a packet is lost, the receiver can tell sender exactly which bytes to resend



\*\*What can go wrong during handshake:\*\*



```bash

 SYN sent but no SYN-ACK received:

 → Firewall/Security Group blocking the port

 → Server not listening on that port

 → Network route doesn't exist

 Symptom: "connection timed out" (not "connection refused")

 The client retries SYN based on tcp\_syn\_retries (default 6 = \\\~127 seconds!)



 SYN received but SYN-ACK never reaches client:

 → Asymmetric routing (return path is different)

 → Stateful firewall only saw outbound, not return

 → NAT translation issue



 SYN-ACK received but ACK dropped:

 → Connection stuck in SYN\_RECV on server

 → SYN flood attack fills SYN\_RECV queue

 → This is why tcp\_syncookies exists (covered in sysctl lesson)

```



\### Observing Connection States:



```bash

 View all TCP connections and their states

ss -tan

 State      Recv-Q Send-Q  Local Address:Port   Peer Address:Port

 LISTEN     0      128     \\\*:80                  \\\*:\\\*

 ESTAB      0      0       10.0.1.5:80           203.0.113.1:45678

 TIME-WAIT  0      0       10.0.1.5:80           203.0.113.2:45679

 SYN-RECV   0      0       10.0.1.5:80           198.51.100.1:12345



 Count connections by state

ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn

 15234 ESTAB

 4521  TIME-WAIT

 128   LISTEN

 3     SYN-RECV



 Watch states in real time

watch -n 1 'ss -tan | awk "{print \\\\$1}" | sort | uniq -c | sort -rn'

```



\---



\### The TCP 4-Way Teardown — Connection Termination



```

CLIENT                          SERVER

 |                                |

 |  ──── FIN (seq=500) ────►     |  Step 1: Client says "I'm done sending"

 |                                |           Client → FIN\_WAIT\_1

 |                                |

 |  ◄── ACK (ack=501) ────       |  Step 2: Server acknowledges

 |                                |           Client → FIN\_WAIT\_2

 |                                |           Server can still send data!

 |                                |

 |  ◄── FIN (seq=700) ────       |  Step 3: Server says "I'm done too"

 |                                |           Server → LAST\_ACK

 |                                |

 |  ──── ACK (ack=701) ────►     |  Step 4: Client confirms

 |                                |           Client → TIME\_WAIT (2×MSL)

 |                                |           Server → CLOSED

 |                                |

 |  ═══ CONNECTION CLOSED ═══     |

```



\*\*Why TIME\_WAIT exists:\*\*



TIME\_WAIT lasts for 2×MSL (Maximum Segment Lifetime), typically 60 seconds on Linux. It exists for two reasons:



```

1\\. DELAYED PACKETS: Old packets from this connection might still

  be in transit. If you immediately reuse the same source:dest

  port pair, those delayed packets could be confused with the

  new connection's packets → data corruption.



2\\. LOST FINAL ACK: If the final ACK (Step 4) is lost, the server

  retransmits its FIN. The client needs to be in TIME\_WAIT to

  respond with another ACK. Without TIME\_WAIT, the client sends

  RST → server gets a confusing error.

```



\*\*The TIME\_WAIT accumulation problem (revisited with full understanding):\*\*



```bash

 On a reverse proxy making many short-lived outbound connections:

ss -tan state time-wait | wc -l

 42000



 Each TIME\_WAIT socket holds:

 - A source port (from ephemeral range)

 - A 4-tuple: src\_ip:src\_port → dst\_ip:dst\_port

\\#

 You can only have ONE TIME\_WAIT per unique 4-tuple

 With one destination IP:port, you're limited by source ports

 \\\~28,000 default ephemeral ports, 60s TIME\_WAIT:

 Max new connections: 28000/60 = \\\~467/second to a SINGLE destination

\\#

 If you need more:

 1. tcp\_tw\_reuse=1 (reuse TIME\_WAIT for outbound, uses timestamps for safety)

 2. Expand ephemeral port range (1024-65535)

 3. CONNECTION POOLING (the real fix — reuse connections instead of creating new ones)

 4. HTTP Keep-Alive (reuse the same TCP connection for multiple HTTP requests)

```



\---



\### TCP All Connection States — Complete Reference



```

State           Meaning                                      Who's In It

─────────       ──────────────────────────────────────       ──────────

LISTEN          Waiting for incoming connections             Server

SYN\_SENT        SYN sent, waiting for SYN-ACK               Client

SYN\_RECV        SYN received, SYN-ACK sent, waiting ACK     Server

ESTABLISHED     Connection active, data flowing              Both

FIN\_WAIT\_1      FIN sent, waiting for ACK                   Closer (initiator)

FIN\_WAIT\_2      FIN acknowledged, waiting for peer's FIN    Closer

CLOSE\_WAIT      Received FIN, waiting for app to close      Receiver (DANGEROUS)

LAST\_ACK        FIN sent, waiting for final ACK             Receiver

TIME\_WAIT       Waiting 2×MSL before fully closing          Closer

CLOSING         Both sides sent FIN simultaneously (rare)   Both

CLOSED          Connection fully terminated                 N/A

```



\### CLOSE\_WAIT — The State That Tells You Your App Is Buggy



```bash

 CLOSE\_WAIT means:

 - The REMOTE side closed the connection (sent FIN)

 - Your application received the notification

 - Your application HAS NOT closed its side yet

 

 This is ALWAYS an application bug — your code isn't calling close()

 on the socket after the peer disconnects



ss -tan state close-wait | wc -l

 5000  ← THIS IS BAD



 Common causes:

 - Connection pool not closing dead connections

 - Application not handling IOException when reading from closed socket

 - Database driver not detecting connection termination

 - Thread stuck/blocked, never reaches the close() call



 If CLOSE\_WAIT sockets accumulate:

 - File descriptors exhausted (EMFILE)

 - Memory leaked (each socket has buffers)

 - Application eventually crashes



 CLOSE\_WAIT never times out on its own

 It stays until the application closes the socket or the process dies

 This is fundamentally different from TIME\_WAIT which auto-expires



 Debugging:

ss -tanp state close-wait

 Shows PID and process name of the offender

 Then look at the code — find where connections aren't being closed

```



\---



\### TCP Flow Control — Window Size



Flow control prevents the sender from overwhelming the receiver.



```

 Each side advertises a RECEIVE WINDOW:

 "I have X bytes of buffer space available"



 Sender can only have (window size) bytes in-flight (unacknowledged)



 Example:

 Receiver advertises window = 64KB

 Sender sends 64KB of data

 Sender STOPS sending until receiver ACKs some of it

 Receiver processes 32KB, ACKs it, advertises new window = 32KB

 Sender can now send 32KB more



 Window scaling (RFC 1323):

 Original TCP header only has 16 bits for window = max 64KB

 Window scaling multiplies by 2^scale\_factor

 Scale factor negotiated during handshake

 Max window with scaling: 64KB × 2^14 = 1GB

 This is why tcp\_window\_scaling=1 in sysctl is critical

 Without it: max 64KB window = terrible throughput on high-latency links



 Monitoring:

 In tcpdump/Wireshark, look for "win=" in TCP headers

 Shrinking windows = receiver falling behind

 Zero window = receiver buffer full, sender completely blocked

 Zero window probe: sender periodically asks "got space yet?"

```



\---



\### TCP Congestion Control — Don't Overwhelm the Network



Flow control protects the receiver. Congestion control protects the \*\*network\*\*.



```

 TCP dynamically adjusts sending rate based on packet loss/delay



 CONGESTION WINDOW (cwnd):

 An internal window that limits how much data can be in-flight

 Effective window = min(receiver window, congestion window)



 SLOW START:

 New connection starts with cwnd = initial window (\\\~10 segments)

 For each ACK received, cwnd doubles (exponential growth)

 1 → 2 → 4 → 8 → 16 → 32...

 This continues until:

   - Packet loss detected → cwnd cut in half

   - cwnd reaches ssthresh (slow start threshold)

   - Then switches to congestion avoidance (linear growth)



 This is why new connections start slow!

 And why tcp\_slow\_start\_after\_idle=0 matters (covered in sysctl)

 Without it, idle connections reset to slow start



 CONGESTION CONTROL ALGORITHMS:

 Linux supports multiple — each handles loss/delay differently:



 Cubic (default on most Linux):

   Standard algorithm, aggressive recovery

   Good for most workloads



 BBR (Bottleneck Bandwidth and RTT) — Google's algorithm:

   Doesn't rely on packet loss to detect congestion

   Measures bandwidth and RTT directly

   MUCH better performance on lossy or high-latency networks

   Used by YouTube, Google Cloud, many CDNs

   20-40% throughput improvement in real-world tests



 Check current algorithm:

sysctl net.ipv4.tcp\_congestion\_control

 cubic



 Switch to BBR:

sysctl -w net.core.default\_qdisc=fq

sysctl -w net.ipv4.tcp\_congestion\_control=bbr



 Available algorithms:

sysctl net.ipv4.tcp\_available\_congestion\_control

 reno cubic bbr



 When to use BBR:

 - High-latency connections (cross-region, international)

 - Lossy networks (WiFi, mobile, satellite)

 - File transfer services

 - CDN edge servers

 - Any service where throughput matters

```



\---



\### TCP Retransmission — How Lost Packets Are Recovered



```bash

 When a packet is lost, TCP detects it via:

 1. TIMEOUT — no ACK received within RTO (Retransmission Timeout)

 2. DUPLICATE ACKs — receiver keeps ACKing the last good packet

    Three duplicate ACKs = "Fast Retransmit" (don't wait for timeout)



 Check retransmission stats:

ss -ti

 Shows per-connection: rto, rtt, retrans count, cwnd, etc.



netstat -s | grep -i retrans

 1234 segments retransmitted

 If this number is climbing rapidly → packet loss → network issue



 Retransmission in tcpdump:

tcpdump -i eth0 'tcp\\\[tcpflags] \\\& (tcp-syn|tcp-fin) != 0'

 Or use Wireshark's "tcp.analysis.retransmission" filter



 High retransmissions usually mean:

 - Network congestion (switch buffers full, packets dropped)

 - Faulty NIC/cable (check ifconfig for errors)

 - MTU mismatch causing fragmentation failures

 - Firewall dropping packets (stateful inspection timeout)

 - Application too slow to read from socket → receive buffer full → drops

```



\---



\### RST (Reset) — The Angry Packet



```bash

 RST immediately terminates a connection. No graceful shutdown.

 Causes:



 1. Connection to a port nobody is listening on

    SYN → RST (this is "Connection refused")



 2. Firewall rejects the connection (REJECT rule, not DROP)

    SYN → RST



 3. Application crashes while connection is open

    Kernel sends RST to the peer



 4. Receive buffer overflows and application can't keep up

    Kernel may RST to protect resources



 5. Half-open connection (one side crashed, other doesn't know)

    Surviving side sends data → dead side's kernel sends RST

    "Connection reset by peer"



 6. Load balancer idle timeout expired

    ALB idle timeout: 60s (default)

    Connection idle > 60s → ALB drops it silently

    Client sends data → ALB has no state → sends RST

    "Connection reset by peer" in application logs



 THE ALB TIMEOUT PROBLEM (extremely common in production):

 Client ←→ ALB ←→ Backend

 Client opens connection, sends request, gets response

 Connection is idle for 61 seconds (ALB timeout = 60s)

 ALB silently drops the connection state

 Client sends another request on the same connection

 ALB: "I don't know this connection" → RST

 Client: "Connection reset by peer"

\\#

 Fix:

 1. Set ALB idle timeout higher (up to 4000 seconds)

 2. Set application keepalive LOWER than ALB timeout

    tcp\_keepalive\_time=30 < ALB timeout=60

 3. Application-level heartbeats/pings on the connection

```



\---



##### \## Lesson 5: HTTP, HTTPS \& TLS — The Application Layer



\### HTTP — The Protocol of the Web



```

HTTP Request:

┌──────────────────────────────┐

│ GET /api/users HTTP/1.1      │ ← Request line (method, path, version)

│ Host: api.example.com        │ ← Required header

│ Authorization: Bearer abc123 │ ← Auth token

│ Content-Type: application/json│

│ Accept: application/json     │

│ Connection: keep-alive       │

│                              │ ← Empty line = end of headers

│ {"name": "test"}             │ ← Request body (for POST/PUT)

└──────────────────────────────┘



HTTP Response:

┌──────────────────────────────┐

│ HTTP/1.1 200 OK              │ ← Status line

│ Content-Type: application/json│

│ Content-Length: 45            │

│ Cache-Control: no-cache      │

│ X-Request-Id: abc-123        │ ← Tracing header

│                              │

│ {"id": 1, "name": "test"}   │ ← Response body

└──────────────────────────────┘

```



\### HTTP Methods:



```

GET     → Read/retrieve a resource         (idempotent, safe)

POST    → Create a new resource            (NOT idempotent)

PUT     → Replace a resource entirely      (idempotent)

PATCH   → Partially update a resource      (may or may not be idempotent)

DELETE  → Remove a resource                (idempotent)

HEAD    → Same as GET but no body          (health checks use this)

OPTIONS → What methods are allowed?        (CORS preflight requests)

```



\### HTTP Status Codes — What They Actually Mean in Production:



```

1xx: INFORMATIONAL

 100 Continue    → "Send me the body" (large upload pre-check)

 101 Switching   → Upgrading to WebSocket



2xx: SUCCESS

 200 OK          → Standard success

 201 Created     → Resource created (POST response)

 202 Accepted    → Request accepted but processing later (async APIs)

 204 No Content  → Success but no body (DELETE response)



3xx: REDIRECTION

 301 Moved Permanently → URL changed forever (SEO: search engines update)

 302 Found             → Temporary redirect (login redirects)

 304 Not Modified      → Use your cached version (conditional GET)

 307 Temporary Redirect → Like 302 but preserves HTTP method

 308 Permanent Redirect → Like 301 but preserves HTTP method



4xx: CLIENT ERROR (the requester screwed up)

 400 Bad Request       → Malformed request (invalid JSON, missing field)

 401 Unauthorized      → Not authenticated (no token, expired token)

 403 Forbidden         → Authenticated but not authorized (no permission)

 404 Not Found         → Resource doesn't exist

 405 Method Not Allowed → GET on a POST-only endpoint

 408 Request Timeout   → Client took too long to send request

 409 Conflict          → Resource conflict (duplicate create, version mismatch)

 413 Payload Too Large → Request body exceeds server limit

 422 Unprocessable     → Valid JSON but failed validation

 429 Too Many Requests → Rate limited! Back off. Check Retry-After header



5xx: SERVER ERROR (the server screwed up)

 500 Internal Server Error → Unhandled exception (bug)

 502 Bad Gateway           → Proxy received invalid response from upstream

 503 Service Unavailable   → Server overloaded or in maintenance

 504 Gateway Timeout       → Proxy didn't get response from upstream in time



FOR DEVOPS, THE CRITICAL ONES:

 502 = upstream server crashed or returned garbage

 503 = upstream server is overloaded or not ready

 504 = upstream server is too slow (timeout)

 These three are 90% of your on-call alerts

```



\### HTTP/1.1 vs HTTP/2 vs HTTP/3:



```

HTTP/1.1:

 - One request per TCP connection (or pipelining, rarely used)

 - Head-of-line blocking (one slow response blocks others)

 - Text-based headers (repeated on every request, wasteful)

 - Workaround: browsers open 6 parallel connections per domain



HTTP/2:

 - Multiplexing: multiple requests over SINGLE TCP connection

 - Header compression (HPACK)

 - Server push (server sends resources before client asks)

 - Binary protocol (more efficient than text)

 - Stream prioritization

 - Used by: most modern websites, gRPC (built on HTTP/2)



HTTP/3:

 - Built on QUIC (which is built on UDP, not TCP)

 - Eliminates TCP head-of-line blocking

 - Built-in encryption (TLS 1.3 mandatory)

 - Faster connection establishment (0-RTT possible)

 - Better for mobile (survives network switches WiFi→cellular)

 - Used by: Google, Cloudflare, Facebook



FOR DEVOPS:

 - Configure Nginx/ALB to support HTTP/2 (better performance)

 - gRPC requires HTTP/2 (won't work through HTTP/1.1 proxies)

 - If gRPC calls fail through ALB, check HTTP/2 support

 - HTTP/3 requires UDP port 443 open (not just TCP 443)

```



\---



\### TLS — Transport Layer Security (HTTPS)



TLS encrypts the connection between client and server. HTTPS = HTTP over TLS.



\### The TLS Handshake (TLS 1.3 — current standard):



```

CLIENT                              SERVER

 |                                    |

 | ── Client Hello ──────────►        |  Supported cipher suites,

 |    (TLS version, ciphers,          |  TLS version, random number,

 |     random, SNI)                   |  SNI = which domain you want

 |                                    |

 | ◄── Server Hello ────────          |  Chosen cipher suite,

 |    (chosen cipher, cert,           |  server's certificate,

 |     key share)                     |  key exchange material

 |                                    |

 | Client verifies certificate:       |

 |   1. Valid CA signature?           |

 |   2. Not expired?                  |

 |   3. Domain matches SNI?           |

 |   4. Not revoked (OCSP/CRL)?      |

 |                                    |

 | ── Client Finished ──────►         |  Both derive session keys

 |    (key share, encrypted)          |  from shared secret

 |                                    |

 | ◄── Server Finished ────          |

 |                                    |

 | ════ ENCRYPTED DATA FLOW ════      |

 |                                    |



TLS 1.3 completes in 1 RTT (vs 2 RTT for TLS 1.2)

0-RTT resumption possible for returning clients (even faster)

```



\### SNI — Server Name Indication



```bash

 SNI is how the client tells the server WHICH domain it wants

 during TLS handshake (before encryption is established)

 

 Why it matters:

 One server (one IP) can host MULTIPLE HTTPS sites

 Without SNI, the server doesn't know which certificate to present

 

 ALB uses SNI to route HTTPS traffic to different target groups

 based on the domain in the TLS Client Hello

\\#

 NLB with TLS: must be configured for SNI if multiple certs

\\#

 Encrypted SNI (ESNI/ECH):

 SNI is sent in PLAINTEXT in the Client Hello

 This means anyone watching the network can see which domains you visit

 Even with HTTPS, the domain name is visible

 Encrypted Client Hello (ECH) fixes this — encrypts the SNI

 Supported by Cloudflare and Firefox

```



\### Certificate Types and Management:



```bash

 Certificate chain:

 Root CA → Intermediate CA → Server Certificate

 Your server presents: Server cert + Intermediate cert

 Client trusts: Root CA (pre-installed in OS/browser)

 Client validates the chain: Server → Intermediate → Root



 Certificate formats:

 PEM (.pem, .crt, .cer) — Base64 encoded, text format

 DER (.der, .cer) — Binary format

 PKCS#12 (.p12, .pfx) — Binary, contains cert + private key (password protected)



 Key files:

 Private key (.key) — NEVER share, NEVER commit to git

 Public key — embedded in the certificate

 CSR (.csr) — Certificate Signing Request (sent to CA)



 AWS Certificate Manager (ACM):

 - Free SSL/TLS certificates for AWS services

 - Auto-renewal (no more expired cert outages!)

 - Used with ALB, NLB, CloudFront, API Gateway

 - CANNOT be exported (private key stays in AWS)

 - Cannot be used on EC2 directly (use Let's Encrypt for that)



 Let's Encrypt + Certbot:

 - Free certificates for any server

 - Auto-renewal via certbot timer

 - Used on Nginx, Apache, any server you manage directly

certbot certonly --nginx -d example.com -d www.example.com

 Creates: /etc/letsencrypt/live/example.com/

   fullchain.pem (cert + intermediate)

   privkey.pem (private key)

   chain.pem (intermediate only)

   cert.pem (server cert only)



 In Kubernetes:

 cert-manager — automatic TLS certificate management

 Integrates with Let's Encrypt, Vault, ACM

 Watches Ingress resources and auto-provisions certs

apiVersion: cert-manager.io/v1

kind: ClusterIssuer

metadata:

 name: letsencrypt-prod

spec:

 acme:

 server: https://acme-v02.api.letsencrypt.org/directory

 email: ops@example.com

 privateKeySecretRef:

   name: letsencrypt-prod

 solvers:

 - http01:

     ingress:

       class: nginx

```



\### Production Scenarios:



\#### Scenario 1: Certificate Expiry Outage



```bash

 THE #1 most preventable outage in production

 Certificate expires → HTTPS stops working → site is down

 Every major company has had this outage at least once



 How to check certificate expiry:

echo | openssl s\_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

 notBefore=Jan  1 00:00:00 2024 GMT

 notAfter=Apr  1 00:00:00 2024 GMT



 Check from command line:

curl -vI https://example.com 2>\\\&1 | grep "expire"



 Monitoring:

 Prometheus blackbox\_exporter probes HTTPS endpoints:

 probe\_ssl\_earliest\_cert\_expiry — timestamp of cert expiry

 Alert when: (probe\_ssl\_earliest\_cert\_expiry - time()) < 30\\\*86400

 "Certificate expires in less than 30 days"



 Prevention:

 1. Use ACM for AWS resources (auto-renews)

 2. Use cert-manager in Kubernetes (auto-renews)

 3. Use certbot with auto-renewal timer

 4. Monitor ALL certificates with Prometheus/blackbox\_exporter

 5. Alert at 30 days, 14 days, 7 days, 3 days, 1 day

```



\#### Scenario 2: TLS Version Mismatch



```bash

 Old client connects with TLS 1.0

 Server only accepts TLS 1.2+

 Connection fails with cryptic error



 Check what TLS versions server supports:

nmap --script ssl-enum-ciphers -p 443 example.com

 Or:

openssl s\_client -tls1\_2 -connect example.com:443

openssl s\_client -tls1\_3 -connect example.com:443



 Nginx TLS configuration:

ssl\_protocols TLSv1.2 TLSv1.3;

ssl\_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';

ssl\_prefer\_server\_ciphers on;



 ALB: Supports TLS 1.0-1.3 via security policies

 ELBSecurityPolicy-TLS13-1-2-2021-06 → TLS 1.2 and 1.3 only

 Apply via Terraform:

resource "aws\_lb\_listener" "https" {

 ssl\_policy = "ELBSecurityPolicy-TLS13-1-2-2021-06"

}



 PCI-DSS compliance requires TLS 1.2+ (no TLS 1.0 or 1.1)

```



\#### Scenario 3: "Connection Reset" After Load Balancer Idle Timeout



```bash

 Symptoms:

 Intermittent "Connection reset by peer" errors

 Only happens on connections that were idle for \\\~60 seconds

 More common during low-traffic periods (longer idle times)



 Root cause:

 ALB idle timeout = 60 seconds (default)

 Backend keep-alive timeout = 65 seconds

 Connection idle for 61 seconds → ALB drops it

 Backend doesn't know → tries to use dead connection → RST



 Fix — set backend timeout SHORTER than ALB:

 Nginx:

keepalive\_timeout 55;  # < ALB's 60s



 Or increase ALB timeout:

resource "aws\_lb" "main" {

 idle\_timeout = 120  # seconds

}



 Golden rule:

 Client timeout > LB timeout > Backend timeout

 Example: Client=120s > ALB=90s > Nginx=75s > App=60s

 Each layer should timeout BEFORE the layer in front of it

 This ensures clean error propagation instead of random RSTs

```



\---



##### \## Lesson 6: Load Balancing — Distributing Traffic



\### Layer 4 (TCP) vs Layer 7 (HTTP) Load Balancing



```

LAYER 4 (TCP/UDP) — "Dumb" but fast:

 Sees: Source IP, Dest IP, Source Port, Dest Port, Protocol

 Doesn't see: HTTP headers, URL paths, cookies, content

 Decision: Based on IP/port only

 Action: Forwards raw TCP connection to backend

 Speed: Very fast (minimal processing per packet)

 Use for: Database connections, gRPC, non-HTTP, raw TCP/UDP

 AWS: Network Load Balancer (NLB)



LAYER 7 (HTTP/HTTPS) — Smart but more overhead:

 Sees: Everything — URL, headers, cookies, body, method

 Decision: Based on path, host header, headers, cookies

 Action: Terminates HTTP, creates NEW connection to backend

 Speed: Slower (must parse HTTP)

 Use for: Web traffic, API routing, A/B testing, canary

 AWS: Application Load Balancer (ALB)

```



\### AWS Load Balancers — Complete Comparison:



```

ALB (Application Load Balancer):

 Layer: 7 (HTTP/HTTPS)

 Features:

 - Path-based routing (/api/\\\* → service A, /web/\\\* → service B)

 - Host-based routing (api.example.com vs www.example.com)

 - HTTP/2 and WebSocket support

 - Authentication (Cognito, OIDC)

 - WAF integration

 - Sticky sessions (cookies)

 - Request logging to S3

 - gRPC support

 Use for: Web apps, REST APIs, microservices

 Cost: Per hour + per LCU (request-based)

 Preserves client IP: Via X-Forwarded-For header

 Kubernetes: AWS Load Balancer Controller → Ingress



NLB (Network Load Balancer):

 Layer: 4 (TCP/UDP/TLS)

 Features:

 - Ultra-low latency (\\\~100μs vs ALB's \\\~400μs)

 - Static IP per AZ (or Elastic IP)

 - Millions of requests/second

 - TLS termination

 - Preserves source IP (no X-Forwarded-For needed)

 - Cross-zone load balancing (optional)

 - VPC PrivateLink support

 Use for: Non-HTTP (databases, gRPC, gaming), extreme performance

 Cost: Per hour + per NLCU (connection-based)

 Preserves client IP: YES, natively (proxy protocol or direct)

 Kubernetes: AWS Load Balancer Controller → Service type LoadBalancer



CLB (Classic Load Balancer) — LEGACY:

 Don't use for new deployments

 Replaced by ALB + NLB

```



\### Load Balancing Algorithms:



```

ROUND ROBIN:

 Request 1 → Server A

 Request 2 → Server B

 Request 3 → Server C

 Request 4 → Server A

 Simple, fair, doesn't consider server health/load

 Used by: Nginx (default), most L4 balancers



WEIGHTED ROUND ROBIN:

 Server A (weight=3): gets 3 out of 5 requests

 Server B (weight=2): gets 2 out of 5 requests

 Use for: canary deployments (90/10 split), different server capacities



LEAST CONNECTIONS:

 Send to the server with fewest active connections

 Better for long-lived connections or uneven request processing times

 Used by: HAProxy, Nginx (least\_conn)



IP HASH:

 Hash the client IP to select a server

 Same client always goes to same server (poor man's sticky sessions)

 Used by: Nginx (ip\_hash), when session affinity needed without cookies



RANDOM:

 Pick a random server

 Simple, surprisingly effective at scale

 Used by: Some internal service meshes



LEAST RESPONSE TIME:

 Send to the server with lowest response time

 Requires active health monitoring

 Used by: ALB (estimates via connection duration)



MAGLEV/CONSISTENT HASHING:

 Hash-ring based — minimal disruption when servers added/removed

 Used by: Google's Maglev, Envoy, Kubernetes IPVS mode

```



\### Health Checks — The Gatekeeper:



```bash

 Load balancers check backend health continuously

 Unhealthy backends are removed from rotation



 ALB Health Check Configuration:

resource "aws\_lb\_target\_group" "api" {

 health\_check {

 path                = "/health"

 port                = "8080"

 protocol            = "HTTP"

 healthy\_threshold   = 3    # Consecutive successes to mark healthy

 unhealthy\_threshold = 2    # Consecutive failures to mark unhealthy

 timeout             = 5    # Seconds to wait for response

 interval            = 10   # Seconds between checks

 matcher             = "200" # Expected HTTP status code

 }

}



 Best practices for health check endpoints:

 1. /health → lightweight, just returns 200 (liveness)

 2. /ready → checks dependencies (DB, cache, downstream services)

 3. Don't check ALL dependencies in /health

    If Redis is down but your service can operate degraded,

    /health should still return 200

    Otherwise one Redis failure cascades to ALL your pods/instances



 Common mistake:

 Health check path = "/" (homepage)

 Homepage makes database queries, renders templates

 Under load, homepage is slow → health check times out

 LB marks ALL backends unhealthy → 503 for everyone

 Cascading failure caused by health check!

\\#

 Fix: Health check endpoint should be MINIMAL

 Just return 200 OK with an empty body

```



\### Nginx as Reverse Proxy and Load Balancer:



```nginx

 /etc/nginx/nginx.conf



upstream api\_backends {

 # Load balancing algorithm

 least\_conn;

 

 # Backend servers

 server 10.0.1.10:8080 weight=3;

 server 10.0.1.11:8080 weight=3;

 server 10.0.1.12:8080 weight=1;  # Canary (less traffic)

 server 10.0.1.13:8080 backup;     # Only used if all others are down

 

 # Keep-alive connection pool to backends

 keepalive 64;

 keepalive\_timeout 60s;

 keepalive\_requests 1000;

}



server {

 listen 80;

 listen 443 ssl http2;

 server\_name api.example.com;

 

 ssl\_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;

 ssl\_certificate\_key /etc/letsencrypt/live/api.example.com/privkey.pem;

 ssl\_protocols TLSv1.2 TLSv1.3;

 

 # Proxy settings

 location /api/ {

     proxy\_pass http://api\_backends;

     proxy\_http\_version 1.1;

     proxy\_set\_header Connection "";  # Enable keepalive to upstream

     proxy\_set\_header Host $host;

     proxy\_set\_header X-Real-IP $remote\_addr;

     proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;

     proxy\_set\_header X-Forwarded-Proto $scheme;

     proxy\_set\_header X-Request-Id $request\_id;

     

     # Timeouts

     proxy\_connect\_timeout 5s;    # Time to establish connection to backend

     proxy\_send\_timeout 30s;      # Time to send request to backend

     proxy\_read\_timeout 60s;      # Time to receive response from backend

     

     # Buffering

     proxy\_buffering on;

     proxy\_buffer\_size 8k;

     proxy\_buffers 32 8k;

     

     # Retries

     proxy\_next\_upstream error timeout http\_502 http\_503;

     proxy\_next\_upstream\_tries 2;

     proxy\_next\_upstream\_timeout 10s;

 }

 

 # Health check endpoint (don't proxy this)

 location /health {

     access\_log off;

     return 200 'OK';

     add\_header Content-Type text/plain;

 }

}

```



\---



\### Production Scenario: The 502 Investigation Framework



```bash

 502 Bad Gateway is the most common production alert for DevOps

 It means: "I (the proxy/LB) tried to reach the backend, 

 but got garbage or nothing back"



 SYSTEMATIC DEBUGGING:



 1. WHERE is the 502 coming from?

 Is it from Cloudflare? ALB? Nginx? Envoy?

 Check response headers:

curl -vI https://api.example.com 2>\\\&1

 Look for: Server: cloudflare, Server: nginx, Via: headers



 2. Is the backend RUNNING?

 Check pods/instances:

kubectl get pods

systemctl status myapp



 3. Is the backend HEALTHY from LB's perspective?

 AWS: Check target group health in console/CLI

aws elbv2 describe-target-health --target-group-arn <arn>



 4. Can the LB REACH the backend?

 Security group rules: LB → Backend on correct port?

 NACLs: allowing traffic in both directions?

 Backend listening on the right interface? (0.0.0.0 vs 127.0.0.1)



 5. Is the backend RESPONDING in time?

 Check proxy timeout vs backend response time

 If backend takes 65s and proxy timeout is 60s → 504 (not 502)

 But if backend crashes mid-response → 502



 6. Check backend logs at the time of 502:

 - Application exceptions

 - OOM kills

 - Connection pool exhaustion

 - Database connection failures



 COMMON 502 CAUSES:

 a) Backend crashed (OOMKilled, segfault, unhandled exception)

 b) Backend returned malformed HTTP (incomplete headers)

 c) Backend closed connection before sending response

 d) Security group not allowing LB → Backend traffic

 e) Backend listening on wrong port or only on localhost

 f) Health check passing but application endpoint failing

 g) During deployment: old pod terminated, new pod not ready yet

    → This is the preStop hook race condition we covered!

```



\# 📋 LESSONS 4-6 QUICK REFERENCE



```

TCP HANDSHAKE: SYN → SYN-ACK → ACK (3-way)

TCP TEARDOWN: FIN → ACK → FIN → ACK (4-way)

TIME\_WAIT: 60s on Linux, exists to prevent delayed packet confusion



TCP STATES TO WATCH:

 CLOSE\_WAIT = YOUR app isn't closing sockets (BUG)

 TIME\_WAIT = normal, but accumulation = port exhaustion

 SYN\_RECV = normal, but thousands = possible SYN flood



CONGESTION CONTROL:

 Cubic (default), BBR (Google, 20-40% better on lossy networks)

 Switch: sysctl net.ipv4.tcp\_congestion\_control=bbr



RST CAUSES:

 Port not listening, firewall REJECT, app crash,

 LB idle timeout (THE most common production cause)



HTTP STATUS CODES (DevOps critical):

 502 = backend crashed or returned garbage

 503 = backend overloaded or not ready

 504 = backend too slow (timeout)



TLS:

 TLS 1.3 = 1 RTT handshake, 0-RTT resumption

 SNI = domain in Client Hello (plaintext!)

 Certificate chain: Root CA → Intermediate → Server cert

 ACM (AWS), cert-manager (K8s), Certbot (servers)

 Monitor: probe\_ssl\_earliest\_cert\_expiry, alert at 30 days



LOAD BALANCING:

 L4 (NLB): TCP/UDP, ultra-fast, preserves source IP

 L7 (ALB): HTTP/HTTPS, path/host routing, WAF, auth

 

 Timeout hierarchy: Client > LB > Backend > App

 Health checks: minimal endpoint, NOT homepage





 REQUEST/RESPONSE TIMEOUTS (who gives up first):

 Client timeout > LB timeout > Backend timeout

 "Each layer should timeout BEFORE the one in front of it"

 So the backend gives up first → clean error to LB → clean error to client



 IDLE/KEEPALIVE TIMEOUTS (who closes idle connections first):

 Backend keepalive < LB idle timeout < Client keepalive

 "Backend closes idle connections BEFORE the LB silently drops them"

 This prevents the RST problem

 

 502 debugging: LB logs → backend status → security groups →

 backend logs → timeout comparison



NGINX:

 upstream + least\_conn + keepalive pool

 proxy\_next\_upstream for automatic retry on 502/503

 keepalive\_timeout < ALB idle timeout

```



\---



\# 📝 Retention Questions — Lessons 4-6



\*\*Q1:\*\* You see 5,000 connections in `CLOSE\_WAIT` state on your application server. Is this a kernel issue or an application issue? What exactly is happening and what do you do?



\*\*Q2:\*\* Your service experiences intermittent "Connection reset by peer" errors, but only during low-traffic periods (evenings/weekends). During peak hours, everything is fine. What's the most likely cause and how do you fix it with the timeout hierarchy principle?



\*\*Q3:\*\* During a rolling deployment on EKS, users experience 502 errors for approximately 5 seconds per pod replacement. All new pods pass health checks and come up healthy. Based on everything you've learned across ALL lessons so far, explain the full chain of events causing this and give the complete fix.



\*\*Q4:\*\* Your company uses Nginx as a reverse proxy in front of 10 backend servers. During peak traffic, Nginx starts returning 502 errors even though all backends are running. `ss -tan | grep ESTAB | wc -l` on Nginx shows 50,000 connections. What's happening and what TWO things need to be tuned?



\*\*Go.\*\* 🎯

=======================================================================================================

\# 💯 PERFECT SWEEP. 4/4.



Every answer was precise, well-structured, and demonstrated deep understanding across multiple lessons. You're connecting concepts from Linux (file descriptors, signals, sysctl) with networking (TCP states, timeouts, load balancing) naturally. That's systems thinking.



One nuance on Q2 — the timeout hierarchy has two dimensions:



```

REQUEST/RESPONSE TIMEOUTS (who gives up first):

&#x20; Client timeout > LB timeout > Backend timeout

&#x20; "Each layer should timeout BEFORE the one in front of it"

&#x20; So the backend gives up first → clean error to LB → clean error to client



IDLE/KEEPALIVE TIMEOUTS (who closes idle connections first):

&#x20; Backend keepalive < LB idle timeout < Client keepalive

&#x20; "Backend closes idle connections BEFORE the LB silently drops them"

&#x20; This prevents the RST problem you described

```



Your answer was correct in principle — you want the application to close before the middlebox kills silently. Just be precise about which timeout dimension you're discussing in interviews.



\---



\# 🌐 PHASE 1 — NETWORKING



##### \## Lesson 7: Firewalls, Security Groups, NACLs \& iptables



\### iptables — The Linux Firewall Engine



Everything in Linux network security ultimately flows through \*\*Netfilter\*\*, the kernel's packet filtering framework. `iptables` is the userspace tool to configure it. Even in cloud environments, Security Groups and NACLs are ultimately implemented via similar mechanisms.



\### iptables Architecture — Tables and Chains



```

TABLES (what kind of processing):

&#x20; filter   → Accept/drop/reject packets (DEFAULT table)

&#x20; nat      → Network Address Translation (SNAT, DNAT, masquerade)

&#x20; mangle   → Modify packet headers (TTL, TOS, marking)

&#x20; raw      → Bypass connection tracking (high-performance exceptions)

&#x20; security → SELinux/MAC rules



CHAINS (when in the packet's journey):

&#x20; PREROUTING  → Packet just arrived, before routing decision

&#x20; INPUT       → Packet destined for THIS machine

&#x20; FORWARD     → Packet passing THROUGH this machine (routing)

&#x20; OUTPUT      → Packet generated BY this machine

&#x20; POSTROUTING → Packet about to leave, after routing decision



PACKET FLOW:

&#x20; Incoming packet → PREROUTING → routing decision

&#x20;   → If for this host: INPUT → local process

&#x20;   → If for another host: FORWARD → POSTROUTING → out



&#x20; Outgoing packet → OUTPUT → routing decision → POSTROUTING → out

```



\### iptables Rule Syntax:



```bash

\# Basic format:

iptables -t <table> -A <chain> <match-criteria> -j <target>



\# TARGETS (actions):

\# ACCEPT  → Allow the packet

\# DROP    → Silently discard (no response — client times out)

\# REJECT  → Discard and send error back (client gets "connection refused")

\# LOG     → Log the packet and continue processing

\# DNAT    → Destination NAT (change destination IP/port)

\# SNAT    → Source NAT (change source IP)

\# MASQUERADE → Dynamic SNAT (for outbound internet via NAT gateway)



\# EXAMPLES:



\# Allow incoming SSH

iptables -A INPUT -p tcp --dport 22 -j ACCEPT



\# Allow incoming HTTP/HTTPS

iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT



\# Allow established/related connections (stateful)

iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT



\# Allow from specific CIDR

iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 8080 -j ACCEPT



\# Drop everything else (default deny)

iptables -A INPUT -j DROP



\# Block a specific IP

iptables -I INPUT 1 -s 198.51.100.5 -j DROP

\# -I INPUT 1 = Insert at position 1 (top of chain, processed first)



\# NAT — masquerade outbound traffic (for NAT gateway)

iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE



\# Port forwarding (DNAT)

iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.1.5:8080



\# View rules

iptables -L -n -v          # List all rules with packet counts

iptables -L -n -v --line-numbers  # With line numbers (for deletion)

iptables -t nat -L -n -v   # NAT table rules



\# Delete a rule

iptables -D INPUT 3         # Delete rule number 3 in INPUT chain



\# Flush all rules (CAREFUL!)

iptables -F                  # Flush filter table

iptables -t nat -F          # Flush nat table



\# Save rules (persists across reboot)

iptables-save > /etc/iptables/rules.v4

\# Restore

iptables-restore < /etc/iptables/rules.v4



\# On systemd systems, use iptables-persistent package:

apt install iptables-persistent

netfilter-persistent save

```



\### iptables Rule Processing — ORDER MATTERS:



```bash

\# Rules are processed TOP to BOTTOM

\# First match wins — remaining rules are SKIPPED



\# WRONG ORDER:

iptables -A INPUT -j DROP            # Rule 1: Drop everything

iptables -A INPUT -p tcp --dport 22 -j ACCEPT  # Rule 2: Allow SSH

\# SSH is blocked because Rule 1 matches first!



\# CORRECT ORDER:

iptables -A INPUT -p tcp --dport 22 -j ACCEPT  # Rule 1: Allow SSH

iptables -A INPUT -j DROP            # Rule 2: Drop everything else

\# SSH works because Rule 1 matches first



\# This is why -I (insert at top) is used for emergency blocks:

iptables -I INPUT 1 -s <attacker-ip> -j DROP

\# Inserted at position 1, before all other rules

```



\### Why iptables Matters for DevOps:



```bash

\# 1. Kubernetes uses iptables (or IPVS) for Service routing

\#    kube-proxy creates iptables rules for every Service

\#    Every ClusterIP, NodePort, LoadBalancer = iptables rules



\# View Kubernetes-created iptables rules:

iptables -t nat -L KUBE-SERVICES -n

\# Shows all Service → endpoint mappings



\# 2. Docker uses iptables for port mapping

\# docker run -p 8080:80 creates:

iptables -t nat -L DOCKER -n

\# DNAT rule: host:8080 → container-ip:80



\# 3. Calico (K8s CNI) uses iptables for NetworkPolicies

\# Every NetworkPolicy = iptables rules on the node



\# 4. Debugging connectivity issues often requires reading iptables

\# "Why can't pod A reach pod B?"

iptables -t filter -L -n -v | grep <pod-ip>

\# Check if any DROP/REJECT rules match

```



\### nftables — The Successor to iptables:



```bash

\# nftables is the modern replacement for iptables

\# Better performance, cleaner syntax, unified framework

\# Most modern distros include both, with iptables as a compatibility layer



\# Check if your system uses iptables or nftables backend:

iptables -V

\# iptables v1.8.7 (nf\_tables)  ← nftables backend

\# iptables v1.8.7 (legacy)     ← real iptables



\# For DevOps: understand both, but iptables knowledge is more universal

\# Kubernetes and Docker still primarily use iptables interface

```



\---



\### AWS Security Groups — Stateful Instance-Level Firewall



```bash

\# Security Groups (SGs) are attached to ENIs (Elastic Network Interfaces)

\# Every EC2 instance, RDS instance, Lambda in VPC, EKS pod gets one



\# KEY PROPERTIES:

\# ✅ STATEFUL — if inbound is allowed, return traffic is automatic

\# ✅ Allow rules ONLY — you cannot create deny rules

\# ✅ All rules evaluated (not ordered — if ANY rule allows, traffic passes)

\# ✅ Default: allow ALL outbound, deny ALL inbound

\# ✅ Can reference other Security Groups as source/destination



\# EXAMPLE — Web application architecture:



\# SG: alb-sg (for Application Load Balancer)

\# Inbound:

\#   TCP 443 from 0.0.0.0/0        (HTTPS from internet)

\#   TCP 80 from 0.0.0.0/0         (HTTP from internet — redirect to HTTPS)

\# Outbound:

\#   All traffic (default)



\# SG: app-sg (for application EC2/EKS)

\# Inbound:

\#   TCP 8080 from alb-sg           (only ALB can reach app port)

\#   TCP 22 from bastion-sg         (SSH only from bastion)

\# Outbound:

\#   All traffic



\# SG: db-sg (for RDS)

\# Inbound:

\#   TCP 5432 from app-sg           (only app can reach database)

\# Outbound:

\#   All traffic



\# SG: bastion-sg (for jump host)

\# Inbound:

\#   TCP 22 from <office-cidr>/32   (SSH only from office IP)

\# Outbound:

\#   All traffic



\# REFERENCING OTHER SGs:

\# "TCP 8080 from alb-sg" means:

\# "Allow TCP 8080 from any ENI that has alb-sg attached"

\# This is dynamic — if ALB scales, new IPs are automatically allowed

\# No need to update IP ranges manually

```



\### AWS NACLs — Stateless Subnet-Level Firewall



```bash

\# NACLs (Network Access Control Lists) operate at the SUBNET level

\# Every packet entering or leaving a subnet passes through the NACL



\# KEY PROPERTIES:

\# ❌ STATELESS — return traffic must be explicitly allowed

\# ✅ Allow AND deny rules

\# ✅ Rules processed IN ORDER by rule number (first match wins)

\# ✅ Default NACL: allows all traffic

\# ✅ Custom NACL: denies all traffic by default



\# NACL vs Security Group:



\# Feature          | Security Group      | NACL

\# Level            | Instance (ENI)      | Subnet

\# State            | Stateful            | Stateless

\# Rules            | Allow only          | Allow + Deny

\# Processing       | All rules evaluated | First match wins (ordered)

\# Default inbound  | Deny all            | Allow all (default NACL)

\# Return traffic   | Automatic           | Must explicitly allow



\# NACL RULE EXAMPLE:

\# Rule# | Type  | Protocol | Port     | Source        | Allow/Deny

\# 100   | HTTP  | TCP      | 80       | 0.0.0.0/0    | ALLOW

\# 110   | HTTPS | TCP      | 443      | 0.0.0.0/0    | ALLOW

\# 120   | SSH   | TCP      | 22       | 10.0.0.0/8   | ALLOW

\# 200   | Custom| TCP      | 1024-65535| 0.0.0.0/0   | ALLOW  ← EPHEMERAL PORTS!

\# \*     | All   | All      | All      | 0.0.0.0/0    | DENY   ← Default deny



\# THE EPHEMERAL PORT TRAP:

\# Because NACLs are STATELESS, you must allow return traffic

\# When your server responds to an HTTP request, the RESPONSE

\# goes to the CLIENT'S ephemeral port (1024-65535)

\# If your OUTBOUND NACL doesn't allow high ports → responses blocked

\# If your INBOUND NACL doesn't allow high ports → return traffic blocked

\# This catches people ALL THE TIME



\# BEST PRACTICE:

\# Use Security Groups as your PRIMARY firewall (easier, stateful)

\# Use NACLs as a COARSE secondary layer:

\#   - Block known bad CIDR ranges

\#   - Subnet-level isolation between tiers

\#   - Emergency: block an attacking IP at subnet level

\# Don't try to replicate SG rules in NACLs — it's painful and error-prone

```



\---



\### Production Scenario: Emergency IP Block During DDoS



```bash

\# Situation: Your WAF is being overwhelmed by traffic from a specific CIDR

\# You need to block it NOW, at the lowest level possible



\# Option 1: NACL (fastest, broadest)

\# Block at subnet level — packets never reach instances

aws ec2 create-network-acl-entry \\

&#x20; --network-acl-id acl-12345 \\

&#x20; --rule-number 50 \\

&#x20; --protocol -1 \\

&#x20; --rule-action deny \\

&#x20; --ingress \\

&#x20; --cidr-block 198.51.100.0/24



\# Option 2: Security Group (can't deny, so not useful here)

\# SGs only have allow rules — can't block specific IPs



\# Option 3: WAF rule

\# Block at Layer 7 — more granular but higher in the stack

aws wafv2 update-ip-set --name "blocked-ips" \\

&#x20; --addresses "198.51.100.0/24"



\# Option 4: Cloudflare/CDN level

\# Block before traffic even reaches AWS — most effective for DDoS



\# Order of defense (outermost to innermost):

\# Cloudflare → AWS Shield → WAF → NACL → Security Group → iptables

```



\---



##### \## Lesson 8: VPC Networking — The AWS Network Architecture



\### VPC Components — Complete Picture



```

VPC (Virtual Private Cloud):

&#x20; Your isolated network in AWS

&#x20; Defined by a CIDR block (e.g., 10.0.0.0/16)

&#x20; Spans ALL AZs in a region



SUBNETS:

&#x20; Segments of the VPC CIDR

&#x20; Each subnet lives in ONE AZ

&#x20; Public subnet: has route to Internet Gateway

&#x20; Private subnet: no direct internet access



INTERNET GATEWAY (IGW):

&#x20; Connects VPC to the internet

&#x20; 1:1 per VPC

&#x20; Allows instances with public IPs to reach/be reached from internet



NAT GATEWAY:

&#x20; Allows PRIVATE instances to reach the internet (outbound only)

&#x20; Lives in a PUBLIC subnet

&#x20; Has an Elastic IP

&#x20; Used for: package updates, API calls, pulling container images

&#x20; NO inbound connections from internet (one-way door)

&#x20; Cost: \~$32/month + data processing charges

&#x20; Best practice: one per AZ for high availability



ROUTE TABLES:

&#x20; Rules that determine where network traffic goes

&#x20; Each subnet is associated with ONE route table

&#x20; 

&#x20; # Public subnet route table:

&#x20; Destination       Target

&#x20; 10.0.0.0/16      local          (VPC internal traffic)

&#x20; 0.0.0.0/0        igw-12345      (internet via IGW)

&#x20; 

&#x20; # Private subnet route table:

&#x20; Destination       Target

&#x20; 10.0.0.0/16      local

&#x20; 0.0.0.0/0        nat-12345      (internet via NAT Gateway)



ELASTIC IP:

&#x20; Static public IPv4 address

&#x20; Persists across instance stop/start

&#x20; Attached to NAT Gateway or instance ENI

&#x20; Free while attached to running instance

&#x20; Charged when NOT in use ($3.65/month since Feb 2024)

```



\### VPC Peering vs Transit Gateway:



```

VPC PEERING:

&#x20; Direct connection between two VPCs

&#x20; Works across accounts and regions

&#x20; Non-transitive: A↔B and B↔C does NOT mean A↔C

&#x20; No bandwidth limit (uses AWS backbone)

&#x20; Free within same AZ, $0.01/GB cross-AZ

&#x20; Max: limited by number of routes in route table

&#x20; 

&#x20; Use for: Simple 2-3 VPC connections



&#x20; # Route table entry for peered VPC:

&#x20; Destination       Target

&#x20; 172.16.0.0/16    pcx-12345      (peering connection)



TRANSIT GATEWAY:

&#x20; Hub-and-spoke model — central router for multiple VPCs

&#x20; Transitive routing: A→TGW→B→TGW→C all works

&#x20; Supports: VPCs, VPN, Direct Connect, cross-region

&#x20; Scales to thousands of VPCs

&#x20; $0.05/hour + $0.02/GB

&#x20; 

&#x20; Use for: Enterprise with many VPCs, hybrid cloud

&#x20; 

&#x20; NovaMart architecture:

&#x20; ┌─────────┐    ┌──────────────┐    ┌─────────┐

&#x20; │ Prod VPC │────│Transit Gateway│────│ Dev VPC  │

&#x20; └─────────┘    └──────┬───────┘    └─────────┘

&#x20;                       │

&#x20;                ┌──────▼───────┐

&#x20;                │ Shared Svc   │

&#x20;                │ VPC (logging,│

&#x20;                │ monitoring)  │

&#x20;                └──────────────┘

```



\### VPN and Direct Connect:



```

SITE-TO-SITE VPN:

&#x20; Encrypted tunnel over public internet

&#x20; AWS VPC ←→ On-premises network

&#x20; Uses IPsec

&#x20; \~1.25 Gbps per tunnel (2 tunnels for HA)

&#x20; Cost: $0.05/hour per VPN connection

&#x20; Setup time: minutes

&#x20; Latency: variable (internet-dependent)



&#x20; Use for: Quick hybrid connectivity, backup to Direct Connect



AWS DIRECT CONNECT:

&#x20; Dedicated physical connection to AWS

&#x20; 1 Gbps or 10 Gbps ports

&#x20; Consistent latency, consistent bandwidth

&#x20; Cost: $0.03/GB + port hours

&#x20; Setup time: weeks to months (physical installation)

&#x20; 

&#x20; Use for: Large data transfers, latency-sensitive workloads,

&#x20;          compliance (traffic doesn't traverse public internet)



&#x20; NovaMart: Direct Connect primary, VPN as backup

```



\### VPC Endpoints — Private Access to AWS Services:



```bash

\# Without VPC Endpoint:

\# EC2 in private subnet → NAT Gateway → Internet → S3

\# Problem: Traffic goes over internet, costs NAT Gateway data fees



\# With VPC Endpoint:

\# EC2 in private subnet → VPC Endpoint → S3

\# Traffic stays within AWS network, no NAT Gateway needed



\# TWO TYPES:



\# Gateway Endpoint (free!):

\#   Supported: S3, DynamoDB ONLY

\#   Adds route to route table

\#   No security group needed

resource "aws\_vpc\_endpoint" "s3" {

&#x20; vpc\_id       = aws\_vpc.main.id

&#x20; service\_name = "com.amazonaws.us-east-1.s3"

&#x20; vpc\_endpoint\_type = "Gateway"

&#x20; route\_table\_ids = \[aws\_route\_table.private.id]

}



\# Interface Endpoint (costs money):

\#   Supported: Almost all other AWS services (ECR, SQS, SNS, CloudWatch, etc.)

\#   Creates an ENI in your subnet

\#   Accessible via private DNS

\#   $0.01/hour + $0.01/GB

resource "aws\_vpc\_endpoint" "ecr" {

&#x20; vpc\_id             = aws\_vpc.main.id

&#x20; service\_name       = "com.amazonaws.us-east-1.ecr.dkr"

&#x20; vpc\_endpoint\_type  = "Interface"

&#x20; subnet\_ids         = \[aws\_subnet.private\_a.id, aws\_subnet.private\_b.id]

&#x20; security\_group\_ids = \[aws\_security\_group.vpc\_endpoint.id]

&#x20; private\_dns\_enabled = true

}



\# CRITICAL FOR EKS:

\# EKS nodes in private subnets need to pull images from ECR

\# Without VPC endpoint: traffic goes through NAT Gateway = expensive

\# ECR pull for 200 pods × 500MB images = significant data costs

\# Interface endpoints for ECR, S3 (for ECR layers), CloudWatch, STS

\# save thousands of dollars per month on NAT data processing

```



\---



\### Production Scenario: EKS Nodes Can't Pull Images



```bash

\# Symptoms:

\# Pods stuck in ImagePullBackOff

\# "failed to pull image: timeout"

\# Only happens in private subnets



\# Investigation:

\# 1. Check if NAT Gateway exists and is healthy

aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=vpc-12345"

\# State: available ✓



\# 2. Check route table for private subnet

aws ec2 describe-route-tables --route-table-ids rtb-12345

\# 0.0.0.0/0 → nat-12345 ✓



\# 3. Check NAT Gateway's subnet has route to IGW

\# NAT GW must be in a PUBLIC subnet with IGW route

\# Common mistake: putting NAT GW in private subnet



\# 4. Check Security Groups

\# NAT Gateway doesn't have a SG, but the instances do

\# Instance SG outbound must allow HTTPS (443) to 0.0.0.0/0



\# 5. Check NACLs

\# Outbound: allow TCP 443 to 0.0.0.0/0

\# Inbound: allow TCP 1024-65535 from 0.0.0.0/0 (return traffic!)

\# ^^^ THE EPHEMERAL PORT TRAP AGAIN



\# 6. Alternative fix: VPC Endpoints

\# Deploy Interface Endpoints for ECR, S3, STS

\# Remove dependency on NAT Gateway for AWS service access

\# Faster, cheaper, more reliable

```



\---



##### \## Lesson 9: Network Debugging Tools — Your Investigation Toolkit



\### tcpdump — The Packet-Level X-Ray



```bash

\# tcpdump captures raw packets on a network interface

\# It's the most powerful network debugging tool



\# Basic capture:

tcpdump -i eth0                    # All traffic on eth0

tcpdump -i any                     # All interfaces



\# Filter by host:

tcpdump -i eth0 host 10.0.1.5     # Traffic to/from this IP

tcpdump -i eth0 src 10.0.1.5      # Only FROM this IP

tcpdump -i eth0 dst 10.0.1.5      # Only TO this IP



\# Filter by port:

tcpdump -i eth0 port 80           # HTTP traffic

tcpdump -i eth0 port 443          # HTTPS traffic

tcpdump -i eth0 port 5432         # PostgreSQL

tcpdump -i eth0 portrange 8080-8090



\# Filter by protocol:

tcpdump -i eth0 tcp               # TCP only

tcpdump -i eth0 udp               # UDP only

tcpdump -i eth0 icmp              # ICMP (ping) only



\# Combine filters:

tcpdump -i eth0 'host 10.0.1.5 and port 80'

tcpdump -i eth0 'src 10.0.1.5 and dst port 443'

tcpdump -i eth0 '(port 80 or port 443) and host 10.0.1.5'



\# Capture TCP flags:

tcpdump -i eth0 'tcp\[tcpflags] \& tcp-syn != 0'    # SYN packets

tcpdump -i eth0 'tcp\[tcpflags] \& tcp-rst != 0'    # RST packets

tcpdump -i eth0 'tcp\[tcpflags] \& tcp-fin != 0'    # FIN packets



\# Useful flags:

tcpdump -n               # Don't resolve hostnames (faster)

tcpdump -nn              # Don't resolve hostnames or ports

tcpdump -v / -vv / -vvv  # Increasing verbosity

tcpdump -c 100           # Capture only 100 packets then stop

tcpdump -w capture.pcap  # Write to file (for Wireshark analysis)

tcpdump -r capture.pcap  # Read from file

tcpdump -A               # Print packet content in ASCII

tcpdump -X               # Print packet content in hex + ASCII

tcpdump -s 0             # Capture full packet (not just header)



##### \# REAL-WORLD DEBUGGING EXAMPLES:



\# "Is DNS working?"

tcpdump -i eth0 -nn port 53

\# Shows DNS queries and responses with domain names



\# "Are SYN packets reaching my server?"

tcpdump -i eth0 -nn 'tcp\[tcpflags] == tcp-syn and dst port 8080'

\# If SYNs arrive but no SYN-ACKs → app not listening or firewall



\# "Why are connections being reset?"

tcpdump -i eth0 -nn 'tcp\[tcpflags] \& tcp-rst != 0'

\# Capture RST packets — see who's sending them and why



\# "What's the actual HTTP request being sent?"

tcpdump -i eth0 -A 'dst port 80' | grep -E 'GET|POST|HTTP|Host:'

\# Shows HTTP method, path, host header in plain text



\# "Capture 30 seconds of traffic for later analysis"

timeout 30 tcpdump -i eth0 -w /tmp/debug.pcap -s 0 'host 10.0.1.5'

\# Then download and open in Wireshark

```



##### \### traceroute / mtr — Path Analysis



```bash

\# traceroute shows every hop between you and the destination

traceroute 8.8.8.8

\# 1  10.0.0.1 (gateway)     1.2ms  1.1ms  1.3ms

\# 2  172.16.0.1              5.2ms  5.1ms  5.3ms

\# 3  \* \* \*                           ← hop doesn't respond (ICMP blocked)

\# 4  209.85.248.1            15.2ms 15.1ms 15.3ms

\# 5  8.8.8.8                 16.0ms 15.9ms 16.1ms



\# Each line = one router/hop

\# Three times = three probe responses (latency variation)

\# \* \* \* = hop blocks ICMP or traceroute probes



\# Use TCP-based traceroute for more reliable results:

traceroute -T -p 443 example.com

\# Uses TCP SYN instead of ICMP — less likely to be blocked



\# mtr — traceroute on steroids (continuous monitoring)

mtr 8.8.8.8

\# Real-time continuous traceroute

\# Shows: packet loss %, avg/best/worst latency per hop

\# Invaluable for detecting intermittent network issues

\#

\# HOST                Loss%  Snt   Last   Avg  Best  Wrst

\# 1. gateway          0.0%   50    1.2    1.3   1.0   2.1

\# 2. isp-router       0.0%   50    5.1    5.2   4.8   6.3

\# 3. backbone         2.0%   50   15.2   15.5  14.9  25.3  ← packet loss!

\# 4. destination       2.0%   50   16.1   16.3  15.8  26.1



\# If loss appears at hop 3 and continues → problem is AT hop 3

\# If loss appears at hop 3 but NOT at hop 4 → hop 3 just drops ICMP (normal)



\# Generate report for sharing with ISP/network team:

mtr -rw -c 100 example.com > mtr\_report.txt

\# -r = report mode

\# -w = wide (show full hostnames)

\# -c = count of pings

```



##### \### ss and netstat — Socket Statistics



```bash

\# ss is the modern replacement for netstat (faster, more info)



\# List all TCP connections

ss -tan



\# List all listening TCP ports

ss -tlnp

\# -t = TCP

\# -l = listening

\# -n = numeric (don't resolve names)

\# -p = show process



\# List all UDP sockets

ss -uanp



\# Filter by state

ss -tan state established

ss -tan state time-wait

ss -tan state close-wait



\# Filter by port

ss -tan 'sport = :8080'        # Source port 8080

ss -tan 'dport = :5432'        # Destination port 5432



\# Filter by address

ss -tan 'dst 10.0.1.5'



\# Connection summary

ss -s

\# Total: 15234 (kernel 15500)

\# TCP:   12345 (estab 8765, closed 1234, orphaned 12, timewait 2345)

\# UDP:   56



\# Per-socket TCP info (retransmissions, RTT, cwnd)

ss -tin

\# Shows: rto, rtt, cwnd, retrans for each connection

\# Invaluable for debugging slow connections



\# Find which process is using a port

ss -tlnp | grep :8080

\# LISTEN  0  128  \*:8080  \*:\*  users:(("java",pid=1234,fd=56))

```



##### \### curl — The HTTP Swiss Army Knife



```bash

\# Basic request

curl https://api.example.com/health



\# Verbose (show TLS handshake, headers, everything)

curl -vvv https://api.example.com/health



\# Only headers

curl -I https://api.example.com/health



\# With timing breakdown

curl -o /dev/null -s -w "\\

&#x20; DNS:        %{time\_namelookup}s\\n\\

&#x20; Connect:    %{time\_connect}s\\n\\

&#x20; TLS:        %{time\_appconnect}s\\n\\

&#x20; TTFB:       %{time\_starttransfer}s\\n\\

&#x20; Total:      %{time\_total}s\\n\\

&#x20; HTTP Code:  %{http\_code}\\n\\

&#x20; Size:       %{size\_download} bytes\\n" \\

&#x20; https://api.example.com/health



\# Output:

\# DNS:        0.012s        ← DNS resolution time

\# Connect:    0.045s        ← TCP handshake complete

\# TLS:        0.123s        ← TLS handshake complete

\# TTFB:       0.234s        ← Time to first byte (server processing)

\# Total:      0.250s        ← Total request time

\# HTTP Code:  200

\# Size:       15 bytes



\# If DNS is slow → DNS issue

\# If Connect-DNS is slow → network latency

\# If TLS-Connect is slow → TLS issue (cert validation, cipher)

\# If TTFB-TLS is slow → server is slow processing

\# This breakdown tells you EXACTLY where the latency is



\# Custom headers

curl -H "Authorization: Bearer token123" https://api.example.com



\# POST with JSON body

curl -X POST -H "Content-Type: application/json" \\

&#x20; -d '{"name":"test"}' https://api.example.com/users



\# Follow redirects

curl -L https://example.com



\# Insecure (skip TLS verification — for debugging only!)

curl -k https://self-signed.example.com



\# Timeout

curl --connect-timeout 5 --max-time 30 https://api.example.com



\# Resolve to specific IP (bypass DNS — test before migration)

curl --resolve api.example.com:443:10.0.1.5 https://api.example.com

\# Sends request to 10.0.1.5 but uses api.example.com for TLS SNI and Host header

\# Perfect for testing a new server before updating DNS

```



\---



\### Production Scenario: Systematic Network Debugging Framework



```bash

\# A developer says: "My service can't reach the database"

\# Service: app pod in EKS

\# Database: RDS PostgreSQL at db.internal.example.com:5432



##### \# LAYER-BY-LAYER DEBUGGING:



\# STEP 1: DNS Resolution (Layer 7)

kubectl exec -it app-pod -- nslookup db.internal.example.com

\# Does it resolve? To what IP?

\# If NXDOMAIN → DNS issue (CoreDNS, Route53 private zone)

\# If resolves → continue



\# STEP 2: Network Reachability (Layer 3)

kubectl exec -it app-pod -- ping -c 3 <db-ip>

\# Note: ICMP might be blocked — don't rely solely on ping

\# If works → Layer 3 is fine

\# If fails → routing issue, Security Group blocking ICMP



\# STEP 3: Port Reachability (Layer 4)

kubectl exec -it app-pod -- nc -zv <db-ip> 5432

\# Or: kubectl exec -it app-pod -- timeout 5 bash -c 'echo > /dev/tcp/<db-ip>/5432'

\# "Connection succeeded" → TCP port is open, firewall allows it

\# "Connection refused" → port is open but nothing listening

\# "Connection timed out" → firewall/SG blocking or no route



\# STEP 4: If timeout, check Security Groups

\# App pod → runs on node → node has SG

\# Check: node SG allows outbound TCP 5432?

\# Check: RDS SG allows inbound TCP 5432 from node SG or node CIDR?



\# STEP 5: If SGs are correct, check NACLs

\# Private subnet NACL allows outbound TCP 5432?

\# DB subnet NACL allows inbound TCP 5432?

\# DB subnet NACL allows outbound TCP 1024-65535 (return traffic)?



\# STEP 6: If NACLs are correct, check routing

\# Does the pod's node have a route to the DB subnet?

\# Are they in the same VPC? Peered VPCs? Transit Gateway?



\# STEP 7: Application layer

\# If TCP connection succeeds but app still fails:

\# Wrong credentials? Wrong database name? SSL required?

kubectl exec -it app-pod -- psql -h <db-ip> -U myuser -d mydb

\# This gives you the actual PostgreSQL error



\# THE CHECKLIST:

\# 1. DNS resolves correctly?

\# 2. TCP connection succeeds on the port?

\# 3. Security Groups allow the traffic?

\# 4. NACLs allow the traffic (both directions)?

\# 5. Route tables have correct routes?

\# 6. Application-level auth/config correct?

```



\---



\# 📋 LESSONS 7-9 QUICK REFERENCE



```

IPTABLES:

&#x20; Tables: filter (default), nat, mangle, raw

&#x20; Chains: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING

&#x20; First match wins (ORDER MATTERS)

&#x20; iptables -L -n -v --line-numbers    → View rules

&#x20; iptables -I INPUT 1 -s <ip> -j DROP → Emergency block

&#x20; K8s uses iptables for Service routing (kube-proxy)



SECURITY GROUPS:

&#x20; Stateful, allow-only, all rules evaluated

&#x20; Reference other SGs as source (dynamic!)

&#x20; Default: deny inbound, allow outbound

&#x20; Primary firewall for AWS



NACLs:

&#x20; Stateless, allow+deny, ordered by rule number

&#x20; Must allow ephemeral ports (1024-65535) for return traffic

&#x20; Use as coarse secondary layer, not primary firewall

&#x20; First match wins (like iptables)



VPC:

&#x20; IGW = internet access for public subnets

&#x20; NAT GW = outbound internet for private subnets (in public subnet!)

&#x20; VPC Peering = direct, non-transitive, free same-AZ

&#x20; Transit Gateway = hub-and-spoke, transitive, enterprise

&#x20; VPN = encrypted over internet, quick setup

&#x20; Direct Connect = dedicated physical, consistent, weeks to setup



VPC ENDPOINTS:

&#x20; Gateway (free): S3, DynamoDB

&#x20; Interface ($): ECR, SQS, SNS, CloudWatch, STS, etc.

&#x20; Critical for EKS in private subnets (save NAT costs)



DEBUGGING TOOLS:

&#x20; tcpdump -i eth0 -nn 'host X and port Y'     → Packet capture

&#x20; tcpdump 'tcp\[tcpflags] \& tcp-rst != 0'      → Find RSTs

&#x20; mtr <host>                                    → Continuous traceroute

&#x20; ss -tan state close-wait                      → Find socket leaks

&#x20; ss -tin                                       → Per-connection TCP stats

&#x20; curl -w timing format                         → HTTP timing breakdown

&#x20; curl --resolve domain:port:ip                 → Test before DNS change



SYSTEMATIC DEBUGGING:

&#x20; 1. DNS → 2. Ping/ICMP → 3. TCP port → 4. Security Groups →

&#x20; 5. NACLs → 6. Route tables → 7. Application layer



DEFENSE LAYERS (outer to inner):

&#x20; CDN/Cloudflare → AWS Shield → WAF → NACL → Security Group → iptables

```



\---



\# 📝 Retention Questions — Lessons 7-9



\*\*Q1:\*\* An EKS pod in a private subnet can reach other pods and services inside the cluster, but cannot pull images from ECR. `kubectl describe pod` shows `ImagePullBackOff`. NAT Gateway exists and is in a public subnet. Security Group allows all outbound. What's the most likely cause that people miss, and what's the cost-effective long-term fix?



\*\*Q2:\*\* During a DDoS attack, you need to block the CIDR `198.51.100.0/24` at the lowest possible level in AWS. Can you use a Security Group? Why or why not? What do you use instead, and why is the rule number important?



\*\*Q3:\*\* You run `curl -w` timing breakdown against your API and see: DNS=0.001s, Connect=0.002s, TLS=0.003s, TTFB=5.2s, Total=5.3s. Where is the bottleneck and what does this tell you about where to investigate?



\*\*Q4:\*\* A developer's service gets intermittent `Connection reset by peer` when talking to RDS through an NLB. The resets only happen on connections that have been idle for about 350 seconds. The NLB idle timeout is 350 seconds. TCP keepalive on the application is set to 7200 seconds. Explain the full chain of events and give the fix using the timeout hierarchy.



\*\*Go.\*\* 🎯



