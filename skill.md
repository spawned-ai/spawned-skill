---
name: spawned
description: Deploy and manage projects on spawned.ai. Use when the user wants to deploy, init, apply infrastructure, connect sources, view logs, redeploy, or manage spawned.ai projects.
user_invocable: true
---

# Spawned — Deploy & Manage on spawned.ai

$ARGUMENTS

Spawned is a declarative infrastructure platform. You define AWS infrastructure in `infra.json`, then apply it. The platform converts JSON into Terraform and provisions it.

## Quick deploy (most common flow)

```bash
# 0. Ensure Dockerfile exists (create one if missing — see below)
spawned init --name <project>          # 1. create project
# write infra.json with ALL components    # 2. define infrastructure (see templates below)
spawned schema update <project> -f infra.json  # 3. upload schema
spawned apply <project> --schema infra.json  # 4. provision + build (streams terraform logs, takes 5-15 min)
curl https://<project>.dev.askrike.app/   # 5. verify
```

### Step 0: Ensure Dockerfile exists

Before deploying, check if a Dockerfile exists in the build path. If not, **create one and push it to the repo**. Spawned builds containers from Dockerfiles — no Dockerfile means the build will fail.

Read the app's code to determine: language, framework, port, entry point. Then create a Dockerfile:

**Python (FastAPI/Flask):**
```dockerfile
FROM python:3.13-slim
WORKDIR /app
RUN pip install uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY . .
EXPOSE 8000
CMD ["uv", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Node.js (Next.js with standalone):**
```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

**Node.js (Express/plain):**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

**Ruby on Rails:**
```dockerfile
FROM ruby:3.4-slim
WORKDIR /app
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev
COPY Gemfile Gemfile.lock ./
RUN bundle install
COPY . .
RUN bundle exec rails assets:precompile
EXPOSE 3000
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
```

**Go:**
```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM alpine:3.19
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
```

After creating the Dockerfile, commit and push it to the repo's main branch before deploying.

For Next.js apps, also ensure `next.config` has `output: "standalone"`.

Timing: ~5 min without DB, ~10-15 min with DB (RDS is slow).

`apply` streams terraform logs so you see errors. If it hangs with no output for >2 min, the PATCH is still processing — wait. If you used `--detach`, check logs with `spawned workflows <project> --logs`.

**IMPORTANT: Deploy everything at once.** Include ALL components (Container, DB, S3, Lambda, etc.) in the first `infra.json` and apply them together on a fresh project. Do NOT try to incrementally add components to a running deployment — `source` fields on Container and Lambda are silently dropped when updating an existing deployment's schema. If a deployment fails, delete it and start fresh rather than trying to fix it in place (`failed` is terminal).

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

```json
{
  "type": "Container",
  "name": "<app>",
  "values": {
    "id": "<app>-container", "name": "<app>", "provider": "aws",
    "image": null,
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
    "bucket_name": "<project>-storage-<random-8-chars>",
    "force_destroy": true, "versioning_enabled": false, "block_public_access": true
  }
}
```
Generate a unique `bucket_name` with: `python3 -c "import random,string; print('<project>-storage-'+''.join(random.choices(string.ascii_lowercase+string.digits,k=8)))"`. S3 names are globally unique. Old buckets persist after deployment deletion, so always use a fresh random suffix.

### Lambda (scheduled)

```json
{
  "type": "Lambda",
  "name": "<app>-worker",
  "values": {
    "id": "<app>-worker-lambda", "name": "<app>-worker", "provider": "aws",
    "code_path": "src/<repo-name>/<build_path>",
    "source": { "type": "git", "url": "https://github.com/<org>/<repo>", "build_path": "<path>" },
    "runtime": "nodejs20.x", "handler": "src/index.handler",
    "timeout": 300, "memory": 512,
    "schedule": "rate(2 hours)",
    "network": { "$ref": "spawned-vpc" }, "public": false
  }
}
```

---

## Required fields that cause silent failures if wrong

| Field | Component | Rule |
|-------|-----------|------|
| `domain` | Container with LB | **REQUIRED**. `<project>.dev.askrike.app`. Without it, URL returns 503 forever. |
| `image` | Container with source | Set to `null` explicitly. |
| `ports` | Container | Must match Dockerfile `EXPOSE` line. |
| `code_path` | Lambda with source | **REQUIRED** even with source. Set to `src/<repo>/<build_path>`. |
| `bucket_name` | S3Bucket | Must be globally unique across all AWS accounts. |
| `version` | Database (postgres) | Use `"16"`. Other versions may be silently rejected. |

Components with errors are silently dropped from the schema with no error message.

---

## Monitoring

`spawned apply` streams terraform logs directly. If you used `--detach`, check with `spawned get <project>`:

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
spawned init --name <project>                  # create project
spawned init --name <project> --aws-account <id>  # on your own AWS
spawned list                                      # list all projects
spawned get <project>                             # status + URL
spawned delete <project>                       # delete

# Infrastructure
spawned apply <project> --schema infra.json    # apply and stream terraform logs (preferred)
spawned apply <project> --schema infra.json --detach  # background (no error output — avoid)
spawned schema <project>                          # view current schema
spawned schema update <project> -f infra.json  # update schema only (no terraform)
spawned export <project>                          # download terraform as zip

# Code / CI-CD
spawned connect <project> --container <c> --repo <url>  # connect git repo (may error — prefer source in infra.json)
spawned sources <project>                         # list connected repos
spawned redeploy <project>                     # rebuild from latest code

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
