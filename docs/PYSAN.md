# PySan — a runtime taint sanitizer for Python web services

> Design note. Not yet implemented; this describes what gets built in hours 00–06 of the finale.

## Why this exists

AddressSanitizer is the reason autonomous cyber-reasoning works on C and C++. It is not the fuzzer that
makes the loop possible — it is the **oracle**. ASan converts an ambiguous condition (memory was used
wrongly) into an unambiguous, reproducible event (the process died, here is the stack).

Python web services have no equivalent. The failure modes that matter — SQL injection, command
injection, path traversal, SSRF, unsafe deserialization, broken object-level authorization — produce a
perfectly ordinary HTTP 200. There is no crash, no signal, and therefore:

- a fuzzer cannot score an input,
- a triage step cannot separate a finding from noise,
- and a patch cannot be shown to have closed anything.

PySan supplies that oracle.

## The core rule

> **Tainted data reaching a dangerous sink is a crash.**

Not a warning, not a score, not a heuristic. A deterministic, reproducible event carrying enough context
to replay it later — which is exactly what gate G1 needs after a patch is applied.

## Sources

Taint originates at every value a remote caller controls:

| Framework | Sources |
| --- | --- |
| Flask | `request.args`, `.form`, `.json`, `.data`, `.headers`, `.cookies`, `.files`, view kwargs |
| FastAPI | path/query/body params, `Request` object, headers, cookies, form and file uploads |
| Django | `request.GET`, `.POST`, `.body`, `.headers`, `.COOKIES`, `.FILES`, URLconf kwargs |

Secondary sources, tracked but ranked lower: environment variables, values read back from the database
(for second-order injection), and inbound message-queue payloads.

## Sinks

| Category | Sinks | CWE |
| --- | --- | --- |
| SQL | `cursor.execute`, `executemany`, SQLAlchemy `text()`, Django `.raw()` / `.extra()` | CWE-89 |
| Command | `os.system`, `os.popen`, `subprocess.*` with `shell=True` | CWE-78 |
| Code | `eval`, `exec`, `compile` | CWE-94 |
| Deserialization | `pickle.loads`, `yaml.load` (unsafe loader), `marshal.loads` | CWE-502 |
| Filesystem | `open`, `os.remove`, `shutil.*`, `send_file`, `send_from_directory` | CWE-22 |
| Network | `requests.*`, `urllib.request.urlopen`, `httpx.*` | CWE-918 |
| Template | Jinja2 `Template()` from a tainted string | CWE-1336 |
| Redirect | `redirect()` with a tainted target | CWE-601 |

Each sink is registered with its CWE, its dangerous-argument positions, and a predicate that decides
whether taint in that position is actually exploitable (a tainted *value* passed as a bound SQL
parameter is safe; a tainted *string* concatenated into the query is not — that distinction is the
difference between a finding and a false positive).

## Propagation strategy

Three mechanisms, in order of preference:

1. **A tainted `str` subclass.** Sources return `TaintedStr`, which carries a provenance record and
   overrides the string operations that produce derived values (`__add__`, `__mod__`, `format`, `join`,
   slicing, `encode`, f-string participation). Cheap, precise for the common path, and survives most
   real application code.
2. **Import-time sink wrapping.** A `sitecustomize` / import hook wraps registered sinks so that every
   call inspects its arguments for taint before dispatching to the real implementation.
3. **Selective tracing** for propagation the subclass cannot follow (C-implemented library internals,
   `bytes` round-trips). Applied narrowly and only to modules on a ranked sink path from stage 02, since
   `sys.settrace` on a whole application is far too slow to fuzz against.

The static pre-focus stage exists partly to make mechanism 3 affordable: PRAHARI already knows which
paths are worth the instrumentation cost before it starts executing anything.

## The PoV artifact

When a tainted value reaches a sink, PySan records:

```
finding_id        stable hash of the taint-path signature (used for dedup)
cwe               from the sink registry
route             method + path template
request           full replayable request: method, URL, headers, body, auth state
taint_path        source → propagation steps → sink, with file:line at each hop
sink              callable, argument position, rendered argument value
stack             Python traceback at the moment of the sink call
response_before   the target's response prior to patching
```

This is the unit that flows into triage (stage 05), constrains patch synthesis (stage 06), and is
replayed by gate G1 (stage 07).

## Known limitations

Stating these plainly, because a jury of security researchers will ask:

- **Implicit flows are not tracked.** `if tainted == "x": y = "safe_value"` launders taint. This is the
  standard limitation of value-level taint tracking and is accepted deliberately: chasing implicit flows
  costs precision and performance far out of proportion to the bugs it finds.
- **Native extensions are opaque.** Taint entering a C extension may not survive it. Mitigated by
  re-tainting at known re-entry points for common libraries.
- **Authorization bugs need a second oracle.** Broken object-level authorization is not a taint problem —
  it is a *policy* problem. PySan detects it via differential identity replay (issue the same request
  under two identities, compare responses) rather than sink hooking. Treated as a separate detector that
  shares the same PoV format.
- **Performance.** The tainted-string layer costs measurably on string-heavy code paths. Acceptable
  because the fuzzing target is a test instance, not production — but it is the reason mechanism 3 is
  applied selectively.
- **Coverage is bounded by routing discovery.** A route that stage 01 never finds is never fuzzed.

## Relationship to prior work

Taint tracking for Python is not new — `python-taint`/PyT did static taint analysis, and commercial IAST
agents (Contrast, Seeker) do runtime instrumentation. What is new here is not the taint layer itself
but **what it is wired to**: using it as a *fuzzing oracle* and as the *acceptance test for an
autonomously generated patch*. No existing tool pairs runtime taint tracking with autonomous patching
and a proof bundle.
