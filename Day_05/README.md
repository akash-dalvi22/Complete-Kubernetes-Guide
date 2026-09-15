# ReplicaSet in Kubernetes

A **ReplicaSet (RS)** ensures that a specified number of identical Pods are running at all times. It is a Kubernetes workload resource used for high availability and self-healing.

The ReplicaSet in this folder is defined in [`replicaset.yml`](replicaset.yml):

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
	name: nginx-replicaset
spec:
	replicas: 3
	selector:
		matchLabels:
			app: nginx
	template:
		metadata:
			labels:
				app: nginx
		spec:
			containers:
			- name: nginx
				image: nginx
```

## Why do we need a ReplicaSet?

A manually created Pod is not self-healing. If a Pod crashes, is deleted, or the node running it fails, Kubernetes does not create a replacement for that standalone Pod.

A ReplicaSet solves this problem by maintaining the desired number of Pods:

- Desired state: `replicas: 3`
- Current state: the number of matching Pods that currently exist
- Controller action: create or delete Pods until current state equals desired state

For example, if one of three Pods is deleted, the ReplicaSet controller creates a replacement automatically. This gives applications availability and resilience during normal failures.

## How a ReplicaSet works

The ReplicaSet controller continuously watches the Kubernetes API server. It compares the desired state in the ReplicaSet with the actual state in the cluster.

1. The ReplicaSet is submitted to the API server.
2. The controller reads `spec.replicas` and the Pod template.
3. The selector finds Pods with the matching labels.
4. If fewer Pods exist than requested, the controller creates Pods from `spec.template`.
5. If too many matching Pods exist, the controller removes the excess Pods.
6. If a managed Pod fails or is deleted, the controller repeats this process and creates a replacement.

The ReplicaSet does not run containers itself. The scheduler places new Pods on nodes, and the kubelet on each node starts and monitors the containers.

## Important fields

| Field | Purpose |
| --- | --- |
| `apiVersion: apps/v1` | Uses the stable ReplicaSet API. |
| `kind: ReplicaSet` | Declares the resource type. |
| `metadata.name` | The unique name of the ReplicaSet in its namespace. |
| `spec.replicas` | Number of desired Pod replicas. |
| `spec.selector` | Identifies which Pods belong to this ReplicaSet. |
| `spec.template` | Blueprint used to create replacement Pods. |
| `template.metadata.labels` | Labels that must match the selector. |
| `template.spec` | Container, image, ports, probes, resources, and other Pod settings. |

### Selector and template labels

The selector and Pod template must match:

```yaml
selector:
	matchLabels:
		app: nginx
template:
	metadata:
		labels:
			app: nginx
```

If the selector does not match the template labels, Kubernetes rejects the manifest. A selector that is too broad can also accidentally manage Pods created by another workload, so labels should be specific and intentional.

## Create and inspect the ReplicaSet

Run these commands from the `Day_05` directory:

```bash
# Validate the manifest without creating resources
kubectl apply --dry-run=client -f replicaset.yml

# Create or update the ReplicaSet
kubectl apply -f replicaset.yml

# List ReplicaSets and their desired/current/ready counts
kubectl get rs

# Show the Pods created by this ReplicaSet
kubectl get pods -l app=nginx -o wide

# Inspect ReplicaSet configuration and events
kubectl describe rs nginx-replicaset

