### _Issue 01 - Fix Docker Container DNS Resolution Failures_
###### Published Time: 2026-03-30


#### [The Error: DNS Resolution Fails Inside Containers](https://devopsil.com/articles/2026-03-30-docker-dns-resolution-failure-fix#the-error-dns-resolution-fails-inside-containers)

Your containerized application cannot reach external services:

```
$ docker exec myapp curl https://api.example.com
curl: (6) Could not resolve host: api.example.com
```

Or with `nslookup`:

```
$ docker exec myapp nslookup google.com
;; connection timed out; no servers could be reached
```

The container has network connectivity but cannot translate hostnames to IP addresses.

#### [Root Cause](https://devopsil.com/articles/2026-03-30-docker-dns-resolution-failure-fix#root-cause)

[Docker](https://devopsil.com/tools/docker "Docker") containers rely on DNS configuration injected at startup. Failures happen because:

1.   **Host DNS uses `systemd-resolved` on 127.0.0.53** — this loopback address is unreachable from inside the container's network namespace
2.   **Docker's embedded DNS (127.0.0.11)** is only available on user-defined networks, not the default `bridge` network
3.   **Corporate firewall or VPN** blocks DNS traffic (port 53) from the Docker bridge interface
4.   **`/etc/resolv.conf` inside the container** points to an unreachable nameserver

#### [Step-by-Step Fix](https://devopsil.com/articles/2026-03-30-docker-dns-resolution-failure-fix#step-by-step-fix)

##### [1. Confirm the problem is DNS, not network](https://devopsil.com/articles/2026-03-30-docker-dns-resolution-failure-fix#1-confirm-the-problem-is-dns-not-network)

```
# Test raw connectivity (use a known IP)
docker exec myapp ping -c 2 8.8.8.8

# If ping works but DNS doesn't, it's a DNS issue
docker exec myapp cat /etc/resolv.conf
```

##### [2. Check what DNS server the container is using](https://devopsil.com/articles/2026-03-30-docker-dns-resolution-failure-fix#2-check-what-dns-server-the-container-is-using)

```
docker exec myapp cat /etc/resolv.conf
```

If you see `nameserver 127.0.0.53`, that is the `systemd-resolved` stub listener and is not reachable from inside Docker.

##### [3. Set an explicit DNS server for Docker](https://devopsil.com/articles/2026-03-30-docker-dns-resolution-failure-fix#3-set-an-explicit-dns-server-for-docker)

Edit `/etc/docker/daemon.json`:

```
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

Restart Docker:

```
sudo systemctl restart docker
```

This forces all containers to use Google's public DNS instead of inheriting the host's broken config.

##### [4. Or specify DNS per container](https://devopsil.com/articles/2026-03-30-docker-dns-resolution-failure-fix#4-or-specify-dns-per-container)

```
docker run --dns 8.8.8.8 --dns 8.8.4.4 myapp:latest
```

In Docker Compose:

```
services:
  myapp:
    image: myapp:latest
    dns:
      - 8.8.8.8
      - 8.8.4.4
```

##### [5. Use a user-defined network (recommended)](https://devopsil.com/articles/2026-03-30-docker-dns-resolution-failure-fix#5-use-a-user-defined-network-recommended)

Docker's embedded DNS server (127.0.0.11) only works on user-defined networks and provides container name resolution:

```
# Create a custom network
docker network create mynet

# Run containers on it
docker run --network mynet --name myapp myapp:latest
docker run --network mynet --name mydb postgres:16

# Now myapp can resolve "mydb" by name
docker exec myapp ping mydb
```

##### [6. Fix systemd-resolved on the host](https://devopsil.com/articles/2026-03-30-docker-dns-resolution-failure-fix#6-fix-systemd-resolved-on-the-host)

Instead of bypassing systemd-resolved, make it work with Docker:

```
# Expose resolved on a real interface
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/docker.conf <<EOF
[Resolve]
DNSStubListenerExtra=172.17.0.1
EOF

sudo systemctl restart systemd-resolved
```

Then set Docker to use the bridge gateway IP:

```
{
  "dns": ["172.17.0.1"]
}
```

##### [7. Check for iptables/firewall interference](https://devopsil.com/articles/2026-03-30-docker-dns-resolution-failure-fix#7-check-for-iptablesfirewall-interference)

```
# Verify Docker's iptables rules exist
sudo iptables -L DOCKER -n -v

# Check if port 53 traffic is being dropped
sudo iptables -L -n -v | grep 53

# If using UFW, allow Docker bridge traffic
sudo ufw allow in on docker0
```

#### [Prevention Tips](https://devopsil.com/articles/2026-03-30-docker-dns-resolution-failure-fix#prevention-tips)

*   **Always use user-defined networks** instead of the default bridge — you get embedded DNS, container name resolution, and better isolation.
*   **Set DNS in `daemon.json`** as a baseline so all containers have a working fallback.
*   **Include a DNS health check** in your container:
```
healthcheck:
  test: ["CMD", "nslookup", "google.com", "||", "exit", "1"]
  interval: 30s
  retries: 3
```
*   **On VPN/corporate networks**, use your internal DNS servers in `daemon.json` rather than public DNS.
*   **Test DNS in CI** — add a `nslookup` step early in pipelines to catch issues before they cascade.
