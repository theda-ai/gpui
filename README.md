# gpui (fork)

A fork of [**gpui**](https://github.com/zed-industries/zed/tree/main/crates/gpui) —
the GPU-accelerated UI framework developed by Zed Industries — carrying one
modification: **zero-copy native-texture presentation** on macOS, so a host
application can present an offscreen `wgpu` canvas without round-tripping frames
through the CPU atlas.

Unlike a snapshot import, this repository keeps the **real upstream history**:
6194 commits on `zed-upstream` reaching back to gpui's first one
(`Start rebuilding with a cleanly-separated UI framework`, 2021-02-20), each with
its original author, date and message. `git log`, `git blame` and `git bisect`
work across upstream code, and taking a newer upstream is a merge rather than a
re-import.

## Branches

| branch         | contents |
| -------------- | -------- |
| `zed-upstream` | filtered upstream history only — no fork changes. Advanced by `scripts/sync-upstream.sh`. |
| `main`         | `zed-upstream` merged at the revision below, plus the fork commits. **Pin this one.** |
| `upstream`     | frozen: the filtered history under the pre-2026-11-09 path list (see below). Kept so old references resolve. |
| `pointer-warp` | an unmerged proposal adding `Window::set_mouse_position`; not part of `main`. |

## What is in the tree

A self-contained Cargo workspace (`default-members = ["crates/gpui"]`) holding
gpui and the crates it needs: `bench_metrics`, `collections`, `gpui`,
`gpui_apple`, `gpui_ios`, `gpui_linux`, `gpui_macos`, `gpui_macros`,
`gpui_platform`, `gpui_shared_string`, `gpui_util`, `gpui_web`, `gpui_wgpu`,
`gpui_windows`, `http_client`, `http_client_tls`, `refineable`,
`reqwest_client`, `scheduler`, `sum_tree`, `util_macros`, `zlog`, `ztracing`,
`ztracing_macro`, plus `assets/fonts` and `tooling/perf`. The authoritative
list of extracted paths is [`scripts/upstream-paths.txt`](scripts/upstream-paths.txt).

`reqwest_client` and `http_client_tls` are carried for Cargo's sake: `gpui`
declares `reqwest_client` as a dev-dependency (which wants `http_client_tls`),
and Cargo resolves a member's path dependencies whether or not they are
dev-dependencies, so without them the workspace manifest cannot load. They cost
consumers nothing — a git dependency never resolves another package's
dev-dependencies. `crates/util` is still in the history but no longer a
workspace member: nothing above depends on it, and upstream's `util` now needs
a `crates/path` this fork does not carry.

The root `Cargo.toml` is upstream's, with `members` pruned to the crates that
actually exist here, `default-members` pointed at `crates/gpui`, and
`[patch]` entries that point at paths this fork lacks removed; everything else,
`[workspace.dependencies]` included, is upstream's own text. See
[`scripts/prune-workspace.py`](scripts/prune-workspace.py).

## The fork delta

On top of `zed-upstream`:

1. `chore(workspace)` — prune the workspace manifest (above).
2. `docs` — this README, `NOTICE`, `.gitignore`.
3. `build` — the upstream sync tooling under `scripts/`.
4. `feat(gpui_apple)` — **the actual patch**: a single-plane `32BGRA`
   external-texture present path in the Metal renderer, so a client can hand
   gpui an already-rendered `IOSurface` (wrapped as a `CVPixelBuffer`) through
   `Window::paint_surface` instead of the biplanar NV12 the surface path
   assumes. Two files:
   * `crates/gpui_apple/src/metal_renderer.rs` — per-surface pipeline selection
     by pixel format, plus the `surfaces_rgba` pipeline state;
   * `crates/gpui_apple/src/shaders.metal` — the `surface_fragment_rgba`
     passthrough fragment shader.

   Until 2026-11-09 the patch lived in `crates/gpui_macos`; upstream moved the
   Metal renderer into the shared `gpui_apple` crate (zed #62649) and the patch
   was re-applied there on the objc2-based renderer. Upstream still accepts only
   NV12 surfaces, so the patch is still needed.

## Revisions

| | fork | upstream (`zed-industries/zed`) |
| --- | --- | --- |
| `zed-upstream` tip merged into `main` | [`9216bab4cb`](../../commit/9216bab4cb) | [`483b746b568256e125538769ce5474a5347069a3`](https://github.com/zed-industries/zed/commit/483b746b568256e125538769ce5474a5347069a3) |
| previous `main` base | [`bc70bb583d05e2059de3280a47fa3690e387289d`](../../commit/bc70bb583d05e2059de3280a47fa3690e387289d) | [`4b7369481dcf36f22ff8f813d411e1d296aebe57`](https://github.com/zed-industries/zed/commit/4b7369481dcf36f22ff8f813d411e1d296aebe57) |

### The 2026-11-09 re-extraction

`crates/bench_metrics`, `crates/gpui_apple` and `crates/gpui_ios` were added to
the path list, because upstream gpui now depends on them. None of them existed
before the previous `main` base, so every commit up to `bc70bb58` came out of
the new extraction with the same SHA, and `main` merged the new history as an
ordinary merge. The old `upstream` branch had already advanced past zed #62649
without `gpui_apple`; it diverges from `zed-upstream` after
[`0a3c2b80`](../../commit/0a3c2b800aabb4be07f2a3f44ac9510fd1851a80) and is
frozen.

## Syncing with upstream

Because this is a path-filtered extraction rather than a whole-repo fork, a sync
re-derives the filtered history and appends the new commits:

```sh
pip install git-filter-repo          # once
scripts/sync-upstream.sh             # advance the `zed-upstream` branch
git checkout main && git merge zed-upstream
```

or `scripts/sync-upstream.sh --merge` to do both, which also re-prunes
`Cargo.toml` for you. Expect the merge to conflict only where upstream touched
`crates/gpui_apple/src/metal_renderer.rs` or `shaders.metal` — the two files the
patch owns. After a merge, check that the workspace still loads
(`cargo metadata --no-deps`): upstream gpui gaining a dependency on a crate this
fork does not carry shows up there, and the fix is a documented addition to
`scripts/upstream-paths.txt` (see the note at its top).

The extraction is reproducible: re-filtering a newer upstream reproduces every
commit already published here bit-for-bit and adds the new ones on top. Two
inputs are load-bearing and the script pins both — the path list, and
`--preserve-commit-hashes` (without it git-filter-repo rewrites commit SHAs
quoted inside commit *messages*, and the result depends on which refs were in the
clone). `sync-upstream.sh` verifies the property instead of assuming it: it
refuses to move `zed-upstream` unless the published tip is an ancestor of what
came back out.

**Fork SHAs are not upstream SHAs.** Rewritten history means new commit ids, so a
patch cannot be pushed to `zed-industries/zed` from here — export it and apply it
to a real Zed checkout.

## Consuming

```toml
gpui          = { git = "https://github.com/theda-ai/gpui", rev = "…" }
gpui_platform = { git = "https://github.com/theda-ai/gpui", rev = "…", features = ["font-kit", "runtime_shaders"] }
```

Pin an exact `rev` on `main` for reproducible builds. All `gpui_*` crates resolve
from this one repository (workspace dependencies), so no `[patch]` juggling is
needed on the consumer side. `Window::paint_surface` takes
`CFRetained<objc2_core_video::CVPixelBuffer>` (upstream moved from the
`core-video` crate to `objc2-core-video`); a consumer that builds the buffer
itself needs the same `objc2-core-video` version as gpui's workspace.

Superseded pins keep working: the pre-history squashed import is kept on the
branch [`legacy/squashed-import-915a9af`](../../tree/legacy/squashed-import-915a9af),
so `rev = "915a9af15612f1dbbf8385f4b8fbaa4833bf3130"` still resolves. It shares no
history with `main` and is frozen — new work goes on `main`.

## Building

The Apple backends (`gpui_apple`, `gpui_macos`, `gpui_ios`) compile and validate
**only on macOS** (Metal + native Apple deps). Linux/CI hosts cannot
cross-compile them.

## License

Licensed under the **Apache License 2.0**, inherited from upstream gpui (see the
per-crate `LICENSE-APACHE` files and `NOTICE`). `zlog`, `ztracing` and
`ztracing_macro` carry upstream's dual `LICENSE-APACHE` / `LICENSE-GPL` files;
consult them before shipping.
