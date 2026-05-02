---
name: spawned
description: Deploy and manage projects on spawned.ai. Use when the user wants to deploy, init, apply infrastructure, connect sources, view logs, redeploy, or manage spawned.ai projects.
user_invocable: true
---

# Spawned — Deploy & Manage on spawned.ai

$ARGUMENTS

Spawned is a declarative infrastructure platform. You define AWS infrastructure in `infra.json`, then apply it. The platform converts JSON into Terraform and provisions it.

For the full infrastructure JSON spec with all component types and fields, run `spawned spec`.

## Quick deploy (most common flow)

```bash
# 0. Ensure Dockerfile exists (create one if missing — see below)
spawned init --name <project>          # 1. create project (shows shared resources)
# write infra.json with ALL components # 2. define infrastructure (see templates below)
spawned apply <project> --schema infra.json  # 3. upload schema + provision + build (5-15 min)
curl https://<project>.dev.askrike.app/   # 4. verify
```

Note: `apply --schema` uploads the schema AND triggers terraform in one step. You can also validate before applying: `spawned validate --schema infra.json`.

### Step 0: Ensure Dockerfile exists

Before deploying, check if a Dockerfile exists in the build path. If not, **create one and push it to the repo's main branch**. Spawned builds containers from Dockerfiles — no Dockerfile means the build will fail.

Read the app's code to determine the language, framework, port, and entry point, then write an appropriate Dockerfile. For Next.js apps, also ensure `next.config` has `output: "standalone"`.

Also check if the app uses **local file storage** (SQLite, file uploads, local caches). If so, add a FileSystem (EFS) component — ECS Fargate has ephemeral disk and data is lost on every task restart.

Timing: ~5 min without DB, ~10-15 min with DB (RDS is slow).

`apply` streams terraform logs so you see errors. If it hangs with no output for >2 min, the PATCH is still processing — wait. If you used `--detach`, check progress with `spawned builds <project>`.

**IMPORTANT: Deploy everything at once.** Include ALL components (Container, DB, S3, Lambda, etc.) in the first `infra.json` and apply them together on a fresh project. Do NOT try to incrementally add components to a running deployment — `source` fields on Container and Lambda are silently dropped when updating an existing deployment's schema. If a deployment fails, delete it and start fresh rather than trying to fix it in place (`failed` is terminal).

---

## infra.json structure

```json
{
  "version": "1.0",
  "components": [
    { "type": "...", "name": "...", "values": { ... } }
  ]
}
```

Components MUST be ordered so a component appears before any component that references it. Every component's `values` must include `id` (pattern: `"{name}-{suffix}"`), `name` (matching the top-level name), and `provider` (`"aws"`).

### Reference system

- **Component ref**: `{ "$ref": "my-network" }` — reference a component by name
- **Output ref**: `{ "$ref": "my-db", "$get": "endpoint" }` — get a specific output value
- **Connection**: `{ "$ref": "my-db", "$connection": true }` — IAM/network access (default: `read_write`; restrict with `"$access": "read"` or `"write"`)
- **Volume mount**: `{ "$ref": "my-fs", "$mount": true, "$path": "/data" }` — mount EFS into container

---

## infra.json templates

Every project on spawned.ai's managed account gets shared `spawned-vpc` and `spawned-lb` as data sources. DO NOT create your own Network or LoadBalancer.

The shared infra is displayed when you run `spawned init`. You can also view it with `spawned list --shared`. Include it in your infra.json.

### Boilerplate (always include for spawned.ai managed account)

