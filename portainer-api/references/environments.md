# Environments (endpoints)

"Environment" and "endpoint" are the same object — the API paths say `endpoints`, newer
param names say `environmentId`. Path params `{id}` and `{environmentId}` are interchangeable.

## Listing — `GET /api/endpoints`

Useful query parameters (all optional):

| Param | Effect |
|---|---|
| `excludeSnapshots=true` | Drops the heavy `Snapshots` array — **use by default** unless you need snapshot counts |
| `search=<text>` | Server-side name search |
| `name=<text>` | Exact-name filter |
| `start`, `limit` | Pagination |
| `sort` + `order` | Sort by `Name`, `Group`, `Status`, `LastCheckIn`, `EdgeID`, `PlatformType`, `Health`, `Id`; order `asc`/`desc` |
| `groupIds=1,2` | Filter by environment group |
| `types=4,7` | Filter by environment type (CSV of ints, see table below) |
| `tagIds=...` | Filter by tag |
| `status=1` | Filter by status value |
| `edgeAsync=true` | Only async edge agents |

Non-admins see only environments their RBAC authorizes — an empty list for a working
token can simply mean "no access", not "no environments".

## Environment types

| Type | Meaning | Health signal |
|---|---|---|
| 1 | Local Docker (socket) | `Status` (1 up / 2 down) |
| 2 | Agent on Docker | `Status` |
| 3 | Azure ACI | `Status` |
| 4 | **Edge agent on Docker** | **check-in time, not `Status`** |
| 5 | Local Kubernetes | `Status` |
| 6 | Agent on Kubernetes | `Status` |
| 7 | **Edge agent on Kubernetes** | **check-in time, not `Status`** |

## Key fields on each environment object

- `Id`, `Name`, `Type`, `Status`, `URL`, `GroupId`, `TagIds`
- `Snapshots[0]` — full inventory snapshot (Docker: `ContainerCount`, `RunningContainerCount`,
  `StoppedContainerCount`, `ImageCount`, `VolumeCount`, plus raw container/image lists;
  K8s equivalent under `Kubernetes.Snapshots`). This is the megabyte-scale payload —
  project into the specific counters you need or exclude it.
- `Heartbeat` (boolean) — the server-computed edge health signal, set at response time
  on both list and inspect calls (CE and EE). It is what the dashboard's environment
  badge renders. **Only computed for edge environments** — it stays `false` for healthy
  direct agents, so never read it on non-edge types.
- `LastCheckInDate` (Unix seconds) and `EdgeCheckinInterval` (seconds) — the inputs
  behind `Heartbeat`. In list responses the server substitutes the global
  `EdgeAgentCheckinInterval` when the per-endpoint value is unset, so you see the
  effective interval.

### Edge health, ready to paste

```bash
curl -fsS -H "X-API-Key: $PORTAINER_API_TOKEN" \
  "$PORTAINER_URL/api/endpoints?excludeSnapshots=true" | jq '
  .[] | {id: .Id, name: .Name, type: .Type,
    up: (if .Type == 4 or .Type == 7 then .Heartbeat else .Status == 1 end)}'
```

How the server computes it (useful when reasoning about flapping or very old
versions that predate the field): up = `now - LastCheckInDate <= 2 × interval + 20`
seconds. For standard edge agents the interval is the per-endpoint
`EdgeCheckinInterval`, else the global setting. For **async** edge agents it is the
smallest of the ping/command/snapshot intervals (per-endpoint override, else global;
bounded at 60s) — so don't apply the standard formula to async agents by hand.

For container counts per environment, keep snapshots and project:

```bash
curl -fsS -H "X-API-Key: $PORTAINER_API_TOKEN" "$PORTAINER_URL/api/endpoints" | jq '
  .[] | {name: .Name,
         running: .Snapshots[0].RunningContainerCount,
         total: .Snapshots[0].ContainerCount}'
```

## Creating an environment — `POST /api/endpoints` (admin, multipart!)

Environment creation is `multipart/form-data`, not JSON — use curl `-F`:

| Form field | Notes |
|---|---|
| `Name` | required |
| `EndpointCreationType` | required: `1` local Docker, `2` agent, `3` Azure, `4` edge agent, `5` local Kubernetes |
| `URL` | host of the Docker daemon / agent, e.g. `tcp://10.0.0.5:9001`; required for type 4 |
| `PublicURL` | where published container ports are reachable |
| `GroupID` | defaults to `1` (Unassigned) |
| `TLS`, `TLSSkipVerify`, `TLSSkipClientVerify` | all must be `true` for type 2 (agent) in the standard setup |
| `ContainerEngine` | `docker` (default) or `podman` |

```bash
# Register a Portainer agent running at 10.0.0.5:9001
curl -fsS -X POST -H "X-API-Key: $PORTAINER_API_TOKEN" \
  -F Name=staging -F EndpointCreationType=2 \
  -F URL=tcp://10.0.0.5:9001 \
  -F TLS=true -F TLSSkipVerify=true -F TLSSkipClientVerify=true \
  "$PORTAINER_URL/api/endpoints"
```

The response includes the new environment's `Id` — capture it; every subsequent call needs it.

For **edge** environments (type 4), creation returns an `EdgeKey` that the edge agent is
started with; the agent dials home, so `URL` is informational. The separate
`POST /api/endpoints/{id}/edge/generate-key` regenerates a key.

## Deleting

- `DELETE /api/endpoints/{id}` — single environment (admin).
- `POST /api/endpoints/delete` — batch, body `{"endpoints": [{"id": N, "deleteCluster": false}, ...]}`.
  (The `DELETE /api/endpoints` batch form exists but is deprecated.)

## Groups and tags

`/api/endpoint_groups` and `/api/tags` are plain CRUD (JSON bodies). Assign an environment
to a group at creation (`GroupID`) or via `PUT /api/endpoints/{id}` with `{"GroupID": N}`.
