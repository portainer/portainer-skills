---
name: portainer-api
description: How to drive the Portainer REST API correctly from curl or any plain-HTTP client using an API token — authentication with X-API-Key, the undocumented Docker/Kubernetes proxy surface, resolving environment IDs, deploying and managing stacks, keeping huge responses out of context with jq, and reading results correctly (edge agent health comes from check-in time, not Status). Use this whenever the task involves calling Portainer over HTTP — curl commands, bash/Python/Ansible automation, generated scripts, webhooks, or debugging a failing Portainer API call — even if the user doesn't say "Portainer API" explicitly. Do NOT use it when Portainer MCP tools (mcp__portainer__*) are available in the session — prefer those tools instead.
compatibility: Requires curl and jq on the machine making the calls.
---

# Portainer API (curl + API token)

Everything in the Portainer UI is also doable over its HTTP API. This skill covers how to call it correctly. The full OpenAPI reference ships in `references/spec/` — one small file per API area plus a greppable operation index, so exact parameters are always one grep away. See *Finding the right endpoint* below.

Read the domain reference when the task touches it:

- `references/environments.md` — listing/creating environments, snapshot fields, edge agent health
- `references/docker-proxy.md` — anything Docker on a specific environment (containers, images, logs, swarm)
- `references/kubernetes-proxy.md` — anything Kubernetes or Helm on a specific environment
- `references/stacks.md` — deploying, updating, and removing stacks (compose/swarm/k8s, string/file/git)

## Authentication: X-API-Key, not Bearer

Portainer accepts two credentials, and mixing them up is the most common failure on this surface:

- **API token** (access token) → header `X-API-Key: ptr_...`. Long-lived, revocable, no expiry mid-run. **This is the right mechanism for automation.**
- **JWT** from `POST /api/auth` → header `Authorization: Bearer <jwt>`. Expires (hours). Only worth it when no token exists yet.

Putting an API token in a `Bearer` header fails with a 401 that looks like a bad credential — if you get 401 with a token you believe is valid, check the header name first.

```bash
# The pattern for every call. Keep the token in an env var, never inline.
curl -fsS -H "X-API-Key: $PORTAINER_API_TOKEN" "$PORTAINER_URL/api/system/version"
```