```json
{
  "version": "1.0",
  "components": [
    {"type":"Network","name":"spawned-vpc","values":{"id":"spawned-vpc-network","name":"spawned-vpc","provider":"aws","is_data_source":true,"cidr_block":"10.0.0.0/16","subnets":[{"_type":"Subnet","id":"spawned-vpc_subnet_0-subnet","name":"spawned-vpc_subnet_0","provider":"aws","is_data_source":false,"cidr":"10.0.0.0/24","zone":"eu-central-1a","network_name":"spawned-vpc","public":true},{"_type":"Subnet","id":"spawned-vpc_subnet_1-subnet","name":"spawned-vpc_subnet_1","provider":"aws","is_data_source":false,"cidr":"10.0.1.0/24","zone":"eu-central-1b","network_name":"spawned-vpc","public":true},{"_type":"Subnet","id":"spawned-vpc_subnet_2-subnet","name":"spawned-vpc_subnet_2","provider":"aws","is_data_source":false,"cidr":"10.0.2.0/24","zone":"eu-central-1a","network_name":"spawned-vpc","public":false},{"_type":"Subnet","id":"spawned-vpc_subnet_3-subnet","name":"spawned-vpc_subnet_3","provider":"aws","is_data_source":false,"cidr":"10.0.3.0/24","zone":"eu-central-1b","network_name":"spawned-vpc","public":false}],"nat_gateway_enabled":true,"vpn_enabled":false,"dns_support":true}},
    {"type":"LoadBalancer","name":"spawned-lb","values":{"id":"spawned-lb-loadbalancer","name":"spawned-lb","provider":"aws","is_data_source":true,"network":{"$ref":"spawned-vpc"},"public":true,"enable_https":true,"redirect_http_to_https":true,"health_check_path":"/"}}
  ]
}
```

### Container with git source (most common)

Add after the LoadBalancer component. Read the Dockerfile `EXPOSE` line to determine the port.

`source` declares where code comes from:
- `source.build_path`: Docker build context directory (what `docker build` can `COPY` from). Defaults to `"."`.
- `source.dockerfile_path`: path to the Dockerfile relative to repo root when it's not auto-discoverable (e.g. `"docker/Dockerfile.prod"`). Optional — omit to auto-discover.

