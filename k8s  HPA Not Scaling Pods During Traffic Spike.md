# Kubernetes Production Scenario: HPA Not Scaling Pods During Traffic Spike

One of the most common Kubernetes production incidents occurs when traffic suddenly increases, users start reporting slow responses and API failures, but the Horizontal Pod Autoscaler (HPA) does not create additional Pods.

This scenario is frequently asked in DevOps, SRE, Platform Engineering, and Kubernetes interviews because it tests your understanding of autoscaling, metrics collection, resource management, and cluster capacity planning.

---

# Scenario

## Problem Statement

During peak business hours, traffic suddenly increases by 10x.

Users report:

- Slow application responses
- Request timeouts
- API failures

Monitoring shows:

✅ Existing Pods Running

✅ Nodes Healthy

✅ Cluster Healthy

✅ HPA Created

However, new Pods are NOT being created.

---

# Interview Question

A production application experiences a sudden traffic spike.

You verify:

- Pods are running
- Cluster is healthy
- HPA exists
- CPU usage is high

Yet HPA is not scaling.

How would you troubleshoot this issue?

---

# Expected Areas to Cover

## Horizontal Pod Autoscaler (HPA)

- CPU Metrics
- Memory Metrics
- Scaling Policies
- Min Replicas
- Max Replicas
- HPA Conditions

## Kubernetes Resources

- Resource Requests
- Resource Limits
- Deployment Configuration
- Scheduling

## Monitoring

- Metrics Server
- Prometheus
- Custom Metrics

## Infrastructure

- Node Capacity
- Cluster Autoscaler
- Scheduler

---

# Step 1: Check HPA Status

First verify whether HPA is receiving metrics.

## Commands

```bash
kubectl get hpa

kubectl describe hpa payment-api
```

Example Output:

```text
NAME          REFERENCE                 TARGETS     MINPODS   MAXPODS   REPLICAS
payment-api   Deployment/payment-api    20%/70%     2         10        2
```

---

## Verify

- Current CPU Utilization
- Memory Utilization
- Desired Replicas
- Current Replicas
- Events
- Scaling Conditions

---

## Common Problem

```text
Current Metrics = Unknown
```

Result:

```text
HPA Cannot Calculate Scaling
```

---

# Step 2: Verify Metrics Server

HPA depends on Metrics Server.

Without metrics, HPA cannot make scaling decisions.

## Commands

```bash
kubectl get pods -n kube-system

kubectl top pods

kubectl top nodes
```

---

## Verify

- Metrics Server Running
- Metrics Available
- CPU Usage Visible
- Memory Usage Visible

---

## Example Failure

```bash
kubectl top pods
```

Output:

```text
error: metrics API not available
```

---

## Root Cause

Metrics Server unavailable.

Result:

```text
No Metrics
↓
No HPA Decision
↓
No Scaling
```

---

# Step 3: Check Metrics Server Logs

If Metrics Server exists but metrics are unavailable:

## Commands

```bash
kubectl logs -n kube-system deployment/metrics-server
```

Look for:

- TLS Errors
- API Connectivity Issues
- Authentication Errors
- Timeout Errors

---

# Step 4: Verify Resource Requests

This is one of the most common interview questions.

HPA calculates scaling using CPU requests.

Without CPU requests, HPA cannot determine utilization percentage.

---

## Commands

```bash
kubectl get deploy payment-api -o yaml
```

---

## Correct Configuration

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"

  limits:
    cpu: "500m"
    memory: "512Mi"
```

---

## Wrong Configuration

```yaml
resources:
  limits:
    cpu: "500m"
```

Missing:

```yaml
requests:
```

---

## Result

```text
No CPU Requests
↓
HPA Cannot Calculate CPU %
↓
No Scaling
```

---

# Interview Gold Question

## Why are CPU Requests Mandatory for HPA?

### Answer

HPA calculates:

```text
CPU Utilization % =
Current CPU Usage ÷ CPU Request
```

Without CPU requests, utilization cannot be calculated.

---

# Step 5: Verify HPA Target Threshold

Check whether the scaling threshold is realistic.

## Command

```bash
kubectl describe hpa payment-api
```

---

Example:

```yaml
targetCPUUtilizationPercentage: 90
```

Current Usage:

```text
CPU = 65%
```

Result:

```text
Threshold Not Reached
↓
No Scaling
```

---

# Step 6: Check Min and Max Replicas

Verify HPA limits.

## Example

```yaml
minReplicas: 2
maxReplicas: 5
```

Current:

```text
Replicas = 5
```

---

Result:

```text
Maximum Replicas Reached
↓
No Additional Scaling
```

---

# Step 7: Check Deployment Events

Inspect deployment and cluster events.

## Commands

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Look for:

- Scaling Failures
- Scheduler Errors
- Resource Constraints
- Admission Controller Errors

---

# Example Event

```text
FailedScheduling:
0/3 nodes available
Insufficient CPU
```

---

# Root Cause

HPA requested Pods.

Scheduler could not place Pods.

---

# Step 8: Verify Node Capacity

Even if HPA creates Pods, nodes must have available resources.

## Commands

```bash
kubectl top nodes
```

---

Example:

```text
CPU Usage: 98%
Memory Usage: 95%
```

---

Result

```text
No Capacity Available
↓
Pods Stay Pending
```

---

# Step 9: Verify Cluster Autoscaler

If nodes are full, Cluster Autoscaler should add more nodes.

## Verify

```bash
kubectl get pods -n kube-system
```

Look for:

```text
cluster-autoscaler
```

---

## Check Logs

```bash
kubectl logs <cluster-autoscaler-pod> -n kube-system
```

---

## Common Issue

```text
Autoscaler Disabled
```

Result:

```text
No New Nodes
↓
No Pod Scheduling
```

---

# Real Production Flow

```text
Traffic Spike
      ↓
