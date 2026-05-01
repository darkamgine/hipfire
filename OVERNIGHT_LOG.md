# Overnight Log 2026-05-01

## Morning Summary (filled at session end)

(empty until termination)

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
