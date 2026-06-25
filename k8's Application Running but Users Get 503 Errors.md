# Kubernetes Production Scenario: Application Running but Users Get 503 Errors

One of the most common production incidents in Kubernetes is when the application appears healthy, all pods are running, deployments are healthy, but users continuously receive **503 Service Unavailable** errors.

This scenario is frequently asked in DevOps, SRE, Platform Engineering, and Kubernetes interviews because it tests troubleshooting methodology rather than command memorization.

---

# Scenario

## Problem Statement

A production application is returning:

```text
503 Service Unavailable
```

Initial checks show:

✅ Pods are Running

✅ Deployment is Healthy

✅ No CrashLoopBackOff

✅ No OOMKilled Events

✅ Containers Started Successfully

Yet users cannot access the application.

---

# Interview Question

A production application is receiving 503 errors.

You verify:

- Pods are Running
- Deployment is Healthy
- No CrashLoopBackOff
- No OOMKilled Events

How would you troubleshoot this issue?

---

# Step 1: Check Ingress Configuration

Ingress is the first entry point for external traffic.

## Commands

```bash
kubectl get ingress

kubectl describe ingress <ingress-name>
```

## Verify

- Host configuration
- Path rules
- Backend service mapping
- TLS configuration
- Ingress controller status

## Example

```yaml
rules:
  - host: app.company.com
    http:
      paths:
        - path: /
          backend:
            service:
              name: payment-service
              port:
                number: 80
```

## Common Issue

Ingress points to the wrong service.

Result:

```text
User → Ingress → Service Not Found → 503
```

---

# Step 2: Check Service Configuration

Services route traffic to Pods.

## Commands

```bash
kubectl get svc

kubectl describe svc payment-service
```

## Verify

- Service Port
- TargetPort
- Selector Labels
- Service Type

## Example

```yaml
spec:
  selector:
    app: payment-service

  ports:
    - port: 80
      targetPort: 8080
```

## Common Issue

Incorrect Service selector.

Result:

```text
Service cannot discover Pods.
```

---

# Step 3: Check Endpoints

Endpoints tell you whether a Service has healthy backend Pods.

## Commands

```bash
kubectl get endpoints

kubectl describe endpoints payment-service
```

## Healthy Output

```bash
NAME               ENDPOINTS
payment-service    10.244.1.12:8080
```

## Problem Output

```bash
NAME               ENDPOINTS
payment-service    <none>
```

---

# Interview Gold Question

## What happens if Endpoints are empty?

### Answer

When Endpoints are empty:

- Service cannot route traffic
- Ingress cannot find backend Pods
- Users receive 503 errors

Even though Pods are Running.

---

# Step 4: Verify Pod Labels

Services discover Pods through label selectors.

## Commands

```bash
kubectl get pods --show-labels
```

## Example

Pod Labels:

```yaml
app=payment-service
env=prod
```

Service Selector:

```yaml
selector:
  app: payment-service
  env: prod
```

## Common Issue

Labels do not match selectors.

Result:

```text
Service → No Matching Pods
Endpoints → Empty
Ingress → 503
```

---

# Step 5: Check Readiness Probe

This is one of the most common causes of production 503 errors.

A Pod can be:

```text
Running = YES
Ready = NO
```

## Commands

```bash
kubectl describe pod <pod-name>
```

## Verify

- Readiness Probe Status
- HTTP Response Codes
- Probe Path
- Probe Port

## Example

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
```

## Common Production Issue

Application starts slowly.

Probe keeps failing.

Pod never becomes Ready.

Result:

```text
Pod Running
Pod Not Ready
Endpoints Empty
Ingress Returns 503
```

---

# Step 6: Verify Application Ports

Port mismatches are a frequent production issue.

## Verify

| Component | Port |
|------------|--------|
| Container Port | 8080 |
| TargetPort | 8080 |
| Service Port | 80 |
| Ingress Backend Port | 80 |

---

## Common Mistake

Container:

```yaml
containerPort: 8080
```

Service:

```yaml
targetPort: 9090
```

Result:

```text
Ingress → Service → Wrong Port → 503
```

---

# Step 7: Check Application Logs

If everything appears correct, inspect the application itself.

## Commands

```bash
kubectl logs <pod-name>

