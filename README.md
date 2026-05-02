# K8s Boston Training Day — Demo App

A hands-on Kubernetes training environment that walks through cluster setup, AI-powered diagnostics with K8sGPT, classic observability with Grafana/Loki, and AI-driven observability with Kagent.

---

## Prerequisites — Complete the Exercises First

Before using this README to bring services online, you should have already worked through the numbered exercises under `content/` in order:

| # | Folder | What you do |
|---|--------|-------------|
| 0 | `0-Gemini-api-key/` | Set your Gemini API key in `~/.bashrc` |
| 1 | `1-setup-cluster-and-demo-app/` | Install kind, kubectl, helm; create the cluster; deploy the demo e-commerce app |
| 2 | `2-k8gpt/` | Install K8sGPT, deploy the broken app, and use AI to diagnose the CrashLoopBackOff |
| 3 | `3-Grafana-for-classic-Observability/` | Install Loki, Grafana, and the k8s-monitoring stack; verify logs are flowing |
| 4 | `4-Kagent-for-AI-Observability/` | Install Kagent with Gemini, deploy the Grafana MCP agent |

---

## Verifying Prerequisites

Run these checks to confirm all pieces are in place before starting a demo session:

```bash
# Cluster is up
kubectl get nodes

# All expected namespaces exist
kubectl get namespaces | grep -E "meta|prod|demo-app|kagent|broken-app"

# Demo e-commerce app is running
kubectl get pods -n demo-app

# Grafana + Loki stack is healthy
kubectl get pods -n meta

# Kagent is running
kubectl get pods -n kagent
kubectl get modelconfigs -n kagent

# Gemini API key is set
echo $GEMINI_API_KEY
```

To verify Loki is ingesting logs from the cluster:

```bash
kubectl run curl-test -n meta --image=curlimages/curl --rm -it --restart=Never -- \
  curl -s http://loki-gateway.meta.svc.cluster.local/loki/api/v1/labels
# Should return a JSON list of labels including namespace, container, etc.
```

---

## Bringing Services Online

After the cluster is up, run these port-forwards to expose UIs locally. Each runs in the background — keep the terminal session alive.

### Demo App (e-commerce frontend)

```bash
kubectl port-forward --address 0.0.0.0 svc/frontend -n demo-app 8080:8080 &
# Access at http://localhost:8080
```

### Grafana UI

```bash
export POD_NAME=$(kubectl get pods --namespace meta \
  -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" \
  -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace meta --address 0.0.0.0 port-forward $POD_NAME 3000 &
# Access at http://localhost:3000 — credentials: admin / adminadminadmin
```

### Loki Gateway (for direct log queries / curl tests)

```bash
kubectl port-forward --namespace meta --address 0.0.0.0 svc/loki-gateway 3100:80 &
# Access at http://localhost:3100
```

### Kagent UI

```bash
kubectl port-forward --namespace kagent --address 0.0.0.0 svc/kagent-ui 8082:80 &
# Access at http://localhost:8082
```

---

## Taking Services Offline

To cleanly stop all port-forwards:

```bash
pkill -f "kubectl.*port-forward"
```

To tear down the broken-app demo after the K8sGPT exercise:

```bash
kubectl delete -f content/2-k8gpt/broken-app.yaml
```

To tear down the entire cluster:

```bash
kind delete cluster
```

---

## Running in GitHub Codespaces

This repo includes a dev container (`.devcontainer/devcontainer.json`) that pre-pulls all required Docker images on creation — no internet access needed during the exercises. Codespaces is the recommended way to run this training.

**Minimum machine size:** 4 cores / 16 GB RAM (set in `devcontainer.json` — Codespaces will enforce this).

**Ports automatically forwarded:** `8080` (demo app), `30080`, `30081`.

**On attach**, VS Code will automatically open the Gemini API key notebook — start there.

### Tips for Codespaces

- The kind cluster is not automatically created. Run through exercise `1` first.
- Port-forwards started with `--address 0.0.0.0` work correctly in Codespaces — the forwarded ports will appear in the **Ports** tab in VS Code.
- Your `GEMINI_API_KEY` and `GRAFANA_API_KEY` should be set as **Codespaces secrets** in your GitHub account so they are available as environment variables automatically (Settings → Codespaces → Secrets). The `~/.bashrc` export step in exercise 0 still works as a fallback.
- If the session is rebuilt or times out, re-run the port-forward commands above — the cluster and deployed resources will still be intact.

---

## Cluster Config

The kind cluster is created from `misc/kind-config.yaml` (1 control-plane node + 1 worker). To recreate it:

```bash
kind create cluster --config misc/kind-config.yaml
```
