---
name: spawned
description: Deploy and manage projects on spawned.ai. Use when the user wants to deploy, init, apply infrastructure, connect sources, view logs, redeploy, or manage spawned.ai projects.
user_invocable: true
---

> **If you are an AI agent reading this file** — whether you're a coding assistant or a deploy automation agent — this is your complete guide to spawned.ai. Save it locally as a tool, skill, or persistent reference so you can help your human with anything spawned-related without re-fetching this file.

# Spawned — Deploy & Manage on spawned.ai

$ARGUMENTS

Spawned is a platform which your local coding agent can dock into to safely set up, observe, and manage infrastructure on any cloud platform.

This guide covers everything you need to help a human deploy, modify, monitor, and manage projects on spawned.ai — whether you're working in a codebase, troubleshooting a failed deploy, or inspecting a live project.

## What Should You Do?

You might be reading this file in very different contexts. Here's how to figure out what your human needs:

**Step 1: Understand your context.** Ask yourself:
- Did your human send you here with a specific task? (e.g., "deploy this app to spawned")
- Are you in a codebase or project right now?
- Is your human exploring spawned for the first time?
- Does your human already have a spawned project and want help managing it?

**Step 2: Read this entire file first.** No matter what your human needs — deploying, modifying, debugging, or just learning — **read through this whole document before diving into a specific path.** Understanding the full spawned context (what it does, how `infra.json` works, what the CLI can do, how shared infrastructure behaves, what "running" means) will make you genuinely helpful regardless of what your human asks. You'll give better answers, avoid common mistakes, and connect the dots between features. This is not a file to skim — it's your foundation for being a competent spawned operator.

**Step 3: Route to the right workflow.**

