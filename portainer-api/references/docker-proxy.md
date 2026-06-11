# Docker via the proxy

`/api/endpoints/{id}/docker/<path>` forwards `<path>` to the environment's Docker daemon.
Requests and responses are **exactly the Docker Engine API** (docs.docker.com/engine/api) —
your Docker API knowledge applies verbatim, with Portainer's auth header on top and
Portainer RBAC filtering what you may touch. This surface is intentionally absent from the
Swagger spec, so don't look for it there.

A version prefix is optional (`/docker/containers/json` and `/docker/v1.41/containers/json`
both work); omit it unless you need to pin behavior.

## Read patterns

```bash
P="$PORTAINER_URL/api/endpoints/$ENV_ID/docker"
H="X-API-Key: $PORTAINER_API_TOKEN"

# All containers, projected — never dump full inspect output
curl -fsS -H "$H" "$P/containers/json?all=true" \
  | jq '.[] | {id: .Id[:12], name: .Names[0], state: .State, image: .Image, status: .Status}'

# Server-side filtering: `filters` is URL-encoded JSON (--data-urlencode + -G does the encoding)
curl -fsS -G -H "$H" "$P/containers/json" \
  --data-urlencode 'filters={"status":["exited"]}'

# Compose-project grouping lives in labels:
curl -fsS -G -H "$H" "$P/containers/json" \
  --data-urlencode 'filters={"label":["com.docker.compose.project=myproj"]}'

# Single container inspect / processes
curl -fsS -H "$H" "$P/containers/$CID/json" | jq '{name: .Name, state: .State.Status, restarts: .RestartCount}'
curl -fsS -H "$H" "$P/containers/$CID/top"
```

## Logs and stats — bound them or they bound you

- **Logs** `GET /containers/{id}/logs` — must pass `stdout=true` and/or `stderr=true`
  (Docker 400s without them). Bound the output with `tail=100` and/or `since=<unix-ts>`.
  Never `follow=true` in a script — it streams forever. The body is Docker's multiplexed
  format when the container has no TTY; expect 8-byte frame headers (stripping them:
  pipe through `sed 's/^.\{8\}//'` is usually good enough for reading).
- **Stats** `GET /containers/{id}/stats?stream=false` — `stream=false` is mandatory for
  a one-shot snapshot; the default streams indefinitely.
- **Events** `GET /events` — streams; only use with `since`/`until` bounds.
- **Exec** — creating an exec (`POST /containers/{id}/exec`) works, but attaching returns a
  multiplexed binary stream over an upgraded connection; from curl this is rarely worth it.
  Prefer `/containers/{id}/top`, logs, or running the command another way.

## Mutations through the proxy

Standard Docker API semantics:

```bash
# Pull an image (note: POST, response is a progress stream — discard it)
curl -fsS -X POST -H "$H" "$P/images/create?fromImage=nginx&tag=alpine" -o /dev/null

# Create + start a container
CID=$(curl -fsS -X POST -H "$H" -H "Content-Type: application/json" \
  "$P/containers/create?name=web" \
  -d '{"Image": "nginx:alpine", "HostConfig": {"PortBindings": {"80/tcp": [{"HostPort": "8080"}]}}}' \
  | jq -r .Id)
curl -fsS -X POST -H "$H" "$P/containers/$CID/start"

# Stop / remove
curl -fsS -X POST -H "$H" "$P/containers/$CID/stop"
curl -fsS -X DELETE -H "$H" "$P/containers/$CID?v=true"
```

**Private registries:** add header `X-Registry-Auth: <base64 of {"registryId":N}>` where
`N` is the Portainer registry ID (`GET /api/registries`). Portainer resolves the actual
credentials server-side — you never handle the registry password.

```bash
curl -fsS -X POST -H "$H" \
  -H "X-Registry-Auth: $(printf '{"registryId":%d}' "$REG_ID" | base64 -w0)" \
  "$P/images/create?fromImage=registry.example.com/app&tag=v2" -o /dev/null
```

**Note:** containers created through the raw proxy are "external" from Portainer's point of
view — for anything compose-shaped, prefer a stack (`references/stacks.md`); it stays
manageable in the UI and supports updates/webhooks.

## Swarm

- **Swarm cluster ID** (needed for swarm stack creation): `GET $P/info | jq -r .Swarm.Cluster.ID`
- Services: `GET $P/services`, service logs `GET $P/services/{id}/logs?stdout=true&tail=100`
- Nodes: `GET $P/nodes`

## Typed Docker helpers (in the Swagger spec)

Portainer also has a few aggregated, non-proxied Docker endpoints — cheaper than
assembling the answer from proxy calls:

- `GET /api/docker/{environmentId}/dashboard` — counts of containers/images/volumes/networks
  plus health rollup
- `GET /api/docker/{environmentId}/images` — image list with usage flags
- `POST /api/docker/{environmentId}/snapshot` — force a snapshot refresh
