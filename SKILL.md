---
name: Infrastructure Poet
summary: Transforms infrastructure-as-code into intuitive, human-readable blueprints
description: Infrastructure Poet parses IaC configurations (Terraform, CloudFormation, Docker Compose, Kubernetes) and generates poetic, narrative-style architecture blueprints that combine technical accuracy with human-centric explanations. Converts complex infrastructure definitions into digestible documentation suitable for cross-functional teams, architecture reviews, and onboarding.
version: 1.2.0
author: OpenClaw DevOps Team
tags:
  - devops
  - infrastructure
  - poetry
  - documentation
  - as-code
dependencies:
  - terraform-docs (>= 0.16.0)
  - yq (>= 4.30.0)
  - jq (>= 1.6)
  - python3 (>= 3.9)
  - pip:
    - infrac戴上ivate==1.4.2
    - ruamel.yaml==0.17.32
  - optional:
    - graphviz (for diagram generation)
    - plantuml (for sequence diagrams)
required_env_vars:
  - INFRA_POET_OUTPUT_DIR (default: ./blueprints)
  - INFRA_POET_TEMPLATE_PATH (optional: path to Jinja2 template)
  - INFRA_POET_DIAGRAM_STYLE (default: minimal, options: minimal, detailed, artistic)
---

# Infrastructure Poet

## Purpose

Infrastructure Poet solves the communication gap between infrastructure engineers and non-technical stakeholders. When you have complex IaC repositories, the raw HCL/YAML/JSON is machine-optimized but human-opaque. This skill generates narrative blueprints that explain:

- **What** the infrastructure does in plain language
- **Why** architectural decisions were made (inferred from naming, comments, structure)
- **How** components interact (dependency graphs → narrative flows)
- **Who** owns each component (from tags, vars, naming conventions)
- **When** resources are created/updated (lifecycle inference)

Real use cases:
1. Generate onboarding documentation for new DevOps engineers joining a project with 50+ Terraform modules
2. Create architecture review materials that product managers can actually understand
3. Convert your multi-environment Terraform layout (dev/staging/prod) into a single coherent narrative explaining differences
4. Transform a tangled web of microservices in Kubernetes manifests into a story about service dependencies
5. Produce compliance documentation that maps infrastructure to business capabilities
6. Create living diagrams that update automatically when IaC changes

## Scope

Infrastructure Poet exposes these CLI commands:

```
infrastructure-poet scan <path> [flags]
infrastructure-poet render <input> [--format=blueprint|diagram|summary] [--template=<path>]
infrastructure-poet validate <blueprint>
infrastructure-poet diff <old_blueprint> <new_blueprint>
infrastructure-poet watch <dir> [--interval=30s]
infrastructure-poet serve --port=8080
```

**Real flags and options:**
- `--format`: output format (blueprint=full narrative, diagram=mermaid/plantuml, summary=oneliner)
- `--template`: custom Jinja2 template override (default: built-in poetic)
- `--exclude`: comma-separated glob patterns (e.g., "*/test/*,*.bak")
- `--include-vars`: expand variable references in narrative
- `--diagram-style`: minimal (basic boxes), detailed (ports/protocols), artistic (colors/themes)
- `--environment`: target environment (auto-detected from workspace/terraform.workspace)
- `--output`: output file path (default: stdout or ./blueprints/`date`.md)
- `--verbose`: show parsing progress and warnings

## Detailed Work Process

**Step 1 — Initialization and Discovery**
```bash
# Set output directory
export INFRA_POET_OUTPUT_DIR="./docs/architecture"
# Run scan to detect IaC types and structure
infrastructure-poet scan ./infra --exclude="*/tests/*,*.git/*"
```
The scanner identifies:
- Terraform modules (by presence of .tf files, variables.tf, outputs.tf)
- CloudFormation templates (.yaml/.json with AWSTemplateFormatVersion)
- Docker Compose files (docker-compose.yml, compose.yaml)
- Kubernetes manifests (k8s/, manifests/ with apiVersion fields)
- Pulumi projects (Pulumi.yaml, index.ts/js)
- Ansible playbooks (playbooks/, tasks/)

Outputs: `./.infracache/scanner-report.json` with file inventory and detected types.

