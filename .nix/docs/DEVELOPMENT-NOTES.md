# Nix Development Environment - Discovery Record

This document records the key discoveries, design decisions, and learnings from integrating Nix into Ghost's development and CI workflows.

## Overview

This integration provides deterministic development environments and reproducible Docker builds using Nix, with Cachix binary caching for CI performance.

**Key Results:**
- Native development: 2-3s startup vs Docker's ~60s
- CI performance: Early measurements show ~6min warm cache vs ~29min cold (limited data)
- Multi-arch: Free ARM64 builds via GitHub's native runners

## File Structure

```
flake.nix, flake.lock         # Root orchestration
.nix/
├── docs/                     # Documentation
├── devShell.nix             # Dev environment definition
├── precache-package/        # CI precaching tool
├── process-compose.yaml     # Native service orchestration
└── treefmt.nix              # Code formatting

.docker-nix/
├── default.nix              # Docker build instructions
└── etc/                     # Container runtime files (/etc/passwd, etc.)

.envrc                       # direnv auto-loader
```

**Design decision:** Docker build source excludes `.nix/` and `flake.{nix,lock}` so that changes to dev tooling or documentation don't trigger 30+ minute Docker rebuilds. Only Ghost source and `.docker-nix/` affect Docker images.

## Development Environment Discoveries

### Python/distutils for node-gyp

**Problem:** Native modules (sqlite3, sharp, re2) failed to compile with `ModuleNotFoundError: No module named 'distutils'`.

**Investigation:** Python 3.13+ removed the `distutils` module. Node.js bundles its own Python for node-gyp, but Ghost's dependencies use an older node-gyp version that requires distutils.

**Discovery:** node-gyp checks multiple environment variables for Python path. Setting just one isn't enough—you need all three:

```nix
pythonWithSetuptools = pkgs.python312.withPackages (ps: [ ps.setuptools ]);

# All three required - node-gyp has multiple detection paths
export PYTHON="${pythonWithSetuptools}/bin/python3"
export npm_config_python="${pythonWithSetuptools}/bin/python3"
export GYP_PYTHON="${pythonWithSetuptools}/bin/python3"
```

### process-compose as Docker Alternative

**Discovery:** Running Ghost's services natively eliminates Docker VM overhead on macOS.

**Performance comparison:**

| Aspect | Docker Compose | process-compose |
|--------|----------------|-----------------|
| Startup | ~60s | ~10s |
| File I/O (macOS) | Slow (osxfs translation) | Native speed |
| Memory | ~4GB overhead | Minimal |
| Debugging | Attach to container | Native tools |

All services (MySQL via Unix socket, Redis, Mailpit, Ghost) run as native processes. Dev data in `.dev-data/` (gitignored). Reset: `rm -rf .dev-data/`.

## Docker Build Discoveries

### Shebang Patching

**Problem:** Build scripts like `tsc` and `vite` failed with "command not found" even though they were in PATH and symlinks existed.

**Investigation:** Node.js scripts in `node_modules/.bin/` use `#!/usr/bin/env node` shebangs. In Nix's build sandbox, `/usr/bin/env` doesn't exist—the sandbox is completely isolated from the host system.

**Solution:** Apply standard Nix shebang patching to `node_modules`:

```nix
buildPhase = ''
  yarn install --frozen-lockfile --offline
  patchShebangs node_modules  # Rewrites #!/usr/bin/env to Nix store paths
'';
```

The `patchShebangs` function recursively rewrites all env-based shebangs to direct paths like `#!/nix/store/abc123.../bin/node`.

### Yarn Workspaces and PATH Shadowing

**Discovery:** Yarn hoists most dependencies to root `node_modules/`, but some end up in workspace-local directories (e.g., `apps/admin/node_modules/vite/`). Yarn creates symlinks in workspace-local `.bin` directories that shadow the root `.bin` in PATH.

After shebang patching, these symlinks work correctly because they point to executables with fixed Nix store paths. Before patching, the symlinks would execute scripts with broken `#!/usr/bin/env` shebangs.

### Native Module Compilation Strategy

**Problem:** Running all postinstall scripts (4000+ packages) caused sandbox violations—arbitrary scripts trying to access the network, write to /tmp, etc.

**Discovery:** Skip all postinstall scripts (`dontYarnBuild = true`), then manually compile only critical native modules:

```nix
buildPhase = ''
  export HUSKY=0
  yarn install --frozen-lockfile --offline
  patchShebangs node_modules

  # Manual compilation with explicit control
  (cd node_modules/sqlite3 && ../.bin/node-gyp rebuild)
  (cd node_modules/sharp && ../.bin/node-gyp rebuild --directory=src)
  (cd node_modules/re2 && ../.bin/node-gyp rebuild)

  # Cleanup build artifacts (~150-200MB savings)
  find node_modules -name "*.o" -delete
  find node_modules -name "*.a" -delete
  find node_modules -type d -name "obj.target" -exec rm -rf {} +
'';

dontYarnBuild = true;
```

**Why this works:** Avoids sandbox violations from random scripts, gives explicit control over compilation, and uses the patched node-gyp from `node_modules` (which already has correct Python path).

### Copying Dotfiles in Docker extraCommands

**Discovery:** The shell glob `*` doesn't match dotfiles. `cp ${ghost-app}/* dest/` silently skips `.github/`, `.npmrc`, etc.

**Solution:** Use `/.` notation to copy directory contents including hidden files:

```nix
extraCommands = ''
  mkdir -p home/ghost
  cp -r ${ghost-app}/. home/ghost/  # /.  instead of /*
  chmod -R +w home/ghost
'';
```

### Image Size Optimization Journey

**Initial problem:** 18.6GB Docker image.

**Discovery 1:** `ghost-app` was included in both `contents` (3.19GB) and copied via `extraCommands` (4.06GB). Only copy it once via `extraCommands`.

**Result:** 18.6GB → 10.1GB (~45% reduction)

**Discovery 2:** Build-only dependencies don't need to be in runtime image:
- Python 3.12 + setuptools (~400MB) - Only for node-gyp compilation
- GCC C++ stdlib (~100MB) - Only for compilation
- git, curl, tar, jq (~50MB) - Not used at runtime

**Result:** Additional ~550-700MB savings

**Current state:** ~10GB (still includes all source + devDependencies for hot reload in development mode)

### Automatic Layer Optimization

**Discovery:** Nix's `buildLayeredImage` with `maxLayers = 100` automatically optimizes layer boundaries based on runtime closure analysis.

Unlike Dockerfiles where layer boundaries are fixed to build steps, Nix analyzes the dependency graph and groups files by how frequently they change together:

```nix
maxLayers = 100;  # Allows up to 100 optimized layers
```

**Practical impact:** Subsequent registry pushes only upload changed layers. Without manual Dockerfile optimization, developers benefit from efficient layer splitting automatically.

**How it works:** Nix computes the closure (all dependencies) of each package, then uses heuristics to partition the closure into layers that minimize expected data transfer on updates.

### Development vs Production Mode

**Design decision:** Docker image runs in development mode with `yarn dev`, which uses `tsx` to transpile TypeScript on-the-fly.

**Rationale:** Matches official `.docker/Dockerfile` behavior for functional parity. Ghost's TypeScript files don't need pre-compilation because `tsx` handles them at runtime. The `ghost-core-tsc-builder` stage exists but isn't strictly necessary for dev mode.

For production, would need to enable tsc pre-compilation and change CMD to direct node execution.

## Runtime Container Discoveries

### System Files Required from Scratch

**Problem:** Container processes failed with various system errors:
- `Error: spawn ps ENOENT` - Missing `ps` command
- `SystemError: uv_os_get_passwd returned ENOENT` - No `/etc/passwd`
- Nx Daemon failures - Missing `/tmp` directory

**Investigation:** Unlike traditional Docker base images (e.g., `node:bullseye-slim`), Nix's `dockerTools.buildLayeredImage` starts from **absolute scratch**—no base filesystem at all.

**Discovery:** System files that Debian/Alpine provide automatically must be created explicitly:

```nix
etcFiles = pkgs.runCommand "docker-etc-files" {} ''
  mkdir -p $out/etc
  cp ${./etc/passwd} $out/etc/passwd
  cp ${./etc/group} $out/etc/group
  cp ${./etc/nsswitch.conf} $out/etc/nsswitch.conf
'';

contents = [
  etcFiles
  pkgs.procps      # ps command (Node.js child_process needs it)
  pkgs.findutils   # find, xargs
  pkgs.gnugrep     # grep
  pkgs.which       # which
];

extraCommands = ''
  mkdir -p tmp
  chmod 1777 tmp  # Sticky bit + rwxrwxrwx (Nx daemon requirement)
'';
```

### Environment Variables for Ghost in Containers

**NX_DAEMON Discovery:**

**Problem:** Nx watch commands failed with "Daemon is not running. The watch command is not supported without the Nx Daemon."

