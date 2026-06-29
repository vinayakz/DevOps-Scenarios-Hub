# Docker Container Exited with Code 137? Here's What Really Happened

One of your containers suddenly died.

You check the logs:

```bash
docker logs <container_id>
```

Nothing useful.

Then you run:

```bash
docker ps -a
```

Output:

```bash
Exited (137)
```

Many engineers ignore this number and restart the container.

Don't.

**Exit code 137 is not a random number.** It tells you exactly how the container was terminated.

---

# Understanding Exit Code 137

In Linux, process exit codes follow a simple rule:

```text
Exit Code = 128 + Signal Number
```

For Exit Code 137:

```text
137 = 128 + 9
```

Signal 9 is:

```text
SIGKILL
```

This means the container's main process (PID 1) was forcefully terminated by a **SIGKILL** signal.

---

# Who Sent the SIGKILL?

Several components can send SIGKILL to a container:

### 1. Linux OOM Killer (Most Common in Production)

When the system runs out of memory, the Linux kernel may kill processes to protect the host.

The container itself isn't crashing.

The kernel is terminating it because memory is exhausted.

### 2. Docker

Someone may have executed:

```bash
docker kill <container_id>
```

Or Docker may have forcefully stopped the container after a timeout.

### 3. Kubernetes

Kubernetes can terminate containers due to:

- OOMKilled events
- Pod eviction
- Force deletion
- Node pressure

### 4. System Services

System processes such as systemd or other host-level services can terminate containers under specific conditions.

### 5. Human Error

An administrator or automation script may have manually killed the container.

---

# First Suspect: Out Of Memory (OOM)

In production environments, **OOM (Out Of Memory)** is the most common cause of Exit Code 137.

When memory becomes scarce, the Linux kernel's OOM Killer selects a process and terminates it.

---

# How to Confirm an OOM Kill

## Method 1: Docker Inspect

Run:

```bash
docker inspect <container_id> | grep -i OOM
```

Example output:

```json
"OOMKilled": true
```

If you see:

```json
"OOMKilled": true
```

You have your answer.

The container was terminated due to memory exhaustion.

---

## Method 2: Check Kernel Logs

### Using dmesg

```bash
dmesg | grep -i kill
```

Example:

```text
Out of memory: Killed process 4127 (node)
anon-rss:1851232kB
```

This confirms that the Linux kernel killed the process.

---

### Using journalctl

```bash
journalctl -k | grep -i kill
```

Example:

```text
kernel: Out of memory: Killed process 4127 (node)
```

Again, this indicates an OOM kill.

---

# Other Reasons for Exit Code 137

## Docker Stop Timeout

When you run:

```bash
docker stop <container_id>
```

Docker first sends:

```text
SIGTERM (15)
```

and waits for the container to shut down gracefully.

If the process does not exit within the configured timeout, Docker sends:

```text
SIGKILL (9)
```

Result:

```text
Exit Code 137
```

---

## Kubernetes Force Termination

Kubernetes may forcefully terminate a container when:

- Grace period expires
- Pod is force deleted
- Node is under resource pressure
- OOM policies are triggered

Useful commands:

```bash
kubectl describe pod <pod-name>
```

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Manual Container Kill

Someone may have executed:

```bash
docker kill <container_id>
```

Unlike `docker stop`, this sends SIGKILL immediately.

Result:

```text
Exit Code 137
```

Check Docker events:

```bash
docker events
```

---

# Useful Investigation Commands

## Check Container Status

```bash
docker ps -a
```

---

## Inspect Container Details

```bash
docker inspect <container_id>
```

---

## Verify OOM Kill

```bash
docker inspect <container_id> | grep -i OOM
```

---

## Check Kernel Logs

```bash
dmesg | grep -i kill
```

---

## Check System Journal

```bash
journalctl -k | grep -i kill
```

---

## Review Container Events

```bash
docker events
```

---

## Kubernetes Events

```bash
kubectl describe pod <pod-name>
```

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

# Related Exit Codes You Should Know

## Exit Code 143

```text
143 = 128 + 15
```

Signal:

```text
SIGTERM
```

Meaning:

- Graceful shutdown request
- Someone or something asked the process to stop
- Not necessarily an error

---

## Exit Code 139

```text
139 = 128 + 11
```

Signal:

```text
SIGSEGV
```

Meaning:

- Segmentation fault
- Application accessed invalid memory
- Usually an application bug, native library issue, or memory corruption problem

---

## Exit Code 137

```text
137 = 128 + 9
```

Signal:

```text
SIGKILL
```

Meaning:

- Forcefully terminated
- Most commonly caused by OOM Killer
- Could also be Docker, Kubernetes, system services, or a manual kill command

---

# Quick Troubleshooting Checklist

When you see:

```text
Exited (137)
```

Check the following in order:

### Step 1

```bash
docker inspect <container_id> | grep -i OOM
```

### Step 2

```bash
dmesg | grep -i kill
```

### Step 3

```bash
journalctl -k | grep -i kill
```

### Step 4

Check Docker events:

```bash
docker events
```

### Step 5

If running Kubernetes:

```bash
kubectl describe pod <pod-name>
```

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

# Key Takeaway

**Exit Code 137 does not mean the application crashed.**

It means the process received a **SIGKILL (Signal 9)**.

In production environments, the first thing to investigate is:

✅ **OOMKilled**

because memory exhaustion is the most common cause of Exit Code 137.

Before restarting the container, always verify:

- Memory usage
- OOM events
- Docker events
- Kubernetes events
- System logs

The exit code is often the first clue to the root cause.
