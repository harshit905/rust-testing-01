# Expected SCA results — ground truth (Rust)

## Summary

| bucket | count | packages |
|--------|-------|----------|
| Vulnerable | 1 | `smallvec@1.6.0` |
| Healthy | 1 | `itoa@1.0.x` (latest 1.x) |
| Unresolved | 0 | — |

`crate_a` and `crate_b` are the local workspace members (the repo's own code).
They have no registry `source`, so they are excluded from the resolved graph and
appear in none of the buckets.

## Vulnerabilities
- **`smallvec@1.6.0`** — buffer overflow in `SmallVec::insert_many`,
  `CVE-2021-25900` / `RUSTSEC-2021-0003` (`GHSA-43w2-9j62-hq99`), fixed in 1.6.1.
  Zero dependencies. Pass condition: smallvec shows the advisory at version 1.6.0.

## Healthy
- **`itoa@1.0.x`** — the range `1` resolves to the latest 1.x, no advisories,
  zero dependencies. This checks range generation. The exact patch (e.g. 1.0.11)
  may vary; "itoa healthy at some 1.0.x" is the pass condition.

## The worst-case element
This is a workspace whose members are in `crate_a/` and `crate_b/`.
- **PASS (whole directory copied):** members load, `cargo generate-lockfile`
  succeeds → smallvec vulnerable + itoa healthy.
- **FAIL (only root Cargo.toml copied):** cargo errors "failed to load manifest
  for workspace member" → whole generation fails → **0 healthy, smallvec/itoa
  unresolved, 0 vulnerabilities** (a false all-clear).

## Pass / fail
- PASS: `smallvec@1.6.0` vulnerable, `itoa@1.0.x` healthy.
- FAIL: 0 healthy, unresolved deps, 0 vulns, or any invented version.
