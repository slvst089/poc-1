### _Issue 03 - Fix Kubernetes ImagePullBackOff: Container Image Won't Pull_
###### Published Time: 2026-03-30


#### [The Error: ImagePullBackOff](https://devopsil.com/articles/2026-03-30-kubernetes-imagepullbackoff-fix#the-error-imagepullbackoff)

Your pod is stuck and never reaches Running:

```
NAME                     READY   STATUS             RESTARTS   AGE
myapp-7d5b8f6c9-q3m2n   0/1     ImagePullBackOff   0          4m
```

Running `kubectl describe pod` shows:

```
Events:
  Warning  Failed   pull image "registry.example.com/myapp:v2.1":
           rpc error: code = Unknown desc = Error response from daemon:
           manifest for registry.example.com/myapp:v2.1 not found
  Warning  Failed   Back-off pulling image "registry.example.com/myapp:v2.1"
```

[Kubernetes](https://devopsil.com/tools/kubernetes "Kubernetes") tried to pull the container image, failed, and is now backing off with exponential delays.
<br><br>
#### [Root Cause](https://devopsil.com/articles/2026-03-30-kubernetes-imagepullbackoff-fix#root-cause)

ImagePullBackOff has four common causes:

1.   **Image tag does not exist** — typo in the tag or image was never pushed
2.   **Missing or invalid registry credentials** — private registry requires an `imagePullSecret`
3.   **Registry is unreachable** — network policy, firewall, or DNS blocking access from cluster nodes
4.   **Rate limiting** — [Docker](https://devopsil.com/tools/docker "Docker") Hub throttles anonymous pulls (100 pulls/6h per IP)
<br><br>
#### [Step-by-Step Fix](https://devopsil.com/articles/2026-03-30-kubernetes-imagepullbackoff-fix#step-by-step-fix)

##### [1. Verify the image exists](https://devopsil.com/articles/2026-03-30-kubernetes-imagepullbackoff-fix#1-verify-the-image-exists)

```
# Check if the image and tag exist in the registry
docker manifest inspect registry.example.com/myapp:v2.1

# For Docker Hub images
docker manifest inspect docker.io/library/nginx:1.25.99
# "no such manifest" = the tag doesn't exist
```

##### [2. Check for typos in the deployment](https://devopsil.com/articles/2026-03-30-kubernetes-imagepullbackoff-fix#2-check-for-typos-in-the-deployment)

```
kubectl get deployment myapp -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Common mistakes: wrong registry URL, missing namespace (`myapp:v1` vs `myorg/myapp:v1`), wrong tag.

##### [3. Fix registry authentication](https://devopsil.com/articles/2026-03-30-kubernetes-imagepullbackoff-fix#3-fix-registry-authentication)

Create a pull secret if your registry is private:

```
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=deploy-bot \
  --docker-password="$REGISTRY_TOKEN" \
  --docker-email=deploy@example.com \
  -n <namespace>
```

Reference it in your pod spec:

```
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: myapp
      image: registry.example.com/myapp:v2.1
```

##### [4. Test pulling from a cluster node directly](https://devopsil.com/articles/2026-03-30-kubernetes-imagepullbackoff-fix#4-test-pulling-from-a-cluster-node-directly)

SSH into a node and try pulling manually:

```
# On the node
crictl pull registry.example.com/myapp:v2.1

# If using Docker as the runtime
docker pull registry.example.com/myapp:v2.1
```

This isolates whether the issue is Kubernetes config or network/registry.

##### [5. Fix Docker Hub rate limiting](https://devopsil.com/articles/2026-03-30-kubernetes-imagepullbackoff-fix#5-fix-docker-hub-rate-limiting)

Authenticate to Docker Hub even for public images:

```
kubectl create secret docker-registry dockerhub-cred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=yourusername \
  --docker-password="$DOCKERHUB_TOKEN" \
  -n <namespace>
```

Or mirror frequently-used images to your own registry.

##### [6. Check network policies](https://devopsil.com/articles/2026-03-30-kubernetes-imagepullbackoff-fix#6-check-network-policies)

```
# Verify nodes can reach the registry
kubectl run test-pull --image=registry.example.com/myapp:v2.1 --restart=Never

# Check if NetworkPolicies block egress
kubectl get networkpolicies -n <namespace> -o yaml | grep -A 10 egress
```

##### [7. Force a fresh pull](https://devopsil.com/articles/2026-03-30-kubernetes-imagepullbackoff-fix#7-force-a-fresh-pull)

```
# Delete the stuck pod to trigger a new attempt
kubectl delete pod <pod-name> -n <namespace>

# Or roll the deployment
kubectl rollout restart deployment myapp -n <namespace>
```
<br><br>
#### [Prevention Tips](https://devopsil.com/articles/2026-03-30-kubernetes-imagepullbackoff-fix#prevention-tips)

*   **Use immutable tags** (e.g., SHA digests `myapp@sha256:abc123...`) instead of mutable tags like `latest`.
*   **Set `imagePullPolicy: IfNotPresent`** for tagged images to reduce pull frequency and avoid rate limits.
*   **Attach `imagePullSecrets` to the ServiceAccount** so every pod in the namespace inherits credentials automatically: 
```
kubectl patch serviceaccount default -n <namespace> \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```
*   **Mirror critical images** to a private registry so you are not dependent on external availability.
*   **Validate image references in CI** before deploying — catch typos before they reach the cluster.
