## _Chapter 02: Linux Networking & Firewall: Configuration and Troubleshooting_
###### Published Time: 2026-03-23

Network configuration and firewall management are foundational skills for any systems engineer, DevOps practitioner, or cloud architect. Whether you are hardening a production server, debugging a mysterious connectivity outage at 3 AM, setting up a NAT gateway for a private subnet, or architecting a bastion host for secure access, you need a deep understanding of how Linux handles networking from interface configuration down to packet filtering. This guide is a comprehensive reference covering every tool and technique you will reach for in production environments.

<br><br>
### [Legacy vs Modern Networking Tools](https://devopsil.com/articles/linux-networking-firewall#legacy-vs-modern-networking-tools)

Before diving in, it is important to understand that Linux networking tooling has undergone a major transition. The old `net-tools` package (`ifconfig`, `route`, `netstat`, `arp`) is deprecated in favor of the `iproute2` suite and `ss`. Here is a mapping:

| Legacy Tool | Modern Replacement | Purpose |
| --- | --- | --- |
| `ifconfig` | `ip addr`, `ip link` | Interface configuration |
| `route` | `ip route` | Routing table management |
| `netstat` | `ss` | Socket and connection statistics |
| `arp` | `ip neigh` | ARP / neighbor table |
| `ifup` / `ifdown` | `nmcli`, `netplan apply` | Bringing interfaces up/down |
| `iptables` | `nft` (nftables) | Packet filtering |

The legacy tools still work on many systems, but they do not support newer kernel features such as network namespaces, policy routing, or multiple routing tables. Always prefer the modern equivalents.

<br><br>
### [Network Interface Configuration with the ip Command](https://devopsil.com/articles/linux-networking-firewall#network-interface-configuration-with-the-ip-command)

The `ip` command from the `iproute2` package is the Swiss Army knife for Linux networking. It has several subcommands, each managing a different networking layer.

##### [ip addr -- Address Management](https://devopsil.com/articles/linux-networking-firewall#ip-addr----address-management)

```
# Show all interfaces with their IP addresses
ip addr show
ip a                               # Short form

# Show only a specific interface
ip addr show dev eth0

# Show only IPv4 addresses
ip -4 addr show

# Show only IPv6 addresses
ip -6 addr show

# Add an IP address to an interface
sudo ip addr add 10.0.1.50/24 dev eth0

# Add a secondary IP (alias) to the same interface
sudo ip addr add 10.0.1.51/24 dev eth0 label eth0:1

# Remove an IP address
sudo ip addr del 10.0.1.50/24 dev eth0

# Flush all addresses from an interface
sudo ip addr flush dev eth0
```

##### [ip link -- Link Layer Management](https://devopsil.com/articles/linux-networking-firewall#ip-link----link-layer-management)

```
# Show all links with MAC addresses, MTU, and state
ip link show

# Bring an interface up or down
sudo ip link set eth0 up
sudo ip link set eth0 down

# Change the MTU (useful for jumbo frames in data center networks)
sudo ip link set eth0 mtu 9000

# Change MAC address (requires interface to be down)
sudo ip link set eth0 down
sudo ip link set eth0 address 02:42:ac:11:00:02
sudo ip link set eth0 up

# Create a VLAN sub-interface
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip addr add 10.10.100.1/24 dev eth0.100
sudo ip link set eth0.100 up
```

##### [ip route -- Routing Table](https://devopsil.com/articles/linux-networking-firewall#ip-route----routing-table)

```
# Display the routing table
ip route show
ip r                               # Short form

# Add a static route to a subnet via a gateway
sudo ip route add 10.0.2.0/24 via 10.0.1.1 dev eth0

# Add a default gateway
sudo ip route add default via 10.0.1.1

# Replace an existing route (add or update)
sudo ip route replace 10.0.2.0/24 via 10.0.1.254

# Delete a route
sudo ip route del 10.0.2.0/24

# Show which route a specific destination would use
ip route get 8.8.8.8

# Add a route with a specific metric (priority)
sudo ip route add 10.0.3.0/24 via 10.0.1.1 metric 100
```

##### [ip neigh -- ARP / Neighbor Table](https://devopsil.com/articles/linux-networking-firewall#ip-neigh----arp--neighbor-table)

```
# Show the ARP table (neighbor cache)
ip neigh show

# Add a static ARP entry
sudo ip neigh add 10.0.1.100 lladdr 00:11:22:33:44:55 dev eth0

# Delete an ARP entry
sudo ip neigh del 10.0.1.100 dev eth0

# Flush all neighbor entries for an interface
sudo ip neigh flush dev eth0
```

<br><br>
### [NetworkManager and nmcli](https://devopsil.com/articles/linux-networking-firewall#networkmanager-and-nmcli)

NetworkManager is the default network management daemon on RHEL, Fedora, CentOS, and many desktop distributions. Its CLI tool `nmcli` is essential for server administration.

```
# Show overall NetworkManager status
nmcli general status

# List all connections (active and inactive)
nmcli connection show

# Show active connections only
nmcli connection show --active

# Show device status
nmcli device status

# Show detailed info for a device
nmcli device show eth0
```

