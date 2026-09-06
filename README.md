# GenLayer Dual-Source Claim Resolver

A reusable GenLayer Intelligent Contract for adjudicating claims against two distinct web evidence sources.

The resolver independently fetches and classifies both sources during GenLayer consensus, explicitly represents disagreement and source failures, and uses a single-shot adjudication policy to prevent repeated stochastic rerolls of unchanged evidence.

## Motivation

AI-assisted adjudication becomes risky when a decision depends on a single source, a single model interpretation, or repeated nondeterministic retries.

This contract explores a more conservative pattern:

1. Fetch two distinct evidence sources.
2. Classify each source as `SUPPORTS`, `REFUTES`, or `INSUFFICIENT`.
3. Have GenLayer validators independently repeat the evidence evaluation.
4. Aggregate both source verdicts into a normalized decision.
5. Finalize the deployed resolver instance after the first accepted adjudication.
6. Treat only dual-source agreement as a definitive binary resolution.

This preserves uncertainty without allowing callers to repeatedly reroll unchanged ambiguous or conflicting evidence until a preferred stochastic result appears.

## Resolution States

The resolver can return:

- `TRUE`
- `FALSE`
- `CONFLICTING_EVIDENCE`
- `UNDETERMINED`
- `SOURCE_UNAVAILABLE`
- `INVALID_RESULT`

Every accepted adjudication sets:

```text
is_finalized = true
```

Only `TRUE` and `FALSE` additionally set:

```text
has_resolved = true
```

Therefore:

- `is_finalized` means the current deployed claim/evidence instance has completed its one allowed adjudication.
- `has_resolved` means a definitive binary conclusion (`TRUE` or `FALSE`) was reached.

Non-definitive outcomes preserve uncertainty but cannot be rerolled on the same instance.

## Decision Logic

| Source 1 | Source 2 | Decision |
| --- | --- | --- |
| SUPPORTS | SUPPORTS | `TRUE` |
| REFUTES | REFUTES | `FALSE` |
| SUPPORTS | REFUTES | `CONFLICTING_EVIDENCE` |
| REFUTES | SUPPORTS | `CONFLICTING_EVIDENCE` |
| Either source unavailable | Any | `SOURCE_UNAVAILABLE` |
| Insufficient evidence | Any non-conflicting result | `UNDETERMINED` |
| Invalid model output | Any | `INVALID_RESULT` |

After any accepted decision above, the resolver instance becomes finalized.

## Single-Shot Adjudication

Each deployed resolver instance permits exactly one accepted adjudication.

After consensus completes:

```text
is_finalized = true
```

A second call to `resolve()` on the same instance is rejected with:

```text
Adjudication already finalized
```

This applies even when the first result was non-definitive, including:

- `CONFLICTING_EVIDENCE`
- `SOURCE_UNAVAILABLE`
- `UNDETERMINED`
- `INVALID_RESULT`

If the claim, evidence, or evidence availability changes, a new resolver instance must be created.

This prevents repeated nondeterministic evaluation of unchanged evidence from being used to grind toward a permanent `TRUE` or `FALSE` result.

## Independent Validator Verification

The contract uses GenLayer nondeterministic execution so validators independently:

- fetch both URLs;
- inspect the source content;
- classify each source;
- aggregate the result.

Validators do not simply trust a leader-provided conclusion.

Consensus is performed over normalized fields such as:

```text
decision
source1_status
source1_verdict
source2_status
source2_verdict
```

This avoids comparing raw webpage content while still requiring validators to independently evaluate the underlying evidence.

## Defensive Design

### External Source Failures

Unavailable web sources do not abort the adjudication workflow.

They become:

```json
{
  "decision": "SOURCE_UNAVAILABLE"
}
```

The result is non-definitive:

```text
has_resolved = false
```

but the current resolver instance is still finalized:

```text
is_finalized = true
```

If the evidence later becomes available, a new resolver instance must be created.

