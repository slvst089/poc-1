### _Issue 02 - Fix Docker 'No Space Left on Device' Error_
###### Published Time: 2026-03-30


#### [The Error: No Space Left on Device](https://devopsil.com/articles/2026-03-30-docker-no-space-left-on-device-fix#the-error-no-space-left-on-device)

A build or container start fails with:

```
Error response from daemon: failed to mkdir /var/lib/docker/overlay2/...
no space left on device
```

Or during `docker build`:

```
write /var/lib/docker/tmp/docker-builder123456/layer.tar: no space left on device
```

[Docker](https://devopsil.com/tools/docker "Docker") stores images, containers, volumes, and build cache under `/var/lib/docker`. Over time, this accumulates gigabytes of unused data.

#### [Root Cause](https://devopsil.com/articles/2026-03-30-docker-no-space-left-on-device-fix#root-cause)

Docker does not automatically garbage-collect old resources. These pile up:

*   **Dangling images** — intermediate build layers with no tag
*   **Stopped containers** — exited containers still hold writable layers
*   **Unused volumes** — data volumes from deleted containers
*   **Build cache** — cached RUN layers from `docker build`

On CI servers this is especially severe because every pipeline run creates new layers.

#### [Step-by-Step Fix](https://devopsil.com/articles/2026-03-30-docker-no-space-left-on-device-fix#step-by-step-fix)

##### [1. Check what is consuming space](https://devopsil.com/articles/2026-03-30-docker-no-space-left-on-device-fix#1-check-what-is-consuming-space)

```
# Docker's built-in disk usage report
docker system df

# Detailed breakdown
docker system df -v

# Check the host filesystem
df -h /var/lib/docker
```

Example output:

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          45        3         12.8GB    11.2GB (87%)
Containers      62        1         3.4GB     3.3GB (97%)
Local Volumes   18        2         8.1GB     7.6GB (93%)
Build Cache     124                 5.2GB     5.2GB
```

##### [2. Run a safe cleanup (unused resources only)](https://devopsil.com/articles/2026-03-30-docker-no-space-left-on-device-fix#2-run-a-safe-cleanup-unused-resources-only)

```
# Remove stopped containers, dangling images, unused networks, and build cache
docker system prune -f

# Output: "Total reclaimed space: 8.3GB"
```

This is safe — it only removes resources not attached to a running container.

##### [3. Deep clean (include unused images and volumes)](https://devopsil.com/articles/2026-03-30-docker-no-space-left-on-device-fix#3-deep-clean-include-unused-images-and-volumes)

```
# Also remove images not used by any container (not just dangling)
docker system prune -a -f

# Also remove orphaned volumes (WARNING: data loss if volumes hold state)
docker volume prune -f
```

##### [4. Target specific resource types](https://devopsil.com/articles/2026-03-30-docker-no-space-left-on-device-fix#4-target-specific-resource-types)

```
# Remove only dangling images
docker image prune -f

# Remove images older than 24 hours
docker image prune -a --filter "until=24h" -f

# Remove stopped containers older than 48 hours
docker container prune --filter "until=48h" -f

# Remove specific large images
docker images --format '{{.Size}}\t{{.Repository}}:{{.Tag}}' | sort -rh | head -20
docker rmi <image-id>
```

##### [5. Clean the build cache specifically](https://devopsil.com/articles/2026-03-30-docker-no-space-left-on-device-fix#5-clean-the-build-cache-specifically)

```
# See build cache size
docker builder prune --dry-run

# Clear all build cache
docker builder prune -a -f

# Keep cache from the last 72 hours
docker builder prune --filter "until=72h" -f
```

##### [6. Move Docker's data directory (if /var is small)](https://devopsil.com/articles/2026-03-30-docker-no-space-left-on-device-fix#6-move-dockers-data-directory-if-var-is-small)

```
# Stop Docker
sudo systemctl stop docker

# Edit or create /etc/docker/daemon.json
sudo tee /etc/docker/daemon.json <<EOF
{
  "data-root": "/mnt/large-disk/docker"
}
EOF

# Move existing data
sudo rsync -aP /var/lib/docker/ /mnt/large-disk/docker/

# Start Docker
sudo systemctl start docker
```

#### [Prevention Tips](https://devopsil.com/articles/2026-03-30-docker-no-space-left-on-device-fix#prevention-tips)

*   **Schedule a cron job** for regular cleanup:
```
# /etc/cron.daily/docker-cleanup
#!/bin/bash
docker system prune -a -f --filter "until=168h"
docker volume prune -f --filter "label!=keep"
```
*   **Use multi-stage builds** to keep final images small — only copy artifacts from build stages.
*   **Set `--rm` on CI builds** (`docker run --rm`) so containers are removed on exit.
*   **Monitor `/var/lib/docker` disk usage** with [Prometheus](https://devopsil.com/tools/prometheus "Prometheus")`node_filesystem_avail_bytes` and alert at 80%.
*   **Configure Docker log rotation** in `daemon.json` to prevent container logs from consuming disk:
```
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```
