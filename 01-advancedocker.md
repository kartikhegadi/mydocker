# Docker Commands Reference Guide

A practical reference for essential Docker commands — build options, runtime configuration, container management, cleanup, and monitoring — each with examples and real-world use cases.

---

## 1. Build Docker Image Without Cache

By default, Docker reuses cached layers from previous builds to speed things up. Sometimes you need a completely fresh build.

```bash
docker build --no-cache -t myapp:latest .
```

**Use case:** When a dependency in a cached layer (e.g., `apt-get update` or `npm install`) needs to be re-fetched because the underlying package versions changed but your Dockerfile didn't, or when debugging a build that "should" be failing but cached layers are masking the real error.

---

## 2. Dockerfile in a Different Directory

If your Dockerfile isn't in the build context's root, point to it explicitly with `-f`.

```bash
docker build -f /path/to/custom/Dockerfile -t myapp:latest .
```

**Example with separate build context:**
```bash
docker build -f docker/prod/Dockerfile -t myapp:prod ./app
```

**Use case:** Projects with multiple Dockerfiles (e.g., `Dockerfile.dev`, `Dockerfile.prod`) stored in a `docker/` subfolder, keeping the project root clean.

---

## 3. Git Commit ID as Docker Tag

Tag images with the Git commit hash for precise traceability between code and image.

```bash
docker build -t myapp:$(git rev-parse --short HEAD) .
```

**Full example:**
```bash
GIT_COMMIT=$(git rev-parse --short HEAD)
docker build -t myapp:$GIT_COMMIT .
docker push myregistry.com/myapp:$GIT_COMMIT
```

**Use case:** CI/CD pipelines where you need to trace exactly which commit produced a running container — critical for debugging production issues and rollbacks.

---

## 4. Date as Docker Tag

Tag images with a timestamp for chronological versioning.

```bash
docker build -t myapp:$(date +%Y%m%d) .
```

**With time included:**
```bash
docker build -t myapp:$(date +%Y%m%d-%H%M%S) .
```

**Use case:** Nightly builds or scheduled deployments where you want to know exactly when an image was built, useful alongside (or instead of) semantic versioning.

---

## 5. Setup RAM & CPU for Docker Container

Limit container resource usage to prevent one container from starving the host or other containers.

```bash
docker run -d \
  --memory="512m" \
  --cpus="1.5" \
  --name myapp \
  myapp:latest
```

**More granular CPU control:**
```bash
docker run -d --cpus="0.5" --memory="256m" --memory-swap="512m" myapp:latest
```

**Use case:** Multi-tenant hosts running several containers, or simulating production resource constraints in staging/testing environments.

---

## 6. Run Commands as Non-Root User in Container

Running as root inside a container is a security risk. Use `--user` or define a user in the Dockerfile.

```bash
docker run -d --user 1000:1000 myapp:latest
```

**In a Dockerfile:**
```dockerfile
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser
```

**Use case:** Hardening production containers — if an attacker exploits the app, they don't get root access inside (or potentially outside, via container breakout) the container.

---

## 7. Running Container as Read-Only

Prevent the container's filesystem from being written to, except explicitly allowed paths.

```bash
docker run -d --read-only --tmpfs /tmp myapp:latest
```

**With a writable volume for logs:**
```bash
docker run -d --read-only --tmpfs /tmp -v app-logs:/var/log/app myapp:latest
```

**Use case:** Security-sensitive applications where you want to guarantee the running process can't modify its own binaries or config files — common in compliance-driven environments.

---

## 8. Setup Automatic Container Restart

Ensure containers come back up after crashes or host reboots.

```bash
docker run -d --restart unless-stopped myapp:latest
```

**Restart policy options:**
| Policy | Behavior |
|---|---|
| `no` | Never restart (default) |
| `on-failure[:max-retries]` | Restart only on non-zero exit |
| `always` | Always restart, even after manual stop |
| `unless-stopped` | Always restart unless explicitly stopped by user |

**Use case:** Production services that must stay up — `unless-stopped` is the most common choice since it survives host reboots but respects a manual `docker stop`.

---

## 9. Healthcheck of Container

