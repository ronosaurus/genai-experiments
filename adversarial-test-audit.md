---
name: adversarial-test-audit
description: Audit a subsystem's test suite with an adversarial mindset - orient on the subsystem, extract the claims its tests and code imply, then try to falsify each claim with the narrowest counterexample that would still pass the existing suite. Use when the user wants to know whether a codebase's tests actually prove it's robust, wants "harder" or "adversarial" tests, asks to find test gaps, or wants a second opinion on whether passing tests can be trusted. Not for generating routine unit tests or increasing line coverage - this is about disproving implicit guarantees, not padding a test count.
---

# Adversarial Test Audit

A disciplined process for finding places where a suite passes but the
guarantee it appears to prove is false, unobserved, or weaker than intended.
Passing tests demonstrate behavior only for the inputs, schedules, fakes, and
oracles somebody chose. This skill finds the smallest legitimate difference
that should matter but currently does not.

Do not skip straight to "write more tests." Each phase produces evidence the
next one depends on. Skipping it creates generic filler (null checks, empty
strings, off-by-one) rather than tests that threaten a real guarantee.

This audit can also identify **coverage theater**: tests that exist primarily
to execute a line or branch for a quality gate but do not establish a
meaningful contract. Treat this as a focused test-quality question within the
selected subsystem, not a repository-wide code-coverage cleanup exercise.

## Scope before starting

Never run this against an entire codebase at once. Pick one subsystem -
a module, a service boundary, a class and its direct collaborators, a rule
engine, a parser, whatever the user names or whatever is small enough to
hold in full. If the user hasn't named a subsystem, ask which one before
proceeding; do not guess at a scope that spans the whole repo.

Treat user-provided requirements, public documentation, and externally
observable API contracts as the authority. Code and existing tests are
evidence of intended behavior, not authority when they conflict. Call out an
ambiguity instead of silently selecting the interpretation that makes a test
easy to write.

## Phase 0 - Orient and establish a baseline

Before forming any opinion about correctness, build an accurate model of
what the subsystem *is*, scoped to just this subsystem:

- Use relevant facts already established in the current session as Phase 0
  evidence: files read, subsystem boundaries, contracts, call paths, test
  behavior, and prior command results. Do not repeat broad discovery merely
  because the skill was invoked later in the session.
- Re-read or independently verify prior session evidence only when it is
  ambiguous, may be stale after edits, conflicts with another source, or is
  material to a reported finding. State which conclusions rely on session
  context versus newly inspected evidence.
- Read the implementation files, not just the tests. Note the public entry
  points, the state it owns, what it depends on (and whether those
  dependencies are mocked, faked, or real in the existing tests), and any
  concurrency, ordering, or lifecycle assumptions.
- Read the existing test suite for this subsystem end to end. Note what
  categories of input it already exercises (happy path, known error cases,
  boundary values) so Phase 1 doesn't waste time re-deriving claims that are
  already well-tested and already correctly falsifiable.
- If the subsystem is unfamiliar, this phase may need its own back-and-forth
  with the user or a dedicated read-only pass - do not compress it just to
  get to the "adversarial" part sooner. A wrong model of the subsystem
  produces adversarial tests that attack a strawman, not the real code.
- Identify the narrowest existing test command for this subsystem and its
  execution cost. Running and rerunning tests is ideal, but any test may be
  expensive when it requires a server, container, database, or shared
  environment. Do not run a test merely to explore a weak hypothesis.
- Before executing an expensive test, deeply model the implementation, public
  contract, production call sites, fixtures, test harness, and dependency
  behavior. Use that analysis to reject impossible candidates and rank the
  remaining ones. Reserve runtime execution for a small, batched set of
  high-confidence, high-consequence candidates. If no baseline was run,
  record that limitation rather than attributing a later failure to the audit
  without evidence.
- Locate every observable effect the tests can inspect: return value,
  exception/error, persisted state, emitted event, dependency call,
  cancellation, timing, and resource cleanup. Also note effects that matter
  but are currently hidden behind mocks or never observed.

