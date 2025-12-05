# Nix for Ghost: The Pitch

## What This Is

A Nix-based build system that runs in parallel with Ghost's existing Docker CI. It provides:

- **Native development**: 2-3 second startup, no Docker VM overhead on macOS
- **Faster CI**: Early measurements show ~6min with warm cache vs ~29min cold (limited data)
- **Reproducible builds**: Mathematically guaranteed identical outputs
- **Distributed caching**: Build once anywhere (CI/dev's laptop, push to cache, everyone always "just downloads"
- **Free multi-arch**: Native multi-arch builds using GitHub's free runners
- **Optimized layer splitting**: Automatic heuristic layering minimizes registry transfer sizes

Existing workflows (`yarn dev`, Docker Compose) continue to work. This is additive, not a replacement.

## What Nix Is

Nix is three things:

Nix is a 20-year-old technology used at CERN, Replit, Shopify, and others. It has a learing curve, but it's [beloved](https://mitchellh.com/writing/nix-with-dockerfiles) by those in the know.

1. **A language** for authoring software packages and development environments
2. **A package repository** (nixpkgs) with 120,000+ packages—the largest, freshest package collection in existence
3. **A package manager** that builds and installs packages using the language and repository

All three work together to provide **content-addressed, reproducible builds**. Think of it as "Git for software dependencies"—same inputs always produce the same outputs, with cryptographic guarantees.

## The Problem

**Local development on macOS:**
- Docker's Linux VM makes file I/O slow (osxfs translation layer)
- 4GB+ memory overhead for Docker Desktop
- Startup time and virtualization overhead

**CI reliability:**
- Builds break randomly (apt repos change, npm registry flakes, base images update)
- Cache misses are common (branch-scoped, per-PR isolation)
- Developers can't help CI (local builds don't transfer to CI)

## The Solution

### Native Development (process-compose)

Instead of Docker containers, run everything natively:

```bash
ghost-dev  # 2-3 second startup

# Running natively:
# - MySQL 8.0 (Unix socket)
# - Redis 7.0
# - Mailpit
# - Ghost (yarn dev)
```

Native I/O, native debugging, minimal overhead. TUI for logs like Docker Compose.

### Content-Addressed Builds

Every package has a hash based on **all** its inputs:

```
hash(source_code + dependencies + build_script + environment)
```

Same hash = identical output. Not "probably the same"—**mathematically guaranteed**.

This enables:
- **Hermetic builds**: Sandboxed, no network access, can't fail due to external changes
- **Distributed caching**: If anyone builds hash `abc123`, everyone can download it
- **Cross-machine cache reuse**: Build on macOS → CI on Linux downloads the same artifact

### The Workflow

```bash
# Developer on fast local machine
git commit -m "feat: awesome"
# Optional precache - verify CI will get cache hit:
nix run .#precache-package "git+file://$PWD?rev=$(git rev-parse HEAD)" packages.aarch64-linux.dockerImage
# If already cached: 3 seconds ("Nothing to push - already on Cachix")
# If not cached: builds, then pushes to binary cache

git push

# CI (minutes later)
# - Computes hash: abc123 (same as local!)
# - Checks binary cache: found!
# - Downloads, runs tests
```

With Docker, this is impossible—local and CI have different contexts. With Nix, content-addressing guarantees the same hash.

## Early Measurements

From GitHub Actions runs in fork (limited data - 2 runs):

| Scenario | Time |
|----------|------|
| Build with cold/partial cache | ~29 min |
| Build with warm cache | ~6 min |