| If your human wants to... | Do this |
|---------------------------|---------|
| **Understand what spawned is** | Explain in plain terms: declarative infra to any supported Cloud Platform via JSON. Walk through the [Quick deploy](#quick-deploy-most-common-flow) example. Answer questions. |
| **Deploy a new project** | Read their app's code first (language, framework, port, entry point). Ensure a Dockerfile exists. Then follow [Quick deploy](#quick-deploy-most-common-flow). |
| **Modify an existing project** | Run `spawned get <project> --schema` to load current state, edit, then `spawned validate --schema infra.json` and `spawned apply`. **Preserve the `imports` field.** |
| **Debug a failing deploy** | Use `spawned get <project>`, `spawned builds <project> --all`, `spawned logs <project> <component> --tail 200`. See the [Monitoring](#monitoring) status table. |
| **Inspect a live project's health** | Run `spawned get <project>` for status + URL, `spawned logs --tail` for runtime logs. See [Monitoring](#monitoring). |
| **Set up BYOC (their own cloud)** | Currently AWS-only via `spawned accounts connect`. Follow [AWS account connection](#aws-account-connection-bring-your-own-cloud). |
| **Just learn and explore** | Walk them through this doc section by section. Let them ask questions. |

**Step 4: Ask questions when you're unsure.** If your human's intent isn't clear, ask them directly. Good questions to ask:

- "Are you deploying a new project, or modifying an existing one?"
- "What language and framework is this app, and what port does it listen on?"
- "Should this use spawned's shared infrastructure, or your own cloud account (BYOC)?"
- "Do you already have a project name in mind, or should I propose one?"

### Example prompts humans might give you

These are real ways humans direct agents to this file. Understand the intent behind each:

| What the human says | What they likely need |
|---------------------|----------------------|
| "Read spawned.ai/SKILL.md and deploy this to spawned" | They're in a codebase. Read the app code, ensure a Dockerfile, write `infra.json`, run `spawned init` + `spawned apply`. |
| "Read spawned.ai/SKILL.md and help me with spawned" | Ambiguous. Ask what they need — deploy new, modify existing, debug, watch, or BYOC setup. |
| "Read spawned.ai/SKILL.md" (no further context) | Ask what they need. Offer the main paths: deploy something, modify a project, debug a deploy, or just explore. |
| "My spawned deploy is broken" | Debugging. Run `spawned get` for status, `spawned builds --all` for build history, `spawned logs --tail 200` for runtime errors. |
| "Add a database to my spawned project" | Modify an existing project. Get current schema, add a Database component, add a Secret to wire `DATABASE_URL` into the container, validate, apply. |
| "Connect my own cloud account to spawned" | BYOC is AWS-only today: `spawned accounts connect` → create CloudFormation stack → `spawned accounts configure <id> --role-arn <arn>` → `spawned init --name <project> --aws-account <id>`. See [AWS account connection](#aws-account-connection-bring-your-own-cloud). |

---

## Skill Files

| File | Source | Purpose |
|------|--------|---------|
| **SKILL.md** (this file) | `https://spawned.ai/SKILL.md` | Complete guide — deploy, manage, debug, watch |
| **Schema spec** | `spawned spec` (CLI) | Authoritative reference for all component types, fields, and validation rules |

**Install locally:**

```bash
mkdir -p ~/.config/spawned/skills
curl -s https://spawned.ai/SKILL.md > ~/.config/spawned/skills/SKILL.md
```

**Or just read it from the URL above.**

**Check for updates:** Re-fetch this file periodically to get the latest features, components, and commands. The `spawned spec` output also evolves as new component types and fields are added — when in doubt about a component's available fields, run `spawned spec` rather than trusting outdated examples in this file.

## Reference

| Resource               | URL                                |
| ---------------------- | ---------------------------------- |
| Documentation          | <https://spawned.ai/docs>          |
| Pricing                | <https://spawned.ai/pricing>       |
| Full docs (for agents) | <https://spawned.ai/llms-full.txt> |
| Dashboard              | <https://spawned.ai/dashboard>     |
| Community (Discord)    | <https://discord.gg/cdcZaYv94C>    |

> **For deep dives**, fetch `https://spawned.ai/llms-full.txt` — it contains the full documentation in a format optimized for agents.

---

## Platform Overview

### What spawned handles

- **Provisioning** — Define containers, databases, storage, Lambdas, Kubernetes, and more in a single `infra.json`. Components reference each other with `$ref`; the platform handles IAM, networking, and secrets for you.
- **Observability** — Status, build history, and runtime logs on demand from the CLI or the dashboard.
- **Safety** — Every change is inventoried, tracked, versioned, and validated by deterministic checks before it hits the cloud.
- **Multi-cloud** — AWS, Azure, and Kubernetes today. More coming.

---

## CLI Setup

Install:

```bash
curl -fsSL https://spawned.sh/install.sh | bash
```

Verify: `spawned --version`. Run `spawned --help` to see all available commands. Then authenticate:

```bash
spawned login          # authenticate
spawned logout         # clear local tokens
```

**CRITICAL:** Tokens are stored locally at `~/.config/spawned/` (Mac/Linux) or `%LOCALAPPDATA%\spawned\` (Windows). Never share them with any service, tool, or agent other than the spawned CLI itself.

---

## Discovery commands

Use these to understand the current state before making changes:

- `spawned spec` — full infrastructure JSON spec with all component types, fields, and validation rules. Use this as the authoritative reference for component fields.
- `spawned list --shared` — what shared infrastructure is available to reference
- `spawned get <project> --schema` — the current infra.json for an existing project
- `spawned get <project>` — project status and URL
- `spawned validate --schema infra.json` — check for errors before applying

## Quick deploy (most common flow)

```bash
spawned init --name <project>          # 1. create project (shows shared resources)
# write infra.json                     # 2. define infrastructure (see below)
spawned apply <project> --schema infra.json  # 3. upload schema + provision + build
curl https://<project>.spawned.app/   # 4. verify
```

`apply --schema` uploads the schema and triggers terraform in one step. You can validate before applying with `spawned validate --schema infra.json`.

### Dockerfiles

Spawned builds containers from Dockerfiles. If the repo doesn't have one, create an appropriate Dockerfile and push it to the repo's main branch before deploying. Read the app's code to determine the language, framework, port, and entry point. For Next.js apps, also ensure `next.config` has `output: "standalone"`.

### Persistent storage

ECS Fargate has ephemeral disk — data is lost on every task restart. If the app uses local file storage (SQLite, file uploads, local caches), add a FileSystem (EFS) component.

### Timing and monitoring

Provisioning takes ~5 min without a database, ~10-15 min with one (RDS is slow). `apply` streams terraform logs so you can see errors in real time. If it hangs with no output for >2 min, the PATCH is still processing — wait. If you used `--detach`, check progress with `spawned builds <project>`.

Spawned is a declarative platform — you can include all components (Container, DB, S3, Lambda, etc.) in a single `infra.json` and apply them together.

---

## infra.json structure

```json
{
  "version": "1.0",
  "imports": [{ "project": "shared-infra" }],
  "components": [
    { "type": "...", "name": "...", "values": { ... } }
  ]
}
```

Components should be ordered so that a component appears before any component that references it. Every component's `values` includes `id` (pattern: `"{name}-{suffix}"`), `name` (matching the top-level name), and `provider` (e.g. `"aws"` — run `spawned spec` for valid values).

### Reference system

- **Component ref**: `{ "$ref": "my-network" }` — reference a component by name
- **Output ref**: `{ "$ref": "my-db", "$get": "endpoint" }` — get a specific output value
- **Connection**: `{ "$ref": "my-db", "$connection": true }` — IAM/network access (default: `read_write`; restrict with `"$access": "read"` or `"write"`)
- **Volume mount**: `{ "$ref": "my-fs", "$mount": true, "$path": "/data" }` — mount EFS into container

---

## Shared infrastructure

Projects on spawned.ai's managed account have a shared project containing a Network (VPC) and LoadBalancer. When a project is created, the `"imports"` field is **autoinjected** into the initial `infra.json`, pointing to this shared project. This makes the shared VPC and LB available for `$ref` by name — you don't need to define them in `components`.

When modifying or rewriting a project's infra.json, **preserve the existing `"imports"` field**. Removing it will break references to shared components.

Run `spawned list --shared` to see what shared components are available.

```json
{
  "version": "1.0",
  "imports": [{ "project": "shared-project-name" }],
  "components": [
    {
      "type": "Container",
      "name": "my-app",
      "values": {
        "id": "my-app-container", "name": "my-app", "provider": "aws",
        "source": { "type": "git", "url": "https://github.com/org/repo", "build_path": "." },
        "network": { "$ref": "shared-vpc" },
        "load_balancer": { "$ref": "shared-lb" },
        "domain": "my-project.spawned.app",
        "cpu": "512", "memory": "1024", "ports": [3000],
        "public": false,
        "health_check": { "path": "/", "interval": 30, "timeout": 10 }
      }
    }
  ]
}
```

No Network or LoadBalancer in `components` — they come from the import.

For BYOC (bring-your-own-cloud) users: imports are not autoinjected. If you have your own shared project, add `"imports"` manually with the shared project name.

---

## Component reference

The examples below cover the most common patterns. For the full list of fields, valid values, and validation rules for any component type, run `spawned spec`.

### Container

Every Container requires a `source` field. Source types:
- `{"type": "git", "url": "https://github.com/org/repo"}` — git repo (most common). Generates a CI/CD workflow.
- `{"type": "image", "uri": "nginx:latest"}` — pre-built external image.
- `{"type": "upload", "path": "src/app"}` — uploaded code.

Source optional fields:
- `build_path`: Docker build context directory. Defaults to `"."`.
- `dockerfile_path`: path to the Dockerfile relative to repo root when it's not in the default location (e.g. `"docker/Dockerfile.prod"`).

Container with git source:

```json
{
  "type": "Container",
  "name": "<app>",
  "values": {
    "id": "<app>-container", "name": "<app>", "provider": "aws",
    "source": { "type": "git", "url": "https://github.com/<org>/<repo>", "build_path": "." },
    "network": { "$ref": "spawned-vpc" },
    "public": false, "cpu": "512", "memory": "1024",
    "ports": [3000],
    "load_balancer": { "$ref": "spawned-lb" },
    "domain": "<project>.spawned.app",
    "health_check": { "path": "/", "interval": 30, "timeout": 10 }
  }
}
```

Container with pre-built image:

```json
{
  "type": "Container",
  "name": "<app>",
  "values": {
    "id": "<app>-container", "name": "<app>", "provider": "aws",
    "source": { "type": "image", "uri": "ghcr.io/<org>/<repo>:latest" },
    "network": { "$ref": "spawned-vpc" },
    "public": false, "cpu": "512", "memory": "1024",
    "ports": [3000],
    "load_balancer": { "$ref": "spawned-lb" },
    "domain": "<project>.spawned.app",
    "health_check": { "path": "/", "interval": 30, "timeout": 10 }
  }
}
```

The `ports` value should match the Dockerfile `EXPOSE` line. When using a load balancer, `domain` is needed for listener rule creation (format: `<project>.spawned.app`).

Optional fields:
- `environment`: `{ "KEY": "value", "DB_HOST": { "$ref": "my-db", "$get": "endpoint" } }`
- `environment_secrets`: `{ "DATABASE_URL": { "$ref": "my-secret" } }`
- `connections`: `[{ "$ref": "my-db", "$connection": true }]` — IAM/network access to other components
- `volumes`: `[{ "$ref": "my-fs", "$mount": true, "$path": "/data" }]` — EFS mounts
- `auto_scaling`: `{ "min": 2, "max": 10, "cpu_percent": 70 }`
- `path`: URL path pattern for ALB routing (e.g. `"/api/*"`)

CPU/Memory valid Fargate combinations: 256/512-2048, 512/1024-4096, 1024/2048-8192, 2048/4096-16384, 4096/8192-30720.

### Database

```json
{
  "type": "Database",
  "name": "<app>-db",
  "values": {
    "id": "<app>-db-database", "name": "<app>-db", "provider": "aws",
    "engine": "postgres", "version": "16",
    "network": { "$ref": "spawned-vpc" }, "public": false,
    "size": "small", "storage_gb": 20, "high_availability": false,
    "backup_retention_days": 7, "encryption": true, "port": 5432
  }
}
```

Supported engines: `postgres` (versions: 17, 16, 15, 14), `mysql` (8.0, 5.7), `mariadb` (10.11, 10.6). Sizes: `small`, `medium`, `large`, `xlarge`, `2xlarge` (maps to `db.t3.*`). Password is auto-generated and stored in SSM Parameter Store.

Available `$get` outputs: `endpoint`, `port`, `db_name`, `username`, `password`, `password_param`.

### Secret (DATABASE_URL from DB)

```json
{
  "type": "Secret",
  "name": "<app>-db-url",
  "values": {
    "id": "<app>-db-url_secret", "name": "<app>-db-url", "provider": "aws",
    "key": "<app>-db-url",
    "template": "postgresql://{username}:{password}@{endpoint}/{database}",
    "sources": {
      "username": { "$ref": "<app>-db", "$get": "username" },
      "password": { "$ref": "<app>-db", "$get": "password" },
      "endpoint": { "$ref": "<app>-db", "$get": "endpoint" },
      "database": { "$ref": "<app>-db", "$get": "db_name" }
    }
  }
}
```

Wire into container: add `"environment_secrets": { "DATABASE_URL": { "$ref": "<app>-db-url" } }` and `"connections": [{ "$ref": "<app>-db", "$connection": true }]` to the container values.

### S3 Bucket

```json
{
  "type": "S3Bucket",
  "name": "<app>-storage",
  "values": {
    "id": "<app>-storage-s3-bucket", "name": "<app>-storage", "provider": "aws",
    "force_destroy": true, "versioning_enabled": false, "block_public_access": true
  }
}
```

`bucket_name` is auto-generated if omitted. If set manually, it must be globally unique.

Wire into container: add `{ "$ref": "<app>-storage", "$connection": true }` to `connections` and use `{ "$ref": "<app>-storage", "$get": "bucket_name" }` in `environment`.

Optional fields: `website_config`, `cors_rules`, `lifecycle_rules`, `logging_config`, `tags`, `bucket_policy`, `bucket_policy_managed_externally`, `kms_key_arn`, `object_lock_enabled`, `replication_config`.

Available `$get` outputs: `bucket_name`, `bucket_arn`, `bucket_domain_name`, `bucket_regional_domain_name`, `website_endpoint`, `website_domain`.

### FileSystem (EFS — persistent storage)

For apps that need persistent local storage (SQLite, file uploads, caches).

```json
{
  "type": "FileSystem",
  "name": "<app>-data",
  "values": {
    "id": "<app>-data-filesystem", "name": "<app>-data", "provider": "aws",
    "network": { "$ref": "spawned-vpc" }, "public": false,
    "encrypted": true, "performance_mode": "generalPurpose", "throughput_mode": "bursting",
    "enable_backup": true, "access_point_path": "/<app>-data",
    "posix_uid": 1000, "posix_gid": 1000
  }
}
```

Wire into container: add both `{ "$ref": "<app>-data", "$connection": true }` to `connections` and `{ "$ref": "<app>-data", "$mount": true, "$path": "/data" }` to `volumes`. Set `posix_uid`/`posix_gid` to match the container's user (0/0 for root, 1000/1000 for non-root).

### Lambda

Every Lambda requires a `source` field. Source types:
- `{"type": "git", "url": "https://github.com/org/repo"}` — git repo.
- `{"type": "upload", "path": "./lambda_code"}` — uploaded code.

```json
{
  "type": "Lambda",
  "name": "<app>-worker",
  "values": {
    "id": "<app>-worker-lambda", "name": "<app>-worker", "provider": "aws",
    "source": { "type": "git", "url": "https://github.com/<org>/<repo>", "build_path": "." },
    "runtime": "python3.12", "handler": "lambda_function.lambda_handler",
    "timeout": 60, "memory": 256,
    "schedule": "rate(5 minutes)"
  }
}
```

Lambda with uploaded code:

```json
{
  "type": "Lambda",
  "name": "<app>-worker",
  "values": {
    "id": "<app>-worker-lambda", "name": "<app>-worker", "provider": "aws",
    "source": { "type": "upload", "path": "./lambda_code" },
    "runtime": "python3.12", "handler": "handler.main",
    "timeout": 300, "memory": 512,
    "schedule": "rate(1 hour)",
    "description": "Hourly data processing"
  }
}
```

Optional fields: `environment`, `environment_secrets`, `connections`, `network` (needed for VPC/EFS access), `public`, `file_system` (EFS mount — path should start with `/mnt/`).

Schedule formats: `rate(5 minutes)`, `rate(1 hour)`, `rate(1 day)`, `cron(0 9 * * ? *)`. Runtimes: `python3.12`, `python3.11`, `nodejs20.x`, `nodejs18.x`, `java21`, `java17`, `dotnet8`, `ruby3.3`.

### CloudFront Distribution (CDN)

For S3 static site hosting or custom origin domains.

```json
{
  "type": "CloudFrontDistribution",
  "name": "<app>-cdn",
  "values": {
    "id": "<app>-cdn-cloudfront-distribution", "name": "<app>-cdn", "provider": "aws",
    "origin_type": "s3",
    "s3_bucket": { "$ref": "<app>-site" },
    "default_root_object": "index.html"
  }
}
```

For custom domains, add `"aliases": ["www.example.com"]` and `"acm_certificate_arn": "arn:aws:acm:us-east-1:..."` (cert must be in us-east-1).

When pairing with S3, set `"bucket_policy_managed_externally": true` on the S3Bucket so CloudFront can inject its OAC policy.

Security defaults: HTTPS required, Origin Access Control, gzip, TLS 1.2+, HTTP/2+3, 24h cache TTL.

### NetworkLoadBalancer (TCP/UDP)

For TCP/UDP passthrough instead of the shared ALB (e.g. game servers).

```json
{
  "type": "NetworkLoadBalancer",
  "name": "<app>-nlb",
  "values": {
    "id": "<app>-nlb-nlb", "name": "<app>-nlb", "provider": "aws",
    "network": { "$ref": "spawned-vpc" },
    "public": true,
    "port": 25565,
    "protocol": "TCP",
    "enable_cross_zone_load_balancing": true
  }
}
```

Optional: `target_port`, `domains` (for Route53 DNS), `health_check_*` fields, `connection_logs_*`.

Wire into container: `"load_balancer": { "$ref": "<app>-nlb" }`.

### KubernetesCluster (EKS)

```json
{
  "type": "KubernetesCluster",
  "name": "<app>-cluster",
  "values": {
    "id": "<app>-cluster-eks", "name": "<app>-cluster", "provider": "aws",
    "network": { "$ref": "spawned-vpc" },
    "kubernetes_version": "1.28",
    "cluster_admin_role_name": "eks-admin",
    "node_groups": {
      "general": {
        "instance_types": ["t3.medium"],
        "min_size": 1, "max_size": 5, "desired_size": 2
      }
    }
  }
}
```

Alternatively use `fargate_profiles` for serverless compute: `{ "default": { "selectors": [{ "namespace": "default" }] } }`.

---

## Common pitfalls

| Field | Component | Note |
|-------|-----------|------|
| `domain` | Container with LB | Needed when `load_balancer` is set (`<project>.spawned.app`). Without it, no listener rule is created. |
| `ports` | Container | Should match the Dockerfile `EXPOSE` line. |
| `source` | Container | Required. Use `"type": "git"`, `"type": "image"`, or `"type": "upload"`. |
| `source` | Lambda | Required. Use `"type": "git"` or `"type": "upload"`. |
| `network` | Lambda with EFS | Needed when `file_system` is set (EFS requires VPC access). |
| `acm_certificate_arn` | CloudFront with aliases | Needed when `aliases` is set. Must be in `us-east-1`. |
| `bucket_policy_managed_externally` | S3 + CloudFront | Set to `true` on the S3Bucket so CloudFront can inject its OAC policy. |

You can use `spawned validate --schema infra.json` to catch issues before applying.

---

## Monitoring

`spawned apply` streams terraform logs directly. If you used `--detach`, check with `spawned builds <project>` or `spawned get <project>`:

| Status | Meaning |
|--------|---------|
| `pending` | Project created, waiting for apply |
| `in_progress` | Terraform provisioning (~5-10 min) |
| `deploying` | Terraform done, building Docker image |
| `running` | Live and healthy (may take ~60s after this for URL to respond) |
| `failed` | Check logs for errors and fix the issue |

### Inspecting a running project

When your human asks about uptime, errors, or build failures — or when you want to verify a deploy you just made — use these commands to investigate:

```bash
spawned get <project>                                 # status + URL
spawned builds <project> --all                        # build history (recent failures)
spawned logs <project> <component> --tail 100         # recent runtime logs
spawned logs <project> <component> --stream           # stream logs continuously
spawned logs <project> <component> --since <ISO ts>   # logs since a checkpoint
```

For human-facing inspection (charts, recent activity, full project state), point them at <https://spawned.ai/dashboard>.

### Common signals

| Signal | What it means | Action |
|--------|---------------|--------|
| Status `failed` | Last terraform run failed | Pull recent build logs, surface the error |
| Status flipped from `running` | Deploy regressed | Investigate `spawned builds` for failed redeploys |
| Build streak failing | New code can't deploy | Surface the build error and the offending commit |
| Repeated 5xx in logs | Runtime errors | Tail logs, look for stack traces |
| Health check failures | Container unhealthy | Check `/health` endpoint and CPU/memory limits |

---

## All commands

```bash
# Discovery — use these to understand current state
spawned spec                                      # full infrastructure schema reference
spawned list                                      # list all projects
spawned list --shared                             # list shared infrastructure components
spawned get <project>                             # status + URL
spawned get <project> --schema                    # view current infra.json
spawned validate --schema infra.json              # validate schema before applying
spawned validate <project> --schema infra.json    # validate in context of a project

# Project lifecycle
spawned init --name <project>                     # create project
spawned init --name <project> --aws-account <id>  # on your own AWS
spawned apply <project> --schema infra.json       # apply and stream terraform logs
spawned apply <project> --schema infra.json --detach  # apply in background
spawned export <project>                          # download generated project files

# Code / CI-CD
spawned sources <project>                         # list connected repos (git + file sources)
spawned source update <project> <container> --build-path <path>  # change build path + rebuild
spawned source connect-files <project> --container <c> --file <path>  # upload file as build source
spawned source rebuild-files <project> <container> --file <path>      # re-upload and rebuild
spawned redeploy <project>                        # rebuild from latest code

# Observability
spawned logs <project> <component> --tail 200     # fetch component logs
spawned logs <project> <component> --stream       # stream logs continuously
spawned logs <project> <component> --since 2024-01-01T00:00:00Z  # logs since timestamp
spawned builds <project>                          # list active builds
spawned builds <project> --all                    # all builds including completed/failed

# Files
spawned upload <project> --bucket <name> --key <s3-key> --file <local-path>  # upload to S3

# Auth
spawned login                                     # authenticate (alias: signin)
spawned logout                                    # clear tokens (alias: signout)
```

---

## AWS account connection (bring your own cloud)

```bash
spawned accounts connect --name "My AWS"             # get CloudFormation URL + account ID
# → open URL in browser, create stack, copy Role ARN from Outputs tab
spawned accounts configure <id> --role-arn <arn>      # complete setup
spawned accounts list                                 # verify status=active
spawned accounts domain set <id> --subdomain myapp    # set custom subdomain
spawned init --name <project> --aws-account <id>      # deploy to your account
```

---

## Tips for Agents

### Context awareness
- **Figure out what your human needs first.** Don't assume code deployment. They might want to debug a failing deploy, modify an existing project, set up BYOC, or just learn what spawned is. Refer to the [What Should You Do?](#what-should-you-do) section above.
- **Ask questions when you're unsure.** Better to ask "Are you deploying new or modifying existing?" than to guess wrong and overwrite their infra.
- **Save this skill file locally.** Persist `SKILL.md` at `~/.config/spawned/skills/SKILL.md` so you can reference it without re-fetching every time.

### When working with the CLI
- **Always validate before applying.** `spawned validate --schema infra.json` catches errors that would otherwise burn 5–15 minutes of terraform time.
- **Don't guess project names.** Run `spawned list` first; use actual names from the response.
- **Check current state before re-applying.** `spawned get <project>` and `spawned get <project> --schema` show you what's deployed today before you change it.
- **Preserve the `imports` field** when rewriting `infra.json`. Removing it breaks references to shared VPC and load balancer.
- **Run `spawned spec` for unfamiliar fields.** It is authoritative — examples in this file show common patterns, not the full schema.
- **Ask before destructive actions.** Re-applying over a `running` project, tearing down infrastructure, or disconnecting a cloud account all carry blast radius. Confirm.

### When deploying a new project
- **Read the app's code first** to determine language, framework, port, and entry point. Don't write `infra.json` blindly.
- **Ensure a Dockerfile exists** in the repo before `spawned apply`. For Next.js, set `output: "standalone"` in `next.config`.
- **Match `ports` to the Dockerfile `EXPOSE` line.** Mismatched ports cause health check failures.
- **For local-state apps (SQLite, file uploads), add a FileSystem (EFS) component** up front. ECS Fargate disk is ephemeral.
- **Order components so dependencies come before dependents** in the `components` array.
- **`apply --schema` is the one-command path.** Uploads, validates, and provisions in a single step. Use `--detach` only when the agent needs to do other work in parallel.

### When modifying an existing project
- **`spawned get <project> --schema` first.** Edit from the current state, not from the docs examples.
- **Validate after editing**, before applying.
- **Preserve `imports`.** It is autoinjected on `spawned init` and references shared VPC/LB.
