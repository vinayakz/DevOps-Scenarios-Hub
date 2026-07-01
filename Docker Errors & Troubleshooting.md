# Common Docker Errors & Troubleshooting: A Practical Guide for DevOps Engineers

Docker is one of the most widely used containerization platforms in modern DevOps. Whether you're deploying microservices, running CI/CD pipelines, or managing production workloads, Docker issues are inevitable.

The ability to quickly identify, troubleshoot, and resolve Docker problems is an essential skill for every DevOps Engineer and is one of the most frequently tested topics in DevOps interviews.

In this guide, you'll learn:

- Common Docker production issues
- Root causes
- Troubleshooting commands
- Real-world scenarios
- Best practices
- Interview questions

---

# Why Docker Troubleshooting Matters

Docker issues can lead to:

- Application downtime
- Failed deployments
- Container crashes
- Image pull failures
- Network connectivity problems
- Storage exhaustion
- CI/CD pipeline failures

A production-ready DevOps Engineer should be able to identify the root cause quickly and restore services with minimal downtime.

---

# Docker Troubleshooting Workflow

Instead of randomly executing commands, follow a structured troubleshooting process.

```text
Application Error
       │
       ▼
docker ps -a
       │
       ▼
docker logs
       │
       ▼
docker inspect
       │
       ▼
docker exec
       │
       ▼
docker network inspect
       │
       ▼
Identify Root Cause
       │
       ▼
Resolve Issue
```

---

# 1. Container Exits Immediately

## Problem

Your container starts and exits immediately.

```text
Exited (1)
```

---

## Common Causes

- Incorrect startup command
- Application crash
- Missing dependencies
- Missing environment variables
- Configuration errors

---

## Troubleshooting Commands

```bash
docker ps -a

docker logs <container-name>

docker inspect <container-name>
```

---

## Example

```bash
docker logs springboot-app
```

Output:

```text
Failed to connect to database
```

---

## Solution

- Review application logs
- Verify ENTRYPOINT/CMD
- Check environment variables
- Ensure required dependencies are available

---

# Scenario-Based Interview Question

### Question

A Docker container exits immediately after startup.

How would you troubleshoot it?

### Answer

1. Check container status.

```bash
docker ps -a
```

2. Review logs.

```bash
docker logs <container>
```

3. Inspect container configuration.

```bash
docker inspect <container>
```

4. Fix the application or startup command.

---

# 2. Port Already in Use

## Problem

Container fails to start.

Error:

```text
Bind for 0.0.0.0:8080 failed:
Port is already allocated
```

---

## Root Cause

Another process or container is already using the port.

---

## Troubleshooting

```bash
docker ps

netstat -tulnp

lsof -i :8080
```

---

## Solution

Stop the conflicting container.

```bash
docker stop <container-id>
```

Or use another port.

```bash
docker run -p 9090:80 nginx
```

---

# Scenario-Based Interview Question

### Question

Your container fails with:

```text
Port is already allocated
```

What will you check?

### Answer

- Running containers
- Host processes
- Existing port bindings

---

# 3. Image Not Found

## Problem

Docker cannot find the requested image.

```text
Unable to find image
```

---

## Root Causes

- Wrong image name
- Wrong image tag
- Image not pulled
- Private repository

---

## Troubleshooting

```bash
docker images

docker pull nginx
```

---

## Solution

- Verify image name
- Verify image tag
- Pull the latest image
- Authenticate to private registry

---

# Scenario-Based Interview Question

### Question

Why would Docker fail to pull an image?

### Answer

Possible reasons:

- Typo in image name
- Incorrect tag
- Registry authentication failure
- Image doesn't exist

---

# 4. Permission Denied

## Problem

Docker command fails.

```text
permission denied while trying to connect to Docker daemon
```

---

## Root Cause

Current user doesn't have permission to access Docker daemon.

---

## Troubleshooting

```bash
systemctl status docker
```

---

## Solution

Add your user to the Docker group.

```bash
sudo usermod -aG docker $USER
```

Logout and login again.

Or temporarily use:

```bash
sudo docker ps
```

---

# 5. Docker Daemon Not Running

## Problem

```text
Cannot connect to the Docker daemon
```

---

## Troubleshooting

```bash
systemctl status docker
```

---

## Solution

Start Docker.

```bash
sudo systemctl start docker
```

Enable Docker at boot.

```bash
sudo systemctl enable docker
```

---

# Scenario-Based Interview Question

### Question

Docker commands suddenly stop working on a production server.

What would you verify first?

### Answer

Check whether Docker daemon is running.

```bash
systemctl status docker
```

---

# 6. No Space Left on Device

## Problem

```text
No space left on device
```

---

## Root Cause

Docker images, containers, volumes, or build cache consume all available disk space.

---

## Troubleshooting

```bash
docker system df

df -h
```

---

## Solution

Remove unused resources.

```bash
docker system prune

docker image prune

docker volume prune
```

---

## Best Practice

Schedule periodic cleanup jobs.

---

# Scenario-Based Interview Question

### Question

Your CI/CD pipeline suddenly starts failing with:

```text
No space left on device
```

How do you troubleshoot?

### Answer

Check Docker disk usage.

```bash
docker system df
```

