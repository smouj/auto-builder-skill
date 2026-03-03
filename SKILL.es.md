---
name: auto-builder
description: Constructor de artefactos e investigaciones automatizado con IA, ejecución paralela, gestión de dependencias y verificación
version: 2.1.0
author: SMOUJBOT
tags: [research, ai, automation]
maintainer: SMOUJBOT
license: MIT
dependencies:
  - python3.11+
  - nodejs18+
  - docker
  - make
  - git
  - curl
required_env:
  - OPENCLAUDE_API_KEY (opcional, para planificación asistida por IA)
  - BUILD_WORKDIR (por defecto: /tmp/builds)
  - MAX_PARALLEL_JOBS (por defecto: 4)
capabilities:
  - parallel_builds
  - dependency_resolution
  - artifact_generation
  - verification_pipeline
  - rollback_snapshots
---

# Habilidad Auto Builder

## Propósito

Auto Builder automatiza el ciclo de vida completo de la construcción de software impulsada por investigación: desde la recopilación de requisitos y resolución de dependencias hasta la compilación en paralelo y la verificación de artefactos. Diseñado para los flujos de trabajo multiagente de OpenClaw, permite la orquestación de construcciones complejas en entornos heterogéneos (Docker, nativo, CI) con un solo comando.

### Casos de Uso Reales

1. **Prototipo de investigación multi-componente** - Construir simultáneamente un backend Node.js, un pipeline de datos Python y un frontend React al probar una nueva técnica de integración de IA, con detección automática de conflictos de dependencias entre componentes.

2. **Pipeline de verificación de parches** - Después de generar un fix de seguridad mediante agente de investigación, construir automáticamente todos los servicios afectados, ejecutar pruebas de integración y generar reports diff comparando artefactos contra el baseline.

3. **Generación de artefactos multi-plataforma** - Producir binarios Linux, imágenes Docker y paquetes npm desde una única definición de artefacto de investigación, con optimizaciones específicas de plataforma aplicadas automáticamente.

4. **Análisis de mapa de calor de dependencias** - Durante la construcción, rastrear qué dependencias se usan realmente vs. las declaradas, generando recomendaciones de optimización y reportes de vulnerabilidades de seguridad para investigación adicional.

5. **Construcciones de investigación reproducibles** - Bloquear todas las dependencias transitivas, capturar hashes del entorno de construcción y generar SBOMs (Software Bill of Materials) para requisitos de reproducibilidad académica.

## Alcance

Auto Builder implementa los siguientes comandos de OpenClaw:

- `openclaw build init <project_name>` - Inicializar configuración de construcción
- `openclaw build plan <spec_file>` - Generar plan de ejecución sin construir
- `openclaw build execute <plan_file>` - Ejecutar plan de construcción validado
- `openclaw build verify <artifact_manifest>` - Verificar artefactos construidos contra checksums
- `openclaw build rollback <snapshot_id>` - Restaurar a estado de construcción anterior
- `openclaw build watch <target>` - Monitorear cambios en archivos y disparar construcciones incrementales
- `openclaw build deps <component>` - Resolver y visualizar grafo de dependencias
- `openclaw build matrix <config>` - Ejecutar construcciones matrix a través de entornos

## Proceso de Trabajo Detallado

### Fase 1: Descubrimiento de Configuración (0-30s)

1. **Ingestión de entrada**: Acepta ya sea:
   - Archivo de especificación `build.yaml` (preferido)
   - Argumentos CLI en línea (legacy)
   - Prompt interactivo de preguntas si ninguno presente

2. **Detección de entorno**:
   ```bash
   # Comandos reales ejecutados internamente:
   uname -m && uname -s
   docker version --format '{{.Server.Version}}' 2>/dev/null || echo "docker: absent"
   node --version
   python3 --version
   make --version | head -1
   ```

3. **Preparación de workspace**:
   ```bash
   mkdir -p "${BUILD_WORKDIR:-/tmp/builds}/$(date +%s)"
   cp -r "$PWD" "${BUILD_WORKDIR:-/tmp/builds}/$(date +%s)/source"
   ```

### Fase 2: Resolución de Dependencias (30s-2m)

1. Parsear archivos de manifiesto:
   - `package.json` → dependencias npm
   - `requirements.txt` / `pyproject.toml` → pip/poetry
   - `go.mod` → módulos go
   - `Cargo.toml` → cargo
   - `pom.xml` → maven
   - `build.gradle` → gradle