**Investigation:** Nx explicitly disables the daemon in Docker containers by default. From `node_modules/nx/src/daemon/client/client.js`:

```javascript
function isDocker() {
  try {
    statSync('/.dockerenv');
    return true;
  } catch {
    try {
      return readFileSync('/proc/self/cgroup', 'utf8')?.includes('docker');
    } catch { }
    return false;
  }
}

// Disables unless NX_DAEMON=true explicitly set
if ((isCI() || isDocker()) && env !== 'true') {
  this._enabled = false;
}
```

**Discovery:** Must explicitly opt-in via environment variable in container config.

**GHOST_DEV_IS_DOCKER Discovery:**

**Problem:** Ghost listened on `127.0.0.1:2368`, making it inaccessible from outside the container.

**Investigation:** Ghost's config loader (`ghost/core/core/shared/config/loader.js`) loads different config files based on this variable. When true, it loads `config.development.docker.json` which sets `server.host` to `0.0.0.0` instead of `127.0.0.1`.

**PATH Discovery:**

**Problem:** Subshells spawned by `concurrently` couldn't find `nx` and other executables, causing infinite error loops.

**Investigation:** While yarn commands automatically add `node_modules/.bin` to PATH, subshells don't inherit this. The container's PATH only included system binaries from Nix packages.

**Discovery:** Must explicitly prepend `node_modules/.bin` to PATH in container environment:

```nix
Env = [
  "NX_DAEMON=true"
  "GHOST_DEV_IS_DOCKER=true"
  "PATH=/home/ghost/node_modules/.bin:/usr/bin:/bin:${nodejs}/bin:${pkgs.yarn}/bin:..."
];
```

### Memory Requirements

**Discovery:** Ghost in dev mode runs multiple concurrent build processes (Ember build, 4+ React/Vite builds, Nx daemon, TypeScript compilation). Container needs 16GB RAM minimum for 48GB host systems. If container is killed with SIGKILL during startup, increase Docker Desktop's memory allocation.

## Multi-Architecture Discoveries

### Free Multi-Arch with GitHub Actions

**Discovery:** As of January 2025, GitHub provides free ARM64 runners (`ubuntu-24.04-arm`) for public repositories.

**Result:** Native multi-arch builds with zero QEMU emulation, zero paid runners:

```yaml
matrix:
  - arch: x86_64, runner: ubuntu-latest
  - arch: aarch64, runner: ubuntu-24.04-arm
```

Same flake, different `--system`, both push to Cachix, Docker manifest combines them automatically.

## CI/CD Discoveries

### Source Determinism Challenge

**Problem:** Local builds and CI builds produce different derivation hashes, preventing cache reuse.

**What we tried:**

1. `builtins.fetchGit { url = ./.; }` - Still impure, depends on local HEAD
2. `git archive` in derivation - Fails in sandbox, `.git` not available
3. `fetchFromGitHub` with commit hash - Can't test uncommitted changes

**Discovery:** Pure source fetching for git repos with submodules requires choosing between:
- Remote fetch (no local iteration)
- Impure local evaluation (different hashes per checkout)

**Design decision:** Accept that CI builds are canonical. Use simple `src = ./.;` for flexibility during local development. First CI run populates Cachix, all subsequent CI runs pull from Cachix with identical hashes. Local builds are for development iteration, not cache population.

### Precaching Evolution

**Initial approach (complex):** Manually replicate `actions/checkout@v5` behavior:
- Phase 1: Fetch from GitHub with fake hash (FOD)
- Phase 2: Fetch again with correct hash
- Phase 3: Import `docker.nix` from checkout (IFD)
- Phase 4: Build and push to Cachix
- Result: ~150 lines of bash, multiple temp files, escaping hell

**Key realization:** Flake URLs are a uniform interface!
- `github:owner/repo/rev` - Fetches deterministically from GitHub
- `git+file://$PWD` - Fetches deterministically from local git
- Both use Nix's native fetching machinery
- Both produce identical hashes for the same commit

**Final solution (simple):**
```bash
nix run .#precache-package <flake-url> <output>
# One command, ~30 lines, no phases, no IFD, no hash discovery
```

**The insight:** We don't need to replicate anything. Nix already knows how to fetch git repos deterministically. We just need to use the right flake URL format.

**Warning system:** The precache tool blocks path-based URLs (`.`, `./path`) by default since they include uncommitted changes, which creates orphaned cache entries. Use `--allow-uncommitted` to override (not recommended).