Define a command Docker runs periodically to verify the container is actually healthy, not just running.

```bash
docker run -d \
  --health-cmd="curl -f http://localhost:8080/health || exit 1" \
  --health-interval=30s \
  --health-timeout=5s \
  --health-retries=3 \
  myapp:latest
```

**In a Dockerfile:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

**Check status:**
```bash
docker inspect --format='{{.State.Health.Status}}' myapp
```

**Use case:** Load balancers and orchestrators (Docker Swarm, Kubernetes-adjacent tooling) use health status to decide whether to route traffic to a container or restart it.

---

## 10. Tailing Logs of Container

Stream container logs in real time, similar to `tail -f`.

```bash
docker logs -f myapp
```

**With timestamps and limited history:**
```bash
docker logs -f --tail 100 --timestamps myapp
```

**Use case:** Live debugging of an application while reproducing an issue, without needing to SSH into the host and find log files manually.

---

## 11. Check If Specific Process Running in Container

Verify a particular process is alive inside a running container.

```bash
docker top myapp
```

**Filter for a specific process:**
```bash
docker top myapp | grep nginx
```

**Alternative using exec:**
```bash
docker exec myapp ps aux | grep node
```

**Use case:** Quickly confirming whether your app's main process (or a worker/cron process) is still running without needing a full shell session.

---

## 12. Running Containers in Restricted Mode

Drop Linux capabilities and apply security profiles to minimize the container's attack surface.

```bash
docker run -d \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges \
  myapp:latest
```

**Use case:** Running untrusted or third-party workloads, or hardening any internet-facing service so that even a compromised process has minimal system capabilities to exploit further.

---

## 13. Get Container Logs on Host Machine

Locate or redirect Docker's log files directly from the host filesystem.

```bash
docker inspect --format='{{.LogPath}}' myapp
```

**Copy logs out to a host file:**
```bash
docker logs myapp > myapp-logs.txt 2>&1
```

**Use case:** Archiving logs for compliance/auditing, or feeding them into an external log aggregator (e.g., ELK, Splunk) via a host-level log shipper.

---

## 14. Docker Prune (Delete Unused Resources)

Clean up unused containers, networks, images, and build cache in one command.

```bash
docker system prune -a
```

**With volumes included (use carefully):**
```bash
docker system prune -a --volumes
```

**Use case:** Reclaiming disk space on CI runners or dev machines after repeated builds — `-a` removes all unused images, not just dangling ones, so use it when you're sure you don't need older tags.

---

## 15. Delete Dangling Images

Dangling images are untagged layers left behind after rebuilds (shown as `<none>`).

```bash
docker image prune
```

**Force without confirmation:**
```bash
docker image prune -f
```

**Use case:** Routine cleanup after frequent rebuilds during development, where each new build leaves the old, now-untagged layer behind.

---

## 16. Delete All Containers

Remove every container, running or stopped.

```bash
docker rm -f $(docker ps -aq)
```

**Stop first, then remove (safer):**
```bash
docker stop $(docker ps -aq) && docker rm $(docker ps -aq)
```

**Use case:** Resetting a local dev environment completely, or cleaning up a CI runner between pipeline runs to avoid leftover state affecting the next build.

---

## 17. List Images Created After a Specific Image

Filter images based on their creation order relative to another image.

```bash
docker images --filter "since=myapp:v1"
```

**Filter "before" instead:**
```bash
docker images --filter "before=myapp:v1"
```

**Use case:** Auditing what new images were built after a known stable release, helpful when tracking down which build introduced a regression.

---

## 18. Rename Containers

Rename an existing container without recreating it.

```bash
docker rename old_name new_name
```

**Example:**
```bash
docker rename quirky_einstein web-server
```

**Use case:** Docker auto-generates random names (like `quirky_einstein`) when you don't specify one — renaming gives you meaningful identifiers for scripts and monitoring dashboards.

---

## 19. Restart Policy Setup in Containers

Update the restart policy of an already-running container without recreating it.

```bash
docker update --restart unless-stopped myapp
```

**Use case:** You started a container without a restart policy and want to add one in production without downtime from a full recreate.

---