2. Generar grafo de dependencias unificado:
   ```bash
   "scripts/deps-resolver.py" \
     --input "${BUILD_WORKDIR}/$(date +%s)/source" \
     --output "${BUILD_WORKDIR}/$(date +%s)/deps-graph.json" \
     --conflict-check \
     --vuln-scan \
     --lock
   ```

3. Detectar dependencias circulares y conflictos de versión antes de proceder.

### Fase 3: Planificación de Construcción (2m-3m)

1. Crear DAG (Directed Acyclic Graph) de ejecución desde:
   - Targets de especificación de construcción
   - Grafo de dependencias
   - Restricciones de plataforma

2. Determinar estrategia de paralelización:
   ```bash
   "scripts/planner.py" \
     --deps "${BUILD_WORKDIR}/$(date +%s)/deps-graph.json" \
     --jobs "${MAX_PARALLEL_JOBS:-4}" \
     --strategy "resource-aware" \
     > "${BUILD_WORKDIR}/$(date +%s)/plan.json"
   ```

3. Salida de preview de plan legible:
   ```
   Phase 1: base-deps [0/3]
   Phase 2: backend [0/2]
   Phase 3: frontend [0/1]
   Phase 4: integration [0/1]
   Estimated: 4m 23s
   ```

### Fase 4: Ejecución Paralela (3m-10m)

1. Ejecutar etapas de construcción respetando dependencias DAG:
   ```bash
   "scripts/executor.py" \
     --plan "${BUILD_WORKDIR}/$(date +%s)/plan.json" \
     --workspace "${BUILD_WORKDIR}/$(date +%s)/build" \
     --log-level INFO \
     --fail-fast \
     --retry 2
   ```

2. Cada paso de construcción:
   - Se ejecuta en contexto de construcción aislado
   - Transmite stdout/stderr a log central
   - Captura artefactos a `artifacts/<component>/`
   - Registra hash de entorno (`sha256sum` de compilador, flags, entradas)
   - Actualiza manifiesto: `build-manifest.json`

3. En fallo: detención inmediata, guardar artefactos de debug, generar reporte de error.

### Fase 5: Verificación (10m-12m)

1. **Verificación de hash**:
   ```bash
   "scripts/verify.py" \
     --manifest "${BUILD_WORKDIR}/$(date +%s)/build-manifest.json" \
     --alg sha256 \
     --strict
   ```

2. **Prueba de integración** (si existe directorio `tests/`):
   ```bash
   cd "${BUILD_WORKDIR}/$(date +%s)/build" && scripts/run-integration-tests.sh
   ```

3. Generar reporte de verificación:
   - Estado pass/fail
   - Tamaños y checksums de artefactos
   - Metadatos de construcción (tiempo, recursos, git commit)
   - SBOM en formato CycloneDX

### Fase 6: Empaquetado de Artefactos (12m-13m)

1. Copiar artefactos finales a `dist/`:
   ```bash
   tar -czf "dist/${PROJECT_NAME}-$(git rev-parse --short HEAD)-$(uname -s).tar.gz" \
     -C "${BUILD_WORKDIR}/$(date +%s)/build/artifacts" .
   ```

2. Generar `release-notes.md` con:
   - Resumen de construcción
   - Cambios de dependencias desde última construcción
   - Problemas conocidos
   - Resultados de verificación

## Reglas de Oro

1. **Inmutabilidad**: Nunca modificar archivos fuente durante la construcción. Todas las transformaciones suceden en workspace aislado.

2. **Reproducibilidad**: Cada construcción debe generar un `build-manifest.json` capturando entradas exactas (git commit, versiones de dependencias, flags de compilador, hashes de entorno).

3. **Aislamiento de fallo**: Un componente de construcción fallido no debe corromper workspace ni prevenir rollback.

4. **Bloqueo de dependencias**: Todas las resoluciones de dependencias deben producir lockfiles (`package-lock.json`, `Pipfile.lock`, `Cargo.lock`, etc.) comprometidos en manifiesto de construcción.

5. **Seguridad paralela**: Pasos de construcción sin relaciones de dependencia pueden ejecutarse concurrentemente, pero no deben compartir estado mutable (no `~/.npm`, `~/.cargo`, etc.). Usar `HOME="${BUILD_WOMEDIR}/$(date +%s)/home"`.

6. **Verificación obligatoria**: Construcción considerada exitosa solo si:
   - Todos los artefactos coinciden con checksums (o generados frescos sin checksum previo)
   - Pruebas de integración pasan (si existe directorio tests)
   - Generación de SBOM exitosa

