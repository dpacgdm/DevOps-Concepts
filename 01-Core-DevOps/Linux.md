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

cat /proc/sys/kernel/pid_max

Once that's exhausted, no new processes can be created. Your system can't fork anything. SSH stops working. Cron jobs fail. Services can't restart. Complete meltdown.

##### You CANNOT kill a zombie process.

A zombie is ALREADY DEAD. It's a corpse sitting in the process table. You're trying to kill a dead body. kill -9 sends SIGKILL to a process — but there's no process to kill. It already exited. The entry is just waiting to be cleaned up.



So How Do You Actually Get Rid of Them?

You kill the PARENT.



Find the zombie

ps aux | awk '\\$8 == "Z"'



Find its parent PID

ps -eo pid,ppid,stat,cmd | grep Z



Kill the PARENT process

kill -SIGCHLD <parent_pid>    # First try: nudge the parent to reap

kill <parent_pid>              # Second try: terminate the parent

kill -9 <parent_pid>           # Last resort: force kill the parent



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

 - Stop accepting new requests

 - Finish processing in-flight requests

 - Close database connections

 - Flush caches/buffers

 - Exit cleanly

7\. If process hasn't exited after grace period → SIGKILL





⚠️ WHERE THIS GOES CATASTROPHICALLY WRONG:

Scenario 1: Application doesn't handle SIGTERM



Your Node.js app doesn't have a SIGTERM handler. Kubernetes sends SIGTERM → app ignores it → 30 seconds of doing nothing → SIGKILL. Users experience 30 seconds of requests going to a pod that's supposed to be dead. During rolling deployments, this means dropped requests and timeouts.



Scenario 2: Cleanup takes longer than 30 seconds



Your app handles SIGTERM but needs 60 seconds to drain connections. SIGKILL arrives at 30 seconds. Connections severed mid-transaction. Database writes half-completed. Data corruption.



**spec:**

**terminationGracePeriodSeconds: 90  # Give it enough time**

**containers:**

**- name: my-app**

  **lifecycle:**

    **preStop:**

      **exec:**

        **command: \["/bin/sh", "-c", "sleep 5"]**

        **# Why sleep 5? Because step 4 (endpoint removal)**

        **# is ASYNCHRONOUS. This gives kube-proxy time to**

        **# update iptables rules so new traffic stops**

        **# arriving BEFORE your app starts shutting down.**



Scenario 3: App traps SIGTERM but doesn't exit



Your app catches SIGTERM, runs cleanup, but has a bug — it never calls exit(). Process hangs. 30 seconds later, SIGKILL. Same problem as Scenario 1 but sneakier.



💡 SIGHUP in the Real World:

Many daemons use SIGHUP to reload configuration without restarting:

Reload Nginx config without downtime

kill -SIGHUP $(cat /var/run/nginx.pid)



Same as

nginx -s reload



This is how you do zero-downtime config changes on traditional servers. Prometheus, HAProxy, Nginx — they all support this.





💡 SIGUSR1/SIGUSR2 in the Real World:

These are custom signals. Applications define their own behavior:

Tell dd to print progress

kill -USR1 $(pgrep dd)



Tell Golang apps to dump goroutine stack traces

kill -USR1 <go-app-pid>





The kill Command — It's Not Just For Killing:

Despite the name, kill is really send_signal:



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

 - Already closed its listener → CONNECTION REFUSED → 502

 - Processing shutdown → SLOW/TIMEOUT → 502

6\. kube-proxy finally updates → traffic stops flowing to old pod

 But damage is done.



The Fix — preStop Hook:



lifecycle:

preStop:

  exec:

    command: \["/bin/sh", "-c", "sleep 5"]



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

content_copy

expand_less

CMD \["node", "server.js"]

Fix 2: Use an Init Process (e.g., Tini)



In some complex scenarios, you might actually need a shell or have multiple processes. In those cases, use a lightweight init system like tini. Tini is designed to act as PID 1, reap zombie processes, and properly forward signals to children.



Updated Dockerfile:



code

Dockerfile

download

content_copy

expand_less

FROM node:18



Install tini

RUN apt-get update \&\& apt-get install -y tini

WORKDIR /app

COPY . .

RUN npm install



Use tini as the entrypoint to manage signals

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



How many FDs is a process using?

ls /proc/<pid>/fd | wc -l



What is each FD pointing to?

ls -la /proc/<pid>/fd

You'll see symlinks like:

0 -> /dev/pts/0 (terminal)

1 -> /dev/pts/0 (terminal)

2 -> /dev/pts/0 (terminal)

3 -> socket:\[12345] (a TCP connection!)

4 -> /var/log/app.log (a log file)





###### **FD Limits — The Silent Production Killer:**

Every system has limits on how many FDs a process can open:



Soft limit (what's enforced by default)

ulimit -Sn

Usually 1024



Hard limit (maximum the soft limit can be raised to)

ulimit -Hn

Usually 65536



System-wide maximum

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



You can set this in your pod spec

securityContext:

ulimits:

- name: nofile

  soft: 65536

  hard: 65536



###### **I/O Redirection — The Full Picture**

You probably know > and >>. But do you know ALL of these?



stdout to file (overwrite)

command > file.txt



stdout to file (append)

command >> file.txt



stderr to file

command 2> errors.txt



stdout AND stderr to same file

command > all.txt 2>\&1

OR (modern bash)

command \&> all.txt



stderr to stdout (merge streams)

command 2>\&1



Discard all output (the black hole)

command > /dev/null 2>\&1



Send stdout to one file, stderr to another

command > output.txt 2> errors.txt



Pipe only stderr

command 2>\&1 1>/dev/null | grep "error"



###### The 2>\&1 Order Matters:

WRONG — stderr still goes to terminal

command 2>\&1 > file.txt

This says: "redirect stderr to where stdout currently points (terminal),

THEN redirect stdout to file." Stderr still hits terminal.



RIGHT — both go to file

command > file.txt 2>\&1

This says: "redirect stdout to file, THEN redirect stderr to

where stdout currently points (file)." Both go to file.





###### **The /proc Filesystem — Your X-Ray Vision**

/proc is a virtual filesystem. Nothing on disk. The kernel generates it on the fly. It's a window into every running process and the kernel's state.



Per-Process Information (/proc/<pid>/):



Command that started this process

cat /proc/<pid>/cmdline | tr '\\0' ' '



Environment variables of the process

cat /proc/<pid>/environ | tr '\\0' '\\n'



Current working directory

ls -la /proc/<pid>/cwd



What binary is running

ls -la /proc/<pid>/exe



Open file descriptors (we covered this)

ls -la /proc/<pid>/fd



Memory map

cat /proc/<pid>/maps



Process status (state, memory usage, threads)

cat /proc/<pid>/status



Network connections this process has

cat /proc/<pid>/net/tcp



System-Wide Information:

CPU info

cat /proc/cpuinfo



Memory info

cat /proc/meminfo



Kernel version

cat /proc/version



Mount points

cat /proc/mounts



Network statistics

cat /proc/net/dev



Current load average

cat /proc/loadavg



System uptime in seconds

cat /proc/uptime



###### **Real-World Uses:**

Scenario 1: An app is misbehaving but you can't find its config file. Where is it reading config from?



Check what files it has open

ls -la /proc/<pid>/fd | grep -i config



Check its environment variables for config paths

cat /proc/<pid>/environ | tr '\\0' '\\n' | grep -i config



Check its command line arguments

cat /proc/<pid>/cmdline | tr '\\0' ' '



Scenario 2: You suspect a process has been secretly modified (security incident). Is the running binary the same as what's on disk?



What binary is the process actually running?

ls -la /proc/<pid>/exe

If this points to "(deleted)" — someone replaced

the binary on disk while it was running. RED FLAG. 🚩



Scenario 3: You need to know what environment variables a running Java process has — but you can't restart it to add debugging:



cat /proc/<pid>/environ | tr '\\0' '\\n'

Now you can see every env var including secrets,

DB connection strings, API keys — everything the

process was given at startup.

(This is also why /proc permissions matter for security)



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



See inode number of a file

ls -i file.txt

1234567 file.txt



See inode usage

df -i

Filesystem    Inodes   IUsed   IFree  IUse%  Mounted on

/dev/sda1    6553600  6553600      0   100%  /





##### The Inode Exhaustion Disaster:

A filesystem has a fixed number of inodes set at creation time. You can have plenty of disk space but zero inodes. When inodes run out:

Can't create new files

Can't create new directories

Applications crash with "No space left on device" — but df -h shows free space

Logs stop writing

Deployments fail



The #1 cause: Millions of tiny files. Common culprits:



\- Session files (/tmp/sess_\*)

\- Mail queue files

\- Container layers (old Docker images)

\- Log rotation creating millions of small compressed logs

\- Package manager cache

\- Kubernetes pods creating temp files that never get cleaned up



###### How to Investigate:

Check inode usage

df -i



Find which directory has the most files

find / -xdev -printf '%h\\n' | sort | uniq -c | sort -rn | head -20



Count files in a specific directory

find /var/log -type f | wc -l



Find and delete old files

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

-x: extended stats

-z: skip idle devices

1: refresh every 1 second



Output:

Device  r/s    w/s    rkB/s   wkB/s  rrqm/s  wrqm/s  %util  await  avgqu-sz

sda     150    300    2400    4800    10      50      98.5   45.2   12.3



Reading the output:



%util = 98.5% → Disk is almost fully saturated. BAD.

await = 45.2ms → Each I/O operation takes 45ms. For SSD should be <1ms, for HDD <10ms. This is SLOW.

avgqu-sz = 12.3 → 12 operations waiting in queue. Requests are piling up.

r/s + w/s = 450 → 450 IOPS. Compare against disk capability.



##### I/O Wait — The CPU Metric That's Really a Disk Problem:



top

%Cpu(s): 5.2 us, 2.1 sy, 0.0 ni, 30.5 id, 62.0 wa, 0.0 hi, 0.2 si

                                                ^^^^^^

                                                62% iowait!



62% iowait means the CPU is spending 62% of its time doing NOTHING — just waiting for disk I/O to complete. This is why your server feels "slow" even though CPU usage looks low. The CPU isn't the bottleneck — the disk is.



Finding the Guilty Process:

**# Which process is doing the most I/O?**

**iotop -oP**

**# -o: only show processes doing I/O**

**# -P: show per-process, not per-thread**



**# Alternative if iotop isn't installed**

**pidstat -d 1**



##### Hard Links vs Soft Links:



Hard link — same inode, different name

ln original.txt hardlink.txt



Both point to the SAME inode

Deleting original.txt doesn't affect hardlink.txt

Cannot cross filesystem boundaries

Cannot link directories



Soft link (symlink) — pointer to a path

ln -s original.txt symlink.txt



Separate inode, points to the PATH of original

Deleting original.txt BREAKS symlink (dangling link)

Can cross filesystem boundaries

Can link directories



Real-world use: When you see /usr/bin/python3 -> /usr/bin/python3.11 — that's a symlink. Package managers use them extensively. Kubernetes uses symlinks inside /proc and for secret/configmap volume mounts.



###### Kubernetes Secret Volumes — The Symlink Trick:

When Kubernetes mounts a Secret as a volume:



/etc/secrets/

├── ..data -> ..2024_01_15_12_00_00.123456789   (symlink)

├── ..2024_01_15_12_00_00.123456789/            (actual directory)

│   └── db-password

└── db-password -> ..data/db-password            (symlink)



When the Secret is updated, Kubernetes:



Creates a new timestamped directory with new values

Atomically swaps the ..data symlink to point to the new directory

Deletes the old directory

This is an atomic update — applications either see the old value or the new value, never a half-written state. Symlinks make this possible.



##### Mount Points \& fstab:



See all mounted filesystems

mount



Or cleaner

findmnt



/etc/fstab — filesystems to mount at boot

<device>        <mount point>  <type>  <options>        <dump>  <pass>

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

df -h          → Disk space

df -i          → Inode usage (THE ONE EVERYONE FORGETS)

du -sh /path   → Directory size

ncdu /         → Interactive disk usage explorer



PERFORMANCE CHECKS:

iostat -xz 1   → Disk I/O metrics

iotop -oP      → Per-process I/O

pidstat -d 1   → Per-process I/O (alternative)

top → %iowait  → CPU waiting on disk



INVESTIGATION:

lsof +D /path  → What processes have files open in this directory

lsof -p <pid>  → All files a process has open

find / -xdev -printf '%h\\n' | sort | uniq -c | sort -rn | head

               → Find directories with most files (inode investigation)



KUBERNETES:

Secrets use symlinks for atomic updates

emptyDir with medium: Memory uses tmpfs

PV/PVC abstract storage from pods





##### Symlinks Across the Entire DevOps/SRE Landscape



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



\*\*Why this matters:\*\* Nginx config says `ssl _certificate /etc/letsencrypt/live/example.com/fullchain.pem`. When the cert renews, the symlink silently points to the new cert. Nginx reload picks it up. Zero config changes.



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

 \~/.terraform.d/plugin-cache/registry.terraform.io/hashicorp/aws/5.0.0/linux _amd64/

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



Now — The Troubleshooting Concepts I Should Have TAUGHT, Not Asked



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

 node _exporter exposes: node _filesystem _files _free



 3. Create filesystem with more inodes if needed

mkfs.ext4 -N 10000000 /dev/sdb1

 -N sets the number of inodes at creation time



 4. Use XFS for large-scale systems

 XFS allocates inodes dynamically — doesn't have fixed inode count

```



\---



🔥 NOW — Fair Troubleshooting Questions



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

Trace a running process

strace -p <pid>



Trace with timestamps

strace -tt -p <pid>



Trace only specific system calls

strace -e open,read,write -p <pid>    # File operations

strace -e connect,accept,sendto -p <pid>  # Network operations

strace -e openat -p <pid>             # What files is it opening?



Trace a new command from start

strace -f -o /tmp/trace.log command args

-f: follow child processes (forks)

-o: write to file instead of stderr



Count system calls (summary)

strace -c -p <pid>

Shows a table of which syscalls were called how many times

and how much time was spent in each

INCREDIBLE for finding bottlenecks

Real-World Uses:

"App won't start but gives no useful error message":



bash

strace -f ./myapp 2>\&1 | tail -50

Look for the LAST system call before it dies

Often you'll see:

openat(AT_FDCWD, "/etc/myapp/config.yml", O_RDONLY) = -1 ENOENT

(No such file or directory)

AH HA — it's looking for a config file that doesn't exist

"App is slow but CPU and memory are fine":



bash

strace -c -p <pid>

% time     seconds  usecs/call     calls    syscall

------ ----------- ----------- --------- ---------

 89.42    4.234567        4234      1000    futex

  5.21    0.246789         246      1000    read

\#

89% of time in futex = lock contention

The app is spending most of its time waiting on locks

"App can't connect to the database":



bash

strace -e connect -p <pid>

connect(5, {sa_family=AF_INET, sin_port=htons(5432),

  sin_addr=inet_addr("10.0.0.5")}, 16) = -1 ETIMEDOUT

\#

Now you know the EXACT IP and port it's trying to reach

And that it's timing out — not connection refused

This tells you it's a NETWORK/FIREWALL issue, not a service-down issue

⚠️ WARNING: strace adds significant overhead to the traced process. It can slow it down 10-100x. Never leave strace attached to a production process longer than necessary. Attach, get your data, detach.





##### **Core Dumps Filling Disk**

When a process crashes with a segfault or abort, Linux can write a core dump — a snapshot of the process's memory at the time of the crash. These can be HUGE.



bash

Check if core dumps are enabled

ulimit -c

0 = disabled

unlimited = enabled with no size limit ← DANGEROUS



Where do core dumps go?

cat /proc/sys/kernel/core_pattern

Possible values:

core                          → in the process's working directory

/var/crash/core.%e.%p.%t     → centralized with metadata

|/usr/lib/systemd/systemd-coredump  → systemd manages them

The Production Disaster:

An app keeps segfaulting in a loop. Each crash writes a 2GB core dump. Restart policy restarts it. Crashes again. Another 2GB dump. Within an hour: disk full.



The Fix:

bash

Option 1: Disable core dumps entirely (common in production)

echo 0 > /proc/sys/kernel/core_pattern

Or in limits.conf:

\* hard core 0



Option 2: Let systemd manage them with size limits

/etc/systemd/coredump.conf

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

-rwsr-xr-x 1 root root 68208 /usr/bin/passwd

   ^

   s = setuid bit



When ANY user runs 'passwd', it executes as ROOT

This is how regular users can change their own password

(because /etc/shadow is only writable by root)

The Security Danger:



If an attacker can place a setuid-root binary on your system, they have instant root access:



bash

Finding all setuid binaries (security audit)

find / -perm -4000 -type f 2>/dev/null

Review this list. Any unexpected setuid binaries = compromised system



Remove setuid from a binary

chmod u-s /path/to/binary

In production/containers:



yaml

Kubernetes — drop ALL capabilities and prevent privilege escalation

securityContext:

allowPrivilegeEscalation: false  # Prevents setuid from working

runAsNonRoot: true

capabilities:

  drop:

    - ALL



##### setgid (Set Group ID)

On files: Runs as the file's group instead of the user's group.



On directories: This is where it's actually useful. Any file created inside the directory inherits the directory's group instead of the creator's primary group:



bash

Shared project directory

mkdir /opt/project

chgrp developers /opt/project

chmod 2775 /opt/project

    ^

    2 = setgid bit



Now when anyone in the developers group creates a file:

touch /opt/project/newfile.txt

ls -la /opt/project/newfile.txt

-rw-rw-r-- 1 deploy developers ...

                    ^^^^^^^^^^

                    Inherited from directory, not user's primary group

This is essential for shared directories where multiple users need to read/write each other's files.



##### Sticky Bit

On directories: Only the file's owner (or root) can delete or rename files within the directory, even if others have write permission.



bash

ls -ld /tmp

drwxrwxrwt 15 root root 4096 Jan 15 03:00 /tmp

         ^

         t = sticky bit



Everyone can write to /tmp

But user A can't delete user B's files in /tmp

Only the file owner or root can delete



Set sticky bit

chmod 1777 /tmp

    ^

    1 = sticky bit

Numeric Special Permission Bits:

text

4000 = setuid

2000 = setgid

1000 = sticky bit



Combined with regular permissions:

chmod 4755 binary      # setuid + rwxr-xr-x

chmod 2775 directory   # setgid + rwxrwxr-x

chmod 1777 /tmp        # sticky + rwxrwxrwx

umask — Default Permission Control

When a new file or directory is created, its permissions are determined by the umask:



bash

umask

0022



For files:   666 - 022 = 644 (rw-r--r--)

For directories: 777 - 022 = 755 (rwxr-xr-x)



More restrictive umask

umask 077

Files: 666 - 077 = 600 (rw-------)

Dirs:  777 - 077 = 700 (rwx------)



Set in /etc/profile or \~/.bashrc for persistence

Why base is 666 for files and not 777: Linux deliberately doesn't give execute permission by default. You must explicitly chmod +x. This prevents accidentally creating executable files.



sudo — Privilege Escalation Done Right

bash

The sudo config file

visudo   # ALWAYS use visudo to edit — it validates syntax

       # A syntax error in sudoers = NOBODY can sudo = locked out



/etc/sudoers format:

user   HOST=(RUNAS_USER:RUNAS_GROUP)  COMMANDS

deploy   ALL=(ALL:ALL)                  ALL           # Full sudo access

nginx    ALL=(ALL)                      NOPASSWD: /bin/systemctl reload nginx

                                      ^^^^^^^^

                                      No password required for this specific command



Group-based sudo (recommended)

%sudo    ALL=(ALL:ALL) ALL

%devops  ALL=(ALL) NOPASSWD: ALL

% means group

sudo Best Practices in Production:

bash

1. NEVER give blanket NOPASSWD: ALL to individual users

Use specific commands:

deploy ALL=(ALL) NOPASSWD: /bin/systemctl restart myapp, /bin/journalctl



2. Use groups, not individual users

/etc/sudoers.d/devops

%devops ALL=(ALL) NOPASSWD: ALL



3. Drop files in /etc/sudoers.d/ instead of editing /etc/sudoers directly

Easier to manage with Ansible/Puppet/Chef



4. Log all sudo commands (usually enabled by default)

Check: /var/log/auth.log or /var/log/secure

grep sudo /var/log/auth.log



5. Restrict root login

/etc/ssh/sshd_config

PermitRootLogin no

Force everyone to use their own account + sudo

This gives you an AUDIT TRAIL of who did what

Production Scenarios — Permission Issues

Scenario 1: Application Can't Write Logs After Deployment

bash

Ansible deploys new code as root

Files now owned by root:root

Application runs as 'appuser'

Application can't write to /var/log/app/



The fix — in your Ansible playbook:

\- name: Set correct ownership

file:

  path: /var/log/app/

  owner: appuser

  group: appgroup

  mode: '0755'

  recurse: yes



Or better — use logrotate with create directive:

/var/log/app/\*.log {

  create 0644 appuser appgroup

  # New log files created with correct ownership

}

Scenario 2: Docker Volume Mount Permission Denied

bash

Container runs as UID 1000 (appuser inside container)

Host directory /data owned by root:root with 755

Container tries to write to /data → Permission denied



Fix 1: Match UIDs

chown 1000:1000 /data



Fix 2: Run container as specific user

docker run -u 1000:1000 -v /data:/data myapp



Fix 3: Use init container in Kubernetes to fix permissions

initContainers:

\- name: fix-permissions

image: busybox

command: \['sh', '-c', 'chown -R 1000:1000 /data']

volumeMounts:

- name: data

  mountPath: /data

Scenario 3: SSH Key Authentication Fails Despite Correct Key

bash

SSH is EXTREMELY strict about permissions

If any of these are wrong, key auth silently fails:



chmod 700 \~/.ssh

chmod 600 \~/.ssh/authorized_keys

chmod 600 \~/.ssh/id_rsa          # Private key

chmod 644 \~/.ssh/id_rsa.pub      # Public key

chown -R user:user \~/.ssh



Debug SSH auth failures:

ssh -vvv user@host   # Verbose output shows exactly where auth fails



Server-side logs:

tail -f /var/log/auth.log

"Authentication refused: bad ownership or modes for directory /home/user/.ssh"

Scenario 4: Cron Job Works Manually But Fails in Crontab

bash

Common causes:



1. PATH is different in cron environment

Cron has minimal PATH: /usr/bin:/bin

Your script uses /usr/local/bin/terraform → not found

Fix: Use full paths in cron scripts

0 2 \* \* \* /usr/local/bin/terraform apply -auto-approve



2. Script not executable

chmod +x /opt/scripts/backup.sh



3. Permission to write output

Cron tries to email output but sendmail isn't configured

Redirect output explicitly:

0 2 \* \* \* /opt/scripts/backup.sh >> /var/log/backup.log 2>\&1



4. Environment variables not available

Cron doesn't load .bashrc or .profile

Define needed vars in the crontab:

SHELL=/bin/bash

PATH=/usr/local/bin:/usr/bin:/bin

AWS_DEFAULT_REGION=us-east-1

0 2 \* \* \* /opt/scripts/backup.sh

Scenario 5: "Operation not permitted" Even as Root

bash

You're root. You try to delete a file. Permission denied.

rm -f /etc/important.conf

rm: cannot remove '/etc/important.conf': Operation not permitted



What? You're ROOT. How?



Answer: File has IMMUTABLE attribute set

lsattr /etc/important.conf

----i--------e-- /etc/important.conf

    ^

    i = immutable — even root can't modify or delete



Remove immutable attribute

chattr -i /etc/important.conf



Now you can delete/modify it



Set immutable (useful for protecting critical configs)

chattr +i /etc/important.conf



Other useful attributes:

chattr +a /var/log/audit.log    # Append-only — can add but not delete/modify

                                # Perfect for audit logs

Summary — Permission Debugging Checklist:

text

When "Permission denied" hits, check IN THIS ORDER:



1\. File/directory ownership:     ls -la <path>

2\. File/directory permissions:   ls -la <path>

3\. Parent directory permissions: ls -la <parent>  (need x to traverse)

4\. Special attributes:           lsattr <path>    (immutable? append-only?)

5\. SELinux/AppArmor:             getenforce        (is MAC enforcing?)

                                ls -laZ <path>   (SELinux context)

6\. ACLs:                         getfacl <path>    (extended ACLs overriding?)

7\. Mount options:                mount | grep <fs>  (mounted read-only? noexec?)

8\. Capabilities:                 getcap <binary>    (Linux capabilities)

Speaking of which — I mentioned SELinux and ACLs. Let me cover those.



SELinux / AppArmor — Mandatory Access Control (MAC)

Regular permissions (rwx) are Discretionary Access Control (DAC) — the file owner decides who gets access. SELinux and AppArmor are Mandatory Access Control — the SYSTEM decides, and even root can be restricted.



SELinux (Red Hat, CentOS, Fedora, Amazon Linux):

bash

Check status

getenforce

Enforcing = actively blocking violations

Permissive = logging violations but not blocking

Disabled = off completely



View SELinux context of files

ls -laZ /var/www/html/

-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html

                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

                       SELinux context



View SELinux context of processes

ps auxZ | grep httpd

system_u:system_r:httpd_t:s0  root  1234 ... /usr/sbin/httpd

                  ^^^^^^^^

                  httpd runs in httpd_t domain

                  It can ONLY access files labeled httpd_sys_content_t

The Classic SELinux Production Disaster:



bash

You deploy a new app that writes to /opt/myapp/data/

App runs fine on your dev box (SELinux disabled)

Deploy to production (SELinux enforcing) → Permission denied

rwx permissions look correct

Ownership looks correct

You spend 3 hours going insane



Check the audit log:

ausearch -m AVC -ts recent

type=AVC msg=audit: avc:  denied  { write } for pid=1234

comm="myapp" name="data" scontext=system_u:system_r:myapp_t

tcontext=system_u:object_r:default_t



The directory has default_t context — wrong!

Fix:

semanage fcontext -a -t myapp_data_t "/opt/myapp/data(/.\*)?"

restorecon -Rv /opt/myapp/data/



Or if you don't have a custom policy:

chcon -R -t httpd_sys_rw_content_t /opt/myapp/data/



QUICK DIAGNOSTIC: Temporarily set permissive to confirm SELinux is the issue

setenforce 0    # Permissive (logs but doesn't block)

Test your app — if it works now, SELinux was the problem

setenforce 1    # Back to enforcing

Then fix the contexts properly

AppArmor (Ubuntu, Debian, SUSE):

bash

Check status

aa-status



Profiles can be in:

enforce — blocking violations

complain — logging violations (like SELinux permissive)



View loaded profiles

cat /sys/kernel/security/apparmor/profiles



If AppArmor blocks your app:

Check logs

journalctl | grep apparmor



Temporarily disable profile for a binary

aa-complain /usr/sbin/nginx



Create/modify profile

aa-genprof /usr/sbin/myapp

ACLs — Access Control Lists (Fine-Grained Permissions)

When standard owner/group/others isn't enough, ACLs let you set permissions for specific users or groups:



bash

View ACLs

getfacl /opt/project/



Set ACL — give 'deploy' user read+write, without changing ownership

setfacl -m u:deploy:rw /opt/project/data.txt



Set ACL — give 'devops' group full access

setfacl -m g:devops:rwx /opt/project/



Set default ACL (inherited by new files created in directory)

setfacl -d -m g:devops:rwx /opt/project/



Remove ACL

setfacl -x u:deploy /opt/project/data.txt



Remove ALL ACLs

setfacl -b /opt/project/data.txt



A + sign in ls -la indicates ACLs are set:

ls -la /opt/project/

drwxrwx---+ 2 root root 4096 ...

          ^

          + means ACLs exist — check with getfacl

Real-world use: Shared CI/CD build directories where multiple service accounts need different levels of access but you don't want to create a group for every combination.



Linux Capabilities — Granular Root Powers

Instead of giving a process full root access, Linux capabilities let you grant specific root privileges:



bash

View capabilities of a binary

getcap /usr/bin/ping

/usr/bin/ping = cap_net_raw+ep

This is why regular users can run ping without root

It only has the NET_RAW capability, not full root



Common capabilities:

CAP_NET_BIND_SERVICE — bind to ports below 1024

CAP_NET_RAW — raw sockets (ping, tcpdump)

CAP_SYS_ADMIN — various admin operations (almost as powerful as root)

CAP_CHOWN — change file ownership

CAP_DAC_OVERRIDE — bypass file permission checks



Set capability on a binary

setcap cap_net_bind_service+ep /usr/local/bin/myapp

Now myapp can bind to port 80 without running as root



In Kubernetes:

securityContext:

capabilities:

  add:

    - NET_BIND_SERVICE    # Only this specific privilege

  drop:

    - ALL                 # Drop everything else

Why this matters: Running containers as root is a massive security risk. Capabilities let you give containers ONLY the specific privileges they need. In FAANG-level security reviews, drop: ALL + selective add is mandatory.





USERS \& GROUPS:

whoami / id                              → Current user info

useradd -m -s /bin/bash <user>          → Create user

usermod -aG <group> <user>              → Add to group (ALWAYS use -a!)

/etc/passwd                              → User accounts

/etc/shadow                              → Password hashes

/etc/group                               → Group memberships



PERMISSIONS:

chmod 755 file        → rwxr-xr-x

chmod 600 file        → rw-------

chown user:group file → Change ownership

r=4, w=2, x=1        → Octal math



SPECIAL BITS:

4000 = setuid         → Runs as file owner (DANGEROUS)

2000 = setgid         → Inherits directory group

1000 = sticky bit     → Only owner can delete (e.g., /tmp)



UMASK:

umask 022             → Files: 644, Dirs: 755

umask 077             → Files: 600, Dirs: 700



SUDO:

visudo                → ALWAYS use this to edit sudoers

usermod -aG sudo user → Grant sudo via group

/etc/sudoers.d/       → Drop-in files (Ansible-friendly)



SPECIAL ATTRIBUTES:

lsattr / chattr       → Immutable (+i), append-only (+a)

chattr +i file        → Even root can't delete



SELINUX:

getenforce            → Check status

setenforce 0          → Permissive (test mode)

ls -Z                 → View SELinux contexts

restorecon -Rv /path  → Fix contexts

ausearch -m AVC       → Check denial logs



APPARMOR:

aa-status             → Check status

aa-complain <profile> → Switch to complain mode



ACLS:

getfacl / setfacl     → Fine-grained per-user/group permissions

setfacl -m u:user:rwx → Grant specific user access

+ in ls -la           → Indicates ACLs present



CAPABILITIES:

getcap / setcap       → View/set granular root privileges

cap_net_bind_service  → Bind to port <1024 without root



KUBERNETES SECURITY:

runAsNonRoot: true

runAsUser: 1000

allowPrivilegeEscalation: false

readOnlyRootFilesystem: true

capabilities: drop: \[ALL]



PERMISSION DEBUGGING ORDER:

1. ls -la             → ownership \& permissions

2. ls -la parent dir  → traversal (x bit)

3. lsattr             → immutable/append-only

4. getenforce / ls -Z → SELinux

5. getfacl             → ACLs

6. mount               → read-only mount?

7. getcap              → capabilities

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

| \*\*PID\*\* | CLONE_NEWPID | Process IDs | Container sees only its own processes. PID 1 inside = your app |

| \*\*NET\*\* | CLONE_NEWNET | Network stack | Container gets its own IP, ports, routing table, iptables |

| \*\*MNT\*\* | CLONE_NEWNS | Mount points | Container sees its own filesystem, not the host's |

| \*\*UTS\*\* | CLONE_NEWUTS | Hostname \& domain | Container can have its own hostname |

| \*\*IPC\*\* | CLONE_NEWIPC | Inter-process communication | Shared memory, semaphores isolated between containers |

| \*\*USER\*\* | CLONE_NEWUSER | User/Group IDs | UID 0 inside container can map to UID 1000 on host (rootless containers) |

| \*\*CGROUP\*\* | CLONE_NEWCGROUP | Cgroup root view | Container sees only its own cgroup hierarchy |

| \*\*TIME\*\* | CLONE_NEWTIME | System clocks | Container can have different boot/monotonic clocks (Linux 5.6+) |



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

cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit                                      _in                                      _bytes

cat /sys/fs/cgroup/memory/docker/<container-id>/memory.usage                                      _in                                      _bytes

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

 usage _usec 123456789

 user _usec 100000000

 system _usec 23456789

 nr _periods 50000

 nr _throttled 12000    ← THROTTLED 12,000 times!

 throttled _usec 600000000  ← 600 seconds total throttle time



 In Kubernetes, this shows up as:

 - Increased latency (P99 spikes)

 - Slow response times

 - App appears "laggy" but CPU usage looks fine



 Prometheus metric:

 container _cpu _cfs _throttled _periods _total

 container _cpu _cfs _throttled _seconds _total



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

pivot _root /path/to/new/root /path/to/put/old/root



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

 nr _throttled → how many times CPU was throttled

 throttled _usec → total throttle time

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

 Python: os.sched _getaffinity(0)

 Go: runtime.GOMAXPROCS() — automatically reads cgroup since Go 1.19

 Node.js: doesn't auto-detect, use UV _THREADPOOL _SIZE env var

```



\---



📋 LESSON 6 QUICK REFERENCE — cgroups \& Namespaces



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

   cpu.stat                        → nr _throttled, throttled _usec

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

 Check: cat cpu.stat → nr _throttled

 Monitor: container _cpu _cfs _throttled _periods _total

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



📝 Retention Questions — Lesson 6 ONLY



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

Service tells systemd when it's ready via sd _notify().

Most reliable for complex services with initialization phases.

Example: PostgreSQL, systemd-aware services

Requires the application to call sd _notify("READY=1")



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

App with slow initialization       → Type=notify (if app supports sd _notify)

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

Environment=NODE _ENV=production

Environment=PORT=8080 DB _HOST=localhost



  From file (preferred — keeps secrets out of unit file)

EnvironmentFile=/etc/myapp/env

  The file format:

  NODE _ENV=production

  DB _HOST=postgres.internal

  DB _PASSWORD=supersecret



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

CapabilityBoundingSet=CAP _NET _BIND _SERVICE

AmbientCapabilities=CAP _NET _BIND _SERVICE

  Only allow binding to ports below 1024



  Network restrictions

RestrictAddressFamilies=AF _INET AF _INET6 AF _UNIX

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

journalctl _PID=1234            # Logs from specific PID

journalctl _UID=1000            # Logs from specific user



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

  via file descriptor inheritance (sd _listen _fds)

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

    daemon _reload: yes

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

  CLI inherits your shell's env (JAVA _HOME, AWS _REGION, etc.)

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



  For apps that don't support watchdog/sd _notify:

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

Environment=NODE _ENV=production

Environment=NODE _OPTIONS="--max-old-space-size=1536"



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

Environment=JAVA _HOME=/usr/lib/jvm/java-17

Environment=JAVA _OPTS="-Xms512m -Xmx1536m -XX:+UseG1GC -XX:+UseContainerSupport"



ExecStart=/usr/lib/jvm/java-17/bin/java 

    $JAVA _OPTS  \\

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







📝 Retention Questions — Lesson 7 ONLY



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

sysctl net.ipv4.tcp _tw _reuse

 net.ipv4.tcp _tw _reuse = 0



 Same thing via /proc

cat /proc/sys/net/ipv4/tcp _tw _reuse

 0



 The mapping:

 sysctl dots = filesystem slashes

 net.ipv4.tcp _tw _reuse → /proc/sys/net/ipv4/tcp _tw _reuse


 Set a parameter (temporary — lost on reboot)

sysctl -w net.ipv4.tcp _tw _reuse=1

 OR

echo 1 > /proc/sys/net/ipv4/tcp _tw _reuse



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



net.core.netdev _max _backlog = 65535

 Maximum packets queued on the INPUT side when interface receives

 packets faster than the kernel can process them

 DEFAULT: 1000

 On 10Gbps+ NICs, packets arrive faster than 1000/burst

 Symptom of too low: packet drops visible in `ifconfig` or `ip -s link`



net.ipv4.tcp _max _syn _backlog = 65535

 Maximum number of half-open connections (SYN _RECV state)

 DEFAULT: 128 or 1024 depending on distro

 During SYN flood attacks or traffic spikes, this fills up

 Legitimate connections get dropped

 Symptom: "connection timeout" for new users during traffic spikes



net.ipv4.tcp _max _tw _buckets = 2000000

 Maximum number of TIME _WAIT sockets

 DEFAULT: varies (often 32768)

 We covered TIME _WAIT exhaustion in Lesson 4

 If you hit this limit, kernel DESTROYS TIME _WAIT sockets prematurely

 That's actually dangerous — can cause connection confusion

 Set high enough that you never hit it, then fix the real problem

 (connection pooling, keep-alive)

```



\#### TIME_WAIT and Connection Reuse



```bash

net.ipv4.tcp _tw _reuse = 1

 Allow reusing TIME _WAIT sockets for new OUTBOUND connections

 DEFAULT: 0 (disabled)

 SAFE for client-side connections (your app connecting to DB, APIs)

 The kernel checks TCP timestamps to ensure safety

 MUST HAVE for reverse proxies, microservices, anything making many outbound calls



 NOTE: tcp _tw _recycle was REMOVED from Linux 4.12+

 It was dangerous behind NAT (broke connections from same source IP)

 If you see old guides recommending it → IGNORE THEM

```



\#### Keep-Alive



```bash

net.ipv4.tcp _keepalive _time = 300

 How many SECONDS before sending the first keepalive probe

 DEFAULT: 7200 (2 hours!) ← Way too long

 If a connection is idle for 2 hours before detecting it's dead,

 you're holding resources for a ghost connection

 Set to 300 (5 minutes) or lower for services behind load balancers



net.ipv4.tcp _keepalive _intvl = 30

 Seconds between subsequent keepalive probes

 DEFAULT: 75



net.ipv4.tcp _keepalive _probes = 5

 Number of failed probes before declaring connection dead

 DEFAULT: 9

 With these settings: dead connection detected in 300 + (30 × 5) = 450 seconds

 vs default: 7200 + (75 × 9) = 7875 seconds (over 2 hours!)



 Why this matters:

 Load balancers (AWS ALB/NLB) have their own idle timeouts

 ALB default: 60 seconds

 If your keepalive _time (7200s) > ALB idle timeout (60s):

   ALB silently drops the connection

   Server doesn't know it's dead

   Next request on that connection → RST → error

   This causes intermittent "connection reset" errors

 FIX: Set keepalive _time LOWER than your load balancer's idle timeout

```



\#### TCP Buffer Sizes



```bash

 ── TCP MEMORY AND BUFFER SIZES ──



net.core.rmem _max = 16777216

 Maximum receive buffer size (16MB)

 DEFAULT: 212992 (208KB)



net.core.wmem _max = 16777216

 Maximum send buffer size (16MB)

 DEFAULT: 212992 (208KB)



net.core.rmem _default = 1048576

net.core.wmem _default = 1048576

 Default buffer sizes (1MB)



net.ipv4.tcp _rmem = 4096 1048576 16777216

 TCP receive buffer: min  default  max

 Kernel auto-tunes between min and max

 DEFAULT: 4096 131072 6291456



net.ipv4.tcp _wmem = 4096 1048576 16777216

 TCP send buffer: min  default  max

 DEFAULT: 4096 16384 4194304



net.ipv4.tcp _mem = 786432 1048576 1572864

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

net.ipv4.ip _local _port _range = 1024 65535

 Range of ports used for outbound connections

 DEFAULT: 32768 60999 ( \~28,000 ports)

 Expanded: 1024 65535 ( \~64,000 ports)

 Critical for reverse proxies and microservices making many outbound calls

 We covered this in TIME _WAIT lesson — doubling ports = doubling headroom

```



###### \#### TCP Performance Features



```bash

net.ipv4.tcp _window _scaling = 1

 Enable TCP window scaling (RFC 1323)

 DEFAULT: 1 (enabled)

 Required for windows larger than 64KB

 NEVER disable this



net.ipv4.tcp _timestamps = 1

 Enable TCP timestamps (RFC 1323)

 DEFAULT: 1

 Required for tcp _tw _reuse to work safely

 Also helps with RTT measurement and PAWS protection

 NEVER disable this



net.ipv4.tcp _sack = 1

 Selective Acknowledgment

 DEFAULT: 1

 Allows receiver to tell sender EXACTLY which packets were lost

 Without SACK: entire window must be retransmitted

 With SACK: only lost packets are retransmitted

 MASSIVE performance improvement on lossy networks

 NEVER disable this



net.ipv4.tcp _fastopen = 3

 TCP Fast Open — send data in the SYN packet

 DEFAULT: 1 (client only)

 3 = enable for both client and server

 Saves one RTT on connection establishment

 Particularly useful for HTTP connections

 Supported by modern browsers and servers



net.ipv4.tcp _slow _start _after _idle = 0

 DEFAULT: 1

 When 1: TCP resets congestion window after idle period

 After the idle timeout, TCP acts like a brand new connection

 (starts with slow start, ramps up slowly)

 For persistent connections (HTTP keep-alive, database pools),

 this means every burst after idle gets artificially throttled

 Set to 0: maintain congestion window across idle periods

 Critical for services with bursty traffic patterns



net.ipv4.tcp _mtu _probing = 1

 Enable Path MTU discovery

 DEFAULT: 0

 Discovers the maximum packet size along the path

 Prevents fragmentation issues, especially in VPNs/tunnels/cloud networks

 AWS VPC has MTU of 9001 (jumbo frames) within VPC but 1500 across VPN

 Without MTU probing, you get mysterious packet loss on cross-VPC traffic

```



\#### SYN Flood Protection



```bash

net.ipv4.tcp _syncookies = 1

 Enable SYN cookies

 DEFAULT: 1 (usually enabled)

 When SYN backlog is full, instead of dropping connections,

 kernel encodes state in the SYN-ACK sequence number

 Allows legitimate connections to complete even during SYN flood

 This is your first line of defense against SYN flood DDoS



net.ipv4.tcp _synack _retries = 2

 Number of times to retry SYN-ACK

 DEFAULT: 5

 During SYN flood, retrying 5 times wastes resources on fake connections

 Reduce to 2 — legitimate clients retry on their own

 Frees up resources faster



net.ipv4.tcp _syn _retries = 3

 Number of times to retry SYN for outbound connections

 DEFAULT: 6 (which can mean  \~127 seconds before giving up!)

 Reduce to 3 for faster failure detection

 Prefer fast failure + application retry over slow kernel retry



net.ipv4.tcp _abort _on  _overflow = 0

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



vm.overcommit _memory = 0

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



vm.overcommit _ratio = 80

 Only used when overcommit _memory = 2

 System allows: swap + (physical _RAM × ratio/100) total memory

 With 32GB RAM, 8GB swap, ratio=80:

 Max = 8GB + (32GB × 0.80) = 33.6GB total allocatable



vm.dirty _ratio = 15

 Percentage of total memory that can be dirty (waiting to be written to disk)

 before the PROCESS is forced to start writing

 DEFAULT: 20

 When a process writes data, it goes to page cache (dirty pages)

 Kernel eventually flushes to disk

 If dirty pages exceed this ratio, the WRITING PROCESS is blocked

 until dirty pages are flushed — causes application latency spikes



vm.dirty _background _ratio = 5


 Percentage of total memory that can be dirty before the kernel

 starts flushing IN THE BACKGROUND

 DEFAULT: 10

 Background flushing doesn't block the application

 Set lower than dirty _ratio to start background flush early

 Prevents hitting dirty _ratio (which blocks the app)



 For databases and write-heavy workloads:

 Lower both ratios to reduce flush storms:

vm.dirty _ratio = 10

vm.dirty _backgroun _ratio = 3

 This ensures smaller, more frequent flushes instead of

 large, infrequent flushes that cause latency spikes



vm.min _free _kbytes = 524288

 Minimum free memory the kernel maintains (in KB)

 DEFAULT: varies based on RAM (usually very low)

 524288 = 512MB

 Kernel reserves this much free memory for critical allocations

 On busy servers, if free memory drops too low, memory allocation

 can stall waiting for reclaim → latency spike

 Set to 512MB-1GB on production servers with 32GB+ RAM

 Too high = wasted memory; too low = allocation stalls



vm.panic _on _oom = 0

 DEFAULT: 0

 0 = OOM killer picks a process to kill

 1 = kernel panics (reboots if panic _on _oom _timeout is set)

 Use 1 for Kubernetes nodes where you'd rather reboot than

 have a half-dead node with random processes killed

 The node comes back clean, kubelet reschedules pods elsewhere

 Some FAANG companies use this on K8s nodes



vm.vfs _cache _pressure = 50

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



fs.inotify.max _user _watches = 524288

 Maximum inotify file watches per user

 DEFAULT: 8192

 inotify = kernel facility to watch files/directories for changes

 Used by: file sync tools, IDEs, webpack, log collectors, Prometheus node _exporter

 8192 is laughably low for any development or monitoring server

 Kubernetes nodes with many pods need this high

 Symptom of too low: "no space left on device" (misleading error from inotify)

 or "ENOSPC: System limit for file watchers reached"



fs.inotify.max _user _instances = 8192

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



net.ipv4.ip _forward = 1

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



net.netfilter.nf _conntrack _max = 1048576

 Maximum entries in the connection tracking table

 DEFAULT: 65536 or 131072

 Every connection through iptables/kube-proxy creates a conntrack entry

 On busy K8s nodes with many services, this fills up FAST

 Symptom: "nf _conntrack: table full, dropping packet"

 This causes RANDOM packet drops — one of the hardest issues to debug

 Monitor: cat /proc/sys/net/netfilter/nf _conntrack _count



net.netfilter.nf _conntrack _tcp _timeout _established = 86400

 How long to keep conntrack entries for established connections

 DEFAULT: 432000 (5 days!)

 Stale entries waste conntrack table space

 86400 (1 day) is sufficient for most workloads



net.netfilter.nf _conntrack _tcp _timeout _time _wait = 30


 Conntrack timeout for TIME _WAIT connections

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

  \[Jan 15 14:23:45] nf _conntrack: table full, dropping packet

  \[Jan 15 14:23:45] nf _conntrack: table full, dropping packet

  \[Jan 15 14:23:46] nf _conntrack: table full, dropping packet

  \\\\#

 Check current usage:

cat /proc/sys/net/netfilter/nf _conntrack _count

 65432

cat /proc/sys/net/netfilter/nf _conntrack _max

 65536

 99.8% full! Packets being dropped!

  \\\\#

 Immediate fix:

sysctl -w net.netfilter.nf _conntrack _max=1048576

  \\\\#

 Permanent fix:

echo "net.netfilter.nf _conntrack _max = 1048576" >> /etc/sysctl.d/40-kubernetes.conf

sysctl --system

  \\\\#

 Monitor in Prometheus:

 node _nf _conntrack _entries / node _nf _conntrack _entries _limit

 Alert when > 75%

```



\---



###### \### Category 5: Security Hardening



```bash

 ── NETWORK SECURITY ──



net.ipv4.conf.all.rp _filter = 1

 Reverse Path Filtering — anti-spoofing

 DEFAULT: varies

 1 = strict mode — drop packets if return path uses different interface

 Prevents IP address spoofing attacks

 ENABLE on all production servers



net.ipv4.conf.default.rp _filter = 1

 Same for new interfaces created after boot



net.ipv4.conf.all.accept _redirects = 0

net.ipv6.conf.all.accept _redirects = 0

 Don't accept ICMP redirects

 Redirects can be used to reroute traffic through attacker's machine

 On servers with static routing, you don't need redirects



net.ipv4.conf.all.send _redirects = 0

 Don't send ICMP redirects

 Only routers should send redirects

 Your server is not a router (even if ip _forward is enabled for K8s)



net.ipv4.conf.all.accept _source _route = 0

net.ipv6.conf.all.accept _source _route = 0

 Reject source-routed packets

 Source routing lets the SENDER specify the route — security risk



net.ipv4.icmp _echo _ignore _broadcasts = 1

 Ignore broadcast ICMP echo (ping)

 Prevents Smurf attacks



net.ipv4.icmp _ignore _bogus _error _responses = 1

 Ignore bogus ICMP error messages

 Prevents log flooding from malformed ICMP



net.ipv4.conf.all.log _martians = 1

 Log packets with impossible source addresses

 Helps detect spoofing attempts

 Check logs: dmesg | grep martian



 ── KERNEL SECURITY ──



kernel.randomize _va _space = 2

 Address Space Layout Randomization (ASLR)

 DEFAULT: 2

 0 = disabled (NEVER do this in production)

 1 = randomize stack, mmap, VDSO

 2 = full randomization (stack, mmap, VDSO, heap)

 Makes buffer overflow exploits much harder



kernel.dmesg _restrict = 1

 Restrict dmesg to root only

 DEFAULT: 0

 Prevents unprivileged users from reading kernel messages

 Kernel messages can contain sensitive info (memory addresses, hardware)



kernel.kptr _restrict = 2

 Hide kernel pointers in /proc/kallsyms

 DEFAULT: 0

 Prevents leaking kernel memory layout (used in exploits)

 1 = hide from non-root

 2 = hide from everyone including root



kernel.yama.ptrace _scope = 2

 Restrict ptrace (used by debuggers, strace)

 DEFAULT: 1

 0 = any process can ptrace any other (DANGEROUS)

 1 = only parent can ptrace child

 2 = only root with CAP _SYS _PTRACE can ptrace

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

net.core.netdev _max _backlog = 65535

net.ipv4.tcp _max _syn _backlog = 65535

net.ipv4.tcp _tw _reuse = 1

net.ipv4.ip _local _port _range = 1024 65535

net.ipv4.tcp _keepalive _time = 60

net.ipv4.tcp _keepalive _intvl = 10

net.ipv4.tcp _keepalive _probes = 6

net.ipv4.tcp _slow _start _after _idle = 0

net.ipv4.tcp _fastopen = 3

net.ipv4.tcp _syncookies = 1

net.core.rmem _max = 16777216

net.core.wmem _max = 16777216

net.ipv4.tcp _rmem = 4096 1048576 16777216

net.ipv4.tcp _wmem = 4096 1048576 16777216

fs.file-max = 2097152

```



###### \#### Database Server (PostgreSQL/MySQL):



```bash

 /etc/sysctl.d/10-database.conf

vm.swappiness = 1

vm.dirty _ratio = 10

vm.dirty _background _ratio = 3

vm.overcommit _memory = 2

vm.overcommit _ratio = 90

net.core.somaxconn = 65535

net.ipv4.tcp _keepalive _time = 300

net.ipv4.tcp _keepalive _intvl = 30

net.ipv4.tcp _keepalive _probes = 5

fs.file-max = 2097152

fs.aio-max-nr = 1048576

 Also: set huge pages for large shared _buffers

vm.nr _hugepages = 1024    # 1024 × 2MB = 2GB huge pages

 Huge pages = locked in RAM, never swapped, lower TLB pressure

 Critical for databases with large buffer pools

```



###### \#### Redis Server:



```bash

 /etc/sysctl.d/10-redis.conf

vm.swappiness = 0              # NEVER swap Redis memory

vm.overcommit _memory = 1       # REQUIRED by Redis for fork/COW

net.core.somaxconn = 65535     # Redis accept queue

net.ipv4.tcp _max _syn _backlog = 65535

 Also in Redis config:

 tcp-backlog 65535            # Must match somaxconn

```



###### \#### Kubernetes Node:



```bash

 /etc/sysctl.d/40-kubernetes.conf

 Network

net.bridge.bridge-nf-call-iptables = 1

net.bridge.bridge-nf-call-ip6tables = 1

net.ipv4.ip _forward = 1

net.ipv4.conf.all.forwarding = 1

net.ipv6.conf.all.forwarding = 1

net.ipv4.tcp _tw _reuse = 1

net.ipv4.ip _local _port _range = 1024 65535

net.core.somaxconn = 65535

net.ipv4.tcp _keepalive _time = 60



 Conntrack

net.netfilter.nf _conntrack _max = 1048576

net.netfilter.nf _conntrack _tcp _timeout _established = 86400

net.netfilter.nf _conntrack _tcp _timeout _time _wait = 30



 Memory

vm.swappiness = 10

vm.panic _on _oom = 0

vm.min _free _kbytes = 524288



 Filesystem

fs.file-max = 2097152

fs.inotify.max _user _watches = 524288

fs.inotify.max _user _instances = 8192



 Security

net.ipv4.conf.all.rp _filter = 1

net.ipv4.conf.all.accept _redirects = 0

net.ipv4.conf.all.send _redirects = 0

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

   - name: net.ipv4.tcp _keepalive _time

     value: "60"

 containers:

 - name: myapp

   image: myapp:latest

```



\*\*But there's a catch:\*\*



```bash

 Kubernetes classifies sysctls as:

 SAFE — namespaced, doesn't affect other pods

   net.ipv4.ping _group _range

   net.ipv4.ip _unprivileged _port _start

  \\\\#

 UNSAFE — could affect the node or other pods

   net.core.somaxconn

   net.ipv4.tcp _keepalive _time

   (most useful ones are "unsafe")



 To allow unsafe sysctls, kubelet must be configured:

 --allowed-unsafe-sysctls="net.core.somaxconn,net.ipv4.tcp _keepalive _time"



 Or in kubelet config:

apiVersion: kubelet.config.k8s.io/v1beta1

kind: KubeletConfiguration

allowedUnsafeSysctls:

 - "net.core.somaxconn"

 - "net.ipv4.tcp _keepalive _time"

 - "net.ipv4.tcp _tw _reuse"



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

   sysctl _file: /etc/sysctl.d/10-production.conf

   reload: yes

   state: present

 loop:

   - { name: 'net.core.somaxconn', value: '65535' }

   - { name: 'net.ipv4.tcp _tw _reuse', value: '1' }

   - { name: 'net.ipv4.ip _local _port _range', value: '1024 65535' }

   - { name: 'vm.swappiness', value: '10' }

   - { name: 'fs.file-max', value: '2097152' }

   # ... etc

```



\### How to Apply sysctl via Terraform (EKS Node Groups):



```hcl

 In the EKS node group launch template user _data:

resource "aws _launch _template" "eks _nodes" {

 user _data = base64encode(<<-USERDATA

   #!/bin/bash

   cat >> /etc/sysctl.d/40-kubernetes.conf << 'EOF'

   net.bridge.bridge-nf-call-iptables = 1

   net.ipv4.ip _forward = 1

   net.netfilter.nf _conntrack _max = 1048576

   vm.swappiness = 10

   fs.file-max = 2097152

   fs.inotify.max _user _watches = 524288

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

cat /proc/sys/vm/dirty _ratio

 20

cat /proc/sys/vm/dirty _background _ratio

 10



 What's happening:

 Application writes dirty pages to page cache

 When dirty pages reach 10% of RAM → background flush starts

 On a 64GB server: 10% = 6.4GB of dirty data!

 The flush writes 6.4GB to disk all at once

 This overwhelms the disk → everything waiting on I/O → latency spike

 Then cache is clean again → normal for 2-3 minutes → repeat



 Fix:

sysctl -w vm.dirty _ratio=10

sysctl -w vm.dirty _background _ratio=3

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

 \[Jan 15 03:42:11] nf _conntrack: table full, dropping packet

 \[Jan 15 03:42:11] nf _conntrack: table full, dropping packet



cat /proc/sys/net/netfilter/nf _conntrack _count

 131072

cat /proc/sys/net/netfilter/nf _conntrack _max

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

sysctl -w net.netfilter.nf _conntrack _max=1048576

sysctl -w net.netfilter.nf _conntrack _tcp _timeout _established=86400

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

 With overcommit _memory=0: kernel denies the fork

 Even though COW means it would only use a fraction of that



 Fix:

sysctl -w vm.overcommit _memory=1

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

 Prometheus node _exporter exposes many of these automatically:



 Conntrack:

 node _nf _conntrack _entries        → current count

 node _nf _conntrack _entries _limit  → max limit

 Alert: entries/limit > 0.75



 Network:

 node _sockstat _TCP _tw             → TIME _WAIT socket count

 node _netstat _Tcp _ListenOverflows → listen queue overflows

 node _netstat _Tcp _ListenDrops     → listen queue drops



 Memory:

 node _memory _Dirty _bytes          → current dirty pages

 node _vmstat _pgmajfault           → major page faults (reading from disk)



 File descriptors:

 node _filefd _allocated            → current system-wide FDs

 node _filefd _maximum              → system-wide FD limit

 Alert: allocated/maximum > 0.75



 Custom check script (for parameters not in node _exporter):

  \\\\#!/bin/bash

echo "conntrack _usage $(cat /proc/sys/net/netfilter/nf _conntrack _count)"

echo "conntrack _max $(cat /proc/sys/net/netfilter/nf _conntrack _max)"

echo "somaxconn $(cat /proc/sys/net/core/somaxconn)"

echo "file _max $(cat /proc/sys/fs/file-max)"

echo "tcp _tw _reuse $(cat /proc/sys/net/ipv4/tcp _tw _reuse)"

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

 failed _when: "item.value not in result.stdout"

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



 MISTAKE 2: Setting net.ipv4.tcp _tw _recycle=1

 REMOVED from kernel 4.12+ because it breaks connections behind NAT

 Many old blog posts still recommend it. IGNORE THEM.



 MISTAKE 3: Setting kernel.shmmax too low before installing PostgreSQL

 PostgreSQL will fail to start: "could not create shared memory segment"

 Check: sysctl kernel.shmmax

 Set: sysctl -w kernel.shmmax=17179869184  (16GB)



 MISTAKE 4: Enabling ip _forward without firewall rules

 You just turned your server into a router

 Without iptables rules, traffic flows through unrestricted

 Potential security breach — anyone who can reach your server can route through it



 MISTAKE 5: Not persisting changes

 sysctl -w changes are TEMPORARY — lost on reboot

 ALWAYS write to /etc/sysctl.d/ AND run sysctl -w

 Or just write the file and run sysctl --system



 MISTAKE 6: Copy-pasting sysctl configs from the internet without understanding

 Every parameter is workload-specific

 Redis needs overcommit _memory=1, but a general server shouldn't

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

cat /sys/kernel/mm/transparent _hugepage/enabled

      \[always] madvise never



 Disable for database servers (Redis, MongoDB, PostgreSQL):

echo never > /sys/kernel/mm/transparent _hugepage/enabled

echo never > /sys/kernel/mm/transparent _hugepage/defrag



 Make persistent via systemd unit or rc.local

 Redis and MongoDB documentation explicitly say: DISABLE THP

```





📋 LESSON 8 QUICK REFERENCE — Kernel Tuning (sysctl)



```

HOW SYSCTL WORKS:

 sysctl -w key=value              → Set temporarily

 /etc/sysctl.d/XX-name.conf      → Set permanently (XX = priority)

 sysctl --system                  → Apply all config files

 sysctl -a                        → View all parameters



NETWORK TUNING:

 net.core.somaxconn = 65535                    → Accept queue size

 net.core.netdev _max _backlog = 65535          → NIC input queue

 net.ipv4.tcp _max _syn _backlog = 65535         → Half-open connection queue

 net.ipv4.tcp _tw _reuse = 1                    → Reuse TIME _WAIT sockets

 net.ipv4.ip _local _port _range = 1024 65535    → Ephemeral port range

 net.ipv4.tcp _keepalive _time = 300            → First keepalive probe

 net.ipv4.tcp _slow _start _after _idle = 0       → Maintain window after idle

 net.ipv4.tcp _fastopen = 3                    → TCP Fast Open (client+server)

 net.ipv4.tcp _syncookies = 1                  → SYN flood protection

 net.core.rmem _max/wmem _max = 16777216        → Max buffer 16MB

 net.ipv4.tcp _mtu _probing = 1                 → Path MTU discovery



MEMORY TUNING:

 vm.swappiness = 10 (server), 1 (database), 0 (Redis)

 vm.overcommit _memory = 0 (default), 1 (Redis), 2 (strict)

 vm.dirty _ratio = 10-15              → Block process above this %

 vm.dirty _background _ratio = 3-5     → Background flush above this %

 vm.min _free _kbytes = 524288         → 512MB reserved free memory

 vm.panic _on _oom = 0 or 1            → Reboot vs OOM kill

 Disable THP for databases: echo never > .../transparent _hugepage/enabled



FILESYSTEM:

 fs.file-max = 2097152               → System-wide FD limit

 fs.inotify.max _user _watches = 524288 → File watchers (K8s needs this)

 fs.inotify.max _user _instances = 8192

 fs.aio-max-nr = 1048576             → Async I/O (databases)



KUBERNETES REQUIRED:

 net.bridge.bridge-nf-call-iptables = 1

 net.ipv4.ip _forward = 1

 net.netfilter.nf _conntrack _max = 1048576

 kernel.panic = 10                    → Auto-reboot after panic



SECURITY:

 net.ipv4.conf.all.rp _filter = 1     → Anti-spoofing

 net.ipv4.conf.all.accept _redirects = 0

 net.ipv4.conf.all.send _redirects = 0

 kernel.randomize _va _space = 2        → ASLR

 kernel.dmesg _restrict = 1            → Root-only dmesg

 kernel.yama.ptrace _scope = 2         → Restrict ptrace



CONNTRACK MONITORING:

 /proc/sys/net/netfilter/nf _conntrack _count  → Current

 /proc/sys/net/netfilter/nf _conntrack _max    → Limit

 Alert when count/max > 75%

 "table full, dropping packet" = IMMEDIATE action needed



COMMON MISTAKES:

 - tcp _tw _recycle removed in kernel 4.12+ → NEVER use

 - swappiness=0 without swap = instant OOM kill

 - ip _forward=1 without firewall = open router

 - sysctl -w without writing config file = lost on reboot

 - Copy-pasting configs without understanding workload needs

```





📝 Retention Questions — Lesson 8



\*\*Q1:\*\* Your Nginx reverse proxy starts dropping connections during a traffic spike. `ss -ltn` shows `Recv-Q: 129, Send-Q: 128` for port 80. What's the exact problem and what TWO things do you need to change (one kernel, one application)?



\*\*Q2:\*\* Your PostgreSQL database has periodic latency spikes lasting 5-10 seconds every few minutes. `iostat` shows burst writes during spikes. What kernel parameter is likely misconfigured, what's happening at the kernel level, and what values would you set?



\*\*Q3:\*\* A Redis server with 20GB of data fails background save with "Cannot allocate memory" even though `free -m` shows 8GB available. What kernel parameter needs to change and WHY does the default behavior cause this failure? Explain the fork + copy-on-write mechanism.



\*\*Q4:\*\* A Kubernetes node running 80+ pods suddenly starts dropping random packets. `dmesg` shows "nf_conntrack: table full, dropping packet." What is conntrack, why did it fill up, what's your immediate fix, and how do you monitor this going forward?



\*\*Q5:\*\* An engineer applies sysctl tuning via `sysctl -w` commands during an incident and everything improves. Two weeks later the server reboots for a kernel update and all the problems return. What went wrong and what's the production-grade way to manage sysctl across a fleet of 200 servers?






