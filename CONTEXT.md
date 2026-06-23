# mruby-engine

Sandboxed Ruby execution environment for an AI agent harness, with
capability-checked filesystem access. Based on Shopify/mruby-engine for
its quota (memory, instruction, time) and CRuby↔mruby value marshaling.

## Branch strategy

- `master` — tracks Shopify upstream
- `main-mr` — our additions; rebases onto master

## Interface

Primary interface is streams: LLM code writes to `$stdout`, errors to
`$stderr`, exceptions surface as `EngineRuntimeError`. `inject`/`extract`
available for structured data round-trips.

Do not use fd dup to capture stdout in tests — eval runs in a pthread and
fd 1 swap is process-wide, which breaks concurrent VMs. Test via filesystem
(write tempfile from mruby, read in CRuby).
