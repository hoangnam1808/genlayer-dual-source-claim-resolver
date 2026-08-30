# GenLayer Dual-Source Claim Resolver

A reusable GenLayer Intelligent Contract for resolving claims against two independent web sources.

The resolver independently fetches and classifies both evidence sources, detects source disagreement and availability failures, and only finalizes a claim when both sources independently support the same conclusion.

## Motivation

AI-assisted adjudication becomes risky when a decision is based on a single source or a single model interpretation.

This contract explores a more conservative pattern:

1. Fetch two independent evidence sources.
2. Classify each source as `SUPPORTS`, `REFUTES`, or `INSUFFICIENT`.
3. Have GenLayer validators independently repeat the evidence evaluation.
4. Aggregate the two source verdicts into a normalized claim state.
5. Finalize only when both sources independently agree.

Ambiguous, conflicting, or unavailable evidence remains retryable.

## Resolution States

The resolver can return:

- `TRUE`
- `FALSE`
- `CONFLICTING_EVIDENCE`
- `UNDETERMINED`
- `SOURCE_UNAVAILABLE`
- `INVALID_RESULT`

Only `TRUE` and `FALSE` set:

```text
has_resolved = true
```

All other states remain retryable.

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

This avoids comparing raw webpage content while still requiring validators to independently verify the underlying evidence.

## Defensive Design

### External source failures

Unavailable web sources do not abort the resolution workflow.

They become:

```json
{
  "decision": "SOURCE_UNAVAILABLE"
}
```

and the claim remains unresolved.

### Evidence disagreement

If one source supports a claim while the other refutes it, the resolver returns:

```json
{
  "decision": "CONFLICTING_EVIDENCE"
}
```

instead of forcing a binary answer.

### Structured AI output validation

Each source evaluator accepts only:

- `SUPPORTS`
- `REFUTES`
- `INSUFFICIENT`

Unexpected model output is normalized into an invalid or non-final state rather than modifying final claim state.

### Untrusted data handling

Claims and webpage contents are explicitly treated as data rather than instructions inside the evaluation prompt.

Basic constructor validation is also applied to claims and source URLs.

## GenLayer Studio Test Results

The contract was tested using `Normal (Full Consensus)` mode.

The current contract source was validated with `genvm-lint check` before redeployment.

```text
✓ Lint passed (3 checks)
✓ Validation passed
```

The nondeterministic web and LLM calls are directly reachable from the GenVM-recognized consensus execution path.

| Test | Result |
| --- | --- |
| Both sources support claim | ✅ `TRUE` |
| Both sources refute claim | ✅ `FALSE` |
| One source unavailable | ✅ `SOURCE_UNAVAILABLE` |
| Sources directly conflict | ✅ `CONFLICTING_EVIDENCE` |
| Conflict state after resolution | ✅ `has_resolved = false` |

During the conflicting-evidence test, one independently evaluating validator disagreed while quorum still reached an accepted consensus. This illustrates why independent validator evaluation is useful for nondeterministic evidence adjudication.

Detailed transaction evidence is documented in `TEST_RESULTS.md`.

## Synthetic Conflict Evidence

The repository includes two intentionally contradictory test sources:

```text
evidence/supports.md
evidence/refutes.md
```

These files are used only to create a deterministic integration test for the `CONFLICTING_EVIDENCE` path.

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
- fact-based automated workflows where conflicting sources should not silently resolve.

## Scope

This project is a reference implementation for evidence-backed Intelligent Contracts.

It does not claim that two sources guarantee objective truth. Two sources can share the same error, bias, stale information, or upstream dependency.

The design instead demonstrates how an Intelligent Contract can explicitly represent disagreement and uncertainty rather than forcing every query into a binary answer.

## Future Work

Potential extensions include:

- three-or-more-source quorum policies;
- source reputation or weighting;
- domain allowlists;
- timestamp/freshness validation;
- structured citations returned with decisions;
- automated GenLayer testing;
- more nuanced semantic-equivalence validation.

## License

MIT
