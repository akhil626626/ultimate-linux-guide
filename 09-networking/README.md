# Networking Commands

1. **`ping google.com`** – Checks connectivity to a remote server.
2. **`ifconfig`** – Displays network interfaces (deprecated, use `ip`).
3. **`ip a`** – Shows IP addresses of network interfaces.
4. **`netstat -tulnp`** – Displays open network connections.
5. **`curl https://example.com`** – Fetches a webpage's content.
6. **`wget https://example.com/file.zip`** – Downloads a file from the internet.

# Linux Network Packet Flow – Practical Troubleshooting Guide

Short, practical commands to trace packet flow on a Linux host (works great for interviews and real incidents).

> ✅ Most commands require `sudo`. Replace `eth0` and ports with your real values.

---

## Quick Triage (10 minutes)

1. **Is traffic hitting the box?**
   ```bash
   tcpdump -i eth0 -nn port 80
ss -tulnp
ping -c3 -n google.com – Quick reachability + DNS resolution check.

ping -c3 -n 8.8.8.8 – Bypass DNS; pure network reachability to the internet.

ip a – Show IPs on all interfaces (modern replacement for ifconfig).

ip r – Show routing table (default gateway, specific routes).

ip neigh – ARP/ND cache (who’s at what MAC).

ss -tulnp – List listening sockets with owning PIDs (modern netstat).

lsof -iTCP -sTCP:LISTEN -n -P – Which process is listening on which TCP port.

curl -vIk https://example.com – HTTPS reachability & TLS handshake (no body).

curl -v http://127.0.0.1:8080 – Test local service bind/health.

curl -v --resolve example.com:443:1.2.3.4 https://example.com/ – Test LB/DNS override.

wget https://example.com/file.zip – Download a file (simple fetch).

openssl s_client -connect host:443 -servername host – Inspect TLS cert/ALPN/handshake.

nc -vz host 443 – Fast TCP port open test (client-side).

traceroute -n host – Hop-by-hop path (UDP by default).

mtr -rwzbc100 host – Continuous traceroute + loss/latency stats (great for flaps).

dig +short A example.com – Resolve A record quickly.

dig @1.1.1.1 example.com – Query a specific DNS resolver.

resolvectl status – Systemd-resolved DNS config/servers (or systemd-resolve --status).

ethtool eth0 – Link speed/duplex/driver info.

ethtool -S eth0 – Per-NIC counters (drops, errors).

ip -s link show eth0 – RX/TX stats + errors per interface.

tcpdump -i eth0 -nn port 80 – See HTTP traffic hitting/leaving eth0.

tcpdump -i any host 10.0.0.5 and port 443 – Filtered capture across all ifaces.

tcpdump -w cap.pcap -G 300 -W 3 -i eth0 – Rolling capture (3×5min files).

tshark -i eth0 -Y 'tcp.port==8443' – CLI decoding (Wireshark engine).

ngrep -d any -W byline 'Host: ' port 80 – Grep HTTP headers in flight.

nft list ruleset – Show nftables firewall rules.

iptables -L -v -n – Show iptables filter rules + counters.

iptables -t nat -L -v -n – Show NAT (SNAT/DNAT/MASQ) behavior.

firewall-cmd --list-all – firewalld active zone/services/ports.

ufw status verbose – UFW firewall status (Ubuntu).

conntrack -S – Conntrack stats (NAT/state table pressure).

conntrack -L | wc -l – Active conntrack entries count.

sysctl net.ipv4.ip_forward – Is IP forwarding enabled (routing/NAT boxes).

iperf3 -s / iperf3 -c host -R – Bandwidth test server/client (reverse too).

iftop -i eth0 / nload / bmon – Live bandwidth per host/interface.

tc -s qdisc show dev eth0 – Queueing/traffic control stats (drops due to qdisc).

bpftool prog show / bpftool net – eBPF programs/hooks affecting packets.

journalctl -k | grep -iE 'eth|mlx|drop|xdp' – Kernel/network driver messages.

dmesg | grep -i mtu – MTU issues, link flaps, offload hints.

ℹ️ Some tools may need install: mtr, iperf3, iftop, nload, bmon, conntrack-tools, ngrep, tshark.

Linux Network Packet Flow – Practical Troubleshooting Guide

✅ Use sudo where needed. Replace eth0, IPs, and ports with real values.

1) Is traffic reaching the host?

tcpdump -i eth0 -nn port 80 – packets arriving/leaving?

No packets? Check upstream (LB/SG/NACL/route). Check correct interface.

2) Is the service actually listening?

ss -tulnp | grep ':80 ' – process bound & on the right IP (0.0.0.0 vs 127.0.0.1).

lsof -iTCP -sTCP:LISTEN -n -P | grep ':80' – confirm owning PID/binary.

3) Local firewall/NAT dropping it?

nft list ruleset or iptables -L -v -n – drops increase?

iptables -t nat -L -v -n – unexpected DNAT/SNAT rules?

firewall-cmd --list-all / ufw status verbose – quick high-level view.

4) Routes and ARP okay?

ip r get <client_ip> – which gateway/interface will reply?

ip neigh | grep <gateway> – ARP resolved? Incomplete entries = L2 issue.

ethtool -S eth0 / ip -s link show eth0 – RX/TX errors/drops.

5) App works locally but not remotely?

curl -v http://127.0.0.1:8080 – local OK?

curl -v http://<server-ip>:8080 – bound only to loopback? Fix bind address.

ss -tulnp | grep 8080 – confirm it listens on external IP.

6) TLS/HTTPS issues?

openssl s_client -connect host:443 -servername host – cert/SNI/ALPN.

curl -vk https://host – verbose TLS errors, HTTP/2 vs 1.1, redirects.

7) DNS problems vs network?

dig +short A host – name resolves?

ping -c3 host vs ping -c3 <ip> – DNS vs reachability.

resolvectl status – which resolvers are used.

8) Path issues / asymmetric routing / MTU

mtr -rwzbc100 host – loss/latency along the path.

ip r – policy routing? multiple default routes?

ping -M do -s 1472 host – test PMTU (adjust size until it passes).

9) Throughput & saturation

iperf3 -c host -R – measure bandwidth both directions.

iftop -i eth0 / bmon – who’s hogging the link.

tc -s qdisc show dev eth0 – drops in queues (bufferbloat/shaping).

10) Deep dive (when all else fails)

bpftool prog/tc – custom eBPF/TC/XDP filters interfering.

conntrack -S – NAT/state exhaustion.

journalctl -k – driver resets, NIC errors, offload quirks.

Handy One-Liners

Which process owns port 8443?
ss -lntp '( sport = :8443 )' ; lsof -iTCP:8443 -sTCP:LISTEN -n -P

Test remote port quickly:
nc -vz host 8443 || echo "blocked or closed"

Follow HTTP headers live:
ngrep -d any -W byline '^(GET|POST|Host:|HTTP/)' tcp and port 80

Confirm egress IP (NAT):
curl -s ifconfig.io

Find drops on interface:
ip -s link show eth0 | awk '/RX:|TX:/{p=1;next} p&&NF{print;exit}'

Typical Workflow (example: “Port 8080 from LB can’t reach app”)

Service up? ss -lntp | grep ':8080 '

Local test: curl -v http://127.0.0.1:8080 (should be 200)

Bind check: curl -v http://<server-ip>:8080 (not just loopback)

Packets arriving? tcpdump -i eth0 -nn port 8080

Firewall counters: nft list ruleset or iptables -L -v -n (look for DROP counters)

Routing/ARP: ip r get <LB-ip>; ip neigh | grep <gateway>

MTU/path: mtr -rwzbc100 <client>; ping -M do -s 1472 <client>

If still failing: check LB SG/NACL, server SELinux (getenforce), app logs.
