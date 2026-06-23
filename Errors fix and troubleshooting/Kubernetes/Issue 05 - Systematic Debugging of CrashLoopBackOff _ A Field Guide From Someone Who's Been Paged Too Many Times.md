### _Issue 05 - Systematic Debugging of CrashLoopBackOff: A Field Guide From Someone Who's Been Paged Too Many Times_
###### Published Time: 2026-03-22


#### [CrashLoopBackOff Is a Symptom, Not a Diagnosis](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#crashloopbackoff-is-a-symptom-not-a-diagnosis)

Here's the thing about CrashLoopBackOff — it tells you exactly one thing: your container started and then exited, and [Kubernetes](https://devopsil.com/tools/kubernetes "Kubernetes") is restarting it with exponential backoff. That's it. The actual problem could be any of two dozen different root causes, and the approach you take to debug it matters.
I've watched engineers spend hours staring at `kubectl get pods` waiting for the status to change, or blindly deleting and recreating pods hoping the problem goes away. Let me tell you why a systematic approach saves you time every single time, and walk you through the decision tree I use when I get paged at 3 AM.
<br><br>
#### [Step 0: Gather the Facts Before You Touch Anything](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#step-0-gather-the-facts-before-you-touch-anything)

Before you start fixing, start observing. Run these commands first and read the output carefully:

```
# Get the pod status and restart count
kubectl get pod $POD_NAME -n $NAMESPACE -o wide

# Get detailed pod information including events and container states
kubectl describe pod $POD_NAME -n $NAMESPACE

# Get the exit code from the last termination
kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
```

The exit code is your most important clue. Write it down before doing anything else.

```
Exit Code 0   → Container exited successfully (shouldn't be restarting — check restartPolicy)
Exit Code 1   → Application error (generic failure, check logs)
Exit Code 2   → Shell/command misuse (bad entrypoint or command syntax)
Exit Code 126 → Permission denied on entrypoint
Exit Code 127 → Entrypoint or command not found
Exit Code 137 → SIGKILL (OOM kill or external termination)
Exit Code 139 → SIGSEGV (segmentation fault — native code crash)
Exit Code 143 → SIGTERM (graceful shutdown, but container didn't stop in time)
```

#### [Step 1: Check the Logs](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#step-1-check-the-logs)

This sounds obvious, but there's a nuance. When a pod is in CrashLoopBackOff, the current container might not have any logs yet because it hasn't started. You need the previous container's logs:

```
# Current container logs (might be empty or very short)
kubectl logs $POD_NAME -n $NAMESPACE

# Previous container logs (this is usually what you want)
kubectl logs $POD_NAME -n $NAMESPACE --previous

# If the pod has multiple containers, specify which one
kubectl logs $POD_NAME -n $NAMESPACE -c $CONTAINER_NAME --previous
```

Here's the thing — if `--previous` returns nothing, the container is crashing before it can write any log output. This usually means the problem is at the OS/runtime level, not the application level. Skip ahead to Step 4.

#### [Step 2: Application-Level Failures (Exit Code 1)](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#step-2-application-level-failures-exit-code-1)

Exit code 1 is the most common and the least specific. Your application started, encountered an error, and exited. The logs from Step 1 should tell you what happened. Common causes:

##### [Missing Configuration](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#missing-configuration)

The application expects an environment variable or config file that doesn't exist:

```
# Check what environment variables are actually set in the container
kubectl exec -it $POD_NAME -n $NAMESPACE -- env 2>/dev/null

# If the pod keeps crashing, use a debug container
kubectl debug -it $POD_NAME -n $NAMESPACE --image=busybox --target=$CONTAINER_NAME -- sh
```

Verify that ConfigMaps and Secrets referenced by the pod actually exist:

```
# List all ConfigMap and Secret references in the pod spec
kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{range .spec.containers[*].envFrom[*]}{.configMapRef.name}{.secretRef.name}{"\n"}{end}'

# Check if they exist
kubectl get configmap $CONFIGMAP_NAME -n $NAMESPACE
kubectl get secret $SECRET_NAME -n $NAMESPACE
```

##### [Failed Database or Service Connections](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#failed-database-or-service-connections)

The application tries to connect to a dependency at startup and fails. Look for connection timeout errors in the logs. Verify the dependency is reachable from the pod's network namespace:

```
# Test connectivity from within the pod's network
kubectl debug -it $POD_NAME -n $NAMESPACE --image=nicolaka/netshoot --target=$CONTAINER_NAME -- \
  nc -zv database-service.production.svc.cluster.local 5432
```

##### [Missing or Incompatible Dependencies](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#missing-or-incompatible-dependencies)

A common one after image updates — the new version expects a library, schema, or file that isn't present:

```
# Shell into the container image to inspect it
kubectl run debug-shell --rm -it --image=$IMAGE_NAME -- /bin/sh
```

#### [Step 3: OOM Kills (Exit Code 137)](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#step-3-oom-kills-exit-code-137)

Exit code 137 means the container received SIGKILL. In Kubernetes, this is almost always an OOM (Out of Memory) kill. The container used more memory than its limit allows.

```
# Confirm it was an OOM kill
kubectl describe pod $POD_NAME -n $NAMESPACE | grep -A3 "Last State"
# Look for: Reason: OOMKilled

# Check the memory limit
kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.containers[0].resources.limits.memory}'

# Check actual memory usage before the kill (if metrics-server is available)
kubectl top pod $POD_NAME -n $NAMESPACE
```

The fix depends on whether the memory usage is legitimate or a leak:

**Legitimate high usage:** Increase the memory limit. But don't guess — check historical usage in your monitoring system first. Set the limit to P99 usage plus 20-30% headroom.

