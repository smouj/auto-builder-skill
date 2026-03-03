---
name: auto-builder
description: AI-powered automated research and artifact builder with parallel execution, dependency management, and verification
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
  - OPENCLAUDE_API_KEY (optional, for AI-assisted planning)
  - BUILD_WORKDIR (default: /tmp/builds)
  - MAX_PARALLEL_JOBS (default: 4)
capabilities:
  - parallel_builds
  - dependency_resolution
  - artifact_generation
  - verification_pipeline
  - rollback_snapshots
---

# Auto Builder Skill

## Purpose

Auto Builder automates the complete lifecycle of research-driven software construction: from requirement gathering and dependency resolution through parallel compilation to artifact verification. Designed for OpenClaw's multi-agent workflows, it enables single-command orchestration of complex builds across heterogeneous environments (Docker, native, CI).

### Real Use Cases

1. **Multi-component research prototype** - Build simultaneously a Node.js backend, Python data pipeline, and React frontend when testing a new AI integration technique, with automatic dependency conflict detection between components.

2. **Patch verification pipeline** - After generating a security fix via research agent, automatically build all affected services, run integration tests, and generate diff reports comparing artifacts against baseline.

3. **Cross-platform artifact generation** - Produce Linux binaries, Docker images, and npm packages from a single research artifact definition, with platform-specific optimizations applied automatically.

4. **Dependency heatmap analysis** - During build, track which dependencies are actually used vs. declared, generating optimization recommendations and security vulnerability reports for further research.

5. **Reproducible research builds** - Lock all transitive dependencies, capture build environment hashes, and generate SBOMs (Software Bill of Materials) for academic reproducibility requirements.

## Scope

Auto Builder implements the following OpenClaw commands:

- `openclaw build init <project_name>` - Initialize build configuration
- `openclaw build plan <spec_file>` - Generate execution plan without building
- `openclaw build execute <plan_file>` - Execute validated build plan
- `openclaw build verify <artifact_manifest>` - Verify built artifacts against checksums
- `openclaw build rollback <snapshot_id>` - Restore to previous build state
- `openclaw build watch <target>` - Monitor file changes and trigger incremental builds
- `openclaw build deps <component>` - Resolve and visualize dependency graph
- `openclaw build matrix <config>` - Execute matrix builds across environments

## Detailed Work Process

### Phase 1: Configuration Discovery (0-30s)

1. **Input ingestion**: Accepts either:
   - `build.yaml` specification file (preferred)
   - Inline CLI arguments (legacy)
   - Interactive question prompt if neither present

2. **Environment detection**:
   ```bash
   # Actual commands run internally:
   uname -m && uname -s
   docker version --format '{{.Server.Version}}' 2>/dev/null || echo "docker: absent"
   node --version
   python3 --version
   make --version | head -1
   ```

3. **Workspace preparation**:
   ```bash
   mkdir -p "${BUILD_WORKDIR:-/tmp/builds}/$(date +%s)"
   cp -r "$PWD" "${BUILD_WORKDIR:-/tmp/builds}/$(date +%s)/source"
   ```

### Phase 2: Dependency Resolution (30s-2m)

1. Parse manifest files:
   - `package.json` → npm dependencies
   - `requirements.txt` / `pyproject.toml` → pip/poetry
   - `go.mod` → go modules
   - `Cargo.toml` → cargo
   - `pom.xml` → maven
   - `build.gradle` → gradle

2. Generate unified dependency graph:
   ```bash
   "scripts/deps-resolver.py" \
     --input "${BUILD_WORKDIR}/$(date +%s)/source" \
     --output "${BUILD_WORKDIR}/$(date +%s)/deps-graph.json" \
     --conflict-check \
     --vuln-scan \
     --lock
   ```

3. Detect circular dependencies and version conflicts before proceeding.

### Phase 3: Build Planning (2m-3m)

1. Create execution DAG (Directed Acyclic Graph) from:
   - Build specification targets
   - Dependency graph
   - Platform constraints

