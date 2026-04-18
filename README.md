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



\*\*Go.\*\* 🎯



===========================================================================================



=====================================================================================

\---



### 🌐 PHASE 1 — NETWORKING FUNDAMENTALS



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

sysctl -w net.ipv4.tcp _mtu _probing=1

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

Format: IP/prefix _length

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

 example.com → "v=spf1 include: _spf.google.com      \~all"

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

  _http. _tcp.example.com → 10 60 8080 web1.example.com

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

 Or in $JAVA _HOME/conf/security/java.security:

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





📋 LESSONS 1-3 QUICK REFERENCE — Networking Foundations



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

 Fix: tcp _mtu _probing=1 or MSS clamping



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



📝 Retention Questions — Networking Lessons 1-3



\*\*Q1:\*\* A developer says external API calls from their Kubernetes pod take 200ms longer than expected. You check CoreDNS and it's at 95% CPU. Based on what you learned, what's the most likely cause, and give me three different ways to fix it.



\*\*Q2:\*\* After a database failover, all services recover within 60 seconds except one Java service that keeps connecting to the old IP until it's restarted. What's happening and how do you fix it permanently?



\*\*Q3:\*\* You're designing an AWS VPC for NovaMart. You need: 3 AZs, public subnets, private app subnets, private database subnets, and room for an EKS cluster with 200 pods per node. What VPC CIDR would you choose and why? How would you handle EKS pod IP exhaustion with AWS VPC CNI?



\*\*Q4:\*\* A user reports your website is down. You run `dig example.com` and get `NXDOMAIN`. But `dig @ns1.route53.aws.com example.com` returns the correct IP. What layer of the DNS chain is broken, and what's your troubleshooting sequence?



\*\*Go.\*\* 🎯

====================================================================================



#### 🌐 PHASE 1 — NETWORKING



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

 |                                |           Client moves to SYN_SENT state

 |                                |

 |  ◄── SYN-ACK (seq=300,        |  Step 2: Server says "OK, I acknowledge"

 |        ack=101) ────           |           Server moves to SYN_RECV state

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

 The client retries SYN based on tcp_syn_retries (default 6 = \\\~127 seconds!)



 SYN received but SYN-ACK never reaches client:

 → Asymmetric routing (return path is different)

 → Stateful firewall only saw outbound, not return

 → NAT translation issue



 SYN-ACK received but ACK dropped:

 → Connection stuck in SYN_RECV on server

 → SYN flood attack fills SYN_RECV queue

 → This is why tcp_syncookies exists (covered in sysctl lesson)

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

 |                                |           Client → FIN_WAIT_1

 |                                |

 |  ◄── ACK (ack=501) ────       |  Step 2: Server acknowledges

 |                                |           Client → FIN_WAIT_2

 |                                |           Server can still send data!

 |                                |

 |  ◄── FIN (seq=700) ────       |  Step 3: Server says "I'm done too"

 |                                |           Server → LAST_ACK

 |                                |

 |  ──── ACK (ack=701) ────►     |  Step 4: Client confirms

 |                                |           Client → TIME_WAIT (2×MSL)

 |                                |           Server → CLOSED

 |                                |

 |  ═══ CONNECTION CLOSED ═══     |

```



\*\*Why TIME_WAIT exists:\*\*



TIME_WAIT lasts for 2×MSL (Maximum Segment Lifetime), typically 60 seconds on Linux. It exists for two reasons:



```

1\\. DELAYED PACKETS: Old packets from this connection might still

  be in transit. If you immediately reuse the same source:dest

  port pair, those delayed packets could be confused with the

  new connection's packets → data corruption.



2\\. LOST FINAL ACK: If the final ACK (Step 4) is lost, the server

  retransmits its FIN. The client needs to be in TIME_WAIT to

  respond with another ACK. Without TIME_WAIT, the client sends

  RST → server gets a confusing error.

```



\*\*The TIME_WAIT accumulation problem (revisited with full understanding):\*\*



```bash

 On a reverse proxy making many short-lived outbound connections:

ss -tan state time-wait | wc -l

 42000



 Each TIME_WAIT socket holds:

 - A source port (from ephemeral range)

 - A 4-tuple: src_ip:src_port → dst_ip:dst_port

\\#

 You can only have ONE TIME_WAIT per unique 4-tuple

 With one destination IP:port, you're limited by source ports

 \\\~28,000 default ephemeral ports, 60s TIME_WAIT:

 Max new connections: 28000/60 = \\\~467/second to a SINGLE destination

\\#

 If you need more:

 1. tcp_tw_reuse=1 (reuse TIME_WAIT for outbound, uses timestamps for safety)

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

SYN_SENT        SYN sent, waiting for SYN-ACK               Client

SYN_RECV        SYN received, SYN-ACK sent, waiting ACK     Server

ESTABLISHED     Connection active, data flowing              Both

FIN_WAIT_1      FIN sent, waiting for ACK                   Closer (initiator)

FIN_WAIT_2      FIN acknowledged, waiting for peer's FIN    Closer

CLOSE_WAIT      Received FIN, waiting for app to close      Receiver (DANGEROUS)

LAST_ACK        FIN sent, waiting for final ACK             Receiver

TIME_WAIT       Waiting 2×MSL before fully closing          Closer

CLOSING         Both sides sent FIN simultaneously (rare)   Both

CLOSED          Connection fully terminated                 N/A

```



\### CLOSE_WAIT — The State That Tells You Your App Is Buggy



```bash

 CLOSE_WAIT means:

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



 If CLOSE_WAIT sockets accumulate:

 - File descriptors exhausted (EMFILE)

 - Memory leaked (each socket has buffers)

 - Application eventually crashes



 CLOSE_WAIT never times out on its own

 It stays until the application closes the socket or the process dies

 This is fundamentally different from TIME_WAIT which auto-expires



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

 Window scaling multiplies by 2^scale_factor

 Scale factor negotiated during handshake

 Max window with scaling: 64KB × 2^14 = 1GB

 This is why tcp_window_scaling=1 in sysctl is critical

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

 And why tcp_slow_start_after_idle=0 matters (covered in sysctl)

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

sysctl net.ipv4.tcp_congestion_control

 cubic



 Switch to BBR:

sysctl -w net.core.default_qdisc=fq

sysctl -w net.ipv4.tcp_congestion_control=bbr



 Available algorithms:

sysctl net.ipv4.tcp_available_congestion_control

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

    tcp_keepalive_time=30 < ALB timeout=60

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

echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

 notBefore=Jan  1 00:00:00 2024 GMT

 notAfter=Apr  1 00:00:00 2024 GMT



 Check from command line:

curl -vI https://example.com 2>\\\&1 | grep "expire"



 Monitoring:

 Prometheus blackbox_exporter probes HTTPS endpoints:

 probe_ssl_earliest_cert_expiry — timestamp of cert expiry

 Alert when: (probe_ssl_earliest_cert_expiry - time()) < 30\\\*86400

 "Certificate expires in less than 30 days"



 Prevention:

 1. Use ACM for AWS resources (auto-renews)

 2. Use cert-manager in Kubernetes (auto-renews)

 3. Use certbot with auto-renewal timer

 4. Monitor ALL certificates with Prometheus/blackbox_exporter

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

openssl s_client -tls1_2 -connect example.com:443

openssl s_client -tls1_3 -connect example.com:443



 Nginx TLS configuration:

ssl_protocols TLSv1.2 TLSv1.3;

ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';

ssl_prefer_server_ciphers on;



 ALB: Supports TLS 1.0-1.3 via security policies

 ELBSecurityPolicy-TLS13-1-2-2021-06 → TLS 1.2 and 1.3 only

 Apply via Terraform:

resource "aws_lb_listener" "https" {

 ssl_policy = "ELBSecurityPolicy-TLS13-1-2-2021-06"

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

keepalive_timeout 55;  # < ALB's 60s



 Or increase ALB timeout:

resource "aws_lb" "main" {

 idle_timeout = 120  # seconds

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

 Used by: HAProxy, Nginx (least_conn)



IP HASH:

 Hash the client IP to select a server

 Same client always goes to same server (poor man's sticky sessions)

 Used by: Nginx (ip_hash), when session affinity needed without cookies



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

resource "aws_lb_target_group" "api" {

 health_check {

 path                = "/health"

 port                = "8080"

 protocol            = "HTTP"

 healthy_threshold   = 3    # Consecutive successes to mark healthy

 unhealthy_threshold = 2    # Consecutive failures to mark unhealthy

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



upstream api_backends {

 # Load balancing algorithm

 least_conn;

 

 # Backend servers

 server 10.0.1.10:8080 weight=3;

 server 10.0.1.11:8080 weight=3;

 server 10.0.1.12:8080 weight=1;  # Canary (less traffic)

 server 10.0.1.13:8080 backup;     # Only used if all others are down

 

 # Keep-alive connection pool to backends

 keepalive 64;

 keepalive_timeout 60s;

 keepalive_requests 1000;

}



server {

 listen 80;

 listen 443 ssl http2;

 server_name api.example.com;

 

 ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;

 ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

 ssl_protocols TLSv1.2 TLSv1.3;

 

 # Proxy settings

 location /api/ {

     proxy_pass http://api_backends;

     proxy_http_version 1.1;

     proxy_set_header Connection "";  # Enable keepalive to upstream

     proxy_set_header Host $host;

     proxy_set_header X-Real-IP $remote_addr;

     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

     proxy_set_header X-Forwarded-Proto $scheme;

     proxy_set_header X-Request-Id $request_id;

     

     # Timeouts

     proxy_connect_timeout 5s;    # Time to establish connection to backend

     proxy_send_timeout 30s;      # Time to send request to backend

     proxy_read_timeout 60s;      # Time to receive response from backend

     

     # Buffering

     proxy_buffering on;

     proxy_buffer_size 8k;

     proxy_buffers 32 8k;

     

     # Retries

     proxy_next_upstream error timeout http_502 http_503;

     proxy_next_upstream_tries 2;

     proxy_next_upstream_timeout 10s;

 }

 

 # Health check endpoint (don't proxy this)

 location /health {

     access_log off;

     return 200 'OK';

     add_header Content-Type text/plain;

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



📋 LESSONS 4-6 QUICK REFERENCE



```

TCP HANDSHAKE: SYN → SYN-ACK → ACK (3-way)

TCP TEARDOWN: FIN → ACK → FIN → ACK (4-way)

TIME_WAIT: 60s on Linux, exists to prevent delayed packet confusion



TCP STATES TO WATCH:

 CLOSE_WAIT = YOUR app isn't closing sockets (BUG)

 TIME_WAIT = normal, but accumulation = port exhaustion

 SYN_RECV = normal, but thousands = possible SYN flood



CONGESTION CONTROL:

 Cubic (default), BBR (Google, 20-40% better on lossy networks)

 Switch: sysctl net.ipv4.tcp_congestion_control=bbr



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

 Monitor: probe_ssl_earliest_cert_expiry, alert at 30 days



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

 upstream + least_conn + keepalive pool

 proxy_next_upstream for automatic retry on 502/503

 keepalive_timeout < ALB idle timeout

```



\---



📝 Retention Questions — Lessons 4-6



\*\*Q1:\*\* You see 5,000 connections in `CLOSE_WAIT` state on your application server. Is this a kernel issue or an application issue? What exactly is happening and what do you do?



\*\*Q2:\*\* Your service experiences intermittent "Connection reset by peer" errors, but only during low-traffic periods (evenings/weekends). During peak hours, everything is fine. What's the most likely cause and how do you fix it with the timeout hierarchy principle?



\*\*Q3:\*\* During a rolling deployment on EKS, users experience 502 errors for approximately 5 seconds per pod replacement. All new pods pass health checks and come up healthy. Based on everything you've learned across ALL lessons so far, explain the full chain of events causing this and give the complete fix.



\*\*Q4:\*\* Your company uses Nginx as a reverse proxy in front of 10 backend servers. During peak traffic, Nginx starts returning 502 errors even though all backends are running. `ss -tan | grep ESTAB | wc -l` on Nginx shows 50,000 connections. What's happening and what TWO things need to be tuned?



\*\*Go.\*\* 🎯

=======================================================================================================

💯 PERFECT SWEEP. 4/4.



Every answer was precise, well-structured, and demonstrated deep understanding across multiple lessons. You're connecting concepts from Linux (file descriptors, signals, sysctl) with networking (TCP states, timeouts, load balancing) naturally. That's systems thinking.



One nuance on Q2 — the timeout hierarchy has two dimensions:



```

REQUEST/RESPONSE TIMEOUTS (who gives up first):

Client timeout > LB timeout > Backend timeout

"Each layer should timeout BEFORE the one in front of it"

So the backend gives up first → clean error to LB → clean error to client



IDLE/KEEPALIVE TIMEOUTS (who closes idle connections first):

Backend keepalive < LB idle timeout < Client keepalive

"Backend closes idle connections BEFORE the LB silently drops them"

This prevents the RST problem you described

```



Your answer was correct in principle — you want the application to close before the middlebox kills silently. Just be precise about which timeout dimension you're discussing in interviews.



\---



🌐 PHASE 1 — NETWORKING



##### \## Lesson 7: Firewalls, Security Groups, NACLs \& iptables



\### iptables — The Linux Firewall Engine



Everything in Linux network security ultimately flows through \*\*Netfilter\*\*, the kernel's packet filtering framework. `iptables` is the userspace tool to configure it. Even in cloud environments, Security Groups and NACLs are ultimately implemented via similar mechanisms.



\### iptables Architecture — Tables and Chains



```

TABLES (what kind of processing):

filter   → Accept/drop/reject packets (DEFAULT table)

nat      → Network Address Translation (SNAT, DNAT, masquerade)

mangle   → Modify packet headers (TTL, TOS, marking)

raw      → Bypass connection tracking (high-performance exceptions)

security → SELinux/MAC rules



CHAINS (when in the packet's journey):

PREROUTING  → Packet just arrived, before routing decision

INPUT       → Packet destined for THIS machine

FORWARD     → Packet passing THROUGH this machine (routing)

OUTPUT      → Packet generated BY this machine

POSTROUTING → Packet about to leave, after routing decision



PACKET FLOW:

Incoming packet → PREROUTING → routing decision

  → If for this host: INPUT → local process

  → If for another host: FORWARD → POSTROUTING → out



Outgoing packet → OUTPUT → routing decision → POSTROUTING → out

```



\### iptables Rule Syntax:



```bash

Basic format:

iptables -t <table> -A <chain> <match-criteria> -j <target>



TARGETS (actions):

ACCEPT  → Allow the packet

DROP    → Silently discard (no response — client times out)

REJECT  → Discard and send error back (client gets "connection refused")

LOG     → Log the packet and continue processing

DNAT    → Destination NAT (change destination IP/port)

SNAT    → Source NAT (change source IP)

MASQUERADE → Dynamic SNAT (for outbound internet via NAT gateway)



EXAMPLES:



Allow incoming SSH

iptables -A INPUT -p tcp --dport 22 -j ACCEPT



Allow incoming HTTP/HTTPS

iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT



Allow established/related connections (stateful)

iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT



Allow from specific CIDR

iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 8080 -j ACCEPT



Drop everything else (default deny)

iptables -A INPUT -j DROP



Block a specific IP

iptables -I INPUT 1 -s 198.51.100.5 -j DROP

-I INPUT 1 = Insert at position 1 (top of chain, processed first)



NAT — masquerade outbound traffic (for NAT gateway)

iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE



Port forwarding (DNAT)

iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.1.5:8080



View rules

iptables -L -n -v          # List all rules with packet counts

iptables -L -n -v --line-numbers  # With line numbers (for deletion)

iptables -t nat -L -n -v   # NAT table rules



Delete a rule

iptables -D INPUT 3         # Delete rule number 3 in INPUT chain



Flush all rules (CAREFUL!)

iptables -F                  # Flush filter table

iptables -t nat -F          # Flush nat table



Save rules (persists across reboot)

iptables-save > /etc/iptables/rules.v4

Restore

iptables-restore < /etc/iptables/rules.v4



On systemd systems, use iptables-persistent package:

apt install iptables-persistent

netfilter-persistent save

```



\### iptables Rule Processing — ORDER MATTERS:



```bash

Rules are processed TOP to BOTTOM

First match wins — remaining rules are SKIPPED



WRONG ORDER:

iptables -A INPUT -j DROP            # Rule 1: Drop everything

iptables -A INPUT -p tcp --dport 22 -j ACCEPT  # Rule 2: Allow SSH

SSH is blocked because Rule 1 matches first!



CORRECT ORDER:

iptables -A INPUT -p tcp --dport 22 -j ACCEPT  # Rule 1: Allow SSH

iptables -A INPUT -j DROP            # Rule 2: Drop everything else

SSH works because Rule 1 matches first



This is why -I (insert at top) is used for emergency blocks:

iptables -I INPUT 1 -s <attacker-ip> -j DROP

Inserted at position 1, before all other rules

```



\### Why iptables Matters for DevOps:



```bash

1. Kubernetes uses iptables (or IPVS) for Service routing

   kube-proxy creates iptables rules for every Service

   Every ClusterIP, NodePort, LoadBalancer = iptables rules



View Kubernetes-created iptables rules:

iptables -t nat -L KUBE-SERVICES -n

Shows all Service → endpoint mappings



2. Docker uses iptables for port mapping

docker run -p 8080:80 creates:

iptables -t nat -L DOCKER -n

DNAT rule: host:8080 → container-ip:80



3. Calico (K8s CNI) uses iptables for NetworkPolicies

Every NetworkPolicy = iptables rules on the node



4. Debugging connectivity issues often requires reading iptables

"Why can't pod A reach pod B?"

iptables -t filter -L -n -v | grep <pod-ip>

Check if any DROP/REJECT rules match

```



\### nftables — The Successor to iptables:



```bash

nftables is the modern replacement for iptables

Better performance, cleaner syntax, unified framework

Most modern distros include both, with iptables as a compatibility layer



Check if your system uses iptables or nftables backend:

iptables -V

iptables v1.8.7 (nf_tables)  ← nftables backend

iptables v1.8.7 (legacy)     ← real iptables



For DevOps: understand both, but iptables knowledge is more universal

Kubernetes and Docker still primarily use iptables interface

```



\---



\### AWS Security Groups — Stateful Instance-Level Firewall



```bash

Security Groups (SGs) are attached to ENIs (Elastic Network Interfaces)

Every EC2 instance, RDS instance, Lambda in VPC, EKS pod gets one



KEY PROPERTIES:

✅ STATEFUL — if inbound is allowed, return traffic is automatic

✅ Allow rules ONLY — you cannot create deny rules

✅ All rules evaluated (not ordered — if ANY rule allows, traffic passes)

✅ Default: allow ALL outbound, deny ALL inbound

✅ Can reference other Security Groups as source/destination



EXAMPLE — Web application architecture:



SG: alb-sg (for Application Load Balancer)

Inbound:

  TCP 443 from 0.0.0.0/0        (HTTPS from internet)

  TCP 80 from 0.0.0.0/0         (HTTP from internet — redirect to HTTPS)

Outbound:

  All traffic (default)



SG: app-sg (for application EC2/EKS)

Inbound:

  TCP 8080 from alb-sg           (only ALB can reach app port)

  TCP 22 from bastion-sg         (SSH only from bastion)

Outbound:

  All traffic



SG: db-sg (for RDS)

Inbound:

  TCP 5432 from app-sg           (only app can reach database)

Outbound:

  All traffic



SG: bastion-sg (for jump host)

Inbound:

  TCP 22 from <office-cidr>/32   (SSH only from office IP)

Outbound:

  All traffic



REFERENCING OTHER SGs:

"TCP 8080 from alb-sg" means:

"Allow TCP 8080 from any ENI that has alb-sg attached"

This is dynamic — if ALB scales, new IPs are automatically allowed

No need to update IP ranges manually

```



\### AWS NACLs — Stateless Subnet-Level Firewall



```bash

NACLs (Network Access Control Lists) operate at the SUBNET level

Every packet entering or leaving a subnet passes through the NACL



KEY PROPERTIES:

❌ STATELESS — return traffic must be explicitly allowed

✅ Allow AND deny rules

✅ Rules processed IN ORDER by rule number (first match wins)

✅ Default NACL: allows all traffic

✅ Custom NACL: denies all traffic by default



NACL vs Security Group:



Feature          | Security Group      | NACL

Level            | Instance (ENI)      | Subnet

State            | Stateful            | Stateless

Rules            | Allow only          | Allow + Deny

Processing       | All rules evaluated | First match wins (ordered)

Default inbound  | Deny all            | Allow all (default NACL)

Return traffic   | Automatic           | Must explicitly allow



NACL RULE EXAMPLE:

Rule# | Type  | Protocol | Port     | Source        | Allow/Deny

100   | HTTP  | TCP      | 80       | 0.0.0.0/0    | ALLOW

110   | HTTPS | TCP      | 443      | 0.0.0.0/0    | ALLOW

120   | SSH   | TCP      | 22       | 10.0.0.0/8   | ALLOW

200   | Custom| TCP      | 1024-65535| 0.0.0.0/0   | ALLOW  ← EPHEMERAL PORTS!

\*     | All   | All      | All      | 0.0.0.0/0    | DENY   ← Default deny



THE EPHEMERAL PORT TRAP:

Because NACLs are STATELESS, you must allow return traffic

When your server responds to an HTTP request, the RESPONSE

goes to the CLIENT'S ephemeral port (1024-65535)

If your OUTBOUND NACL doesn't allow high ports → responses blocked

If your INBOUND NACL doesn't allow high ports → return traffic blocked

This catches people ALL THE TIME



BEST PRACTICE:

Use Security Groups as your PRIMARY firewall (easier, stateful)

Use NACLs as a COARSE secondary layer:

  - Block known bad CIDR ranges

  - Subnet-level isolation between tiers

  - Emergency: block an attacking IP at subnet level

Don't try to replicate SG rules in NACLs — it's painful and error-prone

```



\---



\### Production Scenario: Emergency IP Block During DDoS



```bash

Situation: Your WAF is being overwhelmed by traffic from a specific CIDR

You need to block it NOW, at the lowest level possible



Option 1: NACL (fastest, broadest)

Block at subnet level — packets never reach instances

aws ec2 create-network-acl-entry \\

--network-acl-id acl-12345 \\

--rule-number 50 \\

--protocol -1 \\

--rule-action deny \\

--ingress \\

--cidr-block 198.51.100.0/24



Option 2: Security Group (can't deny, so not useful here)

SGs only have allow rules — can't block specific IPs



Option 3: WAF rule

Block at Layer 7 — more granular but higher in the stack

aws wafv2 update-ip-set --name "blocked-ips" \\

--addresses "198.51.100.0/24"



Option 4: Cloudflare/CDN level

Block before traffic even reaches AWS — most effective for DDoS



Order of defense (outermost to innermost):

Cloudflare → AWS Shield → WAF → NACL → Security Group → iptables

```



\---



##### \## Lesson 8: VPC Networking — The AWS Network Architecture



\### VPC Components — Complete Picture



```

VPC (Virtual Private Cloud):

Your isolated network in AWS

Defined by a CIDR block (e.g., 10.0.0.0/16)

Spans ALL AZs in a region



SUBNETS:

Segments of the VPC CIDR

Each subnet lives in ONE AZ

Public subnet: has route to Internet Gateway

Private subnet: no direct internet access



INTERNET GATEWAY (IGW):

Connects VPC to the internet

1:1 per VPC

Allows instances with public IPs to reach/be reached from internet



NAT GATEWAY:

Allows PRIVATE instances to reach the internet (outbound only)

Lives in a PUBLIC subnet

Has an Elastic IP

Used for: package updates, API calls, pulling container images

NO inbound connections from internet (one-way door)

Cost: \~$32/month + data processing charges

Best practice: one per AZ for high availability



ROUTE TABLES:

Rules that determine where network traffic goes

Each subnet is associated with ONE route table



# Public subnet route table:

Destination       Target

10.0.0.0/16      local          (VPC internal traffic)

0.0.0.0/0        igw-12345      (internet via IGW)



# Private subnet route table:

Destination       Target

10.0.0.0/16      local

0.0.0.0/0        nat-12345      (internet via NAT Gateway)



ELASTIC IP:

Static public IPv4 address

Persists across instance stop/start

Attached to NAT Gateway or instance ENI

Free while attached to running instance

Charged when NOT in use ($3.65/month since Feb 2024)

```



\### VPC Peering vs Transit Gateway:



```

VPC PEERING:

Direct connection between two VPCs

Works across accounts and regions

Non-transitive: A↔B and B↔C does NOT mean A↔C

No bandwidth limit (uses AWS backbone)

Free within same AZ, $0.01/GB cross-AZ

Max: limited by number of routes in route table



Use for: Simple 2-3 VPC connections



# Route table entry for peered VPC:

Destination       Target

172.16.0.0/16    pcx-12345      (peering connection)



TRANSIT GATEWAY:

Hub-and-spoke model — central router for multiple VPCs

Transitive routing: A→TGW→B→TGW→C all works

Supports: VPCs, VPN, Direct Connect, cross-region

Scales to thousands of VPCs

$0.05/hour + $0.02/GB



Use for: Enterprise with many VPCs, hybrid cloud



NovaMart architecture:

┌─────────┐    ┌──────────────┐    ┌─────────┐

│ Prod VPC │────│Transit Gateway│────│ Dev VPC  │

└─────────┘    └──────┬───────┘    └─────────┘

                      │

               ┌──────▼───────┐

               │ Shared Svc   │

               │ VPC (logging,│

               │ monitoring)  │

               └──────────────┘

```



\### VPN and Direct Connect:



```

SITE-TO-SITE VPN:

Encrypted tunnel over public internet

AWS VPC ←→ On-premises network

Uses IPsec

\~1.25 Gbps per tunnel (2 tunnels for HA)

Cost: $0.05/hour per VPN connection

Setup time: minutes

Latency: variable (internet-dependent)



Use for: Quick hybrid connectivity, backup to Direct Connect



AWS DIRECT CONNECT:

Dedicated physical connection to AWS

1 Gbps or 10 Gbps ports

Consistent latency, consistent bandwidth

Cost: $0.03/GB + port hours

Setup time: weeks to months (physical installation)



Use for: Large data transfers, latency-sensitive workloads,

         compliance (traffic doesn't traverse public internet)



NovaMart: Direct Connect primary, VPN as backup

```



\### VPC Endpoints — Private Access to AWS Services:



```bash

Without VPC Endpoint:

EC2 in private subnet → NAT Gateway → Internet → S3

Problem: Traffic goes over internet, costs NAT Gateway data fees



With VPC Endpoint:

EC2 in private subnet → VPC Endpoint → S3

Traffic stays within AWS network, no NAT Gateway needed



TWO TYPES:



Gateway Endpoint (free!):

  Supported: S3, DynamoDB ONLY

  Adds route to route table

  No security group needed

resource "aws_vpc_endpoint" "s3" {

vpc_id       = aws_vpc.main.id

service_name = "com.amazonaws.us-east-1.s3"

vpc_endpoint_type = "Gateway"

route_table_ids = \[aws_route_table.private.id]

}



Interface Endpoint (costs money):

  Supported: Almost all other AWS services (ECR, SQS, SNS, CloudWatch, etc.)

  Creates an ENI in your subnet

  Accessible via private DNS

  $0.01/hour + $0.01/GB

resource "aws_vpc_endpoint" "ecr" {

vpc_id             = aws_vpc.main.id

service_name       = "com.amazonaws.us-east-1.ecr.dkr"

vpc_endpoint_type  = "Interface"

subnet_ids         = \[aws_subnet.private_a.id, aws_subnet.private_b.id]

security_group_ids = \[aws_security_group.vpc_endpoint.id]

private_dns_enabled = true

}



CRITICAL FOR EKS:

EKS nodes in private subnets need to pull images from ECR

Without VPC endpoint: traffic goes through NAT Gateway = expensive

ECR pull for 200 pods × 500MB images = significant data costs

Interface endpoints for ECR, S3 (for ECR layers), CloudWatch, STS

save thousands of dollars per month on NAT data processing

```



\---



\### Production Scenario: EKS Nodes Can't Pull Images



```bash

Symptoms:

Pods stuck in ImagePullBackOff

"failed to pull image: timeout"

Only happens in private subnets



Investigation:

1. Check if NAT Gateway exists and is healthy

aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=vpc-12345"

State: available ✓



2. Check route table for private subnet

aws ec2 describe-route-tables --route-table-ids rtb-12345

0.0.0.0/0 → nat-12345 ✓



3. Check NAT Gateway's subnet has route to IGW

NAT GW must be in a PUBLIC subnet with IGW route

Common mistake: putting NAT GW in private subnet



4. Check Security Groups

NAT Gateway doesn't have a SG, but the instances do

Instance SG outbound must allow HTTPS (443) to 0.0.0.0/0



5. Check NACLs

Outbound: allow TCP 443 to 0.0.0.0/0

Inbound: allow TCP 1024-65535 from 0.0.0.0/0 (return traffic!)

^^^ THE EPHEMERAL PORT TRAP AGAIN



6. Alternative fix: VPC Endpoints

Deploy Interface Endpoints for ECR, S3, STS

Remove dependency on NAT Gateway for AWS service access

Faster, cheaper, more reliable

```



\---



##### \## Lesson 9: Network Debugging Tools — Your Investigation Toolkit



\### tcpdump — The Packet-Level X-Ray



```bash

tcpdump captures raw packets on a network interface

It's the most powerful network debugging tool



Basic capture:

tcpdump -i eth0                    # All traffic on eth0

tcpdump -i any                     # All interfaces



Filter by host:

tcpdump -i eth0 host 10.0.1.5     # Traffic to/from this IP

tcpdump -i eth0 src 10.0.1.5      # Only FROM this IP

tcpdump -i eth0 dst 10.0.1.5      # Only TO this IP



Filter by port:

tcpdump -i eth0 port 80           # HTTP traffic

tcpdump -i eth0 port 443          # HTTPS traffic

tcpdump -i eth0 port 5432         # PostgreSQL

tcpdump -i eth0 portrange 8080-8090



Filter by protocol:

tcpdump -i eth0 tcp               # TCP only

tcpdump -i eth0 udp               # UDP only

tcpdump -i eth0 icmp              # ICMP (ping) only



Combine filters:

tcpdump -i eth0 'host 10.0.1.5 and port 80'

tcpdump -i eth0 'src 10.0.1.5 and dst port 443'

tcpdump -i eth0 '(port 80 or port 443) and host 10.0.1.5'



Capture TCP flags:

tcpdump -i eth0 'tcp\[tcpflags] \& tcp-syn != 0'    # SYN packets

tcpdump -i eth0 'tcp\[tcpflags] \& tcp-rst != 0'    # RST packets

tcpdump -i eth0 'tcp\[tcpflags] \& tcp-fin != 0'    # FIN packets



Useful flags:

tcpdump -n               # Don't resolve hostnames (faster)

tcpdump -nn              # Don't resolve hostnames or ports

tcpdump -v / -vv / -vvv  # Increasing verbosity

tcpdump -c 100           # Capture only 100 packets then stop

tcpdump -w capture.pcap  # Write to file (for Wireshark analysis)

tcpdump -r capture.pcap  # Read from file

tcpdump -A               # Print packet content in ASCII

tcpdump -X               # Print packet content in hex + ASCII

tcpdump -s 0             # Capture full packet (not just header)



##### REAL-WORLD DEBUGGING EXAMPLES:



"Is DNS working?"

tcpdump -i eth0 -nn port 53

Shows DNS queries and responses with domain names



"Are SYN packets reaching my server?"

tcpdump -i eth0 -nn 'tcp\[tcpflags] == tcp-syn and dst port 8080'

If SYNs arrive but no SYN-ACKs → app not listening or firewall



"Why are connections being reset?"

tcpdump -i eth0 -nn 'tcp\[tcpflags] \& tcp-rst != 0'

Capture RST packets — see who's sending them and why



"What's the actual HTTP request being sent?"

tcpdump -i eth0 -A 'dst port 80' | grep -E 'GET|POST|HTTP|Host:'

Shows HTTP method, path, host header in plain text



"Capture 30 seconds of traffic for later analysis"

timeout 30 tcpdump -i eth0 -w /tmp/debug.pcap -s 0 'host 10.0.1.5'

Then download and open in Wireshark

```



##### \### traceroute / mtr — Path Analysis



```bash

traceroute shows every hop between you and the destination

traceroute 8.8.8.8

1  10.0.0.1 (gateway)     1.2ms  1.1ms  1.3ms

2  172.16.0.1              5.2ms  5.1ms  5.3ms

3  \* \* \*                           ← hop doesn't respond (ICMP blocked)

4  209.85.248.1            15.2ms 15.1ms 15.3ms

5  8.8.8.8                 16.0ms 15.9ms 16.1ms



Each line = one router/hop

Three times = three probe responses (latency variation)

\* \* \* = hop blocks ICMP or traceroute probes



Use TCP-based traceroute for more reliable results:

traceroute -T -p 443 example.com

Uses TCP SYN instead of ICMP — less likely to be blocked



mtr — traceroute on steroids (continuous monitoring)

mtr 8.8.8.8

Real-time continuous traceroute

Shows: packet loss %, avg/best/worst latency per hop

Invaluable for detecting intermittent network issues

\#

HOST                Loss%  Snt   Last   Avg  Best  Wrst

1. gateway          0.0%   50    1.2    1.3   1.0   2.1

2. isp-router       0.0%   50    5.1    5.2   4.8   6.3

3. backbone         2.0%   50   15.2   15.5  14.9  25.3  ← packet loss!

4. destination       2.0%   50   16.1   16.3  15.8  26.1



If loss appears at hop 3 and continues → problem is AT hop 3

If loss appears at hop 3 but NOT at hop 4 → hop 3 just drops ICMP (normal)



Generate report for sharing with ISP/network team:

mtr -rw -c 100 example.com > mtr_report.txt

-r = report mode

-w = wide (show full hostnames)

-c = count of pings

```



##### \### ss and netstat — Socket Statistics



```bash

ss is the modern replacement for netstat (faster, more info)



List all TCP connections

ss -tan



List all listening TCP ports

ss -tlnp

-t = TCP

-l = listening

-n = numeric (don't resolve names)

-p = show process



List all UDP sockets

ss -uanp



Filter by state

ss -tan state established

ss -tan state time-wait

ss -tan state close-wait



Filter by port

ss -tan 'sport = :8080'        # Source port 8080

ss -tan 'dport = :5432'        # Destination port 5432



Filter by address

ss -tan 'dst 10.0.1.5'



Connection summary

ss -s

Total: 15234 (kernel 15500)

TCP:   12345 (estab 8765, closed 1234, orphaned 12, timewait 2345)

UDP:   56



Per-socket TCP info (retransmissions, RTT, cwnd)

ss -tin

Shows: rto, rtt, cwnd, retrans for each connection

Invaluable for debugging slow connections



Find which process is using a port

ss -tlnp | grep :8080

LISTEN  0  128  \*:8080  \*:\*  users:(("java",pid=1234,fd=56))

```



##### \### curl — The HTTP Swiss Army Knife



```bash

Basic request

curl https://api.example.com/health



Verbose (show TLS handshake, headers, everything)

curl -vvv https://api.example.com/health



Only headers

curl -I https://api.example.com/health



With timing breakdown

curl -o /dev/null -s -w "\

DNS:        %{time_namelookup}s\n \

Connect:    %{time_connect}s\n \

TLS:        %{time_appconnect}s\n \

TTFB:       %{time_starttransfer}s\n \

Total:      %{time_total}s\n \

HTTP Code:  %{http_code}\n \

Size:       %{size_download} bytes\n" \

https://api.example.com/health



Output:

DNS:        0.012s        ← DNS resolution time

Connect:    0.045s        ← TCP handshake complete

TLS:        0.123s        ← TLS handshake complete

TTFB:       0.234s        ← Time to first byte (server processing)

Total:      0.250s        ← Total request time

HTTP Code:  200

Size:       15 bytes



If DNS is slow → DNS issue

If Connect-DNS is slow → network latency

If TLS-Connect is slow → TLS issue (cert validation, cipher)

If TTFB-TLS is slow → server is slow processing

This breakdown tells you EXACTLY where the latency is



Custom headers

curl -H "Authorization: Bearer token123" https://api.example.com



POST with JSON body

curl -X POST -H "Content-Type: application/json" \\

-d '{"name":"test"}' https://api.example.com/users



Follow redirects

curl -L https://example.com



Insecure (skip TLS verification — for debugging only!)

curl -k https://self-signed.example.com



Timeout

curl --connect-timeout 5 --max-time 30 https://api.example.com



Resolve to specific IP (bypass DNS — test before migration)

curl --resolve api.example.com:443:10.0.1.5 https://api.example.com

Sends request to 10.0.1.5 but uses api.example.com for TLS SNI and Host header

Perfect for testing a new server before updating DNS

```



\---



\### Production Scenario: Systematic Network Debugging Framework



```bash

A developer says: "My service can't reach the database"

Service: app pod in EKS

Database: RDS PostgreSQL at db.internal.example.com:5432



##### LAYER-BY-LAYER DEBUGGING:



STEP 1: DNS Resolution (Layer 7)

kubectl exec -it app-pod -- nslookup db.internal.example.com

Does it resolve? To what IP?

If NXDOMAIN → DNS issue (CoreDNS, Route53 private zone)

If resolves → continue



STEP 2: Network Reachability (Layer 3)

kubectl exec -it app-pod -- ping -c 3 <db-ip>

Note: ICMP might be blocked — don't rely solely on ping

If works → Layer 3 is fine

If fails → routing issue, Security Group blocking ICMP



STEP 3: Port Reachability (Layer 4)

kubectl exec -it app-pod -- nc -zv <db-ip> 5432

Or: kubectl exec -it app-pod -- timeout 5 bash -c 'echo > /dev/tcp/<db-ip>/5432'

"Connection succeeded" → TCP port is open, firewall allows it

"Connection refused" → port is open but nothing listening

"Connection timed out" → firewall/SG blocking or no route



STEP 4: If timeout, check Security Groups

App pod → runs on node → node has SG

Check: node SG allows outbound TCP 5432?

Check: RDS SG allows inbound TCP 5432 from node SG or node CIDR?



STEP 5: If SGs are correct, check NACLs

Private subnet NACL allows outbound TCP 5432?

DB subnet NACL allows inbound TCP 5432?

DB subnet NACL allows outbound TCP 1024-65535 (return traffic)?



STEP 6: If NACLs are correct, check routing

Does the pod's node have a route to the DB subnet?

Are they in the same VPC? Peered VPCs? Transit Gateway?



STEP 7: Application layer

If TCP connection succeeds but app still fails:

Wrong credentials? Wrong database name? SSL required?

kubectl exec -it app-pod -- psql -h <db-ip> -U myuser -d mydb

This gives you the actual PostgreSQL error



THE CHECKLIST:

1. DNS resolves correctly?

2. TCP connection succeeds on the port?

3. Security Groups allow the traffic?

4. NACLs allow the traffic (both directions)?

5. Route tables have correct routes?

6. Application-level auth/config correct?

```



\---



📋 LESSONS 7-9 QUICK REFERENCE



```

IPTABLES:

Tables: filter (default), nat, mangle, raw

Chains: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING

First match wins (ORDER MATTERS)

iptables -L -n -v --line-numbers    → View rules

iptables -I INPUT 1 -s <ip> -j DROP → Emergency block

K8s uses iptables for Service routing (kube-proxy)



SECURITY GROUPS:

Stateful, allow-only, all rules evaluated

Reference other SGs as source (dynamic!)

Default: deny inbound, allow outbound

Primary firewall for AWS



NACLs:

Stateless, allow+deny, ordered by rule number

Must allow ephemeral ports (1024-65535) for return traffic

Use as coarse secondary layer, not primary firewall

First match wins (like iptables)



VPC:

IGW = internet access for public subnets

NAT GW = outbound internet for private subnets (in public subnet!)

VPC Peering = direct, non-transitive, free same-AZ

Transit Gateway = hub-and-spoke, transitive, enterprise

VPN = encrypted over internet, quick setup

Direct Connect = dedicated physical, consistent, weeks to setup



VPC ENDPOINTS:

Gateway (free): S3, DynamoDB

Interface ($): ECR, SQS, SNS, CloudWatch, STS, etc.

Critical for EKS in private subnets (save NAT costs)



DEBUGGING TOOLS:

tcpdump -i eth0 -nn 'host X and port Y'     → Packet capture

tcpdump 'tcp\[tcpflags] \& tcp-rst != 0'      → Find RSTs

mtr <host>                                    → Continuous traceroute

ss -tan state close-wait                      → Find socket leaks

ss -tin                                       → Per-connection TCP stats

curl -w timing format                         → HTTP timing breakdown

curl --resolve domain:port:ip                 → Test before DNS change



SYSTEMATIC DEBUGGING:

1. DNS → 2. Ping/ICMP → 3. TCP port → 4. Security Groups →

5. NACLs → 6. Route tables → 7. Application layer



DEFENSE LAYERS (outer to inner):

CDN/Cloudflare → AWS Shield → WAF → NACL → Security Group → iptables

```



\---



📝 Retention Questions — Lessons 7-9



\*\*Q1:\*\* An EKS pod in a private subnet can reach other pods and services inside the cluster, but cannot pull images from ECR. `kubectl describe pod` shows `ImagePullBackOff`. NAT Gateway exists and is in a public subnet. Security Group allows all outbound. What's the most likely cause that people miss, and what's the cost-effective long-term fix?



\*\*Q2:\*\* During a DDoS attack, you need to block the CIDR `198.51.100.0/24` at the lowest possible level in AWS. Can you use a Security Group? Why or why not? What do you use instead, and why is the rule number important?



\*\*Q3:\*\* You run `curl -w` timing breakdown against your API and see: DNS=0.001s, Connect=0.002s, TLS=0.003s, TTFB=5.2s, Total=5.3s. Where is the bottleneck and what does this tell you about where to investigate?



\*\*Q4:\*\* A developer's service gets intermittent `Connection reset by peer` when talking to RDS through an NLB. The resets only happen on connections that have been idle for about 350 seconds. The NLB idle timeout is 350 seconds. TCP keepalive on the application is set to 7200 seconds. Explain the full chain of events and give the fix using the timeout hierarchy.



\*\*Go.\*\* 🎯

# 🌐 PHASE 1 — NETWORKING

## Lesson 10: Kubernetes Networking Internals

This is where everything we've learned comes together — Linux networking, iptables, DNS, TCP, load balancing — all woven into how Kubernetes actually moves packets.

---

### The Four Kubernetes Networking Requirements

Kubernetes networking is built on four fundamental rules:

```

1. Every pod gets its own IP address

2. Pods on the SAME node can communicate without NAT

3. Pods on DIFFERENT nodes can communicate without NAT

4. Agents on a node (kubelet, kube-proxy) can communicate with all pods on that node

```

No NAT between pods. Every pod is directly routable. This is fundamentally different from Docker's default bridge networking where containers need port mapping.

---

### CNI — Container Network Interface

CNI plugins implement the actual pod networking. They're responsible for:
- Assigning IPs to pods
- Setting up routes between nodes
- Configuring network namespaces

```

POPULAR CNI PLUGINS:



AWS VPC CNI:

- Pods get REAL VPC IPs

- Directly routable in the VPC (no overlay)

- Fast (no encapsulation overhead)

- But: consumes VPC IPs aggressively

- Each ENI has limited IPs (depends on instance type)

- m5.large: 3 ENIs × 10 IPs = 29 pod IPs per node (minus node IP)

- Fix: Enable prefix delegation (ENI gets /28 = 16 IPs per slot)



# Check how many IPs a node can support:

kubectl describe node <node> | grep "allocatable" -A 5

# pods: 29  ← limited by ENI capacity, not kubelet



Calico:

- Overlay (VXLAN/IPIP) or BGP-based routing

- Supports NetworkPolicies (both ingress and egress)

- Can run alongside AWS VPC CNI for NetworkPolicy only

- Most popular for on-prem and multi-cloud

- eBPF mode for high performance



Cilium:

- eBPF-based (bypasses iptables entirely)

- Superior performance at scale

- Advanced NetworkPolicies (L7 — HTTP, gRPC, Kafka aware)

- Built-in observability (Hubble)

- Replacing kube-proxy (no iptables for Services)

- The "next generation" CNI — increasingly adopted at FAANG scale

- AWS EKS supports Cilium as an add-on



Flannel:

- Simple overlay networking (VXLAN)

- No NetworkPolicy support

- Good for learning, NOT for production

- Often paired with Calico for NetworkPolicy



Weave:

- Encrypted overlay by default

- Simple setup

- Less performant than Calico/Cilium at scale

```

### How Pod-to-Pod Communication Works (Same Node):

```

Pod A (172.16.0.5)                     Pod B (172.16.0.6)

|                                       |

eth0 (inside pod network namespace)     eth0

|                                       |

veth-a (virtual ethernet pair)          veth-b

|                                       |

└─────────── cbr0 / cni0 ──────────────┘

             (Linux bridge on node)

             

1. Pod A sends packet to 172.16.0.6

2. Packet goes through veth pair to the bridge

3. Bridge knows 172.16.0.6 is on veth-b (learned via ARP)

4. Bridge forwards packet to veth-b

5. Packet arrives at Pod B's network namespace

No NAT, no encapsulation, no overlay — direct L2 switching

```

### How Pod-to-Pod Communication Works (Different Nodes):

```

Node 1                                    Node 2

Pod A (172.16.0.5)                        Pod B (172.16.1.10)

|                                         |

veth-a → bridge                           bridge → veth-b

            |                             |

            eth0 (10.0.1.100)             eth0 (10.0.1.101)

            |                             |

            └──────── NETWORK ────────────┘



WITH AWS VPC CNI (no overlay):

1. Pod A sends packet to 172.16.1.10

2. Node 1 looks up route table — 172.16.1.0/24 → Node 2

3. Packet sent directly via VPC networking (ENI routing)

4. Node 2 receives packet, routes to Pod B via bridge

Fast! No encapsulation overhead. VPC routes handle everything.



WITH OVERLAY (VXLAN — Calico/Flannel):

1. Pod A sends packet to 172.16.1.10

2. Node 1's CNI agent encapsulates packet in VXLAN:

  [Outer Ethernet] [Outer IP: 10.0.1.100→10.0.1.101] 

  [VXLAN Header] [Inner packet: 172.16.0.5→172.16.1.10]

3. Outer packet traverses the physical network

4. Node 2 decapsulates VXLAN, delivers inner packet to Pod B

Overhead: ~50 bytes per packet (VXLAN header)

Works everywhere (cloud, on-prem, cross-cloud)



WITH BGP (Calico without overlay):

1. Calico runs BGP daemon on each node

2. Nodes exchange routes: "172.16.0.0/24 is on Node 1"

3. Packets routed directly via standard IP routing

4. No encapsulation — best performance

Requires: underlying network supports BGP or L3 routing

```

---

### Kubernetes Services — How They Actually Work

```yaml

apiVersion: v1

kind: Service

metadata:

name: my-service

spec:

type: ClusterIP

selector:

  app: my-app

ports:

- port: 80

  targetPort: 8080

```

**What happens under the hood:**

```

1. Service created → API server assigns ClusterIP (e.g., 10.96.0.15)

2. Endpoints controller watches pods matching selector

3. Creates Endpoints object listing matching pod IPs

4. kube-proxy on EVERY node watches Services and Endpoints

5. kube-proxy creates iptables/IPVS rules on EVERY node



IPTABLES MODE (default):

kube-proxy creates NAT rules:

Destination: 10.96.0.15:80 → DNAT to one of:

 172.16.0.5:8080 (pod 1)

 172.16.0.6:8080 (pod 2)

 172.16.1.10:8080 (pod 3)

Random selection with equal probability



View the actual rules:

iptables -t nat -L KUBE-SERVICES -n | grep my-service

Chain KUBE-SVC-XXXX

-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.333

  -j KUBE-SEP-AAA (→ pod 1)

-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.500

  -j KUBE-SEP-BBB (→ pod 2)

-A KUBE-SVC-XXXX

  -j KUBE-SEP-CCC (→ pod 3)



IPVS MODE (better at scale):

kube-proxy uses Linux IPVS (IP Virtual Server) instead of iptables

IPVS is a kernel-level L4 load balancer

Supports: round-robin, least connections, source hash, etc.

Much faster than iptables at high service counts

iptables: O(n) rule processing — 10,000 services = 10,000 rules scanned

IPVS: O(1) hash lookup — constant time regardless of services



Enable IPVS mode in kube-proxy:

kube-proxy --proxy-mode=ipvs



View IPVS rules:

ipvsadm -Ln

TCP  10.96.0.15:80 rr

 -> 172.16.0.5:8080    Masq    1

 -> 172.16.0.6:8080    Masq    1

 -> 172.16.1.10:8080   Masq    1

```

### Service Types — Complete Breakdown:

```

ClusterIP (default):

- Internal-only virtual IP

- Only accessible from within the cluster

- kube-proxy creates iptables/IPVS rules on every node

- Use for: internal service-to-service communication



NodePort:

- Exposes service on a static port (30000-32767) on EVERY node

- ClusterIP is also created automatically

- External traffic → any_node_ip:NodePort → pod

- kube-proxy handles routing from NodePort to pod

- Use for: development, quick external access

- NOT for production (no TLS termination, no path routing)



LoadBalancer:

- Creates cloud provider load balancer (ALB/NLB in AWS)

- NodePort is also created automatically

- External traffic → LB → NodePort → pod

- With AWS Load Balancer Controller:

  - NLB for Service type LoadBalancer

  - ALB for Ingress resources

- Use for: production external access



ExternalName:

- DNS CNAME alias, no proxy

- my-service.default.svc.cluster.local → CNAME → external-db.example.com

- Use for: referencing external services by cluster-internal DNS name



Headless (ClusterIP: None):

- No ClusterIP assigned

- DNS returns individual pod IPs (A records)

- No load balancing by kube-proxy

- Clients connect directly to specific pods

- Use for: StatefulSets (databases, Kafka, etc.)

- Pod DNS: pod-name.service-name.namespace.svc.cluster.local

```

### externalTrafficPolicy — Source IP Preservation:

```yaml

By default, NodePort/LoadBalancer services do an extra hop:

Client → Node A (where traffic arrives) → Node B (where pod runs)

This second hop:

 1. Adds latency

 2. Loses the client's source IP (SNAT'd to Node A's IP)

 3. Cross-AZ traffic = extra cost



spec:

externalTrafficPolicy: Cluster  # DEFAULT

# Traffic can land on ANY node, kube-proxy routes to correct pod

# Pro: Even load distribution

# Con: Extra hop, source IP lost, cross-AZ cost



spec:

externalTrafficPolicy: Local

# Traffic ONLY goes to pods on the node where it arrived

# If no pod on that node → connection refused (health check handles this)

# Pro: No extra hop, source IP preserved, no cross-AZ cost

# Con: Uneven distribution if pods aren't spread evenly

# REQUIRED when you need the real client IP (logging, rate limiting, geo)



# AWS NLB with externalTrafficPolicy: Local:

# NLB health checks each node

# Nodes without pods for this service → marked unhealthy → no traffic sent

# Result: traffic only goes to nodes with pods, direct delivery, source IP preserved

```

---

### Ingress — L7 Traffic Management

```yaml

apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:

name: my-ingress

annotations:

  kubernetes.io/ingress.class: nginx

  cert-manager.io/cluster-issuer: letsencrypt-prod

  nginx.ingress.kubernetes.io/rate-limit: "100"

spec:

tls:

- hosts:

  - api.example.com

  secretName: api-tls

rules:

- host: api.example.com

  http:

    paths:

    - path: /v1

      pathType: Prefix

      backend:

        service:

          name: api-v1

          port:

            number: 80

    - path: /v2

      pathType: Prefix

      backend:

        service:

          name: api-v2

          port:

            number: 80

- host: admin.example.com

  http:

    paths:

    - path: /

      pathType: Prefix

      backend:

        service:

          name: admin-dashboard

          port:

            number: 80

```

### Ingress Controllers:

```

Nginx Ingress Controller:

- Most popular, battle-tested

- Runs Nginx pods inside the cluster

- Watches Ingress resources, generates nginx.conf

- Supports: path routing, host routing, TLS, rate limiting,

  auth, rewrites, WebSocket, gRPC

- Two flavors:

  - kubernetes/ingress-nginx (community)

  - nginxinc/kubernetes-ingress (NGINX Inc commercial)



AWS Load Balancer Controller:

- Creates actual AWS ALBs/NLBs

- Ingress → ALB (L7)

- Service type LoadBalancer → NLB (L4)

- Supports: path/host routing via ALB rules

- Native AWS integration (WAF, ACM certs, Shield)

- External to the cluster (no ingress pod overhead)



Traefik:

- Auto-discovery of services

- Built-in Let's Encrypt

- Dashboard

- Popular in smaller deployments



Istio Gateway / Envoy:

- Part of Istio service mesh

- Most powerful routing capabilities

- mTLS, traffic splitting, fault injection

- Overkill for simple setups

```

---

### NetworkPolicies — Pod-Level Firewall

```yaml

By default, ALL pods can talk to ALL other pods (no isolation)

NetworkPolicies restrict this



DENY ALL ingress to a namespace:

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

name: deny-all

namespace: production

spec:

podSelector: {}    # Apply to ALL pods in namespace

policyTypes:

- Ingress

# No ingress rules = deny all ingress

# Pods can still make outbound connections



---

Allow specific traffic:

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

name: allow-api-to-db

namespace: production

spec:

podSelector:

  matchLabels:

    app: database        # Apply to database pods

policyTypes:

- Ingress

ingress:

- from:

  - podSelector:

      matchLabels:

        app: api         # Only API pods can reach database

  - namespaceSelector:

      matchLabels:

        env: production  # And only from production namespace

  ports:

  - protocol: TCP

    port: 5432           # Only PostgreSQL port



---

Restrict egress (outbound):

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

name: api-egress

namespace: production

spec:

podSelector:

  matchLabels:

    app: api

policyTypes:

- Egress

egress:

- to:

  - podSelector:

      matchLabels:

        app: database

  ports:

  - protocol: TCP

    port: 5432

- to:                    # Allow DNS (required!)

  - namespaceSelector: {}

    podSelector:

      matchLabels:

        k8s-app: kube-dns

  ports:

  - protocol: UDP

    port: 53

  - protocol: TCP

    port: 53

```

**Critical gotcha:**

```bash

If you create ANY NetworkPolicy that selects a pod,

that pod switches from "allow all" to "deny by default"

for the specified policyTypes



If you add an Ingress policy but forget DNS in Egress:

Pod can't resolve DNS → can't reach ANYTHING by hostname

Always include DNS egress when restricting egress!

#

Also: NetworkPolicies require a CNI that supports them

AWS VPC CNI alone does NOT enforce NetworkPolicies

You need: Calico, Cilium, or Weave alongside VPC CNI

EKS: Install Calico as a NetworkPolicy-only add-on

```

---

### VPC Flow Logs — Network Forensics

```bash

VPC Flow Logs capture metadata about every network flow

(NOT the packet contents — just headers)



Format:

version account-id interface-id srcaddr dstaddr srcport dstport

protocol packets bytes start end action log-status



Example:

2 123456789012 eni-abc123 10.0.1.5 10.0.2.10 45678 80 6 10 5000 

 1610000000 1610000060 ACCEPT OK



Translation:

From 10.0.1.5:45678 to 10.0.2.10:80 (TCP, protocol 6)

10 packets, 5000 bytes, accepted



Enable Flow Logs:

resource "aws_flow_log" "vpc" {

vpc_id          = aws_vpc.main.id

traffic_type    = "ALL"       # ACCEPT, REJECT, or ALL

log_destination = aws_s3_bucket.flow_logs.arn

# Or send to CloudWatch Logs for real-time analysis

}



DEBUGGING WITH FLOW LOGS:



"Why can't pod A reach the database?"

Filter flow logs for:

 srcaddr=<pod-node-ip> dstaddr=<rds-ip> dstport=5432

If action=REJECT → Security Group or NACL blocking

If no entries at all → routing issue (packets never reached the ENI)



"Where is this suspicious traffic coming from?"

Filter for: dstport=22 action=ACCEPT srcaddr NOT in <known-cidrs>

Find unauthorized SSH access



"How much cross-AZ traffic are we generating?"

Filter by src/dst subnets in different AZs

Calculate data transfer costs



Athena query for flow logs in S3:

CREATE EXTERNAL TABLE vpc_flow_logs (...)

ROW FORMAT DELIMITED FIELDS TERMINATED BY ' '

LOCATION 's3://flow-logs-bucket/AWSLogs/...'

#

SELECT srcaddr, dstaddr, dstport, action, SUM(bytes)

FROM vpc_flow_logs

WHERE action = 'REJECT' AND dstport = 5432

GROUP BY srcaddr, dstaddr, dstport, action

```

---

### Production Scenarios:

#### Scenario 1: Service Works Inside Cluster But Not From Outside

```bash

Symptoms:

curl from inside cluster → works

curl from internet → timeout



Debug:



1. Check Service type

kubectl get svc my-service

TYPE: ClusterIP ← This is internal only!

ClusterIP is NOT accessible from outside the cluster



Fix options:

a) Change to LoadBalancer → creates ALB/NLB

b) Create Ingress resource → routes through Ingress Controller

c) Change to NodePort → accessible via node_ip:port (not recommended for prod)



2. If Service type is LoadBalancer:

kubectl get svc my-service

EXTERNAL-IP: <pending>  ← LB not created yet



Check AWS Load Balancer Controller logs:

kubectl logs -n kube-system deployment/aws-load-balancer-controller

Common errors:

- IAM role missing permissions

- Subnet not tagged correctly for auto-discovery

- Security group limit reached



Required subnet tags for ALB:

Public subnets:  kubernetes.io/role/elb = 1

Private subnets: kubernetes.io/role/internal-elb = 1



3. If LB exists but still timeout:

Check: LB security group allows inbound 80/443 from 0.0.0.0/0?

Check: LB target group has healthy targets?

Check: Target security group allows inbound from LB security group?

```

#### Scenario 2: Pod-to-Pod Communication Fails After NetworkPolicy

```bash

Symptoms:

Deployed new NetworkPolicy for database isolation

API service immediately can't reach the database

"Connection timed out" from API to database



Investigation:

kubectl get networkpolicy -n production

NAME              POD-SELECTOR    AGE

db-isolation      app=database    2m



kubectl describe networkpolicy db-isolation -n production

Spec:

 PodSelector: app=database

 Allowing ingress traffic:

   From:

     PodSelector: app=api

   To Port: 5432/TCP



Looks correct... but let's check the API pod labels:

kubectl get pods -n production --show-labels | grep api

api-deployment-xxx   app=api-service   ← MISMATCH!



NetworkPolicy says: allow from app=api

API pod has: app=api-service

Labels don't match → traffic denied



Fix:

Either update NetworkPolicy selector to app=api-service

Or update deployment labels to app=api



LESSON: NetworkPolicy label mismatches are the #1 cause of

"it worked before the policy" issues

ALWAYS verify labels with --show-labels after policy changes

```

#### Scenario 3: Mysterious 10x Latency Increase After IPVS Migration

```bash

Symptoms:

Migrated kube-proxy from iptables to IPVS mode

Service-to-service latency increased 10x

Only affects certain service pairs



Investigation:

IPVS uses connection-based load balancing by default

With iptables: random selection per packet

With IPVS rr (round-robin): all packets for a connection go to same pod



But the real issue:

IPVS session affinity was enabled by default

Timeout: 360 seconds

All connections from Pod A to Service B go to the SAME backend pod

If that pod is slow → ALL requests from A to B are slow

No automatic failover to healthy pods



Check IPVS session persistence:

ipvsadm -Ln --persistent-conn



Fix:

Disable persistence or reduce timeout:

In kube-proxy config:

apiVersion: kubeproxy.config.k8s.io/v1alpha1

kind: KubeProxyConfiguration

mode: ipvs

ipvs:

scheduler: rr

tcpTimeout: 0s         # Disable session persistence

udpTimeout: 0s

```

#### Scenario 4: Conntrack Table Full on EKS Nodes Running Many Services

```bash

Symptoms:

Random packet drops across multiple services

dmesg: "nf_conntrack: table full, dropping packet"

Node has 200+ services configured



Root cause:

Each Service creates iptables rules

Each connection through iptables creates conntrack entry

200 services × 1000 connections each = 200,000 conntrack entries

Default nf_conntrack_max: 131072

EXCEEDED



Immediate fix (on the node):

sysctl -w net.netfilter.nf_conntrack_max=1048576



Permanent fix — in EKS node launch template:

Add to user_data:

echo "net.netfilter.nf_conntrack_max=1048576" >> /etc/sysctl.d/40-k8s.conf

sysctl --system



Long-term fix — migrate to Cilium:

Cilium uses eBPF, which BYPASSES conntrack entirely for pod traffic

No conntrack table = no conntrack exhaustion

This is one of the primary reasons companies adopt Cilium at scale



Monitor:

node_nf_conntrack_entries / node_nf_conntrack_entries_limit

Alert at 75% saturation

```
---

# 📋 LESSON 10 QUICK REFERENCE — Kubernetes Networking

```

CNI PLUGINS:

AWS VPC CNI: Real VPC IPs, no overlay, fast, eats IPs

Calico: Overlay/BGP, NetworkPolicy support, most popular

Cilium: eBPF-based, bypasses iptables, L7 policies, next-gen

Flannel: Simple overlay, no NetworkPolicy, learning only



POD NETWORKING:

Same node: veth pairs → Linux bridge → direct delivery

Different nodes:

  VPC CNI: direct VPC routing (no encapsulation)

  Overlay: VXLAN/IPIP encapsulation (~50 byte overhead)

  BGP: native IP routing (best performance, requires L3 network)



SERVICE TYPES:

ClusterIP: internal only, iptables/IPVS rules

NodePort: every node listens on 30000-32767

LoadBalancer: cloud LB → NodePort → pod

ExternalName: CNAME alias to external service

Headless (ClusterIP: None): returns pod IPs directly



KUBE-PROXY MODES:

iptables: default, O(n) rule scanning, simple

IPVS: O(1) hash lookup, multiple algorithms, better at scale

Cilium: replaces kube-proxy entirely with eBPF



externalTrafficPolicy:

Cluster: extra hop, source IP lost, even distribution

Local: direct delivery, source IP preserved, uneven distribution



INGRESS:

L7 routing (path, host), TLS termination

Controllers: nginx, AWS LB Controller, Traefik, Istio

AWS: Ingress → ALB, Service LB → NLB



NETWORKPOLICY:

Default: all pods open, any NetworkPolicy = deny by default

ALWAYS include DNS egress (UDP/TCP 53) when restricting egress

Label mismatch = silent traffic deny (check --show-labels!)

Requires: Calico, Cilium, or Weave (VPC CNI alone doesn't enforce)



VPC FLOW LOGS:

Captures: src/dst IP, ports, protocol, action (ACCEPT/REJECT)

NOT packet contents

Send to S3 + Athena for analysis

REJECT entries = firewall blocking



PRODUCTION GOTCHAS:

- LB subnet tags required: kubernetes.io/role/elb = 1

- VPC CNI IP exhaustion: prefix delegation or secondary CIDR

- Conntrack exhaustion: increase nf_conntrack_max or use Cilium

- IPVS session persistence causes uneven distribution

- NetworkPolicy label mismatch = silent failure

```


---

# 📝 Retention Questions — Lesson 10

**Q1:** Your EKS cluster uses AWS VPC CNI. A `m5.large` node can only run 29 pods despite having plenty of CPU and memory. Why is it limited to 29, and how do you increase it?

**Q2:** After deploying a NetworkPolicy to restrict database access, the API service can no longer resolve DNS for the database service name. The NetworkPolicy only has Ingress rules on the database pod. Explain why DNS broke and how to fix it.

**Q3:** You're running 500 Kubernetes Services. An engineer proposes migrating kube-proxy from iptables mode to IPVS mode. Explain why this improves performance and what potential gotcha they should watch for during migration.

**Q4:** You need to expose a service externally and preserve the real client IP for rate limiting. The service is behind an AWS NLB. What `externalTrafficPolicy` should you use, and what's the trade-off?

**Go.** 🎯
===========================================================================

# PHASE 2 — Lesson 1B: Git Commands & Workflows (The Missing Half)

---

### Fetch vs Pull — The Distinction That Matters

```bash

\git fetch: downloads objects and refs from remote. Touches NOTHING local.

git fetch origin

\Updates: .git/refs/remotes/origin/main

\Does NOT update: your local main branch, your working directory

\Safe. Always safe. You're just getting information.



\git pull: fetch + merge (or fetch + rebase if configured)

git pull origin main

\Equivalent to:

git fetch origin

git merge origin/main



\git pull --rebase:

git pull --rebase origin main

\Equivalent to:

git fetch origin

git rebase origin/main



\WHY THIS MATTERS:

\git pull with merge creates merge commits on your local branch

\Your history:    A → B → C (local)

\Remote history:  A → B → D (someone pushed)

\git pull creates: A → B → C → M (merge commit)

\                       └→ D ↗

\Ugly. Pollutes history with "Merge branch 'main' of origin..."

\\#

\git pull --rebase:

\Replays your C on top of D:  A → B → D → C'

\Clean linear history.

\\#

\Configure globally:

git config --global pull.rebase true

\Now git pull always rebases instead of merging.

\This is the standard at most serious engineering orgs.

```

---

### Reset — The Three Modes

```bash

\git reset moves the branch pointer AND optionally changes

\the staging area and working directory.



\SETUP: You have 3 commits

\A → B → C (HEAD)



\--soft: moves HEAD only. Staging and working dir UNTOUCHED.

git reset --soft HEAD~1

\HEAD now at B

\Staging area: still has C's changes (ready to commit)

\Working directory: unchanged

\Use case: "I want to redo that commit with a different message"

\          or "I want to combine the last 2 commits"



\--mixed (DEFAULT): moves HEAD + resets staging. Working dir UNTOUCHED.

git reset HEAD~1        # --mixed is the default

git reset --mixed HEAD~1

\HEAD now at B

\Staging area: reset to match B (C's changes are UNSTAGED)

\Working directory: still has C's changes as modified files

\Use case: "I want to unstage everything and re-add selectively"



\--hard: moves HEAD + resets staging + resets working directory.

git reset --hard HEAD~1

\HEAD now at B

\Staging area: matches B

\Working directory: matches B

\C's changes are GONE from working dir

\Use case: "Throw everything away, go back to this state"

\DANGEROUS: working directory changes are NOT recoverable

\           (unless they were committed — then reflog saves you)



\COMMON PATTERNS:

\Undo last commit but keep changes staged:

git reset --soft HEAD~1



\Unstage a file (without losing changes):

git reset HEAD myfile.txt     # old way

git restore --staged myfile.txt  # new way (Git 2.23+)



\Nuke everything and match remote exactly:

git fetch origin

git reset --hard origin/main

\Your local main is now identical to remote. All local changes gone.

```

Visual:

```

\                  --soft       --mixed      --hard

HEAD (branch ptr)  MOVES        MOVES        MOVES

Staging area       unchanged    RESET        RESET

Working directory  unchanged    unchanged    RESET

```

---

### Revert vs Reset — Undo Strategies

```bash

\RESET: rewrites history (moves the branch pointer backward)

\REVERT: creates a NEW commit that undoes a previous commit



\Reset:

\A → B → C → D (HEAD)

git reset --hard HEAD~2

\A → B (HEAD)

\C and D are orphaned. History is rewritten.

\If C and D were pushed, you need --force to push.

\NEVER reset shared branches.



\Revert:

\A → B → C → D (HEAD)

git revert HEAD

\A → B → C → D → D' (HEAD)

\D' is a new commit that undoes D's changes

\History is PRESERVED. No force push needed.

\Safe for shared branches.



\Revert a specific commit (not the latest):

git revert abc1234

\Creates a new commit that undoes abc1234's changes

\May conflict if later commits depend on abc1234



\Revert a merge commit:

git revert -m 1 <merge-commit-hash>

\-m 1 means "keep the first parent's side" (usually main)

\This undoes the merged branch's changes

\GOTCHA: if you later want to re-merge that branch,

\you must REVERT THE REVERT first, or Git thinks

\those changes are already in history



\THE RULE:

\Pushed to shared branch? → git revert

\Local only, not pushed? → git reset

\No exceptions.

```

---

### Cherry-Pick

```bash

\Copy a specific commit from one branch to another

\WITHOUT merging the entire branch



\Scenario: hotfix commit on develop needs to go to main NOW

git checkout main

git cherry-pick abc1234

\Creates a NEW commit on main with the same diff as abc1234

\Different hash (it's a new commit), same changes



\Cherry-pick multiple commits:

git cherry-pick abc1234 def5678



\Cherry-pick a range:

git cherry-pick abc1234..def5678

\Applies everything AFTER abc1234 up to and including def5678



\Cherry-pick without committing (stage only):

git cherry-pick --no-commit abc1234

\Changes are staged but not committed

\Useful when you want to combine multiple cherry-picks into one commit



\CONFLICTS during cherry-pick:

git cherry-pick abc1234

\CONFLICT in file.txt

\Fix the conflict, then:

git add file.txt

git cherry-pick --continue

\Or abort:

git cherry-pick --abort



\WHEN TO USE:

\- Hotfixes that need to go to release branch

\- Extracting a single useful commit from an abandoned branch

\- Backporting fixes to older versions



\WHEN NOT TO USE:

\- Moving large sets of commits (use merge or rebase)

\- Cherry-picking creates duplicate commits in history

\  (same diff, different hashes on different branches)

```

---

### Conflict Resolution — The Full Workflow

```bash

\WHEN CONFLICTS HAPPEN:

\merge, rebase, cherry-pick, stash pop — any operation that

\combines changes from two sources



\Git marks conflicts in the file:

<<<<<<< HEAD

const port = 3000;

=======

const port = 8080;

>>>>>>> feature-branch



\HEAD section: what's on your current branch

\======= divider

\feature-branch section: what's on the incoming branch



\RESOLUTION STEPS:

\1. Open the file

\2. Choose which version (or combine both)

\3. Remove the conflict markers entirely

\4. git add the resolved file

\5. git merge --continue (or git rebase --continue)



\USEFUL TOOLS:

\See all conflicted files:

git diff --name-only --diff-filter=U



\Use a merge tool:

git mergetool

\Opens configured diff tool (vimdiff, meld, VS Code, IntelliJ)



\Accept one side entirely:

git checkout --ours file.txt     # keep current branch version

git checkout --theirs file.txt   # keep incoming branch version



\During rebase conflicts — CAREFUL:

\"ours" and "theirs" are SWAPPED during rebase!

\Because rebase replays YOUR commits onto THEIR base

\--ours = the branch you're rebasing ONTO (main)

\--theirs = YOUR commits being replayed

\This confuses everyone. Every time.



\ABORT if things go wrong:

git merge --abort

git rebase --abort

git cherry-pick --abort

\Returns to pre-operation state. No damage done.

```

---

### Git Log — Actually Useful

```bash

\Basic:

git log --oneline              # short hash + message, one line each

git log --graph --oneline      # ASCII branch visualization

git log --graph --oneline --all  # ALL branches, not just current



\Filter by author:

git log --author="john"        # partial match works



\Filter by date:

git log --since="2024-01-01" --until="2024-06-01"

git log --since="2 weeks ago"



\Filter by message:

git log --grep="hotfix"        # commits whose message contains "hotfix"



\Filter by file:

git log -- path/to/file.txt    # only commits that touched this file

git log -p -- path/to/file.txt # show the actual diff for each commit



\Filter by content change (pickaxe):

git log -S "DATABASE_URL"      # commits that added/removed this STRING

git log -G "port.*8080"        # commits matching this REGEX in diff

\Incredibly powerful for: "who changed this config value and when?"



\Show stats:

git log --stat                 # files changed, insertions/deletions

git log --shortstat            # summary only



\Limit output:

git log -5                     # last 5 commits

git log main..feature          # commits in feature NOT in main

git log feature..main          # commits in main NOT in feature



\Format for scripts:

git log --format="%H %an %s"  # full hash, author name, subject

```

---

### Git Blame, Show, Diff

```bash

\BLAME — who wrote each line:

git blame file.txt

\a1b2c3d (John 2024-01-15 14:30:00 +0000  1) const port = 3000;

\b2c3d4e (Jane 2024-02-20 09:15:00 +0000  2) const host = "0.0.0.0";



\Blame a specific range of lines:

git blame -L 10,20 file.txt    # lines 10-20 only



\Ignore whitespace changes:

git blame -w file.txt



\SHOW — inspect any object:

git show HEAD                   # latest commit diff

git show abc1234                # specific commit

git show abc1234:path/to/file   # file contents at that commit

git show v1.0.0                 # tag details



\DIFF:

git diff                        # working dir vs staging (unstaged changes)

git diff --staged               # staging vs last commit (staged changes)

git diff HEAD                   # working dir vs last commit (all changes)

git diff main..feature          # difference between two branches

git diff abc1234 def5678        # difference between two commits

git diff --stat main..feature   # summary: files changed, lines +/-

git diff --name-only HEAD~3     # just filenames changed in last 3 commits

```

---

### Restore and Switch (Git 2.23+ — The Modern Way)

```bash

\Git 2.23 split `git checkout` into two focused commands:



\OLD (overloaded):

git checkout main                 # switch branches

git checkout -- file.txt          # discard working dir changes

git checkout abc1234 -- file.txt  # restore file from specific commit



\NEW (clear intent):

\git switch — for branch operations:

git switch main                   # switch to main

git switch -c new-branch          # create and switch

git switch -                      # switch to previous branch



\git restore — for file operations:

git restore file.txt              # discard working dir changes (from index)

git restore --staged file.txt     # unstage (move from index to working dir)

git restore --source=HEAD~3 file.txt  # restore from specific commit

git restore --staged --worktree file.txt  # unstage AND discard changes



\git checkout still works. But switch/restore make intent explicit.

\In scripts and CI, prefer the new commands.

```

---

### Git Clean — Remove Untracked Files

```bash

git clean -n          # dry run — show what WOULD be deleted

git clean -f          # delete untracked files

git clean -fd         # delete untracked files AND directories

git clean -fdx        # delete untracked files, dirs, AND ignored files

\                     # WARNING: -x removes things in .gitignore too

\                     # (node_modules, build artifacts, .env files)



\Use case: "I want a pristine state matching the repo exactly"

git reset --hard HEAD

git clean -fd

\Working directory now matches HEAD commit perfectly

```

---

### Git LFS — Large File Storage

```bash

\Problem: Git stores full copies of every version of every file

\Binary files (images, videos, ML models, JARs) bloat the repo

\A 100MB model file × 50 versions = 5GB repo



\Git LFS replaces large files with lightweight POINTERS in the repo

\Actual file content stored on a separate LFS server



\Setup:

git lfs install                              # one-time setup

git lfs track "*.psd"                        # track Photoshop files

git lfs track "models/**"                    # track ML models directory

\This creates/updates .gitattributes:

*.psd filter=lfs diff=lfs merge=lfs -text



git add .gitattributes                       # commit the tracking rules

git add large-file.psd

git commit -m "add design file"

git push                                     # pushes pointer to Git, content to LFS



\What's in the repo:

git cat-file -p HEAD:large-file.psd

\version https://git-lfs.github.com/spec/v1

\oid sha256:abc123...

\size 104857600

\← That's it. A 130-byte pointer, not 100MB.



\CI CONSIDERATION:

\git clone fetches LFS files automatically (if lfs is installed)

\To skip LFS in CI (if you don't need the actual binaries):

GIT_LFS_SKIP_SMUDGE=1 git clone repo.git

\Saves bandwidth and time in pipelines that don't need binary assets

```

---

### .gitattributes — Beyond LFS

```bash

\Line ending normalization (cross-platform teams):

* text=auto                  # Git auto-detects text files, normalizes endings

*.sh text eol=lf             # Shell scripts always LF (even on Windows)

*.bat text eol=crlf          # Batch files always CRLF

*.png binary                 # Never treat as text, never normalize



\Custom merge strategies per file:

database/schema.sql merge=ours    # always keep our version on conflict

package-lock.json merge=ours      # avoid lockfile merge nightmares

\(requires: git config merge.ours.driver true)



\Diff drivers:

*.zip diff=zip                # custom diff for zip files

*.md diff=markdown            # better diff formatting for markdown

```

---

### Git Hooks — In Detail

```bash

\.git/hooks/ contains sample scripts (*.sample)

\Remove .sample suffix to activate



\CLIENT-SIDE HOOKS:

pre-commit        # runs before commit is created

\                 # exit 1 = abort commit

\                 # use for: lint, format, secrets scan



commit-msg        # runs after message entered, before commit finalized

\                 # receives message file path as arg

\                 # use for: enforce message format (JIRA-123: description)



pre-push          # runs before push

\                 # use for: run tests, prevent push to main



post-checkout     # runs after git checkout / git switch

\                 # use for: npm install after branch switch



\SERVER-SIDE HOOKS (Bitbucket/GitLab/self-hosted):

pre-receive       # runs before accepting a push

\                 # use for: enforce branch naming, reject force push

\                 # exit 1 = reject the entire push



update            # like pre-receive but per-branch

\                 # use for: per-branch policies



post-receive      # runs after push is accepted

\                 # use for: trigger CI, notify Slack, deploy



\SHARED HOOKS (team-wide):

\.git/hooks/ is NOT committed (it's inside .git/)

\Solution: use a framework



\pre-commit framework (Python):

\.pre-commit-config.yaml (committed to repo):

repos:

\ - repo: https://github.com/pre-commit/pre-commit-hooks

\   rev: v4.5.0

\   hooks:

\     - id: trailing-whitespace

\     - id: end-of-file-fixer

\     - id: check-yaml

\     - id: check-added-large-files

\       args: ['--maxkb=500']

\ - repo: https://github.com/gitleaks/gitleaks

\   rev: v8.18.1

\   hooks:

\     - id: gitleaks        # scan for secrets



\Install: pre-commit install

\Now every developer who runs pre-commit install

\gets these hooks locally



\Husky (Node.js projects):

\package.json:

"husky": {

\ "hooks": {

\   "pre-commit": "lint-staged",

\   "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"

\ }

}

```

---

### Submodules vs Subtrees

```bash

\SUBMODULES: embed another repo as a subdirectory

git submodule add https://github.com/org/shared-lib.git libs/shared

\Creates .gitmodules file:

[submodule "libs/shared"]

\  path = libs/shared

\  url = https://github.com/org/shared-lib.git



\The parent repo stores a POINTER (commit hash) to the submodule

\NOT the submodule's contents



\Clone with submodules:

git clone --recurse-submodules repo.git

\Or after clone:

git submodule update --init --recursive



\Update submodule to latest:

cd libs/shared

git pull origin main

cd ../..

git add libs/shared

git commit -m "update shared-lib to latest"



\PAIN POINTS (and there are many):

\- Developers forget --recurse-submodules on clone

\- CI must explicitly init submodules

\- Detached HEAD inside submodule is the default

\- Merge conflicts on submodule pointer are confusing

\- Nested submodules compound all these problems



\SUBTREES: copy another repo INTO your repo as actual files

git subtree add --prefix=libs/shared https://github.com/org/shared-lib.git main --squash



\Pull updates:

git subtree pull --prefix=libs/shared https://github.com/org/shared-lib.git main --squash



\Push changes back upstream:

git subtree push --prefix=libs/shared https://github.com/org/shared-lib.git main



\SUBTREE ADVANTAGES:

\- No special clone commands — files are in the repo

\- No .gitmodules, no detached HEAD confusion

\- Works with existing tooling without modifications



\SUBTREE DISADVANTAGES:

\- Repo size grows (full copy of subtree code)

\- History can be messy without --squash

\- Push back to upstream is awkward



\RECOMMENDATION:

\Small shared libraries → subtree (simpler)

\Large external dependencies → submodule (saves space)

\Or better: use a proper package manager (npm, pip, Maven)

\  and publish shared code as packages

```

---

### Git Worktrees

```bash

\Problem: you're working on feature-x, urgent hotfix needed on main

\Without worktrees: stash, switch, fix, switch back, pop stash

\With worktrees: check out another branch in a SEPARATE directory



git worktree add ../hotfix-dir main

\Creates ../hotfix-dir with main checked out

\SHARES the same .git database (no re-clone)



cd ../hotfix-dir

\fix the issue

git commit -am "hotfix"

git push



\Go back to original directory — feature-x is untouched

cd ../original-dir



\Clean up when done:

git worktree remove ../hotfix-dir



\List all worktrees:

git worktree list



\USE CASES:

\- Hotfixes without disrupting current work

\- Running tests on one branch while coding on another

\- Comparing behavior across branches side by side

\- CI that needs multiple branches checked out simultaneously

```

# 📋 QUICK REFERENCE — Git Commands (Lesson 1B)

```

FETCH vs PULL:

\ fetch: download, touch nothing local

\ pull: fetch + merge (or + rebase if configured)

\ Always: git config --global pull.rebase true



RESET:

\ --soft:  HEAD moves. Staging + working dir untouched.

\ --mixed: HEAD moves. Staging reset. Working dir untouched.

\ --hard:  HEAD moves. Staging reset. Working dir reset. DESTRUCTIVE.



UNDO:

\ Pushed? → git revert (new commit, safe)

\ Local only? → git reset (rewrite history)

\ Revert a merge: git revert -m 1 <hash>

\ Re-merge after revert: must revert the revert first



CHERRY-PICK:

\ git cherry-pick <hash>           single commit

\ git cherry-pick --no-commit      stage only

\ Use for hotfixes, backports. Not for bulk moves.



CONFLICTS:

\ ours/theirs SWAP during rebase (everyone forgets this)

\ git checkout --ours/--theirs for bulk resolution

\ Always: git merge --abort / git rebase --abort if lost



LOG:

\ -S "string"    pickaxe: who added/removed this string

\ -G "regex"     grep through diffs

\ main..feature  commits in feature not in main

\ --author, --since, --grep for filtering



LFS:

\ git lfs track "*.bin"

\ GIT_LFS_SKIP_SMUDGE=1 git clone   (skip in CI)

\ Repo stores 130-byte pointer, LFS server stores content



SUBMODULES vs SUBTREES:

\ Submodules: pointer to external repo (saves space, painful workflow)

\ Subtrees: full copy in your repo (simple workflow, larger repo)

\ Best: publish shared code as packages instead



WORKTREES:

\ git worktree add ../dir branch

\ Multiple branches checked out simultaneously

\ Shares .git database — no re-clone

```

---

# 📝 Retention Questions — Lesson 1B

**Q1:** A junior developer on your team ran `git reset --hard HEAD~3` on the `main` branch, then did `git push --force`. Three commits are gone from remote. Two other developers have already pulled the old `main`. Walk through the full recovery process.

**Q2:** A critical bug is found in production. The fix exists as a single commit on the `develop` branch, but `develop` has 40 other untested commits. How do you get ONLY that fix onto the `release` branch? What happens if the fix conflicts?

**Q3:** Your team reverted a merge to `main` because it caused issues. The feature branch was fixed and they want to re-merge it. But `git merge feature` says "Already up to date." Explain why and how to fix it.

**Q4:** Explain the difference between `git reset --mixed HEAD~1` and `git restore --staged .` — when would you use each?

**Go.** 🎯



# PHASE 2 — Lesson 2B: Docker Production Deep Dive (The Missing Half)

---

### ENTRYPOINT vs CMD — The PID 1 Problem

```dockerfile

TWO FORMS:



SHELL FORM (string):

CMD echo "hello"

Actually runs: /bin/sh -c 'echo "hello"'

PID 1 = /bin/sh (NOT your process)

Your process is a CHILD of sh

SIGTERM sent to PID 1 (sh) → sh does NOT forward it to your process

Container doesn't gracefully shut down → waits 10s → SIGKILL

YOUR APP NEVER RECEIVES SIGTERM



EXEC FORM (array):

CMD \["echo", "hello"]

Runs: echo "hello" directly

PID 1 = echo (YOUR process)

SIGTERM goes directly to your process

Graceful shutdown works



ALWAYS USE EXEC FORM IN PRODUCTION



ENTRYPOINT vs CMD:

ENTRYPOINT = the executable (fixed)

CMD = default arguments (overridable)



ENTRYPOINT \["python", "app.py"]

CMD \["--port", "8080"]

docker run myapp → python app.py --port 8080

docker run myapp --port 9090 → python app.py --port 9090

CMD is replaced, ENTRYPOINT stays



Only CMD:

CMD \["python", "app.py"]

docker run myapp → python app.py

docker run myapp /bin/sh → /bin/sh (CMD completely replaced)



Only ENTRYPOINT:

ENTRYPOINT \["python", "app.py"]

docker run myapp → python app.py

docker run myapp --debug → python app.py --debug (args appended)

Can't override without --entrypoint flag



PRODUCTION PATTERN:

ENTRYPOINT \["java", "-jar", "app.jar"]

CMD \[]

Or with a wrapper script:

ENTRYPOINT \["/docker-entrypoint.sh"]

CMD \["start"]

docker run myapp start → /docker-entrypoint.sh start

docker run myapp migrate → /docker-entrypoint.sh migrate

```

### The PID 1 Zombie Reaping Problem

```bash

In Linux, PID 1 (init) has a special responsibility:

It must REAP orphaned child processes (zombies)

A zombie = process that exited but parent hasn't called wait()

Normal OS: systemd (PID 1) reaps zombies

Container: YOUR process is PID 1



Problem scenario:

Your app spawns child processes (workers, scripts)

Child exits → becomes zombie

Your app doesn't call wait() → zombie accumulates

100s of zombies → PID namespace exhaustion → container can't fork



SOLUTION 1: tini (lightweight init)

FROM python:3.11-slim

RUN apt-get update \&\& apt-get install -y tini \&\& rm -rf /var/lib/apt/lists/\*

ENTRYPOINT \["tini", "--"]

CMD \["python", "app.py"]

tini is PID 1 → reaps zombies, forwards signals to your app

tini receives SIGTERM → forwards to python → graceful shutdown

Only 30KB binary



SOLUTION 2: dumb-init (Yelp)

ENTRYPOINT \["dumb-init", "--"]

CMD \["python", "app.py"]

Same concept, slightly different signal handling



SOLUTION 3: Docker's built-in init

docker run --init myapp

Docker injects tini automatically

Or in compose:

services:

 app:

   init: true

In Kubernetes:

No built-in equivalent — must include tini/dumb-init in image

OR use shareProcessNamespace: true (but that has other implications)



WHEN YOU NEED IT:

- Shell scripts that spawn background processes

- Applications using process pools (Gunicorn prefork, Celery workers)

- Anything that forks child processes



WHEN YOU DON'T:

- Single-process containers (Go binary, Node.js single-thread)

- JVM (handles its own threads internally, not separate processes)

```

---

### Signal Handling in Containers

```bash

SHUTDOWN SEQUENCE:

docker stop <container>:

1. Sends SIGTERM to PID 1

2. Waits --stop-timeout (default 10s)

3. Sends SIGKILL



docker kill <container>:

1. Sends SIGKILL immediately (no grace period)



SHELL FORM TRAP:

Dockerfile: CMD python app.py

Actually runs: /bin/sh -c "python app.py"

PID 1 = sh

sh does NOT forward SIGTERM to python

Python never knows it should shut down

After 10s → SIGKILL → dirty shutdown (connections dropped, data loss)



FIX 1: exec form (already covered)

CMD \["python", "app.py"]



FIX 2: exec in shell script

\#!/bin/sh

docker-entrypoint.sh

echo "Starting app..."

exec python app.py

exec REPLACES the shell process with python

python becomes PID 1 → receives SIGTERM directly



FIX 3: trap in shell script

\#!/bin/sh

cleanup() {

   echo "Caught signal, shutting down..."

   kill -TERM "$child"

   wait "$child"

}

trap cleanup SIGTERM SIGINT



python app.py \&

child=$!

wait "$child"

Shell traps signal, forwards to child, waits for clean exit



STOPSIGNAL directive:

Some apps listen for different signals:

STOPSIGNAL SIGQUIT    # Nginx uses SIGQUIT for graceful shutdown

STOPSIGNAL SIGINT     # Some apps prefer SIGINT



Kubernetes:

preStop hook runs BEFORE SIGTERM

terminationGracePeriodSeconds = total time for preStop + shutdown

SIGKILL after grace period — unconditional

```

---

### Volume Types — Bind Mounts vs Volumes vs tmpfs

```bash

THREE TYPES:



1. NAMED VOLUMES (Docker-managed):

docker volume create my-data

docker run -v my-data:/app/data myapp

Stored at: /var/lib/docker/volumes/my-data/\_data

Docker manages lifecycle

Survives container removal

Can be backed by drivers (local, NFS, EBS, etc.)

PREFERRED for persistent data



2. BIND MOUNTS (host path):

docker run -v /host/path:/container/path myapp

Direct mount of host directory into container

Changes visible immediately on both sides

Use for: development (live reload), host config files

DANGER: container can modify host files

NOT portable (depends on host path existing)



3. TMPFS (memory-backed):

docker run --tmpfs /tmp:rw,noexec,nosuid,size=64m myapp

RAM-backed filesystem

Fast, auto-cleaned on container stop

Use for: sensitive temp files (secrets processing), scratch space

Not persisted, not shared between containers



VOLUME DRIVERS:

docker volume create --driver local \\

 --opt type=nfs \\

 --opt o=addr=nfs-server.example.com,rw \\

 --opt device=:/shared/data \\

 nfs-data

local driver with NFS options

Other drivers: REX-Ray (EBS/EFS), Portworx, Flocker



COMPOSE VOLUME SYNTAX:

services:

 app:

   volumes:

     - my-data:/app/data              # named volume

     - ./config:/app/config:ro        # bind mount, read-only

     - type: tmpfs                    # tmpfs

       target: /tmp

       tmpfs:

         size: 67108864               # 64MB



READ-ONLY VOLUMES:

docker run -v my-data:/app/data:ro myapp

Container can read but not write

Use for: shared config, certificates



VOLUME INSPECTION:

docker volume ls

docker volume inspect my-data

Shows: mount point, driver, labels, creation date



ORPHANED VOLUMES:

docker volume ls -f dangling=true

Volumes not referenced by any container

Common source of disk space leak

docker volume prune    # remove all dangling volumes

```

---

### Docker Disk Management

```bash

DISK USAGE:

docker system df

TYPE            TOTAL   ACTIVE   SIZE      RECLAIMABLE

Images          45      12       12.5GB    8.3GB (66%)

Containers      15      8        2.1GB     1.8GB (85%)

Local Volumes   23      8        5.6GB     3.2GB (57%)

Build Cache     -       -        4.2GB     4.2GB



docker system df -v    # verbose — per-image, per-container breakdown



CLEANUP:

Remove stopped containers:

docker container prune



Remove unused images (not referenced by any container):

docker image prune

Remove ALL images not used by running containers:

docker image prune -a



Remove unused volumes:

docker volume prune



Remove unused networks:

docker network prune



NUCLEAR: remove everything unused:

docker system prune -a --volumes

Removes: stopped containers, unused networks, unused images,

          build cache, dangling volumes

DANGEROUS in production — removes cached images = slow next pull



BUILD CACHE:

docker builder prune

Remove build cache (can be massive after many builds)

docker builder prune --keep-storage=5GB

Keep 5GB of most recently used cache



AUTOMATED CLEANUP IN PRODUCTION:

K8s kubelet handles image GC:

--image-gc-high-threshold=85 (start GC when disk is 85% full)

--image-gc-low-threshold=80 (GC until disk is 80% full)

Removes least recently used images first

On EKS: configured via kubelet extra args in launch template



LOG ROTATION (often missed source of disk fill):

Docker logs can grow unbounded by default!

/var/lib/docker/containers/<id>/<id>-json.log



Configure in /etc/docker/daemon.json:

{

 "log-driver": "json-file",

 "log-opts": {

   "max-size": "50m",        

   "max-file": "3"           

 }

}

Each container: max 3 log files × 50MB = 150MB max

Without this: a verbose app can fill a 100GB disk in hours



PER-CONTAINER override:

docker run --log-opt max-size=10m --log-opt max-file=3 myapp

```

---

### Docker Logging Drivers

```bash

Docker supports multiple log drivers:



json-file (DEFAULT):

Writes JSON-formatted logs to disk

docker logs command works

Needs rotation configured (see above)



journald:

Sends to systemd journal

docker logs still works

journalctl CONTAINER\_NAME=my-container



syslog:

Sends to syslog daemon

docker logs does NOT work (logs go to syslog, not local)



fluentd:

Sends to Fluentd collector

docker logs does NOT work

Use for: centralized logging pipelines



awslogs:

Sends directly to AWS CloudWatch Logs

docker logs does NOT work

Use for: ECS tasks, standalone Docker on EC2

{

 "log-driver": "awslogs",

 "log-opts": {

   "awslogs-region": "us-east-1",

   "awslogs-group": "/docker/my-app",

   "awslogs-stream": "my-container",

   "awslogs-create-group": "true"

 }

}



IMPORTANT:

In Kubernetes: DON'T use Docker log drivers

kubelet reads container logs from the default json-file/local driver

kubectl logs depends on this

If you change the driver → kubectl logs stops working

K8s logging: use DaemonSet (Fluentbit/Fluentd) to ship log files

from /var/log/containers/\*.log to your logging backend

```

---

### Docker Restart Policies

```bash

docker run --restart=no myapp          # DEFAULT: never restart

docker run --restart=on-failure:5 myapp  # restart on non-zero exit, max 5 times

docker run --restart=always myapp      # always restart (even on clean exit)

docker run --restart=unless-stopped myapp  # like always, but not after docker stop



BACKOFF:

Docker uses exponential backoff for restarts:

1st: immediate

2nd: 1 second

3rd: 2 seconds

4th: 4 seconds

...up to 1 minute cap

Reset to 0 after container runs successfully for 10+ seconds



COMPOSE:

services:

 app:

   restart: unless-stopped

   # or deploy.restart\_policy for Swarm mode



KUBERNETES EQUIVALENT:

restartPolicy: Always (default for Deployments)

restartPolicy: OnFailure (for Jobs)

restartPolicy: Never (for one-shot pods)

K8s kubelet manages restarts with its own backoff:

CrashLoopBackOff: 10s → 20s → 40s → 80s → 160s → 300s (5 min cap)

```

---

### Multi-Architecture Builds (buildx)

```bash

WHY: ARM instances (Graviton on AWS) are 20-40% cheaper

Your image built on AMD64 laptop won't run on ARM nodes

You need images for BOTH architectures



SETUP:

docker buildx create --name multiarch --driver docker-container --use

docker buildx inspect --bootstrap



BUILD FOR MULTIPLE PLATFORMS:

docker buildx build \\

 --platform linux/amd64,linux/arm64 \\

 --tag my-app:v1.2.3 \\

 --push \\

 .

Builds TWO images, creates a manifest list

When pulled: Docker automatically selects the right architecture

docker pull my-app:v1.2.3 on AMD64 → gets AMD64 image

docker pull my-app:v1.2.3 on ARM64 → gets ARM64 image



Dockerfile considerations for multi-arch:

Most base images support multi-arch (python:3.11-slim, node:20-slim)

But if you compile native code:

FROM --platform=$BUILDPLATFORM golang:1.21 AS builder

ARG TARGETARCH

RUN GOARCH=$TARGETARCH go build -o /app/server

$BUILDPLATFORM = where you're building (your laptop)

$TARGETARCH = target architecture (amd64 or arm64)

Cross-compilation happens on your machine → fast



CHECK IMAGE ARCHITECTURE:

docker manifest inspect my-app:v1.2.3

Shows all platforms available in the manifest list



CI PIPELINE:

Build multi-arch in CI (Jenkins/GitHub Actions):

Use QEMU emulation for cross-platform builds

docker buildx create --name ci-builder \\

 --driver docker-container \\

 --driver-opt image=moby/buildkit:latest

Or: use native ARM runners (GitHub has Graviton runners)



EKS WITH GRAVITON:

m6g.xlarge, c6g.large, r6g.2xlarge = ARM (Graviton)

Karpenter can select these automatically:

requirements:

- key: kubernetes.io/arch

  operator: In

  values: \["amd64", "arm64"]   # allow both

Karpenter picks cheapest → usually Graviton

Your image MUST support arm64 or pods will crash

```

---

### Image Signing and Verification

```bash

PROBLEM: how do you know the image you're pulling

hasn't been tampered with?



COSIGN (Sigstore — industry standard):

Sign:

cosign sign --key cosign.key my-registry/my-app:v1.2.3

Signature stored alongside image in registry



Verify:

cosign verify --key cosign.pub my-registry/my-app:v1.2.3

If signature doesn't match → verification fails



Keyless signing (using OIDC identity — CI pipeline identity):

cosign sign my-registry/my-app:v1.2.3

Uses GitHub Actions OIDC token / Fulcio CA

No key management needed

Signature tied to: "this image was built by THIS CI pipeline in THIS repo"



ENFORCE IN KUBERNETES:

Kyverno policy to require signed images:

apiVersion: kyverno.io/v1

kind: ClusterPolicy

metadata:

 name: verify-images

spec:

 validationFailureAction: Enforce

 rules:

 - name: verify-cosign

   match:

     any:

     - resources:

         kinds:

         - Pod

   verifyImages:

   - imageReferences:

     - "my-registry/\*"

     attestors:

     - entries:

       - keyless:

           issuer: "https://token.actions.githubusercontent.com"

           subject: "https://github.com/myorg/\*"



DOCKER CONTENT TRUST (DCT):

export DOCKER\_CONTENT\_TRUST=1

docker push my-app:v1.2.3

Push signs the image with Notary

docker pull my-app:v1.2.3

Pull verifies signature

Any unsigned image → pull rejected



DCT is older, less flexible than Cosign

Industry is moving toward Cosign/Sigstore

```

---

### Docker-in-Docker vs Docker-out-of-Docker

```bash

CI pipelines need to build Docker images.

How do you build images inside a container?



DOCKER-IN-DOCKER (DinD):

Run a full Docker daemon INSIDE a container

docker run --privileged -d docker:dind

The inner Docker daemon is completely separate

Builds happen inside the inner daemon

\#

PROBLEMS:

--privileged = full host capabilities = SECURITY NIGHTMARE

File system layers get nested (slow, complex)

Build cache is lost when container stops

NOT recommended for production CI



DOCKER-OUT-OF-DOCKER (DooD):

Mount the HOST's Docker socket into the container

docker run -v /var/run/docker.sock:/var/run/docker.sock docker

Container uses the HOST's Docker daemon

Builds appear on the host

Cache is shared with host

\#

PROBLEMS:

Container has FULL CONTROL of host Docker

Can start privileged containers, mount host filesystem

Effectively root on the host

Containers built are siblings, not children (confusing networking)



KANIKO (the right answer for Kubernetes):

Builds OCI images inside a container WITHOUT Docker daemon

No privileged mode, no Docker socket

apiVersion: v1

kind: Pod

metadata:

 name: kaniko-build

spec:

 containers:

 - name: kaniko

   image: gcr.io/kaniko-project/executor:latest

   args:

   - "--dockerfile=Dockerfile"

   - "--context=git://github.com/myorg/myrepo"

   - "--destination=my-registry/my-app:v1.2.3"

   - "--cache=true"                 # layer caching via registry

   - "--cache-repo=my-registry/my-app/cache"

   volumeMounts:

   - name: docker-config

     mountPath: /kaniko/.docker/

 volumes:

 - name: docker-config

   secret:

     secretName: docker-registry-creds

No privileged mode, no Docker socket

Builds happen in userspace using snapshotting

Standard for Kubernetes-based CI (Jenkins on K8s, Tekton, etc.)



BUILDAH (daemonless alternative):

buildah bud -t my-app:v1.2.3 .

buildah push my-app:v1.2.3 docker://my-registry/my-app:v1.2.3

No daemon required

Can run rootless

OCI-compliant images

Good for: non-Kubernetes CI environments

```

---

### Docker Inspect and Filesystem Debugging

```bash

INSPECT — full container metadata:

docker inspect <container>

Returns JSON with EVERYTHING:

- State (running, paused, dead, exit code, OOMKilled, pid)

- NetworkSettings (IP, ports, networks)

- Mounts (volumes, bind mounts)

- Config (env vars, cmd, entrypoint, labels)

- HostConfig (resources, restart policy, capabilities)



Useful filters:

docker inspect --format='{{.State.ExitCode}}' <container>

docker inspect --format='{{.State.OOMKilled}}' <container>

docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container>

docker inspect --format='{{json .Config.Env}}' <container> | jq

docker inspect --format='{{.HostConfig.Memory}}' <container>



DIFF — see what changed in the container filesystem:

docker diff <container>

A /app/data/new-file.txt          (Added)

C /var/log                         (Changed)

D /tmp/old-file.txt               (Deleted)

Useful for: debugging unexpected writes, auditing changes



CP — copy files in/out:

docker cp <container>:/app/config.yaml ./config.yaml    # out

docker cp ./fix.py <container>:/app/fix.py              # in

Use for: extracting logs, injecting hotfixes (NOT for production)



EXPORT — full container filesystem as tar:

docker export <container> > container-fs.tar

For deep forensic analysis



HISTORY — see how image was built:

docker history my-app:v1.2.3 --no-trunc

Shows every layer, command that created it, size

Useful for: understanding third-party images, finding bloat



EVENTS — real-time Docker daemon events:

docker events

container create, start, die, kill, oom, health\_status

image pull, push, tag, delete

volume create, mount, unmount

network connect, disconnect

Useful for: debugging startup issues, OOM events

docker events --filter event=oom --filter event=die

Monitor only death and OOM events

```

---

### Docker Compose Production Patterns

```yaml

DEVELOPMENT vs PRODUCTION compose:



docker-compose.yml (base):

services:

 api:

   image: my-app:${APP\_VERSION}

   restart: unless-stopped

   healthcheck:

     test: \["CMD", "curl", "-f", "http://localhost:8080/health"]

     interval: 30s

     timeout: 5s

     retries: 3

     start\_period: 40s     # don't check health during startup



docker-compose.override.yml (auto-loaded, development):

services:

 api:

   build:

     context: .

     target: development

   volumes:

     - ./src:/app/src     # live reload

   ports:

     - "8080:8080"        # expose to host

   environment:

     - DEBUG=true



docker-compose.prod.yml (explicit, production):

services:

 api:

   deploy:

     replicas: 3

     resources:

       limits:

         cpus: '2'

         memory: 1G

       reservations:

         cpus: '0.5'

         memory: 256M

   logging:

     driver: json-file

     options:

       max-size: "50m"

       max-file: "3"



Usage:

Dev (auto-loads override):

docker compose up



Prod (explicit file):

docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d



PROFILES (selective service startup):

services:

 api:

   profiles: \["app"]

 worker:

   profiles: \["app"]

 debug-tools:

   profiles: \["debug"]       # only started when explicitly requested

 db:

   # no profile = always started



docker compose --profile app up        # api + worker + db

docker compose --profile debug up      # debug-tools + db



DEPENDS\_ON with conditions:

services:

 api:

   depends\_on:

     db:

       condition: service\_healthy      # wait for health check

     redis:

       condition: service\_started      # just wait for start

     migrate:

       condition: service\_completed\_successfully  # wait for completion

```

---

### Container Resource Visibility — /proc and Cgroup Awareness

```bash

CRITICAL ISSUE:

Inside a container, /proc/meminfo and /proc/cpuinfo show HOST values

Not the container's cgroup limits



cat /proc/meminfo

MemTotal: 65536000 kB    ← HOST has 64GB

But container is limited to 512MB via cgroup



cat /proc/cpuinfo | grep processor | wc -l

16                        ← HOST has 16 CPUs

But container is limited to 2 CPUs



WHY THIS MATTERS:

Applications that auto-configure based on /proc:

- JVM: calculates heap from /proc/meminfo (old JVMs)

- Node.js: UV\_THREADPOOL\_SIZE defaults to 4, could set based on CPUs

- Go: GOMAXPROCS defaults to /proc/cpuinfo CPU count

- Python multiprocessing: os.cpu\_count() reads /proc/cpuinfo



FIXES:

Java 10+: reads cgroup limits automatically (UseContainerSupport)

Go: use automaxprocs library (uber-go/automaxprocs)

  import \_ "go.uber.org/automaxprocs"

  Reads cgroup CPU quota, sets GOMAXPROCS correctly

Node.js: UV\_THREADPOOL\_SIZE=<your CPU limit>

Python: read cgroup directly or use container-aware libraries



LXCFS (advanced):

Mounts cgroup-aware /proc into container

/proc/meminfo shows container's memory limit, not host's

/proc/cpuinfo shows container's CPU limit

Used in: multi-tenant environments where apps MUST see correct resources

Not common in K8s (most apps handle this natively now)

```

---

### Container Networking — Advanced Patterns

```bash

DNS RESOLUTION IN CONTAINERS:



Default bridge network:

docker run --name web nginx

docker run --name app myapp

app CANNOT reach web by name "web"

Must use IP address (which changes on restart)

Default bridge: no DNS resolution between containers



Custom bridge network:

docker network create mynet

docker run --network mynet --name web nginx

docker run --network mynet --name app myapp

app CAN reach web by name "web"

Docker's embedded DNS server resolves container names

DNS server at 127.0.0.11 inside the container



MULTIPLE NETWORKS (isolation):

docker network create frontend

docker network create backend

docker run --network frontend --name web nginx

docker run --network frontend --network backend --name api myapp

docker run --network backend --name db postgres

web can reach api (both on frontend)

api can reach db (both on backend)

web CANNOT reach db (different networks, no route)

This is network segmentation without firewall rules



ALIAS:

docker run --network mynet --network-alias search elasticsearch

docker run --network mynet --network-alias search elasticsearch

Both containers respond to DNS name "search"

DNS round-robin load balancing

Primitive but functional for development



HOST NETWORKING WITH SPECIFIC PORTS:

--network=host bypasses ALL Docker networking

Container uses host's network stack directly

Performance: eliminates NAT overhead (\~5-10% improvement in high-throughput)

Use case: performance-critical network applications, monitoring tools

Trade-off: no network isolation, port conflicts



CONTAINER NETWORKING:

docker run --network container:<other-container-name> myapp

Shares ANOTHER container's network namespace

Same IP, same ports, same interfaces

Use case: sidecar pattern without Kubernetes

Debug container sharing network with target container

```

---

### Health Checks — Docker vs Kubernetes

```bash

DOCKER HEALTHCHECK:

In Dockerfile:

HEALTHCHECK --interval=30s --timeout=5s --retries=3 --start-period=60s \\

 CMD curl -f http://localhost:8080/health || exit 1



Or at runtime:

docker run \\

 --health-cmd="curl -f http://localhost:8080/health || exit 1" \\

 --health-interval=30s \\

 --health-timeout=5s \\

 --health-retries=3 \\

 --health-start-period=60s \\

 myapp



States:

starting: within start-period, health not yet checked

healthy: health check passing

unhealthy: failed retries threshold



Docker health check vs K8s probes:

Docker HEALTHCHECK: only affects container STATUS display

  Docker restart policy can use it (restart unhealthy containers)

  docker compose depends\_on: service\_healthy uses it

  But it does NOT control traffic routing

\#

K8s probes: control EVERYTHING

  Liveness: restart on failure

  Readiness: remove from Service endpoints on failure

  Startup: gate liveness/readiness until ready

\#

In K8s: Docker HEALTHCHECK is IGNORED

K8s uses its own probes defined in pod spec

Don't rely on Dockerfile HEALTHCHECK for K8s workloads

But keep it for: docker-compose, standalone Docker, ECS

```

---

### Docker Build Optimization — Advanced

```bash

BUILDKIT CACHE MOUNTS:

Persist package manager caches across builds



Go:

RUN --mount=type=cache,target=/go/pkg/mod \\

   --mount=type=cache,target=/root/.cache/go-build \\

   go build -o /app/server



Python:

RUN --mount=type=cache,target=/root/.cache/pip \\

   pip install -r requirements.txt



Node.js:

RUN --mount=type=cache,target=/root/.cache/npm \\

   npm ci



apt:

RUN --mount=type=cache,target=/var/cache/apt \\

   --mount=type=cache,target=/var/lib/apt/lists \\

   apt-get update \&\& apt-get install -y curl



RESULT: package downloads cached across ALL builds

Even if requirements.txt changes, only new packages downloaded



SECRET MOUNTS (BuildKit):

Pass secrets without embedding in image layers:

RUN --mount=type=secret,id=npm\_token \\

   NPM\_TOKEN=$(cat /run/secrets/npm\_token) npm install

Build:

docker build --secret id=npm\_token,src=.npmrc .

Secret available only during that RUN instruction

NOT stored in any image layer

NOT visible in docker history



SSH MOUNTS (BuildKit):

Forward SSH agent for private repo access:

RUN --mount=type=ssh git clone git@github.com:private/repo.git

Build:

docker build --ssh default .

SSH keys never touch the image



REGISTRY CACHE:

docker buildx build \\

 --cache-from type=registry,ref=my-registry/my-app:cache \\

 --cache-to type=registry,ref=my-registry/my-app:cache,mode=max \\

 --push \\

 -t my-registry/my-app:v1.2.3 .

Pushes build cache to registry

Next build pulls cache from registry

Works across different CI runners (no local cache dependency)

mode=max: cache ALL layers (not just final stage)



BUILD ARGS vs ENV:

ARG BUILD\_VERSION=unknown    # available ONLY during build

ENV APP\_VERSION=${BUILD\_VERSION}  # persisted in image

ARG: use for build-time configuration (versions, flags)

ENV: use for runtime configuration

SECURITY: ARG values visible in docker history!

Never pass secrets via ARG

```

---

### Docker Troubleshooting Playbook

```bash

CONTAINER WON'T START:

docker logs <container>                    # check stdout/stderr

docker inspect <container> | jq '.State'   # check exit code, OOMKilled

Exit code 0: clean exit (CMD finished)

Exit code 1: application error

Exit code 126: permission denied (can't execute CMD)

Exit code 127: command not found (CMD doesn't exist in image)

Exit code 137: SIGKILL (OOM killed or docker kill)

Exit code 139: SIGSEGV (segfault — native code crash)

Exit code 143: SIGTERM (docker stop, graceful)



HIGH DISK USAGE:

docker system df                           # overview

docker system df -v                        # detailed

Check: dangling images, stopped containers, unused volumes

Check: container log sizes

du -sh /var/lib/docker/containers/\*/      # log file sizes per container



NETWORKING ISSUES:

docker exec <container> cat /etc/resolv.conf  # DNS config

docker exec <container> ping <other-container>

docker network inspect <network>              # see connected containers

iptables -t nat -L -n | grep DOCKER           # see port mappings



IMAGE WON'T PULL:

1. Auth: docker login / ECR token expired

2. Network: DNS resolution, proxy settings

3. Rate limit: Docker Hub limits (100 pulls/6h anonymous)

   Fix: use ECR pull-through cache or mirror

4. Image doesn't exist for your architecture (arm64 vs amd64)



CONTAINER RUNNING BUT APP NOT RESPONDING:

docker exec <container> ss -tlnp           # is app listening?

docker exec <container> curl localhost:8080 # can app reach itself?

docker port <container>                     # port mapping correct?

If app binds to 127.0.0.1 inside container → not reachable from outside

Must bind to 0.0.0.0 inside container for port mapping to work

```

===================================================================================
# PHASE 2 — Lesson 3: Kubernetes Architecture & Core Objects

We covered K8s networking in Phase 1. Now we go deeper into the control plane, how scheduling works, and the core objects you'll manage daily.

---

### The Control Plane — What Runs Kubernetes

```

┌─────────────────────── CONTROL PLANE ───────────────────────┐

│                                                              │

│  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐  │

│  │  API Server   │  │   Scheduler   │  │ Controller       │  │

│  │  (kube-apiserver)│ (kube-scheduler)│ │ Manager          │  │

│  │               │  │               │  │ (kube-controller │  │

│  │ THE gatekeeper│  │ WHERE does    │  │  -manager)       │  │

│  │ ALL traffic   │  │ this pod go?  │  │                  │  │

│  │ goes through  │  │               │  │ Reconciliation   │  │

│  │ here. Period. │  │               │  │ loops. Desired   │  │

│  └──────┬───────┘  └───────────────┘  │ vs actual state. │  │

│         │                              └──────────────────┘  │

│         │                                                    │

│  ┌──────▼───────┐  ┌───────────────────────────────────┐     │

│  │    etcd       │  │  Cloud Controller Manager         │     │

│  │               │  │  (cloud-specific: AWS, GCP, Azure)│     │

│  │ The database. │  │  Manages: LBs, nodes, routes      │     │

│  │ All cluster   │  │  EKS: runs as AWS-managed service  │     │

│  │ state lives   │  └───────────────────────────────────┘     │

│  │ here.         │                                           │

│  └──────────────┘                                            │

└──────────────────────────────────────────────────────────────┘



┌─────────────────────── WORKER NODE ─────────────────────────┐

│                                                              │

│  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐  │

│  │   kubelet     │  │  kube-proxy   │  │  Container       │  │

│  │               │  │               │  │  Runtime          │  │

│  │ Agent on each │  │ Network rules │  │  (containerd)    │  │

│  │ node. Talks   │  │ iptables/IPVS │  │                  │  │

│  │ to API server.│  │ for Services  │  │  Actually runs   │  │

│  │ Ensures pods  │  │               │  │  containers      │  │

│  │ are running.  │  │               │  │                  │  │

│  └──────────────┘  └───────────────┘  └──────────────────┘  │

│                                                              │

│  ┌──────────────────────────────────────────────────────┐    │

│  │  Pods    \[ container ] \[ container ] \[ container ]    │    │

│  └──────────────────────────────────────────────────────┘    │

└──────────────────────────────────────────────────────────────┘

```

### What Each Component Actually Does

```bash

API SERVER (kube-apiserver):

- REST API endpoint for ALL cluster operations

- ONLY component that talks to etcd directly

- Authenticates requests (certs, tokens, OIDC)

- Authorizes requests (RBAC)

- Validates objects (admission controllers)

- Serves watch streams (controllers subscribe to changes)

kubectl → API server → etcd

Scheduler → API server → etcd

kubelet → API server → etcd

EVERYTHING goes through the API server. No exceptions.



ETCD:

- Distributed key-value store (Raft consensus)

- Stores ALL cluster state: pods, services, secrets, configmaps

- NOT a general-purpose database — don't abuse it

- Performance sensitive: SSD required, latency < 10ms

- Backup etcd = backup your entire cluster state

- EKS: managed by AWS (you don't touch it)

- Self-managed: 3 or 5 node etcd cluster (odd number for Raft quorum)

\#

Key structure:

/registry/pods/default/my-pod → pod spec JSON

/registry/services/default/my-service → service spec JSON

/registry/secrets/default/my-secret → encrypted secret



SCHEDULER (kube-scheduler):

Watches for unscheduled pods (pods with no node assigned)

Decision process:

  1. FILTERING: eliminate nodes that can't run the pod

     - Insufficient CPU/memory

     - Node taints that pod doesn't tolerate

     - Node affinity/anti-affinity rules

     - PV topology constraints

  2. SCORING: rank remaining nodes

     - Spread pods across nodes (LeastRequestedPriority)

     - Pack pods tightly (MostRequestedPriority) — for cost

     - Prefer nodes with image already cached

     - Topology spread constraints

  3. BINDING: assign pod to highest-scoring node

     - Writes node name to pod spec via API server

\#

The scheduler ONLY decides WHERE. It doesn't start anything.

kubelet on the chosen node sees the assignment and starts the pod.



CONTROLLER MANAGER:

Runs \~30+ control loops, each watching a resource type:

\#

Deployment controller:

  Watches Deployments → creates/updates ReplicaSets

ReplicaSet controller:

  Watches ReplicaSets → creates/deletes Pods to match desired count

Node controller:

  Watches node heartbeats → marks nodes NotReady after timeout

Job controller:

  Watches Jobs → creates Pods, tracks completion

Endpoint controller:

  Watches Services + Pods → creates Endpoint objects

ServiceAccount controller:

  Creates default ServiceAccount for new namespaces

\#

Each controller follows the same pattern:

  1. Watch desired state (spec)

  2. Observe actual state (status)

  3. Take action to reconcile (create/delete/update resources)

  4. Repeat forever

This is the "declarative model" — you declare WHAT, controllers figure out HOW



KUBELET:

Agent on every node. Responsibilities:

  - Registers node with API server

  - Watches for pods assigned to this node

  - Pulls images, creates containers via container runtime

  - Monitors pod health (liveness/readiness probes)

  - Reports node status (capacity, conditions) to API server

  - Mounts volumes, manages secrets/configmaps

  - Evicts pods if node is under pressure (disk, memory, PIDs)

\#

kubelet does NOT manage pods not created by the API server

(except static pods defined in /etc/kubernetes/manifests/)

```

---

### How Pod Creation Actually Works — The Full Chain

```

kubectl apply -f pod.yaml



Step 1: kubectl sends POST /api/v1/namespaces/default/pods to API server



Step 2: API server:

 a) Authenticates (is this user who they claim to be?)

 b) Authorizes via RBAC (can this user create pods in default namespace?)

 c) Runs Admission Controllers:

    - Mutating: inject sidecar (Istio), add labels, set defaults

    - Validating: check resource limits exist, enforce policies (OPA)

 d) Persists pod to etcd (status: Pending, no node assigned)



Step 3: Scheduler watches API server (via watch stream):

 - Sees new pod with no nodeName

 - Runs filtering → scoring → selects best node

 - PATCHes pod with nodeName via API server → etcd updated



Step 4: kubelet on selected node watches API server:

 - Sees pod assigned to this node

 - Pulls image (if not cached) via container runtime

 - Creates containers (containerd → runc)

 - Sets up networking (CNI plugin creates veth, assigns IP)

 - Mounts volumes

 - Starts containers

 - Reports pod status back to API server → etcd updated

 - Status: Running



Step 5: kube-proxy on ALL nodes:

 - If pod matches a Service selector

 - Endpoint controller adds pod IP to Endpoints

 - kube-proxy on every node updates iptables/IPVS rules

 - Pod is now reachable via Service ClusterIP



TOTAL TIME: typically 2-10 seconds for a simple pod

```

---

### Pods — The Atomic Unit

```yaml

apiVersion: v1

kind: Pod

metadata:

 name: my-app

 labels:

   app: my-app

   version: v1

spec:

 # INIT CONTAINERS — run BEFORE main containers, sequentially

 initContainers:

 - name: db-migration

   image: my-app:v1.2.3

   command: \["./migrate", "--up"]

   # Must complete successfully before main containers start

   # Use for: DB migrations, config generation, wait-for-dependency



 containers:

 - name: app

   image: my-app:v1.2.3

   ports:

   - containerPort: 8080

   

   # RESOURCE MANAGEMENT:

   resources:

     requests:

       cpu: 250m        # 0.25 CPU cores — used for SCHEDULING

       memory: 256Mi    # used for scheduling

     limits:

       cpu: 1000m       # 1 CPU core — hard ceiling (throttled beyond this)

       memory: 512Mi    # hard ceiling (OOM killed beyond this)

   

   # requests = what scheduler uses to find a node with enough capacity

   # limits = what cgroups enforce at runtime

   # requests < limits = "burstable" — can use more if available

   # requests = limits = "guaranteed" QoS class (highest priority)

   # no requests/limits = "best effort" QoS class (first to be evicted)



   # PROBES:

   livenessProbe:

     httpGet:

       path: /healthz

       port: 8080

     initialDelaySeconds: 15    # wait before first check

     periodSeconds: 10          # check every 10s

     failureThreshold: 3        # 3 failures → restart container

     # LIVENESS = "is the process alive?"

     # Failure action: RESTART the container

     # Use for: detecting deadlocks, infinite loops, hung processes

     # DON'T check dependencies here (DB, cache) — will cause restart loops



   readinessProbe:

     httpGet:

       path: /ready

       port: 8080

     initialDelaySeconds: 5

     periodSeconds: 5

     failureThreshold: 3

     # READINESS = "is the process ready to receive traffic?"

     # Failure action: REMOVE from Service endpoints (no traffic sent)

     # Use for: warming caches, waiting for dependencies, graceful drain

     # Check dependencies here — if DB is down, stop sending traffic



   startupProbe:

     httpGet:

       path: /healthz

       port: 8080

     failureThreshold: 30

     periodSeconds: 10

     # STARTUP = "has the process finished starting?"

     # While startup probe is running, liveness/readiness are DISABLED

     # failureThreshold × periodSeconds = 300s = 5 min max startup time

     # Use for: slow-starting apps (Java, large ML models)

     # Without this: liveness probe kills the app before it finishes booting



   # LIFECYCLE HOOKS:

   lifecycle:

     postStart:

       exec:

         command: \["/bin/sh", "-c", "echo started > /tmp/started"]

       # Runs AFTER container starts, but NOT guaranteed before ENTRYPOINT

       # Runs in parallel with the main process

       

     preStop:

       exec:

         command: \["/bin/sh", "-c", "sleep 10"]

       # Runs BEFORE container is sent SIGTERM

       # Use for: graceful shutdown, deregister from service discovery

       # The sleep 10 trick: gives kube-proxy time to update iptables

       # and stop routing traffic BEFORE the app shuts down

       # Without this: app receives SIGTERM, starts shutting down,

       # but kube-proxy hasn't updated yet → requests hit a dying pod → 502s



 # TERMINATION SEQUENCE:

 terminationGracePeriodSeconds: 30   # default: 30

 # 1. Pod marked for deletion

 # 2. Endpoints controller removes pod from Service endpoints

 # 3. preStop hook runs (if defined)

 # 4. SIGTERM sent to PID 1 in container

 # 5. Wait up to terminationGracePeriodSeconds

 # 6. SIGKILL sent (unconditional kill)

 # Steps 2 and 3 happen IN PARALLEL — this is why preStop sleep matters



 # VOLUMES:

 volumes:

 - name: config

   configMap:

     name: app-config

 - name: secrets

   secret:

     secretName: app-secrets

 - name: tmp

   emptyDir:

     medium: Memory

     sizeLimit: 64Mi

 - name: data

   persistentVolumeClaim:

     claimName: app-data

```

---

### Deployments and ReplicaSets

```yaml

apiVersion: apps/v1

kind: Deployment

metadata:

 name: api

 namespace: production

spec:

 replicas: 3

 revisionHistoryLimit: 5      # keep 5 old ReplicaSets for rollback

 

 selector:

   matchLabels:

     app: api                  # MUST match template labels

 

 strategy:

   type: RollingUpdate

   rollingUpdate:

     maxSurge: 1              # can create 1 extra pod during update (4 total)

     maxUnavailable: 0        # never have fewer than 3 running

     # maxSurge=1 + maxUnavailable=0 = safest, slowest

     # maxSurge=25% + maxUnavailable=25% = default, balanced

     # maxSurge=0 + maxUnavailable=1 = resource-constrained (no extra pod)



 template:

   metadata:

     labels:

       app: api

   spec:

     containers:

     - name: api

       image: my-api:v1.2.3

       # ... probes, resources, etc.

```

### What Happens During a Rolling Update

```

Current state: 3 pods running v1



kubectl set image deployment/api api=my-api:v1.3.0

(or kubectl apply with updated image tag)



With maxSurge=1, maxUnavailable=0:



Step 1: Create new ReplicaSet (v1.3.0) with 1 replica

Total pods: 3 (v1) + 1 (v1.3.0) = 4



Step 2: Wait for v1.3.0 pod to be Ready (readiness probe passes)



Step 3: Scale down old ReplicaSet to 2

Total pods: 2 (v1) + 1 (v1.3.0) = 3



Step 4: Scale up new ReplicaSet to 2

Total pods: 2 (v1) + 2 (v1.3.0) = 4



Step 5: Wait for second v1.3.0 pod to be Ready



Step 6: Scale down old to 1, scale up new to 3

... repeat until all pods are v1.3.0



Old ReplicaSet kept (scaled to 0) for rollback



ROLLBACK:

kubectl rollout undo deployment/api

Scales old ReplicaSet back up, scales new one down

Instant — no image pull needed (old pods were just scaled to 0)



kubectl rollout undo deployment/api --to-revision=3

Roll back to specific revision



kubectl rollout history deployment/api

See all revisions and their change causes



kubectl rollout status deployment/api

Watch rollout progress in real time

```

### Deployment vs StatefulSet vs DaemonSet vs Job

```

DEPLOYMENT:

 - Stateless workloads

 - Pods are interchangeable (no identity)

 - Pod names: api-deployment-7b8f9c6d4-x2k9w (random suffix)

 - Rolling updates, rollback

 - Use for: APIs, web servers, workers



STATEFULSET:

 - Stateful workloads

 - Pods have STABLE identity: db-0, db-1, db-2

 - Ordered creation: db-0 must be running before db-1 starts

 - Ordered deletion: db-2 deleted before db-1

 - Stable DNS: db-0.db-headless.namespace.svc.cluster.local

 - Each pod gets its own PVC (not shared)

 - Use for: databases, Kafka, ZooKeeper, Redis Cluster, etcd

 

 # REQUIRES headless service:

 apiVersion: v1

 kind: Service

 metadata:

   name: db-headless

 spec:

   clusterIP: None            # headless

   selector:

     app: database

   ports:

   - port: 5432



DAEMONSET:

 - Runs ONE pod on EVERY node (or subset via nodeSelector)

 - New node joins → pod automatically scheduled

 - Node removed → pod removed

 - Use for: log collectors (Fluentd), monitoring agents (node-exporter),

   CNI plugins (Calico), storage daemons (EBS CSI), kube-proxy

 - Updates: rolling or OnDelete



JOB:

 - Run-to-completion workload

 - Pod runs, completes, exits

 - Retries on failure (backoffLimit)

 - parallelism: how many pods run simultaneously

 - completions: how many successful completions needed

 - Use for: database migrations, batch processing, backups



CRONJOB:

 - Job on a schedule (cron syntax)

 - Creates Job objects at scheduled times

 - concurrencyPolicy:

     Allow: multiple jobs can run simultaneously

     Forbid: skip if previous still running

     Replace: kill previous, start new

 - Use for: periodic backups, report generation, cleanup tasks

```

---

### ConfigMaps and Secrets

```yaml

CONFIGMAP — non-sensitive configuration:

apiVersion: v1

kind: ConfigMap

metadata:

 name: app-config

data:

 DATABASE\_HOST: "db.production.svc.cluster.local"

 LOG\_LEVEL: "info"

 config.yaml: |

   server:

     port: 8080

     timeout: 30s

   features:

     new\_checkout: true



Mount as environment variables:

env:

\- name: DATABASE\_HOST

 valueFrom:

   configMapKeyRef:

     name: app-config

     key: DATABASE\_HOST



Mount as file:

volumeMounts:

\- name: config

 mountPath: /etc/app/config.yaml

 subPath: config.yaml          # mount single file, not entire dir

volumes:

\- name: config

 configMap:

   name: app-config



AUTO-RELOAD:

Environment variables: NOT updated on ConfigMap change (requires pod restart)

Mounted files: updated automatically (\~60-90 seconds)

BUT: app must watch the file for changes (not all apps do)



SECRET — sensitive data:

apiVersion: v1

kind: Secret

metadata:

 name: app-secrets

type: Opaque

data:

 DB\_PASSWORD: cGFzc3dvcmQxMjM=     # base64 encoded (NOT encrypted!)

 API\_KEY: c2VjcmV0a2V5MTIz



base64 is NOT encryption. Anyone with cluster access can decode:

echo "cGFzc3dvcmQxMjM=" | base64 -d

password123



ENCRYPTING SECRETS AT REST:

EKS: enable envelope encryption with KMS

AWS encrypts the etcd data using a CMK

Without this: secrets stored in etcd in plaintext



EXTERNAL SECRETS (production approach):

Don't store secrets in K8s at all

Use: External Secrets Operator → pulls from:

  - AWS Secrets Manager

  - HashiCorp Vault

  - Azure Key Vault

  - GCP Secret Manager

Syncs external secrets into K8s Secret objects automatically

Rotation handled by the external provider

```

---

### Namespaces — Not Just Organization

```bash

Namespaces are more than folders. They're:

1. RBAC boundary (who can do what WHERE)

2. Resource quota boundary (how much a team can consume)

3. Network policy boundary (which pods can talk to which)

4. Service DNS scope (svc-name.namespace.svc.cluster.local)



Resource Quotas:

apiVersion: v1

kind: ResourceQuota

metadata:

 name: team-alpha-quota

 namespace: team-alpha

spec:

 hard:

   requests.cpu: "20"          # total CPU requests across all pods

   requests.memory: 40Gi

   limits.cpu: "40"

   limits.memory: 80Gi

   pods: "100"                 # max 100 pods in this namespace

   services: "20"

   persistentvolumeclaims: "30"



LimitRange — defaults and constraints per pod/container:

apiVersion: v1

kind: LimitRange

metadata:

 name: default-limits

 namespace: team-alpha

spec:

 limits:

 - type: Container

   default:                    # applied if no limits specified

     cpu: 500m

     memory: 256Mi

   defaultRequest:             # applied if no requests specified

     cpu: 100m

     memory: 128Mi

   max:                        # hard ceiling per container

     cpu: 4

     memory: 8Gi

   min:                        # minimum per container

     cpu: 50m

     memory: 64Mi



WHY THIS MATTERS:

Without LimitRange, a developer can deploy a pod with no resource limits

That pod can consume ALL node resources → starves other pods

LimitRange ensures every container has at least default limits

ResourceQuota ensures a namespace can't consume the entire cluster

```

---

### Scheduling — Taints, Tolerations, Affinity

```yaml

TAINTS — "this node repels pods unless they tolerate the taint"

Applied to NODES:

kubectl taint nodes node1 dedicated=gpu:NoSchedule

Only pods that tolerate "dedicated=gpu" can be scheduled here



Three taint effects:

NoSchedule: don't schedule new pods (existing pods stay)

PreferNoSchedule: try to avoid, but allow if necessary

NoExecute: evict existing pods AND don't schedule new ones



TOLERATIONS — "this pod can tolerate this taint"

Applied to PODS:

spec:

 tolerations:

 - key: "dedicated"

   operator: "Equal"

   value: "gpu"

   effect: "NoSchedule"



Common pattern — dedicated node pools:

Taint GPU nodes: dedicated=gpu:NoSchedule

ML pods tolerate the taint → scheduled on GPU nodes

Regular pods don't tolerate → stay on regular nodes



NODE AFFINITY — "this pod prefers/requires specific nodes"

spec:

 affinity:

   nodeAffinity:

     requiredDuringSchedulingIgnoredDuringExecution:

       nodeSelectorTerms:

       - matchExpressions:

         - key: topology.kubernetes.io/zone

           operator: In

           values:

           - us-east-1a

           - us-east-1b

       # Pod MUST run in us-east-1a or us-east-1b

       

     preferredDuringSchedulingIgnoredDuringExecution:

     - weight: 80

       preference:

         matchExpressions:

         - key: node.kubernetes.io/instance-type

           operator: In

           values:

           - m5.xlarge

       # PREFER m5.xlarge, but don't fail if unavailable



POD ANTI-AFFINITY — "spread my replicas across nodes/zones"

spec:

 affinity:

   podAntiAffinity:

     requiredDuringSchedulingIgnoredDuringExecution:

     - labelSelector:

         matchExpressions:

         - key: app

           operator: In

           values:

           - api

       topologyKey: kubernetes.io/hostname

       # NEVER put two api pods on the same node

       

     # Or use topology spread constraints (more flexible):

 topologySpreadConstraints:

 - maxSkew: 1                    # max difference in pod count across zones

   topologyKey: topology.kubernetes.io/zone

   whenUnsatisfiable: DoNotSchedule

   labelSelector:

     matchLabels:

       app: api

   # Spread api pods evenly across AZs

   # 3 pods, 3 AZs → 1 per AZ

   # 4 pods, 3 AZs → 2-1-1 (skew=1 allowed)

```

---

### RBAC — Who Can Do What

```yaml

RBAC has 4 objects:

Role/ClusterRole: WHAT actions are allowed

RoleBinding/ClusterRoleBinding: WHO gets those permissions



Role — namespace-scoped:

apiVersion: rbac.authorization.k8s.io/v1

kind: Role

metadata:

 name: pod-reader

 namespace: production

rules:

\- apiGroups: \[""]           # "" = core API group (pods, services, etc.)

 resources: \["pods"]

 verbs: \["get", "list", "watch"]

\- apiGroups: \[""]

 resources: \["pods/log"]    # subresource

 verbs: \["get"]



ClusterRole — cluster-wide:

apiVersion: rbac.authorization.k8s.io/v1

kind: ClusterRole

metadata:

 name: node-reader

rules:

\- apiGroups: \[""]

 resources: \["nodes"]

 verbs: \["get", "list", "watch"]

Nodes are cluster-scoped — can't be in a namespace Role



RoleBinding — binds Role to users/groups/service accounts:

apiVersion: rbac.authorization.k8s.io/v1

kind: RoleBinding

metadata:

 name: read-pods

 namespace: production

subjects:

\- kind: User

 name: jane@company.com

 apiGroup: rbac.authorization.k8s.io

\- kind: Group

 name: developers

 apiGroup: rbac.authorization.k8s.io

\- kind: ServiceAccount

 name: ci-pipeline

 namespace: ci

roleRef:

 kind: Role

 name: pod-reader

 apiGroup: rbac.authorization.k8s.io



TEST RBAC:

kubectl auth can-i create pods --namespace production --as jane@company.com

yes / no



kubectl auth can-i '\*' '\*' --as system:serviceaccount:ci:ci-pipeline

Check if SA has full access (should be no)



PRINCIPLE OF LEAST PRIVILEGE:

Don't give cluster-admin to CI pipelines

Don't give \* verbs unless absolutely necessary

Audit with: kubectl auth can-i --list --as <user>

```

---

### Production Scenarios

#### Scenario 1: Pod Stuck in Pending

```bash

kubectl get pods

NAME         READY   STATUS    AGE

api-xyz      0/1     Pending   15m



kubectl describe pod api-xyz

Events:

"0/10 nodes are available: 4 Insufficient cpu, 

 3 node(s) had taint {dedicated=gpu:NoSchedule}, 

 3 node(s) didn't match Pod's node affinity/selector"



Translation:

10 nodes total

4 nodes: not enough CPU for the pod's requests

3 nodes: GPU-tainted, pod doesn't tolerate

3 nodes: wrong AZ (node affinity constraint)

Result: 0 eligible nodes



Debug checklist:

1. Check resource requests vs node capacity:

kubectl describe nodes | grep -A 5 "Allocated resources"

2. Check taints:

kubectl describe nodes | grep Taints

3. Check affinity rules in pod spec

4. Check ResourceQuota:

kubectl describe quota -n production

Maybe namespace has hit its pod or CPU quota



Fix depends on root cause:

- Scale up node group (cluster autoscaler should handle this)

- Reduce resource requests

- Add tolerations

- Relax affinity constraints

- Increase ResourceQuota

```

#### Scenario 2: Deployment Rollout Stuck

```bash

kubectl rollout status deployment/api

Waiting for deployment "api" rollout to finish: 

1 old replicas are pending termination...



Why it sticks:

maxUnavailable=0 means old pod can't terminate until new pod is Ready

But new pod is never Ready because:



kubectl get pods

api-new-xxx   0/1   CrashLoopBackOff   5



kubectl logs api-new-xxx

Error: missing DATABASE\_URL environment variable



New image requires a new env var that wasn't added

New pod crashes → never becomes Ready

Old pod can't be terminated (maxUnavailable=0)

Rollout is deadlocked



Fix:

kubectl rollout undo deployment/api

Rolls back to previous ReplicaSet immediately

Then fix the ConfigMap/Secret, redeploy properly



PREVENTION:

progressDeadlineSeconds (default: 600 = 10 min)

If rollout hasn't progressed in 10 minutes:

Deployment condition: Progressing=False, reason=ProgressDeadlineExceeded

Alert on this in monitoring

```

#### Scenario 3: Pods Evicted — Node Under Pressure

```bash

kubectl get pods

api-abc   0/1   Evicted   0s



kubectl describe pod api-abc

Status: Failed

Reason: Evicted

Message: The node was low on resource: memory.



kubelet eviction order (by QoS class):

1. BestEffort (no requests/limits) — evicted FIRST

2. Burstable (requests < limits) — evicted SECOND

3. Guaranteed (requests = limits) — evicted LAST



If your production pods are Burstable and a BestEffort

monitoring agent isn't enough to free memory,

YOUR production pods get evicted



Fix:

1. Set requests = limits for critical pods (Guaranteed QoS)

2. Use PodDisruptionBudgets to limit concurrent evictions

3. Set proper resource requests so scheduler doesn't overcommit

4. Monitor node memory with alerts at 80% utilization

5. Use LimitRange to prevent pods with no limits

```

#### Scenario 4: 502 Errors During Deployment

```bash

Symptoms:

During rolling update, 1-2% of requests get 502

App logs show graceful shutdown completing normally



Root cause: RACE CONDITION between:

- Endpoints controller removing pod from Service (async)

- kubelet sending SIGTERM to pod (immediate)

These happen IN PARALLEL

Window: kube-proxy hasn't updated iptables yet, but pod is shutting down

Traffic still routes to dying pod → 502



Fix:

lifecycle:

 preStop:

   exec:

     command: \["sleep", "10"]

preStop runs BEFORE SIGTERM

During those 10 seconds:

- Endpoints controller removes pod from Service

- kube-proxy updates iptables on all nodes

- After 10s, SIGTERM sent to already-deregistered pod

No traffic hits the dying pod → no 502s



Also ensure:

terminationGracePeriodSeconds: 45

Must be > preStop sleep + app shutdown time

10s preStop + 30s app shutdown = 40s needed, 45s gives buffer

```

# 📋 QUICK REFERENCE — K8s Architecture & Core Objects (continued)

```

POD CREATION CHAIN:

 kubectl → API server (auth→RBAC→admission→etcd)

 → scheduler (filter→score→bind) → kubelet (pull→create→network→start)

 → kube-proxy (update endpoints/iptables on ALL nodes)



PROBES:

 Liveness:  is it alive?     Fail → restart container

 Readiness: is it ready?     Fail → remove from Service endpoints

 Startup:   has it started?  Disables liveness/readiness until pass

 DON'T check dependencies in liveness (restart loops)

 DO check dependencies in readiness (stop traffic)



QoS CLASSES:

 Guaranteed: requests = limits          (last to evict)

 Burstable:  requests < limits          (middle)

 BestEffort: no requests or limits      (first to evict)



RESOURCE MANAGEMENT:

 requests: scheduling decision (does node have capacity?)

 limits: runtime enforcement (cgroup hard ceiling)

 CPU over limit → throttled

 Memory over limit → OOM killed



TERMINATION SEQUENCE:

 1. Pod marked for deletion

 2. Endpoints controller removes from Service (async)

 3. preStop hook runs (parallel with step 2)

 4. SIGTERM sent to PID 1

 5. Wait terminationGracePeriodSeconds

 6. SIGKILL

 Steps 2+3 are parallel → preStop sleep(10) prevents 502s



WORKLOAD TYPES:

 Deployment:  stateless, rolling updates, rollback

 StatefulSet: stable identity (pod-0, pod-1), ordered, per-pod PVC

 DaemonSet:   one pod per node (logging, monitoring, CNI)

 Job:         run to completion, retries, parallelism

 CronJob:     Job on schedule, concurrencyPolicy



CONFIGMAP vs SECRET:

 ConfigMap: non-sensitive, plaintext

 Secret: base64 (NOT encrypted by default), enable KMS envelope encryption

 Env vars: NOT updated on change (requires restart)

 Mounted files: updated in \~60-90s (app must watch)

 Production: External Secrets Operator → Vault/AWS Secrets Manager



SCHEDULING:

 Taints (on nodes): repel pods unless tolerated

   NoSchedule / PreferNoSchedule / NoExecute

 Tolerations (on pods): allow scheduling on tainted nodes

 Node affinity: required vs preferred node selection

 Pod anti-affinity: spread replicas across nodes/zones

 TopologySpreadConstraints: maxSkew control across topology domains



RBAC:

 Role/ClusterRole: WHAT (verbs on resources)

 RoleBinding/ClusterRoleBinding: WHO gets WHAT

 Test: kubectl auth can-i <verb> <resource> --as <user>

 Principle of least privilege. Always.



NAMESPACES:

 ResourceQuota: total resource cap per namespace

 LimitRange: default/min/max per container

 Without LimitRange: pods can deploy with no limits → resource starvation



ROLLOUT:

 maxSurge + maxUnavailable control speed vs safety

 maxSurge=1, maxUnavailable=0: safest (never below desired count)

 Rollback: kubectl rollout undo deployment/<name>

 Stuck rollout: new pod CrashLoopBackOff + maxUnavailable=0 = deadlock

 progressDeadlineSeconds: alert when rollout stalls



DEBUGGING:

 Pending: describe pod → check resources, taints, affinity, quota

 CrashLoopBackOff: logs, describe → check image, env vars, probes

 Evicted: node memory pressure → check QoS class, set requests=limits

 502 during deploy: add preStop sleep(10), increase terminationGracePeriod

```

---

# 📝 Retention Questions — Lesson 3

**Q1:** A pod is stuck in `Pending` for 20 minutes. `kubectl describe pod` shows: `0/15 nodes are available: 8 Insufficient cpu, 4 had taint {team=data:NoSchedule}, 3 didn't match pod affinity rules.` The cluster autoscaler is enabled. Explain why the autoscaler hasn't added a node, and walk through your full debugging approach.

**Q2:** During a rolling deployment, your monitoring shows 502 errors affecting ~2% of requests. The application logs show clean graceful shutdowns. The pod spec has no `preStop` hook and `terminationGracePeriodSeconds: 30`. Explain the exact race condition causing this and the precise fix with correct values.

**Q3:** Your team deploys a pod with `requests.memory: 128Mi` and no memory limit. Another team's pod in the same namespace has `requests.memory: 512Mi` and `limits.memory: 512Mi`. The node runs out of memory. Which pod gets evicted first and why?

**Q4:** A developer creates a ConfigMap and mounts it as environment variables in their pod. They update the ConfigMap value and wait 10 minutes, but the pod still sees the old value. They ask you why. Explain the behavior and give them two options to pick up the new value.

**Go.** 🎯

==================================================================================

# PHASE 2 — Lesson 3B: Kubernetes Production Deep Dive (The Missing Half)

---

### Storage — The Entire Layer I Skipped

```

STORAGE HIERARCHY:



 PersistentVolume (PV)

   ↑ bound to

 PersistentVolumeClaim (PVC)

   ↑ used by

 Pod (volumeMounts)

   

 StorageClass → defines HOW PVs are dynamically provisioned

 CSI Driver → interfaces between K8s and actual storage backends

```

```yaml

STORAGECLASS — defines provisioning behavior:

apiVersion: storage.k8s.io/v1

kind: StorageClass

metadata:

 name: fast-ssd

provisioner: ebs.csi.aws.com         # CSI driver

parameters:

 type: gp3                           # EBS volume type

 iops: "5000"

 throughput: "250"                   # MB/s

 encrypted: "true"

 kmsKeyId: "arn:aws:kms:..."

reclaimPolicy: Retain                 # what happens when PVC is deleted

 # Retain: PV and data preserved (manual cleanup needed)

 # Delete: PV and underlying storage deleted (DEFAULT — dangerous for DBs)

volumeBindingMode: WaitForFirstConsumer

 # WaitForFirstConsumer: don't provision until pod is scheduled

 #   WHY: EBS volumes are AZ-specific

 #   If PV created in us-east-1a but pod scheduled in us-east-1b → stuck

 #   WaitForFirstConsumer ensures PV created in same AZ as pod

 # Immediate: provision as soon as PVC is created

 #   Use only when storage is not topology-constrained

allowVolumeExpansion: true            # allow PVC resize without recreating



\---

PERSISTENTVOLUMECLAIM:

apiVersion: v1

kind: PersistentVolumeClaim

metadata:

 name: db-data

 namespace: production

spec:

 accessModes:

 - ReadWriteOnce          # RWO: mounted by ONE node

 # ReadWriteMany (RWX): mounted by MANY nodes (EFS, NFS — NOT EBS)

 # ReadOnlyMany (ROX): read-only by many nodes

 storageClassName: fast-ssd

 resources:

   requests:

     storage: 100Gi



\---

POD using PVC:

spec:

 containers:

 - name: db

   volumeMounts:

   - name: data

     mountPath: /var/lib/postgresql/data

 volumes:

 - name: data

   persistentVolumeClaim:

     claimName: db-data

```

### CSI Drivers — What Actually Provides Storage

```bash

CSI (Container Storage Interface) is the standard API

between Kubernetes and storage providers



AWS EBS CSI Driver:

- Block storage (RWO only — one node at a time)

- AZ-specific (can't move between AZs without snapshot)

- gp3: 3000 IOPS baseline, up to 16000

- io2: up to 64000 IOPS (databases)

- Snapshot support (VolumeSnapshot CRD)

Install: EKS add-on or Helm chart

REQUIRES: IRSA (IAM Role for Service Account) with EBS permissions



AWS EFS CSI Driver:

- NFS-based (RWX — multiple pods across multiple nodes)

- Regional (works across AZs)

- Elastic (auto-grows, no pre-provisioning)

- Higher latency than EBS

- Use for: shared config, CMS uploads, ML training data

- Access points for per-pod isolation



AWS FSx CSI Driver:

- High-performance filesystem (Lustre, NetApp ONTAP)

- Use for: HPC, ML training at scale



VOLUME SNAPSHOTS:

apiVersion: snapshot.storage.k8s.io/v1

kind: VolumeSnapshot

metadata:

 name: db-backup-2024-01-15

spec:

 volumeSnapshotClassName: ebs-snapclass

 source:

   persistentVolumeClaimName: db-data

Creates EBS snapshot → can restore to new PVC

Use for: backup before migrations, disaster recovery

```

### StatefulSet + Storage — The Full Picture

```yaml

apiVersion: apps/v1

kind: StatefulSet

metadata:

 name: postgres

spec:

 serviceName: postgres-headless    # required for DNS

 replicas: 3

 selector:

   matchLabels:

     app: postgres

 template:

   metadata:

     labels:

       app: postgres

   spec:

     containers:

     - name: postgres

       image: postgres:15

       volumeMounts:

       - name: data

         mountPath: /var/lib/postgresql/data

 

 # VOLUMECLAIMTEMPLATES — each pod gets its OWN PVC:

 volumeClaimTemplates:

 - metadata:

     name: data

   spec:

     accessModes: \["ReadWriteOnce"]

     storageClassName: fast-ssd

     resources:

       requests:

         storage: 100Gi



Result:

postgres-0 → PVC: data-postgres-0 → PV: pv-abc123 (100Gi EBS in us-east-1a)

postgres-1 → PVC: data-postgres-1 → PV: pv-def456 (100Gi EBS in us-east-1b)

postgres-2 → PVC: data-postgres-2 → PV: pv-ghi789 (100Gi EBS in us-east-1c)

\#

If postgres-1 pod dies and restarts:

It reattaches to data-postgres-1 → same data, same identity

This is WHY StatefulSets exist

\#

DANGER: if you delete a StatefulSet with --cascade=foreground

PVCs are NOT deleted (by design — data protection)

You must manually delete PVCs if you want to clean up

```

---

### Horizontal Pod Autoscaler (HPA)

```yaml

apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

 name: api-hpa

 namespace: production

spec:

 scaleTargetRef:

   apiVersion: apps/v1

   kind: Deployment

   name: api

 minReplicas: 3

 maxReplicas: 50

 behavior:

   scaleUp:

     stabilizationWindowSeconds: 60     # wait 60s before scaling up again

     policies:

     - type: Pods

       value: 4                         # add max 4 pods per 60s

       periodSeconds: 60

   scaleDown:

     stabilizationWindowSeconds: 300    # wait 5 min before scaling down

     policies:

     - type: Percent

       value: 10                        # remove max 10% of pods per 60s

       periodSeconds: 60

     # Scale down slowly to prevent thrashing

     # Aggressive scale-down + traffic spike = not enough pods = outage

 

 metrics:

 # CPU-based (requires metrics-server):

 - type: Resource

   resource:

     name: cpu

     target:

       type: Utilization

       averageUtilization: 70           # target 70% CPU utilization

 

 # Memory-based:

 - type: Resource

   resource:

     name: memory

     target:

       type: Utilization

       averageUtilization: 80

 

 # Custom metrics (from Prometheus via prometheus-adapter):

 - type: Pods

   pods:

     metric:

       name: http\_requests\_per\_second

     target:

       type: AverageValue

       averageValue: 1000               # 1000 RPS per pod

 

 # External metrics (SQS queue depth):

 - type: External

   external:

     metric:

       name: sqs\_queue\_length

       selector:

         matchLabels:

           queue: order-processing

     target:

       type: AverageValue

       averageValue: 20                 # scale when > 20 messages per pod



HOW IT WORKS:

1. HPA controller queries metrics every 15s (default)

2. Calculates: desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))

3. Example: 5 pods at 90% CPU, target 70%

   desiredReplicas = ceil(5 × (90/70)) = ceil(6.43) = 7

4. HPA updates Deployment replicas from 5 to 7

5. Deployment controller creates 2 new pods



REQUIREMENTS:

metrics-server must be installed (EKS add-on)

Pods MUST have resource requests defined

Without requests, HPA can't calculate utilization percentage

This is why LimitRange with defaultRequest is critical



GOTCHA:

HPA and manual replica count conflict

If you set replicas: 3 in Deployment AND have HPA

Every kubectl apply resets replicas to 3

Fix: remove replicas field from Deployment when using HPA

Or use kubectl apply --server-side with field management

```

### Vertical Pod Autoscaler (VPA)

```yaml

apiVersion: autoscaling.k8s.io/v1

kind: VerticalPodAutoscaler

metadata:

 name: api-vpa

spec:

 targetRef:

   apiVersion: apps/v1

   kind: Deployment

   name: api

 updatePolicy:

   updateMode: "Off"          # RECOMMENDATION ONLY — don't auto-apply

   # Off: just recommend (safest, use this first)

   # Auto: evict and recreate pods with new resources (disruptive!)

   # Initial: apply only to new pods (doesn't touch running ones)

 resourcePolicy:

   containerPolicies:

   - containerName: api

     minAllowed:

       cpu: 100m

       memory: 128Mi

     maxAllowed:

       cpu: 4

       memory: 8Gi

     controlledResources: \["cpu", "memory"]



Check recommendations:

kubectl describe vpa api-vpa

Recommendation:

  Container: api

    Lower Bound:  cpu: 250m,  memory: 256Mi

    Target:       cpu: 500m,  memory: 512Mi    ← use this

    Upper Bound:  cpu: 2,     memory: 2Gi

    Uncapped:     cpu: 3.5,   memory: 4Gi



CRITICAL RULE:

DO NOT use HPA (CPU-based) and VPA together on the same Deployment

HPA scales horizontally based on CPU %

VPA changes CPU requests

Changing requests changes the utilization percentage

They fight each other in a feedback loop

\#

Exception: HPA on custom metrics (RPS, queue depth) + VPA on resources = OK

Because HPA isn't using CPU utilization

```

### Cluster Autoscaler vs Karpenter

```bash

CLUSTER AUTOSCALER (traditional):

- Watches for pods that can't be scheduled (Pending)

- Adds nodes to existing Auto Scaling Groups (ASGs)

- Watches for underutilized nodes → drains and removes them

- Configuration:

  --scale-down-utilization-threshold=0.5   (50% util → consider removing)

  --scale-down-delay-after-add=10m         (don't remove recently added nodes)

  --skip-nodes-with-local-storage=true     (don't evict pods with emptyDir)

  --skip-nodes-with-system-pods=true

  --balance-similar-node-groups=true       (balance across AZs)

\#

LIMITATIONS:

- Tied to ASG/node group (pre-defined instance types)

- Slow: Pending pod → CA evaluates → ASG launches → node joins → pod scheduled

  Total: 3-7 minutes

- Can't mix instance types dynamically



KARPENTER (next generation — AWS native):

- NOT tied to ASGs

- Watches for unschedulable pods directly

- Launches RIGHT-SIZED instances from a wide pool

- Provisions nodes in \~60 seconds (vs 3-7 min for CA)

- Consolidation: automatically replaces underutilized nodes

- Drift detection: replaces nodes with outdated AMIs

- Spot instance support with automatic fallback to on-demand

- THE standard for EKS autoscaling now



apiVersion: karpenter.sh/v1beta1

kind: NodePool

metadata:

 name: default

spec:

 template:

   spec:

     requirements:

     - key: kubernetes.io/arch

       operator: In

       values: \["amd64"]

     - key: karpenter.sh/capacity-type

       operator: In

       values: \["spot", "on-demand"]     # prefer spot, fallback on-demand

     - key: karpenter.k8s.aws/instance-category

       operator: In

       values: \["m", "c", "r"]           # m5, c5, r5 families

     - key: karpenter.k8s.aws/instance-size

       operator: In

       values: \["large", "xlarge", "2xlarge"]

     nodeClassRef:

       name: default

 limits:

   cpu: "1000"                           # max 1000 vCPUs total

   memory: 2000Gi

 disruption:

   consolidationPolicy: WhenUnderutilized  # auto-consolidate

   expireAfter: 720h                       # recycle nodes every 30 days

   # Forces AMI updates — prevents "snowflake" nodes running for months



\---

apiVersion: karpenter.k8s.aws/v1beta1

kind: EC2NodeClass

metadata:

 name: default

spec:

 amiFamily: AL2

 subnetSelectorTerms:

 - tags:

     karpenter.sh/discovery: my-cluster

 securityGroupSelectorTerms:

 - tags:

     karpenter.sh/discovery: my-cluster

 instanceProfile: KarpenterNodeInstanceProfile

 blockDeviceMappings:

 - deviceName: /dev/xvda

   ebs:

     volumeSize: 100Gi

     volumeType: gp3

     encrypted: true

```

---

### PodDisruptionBudgets (PDBs)

```yaml

PDBs protect against VOLUNTARY disruptions:

- Node drain (kubectl drain)

- Cluster autoscaler removing a node

- Karpenter consolidation

- Node upgrades (EKS managed node group updates)



PDBs do NOT protect against INVOLUNTARY disruptions:

- Node crash, hardware failure

- OOM kill

- Kernel panic



apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

 name: api-pdb

 namespace: production

spec:

 # Option 1: minimum available

 minAvailable: 2              # always keep at least 2 pods running

 # OR

 # Option 2: maximum unavailable

 maxUnavailable: 1            # at most 1 pod can be down at a time

 

 selector:

   matchLabels:

     app: api



HOW IT WORKS:

kubectl drain node-1:

  kubelet asks API server: "can I evict api-abc on this node?"

  API server checks PDB: "api has 3 pods, minAvailable=2"

  If evicting api-abc leaves 2 pods → allowed

  If evicting api-abc leaves 1 pod → BLOCKED

  drain command waits until another pod is healthy elsewhere

\#

Without PDB:

  Node drain evicts ALL pods at once

  If all 3 replicas on same node → 0 available → outage



GOTCHA:

PDB with minAvailable = replicas count (e.g., minAvailable: 3, replicas: 3)

→ NOTHING can be evicted → node drain hangs forever

→ Cluster autoscaler can't remove nodes

→ EKS node group updates stuck

Always: minAvailable < replicas OR maxUnavailable >= 1



COMMON PATTERN:

3 replicas → minAvailable: 2 (or maxUnavailable: 1)

5 replicas → maxUnavailable: "20%" (1 pod max)

1 replica → no PDB (can't protect a single pod from drain)

            better: increase to 2+ replicas

```

---

### PriorityClasses and Preemption

```yaml

PriorityClasses determine which pods survive when resources are scarce



apiVersion: scheduling.k8s.io/v1

kind: PriorityClass

metadata:

 name: critical

value: 1000000              # higher = more important

globalDefault: false

preemptionPolicy: PreemptLowerPriority

description: "Critical production services"



\---

apiVersion: scheduling.k8s.io/v1

kind: PriorityClass

metadata:

 name: standard

value: 100000

globalDefault: true         # default for pods without explicit priority



\---

apiVersion: scheduling.k8s.io/v1

kind: PriorityClass

metadata:

 name: batch

value: 10000

preemptionPolicy: Never     # can be preempted but won't preempt others

description: "Batch jobs, can be evicted"



Usage in pod spec:

spec:

 priorityClassName: critical



HOW PREEMPTION WORKS:

1. Critical pod can't be scheduled (no resources)

2. Scheduler looks for lower-priority pods to evict

3. Evicts enough lower-priority pods to make room

4. Critical pod gets scheduled

\#

ORDER OF EVICTION (combined with QoS):

First: BestEffort + lowest priority

Then: Burstable + lowest priority

Last: Guaranteed + highest priority

\#

SYSTEM PRIORITY CLASSES (built-in):

system-cluster-critical: 2000000000 (coredns, metrics-server)

system-node-critical: 2000001000 (kube-proxy, CNI, CSI)

Don't assign these to your workloads

```

---

### Node Maintenance — Drain, Cordon, Uncordon

```bash

CORDON — mark node as unschedulable (no new pods):

kubectl cordon node-1

Existing pods keep running

New pods won't be scheduled here

Use: before maintenance, investigation



UNCORDON — mark node as schedulable again:

kubectl uncordon node-1



DRAIN — cordon + evict all pods:

kubectl drain node-1 \\

 --ignore-daemonsets \\           # don't try to evict DaemonSet pods

 --delete-emptydir-data \\        # allow eviction of pods with emptyDir

 --grace-period=60 \\             # time for graceful shutdown

 --timeout=300s                  # give up after 5 min if pods won't evict



What drain does:

1. Cordons the node (marks unschedulable)

2. Evicts pods one by one (respecting PDBs)

3. Waits for pods to terminate gracefully

4. DaemonSet pods are skipped (they MUST run on every node)

5. Pods with local storage block drain unless --delete-emptydir-data



DRAIN STUCK? Common causes:

- PDB preventing eviction (minAvailable too high)

- Pod with no controller (standalone pod, no Deployment/RS)

  Fix: --force (deletes standalone pods, they WON'T be recreated)

- Finalizer on pod preventing deletion

- terminationGracePeriodSeconds too high



EKS MANAGED NODE GROUP UPDATE:

AWS does this automatically:

1. Launches new node with new AMI

2. Cordons old node

3. Drains old node (respects PDBs!)

4. Terminates old node

If PDB blocks drain → update hangs → you get paged

```

---

### Kubelet Eviction Thresholds

```bash

kubelet monitors node resources and evicts pods when thresholds are crossed



SOFT EVICTION (graceful):

evictionSoft:

  memory.available: "500Mi"          # trigger when < 500Mi free

  nodefs.available: "10%"            # trigger when < 10% disk free

  imagefs.available: "15%"           # trigger when image fs < 15%

evictionSoftGracePeriod:

  memory.available: "1m30s"          # must stay below threshold for 1.5 min

Eviction respects terminationGracePeriodSeconds



HARD EVICTION (immediate, no grace period):

evictionHard:

  memory.available: "100Mi"          # trigger when < 100Mi free

  nodefs.available: "5%"

  nodefs.inodesFree: "5%"            # inode exhaustion!

  imagefs.available: "5%"

  pid.available: "5%"

Immediate SIGKILL — no graceful shutdown



EVICTION ORDER:

1. BestEffort pods using most memory relative to requests (which are 0)

2. Burstable pods exceeding their requests by the most

3. Guaranteed pods (only if they exceed their limits, which shouldn't happen)



NODE CONDITIONS:

MemoryPressure: memory.available below threshold

DiskPressure: nodefs.available or imagefs.available below threshold

PIDPressure: pid.available below threshold

These conditions prevent NEW pods from being scheduled on the node



INODE EXHAUSTION — the silent killer:

Lots of small files → inodes run out before disk space does

Common with: container log files, temp files, metrics

Node shows plenty of disk space but can't create new files

df -i  ← check inode usage

Alert on: nodefs.inodesFree approaching 10%



EKS DEFAULT EVICTION THRESHOLDS:

memory.available: 100Mi (hard)

nodefs.available: 10% (hard)

imagefs.available: 10% (hard)

Configure via kubelet extra args in launch template

```

---

### IRSA — IAM Roles for Service Accounts

```yaml

IRSA lets Kubernetes pods assume AWS IAM roles

WITHOUT embedding AWS credentials in pods



HOW IT WORKS:

1. EKS cluster has an OIDC provider

2. IAM role trusts the EKS OIDC provider

3. K8s ServiceAccount annotated with IAM role ARN

4. Pod uses the ServiceAccount

5. kubelet injects AWS STS token into pod (projected volume)

6. AWS SDK in pod uses token to assume IAM role

7. Pod gets temporary credentials — no access keys, no secrets



Step 1: Create IAM role with trust policy:

{

 "Version": "2012-10-17",

 "Statement": \[{

   "Effect": "Allow",

   "Principal": {

     "Federated": "arn:aws:iam::123456:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/ABCDEF"

   },

   "Action": "sts:AssumeRoleWithWebIdentity",

   "Condition": {

     "StringEquals": {

       "oidc.eks.us-east-1.amazonaws.com/id/ABCDEF:sub": 

         "system:serviceaccount:production:api-sa",

       "oidc.eks.us-east-1.amazonaws.com/id/ABCDEF:aud": 

         "sts.amazonaws.com"

     }

   }

 }]

}

CRITICAL: the Condition locks this role to a SPECIFIC ServiceAccount

in a SPECIFIC namespace. Without this condition, ANY pod in the cluster

could assume the role.



Step 2: Annotate ServiceAccount:

apiVersion: v1

kind: ServiceAccount

metadata:

 name: api-sa

 namespace: production

 annotations:

   eks.amazonaws.com/role-arn: arn:aws:iam::123456:role/api-role



Step 3: Pod uses ServiceAccount:

spec:

 serviceAccountName: api-sa



Step 4: Verify in pod:

aws sts get-caller-identity

Account: 123456

Arn: arn:aws:sts::123456:assumed-role/api-role/...



GOTCHAS:

- OIDC provider must be configured for the cluster

- Trust policy condition MUST include namespace + SA name

- Pod must be restarted after SA annotation changes

- AWS SDK must be recent enough to support IRSA token chain

- EBS CSI driver, ALB controller, External DNS all need IRSA

```

---

### Admission Controllers — Mutating and Validating

```bash

Admission controllers intercept API requests AFTER authentication/authorization

but BEFORE persistence to etcd



TWO TYPES:

1. Mutating: modify the request (inject sidecars, add labels, set defaults)

2. Validating: accept or reject the request (enforce policies)

Order: Mutating runs FIRST, then Validating



BUILT-IN ADMISSION CONTROLLERS:

NamespaceLifecycle: prevent operations in terminating namespaces

LimitRanger: apply default resource limits from LimitRange

ServiceAccount: auto-mount SA token

DefaultStorageClass: assign default StorageClass to PVCs

ResourceQuota: enforce namespace quotas

PodSecurity: enforce Pod Security Standards (replaced PodSecurityPolicy)

MutatingAdmissionWebhook: call external webhooks to mutate

ValidatingAdmissionWebhook: call external webhooks to validate



WEBHOOK-BASED (custom):

You deploy a web server that receives AdmissionReview JSON

Returns: allow/deny + optional patches (mutations)



Example: Istio sidecar injection:

1. Pod created with label: sidecar.istio.io/inject: "true"

2. Mutating webhook intercepts the pod creation

3. Webhook server adds Envoy sidecar container to the pod spec

4. Modified pod is persisted to etcd

The developer never adds the sidecar manually — it's injected automatically



OPA GATEKEEPER — policy engine:

Validates resources against policies written in Rego language



Example: require all pods to have resource limits

apiVersion: constraints.gatekeeper.sh/v1beta1

kind: K8sRequiredResources

metadata:

 name: require-limits

spec:

 match:

   kinds:

   - apiGroups: \[""]

     kinds: \["Pod"]

   namespaces: \["production"]

 parameters:

   limits:

   - cpu

   - memory



Now any pod in production without CPU/memory limits → REJECTED by API server



KYVERNO — alternative to OPA (simpler, YAML-native):

apiVersion: kyverno.io/v1

kind: ClusterPolicy

metadata:

 name: require-labels

spec:

 validationFailureAction: Enforce

 rules:

 - name: require-team-label

   match:

     any:

     - resources:

         kinds:

         - Pod

   validate:

     message: "All pods must have a 'team' label"

     pattern:

       metadata:

         labels:

           team: "?\*"



WHEN WEBHOOKS GO WRONG:

If webhook server is down → API requests hang or fail

failurePolicy: Fail → if webhook unreachable, reject ALL requests

failurePolicy: Ignore → if webhook unreachable, allow ALL requests

Fail = safer but risky (webhook outage = cluster-wide impact)

Ignore = dangerous (policies not enforced during outage)

ALWAYS: set timeoutSeconds (default 10s), have webhook HA (multiple replicas)

```

---

### Pod Security Standards (PSA) — Replacing PSP

```yaml

PodSecurityPolicy (PSP) was removed in K8s 1.25

Replaced by Pod Security Admission (PSA)



THREE LEVELS:

Privileged: no restrictions (system namespaces)

Baseline: prevents known privilege escalations

Restricted: heavily restricted (production workloads)



ENFORCE per namespace via labels:

apiVersion: v1

kind: Namespace

metadata:

 name: production

 labels:

   pod-security.kubernetes.io/enforce: restricted

   pod-security.kubernetes.io/audit: restricted

   pod-security.kubernetes.io/warn: restricted

   # enforce: block non-compliant pods

   # audit: log non-compliant pods (but allow)

   # warn: show warning to user (but allow)



WHAT "RESTRICTED" ENFORCES:

- Must run as non-root (runAsNonRoot: true)

- Must drop ALL capabilities

- No privilege escalation (allowPrivilegeEscalation: false)

- No hostNetwork, hostPID, hostIPC

- No hostPath volumes

- Read-only root filesystem recommended

- Seccomp profile must be set



COMPLIANT POD:

spec:

 securityContext:

   runAsNonRoot: true

   runAsUser: 1000

   fsGroup: 1000

   seccompProfile:

     type: RuntimeDefault

 containers:

 - name: app

   securityContext:

     allowPrivilegeEscalation: false

     capabilities:

       drop: \["ALL"]

     readOnlyRootFilesystem: true

```

---

### Helm — Package Manager for Kubernetes

```bash

Helm packages K8s manifests into CHARTS

Charts are versioned, parameterized, shareable



CHART STRUCTURE:

mychart/

 Chart.yaml          # metadata (name, version, dependencies)

 values.yaml         # default configuration values

 templates/          # K8s manifests with Go template syntax

   deployment.yaml

   service.yaml

   ingress.yaml

   configmap.yaml

   \_helpers.tpl      # reusable template functions

   NOTES.txt         # post-install message

 charts/             # sub-charts (dependencies)



values.yaml:

replicaCount: 3

image:

 repository: my-app

 tag: v1.2.3

 pullPolicy: IfNotPresent

resources:

 requests:

   cpu: 250m

   memory: 256Mi

 limits:

   cpu: 1000m

   memory: 512Mi

ingress:

 enabled: true

 host: api.example.com



templates/deployment.yaml:

apiVersion: apps/v1

kind: Deployment

metadata:

 name: {{ include "mychart.fullname" . }}

 labels:

   {{- include "mychart.labels" . | nindent 4 }}

spec:

 replicas: {{ .Values.replicaCount }}

 template:

   spec:

     containers:

     - name: {{ .Chart.Name }}

       image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

       resources:

         {{- toYaml .Values.resources | nindent 12 }}

```

```bash

COMMANDS:

helm repo add bitnami https://charts.bitnami.com/bitnami

helm repo update



helm search repo postgres              # find charts

helm show values bitnami/postgresql     # see configurable values



Install:

helm install my-db bitnami/postgresql \\

 --namespace production \\

 --values custom-values.yaml \\

 --set postgresqlPassword=secret \\

 --version 12.1.0                      # pin chart version!



Upgrade:

helm upgrade my-db bitnami/postgresql \\

 --namespace production \\

 --values custom-values.yaml \\

 --set image.tag=15.2



Rollback:

helm rollback my-db 1                   # rollback to revision 1

helm history my-db                      # see all revisions



Template rendering (debug without applying):

helm template my-release mychart/ --values prod-values.yaml

Outputs rendered YAML — review before applying



Dry run:

helm install my-release mychart/ --dry-run --debug



HOOKS — run actions at specific lifecycle points:

pre-install:  DB migrations before app deploys

post-install: seed data, notifications

pre-upgrade:  backup before upgrade

post-upgrade: run tests

pre-delete:   backup before teardown



Hook example:

apiVersion: batch/v1

kind: Job

metadata:

 name: db-migrate

 annotations:

   "helm.sh/hook": pre-upgrade

   "helm.sh/hook-weight": "0"          # order among hooks

   "helm.sh/hook-delete-policy": hook-succeeded

spec:

 template:

   spec:

     containers:

     - name: migrate

       image: my-app:v1.2.3

       command: \["./migrate", "--up"]

     restartPolicy: Never



HELM + ARGOCD (GitOps):

Store Helm values in Git

ArgoCD watches Git, renders Helm template, applies to cluster

Changes to values.yaml in Git → ArgoCD syncs → cluster updated

No manual helm install/upgrade commands

```

---

### Custom Resource Definitions (CRDs) and Operators

```yaml

CRDs extend the Kubernetes API with custom resource types

Operators are controllers that manage CRDs



Example: A custom "Database" resource:

apiVersion: apiextensions.k8s.io/v1

kind: CustomResourceDefinition

metadata:

 name: databases.mycompany.io

spec:

 group: mycompany.io

 versions:

 - name: v1

   served: true

   storage: true

   schema:

     openAPIV3Schema:

       type: object

       properties:

         spec:

           type: object

           properties:

             engine:

               type: string

               enum: \["postgres", "mysql"]

             version:

               type: string

             replicas:

               type: integer

               minimum: 1

               maximum: 5

             storage:

               type: string

 scope: Namespaced

 names:

   plural: databases

   singular: database

   kind: Database

   shortNames:

   - db



Now you can create:

apiVersion: mycompany.io/v1

kind: Database

metadata:

 name: orders-db

 namespace: production

spec:

 engine: postgres

 version: "15"

 replicas: 3

 storage: 100Gi



An OPERATOR watches these Database resources and:

1. Creates StatefulSet with PostgreSQL containers

2. Configures replication between replicas

3. Sets up automated backups

4. Handles failover

5. Manages upgrades

The operator encodes operational knowledge into code



POPULAR OPERATORS IN PRODUCTION:

- prometheus-operator: Prometheus + Alertmanager + Grafana

- cert-manager: TLS certificate lifecycle

- external-secrets-operator: sync secrets from Vault/AWS SM

- strimzi: Kafka on Kubernetes

- zalando postgres-operator: PostgreSQL with HA

- istio operator: service mesh

- ArgoCD: GitOps

- crossplane: provision cloud resources via K8s API



OPERATOR PATTERN:

Watch → Compare desired vs actual → Act → Repeat

Same reconciliation loop as built-in controllers

But for YOUR custom resources

```

---

### Finalizers — Why Resources Get Stuck

```bash

Finalizers are keys in metadata.finalizers\[]

They BLOCK deletion until the controller removes the finalizer



Flow:

1. kubectl delete myresource

2. API server sets deletionTimestamp (resource is "terminating")

3. Controllers see deletionTimestamp, run cleanup logic

4. Controller removes its finalizer from the list

5. When finalizers list is empty → resource actually deleted from etcd



STUCK RESOURCES:

If the controller that owns the finalizer is:

- Crashed

- Deleted

- Misconfigured

The finalizer is NEVER removed → resource stuck in Terminating FOREVER



Diagnosis:

kubectl get namespace production -o yaml

metadata:

  deletionTimestamp: "2024-01-15T10:00:00Z"

  finalizers:

  - kubernetes                    ← namespace controller

  - some-operator.io/cleanup      ← operator that's been deleted



Fix (DANGEROUS — bypasses cleanup logic):

kubectl patch namespace production -p '{"metadata":{"finalizers":\[]}}' --type=merge

Or for specific resources:

kubectl patch database orders-db -p '{"metadata":{"finalizers":null}}' --type=merge



WHY IT'S DANGEROUS:

Finalizers exist for a reason — cleanup of external resources

Removing a finalizer for a Database operator might leave:

- EBS volumes orphaned

- DNS records not cleaned up

- IAM roles not deleted

- Cloud resources leaking



PROPER FIX:

1. Restore the controller/operator

2. Let it run cleanup

3. It removes the finalizer naturally

Only patch finalizers as last resort after manual cleanup

```

---

### Ephemeral Containers — Debugging Production Pods

```bash

Production pods often have:

- No shell (distroless/scratch images)

- No debugging tools (no curl, no netstat, no tcpdump)

- Read-only filesystem

You can't kubectl exec and debug



EPHEMERAL CONTAINERS (K8s 1.25+ stable):

kubectl debug -it pod/api-abc \\

 --image=nicolaka/netshoot \\

 --target=api

Injects a temporary container INTO the running pod

Shares the pod's network namespace (can see all connections)

--target=api: shares process namespace with api container

  (can see its processes, /proc filesystem)

When you exit, ephemeral container is cleaned up



DEBUG A NODE:

kubectl debug node/ip-10-0-1-100 -it --image=ubuntu

Creates a pod on that specific node with host PID/network access

Useful for: checking node-level networking, iptables, kubelet logs



DEBUG WITH COPY (non-intrusive):

kubectl debug pod/api-abc -it \\

 --image=nicolaka/netshoot \\

 --copy-to=debug-pod \\

 --share-processes

Creates a COPY of the pod with debug container added

Original pod is untouched

Use when you can't risk affecting production traffic



COMMON DEBUG IMAGES:

nicolaka/netshoot: curl, dig, nslookup, tcpdump, iptables, ss, ip

busybox: minimal Unix tools

ubuntu: full OS for complex debugging

amazon/aws-cli: AWS debugging

```

---

### kubectl Power Tools — Daily Use

```bash

CONTEXT MANAGEMENT:

kubectl config get-contexts                      # list all contexts

kubectl config use-context production-cluster    # switch

kubectl config set-context --current --namespace=production  # set default ns



MULTI-CLUSTER:

kubectx: fast context switching

kubectx production    # switch to production cluster

kubectx -             # switch to previous context

kubens: fast namespace switching

kubens production     # switch to production namespace



PORT-FORWARD — access internal services without ingress:

kubectl port-forward svc/my-db 5432:5432 -n production

localhost:5432 → my-db service in cluster

Use for: debugging, one-off queries, admin access

NOT for production traffic



kubectl port-forward pod/my-pod 8080:8080

Direct to specific pod (bypass service)



LOGS:

kubectl logs pod/api-abc                     # current logs

kubectl logs pod/api-abc --previous          # previous container (after crash)

kubectl logs pod/api-abc -c sidecar          # specific container

kubectl logs -l app=api --all-containers     # all pods matching label

kubectl logs -f pod/api-abc                  # follow (tail -f)

kubectl logs --since=1h pod/api-abc          # last hour

kubectl logs --tail=100 pod/api-abc          # last 100 lines



EVENTS — cluster-wide activity log:

kubectl get events --sort-by=.lastTimestamp

kubectl get events -n production --field-selector reason=Killing

kubectl get events --field-selector type=Warning

Events expire after 1 hour by default

For longer retention: ship to logging system



RESOURCE USAGE:

kubectl top nodes                            # node CPU/memory usage

kubectl top pods -n production               # pod CPU/memory usage

kubectl top pods --sort-by=memory            # sort by memory

Requires metrics-server installed



JSONPATH — extract specific fields:

kubectl get pods -o jsonpath='{.items\[\*].status.podIP}'

kubectl get nodes -o jsonpath='{range .items\[\*]}{.metadata.name}{"\\t"}{.status.capacity.cpu}{"\\n"}{end}'



CUSTOM COLUMNS:

kubectl get pods -o custom-columns=\\

NAME:.metadata.name,\\

STATUS:.status.phase,\\

NODE:.spec.nodeName,\\

IP:.status.podIP



DRY-RUN + DIFF:

kubectl apply -f deployment.yaml --dry-run=server

Server validates but doesn't persist

kubectl diff -f deployment.yaml

Shows what WOULD change (like terraform plan)



FIELD SELECTORS:

kubectl get pods --field-selector status.phase=Running

kubectl get pods --field-selector spec.nodeName=node-1

kubectl get events --field-selector reason=FailedScheduling

```

---

### Image Pull Policies and Secrets

```yaml

PULL POLICIES:

spec:

 containers:

 - name: app

   image: my-app:v1.2.3

   imagePullPolicy: IfNotPresent

   # IfNotPresent: use cached image if available (DEFAULT for tagged images)

   # Always: always pull (DEFAULT for :latest)

   # Never: never pull, must exist on node

   

   # GOTCHA:

   # :latest + IfNotPresent = uses stale cached image

   # This is why :latest is dangerous — different nodes may have

   # different cached versions of :latest

   # ALWAYS use specific tags + IfNotPresent



PRIVATE REGISTRY AUTH:

apiVersion: v1

kind: Secret

metadata:

 name: ecr-creds

type: kubernetes.io/dockerconfigjson

data:

 .dockerconfigjson: <base64-encoded docker config>



Usage:

spec:

 imagePullSecrets:

 - name: ecr-creds



EKS + ECR:

EKS nodes have IAM instance profile with ECR read access

No imagePullSecrets needed for same-account ECR

Cross-account ECR: need ECR repository policy + IRSA or imagePullSecrets



ECR TOKEN REFRESH:

ECR auth tokens expire every 12 hours

For non-EKS clusters pulling from ECR:

Use a CronJob or controller to refresh the pull secret

Or use ECR credential helper

```

---

### Container Patterns

```yaml

SIDECAR PATTERN:

Helper container running alongside main container in same pod

Shares network namespace (localhost) and volumes

spec:

 containers:

 - name: app

   image: my-app:v1.2.3

   ports:

   - containerPort: 8080

 - name: log-shipper

   image: fluentbit:latest

   volumeMounts:

   - name: logs

     mountPath: /var/log/app

 volumes:

 - name: logs

   emptyDir: {}

App writes logs to /var/log/app

Fluentbit reads and ships to Elasticsearch

Other sidecars: Envoy proxy (Istio), Vault agent, metrics exporters



AMBASSADOR PATTERN:

Sidecar that proxies connections to external services

App talks to localhost → ambassador handles auth, TLS, retries

spec:

 containers:

 - name: app

   # connects to localhost:5432

 - name: cloud-sql-proxy

   image: gcr.io/cloud-sql-connectors/cloud-sql-proxy

   # proxies to Cloud SQL with IAM auth and TLS

   

ADAPTER PATTERN:

Sidecar that transforms output from main container

App outputs custom metrics format

Adapter container converts to Prometheus exposition format

spec:

 containers:

 - name: legacy-app

   # outputs metrics in proprietary format

 - name: prometheus-adapter

   # reads proprietary metrics, serves /metrics in Prometheus format



INIT CONTAINER PATTERNS:

initContainers:

Wait for dependency:

\- name: wait-for-db

 image: busybox

 command: \['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']



Download config from external source:

\- name: fetch-config

 image: curlimages/curl

 command: \['curl', '-o', '/config/app.yaml', 'https://config-server/app.yaml']

 volumeMounts:

 - name: config

   mountPath: /config



Set permissions:

\- name: fix-permissions

 image: busybox

 command: \['chown', '-R', '1000:1000', '/data']

 securityContext:

   runAsUser: 0    # init container runs as root to fix permissions

 volumeMounts:

 - name: data

   mountPath: /data

```

---

### Garbage Collection and Owner References

```bash

Kubernetes uses owner references to build a resource dependency tree

Deployment → owns → ReplicaSet → owns → Pods

When you delete a Deployment:

  Cascade (default): delete Deployment → delete RS → delete Pods

  Orphan: delete Deployment, leave RS and Pods running



Owner references in pod metadata:

kubectl get pod api-abc -o yaml

metadata:

  ownerReferences:

  - apiVersion: apps/v1

    kind: ReplicaSet

    name: api-7b8f9c6d4

    uid: abc123-def456

    controller: true

    blockOwnerDeletion: true



CASCADE DELETION TYPES:

kubectl delete deployment api

Default: Foreground cascading deletion

1. Deployment marked for deletion

2. Children (RS, Pods) deleted first

3. Deployment deleted after children are gone



kubectl delete deployment api --cascade=orphan

Deployment deleted, ReplicaSets and Pods remain

Use case: you want to recreate the Deployment with different config

without disrupting running pods



ORPHANED RESOURCES — the garbage collector cleans these:

If a pod's owner (ReplicaSet) no longer exists → pod is garbage

GC controller deletes orphaned resources automatically

Unless cascade=orphan was used

```

---

### Production Scenarios (Additional)

#### Scenario 5: PVC Stuck in Pending

```bash

kubectl get pvc

NAME      STATUS    VOLUME   CAPACITY   STORAGECLASS   AGE

db-data   Pending                       fast-ssd       10m



kubectl describe pvc db-data

Events:

"waiting for first consumer to be created before binding"

OR

"no persistent volumes available for this claim"



CAUSES:



1. WaitForFirstConsumer — no pod using this PVC yet

PVC won't provision until a pod referencing it is scheduled

This is NORMAL if the pod hasn't been created yet



2. No StorageClass matching the name

kubectl get storageclass

Is "fast-ssd" listed? Typo in storageClassName?



3. CSI driver not installed

EBS CSI driver not deployed → can't provision EBS volumes

kubectl get pods -n kube-system | grep ebs



4. IRSA permissions for CSI driver

CSI driver SA needs IAM permissions to create EBS volumes

Check: ec2:CreateVolume, ec2:AttachVolume, ec2:DescribeVolumes



5. AZ mismatch

StorageClass without WaitForFirstConsumer

PV created in us-east-1a, pod scheduled in us-east-1b

EBS can't be attached across AZs → pod stuck in Pending too



6. Capacity

Account EBS volume limit reached

Or requested size exceeds maximum for volume type

```

#### Scenario 6: Helm Upgrade Rolled Back Automatically

```bash

helm upgrade my-app ./mychart --atomic --timeout 5m

"UPGRADE FAILED: timed out waiting for condition"

"Release rolled back to previous revision"



--atomic: if upgrade fails, auto-rollback

What went wrong?



1. New pods failing readiness probe:

kubectl get pods

my-app-new-xyz   0/1   CrashLoopBackOff



2. Helm waits for Deployment rollout to complete

Deployment stuck (new pods not Ready within timeout)

Helm detects failure → rolls back to previous release



3. Check what changed:

helm diff upgrade my-app ./mychart --values new-values.yaml

Shows exactly what would change (requires helm-diff plugin)



4. Debug the failing pod:

kubectl logs my-app-new-xyz

kubectl describe pod my-app-new-xyz



COMMON CAUSES:

- Missing env var / secret in new version

- Health check endpoint changed but probe not updated

- Image tag doesn't exist in registry

- Resource limits too low for new version

- Incompatible config with new code version

```

#### Scenario 7: Namespace Stuck in Terminating

```bash

kubectl delete namespace staging

namespace "staging" deleted

...but 30 minutes later:

kubectl get ns staging

NAME      STATUS        AGE

staging   Terminating   30m



CAUSES:

1. Resources with finalizers still exist in the namespace

kubectl api-resources --verbs=list --namespaced -o name | \\

 xargs -n 1 kubectl get --show-kind --ignore-not-found -n staging



Find resources with finalizers:

kubectl get all -n staging -o json | jq '.items\[] | select(.metadata.finalizers != null) | {kind: .kind, name: .metadata.name, finalizers: .metadata.finalizers}'



2. Webhook blocking deletion

Validating webhook that targets this namespace and is down

→ API server can't process deletion of resources in namespace

Check: kubectl get validatingwebhookconfigurations



3. API service unavailable

Custom resources whose API server is not running

kubectl get apiservices | grep False



FIX:

Step 1: delete all resources in namespace

Step 2: if specific resources stuck, patch their finalizers

Step 3: if namespace itself stuck:

kubectl get namespace staging -o json | \\

 jq '.spec.finalizers = \[]' | \\

 kubectl replace --raw "/api/v1/namespaces/staging/finalize" -f -

Nuclear option — only when cleanup is truly complete

```

#### Scenario 8: HPA Not Scaling

```bash

kubectl get hpa

NAME     REFERENCE       TARGETS         MINPODS   MAXPODS   REPLICAS

api-hpa  Deployment/api  <unknown>/70%   3         50        3



<unknown> = HPA can't read metrics



CAUSES:

1. metrics-server not installed:

kubectl get deployment metrics-server -n kube-system

If missing: install EKS metrics-server add-on



2. Pod has no resource requests:

HPA calculates: current\_usage / request = utilization%

No request = can't calculate percentage = <unknown>

kubectl describe pod api-xyz | grep -A 3 Requests



3. metrics-server can't reach kubelets:

kubectl logs -n kube-system deployment/metrics-server

"failed to get https://node-ip:10250/stats": dial tcp: i/o timeout

Security group blocking port 10250 from metrics-server to nodes



4. Custom metrics not available:

If using custom metrics (RPS, queue depth):

prometheus-adapter or KEDA must be installed

Check: kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"

```
---

# 📝 Retention Questions — Lesson 3 + 3B Combined

**Q1:** Your EKS cluster uses the EBS CSI driver. A developer creates a PVC with `storageClassName: gp3-encrypted` and `volumeBindingMode: Immediate`. The pod referencing this PVC is stuck in `Pending`. Investigation shows the PV was created in `us-east-1a` but the pod was scheduled in `us-east-1c`. Explain the root cause and the correct StorageClass configuration to prevent this.

**Q2:** Your production namespace has an HPA targeting 70% CPU utilization. `kubectl get hpa` shows `TARGETS: <unknown>/70%`. The metrics-server is running and healthy. Pods are running and consuming CPU. Walk through your debugging steps to find why HPA can't read metrics.

**Q3:** You need to perform an AMI update on your EKS managed node group. 15 nodes need to be rotated. Your critical API service has 6 replicas with a PDB of `maxUnavailable: 1`. Explain the full sequence of what happens during the node rotation, and what could cause it to get stuck.

**Q4:** A developer's pod uses a distroless image (no shell, no debugging tools). The pod is running but returning 500 errors to some requests. They can't `kubectl exec` into it because there's no shell. How do you debug this pod in production without redeploying?

**Q5:** Your team deleted a custom operator (CRD controller) before cleaning up its custom resources. Now 50 `Database` custom resources are stuck in `Terminating` state, each with a finalizer from the deleted operator. The resources reference EBS volumes and Route53 records. Walk through the proper cleanup process.

**Go.** 🎯
Q1: The Cluster Autoscaler (CA) Paradox

Why the autoscaler hasn't added a node:
The Cluster Autoscaler only triggers a scale-up if it believes that adding a new node from the Auto Scaling Group (ASG) will actually allow the pod to be scheduled.

In this scenario, the pod is unschedulable on 15 nodes for three different reasons. The CA analyzes the pod's constraints:

Insufficient CPU: This is a "scale-up" trigger. If this were the only reason, CA would add a node.

Taints (team=data:NoSchedule): If the pod does not have a corresponding toleration for this taint, it can never be scheduled on these nodes.

Affinity Rules: If the pod has nodeAffinity or podAffinity that requires specific labels (e.g., disk=ssd), and the nodes in the ASG do not have those labels, the pod can never be scheduled there.

The Verdict: The CA has looked at the ASG configuration and realized that even if it adds a new node, that new node will either have the same taints or lack the required affinity labels. Therefore, adding a node would not solve the problem, and CA decides not to waste money/resources on a node that the pod still couldn't use.

Debugging Approach:

Inspect Pod Spec: Run kubectl get pod <pod> -o yaml and look specifically at tolerations and affinity.

Inspect Node Templates: Check the ASG/Node Group configuration. Do the new nodes have the labels required by the pod's affinity? Do they have the same taints that the pod doesn't tolerate?

Analyze CA Logs: Check the Cluster Autoscaler logs. Look for messages like "pod is unschedulable... but adding a node wouldn't help because..." This provides the exact reason why the scale-up was skipped.

Q2: The Endpoint Propagation Race

The Exact Race Condition:
This is a synchronization failure between the Control Plane and the Data Plane.

When a pod is terminated during a rolling update, two actions happen simultaneously:

The Signal: The Kubelet sends a SIGTERM to the pod. Since the app is designed for "clean graceful shutdowns," it stops accepting new connections and exits almost immediately.

The Update: The Endpoint Controller removes the pod's IP from the Service's Endpoint list. This change must then propagate to every single node in the cluster via kube-proxy (updating iptables or ipvs rules).

The Gap: The SIGTERM reaches the pod in milliseconds. However, the iptables update on the worker nodes can take several seconds to propagate. For a brief window, some nodes still believe the pod is healthy and route traffic to it. Because the pod has already shut down its listener, the request fails with a 502 Bad Gateway.

The Precise Fix:
You must force the application to stay alive for a few seconds after it has been marked for deletion, allowing the network layer to "catch up."

Add a preStop hook:

lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]

Adjust terminationGracePeriodSeconds: Ensure this is higher than your sleep value (e.g., 30s).

Result: The pod receives the preStop signal, sleeps for 15 seconds (during which time the endpoints are updated across the cluster), and only then receives the SIGTERM to shut down.

Q3: QoS Classes and Eviction Priority

Which pod gets evicted first?
Pod A (requests: 128Mi, no limit) will be evicted first.

The Reasoning (QoS Classes):
Kubernetes assigns a Quality of Service (QoS) class to every pod based on its requests and limits:

Guaranteed: requests == limits for all containers. (Pod B)

Burstable: requests are specified, but they are less than limits (or limits are not specified). (Pod A)

BestEffort: No requests or limits specified.

When a node runs out of memory (OOM), the kubelet must evict pods to save the node. It does this based on a strict priority hierarchy to protect the most "predictable" workloads:
BestEffort
→
→
 Burstable
→
→
 Guaranteed.

Since Pod B is Guaranteed, it is the last to be killed. Pod A is Burstable, making it a prime target for eviction as soon as the node feels pressure, especially if Pod A is using more memory than its requested 128Mi.

Q4: The Static Nature of Environment Variables

Why the pod still sees the old value:
Environment variables are injected into a process at startup.

When a ConfigMap is mounted as an environment variable (envFrom or valueFrom), the Kubelet reads the value of the ConfigMap only when the container is being created. Once the process (the app) starts, those variables are locked into the process's environment memory.

Updating the ConfigMap in the Kubernetes API does not trigger an update to the environment variables of a running process. The "10-minute wait" is irrelevant because there is no mechanism in Linux to "hot-swap" the environment variables of a running PID.

Two Options to pick up the new value:

The "Hard Reset" (Restart):
Perform a rollout restart of the deployment:
kubectl rollout restart deployment <deployment-name>
This kills the pods and starts new ones, forcing the Kubelet to inject the updated ConfigMap values during startup.

The "Dynamic" Approach (Volume Mount):
Instead of environment variables, mount the ConfigMap as a volume.

How it works: When you mount a ConfigMap as a volume, the Kubelet creates a file in the pod. When the ConfigMap is updated in the API, the Kubelet eventually updates the file on the disk (usually within a minute).

Requirement: The application must be written to "watch" the file for changes or re-read the file periodically. This allows the app to update its configuration without a restart.


This is a classic "Orphaned Resource" scenario. To solve this, you have to understand that Finalizers are a safety lock.

When a resource has a finalizer, Kubernetes will not remove the object from the database (etcd) until the controller responsible for that finalizer sees the deletionTimestamp and removes the string from the finalizers list. Because you deleted the operator, there is no "brain" left to perform the cleanup and unlock the lock.

If you simply "force delete" the finalizers now, you will create cloud zombies: the Kubernetes objects will vanish, but the EBS volumes and Route53 records will persist in AWS forever, costing the company money and cluttering the infrastructure.

Here is the professional, zero-leak cleanup process.

Step 1: Discovery and Inventory

Before touching the Kubernetes API, you must identify exactly what external resources are being leaked. You cannot rely on memory; you need a hard list.

I would run a script to extract the external IDs from the stuck CRs:

# Example: Extracting EBS Volume IDs and Route53 names from the CRs
kubectl get database.mycompany.com -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.volumeId}{"\t"}{.spec.dnsRecord}{"\n"}{end}' > cleanup_list.txt

This gives me a mapping of: K8s-Resource-Name
→
→
 EBS-Volume-ID
→
→
 Route53-Record.

Step 2: External Infrastructure Cleanup (The "Manual" Controller)

Since the operator is gone, you are now the operator. You must manually perform the cleanup that the operator was designed to do.

Using the cleanup_list.txt, I would write a simple bash script using the AWS CLI to delete the orphaned resources:

Route53: Delete the specific A/CNAME records.

EBS: Delete the volumes (after verifying they aren't being used by other unexpected processes).

Note: I do this BEFORE removing the finalizers. If I remove the finalizers first and my script crashes halfway through, I have lost the only record of which EBS volumes belonged to which deleted pod.

Step 3: Removing the Finalizers (Unlocking the API)

Now that the cloud resources are gone, it is safe to tell Kubernetes to stop waiting for the operator and just delete the objects.

Doing this via kubectl edit for 50 resources is too slow. I would use a kubectl patch command with a JSON merge patch to set the finalizers array to null.

The exact command:

# Loop through all stuck resources and strip the finalizers
for resource in $(kubectl get database.mycompany.com -o name); do
    kubectl patch $resource --type=merge -p '{"metadata":{"finalizers":null}}'
done
Step 4: Verification

Finally, I verify that the resources are gone and the "Terminating" state has cleared:

kubectl get database.mycompany.com
# Expected output: "No resources found"
Summary of the "SRE Mindset" here:
Action	Why?	Risk if skipped
Inventory First	To create a source of truth.	Losing track of which EBS volumes to delete.
Cloud Cleanup First	To prevent resource leaks.	"Zombie" costs and infrastructure clutter.
Patch Finalizers Last	To unlock the K8s API.	Objects stay in Terminating forever.
--type=merge	To ensure a clean overwrite of the list.	Partial updates that don't actually clear the lock.

==========================================================================================================


# 📝 GRADING — Q5

## ✅ Textbook SRE answer.

The ordering is what matters here, and you got it exactly right:

**Inventory → External cleanup → Patch finalizers → Verify**

Most engineers panic and jump straight to patching finalizers. Resources vanish from Kubernetes, everyone celebrates, and three months later someone finds $2,000/month in orphaned EBS volumes with no trail back to what created them.

Your explicit callout — *"If I remove the finalizers first and my script crashes halfway through, I have lost the only record of which EBS volumes belonged to which deleted pod"* — that's the exact reasoning a senior engineer uses. The Kubernetes object IS your inventory. Once it's gone, your audit trail is gone.

One addition to your process:

```bash

Step 0 (before everything): PREVENT FURTHER DAMAGE

If the CRD still exists, developers might create NEW Database resources

With no operator running, these will create K8s objects but NO actual databases

Confusing and dangerous



Option A: Delete the CRD (blocks all new resources of this type)

BUT: deleting a CRD deletes ALL its custom resources → cascading deletion

→ would bypass your careful cleanup process

DO NOT delete the CRD yet



Option B: Use a ValidatingWebhook or Kyverno policy to BLOCK creation

of new Database resources until the operator is restored or fully cleaned up



apiVersion: kyverno.io/v1

kind: ClusterPolicy

metadata:

 name: block-database-creation

spec:

 validationFailureAction: Enforce

 rules:

 - name: block-new-databases

   match:

     any:

     - resources:

         kinds:

         - Database

   validate:

     message: "Database operator is being decommissioned. Contact platform team."

     deny: {}



Step 5 (after cleanup): NOW delete the CRD safely

kubectl delete crd databases.mycompany.io

No custom resources left → clean deletion

```

Also: **document what happened.** Write a postmortem or at minimum a runbook: "How to safely decommission a CRD operator." The next time someone removes an operator, this process should be followed, not reinvented.

---

---

Remaining for Phase 2: one more lesson on **Kubernetes operational patterns** — service mesh basics (Istio architecture, because NovaMart uses it), GitOps with ArgoCD (the deployment model), and advanced troubleshooting workflows. Then Phase 2 closes.

# PHASE 2 — Lesson 4: Kubernetes Operational Patterns

---

### Istio Service Mesh — Architecture

```

NovaMart runs Istio. You need to understand the data plane 

and control plane, not just "it does mTLS."



CONTROL PLANE (istiod):

┌──────────────────────────────────────────┐

│                 istiod                    │

│  ┌──────────┐ ┌─────────┐ ┌───────────┐ │

│  │  Pilot    │ │ Citadel │ │  Galley   │ │

│  │          │ │         │ │           │ │

│  │ Service   │ │ Cert    │ │ Config    │ │

│  │ discovery │ │ mgmt    │ │ validation│ │

│  │ + config  │ │ + mTLS  │ │           │ │

│  │ push to   │ │ rotation│ │           │ │

│  │ Envoy     │ │         │ │           │ │

│  └──────────┘ └─────────┘ └───────────┘ │

└──────────────────┬───────────────────────┘

                  │ xDS API (pushes config to every Envoy)

                  │

DATA PLANE (Envoy sidecars):

┌──────────────────▼───────────────────────┐

│  Pod                                      │

│  ┌────────────┐    ┌──────────────────┐  │

│  │    App     │◄──►│  Envoy Proxy     │  │

│  │ Container  │    │  (sidecar)       │  │

│  │            │    │                  │  │

│  │ Thinks it's│    │ - mTLS           │  │

│  │ talking    │    │ - Load balancing  │  │

│  │ directly   │    │ - Retries         │  │

│  │ to other   │    │ - Circuit breaking│  │

│  │ services   │    │ - Observability   │  │

│  │            │    │ - Rate limiting   │  │

│  └────────────┘    └──────────────────┘  │

└──────────────────────────────────────────┘



HOW TRAFFIC FLOWS:

App A sends request to App B (http://service-b:8080/api)

1. iptables rules (injected by Istio init container) intercept ALL outbound traffic

2. Traffic redirected to App A's Envoy sidecar (localhost:15001)

3. Envoy A applies policies: retry, timeout, circuit breaker

4. Envoy A establishes mTLS connection to Envoy B

5. Envoy B receives, terminates mTLS, forwards to App B on localhost

6. App B processes request, returns response

7. Response flows back through both Envoys

\#

The apps never know Istio exists. Zero code changes.

All security, observability, and traffic control is in the mesh.

```

### Istio Resources — What You'll Configure Daily

```yaml

VIRTUAL SERVICE — traffic routing rules:

apiVersion: networking.istio.io/v1beta1

kind: VirtualService

metadata:

 name: api-routing

spec:

 hosts:

 - api-service

 http:

 # Canary: 10% to v2, 90% to v1

 - route:

   - destination:

       host: api-service

       subset: v2

     weight: 10

   - destination:

       host: api-service

       subset: v1

     weight: 90

   

   # Timeout:

   timeout: 5s

   

   # Retries:

   retries:

     attempts: 3

     perTryTimeout: 2s

     retryOn: 5xx,reset,connect-failure,retriable-4xx



 # Header-based routing (testing in production):

 - match:

   - headers:

       x-debug-user:

         exact: "true"

   route:

   - destination:

       host: api-service

       subset: v2        # debug users always get v2



\---

DESTINATION RULE — defines subsets and connection policies:

apiVersion: networking.istio.io/v1beta1

kind: DestinationRule

metadata:

 name: api-destination

spec:

 host: api-service

 trafficPolicy:

   connectionPool:

     tcp:

       maxConnections: 100

     http:

       h2UpgradePolicy: UPGRADE        # use HTTP/2

       maxRequestsPerConnection: 1000

   

   # Circuit breaker:

   outlierDetection:

     consecutive5xxErrors: 5            # 5 errors in a row

     interval: 30s                      # check every 30s

     baseEjectionTime: 30s             # eject for 30s

     maxEjectionPercent: 50            # never eject more than 50% of hosts

   

   # mTLS:

   tls:

     mode: ISTIO\_MUTUAL                # enforce mTLS between services

 

 subsets:

 - name: v1

   labels:

     version: v1

 - name: v2

   labels:

     version: v2



\---

GATEWAY — external traffic entry point (replaces Ingress):

apiVersion: networking.istio.io/v1beta1

kind: Gateway

metadata:

 name: main-gateway

spec:

 selector:

   istio: ingressgateway            # runs on Istio ingress pods

 servers:

 - port:

     number: 443

     name: https

     protocol: HTTPS

   tls:

     mode: SIMPLE

     credentialName: api-tls-cert   # K8s Secret with TLS cert

   hosts:

   - "api.novamart.com"

   - "admin.novamart.com"



\---

PEER AUTHENTICATION — mTLS policy:

apiVersion: security.istio.io/v1beta1

kind: PeerAuthentication

metadata:

 name: default

 namespace: production

spec:

 mtls:

   mode: STRICT                     # ALL traffic must be mTLS

   # PERMISSIVE: accept both mTLS and plaintext (migration mode)

   # STRICT: only mTLS (production)

   # DISABLE: no mTLS



\---

AUTHORIZATION POLICY — L7 access control:

apiVersion: security.istio.io/v1beta1

kind: AuthorizationPolicy

metadata:

 name: payment-access

 namespace: production

spec:

 selector:

   matchLabels:

     app: payment-service

 action: ALLOW

 rules:

 - from:

   - source:

       principals: \["cluster.local/ns/production/sa/order-service"]

       # Only order-service's service account can call payment-service

   to:

   - operation:

       methods: \["POST"]

       paths: \["/api/v1/charge"]

       # Only POST to /api/v1/charge

 # Everything else → DENIED

 # L7-aware: can restrict by HTTP method, path, headers

 # Way more powerful than NetworkPolicy (which is L3/L4 only)

```

### Istio Observability — The Real Value

```bash

Istio gives you THREE pillars for FREE (no app instrumentation):



1. DISTRIBUTED TRACING:

Every Envoy sidecar generates trace spans

Request flow: Gateway → API → Order → Payment → Notification

Each hop creates a span with timing

Requires: app forwards trace headers (x-request-id, x-b3-traceid, etc.)

Backends: Jaeger, Zipkin, Grafana Tempo



2. METRICS:

Every Envoy emits standard metrics:

istio\_requests\_total{source, destination, response\_code}

istio\_request\_duration\_milliseconds{source, destination}

istio\_tcp\_connections\_opened\_total

Scraped by Prometheus automatically

Dashboards: Grafana with Istio dashboard



3. ACCESS LOGS:

Every request logged by Envoy with:

source, destination, method, path, status, latency, bytes

Configurable: JSON format, filter by status code



KIALI — Istio dashboard:

Visual service mesh topology

Real-time traffic flow between services

Health status, error rates, latency

Configuration validation

"Is my VirtualService actually routing correctly?"

```

---

### ArgoCD — GitOps Deployment Model

```

GITOPS PRINCIPLE:

 Git is the single source of truth for cluster state.

 The desired state is declared in Git.

 An automated process ensures the cluster matches Git.

 No manual kubectl apply. Ever.



┌──────────┐    push     ┌──────────┐

│Developer │────────────►│   Git    │

│          │             │  (Bitbucket)│

└──────────┘             └─────┬────┘

                              │ watch

                        ┌─────▼─────┐

                        │  ArgoCD   │

                        │           │

                        │ Compare:  │

                        │ Git state │

                        │ vs        │

                        │ Cluster   │

                        │ state     │

                        └─────┬─────┘

                              │ sync

                        ┌─────▼─────┐

                        │    EKS    │

                        │  Cluster  │

                        └───────────┘

```

```yaml

ARGOCD APPLICATION:

apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

 name: api-service

 namespace: argocd

spec:

 project: production

 

 source:

   repoURL: https://bitbucket.org/novamart/k8s-manifests.git

   targetRevision: main

   path: services/api/overlays/production

   # Or Helm:

   # path: charts/api

   # helm:

   #   valueFiles:

   #   - values-production.yaml

 

 destination:

   server: https://kubernetes.default.svc

   namespace: production

 

 syncPolicy:

   automated:

     prune: true              # delete resources removed from Git

     selfHeal: true           # revert manual changes to match Git

     # Someone kubectl edits a deployment → ArgoCD reverts it

     # This enforces: Git is the ONLY way to change production

   

   syncOptions:

   - CreateNamespace=true

   - PrunePropagationPolicy=foreground

   - ApplyOutOfSyncOnly=true  # only sync changed resources (faster)

   

   retry:

     limit: 5

     backoff:

       duration: 5s

       maxDuration: 3m

       factor: 2



\---

ARGOCD PROJECT — RBAC for applications:

apiVersion: argoproj.io/v1alpha1

kind: AppProject

metadata:

 name: production

 namespace: argocd

spec:

 description: Production services

 sourceRepos:

 - 'https://bitbucket.org/novamart/\*'

 

 destinations:

 - namespace: production

   server: https://kubernetes.default.svc

 - namespace: production-jobs

   server: https://kubernetes.default.svc

 # Can only deploy to production and production-jobs namespaces

 

 clusterResourceWhitelist:

 - group: ''

   kind: Namespace

 # Can create namespaces but not ClusterRoles, etc.

 

 namespaceResourceBlacklist:

 - group: ''

   kind: ResourceQuota

 # Can't modify ResourceQuotas (platform team controls these)

```

### ArgoCD Patterns

```bash

REPO STRUCTURE (recommended):



Option 1: App of Apps

One "root" Application that deploys other Applications

k8s-manifests/

├── apps/                          # ArgoCD Application definitions

│   ├── api-service.yaml

│   ├── order-service.yaml

│   ├── payment-service.yaml

│   └── monitoring.yaml

├── services/

│   ├── api/

│   │   ├── base/                  # Kustomize base

│   │   │   ├── deployment.yaml

│   │   │   ├── service.yaml

│   │   │   └── kustomization.yaml

│   │   └── overlays/

│   │       ├── staging/

│   │       │   └── kustomization.yaml

│   │       └── production/

│   │           └── kustomization.yaml

│   ├── order/

│   └── payment/

└── platform/

   ├── monitoring/

   ├── istio/

   └── cert-manager/



Option 2: ApplicationSet (dynamic generation)

apiVersion: argoproj.io/v1alpha1

kind: ApplicationSet

metadata:

 name: all-services

spec:

 generators:

 - git:

     repoURL: https://bitbucket.org/novamart/k8s-manifests.git

     revision: main

     directories:

     - path: services/\*           # one Application per directory

 template:

   metadata:

     name: '{{path.basename}}'

   spec:

     project: production

     source:

       repoURL: https://bitbucket.org/novamart/k8s-manifests.git

       targetRevision: main

       path: '{{path}}/overlays/production'

     destination:

       server: https://kubernetes.default.svc

       namespace: production

Automatically creates an ArgoCD Application for every service directory

Add a new service directory → ArgoCD app created automatically



SYNC WAVES — ordered deployment:

metadata:

 annotations:

   argocd.argoproj.io/sync-wave: "0"    # namespaces first

\---

metadata:

 annotations:

   argocd.argoproj.io/sync-wave: "1"    # configmaps/secrets second

\---

metadata:

 annotations:

   argocd.argoproj.io/sync-wave: "2"    # deployments last

Lower number = deployed first

Use for: namespace before resources, DB migration before app



SYNC WINDOWS — maintenance windows:

spec:

 syncWindows:

 - kind: deny

   schedule: '0 22 \* \* 5'         # no deploys Friday 10pm

   duration: 60h                   # until Monday 10am

   clusters: \['production']

 - kind: allow

   schedule: '0 9 \* \* 1-5'        # allow deploys weekdays 9am

   duration: 10h

```

### CI/CD Pipeline → ArgoCD Flow

```

TYPICAL FLOW (NovaMart):



1\. Developer pushes code to Bitbucket (services/api)

2\. Jenkins pipeline triggers:

  a. Build → test → lint → security scan

  b. Docker build → push to ECR (my-app:v1.2.3-abc1234)

  c. Update the image tag in k8s-manifests repo:

     sed -i 's|image:.\*|image: ecr/api:v1.2.3-abc1234|' \\

       services/api/overlays/production/kustomization.yaml

     git commit \&\& git push



3\. ArgoCD detects Git change (3-minute poll or webhook)

4\. ArgoCD compares Git state vs cluster state → OutOfSync

5\. ArgoCD syncs: applies updated manifests to cluster

6\. Deployment rolls out new pods

7\. ArgoCD reports: Synced, Healthy



WHY SEPARATE REPOS:

App code repo (Bitbucket): services/api source code

K8s manifests repo (Bitbucket): deployment configs

\#

Separation because:

- Different change velocity (code changes hourly, infra changes weekly)

- Different access control (devs change code, platform team owns manifests)

- CI pipeline updates manifests repo → ArgoCD syncs

- Clean audit trail: "who changed what in production" = Git log

```

---

### Kustomize — Template-Free Configuration

```yaml

Kustomize overlays base manifests without templating

Built into kubectl (kubectl apply -k)



base/kustomization.yaml:

apiVersion: kustomize.config.k8s.io/v1beta1

kind: Kustomization

resources:

\- deployment.yaml

\- service.yaml

\- configmap.yaml



overlays/production/kustomization.yaml:

apiVersion: kustomize.config.k8s.io/v1beta1

kind: Kustomization

resources:

\- ../../base



namePrefix: prod-



commonLabels:

 environment: production



patches:

\- target:

   kind: Deployment

   name: api

 patch: |-

   - op: replace

     path: /spec/replicas

     value: 10

   - op: replace

     path: /spec/template/spec/containers/0/resources/limits/memory

     value: 1Gi



configMapGenerator:

\- name: app-config

 literals:

 - LOG\_LEVEL=warn

 - FEATURE\_NEW\_CHECKOUT=true

 # Generates ConfigMap with content hash suffix

 # app-config-h2d8f → changes trigger pod restart automatically



images:

\- name: my-app

 newName: 123456.dkr.ecr.us-east-1.amazonaws.com/api

 newTag: v1.2.3



KUSTOMIZE vs HELM:

Kustomize: patches over plain YAML, no templating language

Helm: full Go template engine, charts, repos, hooks

\#

Use Kustomize when: your manifests are simple, overlays per environment

Use Helm when: packaging for distribution, complex logic, conditionals

Many teams: Helm for third-party (prometheus, cert-manager)

            Kustomize for internal services

ArgoCD supports both natively

```

---

### Advanced Troubleshooting Workflows

#### Workflow 1: Pod CrashLoopBackOff — Systematic Debug

```bash

Step 1: Get the error

kubectl describe pod <pod>

Look at: Events, State.Reason, State.ExitCode, LastState



Step 2: Check logs

kubectl logs <pod>                    # current attempt

kubectl logs <pod> --previous         # last crash (critical!)

--previous shows the logs from the LAST container instance

Current logs might be empty if container crashes immediately



Step 3: Exit code analysis

0: clean exit (CMD completed — wrong for long-running services)

1: application error (check logs)

126: permission denied on entrypoint

127: entrypoint not found (wrong CMD/ENTRYPOINT, wrong image)

137: SIGKILL (OOM killed)

139: segfault

143: SIGTERM (graceful shutdown — shouldn't cause CrashLoop)



Step 4: If OOMKilled (exit code 137):

kubectl describe pod <pod> | grep -i oom

State.OOMKilled: true

Increase memory limits or fix memory leak



Step 5: If "exec format error" or "not found":

Wrong architecture (AMD64 image on ARM node)

Or: shell form CMD with missing /bin/sh (scratch/distroless image)

Check: docker manifest inspect <image> for platform



Step 6: If container starts then crashes after seconds:

Likely: failed health check → liveness probe kills it

Check: liveness probe configuration

Is initialDelaySeconds long enough for app startup?

Does the health endpoint actually work?

kubectl get events --field-selector involvedObject.name=<pod>

Look for: "Liveness probe failed" events



Step 7: If all else fails — debug container:

kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>

Inspect the filesystem, environment, network from inside the pod

```

#### Workflow 2: Service Not Reachable — End-to-End Debug

```bash

Service: my-service.production.svc.cluster.local:8080

Client pod gets: "connection refused" or timeout



Step 1: Does the service exist?

kubectl get svc my-service -n production

Check: ClusterIP assigned, ports correct, selector matches pods



Step 2: Does the service have endpoints?

kubectl get endpoints my-service -n production

If EMPTY: no pods match the service selector

kubectl get pods -n production -l <selector-from-service> --show-labels

Labels mismatch? Pods not Ready?



Step 3: Is the pod actually listening?

kubectl exec <pod> -- ss -tlnp

Is the app bound to 0.0.0.0:8080 or 127.0.0.1:8080?

127.0.0.1 → only reachable from within the pod, not via Service



Step 4: Can you reach the pod directly?

kubectl exec <client-pod> -- curl http://<pod-ip>:8080

If this works but Service doesn't → kube-proxy/iptables issue

If this fails → app issue or NetworkPolicy blocking



Step 5: DNS resolution working?

kubectl exec <client-pod> -- nslookup my-service.production.svc.cluster.local

If fails → CoreDNS issue

kubectl get pods -n kube-system -l k8s-app=kube-dns

CoreDNS pods running? Check CoreDNS logs



Step 6: NetworkPolicy blocking?

kubectl get networkpolicy -n production

Any policy selecting the target pod?

Does it allow ingress from the source pod's labels/namespace?



Step 7: Istio/service mesh interference?

If Istio is injected:

kubectl exec <pod> -c istio-proxy -- curl localhost:15000/clusters

Shows Envoy's view of endpoints

Check: is the destination cluster showing healthy endpoints?

istioctl analyze -n production

Reports Istio misconfigurations

```

#### Workflow 3: Node Not Ready — Systematic Triage

```bash

kubectl get nodes

NAME           STATUS     ROLES    AGE   VERSION

ip-10-0-1-50   NotReady   <none>   45d   v1.28.2



Step 1: Describe the node

kubectl describe node ip-10-0-1-50

Conditions:

  Ready: False (reason: KubeletNotReady)

  MemoryPressure: True/False

  DiskPressure: True/False

  PIDPressure: True/False



Step 2: Check kubelet

SSH to node (or use SSM Session Manager):

systemctl status kubelet

journalctl -u kubelet --since "30 minutes ago" | tail -100

Common: certificate expired, API server unreachable,

container runtime unresponsive



Step 3: Check container runtime

systemctl status containerd

crictl ps                          # list running containers

crictl pods                        # list pods

If containerd is down → all containers dead → node NotReady



Step 4: Check disk

df -h                              # disk space

df -i                              # inodes

Disk full → kubelet can't function → DiskPressure → NotReady

Common cause: container logs not rotated, image cache full



Step 5: Check resources

free -h                            # memory

top                                # CPU/process count

Memory exhaustion → OOM → kubelet process killed → NotReady



Step 6: EKS-specific

Check EC2 instance status:

aws ec2 describe-instance-status --instance-ids <id>

System status check failed → hardware issue → AWS should replace

Instance status check failed → OS issue → may need reboot



Step 7: Recovery

If fixable (disk full, restart kubelet):

sudo systemctl restart kubelet

If not fixable:

kubectl drain ip-10-0-1-50 --ignore-daemonsets --delete-emptydir-data

kubectl delete node ip-10-0-1-50

Cluster autoscaler/Karpenter replaces it

```

# 📋 QUICK REFERENCE — K8s Operational Patterns

```

ISTIO SERVICE MESH:

 Control plane: istiod (Pilot + Citadel + Galley)

 Data plane: Envoy sidecars (auto-injected)

 Traffic flow: App → iptables → Envoy A → mTLS → Envoy B → App

 VirtualService: routing, canary %, retries, timeouts, header matching

 DestinationRule: subsets, circuit breaker, connection pools, mTLS

 Gateway: external traffic entry (replaces Ingress)

 PeerAuthentication: STRICT (mTLS only) or PERMISSIVE (migration)

 AuthorizationPolicy: L7 access control (method, path, service account)

 Observability: metrics + traces + access logs for FREE



ARGOCD GITOPS:

 Git = single source of truth

 Application: source (Git) → destination (cluster)

 syncPolicy.automated: prune + selfHeal

 App of Apps: root Application deploys child Applications

 ApplicationSet: dynamic Application generation

 Sync waves: ordered deployment (namespace → config → app)

 Sync windows: prevent deploys during risky periods

 CI updates Git → ArgoCD syncs cluster



KUSTOMIZE:

 base/ + overlays/env/ pattern

 Patches, namePrefix, commonLabels, images

 configMapGenerator: auto-hash suffix → triggers pod restart

 Built into kubectl (kubectl apply -k)



TROUBLESHOOTING:

 CrashLoopBackOff:

   logs --previous → exit code → OOM? permission? architecture?

   → liveness probe misconfigured? → debug container

 Service unreachable:

   svc exists? → endpoints? → pod listening on 0.0.0.0? 

   → direct pod curl → DNS → NetworkPolicy → Istio

 Node NotReady:

   describe → kubelet status → containerd → disk → memory

   → EC2 status → drain + replace

```

---

# 📝 Retention Questions — Lesson 4

**Q1:** NovaMart is rolling out a new version of the payment service. They want to send 5% of traffic to v2, monitor error rates for 30 minutes, then gradually increase to 100%. Write the Istio VirtualService and DestinationRule for the initial 5% canary, and explain what happens if v2's error rate spikes — how does Istio's circuit breaker protect the system?

**Q2:** A developer manually runs `kubectl edit deployment api -n production` to increase replicas from 5 to 10. Two minutes later, the replicas are back to 5. They're confused. Explain what happened and why this is the correct behavior.

**Q3:** Your CI pipeline (Jenkins) builds a new image and needs to trigger a deployment via ArgoCD. Describe the complete flow from code push to running pods, and explain why Jenkins should NOT directly `kubectl apply` to the cluster.

**Q4:** It's 3 AM. PagerDuty alert: Node `ip-10-0-1-50` is `NotReady`. 15 pods were running on it including 3 critical API pods. Walk through your triage and recovery process step by step.

**Go.** 🎯

📝 GRADING — Q5
Q5: ✅ Good, but still missing one critical step
Your reasoning is right:

finalizers are a safety lock
deleting them first leaks cloud resources
inventory first
manual cleanup second
strip finalizers last
That's the correct backbone.

But you skipped the proper first move:

The first thing you should try is restoring the operator
You went straight to becoming the operator manually. That's workable, but not the best answer.

If the operator still exists in source control / Helm chart / image registry, the cleanest path is:

Reinstall or restore the operator
Let it reconcile the deletionTimestamp
Let it clean up EBS + Route53
Let it remove its own finalizers naturally
That preserves intended cleanup logic, ordering, retries, idempotency, and any edge cases the operator encoded that your bash script will absolutely miss because bash scripts are where rigor goes to die.

You should only do manual cleanup if:

the operator image is gone
the CRD/controller version is incompatible
the operator is permanently retired
the cleanup logic is known and simple enough to reproduce safely
What a stronger answer looks like
Proper sequence:
text
1. Freeze further deletes/changes
2. Inventory stuck CRs and referenced external resources
3. Try to restore the deleted operator/controller
4. If restore succeeds:
     - let it reconcile deletions
     - monitor cleanup
     - verify finalizers removed naturally
5. If restore is impossible:
     - manually clean up external dependencies
     - verify cleanup
     - remove finalizers
6. Audit for stragglers and cost leaks
7. Add prevention: uninstall procedure, admission guardrails, runbook
What else you were missing
1. Verification must happen in AWS, not just Kubernetes
Your final verification was:

bash
kubectl get database.mycompany.com
That's not enough. That's half a check.

You also need to verify:

EBS volumes deleted
Route53 records removed
any related snapshots, ENIs, security groups, tags, alarms, backups, IAM artifacts, or load balancer targets aren't left behind
Because "resource disappeared from etcd" does not mean cleanup happened. It just means you've stopped seeing the corpse.

2. You should protect against accidental deletion of in-use resources
You said:

delete the volumes (after verifying they aren't being used)

Good instinct, but too vague.

For EBS, you should explicitly check:

bash
aws ec2 describe-volumes --volume-ids vol-xxx
Look for:

State: in-use vs available
current attachments
tags identifying ownership
snapshots/retention implications
For Route53:

verify hosted zone
exact record set
weighted/failover/alias implications
whether other systems depend on the record
Blind deletion is how people create the second outage while fixing the first.

3. Your patch is fine, but target only stuck objects
Don't patch every CR blindly if some are healthy and not terminating.

Safer pattern:

bash
kubectl get database.mycompany.com -o json | \
jq -r '.items[]
| select(.metadata.deletionTimestamp != null)
| .metadata.name'
Then patch only those.

4. Prevention matters
A senior answer ends with prevention, not just cleanup.

You should add:

documented decommission procedure: delete CRs first, wait for cleanup, uninstall operator last
Helm pre-delete hook / runbook checks
admission policy blocking operator uninstall while managed CRs still exist
monitoring for stuck finalizers / terminating resources
periodic orphaned cloud resource audit by tag
📊 SCORE: 4.5/5
Why not full marks?

You missed restore the operator first, which is the cleanest and most correct production answer.
Verification was Kubernetes-only, not end-to-end.
Prevention wasn't included.
Still, the core judgment was good. You did not fall into the idiot trap of “just patch finalizers and move on.”

PHASE 2 — Lesson 4: Kubernetes Day-2 Operations, Failures, and Production Realities
This is the lesson that should have already existed. Kubernetes basics are nice. Production pain is nicer.

1. CrashLoopBackOff — What It Actually Means
bash
kubectl get pods
# api-abc   0/1   CrashLoopBackOff

# This does NOT mean "Kubernetes crashed the pod."
# It means:
# 1. Container started
# 2. Container exited quickly
# 3. kubelet restarted it
# 4. It exited again
# 5. kubelet applies exponential backoff before next restart

# Backoff sequence:
# 10s → 20s → 40s → 80s → 160s → 300s cap

# Debug:
kubectl logs pod/api-abc --previous
# --previous is critical. Current container may not have logs yet.

kubectl describe pod/api-abc
# Look at:
# - Last State: Terminated
# - Exit Code
# - Reason
# - Events

# Common causes:
# - Missing env var / secret
# - Bad command/entrypoint
# - Port already in use
# - App exits immediately after finishing one-shot task
# - Liveness probe killing a slow-start app
# - OOMKilled
# - Permission denied on mounted volume
# - Wrong architecture image (arm64 on amd64 node)
Exit Codes You Must Know
text
0   = clean exit (probably wrong workload type; should be Job, not Deployment)
1   = generic app failure
126 = command invoked cannot execute (permissions)
127 = command not found
137 = SIGKILL (OOM kill or forced stop)
139 = segfault
143 = SIGTERM (normal stop)
2. ImagePullBackOff / ErrImagePull
bash
kubectl get pods
# api-xyz   0/1   ImagePullBackOff

kubectl describe pod api-xyz
# Events:
# Failed to pull image "myrepo/api:v1.2.3":
# rpc error: code = Unknown desc = Error response from daemon:
# pull access denied / manifest unknown / no basic auth credentials

# Causes:
# 1. Tag doesn't exist
# 2. Wrong registry URL
# 3. Private registry auth failure
# 4. ECR permissions missing
# 5. Rate limiting from Docker Hub
# 6. Architecture mismatch in manifest
# 7. Network path to registry blocked
# 8. Node can't reach S3 for ECR layer download (private EKS gotcha)
Private EKS + ECR Failure Chain
text
Pod scheduled
→ kubelet asks containerd to pull image from ECR
→ manifest request to ECR succeeds
→ actual image layers live in S3
→ node has no S3 Gateway Endpoint / NAT path
→ layer download fails
→ ImagePullBackOff

This is the dependency almost everyone misses.
3. Init Container Failures
bash
kubectl get pods
# api-xyz   0/1   Init:CrashLoopBackOff

# Main app container never starts because init containers run first.
kubectl logs api-xyz -c wait-for-db
kubectl describe pod api-xyz

# Common mistakes:
# - "wait-for-db" loops forever because service DNS wrong
# - migration script fails due to missing secret
# - init container image missing tooling (nc, curl, bash)
# - permission fix init container runs as non-root and can't chown
4. Pending But For Different Reasons Than Scheduling
You already learned CPU, taints, affinity. Here are the rest.

PVC Wait
bash
kubectl get pod
# db-0   0/1   Pending

kubectl describe pod db-0
# Warning  FailedScheduling  pod has unbound immediate PersistentVolumeClaims

# Translation:
# scheduler refuses to bind pod because PVC isn't ready
# This is storage, not compute
Pod Security / Admission Rejection
Sometimes the pod never exists.

bash
kubectl apply -f pod.yaml
# Error from server: admission webhook denied the request:
# container must set securityContext.runAsNonRoot=true

# Or:
# violates PodSecurity "restricted:latest"
This is not a pod problem. It's an API admission problem.

5. Node NotReady — What Usually Broke
bash
kubectl get nodes
# ip-10-0-1-10   NotReady

kubectl describe node ip-10-0-1-10
# Conditions:
#   Ready              False
#   MemoryPressure     False
#   DiskPressure       False
#   PIDPressure        False
# Events:
#   NodeNotReady

# Common causes:
# 1. kubelet stopped / crashed
# 2. container runtime (containerd) dead
# 3. CNI plugin broken → kubelet can't report network ready
# 4. Node lost API server connectivity
# 5. Disk full → kubelet or runtime malfunction
# 6. Expired node cert (self-managed clusters)
# 7. EC2 instance impaired / underlying cloud issue
Node debugging sequence
bash
# On the node:
systemctl status kubelet
systemctl status containerd
journalctl -u kubelet -xe
journalctl -u containerd -xe
df -h
df -i
ip addr
ip route
crictl ps -a
crictl logs <container-id>
For EKS, you often use:

SSM Session Manager
EC2 serial console
node debug pod via kubectl debug node/...
6. DNS Failures in Production
You learned CoreDNS and ndots:5. Here's production pain.

Symptom patterns
text
- Some lookups slow, some instant
- External APIs time out only from K8s
- Pod can ping IP but not hostname
- DNS works on one node, fails on another
- Intermittent NXDOMAIN / SERVFAIL
Real causes
text
1. CoreDNS overloaded
2. NodeLocal DNSCache absent at scale
3. Upstream VPC resolver throttling
4. `ndots:5` causing search path amplification
5. Broken NetworkPolicy blocking kube-dns
6. CoreDNS pods not spread across nodes/AZs
7. One node has conntrack exhaustion → DNS UDP drops
8. MTU mismatch causing fragmented DNS over UDP issues
NodeLocal DNSCache
text
Without NodeLocal DNSCache:
Pod → CoreDNS service → kube-proxy → CoreDNS pod

With NodeLocal DNSCache:
Pod → 169.254.20.10 on same node → local cache → CoreDNS only on miss

Benefits:
- lower latency
- less conntrack churn
- less CoreDNS load
- fewer UDP packet drops
If you're running large EKS clusters and not using NodeLocal DNSCache, you're choosing pain for free.

7. CoreDNS Failure Modes
bash
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs deployment/coredns
kubectl -n kube-system top pods -l k8s-app=kube-dns
Look for:

OOMKilled
throttling due to low CPU limits
SERVFAIL spam
upstream timeout errors
plugin loop errors
no endpoints for kube-dns service
Production fixes
scale replicas
add HPA
add PDB
topology spread across AZs
use NodeLocal DNSCache
ensure egress to upstream resolver
tune cache size
8. CNI Failures and IP Exhaustion
You learned ENI IP limits. Here's how it burns you operationally.

Symptoms
text
- New pods stuck ContainerCreating
- Existing pods fine
- Error mentions failed to assign IP
- aws-node DaemonSet logs show IP allocation failures
bash
kubectl logs -n kube-system ds/aws-node
# failed to assign an IP address to container
# insufficient IP addresses available in subnet
Causes
subnet CIDR exhausted
instance ENI/IP slot limit reached
prefix delegation disabled
warm IP target misconfigured
aws-node DaemonSet unhealthy
Fixes
enable prefix delegation
use larger subnets / secondary CIDR
scale node group across more subnets
tune WARM_IP_TARGET / MINIMUM_IP_TARGET carefully
move to custom networking if needed
9. ContainerCreating — The Most Annoying Fake Status
bash
kubectl get pods
# api-xyz   0/1   ContainerCreating

# That's not a real diagnosis. That's a waiting room.
kubectl describe pod api-xyz
Common actual causes:

image pull in progress / failing
CNI IP allocation failure
volume mount failure
secret/configmap not found
CSI attach/mount timeout
sandbox creation failed
runtime shim issue
Read the events. Always.
10. Probes — How Teams DDoS Themselves
Bad probes kill healthy apps.

Classic failures
liveness hits DB and restarts app when DB blips
startup too slow, liveness kills app forever
readiness too permissive, traffic routed before warmup
probe endpoint expensive, causes CPU spikes
probe timeout too low under GC pauses
Real rules
text
Liveness:
  "Should kubelet restart me because I'm irrecoverably wedged?"
  Keep cheap and internal.

Readiness:
  "Should traffic be sent to me right now?"
  Can include downstream dependency health.

Startup:
  "Don't let liveness/readiness touch me until boot is complete."
11. OOMKilled in Kubernetes — More Than "Increase Memory"
bash
kubectl describe pod api-xyz
# Last State: Terminated
# Reason: OOMKilled
Questions to ask
Did app actually leak?
Is limit unrealistically low?
Is non-heap/native memory the culprit?
Is page cache involved?
Is sidecar consuming most of the memory?
Did VPA recommend something else?
Is requests too low causing bad packing on nodes?
Is node-level memory pressure causing eviction, not cgroup OOM?
Important distinction
text
Container OOM:
  cgroup limit hit
  container killed
  pod may restart
  reason often OOMKilled

Node memory pressure eviction:
  kubelet evicts pod to save node
  reason often Evicted
  different root cause
People mix these up constantly.

12. CPU Throttling — The Silent Performance Killer
bash
# Pod has:
limits.cpu: 500m
requests.cpu: 500m

# App under load wants 1 CPU
# Linux CFS quota throttles it to 0.5 CPU
# Latency spikes, no crash, no obvious logs
Symptoms
high latency
low actual CPU usage on dashboards
app appears "slow" not broken
GC pauses worse
request queueing
What to monitor
container_cpu_cfs_throttled_periods_total
container_cpu_cfs_throttled_seconds_total
Fix
raise or remove CPU limits for latency-sensitive workloads
keep requests accurate
don't blindly set equal request=limit for CPU everywhere
memory limits are safety; CPU limits are throttling
A lot of teams should remove CPU limits on critical services and keep requests only.

13. Pod Eviction vs Preemption vs Deletion
These are different. Stop treating them as synonyms.

text
Eviction:
  kubelet/API-driven removal due to pressure or drain
  respects PDBs for voluntary disruptions

Preemption:
  scheduler evicts lower-priority pods to schedule higher-priority pod

Deletion:
  user/controller requested object removal

OOM kill:
  kernel kills process inside cgroup
Different cause. Different evidence. Different fix.

14. Services With No Endpoints
bash
kubectl get svc api
kubectl get endpoints api
# ENDPOINTS: <none>
Why?

selector labels don't match pod labels
pods not Ready
wrong namespace
headless service confusion
EndpointSlice controller issue
Debug
bash
kubectl get pods -l app=api --show-labels
kubectl describe svc api
kubectl get endpointslices -l kubernetes.io/service-name=api
If readiness fails, service has no endpoints even if pods are Running. Running is not Ready. Kubernetes does not care about your feelings.

15. EndpointSlice vs Endpoints
Large clusters use EndpointSlice. If you're only checking Endpoints, you're already behind.

text
Endpoints:
  legacy single object listing all pod IPs

EndpointSlice:
  scalable split across multiple objects
  default modern mechanism
  supports dual stack, scaling improvements
Use:

bash
kubectl get endpointslices
16. Jobs and CronJobs in Production
CronJob gotchas
missed schedule after controller downtime
overlapping runs
job history explosion
long-running jobs piling up
timezone confusion
yaml
spec:
  schedule: "0 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  startingDeadlineSeconds: 300
  suspend: false
Rules
backups: Forbid
idempotent reconciler jobs: maybe Replace
always set history limits
always make jobs idempotent
always define resource requests/limits
always think about retry side effects
17. Secrets Rotation Reality
Kubernetes makes secrets storage easy, rotation hard.

Truths
env var injected secrets require restart
mounted secret volumes update eventually
app may not reload
DB password rotation can break long-lived connections
external secret refresh timing matters
dual credential overlap window is often required
Real production pattern
create new credential
allow old + new temporarily
update secret store
rotate pods / app reload
confirm new credential in use
revoke old credential
If you rotate secrets without overlap, enjoy your outage.

18. Config Drift and Manual Changes
bash
kubectl edit deployment api
This is how drift starts.

If you use GitOps:

manual change works now
ArgoCD/Flux reverts it later
team gets confused
outage postmortem says "unknown rollback"
Rules
prod changes through Git
emergency hotfixes must be backported to Git immediately
use kubectl diff
audit drift regularly
19. Resource Requests — Scheduling Lies
If requests are too low:

scheduler overpacks nodes
node pressure later
noisy neighbors
autoscaler under-reacts
If requests are too high:

wasted money
pods Pending unnecessarily
poor binpacking
autoscaler overreacts
Senior rule
Requests should reflect steady-state p50/p90, not max fantasy or idle minimum.

Use VPA recommendations as input, not gospel.

20. API Server and Control Plane Bottlenecks
Not every outage is "the app."

Symptoms
kubectl slow everywhere
controllers lagging
deployments stuck
endpoints stale
HPA delayed
webhook timeouts cluster-wide
Causes
bad webhook latency
too many watches
huge CRDs / object size abuse
etcd latency
controller storms after mass resync
excessive churn from short-lived jobs
In managed EKS you don't fix API server internals, but you must recognize the pattern quickly.

21. Webhooks — The Cluster-Wide Footgun
You learned admission controllers. Here's the operational truth:

If a validating or mutating webhook is slow or down:

pod creates fail
deploys fail
cert renewals may fail
namespace deletes may hang
CRDs may fail to create
Always define
timeoutSeconds
failurePolicy
HA replicas
PDB
readiness probes
no dependency on fragile downstreams
A broken webhook can make the cluster look haunted.

22. Ingress / Load Balancer Realities
Common real issues
health check path wrong
targets healthy in one AZ only
readiness probe says ready but app fails LB health check
NLB idle timeout mismatch
ALB target registration lag
wrong service type/annotations
TLS cert attached to wrong listener
source IP expectation broken
Must check
Ingress events
LB controller logs
target group health
SG rules both ways
subnet tags
externalTrafficPolicy effects
cross-zone balancing config
23. Node Pressure From Disk, Not Memory
Disk kills clusters quietly.

Causes
giant container logs
image cache bloat
emptyDir abuse
core dumps
temp files
Prometheus WAL explosion
overlay2 inode exhaustion
Checks
bash
df -h
df -i
du -sh /var/lib/containerd
du -sh /var/log/containers
du -sh /var/lib/kubelet
If you only look at memory, you're debugging blind.

24. "It Works In Docker Compose" — Why It Dies In K8s
Real reasons:

localhost assumption broken
root filesystem read-only
non-root user enforced
service DNS different
probes kill slow startup
missing resource requests/limits
secret/config injection different
no persistent storage semantics
init ordering assumption wrong
sidecars change startup/termination timing
Compose teaches convenience. Kubernetes charges interest on that debt.

25. Runbooks and Postmortem Thinking
Being production-ready means more than knowing commands.

For every recurring class of failure, you should be able to produce:

Runbook
symptoms
dashboards
logs/commands
decision tree
mitigation
escalation path
rollback criteria
Postmortem
impact
timeline
root cause
contributing factors
detection gaps
action items
owner + due date
If knowledge lives only in your head, your team doesn't have a system. It has a dependency.


📋 QUICK REFERENCE — K8s Day-2 Operations
text
CRASHLOOP:
  logs --previous
  describe pod
  check exit code, probe config, env vars, permissions, OOM

IMAGE PULL:
  describe pod events
  verify tag, registry auth, ECR perms, S3 endpoint/NAT path

NODE NOTREADY:
  kubelet, containerd, disk, CNI, API connectivity
  journalctl, crictl, df -h, df -i

DNS:
  CoreDNS load, NodeLocal DNSCache, NetworkPolicy to kube-dns,
  ndots amplification, conntrack drops

PROBES:
  Liveness = deadlock detector
  Readiness = traffic gate
  Startup = slow boot protection
  Never put fragile dependencies in liveness

MEMORY:
  OOMKilled = cgroup kill
  Evicted = kubelet/node pressure
  Different problem, different fix

CPU:
  limits can throttle without obvious failure
  monitor CFS throttling metrics

SERVICE NO ENDPOINTS:
  labels, readiness, namespace, EndpointSlice

CRONJOBS:
  set concurrencyPolicy, history limits, startingDeadlineSeconds
  make jobs idempotent

DRIFT:
  prod via Git, not kubectl edit
  emergency fix must be backported immediately

---

# Lesson 3C: Everything I Owe — Operational Failure Modes

---

### CPU Throttling — The Silent Performance Killer

```bash

CPU LIMITS USE CFS (Completely Fair Scheduler) BANDWIDTH CONTROL

This is NOT "you get 1 CPU core worth of compute"

It's: "you get X microseconds every 100ms window"



Example:

Container: limits.cpu = 1000m (1 core)

CFS period: 100ms (100,000 µs)

CFS quota: 100,000 µs per period

Container can use 100ms of CPU time per 100ms window



THE THROTTLING PROBLEM:

Your app has 8 threads. All 8 burst simultaneously.

In the first 12.5ms, all 8 threads consume 100ms of CPU time

(8 threads × 12.5ms = 100ms quota used)

For the remaining 87.5ms of the window → ALL THREADS FROZEN

EVERY request arriving during those 87.5ms → queued → latency spike



Monitoring shows: average CPU = 30%

Reality: app is frozen 87.5% of each 100ms window during bursts

p99 latency: through the roof

Average latency: looks fine (because most requests hit unfrozen windows)



HOW TO DETECT:

On the node:

cat /sys/fs/cgroup/cpu,cpuacct/kubepods/pod<uid>/<container-id>/cpu.stat

nr\_periods: 125000        ← number of 100ms periods elapsed

nr\_throttled: 45000       ← number of periods where throttling occurred

throttled\_time: 28000000  ← total ns spent throttled



Throttle ratio: 45000/125000 = 36% of periods had throttling

That's BAD.



Prometheus metrics (via cadvisor):

container\_cpu\_cfs\_throttled\_periods\_total

container\_cpu\_cfs\_periods\_total

Throttle ratio = throttled\_periods / total\_periods

Alert when: ratio > 25% sustained for 5 minutes



FIXES:



Option 1: Increase CPU limits

resources:
 requests:
   cpu: 500m
 limits:
   cpu: 2000m    # 4x headroom for bursts



Option 2: Remove CPU limits entirely

resources:
 requests:
   cpu: 500m

 # NO limits.cpu

Container can burst to any available CPU on the node

requests.cpu still guarantees scheduling and proportional sharing

Under contention: CPU divided proportionally by requests

No contention: container uses whatever is available

Used by: Google (Borg doesn't enforce CPU limits), many SRE teams



Option 3: Use GOMAXPROCS / thread tuning

Reduce thread count to match CPU limit

1 CPU limit → GOMAXPROCS=1 (Go)

1 CPU limit → -XX:ActiveProcessorCount=1 (Java)

Fewer threads = less simultaneous burst = less throttling



THE DEBATE:

Pro limits: prevents noisy neighbor, predictable performance

Anti limits: CFS throttling causes more latency issues than it prevents

Google's stance: use requests only, don't set limits

Most companies: set limits conservatively high (2-4x requests)

NovaMart approach: limits = 4x requests for API services,

  no limits for batch jobs on dedicated node pools

```

---

### ContainerCreating — The Umbrella Failure State

```bash

Pod status: ContainerCreating

This can hide MANY different failures:



1. Image pull in progress (slow registry, large image)

kubectl describe pod <pod>

Events: "Pulling image my-app:v1.2.3"

Slow but not broken. Wait or check registry/network.



2. Image pull failure (about to become ImagePullBackOff)

Events: "Failed to pull image: rpc error: code = Unknown desc = 

  Error response from daemon: manifest not found"

Wrong tag, wrong registry, auth failure



3. Volume mount failure

Events: "Unable to attach or mount volumes: 

  timed out waiting for the condition"

EBS volume stuck attaching (previous node didn't detach cleanly)

NFS server unreachable

Secret/ConfigMap doesn't exist



4. CNI failure — no IP available

Events: "Failed to create pod sandbox: 

  rpc error: add cmd: failed to assign an IP address to container"

VPC CNI: ENI IP pool exhausted

Fix: prefix delegation, larger instance, or wait for IP recycling



5. Sandbox creation failure (container runtime issue)

Events: "Failed to create pod sandbox: 

  rpc error: context deadline exceeded"

containerd/runc issue on the node

Often fixed by: restarting containerd or replacing the node



6. Init container still running

If pod has init containers, main containers stay in ContainerCreating

until ALL init containers complete successfully

kubectl get pod <pod> -o jsonpath='{.status.initContainerStatuses}'

Check if init container is stuck/failing



KEY DEBUGGING COMMAND:

kubectl describe pod <pod>

The Events section tells you EXACTLY which sub-step failed

Always check Events, not just Status

```

---

### ImagePullBackOff — Full Debugging

```bash

kubectl describe pod <pod>

Events:

Failed to pull image "my-app:v1.2.3": 

  rpc error: code = Unknown desc = Error response from daemon



CAUSES:



1. Image doesn't exist:

Wrong tag, wrong repository name, typo

Verify: aws ecr describe-images --repository-name my-app --image-ids imageTag=v1.2.3



2. Authentication failure:

ECR token expired (12-hour lifetime)

imagePullSecrets not configured or wrong

Cross-account ECR: repository policy missing

Private registry: credentials wrong

Verify:

kubectl get pod <pod> -o jsonpath='{.spec.imagePullSecrets}'

kubectl get secret <pull-secret> -o jsonpath='{.data.\\.dockerconfigjson}' | base64 -d



3. Network unreachable (private cluster):

No NAT Gateway and no VPC endpoints for ECR + S3

Node can't reach ECR API or S3 (where layers live)

Verify: SSH to node, curl https://api.ecr.us-east-1.amazonaws.com/



4. Rate limiting:

Docker Hub: 100 pulls/6hr (anonymous), 200 pulls/6hr (authenticated)

Events: "toomanyrequests: You have reached your pull rate limit"

Fix: use ECR pull-through cache, or mirror images to private registry



5. Architecture mismatch:

Image built for linux/amd64, node is ARM (Graviton)

Events: "exec format error" after pull succeeds

Verify: docker manifest inspect <image> | grep architecture



6. Image too large + slow network:

5GB image on a node with limited bandwidth

Not technically an error, just slow

Fix: smaller images, pre-pull DaemonSet for large images



BACKOFF PROGRESSION:

ImagePullBackOff follows exponential backoff:

10s → 20s → 40s → 80s → 160s → 300s (5 min cap)

After fixing the issue, pod retries on the next backoff cycle

To force immediate retry: delete and recreate the pod

```

---

### CoreDNS Operational Scaling

```bash

DEFAULT: 2 CoreDNS pods in EKS

At scale, this is NOT enough



SYMPTOMS OF CoreDNS OVERLOAD:

- DNS resolution timeouts from pods

- dmesg: "nf\_conntrack: table full" (DNS uses UDP → conntrack)

- CoreDNS logs: "i/o timeout" when forwarding to upstream

- Prometheus: coredns\_dns\_requests\_total increasing, 

  coredns\_dns\_responses\_total not keeping up



SCALING COREDNS:



1. Horizontal — more replicas:

kubectl -n kube-system edit deployment coredns

Or better: use dns-autoscaler addon

Scales CoreDNS pods based on node/core count in cluster

Formula: replicas = max(ceil(cores / 256), ceil(nodes / 16))

100 nodes → max(ceil(400/256), ceil(100/16)) = max(2, 7) = 7 replicas



2. NodeLocal DNSCache (critical for production at scale):

Runs a DNS cache daemon on EVERY node as a DaemonSet

Pods resolve DNS against the LOCAL cache (avoid cross-node hops)

Cache hit → instant response (no network)

Cache miss → forwards to CoreDNS



Benefits:

- Reduces CoreDNS load by 80-90%

- Eliminates conntrack entries for DNS (uses TCP to CoreDNS)

- Eliminates "DNS 5-second timeout" race condition

  (the infamous glibc parallel A + AAAA query bug)

- Sub-millisecond DNS for cached entries



Enable on EKS:

Deploy node-local-dns DaemonSet

Configure kubelet to use node-local-dns IP (169.254.20.10)

Pods automatically resolve against local cache



3. CoreDNS Corefile tuning:

apiVersion: v1

kind: ConfigMap

metadata:

 name: coredns

 namespace: kube-system

data:

 Corefile: |

   .:53 {

       errors

       health

       kubernetes cluster.local in-addr.arpa ip6.arpa {

           pods insecure

           fallthrough in-addr.arpa ip6.arpa

       }

       forward . /etc/resolv.conf {

           max\_concurrent 5000         # increase from default 1000

       }

       cache 30                        # cache TTL (default 30s)

       loop

       reload

       loadbalance

       # Add: autopath to reduce ndots:5 query amplification

       autopath @kubernetes

       # Without autopath: pod resolving "google.com" generates 5 queries

       # With autopath: CoreDNS short-circuits the search path

   }



4. Reduce ndots-related query amplification:

Default ndots:5 means any name with < 5 dots goes through search path

"redis-master" generates:

  redis-master.default.svc.cluster.local

  redis-master.svc.cluster.local

  redis-master.cluster.local

  redis-master.ec2.internal

  redis-master              ← finally, the actual query

That's 5 DNS queries for one resolution!

\#

Fix per pod:

spec:
 dnsConfig:
   options:
   - name: ndots
     value: "2"         # reduce from 5 to 2

   # Names with < 2 dots go through search path

   # "redis-master" (0 dots) → search path (correct, finds service)

   # "google.com" (1 dot) → search path first, then absolute

   # "api.google.com" (2 dots) → absolute query directly (no wasted queries)

```

---

### EndpointSlice — The Scalable Endpoint

```bash

ENDPOINTS (legacy):

One Endpoints object per Service containing ALL pod IPs

Service with 5000 pods → one Endpoints object with 5000 IPs

Every kube-proxy watches ALL Endpoints

One pod changes → entire Endpoints object updated → pushed to ALL kube-proxies

At scale: massive watch traffic, slow updates



ENDPOINTSLICE (default since K8s 1.21):

Service with 5000 pods → 50 EndpointSlice objects (100 IPs each)

When one pod changes → only ONE EndpointSlice updated

Pushed only to kube-proxies that need it

100x reduction in API server and etcd load at scale



kubectl get endpointslice -n production

NAME                   ADDRESSTYPE   PORTS   ENDPOINTS               AGE

api-service-abc12      IPv4          8080    10.0.1.5,10.0.1.6...   5d

api-service-def34      IPv4          8080    10.0.2.7,10.0.2.8...   5d



kubectl get endpointslice api-service-abc12 -o yaml

Shows: individual endpoint conditions (ready, serving, terminating)

More granular than legacy Endpoints



WHY YOU SHOULD KNOW THIS:

If you're debugging "Service has endpoints but traffic isn't reaching pods"

Check EndpointSlices, not just Endpoints:

kubectl get endpointslice -l kubernetes.io/service-name=my-service

Look for: conditions.ready = false on specific endpoints

A pod might be in the EndpointSlice but marked not-ready

```

---

### Probe Anti-Patterns

```yaml

ANTI-PATTERN 1: Checking dependencies in liveness probe

livenessProbe:
 httpGet:
   path: /health
   port: 8080

/health checks: DB connection + Redis + external API

If Redis goes down:

→ liveness fails → container restarted

→ new container starts → Redis still down → liveness fails again

→ CrashLoopBackOff for ALL pods

→ TOTAL OUTAGE of your service because Redis had a blip



FIX: liveness checks ONLY the process itself (is it responsive?)

/healthz: return 200 if the HTTP server can respond. That's it.

/ready: check dependencies. Readiness probe removes from traffic.



ANTI-PATTERN 2: initialDelaySeconds too low

livenessProbe:
 httpGet:
   path: /healthz
   port: 8080
 initialDelaySeconds: 5      # Java app takes 45s to start
 failureThreshold: 3
 periodSeconds: 10

Sequence: pod starts → 5s delay → check → fail (app still booting)

→ 10s → check → fail → 10s → check → fail → RESTART

→ pod starts again → same thing → CrashLoopBackOff

App never gets a chance to boot

\#

FIX: use startupProbe for slow starters

startupProbe:
 httpGet:
   path: /healthz
   port: 8080
 failureThreshold: 30         # 30 × 10s = 300s = 5 min to start
 periodSeconds: 10

Liveness/readiness disabled until startup probe passes



ANTI-PATTERN 3: Probe hitting expensive endpoint

livenessProbe:
 httpGet:
   path: /api/v1/status       # runs DB query, aggregates metrics
   port: 8080
 periodSeconds: 5

Every 5 seconds: full status check with DB query

Under load: probe adds 20% overhead

Probe timeout → liveness fails → restart → makes load worse

\#

FIX: dedicated lightweight health endpoint

/healthz: return 200, no computation, < 1ms response



ANTI-PATTERN 4: Same endpoint for liveness AND readiness

They serve different purposes:

Liveness: "restart me if I'm broken"

Readiness: "stop sending traffic if I'm not ready"

Same endpoint = same failure mode = wrong action

Readiness failure on dependency → should remove from traffic

If liveness uses same endpoint → restarts instead → wrong

\#

FIX: separate endpoints

/healthz → liveness (process health only)

/ready → readiness (process + dependencies)



ANTI-PATTERN 5: Aggressive probe settings on high-latency apps

livenessProbe:
 timeoutSeconds: 1            # 1 second timeout
 periodSeconds: 5
 failureThreshold: 1          # ONE failure = restart

Under GC pause or load spike, one slow response = restart

Creates cascading restarts under load

\#

FIX: generous thresholds

livenessProbe:

 timeoutSeconds: 5

 periodSeconds: 15

 failureThreshold: 3          # 3 failures = 45s before restart

```

---

### OOMKilled vs Evicted vs Preempted vs Deleted — The Comparison

```bash

Four ways a pod can die. Each has different cause, behavior, and response.



1. OOMKilled (container-level):

CAUSE: Container exceeded its memory LIMIT

WHO: Linux kernel OOM killer (cgroup enforcement)

SCOPE: single container (other containers in pod may survive)

RESTART: yes (if restartPolicy allows)

DETECT:

kubectl describe pod <pod>

State: Terminated

Reason: OOMKilled

Exit Code: 137

FIX: increase memory limits, fix memory leak, tune JVM/runtime



2. Evicted (pod-level, node pressure):

CAUSE: Node is running low on resources (memory, disk, PIDs)

WHO: kubelet eviction manager

SCOPE: entire pod removed from node

RESTART: pod is NOT restarted on same node

  Controller (Deployment/RS) creates NEW pod, scheduler places elsewhere

DETECT:

kubectl describe pod <pod>

Status: Failed

Reason: Evicted

Message: "The node was low on resource: memory"

ORDER: BestEffort → Burstable (by overshoot) → Guaranteed

FIX: right-size resource requests, use Guaranteed QoS for critical pods



3. Preempted (pod-level, scheduling priority):

CAUSE: Higher-priority pod needs resources, no room on any node

WHO: kube-scheduler

SCOPE: lower-priority pods evicted to make room

RESTART: preempted pod goes back to Pending, may schedule elsewhere

DETECT:

kubectl get events --field-selector reason=Preempted

FIX: assign appropriate PriorityClass to critical workloads



4. Deleted (pod-level, intentional):

CAUSE: kubectl delete, scale down, rolling update, node drain

WHO: API server / controllers

SCOPE: specific pod

RESTART: depends on controller

Normal lifecycle — not an incident



THE COMPARISON TABLE:

┌──────────────┬─────────────┬──────────┬─────────────────┐

│ Type         │ Who decides  │ Why      │ Pod restarted?  │

├──────────────┼─────────────┼──────────┼─────────────────┤

│ OOMKilled    │ Kernel      │ Mem limit│ Container yes   │

│ Evicted      │ kubelet     │ Node     │ New pod, new    │

│              │             │ pressure │ node            │

│ Preempted    │ Scheduler   │ Priority │ Back to Pending │

│ Deleted      │ User/       │ Intent   │ If controller   │

│              │ Controller  │          │ exists          │

└──────────────┴─────────────┴──────────┴─────────────────┘

```

---

### CronJob Production Safeguards

```yaml

apiVersion: batch/v1

kind: CronJob

metadata:

 name: db-backup

 namespace: production

spec:

 schedule: "0 \*/6 \* \* \*"              # every 6 hours

 
 concurrencyPolicy: Forbid

 # Forbid: skip if previous job still running

 # Allow: start new job even if previous running (DEFAULT — dangerous)

 # Replace: kill previous job, start new one

 startingDeadlineSeconds: 300

 # If CronJob controller was down and missed the schedule,

 # only start the job if it's within 300s of scheduled time

 # Without this: controller comes back up, fires ALL missed jobs at once

 # 100 missed jobs × db-backup = hammers the database

 successfulJobsHistoryLimit: 3        # keep 3 successful job pods
 failedJobsHistoryLimit: 5            # keep 5 failed job pods for debugging

 # Without limits: thousands of completed pods accumulate

 # Fills up etcd, clutters kubectl get pods output

 jobTemplate:
   spec:
     activeDeadlineSeconds: 3600      # kill job after 1 hour

     # Prevents hung jobs from running forever

     # DB backup usually takes 20 min → 1 hour is safe deadline

     # Without this: deadlocked job runs forever, blocks next schedule

     backoffLimit: 3                  # retry failed pods 3 times

     # Default is 6 — too many for something that runs every 6 hours

     ttlSecondsAfterFinished: 86400   # auto-delete job after 24 hours

     # Cleaner than relying on history limits alone
     template:
       spec:
         restartPolicy: OnFailure     # REQUIRED for Jobs (not Always)
         containers:
         - name: backup
           image: backup-tool:v1.0
           resources:
             requests:
               cpu: 500m
               memory: 1Gi
             limits:
               memory: 2Gi            # prevent OOM from backup spike

```

---

### Secret Rotation — The Reality

```bash

KUBERNETES SECRETS ARE NOT AUTOMATICALLY ROTATED

If you put a DB password in a Secret, it stays there forever

Until someone manually updates it



THE ROTATION CHAIN:



1. External secret store rotates the secret:

   AWS Secrets Manager → automatic rotation via Lambda

   HashiCorp Vault → dynamic secrets with TTL



2. External Secrets Operator syncs to K8s:

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
 name: db-creds
spec:
 refreshInterval: 1h              # poll every hour
 secretStoreRef:
   name: aws-secrets-manager
   kind: ClusterSecretStore
 target:
   name: db-credentials           # K8s Secret to create/update
 data:
 - secretKey: password
   remoteRef:
     key: production/db/password  # path in AWS Secrets Manager



3. Pod picks up new secret:

IF mounted as volume: kubelet updates file within \~60s

IF mounted as env var: pod must be RESTARTED



For env vars: use Reloader or rolling restart after rotation

For volumes: app must re-read the file (not all apps do this)



THE GOTCHA NOBODY MENTIONS:

Rotating a secret means there's a window where:

- New pods get the NEW secret

- Old pods still have the OLD secret

- If the old secret is revoked → old pods break

\#

Solution: dual-write

1. Add new credential to the database (both old and new work)

2. Update the secret in K8s

3. Wait for all pods to pick up new secret

4. Remove old credential from database

This is why Vault dynamic secrets are preferred — they handle this

```

---

### Resource Request Tuning — The Methodology

```bash

THE PROBLEM:

Developers guess resource requests

cpu: 500m, memory: 512Mi ← where did these numbers come from?

Usually: copied from another service, or "seemed reasonable"

Result: 40-60% of cluster resources WASTED (reserved but unused)



THE METHODOLOGY:

Step 1: Deploy with VPA in recommendation mode

apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
 name: api-vpa
spec:
 targetRef:
   apiVersion: apps/v1
   kind: Deployment
   name: api
 updatePolicy:
   updateMode: "Off"          # just recommend, don't touch pods



Step 2: Wait 1-2 weeks (capture normal + peak traffic patterns)



Step 3: Read recommendations

kubectl describe vpa api-vpa

Target:       cpu: 125m,  memory: 256Mi    ← what the app actually uses

Upper Bound:  cpu: 500m,  memory: 1Gi      ← peak usage estimate



Step 4: Set requests and limits:

resources:
 requests:
   cpu: 150m          # slightly above VPA target (buffer)
   memory: 300Mi      # slightly above VPA target
 limits:
   cpu: 600m          # 4x requests (burst headroom)
   memory: 512Mi      # \~1.5-2x requests (memory spikes)



Step 5: Monitor and iterate

Prometheus queries:

Actual CPU usage vs requests:

container\_cpu\_usage\_seconds\_total / kube\_pod\_container\_resource\_requests{resource="cpu"}

If consistently < 0.5 → over-provisioned, reduce requests

If consistently > 0.9 → under-provisioned, increase requests



Actual memory vs requests:

container\_memory\_working\_set\_bytes / kube\_pod\_container\_resource\_requests{resource="memory"}



TOOLS:

Kubecost: shows cost per namespace/deployment, identifies waste

Goldilocks: runs VPA in recommend mode for all deployments,

  provides dashboard with suggested requests/limits

kubectl-resource-recommender: CLI tool for recommendations



THE RULE:

Requests = what you actually need (based on data)

Limits.memory = 1.5-2x requests (protect against leaks)

Limits.cpu = 2-4x requests (allow bursts) or remove entirely

Review quarterly as traffic patterns change

```

---

### etcd as a Failure Mode

```bash

SYMPTOMS OF ETCD PROBLEMS:

- kubectl commands hang or timeout

- Pod scheduling delays (scheduler can't read/write etcd)

- API server returning 500s or "etcdserver: request timed out"

- Existing pods keep running (they don't need etcd)

- But no NEW operations succeed



ROOT CAUSES:



1. Disk I/O latency:

etcd uses fsync for WAL writes — MUST be < 10ms

If disk latency spikes (noisy neighbor, EBS throttling):

etcd: "W | wal: sync duration exceeds 1s"

FIX: use gp3 with provisioned IOPS, or io2

EKS managed: AWS handles this (but you can hit limits at scale)



2. Database size / fragmentation:

etcd has a default 2GB database size limit

Lots of Events, Endpoints, ConfigMaps → DB grows

Compaction removes old revisions but doesn't free disk space

Defragmentation actually frees space but locks the member briefly

\#

Self-managed etcd:

etcdctl endpoint status --cluster

Shows: DB size, Raft index, leader

etcdctl compact $(etcdctl endpoint status -w json | jq -r '.\[] | .Status.header.revision')

etcdctl defrag --cluster

EKS: AWS handles compaction/defrag automatically



3. Too many objects:

200 services × 50 endpoints each = 10,000 Endpoint objects

Every change → etcd write → watch notification → all kube-proxies

Fix: EndpointSlices (default now), reduce object churn

Reduce CronJob history limits, Event TTL



4. Slow watchers:

Controllers/operators that watch too broadly

A badly-written operator watching ALL pods across ALL namespaces

Generates massive watch traffic through API server to etcd

Fix: use label selectors, namespace scopes in watches



MONITORING:

etcd\_disk\_wal\_fsync\_duration\_seconds → must be < 10ms p99

etcd\_server\_slow\_apply\_total → increasing = etcd can't keep up

etcd\_mvcc\_db\_total\_size\_in\_bytes → approaching 2GB = danger

apiserver\_storage\_objects → total objects in etcd

apiserver\_request\_duration\_seconds → API server latency (etcd is usually the cause)



EKS:

etcd is managed — you can't SSH to it

But you CAN see symptoms: API server latency, scheduling delays

If you suspect etcd: open AWS support ticket

Monitor via: CloudWatch metrics for EKS control plane

```

---

### Runbook Structure — How to Write One

```markdown

RUNBOOK: Service X Returns 502 Errors

\## Severity: SEV2

\## Owner: Platform Engineering

\## Last Updated: 2024-01-15

\## Last Tested: 2024-01-01

\## Symptoms

\- Grafana dashboard "Service X" shows 502 error rate > 1%

\- PagerDuty alert: "service-x-502-rate-high"

\- Customer reports: checkout failures

\## Impact

\- Checkout flow degraded

\- Revenue impact: \~$5K/min during peak hours

\## Quick Diagnosis (< 2 minutes)

1\. Check pod health:

  ```

  kubectl get pods -n production -l app=service-x

  ```

  - All pods Running and Ready (1/1)?

  - Any CrashLoopBackOff or Pending?

2\. Check endpoints exist:

  ```

  kubectl get endpoints service-x -n production

  ```

  - If empty → readiness probe failing

3\. Check recent deployments:

  ```

  kubectl rollout history deployment/service-x -n production

  ```

  - Was there a deploy in the last 30 min?

\## Resolution Steps

\### If pods are crashing:

1\. Check logs: `kubectl logs -l app=service-x --previous`

2\. If OOMKilled: increase memory limits (runbook-link)

3\. If config error: rollback: `kubectl rollout undo deployment/service-x`



\### If pods are healthy but 502 persists:

1\. Check upstream dependency:

  ```

  kubectl exec <pod> -- curl -v http://payment-service:8080/health

  ```

2\. If payment-service down → escalate to Payment team

3\. Check Istio circuit breaker: 

  ```

  istioctl proxy-config cluster <pod> | grep payment

  ```



\### If recent deployment:

1\. Rollback immediately: `kubectl rollout undo deployment/service-x`

2\. Verify recovery

3\. Investigate the failed deployment after service is stable



\## Escalation

\- If not resolved in 15 min → page on-call SRE lead

\- If revenue impact > $50K → escalate to SEV1, page VP Engineering



\## Post-Incident

\- Write postmortem within 48 hours

\- Link to template: \[postmortem-template]

```

```



📝 Retention Questions — Lesson 4

Q1: A pod is Running but the Service has no endpoints. Give the full list of realistic causes and the exact commands you'd run to isolate which one it is.

Q2: Your API pods are not restarting, not OOMKilled, and not throwing obvious errors — but p99 latency doubled after a deploy. CPU usage looks "low." What silent Kubernetes-level bottleneck should you suspect first, how do you verify it, and what are the likely fixes?

Q3: In a private EKS cluster, new pods are stuck in ImagePullBackOff. The ECR repository exists, IAM looks correct, and DNS resolution works. Explain the hidden dependency that often breaks image pulls and how to confirm it.

Q4: A CronJob scheduled every 5 minutes accidentally starts overlapping runs and overwhelms the database. What fields should have been set to prevent this, and what other production safeguards should be standard for CronJobs?

Q5: Your cluster suddenly shows widespread failures creating new pods, but existing pods are mostly fine. kubectl itself feels slow. What class of component should you suspect, what are the likely culprits, and why can one bad component create cluster-wide symptoms?

Go. 🎯

##### Q1: The Missing Endpoints Mystery

When a pod is Running but the Service has no endpoints, the "link" between the Service's selector and the Pod's labels is broken, or the Pod is not considered "Ready."

Realistic Causes \& Isolation Commands:

Label Selector Mismatch: The most common cause. The Service is looking for app: api-v1, but the pod is labeled app: api-v2.

Command: kubectl get pods --show-labels (Check pod labels) and kubectl describe svc <svc-name> (Check the Selector field).

Readiness Probe Failure: A pod can be Running (the process started) but not Ready (the probe failed). K8s will not add a pod to the Service endpoints until the readiness probe passes.

Command: kubectl get pods (Check if it says 1/1 or 0/1 in the READY column) and kubectl describe pod <pod-name> (Check the Events section for Readiness probe failed).

Namespace Mismatch: The Service and Pod are in different namespaces. Services only target pods within their own namespace unless specifically configured otherwise (rare).

Command: kubectl get pods -n <namespace> and kubectl get svc -n <namespace>.

Port Mismatch: The targetPort in the Service definition does not match the containerPort the application is actually listening on.

Command: kubectl describe svc <svc-name> (Check targetPort) and kubectl get pod <pod-name> -o yaml (Check containerPort).

##### Q2: The Silent Killer: CPU Throttling

The Bottleneck: CFS (Completely Fair Scheduler) Quota Throttling.

Reasoning:

You see "low" average CPU usage because you are likely looking at a 1-minute or 5-minute average. However, CPU limits are enforced in 100ms windows. If your application is multi-threaded and bursts, it can consume its entire 100ms quota in the first 20ms of the window. The kernel then throttles (freezes) the process for the remaining 80ms.

This creates a "stutter" effect. The average CPU looks fine (30%), but the p99 latency spikes because requests arriving during that 80ms freeze are queued, adding massive latency.

How to Verify:

Check the cgroup stats on the worker node:

Command: cat /sys/fs/cgroup/cpu/cpu.stat (Cgroup v1) or cat /sys/fs/cgroup/cpu.stat (Cgroup v2).

What to look for: nr\_throttled and throttled\_time. If these numbers are increasing rapidly during the latency spikes, you are being throttled.

Likely Fixes:

Increase CPU Limits: Give the pod more headroom to handle bursts.

Remove CPU Limits: In high-performance environments, many SREs remove limits.cpu entirely and rely only on requests.cpu to avoid the CFS "cliff," using HPA to manage node saturation.

##### Q3: The Hidden ECR Dependency (S3 Gateway)

The Hidden Dependency: S3 Gateway Endpoint.

Explanation:

ECR is not a single service; it is a split architecture. The ECR API (used for auth and manifests) is accessed via an Interface Endpoint (PrivateLink). However, the actual image layers (the blobs) are stored in Amazon S3.

In a private EKS cluster, if you have an ECR Interface Endpoint but no S3 Gateway Endpoint, the following happens:

The node calls ECR 

→

→

"Here is the image manifest." (Succeeds).

The node tries to pull the layer from S3 

→

→

"I don't have a route to S3." (Fails).

The result is ImagePullBackOff.

How to Confirm:

Route Table Check: Check the VPC Route Table for the private subnet. Look for a route to pl-xxxxxxxx (the S3 Prefix List). If it's missing, there's no S3 endpoint.

Network Reachability Analyzer: Use the AWS VPC Reachability Analyzer to test a path from the Node's ENI to the S3 service.

##### Q4: CronJob Overlap and Safeguards

Field to prevent overlap: concurrencyPolicy: Forbid.

By default, concurrencyPolicy is Allow, meaning if Job A takes 6 minutes to run but is scheduled every 5 minutes, Job B will start while Job A is still running. Setting this to Forbid tells Kubernetes: "If the previous job is still running, skip the next scheduled run."

Standard Production Safeguards for CronJobs:

startingDeadlineSeconds: Prevents a "thundering herd" of jobs from starting if the controller was down for an hour. If the job misses its window by more than 

𝑋

X

seconds, it's skipped.

activeDeadlineSeconds: A hard timeout. If a job hangs (e.g., DB deadlock), this kills the pod after 

𝑋

X

seconds so it doesn't consume resources forever.

successfulJobsHistoryLimit \& failedJobsHistoryLimit: Prevents the cluster from being cluttered with thousands of completed Pod objects.

##### Q5: The Control Plane Heart Attack (etcd)

Component to suspect: etcd.

The Culprits:

Disk I/O Latency: etcd is extremely sensitive to disk write latency (fdatasync). If the underlying disk is slow or saturated, etcd cannot commit entries fast enough.

etcd Database Fragmentation: Over time, etcd's keyspace becomes fragmented, slowing down reads/writes.

Resource Exhaustion: Memory pressure on the etcd nodes leading to swapping.

Why one component creates cluster-wide symptoms:

etcd is the single source of truth for the entire cluster. Every time you run kubectl, create a pod, or update a service, the API server must write to etcd.

If etcd is slow, kubectl (which talks to the API server) feels sluggish.

If etcd cannot commit the "Pod Created" event, the Scheduler and Kubelet never receive the instruction to start the pod, leading to failures in creating new pods.

Existing pods are fine because they are already running and don't need to talk to etcd for their daily operations (unless they are updating status).

##### Q6: Istio Canary and Circuit Breaking
VirtualService (Traffic Split):
apiVersion, networking.istio.io/v1alpha3
kind: VirtualService
metadata:
 name: payment-service
spec:
 hosts:
 - payment.novamart.com
 http:
 - route:
   - destination:
       host: payment-service
       subset: v1
     weight: 95
   - destination:
       host: payment-service
       subset: v2
     weight: 5



DestinationRule (Subsets):

apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
 name: payment-service
spec:
 host: payment-service
 subsets:
 - name: v1
   labels:
     version: v1
 - name: v2
   labels:
     version: v2



Circuit Breaker Protection:

If v2's error rate spikes, Istio's Outlier Detection (the circuit breaker) kicks in. You define a outlierDetection block in the DestinationRule. If a pod returns a 5xx error 

𝑋

X

times in a row, Istio ejects that specific pod from the load balancing pool for a predetermined period.



This protects the system by ensuring that the 5% of traffic sent to v2 doesn't just fail—it fails fast, and the faulty pods are removed entirely until they recover, preventing the "error storm" from affecting the overall system stability.



##### Q7: The GitOps "Reconciliation Loop"

What happened:

The cluster is managed by a GitOps Controller (like ArgoCD or Flux).

In a GitOps model, the Git repository is the Desired State, and the cluster is the Actual State.

The developer changed the Actual State (Pod count = 10) via kubectl edit.

The ArgoCD controller performed a "drift detection" scan and noticed: Git (5) != Cluster (10).

The controller automatically performed a Sync/Reconciliation, overwriting the manual change to bring the cluster back into alignment with the source of truth (Git).

Why this is correct:

This prevents "Configuration Drift." If you allow manual changes, you lose the ability to audit who changed what and you cannot reliably recreate the environment from Git in case of a disaster.

##### Q8: The Jenkins 

→

→

ArgoCD GitOps Flow

The Complete Flow:

Code Push: Developer pushes code to Git.

Build (Jenkins): Jenkins triggers a build, runs tests, and builds a Docker image.

Push Image: Jenkins pushes the image to ECR with a unique tag (e.g., the Git SHA).

Manifest Update: Instead of touching the cluster, Jenkins clones the GitOps Manifest Repo and updates the image tag in the YAML (e.g., image: payment:v1 

→

→

image: payment:sha-abc123).

Commit \& Push: Jenkins commits this change back to the Manifest Repo.

Sync (ArgoCD): ArgoCD detects the commit in the Manifest Repo and applies the change to the EKS cluster.

Why Jenkins should NOT kubectl apply:

Security: Jenkins would need cluster-admin permissions, which is a huge security risk. If Jenkins is compromised, the whole cluster is gone.

Audit Trail: kubectl apply is a "fire and forget" action. GitOps provides a permanent history of exactly when and why a version changed.

Drift Detection: kubectl apply cannot detect if someone manually changed a setting. ArgoCD can.

Safe Rollbacks: Rollbacks in GitOps are just git revert on the manifest repo, which is safer and faster than trying to remember which image tag was stable.

##### Q9: 3 AM Node Failure Triage

Step 1: Blast Radius Assessment (The "Panic" Phase)

I immediately check if the 3 critical API pods are still healthy via the Load Balancer. If the pods were spread across nodes, the service might still be up. If they were all on ip-10-0-1-50, I have a P1 outage.

Step 2: Node Health Diagnostics

I attempt to determine if the node is "dead" or "unreachable":

AWS Console: Check if the instance is Running or if it has failed a Status Check (System/Instance).

SSH Attempt: Try to SSH in. If it times out, the network/kernel is dead.

Dmesg/Logs: If I can get in, I check dmesg for "Kernel Panic" or "Out of Memory" (OOM) kills of the kubelet.

Step 3: Recovery (Forcing Reschedule)

Since the node is NotReady, Kubernetes will eventually evict the pods, but the default timeout is 5 minutes (too long for critical APIs).

Taint the Node: If the node is unresponsive, I manually taint it to ensure no new pods land on it:

kubectl taint nodes ip-10-0-1-50 key=value:NoExecute

This forces the 15 pods to evacuate immediately and reschedule on healthy nodes.

Verify Pod Recovery: I monitor kubectl get pods -w to ensure the 3 critical API pods are Running and Ready on other nodes.

Step 4: Root Cause Analysis (Post-Recovery)

Once the service is restored, I investigate the failed node:

Check CloudWatch metrics for CPU/Memory spikes.

Analyze the kubelet logs from the node's disk.

Check if there was a "noisy neighbor" pod that crashed the kernel.



================================================================================================
# Phase 3: CI/CD Pipelines
## Lesson 1: Jenkins Architecture & Pipeline Fundamentals

---

## Why This Matters at NovaMart

You have 200+ microservices. Each one needs to build, test, scan, package, and deploy — reliably, securely, and fast. Jenkins is your CI engine. ArgoCD handles CD (you've seen the basics). If Jenkins is slow, flaky, or insecure, **every team in the company is blocked**. A misconfigured Jenkins is one of the most common vectors for supply chain attacks. A poorly architected Jenkins is a bottleneck that makes 50 engineers hate you personally.

Let's make sure that doesn't happen.

---

## 1. Jenkins Architecture — How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                     JENKINS CONTROLLER                          │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────────────┐  │
│  │ Web UI   │  │ REST API │  │ Job Queue  │  │ Config Store │  │
│  │ (Stapler)│  │          │  │ (Build Q)  │  │ ($JENKINS_   │  │
│  └──────────┘  └──────────┘  └─────┬─────┘  │  HOME/jobs/) │  │
│                                    │         └──────────────┘  │
│  ┌──────────────┐  ┌──────────┐    │    ┌───────────────────┐  │
│  │ Credentials  │  │ Plugin   │    │    │ Executor Pool     │  │
│  │ Store        │  │ Manager  │    │    │ (0 on controller  │  │
│  │ (encrypted)  │  │ (200+)   │    │    │  in production!)  │  │
│  └──────────────┘  └──────────┘    │    └───────────────────┘  │
│                                    │                            │
└────────────────────────────────────┼────────────────────────────┘
                                     │ JNLP (WebSocket/TCP)
                                     │ or SSH
                    ┌────────────────┼────────────────┐
                    │                │                 │
            ┌───────▼──────┐ ┌──────▼───────┐ ┌──────▼───────┐
            │  Agent (Node) │ │  Agent (Node) │ │  Agent (Node) │
            │  Label: java  │ │  Label: docker│ │  Label: gpu   │
            │               │ │               │ │               │
            │ Executors: 2  │ │ Executors: 4  │ │ Executors: 1  │
            │ Workspace:    │ │ Workspace:    │ │ Workspace:    │
            │ /var/jenkins  │ │ /var/jenkins  │ │ /var/jenkins  │
            └───────────────┘ └───────────────┘ └───────────────┘
```

### The Controller (formerly "Master")

The controller is the **brain** — it does NOT build things in production. It:

- **Schedules builds** — matches jobs to agents based on labels
- **Stores configuration** — all job definitions live under `$JENKINS_HOME/jobs/`
- **Manages plugins** — the plugin ecosystem is Jenkins's greatest strength and greatest liability
- **Serves the UI and API** — Stapler framework (Java, old, memory-hungry)
- **Holds credentials** — encrypted with a master key stored on disk
- **Manages the build queue** — FIFO by default, with priority plugins available

**Critical production setting:**
```groovy
// Set controller executors to ZERO
// Jenkins → Manage → Nodes → Built-In Node → # of executors = 0
// WHY: Builds on controller = security risk + stability risk
// A rogue build can OOM/crash the controller, killing ALL pipelines
```

`$JENKINS_HOME` structure:
```
$JENKINS_HOME/
├── config.xml                 # Global config (security, cloud config)
├── credentials.xml            # Encrypted credentials
├── secrets/                   # Master encryption key, agent secrets
│   ├── master.key             # THIS FILE = keys to the kingdom
│   ├── hudson.util.Secret     # Encryption key for credentials
│   └── initialAdminPassword   # First-time setup
├── jobs/
│   └── my-pipeline/
│       ├── config.xml         # Job definition
│       └── builds/
│           ├── 1/             # Build #1
│           │   ├── log        # Console output
│           │   ├── build.xml  # Build metadata
│           │   └── changelog.xml
│           └── lastSuccessfulBuild → 1  # Symlink
├── nodes/                     # Agent definitions
├── plugins/                   # Installed plugins (.jpi/.hpi)
├── users/                     # User configs
├── workspace/                 # Controller workspaces (should be EMPTY)
└── war/                       # Exploded WAR file
```

### Agents (formerly "Slaves")

Agents are **disposable compute** that run actual builds. Connection methods:

| Method | How | Use Case | Pros | Cons |
|--------|-----|----------|------|------|
| **JNLP/WebSocket** | Agent connects TO controller | Cloud agents, NAT'd agents | Outbound only, firewall-friendly | Agent needs controller URL |
| **SSH** | Controller connects TO agent | Static agents, VMs | Standard, well-understood | Controller needs SSH access |
| **Kubernetes Plugin** | Dynamic pod per build | EKS/GKE/AKS | Elastic, isolated, clean | Pod startup latency (~15-30s) |
| **EC2 Plugin** | Dynamic EC2 per build | AWS heavy shops | Full VM isolation | Longer startup (~2-5min) |
| **Docker Plugin** | Container per build | Single-host dev | Quick, lightweight | Single host limitation |

### Jenkins on Kubernetes (NovaMart Pattern)

```
┌─────────────────── EKS Cluster ───────────────────────┐
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │           jenkins namespace                       │  │
│  │                                                   │  │
│  │  ┌─────────────────────┐   ┌──────────────────┐  │  │
│  │  │  Jenkins Controller │   │  EBS Volume (gp3) │  │  │
│  │  │  (StatefulSet,      │───│  $JENKINS_HOME    │  │  │
│  │  │   1 replica)        │   │  100Gi            │  │  │
│  │  │  CPU: 2, Mem: 4Gi   │   └──────────────────┘  │  │
│  │  └────────┬────────────┘                          │  │
│  │           │ K8s API (creates pods)                │  │
│  └───────────┼──────────────────────────────────────┘  │
│              │                                         │
│  ┌───────────▼──────────────────────────────────────┐  │
│  │       jenkins-agents namespace                    │  │
│  │                                                   │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐         │  │
│  │  │ Build Pod│ │ Build Pod│ │ Build Pod│  (dynamic│  │
│  │  │ maven +  │ │ go +     │ │ node +   │  on-     │  │
│  │  │ docker   │ │ trivy    │ │ chrome   │  demand) │  │
│  │  └──────────┘ └──────────┘ └──────────┘         │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

**Kubernetes plugin pod template:**
```groovy
// This defines WHAT the build agent pod looks like
podTemplate(
    label: 'java-builder',
    serviceAccount: 'jenkins-agent',     // IRSA for AWS access
    namespace: 'jenkins-agents',
    containers: [
        containerTemplate(
            name: 'maven',
            image: 'maven:3.9.6-eclipse-temurin-17',
            command: 'cat',              // Keep alive, don't run maven immediately
            ttyEnabled: true,
            resourceRequestCpu: '1',
            resourceRequestMemory: '2Gi',
            resourceLimitCpu: '2',
            resourceLimitMemory: '4Gi'
        ),
        containerTemplate(
            name: 'docker',
            image: 'docker:24-dind',     // Docker-in-Docker
            privileged: true,            // REQUIRED for DinD (security concern)
            resourceRequestCpu: '500m',
            resourceRequestMemory: '1Gi'
        ),
        containerTemplate(
            name: 'trivy',
            image: 'aquasec/trivy:0.50.0',
            command: 'cat',
            ttyEnabled: true
        )
    ],
    volumes: [
        // Maven cache — speeds up builds dramatically
        persistentVolumeClaim(
            mountPath: '/root/.m2/repository',
            claimName: 'maven-cache',
            readOnly: false
        ),
        // DinD needs this
        emptyDirVolume(mountPath: '/var/lib/docker', memory: false)
    ]
) {
    node('java-builder') {
        stage('Build') {
            container('maven') {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Scan') {
            container('trivy') {
                sh 'trivy image --severity HIGH,CRITICAL myimage:latest'
            }
        }
    }
}
```

### How It Breaks: Controller Failures

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **OOM crash** | Controller restarts, builds lost mid-flight | Too many plugins, builds on controller, big pipelines in memory | Set executors=0, increase JVM heap `-Xmx4g`, audit plugins |
| **Disk full** | UI unresponsive, builds fail to write logs | Build logs + artifacts + workspace accumulation | Discard old builds policy, `logRotator(daysToKeep:30, numToKeep:50)`, workspace cleanup |
| **Plugin hell** | Startup failures, ClassNotFound, UI 500s | Plugin version incompatibility, untested upgrade | Plugin compatibility checker, staging Jenkins, pin plugin versions |
| **Credential leak** | Secrets in console output, logs shipped to ELK | Pipeline `echo`s secrets, bad masking | `MaskPasswordsBuildWrapper`, never use `sh "echo $SECRET"`, credential binding plugin |
| **Split brain** | Two controllers think they're active | Someone ran two instances on same `$JENKINS_HOME` | NEVER share `$JENKINS_HOME`, use HA plugin properly |
| **Slow UI** | Pages take 30+ seconds | Too many jobs (>1000 in one view), too many builds retained | Folders, views, aggressive build discard, Performance plugin |
| **master.key lost** | All credentials unrecoverable | EBS snapshot missed `secrets/` dir, migration error | Backup `secrets/` dir separately, test restore regularly |
| **Agent disconnects** | Builds stuck "waiting for agent" | Network issues, JNLP port blocked, pod evicted | WebSocket (no extra port), agent health monitoring, pod priority class |

### How It Breaks: Agent/Build Failures

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **Workspace collision** | Flaky builds, wrong files | Concurrent builds sharing workspace | `ws()` step for custom workspace, or K8s plugin (unique pod per build) |
| **Docker socket sharing** | Container escape, cross-build contamination | DooD (Docker outside of Docker) | Kaniko for image builds, or DinD with `--userns-remap` |
| **DNS resolution fails** | `mvn` can't pull deps, npm install fails | K8s DNS issue, CoreDNS overwhelmed | NodeLocal DNSCache, check pod DNS config |
| **Pod pending** | Build queued forever | No capacity, resource requests too high, node taint | Karpenter, right-size agent pods, spot instances |
| **Maven/Gradle OOM** | Build killed mid-compilation | Default JVM heap too small in container | `MAVEN_OPTS=-Xmx1g`, match to container memory limit |
| **Stale cache** | Build uses old dependency version | PVC cache has stale artifacts | Cache TTL, periodic cache wipe CronJob |

---

## 2. Jenkinsfile — Declarative vs Scripted

### Declarative Pipeline (Use This)

```groovy
// Jenkinsfile (Declarative)
pipeline {
    agent none  // Don't allocate globally — each stage picks its own

    options {
        timeout(time: 30, unit: 'MINUTES')      // Kill runaway builds
        timestamps()                              // Timestamp every log line
        disableConcurrentBuilds()                 // One build at a time (per branch)
        buildDiscarder(logRotator(                // Disk management
            numToKeepStr: '20',
            daysToKeepStr: '30',
            artifactNumToKeepStr: '5'
        ))
        retry(0)                                  // Don't retry the whole pipeline
    }

    environment {
        // Available to all stages
        REGISTRY = '123456789.dkr.ecr.us-east-1.amazonaws.com'
        APP_NAME = 'order-service'
        // credentials() helper — binds from Jenkins credential store
        SONAR_TOKEN = credentials('sonarqube-token')     // Secret text
        DOCKER_CREDS = credentials('ecr-creds')          // Username/password
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target env')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip test stage')
        string(name: 'IMAGE_TAG', defaultValue: '', description: 'Override image tag')
    }

    stages {
        stage('Checkout') {
            agent { label 'lightweight' }
            steps {
                checkout scm
                script {
                    // Determine image tag
                    env.GIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG = params.IMAGE_TAG ?: "${env.BRANCH_NAME}-${env.GIT_SHORT}-${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Build & Unit Test') {
            agent {
                kubernetes {
                    yaml '''
                    apiVersion: v1
                    kind: Pod
                    spec:
                      containers:
                      - name: maven
                        image: maven:3.9.6-eclipse-temurin-17
                        command: ["cat"]
                        tty: true
                        resources:
                          requests:
                            cpu: "1"
                            memory: "2Gi"
                          limits:
                            cpu: "2"
                            memory: "4Gi"
                        volumeMounts:
                        - name: maven-cache
                          mountPath: /root/.m2/repository
                      volumes:
                      - name: maven-cache
                        persistentVolumeClaim:
                          claimName: maven-cache-pvc
                    '''
                }
            }
            steps {
                container('maven') {
                    sh '''
                        mvn clean verify \
                            -Dmaven.test.failure.ignore=false \
                            -T 1C \
                            -B \
                            --no-transfer-progress
                    '''
                    // -T 1C = 1 thread per CPU core (parallel build)
                    // -B = batch mode (no interactive)
                    // --no-transfer-progress = clean logs
                }
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'    // Test results
                    jacoco(execPattern: '**/target/jacoco.exec') // Coverage
                }
            }
        }

        stage('SonarQube Analysis') {
            agent { label 'lightweight' }
            steps {
                withSonarQubeEnv('sonarqube-server') {  // Configured in Jenkins global
                    sh '''
                        mvn sonar:sonar \
                            -Dsonar.projectKey=${APP_NAME} \
                            -Dsonar.branch.name=${BRANCH_NAME}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                    // Blocks until SonarQube webhook fires back
                    // REQUIRES: SonarQube → Administration → Webhooks → Jenkins URL
                }
            }
        }

        stage('Build & Push Image') {
            agent {
                kubernetes {
                    yaml '''
                    apiVersion: v1
                    kind: Pod
                    spec:
                      containers:
                      - name: kaniko
                        image: gcr.io/kaniko-project/executor:debug
                        command: ["cat"]
                        tty: true
                        volumeMounts:
                        - name: docker-config
                          mountPath: /kaniko/.docker
                      volumes:
                      - name: docker-config
                        secret:
                          secretName: ecr-docker-config
                    '''
                }
            }
            steps {
                container('kaniko') {
                    sh """
                        /kaniko/executor \
                            --context=dir://\${WORKSPACE} \
                            --dockerfile=Dockerfile \
                            --destination=${REGISTRY}/${APP_NAME}:${IMAGE_TAG} \
                            --destination=${REGISTRY}/${APP_NAME}:latest-${BRANCH_NAME} \
                            --cache=true \
                            --cache-repo=${REGISTRY}/${APP_NAME}/cache \
                            --snapshot-mode=redo \
                            --compressed-caching=false
                    """
                    // --cache=true: Layer caching in registry (HUGE speed boost)
                    // No Docker daemon needed. No privileged mode.
                }
            }
        }

        stage('Security Scan') {
            parallel {    // Run scans in parallel — saves 3-5 minutes
                stage('Trivy') {
                    agent { label 'scanner' }
                    steps {
                        sh """
                            trivy image \
                                --severity HIGH,CRITICAL \
                                --exit-code 1 \
                                --ignore-unfixed \
                                --format json \
                                --output trivy-report.json \
                                ${REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                        """
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'trivy-report.json'
                        }
                    }
                }
                stage('BlackDuck') {
                    agent { label 'scanner' }
                    steps {
                        sh """
                            bash <(curl -s https://detect.synopsys.com/detect.sh) \
                                --blackduck.url=\${BLACKDUCK_URL} \
                                --blackduck.api.token=\${BLACKDUCK_TOKEN} \
                                --detect.project.name=${APP_NAME} \
                                --detect.policy.check.fail.on.severities=CRITICAL
                        """
                        // BlackDuck: license compliance + vulnerability (SCA)
                        // Trivy: container image scanning (CVE)
                        // Different tools, different purposes, both required
                    }
                }
            }
        }

        stage('Update Manifests') {
            when {
                branch 'main'    // Only update manifests for main branch
            }
            agent { label 'lightweight' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'bitbucket-pat',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PAT'
                )]) {
                    sh """
                        git clone https://\${GIT_USER}:\${GIT_PAT}@bitbucket.org/novamart/k8s-manifests.git
                        cd k8s-manifests/apps/${APP_NAME}/overlays/dev
                        kustomize edit set image ${REGISTRY}/${APP_NAME}=${REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                        git add .
                        git commit -m "chore: update ${APP_NAME} to ${IMAGE_TAG}"
                        git push origin main
                    """
                    // ArgoCD watches this repo → detects diff → syncs to cluster
                    // This is the GitOps bridge: CI (Jenkins) → CD (ArgoCD)
                }
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ ${APP_NAME} ${IMAGE_TAG} pipeline succeeded"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ ${APP_NAME} pipeline FAILED: ${BUILD_URL}"
            )
        }
        always {
            // Clean up workspace on the agent
            cleanWs()
        }
    }
}
```

### Declarative vs Scripted — The Real Comparison

```groovy
// SCRIPTED (old style — full Groovy power, no guardrails)
node('java-builder') {
    try {
        stage('Build') {
            checkout scm
            sh 'mvn clean package'
        }
        stage('Test') {
            sh 'mvn test'
        }
    } catch (e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        cleanWs()
    }
}

// DECLARATIVE (modern — structured, validated before execution)
pipeline {
    agent { label 'java-builder' }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }
    post {
        always { cleanWs() }
    }
}
```

| Aspect | Declarative | Scripted |
|--------|-------------|----------|
| Syntax validation | Before execution | At runtime (fails mid-pipeline) |
| `post` blocks | Built-in (`always`, `success`, `failure`, `unstable`, `changed`) | Manual `try/catch/finally` |
| `when` conditions | Built-in | Manual `if` statements |
| `options` | Structured block | Manual calls scattered |
| Parallel | `parallel` block inside stage | `parallel` as a step |
| Flexibility | Limited — `script {}` block for escape hatch | Full Groovy |
| Recommended | ✅ Yes | Only when declarative can't express what you need |

**Rule:** Start with declarative. Use `script {}` escape hatch when needed. If you find yourself using `script {}` in every stage, refactor — don't switch to scripted.

---

## 3. Shared Libraries — The Golden Path

At NovaMart, you do NOT want 200 teams writing 200 different Jenkinsfiles. You want **one standard pipeline** with escape hatches.

```
shared-library repo structure:
├── vars/                          # Global functions (the API)
│   ├── standardPipeline.groovy    # The golden path
│   ├── dockerBuild.groovy         # Reusable: build + push image
│   ├── trivyScan.groovy           # Reusable: security scan
│   ├── notifySlack.groovy         # Reusable: notification
│   └── updateManifests.groovy     # Reusable: GitOps manifest update
├── src/                           # Classes (complex logic)
│   └── com/novamart/pipeline/
│       ├── Config.groovy
│       └── Utils.groovy
├── resources/                     # Static files (templates, configs)
│   └── pod-templates/
│       ├── java-builder.yaml
│       └── node-builder.yaml
└── test/                          # Unit tests (yes, test your pipelines)
```

**The golden path template:**
```groovy
// vars/standardPipeline.groovy
def call(Map config) {
    // Validate required config
    assert config.appName : "appName is required"
    assert config.language : "language is required (java, go, node)"

    pipeline {
        agent none
        options {
            timeout(time: config.timeout ?: 30, unit: 'MINUTES')
            timestamps()
            buildDiscarder(logRotator(numToKeepStr: '20'))
        }

        stages {
            stage('Build') {
                agent {
                    kubernetes {
                        yaml libraryResource("pod-templates/${config.language}-builder.yaml")
                    }
                }
                steps {
                    script {
                        switch(config.language) {
                            case 'java':
                                container('maven') {
                                    sh "mvn clean verify -T 1C -B --no-transfer-progress"
                                }
                                break
                            case 'go':
                                container('golang') {
                                    sh "go build -o app ./cmd/..."
                                    sh "go test -race -coverprofile=coverage.out ./..."
                                }
                                break
                            case 'node':
                                container('node') {
                                    sh "npm ci --prefer-offline"
                                    sh "npm test -- --ci --coverage"
                                }
                                break
                        }
                    }
                }
            }

            stage('SonarQube') {
                when { expression { config.sonar != false } }
                steps { sonarAnalysis(config.appName) }
            }

            stage('Build Image') {
                steps { dockerBuild(config.appName) }
            }

            stage('Security Scan') {
                parallel {
                    stage('Trivy')    { steps { trivyScan(config.appName) } }
                    stage('BlackDuck') {
                        when { expression { config.blackduck != false } }
                        steps { blackduckScan(config.appName) }
                    }
                }
            }

            stage('Update Manifests') {
                when { branch 'main' }
                steps { updateManifests(config.appName) }
            }
        }

        post {
            success { notifySlack(config.appName, 'SUCCESS') }
            failure { notifySlack(config.appName, 'FAILURE') }
        }
    }
}
```

**Consuming team's Jenkinsfile — 5 lines:**
```groovy
// Jenkinsfile in the order-service repo
@Library('novamart-pipeline-library@v2.3.0') _    // Pin version!

standardPipeline(
    appName: 'order-service',
    language: 'java',
    timeout: 45
)
```

**How shared libraries break:**

| Failure | Impact | Fix |
|---------|--------|-----|
| Library `@main` (unpinned) | Breaking change affects ALL 200 services at once | Always pin: `@Library('lib@v2.3.0')` |
| No tests | Broken template discovered by team A at 2 AM | Unit test with `JenkinsPipelineUnit`, integration test with test Jenkins |
| Over-abstraction | Teams can't debug, can't customize, hate platform team | Provide escape hatches, good docs, `config.customStages` hook |
| CPS serialization | `java.io.NotSerializableException` at random | Mark non-serializable code with `@NonCPS`, avoid storing non-serializable objects |

**CPS (Continuation Passing Style) — Jenkins's Cursed Execution Model:**
```groovy
// Jenkins pipelines can pause/resume (survive controller restart)
// This means ALL variables must be serializable
// This WILL bite you:

// BROKEN — JsonSlurper returns non-serializable LazyMap
def data = new groovy.json.JsonSlurper().parseText(response)

// FIXED — Use @NonCPS helper or readJSON step
@NonCPS
def parseJson(String text) {
    return new groovy.json.JsonSlurper().parseText(text)
}
// OR just use the pipeline step:
def data = readJSON(text: response)
```

---

## 4. Pipeline Security — This Is Where People Get Owned

### Credential Management

```groovy
// GOOD — credentials() binding, auto-masked in logs
withCredentials([
    string(credentialsId: 'api-key', variable: 'API_KEY'),
    usernamePassword(credentialsId: 'db-creds', 
                     usernameVariable: 'DB_USER', 
                     passwordVariable: 'DB_PASS'),
    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
    sshUserPrivateKey(credentialsId: 'deploy-key', 
                      keyFileVariable: 'SSH_KEY',
                      usernameVariable: 'SSH_USER')
]) {
    sh 'deploy.sh'  // Variables available as env vars in this block only
}

// BAD — secret leaks into console log
sh "echo ${API_KEY}"                    // Printed in plaintext
sh "curl -u ${DB_USER}:${DB_PASS} ..."  // If curl errors, full URL with creds in logs

// WORSE — secret baked into image layer
sh "docker build --build-arg SECRET=${API_KEY} ."  // Visible in docker history

// RIGHT WAY for Docker secrets during build
sh """
    echo '${API_KEY}' > /tmp/secret.txt
    docker build --secret id=mysecret,src=/tmp/secret.txt .
    rm /tmp/secret.txt
"""
// In Dockerfile: RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret
```

### Script Security & Sandbox

Jenkins runs Groovy in a **sandbox**. Unapproved method calls are blocked.

```
Pipeline: Script Security Plugin
├── Sandbox enabled by default for all Jenkinsfiles
├── Approved signatures in: Manage Jenkins → In-process Script Approval
├── Each new method call needs admin approval
└── Admins see queue of pending approvals
```

**The danger:**
```groovy
// If someone gets "script approval" access, they can approve:
new File('/etc/passwd').text           // Read any file on controller
"rm -rf /".execute()                    // Execute arbitrary commands
System.getenv()                         // Dump all environment variables

// MITIGATION:
// 1. Restrict Script Approval to senior admins only
// 2. Use Role-Based Access (Role Strategy plugin)
// 3. Move complex logic to shared libraries (trusted, no sandbox)
// 4. Audit approved scripts regularly
```

### Folder-Level RBAC (NovaMart Pattern)

```
Jenkins
├── Platform/                    (Platform team — full access)
│   ├── infra-pipelines/
│   └── shared-library-tests/
├── Order-Team/                  (Order team — their folder only)
│   ├── order-service/
│   └── order-worker/
├── Payment-Team/                (Payment team — their folder only)
│   ├── payment-service/
│   └── payment-gateway/
└── Security-Scans/              (Security team — read only for others)
```

Key plugins: **Folder-level credentials** (credentials scoped to folder, not global), **Role-Based Authorization Strategy** (matrix-based RBAC), **Audit Trail** (who did what).

---

## 5. Pipeline Optimization

### The Build Time Budget

```
Typical unoptimized Java pipeline:    Optimized:
─────────────────────────────         ────────────────────────
Checkout:        30s                  Checkout:       10s (shallow clone)
Dependency DL:   3min                 Dependency DL:  15s (PVC cache)
Compile:         2min                 Compile:        1min (parallel -T 1C)
Unit Tests:      5min                 Unit Tests:     2min (parallel surefire)
SonarQube:       3min                 SonarQube:      2min (incremental)
Docker Build:    4min                 Docker Build:   45s (layer cache, Kaniko)
Docker Push:     2min                 Docker Push:    20s (only changed layers)
Trivy Scan:      2min ─┐             Trivy+BD:       2min (parallel!)
BlackDuck:       3min ─┘             Update Manifest: 10s
Update Manifest: 30s                  ────────────────────────
─────────────────────────────         Total:          ~8min
Total:           ~25min
```

### Optimization Techniques

**1. Shallow Clone**
```groovy
checkout([
    $class: 'GitSCM',
    branches: [[name: env.BRANCH_NAME]],
    extensions: [
        [$class: 'CloneOption', depth: 1, shallow: true, noTags: true],
        [$class: 'SparseCheckoutPaths', sparseCheckoutPaths: [
            [$class: 'SparseCheckoutPath', path: 'services/order-service/']
        ]]
    ],
    userRemoteConfigs: [[url: env.REPO_URL, credentialsId: 'bitbucket-pat']]
])
```

**2. Dependency Caching (PVC in K8s)**
```yaml
# maven-cache-pvc.yaml — shared across builds
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: maven-cache-pvc
  namespace: jenkins-agents
spec:
  accessModes: [ReadWriteMany]    # EFS for multi-node access
  storageClassName: efs-sc
  resources:
    requests:
      storage: 50Gi
```

**3. Parallel Stages**
```groovy
stage('Quality & Security') {
    parallel {
        stage('SonarQube')  { steps { /* ... */ } }
        stage('Trivy')      { steps { /* ... */ } }
        stage('BlackDuck')  { steps { /* ... */ } }
        stage('Lint')       { steps { /* ... */ } }
    }
}
```

**4. Conditional Execution — Don't Run What You Don't Need**
```groovy
stage('Integration Tests') {
    when {
        anyOf {
            branch 'main'
            branch 'release/*'
            changeRequest target: 'main'    // PRs to main
        }
    }
    steps { /* expensive integration tests */ }
}

stage('Deploy to Prod') {
    when {
        allOf {
            branch 'main'
            not { changeRequest() }     // Not a PR
        }
    }
    steps { /* ... */ }
}
```

**5. Abort on Stale Builds**
```groovy
// If a new commit comes in, kill the running build for the same branch
options {
    disableConcurrentBuilds(abortPrevious: true)
}
```

---

## 6. Webhook Flow — End to End

```
Developer pushes to Bitbucket
         │
         ▼
Bitbucket Webhook fires ─────────► Jenkins /github-webhook/ endpoint
  (POST with payload:                        │
   repo, branch, commit,                     ▼
   changed files)              Jenkins Multibranch Pipeline
                               scans for Jenkinsfile in branch
                                             │
                                             ▼
                               Pipeline executes ──► Build, Test, Scan
                                             │
                                             ▼
                               Jenkins fires Bitbucket Build Status API
                               (INPROGRESS → SUCCESSFUL/FAILED)
                                             │
                                             ▼
                               PR in Bitbucket shows ✅ or ❌
                               (branch protection: must pass to merge)
```

**Webhook configuration gotchas:**

| Issue | Symptom | Fix |
|-------|---------|-----|
| Webhook URL wrong | Pushes don't trigger builds | Verify: `https://jenkins.novamart.com/bitbucket-hook/` |
| Jenkins behind VPN | Bitbucket cloud can't reach webhook | Webhook relay (smee.io) or allow Bitbucket IPs |
| Multibranch not scanning | New branch not discovered | Branch source: periodic scan as backup (every 5min) |
| Too many triggers | Every push triggers, even docs | Path-based filtering: `when { changeset "services/order-service/**" }` |
| Webhook secret missing | Anyone can trigger builds | Always configure webhook secret/token |

---


---

## Quick Reference Card

```
JENKINS ARCHITECTURE:
  Controller = brain (UI, queue, config, credentials) — NEVER build here
  Agent = muscle (runs builds) — disposable, labeled, dynamic on K8s
  $JENKINS_HOME/secrets/master.key = encrypt key for ALL credentials
  
PIPELINE:
  Declarative > Scripted (always start declarative)
  agent none → per-stage agents (K8s pods)
  options { timeout, timestamps, buildDiscarder, disableConcurrentBuilds }
  environment { REGISTRY = '...'; CREDS = credentials('id') }
  when { branch 'main'; changeset 'path/**' }
  parallel { stage('A') {...}; stage('B') {...} }
  post { always/success/failure/unstable/changed }

SHARED LIBRARIES:
  vars/ = global functions (the API surface)
  Pin version: @Library('lib@v2.3.0') _
  Golden path: standardPipeline(appName: 'x', language: 'java')

SECURITY:
  withCredentials([...]) { } — scoped, masked
  Never: echo $SECRET, --build-arg SECRET=, sh "curl -u $user:$pass"
  Script approval = privilege escalation vector — lock it down
  Folder-level credentials > global credentials

OPTIMIZATION:
  Shallow clone (--depth 1)
  PVC cache (maven, npm, go modules)
  Parallel stages (scans, tests)
  Conditional stages (when {})
  Kill stale builds (abortPrevious: true)
  Kaniko (no Docker daemon, cached layers in registry)

CPS GOTCHA:
  JsonSlurper → @NonCPS or readJSON step
  All pipeline variables must be Serializable

GITOPS BRIDGE:
  Jenkins (CI) → builds image → updates manifest repo → ArgoCD (CD) syncs
```

---

## Retention Questions

**Q1.** You're onboarding at NovaMart. The current Jenkins setup has the controller running builds (executors = 4), all 200 services use `@Library('pipeline-lib') _` without version pinning, and credentials are stored globally. The security team is flagging risk. You're asked to write a remediation plan. What are your top 5 changes, ordered by risk severity, and what's the blast radius of each if left unfixed?

**Q2.** A developer reports their pipeline takes 22 minutes. You look at the Jenkinsfile and see: sequential stages (build → test → sonar → trivy → blackduck → docker build → push → deploy), `checkout scm` doing a full clone of a 4GB monorepo, Maven downloading dependencies fresh every build, and Docker building from scratch each time (no layer cache). Walk me through your optimization plan — what changes would you make, what's the expected time savings for each, and what's the total target build time?

**Q3.** At 3 AM, PagerDuty alerts: "Jenkins controller is down." When it comes back up after restart, you discover 15 in-flight builds were lost, 3 teams can't deploy because their builds are stuck in "pending" (no agents available), and the SonarQube quality gate webhook is returning 404. Triage this incident: what do you investigate first, what's your recovery sequence, and what changes do you implement to prevent recurrence?

**Q4.** Explain the CPS (Continuation Passing Style) execution model in Jenkins pipelines. Why does it exist? What specific problems does it cause? Give a code example that would break due to CPS and show the fix. Then explain why shared library code in `src/` is trusted (no sandbox) while Jenkinsfile code is sandboxed — and what security implication this creates.


These are not "homework" questions; these are **architectural audits**. If you answer these like a student, you fail. I need you to answer them as the Senior Engineer who is responsible for the uptime of NovaMart's deployment engine.

---

### Q1: The NovaMart Remediation Plan

The current setup is a "ticking time bomb." The combination of a controller-as-worker and unpinned shared libraries means a single bad commit in the library could crash the controller and block 200 services simultaneously.

**Remediation Plan (Ordered by Risk Severity):**

| Priority | Change | Risk if Left Unfixed | Blast Radius |
| :--- | :--- | :--- | :--- |
| **1. CRITICAL** | **Move Builds to Agents** (K8s Pods) | **Systemic Outage.** A single heavy build (e.g., a massive Java compile) consumes all Controller CPU/RAM, killing the UI and API. | **100%** of all pipelines. |
| **2. HIGH** | **Credential Scoping/Vault Integration** | **Lateral Movement.** Global creds mean any developer with "Job Configure" access can steal the production DB password or AWS keys. | All production secrets across all 200 services. |
| **3. HIGH** | **Version Pinning for Shared Libs** | **Mass-Failure.** A change to `pipeline-lib` is instantly applied to all 200 services. One syntax error = 200 broken pipelines. | All active deployments. |
| **4. MEDIUM** | **Implement JCasC (Config-as-Code)** | **Configuration Drift.** Manual UI changes are untracked. If the controller dies, you cannot reliably recreate the setup. | Recovery Time Objective (RTO) increases from mins to days. |
| **5. LOW** | **Set Global Timeouts & Concurrency Limits** | **Resource Starvation.** A "zombie" build or a loop in a pipeline occupies an executor forever, blocking other teams. | Specific teams/services (Queue congestion). |

---

### Q2: The Pipeline Optimization Audit

The current pipeline is a "sequential disaster." It treats a modern CI process like a 1990s batch job.

**Optimization Plan:**

1.  **Shallow Clone (`--depth 1`):**
    *   **Change:** Replace `checkout scm` with a shallow clone. We don't need 10 years of history to build a JAR.
    *   **Saving:** ~5–8 minutes (depending on network/repo size).
2.  **Parallelize "Quality" Stages:**
    *   **Change:** Run `Sonar`, `Trivy`, and `Blackduck` in a `parallel` block. They don't depend on each other.
    *   **Saving:** $\sum(\text{times}) \rightarrow \max(\text{times})$. Saving ~5–10 minutes.
3.  **Dependency Caching (Maven/Gradle):**
    *   **Change:** Mount a persistent volume (PV) to the agent at `/root/.m2` or use a local Nexus/Artifactory proxy. Stop downloading the internet on every build.
    *   **Saving:** ~5–10 minutes.
4.  **Docker Layer Caching & BuildKit:**
    *   **Change:** Use `--cache-from` pointing to the latest image in ECR and enable BuildKit (`DOCKER_BUILDKIT=1`). Order Dockerfile to put `COPY pom.xml` before `COPY src`.
    *   **Saving:** ~5 minutes (for incremental builds).

**Expected Total Build Time:**
$22\text{m} \rightarrow \mathbf{5\text{--}8\text{m}}$.

---

### Q3: 3 AM Triage: Jenkins Controller Outage

**Investigation Order:**
1.  **Host Health:** Check CPU/RAM/Disk of the controller node. If disk is 100%, the JVM likely crashed due to an inability to write logs or the `JENKINS_HOME` directory.
2.  **K8s Agent State:** Since builds are pending, check if the Agent pods are stuck in `Pending` or `ImagePullBackOff`. The controller might be up, but the "muscle" is missing.
3.  **Webhook Connectivity:** The 404 on the SonarQube webhook suggests a **Networking/DNS change** or a **misconfigured Ingress**. If the controller IP changed during the restart, the webhook is hitting a dead end.

**Recovery Sequence:**
`Controller Health` $\rightarrow$ `Agent Provisioning` $\rightarrow$ `External Webhook Verification` $\rightarrow$ `Priority-based Build Re-triggering`.

**Prevention:**
1.  **High Availability (HA):** Move to a multi-node Jenkins setup or a managed service.
2.  **Externalized State:** Store `JENKINS_HOME` on a high-performance EFS/SSD volume to prevent disk-related crashes.
3.  **Health Checks:** Implement a `/health` check in the controller that alerts PagerDuty *before* the crash (e.g., when heap usage hits 90%).

---

### Q4: The CPS Execution Model & Security

**What is CPS?**
Jenkins pipelines are not standard Groovy scripts; they are executed in **Continuation Passing Style (CPS)**. 

**Why it exists:**
Jenkins pipelines can take hours. If the Jenkins controller restarts, you can't lose the progress of a 3-hour build. CPS allows Jenkins to "pause" a method, save its entire state (the stack, variables, and program counter) to disk as a serialized object, and "resume" it after a restart.

**The Problem: `NotSerializableException`**
Since every variable must be serializable to be saved to disk, you cannot use non-serializable Java objects (like `Socket`, `InputStream`, or certain Third-Party Library objects) across a "pause point" (any step like `sh`, `sleep`, or `echo`).

**Example of a Break:**
```groovy
node {
    def scanner = new java.util.Scanner(System.in) // Non-serializable object
    sh 'echo Hello' // PAUSE POINT: Jenkins tries to save 'scanner' to disk
    println scanner.nextLine() // CRASH: NotSerializableException
}
```
**The Fix:** Use the `@NonCPS` annotation. This tells Jenkins: *"Run this method as a standard Java method; do not try to pause it or save its state."*

**The Sandbox vs. `src/` Security Implication:**
*   **Jenkinsfile (Sandboxed):** Runs in a restricted environment. It cannot call arbitrary Java methods because a developer could write `System.exit(0)` and kill the controller. It requires an admin to "approve" specific signatures.
*   **Shared Library `src/` (Trusted):** Code in `src/` is executed **outside the sandbox**. It has full access to the JVM and Jenkins internal API.

**Security Implication:** A Shared Library is a **super-user**. If an attacker can commit code to the `pipeline-lib` repository, they can execute arbitrary code on the Jenkins Controller, steal all global credentials, and potentially gain root access to the underlying node. **Access to the Shared Library repo must be as strictly guarded as the production environment itself.**


Q1: Remediation Plan — 4.2/5
What you nailed:

Correct priority ordering. Controller-as-worker is the highest risk — systemic, not targeted.
Credential scoping and blast radius analysis is solid.
JCasC mention is excellent — most people forget configuration drift as a risk.
Blast radius descriptions are precise and business-aware.
What you missed:

No mention of Script Approval lockdown. You flagged credentials but ignored the fact that with builds running on the controller + global credentials + no script approval restrictions, any developer can write a Jenkinsfile that calls System.getenv() or reads master.key directly. This is worse than credential scoping — it's arbitrary code execution on the brain of your CI system. Should be Priority 2 or merged into your credential item.

No audit trail. You can't scope credentials retroactively without knowing who's using what. Step 0 of the credential remediation is: enable Audit Trail plugin, log all credential access for 2 weeks, THEN scope. Otherwise you'll break pipelines blindly.

Missing: RBAC / folder isolation. You mentioned JCasC but not folder-level RBAC. Without it, Team A can modify Team B's pipelines. Combined with global credentials, that's cross-team lateral movement.

Concurrency limits as "LOW" is debatable. A zombie build occupying the controller's 4 executors IS a systemic outage since builds run on the controller. With your Priority 1 fix in place (agents), then yes it drops to LOW. But in the current broken state, it's higher.

Production consequence of the gap: You'd implement credential scoping but leave script approval wide open — an attacker writes a Jenkinsfile that bypasses credential binding entirely by reading /var/jenkins_home/secrets/master.key directly. Game over.

Q2: Pipeline Optimization — 3.8/5
What you nailed:

Correct identification of all 4 major bottlenecks (clone, sequential, deps, docker cache).
Parallel block reasoning is correct (sum → max).
Target time range is realistic (5-8min).
Dockerfile layer ordering mention is good.
What you missed:

No mention of Kaniko. You said "BuildKit + --cache-from" which requires a Docker daemon (DinD, privileged mode). The lesson explicitly covered Kaniko as the production pattern — no daemon, no privileged, registry-based caching. This is the NovaMart standard.

Sparse checkout not mentioned. Shallow clone helps with history depth but this is a 4GB monorepo. You still download all 200 services' source code. Sparse checkout (sparseCheckoutPaths) limits to just the service directory — that's where the bulk of the clone savings come from.

No abortPrevious: true. If a developer pushes 3 commits in 5 minutes, you're running 3 full pipelines. The first two are waste. This is free optimization.

No conditional stage execution. Why run BlackDuck on a feature branch? Why run integration tests on a docs change? when { changeset } and when { branch } eliminate entire stages.

Savings math is hand-wavy. "5-10 minutes" for dependency caching is a range so wide it's not useful. Be specific: fresh Maven download of Spring Boot dependencies = ~3-4min. PVC cache hit = ~15s. Saving: ~3.5min. That's how you justify infra spend to management.

Production consequence: You optimize but still run privileged DinD containers and waste CI minutes on redundant builds.

Q3: 3 AM Triage — 3.5/5
What you nailed:

Correct instinct to check host health (disk/CPU/RAM) first.
Identifying the webhook 404 as a networking/DNS/ingress issue is correct.
Recovery sequence order is logical.
HA and externalized state for prevention are appropriate.
What you missed:

You skipped the 15 lost in-flight builds. This is the most urgent user-facing issue. Which 15 builds? Were any deploying to production mid-flight? A build that was pushing a Docker image and got killed might have pushed a partial/corrupt image tag. A build that was updating the manifest repo might have left a half-committed state. You need to audit the state of each lost build, not just "re-trigger" them.

"Pending" builds — wrong diagnosis direction. You jumped to "Agent pods stuck in Pending/ImagePullBackOff." More likely: the controller just restarted, the Kubernetes plugin hasn't re-registered, or the pod templates are misconfigured after the restart. First check: kubectl get pods -n jenkins-agents — are there ANY agent pods? If zero, the issue is controller → K8s API connectivity, not agent health.

Prevention is incomplete:

No mention of $JENKINS_HOME backup strategy (EBS snapshots, scheduled)
No mention of build distributed persistence (Durable Task plugin, which allows builds to survive controller restart — this is why CPS exists)
No mention of PDBs for the Jenkins controller StatefulSet
"/health check that alerts at 90% heap" — Jenkins has a built-in health check endpoint (/login returns 200) but for JVM heap, you'd use Prometheus JMX exporter + alerting, not a custom health check
"Multi-node Jenkins" for HA is vague. Jenkins doesn't natively support multi-controller. You either mean CloudBees CI (commercial, controller-per-team), or you mean active-passive with shared storage. Saying "multi-node" without specifics suggests you're guessing.

Production consequence: You restart Jenkins, re-trigger builds, but miss that build #4532 of payment-service pushed a corrupt image tag that ArgoCD is now trying to deploy to production.

Q4: CPS Explanation — 4/5
What you nailed:

Correct explanation of WHY CPS exists (survive controller restarts during long builds).
"Pause point" concept is accurate — any pipeline step is a potential serialization boundary.
The security implication of shared library src/ is perfectly stated: "A Shared Library is a super-user."
The fix (@NonCPS) is correct.
What you missed:

Your code example is wrong. new java.util.Scanner(System.in) isn't what causes CPS issues in practice — nobody reads stdin in a Jenkins pipeline. The classic real-world break is JsonSlurper (returns LazyMap, which is non-serializable) or HttpURLConnection. The lesson explicitly covered this. Use the real-world example.

You didn't mention readJSON/readYaml pipeline steps as the preferred alternative to @NonCPS. The @NonCPS annotation has a limitation: it cannot call CPS-transformed methods (pipeline steps like sh, echo) inside it. The better fix for JSON parsing is readJSON(text: response) — a proper pipeline step that's CPS-safe.

Missing: CPS performance impact. CPS transformation makes Groovy pipelines significantly slower than native Groovy. Tight loops, heavy string manipulation, large data structures — all suffer. This is why you push logic into shell scripts (sh) or external tools rather than doing it in Groovy.

Sandbox bypass via shared library — incomplete threat model. You correctly said an attacker with commit access to the library repo can execute arbitrary code. But you didn't mention: even without direct commit access, if the library is loaded from @Library('lib') _ without version pinning, a developer who can push to the library's default branch gets the same power. This connects directly to Q1 (version pinning).

===========================================================================================