**Memory leak:** The container's memory grows steadily over time until it hits the limit. Increasing the limit only delays the crash. You need to fix the leak in the application code, or as a temporary measure, reduce the livenessProbe threshold to restart the container before it hits the OOM limit.

```
# Temporary workaround for a memory leak: restart before OOM
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

Let me tell you why this is a workaround and not a fix: you're trading OOM kills for regular restarts, which is marginally better for your users but still causes downtime. Fix the leak.

#### [Step 4: Container Won't Start (Exit Codes 126, 127)](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#step-4-container-wont-start-exit-codes-126-127)

These exit codes mean the container runtime couldn't execute the entrypoint.

**Exit code 127 — command not found:**

```
# Check the entrypoint/command in the pod spec
kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.containers[0].command}'
kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.containers[0].args}'

# Common causes:
# 1. Typo in the command path
# 2. The binary exists but the base image changed (e.g., switched from debian to alpine, /bin/bash → /bin/sh)
# 3. Multi-stage Docker build didn't copy the binary
```

**Exit code 126 — permission denied:**

```
# The entrypoint exists but isn't executable
# Check file permissions in the image
kubectl run debug --rm -it --image=$IMAGE_NAME -- ls -la /app/entrypoint.sh

# Fix: chmod +x in the Dockerfile, or adjust securityContext
```

Also check if the securityContext is preventing execution:

```
# A restrictive securityContext can prevent binary execution
securityContext:
  readOnlyRootFilesystem: true  # App might need to write temp files
  runAsNonRoot: true            # Binary might be owned by root
  runAsUser: 1000               # User might not have permission to execute
```

#### [Step 5: Image Pull Issues Masquerading as CrashLoop](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#step-5-image-pull-issues-masquerading-as-crashloop)

Sometimes what looks like CrashLoopBackOff started as an image pull problem. The pod pulled a wrong or corrupted image, the container starts with unexpected contents, and crashes immediately.

```
# Verify the exact image being used (including digest)
kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.status.containerStatuses[0].imageID}'

# Check for ImagePullBackOff in events
kubectl describe pod $POD_NAME -n $NAMESPACE | grep -A5 "Events"
```

A particularly nasty variant: the `latest` tag was updated and the new image has a breaking change. The pod restarts, pulls the new image, crashes, restarts, pulls the same broken image, crashes. This is why you should never use `latest` in production. Pin your image tags or use digests.

#### [Step 6: Liveness Probe Killing Healthy Containers](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#step-6-liveness-probe-killing-healthy-containers)

Here's the thing that trips up even experienced operators: a misconfigured liveness probe can cause CrashLoopBackOff that has nothing to do with the application being unhealthy.

```
# Check the liveness probe configuration
kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.containers[0].livenessProbe}' | jq .
```

Common probe misconfigurations:

```
# Problem: initialDelaySeconds is too short for the app to start
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5   # App takes 30 seconds to start
  periodSeconds: 10
  failureThreshold: 3

# Fix: use a startupProbe for slow-starting apps
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
  # Allows up to 300 seconds (30 * 10) for startup
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
```

The startupProbe disables the liveness probe until the container passes the startup check. This is the correct solution for applications with variable startup times, not increasing `initialDelaySeconds` to some arbitrary high number.

#### [Step 7: Volume Mount Failures](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#step-7-volume-mount-failures)

If a container depends on a volume that fails to mount, it can crash at startup with confusing errors:

```
# Check for volume-related events
kubectl describe pod $POD_NAME -n $NAMESPACE | grep -i -A2 "volume\|mount\|attach"

# Check if PVCs are bound
kubectl get pvc -n $NAMESPACE

# Common issues:
# - PVC is Pending (no PV available or storage class misconfigured)
# - Secret or ConfigMap referenced as volume doesn't exist
# - ReadOnlyRootFilesystem conflicts with app needing to write
```

#### [The Debugging Decision Tree](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#the-debugging-decision-tree)

When you get paged, follow this order:

```
1. kubectl describe pod → get exit code and events
2. kubectl logs --previous → get application error output
   ├── Got logs? → Read them. The answer is usually there.
   └── No logs? → Problem is pre-application (image, permissions, volumes)
3. Exit code 137? → OOM kill. Check memory limits and usage.
4. Exit code 1? → App error. Check config, dependencies, connectivity.
5. Exit code 127/126? → Binary not found or not executable. Check image and securityContext.
6. No obvious exit code? → Check liveness probe. Check volume mounts. Check init containers.
7. Still stuck? → kubectl debug with a debug container and investigate from inside.
```

#### [Prevention: Stop CrashLoops Before They Happen](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#prevention-stop-crashloops-before-they-happen)

The best debugging session is the one that never happens. Here's what I enforce on every cluster I manage:

```
# 1. Always use startupProbes for apps that take time to initialize
# 2. Always set resource limits (especially memory)
# 3. Never use :latest tags in production
# 4. Always have readiness probes (separate from liveness)
# 5. Run pre-deploy checks that verify ConfigMaps and Secrets exist
```

Set up alerts on container restart counts:

```
# Alert when a container has restarted more than 3 times in 15 minutes
increase(kube_pod_container_status_restarts_total[15m]) > 3
```

This catches CrashLoopBackOff early, often before it pages you at 3 AM.
<br><br>
#### [Final Thoughts](https://devopsil.com/articles/2026-03-22-kubernetes-troubleshooting-crashloopbackoff#final-thoughts)
CrashLoopBackOff feels scary because it's vague. But once you have a systematic approach — exit code, logs, then targeted investigation — it becomes a mechanical process. The exit code tells you the category, the logs tell you the specifics, and the fix follows from the diagnosis.
Let me tell you why I wrote this as a decision tree rather than a list of tips: at 3 AM, you don't want to think creatively. You want a checklist. Follow the steps, gather the data, and the root cause will present itself. Every single time.
