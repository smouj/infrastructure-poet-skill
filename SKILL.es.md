---
name: Infrastructure Poet
summary: Transforma infraestructura como código en planos intuitivos y legibles para humanos
description: Infrastructure Poet analiza configuraciones de IaC (Terraform, CloudFormation, Docker Compose, Kubernetes) y genera planos de arquitectura poéticos, con estilo narrativo, que combinan precisión técnica con explicaciones centradas en el ser humano. Convierte definiciones de infraestructura complejas en documentación comprensible para equipos multifuncionales, revisiones de arquitectura y onboarding.
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

## Propósito

Infrastructure Poet soluciona la brecha de comunicación entre ingenieros de infraestructura y stakeholders no técnicos. Cuando tienes repositorios de IaC complejos, el HCL/YAML/JSON sin procesar está optimizado para máquinas pero opaco para humanos. Esta habilidad genera planos narrativos que explican:

- **Qué** hace la infraestructura en lenguaje llano
- **Por qué** se tomaron decisiones arquitectónicas (inferidas de nombres, comentarios, estructura)
- **Cómo** interactúan los componentes (grafos de dependencias → flujos narrativos)
- **Quién** es dueño de cada componente (de tags, vars, convenciones de nomenclatura)
- **Cuándo** se crean/actualizan recursos (inferencia de ciclo de vida)

Casos de uso reales:
1. Generar documentación de onboarding para nuevos ingenieros DevOps que se unen a un proyecto con 50+ módulos de Terraform
2. Crear materiales de revisión de arquitectura que los gerentes de producto puedan entender realmente
3. Convertir tu disposición de Terraform multi-ambiente (dev/staging/prod) en una narrativa coherente que explique diferencias
4. Transformar una red enredada de microservicios en manifiestos de Kubernetes en una historia sobre dependencias de servicio
5. Producir documentación de cumplimiento que mapee infraestructura a capacidades de negocio
6. Crear diagramas vivos que se actualicen automáticamente cuando cambia el IaC

## Alcance

Infrastructure Poet expone estos comandos CLI:

```
infrastructure-poet scan <path> [flags]
infrastructure-poet render <input> [--format=blueprint|diagram|summary] [--template=<path>]
infrastructure-poet validate <blueprint>
infrastructure-poet diff <old_blueprint> <new_blueprint>
infrastructure-poet watch <dir> [--interval=30s]
infrastructure-poet serve --port=8080
```