7. **Límites de recursos**: Forzar `MAX_PARALLEL_JOBS` y límites de memoria vía `ulimit -v` y `timeout` para prevenir OOM kills.

8. **Retención de logs**: Mantener todos logs de construcción en `logs/` con rotación. Nunca descartar logs post-verificación.

9. **Preparación de rollback**: Antes de cada fase de construcción, crear snapshot de workspace actual via:
   ```bash
   tar -czf "snapshots/$(date +%s)-pre-${PHASE}.tar.gz" "${BUILD_WORKDIR}"
   ```

10. **Escaneo de seguridad**: Ejecutar OWASP dependency-check en todas dependencias resueltas antes que comience la construcción:
    ```bash
    dependency-check --project "$PROJECT_NAME" --scan "${BUILD_WORKDIR}/$(date +%s)/source" --format JSON --out "${BUILD_WORKDIR}/$(date +%s)/vuln-report.json"
    ```

## Ejemplos

### Ejemplo 1: Inicializar construcción de microservicio

**Entrada:**
```bash
openclaw build init my-research-service
```

**`build.yaml` generado:**
```yaml
project: my-research-service
version: 0.1.0
components:
  - name: api
    type: nodejs
    path: services/api
    build: npm ci && npm run build
    artifacts:
      - dist/bundle.js
      - package.json
  
  - name: worker
    type: python
    path: services/worker
    build: pip install -r requirements.txt && python setup.py bdist_wheel
    artifacts:
      - dist/*.whl
      - requirements.txt

  - name: ui
    type: react
    path: ui/dashboard
    build: npm run build
    artifacts:
      - build/
      - static/

integrations:
  - after: ["api", "worker"]
    run: scripts/run-e2e-tests.sh

docker:
  base_image: node:18-alpine
  multi_stage: true
```

### Ejemplo 2: Ejecutar construcción de artefacto de investigación

**Entrada:**
```bash
openclaw build execute build.yaml
```

**Salida:**
```
[INFO] 2025-06-12 10:15:23 - Starting build: my-research-service v0.1.0
[INFO] Workspace: /tmp/builds/1718204123
[PHASE] Dependency resolution (30s)
[OK] api: npm dependencies resolved (142 packages)
[OK] worker: pip dependencies resolved (67 packages)
[OK] ui: npm dependencies resolved (203 packages)
[PHASE] Build planning (45s)
[PLAN] 3 stages, 1 parallel job, estimated 4m 23s
[PHASE] Build execution (4m 12s)
[RUN] api: STARTED (job 1/3)
[RUN] worker: STARTED (job 2/3)
[RUN] ui: WAITING (dependency: api, worker)
[OK] worker: PASSED (2m 34s) - artifacts: dist/worker-0.1.0-py3-none-any.whl
[OK] api: PASSED (3m 01s) - artifacts: dist/bundle.js
[RUN] ui: STARTED (job 1/1)
[OK] ui: PASSED (1m 11s) - artifacts: build/
[PHASE] Verification (15s)
[VERIFY] All checksums matched (12/12)
[VERIFY] Integration tests: 45 passed, 0 failed
[VERIFY] SBOM generated: sbom/cdx.json
[DONE] Build successful. Artifacts in dist/
[SUMMARY] my-research-service-abc123-linux.tar.gz (145 MB)
```

### Ejemplo 3: Verificar artefactos después de intervención manual

**Entrada:**
```bash
openclaw build verify dist/my-research-service-abc123-linux.tar.gz
```

**Salida:**
```
[VERIFY] Extracting: dist/my-research-service-abc123-linux.tar.gz
[VERIFY] Loading manifest: build-manifest.json
[CHECK] api/bundle.js - SHA256: 7f8c9d... (expected: 7f8c9d...) ✓
[CHECK] worker-0.1.0-py3-none-any.whl - SHA256: 3a1b2c... (expected: 3a1b2c...) ✓
[CHECK] ui/build/index.html - SHA256: 9e8f7a... (expected: 9e8f7a...) ✓
[CHECK] sast-report.json - Present ✓
[CHECK] sbom/cdx.json - Valid JSON ✓
[RESULT] 12/12 checks passed
[STATUS] VERIFIED
```

### Ejemplo 4: Rollback a estado de construcción anterior

**Entrada:**
```bash
openclaw build rollback snapshots/1718204123-pre-phase2.tar.gz
```