##### [Creating and Managing Connections](https://devopsil.com/articles/linux-networking-firewall#creating-and-managing-connections)

```
# Create a new static IP connection
nmcli connection add \
  con-name "prod-static" \
  type ethernet \
  ifname eth0 \
  ipv4.addresses 10.0.1.50/24 \
  ipv4.gateway 10.0.1.1 \
  ipv4.dns "8.8.8.8,8.8.4.4" \
  ipv4.dns-search "example.com,internal.example.com" \
  ipv4.method manual

# Create a DHCP connection
nmcli connection add \
  con-name "dhcp-eth0" \
  type ethernet \
  ifname eth0 \
  ipv4.method auto

# Modify an existing connection
nmcli connection modify "prod-static" ipv4.addresses "10.0.1.51/24"
nmcli connection modify "prod-static" +ipv4.dns "1.1.1.1"

# Activate a connection
nmcli connection up "prod-static"

# Deactivate a connection
nmcli connection down "prod-static"

# Delete a connection
nmcli connection delete "prod-static"

# Reload connection files from disk
nmcli connection reload
```

##### [Bond and VLAN with nmcli](https://devopsil.com/articles/linux-networking-firewall#bond-and-vlan-with-nmcli)

```
# Create a bond master
nmcli connection add type bond con-name bond0 ifname bond0 \
  bond.options "mode=802.3ad,miimon=100,lacp_rate=fast"

# Add slave interfaces
nmcli connection add type ethernet con-name bond0-slave1 \
  ifname eth0 master bond0
nmcli connection add type ethernet con-name bond0-slave2 \
  ifname eth1 master bond0

# Create a VLAN on top of the bond
nmcli connection add type vlan con-name vlan200 \
  ifname bond0.200 dev bond0 id 200 \
  ipv4.addresses 10.20.0.1/24 ipv4.method manual
```

<br><br>
### [Netplan Configuration (Ubuntu)](https://devopsil.com/articles/linux-networking-firewall#netplan-configuration-ubuntu)

Ubuntu 18.04 and later use Netplan as the network configuration abstraction layer. Netplan files live in `/etc/netplan/` and use YAML syntax. Netplan renders configuration to either `systemd-networkd` or `NetworkManager` as the backend.

##### [Static IP Configuration](https://devopsil.com/articles/linux-networking-firewall#static-ip-configuration)

```
# /etc/netplan/01-static.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 10.0.1.50/24
        - 10.0.1.51/24    # Secondary IP
      routes:
        - to: default
          via: 10.0.1.1
        - to: 10.0.2.0/24
          via: 10.0.1.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
        search:
          - example.com
          - internal.example.com
      mtu: 1500
```

##### [Bonding Configuration](https://devopsil.com/articles/linux-networking-firewall#bonding-configuration)

```
# /etc/netplan/02-bond.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
    eth1:
      dhcp4: false
  bonds:
    bond0:
      interfaces:
        - eth0
        - eth1
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        mii-monitor-interval: 100
      addresses:
        - 10.0.1.50/24
      routes:
        - to: default
          via: 10.0.1.1
```

##### [VLAN Configuration](https://devopsil.com/articles/linux-networking-firewall#vlan-configuration)

```
# /etc/netplan/03-vlan.yaml
network:
  version: 2
  vlans:
    vlan100:
      id: 100
      link: eth0
      addresses:
        - 10.10.100.5/24
      routes:
        - to: 10.10.0.0/16
          via: 10.10.100.1
```

##### [Applying Netplan](https://devopsil.com/articles/linux-networking-firewall#applying-netplan)

```
# Validate the configuration
sudo netplan generate

# Apply with automatic rollback (120 seconds to confirm)
sudo netplan try

# Apply permanently
sudo netplan apply

# Debug: see what systemd-networkd would receive
sudo netplan generate --mapping
```

<br><br>
### [DNS Configuration](https://devopsil.com/articles/linux-networking-firewall#dns-configuration)

##### [/etc/resolv.conf](https://devopsil.com/articles/linux-networking-firewall#etcresolvconf)

The traditional DNS resolver configuration file:

```
nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 1.1.1.1
search example.com internal.example.com
options timeout:2 attempts:3 rotate
```

The `rotate` option distributes queries across nameservers instead of always hitting the first one. The `timeout` and `attempts` options control retry behavior for slow or unreliable DNS servers.

##### [systemd-resolved](https://devopsil.com/articles/linux-networking-firewall#systemd-resolved)

On modern systems, `/etc/resolv.conf` is often a symlink managed by `systemd-resolved`. This service provides a local caching DNS stub resolver listening on `127.0.0.53`.

```
# Check if resolv.conf is a symlink
ls -la /etc/resolv.conf

# View resolved configuration and status
resolvectl status

# Query DNS through resolved
resolvectl query example.com

# Set DNS for an interface
sudo resolvectl dns eth0 8.8.8.8 8.8.4.4
sudo resolvectl domain eth0 example.com

# Flush the DNS cache
sudo resolvectl flush-caches

# Show cache statistics
sudo resolvectl statistics
```