Getting a token when only username/password exists — note that **creating a token requires a JWT** (you can't mint a token with a token), and the body must repeat the password for internal auth:

```bash
JWT=$(curl -fsS -X POST "$PORTAINER_URL/api/auth" \
  -d '{"username":"admin","password":"'"$PW"'"}' | jq -r .jwt)
curl -fsS -X POST "$PORTAINER_URL/api/users/1/tokens" \
  -H "Authorization: Bearer $JWT" \
  -d '{"description":"automation","password":"'"$PW"'"}' | jq -r .rawAPIKey
```

The user ID is in the JWT response context or `GET /api/users/me`. Tokens can also be created in the UI (My account → Access tokens).

**TLS:** Portainer commonly runs on `https://host:9443` with a self-signed cert. Prefer `--cacert` with the instance's CA; fall back to `-k` only for throwaway/demo instances, and say so in generated scripts so nobody ships it to production.

## URL anatomy — one base, three surfaces

Everything lives under `/api`. There are three kinds of paths, and knowing which one you're on tells you where its documentation lives:

1. **Typed Portainer endpoints** — `/api/stacks`, `/api/endpoints`, `/api/users`, `/api/kubernetes/{id}/...`, `/api/docker/{environmentId}/dashboard`, ... These are in the bundled spec.
2. **The Docker proxy** — `/api/endpoints/{id}/docker/<any Docker Engine API path>`. Portainer forwards the request to the environment's Docker daemon verbatim; request and response bodies are exactly the Docker Engine API's. **This surface is deliberately absent from the spec** ("not documented below due to Swagger limitations" per the spec itself) — don't conclude it doesn't exist because you can't find it there. Your Docker API knowledge applies directly.
3. **The Kubernetes proxy** — `/api/endpoints/{id}/kubernetes/<any K8s API path>`, same idea for the cluster's API server.

```bash
# Docker Engine API, proxied — exactly like talking to the daemon:
curl -fsS -H "X-API-Key: $PORTAINER_API_TOKEN" \
  "$PORTAINER_URL/api/endpoints/3/docker/containers/json?all=true"
# Kubernetes API, proxied:
curl -fsS -H "X-API-Key: $PORTAINER_API_TOKEN" \
  "$PORTAINER_URL/api/endpoints/3/kubernetes/api/v1/namespaces/default/pods"
```

## First moves on any task

**1. Check what you're talking to.** `GET /api/system/version` returns the version; `GET /api/settings/public` tells you edition-relevant config without auth. The bundled spec describes **Enterprise Edition (EE)**, the superset (its exact version is in the `references/spec/INDEX.md` header); on Community Edition, EE-only paths (GitOps, observability, policies, licenses, KaaS/cloud provisioning, edge update schedules) return 404 — a 404 there means "wrong edition", not "wrong URL". Compare the instance's version against the spec's: on a minor mismatch, treat the spec as approximate at the edges — recently added endpoints and fields are the suspect set when a documented call misbehaves (see *Version skew* below).

**2. Resolve the environment ID before any environment-scoped call.** IDs are assigned in creation order — `1` is whatever was added first, not "local". Never guess; a wrong ID gives a 404 from the proxy.

```bash
curl -fsS -H "X-API-Key: $PORTAINER_API_TOKEN" \
  "$PORTAINER_URL/api/endpoints?excludeSnapshots=true" \
  | jq '.[] | {id: .Id, name: .Name, type: .Type, status: .Status}'
```

Then use the ID whose name matches what the user means. `?search=<name>` narrows server-side if the list is long.

## Keep responses out of your context

There is no response cap protecting you: `GET /api/endpoints` *with* snapshots can be megabytes, a K8s object list with `managedFields` is routinely 10× the useful content. The discipline:

- **Narrow upstream when the API supports it.** `excludeSnapshots=true` on environment lists; Docker `filters` (URL-encoded JSON); K8s `labelSelector`/`fieldSelector`; `tail`/`since` on logs. Server-side narrowing beats any client-side trick.
- **Project with `jq` immediately.** Pipe every list-shaped response through a `jq` projection that keeps only the fields the question needs — same habit as a SQL `SELECT` list instead of `*`.
- **For exploratory reads, write to a file first** (`curl ... -o /tmp/out.json`), then query the file with `jq` repeatedly. Never dump an unknown-size body to stdout.

The heavy fields, so you know what to cut: `Snapshots[0]` on environments (full container/image/volume inventory per env), `metadata.managedFields` and `status` blocks on K8s objects, `StackFileContent` and `Env` on stacks.

## Read results correctly

**Edge agent health does not live in `Status`.** `Status` means up(1)/down(2) only for *directly connected* environments. For edge agents (`Type` 4 = Docker, 7 = Kubernetes) it stays `0` forever — reading that as "down" is a false alarm. The authoritative signal is the boolean `Heartbeat` field: both CE and EE compute it server-side at response time on `GET /api/endpoints` and `GET /api/endpoints/{id}` (up = checked in within `2 × check-in interval + 20` seconds — the same value the dashboard badge renders). Two traps: `Heartbeat` is only computed for edge environments, so it reads `false` on a perfectly healthy direct agent — never use it for non-edge types; and it's computed per response, so a stale cached body lies.

```bash
curl -fsS -H "X-API-Key: $PORTAINER_API_TOKEN" \
  "$PORTAINER_URL/api/endpoints?excludeSnapshots=true" | jq '
  .[] | {name: .Name, type: .Type,
    up: (if .Type == 4 or .Type == 7 then .Heartbeat else .Status == 1 end)}'
```

**You will see real secrets.** Unlike the MCP server, the raw API does not redact anything: stack `Env` values, container environment variables, and K8s Secrets come back in cleartext to any caller whose RBAC allows the read. Don't echo them into chat, logs, or generated scripts unless the user explicitly needs the value.

## Mutations

- Mutating endpoints take JSON bodies (`Content-Type: application/json`, curl sets it with `-d` plus an explicit header) **except** file uploads and environment creation, which are `multipart/form-data` (curl `-F`). The spec makes this visible: check the operation's `requestBody.content` key — `application/json` vs `multipart/form-data`.
- Many mutations need the environment as a **query parameter** even though the resource has its own ID: stack create/start/stop/delete all require `?endpointId=N`. Forgetting it is a 400, not a helpful message.
- Errors come back as `{"message": "...", "details": "..."}` — always print the body on failure (`curl -fsS` hides it; use `--fail-with-body` or check the body before failing) because the `details` string is usually the actual answer.
- Most mutations need admin or matching RBAC; 403 with a valid token means the *role* is insufficient, not the credential.

Stack deployment has nine creation endpoints (3 platforms × 3 sources) with different body shapes — read `references/stacks.md` before writing one.

## Error decoding

| Code | Almost always means |
|---|---|
| 401 | Token in the wrong header (`Bearer` instead of `X-API-Key`), or token invalid/revoked |
| 403 | Credential fine, RBAC role insufficient (admin-only endpoint, or resource not authorized) |
| 404 | Wrong environment ID, EE-only endpoint on CE, endpoint newer than the instance (version skew), or genuinely missing resource — check in that order |
| 400 | Missing required query param (`endpointId` is the classic), malformed body, or a body shape that changed across versions — read `details` |
| 409 / 500 + message | Conflict (e.g. stack name already in use) — the `message`/`details` body names it |

**Version skew also fails silently, twice.** Portainer ignores unknown JSON body fields, so on an instance older than the bundled spec a mutation can return 200 *without applying* a newer field — when versions mismatch, verify the effect with a follow-up GET instead of trusting the status code. And a `jq` projection over a field the instance doesn't have yet returns `null`, which reads as "not set" when the truth is "not supported in this version".

## Finding the right endpoint

The spec lives in `references/spec/` as one **self-contained file per API area**: each chunk holds that area's paths plus every schema they reference, so a `$ref` never points outside the file you're reading. Two generated entry points sit alongside:

- `INDEX.md` — one line per operation, `METHOD /path → chunk file — summary`. **Grep this first**; it resolves any path or keyword to the file that documents it.
- `openapi.yaml` — the root document stitching the chunks together via `$ref`. For tooling, not for reading.

The lookup procedure — two greps answer almost any "what does this take" question:

```bash
# 1. Resolve the operation to its chunk (search by path fragment or keyword):
grep '/stacks/create' references/spec/INDEX.md
#    POST /stacks/create/standalone/string → stacks.yaml — Deploy a new compose stack from a text
# 2. Jump to the operation — path keys sit at 2-space indent, paths start ~line 14:
grep -n '^  /stacks/create/standalone/string:' references/spec/stacks.yaml
# 3. A $ref names a schema in the SAME file, at 4-space indent under components.schemas:
grep -n '^    stacks.composeStackFromFileContentPayload:' references/spec/stacks.yaml
```

Most chunks are a few hundred to ~2k lines — reading a whole operation (~40–90 lines) or a small chunk end-to-end is fine. Two exceptions demand grep-first discipline: `kubernetes.yaml` (~8k lines) and `endpoints.yaml` (~3.3k lines). Never read those whole.

The routing table — which chunk covers what. Entries marked **EE** are Enterprise Edition only (404 on CE):

| Need | Paths (relative to `/api`) | Spec chunk |
|---|---|---|
| Environments | `/endpoints`, `/endpoints/{id}`, `/endpoints/snapshot` | `endpoints.yaml` |
| Docker (proxied) | `/endpoints/{id}/docker/*` | not in the spec — Docker Engine API; see `references/docker-proxy.md` |
| Kubernetes (proxied) | `/endpoints/{id}/kubernetes/*` | not in the spec — K8s API; see `references/kubernetes-proxy.md` |
| Docker (typed) | `/docker/{environmentId}/{dashboard,images,snapshot}` | `docker.yaml` |
| Kubernetes (typed) | `/kubernetes/{id}/...` (namespaces, applications, services, ingresses, nodes, configmaps, secrets, dashboard) | `kubernetes.yaml` |
| Helm | `/endpoints/{id}/kubernetes/helm*`, `/templates/helm` | `helm.yaml` |
| Stacks | `/stacks`, `/stacks/create/...`, `/stacks/{id}/...` | `stacks.yaml` |
| Edge | `/edge_stacks`, `/edge_groups`, `/edge_jobs`, `/edge_configurations` | `edge_stacks.yaml`, `edge_groups.yaml`, `edge_jobs.yaml`, `edge.yaml`, `edge_configs.yaml`, `edge_update_schedules.yaml` (**EE**) |
| Interactive TTY (container exec/attach, pod shell) | `/websocket/*` | `websocket.yaml` — WebSocket upgrade, `wscat`/client lib, not curl |
| Users, teams, tokens | `/users`, `/users/{id}/tokens`, `/teams`, `/team_memberships` | `users.yaml`, `teams.yaml`, `team_memberships.yaml`, `roles.yaml` (**EE**) |
| Auth | `/auth`, `/auth/logout` | `auth.yaml` |
| Registries | `/registries`, `/registries/{id}` | `registries.yaml` |
| Templates | `/templates`, `/custom_templates` | `templates.yaml`, `custom_templates.yaml` |
| Settings & system | `/settings`, `/settings/public`, `/system/*` | `settings.yaml`, `system.yaml`, `status.yaml` |
| Webhooks | `/webhooks` | `webhooks.yaml` |
| Backup | `/backup`, `/restore` | `backup.yaml` (**EE** for S3/Azure) |
| EE-only suites | `/gitops/*`, `/observability/*`, `/policies/*`, `/licenses`, `/cloud/*`, `/omni/*` | `gitops.yaml`, `observability.yaml`, `policies.yaml`, `license.yaml`, `kaas.yaml`, `cloud_credentials.yaml`, `omni.yaml` |

**Trust the spec, with two caveats.** The spec is generated from Go annotations and a few defects leak through: enum lists on Go-internal types (`os.FileMode`, `time.Duration`, `policies.PolicyType`) are Go constants, not API values — when an enum looks like leaked language internals, trust the field's description and example instead. And a handful of operations (`SharedGit*`, `providerInfo`, `provisionCluster`) ship structurally broken parameter schemas.

## When the skill itself is wrong

If reality contradicts this skill — the spec documents a body the API rejects, an example fails as written, the routing table points at the wrong chunk, guidance sent you down a wrong path — offer to file an issue on [`portainer/portainer-skills`](https://github.com/portainer/portainer-skills) so the gap gets fixed for everyone. This is for *skill defects only*: not user errors, not instance misconfiguration, not expected edition/version skew, and not Portainer product bugs (those belong on `portainer/portainer`).

1. **Ask before drafting.** One line — "this looks like a defect in the portainer-api skill; want me to file an issue on `portainer/portainer-skills` on your behalf?" If the user isn't keen, drop it entirely; don't draft anything.
2. **Draft and scrub.** Replace hostnames/IPs/URLs, usernames, and resource names with placeholders (`<portainer-host>`, `<stack-name>`); never include tokens, env values, or response dumps.
3. **Include**: a title prefixed `[portainer-api]`, the bundled spec version (`INDEX.md` header), the target's version/edition, what the skill or spec said (quote it), the sanitized call (method + path), what actually happened (status + `message`/`details`), and what you expected.
4. **Show the draft, then file on approval** — `gh issue create --repo portainer/portainer-skills --title … --body …` if `gh` is available and authenticated; otherwise hand the user a prefilled link they can open themselves: `https://github.com/portainer/portainer-skills/issues/new?title=<url-encoded>&body=<url-encoded>`. Never file silently.
