### 🌐 PHASE 1 — NETWORKING FUNDAMENTALS

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
