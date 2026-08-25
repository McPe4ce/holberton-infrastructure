# Web Infrastructure Design

Incremental design of the infrastructure serving `www.foobar.com` (resolving to
`8.8.8.8`). Each stage adds one idea to the previous one, and is documented with a
rendered schema plus a short section explaining what was added and what is still
weak.

## Contents

| # | Stage | Schema | Adds | Remaining SPOF |
| - | ----- | ------ | ---- | -------------- |
| 1 | [Single server](#1--single-server) | [`single_server_stack.pdf`](single_server_stack.pdf) | The four basic services on one host | Everything |
| 2 | [Redundant web tier](#2--redundant-web-tier) | [`redundant_web_tier.pdf`](redundant_web_tier.pdf) | Load balancer, second web/app node, DB replica | Load balancer, DB primary |

Each section follows the same shape: **Objective → Components → Key points →
Limitations**.

---

## 1 · Single server

**Objective:** model the request path of a minimal web application and distinguish
the role of each basic component.
**Schema:** [`single_server_stack.pdf`](single_server_stack.pdf)

### Components

| Component | Role |
| --------- | ---- |
| **User** | A browser that wants a page. |
| **www.foobar.com** | The human-readable name typed into the browser. |
| **DNS** | Resolves that name to an IP address. |
| **Web server (Nginx)** | The front door on port 80/443: terminates the HTTP connection, serves static files (images, CSS, JS) from disk, forwards anything dynamic. |
| **Application server** | The runtime that executes the application (Gunicorn, PHP-FPM, Puma, Node). It turns a forwarded request into a generated response — the piece that makes the site dynamic. |
| **Application code** | The business logic: routing, templates, validation. It decides *what* the answer should be, and asks the database for state. |
| **Database (PotsgreSQL/MySQL)** | The persistent store. Holds data that must survive a restart and answers the queries the code sends it. |

**Request path:** user → `www.foobar.com` → DNS → `A record → 8.8.8.8` → Nginx →
application server → application code → database, then the result set back out
along the same chain as an HTTP response.

### Key points

**What a server is.** A server is a computer that answers client requests over a
network. It may be **physical** — a machine you could unplug — or **virtual**, a
slice of a larger host carved out by a hypervisor; from the outside the two are
indistinguishable.

**It runs an OS.** Like any computer it runs an operating system (typically Linux),
which owns the hardware, the network stack and the process table. Everything else
in this design is just a process the OS schedules.

**It lives in a data center.** Servers are normally hosted in a facility providing
redundant power, cooling, physical security and network transit — the things a
machine expected to stay up 24/7 needs. Renting from a cloud provider means renting
space in theirs.

**The server is not the services on it.** "Server" is the *host*; Nginx, the
application server, the code and the database are four separate *services* sharing
it. They live together here by deployment choice, not by definition — each could be
moved to its own host without changing what it does.

**DNS and the A record.** DNS is the internet's phone book: it maps a name to the IP
address the network actually routes to, so the site can move hosts without visitors
learning a new number. `www` is an **A record** because an A (address) record maps a
hostname straight to an IPv4 address, and here the target *is* an address —
`8.8.8.8`. A `CNAME` would be correct only if `www` pointed at another *name*.

**TCP/IP.** User and server sit on different networks and talk over the **TCP/IP**
suite: IP routes packets to `8.8.8.8`, TCP opens a reliable ordered connection (the
three-way handshake) on port 80/443, and HTTP is the language spoken *inside* that
connection.

**LAMP-like, not LAMP.** LAMP = **L**inux, **A**pache, **M**ySQL, **P**HP — an OS, a
web server, a database and a language runtime on one box. This design keeps that
shape but swaps parts: **Nginx instead of Apache**, optionally **PostgreSQL instead
of MySQL**, and the application server is its own explicit tier rather than PHP
embedded in Apache via `mod_php`. The Nginx variant is usually called **LEMP**.

### Limitations

- **Single point of failure** — one machine holds everything. Lose its power, disk
  or network link, or crash any one of the four services, and the whole site is
  down. There is no second copy and no path around the failure.
- **Downtime on maintenance** — deploying code, editing the Nginx config or
  patching the OS or database means restarting the stack, and there is nowhere to
  shift traffic while that happens.
- **Hard capacity ceiling** — the four services share one machine's CPU, RAM, disk
  I/O and bandwidth and compete for them; a heavy query slows Nginx on the same box.
  Growth means *vertical* scaling (a bigger server), which caps out and needs a
  reboot.

---

## 2 · Redundant web tier

**Objective:** explain how a load balancer and redundant web/application paths
improve availability and capacity, and recognise what stays vulnerable.
**Schema:** [`redundant_web_tier.pdf`](redundant_web_tier.pdf)

### Components added

| Component | Role |
| --------- | ---- |
| **Load balancer (HAProxy)** | Single public entry point on `8.8.8.8`. Accepts every request and distributes it across the two web/app nodes. |
| **Web/App node 1** | Full path — Web server (Nginx) + Application server + app files + **Database primary (MySQL)**, read **and** write. |
| **Web/App node 2** | Identical path — Web server (Nginx) + Application server + app files + **Database replica (MySQL)**, **read-only**. |
| **replication** | Directed flow **primary → replica**: the primary streams its committed changes so the replica holds an up-to-date copy. |

### Key points

**Why the load balancer.** The DNS A record can only point at one address, so with a
second server there has to be something in front to own that address and spread
requests. HAProxy takes the traffic, checks which backends are healthy, and stops
sending to any node that fails a health check — which is what turns a second machine
into actual availability rather than just a spare.

**Why the second path.** Duplicating the whole web + app path removes the single
server as a total-outage cause: a crash, a deploy or a reboot on one node leaves the
other serving. It also adds capacity, since two nodes' CPU and RAM handle requests
in parallel instead of one.

**Active-active.** Both paths are **active-active**: HAProxy sends live traffic to
node 1 and node 2 at the same time, so both contribute capacity and a failure only
costs the share the dead node was carrying. In **active-passive**, one node would
serve everything while the other idled as a standby, taking over only after a
failover — simpler and easier to keep consistent, but the standby's capacity is
wasted during normal operation.

**Distribution method — round robin.** HAProxy hands request 1 to node 1, request 2
to node 2, request 3 back to node 1, and so on, cycling through healthy backends in
order. It is simple and needs no state, but it assumes requests cost roughly the
same and that nodes are equally powerful; it also sends a returning user to a
different node each time, so sessions have to live somewhere shared rather than in
node memory. (`leastconn` and `source` hashing are the usual alternatives.)

**Primary, replica, replication.** The **primary** is the only writable database:
every `INSERT`/`UPDATE`/`DELETE` goes there. The **replica** applies the primary's
replication stream and serves **reads only** — it exists to take read load off the
primary and to hold a warm, current copy of the data. Replication is **one-way and
does not perform failover**: if the primary dies, the replica keeps answering reads
but the application cannot write until an operator manually promotes it and
repoints the application. Replication protects the *data*, not write *availability*.

**Cost of redundancy.** All of this buys availability and capacity by buying more
of everything: a second server plus the load balancer host, more storage for the
replicated copy, and more operational work — two machines to patch and keep
identical, replication lag to watch, health checks and failover procedures to
maintain and test. Redundancy is a trade, not a free upgrade.

### Limitations

- **Load balancer is a SPOF** — a single HAProxy owns the public address; if it
  goes down, both healthy nodes become unreachable.
- **Writable primary is a SPOF** — one database accepts writes. Losing it leaves
  the site readable at best, and restoring writes needs a manual promotion.
- **No HTTPS** — the schema labels the transport HTTP/HTTPS, but nothing here
  actually terminates TLS: no certificate, no termination point on the load
  balancer. Traffic is effectively plaintext and open to interception.
- **No firewalls** — every host is directly exposed; nothing restricts which ports
  and sources can reach the nodes or the databases.
- **No monitoring** — beyond HAProxy's own health checks there is no metric
  collection, log aggregation or alerting, so a failure, a saturating node or
  growing replication lag is noticed only when users complain.
