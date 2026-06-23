### _Issue 04 - Fix Kubernetes OOMKilled: Pod Keeps Getting Killed for Memory_
###### Published Time: 2026-03-30


#### [The Error: OOMKilled (Exit Code 137)](https://devopsil.com/articles/2026-03-30-kubernetes-oomkilled-fix#the-error-oomkilled-exit-code-137)

You run `kubectl get pods` and see:

```
NAME                     READY   STATUS      RESTARTS   AGE
myapp-6b8f9d4c7-x2k9j   0/1     OOMKilled   5          12m
```

Or in `kubectl describe pod`:

```
State:          Terminated
Reason:         OOMKilled
Exit Code:      137
```

This means the Linux kernel's Out-Of-Memory (OOM) killer terminated your container because it exceeded its memory limit.
<br><br>
#### [Root Cause](https://devopsil.com/articles/2026-03-30-kubernetes-oomkilled-fix#root-cause)

[Kubernetes](https://devopsil.com/tools/kubernetes "Kubernetes") enforces memory limits set in your pod spec. When a container tries to allocate memory beyond its `resources.limits.memory` value, the kernel kills the process with signal 9 (SIGKILL). Common causes:

*   Memory limit set too low for the workload
*   Application memory leak (especially in long-running services)
*   JVM heap not aligned with container memory limit
*   Sidecar containers consuming shared memory budget
<br><br>
#### [Step-by-Step Fix](https://devopsil.com/articles/2026-03-30-kubernetes-oomkilled-fix#step-by-step-fix)

##### [1. Check current memory usage and limits](https://devopsil.com/articles/2026-03-30-kubernetes-oomkilled-fix#1-check-current-memory-usage-and-limits)

```
# See the pod's resource configuration
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Limits\|Requests"

# Check actual memory usage across pods
kubectl top pods -n <namespace> --sort-by=memory
```

##### [2. Review recent memory consumption](https://devopsil.com/articles/2026-03-30-kubernetes-oomkilled-fix#2-review-recent-memory-consumption)

```
# Get previous container logs (before it was killed)
kubectl logs <pod-name> -n <namespace> --previous

# Check events for OOM history
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | grep OOM
```

##### [3. Increase the memory limit](https://devopsil.com/articles/2026-03-30-kubernetes-oomkilled-fix#3-increase-the-memory-limit)

Edit your deployment manifest:

```
resources:
  requests:
    memory: "256Mi"   # Scheduler guarantee
    cpu: "100m"
  limits:
    memory: "512Mi"   # Hard ceiling — OOMKilled if exceeded
    cpu: "500m"
```

Apply the change:

```
kubectl apply -f deployment.yaml
```

##### [4. For JVM applications, align heap with container limits](https://devopsil.com/articles/2026-03-30-kubernetes-oomkilled-fix#4-for-jvm-applications-align-heap-with-container-limits)

The JVM does not automatically respect container memory limits in older versions. Set heap explicitly:

```
env:
  - name: JAVA_OPTS
    value: "-XX:MaxRAMPercentage=75.0 -XX:+UseContainerSupport"
```

This tells the JVM to use at most 75% of the container's memory limit, leaving room for non-heap memory (metaspace, thread stacks, NIO buffers).

##### [5. Profile memory to find leaks](https://devopsil.com/articles/2026-03-30-kubernetes-oomkilled-fix#5-profile-memory-to-find-leaks)

If the limit seems adequate, the application may be leaking:

```
# Port-forward and attach a profiler
kubectl port-forward <pod-name> 8080:8080

# For Node.js, enable heap snapshots
node --inspect=0.0.0.0:9229 app.js

# For Go, expose pprof
# Import _ "net/http/pprof" and hit /debug/pprof/heap
```

##### [6. Use a VPA to right-size automatically](https://devopsil.com/articles/2026-03-30-kubernetes-oomkilled-fix#6-use-a-vpa-to-right-size-automatically)

```
# Install Vertical Pod Autoscaler, then create a recommendation
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"   # "Off" = recommend only, "Auto" = apply changes
EOF

# Check recommendations
kubectl describe vpa myapp-vpa
```
<br><br>
#### [Prevention Tips](https://devopsil.com/articles/2026-03-30-kubernetes-oomkilled-fix#prevention-tips)

*   **Set requests close to actual usage** and limits 20-50% above requests to absorb spikes.
*   **Never omit memory limits** in production — unbounded pods can destabilize the entire node.
*   **Monitor with [Prometheus](https://devopsil.com/tools/prometheus "Prometheus")**: alert on `container_memory_working_set_bytes` approaching limits.
*   **Use `--previous` logs** immediately after an OOMKill to capture the state before restart.
*   **For JVMs**, always set `-XX:+UseContainerSupport` (default in JDK 11+) and cap `MaxRAMPercentage`.
