# Vendor policy

REA integrates third-party reverse-engineering engines by pinning upstream source
under `third_party/<name>/` (git submodule, subtree, or documented caller-supplied
tool) and calling them across a process boundary. This policy governs how a new
upstream enters the tree, how its provenance is recorded, and the supply-chain
hygiene required before the integration ships.

Scope: every path under `third_party/`, plus the CI job that enforces the
import boundary (`.github/workflows/ci.yml` — `vendor-boundary`) and the vendor
PR template (`.github/PULL_REQUEST_TEMPLATE/vendor.md`).

## Import boundary — no `third_party/` imports from REA source

REA source under `src/` and `bridge/` must never `import` or `require` a path
under `third_party/`. Upstream code is executed as a subprocess, a JAR, a WASM
module, or a caller-supplied binary; it is never linked into REA's TypeScript
or Python graph. The `vendor-boundary` CI job greps for cross-boundary imports
on every PR and on `main` and fails the build on any match.

Reason: REA is MIT; several upstreams are GPL-family or have attribution
requirements that an in-process import would propagate to REA's own
distribution. Shelling out across a process boundary keeps the license
boundary exact and matches how real operator setups supply engines.

## GPL-3.0 (and copyleft) subtree carve-out

A GPL-3.0, AGPL, or other strong-copyleft upstream enters only under
`third_party/<name>/`, isolated from REA's source graph by the import
boundary above. The upstream's own `LICENSE`, `NOTICE`, and copyright
headers stay untouched. REA's adapter (under `src/` or `bridge/`) must:

- invoke the upstream as a separate process, JAR, or WASM worker,
- never link, bundle, or re-export upstream types/functions into REA's
  TypeScript or Python source, and
- preserve the upstream's license file verbatim in `third_party/<name>/`.

If an upstream's license ever changes to one incompatible with REA's MIT
distribution (new GPL-3.0 subtree, surprise AGPL relicense), the carve-out
restricts the blast radius to `third_party/<name>/` — the boundary CI job
blocks accidental in-process imports, and the integration can be excised
without touching REA's own code.

## Ingestion rule — pin an exact commit

Every new upstream enters via:

```sh
# Submodule (preferred for engines REA invokes by path):
git submodule add --name <name> <upstream-url> third_party/<name>
cd third_party/<name> && git checkout <40-char-commit-sha>

# OR subtree (preferred for source we need present at build time):
git archive --remote=<upstream-url> <40-char-commit-sha> \
  | tar -xf - -C third_party/<name>/
```

The pinned reference is always a 40-character commit SHA, not a tag, branch,
or `HEAD`. Tags move; branches move; commits don't. Submodule gitlinks already
pin a commit; subtree ingestion records the SHA in the PR body and in
`third_party/README.md`.

## Scrub — do not vendor secrets, PII, or operator state

Before committing a `third_party/<name>/` entry, grep the ingested tree for
patterns that should never enter REA's history. The vendor PR template requires
the operator to paste clean grep output.

Repo-agnostic patterns:

```sh
# Private keys and keystores
grep -rnE -- '-----BEGIN (RSA |EC |DSA |OPENSSH |PGP )?PRIVATE KEY-----' third_party/<name>/
grep -rnE -- '-----BEGIN ENCRYPTED PRIVATE KEY-----' third_party/<name>/

# Generic bearer / API tokens
grep -rnE -- '(bearer|authorization)\s*[:=]\s*["'\'']?[A-Za-z0-9_\-\.]{20,}' third_party/<name>/
grep -rnE -- '(sk|pk)_(live|test)_[A-Za-z0-9]{20,}' third_party/<name>/

# AWS-style credentials
grep -rnE -- 'AKIA[0-9A-Z]{16}' third_party/<name>/
grep -rnE -- 'aws_secret_access_key\s*=' third_party/<name>/

# MAC / BSSID / CIDR / IMEI patterns (keep when documenting hw, drop when it's operator state)
grep -rnE -- '([0-9a-fA-F]{2}:){5}[0-9a-fA-F]{2}' third_party/<name>/
```