Local development: 2-3 second startup (vs Docker's variable timing and VM overhead).

More runs needed for reliable benchmarks.

## Why It Sounds Too Good

**It does.** Nix's guarantees sound like marketing because the industry has normalized dysfunction:

- "It worked yesterday" as an acceptable failure mode
- Random cache misses
- Repositories that change without warning
- Docker Desktop eating 4GB RAM

Nix solves these through **pure functional programming**—no mutable state, no network during builds, no hidden dependencies. That purity makes it harder to learn, but the guarantees are real.

**Nix is 20 years old.** Used at CERN, Replit, Shopify, Fly.io. The nixpkgs repository has 100,000+ packages—largest in existence. It hasn't gone mainstream because of the learning curve, not because it doesn't work.

## Security Benefits

Traditional CI needs additional tools for CVE scanning: Snyk, Trivy, Dependabot. Each adds complexity.

**Nix has this built-in:**

```
error: Package nodejs-22.1.0-abc123 is marked as insecure
  reason: CVE-2024-12345
  replacement: Update to nodejs-22.1.1
```

Build fails at evaluation time. No additional tooling. No delayed detection.

**Why:** Content-addressing makes security a mathematical property. NixOS security team marks vulnerable hashes as insecure. If your build references one, Nix refuses.

Example: XZ backdoor (2024) → Nix builds automatically failed for compromised hash. Update was one line in `flake.lock`.

## Binary Cache

Uses [Cachix](https://cachix.org), a Nix binary cache SaaS at ~$100/month. Provides:
- Secure storage with authentication
- Global CDN
- Unlimited retention (artifacts never expire)
- Fully owned by the organization

Alternative: Self-hosted binary cache (more complex, but possible).

## The Tradeoffs

**Costs:**
- Learning curve for 1-2 maintainers
- First build takes time (~29min measured)
- Nix store uses disk space
- ~$100/month for binary cache

**Benefits:**
- 2-3 second dev startup
- Native I/O performance (no VM)
- Faster CI with warm cache (~6min measured, limited data)
- Reproducibility eliminates entire bug classes
- Free multi-arch
- Built-in CVE detection

For Ghost's contribution volume (hundreds of CI runs/week), the time savings are significant.

## What This Isn't

**Not** a proposal to rewrite Ghost in Nix. **Not** forcing the team to learn a new language.

Most developers:
- Keep using `yarn dev`, `docker-compose up`
- Benefit from faster CI (built by others)
- Optionally try `nix develop` (faster, native)

1-2 people maintain it, like any infrastructure (GitHub Actions, Dockerfiles).

## Adoption Risk

**Current state:** Both CI workflows run in parallel
- Existing Docker CI remains trusted
- Nix CI proves speed/reliability
- Easy rollback: delete one workflow file

**Phase 2:** Developers opt in to local dev shell
- No pressure to switch
- Try it, keep it if useful

**Phase 3:** If confident, make Nix CI primary
- One-line change (rename workflow files)
- Instant rollback if issues

## Decision Criteria

**Adopt if:**
- ✅ High CI volume (Ghost has hundreds of runs/week)
- ✅ Developers use macOS (Docker I/O is slow)
- ✅ Fast iteration matters
- ✅ 1-2 people willing to maintain
- ✅ Multi-arch support needed

**Skip if:**
- ❌ Low CI volume (<50 builds/month)
- ❌ No maintainer capacity
- ❌ Current CI speed is acceptable
- ❌ Team strongly opposes complexity

For Ghost: active OSS project, frequent PRs, multi-platform images, many macOS developers.

## FAQ

**Q: What if the maintainer leaves?**

Same risk as any infrastructure. Mitigations:
- Extensive documentation ([README-new.md](./README-new.md))
- Standard Nix patterns (no clever hacks)
- 10,000+ NixOS contributors (can hire)
- Escape hatch: revert to Docker-only

**Q: First build is slower, why?**

Cold cache builds everything from scratch (~29min measured). Warm cache is faster (~6min). More data needed for comparison to Docker baseline.

**Q: Windows developers?**

Keep using Docker/yarn. Nix on Windows is experimental. WSL2 works if desired.

**Q: How much does binary cache cost?**

Cachix: ~$100/month for more than enough capacity. Self-hosted is possible but more complex.

## Simple vs Easy

Nix is **simple** (pure functions, immutable data, can't fail in surprising ways) but not **easy** (unfamiliar model).

Docker is **easy** (familiar syntax) but **complex** (mutable state, hidden dependencies, brittle caching).

The upfront cost is learning the model. The forever benefit: problems stay solved.

