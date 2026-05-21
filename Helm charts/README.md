# Helm Chart

Absract framework sits over Kubernetes Cluster. Minimal helm chart command to communicate with our K8 cluster.

## How Kubernetes help with containers

1. Scalability - You can scale number of docker containers to serve millions of reqeusts
2. Automate Docker Deployment - k8s can help you to manage automate the docker deployment
3. Auto-healing - If a container is not healthy the k8s can auto replace the container
4. Rollout and Rollback - k8s monitors the uhealthy docker container and restart unresponsive.

## helm commands

- helm install <chart-file>
- helm uninstall <chart-file>

## Helm Char Architecture

![HELM CHAR ARCHITECTURE](./Images/Helm-chart-architecture.png)

- left hand side is helm chart
- right side is kubernets cluster

## Basic HELM cli commands

- helm create <helloworld(helm chart name)>
- helm install myhelloworld<FIRST_ARGUMENT_RELEASE_NAME> helloworld<SECOND_ARGUMENT_CHART_NAME>
- helm delete myhelloworld
- helm list -A

### to check where exactly our service is running

```
kubectl get service
```

```
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes     ClusterIP   10.**.**.**       <none>        443/TCP        71m
myhelloworld   NodePort    10.**.**.**       <none>        80:31486/TCP   3m7s
```

### helm cli more commands

```
helm create
helm install
helm upgrade <release name> <chart name> -> to install new changes and will increase revision number
helm rollback <release name> <revision_number(to which you want to rollback)>

helm -debug -dry-run (validate your helm chart before install)
example:- helm install myhelloworldrelease --debug --dry-run helloworld


helm template <chart name>
helm lint <chart nane> (use to verify if there any misconfiguration or error associated with helm chart)
hlem uninstall <chart release name>
```

![HELM TEMPLATE COMMAND](./Images/helm-template-command.png)

## Incident: Intermittent connection failures caused by Kubernetes probes

### Summary

We saw intermittent failures when calling a service deployed via Helm in Kubernetes. Symptoms included:

- `curl: (52) Empty reply from server`
- `curl: (56) Recv failure: Connection was aborted`
- requests sometimes connecting but failing immediately
- inconsistent behavior when accessing via `localhost`

At first this looked like a networking / WSL / Docker port-forwarding issue, but the root cause was Kubernetes health probes.

---

### Root cause

The issue was caused by misconfigured **liveness and/or readiness probes**.

When the probes were not matching the actual application behavior (wrong path, wrong port, or too strict timing), Kubernetes treated the pod as unhealthy:

- **Readiness probe failures** → pod marked NotReady → removed from service endpoints
- **Liveness probe failures** → container restarted repeatedly

This caused traffic to either be dropped or hit a container that was restarting.

---

### What this looked like in practice

| Symptom | Actual cause |
|----------|--------------|
| `Empty reply from server` | container closed connection during restart |
| `Connection aborted` | request hit pod during restart cycle |
| intermittent access | pod switching between Ready / NotReady |
| curl connects but fails immediately | liveness probe triggering restarts |
| 404 on `/` | valid response, but wrong endpoint for probes |

---

### Evidence

Application logs showed requests were actually reaching the container:

```
"GET / HTTP/1.1" 404
```

This confirmed that networking, ingress, and port forwarding were working correctly. The issue was not transport-level.

---

### Fix

The issue was resolved by:

- fixing probe paths (using correct endpoints like `/health`)
- ensuring probes match actual application routes
- increasing startup tolerance (`initialDelaySeconds`, timeouts)
- separating health endpoints from application routes

After this:

- pods stayed `1/1 Ready`
- no restart loops
- stable responses from the service

---

### Lesson learned

Kubernetes probe misconfigurations can look exactly like network or connectivity issues.

Before debugging networking layers (WSL, Docker, ingress), always check:

- `kubectl get pods`
- readiness state
- restart count
- probe configuration

In this case, the network was fine — the service was being restarted / removed due to failing health checks.