**Flags y opciones reales:**
- `--format`: formato de salida (blueprint=narrativa completa, diagram=mermaid/plantuml, summary=oneliner)
- `--template`: override de plantilla Jinja2 personalizada (por defecto: built-in poetic)
- `--exclude`: patrones glob separados por comas (ej., \"*/test/*,*.bak\")
- `--include-vars`: expandir referencias de variables en narrativa
- `--diagram-style`: minimal (cajas básicas), detailed (puertos/protocolos), artistic (colores/temas)
- `--environment`: ambiente objetivo (auto-detectado de workspace/terraform.workspace)
- `--output`: ruta de archivo de salida (por defecto: stdout o ./blueprints/`date`.md)
- `--verbose`: mostrar progreso de parseo y warnings

## Proceso de Trabajo Detallado

**Paso 1 — Inicialización y Descubrimiento**
```bash
# Set output directory
export INFRA_POET_OUTPUT_DIR=\"./docs/architecture\"
# Run scan to detect IaC types and structure
infrastructure-poet scan ./infra --exclude=\"*/tests/*,*.git/*\"
```
El scanner identifica:
- Módulos Terraform (por presencia de .tf files, variables.tf, outputs.tf)
- Plantillas CloudFormation (.yaml/.json con AWSTemplateFormatVersion)
- Archivos Docker Compose (docker-compose.yml, compose.yaml)
- Manifiestos Kubernetes (k8s/, manifests/ con apiVersion fields)
- Proyectos Pulumi (Pulumi.yaml, index.ts/js)
- Playbooks Ansible (playbooks/, tasks/)

Outputs: `./.infracache/scanner-report.json` con inventario de archivos y tipos detectados.

**Paso 2 — Renderizar Blueprint**
```bash
# Generar narrativa completa de blueprint de todo el IaC detectado
infrastructure-poet render ./infra --format=blueprint --include-vars --environment=prod > ./docs/architecture/blueprint.md

# Generar solo un diagrama
infrastructure-poet render ./infra --format=diagram --diagram-style=artistic > ./docs/architecture/diagram.puml

# Renderizar módulo Terraform individual con plantilla personalizada
infrastructure-poet render ./modules/networking --template=./templates/network-narrative.md > ./docs/networking.md
```

El pipeline del renderizador:
1. Parsear cada archivo IaC usando parser apropiado (terraform-docs para TF, cfn-flip para CFN, etc.)
2. Construir AST intermedio (árbol de sintaxis abstracta) representando recursos, variables, dependencias
3. Aplicar motor narrativo:
   - Identificar recursos \"actor\" (instancias ec2, pods k8s, contenedores)
   - Identificar recursos \"escenario\" (vpc, subnets, namespaces)
   - Identificar \"props\" (security groups, configmaps, secrets)
   - Detectar relaciones (depends_on, referencias en ARNs, enlaces de servicio)
4. Aplicar plantilla poética: convertir AST a lenguaje natural con voz consistente
5. Inyectar valores de variable si `--include-vars` habilitado (lee terraform.tfvars, flags --var)
6. Output a archivo o stdout

**Paso 3 — Validar Consistencia**
```bash
# Verificar que blueprint renderizado coincide con IaC fuente (sin deriva)
infrastructure-poet validate ./docs/architecture/blueprint.md --source=./infra
```
El validador compara:
- Coincidencia de conteo de recursos
- Coincidencia de conteo de aristas de dependencia (dentro de tolerancia para inferido vs explícito)
- Resolución de referencias de variables
- Valores específicos de ambiente (tags, región, tamaños)

Retorna código de salida 0 si válido, 1 con diff si se detecta deriva.

**Paso 4 — Observar cambios (integración CI/CD)**
```bash
# En background, auto-actualizar blueprints cuando cambian archivos IaC
infrastructure-poet watch ./infra --interval=30s --output=./docs/architecture/
```
Útil en loop de desarrollo: editar Terraform → blueprint se actualiza automáticamente → commit cambio de documentación junto con código. Puede correrse como GitHub Action o GitLab CI job:
```yaml
# .github/workflows/infra-poet.yml
- name: Update Architecture Blueprint
  run: |
    infrastructure-poet render ./infra --format=blueprint > ./docs/blueprint.md
    git diff --exit-code ./docs/blueprint.md || echo \"Blueprint changed, please review\"
```

**Paso 5 — Servir Blueprints en Vivo**
```bash
# Iniciar servidor web que sirve blueprints y diagramas
infrastructure-poet serve --port=8080 --refresh=60s
```
Provee UI en http://localhost:8080 mostrando:
- Último blueprint con syntax highlighting
- Diagramas Mermaid interactivos (zoom, pan)
- Versiones históricas (últimas 10 renders)
- Vista diff entre versiones

## Reglas de Oro Específicas de Infrastructure Poet

1. **Nunca modificar IaC fuente**. Infrastructure Poet es read-only y nunca debe escribir en archivos .tf, .yml, o plantillas.
2. **Siempre preservar secreto de variables**. Incluso con `--include-vars`, enmascarar secrets marcados `sensitive = true` en Terraform o `Kind: Secret` en K8s. Output `***REDACTED***`.
3. **Respetar límites de ambiente**. Cuando `--environment=prod` esté seteado, no mencionar nombres de recursos staging/dev en blueprint prod. Usar detección multi-ambiente: comparar terraform.workspace, overlays de Kustomize, o nombres de directorio.
4. **Renderizar determinísticamente**. Dada misma entrada y flags, output byte-for-byte idéntico. Sin timestamps, seeds aleatorias, o datos específicos de host en output.
5. **Fallar rápido en errores de parseo**. Si cualquier archivo IaC tiene errores de sintaxis, salir con non-zero antes de generar output parcial. Reportar archivo y número de línea en error.
6. **Mantener compatibilidad hacia atrás**. Versión de formato de blueprint debe ser declarada en frontmatter. Versiones antiguas de Infrastructure Poet pueden renderizar blueprints v1.x (degradación gracefully).
7. **Minimizar latencia de llamadas externas**. Cachear AST parseado en `./.infracache/` keyeado por hash de archivo fuente. Reusar si archivo no cambió.
8. **Extensibilidad vía plugins**. Soportar plantillas narrativas personalizadas vía Jinja2. No enviar con plantillas que embeban terminología propietaria a menos que sea provista por el usuario.
9. **No inferir postura de seguridad**. La narrativa debe declarar \"security configuration requires manual review\" para cada security group, política IAM, y NetworkPolicy. Nunca afirmar \"secure by default.\"
10. **Loguear mínimamente en producción**. Cuando se use en CI, solo output blueprint a stdout; todos los diagnostics ir a stderr.

## Ejemplos con Entradas/Salidas Reales

**Ejemplo 1 — Módulo Terraform básico**

Archivo de entrada: `./modules/webapp/main.tf`
```hcl
resource \"aws_instance\" \"app_server\" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = module.vpc.public_subnets[0]
  tags = {
    Name = \"app-server-${var.environment}\"
    Owner = \"platform-team\"
  }
}

resource \"aws_security_group\" \"app_sg\" {
  name_prefix = \"app-sg-${var.environment}\"
  vpc_id      = module.vpc.vpc_id
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = \"tcp\"
    cidr_blocks = [\"0.0.0.0/0\"]
  }
}
```

Comando:
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

**Ejemplo 2 — Microservicios Docker Compose**

Entrada: `docker-compose.yml`
```yaml
version: '3.8'
services:
  frontend:
    image: myapp/frontend:${FRONTEND_VERSION:-latest}
    ports:
      - \"80:3000\"
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
      - \"3001:3000\"
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

Comando:
```bash
infrastructure-poet render docker-compose.yml --format=blueprint > blueprint.md
```

Output (extracto):
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
1. `redis` y `postgres` arrancan primero (sin dependencias)
2. `api` espera a que `postgres` y `redis` estén listos
3. `frontend` arranca después de que `api` esté disponible

**Data Flow:**
- User → `frontend:80` → `api:3001` → `postgres:5432`
- `api` lee/escribe cache desde `redis:6379`
- Database password se inyecta desde Docker secret externo `db_password`

**Networking:**
Todos los servicios se comunican a través del bridge `app-network`. Frontend expone puerto 80 al host, forwarding a puerto de contenedor 3000. API expone puerto 3001 similar.

**Nota:** El secret externo `db_password` debe existir en Docker Swarm. Si corre localmente, crear con `echo \"changeme\" | docker secret create db_password -`.
```

**Ejemplo 3 — Set de manifiestos Kubernetes**

Entrada: `k8s/production/` (múltiples archivos YAML)

Comando:
```bash
infrastructure-poet render k8s/production --format=diagram --diagram-style=minimal > k8s-diagram.puml
```

Output (PlantUML):
```plantuml
@startuml
skinparam backgroundColor #EEEBD3

package \"namespace: webapp-prod\" {
  [Deployment: webapp]\nreplicas: 3 as webapp_deploy
  [Service: webapp-svc]\nport: 80 → 8080 as webapp_svc
  [Ingress: webapp-ing]\nhost: app.example.com as webapp_ing
  [ConfigMap: webapp-config] as webapp_cm
  [Secret: db-creds] as webapp_secret
}