If you need to bypass `systemd-resolved` and manage `/etc/resolv.conf` directly (common on servers), remove the symlink and create a static file:

```
sudo rm /etc/resolv.conf
sudo tee /etc/resolv.conf > /dev/null <<DNSEOF
nameserver 8.8.8.8
nameserver 1.1.1.1
search example.com
DNSEOF
```

##### [/etc/hosts](https://devopsil.com/articles/linux-networking-firewall#etchosts)

Static hostname-to-IP mappings that are resolved before DNS queries:

```
127.0.0.1       localhost
::1             localhost ip6-localhost

# Internal services
10.0.1.10       db-primary.internal db-primary
10.0.1.11       db-replica.internal db-replica
10.0.1.20       redis.internal redis
10.0.1.30       kafka-01.internal kafka-01
```

Resolution order is controlled by `/etc/nsswitch.conf`:

```
hosts: files dns myhostname
```

This means `/etc/hosts` is checked first, then DNS, then the local hostname fallback.

<br><br>
### [Firewall: iptables in Depth](https://devopsil.com/articles/linux-networking-firewall#firewall-iptables-in-depth)

iptables is the traditional Linux packet filtering framework. Even though nftables is the modern successor, iptables remains widely deployed and understanding its architecture is essential because firewalld and UFW both translate their rules into iptables or nftables under the hood.

##### [Architecture: Tables, Chains, and Rules](https://devopsil.com/articles/linux-networking-firewall#architecture-tables-chains-and-rules)

iptables organizes rules into **tables**, each containing **chains**:

| Table | Purpose | Default Chains |
| --- | --- | --- |
| `filter` | Packet filtering (accept/drop) | INPUT, FORWARD, OUTPUT |
| `nat` | Network Address Translation | PREROUTING, INPUT, OUTPUT, POSTROUTING |
| `mangle` | Packet header modification | PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING |
| `raw` | Bypass connection tracking | PREROUTING, OUTPUT |
| `security` | SELinux security marking | INPUT, FORWARD, OUTPUT |

The **filter** table is the default and most commonly used. When you run `iptables -A INPUT`, you are implicitly working with the filter table.

Each rule in a chain specifies matching criteria and a **target** (action):

| Target | Behavior |
| --- | --- |
| `ACCEPT` | Allow the packet through |
| `DROP` | Silently discard the packet |
| `REJECT` | Discard and send an error response (ICMP or TCP RST) |
| `LOG` | Log the packet to syslog, then continue processing |
| `DNAT` | Rewrite destination address (nat table) |
| `SNAT` | Rewrite source address (nat table) |
| `MASQUERADE` | SNAT using the outgoing interface address (nat table) |

##### [Building a Production Ruleset](https://devopsil.com/articles/linux-networking-firewall#building-a-production-ruleset)

```
# List all rules with line numbers, numeric output, verbose counters
sudo iptables -L -n -v --line-numbers

# List rules in a specific table
sudo iptables -t nat -L -n -v --line-numbers

# --- Step 1: Set default policies ---
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# --- Step 2: Allow loopback (critical for local services) ---
sudo iptables -A INPUT -i lo -j ACCEPT

# --- Step 3: Allow established and related connections ---
# This is the most important rule. Without it, return traffic is blocked.
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# --- Step 4: Drop invalid packets ---
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# --- Step 5: Allow ICMP (ping) with rate limiting ---
sudo iptables -A INPUT -p icmp --icmp-type echo-request \
  -m limit --limit 5/second --limit-burst 10 -j ACCEPT

# --- Step 6: Allow SSH from management subnet only ---
sudo iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/24 -j ACCEPT

# --- Step 7: Allow web traffic from anywhere ---
sudo iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT

# --- Step 8: Allow monitoring (Node Exporter) from Prometheus server ---
sudo iptables -A INPUT -p tcp --dport 9100 -s 10.0.5.10/32 -j ACCEPT

# --- Step 9: Rate limit new connections to prevent SYN floods ---
sudo iptables -A INPUT -p tcp --syn -m limit --limit 25/second \
  --limit-burst 50 -j ACCEPT

# --- Step 10: Log dropped packets for debugging ---
sudo iptables -A INPUT -m limit --limit 5/min -j LOG \
  --log-prefix "iptables-dropped: " --log-level 4
```

##### [Rule Management](https://devopsil.com/articles/linux-networking-firewall#rule-management)

```
# Insert a rule at a specific position (position 1 means top)
sudo iptables -I INPUT 1 -p tcp --dport 8080 -j ACCEPT

# Delete a rule by line number
sudo iptables -D INPUT 3

# Delete a rule by specification
sudo iptables -D INPUT -p tcp --dport 8080 -j ACCEPT

# Replace a rule at a specific position
sudo iptables -R INPUT 3 -p tcp --dport 8443 -j ACCEPT

# Flush all rules in a chain (careful on remote servers!)
sudo iptables -F INPUT

# Flush all rules in all chains
sudo iptables -F

# Delete all user-defined chains
sudo iptables -X

# Zero all counters
sudo iptables -Z
```