2. Determine parallelization strategy:
   ```bash
   "scripts/planner.py" \
     --deps "${BUILD_WORKDIR}/$(date +%s)/deps-graph.json" \
     --jobs "${MAX_PARALLEL_JOBS:-4}" \
     --strategy "resource-aware" \
     > "${BUILD_WORKDIR}/$(date +%s)/plan.json"
   ```

3. Output human-readable plan preview:
   ```
   Phase 1: base-deps [0/3]
   Phase 2: backend [0/2]
   Phase 3: frontend [0/1]
   Phase 4: integration [0/1]
   Estimated: 4m 23s
   ```

### Phase 4: Parallel Execution (3m-10m)

1. Execute build stages respecting DAG dependencies:
   ```bash
   "scripts/executor.py" \
     --plan "${BUILD_WORKDIR}/$(date +%s)/plan.json" \
     --workspace "${BUILD_WORKDIR}/$(date +%s)/build" \
     --log-level INFO \
     --fail-fast \
     --retry 2
   ```

2. Each build step:
   - Runs in isolated build context
   - Streams stdout/stderr to central log
   - Captures artifacts to `artifacts/<component>/`
   - Records environment hash (`sha256sum` of compiler, flags, inputs)
   - Updates manifest: `build-manifest.json`

3. On failure: immediate halt, save debug artifacts, generate error report.

### Phase 5: Verification (10m-12m)

1. **Hash verification**:
   ```bash
   "scripts/verify.py" \
     --manifest "${BUILD_WORKDIR}/$(date +%s)/build-manifest.json" \
     --alg sha256 \
     --strict
   ```

2. **Integration test** (if `tests/` directory exists):
   ```bash
   cd "${BUILD_WORKDIR}/$(date +%s)/build" && scripts/run-integration-tests.sh
   ```

3. Generate verification report:
   - Pass/fail status
   - Artifact sizes and checksums
   - Build metadata (time, resources, git commit)
   - SBOM in CycloneDX format

### Phase 6: Artifact Packaging (12m-13m)

1. Copy final artifacts to `dist/`:
   ```bash
   tar -czf "dist/${PROJECT_NAME}-$(git rev-parse --short HEAD)-$(uname -s).tar.gz" \
     -C "${BUILD_WORKDIR}/$(date +%s)/build/artifacts" .
   ```

2. Generate `release-notes.md` with:
   - Build summary
   - Dependency changes since last build
   - Known issues
   - Verification results

## Golden Rules

1. **Immutability**: Never modify source files during build. All transformations happen in isolated workspace.

2. **Reproducibility**: Every build must generate a `build-manifest.json` capturing exact inputs (git commit, dependency versions, compiler flags, environment hashes).

3. **Failure isolation**: A failed component build must not corrupt workspace or prevent rollback.

4. **Dependency locking**: All dependency resolutions must produce lockfiles (`package-lock.json`, `Pipfile.lock`, `Cargo.lock`, etc.) committed to build manifest.

5. **Parallel safety**: Build steps without dependency relationships can run concurrently, but must not share mutable state (no `~/.npm`, `~/.cargo`, etc.). Use `HOME="${BUILD_WOMEDIR}/$(date +%s)/home"`.

6. **Verification mandatory**: Build is considered successful only if:
   - All artifacts match checksums (or generated fresh with no prior checksum)
   - Integration tests pass (if tests directory exists)
   - SBOM generation succeeds

7. **Resource limits**: Enforce `MAX_PARALLEL_JOBS` and memory limits via `ulimit -v` and `timeout` to prevent OOM kills.

8. **Log retention**: Keep all build logs in `logs/` with rotation. Never discard logs post-verification.

9. **Rollback readiness**: Before each build phase, create snapshot of current workspace via:
   ```bash
   tar -czf "snapshots/$(date +%s)-pre-${PHASE}.tar.gz" "${BUILD_WORKDIR}"
   ```

