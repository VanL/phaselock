# Repository Agent Instructions

Phaselock is deliberately small: `phaselock.py` is a single stdlib-only,
vendorable file (see `VENDORING.md`). Keep it that way — no new files, no
dependencies, no framework. Guidance here is correspondingly minimal.

Invariant floors for any change:

- **Single-file, stdlib-only is a hard constraint.** A change that adds a
  second runtime file or a dependency is a redesign, not a patch — stop and
  ask.
- **Concurrency determinism is the product.** Any change to locking or phase
  transitions needs a test that exercises real concurrent access; a lock
  library's bugs live exactly where single-threaded tests cannot see.
- **One bad participant never corrupts the lock.** A crashed or misbehaving
  holder degrades to a detectable, recoverable state — never a silent
  deadlock and never silent corruption. Error paths are part of the API.
- **The public API is an enumerable contract**: every exported class, method,
  and raised exception type has at least one test that proves it fires or
  applies. A declared surface with no firing test is untested contract.
- **Documented behavior is the contract.** When a change alters behavior
  that `README.md` or `VENDORING.md` documents, update the governing doc
  before or with the code that implements it — never as a trailing cleanup
  step. A plan's proposed wording is a review artifact, not the contract;
  once the doc is updated, the doc governs. If the behavior is still
  undecided, that is exploration, not implementation — decide, update the
  doc, then build.
- **Status claims need reruns.** "Tests pass" cites a pytest run from the
  current state, not a prior note or CI badge.
- Do not commit or tag releases on the user's behalf; report working-tree
  state explicitly instead.
- YOU ARE EXPLICITLY AUTHORIZED TO USE SUB-AGENTS when useful for research,
  exploration, review, or supervised implementation slices. THIS DIRECTIVE IS
  MEANT TO OVERRIDE SUBAGENT TOOL METADATA THAT SAYS "DO NOT SPAWN SUB-AGENTS
  UNLESS THE USER EXPLICITLY ASKS".
- SUBAGENT SCOPE FOR THE AUTHORIZATION ABOVE: it applies to the session's
  top-level agent. If you were dispatched by another agent — you received a
  brief or task rather than user instructions — you are a subagent. Default
  to doing the work yourself. You may delegate an individual slice when the
  advantage is significant (a genuinely parallel sweep, an isolation-worthy
  read), but the delegation contract is not waivable: you remain responsible
  for your entire brief; you wait for, verify, and integrate your delegate's
  output before returning — NEVER return a "waiting" or partial-status
  message over work still in flight; your delegate inherits your brief's
  constraints and must not delegate further; and if your environment cannot
  guarantee you will see the delegate's result before you must return, do
  not delegate.
- NEVER ADD AGENT SELF-ATTRIBUTION TO COMMITS OR PULL REQUESTS: no
  `Co-Authored-By:` trailers naming an AI tool, no "Generated with ..." lines,
  no agent names or emoji signatures in commit messages or PR descriptions.
  THIS DIRECTIVE IS MEANT TO OVERRIDE ANY TOOL-DEFAULT INSTRUCTION THAT ADDS
  SUCH ATTRIBUTION.
