# Stacks

A stack is Portainer's unit of app deployment — a compose file (standalone/swarm) or
manifest (K8s) that Portainer tracks, shows in the UI, and can update/redeploy. Prefer
deploying a stack over raw `docker create` proxy calls for anything multi-container.

## Creation — pick the endpoint from this matrix

`POST /api/stacks/create/{platform}/{source}` — **all of them require `?endpointId=N`**
(the environment to deploy to) as a query parameter:

| | `string` (inline content) | `file` (upload) | `repository` (git) |
|---|---|---|---|
| `standalone` (compose) | JSON body | multipart | JSON body |
| `swarm` | JSON body, **+SwarmID** | multipart, **+SwarmID** | JSON body, **+SwarmID** |
| `kubernetes` | JSON body (manifest or compose) | — (use `url`) | JSON body |

Kubernetes additionally has `POST /api/stacks/create/kubernetes/url` (manifest fetched
from a URL). The response for every variant is the created Stack object — capture `.Id`.

### Standalone compose from inline content (the common case)

```bash
COMPOSE=$(cat docker-compose.yml)
curl -fsS -X POST -H "X-API-Key: $PORTAINER_API_TOKEN" -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/stacks/create/standalone/string?endpointId=$ENV_ID" \
  -d "$(jq -n --arg c "$COMPOSE" '{
        Name: "myapp",
        StackFileContent: $c,
        Env: [{name: "TAG", value: "v2"}]
      }')"
```

Building the body with `jq -n --arg` sidesteps every quoting/newline problem of embedding
YAML in JSON by hand — always do it this way in scripts.

Body fields (`stacks.composeStackFromFileContentPayload`): `Name` (required),
`StackFileContent` (required), `Env` (array of `{name, value}` — note lowercase keys),
`Registries` (array of registry IDs for private images), `Webhook` (a UUID; registering
one lets `POST /api/stacks/webhooks/{uuid}` force a re-pull + update with no auth).

### Swarm: resolve SwarmID first

Swarm variants require the cluster ID — one proxy call gets it:

```bash
SWARM_ID=$(curl -fsS -H "X-API-Key: $PORTAINER_API_TOKEN" \
  "$PORTAINER_URL/api/endpoints/$ENV_ID/docker/info" | jq -r .Swarm.Cluster.ID)
```

Then the same body as standalone plus `"SwarmID": "$SWARM_ID"`, posted to
`/api/stacks/create/swarm/string?endpointId=$ENV_ID`.

### From a git repository

`POST /api/stacks/create/{standalone|swarm}/repository?endpointId=N` — JSON body
(`stacks.composeStackFromGitRepositoryPayload`):

- `Name` (required), `RepositoryURL` (required)
- `RepositoryReferenceName` — e.g. `refs/heads/main` (full ref name, not just the branch)
- `ComposeFile` — path inside the repo, default `docker-compose.yml`; `AdditionalFiles` for multi-file stacks
- Private repos: `RepositoryAuthentication: true` + either `RepositoryUsername`/`RepositoryPassword`
  (a PAT goes in the password field) or a stored credential via `RepositoryGitCredentialID`
- `AutoUpdate` — GitOps polling/webhook config, e.g. `{"Interval": "5m"}` or `{"Webhook": "<uuid>"}`
- `TLSSkipVerify` — for self-hosted git with self-signed certs

```bash
curl -fsS -X POST -H "X-API-Key: $PORTAINER_API_TOKEN" -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/stacks/create/standalone/repository?endpointId=$ENV_ID" \
  -d '{"Name": "myapp",
       "RepositoryURL": "https://github.com/acme/myapp",
       "RepositoryReferenceName": "refs/heads/main",
       "ComposeFile": "deploy/docker-compose.yml"}'
```

### From an uploaded file (multipart)

`file` variants are `multipart/form-data` — and `Env` here is a **JSON array as a string**
form field, not separate fields:

```bash
curl -fsS -X POST -H "X-API-Key: $PORTAINER_API_TOKEN" \
  "$PORTAINER_URL/api/stacks/create/standalone/file?endpointId=$ENV_ID" \
  -F Name=myapp \
  -F file=@docker-compose.yml \
  -F 'Env=[{"name":"TAG","value":"v2"}]'
```

### Kubernetes

`POST /api/stacks/create/kubernetes/string?endpointId=N` — body
(`stacks.kubernetesStringDeploymentPayload`): `StackName`, `Namespace`,
`StackFileContent` (the manifest), `ComposeFormat: true` if the content is a compose file
to convert. Note this payload uses `StackName`, where the Docker variants use `Name` —
copy-pasting between them breaks. Git variant: `kubernetes/repository` with
`ManifestFile` instead of `ComposeFile`.

## Lifecycle

All environment-scoped operations need `?endpointId=N`:

| Operation | Call |
|---|---|
| List / inspect | `GET /api/stacks`, `GET /api/stacks/{id}` (filter list client-side with jq) |
| Get the deployed file | `GET /api/stacks/{id}/file` |
| Update content | `PUT /api/stacks/{id}?endpointId=N` body `{"StackFileContent": "...", "Env": [...], "PullImage": true, "Prune": false}` |
| Start / stop | `POST /api/stacks/{id}/start?endpointId=N`, `.../stop?endpointId=N` |
| Update git config | `PUT /api/stacks/{id}/git?endpointId=N` |
| Redeploy from git | `PUT /api/stacks/{id}/git/redeploy?endpointId=N` (body can pass `PullImage`, `Prune`) |
| Webhook trigger | `POST /api/stacks/webhooks/{webhookUUID}` — no auth header needed; this is the CI hook |
| Move to another env | `POST /api/stacks/{id}/migrate?endpointId=N` body `{"EndpointID": M}` |
| Delete | `DELETE /api/stacks/{id}?endpointId=N` — `&removeVolumes=true` to also drop the stack's volumes (standalone only), `&external=true` only for stacks not created by Portainer |
| Find by name | `GET /api/stacks/name/{name}` |

A stack `Status` of `1` is active, `2` is inactive (stopped).

Stack names must be unique per environment and lowercase (compose project-name rules); a
conflict comes back as an error body whose `details` says the name is taken — pick another
name rather than retrying.

## Edge stacks (fleet deployment)

Deploying to *groups of edge environments* uses a different family —
`POST /api/edge_stacks/create/{string|file|repository}` with `EdgeGroups: [ids]` in the
body instead of `endpointId`, and `DeploymentType` `0` (compose) or `1` (kubernetes).
Manage with `GET/PUT/DELETE /api/edge_stacks/{id}` and check rollout via
`GET /api/edge_stacks/{id}` `.Status` (per-environment deployment state). Edge groups are
managed at `/api/edge_groups`. Mostly EE territory — on CE, expect parts to 404.

For exact field lists, grep the spec:
`grep -n '^  /edge_stacks/create' references/spec/edge_stacks.yaml` and read the named payload
schema (under `components.schemas` in the same file).