##### [NAT and Masquerading](https://devopsil.com/articles/linux-networking-firewall#nat-and-masquerading)

NAT is configured in the `nat` table. This is essential for building gateways, load balancers, and container networks.

```
# Enable IP forwarding (required for any routing/NAT)
sudo sysctl -w net.ipv4.ip_forward=1

# Make it permanent
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/99-forwarding.conf
sudo sysctl --system

# --- MASQUERADE: Dynamic SNAT for outbound traffic ---
# Use this when the public IP might change (DHCP, cloud instances)
sudo iptables -t nat -A POSTROUTING -s 10.0.1.0/24 -o eth0 -j MASQUERADE

# --- SNAT: Static source NAT (when public IP is fixed) ---
sudo iptables -t nat -A POSTROUTING -s 10.0.1.0/24 -o eth0 \
  -j SNAT --to-source 203.0.113.50

# --- DNAT: Port forwarding from public to internal ---
# Forward port 8080 on the gateway to port 80 on an internal web server
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 \
  -j DNAT --to-destination 10.0.1.50:80

# Allow the forwarded traffic through
sudo iptables -A FORWARD -i eth0 -o eth1 -p tcp \
  -d 10.0.1.50 --dport 80 -m conntrack --ctstate NEW -j ACCEPT

# Allow return traffic for forwarded connections
sudo iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

##### [Persisting iptables Rules](https://devopsil.com/articles/linux-networking-firewall#persisting-iptables-rules)

Rules are in-memory only and are lost on reboot unless explicitly saved:

```
# --- Debian/Ubuntu ---
sudo apt install iptables-persistent netfilter-persistent
sudo netfilter-persistent save       # Saves to /etc/iptables/rules.v4 and rules.v6
sudo netfilter-persistent reload     # Reload saved rules

# --- RHEL/CentOS ---
sudo iptables-save > /etc/sysconfig/iptables
sudo ip6tables-save > /etc/sysconfig/ip6tables
sudo systemctl enable iptables
sudo systemctl restart iptables

# --- Manual save/restore (any distro) ---
sudo iptables-save > /root/iptables-backup.rules
sudo iptables-restore < /root/iptables-backup.rules
```

<br><br>
### [nftables: The Modern Replacement](https://devopsil.com/articles/linux-networking-firewall#nftables-the-modern-replacement)

nftables is the successor to iptables, porting all functionality into a single coherent framework. It uses the `nft` command and offers a cleaner syntax, better performance through set-based matching, and atomic rule replacement.

##### [Key Differences from iptables](https://devopsil.com/articles/linux-networking-firewall#key-differences-from-iptables)

| Feature | iptables | nftables |
| --- | --- | --- |
| Separate tools per protocol | `iptables`, `ip6tables`, `arptables`, `ebtables` | Single `nft` binary |
| Rule update | One rule at a time | Atomic ruleset replacement |
| Matching | Linear chain traversal | Sets and maps for O(1) lookups |
| Syntax | Flag-based CLI | Structured, readable syntax |

##### [nftables Ruleset Configuration](https://devopsil.com/articles/linux-networking-firewall#nftables-ruleset-configuration)

The primary configuration file is `/etc/nftables.conf`:

```
#!/usr/sbin/nft -f
flush ruleset

# Define variables for reuse
define LAN_NET = 10.0.1.0/24
define MGMT_NET = 10.0.0.0/24
define WEB_PORTS = { 80, 443 }
define MONITOR_PORTS = { 9090, 9100, 3000 }

table inet filter {
    # Named set for whitelisted IPs
    set trusted_ips {
        type ipv4_addr
        flags interval
        elements = { 10.0.0.0/24, 10.0.5.0/24, 192.168.1.0/24 }
    }

    chain input {
        type filter hook input priority 0; policy drop;

        # Connection tracking
        ct state established,related accept
        ct state invalid drop

        # Loopback
        iifname "lo" accept

        # ICMP
        ip protocol icmp icmp type echo-request \
            limit rate 5/second burst 10 packets accept

        # SSH from management network only
        tcp dport 22 ip saddr $MGMT_NET accept

        # Web traffic from anywhere
        tcp dport $WEB_PORTS accept

        # Monitoring from trusted IPs
        tcp dport $MONITOR_PORTS ip saddr @trusted_ips accept

        # Log and drop everything else
        log prefix "nft-drop: " counter drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
        ct state established,related accept
        ct state invalid drop

        # Allow LAN to reach the internet
        iifname "eth1" oifname "eth0" ip saddr $LAN_NET accept
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}

# NAT table
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat;

        # Port forward 8080 to internal web server
        tcp dport 8080 dnat to 10.0.1.50:80
    }

    chain postrouting {
        type nat hook postrouting priority srcnat;

        # Masquerade outbound traffic from LAN
        oifname "eth0" ip saddr 10.0.1.0/24 masquerade
    }
}
```

##### [Managing nftables](https://devopsil.com/articles/linux-networking-firewall#managing-nftables)

```
# Load the configuration file
sudo nft -f /etc/nftables.conf

