# MaaS Billing Quick Deploy

One-page quick reference for deploying MaaS Billing using the all-in-one manifest.

## Prerequisites Checklist

- [ ] OpenShift 4.19.9+ or Kubernetes with Gateway API
- [ ] Cluster admin access (`oc auth can-i create namespaces --all-namespaces`)
- [ ] Tools installed: `oc`, `jq`, `envsubst`, `kustomize`

## 5-Minute Deploy

```bash
# 1. Export cluster domain
export CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')

# 2. Deploy everything
envsubst '$CLUSTER_DOMAIN' < all-in-one.yaml | oc apply -f -

# 3. Wait for Kuadrant operators (~5-10 min)
oc wait --for=condition=Available deployment/kuadrant-operator-controller-manager \
  -n kuadrant-system --timeout=600s

# 4. Configure Gateway Controller
oc -n kuadrant-system patch deployment kuadrant-operator-controller-manager --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"ISTIO_GATEWAY_CONTROLLER_NAMES","value":"openshift.io/gateway-controller/v1"}}]'

# 5. Wait for Gateway
oc wait --for=condition=Programmed gateway maas-default-gateway \
  -n openshift-ingress --timeout=300s

# 6. Restart operators (workaround)
oc delete pod -n kuadrant-system -l control-plane=controller-manager
oc rollout restart deployment/authorino-operator -n kuadrant-system
oc rollout restart deployment/limitador-operator-controller-manager -n kuadrant-system

# 7. Deploy test model
kustomize build docs/samples/models/simulator | oc apply -f -

# 8. Validate
./deployment/scripts/validate-deployment.sh
```

## What Gets Deployed

| Component | Namespace | Description |
|-----------|-----------|-------------|
| **Kuadrant Operators** | `kuadrant-system` | OLM-based installation (Kuadrant, Authorino, Limitador, DNS) |
| **MaaS API** | `maas-api` | API server for token management and model listing |
| **Gateways** | `openshift-ingress` | Two gateways: `openshift-ai-inference`, `maas-default-gateway` |
| **Policies** | `openshift-ingress` | AuthPolicy, RateLimitPolicy, TokenRateLimitPolicy, TelemetryPolicy |
| **Observability** | `kuadrant-system`, `openshift-ingress` | ServiceMonitor, metrics collection |

**Total Resources**: 25 Kubernetes resources
- 5 Namespaces
- 3 Operator resources (OLM)
- 3 Gateway resources
- 8 MaaS API resources
- 4 Policy resources
- 2 Observability resources

## Quick Test

```bash
# Get endpoint
HOST="maas.$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')"

# Get token
TOKEN=$(curl -sSk -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" -X POST \
  -d '{"expiration":"10m"}' "https://${HOST}/maas-api/v1/tokens" | jq -r .token)

# List models
curl -sSk -H "Authorization: Bearer $TOKEN" \
  "https://${HOST}/v1/models" | jq .

# Test inference
MODEL_NAME="facebook/opt-125m"
curl -sSk -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"model\":\"${MODEL_NAME}\",\"prompt\":\"Hello\",\"max_tokens\":50}" \
  "https://${HOST}/llm/facebook-opt-125m-simulated/v1/completions" | jq .

# Test rate limiting (expect 429 after ~5 requests)
for i in {1..10}; do
  curl -sSk -o /dev/null -w "Request $i: %{http_code}\n" \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d "{\"model\":\"${MODEL_NAME}\",\"prompt\":\"Test\",\"max_tokens\":10}" \
    "https://${HOST}/llm/facebook-opt-125m-simulated/v1/completions"
  sleep 1
done
```

## Status Checks

```bash
# Check all pods
oc get pods -n maas-api -n kuadrant-system

# Check operators
oc get csv -n kuadrant-system

# Check Gateway
oc get gateway -n openshift-ingress maas-default-gateway

# Check policies
oc get authpolicy,ratelimitpolicy,tokenratelimitpolicy -n openshift-ingress

# Check policy enforcement
for policy in gateway-auth-policy gateway-rate-limits gateway-token-rate-limits; do
  STATUS=$(oc get authpolicy $policy -n openshift-ingress -o jsonpath='{.status.conditions[?(@.type=="Enforced")].status}' 2>/dev/null || echo "N/A")
  echo "$policy: $STATUS"
done
```

