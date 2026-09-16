# Deployment in Kubernetes

A **Deployment** is a Kubernetes workload resource used to run and manage a set of identical Pods. It is the standard way to deploy and update stateless applications in Kubernetes.

The Deployment in this folder is defined in [`deployment.yml`](deployment.yml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  namespace: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-container
          image: nginx:latest
          ports:
            - containerPort: 80
```

## Why do we need a Deployment?

A manually created Pod is temporary. If it crashes, is deleted, or the node running it fails, Kubernetes does not create a replacement for that standalone Pod.

A ReplicaSet improves this by maintaining the requested number of Pods, but it does not provide a complete application release process. Updating a ReplicaSet does not automatically perform a controlled replacement of healthy old Pods, and it does not keep a convenient revision history for rollbacks.

A Deployment solves these operational problems by:

- Maintaining the desired number of Pods through ReplicaSets.
- Performing controlled rolling updates.
- Keeping rollout history.
- Allowing a failed release to be rolled back.
- Scaling the application up or down.
- Pausing and resuming a rollout.
- Providing a declarative desired state that can be stored in Git.

For most stateless services, the usual relationship is:

```text
Deployment -> ReplicaSet -> Pods -> Containers
```

A Deployment does not expose an application to clients. Use a **Service** for stable networking and load balancing, and use an **Ingress** or Gateway when HTTP routing from outside the cluster is required.

## How a Deployment works

Kubernetes controllers continuously compare the desired state in the API server with the actual state in the cluster.

1. The Deployment manifest is submitted to the Kubernetes API server.
2. The Deployment controller creates a ReplicaSet for the Pod template.
3. The ReplicaSet creates Pods from that template.
4. The scheduler selects nodes for the Pods.
5. The kubelet on each selected node starts and monitors the containers.
6. The Deployment controller watches the rollout and reports its status.
7. When the Pod template changes, the Deployment creates a new ReplicaSet.
8. The new ReplicaSet is scaled up while the old ReplicaSet is scaled down according to the rollout strategy.

The Deployment controller does not run containers itself. It manages ReplicaSets, while the scheduler and kubelets perform Pod placement and container execution.

## Important fields in this example

| Field | Purpose |
| --- | --- |
| `apiVersion: apps/v1` | Uses the stable Deployment API. |
| `kind: Deployment` | Declares the resource type. |
| `metadata.name` | Name of the Deployment. |
| `metadata.namespace` | Namespace where the Deployment and its Pods are created. The namespace must already exist. |
| `spec.replicas` | Desired number of Pods. This example requests three. |
| `spec.selector` | Identifies the Pods managed by this Deployment. |
| `spec.template` | Blueprint used to create the Pods. |
| `template.metadata.labels` | Labels that must match the selector. |
| `containers[].image` | Container image used by each Pod. |
| `containerPort` | Documents the port the container listens on; it does not expose the Pod externally. |

### Selector and template labels

The selector must match the labels in the Pod template:

```yaml
selector:
  matchLabels:
    app: my-app
template:
  metadata:
    labels:
      app: my-app
```

The selector is effectively part of the Deployment identity and cannot be changed after creation. Choose stable, specific labels and avoid selectors that could match Pods belonging to another workload.

## Create and inspect the Deployment

Run these commands from the `Day_06` directory. The `nginx` namespace must exist first; it is defined in Day 04 of this guide.

```bash
# Confirm that the namespace exists
kubectl get namespace nginx

# Validate the manifest without creating resources
kubectl apply --dry-run=client -f deployment.yml

# Create or update the Deployment
kubectl apply -f deployment.yml

# View Deployment status
kubectl get deployment my-deployment -n nginx

# View the ReplicaSets created by the Deployment
kubectl get rs -n nginx

# View the Pods created by this Deployment
kubectl get pods -n nginx -l app=my-app -o wide

# Inspect the Deployment and recent events
kubectl describe deployment my-deployment -n nginx

# View the full resource definition
kubectl get deployment my-deployment -n nginx -o yaml
```

The `kubectl get deployment` columns commonly show:

- `READY`: ready Pods compared with desired Pods, such as `3/3`.
- `UP-TO-DATE`: Pods using the latest Pod template.
- `AVAILABLE`: Pods available to serve the application.
- `AGE`: how long the Deployment has existed.

## Rolling updates

A Deployment creates a new ReplicaSet when its Pod template changes. For example, changing the image from `nginx:latest` to a versioned image creates a new revision:

```bash
kubectl set image deployment/my-deployment \
  my-container=nginx:1.27 \
  -n nginx

kubectl rollout status deployment/my-deployment -n nginx
kubectl get rs -n nginx
kubectl get pods -n nginx -l app=my-app
```

During a normal rolling update, Kubernetes starts new Pods and removes old Pods gradually. This helps keep the application available, provided there are enough resources and the application can run multiple instances.

The default strategy is `RollingUpdate`. It is controlled by settings such as:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
```

- `maxUnavailable` is the maximum number of unavailable Pods during an update.
- `maxSurge` is the maximum number of extra Pods that can be created above the desired count.

A `Recreate` strategy stops old Pods before starting new ones. It may cause downtime, but it can be useful when two versions cannot safely run at the same time.

## Rollout history and rollback

Every change to the Pod template creates a new Deployment revision. Inspect and control revisions with:

```bash
# Show rollout history
kubectl rollout history deployment/my-deployment -n nginx

# Show details for a specific revision
kubectl rollout history deployment/my-deployment -n nginx --revision=2

# Roll back to the previous revision
kubectl rollout undo deployment/my-deployment -n nginx

# Roll back to a specific revision
kubectl rollout undo deployment/my-deployment -n nginx --to-revision=1

# Watch the rollback complete
kubectl rollout status deployment/my-deployment -n nginx
```

A rollback changes the Pod template back to an earlier revision. It does not restore database data or undo external side effects, so database migrations and application compatibility still need a separate rollback plan.

## Scaling a Deployment

Imperatively:

```bash
kubectl scale deployment my-deployment --replicas=5 -n nginx
kubectl get deployment my-deployment -n nginx
```

Declaratively, change `spec.replicas` in `deployment.yml` and apply it again:

```bash
kubectl apply -f deployment.yml
```

The declarative approach is preferred because the desired state stays recorded in version control. For automatic scaling based on CPU or memory, use a **Horizontal Pod Autoscaler** and configure resource requests first.

## Pause and resume a rollout

Pausing is useful when several related changes should be applied before Kubernetes finishes a rollout:

```bash
kubectl rollout pause deployment/my-deployment -n nginx
kubectl rollout resume deployment/my-deployment -n nginx
```

Always check the rollout after resuming:

```bash
kubectl rollout status deployment/my-deployment -n nginx
```

## Readiness, liveness, and startup probes

The sample manifest has no health probes. Without a readiness probe, Kubernetes may consider a running container ready even when the application cannot serve traffic.

- **Readiness probe**: decides whether a Pod should receive traffic from a Service.
- **Liveness probe**: restarts a container that is unhealthy over time.
- **Startup probe**: gives a slow-starting application time to initialize before liveness checks begin.

Example for an HTTP application:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 15
  periodSeconds: 20
```

Do not use a liveness probe to check a dependency that may be temporarily unavailable. Otherwise, a dependency problem can cause Kubernetes to restart every application instance.

## Production improvements for this example

The example is intentionally small for learning. Before using a Deployment in production, consider:

- Pin image versions such as `nginx:1.27.1` instead of using the mutable `latest` tag.
- Add readiness, liveness, and startup probes where appropriate.
- Define CPU and memory requests and limits.
- Use a Service for stable access to the Pods.
- Add graceful termination settings and a suitable `terminationGracePeriodSeconds`.
- Use PodDisruptionBudgets for important replicated workloads.
- Spread replicas across nodes or zones with topology spread constraints or anti-affinity.
- Store configuration in ConfigMaps and sensitive values in Secrets.
- Use security contexts and run containers with the minimum required permissions.
- Set an update strategy that matches the application's availability requirements.
- Monitor rollout status, application metrics, logs, and events.

Three replicas do not automatically guarantee high availability. If all Pods are scheduled onto one failed node, the application can still become unavailable until replacement Pods are scheduled elsewhere.

## Deployment versus ReplicaSet versus Pod

| Resource | Main responsibility |
| --- | --- |
| Pod | Runs one or more tightly coupled containers. It is the smallest deployable unit. |
| ReplicaSet | Maintains a desired number of matching Pods. |
| Deployment | Manages ReplicaSets and provides versioned rolling updates, history, and rollback. |

In normal application delivery, create a Deployment rather than creating a ReplicaSet or individual Pods directly.

## Deployment versus StatefulSet and DaemonSet

| Workload | Use it when |
| --- | --- |
| Deployment | The application is usually stateless and interchangeable replicas are suitable. |
| StatefulSet | Each replica needs stable identity, ordered operations, or persistent storage association. |
| DaemonSet | One Pod should run on every eligible node, such as a logging or monitoring agent. |
| Job | A task should run to completion. |
| CronJob | A task should run on a schedule. |

## Troubleshooting checklist

Start with the Deployment, ReplicaSets, Pods, and events:

```bash
kubectl get deployment my-deployment -n nginx
kubectl rollout status deployment/my-deployment -n nginx
kubectl get rs -n nginx
kubectl get pods -n nginx -l app=my-app -o wide
kubectl describe deployment my-deployment -n nginx
kubectl describe pod <pod-name> -n nginx
kubectl logs <pod-name> -n nginx
kubectl get events -n nginx --sort-by=.lastTimestamp
```

| Symptom | Possible cause |
| --- | --- |
| `READY` is lower than desired | Pods are Pending, failing readiness, or restarting. |
| Pods are `Pending` | Insufficient resources, taints, affinity rules, or no suitable node. |
| Pods show `ImagePullBackOff` | Incorrect image/tag or missing registry credentials. |
| Pods show `CrashLoopBackOff` | Application crash, bad command, configuration problem, or failed liveness probe. |
| Rollout is stuck | New Pods are not becoming ready, a quota is reached, or nodes lack capacity. |
| Old Pods remain | The rollout is still progressing, is paused, or the old ReplicaSet is retained as history. |
| The Deployment is not found | The wrong namespace was used. Include `-n nginx` in commands. |
| Application is unreachable | A Deployment does not provide networking; create or inspect a Service. |

## Interview questions and answers

### What is a Deployment in Kubernetes?

A Deployment is a Kubernetes controller that declaratively manages application Pods through ReplicaSets. It supports scaling, rolling updates, rollout history, and rollback.

### What happens when a Deployment is updated?

When the Pod template changes, the Deployment creates a new ReplicaSet. It gradually scales up the new ReplicaSet and scales down the old one according to the update strategy.

### What is the difference between scaling and updating?

Scaling changes the number of desired Pod replicas. Updating changes the Pod template, such as the image, environment variables, or container configuration, and normally creates a new ReplicaSet revision.

### Does changing `replicas` create a new revision?

No. Changing the replica count changes the scale of the current revision. A new revision is created when the Pod template changes.

### How does a Deployment perform a rollback?

It changes the Pod template back to a previous revision recorded in Deployment history, then performs another rollout to that template.

### What is the difference between a Deployment and a ReplicaSet?

A ReplicaSet only maintains the desired number of matching Pods. A Deployment manages ReplicaSets and adds controlled updates, rollout history, and rollback.

### What is the default Deployment update strategy?

`RollingUpdate` is the default. It replaces Pods gradually. `Recreate` removes all old Pods before creating new ones and can cause downtime.

### Does a Deployment expose Pods outside the cluster?

No. A Deployment manages Pods but does not provide a stable network endpoint. A Service is used for stable networking and load balancing.

### What is the purpose of a readiness probe?

It tells Kubernetes whether a Pod is ready to receive traffic. A failed readiness probe removes the Pod from Service endpoints without necessarily restarting the container.

### What is the difference between `kubectl apply` and `kubectl create`?

`kubectl create` creates a resource and fails if it already exists. `kubectl apply` creates or updates a resource to match the declarative configuration and is commonly used with manifests stored in Git.

### Can a Deployment manage a Stateful application?

It can run the containers, but it does not provide stable per-replica identity or ordered storage behavior. Stateful applications commonly require a StatefulSet instead.

### What happens if a Deployment Pod is deleted?

The ReplicaSet managed by the Deployment notices that the actual count is below the desired count and creates a replacement Pod.

### What is the purpose of `maxUnavailable` and `maxSurge`?

They control how many Pods may be unavailable and how many extra Pods may exist during a rolling update. Together they balance availability and rollout speed.

## Key takeaways

- Use a Deployment for most stateless application workloads.
- A Deployment manages ReplicaSets, and ReplicaSets manage Pods.
- Changing the Pod template creates a new rollout revision.
- Use `kubectl rollout status` to verify progress and `kubectl rollout undo` to recover from a bad release.
- A Deployment provides workload management, not stable networking; use a Service for that.
- Production Deployments should use versioned images, health probes, resource settings, security controls, and an availability-aware rollout strategy.
