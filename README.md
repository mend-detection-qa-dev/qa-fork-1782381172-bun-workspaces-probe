# bun-workspaces-probe

Mend SCA detection probe — Tier 1, entry #4.

## Pattern bundle

- `workspace-basic` — root `package.json` declares `"workspaces": ["packages/*"]`; Mend must discover and scan both workspace members.
- `workspace-cross-reference` — `packages/web` depends on the sibling `packages/core` via `"core": "workspace:*"`; Mend must resolve the `workspace:*` protocol to a local workspace member and report it with `source: local`, NOT attempt an npm registry lookup.

## Why bundled

Both patterns exercise the same underlying Mend code path: workspace member discovery from the `workspaces[]` field in the root manifest, and `workspace:*` protocol resolution from the `workspaces` section of `bun.lock`. Bundling them in a single probe means a workspace-parser failure surfaces as one compound failure in ReportPortal rather than two misleadingly separate regression entries.

The standalone separation (pattern #18, `bun-workspace-protocol-ranges-probe`) covers `workspace:^` and `workspace:~` semver-ranged protocols, which require a distinct code path beyond the basic `workspace:*` wildcard handled here.

## Mend config

No `.whitesource` is emitted for this probe. Bun is NOT in Mend's `install-tool` supported list — `scanSettings.versioning` cannot pin a Bun toolchain, and there is no UA pre-step toggle for Bun. Detection is lockfile-driven only: Mend must parse `bun.lock` (text JSONC, Bun 1.2+ format) statically.

Because Bun is not a recognized resolver, the UA may fall back to the npm resolver and attempt to parse `bun.lock` as if it were `package-lock.json`. The `bun.lock` JSONC format (including trailing commas and `//` comments) will cause a JSON.parse failure on any non-JSONC-aware parser. That is an intended probe target.

## Workspace layout

```
bun-workspaces-probe/               <- root workspace (no direct npm deps)
├── package.json                    <- workspaces: ["packages/*"]
├── bun.lock                        <- shared JSONC lockfile (Bun 1.2+)
├── index.ts                        <- minimal stub
├── packages/
│   ├── core/
│   │   └── package.json            <- dependencies: { "zod": "^3.25.0" }
│   └── web/
│       └── package.json            <- dependencies: { "core": "workspace:*", "hono": "^4.12.0" }
└── expected-tree.json
```

Workspace edges:

```
root (bun-workspaces-probe)
  └── core  [source: local, path: packages/core]
      └── zod@3.25.68  [source: registry]
  └── web   [source: local, path: packages/web]
      ├── core  [source: local, workspace:*]   <- cross-reference
      └── hono@4.12.18  [source: registry]
```

The `web -> core` edge is encoded with `"core": "workspace:*"` in `packages/web/package.json`. In `bun.lock`, the `packages/web` workspace entry carries the protocol verbatim (`"core": "workspace:*"`), and the `packages` object contains NO entry for `core` — local workspace members are resolved from the `workspaces` section, not the `packages` section.

## Key failure modes this probe targets

| Failure | Symptom in Mend output | BUN_COVERAGE_PLAN.md ref |
|---|---|---|
| `workspace:*` resolved as registry lookup | `core` appears as `source: registry` or a 404 ghost dep | §4, row 3 |
| Local member dropped from packages section | `core` missing from `web` sub-tree entirely | §4, row 3 |
| JSONC parse failure | Empty dependency tree for all workspaces | §4, row 1 |
| Workspace members not discovered | Only root reported; `packages/core` and `packages/web` absent | §2.1 entry #9 |

## Resolver notes (UA — javascript.md)

Bun does NOT appear in the UA `javascript.md` resolver file. The UA resolver selection table maps lock files as follows:

- `yarn.lock` -> YarnLockCollector
- `package-lock.json` / `npm-shrinkwrap.json` -> NpmLockCollector
- `pnpm-lock.yaml` -> PnpmLockCollector
- `bun.lock` -> **not listed** (no registered resolver)

The UA will likely fall back to npm ls or filesystem scan when `bun.lock` is the only lockfile present. Neither fallback understands `workspace:*`. This probe documents this gap and encodes the CORRECT expected tree — the comparator will flag the gap when the actual Mend output diverges.

## Tracked in

`docs/BUN_COVERAGE_PLAN.md` §11.1 entry #4