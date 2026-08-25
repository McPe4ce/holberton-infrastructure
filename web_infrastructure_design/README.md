# Web Infrastructure Design

Design documents for a website served at `www.foobar.com`, built up one stage at a
time. Each stage has a Mermaid schema (`.mmd`) and a matching section below.

| Stage | Schema | Section |
| --- | --- | --- |
| 1. Single server | [`single_server_stack.pdf`](single_server_stack.pdf) |

---


### What a server is

A server is a computer whose job is to answer requests from clients over a network.
It can be **physical** — a real machine you could unplug — or **virtual**, a slice of
a bigger physical host carved out by a hypervisor or a cloud provider; from the
outside the two are indistinguishable, since both answer on `8.8.8.8`.

Like any computer, a server runs an **operating system** (typically Linux here:
Ubuntu, Debian, CentOS). The OS owns the hardware, the network stack, the
filesystem and the process table, and everything else in this design is just a
process it schedules.

Servers are usually **hosted in a data center**: a facility that supplies the
redundant power, cooling, physical security and network transit that a machine
expected to stay up 24/7 needs. Renting from a hosting or cloud provider means
renting space in theirs.

Finally, the **server is not the services on it**. "Server" is the host; Nginx, the
application server, the application code and the database are four separate
services sharing that one host. In this design all four happen to live on the same
machine, but that is a deployment choice, not a definition — each could be moved to
its own server without changing what any of them does.

### The role of DNS, and why `www` is an A record

**DNS** (Domain Name System) is the internet's phone book: it translates a
human-readable name into the IP address the network actually routes to. Without it
the user would have to type `8.8.8.8` instead of `www.foobar.com`, and the site
could never change host without every visitor learning a new number.

`www` here is an **A record** because an A (address) record is precisely the record
type that maps a hostname directly to an IPv4 address — and in this scenario
`www.foobar.com` points at a literal IPv4 address, `8.8.8.8`. A `CNAME` would be
the right choice only if `www` were an alias pointing at *another name*
(`foobar.com`, or a load balancer's hostname); since the target is an address, not
a name, `A` is the correct record. The browser gets `8.8.8.8` back and opens its
connection there, which is why the schema labels that hop `A record → 8.8.8.8`.

### The role of each component

**Web server (Nginx)** — the front door on port 80/443. It terminates the HTTP
connection, serves static assets (images, CSS, JS) straight from disk, and hands
anything dynamic on to the application server. It also handles concerns that have
nothing to do with business logic: connection handling, keep-alives, gzip, request
logging.

**Application server** — the runtime that actually executes the application
(Gunicorn/uWSGI for Python, PHP-FPM for PHP, Puma for Ruby, Node for JS). It
receives the request forwarded by Nginx, turns it into something the code
understands, runs the code, and returns the generated response. It is the piece
that makes the site *dynamic* rather than a folder of files.

**Application code** — the business logic written by the developers: routing,
templates, validation, everything that decides *what* the answer to this particular
request should be. It is data-less by itself; it asks the database for state.

**Database (PotsgreSQL/MySQL)** — the persistent store. It holds the data that has
to survive a restart (users, posts, orders) and answers the queries the application
code sends it, returning result sets that the code renders into the page.

### How the user and the server talk

The user's browser and the server are on separate networks and communicate over the
internet using the **TCP/IP** protocol suite. IP routes packets to `8.8.8.8`; TCP
opens a reliable, ordered connection (the three-way handshake) on port 80/443 and
guarantees that the bytes of the request and the response arrive intact and in
order; HTTP is the application-level language spoken *inside* that TCP connection.

### LAMP, and why this stack is only LAMP-like

**LAMP** stands for **L**inux, **A**pache, **M**ySQL, **P**HP (or Perl/Python) — the
classic four-layer open-source stack: an OS, a web server, a database, and a
language runtime, all on one box.

This design is **LAMP-like** rather than literal LAMP because it keeps the shape but
swaps components: the web server is **Nginx, not Apache**, and the database may be
**PostgreSQL rather than MySQL**. It also makes the application server an explicit
tier of its own, whereas classic LAMP embeds PHP inside Apache via `mod_php`. Same
roles, different implementations — which is why the pattern is often written LEMP
(Linux, Enginx, MySQL, PHP) when Nginx is in the front.

### Issues with this infrastructure

**Single point of failure (SPOF)** — everything is on one server. If that machine
loses power, its disk, or its network link — or if Nginx, the application server or
the database crashes — the entire site is down. There is no second copy of anything
and no path around the failure.

**Downtime during maintenance** — any change that requires restarting the web stack
(deploying new application code, changing an Nginx config, applying an OS or
database security patch, rebooting the kernel) drops live connections and makes the
site unreachable for the duration. With a single server there is nowhere to shift
traffic while the work happens.

**Cannot scale with traffic** — the whole stack shares one machine's CPU, RAM, disk
I/O and bandwidth, and the four services compete with each other for them; a heavy
database query slows down Nginx on the same box. Growth can only be handled by
*vertical* scaling (a bigger server), which has a hard ceiling and requires a
reboot; there is no way to add a second machine without changing the design.