### Evidence Disagreement

If one source supports a claim while the other refutes it, the resolver returns:

```json
{
  "decision": "CONFLICTING_EVIDENCE"
}
```

instead of forcing a binary answer.

The disagreement is preserved in contract state while the current adjudication is finalized.

### Retry Protection

Once any accepted adjudication completes, another call to `resolve()` on the same instance is rejected.

This specifically prevents unchanged ambiguous or conflicting evidence from being repeatedly rerolled until one stochastic round happens to produce matching `SUPPORTS` or matching `REFUTES`.

### Structured AI Output Validation

Each source evaluator accepts only:

- `SUPPORTS`
- `REFUTES`
- `INSUFFICIENT`

Unexpected model output is normalized into an invalid non-binary outcome rather than silently producing a definitive claim resolution.

### Untrusted Data Handling

Claims and webpage contents are explicitly treated as data rather than instructions inside the evaluation prompt.

Basic constructor validation is also applied to claims and source URLs.

## GenVM Validation

The current V3 contract source was validated before redeployment using:

```text
genvm-lint check dual_source_claim_resolver_v3.py
```

Result:

```text
✓ Lint passed (3 checks)
✓ Validation passed
```

The nondeterministic web and LLM calls are directly reachable from the GenVM-recognized consensus execution path.

## GenLayer Studio Test Results

The V3 contract was tested using `Normal (Full Consensus)` mode.

| Test | Result |
| --- | --- |
| Both sources support claim | ✅ `TRUE`, `is_finalized = true`, `has_resolved = true` |
| Both sources refute claim | ✅ `FALSE`, `is_finalized = true`, `has_resolved = true` |
| One source unavailable | ✅ `SOURCE_UNAVAILABLE`, `is_finalized = true`, `has_resolved = false` |
| Sources directly conflict | ✅ `CONFLICTING_EVIDENCE`, `is_finalized = true`, `has_resolved = false` |
| Retry unchanged conflicting evidence | ✅ Rejected with `Adjudication already finalized` |

The retry-protection test directly verifies that a non-definitive result cannot be repeatedly rerolled on the same deployed instance.

Detailed transaction evidence is documented in `TEST_RESULTS.md`.

## Synthetic Conflict Evidence

The repository includes two intentionally contradictory test sources:

```text
evidence/supports.md
evidence/refutes.md
```

These files are used only to create a controlled integration test for the `CONFLICTING_EVIDENCE` path.

They are not presented as real-world evidence.

## Contract

The Intelligent Contract is available at:

```text
contracts/dual_source_claim_resolver.py
```

## Potential Use Cases

The resolver pattern can be adapted for:

- prediction-market adjudication;
- protocol milestone verification;
- public-event resolution;
- governance claims;
- compliance or disclosure checks;
- evidence-backed agent workflows;
- fact-based automated processes where disagreement should not silently resolve.

## Scope

This project is a reference implementation for evidence-backed Intelligent Contracts.

It does not claim that agreement between two sources guarantees objective truth.

Two sources can share the same error, bias, stale information, publisher, upstream dependency, or other correlated failure.

V3 currently requires distinct URL strings but does not enforce stronger source-independence guarantees such as separate domains, publishers, or provenance chains.

The design instead demonstrates how an Intelligent Contract can:

- independently evaluate multiple evidence sources;
- explicitly represent disagreement and uncertainty;
- separate binary resolution from adjudication finalization;
- prevent repeated stochastic rerolls of unchanged evidence.

## Future Work

Potential extensions include:

- stronger source-independence requirements beyond unequal URLs;
- domain or publisher diversity checks;
- three-or-more-source quorum policies;
- source reputation or weighting;
- domain allowlists;
- timestamp and freshness validation;
- structured citations returned with decisions;
- evidence-version identifiers;
- authorized evidence-update policies;
- automated GenLayer testing;
- more nuanced semantic-equivalence validation.

## License

MIT