**Step 2 — Render Blueprint**
```bash
# Generate full narrative blueprint from all detected IaC
infrastructure-poet render ./infra --format=blueprint --include-vars --environment=prod > ./docs/architecture/blueprint.md

# Generate only a diagram
infrastructure-poet render ./infra --format=diagram --diagram-style=artistic > ./docs/architecture/diagram.puml

# Render single Terraform module with custom template
infrastructure-poet render ./modules/networking --template=./templates/network-narrative.md > ./docs/networking.md
```

The renderer pipeline:
1. Parse each IaC file using appropriate parser (terraform-docs for TF, cfn-flip for CFN, etc.)
2. Build intermediate AST (abstract syntax tree) representing resources, variables, dependencies
3. Apply narrative engine:
   - Identify "actor" resources (ec2 instances, k8s pods, containers)
   - Identify "stage" resources (vpc, subnets, namespaces)
   - Identify "props" (security groups, configmaps, secrets)
   - Detect relationships (depends_on, references in ARNs, service linking)
4. Apply poetic template: turn AST into natural language with consistent voice
5. Inject variable values if `--include-vars` enabled (reads terraform.tfvars, --var flags)
6. Output to file or stdout

**Step 3 — Validate Consistency**
```bash
# Check that rendered blueprint matches source IaC (no drift)
infrastructure-poet validate ./docs/architecture/blueprint.md --source=./infra
```
Validator compares:
- Resource count match
- Dependency edge count match (within tolerance for inferred vs explicit)
- Variable references resolution
- Environment-specific values (tags, region, sizes)

Returns exit code 0 if valid, 1 with diff if drift detected.

**Step 4 — Watch for Changes (CI/CD integration)**
```bash
# In background, auto-update blueprints when IaC files change
infrastructure-poet watch ./infra --interval=30s --output=./docs/architecture/
```
Useful in dev loop: edit Terraform → blueprint auto-updates → commit documentation change alongside code. Can be run as a GitHub Action or GitLab CI job:
```yaml
# .github/workflows/infra-poet.yml
- name: Update Architecture Blueprint
  run: |
    infrastructure-poet render ./infra --format=blueprint > ./docs/blueprint.md
    git diff --exit-code ./docs/blueprint.md || echo "Blueprint changed, please review"
```

**Step 5 — Serve Live Blueprints**
```bash
# Start web server that serves blueprints and diagrams
infrastructure-poet serve --port=8080 --refresh=60s
```
Provides UI at http://localhost:8080 showing:
- Latest blueprint with syntax highlighting
- Interactive Mermaid diagrams (zoom, pan)
- Historical versions (last 10 renders)
- Diff view between versions

## Golden Rules Specific to Infrastructure Poet

1. **Never modify source IaC**. Infrastructure Poet is read-only and must never write to .tf, .yml, or template files.
2. **Always preserve variable secrecy**. Even with `--include-vars`, mask secrets marked `sensitive = true` in Terraform or `Kind: Secret` in K8s. Output `***REDACTED***`.
3. **Respect environment boundaries**. When `--environment=prod` is set, do not mention staging/dev resource names in prod blueprint. Use multi-environment detection: compare terraform.workspace, Kustomize overlays, or directory names.
4. **Render deterministically**. Given same input and flags, output byte-for-byte identical. No timestamps, random seeds, or host-specific data in output.
5. **Fail fast on parse errors**. If any IaC file has syntax errors, exit non-zero before generating partial output. Report file and line number in error.
6. **Maintain backward compatibility**. Blueprint format version must be declared in frontmatter. Older Infrastructure Poet versions can render v1.x blueprints (graceful degradation).
7. **Minimize external call latency**. Cache parsed AST in `./.infracache/` keyed by file hash. Reuse if file unchanged.
8. **Extensibility via plugins**. Support custom narrative templates via Jinja2. Do not ship with templates that embed proprietary terminology unless user-provided.
9. **Do not infer security posture**. Narrative must state "security configuration requires manual review" for every security group, IAM policy, and NetworkPolicy. Never claim "secure by default."
10. **Log minimally in production**. When used in CI, only output blueprint to stdout; all diagnostics go to stderr.

## Examples with Real Inputs/Outputs

**Example 1 — Basic Terraform module**

Input file: `./modules/webapp/main.tf`
```hcl
resource "aws_instance" "app_server" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = module.vpc.public_subnets[0]
  tags = {
    Name = "app-server-${var.environment}"
    Owner = "platform-team"
  }
}

resource "aws_security_group" "app_sg" {
  name_prefix = "app-sg-${var.environment}"
  vpc_id      = module.vpc.vpc_id
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Command:
```bash
infrastructure-poet render ./modules/webapp --environment=prod --include-vars
```

Output: `./blueprints/webapp_prod_20241215.md`
```markdown
# Web Application Infrastructure Blueprint