# List the entire ruleset
sudo nft list ruleset

# List a specific table
sudo nft list table inet filter

# Add a rule interactively
sudo nft add rule inet filter input tcp dport 8443 accept

# Insert a rule at the beginning of a chain
sudo nft insert rule inet filter input position 0 tcp dport 2222 accept

# Delete a rule by handle (find handle with -a flag)
sudo nft -a list chain inet filter input
sudo nft delete rule inet filter input handle 15

# Add an element to a named set
sudo nft add element inet filter trusted_ips { 172.16.0.0/16 }

# Enable and start the service
sudo systemctl enable nftables
sudo systemctl start nftables
```

<br><br>
### [firewalld (RHEL/CentOS/Fedora)](https://devopsil.com/articles/linux-networking-firewall#firewalld-rhelcentosfedora)

firewalld provides a dynamic firewall management daemon with the concept of **zones**. Each zone defines a trust level, and network interfaces are assigned to zones.

##### [Zone Concepts](https://devopsil.com/articles/linux-networking-firewall#zone-concepts)

| Zone | Default Behavior |
| --- | --- |
| `drop` | Drop all incoming, no reply |
| `block` | Reject all incoming with ICMP error |
| `public` | Untrusted networks, selective incoming |
| `external` | External NAT gateway |
| `dmz` | Demilitarized zone, limited incoming |
| `work` | Trusted work network |
| `home` | Home network |
| `internal` | Internal network |
| `trusted` | Accept all traffic |

##### [Common firewall-cmd Operations](https://devopsil.com/articles/linux-networking-firewall#common-firewall-cmd-operations)

```
# Check daemon status
sudo firewall-cmd --state

# List all zones with their configurations
sudo firewall-cmd --list-all-zones

# Show active zones and their interfaces
sudo firewall-cmd --get-active-zones

# List everything in the public zone
sudo firewall-cmd --zone=public --list-all

# Add a service (runtime only)
sudo firewall-cmd --zone=public --add-service=http

# Add a service permanently
sudo firewall-cmd --zone=public --add-service=http --permanent
sudo firewall-cmd --zone=public --add-service=https --permanent

# Open a specific port
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
sudo firewall-cmd --zone=public --add-port=5000-5100/tcp --permanent

# Remove a service or port
sudo firewall-cmd --zone=public --remove-service=http --permanent
sudo firewall-cmd --zone=public --remove-port=8080/tcp --permanent

# Assign an interface to a zone
sudo firewall-cmd --zone=internal --change-interface=eth1 --permanent

# Reload to apply permanent changes
sudo firewall-cmd --reload
```

##### [Rich Rules](https://devopsil.com/articles/linux-networking-firewall#rich-rules)

Rich rules provide fine-grained control similar to raw iptables rules:

```
# Allow MySQL from a specific subnet
sudo firewall-cmd --zone=public --add-rich-rule='
  rule family="ipv4"
  source address="10.0.1.0/24"
  port port="3306" protocol="tcp"
  accept' --permanent

# Rate limit SSH connections
sudo firewall-cmd --zone=public --add-rich-rule='
  rule family="ipv4"
  service name="ssh"
  accept
  limit value="10/m"' --permanent

# Log and drop traffic from a malicious IP
sudo firewall-cmd --zone=public --add-rich-rule='
  rule family="ipv4"
  source address="198.51.100.99"
  log prefix="blocked-attacker: " level="warning"
  drop' --permanent

# Port forwarding: forward port 80 to internal server
sudo firewall-cmd --zone=public --add-forward-port=\
port=80:proto=tcp:toaddr=10.0.1.50:toport=80 --permanent

# Enable masquerading for NAT
sudo firewall-cmd --zone=external --add-masquerade --permanent

sudo firewall-cmd --reload
```

##### [UFW (Uncomplicated Firewall) for Ubuntu](https://devopsil.com/articles/linux-networking-firewall#ufw-uncomplicated-firewall-for-ubuntu)

UFW is Ubuntu's simplified firewall interface. It wraps iptables/nftables with a human-friendly CLI.

```
# Enable UFW (be sure SSH is allowed first!)
sudo ufw allow ssh
sudo ufw enable

# Check status and rules
sudo ufw status verbose
sudo ufw status numbered

# Allow by service name
sudo ufw allow http
sudo ufw allow https

# Allow a specific port
sudo ufw allow 8080/tcp

# Allow a port range
sudo ufw allow 5000:5100/tcp

# Allow from a specific IP
sudo ufw allow from 10.0.1.0/24

# Allow from a specific IP to a specific port
sudo ufw allow from 10.0.0.0/24 to any port 22 proto tcp

# Deny a specific IP
sudo ufw deny from 198.51.100.99

# Delete a rule by number
sudo ufw status numbered
sudo ufw delete 3

# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Enable logging
sudo ufw logging on

