---
name: spawned-deploy
description: Deploy and manage projects on spawned.ai. Use when the user wants to deploy, init, apply infrastructure, connect sources, view logs, redeploy, or manage spawned.ai projects.
user_invocable: true
---

# Spawned — Deploy & Manage on spawned.ai

$ARGUMENTS

Spawned is a declarative infrastructure platform. You define AWS infrastructure in `infra.json`, then apply it. The platform converts JSON into Terraform and provisions it.

## Quick deploy (most common flow)

```bash
spawned init --name <project> -y          # 1. create project
# write infra.json                        # 2. define infrastructure (see templates below)
spawned schema update <project> -f infra.json -y  # 3. upload schema
spawned apply <project> --schema infra.json -y --detach  # 4. provision + build
spawned get <project>                     # 5. monitor (pending → in_progress → deploying → running)
curl https://<project>.dev.askrike.app/   # 6. verify
```

Timing: ~5 min without DB, ~10-15 min with DB (RDS is slow).

---

## infra.json templates

Every project gets shared `spawned-vpc` and `spawned-lb` as data sources. DO NOT create your own Network or LoadBalancer.

### Boilerplate (always include)

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

`source` declares where code comes from. `build` declares how to prepare it:
- `source.build_path`: Docker build context directory (what `docker build` can `COPY` from). Defaults to `"."`.
- `build.dockerfile`: path to the Dockerfile when it is not at the root of `build_path` (e.g. `"docker/Dockerfile.prod"`). Optional — omit to auto-discover.

```json
{
  "type": "Container",
  "name": "<app>",
  "values": {
    "id": "<app>-container", "name": "<app>", "provider": "aws",
    "image": null,
    "source": { "type": "git", "url": "https://github.com/<org>/<repo>", "build_path": "." },
    "build": { "dockerfile": "<optional: path/to/Dockerfile>" },
    "network": { "$ref": "spawned-vpc" },
    "public": false, "cpu": "512", "memory": "1024",
    "ports": [3000],
    "load_balancer": { "$ref": "spawned-lb" },
    "domain": "<project>.dev.askrike.app",
    "health_check": { "path": "/", "interval": 30, "timeout": 10 }
  }
}
```

For a pre-built external image (no source/build needed):

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

### Secret (DATABASE_URL from DB)

```json
{
  "type": "Secret",
  "name": "<app>-db-url",
  "values": {
    "id": "<app>-db-url-secret", "name": "<app>-db-url", "provider": "aws",
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
    "bucket_name": "<globally-unique-name>",
    "force_destroy": true, "versioning_enabled": false, "block_public_access": true
  }
}
```

S3 with static site content (source + build):

```json
{
  "type": "S3Bucket",
  "name": "<app>-site",
  "values": {
    "id": "<app>-site-s3-bucket", "name": "<app>-site", "provider": "aws",
    "source": { "type": "git", "url": "https://github.com/<org>/<repo>", "build_path": "." },
    "build": { "commands": ["npm ci", "npm run build"], "output": "dist" },
    "website_config": { "index_document": "index.html", "error_document": "index.html" },
    "block_public_access": false,
    "force_destroy": true
  }
}
```

### Lambda (scheduled)

`source` declares where code comes from. `build` is optional — use it for pip requirements or custom build commands.

```json
{
  "type": "Lambda",
  "name": "<app>-worker",
  "values": {
    "id": "<app>-worker-lambda", "name": "<app>-worker", "provider": "aws",
    "source": { "type": "git", "url": "https://github.com/<org>/<repo>", "build_path": "." },
    "runtime": "nodejs20.x", "handler": "src/index.handler",
    "timeout": 300, "memory": 512,
    "schedule": "rate(2 hours)",
    "network": { "$ref": "spawned-vpc" }, "public": false
  }
}
```

Lambda with pip requirements:

```json
{
  "type": "Lambda",
  "name": "<app>-worker",
  "values": {
    "id": "<app>-worker-lambda", "name": "<app>-worker", "provider": "aws",
    "source": { "type": "upload", "id": "<upload-id>", "path": "src/<app>" },
    "build": { "requirements": "requirements.txt" },
    "runtime": "python3.12", "handler": "lambda_function.lambda_handler",
    "timeout": 60, "memory": 256
  }
}
```

---

## Required fields that cause silent failures if wrong

| Field | Component | Rule |
|-------|-----------|------|
| `domain` | Container with LB | **REQUIRED** when `load_balancer` is set. `<project>.dev.askrike.app`. Without it, no listener rule is created and deployment fails. |
| `image` | Container with source | Set to `null` explicitly when using `source`/`build`. |
| `ports` | Container | Must match Dockerfile `EXPOSE` line. |
| `source` | Lambda | **REQUIRED**. Must have `"type"` (`"git"` or `"upload"`) and type-specific fields. |
| `source.path` | Lambda with upload source | **REQUIRED**. Path where uploaded code lives (e.g. `"src/my-app"`). |
| `build.dockerfile` | Container/Lambda | Only set when code needs a Docker build. Omit to auto-discover for containers. |
| `bucket_name` | S3Bucket | Must be globally unique across all AWS accounts. |
| `version` | Database (postgres) | Use `"16"`. Other versions may be silently rejected. |

Components with errors are silently dropped from the schema with no error message.

### Source and build pattern

`source` and `build` are separate concerns used across Lambda, Container, and S3Bucket:

- **`source`**: Where code comes from. Two types:
  - `{"type": "git", "url": "...", "build_path": "."}` — pulled from git repo
  - `{"type": "upload", "id": "...", "path": "src/..."}` — uploaded by user
- **`build`**: How to prepare the code (optional, omit if code is ready as-is):
  - `{"requirements": "requirements.txt"}` — install pip dependencies (Lambda)
  - `{"commands": ["npm ci", "npm run build"], "output": "dist"}` — run build commands (S3)
  - `{"dockerfile": "Dockerfile.prod"}` — specify Dockerfile (Container/Lambda)

---

## Monitoring

After `spawned apply --detach`, use `spawned get <project>` to track status:

| Status | Meaning |
|--------|---------|
| `pending` | Project created, waiting for apply |
| `in_progress` | Terraform provisioning (~5-10 min) |
| `deploying` | Terraform done, building Docker image |
| `running` | Live and healthy (may take ~60s after this for URL to respond) |
| `failed` | Terminal — must `spawned delete` and start over |

---

## All commands

```bash
# Project lifecycle
spawned init --name <project> -y                  # create project
spawned init --name <project> --aws-account <id>  # on your own AWS
spawned list                                      # list all projects
spawned get <project>                             # status + URL
spawned delete <project> -y                       # delete

# Infrastructure
spawned apply <project> --schema infra.json -y    # apply and stream logs
spawned apply <project> --schema infra.json -y --detach  # apply in background
spawned schema <project>                          # view current schema
spawned schema update <project> -f infra.json -y  # update schema only (no terraform)
spawned export <project>                          # download terraform as zip

# Code / CI-CD
spawned connect <project> --container <c> --repo <url>  # connect git repo (legacy, prefer source in infra.json)
spawned sources <project>                         # list connected repos
spawned redeploy <project> -y                     # rebuild from latest code

# Observability
spawned logs <project>                            # stream deployment logs
spawned logs <project> <component> --tail 200     # component logs
spawned workflows <project>                       # workflow status

# Reference
spawned llm-help                                  # full infrastructure JSON spec

# Auth
spawned login                                     # authenticate
spawned logout
```

---

## AWS account connection (bring your own cloud)

```bash
spawned accounts connect --name "My AWS"          # get CloudFormation URL + account ID
# → open URL in browser, create stack, copy Role ARN from Outputs tab
spawned accounts configure <id> --role-arn <arn>   # complete setup
spawned accounts list                              # verify status=active
spawned init --name <project> --aws-account <id>   # deploy to your account
```
