
# PRD: mrruby_engine

## Problem Statement

AI agent harnesses (like mrruby) let an LLM write and execute Ruby code on the host machine. The code is arbitrary — the LLM can generate anything, including infinite loops, recursive stack exhaustion, unbounded memory allocation, and filesystem access to secrets. There is no mechanism to constrain what the LLM's code can do, short of the LLM refusing to do so.

## Solution

`mrruby_engine` is a CRuby C extension that embeds mruby as a sandboxed execution environment. The LLM's generated code runs inside an mruby VM. The VM exposes only the Ruby API surface that the host explicitly grants at creation time. Everything else — filesystem paths not in the allowlist, sensitive filenames, process spawning, dynamic code evaluation — is either absent or raises `Errno::EACCES`.

The host CRuby process creates one VM per agent, with a specific capability profile. Multiple VMs coexist in the same process, each with different privileges.

## User Stories

1. As an AI harness developer, I want to create an mruby VM with one method call, so that I can sandbox LLM-generated code without managing C-level setup.
2. As an AI harness developer, I want to pass a string of Ruby code to the VM and get a string result back, so that I can use the result in the agent loop.
3. As an AI harness developer, I want mruby exceptions to surface as CRuby `RuntimeError`, so that I can rescue them like any other Ruby error.
4. As an AI harness developer, I want the VM to be garbage-collected when the Ruby object goes out of scope, so that I don't leak `mrb_state` handles.
5. As an AI harness developer, I want to run multiple VMs in the same process simultaneously, so that multiple agents can operate in parallel with different privileges.
6. As an AI harness developer, I want to grant a VM read access to specific directory paths, so that the LLM can read project files within those boundaries.
7. As an AI harness developer, I want to grant a VM write access to specific directory paths, so that the LLM can write files only where I allow.
8. As an AI harness developer, I want path permissions to be fixed at VM creation time, so that running LLM code cannot escalate its own privileges.
9. As an AI harness developer, I want denied path access to raise `Errno::EACCES` inside the VM, so that the LLM can observe and handle the denial like a real OS permission error.
10. As an AI harness developer, I want `.env` and other sensitive filenames to be blocked unconditionally, so that the LLM can never read secrets regardless of directory grants.
11. As an AI harness developer, I want `Dir.tmpdir` inside the VM to return the agent's designated scratch path, so that the LLM can use standard Ruby temp conventions without knowing about the sandbox.
12. As an AI harness developer, I want `Dir.entries`, `File.read`, `File.write`, and related methods to check capabilities transparently, so that the LLM uses normal Ruby and the sandbox is invisible to it.
13. As an AI harness developer, I want `eval`, `require`, `load`, backticks, and `system` to be unavailable inside the VM, so that the LLM cannot execute code strings or spawn processes beyond what the engine allows.
14. As an AI harness developer, I want to pass an allowed read path list at construction time using standard Ruby keyword args, so that the interface is idiomatic.
15. As an AI harness developer, I want path checking to use `realpath` before comparing against the allowlist, so that symlinks and relative paths cannot bypass the policy.
16. As an orchestrator developer, I want to define named agent profiles (e.g. `test_agent`, `explorer`) with specific capability sets, so that the LLM can request a profile by name without ever specifying capabilities directly.
17. As an AI harness developer, I want the VM to raise after executing N instructions, so that an infinite loop in LLM-generated code cannot hang the host process.
18. As an AI harness developer, I want the VM to raise after allocating N bytes, so that unbounded memory growth in LLM-generated code cannot OOM the host process.
19. As an AI harness developer, I want both quotas to surface as `RuntimeError` inside the VM, so that the agent loop can catch and report them without crashing.

## Implementation Decisions

- **Two compilation units.** `ruby.h` and `mruby.h` both define `RBasic`, `RObject`, `RArray`, `RString`, and several macros. They cannot appear in the same `.c` file. The extension uses two files: `mrruby_sandbox.c` (includes only mruby headers, contains all mruby logic) and `mrruby_engine.c` (includes only `ruby.h`, contains the Ruby binding). They communicate through an opaque `mrruby_sandbox_t*` pointer declared in `mrruby_sandbox.h` with a plain C interface.

- **Sandbox interface.** The C boundary between the two compilation units:
  ```c
  mrruby_sandbox_t *mrruby_sandbox_create(void);
  void              mrruby_sandbox_destroy(mrruby_sandbox_t *sb);
  void              mrruby_sandbox_enable_filtering(mrruby_sandbox_t *sb);
  void              mrruby_sandbox_add_read_path(mrruby_sandbox_t *sb, const char *path);
  char             *mrruby_sandbox_eval(mrruby_sandbox_t *sb, const char *code, char **err_out);
  ```
  `eval` returns a `malloc`'d string on success; on error returns `NULL` and sets `*err_out` to a `malloc`'d message. Caller frees both.

- **Ruby API.** One class: `MrubyEngine`. Constructor accepts optional keyword args:
  ```ruby
  MrubyEngine.new                          # no filtering
  MrubyEngine.new(read: ["/path/a"])       # filtering enabled, only /path/a readable
  ```
  One method: `engine.eval(code_string) → String`. Raises `RuntimeError` on mruby exception.

- **Capability model.** Positive-only. Paths not in the allowlist are denied. No deny rules. If a narrower grant is needed within an already-granted tree, grant only the specific subdirectories instead.

- **Hardcoded blocklist.** The following filenames are always denied regardless of path grants: `.env`, `.env.*`, `*.pem`, `*.key`, `credentials.*`. This list lives in C and is not configurable.

- **Path resolution.** All incoming paths are resolved with `realpath()` before comparison against the allowlist. Allowlist entries are also resolved at `add_read_path` time. This prevents symlink traversal and relative-path bypass.

