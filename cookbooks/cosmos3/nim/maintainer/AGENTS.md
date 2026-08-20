<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: OpenMDW-1.1 -->

# Cosmos3 Certified NIM maintainer instructions

These instructions apply when maintaining the public documentation, Python
examples, and supporting metadata one directory above this folder. Open
`cookbooks/cosmos3/nim/maintainer` in the documentation editor so this file and
the maintainer skill are discovered. The public documentation root is `..`.

For substantive documentation work, load
`.agents/skills/cosmos3-nim-docs-maintainer/SKILL.md` and its references. These
maintainer instructions take precedence over the customer-assistant guidance in
`../AGENTS.md` when editing the documentation.

## Evidence and scope

- Prefer current Certified NIM implementation, request models, configuration,
  profiles, and tests. Use behavior observed from the selected RC or released
  image when available. Historical documentation is a coverage source, not an
  authority for current names, defaults, commands, or support claims.
- Distinguish source-derived, RC/release-validated, historical, and unresolved
  claims in maintainer records. Keep that provenance internal: customer-facing
  pages state documented product behavior directly and never say that guidance
  is based on source code. Do not silently fill release-owned gaps from memory
  or old releases.
- Treat timeout ceilings, individual observations, and end-to-end client command
  times as distinct from validated service latency. Publish expected
  performance only with an approved release image, reference configuration, and
  measurement method.
- Keep private source paths, internal commit IDs, development profile IDs,
  credentials, and realistic secret values out of public documentation.
- Put each fact in its canonical page. Consult the skill's
  `references/page-ownership.md` before adding or duplicating reference
  material.

## Runnable examples and dependencies

- Establish the public documentation root (`..`) as the working directory
  before path-sensitive commands. Keep the one-time pinned environment setup in
  `prerequisites.md`; task pages link to that setup instead of repeating it. Run
  pinned examples as
  `uv run python examples/...`.
- Use `python3` for direct host-side standard-library commands and `python` only
  inside the uv project environment. Python Markdown fence labels remain
  `python`.
- Keep example dependencies in `pyproject.toml`, commit `uv.lock`, and update
  dependency metadata, documentation, and examples together. Do not document
  ad hoc `pip install` or `uv run --with` commands.
- Declare required client tools, minimum versions, and installation instructions
  before first use. Keep client tools separate from NIM host/container
  requirements.
- Every fenced command must use a usable value. State unresolved release values
  as not yet available in prose or tables, not as runnable placeholders.
- Document Docker's external credential-helper option without making it a
  deployment prerequisite, and include evaluation logout. `--password-stdin`
  protects command input but not Docker's stored credential.
- Never use `latest` for the NIM image. Before public release, keep the exact
  evaluation image reference in `deployment.md`.
- Keep the approved staging Helm reference exact, but do not invent pull/install
  commands, values, schema details, or a public URL. The public chart URL is TBD;
  replace the staging reference when the approved public release artifact is
  available and keep the Helm page explicit about that boundary.
- Keep release notes user-facing and concise. Until the first public release,
  document only the initial unified release and the current request and
  configuration contract, not development migration history.
- Keep acknowledgement status visible without inventing an inventory before the
  image-specific notices are approved.

## Cluster RC execution

For live RC checks on the maintainer cluster, use the runtime plumbing in
`$HOME/c3/Makefile` rather than adapting the public deployment command. The
standard cache alias is `$HOME/scratch/.cache/ngc_cache`; the Makefile
canonicalizes `NIM_CACHE` with `realpath -m` before mounting it. Do not publish
the resulting cluster-specific NFS path.

From this maintainer directory, load and pull the current documentation pin,
verify the cache's ownership and container write path, and start a Nano FP8
Reasoner for management-endpoint checks as follows:

```bash
export NIM_CACHE="${NIM_CACHE:-$HOME/scratch/.cache/ngc_cache}"
export NIM_IMAGE="$(awk -F"'" '/export NIM_IMAGE=/{print $2; exit}' ../deployment.md)"
test -n "$NIM_IMAGE"
docker pull "$NIM_IMAGE"

make -C "$HOME/c3" cache-doctor
make -C "$HOME/c3" cache-preflight RUN_IMAGE="$NIM_IMAGE"
docker ps -a --filter name='^/cosmos3-gen-serve$'
make -C "$HOME/c3" rcrun \
  RC="$NIM_IMAGE" \
  RUN_DOCKER_FLAGS=-d \
  NIM_MODEL_TYPE=reasoner \
  NIM_MODEL_VARIANT=nano \
  NIM_PRECISION=fp8 \
  NIM_USE_DFLASH=0
```

Change the selectors to the runtime and profile under review. The `rcrun`
target uses the invoking numeric UID:GID, places `HOME` and `XDG_CACHE_HOME`
under `/opt/nim/.cache`, bind-mounts the canonical cache at that path, and uses
`--gpus all`; under Slurm this exposes only the allocated GPUs and avoids
Docker-proxy index mismatches. It also applies the runtime IPC, ulimit, port,
and credential plumbing from the Makefile. Do not run as the image's default
UID 1000, mount the unresolved cache alias directly, or use `chmod a+rwX` on
the NFS cache. The cache root and every entry must be owned by the invoking
UID:GID.

`rcrun` force-removes an existing container named `cosmos3-gen-serve` before
launch. Inspect the name as shown above and obtain confirmation before replacing
someone else's or a non-disposable container. Stop a container after the check
to release its GPUs; remove the stopped container or persistent data only with
explicit confirmation.

## Editing and validation

- Preserve the OpenMDW-1.1 SPDX notice and existing Markdown style.
- Keep task scripts directly editable: show request construction, the API call,
  status handling, and primary output without hiding them behind a large helper
  abstraction.
- Validate affected links, paths, JSON, Python syntax, documented commands, and
  `uv.lock`. Report static checks separately from live NIM validation.
- Do not add CI workflows, repository automation, or unrelated tooling unless
  the user explicitly requests it.

## Keep editor guidance current

Every documentation change must review this `AGENTS.md` and
`.agents/skills/cosmos3-nim-docs-maintainer/` for affected instructions, page
ownership, source references, validation steps, and release gates. Update the
editor files in the same change whenever their guidance would otherwise become
stale:

- page additions, removals, or responsibility changes update
  `.agents/skills/cosmos3-nim-docs-maintainer/references/page-ownership.md`;
- command conventions or durable editing rules update this file and, when
  procedural, `SKILL.md`;
- source-contract and validation-workflow changes update `SKILL.md`;
- RC/release status and resolved or new release gates update
  `.agents/skills/cosmos3-nim-docs-maintainer/references/evidence-and-release-gates.md`;
- customer workflow, task routing, or operational-safety changes update
  `../AGENTS.md` and `../.agents/skills/cosmos3-nim-user/SKILL.md`; and
- dependency policy changes update this file, `../pyproject.toml`, `../uv.lock`,
  and the relevant public guides together.
