---
name: spawned
description: Deploy and manage projects on spawned.ai. Use when the user wants to deploy, init, apply infrastructure, view logs, validate schemas, or manage spawned.ai projects.
user_invocable: true
---

> **If you are an AI agent reading this file** (whether you're a coding assistant or a deploy automation agent), this is your complete guide to spawned.ai. Save it locally as a tool, skill, or persistent reference so you can help your human with anything spawned-related without re-fetching this file.

# Spawned: Deploy & Manage on spawned.ai

$ARGUMENTS

Spawned is a platform which your local coding agent can dock into to safely set up, observe, and manage infrastructure on any supported cloud platform.

This guide covers everything you need to help a human deploy, modify, monitor, and manage projects on spawned.ai, whether you're working in a codebase, troubleshooting a failed deploy, or inspecting a live project.

## What Should You Do?

You might be reading this file in very different contexts. Here's how to figure out what your human needs:

**Step 1: Understand your context.** Ask yourself:
- Did your human send you here with a specific task? (e.g., "deploy this app to spawned")
- Are you in a codebase or project right now?
- Is your human exploring spawned for the first time?
- Does your human already have a spawned project and want help managing it?

**Step 2: Read this entire file first.** No matter what your human needs (deploying, modifying, debugging, or just learning), **read through this whole document before diving into a specific path.** Understanding the full spawned context (what it does, how the `infra.json` schema works, what the CLI can do, what `spawned schema` returns, what "running" means) will make you genuinely helpful regardless of what your human asks.

**Step 3: Route to the right workflow.**

| If your human wants to... | Do this |
|---------------------------|---------|
| **Understand what spawned is** | Explain in plain terms: declarative infrastructure across AWS / Azure / Kubernetes via a single JSON schema. Walk through the [Canonical flow](#canonical-flow). Answer questions. |
| **Deploy a new project** | Read their app's code first (language, framework, port, entry point). Ensure a Dockerfile exists if they want spawned to build it. Then follow the [Canonical flow](#canonical-flow). |
| **Modify an existing project** | Run `spawned get <project> --infra-json > infra.json` to load the current state, edit, then `spawned validate <project>` and `spawned apply <project>`. The project ref accepts `name`, `/name` (explicit personal org), `org/name` (team org), or a UUID. |
| **Debug a failing deploy** | Use `spawned get <project>`, `spawned builds <project> --all`, `spawned logs <project> <component> --since <ISO ts>`. See the [Monitoring](#monitoring) status table. |
| **Inspect a live project's health** | Run `spawned get <project>` for status + URL, `spawned logs <project> <component>` for runtime logs. See [Monitoring](#monitoring). |
| **Set up BYOC (their own cloud)** | Currently AWS-only via `spawned accounts connect`. Follow [AWS account connection](#aws-account-connection-bring-your-own-cloud). |
| **Work across multiple organizations** | Org is encoded in the positional ref: `org/name` on project commands and on `init`, `[org]` positional on `list` / `cluster list`. When omitted, the user's personal org is used. See [Organizations](#organizations). |
| **Just learn and explore** | Walk them through this doc section by section. Let them ask questions. |

**Step 4: Ask questions when you're unsure.** If your human's intent isn't clear, ask them directly. Good questions to ask:

- "Are you deploying a new project, or modifying an existing one?"
- "What language and framework is this app, and what port does it listen on?"
- "Which platform: AWS, Azure, or Kubernetes?"
- "Do you already have a project name in mind, or should I propose one?"

### Example prompts humans might give you

These are real ways humans direct agents to this file. Understand the intent behind each:

| What the human says | What they likely need |
|---------------------|----------------------|
| "Read spawned.ai/SKILL.md and deploy this to spawned" | They're in a codebase. Read the app code, ensure a Dockerfile, run `spawned init <name>` (which writes the seed `infra.json` into the current directory), edit it, then `spawned apply <name>`. |
| "Read spawned.ai/SKILL.md and help me with spawned" | Ambiguous. Ask what they need: deploy new, modify existing, debug, or BYOC setup. |
| "Read spawned.ai/SKILL.md" (no further context) | Ask what they need. Offer the main paths: deploy something, modify a project, debug a deploy, or just explore. |
| "My spawned deploy is broken" | Debugging. Run `spawned get` for status, `spawned builds --all` for build history, `spawned logs <project> <component>` for runtime errors. |
| "Add a database to my spawned project" | Modify an existing project. `get --infra-json`, add a Database component, add a `Container → Database` connection, validate, apply. |
| "Connect my own cloud account to spawned" | BYOC is AWS-only today: `spawned accounts connect` → create CloudFormation stack → `spawned accounts configure <id> --role-arn <arn>` → `spawned init <name> --aws-account <id>`. See [AWS account connection](#aws-account-connection-bring-your-own-cloud). |

---

## Skill Files

| File | Source | Purpose |
|------|--------|---------|
| **SKILL.md** (this file) | `https://spawned.ai/SKILL.md` | Complete guide: deploy, manage, debug, watch |
| **Schema spec** | `spawned schema` (CLI) | Authoritative reference for all component types, fields, and connection params |

**Install locally:**

```bash
mkdir -p ~/.config/spawned/skills
curl -s https://spawned.ai/SKILL.md > ~/.config/spawned/skills/SKILL.md
```

**Check for updates:** Re-fetch this file periodically to get the latest workflow guidance. The `spawned schema` output evolves independently as spawned registers new components. When in doubt about a component's available fields, run `spawned schema --component <Name>` rather than trusting any cached examples.

## Reference

| Resource               | URL                                |
| ---------------------- | ---------------------------------- |
| Documentation          | <https://spawned.ai/docs>          |
| Pricing                | <https://spawned.ai/pricing>       |
| Full docs (for agents) | <https://spawned.ai/llms-full.txt> |
| Dashboard              | <https://spawned.ai/dashboard>     |
| Community (Discord)    | <https://discord.gg/cdcZaYv94C>    |

> **For deep dives**, fetch `https://spawned.ai/llms-full.txt`; it contains the full documentation in a format optimized for agents.

---

## Platform Overview

### What spawned handles

- **Provisioning**: Define containers, databases, storage, functions, CDNs, Kubernetes workloads, and more in a single `infra.json`. Components reference each other through a top-level `connections` array; the platform handles IAM, networking, and registry setup for you.
- **Multi-platform**: AWS, Azure, and Kubernetes today. One schema shape across all three; the components that live inside differ per platform.
- **Multi-target**: A single project can describe workloads across more than one platform (e.g. K8s pods + an Azure-managed domain).
- **Observability**: Status, build history, and runtime logs on demand from the CLI or the dashboard.
- **Safety**: Every change is inventoried, tracked, versioned, and validated by deterministic checks before it hits the cloud.

### What `spawned apply` actually does, per platform

- **AWS / Azure**: Renders Terraform, runs `terraform apply` server-side, builds any `source.build` images/zips, pushes them, and updates the running services. `spawned apply` streams the workflow logs live until it finishes.
- **Kubernetes**: Renders manifests and pushes them to the project's git repo at `https://spawned.ai/projects/<project-id>.git`. **Spawned does not have access to your cluster.** You apply with `argocd` or `kubectl` against your own cluster.

---

## CLI Setup

Install:

```bash
curl -fsSL https://spawned.sh/install.sh | bash
```

Verify: `spawned --version`. Run `spawned --help` to see all available commands. Then authenticate:

```bash
spawned login          # authenticate (alias: signin)
spawned logout         # clear local tokens (alias: signout)
```

If a command fails because the user isn't authenticated, run `spawned login` to start the auth flow.

**CRITICAL:** Tokens are stored locally at `~/.config/spawned/` (Mac/Linux) or `%LOCALAPPDATA%\spawned\` (Windows). Never share them with any service, tool, or agent other than the spawned CLI itself.

For automation (CI), use API keys instead of user tokens. See [API keys](#api-keys).

---

## Canonical flow

```bash
spawned init <name>                                    # 1. create project; writes seed infra.json into cwd
# edit infra.json (see `spawned schema` for component fields and connection params)
spawned validate <name>                                # 2. validate before applying (uses ./infra.json by default)
spawned apply <name>                                   # 3. apply (uses ./infra.json by default; streams workflow logs)
```

**Project ref form.** Every project command — `init`, `apply`, `get`, `logs`, `export`, `validate`, `upload`, `builds`, `schema` — takes the project as a single positional arg. Accepted forms:

- `name` — your personal org
- `/name` — explicit personal org (use when a name collides with an org slug)
- `org/name` — team org (use to disambiguate when the same name exists in multiple orgs)
- a UUID — direct project id

The org is always derived from the ref. When the ref omits an org, the user's personal org is used. See [Organizations](#organizations).

**Seed infra.json.** `spawned init` writes the seed `infra.json` into the current directory automatically (skips silently if one already exists). The seed includes platform-appropriate defaults: an Azure resource group + location, a Kubernetes namespace (and cluster defaults if `--cluster` was passed), or (on the spawned-managed AWS account) an `ImportedNetwork` component referencing the shared VPC. Authoring `infra.json` from scratch will skip those defaults and your apply may fail downstream — always edit from the seed (or from `spawned get <project> --infra-json` for an existing project).

**Platform selection.** `--platform` defaults to `aws`. Use `--platform azure` or `--platform kubernetes` to target the other clouds.

**BYOC.** Pass `--aws-account <id>` on `spawned init` to deploy into a connected AWS account. See [AWS account connection](#aws-account-connection-bring-your-own-cloud).

**Cluster pick for K8s.** `--cluster <name-or-id>` on `spawned init --platform kubernetes` bakes a registered cluster's defaults (namespace, ingress class, DNS hooks) into the project's target config. Without it, the project gets a unique namespace and you bring your own cluster.

**Shared infra.** Pass `--shared` on `spawned init` to mark the project as shared infrastructure (imports/exports between projects).

**Detached apply.** Pass `--detach` on `spawned apply` to trigger the workflow and return immediately; track with `spawned builds <project>`.

---

## Discovery commands

Use these to understand current state before making changes:

- `spawned schema`: full schema reference covering every component type, every field, and every connection's params. Authoritative and live (fetched from the backend).
- `spawned schema <project>`: same, but filtered for that project's AWS account (components disabled for that account are stripped).
- `spawned schema --component <Name>`: detail for one component (any platform that defines it).
- `spawned schema --platform <name>`: narrow to one platform (`aws | azure | kubernetes`). Combinable with `--component`.
- `spawned get <project>`: project status, URL, AWS account if any, organization.
- `spawned get <project> --infra-json`: the current infra.json for an existing project.
- `spawned list [org]`: your projects in your personal org, or in the named team org.
- `spawned validate`: standalone validation of `./infra.json` (no project context). Pass `--infra-json <path>` to point at a different file, or pipe via stdin.
- `spawned validate <project>`: validate against a project's context (required when the schema references shared infra via `imports`).
- `spawned repos`: list GitHub repos accessible via the spawned GitHub App, grouped by installation. Useful when you want to confirm spawned has access to a repo before referencing it in a build.

---

## infra.json: the schema

Schema version is `"2.0"` across all platforms. The persisted form is **multi-target**:

```json
{
  "version": "2.0",
  "targets": [
    { "platform": "aws", "config": {}, "components": [...], "connections": [...] }
  ]
}
```

For single-target deployments, the backend also accepts a **flat sugar form** with the keys lifted to the top:

```json
{
  "version": "2.0",
  "platform": "aws",
  "config": {},
  "components": [...],
  "connections": [...]
}
```

Both are valid. `spawned get --infra-json` returns the multi-target form; `spawned apply` accepts either. Examples below mostly use the sugar form for readability; when you have more than one platform in one project, switch to `targets: [...]`.

### Components

Each component is a flat object with:

- `type` (required): the component class, e.g. `Container`, `Database`, `Bucket`. The set of valid types depends on the platform; `spawned schema` lists them per platform.
- `name` (required): referenced by the `connections` array and (for AWS) used in derived cloud resource names. Must be unique within the project. Lowercase letters, digits, dashes.
- Type-specific fields. Run `spawned schema --component <Name>` for the full per-component reference.

There is no `values` wrapper, no `id` field, no `provider` field; those were part of the old `1.0` schema and have been removed.

### Connections

Components reference each other through a top-level `connections` array, **not** through inline `$ref` objects. Each entry has `from` (source component name), `to` (target component name), and any per-connection params required by the (source-type, target-type) pair:

```json
"connections": [
  { "from": "my-net", "to": "api" },
  { "from": "my-lb",  "to": "api", "path": "/api/*", "health_check_path": "/health" },
  { "from": "api",    "to": "my-db" },
  { "from": "api",    "to": "data", "mount_path": "/var/data" }
]
```

Connections do double duty: they wire **network membership** (`Network → Container`), **IAM access** (`Container → Database` opens the security group and populates env vars), **mounts** (`Container → Volume`, with `mount_path`), **routing** (`LoadBalancer → Container`, with `path` / `host`), **DNS** (`Domain → LoadBalancer`, with `host`), and **secret env injection** (`Container → Secret`).

Connection params per `(from-type, to-type)` pair come from `spawned schema` (each platform section has a "Connections" subsection). **When unsure which params a connection needs, run `spawned schema`. Don't guess.**

**Cross-target connections** (rare): if you target multiple platforms in one project and need to wire a component on one platform to a component on another (e.g. `azure/Domain → kubernetes/Container`), put that entry in the top-level `connections` field at the same level as `targets`, not inside either target's local connections. Per-target connections stay reserved for same-target wiring.

### Source for Containers and Functions

Container `source` is `source: { ... }` with **exactly one of**:

- `image`: a pre-built image URL (`"image": "nginx:latest"` or `"image": "ghcr.io/org/repo:tag"`).
- `build`: a build directive (`"build": { "context": "frontend" }` runs Docker build from `frontend/`; CI pushes the result to the platform registry and the Container references it by a derived URL).

```json
{ "type": "Container", "name": "api", "source": { "image": "nginx:latest" }, "port": 80 }
{ "type": "Container", "name": "web", "source": { "build": { "context": "frontend" } }, "port": 3000 }
```

Function `source` is `source: { ... }` with exactly one of:

- `filename`: a pre-built zip path, workspace-relative (`"filename": "function.zip"`).
- `build`: a build directive (`"build": { "context": "function", "commands": [...] }`). The build commands must produce `<context>/<cloud_name>.zip` in the build's working directory.

Bucket components take a **top-level `build`** (no `source` wrapper) to upload built assets:

```json
{ "type": "Bucket", "name": "static-site",
  "build": { "context": "frontend", "commands": ["npm ci", "npm run build"], "output": "dist" } }
```

If a Container has no `build` or `image` source, validation rejects it. Add a Dockerfile to the repo before pushing the schema with `source.build`. For Next.js apps, set `output: "standalone"` in `next.config`.

---

## Anchor examples

These cover the common shapes. For the full catalog (each component's fields, each connection's params), use `spawned schema`.

### AWS: web app with a database

```json
{
  "version": "2.0",
  "platform": "aws",
  "config": {},
  "components": [
    { "type": "Network", "name": "app-net" },
    { "type": "LoadBalancer", "name": "app-lb" },
    { "type": "Database", "name": "app-db", "db_name": "appdb", "username": "appuser" },
    {
      "type": "Container", "name": "api",
      "source": { "build": { "context": "." } },
      "port": 8000,
      "env": { "APP_ENV": "production" }
    }
  ],
  "connections": [
    { "from": "app-net", "to": "app-lb" },
    { "from": "app-net", "to": "api" },
    { "from": "app-net", "to": "app-db" },
    { "from": "app-lb",  "to": "api", "health_check_path": "/health" },
    { "from": "api",     "to": "app-db" }
  ]
}
```

What this does: `Network → *` puts everything in the same VPC; `LoadBalancer → Container` exposes the API behind the ALB; `Container → Database` opens the security group and auto-injects connection env vars (`DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`) into the Container.

**On the spawned-managed AWS account.** When you `spawned init <name>` without `--aws-account`, the backend stamps an `ImportedNetwork` component into your seeded schema (referencing the shared VPC) instead of a fresh `Network`. Don't delete it; reference it by name from your other components' connections. The seed `infra.json` is written into your cwd; open it after `init` to see exactly what defaults were applied. For an existing project, run `spawned get <project> --infra-json` to see the current state.

### AWS: persistent storage (Container + Volume)

ECS Fargate disk is ephemeral. For SQLite, uploads, or any local state, attach a Volume (rendered as EFS).

```json
{
  "version": "2.0",
  "platform": "aws",
  "config": {},
  "components": [
    { "type": "Network", "name": "app-net" },
    { "type": "Volume", "name": "app-data" },
    {
      "type": "Container", "name": "app",
      "source": { "image": "myapp:latest" },
      "port": 8080
    }
  ],
  "connections": [
    { "from": "app-net", "to": "app" },
    { "from": "app-net", "to": "app-data" },
    { "from": "app", "to": "app-data", "mount_path": "/var/data" }
  ]
}
```

The Volume needs to live in the same Network as its consumer (EFS lives in VPC subnets). The `mount_path` on the `Container → Volume` connection is where the volume lands inside the container.

### Kubernetes: Container + Volume

```json
{
  "version": "2.0",
  "platform": "kubernetes",
  "config": { "namespace": "myapp" },
  "components": [
    { "type": "Volume", "name": "data", "size": "1Gi" },
    {
      "type": "Container", "name": "web",
      "image": "nginx:latest",
      "ports": [{ "name": "http", "port": 80, "protocol": "http" }]
    }
  ],
  "connections": [
    { "from": "web", "to": "data", "mount_path": "/usr/share/nginx/html" }
  ]
}
```

K8s shape differences worth knowing: `image` is top-level (no `source` wrapper); `ports` is a list of port objects with `name` / `port` / `protocol`; a Container with a Volume mount is promoted to a StatefulSet automatically.

To expose a K8s Container externally: add an `ImportedDomain` and a `Domain → Container` connection with a `host` param. With `protocol: "http"` the connection emits an Ingress; with `protocol: "tcp"` it emits a LoadBalancer Service. The cluster must already have a matching ingress controller; that's a property of the cluster, not the schema. If the project was created with `--cluster <name-or-id>`, the cluster's defaults (ingress class, TLS, DNS auto-publish) are already in `target.config`.

### Azure

Azure components register the same way. The simplest shape is an Azure Container Apps workload:

```json
{
  "version": "2.0",
  "platform": "azure",
  "config": { "resource_group": "myapp-rg", "location": "westeurope" },
  "components": [
    { "type": "Container", "name": "nginx", "image": "nginx:latest", "port": 80 }
  ]
}
```

For Azure component fields, connection params, secret/vault wiring, and managed domains, run `spawned schema` and pick the Azure section.

---

## Common pitfalls

| Issue | Where | Note |
|-------|-------|------|
| Old `1.0`-style schema | Anywhere | If your example uses `version: "1.0"`, a `values` wrapper around component fields, an `id`/`provider` per component, or `$ref`/`$get`/`$connection`/`$mount` reference objects, it's outdated. Use the `2.0` shape and the `connections` array. |
| `imports` field | Old skill mentions an autoinjected `imports` array. The runtime does not autoinject one. On spawned-managed AWS, an `ImportedNetwork` component is seeded into `components[]` instead. |
| Missing `Network` connection | AWS | Containers, Databases, Volumes, LoadBalancers, Functions on AWS must each have a `Network → <component>` connection. Without it, they have nowhere to live. |
| `health_check_path` | AWS LoadBalancer → Container | Default is `/`. If your app doesn't answer 200 on `/`, set this explicitly to a health endpoint your app serves (e.g. `/health`). Mismatched health checks fail silently for ~5 minutes during deploy. |
| `ports` on the Dockerfile | AWS / K8s Container with build | The port your app listens on must match the `EXPOSE` line. Mismatches cause health-check failures. |
| K8s `image` vs AWS `source.image` | Platform-specific | On K8s, image is top-level: `"image": "nginx:latest"`. On AWS, image is nested: `"source": { "image": "nginx:latest" }`. Don't cross the streams. |
| `--aws-account` on non-AWS | CLI | Rejected by `spawned init`. AWS account binding only applies to `--platform aws`. |
| `--cluster` on non-K8s | CLI | Rejected by `spawned init`. Cluster picks only apply to `--platform kubernetes`. |
| Authoring `infra.json` from scratch | Workflow | Skips the backend's seed defaults (resource group / namespace / shared network). For a new project, run `spawned init <name>` in the target directory — it writes the seed `infra.json` for you. For an existing project, pull with `spawned get <project> --infra-json > infra.json`. |
| Targeting a team org | CLI | Encode the org in the positional ref: `spawned init team/app`, `spawned apply team/app`. For org-scoped listings, pass org as a positional: `spawned list team`, `spawned cluster list team`. |

Run `spawned validate --infra-json infra.json` (or with a project for import resolution) to catch issues before applying.

**If commands, flags, or schema fields don't match what's described here, the skill may be out of date.** Tell the human to update it. For a plugin install:

```
/plugin marketplace update spawned
/plugin update spawned@spawned
```

The latest guide is always at https://spawned.ai/SKILL.md.

---

## Monitoring

`spawned apply` streams workflow logs live on AWS / Azure projects (terraform plan/apply, image build, deploy steps). On Kubernetes, `apply` exits after pushing manifests; there is nothing for spawned to stream from your cluster.

If you used `--detach`, or if you want to inspect a running project:

| Status | Meaning |
|--------|---------|
| `pending` / `created` | Project exists, no apply yet |
| `in_progress` | Workflow is running (terraform, build, deploy) |
| `running` | Live and healthy (may take ~60s after this for URL to respond) |
| `failed` | Last workflow run failed; pull logs to find the cause |

### Inspecting a running project

```bash
spawned get <project>                                 # status + URL + org
spawned builds <project>                              # active build runs
spawned builds <project> --all                        # full history (includes completed/failed)
spawned builds <project> --logs <run-id>              # logs for one specific run
spawned logs <project> <component>                    # stream runtime logs (seeds with recent, then tails)
spawned logs <project> <component> --since <ISO ts>   # resume from a checkpoint
```

For human-facing inspection (charts, recent activity, full project state), point them at <https://spawned.ai/dashboard>.

### Common signals

| Signal | What it means | Action |
|--------|---------------|--------|
| Status `failed` | Last workflow failed | Run `spawned builds <project> --all`, get the failed run id, then `--logs <run-id>` to read the error |
| Status flipped from `running` | A redeploy broke a healthy project | Check `spawned builds --all` for the offending run |
| Build streak failing | New code can't deploy | Surface the build error and the offending commit; check the Dockerfile and `EXPOSE` port |
| Repeated 5xx in runtime logs | App crashed or is misconfigured | `spawned logs <project> <component>` and look for stack traces |
| Health check failures | Container marked unhealthy by the LB | Check the app's health endpoint matches `health_check_path` on the connection; check CPU/memory limits |

On Kubernetes, runtime logs come from your own cluster, not spawned. `spawned logs` is unavailable for K8s projects; use `kubectl logs` / `kubectl get` against your cluster instead.

---

## Organizations

Spawned projects belong to an organization (your personal org or a team org). The CLI infers the org from the positional ref on every command. When a ref omits the org, the user's personal org is used.

```bash
spawned org list                  # list organizations you belong to
```

How to target a specific org per command:

| Surface | Form |
|---------|------|
| Project commands (`init`, `apply`, `get`, `logs`, `export`, `validate`, `upload`, `builds`, `schema`) | `org/name` positional ref. Use `/name` to force the personal org when a name collides with an org slug. |
| Org-scoped listings (`list`, `cluster list`) | `[org]` positional. Omit for personal org. |

`spawned init`, `spawned apply`, and `spawned upload` print a one-line banner (`Applying to <name> (org: <slug>)`) before mutating, so you can confirm the target.

`spawned config` persists only the default AWS account, not an org (see the `# CLI defaults` block in [All commands](#all-commands)).

---

## API keys

For non-interactive use (CI, scripts): long-lived tokens stored on the spawned account, separate from your interactive login.

```bash
spawned apikeys create "my-ci-key"     # creates a key; full value is shown ONCE
spawned apikeys list                   # list keys with prefixes (no full value)
spawned apikeys revoke <key-id>        # permanently revoke a key
```

The full key string is printed only at creation time. Save it immediately into the secret manager you'll consume it from.

---

## All commands

```bash
# Discovery
spawned schema                                            # full infrastructure schema reference
spawned schema <project>                                  # filtered to the project's AWS account
spawned schema --component <Name>                         # detail for one component
spawned schema --platform <aws|azure|kubernetes>          # narrow to one platform
spawned list                                              # list projects in personal org
spawned list <org>                                        # list projects in a team org
spawned get <project>                                     # status + URL
spawned get <project> --infra-json                        # view current infra.json
spawned validate                                          # validate ./infra.json standalone
spawned validate --infra-json <path>                      # explicit file path
spawned validate <project>                                # validate against project context
cat infra.json | spawned validate                         # stdin form
spawned repos                                             # list GitHub repos accessible to the spawned GitHub App
spawned repos --installations                             # list installations only
spawned repos --installation <id>                         # repos under one installation

# Project lifecycle
spawned init <name>                                       # create in personal org (AWS, default)
spawned init /<name>                                      # explicit personal org (escape from org/name parsing)
spawned init <org>/<name>                                 # create in a team org
spawned init <name> --aws-account <id>                    # bring-your-own AWS
spawned init <name> --shared                              # mark as shared infrastructure
spawned init <name> --platform azure                      # Azure
spawned init <name> --platform kubernetes                 # Kubernetes (bring-your-own-cluster)
spawned init <name> --platform kubernetes --cluster <ref> # Kubernetes with a registered cluster's defaults
spawned apply <project>                                   # apply ./infra.json and stream workflow logs
spawned apply <project> --infra-json <path>               # explicit file path
spawned apply <project> --detach                          # apply in background
cat infra.json | spawned apply <project>                  # stdin form
spawned export <project>                                  # download generated project files

# Observability
spawned logs <project> <component>                        # stream runtime logs
spawned logs <project> <component> --since <ISO ts>       # resume from a checkpoint
spawned builds <project>                                  # active builds
spawned builds <project> --all                            # all builds including completed/failed
spawned builds <project> --logs <run-id>                  # logs for a specific build run

# Files
spawned upload <project> --component <name> --key <s3-key> --file <local-path>
                                                          # upload to a Bucket component
                                                          # optional: --content-type <mime>

# Organizations & clusters
spawned org list                                          # list orgs you belong to
spawned cluster list                                      # K8s clusters in your personal org
spawned cluster list <org>                                # K8s clusters in a team org

# AWS accounts (BYOC)
spawned accounts list
spawned accounts connect [--name "Display name"]
spawned accounts configure <account-id> --role-arn <arn> [--name "Display name"]
spawned accounts domain set <account-id> --subdomain <name>
spawned accounts domain delete <account-id>

# API keys
spawned apikeys create <name>
spawned apikeys list
spawned apikeys revoke <key-id>

# CLI defaults
spawned config set account <value>        # only key supported is `account` (default --aws-account)
spawned config get account
spawned config unset account

# Auth
spawned login                              # alias: signin
spawned logout                             # alias: signout
```

---

## AWS account connection (bring your own cloud)

```bash
spawned accounts connect --name "My AWS"              # get CloudFormation URL + account ID
# → open the URL in a browser, create the stack, copy the Role ARN from the Outputs tab
spawned accounts configure <id> --role-arn <arn>      # complete setup
spawned accounts list                                 # verify status=active
spawned accounts domain set <id> --subdomain myapp    # optional: set a custom subdomain
spawned accounts domain delete <id>                   # remove that subdomain
spawned init <name> --aws-account <id>                # deploy into your account
```

To avoid passing `--aws-account` every time, persist a default:

```bash
spawned config set account <id>                       # subsequent `spawned init` uses this account
spawned config get account
spawned config unset account
```

The stack grants spawned an IAM role that can provision into your account. On `spawned init --aws-account <id>`, the seed schema is plain; no `ImportedNetwork` is stamped because your account doesn't share a VPC with anyone. Add your own `Network` component if you want one.

---

## Tips for Agents

### Context awareness
- **Figure out what your human needs first.** Don't assume code deployment. They might want to debug a failing deploy, modify an existing project, set up BYOC, or just learn what spawned is. Refer to [What Should You Do?](#what-should-you-do).
- **Ask questions when you're unsure.** Better to ask "Are you deploying new or modifying existing?" than to guess wrong and overwrite their infra.
- **Save this skill file locally.** Persist `SKILL.md` at `~/.config/spawned/skills/SKILL.md` so you can reference it without re-fetching.

### When working with the CLI
- **For a new project, edit the seed `infra.json`** that `spawned init <name>` drops into the current directory. For an existing project, pull the current state with `spawned get <project> --infra-json > infra.json`. Authoring from scratch skips the platform-appropriate defaults.
- **Always validate before applying.** `spawned validate <project>` (or just `spawned validate` for standalone) catches errors that would otherwise burn 5–15 minutes of workflow time.
- **Don't guess project names.** Run `spawned list` (or `spawned list <org>` for a team org) first; use actual names from the response.
- **Disambiguate with `org/name`** when the same project name exists in multiple orgs you belong to. Use `/name` to force your personal org if a name collides with an org slug.
- **Check current state before re-applying.** `spawned get <project>` and `spawned get <project> --infra-json` show you what's deployed before you change anything.
- **Run `spawned schema` for unfamiliar fields.** It is authoritative; examples in this file show the model, not the full catalog. `spawned schema --component <Name>` is the right call for a single component's fields. Add `--platform <name>` to narrow further, or pass a project ref to filter by what's enabled on that project's AWS account.
- **Ask before destructive actions.** Re-applying over a `running` project, tearing down a Bucket/Database, or disconnecting a cloud account all carry blast radius. The CLI prints a `Applying to X (org: Y)` banner before mutating — surface that to the human and confirm.
- **Don't manually delete the auto-seeded `ImportedNetwork`** on the spawned-managed AWS account. Removing it leaves your other components without a VPC.

### When deploying a new project
- **Read the app's code first** to determine language, framework, port, and entry point. Don't write `infra.json` blindly.
- **Ensure a Dockerfile exists** in the repo before you push a Container with `source.build`. For Next.js, set `output: "standalone"` in `next.config`.
- **Match the Container port to the Dockerfile `EXPOSE` line.** Mismatched ports cause health-check failures.
- **For local-state apps (SQLite, file uploads), add a Volume** up front. ECS Fargate disk is ephemeral.
- **Order components however you like**: references in the `connections` array are resolved by `name`, not by file order. (Earlier orderings were required by the old `1.0` schema; not anymore.)

### When modifying an existing project
- **`spawned get <project> --infra-json` first.** Edit from the current state, not from this doc's examples.
- **Validate after editing**, before applying.
- **Leave the seed components in place** (e.g. `ImportedNetwork` on spawned-managed AWS, the resource_group on Azure target.config, the namespace on K8s target.config). Removing them breaks the project.