**Example - verifying cache warmth:**
```
$ nix run .#precache-package "git+file://$PWD?rev=$(git rev-parse HEAD)" packages.aarch64-linux.dockerImage

Building from flake...
✅ Build succeeded: /nix/store/7mbif79mb2i706fj6f1l9r452n1mn3d1-ghost.tar.gz

Pushing to binary cache...
Nothing to push - all store paths are already on Cachix.

✅ Successfully precached to hello-stocha!

# Takes ~3 seconds when already cached
```

This provides instant confirmation that CI will get a cache hit.

### GitHub PR Merge Commits and Cache Misses

**Problem:** Initial CI runs showed no cache hits despite local precaching, and derivation hashes didn't match between local and CI builds.

**Investigation:** When GitHub Actions runs on pull requests, it uses synthetic merge commits (`refs/pull/N/merge`) that combine the PR branch with the current base branch. These commits:
- Don't exist in the local repository
- Change with every base branch update
- Are ephemeral (deleted after PR closes)
- Have different source hashes than the actual PR commits

**Example:**
```bash
# Local commit
git rev-parse HEAD
# → abc123 (your actual commit)

# GitHub Actions on PR
echo ${{ github.sha }}
# → merge789 (synthetic merge commit at refs/pull/1/merge)

# Different commits = different source hashes = cache miss
nix eval .#packages.aarch64-linux.dockerImage.drvPath
# Local: /nix/store/hash1...
# CI:    /nix/store/hash2...  # DIFFERENT!
```

**Solution:** Use the actual head SHA for PRs instead of the merge commit:

```yaml
# In GitHub Actions workflow
SHA="${{ github.event.pull_request.head.sha || github.sha }}"
nix build "github:${{ github.repository }}/$SHA#packages.${{ matrix.system }}.dockerImage"
```

**Why this matters:**
- ✅ Local precaching and CI use identical commits
- ✅ Cache persists after PR merge (not orphaned)
- ✅ Derivation hashes match between local and CI
- ✅ `precache-package` tool works as intended

**Before fix:**
```bash
# Local precache
nix run .#precache-package "git+file://$PWD?rev=abc123" ...
# Pushes to Cachix

# CI builds merge commit merge789
# → Cache miss, rebuilds everything ❌
```

**After fix:**
```bash
# Local precache
nix run .#precache-package "git+file://$PWD?rev=abc123" ...
# Pushes to Cachix

# CI builds actual commit abc123
# → Cache hit! ✅
```

This was the root cause of apparent cache failures during development. The Nix infrastructure was working correctly—builds just used different source commits.

### What "self" Really Means

**Confusion:** What does `src = self` in a flake include?
- **Not:** Current filesystem state
- **Not:** Only committed changes (in the Git sense)
- **Actually:** Git-**tracked** files in their current state

This means:
- ✅ Tracked files (whether committed or not)
- ❌ Untracked files (ignored by git)
- ❌ `.git` directory
- ⚠️ Uncommitted changes to tracked files (included!)

**For precaching:** Use `git+file://$PWD` (committed state only), not `.` (includes uncommitted changes).

### Source Filtering for Docker Builds

**Problem:** Changes to documentation, Nix tooling, or CI configs triggered full 30-minute Docker rebuilds.

**Solution:** Exclude non-application files from the source derivation:

```nix
filter = path: type:
  let
    baseName = baseNameOf path;
    relPath = pkgs.lib.removePrefix (toString self + "/") (toString path);
  in
  # Exclude Nix infrastructure, CI orchestration, and documentation
  baseName != "flake.nix" &&
  baseName != "flake.lock" &&
  baseName != "compose.yml" &&
  baseName != ".editorconfig" &&
  !(pkgs.lib.hasPrefix ".nix/" relPath) &&
  !(pkgs.lib.hasPrefix ".github/" relPath) &&
  !(pkgs.lib.hasPrefix ".vscode/" relPath) &&
  !(pkgs.lib.hasPrefix ".cursor/" relPath) &&
  !(pkgs.lib.hasPrefix ".claude/" relPath) &&
  !(pkgs.lib.hasPrefix "docs/" relPath) &&
  !(pkgs.lib.hasPrefix "adr/" relPath) &&
  !(pkgs.lib.hasSuffix ".md" baseName) &&
  (pkgs.lib.sources.cleanSourceFilter path type);
```

**Rationale:** Build orchestration and documentation don't affect application behavior. Only Ghost source code and `.docker-nix/` should invalidate builds.

### Precaching with Full Closure

