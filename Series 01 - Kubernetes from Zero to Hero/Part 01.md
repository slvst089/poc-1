## _Part 01 of 08 - Kubectl Cheat Sheet: Every Command You Need_
##### Published Time: 2026-03-20

#### [Cluster & Context](https://devopsil.com/articles/2026-03-20-kubectl-cheat-sheet#cluster--context)

```
# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context my-cluster

# Set default namespace
kubectl config set-context --current --namespace=production
```

#### [Pods](https://devopsil.com/articles/2026-03-20-kubectl-cheat-sheet#pods)

```
# List pods
kubectl get pods
kubectl get pods -A                    # All namespaces
kubectl get pods -o wide               # With node/IP info
kubectl get pods --sort-by=.status.startTime

# Pod details
kubectl describe pod my-pod
kubectl get pod my-pod -o yaml

# Logs
kubectl logs my-pod
kubectl logs my-pod -c my-container    # Specific container
kubectl logs my-pod --previous         # Previous crash
kubectl logs -f my-pod                 # Follow/stream
kubectl logs -l app=nginx              # By label

# Execute into pod
kubectl exec -it my-pod -- /bin/sh
kubectl exec -it my-pod -c my-container -- bash

# Port forward
kubectl port-forward pod/my-pod 8080:80
kubectl port-forward svc/my-service 8080:80

# Delete
kubectl delete pod my-pod
kubectl delete pod my-pod --force --grace-period=0  # Force kill
```

#### [Deployments](https://devopsil.com/articles/2026-03-20-kubectl-cheat-sheet#deployments)

```
# Create/update
kubectl apply -f deployment.yaml
kubectl create deployment nginx --image=nginx:1.25

# Scale
kubectl scale deployment nginx --replicas=5

# Rollout
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2
kubectl rollout restart deployment/nginx

# Update image
kubectl set image deployment/nginx nginx=nginx:1.26
```

#### [Services & Networking](https://devopsil.com/articles/2026-03-20-kubectl-cheat-sheet#services--networking)

```
# List services
kubectl get svc
kubectl get endpoints

# Expose deployment
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# DNS debugging
kubectl run dnstest --image=busybox:1.36 --rm -it -- nslookup my-service
```

#### [Debugging](https://devopsil.com/articles/2026-03-20-kubectl-cheat-sheet#debugging)

```
# Resource usage
kubectl top pods
kubectl top nodes

# Events (sorted by time)
kubectl get events --sort-by=.lastTimestamp

# Debug with ephemeral container
kubectl debug my-pod -it --image=busybox

# Check why pod isn't running
kubectl describe pod my-pod | grep -A 5 "Events"

# Node conditions
kubectl describe node my-node | grep -A 5 "Conditions"
```

#### [Resource Management](https://devopsil.com/articles/2026-03-20-kubectl-cheat-sheet#resource-management)

```
# Apply/delete from file
kubectl apply -f .                     # All YAML in directory
kubectl apply -f https://url/to/manifest.yaml
kubectl delete -f deployment.yaml

# Dry run + diff
kubectl apply -f deployment.yaml --dry-run=client
kubectl diff -f deployment.yaml

# Edit live resource
kubectl edit deployment nginx

# Patch
kubectl patch deployment nginx -p '{"spec":{"replicas":3}}'
```

#### [Labels & Selectors](https://devopsil.com/articles/2026-03-20-kubectl-cheat-sheet#labels--selectors)

```
# Filter by label
kubectl get pods -l app=nginx
kubectl get pods -l 'app in (nginx, redis)'

# Add/remove labels
kubectl label pod my-pod env=prod
kubectl label pod my-pod env-                # Remove
```

#### [Secrets & ConfigMaps](https://devopsil.com/articles/2026-03-20-kubectl-cheat-sheet#secrets--configmaps)

```
# Create secret
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=s3cret

# Create configmap
kubectl create configmap app-config \
  --from-file=config.yaml \
  --from-literal=LOG_LEVEL=info

# View (base64 decoded)
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

#### [Quick JSON Queries](https://devopsil.com/articles/2026-03-20-kubectl-cheat-sheet#quick-json-queries)

```
# Get all pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# Get container images
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Get pods not running
kubectl get pods --field-selector=status.phase!=Running