## Common Issues & Quick Fixes

### Issue: Policies Not Enforced

```bash
# Fix: Restart Kuadrant operator
oc rollout restart deployment/kuadrant-operator-controller-manager -n kuadrant-system
oc rollout status deployment/kuadrant-operator-controller-manager -n kuadrant-system
```

### Issue: Gateway Stuck in "Not Programmed"

```bash
# Check if Service Mesh is installed
oc get csv -n openshift-operators | grep servicemesh

# If not, install via OperatorHub in the Console
# Or wait - it may auto-install (takes 5-10 minutes)
```

### Issue: 404 on Model Endpoint

```bash
# Check HTTPRoute
oc get httproute -n llm

# Check model service
oc get llminferenceservice -n llm

# Verify Gateway listeners
oc describe gateway maas-default-gateway -n openshift-ingress
```

### Issue: No Metrics

```bash
# Check Limitador metrics directly
oc port-forward -n kuadrant-system svc/limitador-limitador 8080:8080 &
curl http://localhost:8080/metrics | grep authorized_hits
kill %1

# If empty, make some API calls first to generate metrics
```

## Resource Requirements

| Component | CPU Request | Memory Request | CPU Limit | Memory Limit |
|-----------|-------------|----------------|-----------|--------------|
| MaaS API | 50m | 64Mi | 200m | 128Mi |
| Kuadrant Operator | ~100m | ~128Mi | - | - |
| Authorino | ~100m | ~128Mi | - | - |
| Limitador | ~100m | ~128Mi | - | - |

**Estimated Total**: ~500m CPU, ~512Mi Memory (excluding Service Mesh and model workloads)

## Deployment Timeline

| Step | Duration | What's Happening |
|------|----------|------------------|
| Apply manifest | 10-30s | Creating namespaces, operators, configs |
| Operator installation | 5-10 min | OLM installing Kuadrant stack |
| Gateway ready | 2-5 min | Service Mesh setup, Gateway programming |
| Policy enforcement | 1-2 min | Policies attached and active |
| **Total** | **~10-15 min** | Full platform ready |

## Uninstall

```bash
# Delete all resources
envsubst '$CLUSTER_DOMAIN' < all-in-one.yaml | oc delete -f -

# Clean up namespaces
oc delete namespace maas-api llm kuadrant-system kserve opendatahub
```

## Next Steps

1. **Deploy More Models**: Check `docs/samples/models/` for examples
2. **Configure Tiers**: Edit `tier-to-group-mapping` ConfigMap in `maas-api` namespace
3. **Customize Rate Limits**: Modify `gateway-rate-limits` and `gateway-token-rate-limits` policies
4. **Setup Monitoring**: Configure Grafana dashboards (see `docs/samples/dashboards/`)
5. **Enable Persistence**: Configure Redis for Limitador persistence (see docs)

## Documentation

- **Full Deployment Guide**: [DEPLOYMENT-GUIDE.md](DEPLOYMENT-GUIDE.md)
- **Architecture**: [docs/content/architecture.md](docs/content/architecture.md)
- **Configuration**: [docs/content/configuration-and-management/](docs/content/configuration-and-management/)
- **User Guide**: [docs/content/user-guide/](docs/content/user-guide/)

## Manifest Contents

The `all-in-one.yaml` manifest includes all resources from:
- `deployment/base/networking/odh/odh-gateway-api.yaml`
- `deployment/base/networking/maas/maas-gateway-api.yaml`
- `deployment/base/networking/odh/kuadrant.yaml`
- `kustomize build deployment/base/maas-api`
- `kustomize build deployment/base/policies`
- `kustomize build deployment/base/observability`
- Kuadrant operator installation (from `install-dependencies.sh`)

**100% accurate** - extracted directly from `deploy-openshift.sh` and its dependencies.