**Generated:** 2024-12-15T14:30:00Z  
**Source:** ./modules/webapp  
**Environment:** Production  
**IaC Type:** Terraform (HCL)

---

## Narrative

The *webapp* module provisions a single **application server** (`app_server`) that runs on AWS EC2. The server lives in the first public subnet of a VPC (referenced from `module.vpc`), making it internet-accessible. Its instance type is configured via variable `instance_type`, allowing the same code to run different sizes in different environments.

A security group (`app_sg`) acts as a firewall for the server. It allows inbound TCP traffic on port 8080 from anywhere (0.0.0.0/0). **Note:** This wide-open rule is typical for public web services but should be reviewed for security compliance.

The server is tagged with:
- Name: `app-server-prod`
- Owner: `platform-team`

These tags help with cost allocation and operational ownership.

---

## Variables Resolved

- `environment`: prod
- `ami_id`: ami-0c55b159cbfafe1f0 (Amazon Linux 2023)
- `instance_type`: t3.medium

---
```

**Example 2 — Docker Compose microservices**

Input: `docker-compose.yml`
```yaml
version: '3.8'
services:
  frontend:
    image: myapp/frontend:${FRONTEND_VERSION:-latest}
    ports:
      - "80:3000"
    depends_on:
      - api
      - redis
    environment:
      - API_URL=http://api:3001
    networks:
      - app-network

  api:
    image: myapp/api:${API_VERSION:-latest}
    ports:
      - "3001:3000"
    environment:
      - REDIS_HOST=redis
      - DB_HOST=postgres
    networks:
      - app-network
    depends_on:
      - postgres
      - redis

  redis:
    image: redis:7-alpine
    networks:
      - app-network

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - app-network
    secrets:
      - db_password

volumes:
  pgdata:

networks:
  app-network:
    driver: bridge

secrets:
  db_password:
    external: true
```

Command:
```bash
infrastructure-poet render docker-compose.yml --format=blueprint > blueprint.md
```

Output (excerpt):
```markdown
# Docker Compose Blueprint

**Compose file:** docker-compose.yml  
**Version:** 3.8

## Component Map

| Service | Image | Role | Dependencies |
|---------|-------|------|--------------|
| frontend | myapp/frontend | User-facing UI | api, redis |
| api | myapp/api | Business logic layer | postgres, redis |
| redis | redis:7-alpine | Caching layer | none |
| postgres | postgres:15 | Primary database | none |

---

## Narrative Flow

**Startup Sequence:**
1. `redis` and `postgres` start first (no dependencies)
2. `api` waits for `postgres` and `redis` to be ready
3. `frontend` starts after `api` is available

**Data Flow:**
- User → `frontend:80` → `api:3001` → `postgres:5432`
- `api` reads/writes cache from `redis:6379`
- Database password is injected from external Docker secret `db_password`

**Networking:**
All services communicate through the `app-network` bridge. Frontend exposes port 80 to the host, forwarding to container port 3000. API exposes port 3001 similarly.

**Note:** External secret `db_password` must exist in Docker Swarm. If running locally, create with `echo "changeme" | docker secret create db_password -`.
```

**Example 3 — Kubernetes manifest set**

Input: `k8s/production/` (multiple YAML files)

Command:
```bash
infrastructure-poet render k8s/production --format=diagram --diagram-style=minimal > k8s-diagram.puml
```

Output (PlantUML):
```plantuml
@startuml
skinparam backgroundColor #EEEBD3

package "namespace: webapp-prod" {
  [Deployment: webapp]\nreplicas: 3 as webapp_deploy
  [Service: webapp-svc]\nport: 80 → 8080 as webapp_svc
  [Ingress: webapp-ing]\nhost: app.example.com as webapp_ing
  [ConfigMap: webapp-config] as webapp_cm
  [Secret: db-creds] as webapp_secret
}

package "namespace: data-prod" {
  [StatefulSet: postgres]\nreplicas: 1 as pg_sts
  [Service: postgres-svc]\nport: 5432 as pg_svc
  [PersistentVolumeClaim: pg-data] as pg_pvc
}

webapp_deploy --> webapp_svc : targets
webapp_ing --> webapp_svc : routes
webapp_deploy --> webapp_cm : mounts config
webapp_deploy --> webapp_secret : mounts creds
webapp_svc --> pg_svc : connects
pg_sts --> pg_pvc : claims storage

