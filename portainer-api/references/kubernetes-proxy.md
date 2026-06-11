# Kubernetes via Portainer

Two surfaces, pick deliberately:

1. **Raw proxy** — `/api/endpoints/{id}/kubernetes/<K8s API path>`: the full Kubernetes
   API, verbatim. Absent from the Swagger spec (same reason as the Docker proxy). Use for
   anything the typed endpoints don't cover.
2. **Typed endpoints** — `/api/kubernetes/{id}/...`: Portainer-aware views (RBAC-filtered,
   sometimes enriched). In the Swagger spec. Prefer them when one matches the question —
   less output, already scoped.

## Raw proxy

```bash
K="$PORTAINER_URL/api/endpoints/$ENV_ID/kubernetes"
H="X-API-Key: $PORTAINER_API_TOKEN"

# Pods across all namespaces, projected
curl -fsS -H "$H" "$K/api/v1/pods" | jq '
  .items[] | {name: .metadata.name, ns: .metadata.namespace,
              phase: .status.phase, node: .spec.nodeName}'

# Narrow server-side — label and field selectors work as usual
curl -fsS -G -H "$H" "$K/api/v1/pods" \
  --data-urlencode 'labelSelector=app=web' \
  --data-urlencode 'fieldSelector=status.phase!=Running'

# Deployments readiness
curl -fsS -H "$H" "$K/apis/apps/v1/deployments" | jq '
  .items[] | {name: .metadata.name, ns: .metadata.namespace,
              want: .spec.replicas, ready: (.status.readyReplicas // 0)}'

# Pod logs — bound them like Docker logs
curl -fsS -H "$H" "$K/api/v1/namespaces/$NS/pods/$POD/log?tailLines=100"
```

The noise to project out: `metadata.managedFields` (routinely 30–70% of an object) and the
full `status` block when you only need the spec. Mutations are standard K8s API
(`POST`/`PUT`/`PATCH` with the right `Content-Type`; for `PATCH` that's
`application/strategic-merge-patch+json` or `application/merge-patch+json`).

## Typed endpoints (Swagger-documented)

| Question | Endpoint |
|---|---|
| Namespaces visible to me | `GET /api/kubernetes/{id}/namespaces` |
| Applications (Portainer's rollup of deploy/sts/ds/pods) | `GET /api/kubernetes/{id}/applications` |
| Services / Ingresses | `GET /api/kubernetes/{id}/services`, `/ingresses` |
| Nodes & resource usage | `GET /api/kubernetes/{id}/nodes`, `/dashboard` |
| ConfigMaps / Secrets | `GET /api/kubernetes/{id}/configmaps`, `/secrets` |
| Volumes | `GET /api/kubernetes/{id}/volumes` |

Remember the typed K8s paths use `/api/kubernetes/{id}/...` (resource style) while the
proxy is `/api/endpoints/{id}/kubernetes/...` — easy to transpose, and the 404 you get
from the wrong one looks like a missing resource.

`GET .../secrets` returns real secret data given sufficient RBAC — handle accordingly.

## Helm

Helm goes through Portainer's own endpoints (the raw K8s API can't see release metadata):

```bash
# List releases
curl -fsS -H "$H" "$PORTAINER_URL/api/endpoints/$ENV_ID/kubernetes/helm" \
  | jq '.[] | {name: .name, ns: .namespace, status: .status, chart: .chart}'

# Install (JSON body)
curl -fsS -X POST -H "$H" -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/endpoints/$ENV_ID/kubernetes/helm" \
  -d '{"chart": "nginx", "repo": "https://charts.bitnami.com/bitnami",
       "name": "my-nginx", "namespace": "default", "values": "replicaCount: 2"}'

# History / rollback / uninstall
curl -fsS -H "$H" "$PORTAINER_URL/api/endpoints/$ENV_ID/kubernetes/helm/$RELEASE/history"
curl -fsS -X POST -H "$H" "$PORTAINER_URL/api/endpoints/$ENV_ID/kubernetes/helm/$RELEASE/rollback?namespace=$NS"
curl -fsS -X DELETE -H "$H" "$PORTAINER_URL/api/endpoints/$ENV_ID/kubernetes/helm/$RELEASE?namespace=$NS"
```

For exact body fields, grep the spec: `grep -n '^  /endpoints/{id}/kubernetes/helm' references/spec/helm.yaml`.

## Kubeconfig

`GET /api/kubernetes/{id}/config` returns a kubeconfig scoped to your Portainer identity —
sometimes handing the task to `kubectl` against that config is simpler than proxying
everything through curl. Note the issued config routes through Portainer, so the Portainer
instance must stay reachable from wherever kubectl runs.
