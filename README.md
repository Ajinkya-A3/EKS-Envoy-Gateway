# Envoy Gateway on Amazon EKS — Complete Setup Guide

> A step-by-step guide to deploying Envoy Gateway on EKS with AWS Load Balancer Controller (NLB), including explanation of why each piece exists and how they connect.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Understanding Pod Containers — What 2/2 Means](#understanding-pod-containers--what-22-means)
- [Prerequisites](#prerequisites)
- [How It Works — The Big Picture](#how-it-works--the-big-picture)
- [Step 1 — Install Gateway API CRDs](#step-1--install-gateway-api-crds)
- [Step 2 — Install Envoy Gateway](#step-2--install-envoy-gateway)
- [Step 3 — Configure EnvoyProxy (NLB + HPA + Resources)](#step-3--configure-envoyproxy-nlb--hpa--resources)
- [Step 4 — Create GatewayClass](#step-4--create-gatewayclass)
- [Step 5 — Create the Gateway](#step-5--create-the-gateway)
- [Step 6 — Deploy a Sample App and HTTPRoute](#step-6--deploy-a-sample-app-and-httproute)
- [Step 7 — Verify Everything Works](#step-7--verify-everything-works)
- [HPA Behaviour and Scaling Explained](#hpa-behaviour-and-scaling-explained)
- [Resource Sizing and Throughput Guide](#resource-sizing-and-throughput-guide)
- [Common Problem: LoadBalancer Stuck in Pending](#common-problem-loadbalancer-stuck-in-pending)
- [Subnet Tagging Reference](#subnet-tagging-reference)
- [Troubleshooting Commands](#troubleshooting-commands)
- [Architecture Diagram](#architecture-diagram)
- [Quick Setup Checklist](#quick-setup-checklist)

---

## Architecture Overview

```
Internet
   │
   ▼
AWS NLB  (provisioned by AWS Load Balancer Controller)
   │
   ▼
Envoy Proxy Pods  (managed by Envoy Gateway, autoscaled by HPA)
   │
   ▼
Your Application Services  (via HTTPRoute rules)
```

**Key components:**

| Component | Role |
|---|---|
| **Envoy Gateway** | The control plane — watches Gateway API resources and manages Envoy Proxy pods |
| **Envoy Proxy** | The data plane — the actual proxy that routes traffic |
| **GatewayClass** | Tells Kubernetes which controller (Envoy Gateway) owns which Gateways |
| **EnvoyProxy CRD** | Custom config for the Envoy service — NLB annotations, resources, and HPA live here |
| **Gateway** | Declares listeners (ports/protocols) and references the EnvoyProxy config |
| **HTTPRoute** | Defines routing rules (paths, headers, backends) attached to a Gateway |
| **AWS LBC** | Watches the Envoy service and provisions the actual NLB in AWS |
| **HPA** | Automatically scales proxy pod count based on CPU utilization |

---

## Understanding Pod Containers — What 2/2 Means

This is one of the most commonly misunderstood parts of `kubectl get pods` output.

```
NAME                                                READY   STATUS
envoy-gateway-external-gateway-4b0681b0-xxx-79gmv   2/2     Running
envoy-gateway-external-gateway-4b0681b0-xxx-625bn   2/2     Running
envoy-gateway-external-gateway-4b0681b0-xxx-z6pd5   2/2     Running
```

### What `2/2` actually means

`READY` shows `running_containers / total_containers` **inside a single pod**. It does NOT mean 2 pods — it means 2 containers sharing the same pod.

```
Pod: envoy-gateway-external-gateway-xxx-79gmv   ← this is ONE pod
│
├── Container 1: envoy            (main proxy process)
└── Container 2: shutdown-manager (graceful drain sidecar)
```

### The Two Containers Explained

**`envoy` (main container)**
The actual Envoy proxy process. This is what receives traffic from the NLB, applies your HTTPRoute rules, and forwards requests to your application services.

**`shutdown-manager` (sidecar container)**
A lightweight sidecar injected by Envoy Gateway. Its only job is to manage graceful shutdown — it coordinates connection draining between the NLB deregistration window and the Envoy process shutdown. Without it, in-flight requests would be dropped during rolling updates or scale-down events.

### So your actual pod count

With `minReplicas: 1` you have:

```
Deployment
└── 1 Pod  (HPA scales this count up to 5)
    ├── Container 1: envoy            → handles traffic
    └── Container 2: shutdown-manager → handles graceful shutdown
                                        READY = 2/2
```

Total containers running = 2 per pod. Pods = 1 to 5 depending on load. The HPA controls pod count, not container count — containers inside a pod are fixed by the spec.

---

## Prerequisites

Before starting, confirm you have the following in place:

- [x] EKS cluster running (Kubernetes 1.26+)
- [x] `kubectl` configured for your cluster
- [x] OIDC provider associated with your cluster
- [x] IRSA configured for AWS Load Balancer Controller
- [x] AWS Load Balancer Controller installed (`aws-load-balancer-controller` in `kube-system`)
- [x] VPC CNI plugin installed and configured
- [x] `helm` v3 installed locally
- [x] Public/private subnets properly tagged (see [Subnet Tagging](#subnet-tagging-reference))
- [x] Metrics Server installed (required for HPA CPU metrics)

**Verify AWS Load Balancer Controller is running:**

```bash
kubectl get deployment aws-load-balancer-controller -n kube-system
# Expected: READY 1/1 or 2/2
```

**Verify Metrics Server is running (required for HPA):**

```bash
kubectl get deployment metrics-server -n kube-system
# Expected: READY 1/1
# If missing: kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## How It Works — The Big Picture

### Why does LoadBalancer get stuck in Pending?

When Envoy Gateway creates a `Service` of type `LoadBalancer`, Kubernetes' in-tree cloud controller tries to provision a Classic Load Balancer (CLB). On modern EKS clusters this fails silently — the CLB never gets created and the service stays `<pending>`.

The fix is to tell the AWS Load Balancer Controller (not the in-tree controller) to handle this service by adding the annotation:

```
service.beta.kubernetes.io/aws-load-balancer-type: "external"
```

This signals to the in-tree controller: *"ignore this service, someone else will handle it"* — and the AWS LBC picks it up and creates a proper Network Load Balancer (NLB).

### Why NLB and not ALB?

Envoy Gateway operates at Layer 7 (HTTP). Putting an ALB (also Layer 7) in front of it duplicates functionality and adds cost. An NLB operates at Layer 4 (TCP), so it simply forwards raw TCP connections to Envoy, which then handles all the HTTP routing. This is the recommended pattern.

### Where do annotations go?

Annotations do **not** go on the `Gateway` resource's metadata. Envoy Gateway creates its own `Service` object internally — you inject configuration into that generated service via the **`EnvoyProxy` CRD**, which the `Gateway` references.

```
EnvoyProxy (annotations + resources + HPA) ← referenced by → Gateway ← matched by → GatewayClass
```

---

## Step 1 — Install Gateway API CRDs

Envoy Gateway uses the Kubernetes Gateway API, which is not installed by default.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```

**What this installs:**
- `GatewayClass` CRD
- `Gateway` CRD
- `HTTPRoute` CRD
- `ReferenceGrant` CRD

Verify:

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

---

## Step 2 — Install Envoy Gateway

Install Envoy Gateway via Helm:

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.3.1 \
  -n envoy-gateway-system \
  --create-namespace
```

**What this creates:**
- `envoy-gateway` Deployment (the control plane)
- Required RBAC (ClusterRole, ClusterRoleBinding)
- `EnvoyProxy` CRD
- Webhooks for validating Gateway API resources

Wait for it to be ready:

```bash
kubectl rollout status deployment/envoy-gateway -n envoy-gateway-system
```

---

## Step 3 — Configure EnvoyProxy (NLB + HPA + Resources)

This is the most important step. It controls the proxy pod resources, autoscaling behaviour, graceful shutdown, and NLB provisioning — all in one place.

Create `envoy-proxy-config.yaml`:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: external-proxy-config
  namespace: gateway
spec:
  provider:
    type: Kubernetes
    kubernetes:

      # ── Deployment configuration ──────────────────────────────────────
      envoyDeployment:

        # Patch the generated Deployment using StrategicMerge.
        # This targets the shutdown-manager sidecar container specifically
        # and adds a preStop hook for graceful connection draining.
        patch:
          type: StrategicMerge
          value:
            spec:
              template:
                spec:
                  containers:
                    - name: shutdown-manager
                      lifecycle:
                        preStop:
                          exec:
                            # Sleep 120s on shutdown.
                            # NLB deregistration takes up to 60s — this gives
                            # a full 60s extra buffer for in-flight requests
                            # to complete before Envoy actually stops.
                            command: ["/bin/sh", "-c", "sleep 120"]

        # Resource requests and limits for the main envoy container.
        # shutdown-manager sidecar uses negligible resources (<10m CPU, <32Mi).
        container:
          resources:
            requests:
              cpu: 250m       # Guaranteed CPU for scheduling
              memory: 512Mi   # Guaranteed memory for scheduling
            limits:
              # No CPU limit — intentional.
              # CPU throttling causes latency spikes in Envoy.
              # HPA handles scale-out instead of relying on limits.
              memory: 1Gi     # Hard ceiling — prevents OOMKill during bursts

      # ── Horizontal Pod Autoscaler ─────────────────────────────────────
      envoyHpa:
        # Minimum 1 pod — cost-efficient for low/no traffic periods.
        # Acceptable if you can tolerate ~30s cold-start when scaling from 0
        # or brief single-point-of-failure window at 1 pod.
        # Increase to 2 or 3 if you need HA at all times.
        minReplicas: 1

        # Maximum 5 pods.
        # At 250m CPU per pod, 5 pods = 1.25 vCPU total proxy capacity.
        # Caps cost while handling significant traffic spikes.
        maxReplicas: 5

        metrics:
          - type: Resource
            resource:
              name: cpu
              target:
                type: Utilization
                # Scale out when average CPU across pods hits 60% of request.
                # 60% of 250m = 150m actual CPU triggers a new pod.
                # Conservative enough to scale before latency degrades,
                # without scaling too aggressively on short spikes.
                averageUtilization: 60

      # ── NLB Service configuration ─────────────────────────────────────
      envoyService:
        annotations:
          # Stops the in-tree CLB provisioner.
          # AWS Load Balancer Controller takes over and creates an NLB instead.
          service.beta.kubernetes.io/aws-load-balancer-type: "external"

          # Creates a public-facing NLB in your public subnets.
          # Change to "internal" for private/internal traffic only.
          service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"

          # Route NLB traffic directly to Pod IPs via VPC CNI.
          # Lower latency than instance mode — bypasses kube-proxy and NodePort.
          service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip

          # NLB health check uses Envoy's built-in admin port (19002).
          # /healthz returns 200 only when Envoy is fully ready.
          # This prevents the NLB from sending traffic to starting/draining pods.
          service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: HTTP
          service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "19002"
          service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/healthz"
```

Apply it:

```bash
kubectl apply -f envoy-proxy-config.yaml
```

---

## Step 4 — Create GatewayClass

The `GatewayClass` is a cluster-scoped resource that registers Envoy Gateway as a controller.

Create `gatewayclass.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

Apply it:

```bash
kubectl apply -f gatewayclass.yaml
```

Verify it's accepted:

```bash
kubectl get gatewayclass eg
# Expected: ACCEPTED = True
```

---

## Step 5 — Create the Gateway

The `Gateway` declares what ports and protocols to listen on, and links to your `EnvoyProxy` config via `parametersRef`.

Create `gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: external-gateway
  namespace: gateway
spec:
  gatewayClassName: eg
  infrastructure:
    parametersRef:
      group: gateway.envoyproxy.io
      kind: EnvoyProxy
      name: external-proxy-config   # Must match metadata.name in Step 3
  listeners:
    - name: http
      protocol: HTTP
      port: 80

    # Optional: HTTPS listener
    # - name: https
    #   protocol: HTTPS
    #   port: 443
    #   tls:
    #     mode: Terminate
    #     certificateRefs:
    #       - name: my-tls-secret
```

Apply it:

```bash
kubectl apply -f gateway.yaml
```

**What happens automatically after apply:**

1. Envoy Gateway detects the new `Gateway`
2. Creates an `envoy-gateway-external-gateway-*` Deployment (proxy pods)
3. Creates an `envoy-gateway-external-gateway-*` Service of type `LoadBalancer` with your annotations
4. Creates an HPA targeting that Deployment (minReplicas: 1, maxReplicas: 5)
5. AWS Load Balancer Controller provisions an NLB
6. NLB DNS hostname is written back to `gateway.status.addresses`

Watch the Gateway get its address:

```bash
kubectl get gateway external-gateway -n gateway -w
# Wait until ADDRESS column is populated with an NLB hostname
```

---

## Step 6 — Deploy a Sample App and HTTPRoute

Deploy a test backend:

```yaml
# backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: gcr.io/k8s-staging-ingressconformance/echoserver:v20221109-6717e5c
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: gateway
spec:
  selector:
    app: backend
  ports:
    - port: 3000
      targetPort: 3000
```

Create the `HTTPRoute`:

```yaml
# httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend-route
  namespace: gateway
spec:
  parentRefs:
    - name: external-gateway
  hostnames:
    - "api.example.com"   # Replace with your domain
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: backend
          port: 3000
```

Apply both:

```bash
kubectl apply -f backend.yaml
kubectl apply -f httproute.yaml
```

---

## Step 7 — Verify Everything Works

```bash
# 1. Get the NLB hostname
export NLB_HOST=$(kubectl get gateway external-gateway \
  -n gateway \
  -o jsonpath='{.status.addresses[0].value}')
echo $NLB_HOST

# 2. Test with curl
curl -H "Host: api.example.com" http://$NLB_HOST/

# 3. Check all resources
kubectl get gatewayclass,gateway,httproute -A

# 4. Check HPA status
kubectl get hpa -n gateway

# 5. Check proxy pods (should show 2/2 READY)
kubectl get pods -n gateway

# 6. Check the Envoy service has an EXTERNAL-IP
kubectl get svc -n gateway
```

Expected HPA output:

```
NAME                              REFERENCE                        TARGETS   MINPODS   MAXPODS   REPLICAS
envoy-gateway-external-gateway    Deployment/envoy-external-gw     12%/60%   1         5         1
```

Expected pod output:

```
NAME                                                  READY   STATUS
envoy-gateway-external-gateway-4b0681b0-xxx-79gmv     2/2     Running
```

---

## HPA Behaviour and Scaling Explained

### How HPA works with Envoy Gateway

When you define `envoyHpa` in the `EnvoyProxy` CRD, Envoy Gateway automatically creates a Kubernetes `HPA` object targeting the proxy Deployment. HPA does not scale containers — it scales **pod count** by adjusting `Deployment.spec.replicas`.

```
HPA watches CPU metrics across all proxy pods
        │
        ▼
Average CPU across pods exceeds 60% of 250m request (= 150m)
        │
        ▼
HPA increases Deployment replicas (e.g. 1 → 2)
        │
        ▼
Deployment controller creates a new pod
        │
        ▼
New pod passes NLB health check (/healthz on port 19002)
        │
        ▼
NLB begins routing traffic to new pod
```

### Why minReplicas: 1

Setting minimum to 1 means the system runs a single proxy pod during idle or low-traffic periods. This keeps costs low. The trade-off is:

- At 1 pod, there is no redundancy — if that pod crashes or the node fails, there is a brief outage until a replacement starts (~15-30 seconds)
- Scaling from 1 → 2 takes 15-60 seconds for the new pod to become healthy and registered with the NLB

If your service requires zero downtime at all times, set `minReplicas: 2` or `3` (one per AZ).

### Why maxReplicas: 5

At 250m CPU per pod, 5 pods gives you 1.25 vCPU total proxy capacity. This is appropriate for moderate production traffic. Adjust based on your observed RPS:

| Max Replicas | Approx max RPS (HTTP) | Approx max RPS (TLS) |
|---|---|---|
| 3 | ~6,000 - 9,000 | ~1,500 - 2,400 |
| 5 | ~10,000 - 15,000 | ~2,500 - 4,000 |
| 10 | ~20,000 - 30,000 | ~5,000 - 8,000 |

### Why CPU target: 60%

```
60% of 250m request = 150m actual CPU triggers scale-out

Scale-out happens before Envoy is stressed (not at 80-90%)
New pod is ready before latency degrades
Short CPU spikes (< 3 minutes) do not trigger scaling
```

The HPA has a built-in stabilisation window of 3 minutes for scale-out and 5 minutes for scale-in by default. This prevents thrashing during brief traffic spikes.

### Scaling timeline

```
t=0    Traffic spike — CPU rises above 150m average
t=180  HPA stabilisation window passes — scale-out triggered
t=185  New pod starts (image already cached on node: ~5s)
t=200  Envoy container ready, /healthz returns 200
t=215  NLB health check passes — pod added to target group
t=220  New pod receives traffic
```

---

## Resource Sizing and Throughput Guide

### What 250m CPU and 512Mi handles

Envoy is a high-performance C++ proxy. Memory usage is predictable; CPU varies significantly based on what Envoy is doing.

| Workload | Est. RPS per pod (250m CPU) |
|---|---|
| Plain HTTP routing | 2,000 - 3,000 |
| TLS termination | 500 - 800 |
| TLS + JWT validation | 300 - 500 |
| TLS + header manipulation | 400 - 700 |

TLS handshakes are the dominant CPU cost. If you terminate TLS at Envoy, budget more CPU per pod or lower your HPA target.

### Memory breakdown (512Mi per pod)

```
512Mi total
├── ~50MB  Envoy base process
├── ~50MB  Route config (grows with number of HTTPRoutes)
├── ~100MB Connection buffers (~2,000 concurrent connections)
└── ~312MB headroom before 1Gi limit triggers OOMKill
```

### Why no CPU limit

CPU limits in Kubernetes work via CFS throttling — even if the node has spare CPU, a container at its limit gets paused. For a proxy this causes measurable latency spikes. By omitting the CPU limit, Envoy can burst to use spare node capacity during traffic spikes, and the HPA handles scaling instead of relying on throttling.

### Monitoring actual usage

```bash
# Current CPU and memory usage per pod
kubectl top pods -n gateway

# HPA current vs target
kubectl get hpa -n gateway

# If actual CPU is consistently above 150m → increase cpu request
# If actual memory is consistently above 400Mi → increase memory request
```

---

## Graceful Shutdown Flow

The combination of `preStop: sleep 120` and `deregistration_delay: 60` ensures zero dropped connections during pod termination.

```
t=0    Pod termination triggered (rolling update, scale-down, node drain)
t=0    preStop hook runs: sleep 120 begins
t=0    NLB begins deregistering this pod (60s drain window starts)
t=60   NLB stops routing NEW connections to this pod
t=60   All in-flight requests that arrived before t=60 are completing
t=120  preStop sleep ends — Kubernetes sends SIGTERM to containers
t=120  Envoy begins graceful shutdown (drains remaining connections)
t=125  All connections drained — pod exits cleanly

Result: Zero dropped connections ✓
```

---

## Common Problem: LoadBalancer Stuck in Pending

If the service `EXTERNAL-IP` shows `<pending>` for more than 3 minutes:

**Step 1 — Check AWS LBC logs:**

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller \
  --tail=100 | grep -iE "error|warn|envoy"
```

**Step 2 — Describe the Envoy service:**

```bash
kubectl get svc -n gateway
kubectl describe svc <envoy-service-name> -n gateway
```

**Step 3 — Common fixes:**

| Root Cause | Symptom | Fix |
|---|---|---|
| Missing `aws-load-balancer-type: external` | CLB creation attempt in logs | Add annotation to EnvoyProxy |
| Subnet not tagged | `no matching subnet found` in logs | Tag subnets (see below) |
| IAM permissions missing | `AccessDenied` in logs | Add ELB permissions to LBC role |
| Duplicate annotation keys | One attribute silently missing | Combine into single annotation value |
| VPC CNI not installed | `ip` target type fails | Use `instance` target type or install VPC CNI |
| Metrics Server missing | HPA shows `<unknown>` targets | Install metrics-server |

---

## Subnet Tagging Reference

The AWS Load Balancer Controller discovers subnets via tags. Without these, the NLB cannot be created.

**For internet-facing (public) NLB:**

```bash
aws ec2 create-tags \
  --resources <public-subnet-id-1> <public-subnet-id-2> \
  --tags \
    Key=kubernetes.io/role/elb,Value=1 \
    Key=kubernetes.io/cluster/<YOUR_CLUSTER_NAME>,Value=shared
```

**For internal (private) NLB:**

```bash
aws ec2 create-tags \
  --resources <private-subnet-id-1> <private-subnet-id-2> \
  --tags \
    Key=kubernetes.io/role/internal-elb,Value=1 \
    Key=kubernetes.io/cluster/<YOUR_CLUSTER_NAME>,Value=shared
```

**Verify tags:**

```bash
aws ec2 describe-subnets \
  --filters "Name=tag:kubernetes.io/cluster/<YOUR_CLUSTER_NAME>,Values=shared" \
  --query 'Subnets[*].[SubnetId,Tags]'
```

---

## Troubleshooting Commands

```bash
# Envoy Gateway control plane logs
kubectl logs -n envoy-gateway-system deployment/envoy-gateway --tail=100

# AWS Load Balancer Controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=100

# Gateway status (check status.addresses and status.conditions)
kubectl describe gateway external-gateway -n gateway

# HTTPRoute status (check if Accepted and Resolved)
kubectl describe httproute backend-route -n gateway

# HPA status and current metrics
kubectl describe hpa -n gateway

# All proxy-related services
kubectl get svc -n gateway

# Proxy pods with container count
kubectl get pods -n gateway

# Check which containers are in a pod
kubectl get pod <pod-name> -n gateway -o jsonpath='{.spec.containers[*].name}'

# Gateway API resource conditions
kubectl get gateway external-gateway -n gateway -o jsonpath='{.status.conditions}' | jq .

# Watch pods during a scaling event
kubectl get pods -n gateway -w
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            EKS Cluster                                  │
│                                                                         │
│  ┌──────────────────────┐  watches  ┌──────────────────────────────┐    │
│  │   Envoy Gateway      │ ────────> │   Gateway API CRDs           │    │
│  │   (Control Plane)    │           │   GatewayClass               │    │
│  │   1/1  Running       │           │   Gateway                    │    │
│  └──────────┬───────────┘           │   HTTPRoute                  │    │
│             │ creates               └──────────────────────────────┘    │
│             ▼                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Proxy Deployment  (HPA: min 1 → max 5 pods)                     │   │
│  │                                                                  │   │
│  │  Pod 1 [2/2]              Pod 2 [2/2]          Pod N [2/2]       │   │
│  │  ├─ envoy (main)          ├─ envoy (main)       ├─ envoy         │   │
│  │  └─ shutdown-manager      └─ shutdown-manager   └─ shutdown-mgr  │   │
│  └──────────┬───────────────────────────────────────────────────────┘   │
│             │ exposed via                                               │
│             ▼                                                           │
│  ┌──────────────────────┐                                               │
│  │  Service (LB type)   │ <── AWS LBC reads annotations                 │
│  │  + NLB Annotations   │         and provisions NLB                    │
│  └──────────────────────┘                                               │
│                                                                         │
│  ┌──────────────────────┐                                               │
│  │  HPA                 │ ── watches CPU → adjusts pod count            │
│  │  min:1  max:5        │                                               │
│  │  target: 60% CPU     │                                               │
│  └──────────────────────┘                                               │
│                                                                         │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
                        ┌──────────────────────┐
                        │      AWS NLB         │
                        │  (internet-facing)   │
                        │Health: :19002/healthz│
                        └──────────────────────┘
                                   │
                                   ▼
                             Internet Traffic
```

---

## Quick Setup Checklist

- [ ] Gateway API CRDs installed
- [ ] Metrics Server installed (required for HPA)
- [ ] Envoy Gateway Helm chart installed
- [ ] `EnvoyProxy` CRD created with NLB annotations, resources, and HPA config
- [ ] `GatewayClass` created and Accepted
- [ ] `Gateway` created and referencing `EnvoyProxy` via `parametersRef`
- [ ] Public subnets tagged with `kubernetes.io/role/elb: 1`
- [ ] Cluster name tag on subnets
- [ ] AWS LBC IAM role has ELB + EC2 describe permissions
- [ ] `HTTPRoute` created and Accepted
- [ ] NLB hostname visible in `kubectl get gateway`
- [ ] HPA shows real CPU metrics (not `<unknown>`)
- [ ] `kubectl get pods -n gateway` shows `2/2 Running` (envoy + shutdown-manager)

---

## References

- [Envoy Gateway Official Docs](https://gateway.envoyproxy.io/docs/)
- [AWS Load Balancer Controller Docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Gateway API Spec](https://gateway-api.sigs.k8s.io/)
- [Kubernetes HPA Docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [AWS EKS NLB Service Annotations](https://docs.aws.amazon.com/eks/latest/userguide/auto-configure-nlb.html)
- [Envoy Proxy Performance Tuning](https://www.envoyproxy.io/docs/envoy/latest/operations/performance)