**Salida:**
```
[ROLLBACK] Found snapshot: snapshots/1718204123-pre-phase2.tar.gz
[WARN] Current workspace has uncommitted changes. Create commit? (y/n) y
[COMMIT] Created backup commit: rollback-backup-1718204156
[RESTORE] Extracting snapshot to current directory...
[RESTORE] Overwriting 247 files
[RESTORE] Restored build-manifest.json from snapshot
[INFO] Workspace restored to state before Phase 2
[SUGGEST] Re-run: openclaw build execute build.yaml --resume-from phase2
```

### Ejemplo 5: Generar reporte de conflicto de dependencias

**Entrada:**
```bash
openclaw build deps api
```

**Salida:**
```
[DEPSCAN] Analyzing: services/api/package.json
[GRAPH] Nodes: 142 | Edges: 287
[CONFLICTS] 3 potential issues:
  1. lodash (required: ^4.17.20, ^4.17.15 by axios)
  2. react (required: ^18.2.0, ^17.0.2 by @testing-library/react)
  3. typescript (required: ^5.0.0, ^4.9.5 by eslint)
[VULNS] 5 CVEs detected:
  - minimist 1.2.5 (CVE-2021-44906) - HIGH
  - debug 4.3.2 (CVE-2021-23337) - MEDIUM
[RECOMMEND] Run: openclaw build execute build.yaml --fix-deps --upgrade-minor
```

### Ejemplo 6: Construcción matrix a través de entornos

**Entrada:**
```bash
openclaw build matrix .github/workflows/matrix.yml
```

**Config matrix:
```yaml
dimensions:
  os: [ubuntu-latest, macos-latest, windows-latest]
  node: [18, 20, 22]
  python: [3.9, 3.11]
exclude:
  - os: windows-latest
    python: 3.11  # No soportado aún en CI windows
strategy: all-combinations
```

**Salida:**
```
[MATRIX] Configuration: 3×3×2 = 18 jobs (excluyendo 1) = 17 jobs
[PLAN] Lotes seriales (4 jobs cada uno)
  Batch 1: ubuntu-18/3.9, ubuntu-18/3.11, ubuntu-20/3.9, ubuntu-20/3.11
  Batch 2: macos-18/3.9, macos-18/3.11, macos-20/3.9, macos-20/3.11
  Batch 3: windows-18/3.9, windows-20/3.9 (restantes: 5)
[EXEC] Batch 1 lanzado (4 parallel jobs)
[RESULT] 17/17 passed, 0 failed, 0 skipped (25m 34s)
[ARTIFACTS] Uploaded: matrix-report.html, per-env logs/
```

## Comandos de Rollback

### Rollback completo de workspace

```bash
# Listar snapshots disponibles
ls -1rt snapshots/ | tail -10

# Restaurar a snapshot específico
openclaw build rollback snapshots/1718204123-pre-phase2.tar.gz

# Rollback automático en detección de estado corrupto
openclaw build rollback --auto --threshold 3  # auto-rollback después de 3 fallos
```

### Rollback parcial de artefactos

```bash
# Rollback solo del componente 'api' desde última construcción buena conocida
openclaw build rollback \
  --component api \
  --from-manifest build-manifest.json.bak \
  --target dist/api/
```

### Rollback de base de datos/migraciones

Si la construcción incluye migraciones de base de datos, Auto Builder genera scripts de rollback correspondientes:

```bash
# Después de construcción, tabla migrations actualizada. Para revertir:
openclaw build rollback-migrations \
  --target ts:20240612_1015_add_user_table \
  --dry-run  # Muestra SQL sin ejecutar

# Ejecutar rollback
openclaw build rollback-migrations --target ts:20240612_1015_add_user_table
# Equivalente a: dolphin-schema-cli down --target ts:20240612_1015_add_user_table
```

### Rollback de archivos de configuración

```bash
# Si build.yaml modificado durante construcción, restaurar backup
openclaw build rollback-config build.yaml
# Restaura desde: build.yaml.auto-builder.bak
```

### Rollback de lockfiles de dependencias

```bash
# Regenerar lockfiles desde snapshot de resolución de dependencias anterior
openclaw build rollback-deps \
  --manifest build-manifest.json \
  --output lockfiles/
# Restaura: package-lock.json, Pipfile.lock, Cargo.lock desde hashes registrados
```

### Desinstalación/reversión completa

```bash
# Remover todos artefactos de construcción, logs y workspace temporal
openclaw build clean --all --aggressive
# Equivalente a:
rm -rf /tmp/builds/*/ "${BUILD_WORKDIR}"
find . -name "*.auto-builder.bak" -delete
docker system prune -af  # si --docker-cleanup habilitado
```