**Problem:** Local precaching only pushed the final `dockerImage`, not intermediate build stages like `development-base`. When source changed, CI couldn't reuse the expensive dependency installation stage.

**Discovery:** `cachix watch-exec` monitors the Nix store during builds and pushes everything that gets built, including intermediate derivations.

**Solution:**
```bash
cachix watch-exec "$CACHE_NAME" -- nix build "$FLAKE_URL#$OUTPUT"
```

**Impact:** After local precache, CI can reuse `development-base` and other intermediate stages even when the final image hash changes due to source modifications.

**Additional optimization:** Check if output already exists in Cachix before building:

```bash
OUT_PATH=$(nix eval --raw "$FLAKE_URL#$OUTPUT.outPath")
if nix path-info --store "https://$CACHE_NAME.cachix.org" "$OUT_PATH" >/dev/null 2>&1; then
  echo "Already cached, skipping"
  exit 0
fi
```

**Why this works:** `nix eval` computes the output path without building. Querying with the store path (not the flake reference) correctly detects cached artifacts.

### Conditional Disk Cleanup in CI

**Problem:** GitHub Actions runners have limited disk space (~14GB free). Nix builds can require 25-30GB when building from scratch. However, cleanup operations (removing dotnet, Android SDK) take 30-60 seconds even when unnecessary.

**Solution:** Check if the Docker image is cached before freeing disk space:

```yaml
- name: Check if Docker image is cached
  run: |
    OUT_PATH=$(nix eval --raw "github:$REPO/$SHA#packages.$SYSTEM.dockerImage.outPath")
    if nix path-info --store "https://$CACHE.cachix.org" "$OUT_PATH" >/dev/null 2>&1; then
      echo "cache_hit=true"
    else
      echo "cache_hit=false"
    fi

- name: Free disk space (only if building from scratch)
  if: cache_hit == 'false'
  run: |
    sudo rm -rf /usr/share/dotnet
    sudo rm -rf /usr/local/lib/android
```

**Result:** Warm cache builds skip cleanup entirely, saving time. Cold cache builds get necessary space.

### Docker Image Push Optimization with skopeo

**Problem:** `docker load` takes 1m49s to load a 10GB tarball into the Docker daemon before tagging and pushing.

**Discovery:** `skopeo` can copy directly from tarball to registry without loading into Docker daemon.

**Solution:**
```bash
# Instead of:
docker load < result           # 1m49s
docker tag ghost:latest $TAG
docker push $TAG

# Use:
nix run nixpkgs#skopeo -- copy docker-archive:result docker://$TAG
```

**Benefits:**
- Saves ~2 minutes per build
- Uses `skopeo` from nixpkgs (no apt installation overhead)
- Direct tarball-to-registry copy

**Performance impact:** Combined with other optimizations, warm cache builds complete in ~3 minutes.

### Cachix Performance Results

Measurements from GitHub Actions across multiple iterations:

**Cold cache (first build):**
- Total workflow time: ~29 minutes
- Multi-arch build (x86_64 + ARM64 in parallel)

**Warm cache (final optimized):**
- Total workflow time: ~3 minutes
- 100% cache hit from Cachix
- Includes: checkout, Nix setup, download artifacts, push to registry

**Optimization progression:**
1. Initial (merge commits): ~29 min every build (no cache reuse)
2. After head SHA fix: ~6 min (cache reuse working)
3. After skopeo: ~5 min (eliminated docker load)
4. After skopeo from nixpkgs: ~4 min (eliminated apt-get install)
5. After conditional cleanup + full closure precache: ~3 min (optimal)

**Key insight:** Once Cachix is populated, builds become pure artifact downloads. No compilation occurs.

**Economic difference:**

Traditional Docker CI:
- Registry-based layer cache (limited effectiveness)
- Cache scoped per-PR or per-branch
- Frequent cache misses

Nix + Cachix:
- Content-addressed binary cache (global)
- One build populates cache for everyone
- Near-zero rebuild costs after first run

## Architecture Decisions

### Build Stage Strategy

**Design:** Multi-stage derivations mirroring Dockerfile structure:

1. `yarnOfflineCache` - FOD downloads all yarn dependencies once
2. `development-base` - yarn install + shebang patching (reused by all builders)
3. Workspace builders - Build individual apps (shade, framework, stats, posts, etc.)
4. `ghost-app` - Assemble all artifacts
5. `dockerImage` - Package into layered image