Output of this phase: a short internal model of the subsystem (its
boundaries, its state, its dependencies, what's real vs. mocked in tests).
This does not need to be shown to the user in full, but should be summarized
in 3-5 bullets before moving on, so the user can correct a wrong mental model
early.

## Phase 1 - Extract and rank claims

From the oriented model, state every guarantee the subsystem appears to make,
in plain language, independent of test code. Pull claims from:

- Function/method names and public API shape ("retryWithBackoff",
  "isIdempotent", "close()")
- Docstrings, comments, and error messages
- What the *existing tests* assert (a test asserting `result == expected`
  for three inputs implies a claim like "handles these three shapes
  correctly" - state the claim, not just the assertion)
- Implicit contracts inferred from usage elsewhere in the subsystem (e.g. a
  cache that's read without a lock implies a thread-safety claim, whether or
  not anyone wrote that down)

Write these as a numbered list of falsifiable statements, e.g.:
"Claim 3: concurrent calls to `withdraw()` never allow the balance to go
negative, even under contention." A claim must be specific enough that a
single counterexample would disprove it - "handles errors gracefully" is not
a claim, "returns the prior committed state on a mid-write crash" is.

For each claim, record:

| Field | Meaning |
| --- | --- |
| Source | Requirement/docs, API contract, production call site, implementation, or existing test |
| Current evidence | Exact test(s) and assertion(s), or `none` |
| Oracle strength | Whether the test observes the relevant effect rather than merely successful execution |
| Consequence | Impact if false: correctness, data integrity, security, availability, cost, or user experience |
| Priority | `P0` for security/data loss/corruption, `P1` for high-impact correctness or availability, `P2` otherwise |

An execution-only assertion, a broad "no exception" assertion, or an
assertion against a mock configured with the expected result is weak evidence.
Mark it as such. In particular, inspect whether a mock's behavior eliminates
the failure mode the test claims to cover, and whether the asserted
interaction would be true even if the production result were wrong.

For each existing test, ask: "What realistic regression would this test
catch?" A test is a coverage-theater candidate when it has no answer beyond
"it executes this line/branch," or when its assertion is tautological,
execution-only, over-mocked, or insensitive to the meaningful output or side
effect. Do not infer motive from the test author or report a test as useless
solely because it is simple; setup, wiring, and boundary tests can provide
valuable regression protection. Report the missing claim or weak oracle, not
an accusation about why the test was written.

When several coverage-theater candidates exercise the same behavior, propose
one stronger contract-level test to replace or consolidate them. Do not remove
or rewrite them automatically: first show the human which regression each
test fails to protect against, the replacement's stronger oracle, and the
expected raw coverage change. Raw coverage may temporarily decline while
low-value tests are consolidated, then recover as valuable behavior-focused
tests are added. The goal is a more meaningful coverage signal, not preserving
or inflating a percentage quality gate.

## Phase 2 - Falsify

For each P0/P1 claim first, then worthwhile P2 claims, search for the
narrowest input, sequence, or state that violates it while remaining a
legitimate call into the subsystem - not an API misuse the type system already
prevents. Start with the transformation most likely to invalidate the
assumption embedded in the implementation or test.

- Boundary and off-by-one values, but only where they map to a real claim -
  don't generate these generically
- Ordering and interleaving: reentrant calls, out-of-order arrivals,
  duplicate deliveries, cancellation at each await/yield point, and concurrent
  access to shared state. Use deterministic barriers, latches, fake clocks,
  or controlled schedulers when available; never introduce sleep-based tests
  as evidence of a race.
- Partial failure: crash/exception mid-operation, one dependency call
  succeeding and the next failing, retried operations that aren't actually
  idempotent, and recovery after a process restart where applicable
- Resource lifecycle: double-close, use-after-close, exhaustion (pool limits,
  queue depth, disk/memory pressure) where the subsystem claims to handle it
- Surface-pattern collisions: an input that satisfies whatever check the code
  performs (a regex, a type check, a name match, a range check) without
  satisfying the property that check was meant to stand in for - the same
  class of bug as a name-based classifier matching the wrong receiver
- Adversarial/hostile input where the subsystem trusts data it shouldn't
  (this is where a security/injection framing applies, if relevant to the
  subsystem)
- Equivalence and metamorphic relationships: equivalent encodings, reordered
  inputs where order should not matter, split-vs-batched work, repeated
  idempotent calls, or transformations that should preserve a result
- Cross-implementation comparisons: compare a parser, serializer, cache, or
  optimizer with its documented reference behavior, a simpler trusted
  implementation, or round-trip invariants when one exists
- Test-double gaps: replace a permissive mock with a realistic fake, inject
  the one failure the dependency can actually return, or inspect persisted
  output rather than only interaction counts

For each surviving counterexample, produce:
1. The claim it violates (reference back to Phase 1's numbering)
2. The concrete input/sequence/state
3. The expected externally observable behavior and why it follows from the
   authoritative source
4. Why the existing suite does not catch it, including the weak/missing oracle
5. A minimal test case in the project's existing test style/framework - do not
   invent a new harness or pattern if one already exists

Discard candidates that don't survive: if on inspection the "counterexample"
is actually prevented by a type, a prior check, or is simply not reachable
through the public API, drop it rather than padding the output. The goal is
a small number of tests that actually threaten a claim, not a long list of
speculative ones.

## Phase 3 - Confirm the finding

Do not call a candidate a defect solely because it sounds plausible. For each
candidate selected for reporting, establish the strongest evidence practical
for its execution cost:

1. First confirm the control and data flow statically: trace the concrete
   input through the implementation and verify that no type, validation, or
   caller constraint prevents the outcome.
2. If execution is affordable, write or run the narrow reproducer against the
   current implementation. Batch selected candidates into one run rather than
   restarting the environment for each one.
3. When execution is not affordable, preserve the concrete code path and
   contract evidence that supports the candidate; do not claim runtime
   reproduction.
4. If executed, confirm that a failure occurs for the claimed reason, not due
   to broken setup, unspecified behavior, or an unrelated pre-existing issue.
5. Ensure the assertion observes the contract directly. Prefer checking the
   returned/persisted/emitted result over private state or incidental call
   order.
6. Make the test deterministic. Control time, randomness, process state, and
   concurrency; restore global state; and avoid test-order dependencies.
7. When practical, perform a mutation check: identify the smallest plausible
   wrong implementation that the proposed test would fail to detect. Improve
   the oracle or discard the test if it cannot distinguish that mutation.

### Why this is not full mutation testing

Do not run a mutation-testing tool as the default audit method. Mutation
testing asks whether the existing suite detects artificial implementation
changes; this audit asks whether meaningful contracts can be violated by
legitimate inputs, states, and interactions. A high mutation score can still
miss the wrong contract, unrealistic test doubles, unobserved side effects,
integration boundaries, and failure sequences that this skill targets.

Use the focused mutation check above only to test whether a *proposed
adversarial test's oracle* can distinguish one plausible wrong behavior. Run
full mutation testing only when the user explicitly requests it and its cost
is justified by the subsystem's risk and test-runtime budget.

Classify the result precisely:

- **Confirmed defect:** The reproducer fails against the current implementation
  and the expected behavior has an authoritative basis.
- **Missing regression protection:** Current behavior is correct, but a small
  plausible mutation would pass the suite. Add a regression test only when
  the claim is worth protecting.
- **Contract ambiguity:** Sources conflict or do not specify the expected
  behavior. Report the decision needed; do not encode a guess as a test.
- **Rejected:** Unreachable, prevented by contract/type validation, already
  covered by a strong test, or non-deterministic without a controllable seam.
- **High-confidence hypothesis:** A concrete, reachable code path and
  authoritative contract imply a defect, but runtime confirmation was deferred
  because test execution is expensive. Include the exact test to run later.

If adding code is requested, add only confirmed-defect regressions and
high-value missing-protection tests. A test that exposes a confirmed defect is
expected to fail until the implementation is fixed; do not conceal that fact,
weaken its assertion, or change production behavior unless the user asked for
the fix too.

## Stop rules - Decide when it is good enough

The objective is proportionate confidence, not hypothetical completeness.
Stop searching and report the subsystem as sufficiently covered when all of
the following are true:

1. Every P0/P1 claim has either strong direct evidence, a reported finding, or
   an explicitly documented contract ambiguity.
2. The highest-risk assumptions at each relevant boundary have been examined:
   input validation, state transitions, failure/rollback behavior, and
   lifecycle/concurrency behavior where the subsystem uses them.
3. Every remaining candidate is P2 or lower, requires an unsupported
   environmental assumption, duplicates an examined failure mode, or lacks a
   concrete code path from a legitimate public call to a violated contract.
4. The next candidate would not change a release, design, or test decision if
   it were true.

Apply this value filter to every candidate before spending further analysis or
test-runtime budget:

- Is there a specific authoritative claim and a concrete reachable code path?
- Would failure cause meaningful user, security, data, availability, or cost
  impact?
- Is it materially different from an already-examined scenario?
- Would the result change what the team does?

Reject the candidate when any answer is no. Do not report generic hardening
ideas, style preferences, theoretical races without a plausible interleaving,
or edge cases that do not change a meaningful outcome. Summarize rejected
candidate categories once instead of listing every variation.

End the audit with one confidence statement: **ready** when the stop rules are
met with no unresolved P0/P1 finding; **ready with accepted risk** when
remaining items are explicitly deferred or ambiguous; or **not ready** when a
confirmed or high-confidence P0/P1 finding remains. State the small set of
risks that informed that judgment.

## Output

The default deliverable is a human-reviewable Markdown decision report, not
test or production code. It should give a reviewer enough evidence to decide
which findings justify the cost and risk of implementation. Do not create a
report file unless the user asks for one; present the report in the response
by default. Produce HTML only when the user explicitly requests HTML or a
standalone artifact.

Use this report structure:

1. **Audit scope and confidence:** subsystem boundary, authoritative sources,
   what was read, what was or was not executed, and the final `ready`, `ready
   with accepted risk`, or `not ready` verdict.
2. **System model:** the 3-5 bullet orientation summary, including state,
   dependencies, externally observable effects, and key assumptions.
3. **Claims coverage:** a prioritized claims table with source, current
   evidence, oracle strength, consequence, and status.
4. **Findings for decision:** only confirmed defects, high-confidence
   hypotheses, high-value missing regression protection, and contract
   ambiguities. For each, include the evidence trail, concrete reproducer,
   expected versus observed behavior, impact, confidence, runtime-validation
   status, and the smallest proposed next action.
5. **Rejected categories:** a concise summary of hypotheses intentionally
   excluded by the value filter, so the reviewer can see that the audit
   stopped deliberately rather than overlooking them.
6. **Decision queue:** a short ordered list of the decisions the human must
   make, such as accept risk, clarify a contract, authorize a costly test
   batch, add a regression test, or fix production code.

When coverage-theater candidates are found, include a **coverage signal**
subsection under the decision queue. Group candidates by duplicated behavior,
show their weak or absent claim, and offer retain, strengthen, consolidate, or
remove as explicit human decisions. Never imply that a changed raw coverage
percentage alone establishes better or worse test quality.

For every reported finding, include its classification, claim number,
priority, concrete reproducer, authoritative expected behavior, current
observed behavior, why existing tests miss it, and the proposed test or patch.
State the exact command and result when one was run. Otherwise, state the
static evidence, the deferred test command, and why execution was deferred.
Conclude with the stop-rule confidence statement and accepted risks.

Stop after delivering the report unless the user explicitly authorizes one or
more items from the decision queue. If test changes are authorized, show the
proposed tests and their expected pre-fix/post-fix status before editing. If
both test and production fixes are authorized, make the regression test fail
first where feasible, fix the implementation, and run only the agreed
validation batch.
