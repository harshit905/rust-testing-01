# Expected SCA results — ground truth (Rust, richer worst case)

A Cargo `[workspace]` with two member crates and NO `Cargo.lock`, so the scanner
must run `cargo generate-lockfile`. Worst-case elements: the workspace itself
(members in sibling dirs) and a vulnerable crate that pulls transitives.

## Summary

| bucket | packages |
|--------|----------|
| Vulnerable | `smallvec@1.6.0`, `time@0.1.43` |
| Healthy | `itoa@1.0.x`, plus `time`'s transitives (`libc`, the `winapi` family, `wasi`) |
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
