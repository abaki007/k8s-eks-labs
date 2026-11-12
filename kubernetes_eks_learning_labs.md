# Kubernetes and EKS Learning Labs

A practical, Helm-first learning plan for a platform engineer with AWS, Terraform, Lambda, serverless and Control Tower experience.

This document consolidates the agreed labs, including the revised Helm-first Topic 2. It uses one application throughout and introduces one platform capability at a time.

## Contents

1. [Lab conventions](#lab-conventions)
2. [Kubernetes and EKS fundamentals](#1-kubernetes-and-eks-fundamentals)
3. [Helm and deploying your first application](#2-helm-and-deploying-your-first-application)
4. [Debugging and operations](#3-debugging-and-operations)
5. [Consul service mesh and mTLS](#4-consul-service-mesh-and-mtls)
6. [HPA and Pod scaling](#5-hpa-and-pod-scaling)
7. [Argo CD and GitOps](#6-argo-cd-and-gitops)
8. [Observability and RCA](#7-observability-and-rca)
9. [Additional platform topics](#8-additional-platform-topics)
10. [Further labs](#9-further-labs)
11. [Final capstone](#10-final-capstone)

## Lab conventions

- **Environment:** non-production EKS; optional K3s for portable Kubernetes labs.
- **Namespace:** `k8slearning`.
- **Application:** `hello`, later deployed as `frontend` and `backend`.
- **Repository:** application code, chart, environment values, GitOps configuration and lab notes.
- **Deployment approach:** use your existing Helm conventions from the outset. Do not maintain a separate plain-YAML deployment workflow.
- **Learning flow:** chart values -> Helm templates -> rendered Kubernetes resources -> live cluster behaviour.
- **Rendered YAML:** an inspection aid, not another deployment method or a separately maintained file.
- **Changes:** make durable configuration changes through chart templates and lab-specific values. Once Argo CD is introduced, make them through Git.
- **kubectl:** use primarily for inspection, testing and diagnosis, plus explicitly identified operational exercises.
- **Safety:** namespace-scoped faults only. Node maintenance, certificate experiments and infrastructure changes require a dedicated sandbox.
- **Existing Consul:** reuse the installed deployment. Inspect its configuration before onboarding workloads; do not reinstall it or alter shared security defaults.
- **Versions:** use documentation matching your installed Kubernetes, Helm, Consul and Argo CD versions.
- **Secrets:** keep credentials and secrets out of Git and lab notes.
- **Evidence:** record expectations, observations, commands, recovery and verification.

### Suggested lab record

For each lab, capture:

1. Objective and starting configuration.
2. Expected behaviour.
3. Changes made.
4. Evidence collected.
5. Explanation of observed behaviour.
6. Recovery or cleanup.
7. Completion check.

---

# 1. Kubernetes and EKS fundamentals

## Lab 1.1: Connect to and inventory your cluster

**Objective:** understand kubeconfig, identity, contexts and basic discovery.

### Exercises

For EKS:

```bash
aws sts get-caller-identity

aws eks update-kubeconfig \
  --region <region> \
  --name <cluster-name> \
  --alias learning-eks

kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
kubectl get pods -A
kubectl auth can-i create deployments -n k8slearning
```

`update-kubeconfig` configures access information and changes the current context. It does not grant Kubernetes permissions or provide network access to a private endpoint.

Then:

### 1. Create the learning namespace if authorised and it does not exist

Check access before making a change:

```bash
kubectl auth can-i create namespaces
kubectl get namespace k8slearningdan
```

If the first command returns `yes`, create the namespace only when it is absent:

```bash
kubectl get namespace k8slearningdan >/dev/null 2>&1 || \
  kubectl create namespace k8slearningdan

kubectl get namespace k8slearningdan
```

If the conditional command fails, use the diagnostics in step 5 rather than assuming that the namespace is absent.

### 2. Set it as the default namespace for the learning context

```bash
kubectl config set-context learning-eks --namespace=k8slearningdan
kubectl config use-context learning-eks

kubectl config view --minify \
  -o jsonpath='{.current-context}{" namespace="}{..namespace}{"\n"}'
```

Alternatively, update whichever context is currently active:

```bash
CONTEXT="$(kubectl config current-context)"
kubectl config set-context "$CONTEXT" --namespace=k8slearningdan
```

### 3. Identify CoreDNS, networking components and observability agents

Inspect CoreDNS:

```bash
kubectl -n kube-system get deployment,service,configmap,pods \
  -l k8s-app=kube-dns -o wide
kubectl -n kube-system get deployment coredns -o wide
kubectl -n kube-system get configmap coredns -o yaml
```

Inspect the standard EKS networking components and look for other CNI or networking controllers:

```bash
kubectl -n kube-system get daemonset aws-node kube-proxy -o wide
kubectl -n kube-system get pods -l k8s-app=aws-node -o wide
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

kubectl get daemonsets,deployments,pods -A -o wide | \
  grep -Ei 'aws-node|kube-proxy|vpc-cni|calico|cilium|weave|flannel|network|ingress|load-balancer'

kubectl get networkpolicies -A
kubectl get ingressclasses
kubectl get csidrivers
```

Identify common logging, metrics and tracing agents:

```bash
kubectl get daemonsets -A -o wide

kubectl get daemonsets,deployments,statefulsets,pods -A -o wide | \
  grep -Ei 'cloudwatch|fluent-bit|fluentd|prometheus|grafana|datadog|dynatrace|newrelic|splunk|otel|opentelemetry|filebeat|metricbeat|vector|loki|tempo|jaeger'
```

### 4. Record Kubernetes version, node types, add-ons and deployment ownership

Record the client, API server and kubelet versions:

```bash
kubectl version

kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion,OS:.status.nodeInfo.osImage,RUNTIME:.status.nodeInfo.containerRuntimeVersion'

aws eks describe-cluster \
  --region <region> \
  --name <cluster-name> \
  --query 'cluster.{Name:name,Version:version,PlatformVersion:platformVersion,Status:status}' \
  --output table
```

Record node instance types, node groups, compute types and architectures:

```bash
kubectl get nodes \
  -L node.kubernetes.io/instance-type,eks.amazonaws.com/nodegroup,eks.amazonaws.com/compute-type,kubernetes.io/arch \
  -o wide

aws eks list-nodegroups \
  --region <region> \
  --cluster-name <cluster-name>
```

Record EKS-managed add-ons:

```bash
aws eks list-addons \
  --region <region> \
  --cluster-name <cluster-name>

for ADDON in $(aws eks list-addons \
  --region <region> \
  --cluster-name <cluster-name> \
  --query 'addons[]' \
  --output text); do
  aws eks describe-addon \
    --region <region> \
    --cluster-name <cluster-name> \
    --addon-name "$ADDON" \
    --query 'addon.{Name:addonName,Version:addonVersion,Status:status,Owner:owner,Publisher:publisher,ServiceAccountRoleArn:serviceAccountRoleArn}' \
    --output table
done
```

Inspect deployment ownership labels and common deployment managers:

```bash
kubectl get deployments -A \
  -L app.kubernetes.io/managed-by,app.kubernetes.io/instance,app.kubernetes.io/part-of

kubectl get deployments -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,DEPLOYMENT:.metadata.name,OWNER-KIND:.metadata.ownerReferences[0].kind,OWNER-NAME:.metadata.ownerReferences[0].name,MANAGED-BY:.metadata.labels.app\.kubernetes\.io/managed-by'

helm list -A
kubectl get applications.argoproj.io -A 2>/dev/null
kubectl get kustomizations.kustomize.toolkit.fluxcd.io -A 2>/dev/null
kubectl get helmreleases.helm.toolkit.fluxcd.io -A 2>/dev/null
```

Most Deployments do not have Kubernetes `ownerReferences`. Helm, Argo CD, Flux and Terraform ownership is usually inferred from labels, annotations and the external controller's inventory.

### 5. Distinguish authentication, RBAC and endpoint connectivity failures

Test endpoint connectivity first:

```bash
kubectl cluster-info
kubectl get --raw='/readyz?verbose'
kubectl get namespaces --request-timeout=10s

ENDPOINT="$(kubectl config view --minify \
  -o jsonpath='{.clusters[0].cluster.server}')"
ENDPOINT_HOST="${ENDPOINT#https://}"
ENDPOINT_HOST="${ENDPOINT_HOST%%/*}"

getent hosts "$ENDPOINT_HOST"
curl -v --connect-timeout 10 "$ENDPOINT/readyz"
```

Timeouts, DNS failures, refused connections and TLS handshake timeouts indicate that the client cannot reliably reach the API endpoint. For a private EKS endpoint, verify VPN or network access, routing, DNS and security groups.

Test AWS identity and EKS authentication:

```bash
aws sts get-caller-identity
aws eks get-token \
  --region <region> \
  --cluster-name <cluster-name> \
  --query 'status.expirationTimestamp' \
  --output text
kubectl get namespaces
```

`401 Unauthorized`, credential prompts and token-generation errors indicate authentication failure: the endpoint was reached, but the client identity could not be established. Query only the token expiry as shown above; do not print or copy the bearer token into notes or tickets.

Test Kubernetes authorisation:

```bash
kubectl auth can-i get namespaces
kubectl auth can-i create namespaces
kubectl auth can-i create deployments -n k8slearning
kubectl auth can-i --list -n k8slearning
```

`403 Forbidden` or `User "..." cannot ...` indicates RBAC denial: the identity was authenticated, but it does not have permission for the requested operation.

| Failure | Typical signal | Meaning |
|---|---|---|
| Endpoint connectivity | Timeout, DNS error or refused connection | The client cannot reliably reach the API endpoint |
| Authentication | `401`, credentials requested or token failure | The endpoint was reached, but identity could not be established |
| RBAC authorisation | `403 Forbidden` or `cannot get/create` | Identity was established, but it lacks permission |

**Completion check:** identify the cluster, AWS identity and namespace before making changes.

### Optional home-lab branch

Install a single-node K3s cluster on a disposable Linux VM, then repeat discovery. Add an agent node later for scheduling and maintenance labs.

Default single-server K3s uses SQLite, not etcd. Learn etcd's role in Kubernetes, but do not expect a default K3s installation to expose an etcd cluster.


## Lab 1.2: Understand architecture and reconciliation

**Objective:** explain what each component does.

| Component | Question to answer |
|---|---|
| API server | How do clients and controllers interact with cluster state? |
| etcd | What state is persisted? |
| Scheduler | How is a node selected for an unscheduled Pod? |
| Controllers | How is desired state reconciled? |
| Kubelet | What ensures containers run on a node? |
| Container runtime | What starts the containers? |
| CNI | How does Pod networking work? |
| CoreDNS | How do applications resolve Services? |
| Service implementation | How does Service traffic reach backend Pods? |

### Exercises

These exercises use the `hello` release, deployed from the chart in this repository (`charts/hello`) with the dev values (`environments/dev.yaml`). You can also substitute the names of an existing approved learning workload.

```bash
NAMESPACE=k8slearning
RELEASE=hello
DEPLOYMENT=hello
CHART=./charts/hello
VALUES_FILE=environments/dev.yaml
```

If `hello` isn't deployed yet, install it from the repository root:

```bash
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install "$RELEASE" "$CHART" \
  -n "$NAMESPACE" \
  -f "$VALUES_FILE" \
  --wait \
  --timeout 5m
```

The chart deploys `traefik/whoami`, a small public image that replies with the hostname of the Pod that answered. That makes it easy to see which replica served a request, and that a replaced Pod has a new name. Topic 2 later swaps it for the image you build from `app.py`.

### 1. Inspect the learning Deployment

Confirm that the workload exists and inspect its desired and current state:

```bash
kubectl get deployment "$DEPLOYMENT" \
  -n "$NAMESPACE" \
  -o wide

kubectl describe deployment "$DEPLOYMENT" \
  -n "$NAMESPACE"

kubectl get deployment "$DEPLOYMENT" \
  -n "$NAMESPACE" \
  -o yaml
```

If the Deployment isn't found, install `hello` with the commands above first. Do not modify an unrelated or production Deployment for this exercise.

### 2. Inspect its ReplicaSet and Pods

List the workload resources using the standard Helm instance label:

```bash
kubectl get deployment,replicaset,pods \
  -n "$NAMESPACE" \
  -l app.kubernetes.io/instance="$RELEASE" \
  -o wide
```

If the chart does not use that standard label, inspect the Deployment selector and use the labels it shows:

```bash
kubectl get deployment "$DEPLOYMENT" \
  -n "$NAMESPACE" \
  -o jsonpath='{.spec.selector.matchLabels}{"\n"}'

kubectl get replicasets,pods \
  -n "$NAMESPACE" \
  --show-labels
```

Select the newest ReplicaSet and one of its Pods, then inspect them:

```bash
REPLICASET=$(kubectl get replicaset \
  -n "$NAMESPACE" \
  -l app.kubernetes.io/instance="$RELEASE" \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1:].metadata.name}')

POD=$(kubectl get pod \
  -n "$NAMESPACE" \
  -l app.kubernetes.io/instance="$RELEASE" \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[0].metadata.name}')

kubectl describe replicaset "$REPLICASET" -n "$NAMESPACE"
kubectl describe pod "$POD" -n "$NAMESPACE"
```

### 3. Follow ownerReferences

Display the controlling owner for the selected Pod:

```bash
kubectl get pod "$POD" \
  -n "$NAMESPACE" \
  -o custom-columns='NAME:.metadata.name,OWNER-KIND:.metadata.ownerReferences[0].kind,OWNER-NAME:.metadata.ownerReferences[0].name'
```

Read the Pod's controlling ReplicaSet and then that ReplicaSet's controlling Deployment:

```bash
POD_OWNER=$(kubectl get pod "$POD" \
  -n "$NAMESPACE" \
  -o jsonpath='{.metadata.ownerReferences[?(@.controller==true)].name}')

kubectl get replicaset "$POD_OWNER" \
  -n "$NAMESPACE" \
  -o custom-columns='NAME:.metadata.name,OWNER-KIND:.metadata.ownerReferences[0].kind,OWNER-NAME:.metadata.ownerReferences[0].name'

DEPLOYMENT_OWNER=$(kubectl get replicaset "$POD_OWNER" \
  -n "$NAMESPACE" \
  -o jsonpath='{.metadata.ownerReferences[?(@.controller==true)].name}')

printf 'Pod %s -> ReplicaSet %s -> Deployment %s\n' \
  "$POD" "$POD_OWNER" "$DEPLOYMENT_OWNER"
```

The chain should be `Pod -> ReplicaSet -> Deployment`. The Deployment controller manages ReplicaSets, while the ReplicaSet controller maintains the requested number of Pods.

### 4. Delete one lab Pod

Confirm the target and check your permission before deleting anything:

```bash
printf 'Namespace: %s\nPod: %s\n' "$NAMESPACE" "$POD"
kubectl auth can-i delete pods -n "$NAMESPACE"
kubectl get pod "$POD" -n "$NAMESPACE" -o wide
```

If the authorisation command returns `yes`, delete only that Pod:

```bash
kubectl delete pod "$POD" -n "$NAMESPACE"
```

### 5. Observe its replacement

Watch the ReplicaSet create a replacement Pod:

```bash
kubectl get pods \
  -n "$NAMESPACE" \
  -l app.kubernetes.io/instance="$RELEASE" \
  --watch
```

Stop the watch with `Ctrl+C` after the replacement Pod reports `Running` and ready. Then compare Pod names and creation times:

```bash
kubectl get pods \
  -n "$NAMESPACE" \
  -l app.kubernetes.io/instance="$RELEASE" \
  --sort-by=.metadata.creationTimestamp \
  -o custom-columns='NAME:.metadata.name,CREATED:.metadata.creationTimestamp,PHASE:.status.phase,READY:.status.containerStatuses[*].ready'
```

The new Pod has a different name because the ReplicaSet restores its desired replica count; Kubernetes does not restore the deleted Pod object itself.

### 6. Change the replica count through Helm values

Inspect the chart to confirm the replica value key. The example chart uses `replicaCount`:

```bash
helm show values "$CHART" | grep -n -E 'replica|autoscaling'
helm get values "$RELEASE" -n "$NAMESPACE"
```

Check whether a HorizontalPodAutoscaler manages the Deployment:

```bash
kubectl get hpa -n "$NAMESPACE"
```

For a workload that is not HPA-managed, apply a temporary Helm value override:

```bash
helm upgrade "$RELEASE" "$CHART" \
  -n "$NAMESPACE" \
  -f "$VALUES_FILE" \
  --set replicaCount=3 \
  --wait \
  --timeout 5m
```

Use the chart's actual replica key if it differs from `replicaCount`. In a Git-managed environment, make the value change in the approved values file and deploy it through the normal delivery workflow instead of using an imperative override.

### 7. Compare desired replicas with available replicas

Wait for the rollout and compare the Deployment counters:

```bash
kubectl rollout status deployment/"$DEPLOYMENT" \
  -n "$NAMESPACE" \
  --timeout=5m

kubectl get deployment "$DEPLOYMENT" \
  -n "$NAMESPACE" \
  -o custom-columns='NAME:.metadata.name,DESIRED:.spec.replicas,CURRENT:.status.replicas,UPDATED:.status.updatedReplicas,READY:.status.readyReplicas,AVAILABLE:.status.availableReplicas,UNAVAILABLE:.status.unavailableReplicas'

kubectl get replicasets,pods \
  -n "$NAMESPACE" \
  -l app.kubernetes.io/instance="$RELEASE" \
  -o wide
```

When reconciliation and readiness checks have completed successfully, desired, updated, ready and available replicas should all equal the configured replica count.

**Completion check:** explain why deleting a managed Pod does not remove the application.

---

# 2. Helm and deploying your first application

Use Helm from the outset to match your codebase. Learn the underlying Kubernetes resources by inspecting what Helm generates.

## Lab 2.1: Understand an existing chart

**Objective:** connect your codebase's chart structure to Kubernetes concepts.

### Exercises

Set the paths for the chart and its development values file. Adapt them to your repository:

```bash
CHART=./charts/hello
VALUES_FILE=./environments/dev.yaml
RELEASE=hello
```

The commands in this lab inspect and render the chart locally. They do not change the cluster.

### 1. Identify Chart.yaml, values files and templates

List the chart files:

```bash
find "$CHART" \
  -maxdepth 3 \
  -type f \
  -print | sort
```

Inspect the chart metadata and default values:

```bash
sed -n '1,200p' "$CHART/Chart.yaml"
sed -n '1,240p' "$CHART/values.yaml"
```

Find additional values files and template files:

```bash
find . \
  -type f \
  \( -name 'values.yaml' -o -name 'values-*.yaml' -o -name '*-values.yaml' \) \
  -print | sort

find "$CHART/templates" \
  -type f \
  \( -name '*.yaml' -o -name '*.yml' -o -name '*.tpl' \) \
  -print | sort
```

Validate the chart structure and metadata:

```bash
helm lint "$CHART"
helm show chart "$CHART"
helm show values "$CHART"
```

### 2. Locate Deployment, Service and configuration templates

Search the templates for common Kubernetes resource kinds:

```bash
grep -R -n -E \
  'kind:[[:space:]]*(Deployment|StatefulSet|DaemonSet|Service|ConfigMap|Secret)' \
  "$CHART/templates"
```

Search by conventional template file names as well:

```bash
find "$CHART/templates" \
  -type f \
  \( -iname '*deployment*' \
     -o -iname '*statefulset*' \
     -o -iname '*daemonset*' \
     -o -iname '*service*' \
     -o -iname '*configmap*' \
     -o -iname '*secret*' \) \
  -print | sort
```

Render the chart and list the resource kinds and names Helm would submit:

```bash
helm template "$RELEASE" "$CHART" \
  -f "$VALUES_FILE" \
  | grep -E '^(kind:|  name:)'
```

Inspect a specific rendered template when its path is known:

```bash
helm template "$RELEASE" "$CHART" \
  -f "$VALUES_FILE" \
  --show-only templates/deployment.yaml

helm template "$RELEASE" "$CHART" \
  -f "$VALUES_FILE" \
  --show-only templates/service.yaml
```

Adapt the `--show-only` paths if the repository uses different template names.

### 3. Trace values into rendered resources

Locate image, port, replica, probe and resource settings in the default and environment values:

```bash
grep -n -E \
  'image:|repository:|tag:|replica|port:|targetPort:|readiness|liveness|startup|resources:|requests:|limits:' \
  "$CHART/values.yaml" \
  "$VALUES_FILE"
```

Locate where templates consume those values:

```bash
grep -R -n -E \
  '\.Values\..*(image|repository|tag|replica|port|probe|resources|requests|limits)' \
  "$CHART/templates"
```

Render the effective development configuration and inspect the resulting fields:

```bash
helm template "$RELEASE" "$CHART" \
  -f "$VALUES_FILE" \
  | grep -n -E \
    'replicas:|image:|containerPort:|port:|targetPort:|startupProbe:|readinessProbe:|livenessProbe:|resources:|requests:|limits:'
```

Use a temporary command-line override to confirm that a value controls the expected rendered field:

```bash
helm template "$RELEASE" "$CHART" \
  -f "$VALUES_FILE" \
  --set replicaCount=3 \
  | grep -n -E 'kind: Deployment|replicas:'
```

Use the chart's actual value path if it does not use `replicaCount`. `helm template` renders locally and does not apply the override to the cluster.

### 4. Identify dependencies and shared helpers

Inspect dependency declarations and the dependency lock file, if present:

```bash
grep -n -A 30 '^dependencies:' "$CHART/Chart.yaml"
test ! -f "$CHART/Chart.lock" || sed -n '1,200p' "$CHART/Chart.lock"
helm dependency list "$CHART"
```

Locate helper templates:

```bash
find "$CHART" \
  -type f \
  -name '*.tpl' \
  -print | sort

grep -R -n -E \
  '\{\{[-]?[[:space:]]*(define|include|template)[[:space:]]' \
  "$CHART"
```

Inspect the usual helper file when present:

```bash
test ! -f "$CHART/templates/_helpers.tpl" || \
  sed -n '1,240p' "$CHART/templates/_helpers.tpl"
```

An `include` or `template` call using a name defined outside the application chart may come from a dependency or shared library chart.

### 5. Determine the chart ownership model

Find all charts in the repository:

```bash
find . \
  -type f \
  -name Chart.yaml \
  -not -path '*/.git/*' \
  -print | sort
```

Identify library charts:

```bash
grep -R -n -E '^type:[[:space:]]*library([[:space:]]|$)' \
  --include='Chart.yaml' .
```

Inspect the name, type and dependencies of each relevant `Chart.yaml`:

```bash
helm show chart <path-to-chart>
helm dependency list <path-to-chart>
```

Interpret the results:

- An application-specific chart usually contains workload templates for one service.
- A shared application chart is normally referenced as a dependency by several applications and exposes reusable values.
- A chart with `type: library` supplies helpers and cannot be installed by itself.
- A repository may combine application-specific wrapper charts with a shared application or library chart.

### 6. Identify environment-specific values

Find likely environment values files:

```bash
find . \
  -type f \
  \( -iname '*dev*.yaml' \
     -o -iname '*test*.yaml' \
     -o -iname '*staging*.yaml' \
     -o -iname '*prod*.yaml' \
     -o -path '*/environments/*.yaml' \) \
  -not -path '*/.git/*' \
  -print | sort
```

Search deployment configuration for Helm values arguments:

```bash
grep -R -n -E \
  'helm (template|install|upgrade)|--values([=[:space:]]|$)|(^|[[:space:]])-f[[:space:]]|valueFiles:' \
  --include='*.yaml' \
  --include='*.yml' \
  --include='*.sh' \
  --include='Jenkinsfile' \
  .github pipelines scripts deploy 2>/dev/null
```

Compare defaults with one environment overlay:

```bash
diff -u "$CHART/values.yaml" "$VALUES_FILE"
```

Render with the same ordered values files and `--set` arguments used by the delivery pipeline:

```bash
helm template "$RELEASE" "$CHART" \
  -f "$CHART/values.yaml" \
  -f "$VALUES_FILE"
```

Helm applies values from left to right: later files and command-line overrides take precedence. Do not print decrypted secret values or commit rendered Secrets.

Helm application charts deploy resources; library charts provide reusable template helpers. Dependencies let charts compose other charts.

### Illustrative chart structure

Follow your codebase's conventions rather than imposing this layout:

```text
charts/hello/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
    configmap.yaml
```

`Chart.yaml` is required. The remaining files depend on what the chart does. The `deployment.yaml` above is a Helm template, not a separate plain-manifest deployment workflow.

**Completion check:** explain which chart settings control how the application runs and how it is reached.

## Lab 2.2: Add the hello-world application

**Objective:** deploy a simple application using the same conventions as your codebase.

### Starter application

This dependency-free Python application provides:

- Configurable message.
- Hostname and application version.
- Health endpoints.
- JSON request logs and request IDs.
- Optional backend call.
- Bounded CPU-work endpoint for HPA testing.
- Deliberate startup and readiness failure controls.

### `app.py`

```python
import hashlib
import json
import os
import socket
import time
import uuid
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.request import Request, urlopen

if os.getenv("FAIL_STARTUP") == "true":
    raise RuntimeError("Deliberate learning-lab startup failure")


class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        started = time.monotonic()
        request_id = self.headers.get("X-Request-ID") or str(uuid.uuid4())
        path = self.path.split("?")[0]
        status = 200

        body = {
            "message": os.getenv("MESSAGE", "Hello from Kubernetes"),
            "version": os.getenv("APP_VERSION", "v1"),
            "hostname": socket.gethostname(),
            "request_id": request_id,
        }

        try:
            if path == "/readyz":
                if os.getenv("FAIL_READINESS") == "true":
                    status = 503
                body = {"ready": status == 200}

            elif path == "/healthz":
                body = {"healthy": True}

            elif path == "/work":
                hashlib.pbkdf2_hmac(
                    "sha256", b"learning", b"lab", 200000
                )
                body["work_complete"] = True

            elif path == "/" and os.getenv("UPSTREAM_URL"):
                request = Request(
                    os.environ["UPSTREAM_URL"],
                    headers={"X-Request-ID": request_id},
                )
                with urlopen(request, timeout=3) as response:
                    body["backend"] = json.load(response)

            elif path != "/":
                status = 404
                body = {"error": "not found"}

        except Exception as error:
            status = 502
            body = {"error": str(error), "request_id": request_id}

        payload = json.dumps(body).encode()
        self.send_response(status)
        self.send_header("Content-Type", "application/json")
        self.send_header("X-Request-ID", request_id)
        self.send_header("Content-Length", str(len(payload)))
        self.end_headers()
        self.wfile.write(payload)

        print(json.dumps({
            "request_id": request_id,
            "path": path,
            "status": status,
            "duration_ms": round(
                (time.monotonic() - started) * 1000, 2
            ),
        }), flush=True)

    def log_message(self, *args):
        pass


ThreadingHTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
```

### `Dockerfile`

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
USER 10001:10001
EXPOSE 8080
CMD ["python", "-u", "app.py"]
```

For repeatable builds, pin your approved base image by digest.

### Exercises

1. Run the application locally.
2. Build and run its container.
3. Test `/`, `/healthz`, `/readyz` and `/work`.
4. Publish the image to ECR or your lab registry.
5. Confirm image architecture matches worker nodes.
6. Point the existing `charts/hello` chart at your image: set `image.repository` and `image.tag`, and set `args` to `[]`, since that flag is only for the starter image. Or use your shared application chart if that is your established pattern.
7. Configure the message, Service port, probes and resource requests/limits. This app's health endpoint is `/healthz`, so set `probes.path` to that; it also has `/readyz` for readiness.
8. Keep deployment in a dedicated learning namespace.

**Completion check:** the container returns its message, produces structured logs and uses the same chart conventions as your day-to-day work.

**Boundary:** this is a learning application, not a production server. Keep `/work` private and use bounded load tests. The request IDs provide correlation, not distributed tracing.

## Lab 2.3: Inspect, deploy and test

**Objective:** understand Kubernetes resources without maintaining a second set of manifests.

### Exercises

1. Render the chart locally.
2. Inspect the generated Deployment and Service.
3. Deploy through Helm.
4. Inspect the live Deployment, ReplicaSet, Pods and Service.
5. Port-forward and test the application.
6. Test Service DNS from another lab Pod.
7. Trace the deployment lifecycle.

Adapt paths and release names to your codebase:

```bash
helm lint ./charts/hello

helm template hello ./charts/hello \
  -f environments/dev.yaml

helm upgrade --install hello ./charts/hello \
  -n k8slearning \
  -f environments/dev.yaml

helm get values hello -n k8slearning
helm get manifest hello -n k8slearning
helm history hello -n k8slearning
```

### What happens during deployment?

1. Helm renders templates using supplied values and submits resources to the API server.
2. The API server processes and persists accepted state.
3. Deployment and ReplicaSet controllers reconcile the requested workload.
4. The scheduler selects nodes for unscheduled Pods.
5. Kubelets and container runtimes start containers.
6. Readiness and endpoint information determine which Pods receive Service traffic.

These are asynchronous reconciliation loops, not one synchronous script. Controllers and the scheduler interact through the API server rather than directly updating etcd.

**Completion check:** explain resource accepted versus Pod running versus application ready. Trace a chart value into a live resource.

## Lab 2.4: Change and recover

**Objective:** understand updates and reconciliation through Helm.

### Exercises

1. Change the message through lab values.
2. Change the application image version.
3. Change replica count.
4. Observe which changes trigger Pod replacement.
5. Delete a lab Pod and observe its replacement.
6. Introduce a bad image version through lab values.
7. Diagnose the failure and recover through Helm.
8. Inspect release history and practise recovering an earlier revision.
9. Explain chart version versus application image version.

**Completion check:** distinguish a Helm upgrade, a Deployment rollout and Kubernetes replacing a failed Pod.

---

# 3. Debugging and operations

## Lab 3.1: Inspect Pods and perform controlled restarts

**Objective:** gather evidence before changing anything.

```bash
kubectl get pods -n k8slearning -o wide
kubectl describe pod <pod> -n k8slearning

kubectl logs <pod> -n k8slearning -c <container>
kubectl logs <pod> -n k8slearning -c <container> \
  --follow --tail=100

kubectl logs <pod> -n k8slearning -c <container> \
  --previous

kubectl exec -it <pod> -n k8slearning \
  -c <container> -- sh

kubectl get events -n k8slearning \
  --sort-by=.metadata.creationTimestamp

kubectl rollout restart deployment/hello -n k8slearning
kubectl rollout status deployment/hello -n k8slearning
```

### Exercises

1. Inspect environment variables and DNS configuration without exposing secrets.
2. Compare application logs with events.
3. Replace one Pod, then perform a rolling restart.
4. Debug an image without a shell using an approved ephemeral debug container.
5. Explain why changing files inside a running container is not a durable fix.
6. Explain why a Service is not an application process to restart.

**Completion check:** choose logs, events, exec or debug containers based on the symptom.

## Lab 3.2: Break and fix common failures

**Objective:** diagnose failures without immediately restarting everything.

Apply one fault at a time through a separate lab values file. Restore baseline after each exercise.

| Fault | Expected symptom | Investigation and recovery |
|---|---|---|
| Non-existent image tag | Image pull failure | Inspect events; correct image reference. |
| `FAIL_STARTUP=true` | Repeated crashes | Read current/previous logs; remove setting. |
| `FAIL_READINESS=true` | Running but unready | Inspect probes and endpoints; restore readiness. |
| Incorrect Service selector | No usable Service backends | Compare labels and EndpointSlices. |
| Incorrect `targetPort` | Connection failure | Compare Service mapping with listener. |
| Missing referenced ConfigMap | Container configuration failure | Inspect events and referenced names. |
| Impossible CPU request | Pending Pod | Inspect scheduling events and capacity. |
| Wrong backend DNS name | Frontend returns 502 | Compare DNS, Service and application logs. |

**Optional memory lab:** use a dedicated, bounded memory-allocation test container to trigger an OOM kill. Do not consume arbitrary node memory.

**Completion check:** produce short RCAs for three different failures, including evidence and verification.

---

# 4. Consul service mesh and mTLS

## Lab 4.1: Inspect the existing Consul deployment

**Objective:** 
- understand the installed configuration before onboarding workloads.
- Understand Service Mesh components and how they work (Envoy)
- What other Service Mesh available?
- How to decide which to use?

Record:

- Consul server, Kubernetes integration and dataplane versions.
- Helm release and values.
- Injector configuration and namespace selection.
- ACL and workload identity requirements.
- Transparent proxy versus explicit upstream configuration.
- Existing intentions and security defaults.

### Exercises

1. Find the Consul UI Service.
2. Access through an approved private route or local port-forward.
3. Authenticate with an appropriately scoped identity.
4. Inspect existing services and health.
5. Identify injector and controller logs.

Do not reinstall Consul or change shared mesh defaults.

**Completion check:** distinguish Consul servers, workload sidecars, injection and service registration.

## Lab 4.2: Onboard frontend and backend

**Objective:** secure a real service-to-service call.

### Exercises

1. Deploy the same image as `frontend` and `backend` through Helm.
2. Configure frontend `UPSTREAM_URL` to call the backend Service.
3. Verify the call before enabling the mesh.
4. Update the chart with required Pod-template annotations and ServiceAccounts.
5. Onboard both services using configuration matching the installed Consul version.
6. Inspect injected containers and registered identities.
7. Add a narrowly scoped frontend-to-backend allow intention.
8. Verify requests and inspect proxy certificate information.

Consul uses proxies for mTLS and intentions for service authorisation. A successful request alone is not proof of encrypted transport.

**Completion check:** demonstrate service registration, working traffic and evidence of proxy-to-proxy mTLS.

## Lab 4.3: Diagnose mesh failures

| Lab fault | Learning objective |
|---|---|
| Deny frontend-to-backend intention | Authorisation failure versus TLS failure. |
| Incorrect upstream service name | Discovery/routing failure. |
| Remove injection from a new lab workload | Mesh membership and mixed-traffic behaviour. |
| Incorrect lab workload identity | Registration/authentication failure. |
| Block only a lab workload's required mesh connectivity | Control-plane reachability versus application reachability. |

For actual certificate failures, use a separate disposable mesh to experiment with expired certificates or trust mismatch. Do not rotate the shared CA or change node clocks.

Consul troubleshooting checks can inspect proxy configuration and certificate-related problems. Check intentions separately rather than assuming proxy validation covers them.

**Completion check:** distinguish network, discovery, authorisation and certificate failures using evidence.

---

# 5. HPA and Pod scaling

## Lab 5.1: Enable resource metrics and configure HPA

**Objective:** understand the scaling feedback loop.

### Exercises

1. Check whether Metrics Server is already installed.
2. Verify metrics availability.
3. Add CPU requests to the application through Helm values.
4. Add an `autoscaling/v2` HPA template using your chart conventions.
5. Start with a suggested lab range of two to four replicas.
6. Target CPU utilisation, for example 60%.
7. Inspect HPA conditions and events.

```bash
kubectl top nodes
kubectl top pods -n k8slearning
kubectl get hpa -n k8slearning
kubectl describe hpa hello -n k8slearning
```

CPU utilisation targets are calculated relative to CPU requests. Resource-based HPA commonly obtains metrics through Metrics Server; this is not historical monitoring.

**Consul consideration:** Pod-level CPU measurements can include sidecars. Ensure relevant requests exist, or investigate a supported container-specific metric for the application container.

**Completion check:** explain why changing a CPU request can change scaling behaviour even when actual CPU consumption stays similar.

## Lab 5.2: Generate load and diagnose scaling

### Exercises

1. Run a bounded load generator against `/work`.
2. Watch CPU, desired replicas, actual replicas and latency.
3. Stop load and observe scale-down.
4. Remove CPU requests from a non-meshed lab copy and inspect the result.
5. Set an unattainable scheduling request and observe Pending replicas.
6. Restore baseline.

**Completion check:** distinguish unavailable metrics, maximum replicas reached, insufficient node capacity and insufficient load.

---

# 6. Argo CD and GitOps

## Lab 6.1: Deploy from Git

**Objective:** separate image creation from deployment reconciliation.

**Flow:** code -> CI test/build -> registry -> deployment configuration commit -> Argo CD -> cluster.

### Exercises

1. Install or reuse an approved Argo CD instance.
2. Configure repository access and a namespace-scoped application/project.
3. Deploy the chart into a fresh GitOps learning namespace.
4. Start with manual sync.
5. Inspect desired versus live manifests.
6. Update the image version through Git.
7. Confirm rollout and application version.

Argo CD renders Helm charts but owns their application lifecycle. It does not create a conventional Helm-managed release for the application.

After this transition, do not also use manual Helm upgrades to manage the same application resources.

**Completion check:** release a version without manually applying application manifests.

## Lab 6.2: Drift, recovery and ownership

### Exercises

1. Change a live lab setting and observe drift.
2. Enable self-healing and repeat.
3. Remove a disposable resource from Git and test pruning.
4. Commit a bad image version and recover by reverting Git.
5. Break a chart value and inspect rendering/sync errors.
6. Run HPA while GitOps is enabled.

**Ownership rule:** when HPA is enabled, design the chart so GitOps does not continuously force a fixed replica count. Investigate narrowly scoped diff customisation only where necessary, rather than ignoring broad resource differences.

Automatic sync, self-healing and pruning are distinct settings. Practise Git-based recovery; Argo CD direct rollback is unavailable while automated sync is enabled.

**Completion check:** explain which system owns infrastructure, application manifests, injected configuration and replica count.

---

# 7. Observability and RCA

## Lab 7.1: Follow one request through the system

**Objective:** correlate application behaviour with platform evidence.

### Exercises

1. Centralise application and proxy logs using existing tooling.
2. Search frontend and backend logs by request ID.
3. Build a dashboard for:
   - Request rate, errors and latency.
   - CPU/memory.
   - Restarts and readiness.
   - Desired versus available replicas.
4. Add OpenTelemetry instrumentation to a later application version.
5. Follow a frontend-to-backend trace.
6. Compare traces, logs, metrics, events and Git changes during a failure.
7. Identify relevant EKS control-plane logs and access requirements.

The starter app provides request-ID correlation, not distributed tracing. Instrumentation and a trace backend are a separate exercise.

Use existing organisational tooling where possible. Options discussed include CloudWatch, Prometheus/Grafana and OpenTelemetry integrations.

**Completion check:** explain where an error originated and what changed, not just that an error occurred.

### RCA template

- Impact.
- Timeline.
- Evidence.
- Root cause.
- Contributing factors.
- Recovery.
- Prevention.
- Verification.

---

# 8. Additional platform topics

## Lab 8.1: NetworkPolicy

**Objective:** restrict connectivity without blocking required dependencies.

### Exercises

1. Verify the network implementation actually enforces NetworkPolicy.
2. Test baseline connectivity.
3. Apply default-deny ingress to the lab backend.
4. Allow only the frontend.
5. Add controlled egress restrictions.
6. Permit the actual DNS path and required mesh dependencies.
7. Test from authorised and unauthorised Pods.
8. Introduce a selector error and diagnose it.

**Pitfalls:** unsupported enforcement, incorrect labels, blocked DNS, missing source egress/destination ingress permissions and broad overlapping allow rules. NetworkPolicies are additive. Creating an object alone does not prove enforcement.

**Completion check:** demonstrate a successful allowed connection and failed prohibited connection.

## Lab 8.2: Security policies and least privilege

**Objective:** distinguish workload admission, Kubernetes permissions and AWS permissions.

### Exercises

1. Apply Pod Security Admission warnings to a dedicated namespace.
2. Inspect violations.
3. Harden the application through Helm:
   - Non-root execution.
   - No privilege escalation.
   - Dropped capabilities.
   - Appropriate seccomp profile.
   - Read-only root filesystem where compatible.
4. Enforce a suitable policy after testing.
5. Create a read-only namespace Role and RoleBinding.
6. Demonstrate an allowed read and denied write.
7. Give a workload narrowly scoped AWS access using your chosen EKS identity mechanism.

Pod Security Admission supports `warn`, `audit` and `enforce`, with predefined security levels. Use policy versions matching the cluster.

**Mesh pitfall:** inspect injected containers too. The application may comply while an injected component requires different permissions.

**Completion check:** identify whether a failure comes from admission, Kubernetes RBAC or AWS IAM.

## Lab 8.3: Node scaling and maintenance

**Objective:** understand what happens when more Pods need more infrastructure.

### Exercises

1. Increase demand until additional capacity is required.
2. Observe scheduling events and node provisioning.
3. Compare mechanisms relevant to your environment:
   - Cluster Autoscaler.
   - Karpenter.
   - EKS Auto Mode.
4. Add topology spreading and a PodDisruptionBudget through Helm.
5. On a dedicated sandbox node, practise cordon, drain and uncordon.
6. Observe application availability and rescheduling.

Choose one node-provisioning mechanism for the hands-on lab rather than enabling competing provisioners without a design.

**Completion check:** explain why HPA alone cannot guarantee additional node capacity.

## Lab 8.4: Multi-region/global architecture

**Objective:** understand global deployment as an architecture problem, not another HPA setting.

Use separate regional clusters rather than stretching a single Kubernetes cluster across AWS Regions.

| Pattern | Learning focus |
|---|---|
| Multi-AZ, single Region | Availability within a Region, topology and capacity. |
| Active/passive Regions | Recovery time, standby capacity and failover. |
| Active/active Regions | Traffic distribution, data consistency and operational complexity. |
| Regional cells | Isolation, tenant placement and independent failure domains. |

### Exercises

1. Define availability, RTO and RPO requirements.
2. Deploy the same application configuration to two clusters.
3. Use distinct regional settings and identities.
4. Design global traffic routing and health checks.
5. Simulate application unavailability in one regional lab.
6. Measure failover and remaining capacity.
7. Document data replication, secrets, image availability and dependency assumptions.

### Questions to answer

- Can the surviving Region handle displaced traffic?
- Is the application stateless?
- Where does authoritative data live?
- What happens during replication lag or a network partition?
- Can both Regions deploy and recover independently?

**Completion check:** produce an architecture decision record comparing active/passive and active/active.

---

# 9. Further labs

| Topic | Exercise |
|---|---|
| Persistent storage | Mount a PVC, replace a Pod and test persistence and topology constraints. |
| External traffic and TLS | Expose the application through the approved ingress/gateway implementation. |
| Secrets management | Retrieve a test secret without committing it to Git. |
| ResourceQuota and LimitRange | Trigger and diagnose a namespace resource rejection. |
| Upgrade readiness | Identify deprecated APIs and test chart compatibility in a sandbox. |
| Backup and recovery | Rebuild the application from Git and separately restore stateful data. |

---

# 10. Final capstone

Deploy the application with **Helm + Consul + HPA + Argo CD + telemetry**, then have someone select an undisclosed fault from the fault catalogue.

## Deliverables

1. Diagnose the failure using evidence.
2. Recover through the correct owning system.
3. Verify service health and security controls.
4. Write a one-page RCA.
5. Rebuild the application in a clean namespace.

## Repository checklist

- [ ] Application source and container build definition.
- [ ] Helm chart or shared-chart configuration.
- [ ] Environment-specific values.
- [ ] Lab-specific Consul configuration.
- [ ] HPA configuration.
- [ ] Argo CD configuration.
- [ ] Network and security policy configuration.
- [ ] Architecture sketch.
- [ ] Troubleshooting runbook.
- [ ] Example RCAs.
- [ ] Cleanup instructions.

## Consolidation notes

The standalone plain-manifest deployment lab has been removed. Kubernetes resource inspection remains part of the Helm workflow. The hello-world code and deliberate fault scenarios are included, and the later labs consistently use chart configuration and Git ownership rather than introducing a competing deployment process.