package \"namespace: data-prod\" {
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

**Ejemplo 4 — Operación diff**

Dadas dos versiones de blueprint: `blueprint_v1.md` y `blueprint_v2.md`

Comando:
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

**Ejemplo 5 — Modo watch en desarrollo**

```bash
# Terminal 1: correr watch
infrastructure-poet watch ./infra --interval=10s --output=./docs/live/

# Terminal 2: editar un archivo Terraform
vim ./infra/networking/vpc.tf  # agregar una nueva subnet

# Dentro de 10 segundos, ./docs/live/blueprint_20241215_143025.md se actualiza automáticamente
# Muestra diff:
+ ## New Subnet Added
+ - Name: public-subnet-3
+ - CIDR: 10.0.3.0/24
+ - AZ: us-east-1b
+ - Purpose: Additional availability zone for high availability
```

## Comandos de Rollback Específicos de Infrastructure Poet

**Escenario 1 — Rollback blueprint a versión anterior**
```bash
# Los blueprints tienen timestamp, listar versiones disponibles
ls -1t ./blueprints/blueprint_*.md

# Copiar la versión anterior sobre la actual
cp ./blueprints/blueprint_20241215_143025.md ./docs/architecture/blueprint.md

# O usar rollback incorporado si se usa modo serve
curl -X POST http://localhost:8080/rollback?version=20241215T14:30:00Z
```