# Rate limit SSH (blocks IPs with 6+ connections in 30 seconds)
sudo ufw limit ssh

# Reset all rules
sudo ufw reset
```

For more complex rules, you can edit the UFW configuration files directly at `/etc/ufw/before.rules` (processed before user rules) and `/etc/ufw/after.rules`.

<br><br>
### [TCP/IP Troubleshooting Tools](https://devopsil.com/articles/linux-networking-firewall#tcpip-troubleshooting-tools)

##### [Connectivity Testing](https://devopsil.com/articles/linux-networking-firewall#connectivity-testing)

```
# Basic ICMP connectivity test
ping -c 4 google.com
ping -c 4 -W 2 10.0.1.1           # 2-second timeout per packet

# IPv6 ping
ping6 -c 4 ::1

# Trace the route packets take
traceroute google.com
traceroute -n -T -p 443 google.com  # TCP traceroute on port 443 (bypasses ICMP blocks)
tracepath google.com                 # No root required

# mtr: continuous combined ping and traceroute
mtr google.com                       # Interactive mode
mtr -r -c 100 google.com            # Report mode, 100 cycles
mtr -T -P 443 google.com            # TCP mode on port 443
```

##### [DNS Troubleshooting](https://devopsil.com/articles/linux-networking-firewall#dns-troubleshooting)

```
# Full DNS query with all sections
dig example.com

# Short output (just the answer)
dig +short example.com

# Query a specific DNS server
dig @8.8.8.8 example.com

# Query specific record types
dig example.com MX                  # Mail servers
dig example.com NS                  # Name servers
dig example.com TXT                 # TXT records (SPF, DKIM)
dig example.com AAAA                # IPv6 addresses

# Trace the full DNS delegation path
dig +trace example.com

# Reverse DNS lookup
dig -x 8.8.8.8

# nslookup (simpler alternative)
nslookup example.com
nslookup example.com 8.8.8.8       # Query specific server
nslookup -type=MX example.com
```

##### [Port and Connection Analysis with ss](https://devopsil.com/articles/linux-networking-firewall#port-and-connection-analysis-with-ss)

```
# TCP listening sockets with process info
ss -tlnp

# UDP listening sockets
ss -ulnp

# All established TCP connections
ss -tn state established

# Connection summary statistics
ss -s

# Connections to a specific destination
ss -tn dst 10.0.1.50

# Connections on a specific port
ss -tn sport = :443

# Show timer information (useful for debugging stuck connections)
ss -tn -o state established

# Count connections per state
ss -tan | tail -n +2 | awk '{print $1}' | sort | uniq -c | sort -rn
```

##### [Packet Capture with tcpdump](https://devopsil.com/articles/linux-networking-firewall#packet-capture-with-tcpdump)

```
# Capture all traffic on an interface
sudo tcpdump -i eth0

# Capture only TCP traffic on port 80
sudo tcpdump -i eth0 tcp port 80

# Capture traffic to/from a specific host
sudo tcpdump -i eth0 host 10.0.1.50

# Capture only SYN packets (new connection attempts)
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'

# Capture DNS queries and responses
sudo tcpdump -i eth0 port 53 -vv

# Show packet contents in ASCII and hex
sudo tcpdump -i eth0 -A -X port 80

# Write capture to a file for Wireshark analysis
sudo tcpdump -i eth0 -w /tmp/capture.pcap -c 10000

# Read a saved capture
sudo tcpdump -r /tmp/capture.pcap

# Capture with a complex filter: HTTP traffic from a specific subnet
sudo tcpdump -i eth0 'src net 10.0.1.0/24 and (dst port 80 or dst port 443)'
```

##### [HTTP Testing and Port Scanning](https://devopsil.com/articles/linux-networking-firewall#http-testing-and-port-scanning)

```
# HTTP response headers
curl -I https://example.com

# Verbose output showing TLS handshake
curl -v https://example.com

# Timing breakdown of a request
curl -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTotal: %{time_total}s\nHTTP Code: %{http_code}\n" \
  -o /dev/null -s https://example.com

# Download a file with progress
wget -O /tmp/file.tar.gz https://example.com/file.tar.gz

# Test TCP connectivity with nc (netcat)
nc -zv 10.0.1.50 80                 # Test if port 80 is open
nc -zv 10.0.1.50 1-1024             # Scan a port range

# Listen on a port (useful for testing)
nc -l -p 8080                       # Listen on port 8080

# Send data through a port
echo "test" | nc 10.0.1.50 8080