Application Pods
      ↓
Metrics Server
      ↓
HPA
      ↓
Deployment
      ↓
Scheduler
      ↓
Node Capacity
      ↓
New Pods Created
```

If any component fails, scaling stops.

---

# Common Root Causes

| Issue | Result |
|---------|---------|
| Metrics Server Missing | No Scaling |
| Metrics Server Down | No Scaling |
| CPU Requests Missing | HPA Doesn't Work |
| Wrong Threshold | No Scaling |
| Max Replicas Reached | No More Pods |
| Node Capacity Exhausted | Pending Pods |
| Scheduler Failure | No Scaling |
| HPA Misconfiguration | No Scaling |
| Cluster Autoscaler Disabled | No New Nodes |

---

# Real Production Scenario

## Problem

Traffic increases 10x.

Users report:

```text
API Timeout
```

Pods:

```text
Running
```

HPA:

```text
Exists
```

But replicas remain unchanged.

---

## Investigation

Check HPA:

```bash
kubectl describe hpa payment-api
```

Output:

```text
current CPU utilization: unknown
```

---

Check Metrics:

```bash
kubectl top pods
```

Output:

```text
error: metrics API not available
```

---

## Root Cause

Metrics Server was down.

---

## Result

```text
No Metrics
↓
HPA Cannot Calculate Utilization
↓
No Scaling
```

---

## Resolution

Restart Metrics Server.

```bash
kubectl rollout restart deployment metrics-server -n kube-system
```

Metrics returned.

HPA started scaling Pods.

---

# Scenario-Based Interview Questions

## Scenario 1

### Question

Traffic increases rapidly.

CPU usage reaches 95%.

HPA does not scale.

What is your first check?

### Answer

Verify Metrics Server.

```bash
kubectl top pods
```

No metrics means no HPA decisions.

---

## Scenario 2

### Question

HPA shows:

```text
CPU Utilization: Unknown
```

What could be wrong?

### Answer

- Metrics Server Down
- CPU Requests Missing
- Metrics API Failure

---

## Scenario 3

### Question

HPA wants 10 replicas but only 4 exist.

Why?

### Answer

Possible causes:

- Node Capacity Exhausted
- Scheduler Failure
- Cluster Autoscaler Disabled

---

## Scenario 4

### Question

Traffic is high but CPU usage remains low.

HPA never scales.

Why?

### Answer

Application bottleneck may be:

- Database
- Network
- External API

CPU-based scaling will not trigger.

---

## Scenario 5

### Question

Can HPA scale based on memory?

### Answer

Yes.

Example:

```yaml
metrics:
- type: Resource
  resource:
    name: memory
```

---

## Scenario 6

### Question

What happens if Metrics Server goes down?

### Answer

HPA loses visibility into resource usage.

Result:

```text
No Metrics
↓
No Scaling
```

---

# Follow-Up Interview Questions

## How does HPA calculate scaling?

### Answer

```text
Current Usage ÷ Requested Resource
```

Compared against target utilization.

---

## Why are CPU Requests mandatory?

### Answer

HPA calculates utilization using CPU requests as the baseline.

---

## Difference Between HPA and Cluster Autoscaler?

### HPA

Scales Pods.

### Cluster Autoscaler

Scales Nodes.

---

## Can HPA scale using custom metrics?

### Answer

Yes.

Examples:

- Requests Per Second
- Queue Length
- Kafka Lag
- Prometheus Metrics

---

# Interview Tip

Most engineers start with:

```bash
kubectl get hpa
```

Senior engineers follow:

```text
Traffic
↓
Metrics
↓
HPA
↓
Deployment
↓
Scheduler
↓
Nodes
↓
Root Cause
```

This demonstrates a production-ready troubleshooting mindset.

---

# Key Takeaway

## HPA Relies on Metrics

Always verify:

✅ Metrics Server

✅ CPU Requests

✅ HPA Configuration

✅ Scaling Events

✅ Deployment Configuration

✅ Scheduler

✅ Node Capacity

✅ Cluster Autoscaler

Remember:

```text
No Metrics
=
No Scaling
```

A healthy HPA is not determined by its existence, but by its ability to receive metrics, evaluate thresholds, and successfully create new Pods during traffic spikes.