## 20. Pause & Unpause Containers

Freeze all processes in a container (via cgroups freezer) without stopping it.

```bash
docker pause myapp
docker unpause myapp
```

**Use case:** Temporarily freeing up CPU for another urgent process on a busy host, or freezing container state for a consistent snapshot/backup without fully stopping the service.

---

## 21. Docker Diff Command

See what files have changed (added/modified/deleted) inside a container compared to its image.

```bash
docker diff myapp
```

**Sample output interpretation:**
```
A /app/uploads/file.txt    # Added
C /etc/nginx/nginx.conf    # Changed
D /tmp/cache.tmp           # Deleted
```

**Use case:** Debugging unexpected behavior by checking if something modified config files at runtime, or auditing whether a container is writing data outside intended volumes.

---

## 22. Export Docker Images

Save an image to a `.tar` file for transfer without using a registry.

```bash
docker save -o myapp.tar myapp:latest
```

**Compressed version:**
```bash
docker save myapp:latest | gzip > myapp.tar.gz
```

**Use case:** Air-gapped environments with no internet/registry access, or quickly transferring an image between machines on the same network via USB/SCP.

---

## 23. Import Docker Image from tar Files

Load a previously saved image back into Docker.

```bash
docker load -i myapp.tar
```

**From a gzipped file:**
```bash
gunzip -c myapp.tar.gz | docker load
```

**Use case:** Restoring images on offline/air-gapped servers, or rapidly provisioning a new machine with pre-built images instead of rebuilding from scratch.

---

## 24. Copy Files From Container to Host Machine

Transfer files between a running (or stopped) container and the host filesystem.

```bash
docker cp myapp:/app/logs/error.log ./error.log
```

**Copy host file into a container:**
```bash
docker cp ./config.json myapp:/app/config.json
```

**Copy an entire directory:**
```bash
docker cp myapp:/app/data ./backup-data
```

**Use case:** Extracting generated reports, crash dumps, or log files from a container for analysis, or injecting an updated config file without rebuilding the image.

---

## 25. Real-Time Container Resource Usage (RAM & CPU)

Monitor live CPU, memory, network, and I/O usage across containers.

```bash
docker stats
```

**For specific containers only:**
```bash
docker stats myapp web-server
```

**Non-streaming, one-shot snapshot (useful for scripts):**
```bash
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

**Use case:** Diagnosing memory leaks or CPU spikes in real time, or building a quick monitoring script that logs resource usage at intervals for capacity planning.

---

## Quick Reference Table

| # | Command | Purpose |
|---|---|---|
| 1 | `docker build --no-cache` | Fresh build, ignore cache |
| 2 | `docker build -f path/Dockerfile` | Custom Dockerfile location |
| 3 | `docker build -t app:$(git rev-parse --short HEAD)` | Git-based tagging |
| 4 | `docker build -t app:$(date +%Y%m%d)` | Date-based tagging |
| 5 | `--memory --cpus` | Resource limits |
| 6 | `--user` | Run as non-root |
| 7 | `--read-only` | Immutable filesystem |
| 8 | `--restart unless-stopped` | Auto-restart |
| 9 | `--health-cmd` | Container health checks |
| 10 | `docker logs -f` | Live log tailing |
| 11 | `docker top` | Check running processes |
| 12 | `--cap-drop=ALL` | Restricted/hardened mode |
| 13 | `docker inspect --format='{{.LogPath}}'` | Host log file location |
| 14 | `docker system prune -a` | Full cleanup |
| 15 | `docker image prune` | Remove dangling images |
| 16 | `docker rm -f $(docker ps -aq)` | Delete all containers |
| 17 | `docker images --filter since=` | List images by creation order |
| 18 | `docker rename` | Rename a container |
| 19 | `docker update --restart` | Change restart policy live |
| 20 | `docker pause` / `unpause` | Freeze/unfreeze container |
| 21 | `docker diff` | Show filesystem changes |
| 22 | `docker save` | Export image to tar |
| 23 | `docker load` | Import image from tar |
| 24 | `docker cp` | Copy files in/out of container |
| 25 | `docker stats` | Real-time resource monitoring |
