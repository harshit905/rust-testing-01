# Expected SCA results — ground truth (Rust, richer worst case)

A Cargo `[workspace]` with two member crates and NO `Cargo.lock`, so the scanner
must run `cargo generate-lockfile`. Worst-case elements: the workspace itself
(members in sibling dirs) and a vulnerable crate that pulls transitives.

## Summary

| bucket | packages |
|--------|----------|
| Vulnerable | `smallvec@1.6.0`, `time@0.1.43`, `generic-array@0.13.2` (workspace-inherited), `nix@0.17.0` (target-specific), `regex@1.5.4` (dev) |
| Healthy | `itoa@1.0.x`, `lazy_static@1.4.0`, `typenum@1.x`, `aho-corasick@0.7.20`, `memchr@2.x`, `regex-syntax@0.6.29`, `bitflags@1.x`, `cfg-if@0.1.x`, `void@1.0.2`, plus `time`'s transitives (`libc`, the `winapi` family, `wasi`) and possibly `cc` |
| Unresolved | none |

`crate_a` and `crate_b` are local workspace members (the repo's own code) with no
registry `source`, so they are excluded from the resolved graph.

## Vulnerabilities
- **`smallvec@1.6.0`** — buffer overflow, `CVE-2021-25900` / `RUSTSEC-2021-0003`
  (`GHSA-43w2-9j62-hq99`), fixed in 1.6.1. Zero dependencies.
- **`time@0.1.43`** — segfault via `localtime`, `RUSTSEC-2020-0071` /
  `CVE-2020-26235`. Pulls transitives (below).

## Healthy — includes transitives (the new thing to analyze)
- **`itoa@1.0.x`** — the range `1` resolves to the latest 1.x, no advisories.
- **`time`'s transitives** — `libc`, `winapi`, `winapi-i686-pc-windows-gnu`,
  `winapi-x86_64-pc-windows-gnu`, `wasi` (Cargo.lock records all target
  platforms). All healthy. Their presence proves transitive discovery works.

## Worst-case feature — the workspace
Members live in `crate_a/` and `crate_b/`. Generation must copy the whole
directory; if only the root `Cargo.toml` is copied, `cargo generate-lockfile`
errors on the missing members → whole generation fails → **0 healthy, unresolved
deps, 0 vulnerabilities** (a false all-clear).

## Pass / fail
- PASS: smallvec and time vulnerable; itoa and time's transitives healthy; 0
  unresolved.
- FINDINGS to flag: missing transitives, 0 healthy (workspace not handled), 0
  vulns (false all-clear), or any invented version.

## New edge case (regression re-test) — `package =` rename
`crate_b` declares `mylazy = { package = "lazy_static", version = "=1.4.0" }`.
The generated `Cargo.lock` keys by the real crate name `lazy_static`.
- **PASS:** `lazy_static@1.4.0` is healthy and marked **direct** (its `package =`
  rename target is registered for classification).
- **FAIL:** `lazy_static` marked **transitive** (the alias/real-name mismatch).

## Round 2 edge cases

### A. Workspace-inherited dependency (`generic-array = { workspace = true }`)
The version `=0.13.2` lives only in the root `[workspace.dependencies]`;
`crate_a` inherits it. Requires Cargo >= 1.64, so an old resolver toolchain
fails the WHOLE generation here.
- **PASS:** `generic-array@0.13.2` vulnerable (`RUSTSEC-2020-0146`, fixed
  0.13.3), marked **direct**; its transitive `typenum@1.x` healthy.
- **FAIL:** generic-array marked transitive (the `workspace = true` table
  entry wasn't registered as direct), or generation fails (old cargo).

### B. Target-specific dependency (`[target.'cfg(unix)'.dependencies] nix = "=0.17.0"`)
Cargo.lock records target deps for every platform.
- **PASS:** `nix@0.17.0` vulnerable (`RUSTSEC-2021-0119` / `CVE-2021-45707`,
  out-of-bounds write in `getgrouplist`, fixed 0.20.2), marked **direct**; its
  transitives `bitflags`, `cfg-if`, `void`, `libc` (and possibly `cc`) healthy.
- **FAIL:** nix marked transitive (only plain `[dependencies]` is read) or
  missing.

### C. Dev dependency (`[dev-dependencies] regex = "=1.5.4"` in `crate_b`)
Cargo.lock has no scope information; the scanner must read `Cargo.toml`.
- **PASS:** `regex@1.5.4` vulnerable (`RUSTSEC-2022-0013` / `CVE-2022-24713`
  ReDoS, fixed 1.5.5), scope **DEV**; its transitives `aho-corasick@0.7.20`,
  `memchr@2.x`, `regex-syntax@0.6.29` healthy.
- **FAIL:** regex marked production, or missing.

### Round 2 pass / fail (combined)
- PASS: 5 vulnerable (smallvec, time, generic-array direct, nix direct, regex
  DEV); lazy_static direct; all transitives healthy; 0 unresolved.
