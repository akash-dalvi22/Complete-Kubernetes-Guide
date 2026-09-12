# Day 04: Namespaces and Pods

## What Is a Namespace?

A namespace is a logical partition inside a Kubernetes cluster. It creates a scope for names and resources, so resources with the same name can exist in different namespaces.

Namespaces are useful for:

- Separating environments such as `development`, `staging`, and `production`.
- Separating teams or applications in a shared cluster.
- Applying access control with `Role` and `RoleBinding`.
- Applying resource limits and quotas with `ResourceQuota` and `LimitRange`.
- Organizing and filtering resources with labels and commands.

Namespaces do not provide complete security or physical isolation. Network traffic, for example, must be restricted with `NetworkPolicy` when isolation is required.

### Common Namespace Commands

```bash
# List namespaces
kubectl get namespaces

# Create a namespace
kubectl create namespace development

# List resources in a namespace
kubectl get all -n development

# Use a namespace by default in the current kubectl context
kubectl config set-context --current --namespace=development

# View namespace details and events
kubectl describe namespace development
kubectl get events -n development --sort-by=.lastTimestamp
```

Most Kubernetes objects are namespaced, but some objects, such as `Node`, `PersistentVolume`, and `Namespace`, are cluster-scoped.

## What Is a Pod?

A Pod is the smallest deployable unit in Kubernetes. It represents one or more containers that are scheduled together on the same node.

Containers in the same Pod:

- Share the Pod's network namespace and can communicate through `localhost`.
- Can share storage volumes.
- Always run together on the same node.
- Share the same lifecycle and IP address.

Although a Pod can contain multiple containers, the usual pattern is one main application container per Pod plus optional helper containers (sidecars). Pods are designed to be replaceable and should normally be managed by a controller such as a `Deployment`, `StatefulSet`, `DaemonSet`, or `Job` instead of being created directly.

### Why Kubernetes Uses Pods

Kubernetes needs a unit that describes the containers which must be scheduled, started, stopped, networked, and monitored together. The Pod provides that unit while Kubernetes controllers maintain the desired number of healthy Pod replicas.

Pods are temporary. A failed Pod may be deleted and replaced with a new Pod that has a different name and IP address, so applications should use a `Service` for stable networking.

### Example Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: nginx
	labels:
		app: nginx
spec:
	containers:
		- name: nginx
			image: nginx:1.27
			ports:
				- containerPort: 80
```

Apply and inspect it with:

```bash
kubectl apply -f pod.yml -n development
kubectl get pod nginx -n development -o wide
kubectl describe pod nginx -n development
```

## Common Pod Errors and Troubleshooting

Start with the Pod status, details, events, and logs. Always include the namespace when the Pod is not in `default`.

```bash
kubectl get pods -A
kubectl get pod <pod-name> -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl logs <pod-name> -n <namespace> -c <container-name>
```

### `Pending`

The Pod has not been scheduled or cannot start. Common causes include insufficient CPU or memory, node taints, an invalid node selector or affinity rule, an unbound PersistentVolumeClaim, or unavailable nodes.

Troubleshooting:

```bash
kubectl describe pod <pod-name> -n <namespace>  # Check Events at the end
kubectl get nodes
kubectl describe nodes
kubectl get pvc -n <namespace>
```

Check the scheduler event for the exact reason. Add capacity, correct scheduling rules, tolerate the required taint, or fix storage rather than deleting the Pod repeatedly.

### `ContainerCreating` or `Waiting`

The Pod is scheduled, but a container has not started. Common causes are image pulls, volume mounts, Secret or ConfigMap references, CNI networking, or container runtime problems.

Troubleshooting:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl get secret,configmap -n <namespace>
```

### `ErrImagePull` or `ImagePullBackOff`

Kubernetes cannot download the container image. The image name or tag may be wrong, the registry may be unavailable, or the registry may require authentication.

Troubleshooting:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get secret -n <namespace>
```

Verify the image name and tag, confirm registry connectivity, and configure an `imagePullSecret` for a private registry. `ImagePullBackOff` means Kubernetes is retrying with increasing delays; it is not a separate root cause.

### `CrashLoopBackOff`

The container starts and then exits repeatedly. Typical causes include application errors, invalid arguments or configuration, missing environment variables, permission problems, or an incorrect health check.

Troubleshooting:

```bash
kubectl logs <pod-name> -n <namespace> -c <container-name>
kubectl logs <pod-name> -n <namespace> -c <container-name> --previous
kubectl describe pod <pod-name> -n <namespace>
```

Read the previous container's logs, inspect the exit code and events, then fix the application or configuration. Do not rely only on restarting the Pod because the restart does not remove the underlying cause.

### `Error` or `OOMKilled`

`Error` means a container terminated unsuccessfully. `OOMKilled` means the container exceeded its memory limit or the node ran out of memory.

Troubleshooting:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl top pod <pod-name> -n <namespace>
kubectl top node
```

Check the termination reason and exit code. Reduce memory usage, fix a leak, or set realistic resource requests and limits. Ensure the cluster has enough node memory.

### `CreateContainerConfigError`

Kubernetes cannot build the container configuration, commonly because a referenced Secret or ConfigMap key does not exist.

Troubleshooting:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get secret,configmap -n <namespace>
```

Compare the Pod manifest with the actual Secret or ConfigMap names and keys, then create or correct the missing configuration.

### `RunContainerError`

The container runtime cannot start the container. Common causes include an invalid command, bad volume mounts, permission issues, or incompatible security settings.

Troubleshooting:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
```

Verify the image entrypoint, command, arguments, mounted paths, security context, and volume configuration.

### `Terminating` for Too Long

A Pod may remain in `Terminating` because of a node problem, a volume detach issue, or a finalizer waiting for cleanup.

Troubleshooting:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get pod <pod-name> -n <namespace> -o yaml
kubectl get nodes
```

Find the blocking finalizer or node/storage problem first. Force deletion should be a last resort because it can leave processes or attached storage behind:

```bash
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

## A Practical Troubleshooting Flow

1. Confirm the namespace and Pod name: `kubectl get pods -A`.
2. Check status and node placement: `kubectl get pod <pod-name> -n <namespace> -o wide`.
3. Read events: `kubectl describe pod <pod-name> -n <namespace>`.
4. Read current and previous logs: `kubectl logs` with and without `--previous`.
5. Check dependencies such as Secrets, ConfigMaps, PVCs, Services, and image registries.
6. Check node health and capacity with `kubectl get nodes`, `kubectl describe node`, and `kubectl top`.
7. Fix the root cause, then watch the rollout: `kubectl get pods -n <namespace> -w`.

Avoid deleting a failing Pod before collecting its events and logs. Those details are often the best evidence of what went wrong.