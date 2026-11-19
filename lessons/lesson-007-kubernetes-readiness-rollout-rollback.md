# Lesson 007 — Kubernetes: readiness, rolling updates and rollback

**Domain:** Kubernetes operations / workload availability  
**Duration:** 30 minutes  
**Environment:** Windows 11 PowerShell, Docker Desktop using Linux containers, `kind` and `kubectl`  
**Version assumptions:** Kubernetes `apps/v1` Deployment API; a `kubectl` client compatible with the cluster (within the supported version-skew policy); `kind` using its default node image. The example NGINX tags below were checked against the Docker Official Image listing on 2026-11-18. They are lab tags, not production pinning guidance.

## Objective

Operate a disposable one-node Kubernetes cluster and practise a controlled Deployment update with health checks and rollback.

You will:
1. Create an isolated `kind` cluster and explicitly verify its Kubernetes context before issuing commands.
2. Deploy two NGINX replicas with CPU/memory requests and limits.
3. Configure readiness and liveness HTTP probes, then verify the Service has ready endpoints.
4. Perform a rolling image update using `maxUnavailable: 0` and `maxSurge: 1`.
5. Inspect rollout history and roll back to the original image.
6. Delete only the cluster created for this lesson.

**Safety boundary:** This lesson must run only against the disposable local cluster created below. Every `kubectl` command explicitly names its context. Do not substitute a shared, staging or production context. If the required local tools are missing, do the manifest review and defer cluster execution; do not point the commands at another cluster.

## Prerequisites and preflight (3 minutes)

Required:
- Docker Desktop already installed and running with Linux containers.
- `kubectl` and `kind` available in PowerShell.
- Docker Desktop allocated at least 6 GB RAM for the local Kubernetes node; 8 GB is recommended by the `kind` documentation.
- Internet access to pull the Kubernetes node and NGINX images.

Open **PowerShell on Windows 11** and check:

```powershell
docker version
kubectl version --client
kind version
```

Expected: Docker Client and Server information, a `kubectl` client version, and a `kind` version. If any required tool is missing, stop here rather than installing or reconfiguring software during this 30-minute exercise.

## 30-minute plan

| Time | Activity |
|---:|---|
| 0–3 min | Check prerequisites and confirm local resource capacity |
| 3–7 min | Create and verify the isolated cluster |
| 7–14 min | Write and apply the Deployment and Service manifests |
| 14–19 min | Verify probes, resources and ready Service endpoints |
| 19–25 min | Roll out a new image and inspect revision history |
| 25–28 min | Roll back and verify the original image |
| 28–30 min | Delete the disposable cluster and self-check |
| **Total** | **30 minutes** |

## 1. Create an isolated local cluster (4 minutes)

Choose a unique cluster name. First check that it does not already exist:

```powershell
$ClusterName = "tech-refresher-20261118-007"
kind get clusters

if ((kind get clusters) -contains $ClusterName) {
    throw "Cluster name already exists. Choose a different unique name before continuing."
}

kind create cluster --name $ClusterName --wait 90s
```

`kind` creates a local Kubernetes cluster using Docker containers as nodes. It changes the current `kubectl` context, so verify it immediately:

```powershell
$KubeContext = "kind-$ClusterName"
$CurrentContext = kubectl config current-context

if ($CurrentContext -ne $KubeContext) {
    throw "STOP: current context is not the disposable lesson cluster."
}

kubectl --context $KubeContext cluster-info
kubectl --context $KubeContext get nodes -o wide
```

Expected: one Ready node and cluster information for the context `kind-tech-refresher-20261118-007`. Do not continue if the context check fails.

## 2. Create the lab directory and manifests (7 minutes)

Create a dedicated working directory:

```powershell
$Lab = Join-Path $HOME "labs\tech-refresher\lesson-007-kubernetes-rollout"
New-Item -ItemType Directory -Force -Path $Lab | Out-Null
Set-Location $Lab
Get-Location
```

Create `deployment.yaml`:

```powershell
@'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-demo
  namespace: refresher-007
spec:
  replicas: 2
  revisionHistoryLimit: 3
  minReadySeconds: 2
  progressDeadlineSeconds: 180
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: web-demo
  template:
    metadata:
      labels:
        app: web-demo
    spec:
      terminationGracePeriodSeconds: 15
      containers:
        - name: nginx
          image: nginx:1.30.4-alpine3.24
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 250m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 1
            failureThreshold: 3
'@ | Set-Content -Path .\deployment.yaml -Encoding ascii
```

Create `service.yaml`:

```powershell
@'
apiVersion: v1
kind: Service
metadata:
  name: web-demo
  namespace: refresher-007
spec:
  type: ClusterIP
  selector:
    app: web-demo
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
'@ | Set-Content -Path .\service.yaml -Encoding ascii
```

Review the important choices:

- **Readiness probe:** decides whether a Pod should receive Service traffic. A failed readiness probe marks the Pod unready; it does not itself restart the container.
- **Liveness probe:** detects a container that should be restarted. Do not make liveness depend on an external database or API; a dependency outage could otherwise trigger unnecessary restart loops.
- **Startup probe:** not needed for this fast-starting NGINX example. Use one for applications with slow or variable startup; it delays liveness/readiness checks until startup succeeds.
- **Requests:** influence scheduling. **Limits:** constrain runtime consumption; CPU can be throttled and a container exceeding its memory limit can be OOM-killed.
- **Rolling update:** with two desired replicas, `maxUnavailable: 0` and `maxSurge: 1` allow one extra Pod while keeping the desired number available, assuming the cluster has enough capacity and the new Pod becomes Ready.

## 3. Deploy and inspect health (5 minutes)

Create the namespace and apply both resources to the verified local context:

```powershell
kubectl --context $KubeContext create namespace refresher-007
kubectl --context $KubeContext apply -f .\deployment.yaml
kubectl --context $KubeContext apply -f .\service.yaml

kubectl --context $KubeContext -n refresher-007 rollout status deployment/web-demo --timeout=120s
kubectl --context $KubeContext -n refresher-007 get pods -o wide
kubectl --context $KubeContext -n refresher-007 get endpointslices -l kubernetes.io/service-name=web-demo
```

Expected:
- Deployment rollout reports successfully.
- Two Pods show `READY 1/1` and `STATUS Running`.
- The EndpointSlice shows the two ready Pod endpoints for the Service.

Inspect the effective configuration and recent events:

```powershell
kubectl --context $KubeContext -n refresher-007 describe deployment web-demo
kubectl --context $KubeContext -n refresher-007 get events --sort-by=.lastTimestamp
```

Look for the configured image, resource requests/limits, probe definitions, available replicas and absence of warning events.

## 4. Perform a rolling update (6 minutes)

Update the image from the stable NGINX line to the newer mainline line. The exact tags are for this lab only:

```powershell
kubectl --context $KubeContext -n refresher-007 set image deployment/web-demo nginx=nginx:1.31.5-alpine3.24

kubectl --context $KubeContext -n refresher-007 rollout status deployment/web-demo --timeout=180s
kubectl --context $KubeContext -n refresher-007 rollout history deployment/web-demo
kubectl --context $KubeContext -n refresher-007 get replicasets
kubectl --context $KubeContext -n refresher-007 get pods -o wide
```

Expected: the new ReplicaSet reaches two ready replicas and the old ReplicaSet scales down. The rollout status command waits for the Deployment controller to complete the rollout.

If it stalls, do not keep changing the image. Inspect first:

```powershell
kubectl --context $KubeContext -n refresher-007 describe deployment web-demo
kubectl --context $KubeContext -n refresher-007 get pods
kubectl --context $KubeContext -n refresher-007 get events --sort-by=.lastTimestamp
```

Typical causes include image-pull failures, failed probes, insufficient resources or scheduling constraints.

## 5. Roll back and verify (3 minutes)

Return to the previous Deployment revision:

```powershell
kubectl --context $KubeContext -n refresher-007 rollout undo deployment/web-demo
kubectl --context $KubeContext -n refresher-007 rollout status deployment/web-demo --timeout=180s

kubectl --context $KubeContext -n refresher-007 get deployment web-demo -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Expected final image:

```text
nginx:1.30.4-alpine3.24
```

Also confirm two ready replicas:

```powershell
kubectl --context $KubeContext -n refresher-007 get deployment web-demo
kubectl --context $KubeContext -n refresher-007 get endpointslices -l kubernetes.io/service-name=web-demo
```

**Operational caveat:** Deployment rollback restores a previous Pod template revision. It does not automatically reverse database schema changes, external configuration changes, messages already processed, or other side effects. A rollback strategy must account for those separately.

## 6. Clean up (2 minutes)

Delete only the cluster created by this lesson:

```powershell
kind delete cluster --name $ClusterName
kind get clusters
```

Expected: the lesson cluster is no longer listed. Keep the YAML files for review. Do not run `docker system prune` or delete other Docker resources.

## Quick self-check

1. What is the operational difference between readiness and liveness?
2. Why should a liveness probe normally avoid checking an external database?
3. What do `maxUnavailable: 0` and `maxSurge: 1` do during this two-replica update?
4. Which events or Pod conditions would you inspect if the rollout stalled?
5. What important state does `kubectl rollout undo` not restore?

**Expected outcome:** a disposable cluster; two healthy NGINX replicas behind a ClusterIP Service; configured resource controls and probes; a completed image rollout; a successful rollback to the original image; and removal of only the lesson cluster.

## Production and interview relevance

This exercise covers a common Kubernetes operational sequence: deploy, wait for readiness, inspect the controller and Pod state, detect a stalled rollout, and restore a known-good Pod template. In production, rollout safety also depends on capacity, PodDisruptionBudgets, traffic draining, backward-compatible application/database changes, observability and an explicit rollback decision. `maxUnavailable: 0` is not a universal zero-downtime guarantee.

For production image promotion, do not rely on mutable tags such as `stable` or `mainline`. Resolve, scan and approve an image, then deploy the immutable digest through environments. Keep environment configuration outside the image.

## In simple words — what did we actually do?

We created a small Kubernetes environment on the local computer, deployed two copies of a web server, and taught Kubernetes how to decide whether those copies were healthy enough to receive traffic.

The two health checks have different jobs:

- **Readiness:** “Can this copy handle requests right now?” If not, Kubernetes stops sending it Service traffic while leaving the process running.
- **Liveness:** “Is this process still functioning?” If it repeatedly fails, Kubernetes restarts the container.
- **Startup:** “Has this slow-starting application finished initialising?” It protects an application from being restarted before it has had enough time to start.

Then we changed the NGINX image. Kubernetes gradually created new Pods and removed old ones, using the readiness results to decide when the new copies were ready. Finally, we rolled back to the previous Deployment revision.

### How this fits into a DevOps delivery flow

| Tool or component | Responsibility |
|---|---|
| Maven / Ant | Build the application artifact. |
| Jenkins | Test, package, scan and promote the approved image. |
| Docker | Package the application and runtime. |
| Image registry | Store versioned, approved images. |
| Kubernetes Deployment | Control replicas and perform rolling updates. |
| Kubernetes probes and Service | Keep traffic directed to ready Pods and detect unhealthy processes. |
| Observability | Show rollout status, restarts, latency, errors and resource pressure. |

### Three things to remember

- **Ready is not the same as running:** a process can be running but not ready to serve traffic.
- **A rollout is a controlled replacement:** watch readiness, events and ReplicaSets rather than assuming an image update succeeded.
- **Rollback has boundaries:** reverting a Deployment does not reverse database or external side effects.

This was a local Kubernetes exercise. It did not deploy to or modify a remote cluster.

## Completion checkpoint

When finished, report:

- Docker, `kubectl` and `kind` versions.
- Whether the isolated cluster became Ready.
- Whether both Pods reached `1/1 Running` and appeared as Service endpoints.
- Whether the rolling update completed.
- Whether rollback restored `nginx:1.30.4-alpine3.24`.
- Whether the disposable cluster was deleted.
- One production consideration you would add before allowing this rollout through a release pipeline.
- Confidence from 1–5.

Do not paste kubeconfig contents, credentials, tokens, private registry details or production configuration. Do not mark the lesson complete until the checks have actually been run.

## References

- [Kubernetes Docs — Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) — probe behaviour and configuration.
- [Kubernetes Docs — Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/) — rolling update strategy, rollout status, diagnostics and rollback.
## 🧸 For Dummies

Imagine you run a pizza shop.

You have two little kitchens making pizzas. If one kitchen is busy or broken, the other can still serve customers.

Here is what the Kubernetes words mean:

- **Docker** is like a lunchbox. It packs the pizza-making tools and instructions together so they can be used elsewhere.
- **A container** is like one packed-up kitchen, ready to work.
- **A Pod** is the little home where Kubernetes runs our container. In this lesson, each Pod runs one NGINX web server.
- **A Deployment** is the manager's instruction: “Keep two kitchens working for me.”
- **A replica** is simply another copy. Two replicas means two copies of our web server.
- **A Service** is the person at the front counter who sends each customer to a working kitchen. Customers do not need to know which kitchen they get.
- **A readiness check** asks: “Is this kitchen ready to serve customers?” If the answer is no, customers are sent elsewhere.
- **A liveness check** asks: “Is this kitchen still working properly?” If it is stuck, Kubernetes can restart it.
- **A rolling update** means replacing the old kitchens one at a time, instead of closing the whole shop to change everything.
- **A rollback** means: “The new version is causing trouble. Put the previous version back.”

### So what are we actually doing today?

1. We ask Docker to help us build a tiny practice Kubernetes environment on our own computer.
2. We tell Kubernetes to start two little web servers.
3. We check that both are ready to serve visitors.
4. We tell Kubernetes to replace the old web-server version with a newer one, carefully.
5. We pretend we do not like the new version and ask Kubernetes to bring the old one back.
6. We close our practice environment when we finish.

### Why does this matter?

Imagine a bank's website. The bank cannot casually switch everything off whenever it wants to install a new version.

Kubernetes helps the engineers replace application copies gradually, check that the new copies are working, and recover the previous version if needed.

**The one thing to remember:** Docker packs and runs the application; Kubernetes looks after the running copies. kind gives us a small practice environment on our own computer.

If the technical terms still feel confusing, that is okay. First understand the pizza-shop story; the commands will make more sense as you see each part working.