**Escenario 2 — Revertir a generación de commit Git específico**
```bash
# Si IaC cambió pero blueprint debería coincidir con versión antigua:
git checkout abc123 -- infra/
infrastructure-poet render ./infra > ./docs/blueprint.md
git checkout main -- infra/  # volver a código más reciente
```

**Escenario 3 — Limpiar cache y forzar parseo fresco**
```bash
# Si corrupción de cache o actualización de parser
rm -rf ./.infracache/
infrastructure-poet render ./infra --force-rebuild
```

**Escenario 4 — Deshacer cambios de plantilla personalizada**
```bash
# Si modificaste una plantilla Jinja2 personalizada y quieres volver a la incorporada
infrastructure-poet render ./infra --template=default > blueprint.md
# Para resetear archivo de plantilla personalizada:
git checkout -- ./templates/custom-narrative.md
```

**Escenario 5 — Rollback CI/CD (ejemplo GitHub Actions)**
```yaml
- name: Check blueprint drift and rollback if needed
  run: |
    infrastructure-poet render ./infra > ./new-blueprint.md
    if ! infrastructure-poet validate ./docs/blueprint.md --source=./infra; then
      echo \"ERROR: Blueprint drift detected!\"
      echo \"Restoring previous blueprint from Git...\"
      git checkout HEAD -- ./docs/blueprint.md
      exit 1
    fi
```

**Escenario 6 — Rollback completo de actualizaciones de Infrastructure Poet**
Si actualizar versión de Infrastructure Poet rompe el renderizado:
```bash
# Reinstalar versión anterior (si se usa pip)
pip install 'infrastructure-poet==1.1.0'
# Limpiar cache para evitar conflictos de versión
rm -rf ./.infracache/
```

## Pasos de Verificación Específicos de Esta Habilidad

1. **Verificar detección de parser**
```bash
infrastructure-poet scan ./infra | jq '.file_types'
# Debería listar tipos IaC detectados: [\"terraform\", \"docker-compose\", \"kubernetes\"]
```

2. **Verificar completitud de blueprint**
```bash
# Contar recursos en blueprint vs fuente
grep -c \"^\s*-\s*\*\*\" ./docs/blueprint.md  # contar headers de recursos
terraform-docs json ./infra | jq '.resources | length'  # conteo esperado
# Los números deberían coincidir (dentro de 1 si se ignoran recursos implícitamente-creados)
```

3. **Probar resolución de variables**
```bash
# Asegurar que no queden variables placeholder
grep -n '\${[^}]*}' ./docs/blueprint.md && echo \"ERROR: vars sin resolver\" || echo \"OK\"
```

4. **Validar sintaxis de diagrama**
```bash
# Si se generó diagrama, verificar que compile
plantuml -plantuml ./docs/diagram.puml 2>&1 | grep -i error && echo \"INVALID\" || echo \"VALID\"
# O para Mermaid:
mmdc -i ./docs/diagram.mmd -o /dev/null 2>&1 | grep -i error && echo \"INVALID\" || echo \"VALID\"
```

5. **Verificar enmascaramiento de secrets**
```bash
# Nunca debería contener valores de secret reales
if grep -q \"AWS_SECRET_ACCESS_KEY\" ./docs/blueprint.md; then
  echo \"FAIL: secret filtrado\" && exit 1
fi
```

6. **Liveness de modo watch**
```bash
# Iniciar watch en background
infrastructure-poet watch ./infra --interval=5s &
WATCH_PID=$!
sleep 15
touch ./infra/new_resource.tf
sleep 10
grep \"new_resource\" ./docs/live/blueprint_*.md && echo \"OK: cambio detectado\" || echo \"FAIL: sin actualización\"
kill $WATCH_PID
```

## Solución de Problemas Comunes