10. **Security scanning**: Run OWASP dependency-check on all resolved dependencies before build starts:
    ```bash
    dependency-check --project "$PROJECT_NAME" --scan "${BUILD_WORKDIR}/$(date +%s)/source" --format JSON --out "${BUILD_WORKDIR}/$(date +%s)/vuln-report.json"
    ```

## Examples

### Example 1: Initialize a microservice build

**Input:**
```bash
openclaw build init my-research-service
```

**Generated `build.yaml`:**
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

### Example 2: Execute a research artifact build

**Input:**
```bash
openclaw build execute build.yaml
```

**Output:**
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

### Example 3: Verify artifacts after manual intervention

**Input:**
```bash
openclaw build verify dist/my-research-service-abc123-linux.tar.gz
```

**Output:**
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

### Example 4: Rollback to previous build state

**Input:**
```bash
openclaw build rollback snapshots/1718204123-pre-phase2.tar.gz
```

**Output:**
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

### Example 5: Generate dependency conflict report

**Input:**
```bash
openclaw build deps api
```

**Output:**
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

### Example 6: Matrix build across environments

**Input:**
```bash
openclaw build matrix .github/workflows/matrix.yml
```

**Matrix config:
```yaml
dimensions:
  os: [ubuntu-latest, macos-latest, windows-latest]
  node: [18, 20, 22]
  python: [3.9, 3.11]
exclude:
  - os: windows-latest
    python: 3.11  # Not yet supported on CI windows
strategy: all-combinations
```

**Output:**
```
[MATRIX] Configuration: 3×3×2 = 18 jobs (excluding 1) = 17 jobs
[PLAN] Serial batches (4 jobs each)
  Batch 1: ubuntu-18/3.9, ubuntu-18/3.11, ubuntu-20/3.9, ubuntu-20/3.11
  Batch 2: macos-18/3.9, macos-18/3.11, macos-20/3.9, macos-20/3.11
  Batch 3: windows-18/3.9, windows-20/3.9 (remaining: 5)
[EXEC] Batch 1 launched (4 parallel jobs)
[RESULT] 17/17 passed, 0 failed, 0 skipped (25m 34s)
[ARTIFACTS] Uploaded: matrix-report.html, per-env logs/
```

## Rollback Commands

### Full workspace rollback

```bash
# List available snapshots
ls -1rt snapshots/ | tail -10

# Restore to specific snapshot
openclaw build rollback snapshots/1718204123-pre-phase2.tar.gz

# Automated rollback on detection of corrupted state
openclaw build rollback --auto --threshold 3  # auto-rollback after 3 failures
```

### Partial artifact rollback

```bash
# Rollback only the 'api' component from last known good build
openclaw build rollback \
  --component api \
  --from-manifest build-manifest.json.bak \
  --target dist/api/
```

### Database/migration rollback

If build includes database migrations, Auto Builder generates corresponding rollback scripts:

```bash
# After build, migrations table updated. To revert:
openclaw build rollback-migrations \
  --target ts:20240612_1015_add_user_table \
  --dry-run  # Shows SQL without executing

# Execute rollback
openclaw build rollback-migrations --target ts:20240612_1015_add_user_table
# Equivalent to: dolphin-schema-cli down --target ts:20240612_1015_add_user_table
```

### Configuration file rollback

```bash
# If build.yaml modified during build, restore backup
openclaw build rollback-config build.yaml
# Restores from: build.yaml.auto-builder.bak
```

### Dependency lockfile rollback

```bash
# Regenerate lockfiles from previous dependency resolution snapshot
openclaw build rollback-deps \
  --manifest build-manifest.json \
  --output lockfiles/
# Restores: package-lock.json, Pipfile.lock, Cargo.lock from recorded hashes
```

### Complete uninstall/revert

```bash
# Remove all build artifacts, logs, and temporary workspace
openclaw build clean --all --aggressive
# Equivalent to:
rm -rf /tmp/builds/*/ "${BUILD_WORKDIR}"
find . -name "*.auto-builder.bak" -delete
docker system prune -af  # if --docker-cleanup enabled
```

