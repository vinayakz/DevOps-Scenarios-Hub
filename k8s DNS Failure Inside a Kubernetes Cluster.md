# Kubernetes Production Troubleshooting: DNS Failure Inside a Kubernetes Cluster

DNS is one of the most critical components in a Kubernetes cluster. When DNS fails, applications may be healthy, Pods may be running, and Services may exist, but applications suddenly lose the ability to communicate with each other.

DNS-related issues are among the most common production incidents and are frequently asked in Kubernetes, DevOps, SRE, and Platform Engineering interviews.

---

# Scenario

## Problem Statement

A production application suddenly stops communicating with backend services.

Users report:

```text
Application Timeout
Connection Refused
Service Unreachable
```

Pods are healthy.

Services exist.

No application crashes.

However, service names are not resolving.

---

# What is DNS in Kubernetes?

DNS enables Pods to communicate with Services using names instead of IP addresses.

### Example

```text
Frontend Pod
    ↓
backend-service.default.svc.cluster.local
    ↓
Backend Service
```

Instead of connecting to:

```text
10.96.145.12
```

Applications connect using:

```text
backend-service
```

or

```text
backend-service.default.svc.cluster.local
```

Without DNS, service discovery fails and applications cannot communicate.

---

# Common Symptoms

You may observe:

- Application cannot reach another service
- Service name resolution fails
- nslookup fails
- dig command fails
- Intermittent connection issues
- Application works using IP but not Service Name
- API requests timing out
- Database hostname not resolving

---

# Interview Question

A production application cannot communicate with another service inside Kubernetes.

Pods are healthy.

Services are healthy.

However, DNS lookups fail.

How would you troubleshoot the issue?

---

# Step 1: Check CoreDNS Pods

CoreDNS is responsible for DNS resolution inside Kubernetes.

## Commands

```bash
kubectl get pods -n kube-system
```

Expected:

```text
coredns-6d4b75cb6d-abcde    Running
coredns-6d4b75cb6d-fghij    Running
```

---

## If CoreDNS is Not Running

Inspect the Pod:

```bash
kubectl describe pod <coredns-pod> -n kube-system
```

Check:

- Events
- CrashLoopBackOff
- Resource Issues
- Image Pull Errors

---

# Step 2: Check CoreDNS Logs

Verify whether DNS requests are reaching CoreDNS.

## Commands

```bash
kubectl logs -n kube-system <coredns-pod>
```

---

## Look For

- Crash errors
- Configuration issues
- DNS timeout messages
- Upstream DNS failures
- Forwarding errors

### Example

```text
plugin/errors: read udp timeout
```

or

```text
upstream DNS server unreachable
```

---

# Step 3: Verify CoreDNS Configuration

CoreDNS configuration is stored in a ConfigMap.

## Command

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

---

## Verify

- Forwarders
- DNS Zones
- Corefile Syntax
- Cluster Domain

### Example

```yaml
forward . 8.8.8.8 8.8.4.4
```

---

## Common Misconfiguration

```yaml
forward . 192.168.1.250
```

If the upstream DNS server is unavailable:

```text
DNS Queries Fail
```

---

# Step 4: Verify DNS Resolution from a Pod

Connect to a running Pod.

## Command

```bash
kubectl exec -it <pod-name> -- sh
```

Run:

```bash
nslookup kubernetes.default

dig kubernetes.default
```

---

## Expected Output

```text
Name: kubernetes.default
Address: 10.96.0.1
```

---

## Failure Example

```text
server can't find kubernetes.default
```

This indicates DNS resolution is broken.

---

# Step 5: Verify DNS Service

Check the Kubernetes DNS Service.

## Command

```bash
kubectl get svc -n kube-system
```

Expected:

```text
NAME       TYPE        CLUSTER-IP
kube-dns   ClusterIP   10.96.0.10
```

---

## Verify Endpoints

```bash
kubectl get endpoints kube-dns -n kube-system
```

Expected:

```text
10.244.0.5:53
10.244.1.7:53
```

---

## Problem

```text
Endpoints = <none>
```

Result:

```text
DNS Requests Cannot Reach CoreDNS
```

---

# Step 6: Check Network Policies

Network Policies may block DNS traffic.

## Commands

```bash
kubectl get networkpolicy -A
```

Inspect:

```bash
kubectl describe networkpolicy <policy-name>
```

---

## DNS Requires

| Protocol | Port |
|-----------|--------|
| UDP | 53 |
| TCP | 53 |

---

## Common Issue

Network policy blocks:

```text
Pod → CoreDNS
```

Result:

```text
DNS Resolution Fails
```

---

# Step 7: Check Network Connectivity

Verify that Pods can reach the DNS Service.

## Commands

```bash
kubectl exec -it <pod-name> -- ping kube-dns.kube-system.svc.cluster.local
```

or

```bash
kubectl exec -it <pod-name> -- wget kube-dns.kube-system.svc.cluster.local
```

---

## Failure

If packets cannot reach CoreDNS:

```text
DNS Traffic Blocked
```