Then remove unused images, volumes, and stopped containers.

---

# 7. ImagePullBackOff / Pull Access Denied

## Problem

```text
ImagePullBackOff
```

or

```text
pull access denied
```

---

## Root Causes

- Wrong image name
- Missing image tag
- Private registry authentication failure
- Repository permissions

---

## Troubleshooting

```bash
docker login

docker pull <image-name>
```

---

## Solution

- Verify image exists
- Authenticate with registry
- Check repository permissions

---

# Scenario-Based Interview Question

### Question

A deployment fails because Kubernetes reports:

```text
ImagePullBackOff
```

What would you verify?

### Answer

- Image name
- Image tag
- Registry credentials
- Repository permissions

---

# 8. Containers Cannot Communicate

## Problem

Container A cannot connect to Container B.

---

## Troubleshooting

```bash
docker network ls

docker network inspect bridge
```

---

## Verify

- Same Docker network
- Correct container names
- Exposed ports

---

## Solution

Connect both containers to the same network.

```bash
docker network create app-network

docker run --network app-network ...
```

---

# Scenario-Based Interview Question

### Question

Two containers are running but cannot communicate.

What would you check?

### Answer

- Docker network
- DNS resolution
- Network aliases
- Firewall rules

---

# 9. Container Keeps Restarting

## Problem

Container continuously restarts.

```text
Restarting (1)
```

---

## Root Causes

- Application crash
- Health check failure
- Missing dependencies
- Incorrect environment variables

---

## Troubleshooting

```bash
docker logs <container>

docker inspect <container>
```

---

## Solution

Fix the underlying application issue before restarting.

---

# 10. Health Check Failing

## Problem

Container status:

```text
Unhealthy
```

---

## Troubleshooting

```bash
docker inspect <container>
```

Check:

```text
Health
```

---

## Solution

- Verify health endpoint
- Verify application startup
- Increase health check timeout

---

# Useful Docker Troubleshooting Commands

| Command | Purpose |
|----------|----------|
| `docker ps -a` | List all containers |
| `docker logs <container>` | View container logs |
| `docker inspect <container>` | Inspect container configuration |
| `docker exec -it <container> bash` | Access container shell |
| `docker images` | List images |
| `docker network ls` | List Docker networks |
| `docker network inspect` | Inspect network |
| `docker volume ls` | List volumes |
| `docker system df` | Check Docker disk usage |
| `docker system prune` | Remove unused resources |

---

# Real-World Production Incident

## Problem

A Spring Boot application is deployed inside Docker.

Immediately after deployment, the container exits.

---

### Step 1

Check container status.

```bash
docker ps -a
```

Output:

```text
Exited (1)
```

---

### Step 2

Check logs.

```bash
docker logs springboot-app
```

Output:

```text
Database connection failed
```

---

### Step 3

Inspect configuration.

```bash
docker inspect springboot-app
```

Verify:

- Environment variables
- Network
- Volumes

---

### Step 4

Access the container.

```bash
docker exec -it springboot-app bash
```

Verify:

- Configuration files
- Database hostname
- Environment variables

---

### Step 5

Fix database configuration.

Restart the container.

Application starts successfully.

---

# Advanced Scenario-Based Interview Questions

## Scenario 1

### Question

Container is running but application is inaccessible.

What would you check?

### Answer

- Port mapping
- Application logs
- Listening port
- Firewall
- Docker network

---

## Scenario 2

### Question

Container suddenly stops after working for several days.

### Answer

Check:

- Logs
- Disk space
- OOM events
- Resource usage

---

## Scenario 3

### Question

Docker daemon is running but containers won't start.

### Answer

Verify:

- Storage driver
- Available disk space
- Docker events
- System logs

---

## Scenario 4

### Question

A container works locally but fails in production.

### Answer

Compare:

- Environment variables
- Mounted volumes
- Docker image version
- Network configuration

---

## Scenario 5

### Question

How do you troubleshoot a Docker networking issue?

### Answer

1. Verify network.

```bash
docker network ls
```

2. Inspect network.

```bash
docker network inspect bridge
```

3. Test connectivity.

```bash
ping <container-name>
```

4. Verify exposed ports.

---

# Interview Tips

Most candidates immediately execute:

```bash
docker logs
```

A Senior DevOps Engineer follows this workflow:

```text
Application
      ↓
Container
      ↓
Logs
      ↓
Inspect
      ↓
Network
      ↓
Storage
      ↓
Root Cause
```

This structured troubleshooting approach demonstrates strong production support skills.

---

# Key Takeaways

Every DevOps Engineer should master these Docker troubleshooting areas:

- ✅ Container Lifecycle
- ✅ Docker Daemon
- ✅ Image Management
- ✅ Port Mapping
- ✅ Networking
- ✅ Volumes
- ✅ Storage Cleanup
- ✅ Logs & Inspection
- ✅ Registry Authentication
- ✅ Resource Management

Remember:

> **Don't jump straight to `docker logs`.** Follow a systematic troubleshooting process:
>
> **Application → Container → Logs → Inspect → Network → Storage → Root Cause**

A methodical approach not only resolves incidents faster but also demonstrates the production troubleshooting mindset expected from senior DevOps engineers.
