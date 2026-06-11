# portainer-api

An agent skill that teaches an AI agent to drive the Portainer REST API from curl or any http client.
It includes guidance around authentication, the Docker/Kubernetes proxy surfaces and how to navigate the bundled OpenAPI reference without flooding context.

## Install

> [!NOTE]
> Make sure to have `jq` installed as it is used by the skill for filtering data.

Clone the repo and copy this directory into a skills location:

```bash
git clone https://github.com/portainer/portainer-skills.git
cp -r portainer-skills/portainer-api ~/.claude/skills/portainer-api
```

## Versioning

Releases are tagged to match the Portainer version whose spec they bundle —
install the tag matching your Portainer server:

```bash
git clone --depth 1 --branch 2.42.0 https://github.com/portainer/portainer-skills.git
```

If that version is not available, clone and regenerate the specs:

```bash
make specs VERSION=2.39.3
```

## Updating the bundled spec

```bash
make specs VERSION=2.42.0
```

This sparse-clones the version's per-tag chunk files published by
[portainer-api-docs](https://github.com/portainer/portainer-api-docs)
into `upstream/` (generated there by `chunk-by-tag.js`: each chunk carries
its tag's paths plus all transitively referenced schemas, so `$ref`s never
leave the file), then post-processes them locally via
`scripts/postprocess_specs.py`:

- strips the `info.description` preamble repeated in every chunk,
- filters each chunk's `tags:` list to the tags it actually uses and drops
  the Redoc-only `x-tagGroups:` block,
- regenerates `references/spec/INDEX.md`,
- re-parses every chunk and fails loudly if anything broke.
