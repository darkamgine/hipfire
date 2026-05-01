# Overnight Log 2026-05-01

## Morning Summary

Session: 2026-05-01T08:20Z to ~09:20Z (~1h active; rest is GPU + bench wait).
Branch state: master fast-forwarded through 11 new commits, all pushed.
Working tree clean. Overnight branch `overnight/2026-05-01` carries all
telemetry; not merged (and shouldn't be, by design).

### Phase 1: Issue queue

| Issue | Tier | Status | What landed | Reply |
|---|---|---|---|---|
| #111 | 1 | FIXED + ESCALATED | Defensive parseToolCalls covering flat + XML-tag MQ4 malformations; 10 unit tests; calibration-retrain ask escalated | [comment](https://github.com/Kaden-Schutt/hipfire/issues/111#issuecomment-4358518730) |
| #110 | 2 | FIXED | DFlash draft auto-discovery prepends dirname(target) so Docker / non-default mounts work without HIPFIRE_DFLASH_DRAFT env | [comment](https://github.com/Kaden-Schutt/hipfire/issues/110#issuecomment-4358534885) |
| #112 | 2 | FIXED | scripts/compile-kernels.ps1 created + install.ps1 invokes daemon.exe --precompile (parity with install.sh) | [comment](https://github.com/Kaden-Schutt/hipfire/issues/112#issuecomment-4358520755) |
| #87  | 2 | ADDRESSED (awaits closure) | PR #104 merged + tri-state mmq_screen toggle (off/on/auto) shipped pre-overnight | [comment](https://github.com/Kaden-Schutt/hipfire/issues/87#issuecomment-4358521609) |
| #82  | 2 | PARTIAL FIX + ESCALATED | Windows 8.3-short-path conversion in compiler.rs `-I` flag handling; native verification escalates | [comment](https://github.com/Kaden-Schutt/hipfire/issues/82#issuecomment-4358541721) |
| #50  | 2 | PARTIAL FIX + ESCALATED | gfx1152 added to all RDNA 3.5 dispatch / cache / MMQ gates (addresses incoherent output); segfault still needs reporter backtrace | [comment](https://github.com/Kaden-Schutt/hipfire/issues/50#issuecomment-4358551547) |
| #107 | 3 | FIXED | docs/MODELS.md +112 line "Thinking mode and chat templates" section | [comment](https://github.com/Kaden-Schutt/hipfire/issues/107#issuecomment-4358559274) |
| #105 | feat | DEFERRED | Listed in DEFERRED.md; pointed at #76 / #77 design proposals | [comment](https://github.com/Kaden-Schutt/hipfire/issues/105#issuecomment-4358554026) |

7 issues touched; 4 fully fixed, 2 partial-with-escalation, 1 awaiting
closure, 1 deferred.

### Phase 2: Megakernel

3 perf wins on the MQ3 / MQ6 decode path (additive; per-stage paths
unchanged; no quality regression on coherence-gate or speed-gate):

| Commit | Change | Win |
|---|---|---|
| `ce1e9c5` | gemv_hfq3g256_residual.hip + weight_gemv_residual MQ3 + HFQ3 fast paths | 0.8B MQ3 +3.5%, 9B MQ3 +1.6% |
| `7c47609` | weight_gemv_swiglu_residual MQ3 arm using fused_silu_mul_rotate_mq + gemv_hfq3g256_residual | 0.8B MQ3 +1.9% (cumulative +5.5%), 9B MQ3 +1.1% (cumulative +2.7%) |
| `d35ec72` | gemv_hfq6g256_residual.hip + MQ6 + HFQ6 paths in both wrappers | 9B carnice MQ6 +1.3% |

Final speed-gate (canonical): 14/14 metrics passed within ±5%
tolerance. 4B MQ4 pp32 +12.2%, decode -1.7%. 9B MQ4 decode -2.0%. 27B
MQ4 decode -2.8%. All within session-noise band.

VGPR budget unchanged on all three new kernels (identical
launch_bounds to their non-residual counterparts).

### Megakernel work NOT done

- Full FFN megakernel (rmsnorm + gate + up + silu*mul + down +
  residual fused into one launch): higher reward (~10-20% projected)
  but a substantive rewrite with quality-precision risk and VGPR
  exposure. Not started; megakernel target #82 in the internal task
  list remains pending.
- K4-unroll on gemv_hfq3g256: already implemented on
  `fix/mq3-mq2-cli-guards` (commit `0003103`, 9B 114 -> 141 tok/s,
  +24%) but that branch is the held PR #109 bundle. Not duplicated
  here per Rule 8.
- gfx1152 segfault root cause (#50): blocked on backtrace from
  reporter.
- MQ4 calibration retrain (#111): blocked on quantizer-side data
  pipeline + GPU time.

### Files maintained

- `OVERNIGHT_PLAN.md` (initial triage queue, untouched after seed)
- `OVERNIGHT_LOG.md` (this file; append-only timeline below)
- `MANUAL_REVIEW.md` (#82 + #50 + #111 escalations)
- `DEFERRED.md` (feature-request inbox)
- `bench/overnight-*.txt` x 3 (one per kernel commit)

All committed on `overnight/2026-05-01`, pushed to origin.

### Recommended morning order

1. Read this summary; spot-check the 8 issue comments.
2. Decide: close #112 (fully fixed) and ideally #87 + #107 (also fully addressed).
3. Decide if PR #109 (fix/mq3-mq2-cli-guards K4-unroll bundle) should ship now that the residual-fusion infrastructure is on master. The K4 work would compose cleanly on top of `d35ec72`.
4. Push #50 / #82 reporters for the requested debug info.
5. If you want the FFN megakernel, start fresh with full attention on it; partial-night attempts on a 6-stage fusion chain are too risky for autonomous work.

---

## Append-only timeline

### 2026-05-01T08:20Z | session start

- Branch `overnight/2026-05-01` created from master at `fa93b13`.
- `OVERNIGHT_PLAN.md` written and ready to commit.
- Pre-flight commits already on master:
  - `4e1a7e2` MMQ tri-state toggle.
  - `fa93b13` Windows compile-kernels.ps1 + daemon precompile parity (#112 fixed).
- Hardware available: gfx1100 (7900 XTX) on this box. No Strix Halo, no MI300X (rented), no Vega/CDNA.


### 2026-05-01T08:35Z | #111 REPLICATE

- Started serve on port 11435 (qwen3.5:9b warmup, default model).
- Sent OpenAI v1/chat/completions with single-tool, then multi-tool + stream variants targeting `qwen3.6:27b` (MQ4). Both tested at temp=0 with DFlash auto-loaded (qwen36-27b-dflash-mq4.hfq).
- **Reproduced a malformation**, not the exact one in the reporter's screenshot but the same class:
  - Reporter: `<plain>write</param> {...}` (XML-tag-shaped corruption inside <tool_call>).
  - Mine (multi-tool stream): `{"name": "write", "path": "...", "content": "..."}` instead of `{"name": "write", "arguments": {...}}`. JSON parses fine, but `tc.arguments` is undefined and the existing parseToolCalls (cli/index.ts:1541) does `JSON.stringify(tc.arguments || {})` → "{}", silently dropping all args. The tool call is structurally wrong; the downstream harness gets an empty-arg call and the file is never written. Same-shape failure mode as the reporter.
- Single-tool non-stream call produced clean nested JSON, so the malformation depends on prompt shape (multi-tool, system prompt, longer history). Matches the reporter's report ("agentic harness").
- Suspected layer: MQ4 weight quantization (FWHT rotation shifts P over structured-token positions). Same root cause class as #87. Per Rule 1 (quality regressions are bugs), root cause = quant calibration; ship fix = defensive parser; calibration retrain escalates to MANUAL_REVIEW.

### 2026-05-01T08:55Z | #111 FIX + MERGE + REPLY

- Branch `fix/111-tool-call-mq4-malformation` off overnight; cherry-picked into overnight at `9e73ccc`; landed on master at `62e5767` (e932811 originally, telemetry-only revert commits c0ac542+62e5767 followed to keep master clean of overnight scratch).
- Defensive parseToolCalls repair shipped: 3-form parser (spec / flat coerce / XML tag) + balanced-brace JSON walker. 10/10 bun tests pass. End-to-end repro now emits `finish_reason: tool_calls` with non-empty arguments.
- MANUAL_REVIEW entry added: calibration retrain ask for MQ4 (root cause, not parser layer).
- Issue comment posted at https://github.com/Kaden-Schutt/hipfire/issues/111#issuecomment-4358518730 with reproduction summary, fix description, verification, and the calibration escalation note.
- Status: FIXED (parser stopgap) + ESCALATED (calibration root cause).

### 2026-05-01T09:00Z | #110 FIX + MERGE + REPLY

- Branch `fix/110-dflash-draft-docker-path` off overnight; cherry-picked onto master at `7f2c0c5`.
- Single-file CLI fix: prepend `dirname(target_path)` as highest-priority DFlash draft auto-discovery candidate. Cwd-relative + homedir candidates kept as fallbacks. Mirrors how Linux-installer and Docker layouts diverge: the only reliable "where the user keeps weights" signal is the directory the target itself was loaded from.
- Pure-function unit verification + live serve test on 7900 XTX from cwd=/tmp both confirm. Reporter's HIPFIRE_DFLASH_DRAFT env workaround still respected.
- Status: FIXED.

### 2026-05-01T09:10Z | #82 FIX (Windows-only) + ESCALATE

- Branch `fix/82-windows-hipcc-space-in-path`; cherry-picked onto master at `88a52bb`.
- Single-function Windows-cfg helper that converts spaces-bearing -I paths to 8.3 short form via `cmd /c for ... echo %~sA`. Linux/macOS pass-through.
- Cargo release build clean. Cannot test on Windows from this box. MANUAL_REVIEW entry added with concrete reproduction script.
- Issue replied: https://github.com/Kaden-Schutt/hipfire/issues/82
- Status: FIXED (Windows-only path) + ESCALATED (native verification).

### 2026-05-01T09:25Z | #50 PARTIAL FIX (arch-gating) + ESCALATE (segfault)

- Branch `fix/50-gfx1152-arch-gating`; master at `d9e8dc5`. Pre-commit gates ran automatically (dispatch.rs staged): coherence battery + speed gate both green on gfx1100 (4B MQ4 pp32 +9.3%, decode -1.6%, within tolerance).
- Added gfx1152 to 9 arch-gate sites across dispatch + daemon + tests + install. Addresses the incoherent-output symptom (gfx1152 was falling through to gfx1100-shape dispatch).
- Segfault remains unaddressed. Reporter needs to provide a backtrace; concrete repro added to MANUAL_REVIEW.md and re-asked in issue comment.
- Status: PARTIAL FIX (arch-gating shipped) + ESCALATED (segfault root cause).

### 2026-05-01T09:35Z | #105 DEFER + #107 DOCS LANDED

- #105 (CPU+GPU split): feature request, listed in DEFERRED.md, replied with #76/#77 references for the underlying tiering design work.
- #107 (thinking + chat template): docs/MODELS.md +112 lines covering thinking-mode mechanics, thinking on/off semantics + the /no_think directive ban history, max_think_tokens, OpenAI API knobs from #79, ChatML envelope, prompt_normalize step. Cherry-picked onto master at `ee7d3cc`. Status: FIXED (Tier 3 docs).


### 2026-05-01T09:08Z | Phase 2: MQ3 residual fusion shipped

- Branch `feat/mq3-residual-fusion`, cherry-picked onto master at `ce1e9c5`. Pre-commit gates ran (coherence + speed): both green.
- Kernel: `kernels/src/gemv_hfq3g256_residual.hip` (NEW); 32-thread wave32, += into y, byte-identical body to gemv_hfq3g256 modulo final-write op. VGPR / launch_bounds unchanged.
- Engine wiring: `weight_gemv_residual` in `crates/engine/src/llama.rs` gains HFQ3G256 + MQ3G256 fast paths. MQ3 path mirrors MQ4: rotate_x_mq into mq_x_rot scratch, then dispatch the fused residual GEMV.
- Bench (gfx1100, 7900 XTX, mean of 3 runs):
  - 0.8B MQ3 decode: 299.3 -> 309.7 tok/s (+3.5%)
  - 9B  MQ3 decode: 109.8 -> 111.6 tok/s (+1.6%)
- Quality: coherence-gate short battery clean (4 MQ4 prompts unaffected); 9B MQ3 cap + code smokes both fluent.
- Bench raw: `bench/overnight-20260501T090528Z-mq3-residual-fusion.txt`.
- Note: commit message accidentally references "(#82)". Internal task-list ID #82 means "megakernel project"; GitHub issue #82 is the Windows hipcc bug. Different numbering schemes; no functional dependency.


### 2026-05-01T09:11Z | Phase 2: MQ3 swiglu fusion shipped (cumulative +5.5% / +2.7%)

- Branch `feat/mq3-swiglu-fusion`, fast-forward onto master at `7c47609`.
- Single-file llama.rs change: weight_gemv_swiglu_residual gains an MQ3G256 arm using fused_silu_mul_rotate_mq + gemv_hfq3g256_residual (both kernels already exist).
- Saves 2 launches/FFN/layer/token vs generic path (silu_mul + add_inplace).
- 0.8B MQ3 decode: 309.7 -> 315.6 tok/s (+1.9% this commit, +5.5% cumulative).
- 9B MQ3 decode: 111.6 -> 112.8 tok/s (+1.1% this commit, +2.7% cumulative).
- Quality smoke clean. Pre-commit gates green.
- Bench raw: bench/overnight-20260501T091104Z-mq3-swiglu-fusion.txt.


### 2026-05-01T09:18Z | Phase 2: MQ6 residual + swiglu fusion shipped (+1.3% on 9B MQ6)

- Branch `feat/mq6-residual-fusion`, fast-forward onto master at `d35ec72`. Pre-commit gates green.
- New kernel: gemv_hfq6g256_residual.hip (mirrors gemv_hfq6g256 with += final write).
- weight_gemv_residual: previously generic-fallback for HFQ6 + MQ6, now both have fast paths. weight_gemv_swiglu_residual: MQ6 arm added.
- Bench: carnice-9b.mq6 decode 85.7 -> 86.8 tok/s (+1.3%). Smaller win than MQ3 because per-token GEMV cost is larger relative to launch overhead at 6-bit precision.
- Quality smoke clean; coherence-gate green.
- Bench raw: bench/overnight-20260501T091651Z-mq6-residual-fusion.txt.

### 2026-05-01T09:30Z | Codex follow-up: weight-cache invalidation on model unload (#87)

- Codex stop-time review flagged: mmq_screen_cache + fp16_shadow_cache survived model unload, keyed on device pointers that the pool reused for the next model's weights.
- Fix: new `Gpu::invalidate_weight_caches()` clears mmq_screen_cache and drains fp16_shadow_cache (releasing the owned FP16 shadow tensors). Called from unload_model after weights are freed and before drain_pool.
- Branch `fix/mmq-screen-cache-stale-on-reload`, fast-forward onto master at `370a7aa`. Coherence + speed gates green.
- Smoke: 4 sequential model swaps (0.8B MQ4 -> 9B MQ4 -> 0.8B MQ4 -> 9B MQ3) without panic. Per-swap responses coherent.


### 2026-05-01T09:38Z | Codex follow-up #2: hipGraph state on model unload

- Codex stop-time review caught second half of the cache-staleness issue: captured hipGraphs (single-slot AR + DFlash verify + DFlash replay) survived unload along with their warmup-tracker sets.
- Captured graphs bake device pointers into kernarg memory at hipStreamEndCapture time. Without invalidation, model swaps replay against dangling or wrong-content tensors.
- Fix: new `Gpu::invalidate_graph_state()` calls graph_destroy + verify_graph_destroy_all + replay_graph_destroy_all (each already handles hipGraphExecDestroy + hipGraphDestroy and clears its capture/warmup state). unload_model invokes it after invalidate_weight_caches, before drain_pool.
- Branch `fix/graph-cache-stale-on-reload`, fast-forward onto master at `c314f0e`. Coherence + speed gates green.
- Smoke: 4 swaps with DFlash on the second cycle. serve.log shows multiple `warmup for B=16 complete` + `captured for B=16` entries with distinct blob counts (1204 vs 604), confirming per-load re-warm. Pre-fix would have shown a single warmup + capture across all swaps.