- **`Dir.tmpdir` injection.** At VM creation time, the engine overrides `Dir.tmpdir` to return the agent's designated scratch path. The LLM uses standard Ruby and is unaware of the substitution.

- **No escape hatches.** The mruby build must omit: `mruby-eval`, `mruby-require`, `mruby-io`, `mruby-dir`, `mruby-socket`, `mruby-process`. The engine registers its own `Dir` and `File` implementations. The LLM cannot call `eval`, `require`, `load`, backticks, or `system`.

- **mrb_state user data.** `mrb->ud` holds a pointer back to the `mrruby_sandbox_t`. Capability-checked C methods retrieve their allowlist through this pointer.

- **Lifecycle.** `mrb_state` is opened in `initialize` and closed in the `rb_data_type_t` free function, so Ruby GC drives cleanup. `RUBY_TYPED_FREE_IMMEDIATELY` is set.

- **mruby build.** The installed `libmruby.a` at `~/.rubies/mruby-4.0.0/` is linked statically into the `.bundle`. The mruby source at `~/src/language/mruby/` is used for rebuilds. A restricted `build_config/mrruby.rb` omits all I/O, network, and eval gems.

- **Resource quotas.** The mruby build already includes `-DMRB_USE_DEBUG_HOOK`. An instruction quota is wired via `mrb->code_fetch_hook` — the hook increments a counter on each instruction and calls `mrb_raise` when the limit is reached. A memory quota uses a custom allocator passed to `mrb_open_allocf` — the allocator tracks total bytes and raises when the limit is exceeded. Both limits are set at VM creation time and stored in `mrruby_sandbox_t`. Both surface as `RuntimeError` to the CRuby caller. These are required, not optional, because the LLM code is arbitrary.

- **Short-term phase.** For initial integration testing, the engine runs with no capability filtering and generous quota defaults. Filtering and quota tuning are introduced as a second step once the CRuby ↔ mruby round-trip is proven.

## Testing Decisions

A good test exercises the Ruby API only — `MrubyEngine.new`, `engine.eval(code)`, and the raised exceptions. It does not inspect C struct internals, mock `mrb_state`, or test mruby behavior directly.

- **Test 1 (hello world).** `engine.eval("'hello world'")` returns `"hello world"`. Proves the C extension loads and the round-trip works.
- **Test 2 (Dir.pwd).** `engine.eval("Dir.pwd")` returns a non-empty string. Proves the built-in Dir is available.
- **Test 3 (capability allow).** Engine created with `read: [Dir.pwd]`. `engine.eval("Dir.entries('.')...")` returns an array-like string. Proves the allowlist permits access.
- **Test 4 (capability deny).** Same engine. `engine.eval("Dir.entries('/etc')")` raises `RuntimeError` matching "Permission denied". Proves out-of-allowlist paths are blocked.
- **Test 5 (blocklist).** Engine with `read: [Dir.pwd]`. `engine.eval("File.read('.env')")` raises regardless of directory grant.
- **Test 6 (no eval).** `engine.eval("eval('1+1')")` raises. Proves the escape hatch is absent.
- **Test 7 (instruction quota).** Engine created with a low instruction limit. `engine.eval("loop do; end")` raises `RuntimeError` and returns. Host process survives.
- **Test 8 (memory quota).** Engine created with a low memory limit. `engine.eval("a=[]; loop{ a << 'x'*100_000 }")` raises `RuntimeError` and returns. Host process survives.
- **Test 9 (GC cleanup).** Create and discard many engines in a loop without explicit `destroy`. No memory leak or crash after GC. Proves the TypedData free function fires.

Tests live in `test/` using minitest. There is no prior art in this repo; use the standard `Minitest::Test` pattern with one `setup` creating the engine.

## Out of Scope

- **Write capability.** `File.write` path checking is not in scope for the initial build. Read-only capability is the first milestone.
- **Network access.** HTTP, socket, or any network capability is explicitly deferred.
- **`Errno::EACCES` as a proper mruby class.** The initial implementation raises `RuntimeError` with the message "Permission denied". A proper `Errno` module with POSIX codes comes later.
- **Agent profile system.** Named profiles (e.g. `test_agent`) and the orchestrator lookup table are orchestrator concerns, not gem concerns.
- **Spinel target.** The gem targets CRuby only. Spinel uses `__ffi_link__` against `libmruby.a` directly and does not use this gem.
- **HTTP/network access.** libcurl wrapper and URL allowlist are explicitly deferred to a future phase.
- **Interactive privilege prompting.** Orchestrator-level "request a new profile at runtime" flow is deferred.
- **mruby-engine upstream contribution.** The gem starts fresh; upstream contribution is a future consideration once the capability model is stable.

## Further Notes

- The gem name is `mrruby_engine` (`mrruby-engine` hyphenated). This avoids conflict with Shopify's `mruby_engine` gem, which has the same name but different goals and is no longer actively maintained under that API.
- The primary security boundary is LLM-generated code vs. the host OS. The per-VM privilege model is secondary. The orchestrator (CRuby process) is trusted and is not sandboxed by this gem.
- The gem is designed to be the sole tool interface for AI agent harnesses: the LLM writes Ruby, the engine runs it, the result comes back. The orchestrator never exposes raw shell access.
- `eval` is excluded for now but the capability model does not fundamentally prevent it. Since all filesystem and network access is enforced at the C level, `eval`'d code goes through the same checks as submitted code. The primary concern with allowing `eval` is auditability: code constructed and executed at runtime is opaque to the outside log. If auditability is not required, `mruby-eval` (437 lines) could be included in a future build variant.