kubectl logs -f <pod-name>
```

## Verify

- Application Startup
- Database Connectivity
- External API Connectivity
- Port Binding
- Runtime Errors

## Example

```text
ERROR:
Database connection refused
```

The application may be running but unable to serve traffic.

---

# Production Troubleshooting Flow

A Senior DevOps Engineer follows a structured workflow:

```text
Ingress
   ↓
Service
   ↓
Endpoints
   ↓
Readiness Probe
   ↓
Pods
   ↓
Application Logs
```

This approach identifies root causes much faster than randomly checking logs.

---

# Common Root Causes of 503 Errors

| Issue | Result |
|---------|---------|
| Readiness Probe Failed | 503 |
| Endpoints Empty | 503 |
| Wrong Service Selector | 503 |
| Port Mismatch | 503 |
| Wrong Ingress Backend | 503 |
| Application Not Listening | 503 |
| Database Dependency Failure | 503 |
| TLS Misconfiguration | 503 |

---

# Real Production Scenario

## Situation

Users report:

```text
503 Service Unavailable
```

Check Pods:

```bash
kubectl get pods
```

Output:

```text
payment-app-7b5f89c6f4 Running
```

Looks healthy.

Check Endpoints:

```bash
kubectl get endpoints payment-service
```

Output:

```text
payment-service   <none>
```

Check Pod:

```bash
kubectl describe pod payment-app
```

Output:

```text
Readiness probe failed:
HTTP probe failed with status code 500
```

## Root Cause

Application was running but not Ready.

Service excluded the Pod.

Endpoints became empty.

Ingress returned 503.

---

# Advanced Scenario-Based Interview Questions

## Scenario 1

### Question

Pods are Running.

Endpoints are Empty.

What is the first thing you check?

### Answer

Verify Service selectors and Pod labels.

```bash
kubectl get svc
kubectl get pods --show-labels
```

---

## Scenario 2

### Question

Endpoints exist but users still get 503.

What could be wrong?

### Answer

Possible causes:

- Wrong TargetPort
- Application not listening on expected port
- Ingress backend misconfiguration
- TLS issue

---

## Scenario 3

### Question

Pods are Running and Ready.

Service is healthy.

Ingress returns 503.

What next?

### Answer

Check:

```bash
kubectl describe ingress
kubectl logs -n ingress-nginx <controller-pod>
```

Common causes:

- Wrong backend service
- Invalid host rule
- Ingress controller issue

---

## Scenario 4

### Question

A deployment rollout completed successfully but traffic immediately starts returning 503 errors.

What could be the reason?

### Answer

The new version may fail readiness checks.

Result:

```text
Pods Running
Pods Not Ready
Endpoints Empty
503 Errors
```

---

## Scenario 5

### Question

How can a Pod be Running but unavailable?

### Answer

Because Running only means the container process exists.

The Pod may still fail:

- Readiness Probe
- Dependency Checks
- Database Connections
- Startup Validation

Running ≠ Ready

---

# Follow-Up Interview Questions

## What is the difference between Running and Ready?

### Running

Container process exists.

### Ready

Pod can receive production traffic.

---

## Why does Ingress return 503?

Because no healthy backend endpoints are available.

---

## What are Kubernetes Endpoints?

Endpoints are the list of healthy Pods associated with a Service.

---

## How does a Service discover Pods?

Using label selectors.

---

## Can Pods be Running but unavailable?

Yes.

Common reasons:

- Readiness probe failure
- Dependency failure
- Wrong application port
- Initialization issues

---

# Interview Tip

Most engineers start with:

```bash
kubectl logs
```

Senior DevOps Engineers start with:

```text
Ingress
→ Service
→ Endpoints
→ Readiness
→ Pods
→ Logs
```

This demonstrates a production troubleshooting mindset and systematic debugging approach.

---

# Key Takeaway

## Running Pods ≠ Healthy Application

Always verify:

- Ingress Configuration
- Service Mapping
- Endpoints
- Pod Labels
- Readiness Probe
- Application Logs

Before blaming the application.

A healthy Kubernetes application is not defined by Running Pods, but by Ready Pods serving traffic through valid Endpoints.

🚀 That's the difference between a Kubernetes Administrator and a Production-Ready DevOps Engineer.