# ncat (nmap's netcat) with SSL support
ncat --ssl -v example.com 443
```

<br><br>
### [Network Bonding and VLANs](https://devopsil.com/articles/linux-networking-firewall#network-bonding-and-vlans)

Network bonding (also called NIC teaming) combines multiple network interfaces into a single logical interface for redundancy or increased throughput.

##### [Bonding Modes](https://devopsil.com/articles/linux-networking-firewall#bonding-modes)

| Mode | Name | Description |
| --- | --- | --- |
| 0 | balance-rr | Round-robin across slaves |
| 1 | active-backup | One active, others standby |
| 2 | balance-xor | XOR-based hash distribution |
| 3 | broadcast | Transmit on all slaves |
| 4 | 802.3ad | LACP (Link Aggregation Control Protocol) |
| 5 | balance-tlb | Adaptive transmit load balancing |
| 6 | balance-alb | Adaptive load balancing (tx and rx) |

Mode 1 (active-backup) is the simplest and most common for redundancy. Mode 4 (802.3ad / LACP) requires switch support but provides both redundancy and increased throughput.

##### [Manual Bond Configuration](https://devopsil.com/articles/linux-networking-firewall#manual-bond-configuration)

```
# Load the bonding module
sudo modprobe bonding

# Create bond interface
sudo ip link add bond0 type bond mode 802.3ad

# Set bond parameters
sudo ip link set bond0 type bond miimon 100 lacp_rate fast

# Add slave interfaces (must be down first)
sudo ip link set eth0 down
sudo ip link set eth1 down
sudo ip link set eth0 master bond0
sudo ip link set eth1 master bond0

# Configure IP and bring up
sudo ip addr add 10.0.1.50/24 dev bond0
sudo ip link set bond0 up

# Verify bond status
cat /proc/net/bonding/bond0
```

##### [VLAN Tagging](https://devopsil.com/articles/linux-networking-firewall#vlan-tagging)

```
# Load the 8021q module
sudo modprobe 8021q

# Create a VLAN interface
sudo ip link add link eth0 name eth0.100 type vlan id 100

# Assign an IP and bring it up
sudo ip addr add 10.10.100.5/24 dev eth0.100
sudo ip link set eth0.100 up

# Verify VLAN configuration
cat /proc/net/vlan/eth0.100

# Stack VLANs on a bond
sudo ip link add link bond0 name bond0.200 type vlan id 200
sudo ip addr add 10.20.0.5/24 dev bond0.200
sudo ip link set bond0.200 up
```

<br><br>
### [SSH Hardening](https://devopsil.com/articles/linux-networking-firewall#ssh-hardening)

SSH is the primary remote access method for Linux servers and a top target for attackers. Hardening SSH is one of the first things to do on any server.

##### [Key-Based Authentication](https://devopsil.com/articles/linux-networking-firewall#key-based-authentication)

```
# Generate an Ed25519 key pair (recommended over RSA)
ssh-keygen -t ed25519 -C "admin@example.com"

# Copy public key to the server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# Or manually append to authorized_keys
cat ~/.ssh/id_ed25519.pub | ssh user@server \
  'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

##### [sshd_config Hardening](https://devopsil.com/articles/linux-networking-firewall#sshd_config-hardening)

Edit `/etc/ssh/sshd_config` with these production-recommended settings:

```
# Disable password authentication (keys only)
PasswordAuthentication no
ChallengeResponseAuthentication no

# Disable root login
PermitRootLogin no

# Change the default port (obscurity, not security, but reduces noise)
Port 2222

# Restrict to specific users or groups
AllowUsers deploy admin
AllowGroups ssh-users

# Limit authentication attempts
MaxAuthTries 3
MaxSessions 5

# Disable unused authentication methods
KbdInteractiveAuthentication no
GSSAPIAuthentication no

# Set idle timeout (disconnect after 5 minutes of inactivity)
ClientAliveInterval 300
ClientAliveCountMax 0

# Use only strong ciphers and algorithms
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# Disable X11 and agent forwarding if not needed
X11Forwarding no
AllowAgentForwarding no
```

After editing, validate and restart:

```
sudo sshd -t                        # Test configuration syntax
sudo systemctl restart sshd
```

##### [fail2ban for Brute Force Protection](https://devopsil.com/articles/linux-networking-firewall#fail2ban-for-brute-force-protection)

```
# Install fail2ban
sudo apt install fail2ban            # Debian/Ubuntu
sudo dnf install fail2ban            # RHEL/Fedora

# Create a local configuration override
sudo tee /etc/fail2ban/jail.local > /dev/null <<F2BEOF
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5
banaction = iptables-multiport

[sshd]
enabled = true
port = 2222
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400
F2BEOF

# Start and enable
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Check status
sudo fail2ban-client status sshd

# Unban an IP
sudo fail2ban-client set sshd unbanip 203.0.113.50
```

<br><br>
### [Practical Scenarios](https://devopsil.com/articles/linux-networking-firewall#practical-scenarios)

##### [Scenario 1: Setting Up a Bastion Host](https://devopsil.com/articles/linux-networking-firewall#scenario-1-setting-up-a-bastion-host)

A bastion host (jump box) is the single entry point into a private network. All SSH traffic to internal servers must pass through it.

```
#!/bin/bash
# bastion-setup.sh -- Configure a bastion host
set -euo pipefail

INTERNAL_NET="10.0.1.0/24"
MGMT_PORT=2222

# --- Firewall rules ---
iptables -F
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Loopback and established connections
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# SSH from the internet on non-standard port
iptables -A INPUT -p tcp --dport $MGMT_PORT -j ACCEPT

# Allow forwarding of SSH traffic to internal servers
iptables -A FORWARD -i eth0 -o eth1 -p tcp --dport 22 -d $INTERNAL_NET \
  -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# ICMP for diagnostics
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Log drops
iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "bastion-drop: "

netfilter-persistent save
echo "Bastion firewall configured."
```