# View the complete resource definition
kubectl get rs nginx-replicaset -o yaml
```

The `kubectl get rs` columns commonly show:

- `DESIRED`: replicas requested in the manifest
- `CURRENT`: Pods currently created for the ReplicaSet
- `READY`: Pods that are ready to receive traffic
- `AGE`: how long the ReplicaSet has existed

## Test self-healing

First list the Pods, then delete one of them:

```bash
kubectl get pods -l app=nginx
kubectl delete pod <pod-name>
kubectl get pods -l app=nginx -w
```

The ReplicaSet should create a new Pod. The replacement gets a different Pod name because Pods are disposable instances, while the ReplicaSet provides the stable desired count.

## Scale a ReplicaSet

Imperatively:

```bash
kubectl scale rs nginx-replicaset --replicas=5
kubectl get rs nginx-replicaset
```

Declaratively, change `spec.replicas` in `replicaset.yml` and apply it again:

```bash
kubectl apply -f replicaset.yml
```

The declarative approach is preferred because the desired state remains recorded in version control.

## ReplicaSet versus ReplicationController

ReplicaSet is the newer replacement for the older ReplicationController resource.

| ReplicaSet | ReplicationController |
| --- | --- |
| Uses `apps/v1`. | Uses the older core API. |
| Supports set-based selectors such as `matchExpressions`. | Primarily supports equality-based selectors. |
| Used by modern workload controllers. | Kept mainly for legacy workloads. |

For new applications, use a ReplicaSet or, more commonly, a Deployment.

## ReplicaSet versus Deployment

A Deployment manages ReplicaSets and adds rollout functionality. In production, create a Deployment instead of creating a ReplicaSet directly in most cases.

| ReplicaSet | Deployment |
| --- | --- |
| Maintains a number of identical Pods. | Manages ReplicaSets and maintains Pods. |
| Provides self-healing and scaling. | Also provides rolling updates and rollback. |
| Updating the Pod image does not provide a coordinated rollout. | Creates a new ReplicaSet for a new Pod template and gradually replaces old Pods. |
| Useful for learning, simple controllers, or direct low-level control. | Recommended for stateless application releases. |

Changing the image in a ReplicaSet template affects newly created replacement Pods, but it does not automatically replace healthy existing Pods. A Deployment is the correct tool for controlled application updates.

## ReplicaSet does not provide these features

A ReplicaSet alone does not:

- Expose Pods through a stable network endpoint. Use a **Service**.
- Provide application rollout history or rollback. Use a **Deployment**.
- Guarantee that Pods run on different nodes. Use scheduling rules such as topology spread constraints or anti-affinity.
- Store application data safely. Use an appropriate persistent storage design.
- Make an application highly available if the application itself cannot handle multiple instances.

## Troubleshooting checklist

If the expected number of ready Pods is not available:

```bash
kubectl get rs nginx-replicaset
kubectl get pods -l app=nginx -o wide
kubectl describe rs nginx-replicaset
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
```

Look for these common causes:

| Symptom | Possible cause |
| --- | --- |
| `DESIRED` is correct but `CURRENT` is low | Invalid Pod template, admission rejection, or controller/event error. |
| Pod is `Pending` | Insufficient node resources, taints, affinity rules, or no suitable node. |
| Pod is `ImagePullBackOff` | Incorrect image name/tag or registry authentication problem. |
| Pod is repeatedly restarting | Application crash, bad command, configuration error, or failed liveness probe. |
| `READY` is lower than `CURRENT` | Readiness probe failure or the application is not ready. |
| ReplicaSet controls unexpected Pods | Selector is too broad or labels overlap with another workload. |

Useful status checks include:

```bash
kubectl get rs nginx-replicaset -o jsonpath='{.status}'
kubectl get pods -l app=nginx --show-labels
kubectl logs <pod-name>
```

## Interview questions and answers

### What is a ReplicaSet?

A ReplicaSet is a Kubernetes controller that maintains a stable number of identical Pods by creating or deleting Pods to match the desired replica count.

### What happens if a Pod managed by a ReplicaSet is deleted?

The ReplicaSet controller notices that the actual count is below the desired count and creates a replacement Pod from its template.

### How does a ReplicaSet identify its Pods?

It uses `spec.selector`, which matches labels on Pods. The selector must match the labels in `spec.template.metadata.labels`.

### Can a ReplicaSet perform rolling updates?

No. A ReplicaSet maintains Pod count but does not provide Deployment-style rollout and rollback. Use a Deployment for versioned updates.

### Can two ReplicaSets use the same selector?

They should not. Overlapping selectors can cause controllers to manage the same Pods and lead to unexpected behavior. Each workload should have an unambiguous selector.

### What is the difference between desired, current, and ready replicas?

`desired` is the requested count, `current` is the number of Pods created, and `ready` is the number currently passing readiness checks.

### Is a ReplicaSet namespaced?

Yes. A ReplicaSet exists inside a namespace, and its name only needs to be unique within that namespace.

## Best practices

- Prefer a Deployment for production stateless applications.
- Use stable, meaningful labels such as `app`, `component`, and `version`.
- Keep selectors specific and never reuse a selector across unrelated workloads.
- Define resource requests and limits for predictable scheduling and cluster capacity planning.
- Add readiness and liveness probes where appropriate.
- Pin image versions instead of relying on the mutable `latest` tag.
- Keep manifests in version control and use `kubectl apply` for repeatable changes.
- Use namespaces and RBAC to separate teams and workloads.

## Quick revision

> **ReplicaSet = desired number of identical Pods + label selector + self-healing.**

Remember the hierarchy:

```text
Deployment -> ReplicaSet -> Pod -> Container
```

A Deployment is normally the resource you operate, the ReplicaSet is the controller that maintains one version of that application, and Pods are the replaceable running instances.
