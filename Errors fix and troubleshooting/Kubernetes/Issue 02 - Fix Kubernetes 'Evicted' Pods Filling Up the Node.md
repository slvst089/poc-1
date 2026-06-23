### _Issue 02 - Fix Kubernetes 'Evicted' Pods Filling Up the Node_
###### Published Time: 2026-03-30


#### [The Symptom](https://devopsil.com/articles/2026-03-30-kubernetes-evicted-pods-cleanup-fix#the-symptom)

Running `kubectl get pods` shows dozens or hundreds of pods in `Evicted` status:

```
NAME                       READY   STATUS    RESTARTS   AGE
my-app-6d8f9b7c5-a2k4l    0/1     Evicted   0          3d
my-app-6d8f9b7c5-b3m5n    0/1     Evicted   0          3d
my-app-6d8f9b7c5-c4p6q    0/1     Evicted   0          2d
my-app-6d8f9b7c5-d5r7s    0/1     Evicted   0          2d
# ... hundreds more
```

These evicted pods consume etcd storage and clutter your view, and the underlying cause may still be actively evicting new pods.
<br><br>
#### [Root Cause](https://devopsil.com/articles/2026-03-30-kubernetes-evicted-pods-cleanup-fix#root-cause)

[Kubernetes](https://devopsil.com/tools/kubernetes "Kubernetes") evicts pods when a node runs out of critical resources. The kubelet monitors disk space, memory, and inodes, and when usage crosses a threshold, it starts evicting pods to protect the node. The most common triggers are:

1.   **Disk pressure:** Node's root or image filesystem exceeds the eviction threshold (default: 85% usage for soft eviction, 100% minus `imagefs.available` for hard eviction).
2.   **Memory pressure:** Node is running out of RAM and the OOM killer has not freed enough.
3.   **Excessive container logs** filling up the disk.
4.   **Large container images** consuming image storage.
5.   **Ephemeral storage** usage by pods exceeding their limits.

Evicted pods are not automatically cleaned up by Kubernetes. They remain as terminated pod records until manually deleted or garbage collected.
<br><br>
#### [Step-by-Step Fix](https://devopsil.com/articles/2026-03-30-kubernetes-evicted-pods-cleanup-fix#step-by-step-fix)

##### [1. Clean up evicted pods immediately](https://devopsil.com/articles/2026-03-30-kubernetes-evicted-pods-cleanup-fix#1-clean-up-evicted-pods-immediately)

```
# Delete all evicted pods across all namespaces
kubectl get pods --all-namespaces --field-selector=status.phase=Failed \
  -o json | kubectl delete -f -

# Or target a specific namespace
kubectl get pods -n my-namespace --field-selector=status.phase=Failed \
  -o name | xargs kubectl delete -n my-namespace
```

Quick one-liner:

```
kubectl get pods --all-namespaces | awk '$4=="Evicted" {print "kubectl delete pod -n " $1 " " $2}' | sh
```

##### [2. Identify the eviction cause](https://devopsil.com/articles/2026-03-30-kubernetes-evicted-pods-cleanup-fix#2-identify-the-eviction-cause)

```
kubectl describe pod <evicted-pod-name> -n <namespace>
```

Look at the `Status` and `Message` fields:

```
Status:    Failed
Reason:    Evicted
Message:   The node was low on resource: ephemeral-storage.
```

Check node conditions:

```
kubectl describe node <node-name> | grep -A5 "Conditions"
```

Look for `DiskPressure`, `MemoryPressure`, or `PIDPressure` conditions showing `True`.

##### [3. Fix disk pressure](https://devopsil.com/articles/2026-03-30-kubernetes-evicted-pods-cleanup-fix#3-fix-disk-pressure)

```
# Check disk usage on the node
df -h /var/lib/kubelet
df -h /var/lib/docker   # or /var/lib/containerd

# Clean unused container images
docker system prune -af    # Docker runtime
crictl rmi --prune          # containerd runtime

# Clean old container logs
find /var/log/containers -name "*.log" -mtime +7 -delete
```

##### [4. Configure proper resource limits](https://devopsil.com/articles/2026-03-30-kubernetes-evicted-pods-cleanup-fix#4-configure-proper-resource-limits)

Add ephemeral storage limits to your pod specs to prevent individual pods from consuming too much disk:

```
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "2Gi"
```

##### [5. Set up log rotation](https://devopsil.com/articles/2026-03-30-kubernetes-evicted-pods-cleanup-fix#5-set-up-log-rotation)

Configure the kubelet to rotate container logs:

```
# kubelet config
containerLogMaxSize: "50Mi"
containerLogMaxFiles: 3
```

Or if using a logging agent, ensure it handles rotation:

```
# Docker daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
```

##### [6. Enable automatic cleanup of terminated pods](https://devopsil.com/articles/2026-03-30-kubernetes-evicted-pods-cleanup-fix#6-enable-automatic-cleanup-of-terminated-pods)

Configure the kube-controller-manager to garbage collect terminated pods:

```
--terminated-pod-gc-threshold=100
```

This limits the number of terminated pods kept per node. Most managed Kubernetes services (EKS, GKE, AKS) already set a reasonable default.
<br><br>
#### [Prevention Tips](https://devopsil.com/articles/2026-03-30-kubernetes-evicted-pods-cleanup-fix#prevention-tips)

*   **Always set resource requests and limits.** Pods without limits can consume unbounded resources and trigger evictions of other pods.
*   **Monitor node disk usage.** Alert when disk usage exceeds 70% so you can intervene before evictions start.
*   **Use a CronJob to clean evicted pods.** Schedule a periodic cleanup so evicted pods do not accumulate.
*   **Right-size your nodes.** If evictions are frequent, your nodes may be too small for the workloads. Consider larger instance types or adding more nodes.
*   **Implement Pod Disruption Budgets.** PDBs do not prevent evictions from resource pressure, but they protect against voluntary disruptions during maintenance.