On the client side, use SSH ProxyJump to reach internal servers through the bastion:

```
# ~/.ssh/config
Host bastion
    HostName bastion.example.com
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

Host internal-*
    ProxyJump bastion
    User deploy

Host internal-web
    HostName 10.0.1.50

Host internal-db
    HostName 10.0.1.10
```

Then simply run:

```
ssh internal-web                     # Automatically tunnels through the bastion
```

##### [Scenario 2: Debugging Connectivity Issues](https://devopsil.com/articles/linux-networking-firewall#scenario-2-debugging-connectivity-issues)

When a service is unreachable, follow this systematic troubleshooting ladder:

```
# Step 1: Check if the interface is up and has an IP
ip addr show eth0
ip link show eth0

# Step 2: Check routing -- can we reach the gateway?
ip route show
ping -c 2 10.0.1.1

# Step 3: Check DNS resolution
dig +short example.com
resolvectl query example.com

# Step 4: Test TCP connectivity to the destination
nc -zv 10.0.1.50 80
curl -v --connect-timeout 5 http://10.0.1.50

# Step 5: Check if the service is actually listening
ss -tlnp | grep ':80'

# Step 6: Check firewall rules (is traffic being dropped?)
sudo iptables -L -n -v --line-numbers | grep -i drop
sudo nft list ruleset 2>/dev/null
sudo journalctl -k --since "5 minutes ago" | grep -i drop

# Step 7: Capture packets to see what is happening on the wire
sudo tcpdump -i eth0 -n host 10.0.1.50 and port 80 -c 20

# Step 8: Check for network namespaces (containers, pods)
sudo ip netns list
sudo ip netns exec my-namespace ss -tlnp
```

##### [Scenario 3: Configuring a NAT Gateway](https://devopsil.com/articles/linux-networking-firewall#scenario-3-configuring-a-nat-gateway)

A common architecture pattern is placing private instances behind a NAT gateway so they can reach the internet without public IPs.

```
#!/bin/bash
# nat-gateway-setup.sh
set -euo pipefail

# eth0 = public interface (internet-facing)
# eth1 = private interface (connected to 10.0.1.0/24)

PRIVATE_NET="10.0.1.0/24"
PUBLIC_IF="eth0"
PRIVATE_IF="eth1"

# Enable IP forwarding
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/99-nat-gateway.conf

# Flush existing rules
iptables -F
iptables -t nat -F

# Default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Loopback and established
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# SSH from private network
iptables -A INPUT -i $PRIVATE_IF -p tcp --dport 22 -s $PRIVATE_NET -j ACCEPT

# ICMP
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# --- Forwarding rules ---
# Allow private network to reach the internet
iptables -A FORWARD -i $PRIVATE_IF -o $PUBLIC_IF -s $PRIVATE_NET \
  -m conntrack --ctstate NEW -j ACCEPT

# Allow return traffic
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# --- NAT ---
iptables -t nat -A POSTROUTING -s $PRIVATE_NET -o $PUBLIC_IF -j MASQUERADE

# Save rules
netfilter-persistent save

echo "NAT gateway configured."
echo "Private hosts should set their default gateway to this server's $PRIVATE_IF IP."
```

On private hosts, configure the default route to point to the NAT gateway:

```
# On private host 10.0.1.50
sudo ip route del default
sudo ip route add default via 10.0.1.1   # NAT gateway's private IP
```

Or in Netplan:

```
# /etc/netplan/01-private.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 10.0.1.50/24
      routes:
        - to: default
          via: 10.0.1.1
      nameservers:
        addresses:
          - 8.8.8.8
```

<br><br>
### [Key Takeaways](https://devopsil.com/articles/linux-networking-firewall#key-takeaways)

Linux networking is built on layers: interface configuration (`ip`, Netplan, `nmcli`), name resolution (DNS, `/etc/hosts`), and packet filtering (`iptables`, `nftables`, `firewalld`, `ufw`). Each layer has both legacy and modern tools, and production environments often mix them.

For troubleshooting, follow a systematic bottom-up approach. Start at the link layer (`ip link`), verify addressing (`ip addr`), check routing (`ip route`), test DNS (`dig`), inspect firewall rules, and finally capture packets with `tcpdump`. This layered methodology eliminates guesswork and gets you to the root cause faster.

When hardening servers, defense in depth is the principle to follow: restrict SSH access to key-based authentication through a bastion host, deploy fail2ban for brute force protection, use a host firewall with a default-deny policy, and keep only the minimum necessary ports open. Always test firewall changes carefully on remote servers. Use `iptables -I INPUT 1` to insert a temporary allow-all rule before making changes, or use `netplan try` and `at` commands to schedule automatic rollbacks. Locking yourself out of SSH is a rite of passage, but only once.
