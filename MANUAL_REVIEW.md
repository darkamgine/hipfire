# Manual Review (morning triage list)

Sorted by what unblocks the most downstream work first.

(empty at session start; appended as escalations occur)

## #111 MQ4 tool-call structured-output drift

- Why escalated: defensive parser ships value tonight, but the actual root cause is MQ4 calibration; that needs a quantizer-side retrain + held-out eval, not a runtime patch.
- What was tried:
  - Reproduced the malformation class on `qwen3.6:27b.mq4` at temp=0 with multi-tool stream prompts (7900 XTX, gfx1100). My capture: flat shape `{"name": "write", "path": ..., "content": ...}`. Reporter's: `<plain>NAME</param> {ARGS}`. Both are MQ4 quantization drift on structured tokens (`{`, `"`, `:`, tag names).
  - Confirmed the spec form parses cleanly on single-tool low-context prompts; corruption emerges with multi-tool + system-prompt context (which shifts the per-position activation distribution).
  - Implemented and shipped (fix/111-tool-call-mq4-malformation): defensive parseToolCalls upgrade with two repair paths (flat coerce, XML-tag scan) plus balanced-brace JSON extraction. 10/10 unit tests in `cli/parse_tool_calls.test.ts`. End-to-end repro now emits `finish_reason: tool_calls` with non-empty arguments instead of a silent no-op.
- Hypothesis: MQ4 (FWHT-rotated 4-bit) calibration set was not weighted on tool-call-shape data. The structured tokens (`{`, `"`, `:`, tag-name fragments) sit on tighter argmax margins than narrative prose, so any quant-induced shift flips them first. This matches the row-3994 outlier story in #87 (Wo projections) but at a different layer in the stack: there it was MMQ activation drift, here it is weight quant calibration.
- Suggested next step: add tool-call-shape calibration samples (OpenAI tool messages, ChatML `<|im_start|>...assistant\n<tool_call>\n{...}\n</tool_call>`) to the `hipfire-quantize` MQ4 calibration corpus weighted ~5x; requantize qwen3.5:9b + qwen3.6:27b; rerun the multi-tool repro on the new artifacts; if clean, re-quantize the rest of the family and ship a `mq4` rev bump. If insufficient, escalate to MQ4-Lloyd (a Lloyd-Max codebook variant of MQ4 already evaluated and deprioritized for narrative ppl, but the structured-token tail behaviour could be different).
- Files touched: `cli/index.ts` (parser + helpers), `cli/parse_tool_calls.test.ts` (new tests).
- Branch: `fix/111-tool-call-mq4-malformation`
- Commits: `62e5767` on master (cherry of `9e73ccc` on overnight branch).
