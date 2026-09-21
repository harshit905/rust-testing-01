# SCA test repo — Rust, lock-less workspace, worst case

A Cargo `[workspace]` with two member crates and NO `Cargo.lock`. The scanner
must run `cargo generate-lockfile`. The worst-case element is the workspace
itself: the member crates live in sibling directories, so the resolver must copy
the whole directory or generation fails.

See `EXPECTED_RESULTS.md` for the ground truth.

## Run it
1. New GitHub repo, e.g. `harshit905/sca-test-rust`.
2. `git remote add origin <url>` then `git push -u origin main`.
3. Scan in CodeAnt, compare to `EXPECTED_RESULTS.md`.

Do NOT commit `Cargo.lock`.
