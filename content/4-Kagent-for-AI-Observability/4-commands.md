Create Kagent namespace

```bash
kubectl create namespace kagent
```

Create secret with Google Gemini Key

```bash
source ~/.bashrc
export GOOGLE_API_KEY=$GEMINI_API_KEY
export GRAFANA_API_KEY=$GRAFANA_API_KEY
kubectl create secret generic kagent-gemini \
  -n kagent \
  --from-literal GOOGLE_API_KEY=${GOOGLE_API_KEY}
```

Create the Kagent resource manifest for the Google Gemini Model

```bash
cat > gemini-model.yaml <<'EOF'
apiVersion: kagent.dev/v1alpha2
kind: ModelConfig
metadata:
  name: gemini-model-config
  namespace: kagent
spec:
  apiKeySecret: kagent-gemini
  apiKeySecretKey: GOOGLE_API_KEY
  model: gemini-2.5-flash
  provider: Gemini
  gemini: {}
EOF
[ -f gemini-model.yaml ] && echo "gemini-model.yaml created"
```

Install CRDs for Kagent

```bash
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
    --namespace kagent \
    --create-namespace
```

Create Kagent Resource object for the Google Gemini model

```bash
kubectl apply -f gemini-model.yaml
```

Install kagent

```bash
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent --namespace kagent  \
  --set providers.default=gemini \
  --set providers.gemini.provider=Gemini \
  --set providers.gemini.apiKeySecretRef=kagent-gemini \
  --set providers.gemini.apiKeySecretKey=GOOGLE_API_KEY \
  --set providers.gemini.model=gemini-2.5-flash \
  --set agents.k8s-agent.enabled=true \
  --set agents.argo-rollouts-agent.enabled=false \
  --set agents.cilium-debug-agent.enabled=false \
  --set agents.cilium-manager-agent.enabled=false \
  --set agents.cilium-policy-agent.enabled=false \
  --set agents.helm-agent.enabled=false \
  --set agents.istio-agent.enabled=false \
  --set agents.kgateway-agent.enabled=true \
  --set agents.observability-agent.enabled=true \
  --set agents.promql-agent.enabled=true \
  --set tools.grafana-mcp.enabled=true \
  --set grafana-mcp.grafana.url=http://grafana.meta.svc.cluster.local \
  --set grafana-mcp.grafana.serviceAccountToken=${GRAFANA_API_KEY} \
  --set tools.querydoc.enabled=true \
  --set kagent-tools.enabled=true \
 --set querydoc.resources.requests.cpu="20m" \
 --set kagent-tools.resources.requests.cpu="20m" \
 --set grafana-mcp.resources.requests.cpu="20m" \
 --set controller.resources.requests.cpu="20m" \
 --set agents.promql-agent.resources.requests.cpu="20m" \
 --set agents.observability-agent.resources.requests.cpu="20m" \
 --set agents.kgateway-agent.resources.requests.cpu="20m" \
 --set agents.k8s-agent.resources.requests.cpu="20m" \
 --set querydoc.resources.requests.memory="25Mi" \
 --set kagent-tools.resources.requests.memory="25Mi" \
 --set grafana-mcp.resources.requests.memory="25Mi" \
 --set controller.resources.requests.memory="25Mi" \
 --set agents.promql-agent.resources.requests.memory="25Mi" \
 --set agents.observability-agent.resources.requests.memory="25Mi" \
 --set agents.kgateway-agent.resources.requests.memory="25Mi" \
 --set agents.k8s-agent.resources.requests.memory="25Mi"

```

```bash
kubectl set env deployment/kagent-grafana-mcp -n kagent \
  GRAFANA_URL=http://grafana.meta.svc.cluster.local:80 \
  GRAFANA_API_KEY=${GRAFANA_API_KEY}
```

Verification

```bash
kubectl get pods -n kagent
kubectl get modelconfigs -n kagent
kubectl describe modelconfig gemini-model-config -n kagent
```

Create the Grafana Agent

```bash
cat > grafana-agent.yaml <<'EOF'
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: grafana-agent
  namespace: kagent
spec:
  declarative:
    modelConfig: gemini-model-config
    systemMessage: |
      You are a Grafana-aware agent for cluster investigation.
      You can search dashboards, query Prometheus metrics, and read Loki logs
      from the Grafana instance. Be concise and cite the dashboards or queries you used.
    tools:
      - type: McpServer
        mcpServer:
          apiGroup: kagent.dev
          kind: RemoteMCPServer
          name: grafana-mcp
          toolNames:
            - add_activity_to_incident
            - alerting_manage_routing
            - alerting_manage_rules
            - create_annotation
            - create_folder
            - create_incident
            - find_error_pattern_logs
            - find_slow_requests
            - generate_deeplink
            - get_alert_group
            - get_annotation_tags
            - get_annotations
            - get_assertions
            - get_current_oncall_users
            - get_dashboard_by_uid
            - get_dashboard_panel_queries
            - get_dashboard_property
            - get_dashboard_summary
            - get_datasource
            - get_incident
            - get_oncall_shift
            - get_panel_image
            - get_sift_analysis
            - get_sift_investigation
            - list_alert_groups
            - list_datasources
            - list_incidents
            - list_loki_label_names
            - list_loki_label_values
            - list_oncall_schedules
            - list_oncall_teams
            - list_oncall_users
            - list_prometheus_label_names
            - list_prometheus_label_values
            - list_prometheus_metric_metadata
            - list_prometheus_metric_names
            - list_pyroscope_label_names
            - list_pyroscope_label_values
            - list_pyroscope_profile_types
            - list_sift_investigations
            - query_loki_logs
            - query_loki_patterns
            - query_loki_stats
            - query_prometheus
            - query_prometheus_histogram
            - query_pyroscope
            - search_dashboards
            - search_folders
            - update_annotation
EOF
kubectl apply -f grafana-agent.yaml
```

Create the RemoteMCPServer CR for Grafana MCP

NOTE (kagent 0.9.1 bug): The helm chart deploys the grafana-mcp pod/service but does NOT create
the RemoteMCPServer CR. Create it BEFORE applying grafana-agent.yaml and wait for ACCEPTED=True
so tools are discovered. If toolNames are omitted from the Agent spec, the controller writes
"tools":null and the agent pod crashes with a Pydantic ValidationError.

Also note: the grafana-mcp endpoint is /mcp (STREAMABLE_HTTP), not /sse.

```bash
kubectl apply -f - <<'EOF'
apiVersion: kagent.dev/v1alpha2
kind: RemoteMCPServer
metadata:
  name: grafana-mcp
  namespace: kagent
spec:
  description: "Grafana MCP server for dashboard search, Prometheus queries, and Loki logs"
  url: "http://kagent-grafana-mcp.kagent.svc.cluster.local:8000/mcp"
  protocol: STREAMABLE_HTTP
EOF

# Wait for tools to be discovered before applying the agent
echo "Waiting for RemoteMCPServer to discover tools..."
kubectl wait --for=condition=Accepted remotemcpserver/grafana-mcp -n kagent --timeout=60s
```

Verify grafana-agent

```bash
kubectl get agent grafana-agent -n kagent
kubectl get remotemcpserver grafana-mcp -n kagent
```

If we have everything, we can now forward kagent UI to our port 8082

Forward Kagent UI

```bash
nohup  kubectl port-forward --address 0.0.0.0  -n kagent svc/kagent-ui 8082:8080 > /dev/null 2>&1 &
```


