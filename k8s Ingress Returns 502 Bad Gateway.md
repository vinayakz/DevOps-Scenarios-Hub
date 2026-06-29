# Kubernetes Production Scenario: Ingress Returns 502 Bad Gateway

One of the most common production incidents in Kubernetes is when a deployment completes successfully, Pods are running, but users are unable to access the application through the Ingress.

Instead of seeing the application, users receive a **502 Bad Gateway** error.

This scenario is frequently asked in DevOps, SRE, Platform Engineering, and Kubernetes interviews because it tests your understanding of Kubernetes networking, Service discovery, Ingress configuration, and production troubleshooting.

---

# Scenario

## Problem Statement

It's Friday evening, and a production deployment has just completed successfully.

Monitoring shows:

- ✅ Pods are Running
- ✅ Deployment is Healthy
- ✅ Service Exists
- ✅ Ingress Resource Exists
- ✅ DNS Resolves Correctly

However, users report that the application is unavailable.

The browser displays:

```text
502 Bad Gateway
```

NGINX Ingress logs show:

```text
upstream connection failed
```

---

# Interview Question

A production deployment completed successfully.

Pods are healthy.

Service exists.

Ingress is configured.

Users receive **502 Bad Gateway**.

How would you troubleshoot this issue?

Where would you start?

How would you identify the root cause?

---

# Expected Areas to Cover

## Ingress

- Rules
- Host
- Paths
- Backend Service
- Annotations
- TLS Configuration

## Service

- ClusterIP
- Port
- TargetPort
- Selectors
- Endpoints

## Pods

- Running Status
- Readiness Probe
- Listening Port
- Application Logs

## Networking

- DNS Resolution
- Network Policies
- Connectivity

---

# Step 1: Verify the Ingress

Ingress is the entry point for external traffic.

## Commands

```bash
kubectl get ingress

kubectl describe ingress app-ingress
```

---

## Verify

- Hostname
- Path Rules
- Backend Service
- TLS Configuration
- Ingress Class
- Annotations

Example:

```yaml
rules:
- host: app.company.com
  http:
    paths:
    - path: /
      backend:
        service:
          name: app-service
          port:
            number: 80
```

---

## Common Issue

Ingress points to the wrong backend Service.

Result:

```text
User
 ↓
Ingress
 ↓
Wrong Backend
 ↓
502 Bad Gateway
```

---

# Step 2: Verify the Service

The Service routes traffic from Ingress to the Pods.

## Commands

```bash
kubectl get svc

kubectl describe svc app-service
```

---

## Verify

- Service Port
- TargetPort
- Selectors
- ClusterIP

Example:

```yaml
ports:
- port: 80
  targetPort: 8080
```

---

## Common Issue

Incorrect TargetPort.

Result:

```text
Ingress
 ↓
Service
 ↓
Wrong Port
 ↓
502 Bad Gateway
```

---

# Step 3: Verify Endpoints

Endpoints tell us whether the Service has discovered healthy Pods.

## Commands

```bash
kubectl get endpoints app-service

kubectl describe endpoints app-service
```

---

## Healthy Output

```text
NAME           ENDPOINTS
app-service    10.1.1.10:8080
               10.1.1.11:8080
```

---

## Problem Output

```text
NAME           ENDPOINTS
app-service    <none>
```

---

## Root Cause

No healthy Pods are attached to the Service.

Result:

```text
Ingress
 ↓
Service
 ↓
No Endpoints
 ↓
502 Bad Gateway
```

---

# Interview Gold Question

## What happens if Endpoints are empty?

### Answer

The Service has no healthy backend Pods.

Ingress cannot forward traffic.

Users receive **502 Bad Gateway** or **503 Service Unavailable**, depending on the Ingress controller.

---

# Step 4: Verify Pod Labels

Services discover Pods using labels.

## Commands

```bash
kubectl get pods --show-labels
```

Verify:

```yaml
app=payment-api
env=prod
```

Compare with:

```yaml
selector:
  app: payment-api
  env: prod
```

---

## Common Issue

Selectors do not match Pod labels.

Result:

```text
No Endpoints

↓

502 Bad Gateway
```

---

# Step 5: Verify Container Port

One of the most common production mistakes.

Example:

Container:

```yaml
containerPort: 8080
```

Service:

```yaml
targetPort: 80
```

Traffic Flow:

```text
Ingress
 ↓
Service
 ↓
Port 80
 ↓
Application Listening on 8080
 ↓
Connection Refused
 ↓
502 Bad Gateway
```

---

# Step 6: Verify Application Logs

If networking looks correct, inspect the application.

## Commands

```bash
kubectl logs deployment/app

kubectl logs <pod-name>
```

---

## Verify

- Application Started
- Listening Port
- Startup Errors
- Database Connectivity
- Runtime Exceptions

Example:

```text
ERROR

Application failed to bind port 8080
```

---

# Step 7: Test the Service Directly

Bypass the Ingress.

## Commands

```bash
kubectl port-forward svc/app-service 8080:80
```

Open:

```text
http://localhost:8080
```

---

## Result

If the Service fails even with port-forwarding:

Root cause is behind the Ingress.

Possible causes:

- Application
- Service
- Endpoints
- TargetPort

---

# Step 8: Check Readiness Probe

Pods may be Running but not Ready.

## Commands

```bash
kubectl describe pod <pod-name>
```

Verify:

- Readiness Probe
- Health Endpoint
- Probe Port

Example:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
```

---

## Result

```text
Running

Not Ready

↓

Service Removes Pod

↓

No Endpoints

↓

502 Bad Gateway
```

---

# Step 9: Check Network Policies

Network policies may block traffic between the Ingress Controller and the application Pods.

## Commands

```bash
kubectl get networkpolicy -A

kubectl describe networkpolicy <policy-name>
```

Verify:

- Ingress Rules
- Egress Rules
- Allowed Ports

---

# Real Production Flow

```text
User
   ↓
DNS
   ↓
Ingress
   ↓
Service
   ↓
Endpoints
   ↓
Pods
   ↓
Application
```

A failure anywhere in this path can result in:

```text
502 Bad Gateway
```

---

# Common Root Causes

| Issue | Result |
|--------|--------|
| Wrong Backend Service | 502 |
| Wrong TargetPort | 502 |
| Empty Endpoints | 502 |
| Incorrect Pod Labels | 502 |
| Readiness Probe Failed | 502 |
| Application Not Listening | 502 |
| Network Policy Blocking Traffic | 502 |
| Container Port Mismatch | 502 |
| Ingress Annotation Error | 502 |

---

# Real Production Scenario

## Problem

A new application version is deployed.

Deployment succeeds.

Users immediately report:

```text
502 Bad Gateway
```

---

## Investigation

Check Pods:

```bash
kubectl get pods
```

Output:

```text
Running
```

---

Check Ingress:

```bash
kubectl describe ingress app-ingress
```

Looks correct.

---

Check Service:

```bash
kubectl describe svc app-service
```

Output:

```yaml
targetPort: 80
```

---

Check Deployment:

```yaml
containerPort: 8080
```

---

## Root Cause

The Service was forwarding traffic to port **80**, while the application was listening on **8080**.

Traffic never reached the application.

---

## Resolution

Update the Service:

```yaml
targetPort: 8080
```

Redeploy the Service.

Users could access the application successfully.

---

# Scenario-Based Interview Questions

## Scenario 1

### Question

Pods are Running.

Ingress returns **502 Bad Gateway**.

What is your first check?

### Answer

Verify:

- Ingress Backend
- Service
- Endpoints

---

## Scenario 2

### Question

Endpoints are empty.

What does that indicate?

### Answer

The Service cannot discover healthy Pods.

Possible causes:

- Label mismatch
- Readiness failure

---

## Scenario 3

### Question

Pods are Ready.

Endpoints exist.

Service is healthy.

Still getting 502.

What next?

### Answer

Check:

- TargetPort
- ContainerPort
- Application Logs

---

## Scenario 4

### Question

Ingress logs show:

```text
upstream connection refused
```

What does it mean?

### Answer

Ingress reached the backend Service, but the application refused the TCP connection.

Possible reasons:

- Wrong TargetPort
- Application not listening
- Crash after startup

---

## Scenario 5

### Question

Pods are Running but Service has no Endpoints.

Why?

### Answer

Possible reasons:

- Wrong labels
- Failed Readiness Probe
- Incorrect Service selector

---

# Follow-Up Interview Questions

## What is the difference between 502 and 503?

### 502 Bad Gateway

Ingress reached the backend but could not establish a valid connection.

Examples:

- Wrong TargetPort
- Connection Refused
- Backend Failure

### 503 Service Unavailable

No healthy backend Pods are available.

Examples:

- Empty Endpoints
- Pods Not Ready
- No Available Replicas

---

## Why does Ingress return 502?

Because it cannot establish a successful connection to the backend Service or application.

---

## How do Services discover Pods?

Using label selectors.

---

## Can Pods be Running but still fail?

Yes.

Examples:

- Readiness Probe Failure
- Wrong Listening Port
- Application Startup Failure

---

## Which logs should you check first?

1. Ingress Controller Logs
2. Application Logs
3. Kubernetes Events

---

# Interview Tip

Most candidates immediately run:

```bash
kubectl logs
```

Senior DevOps Engineers troubleshoot in this order:

```text
Ingress
   ↓
Service
   ↓
Endpoints
   ↓
TargetPort
   ↓
Pods
   ↓
Application
   ↓
Root Cause
```

This demonstrates strong Kubernetes networking knowledge and a production-ready troubleshooting mindset.

---

# Key Takeaway

## Running Pods ≠ Accessible Application

Always verify:

- ✅ Ingress Configuration
- ✅ Backend Service
- ✅ Service Port and TargetPort
- ✅ Endpoints
- ✅ Pod Labels
- ✅ Readiness Probe
- ✅ Application Logs
- ✅ Network Policies

Remember:

```text
User
 ↓
DNS
 ↓
Ingress
 ↓
Service
 ↓
Endpoints
 ↓
Pods
 ↓
Application
```

A failure at any layer in this path can result in a **502 Bad Gateway** error.

Understanding this traffic flow is essential for diagnosing real-world Kubernetes production incidents.
