« [Back to syllabus](syllabus.md) · Prerequisite: [Module 1](module-1-process-fundamentals.md)

# Module 3 - Networking

Raw TCP first, then name resolution (DNS) and packet filtering/isolation (iptables, network namespaces) built on top of it, then a real-world consumer (reverse proxies), then the two encryption/auth layers built on top of the handshake (TLS, SSH). Network namespaces here are also a direct preview of Module 5 (containers).

---

## 14. TCP/IP handshake, ports, and sockets
- [ ] Done

**Resource:** Julia Evans "Bite Size Networking!" zine (illustrated) - https://wizardzines.com/zines/bite-size-networking/
Runner-ups: *Beej's Guide to Network Programming* for depth; ByteByteGo's short animated "How TCP Handshake Works" for a quick SYN/SYN-ACK/ACK visual.

**What you'll learn:** How ports and sockets actually map onto a TCP connection, and the three-way handshake mechanism - analogy-first, mechanism-accurate, in Evans' usual style.

**Confirm it:**
```
ss -tlnp
```
Pick one listening socket and identify its local port, the process/PID that owns it, and whether it's bound to `0.0.0.0` (all interfaces) or a specific address.

---

## 15. DNS - how a name becomes an IP
- [ ] Done

**Resource:** "How DNS Works!" - Julia Evans (zine) - https://wizardzines.com/zines/dns/ - comics: "life of a DNS query" - https://wizardzines.com/comics/life-of-a-dns-query/ and "the DNS hierarchy" - https://wizardzines.com/comics/dns-hierarchy/
Since NetworkChuck's "What is DNS? (and how it makes the Internet work)" is literally the reference bar this whole syllabus is built against, watch it here too - it's a legitimately good fit for this exact topic.

**What you'll learn:** DNS resolution is a *recursive walk down a hierarchy* (root → TLD → authoritative nameserver), not a single lookup - your resolver asks the root "who handles .com?", then asks that server "who handles wizardzines.com?", and so on, caching at each level (which is exactly why DNS changes propagate slowly).

**Confirm it:**
```
dig +trace wizardzines.com
```
Read the output top to bottom and identify each hop: root servers → `.com` TLD servers → the domain's own authoritative nameservers, ending in the final A record.

---

## 16. iptables/nftables and network namespaces
- [ ] Done

**⚠️ Partially flagged - the network-namespace half is well illustrated; the packet-filtering half has no comic-style resource, only diagrams + prose.**

**Resource (pair):** "network namespaces" - Julia Evans (comic) - https://wizardzines.com/comics/network-namespaces/ - plus "iptables Processing Flowchart" - Phil Hagen - https://stuffphilwrites.com/2014/09/iptables-processing-flowchart/ (visual packet-flow diagram) and "A Deep Dive into Iptables and Netfilter Architecture" - DigitalOcean - https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture (accurate mechanism prose).

**What you'll learn:** A network namespace gives a process its own isolated network stack (interfaces, routes, iptables rules) - the exact primitive containers use for network isolation (see Module 5). Separately, `iptables`/`nftables` rules are organized into tables and chains that a packet traverses through five netfilter hook points (`PREROUTING` → routing decision → `INPUT`/`FORWARD` → `OUTPUT` → `POSTROUTING`); mangle rules run before nat, which runs before filter, on every path a packet can take.

**Confirm it:**
```
ip netns add testns
ip netns exec testns ip addr
sudo iptables -L -v -n --line-numbers
```
Confirm the new namespace starts with no network interfaces at all (not even loopback up) - true isolation, not just a filtered view. Then read your host's real iptables ruleset and identify which chain (`INPUT`, `FORWARD`, etc.) each rule belongs to.

---

## 17. How a reverse proxy actually works under the hood
- [ ] Done

**Resource:** "The Architecture of NGINX" - Hussein Nasser - https://medium.com/@hnasr/the-architecture-of-nginx-2b32fc0b7877

**What you'll learn:** One master process + one worker per CPU core; how workers accept connections - older nginx had all workers contend on a shared listener socket, newer versions use `SO_REUSEPORT` so the kernel load-balances across per-worker accept queues.

**Confirm it:**
```
ps aux | grep nginx
```
(If nginx isn't installed, `apt info nginx` or install it in a throwaway container.) Identify the single master process vs the pool of worker processes, and note the worker count roughly matches `nproc`.

---

## 18. TLS/HTTPS handshake and certificate chains
- [ ] Done

**Resource:** "How HTTPS Works" comic - https://howhttps.works/ - paired with Cloudflare's "What happens in a TLS handshake?" - https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/
Deep runner-up: "The Illustrated TLS 1.3 Connection" - https://tls13.xargs.org/ - annotates every byte of a real handshake.

**What you'll learn:** Client hello (ciphers + random) → server hello + certificate + random → client validates the cert against the CA's signature → key exchange → both sides derive matching session keys. Crucially: the asymmetric handshake only sets up keys - the actual session traffic is symmetric encryption.

**Confirm it:**
```
openssl s_client -connect example.com:443 -brief
```
Read the output and identify: the negotiated TLS version/cipher, and at least one certificate in the chain plus its issuer (the CA).

---

## 19. SSH key exchange and authentication flow
- [ ] Done

**Resource:** "Deep Dive - How Does SSH Really Work?" - tusharf5 - https://tusharf5.com/posts/ssh-deep-dive/
Runner-up for the certificate model: smallstep's "If you're not using SSH certificates you're doing SSH wrong" - https://smallstep.com/blog/use-ssh-certificates/

**What you'll learn:** SSH is two separate stages - (1) a Diffie-Hellman-style key exchange where both sides independently compute the *same* shared symmetric key without ever transmitting it, then (2) asymmetric authentication of the client (server challenges, client signs with its private key, server checks `authorized_keys`). The key pair is only ever used for stage 2 - it does not encrypt the session itself.

**Confirm it:**
```
ssh -vvv user@yourserver 2>&1 | grep -iE "kex|authentication"
```
Find the line naming the negotiated key-exchange algorithm, and the line(s) showing which authentication method succeeded (e.g. `publickey`).

---

## 🧪 Module 3 capstone - lock it in on a throwaway VM

On a disposable VM (or two, if you want a real client/server split):

1. Point a subdomain you control at the VM's IP (or just edit `/etc/hosts` locally if you don't want to touch real DNS), then `dig +trace` it and confirm you can explain every hop.
2. Install nginx as a reverse proxy in front of a trivial backend (`python3 -m http.server 8000` is enough), and generate a self-signed cert so nginx terminates TLS - confirm with `openssl s_client -connect <vm-ip>:443`.
3. Write an `nftables` (or `iptables`) rule set that only allows inbound 443 and 22, drops everything else, and verify with `iptables -L -v -n` that traffic is actually being counted against the right rule.
4. SSH into the VM with key-based auth and `ssh -vvv`, confirming the kex algorithm and that `publickey` auth succeeded - no password fallback.
5. Tear the VM down.