Possible causes:

- CNI issue
- Network policy issue
- Node networking issue

---

# Step 8: Verify CNI Plugin Health

DNS traffic relies on Pod networking.

If the CNI plugin is broken, DNS may fail.

---

## Common CNI Plugins

- Calico
- Flannel
- Cilium
- Weave
- Canal

---

## Commands

```bash
kubectl get pods -n kube-system
```

Look for:

```text
calico-node
cilium-agent
flannel
```

---

## Verify

- Pods Running
- No CrashLoopBackOff
- No Network Errors

---

# Production Troubleshooting Flow

A Senior DevOps Engineer follows this workflow:

```text
DNS Failure
      ↓
Check CoreDNS Pods
      ↓
Check CoreDNS Logs
      ↓
Check CoreDNS ConfigMap
      ↓
Run nslookup / dig
      ↓
Check kube-dns Service
      ↓
Check Endpoints
      ↓
Check Network Policies
      ↓
Check CNI Plugin
      ↓
Escalate to L2 / L3 if required
```

---

# Common Root Causes

| Issue | Impact |
|---------|---------|
| CoreDNS Pod Crash | DNS Failure |
| Invalid CoreDNS Config | DNS Failure |
| Empty kube-dns Endpoints | DNS Failure |
| Network Policy Blocking Port 53 | DNS Failure |
| CNI Plugin Failure | DNS Failure |
| Node Networking Issue | DNS Failure |
| Firewall Blocking DNS Traffic | DNS Failure |
| Resource Starvation on CoreDNS | Intermittent DNS Failures |

---

# Real Production Scenario #1

## Problem

Microservices suddenly stop communicating.

Users report:

```text
API Timeout
```

---

### Investigation

Check CoreDNS:

```bash
kubectl get pods -n kube-system
```

Output:

```text
coredns-abcde Running
```

Looks healthy.

---

Run:

```bash
kubectl exec -it app-pod -- nslookup backend-service
```

Output:

```text
connection timed out
```

---

Check Network Policies:

```bash
kubectl get networkpolicy -A
```

Found:

```yaml
egress:
- ports:
  - port: 80
```

Port 53 was not allowed.

---

## Root Cause

DNS traffic blocked by Network Policy.

---

## Fix

Allow:

```yaml
ports:
- protocol: UDP
  port: 53

- protocol: TCP
  port: 53
```

---

# Real Production Scenario #2

## Problem

Pods cannot resolve external domains.

```bash
nslookup google.com
```

Fails.

Internal Services work.

---

## Investigation

Check CoreDNS ConfigMap.

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Found:

```yaml
forward . 192.168.1.100
```

Upstream DNS server was unavailable.

---

## Root Cause

Broken DNS Forwarder.

---

## Fix

Update CoreDNS:

```yaml
forward . 8.8.8.8 8.8.4.4
```

Restart CoreDNS.

---

# Scenario-Based Interview Questions

## Scenario 1

### Question

Pods can communicate using IP addresses but not Service Names.

What does this indicate?

### Answer

DNS failure.

Network connectivity exists, but DNS resolution is broken.

---

## Scenario 2

### Question

CoreDNS Pods are Running but DNS queries fail.

What would you check next?

### Answer

- CoreDNS Logs
- CoreDNS ConfigMap
- kube-dns Service
- Endpoints

---

## Scenario 3

### Question

DNS works for internal services but external domains fail.

What is the likely issue?

### Answer

Incorrect upstream DNS forwarders in CoreDNS configuration.

---

## Scenario 4

### Question

DNS works on some nodes but fails on others.

What could be wrong?

### Answer

- CNI Plugin issue
- Node networking issue
- Firewall issue
- DNS packet drops

---

## Scenario 5

### Question

Why would DNS fail intermittently?

### Answer

Possible reasons:

- CoreDNS resource starvation
- High CPU usage
- Memory pressure
- Packet loss
- Node networking issues

---

# Interview Answer

### Q: How would you troubleshoot DNS failure inside a Kubernetes cluster?

### Answer

First, I would verify that CoreDNS Pods are running in the kube-system namespace. Then I would review CoreDNS logs and validate the CoreDNS ConfigMap. Next, I would test DNS resolution from a Pod using nslookup or dig. If DNS resolution still fails, I would verify the kube-dns Service and its Endpoints. After that, I would inspect Network Policies to ensure UDP and TCP port 53 are allowed. Finally, I would verify the health of the CNI plugin and node networking. Based on the findings, I would either resolve the issue or escalate it to the L2/L3 platform team.

---

# Key Takeaway

## Healthy Pods ≠ Healthy DNS

Always verify:

✅ CoreDNS Pods

✅ CoreDNS Logs

✅ CoreDNS Configuration

✅ kube-dns Service

✅ Endpoints

✅ Network Policies

✅ CNI Plugin

✅ Node Networking

Before blaming the application.

A Kubernetes cluster is only as reliable as its DNS infrastructure. Most production communication failures eventually trace back to DNS, networking, or service discovery issues.