note right of webapp_secret
  **SECRET**: Never check into VCS
  Rotate every 90 days
end note
@enduml
```

**Example 4 — Diff operation**

Given two blueprint versions: `blueprint_v1.md` and `blueprint_v2.md`

Command:
```bash
infrastructure-poet diff blueprint_v1.md blueprint_v2.md
```

Output:
```diff
--- blueprint_v1.md
+++ blueprint_v2.md
@@ -45,7 +45,7 @@
   - `instance_type`: t3.medium
 
 ## Changes Detected
-
+- **Resource Added:** aws_s3_bucket.logs (for centralized logging)
 - **Variable Changed:** `instance_type` from `t3.medium` to `t3.large`
 
 ### Implications
 Adding an S3 bucket for logs introduces new cost (~$0.023/GB/month).
+Ensure lifecycle policy is set to avoid unbounded growth.
```

**Example 5 — Watch mode in development**

```bash
# Terminal 1: run watch
infrastructure-poet watch ./infra --interval=10s --output=./docs/live/

# Terminal 2: edit a Terraform file
vim ./infra/networking/vpc.tf  # add a new subnet

# Within 10 seconds, ./docs/live/blueprint_20241215_143025.md updates automatically
# Shows diff:
+ ## New Subnet Added
+ - Name: public-subnet-3
+ - CIDR: 10.0.3.0/24
+ - AZ: us-east-1b
+ - Purpose: Additional availability zone for high availability
```

## Rollback Commands Specific to Infrastructure Poet

**Scenario 1 — Rollback blueprint to previous version**
```bash
# Blueprints are timestamped, list available versions
ls -1t ./blueprints/blueprint_*.md

# Copy the previous version over current
cp ./blueprints/blueprint_20241215_143025.md ./docs/architecture/blueprint.md

# Or use built-in rollback if using serve mode
curl -X POST http://localhost:8080/rollback?version=20241215T14:30:00Z
```

**Scenario 2 — Revert to generation from specific Git commit**
```bash
# If IaC changed but blueprint should match older version:
git checkout abc123 -- infra/
infrastructure-poet render ./infra > ./docs/blueprint.md
git checkout main -- infra/  # return to latest code
```

**Scenario 3 — Clear cache and force fresh parse**
```bash
# If cache corruption or parser upgrade
rm -rf ./.infracache/
infrastructure-poet render ./infra --force-rebuild
```

**Scenario 4 — Undo custom template changes**
```bash
# If you modified a custom Jinja2 template and want to revert to built-in
infrastructure-poet render ./infra --template=default > blueprint.md
# To reset custom template file:
git checkout -- ./templates/custom-narrative.md
```

**Scenario 5 — CI/CD rollback (GitHub Actions example)**
```yaml
- name: Check blueprint drift and rollback if needed
  run: |
    infrastructure-poet render ./infra > ./new-blueprint.md
    if ! infrastructure-poet validate ./docs/blueprint.md --source=./infra; then
      echo "ERROR: Blueprint drift detected!"
      echo "Restoring previous blueprint from Git..."
      git checkout HEAD -- ./docs/blueprint.md
      exit 1
    fi
```

**Scenario 6 — Complete rollback of Infrastructure Poet upgrades**
If upgrading Infrastructure Poet version breaks rendering:
```bash
# Reinstall previous version (if using pip)
pip install 'infrastructure-poet==1.1.0'
# Clear cache to avoid version conflicts
rm -rf ./.infracache/
```

## Verification Steps Specific to This Skill

1. **Check parser detection**
```bash
infrastructure-poet scan ./infra | jq '.file_types'
# Should list detected IaC types: ["terraform", "docker-compose", "kubernetes"]
```

2. **Verify blueprint completeness**
```bash
# Count resources in blueprint vs source
grep -c "^\s*-\s*\*\*" ./docs/blueprint.md  # count resource headers
terraform-docs json ./infra | jq '.resources | length'  # expected count
# Numbers should match (within 1 if ignoring implicitly-created resources)
```

3. **Test variable resolution**
```bash
# Ensure no placeholder variables remain
grep -n '\${[^}]*}' ./docs/blueprint.md && echo "ERROR: unresolved vars" || echo "OK"
```

4. **Validate diagram syntax**
```bash
# If diagram generated, check it compiles
plantuml -plantuml ./docs/diagram.puml 2>&1 | grep -i error && echo "INVALID" || echo "VALID"
# Or for Mermaid:
mmdc -i ./docs/diagram.mmd -o /dev/null 2>&1 | grep -i error && echo "INVALID" || echo "VALID"
```

5. **Check secret masking**
```bash
# Should never contain actual secret values
if grep -q "AWS_SECRET_ACCESS_KEY" ./docs/blueprint.md; then
  echo "FAIL: secret leaked" && exit 1