**Key insight:** The `development-base` stage does ALL dependency installation and shebang patching once. All subsequent builders reuse this via Nix's content-addressed store, making builds fast and cacheable.

**Debugging benefit:** Each stage is individually inspectable without rebuilding entire Docker image:

```bash
nix build .#packages.aarch64-linux.development-base
nix build .#packages.aarch64-linux.shade-builder
ls -la result/
```

### DRY Refactoring

Extracted common patterns to reduce duplication:

- `commonBuildAttrs` - All env vars + build inputs (eliminated ~80 lines)
- `commonBuildPhase` - Standard HOME/PATH setup (eliminated ~35 lines)
- `commonAdminDeps` - Shade + framework deps (eliminated 4 duplicates)

**Result:** ~150 lines removed, single source of truth for build configuration.

### Error Handling Philosophy

**Decision:** Remove all `|| true` error suppression.

**Rationale:** Silent failures hide build problems. If a builder fails to produce expected output, the build should fail immediately with a clear error, not silently continue and fail later with a confusing message.

All native module builds (sqlite3, sharp, re2) are required and must succeed.

### Why Not Standard Tools?

**Question:** "Shouldn't Cachix or Nix have precaching built-in?"

**Reality:** The precaching workflow should be standard, but isn't:
- Cachix focuses on receiving/serving, not orchestrating precaching
- Nix doesn't have `--push-to-cache` built-in
- The flake URL insight (uniform interface) isn't well-documented

**What we built:** A pattern that could be:
- A Cachix feature (`cachix precache <flake-url> <output>`)
- A Nix feature (`nix build --push-to <cache>`)
- A community template/convention

For now, it lives in `.nix/precache-package/`.

## Lessons Learned

### 1. Apply Standard Patterns

Standard Nix patterns (shebang patching, manual native module compilation, explicit env vars) are essential for Node.js monorepo builds. Ghost's complexity required careful application of these patterns but no novel techniques.

### 2. Cachix Changes CI Economics

Content-addressed binary cache with global sharing eliminates per-PR cache isolation. One build populates cache for everyone. Early measurements show significant speedup with warm cache.

### 3. Multi-Arch is Nearly Free with Nix

No complex QEMU emulation, no slow cross-compilation, no platform-specific Dockerfiles. Same flake builds both architectures natively on appropriate runners.

### 4. Hermetic Builds Prevent Future Failures

Docker builds can fail months later due to base image updates, registry flakes, repo changes, or expired GPG keys. Nix builds fail immediately or never—all inputs content-addressed, no network access during build, dependencies cryptographically verified.

### 5. Source Determinism is Harder Than Expected

The "simple" problem of "clean git checkout with submodules" has no perfect solution without sacrificing either local iteration speed or pure evaluation guarantees. Pragmatic answer: CI reproducibility matters most.

### 6. First Build Cost is Real

Cold cache: ~29 minutes measured in early testing. Mitigation: Let CI handle first builds, cache persists indefinitely (one-time cost), consider pre-populating before going live.

### 7. Explicit is Better Than Inherited

Docker base images hide dependencies (system utilities, env vars, default configs). Nix requires explicit declarations for everything. Result: Longer initial setup, zero surprises in production.

### 8. Nix Learning Curve Pays Dividends

Initial investment: Learn language, understand derivations, debug sandbox violations. Long-term payoff: Builds never randomly break, multi-arch for free, binary caching that actually works, confidence in deployments.

### 9. Measure Everything

Before claiming performance improvements, get real numbers with setup overhead included. Early measurements from fork (2 runs): cold cache ~29min, warm cache ~6min. Without comparable Docker baseline from same environment, focus on cache reliability (100% hit rate) rather than absolute time comparisons. More data needed for robust benchmarking.

## Value Proposition Summary

**Development:**
- 2-3s startup with native I/O (no Docker VM overhead)
- Native debugging, profilers, full performance
- Minimal memory overhead

**CI/CD:**
- Faster builds with warm cache (early measurements: ~6min vs ~29min cold)
- Free multi-arch (GitHub's ARM64 runners)
- Global binary cache (not per-PR)

**Reproducibility:**
- Bit-for-bit identical builds across machines
- Hermetic sandbox prevents external failures
- No "works on my machine" problems

**Maintainability:**
- Explicit dependencies (no base image surprises)
- Individual build stage debugging
- Content-addressed caching (unchanged stages never rebuild)

**Security:**
- Cryptographically verified dependencies
- No GPG key management (e.g., Stripe CLI from nixpkgs)
- Transparent supply chain
- Minimal runtime attack surface