JS/TS/Node-flavoured patterns (REA publishes `rea-agents` to npm):

```sh
# Hardcoded env-var assignments in source
grep -rnE -- 'process\.env\.[A-Z_]+_(KEY|TOKEN|SECRET)\s*=\s*["'\''][^"'\'']+["'\'']' third_party/<name>/

# npm registry tokens
grep -rnE -- 'npm_[A-Za-z0-9]{36}' third_party/<name>/
grep -rnE -- 'NPM_TOKEN\s*=\s*["'\''][^"'\'']+["'\'']' third_party/<name>/

# .npmrc files with real tokens
find third_party/<name>/ -name '.npmrc' -exec grep -lE '_authToken=.+' {} +

# Bearer tokens baked into test fixtures
grep -rnE -- 'bearer\s+[A-Za-z0-9_\-\.]{30,}' third_party/<name>/tests third_party/<name>/fixtures 2>/dev/null
```

An upstream that ships secrets in its own test fixtures (test-only keys, dummy
JWTs for a fixture flow) can pass the scrub by documenting the finding in the
PR body — a test fixture value is not an operator secret and is often part of
the upstream's public distribution. The rule is "don't introduce new secrets
into REA's tree"; it is not "rewrite upstream tests."

Hardware identifier regexes (serial, IMEI, MEID, keybox) that other repos
include in their scrub are omitted here: REA's domain is static analysis of
binaries, not device operator state. Add one if a future upstream carries
device-captured artifacts.

## Supply-chain hardening — 5 immediate items

Every vendor PR satisfies these before merge:

1. **Pinned commit SHA**, 40 chars, recorded in both the PR body and
   `third_party/README.md`. No tags, no branches, no `HEAD`.
2. **SPDX license id** stated in the PR body and documented in
   `third_party/<name>/README.md` or `THIRD_PARTY_NOTICES.md`. An
   `"UNKNOWN"` or missing license blocks merge.
3. **Scrub output pasted** from the patterns above into the PR body —
   zero matches, or each match annotated as a benign upstream test fixture.
4. **Upstream URL** canonicalized (the fork or redistribution REA actually
   consumes, not an aggregator mirror). Documented alongside the SHA.
5. **Boundary CI green** — `vendor-boundary` grep passes, confirming no
   REA source imports the new subtree.

## Supply-chain hardening — 5 tooling-gated items (defer when applicable)

These require either the upstream to be present at CI time or extra tooling
in the pipeline. Defer each until the trigger arrives; don't scaffold them
all on day one.

6. **SBOM** generated against `third_party/<name>/` via `syft` and attached
   to the release. Trigger: first subtree with no upstream SBOM of its own.
7. **CVE scan** of `third_party/<name>/` via `grype` (or equivalent).
   Trigger: an upstream with its own vendored dependency graph.
8. **Transitive license check** — confirm the upstream's own
   `node_modules/` or `vendor/` dir doesn't drag in an incompatible
   license. Trigger: an upstream REA executes rather than just documents.
9. **Signed releases** — cosign verification on upstream release artifacts
   when the upstream publishes signatures. Trigger: first upstream that
   signs its releases.
10. **Dedicated vendor-CI workflow** — a separate `.github/workflows/`
    file that runs vendor-specific checks (upstream test suite,
    cross-platform smoke) on vendor PRs. Trigger: a vendor PR with
    checks heavier than the boundary grep (5–10 minutes or more).

## Vendor PR template

`.github/PULL_REQUEST_TEMPLATE/vendor.md` records the five immediate items
above. Pick it from GitHub's template selector, or append
`?template=vendor.md` to the compare URL.

## Enforcement

- `.github/workflows/ci.yml` — `vendor-boundary` job (required via the
  `required:` aggregate) greps for cross-boundary imports on every PR.
- `.github/CODEOWNERS` already routes `third_party/` edits to maintainers.
- A vendor-PR title should follow REA's convention (`feat(vendor)`,
  `chore(vendor)`, or `docs(vendor)`) so the squash-merge subject identifies
  the integration in the release changelog.
