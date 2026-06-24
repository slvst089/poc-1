## _Chapter 03: Linux Networking Commands: Cheat Sheet_
###### Published Time: 2026-03-23

##### [Interfaces & IP](https://devopsil.com/articles/2026-03-23-linux-networking-cheat-sheet#interfaces--ip)

```
ip addr show                          # All interfaces
ip -br addr                           # Compact view
ip -s link show eth0                  # Link stats (errors, drops)
sudo ip addr add 10.0.0.5/24 dev eth0
sudo ip addr del 10.0.0.5/24 dev eth0
sudo ip link set eth0 up              # Bring up
sudo ip link set eth0 down            # Bring down
```

##### [Routing](https://devopsil.com/articles/2026-03-23-linux-networking-cheat-sheet#routing)

```
ip route show                         # Route table
ip route get 8.8.8.8                  # Which route for this IP?
sudo ip route add 10.10.0.0/16 via 10.0.0.1 dev eth0
sudo ip route add default via 10.0.0.1
sudo ip route del 10.10.0.0/16
ip neigh show                         # ARP / neighbor table
```

##### [DNS](https://devopsil.com/articles/2026-03-23-linux-networking-cheat-sheet#dns)

```
dig example.com +short                # Quick A record lookup
dig example.com MX                    # MX records
dig @8.8.8.8 example.com             # Query specific nameserver
dig -x 93.184.216.34                  # Reverse lookup
dig example.com +trace                # Full resolution chain
host example.com                      # Simple lookup
resolvectl status                     # systemd-resolved info
```

##### [Connectivity](https://devopsil.com/articles/2026-03-23-linux-networking-cheat-sheet#connectivity)

```
ping -c 4 10.0.0.1
traceroute -n example.com             # Skip DNS (faster)
mtr example.com                       # Continuous traceroute
nc -zv 10.0.0.5 443                  # TCP port check
curl -Iso /dev/null -w "%{http_code}" https://example.com
curl --connect-timeout 5 -v https://example.com
```

##### [Connections & Ports](https://devopsil.com/articles/2026-03-23-linux-networking-cheat-sheet#connections--ports)

```
ss -tlnp                              # TCP listeners with process
ss -ulnp                              # UDP listeners
ss -tnp                               # Established connections
ss -tnp sport = :443                  # Filter by port
ss -s                                 # Socket summary
sudo lsof -i :8080                    # Process on port
sudo fuser 8080/tcp                   # PID on port
```

##### [Firewall](https://devopsil.com/articles/2026-03-23-linux-networking-cheat-sheet#firewall)

```
# iptables
sudo iptables -L -n -v               # List rules
sudo iptables -L -n -v -t nat        # NAT table
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -A INPUT -s 203.0.113.5 -j DROP

# nftables
sudo nft list ruleset

# firewalld
sudo firewall-cmd --list-all
sudo firewall-cmd --add-port=8080/tcp --permanent && sudo firewall-cmd --reload
```

##### [Packet Capture](https://devopsil.com/articles/2026-03-23-linux-networking-cheat-sheet#packet-capture)

```
sudo tcpdump -i eth0                  # All traffic on interface
sudo tcpdump -i eth0 host 10.0.0.5 and port 443
sudo tcpdump -i eth0 -w capture.pcap # Write to file
sudo tcpdump -i eth0 -c 100          # Capture 100 packets
sudo tcpdump -i any port 53          # DNS traffic only
```

##### [Bandwidth](https://devopsil.com/articles/2026-03-23-linux-networking-cheat-sheet#bandwidth)

```
iperf3 -s                             # Server side
iperf3 -c 10.0.0.5                   # Client side
nload eth0                            # Real-time throughput
```

##### [Quick Reference](https://devopsil.com/articles/2026-03-23-linux-networking-cheat-sheet#quick-reference)

| Task | Command |
| --- | --- |
| Public IP | `curl -s ifconfig.me` |
| Hostname | `hostnamectl` |
| Scan remote ports | `nmap -Pn 10.0.0.5` |
| SSL cert dates | `openssl s_client -connect example.com:443 </dev/null 2>/dev/null | openssl x509 -noout -dates` |
| MAC address | `ip link show eth0 | awk '/ether/{print $2}'` |
| Flush DNS cache | `sudo resolvectl flush-caches` |