fi
```

6. **Watch mode liveness**
```bash
# Start watch in background
infrastructure-poet watch ./infra --interval=5s &
WATCH_PID=$!
sleep 15
touch ./infra/new_resource.tf
sleep 10
grep "new_resource" ./docs/live/blueprint_*.md && echo "OK: detected change" || echo "FAIL: no update"
kill $WATCH_PID
```

## Troubleshooting Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `Error: No IaC files detected in ./path` | Wrong directory or unsupported file types | Run `infrastructure-poet scan .` to see what it finds; ensure .tf, .yml, .yaml files present |
| `Parser crashed: unexpected token` | IaC syntax error in source file | Syntax-check source: `terraform validate`, `yamllint`, `kubeval`. Fix the original IaC. |
| `Variable 'xxx' not resolved` | Missing tfvars file or --var flag | Pass `-var-file=terraform.tfvars` or set environment variables. Use `terraform-docs json` to see expected variables. |
| `Diagram output empty` | No dependency relationships detected | Add explicit `depends_on` or reference resources by name (e.g., `module.vpc.vpc_id`). Poetry infers from references. |
| `Blueprint differs from source (validate fails)` | IaC changed after blueprint generation | Re-run `render`. If drift persists unexpectedly, check for Terraform `count`/`for_each` expansions; validator accounts for these. |
| `MemoryError during render of large repo` | Too many resources (>500) in single run | Render per-module instead of whole repo: `infrastructure-poet render ./modules/module1`, `./module2`. |
| `Template syntax error in custom template` | Jinja2 syntax mistake | Test template with `python -c "from jinja2 import Template; Template(open('template.md').read())"` |
| `Watch mode doesn't update` | File change not detected due to inotify limits | Increase inotify watches: `sysctl fs.inotify.max_user_watches=524288` |
| `Secrets still visible in blueprint` | Secret not marked `sensitive = true` in Terraform or `Kind: Secret` in K8s | Mark secrets properly. As last resort, use `--exclude-secrets` flag to skip all secret resources entirely (they won't appear in blueprint). |
| `Port 8080 already in use` (serve) | Another Infrastructure Poet instance or app running | Kill existing process: `pkill -f infrastructure-poet` or use `--port=8081` |

## Dependencies and Requirements

**System:**
- Linux/macOS (Windows WSL2 supported)
- Python 3.9+ with pip
- GCC/Build tools for any Python packages with C extensions (usually not needed)

**Install:**
```bash
# Official release
curl -fsSL https://get.infrastructure-poet.io | sh

# Or from source
git clone https://github.com/openclaw/infrastructure-poet.git
cd infrastructure-poet
pip install -e .
```

**Verify install:**
```bash
infrastructure-poet version
# infrastructure-poet 1.2.0 (commit: abc123)
# python 3.11.5, terraform-docs 0.16.0, yq 4.30.0
```

**Configuration file (optional):** `~/.infrastructure-poet/config.yaml`
```yaml
output_dir: ./blueprints
default_environment: production
exclude_patterns:
  - "*/test/*"
  - "*.bak"
  - ".git/*"
template_path: ~/.infrastructure-poet/templates/
diagram:
  style: minimal
  max_nodes: 50  # warn if diagram exceeds this
cache_dir: ./.infracache
```

**Plugin system:** Place Python modules in `~/.infrastructure-poet/plugins/` that implement `parse_<type>` and `render_<type>` hooks for custom IaC formats.

## Performance Notes

- **Caching**: AST cache lives in `.infracache/`. Keyed by sha256 of source files. Invalidated when any file mtime changes. Cache hit renders in ~0.1s; miss in ~2-5s depending on repo size.
- **Parallel parsing**: By default, parses all IaC files concurrently (up to 8 workers). Use `--workers=1` for debugging.
- **Memory scaling**: ~50MB per 100 resources. A 1000-resource repo needs ~500MB. Use `render --chunk-size=200` to limit.
- **Diagram generation**: For large graphs (>200 nodes), use `--diagram-style=minimal` to avoid renderer timeouts.

```