| Problema | Causa | Solución |
|----------|-------|----------|
| `Error: No IaC files detected in ./path` | Directorio incorrecto o tipos de archivo no soportados | Correr `infrastructure-poet scan .` para ver qué encuentra; asegurar que haya archivos .tf, .yml, .yaml presentes |
| `Parser crashed: unexpected token` | Error de sintaxis en archivo fuente IaC | Syntax-check fuente: `terraform validate`, `yamllint`, `kubeval`. Arreglar IaC original. |
| `Variable 'xxx' not resolved` | Falta archivo tfvars o flag --var | Pasar `-var-file=terraform.tfvars` o setear variables de entorno. Usar `terraform-docs json` para ver variables esperadas. |
| `Diagram output empty` | No se detectaron relaciones de dependencia | Agregar `depends_on` explícito o referenciar recursos por nombre (ej., `module.vpc.vpc_id`). La poesía infiere desde referencias. |
| `Blueprint differs from source (validate fails)` | IaC cambió después de generación de blueprint | Re-correr `render`. Si deriva persiste inesperadamente, revisar `terraform count`/`for_each` expansions; el validador tiene en cuenta estos. |
| `MemoryError during render of large repo` | Muchos recursos (>500) en single run | Renderizar por módulo en vez de repo completo: `infrastructure-poet render ./modules/module1`, `./module2`. |
| `Template syntax error in custom template` | Error de sintaxis Jinja2 | Testear plantilla con `python -c \"from jinja2 import Template; Template(open('template.md').read())\"` |
| `Watch mode doesn't update` | Cambio de archivo no detectado por límites de inotify | Aumentar inotify watches: `sysctl fs.inotify.max_user_watches=524288` |
| `Secrets still visible in blueprint` | Secret no marcado `sensitive = true` en Terraform o `Kind: Secret` en K8s | Marcar secrets apropiadamente. Como último recurso, usar flag `--exclude-secrets` para omitir todos los recursos secretos completamente (no aparecerán en blueprint). |
| `Port 8080 already in use` (serve) | Otra instancia de Infrastructure Poet o app corriendo | Matar proceso existente: `pkill -f infrastructure-poet` o usar `--port=8081` |

## Dependencias y Requisitos

**Sistema:**
- Linux/macOS (Windows WSL2 soportado)
- Python 3.9+ con pip
- GCC/Build tools para cualquier paquete Python con extensiones C (generalmente no necesario)

**Instalar:**
```bash
# Release oficial
curl -fsSL https://get.infrastructure-poet.io | sh

# O desde source
git clone https://github.com/openclaw/infrastructure-poet.git
cd infrastructure-poet
pip install -e .
```

**Verificar instalación:**
```bash
infrastructure-poet version
# infrastructure-poet 1.2.0 (commit: abc123)
# python 3.11.5, terraform-docs 0.16.0, yq 4.30.0
```

**Archivo de configuración (opcional):** `~/.infrastructure-poet/config.yaml`
```yaml
output_dir: ./blueprints
default_environment: production
exclude_patterns:
  - \"*/test/*\"
  - \"*.bak\"
  - \".git/*\"
template_path: ~/.infrastructure-poet/templates/
diagram:
  style: minimal
  max_nodes: 50  # warn if diagram exceeds this
cache_dir: ./.infracache
```

**Sistema de plugins:** Colocar módulos Python en `~/.infrastructure-poet/plugins/` que implementen hooks `parse_<type>` y `render_<type>` para formatos IaC personalizados.

## Notas de Rendimiento

- **Cacheo**: AST cache vive en `.infracache/`. Keyeado por sha256 de archivos fuente. Invalidado cuando cualquier archivo mtime cambia. Render con cache hit en ~0.1s; miss en ~2-5s dependiendo de tamaño de repo.
- **Parseo paralelo**: Por defecto, parsea todos archivos IaC concurrentemente (hasta 8 workers). Usar `--workers=1` para debugging.
- **Escalamiento de memoria**: ~50MB por 100 recursos. Un repo de 1000 recursos necesita ~500MB. Usar `render --chunk-size=200` para limitar.
- **Generación de diagramas**: Para grafos grandes (>200 nodos), usar `--diagram-style=minimal` para evitar timeouts de renderizado.
```
```