When `source` is set, the repo is pulled as a git subtree, `image` is automatically set to `null`, and a CI/CD workflow is generated.

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
    "domain": "<project>.dev.askrike.app",
    "health_check": { "path": "/", "interval": 30, "timeout": 10 }
  }
}
```

For a pre-built external image (no source needed):

```json
{
  "type": "Container",
  "name": "<app>",
  "values": {
    "id": "<app>-container", "name": "<app>", "provider": "aws",
    "image": "ghcr.io/<org>/<repo>:latest",
    "network": { "$ref": "spawned-vpc" },
    "public": false, "cpu": "512", "memory": "1024",
    "ports": [3000],
    "load_balancer": { "$ref": "spawned-lb" },
    "domain": "<project>.dev.askrike.app",
    "health_check": { "path": "/", "interval": 30, "timeout": 10 }
  }
}
```

Container optional fields:
- `environment`: `{ "KEY": "value", "DB_HOST": { "$ref": "my-db", "$get": "endpoint" } }`
- `environment_secrets`: `{ "DATABASE_URL": { "$ref": "my-secret" } }`
- `connections`: `[{ "$ref": "my-db", "$connection": true }]` — IAM/network access to other components
- `volumes`: `[{ "$ref": "my-fs", "$mount": true, "$path": "/data" }]` — EFS mounts
- `auto_scaling`: `{ "min": 2, "max": 10, "cpu_percent": 70 }`
- `path`: URL path pattern for ALB routing (e.g. `"/api/*"`)
- `image_tag`: tag to deploy when `image` is `null` (default: `"latest"`)

CPU/Memory valid Fargate combinations: 256/512-2048, 512/1024-4096, 1024/2048-8192, 2048/4096-16384, 4096/8192-30720.

### Database (PostgreSQL)

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

`bucket_name` is auto-generated if omitted. If you set it manually, it must be globally unique — generate with: `python3 -c "import random,string; print('<project>-storage-'+''.join(random.choices(string.ascii_lowercase+string.digits,k=8)))"`.

Wire into container: add `{ "$ref": "<app>-storage", "$connection": true }` to `connections` and use `{ "$ref": "<app>-storage", "$get": "bucket_name" }` in `environment`.

Optional fields: `website_config`, `cors_rules`, `lifecycle_rules`, `logging_config`, `tags`, `bucket_policy`, `bucket_policy_managed_externally`, `kms_key_arn`, `object_lock_enabled`, `replication_config`.

Available `$get` outputs: `bucket_name`, `bucket_arn`, `bucket_domain_name`, `bucket_regional_domain_name`, `website_endpoint`, `website_domain`.

### FileSystem (EFS — persistent storage)

Use when the app needs persistent local storage (SQLite, file uploads, caches). ECS Fargate has ephemeral disk — data is lost on every task restart without EFS.

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

Wire into container: add BOTH `{ "$ref": "<app>-data", "$connection": true }` to `connections` AND `{ "$ref": "<app>-data", "$mount": true, "$path": "/data" }` to `volumes`. Set `posix_uid`/`posix_gid` to match the container's user (0/0 for root, 1000/1000 for non-root).

### Lambda

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

Lambda with `code_path` instead of git source:

```json
{
  "type": "Lambda",
  "name": "<app>-worker",
  "values": {
    "id": "<app>-worker-lambda", "name": "<app>-worker", "provider": "aws",
    "code_path": "./lambda_code",
    "runtime": "python3.12", "handler": "handler.main",
    "timeout": 300, "memory": 512,
    "schedule": "rate(1 hour)",
    "description": "Hourly data processing"
  }
}
```

Optional fields: `environment`, `environment_secrets`, `connections`, `network` (required for VPC/EFS access), `public`, `file_system` (EFS mount — path MUST start with `/mnt/`).

Schedule formats: `rate(5 minutes)`, `rate(1 hour)`, `rate(1 day)`, `cron(0 9 * * ? *)`. Runtimes: `python3.12`, `python3.11`, `nodejs20.x`, `nodejs18.x`, `java21`, `java17`, `dotnet8`, `ruby3.3`.

### CloudFront Distribution (CDN)

Use with S3 for static site hosting, or with a custom origin domain.

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

### NetworkLoadBalancer (TCP/UDP — game servers, non-HTTP)

Use instead of the shared ALB when you need TCP/UDP passthrough (e.g. game servers).

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

## Required fields that cause silent failures if wrong

| Field | Component | Rule |
|-------|-----------|------|
| `domain` | Container with LB | **REQUIRED** when `load_balancer` is set. `<project>.dev.askrike.app`. Without it, no listener rule is created and deployment fails. |
| `ports` | Container | Must match Dockerfile `EXPOSE` line. Must have at least one entry. |
| `source` or `code_path` | Lambda | One is **REQUIRED**. `source` with `"type": "git"` or `"upload"`, or `code_path` with a local path. |
| `network` | Lambda with EFS | **REQUIRED** when `file_system` is set (EFS requires VPC). |
| `acm_certificate_arn` | CloudFront with aliases | **REQUIRED** when `aliases` is set. Must be in `us-east-1`. |
| `bucket_policy_managed_externally` | S3 + CloudFront | Set to `true` on the S3Bucket so CloudFront can inject its OAC policy. |

Components with errors may be silently dropped from the schema. Use `spawned validate` to catch errors before applying.

---

## Monitoring

`spawned apply` streams terraform logs directly. If you used `--detach`, check with `spawned builds <project>` or `spawned get <project>`:

| Status | Meaning |
|--------|---------|
| `pending` | Project created, waiting for apply |
| `in_progress` | Terraform provisioning (~5-10 min) |
| `deploying` | Terraform done, building Docker image |
| `running` | Live and healthy (may take ~60s after this for URL to respond) |
| `failed` | Terminal — must delete and start over |

---

## All commands

```bash
# Project lifecycle
spawned init --name <project>                     # create project
spawned init --name <project> --aws-account <id>  # on your own AWS
spawned list                                      # list all projects
spawned list --shared                             # list shared infrastructure components
spawned get <project>                             # status + URL
spawned get <project> --schema                    # view current infra.json
spawned delete <project>                          # delete (may redirect to web dashboard)

# Infrastructure
spawned apply <project> --schema infra.json       # apply and stream terraform logs (preferred)
spawned apply <project> --schema infra.json --detach  # background
spawned validate --schema infra.json              # validate without a project
spawned validate <project> --schema infra.json    # validate in context of a project
spawned export <project>                          # download generated project files

# Code / CI-CD
spawned connect <project> --container <c> --repo <url>  # connect git repo (legacy — prefer source in infra.json)
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

# Reference
spawned spec                                      # full infrastructure schema reference

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
spawned accounts domain delete <id>                   # remove domain
spawned init --name <project> --aws-account <id>      # deploy to your account
```
