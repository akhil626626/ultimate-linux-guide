# Networking Commands (Cheat-Sheet)

> Replace `eth0`, hosts, and ports with your real values. Most commands need `sudo`.

## Basics

```bash
ping -c3 -n google.com              # Reachability + DNS
ping -c3 -n 8.8.8.8                 # Reachability bypassing DNS
ip a                                # IPs on all interfaces
ip r                                # Routing table (default gateway, routes)
ip neigh                            # ARP/ND cache (L2 neighbors)
hostname -I                         # All local IPs (space-separated)
```

## Sockets & Processes

```bash
ss -tulnp                           # Listening sockets + owning PIDs (modern netstat)
lsof -iTCP -sTCP:LISTEN -n -P       # Which process owns which TCP port
ss -antp | grep ESTAB               # Active TCP connections (established)
```

## HTTP / TLS / Port Tests

```bash
curl -vIk https://example.com       # HTTPS reachability & TLS handshake (no body)
curl -v http://127.0.0.1:8080       # Hit local service directly
curl -v --resolve example.com:443:1.2.3.4 https://example.com/  # DNS override
openssl s_client -connect host:443 -servername host             # Inspect TLS/SNI/certs
nc -vz host 443                     # Fast "is TCP port open?" check
```

## DNS

```bash
dig +short A example.com            # Quick A-record lookup
dig @1.1.1.1 example.com            # Ask a specific resolver
resolvectl status                   # System DNS settings (systemd-resolved)
```

## Packet Capture (pcap)

```bash
tcpdump -i eth0 -nn port 80         # See HTTP packets on eth0
tcpdump -i any host 10.0.0.5 and port 443   # Capture across all ifaces w/ filter
tcpdump -w cap.pcap -G 300 -W 3 -i eth0     # Rolling capture (3×5-min files)
tshark -i eth0 -Y 'tcp.port==8443'  # CLI decode via Wireshark engine
ngrep -d any -W byline '^(GET|POST|Host:|HTTP/)' tcp and port 80  # Follow HTTP headers
```

## Firewall & NAT

```bash
nft list ruleset                    # nftables rules
iptables -L -v -n                   # iptables filter rules + counters
iptables -t nat -L -v -n            # NAT (SNAT/DNAT/MASQ)
firewall-cmd --list-all             # firewalld (RHEL/OL)
ufw status verbose                  # UFW (Ubuntu/Debian)
```

## Routes, ARP, MTU

```bash
ip r get <client_ip>                # Which interface/gateway will reply?
ip neigh | grep <gateway_ip>        # ARP entry (avoid INCOMPLETE)
ping -M do -s 1472 host             # PMTU probe (adjust size until success)
traceroute -n host                  # Hop-by-hop path (numeric)
mtr -rwzbc100 host                  # Continuous trace w/ loss & latency
```

## Interface & NIC Stats

```bash
ip -s link show eth0                # RX/TX bytes, errors, drops
ethtool eth0                        # Link speed/duplex/driver
ethtool -S eth0                     # Per-NIC counters (drops, errors)
```

## Throughput & Live Monitoring

```bash
iperf3 -s                           # Start test server
iperf3 -c host -R                   # Client test (reverse too)
iftop -i eth0                       # Live bandwidth by host
nload                               # Simple per-iface graphs
bmon                                # Lightweight bandwidth monitor
tc -s qdisc show dev eth0           # Queueing (qdisc) stats/drops
```

## Conntrack / NAT State

```bash
conntrack -S                        # Conntrack stats (state table pressure)
conntrack -L | wc -l                # Active conntrack entries count
```

## Logs / Kernel

```bash
journalctl -k | grep -iE 'eth|mlx|drop|xdp'   # Driver/NIC/kernel messages
dmesg | grep -i mtu                           # MTU/offload/link flap hints
getenforce                                    # SELinux mode (Enforcing/Permissive)
```

---

# Linux Packet-Flow: Fast Playbooks

## “LB → port 8080 can’t reach my app”

```bash
ss -lntp | grep ':8080 '            # 1) Service listening?
curl -v http://127.0.0.1:8080       # 2) Local health
curl -v http://<server-ip>:8080     # 3) External bind (not only 127.0.0.1)
tcpdump -i eth0 -nn port 8080       # 4) Packets arrive/leave host?
nft list ruleset                    # 5) Firewall drops/counters?
ip r get <lb_ip>                    # 6) Return path correct?
ip neigh | grep <gateway_ip>        # 7) ARP ok (no INCOMPLETE)
```

## “HTTPS failing / certificate issues”

```bash
openssl s_client -connect host:443 -servername host   # Check chain, SNI, ALPN
curl -vk https://host                                # Verbose TLS errors
```

## “Slow / packet loss suspected”

```bash
mtr -rwzbc100 host                   # Loss & latency per hop
tc -s qdisc show dev eth0            # Queue drops / bufferbloat
iperf3 -c host -R                    # Throughput both directions
```

## “NAT/state exhaustion”

```bash
conntrack -S                         # Look for high insert_fail / drop
conntrack -L | wc -l                 # Active entries count
```

---

# Handy One-Liners

```bash
ss -lntp '( sport = :8443 )' ; lsof -iTCP:8443 -sTCP:LISTEN -n -P   # Who owns 8443?
nc -vz host 8443 || echo "blocked or closed"                        # Quick remote port check
curl -s ifconfig.io                                                 # Confirm egress IP (NAT)
ip -s link show eth0 | awk '/RX:|TX:/{p=1;next} p&&NF{print;exit}'  # Quick RX/TX errors/drops
```