## Troubleshooting

### Issue: "Dependency resolution failed: conflicts detected"

**Fix**:
```bash
# View detailed conflict report
openclaw build deps --format markdown > conflicts.md

# Auto-resolve with semantic versioning ranges
openclaw build execute build.yaml --resolve-conflicts strategy=widest

# Manual override: edit build.yaml and specify exact versions
```

### Issue: "Build hangs at parallel job limit"

**Fix**:
```bash
# Check actual parallelism
ps aux | grep -E 'make|npm|cargo|pip' | wc -l

# Increase if system has capacity
export MAX_PARALLEL_JOBS=12
openclaw build execute build.yaml

# Reduce if hitting resource limits
export MAX_PARALLEL_JOBS=2
```

### Issue: "Verification failed: checksum mismatch"

**Diagnosis**:
```bash
# Compare expected vs actual
jq -r '.artifacts[] | "\(.path) \(.sha256)"' build-manifest.json | sort > expected.txt
find artifacts -type f -exec sha256sum {} \; | sort > actual.txt
diff -u expected.txt actual.txt || true
```

**Common causes**:
- Build is non-deterministic (timestamps, random IDs). Add `SOURCE_DATE_EPOCH=1` to build environment.
- Post-build modification. Verify no `postprocess` scripts running.
- Dependency drift. Re-run with `--strict-lock`.

**Manual override** (if artifact actually correct):
```bash
openclaw build verify --update-manifest dist/my-artifact.tar.gz
# Regenerates checksums in manifest from current artifacts
```

### Issue: "SnapshotNotFoundError after crash"

**Recovery**:
```bash
# Search for any partial snapshots
find /tmp/builds -name "*.tar.gz" -mtime -1 -ls

# If found, manual restore
tar -xzf /tmp/builds/1718204123/snapshots/pre-phase3.tar.gz -C /tmp/builds/restore-1718204123
rsync -av /tmp/builds/restore-1718204123/ ./
```

### Issue: "Docker daemon not accessible inside build"

**Fix**: Auto Builder uses Docker when needed. Ensure:
```bash
# Current user in docker group
groups $USER | grep docker || sudo usermod -aG docker $USER
# Then re-login

# Docker running
systemctl status docker || sudo systemctl start docker

# Test
docker run --rm alpine:latest echo ok
```

### Performance tuning

For large projects (>500 deps, >100 modules):

```bash
# Increase parallel jobs (monitor memory usage)
export MAX_PARALLEL_JOBS=$(nproc)

# Use local package cache (npm/pip/cargo already do this)
# But ensure cache is on SSD, not NFS
export NPM_CONFIG_CACHE="/dev/shm/npm-cache"
export PIP_CACHE_DIR="/dev/shm/pip-cache"

# Skip vulnerability scan for speed (not recommended for prod)
openclaw build execute build.yaml --skip-vuln-scan

# Use ccache for C/C++ builds
export CCACHE_DIR="/dev/shm/ccache"
```

### Debug mode

```bash
# Verbose logging
openclaw build execute build.yaml --log-level DEBUG 2>&1 | tee build-debug.log

# Keep failed workspace for inspection
openclaw build execute build.yaml --keep-failed-workspace

# Single-stepping: run each phase interactively
openclaw build execute build.yaml --interactive
# Then at each pause: [ENTER] to continue, [s] to skip, [q] to quit
```

## Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `BUILD_WORKDIR` | `/tmp/builds` | Root build workspace |
| `MAX_PARALLEL_JOBS` | `4` | Max concurrent build jobs |
| `OPENCLAUDE_API_KEY` | - | AI-assisted conflict resolution |
| `BUILD_TIMEOUT` | `3600` | Global timeout (seconds) |
| `ARTIFACT_RETENTION_DAYS` | `30` | Auto-clean old artifacts |
| `DOCKER_REGISTRY` | - | Push multi-stage images |
| `CI` | `false` | Set true in CI to enable CI-specific behaviors |

```