## Solución de Problemas

### Problema: "Dependency resolution failed: conflicts detected"

**Fix**:
```bash
# Ver reporte de conflicto detallado
openclaw build deps --format markdown > conflicts.md

# Auto-resolver con rangos de versiones semánticas
openclaw build execute build.yaml --resolve-conflicts strategy=widest

# Override manual: editar build.yaml y especificar versiones exactas
```

### Problema: "Build hangs at parallel job limit"

**Fix**:
```bash
# Ver paralelismo actual
ps aux | grep -E 'make|npm|cargo|pip' | wc -l

# Aumentar si sistema tiene capacidad
export MAX_PARALLEL_JOBS=12
openclaw build execute build.yaml

# Reducir si alcanzando límites de recursos
export MAX_PARALLEL_JOBS=2
```

### Problema: "Verification failed: checksum mismatch"

**Diagnóstico**:
```bash
# Comparar esperado vs actual
jq -r '.artifacts[] | "\(.path) \(.sha256)"' build-manifest.json | sort > expected.txt
find artifacts -type f -exec sha256sum {} \; | sort > actual.txt
diff -u expected.txt actual.txt || true
```

**Causas comunes**:
- Construcción no determinista (timestamps, IDs aleatorios). Añadir `SOURCE_DATE_EPOCH=1` a entorno de construcción.
- Modificación post-construcción. Verificar no haya scripts `postprocess` ejecutándose.
- Drift de dependencias. Re-ejecutar con `--strict-lock`.

**Override manual** (si artefacto realmente correcto):
```bash
openclaw build verify --update-manifest dist/my-artifact.tar.gz
# Regenera checksums en manifiesto desde artefactos actuales
```

### Problema: "SnapshotNotFoundError after crash"

**Recuperación**:
```bash
# Buscar cualquier snapshot parcial
find /tmp/builds -name "*.tar.gz" -mtime -1 -ls

# Si encontrado, restore manual
tar -xzf /tmp/builds/1718204123/snapshots/pre-phase3.tar.gz -C /tmp/builds/restore-1718204123
rsync -av /tmp/builds/restore-1718204123/ ./
```

### Problema: "Docker daemon not accessible inside build"

**Fix**: Auto Builder usa Docker cuando necesario. Asegurar:
```bash
# Usuario actual en grupo docker
groups $USER | grep docker || sudo usermod -aG docker $USER
# Luego re-login

# Docker ejecutándose
systemctl status docker || sudo systemctl start docker

# Test
docker run --rm alpine:latest echo ok
```

### Ajuste de rendimiento

Para proyectos grandes (>500 deps, >100 módulos):

```bash
# Aumentar jobs paralelos (monitorear uso de memoria)
export MAX_PARALLEL_JOBS=$(nproc)

# Usar cache local de paquetes (npm/pip/cargo ya lo hacen)
# Pero asegurar cache en SSD, no NFS
export NPM_CONFIG_CACHE="/dev/shm/npm-cache"
export PIP_CACHE_DIR="/dev/shm/pip-cache"

# Omitir scan de vulnerabilidades para velocidad (no recomendado para prod)
openclaw build execute build.yaml --skip-vuln-scan

# Usar ccache para construcciones C/C++
export CCACHE_DIR="/dev/shm/ccache"
```

### Modo debug

```bash
# Logging verbose
openclaw build execute build.yaml --log-level DEBUG 2>&1 | tee build-debug.log

# Mantener workspace fallido para inspección
openclaw build execute build.yaml --keep-failed-workspace

# Single-stepping: ejecutar cada fase interactivamente
openclaw build execute build.yaml --interactive
# Luego en cada pausa: [ENTER] para continuar, [s] para skip, [q] para quit
```

## Variables de Entorno

| Variable | Default | Propósito |
|----------|---------|----------|
| `BUILD_WORKDIR` | `/tmp/builds` | Workspace raíz de construcción |
| `MAX_PARALLEL_JOBS` | `4` | Máximo jobs de construcción concurrentes |
| `OPENCLAUDE_API_KEY` | - | Resolución de conflictos asistida por IA |
| `BUILD_TIMEOUT` | `3600` | Timeout global (segundos) |
| `ARTIFACT_RETENTION_DAYS` | `30` | Auto-clean de artefactos antiguos |
| `DOCKER_REGISTRY` | - | Push de imágenes multi-stage |
| `CI` | `false` | Set true en CI para habilitar comportamientos específicos de CI |
```