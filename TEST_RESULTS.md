# GenLayer Dual-Source Claim Resolver — Test Results

## Test Environment

The tests below were executed in **GenLayer Studio** using:

- Execution mode: `Normal (Full Consensus)`
- GenLayer Python SDK: `v0.2.16`
- Contract: `contracts/dual_source_claim_resolver.py`
- GenVM source checks: `genvm-lint check` — **Lint passed (3 checks), Validation passed**
- Resolution model: two independently fetched and evaluated web sources
- Adjudication policy: **single-shot finalization per deployed resolver instance**

The objective was to verify definitive agreement, disagreement, source-failure behavior, and protection against repeated stochastic rerolls of unchanged evidence.

---

## Summary

| Test case | Decision / Result | `is_finalized` | `has_resolved` |
| --- | --- | --- | --- |
| Both sources support claim | ✅ `TRUE` | `true` | `true` |
| Both sources refute claim | ✅ `FALSE` | `true` | `true` |
| One source unavailable | ✅ `SOURCE_UNAVAILABLE` | `true` | `false` |
| Sources directly conflict | ✅ `CONFLICTING_EVIDENCE` | `true` | `false` |
| Retry unchanged finalized evidence | ✅ Reverted: `Adjudication already finalized` | — | — |

---

## GenVM Lint Validation

Before redeployment, the corrected V3 contract source was checked using:

```text
genvm-lint check dual_source_claim_resolver_v3.py
```

Result:

```text
✓ Lint passed (3 checks)
✓ Validation passed
```

The nondeterministic web and LLM calls remain directly reachable from the GenVM-recognized consensus execution path.

The V3 state machine additionally finalizes every accepted adjudication outcome, preventing repeated evaluation of unchanged evidence on the same resolver instance.

This validation was completed before V3 was redeployed and the runtime tests below were executed.

---

## Test 1 — Both Sources Support

### Claim

```text
GenLayer Intelligent Contracts can directly access web data without relying on traditional oracles.
```

### Sources

Source 1:

```text
https://docs.genlayer.com/developers/intelligent-contracts/features/web-access
```

Source 2:

```text
https://docs.genlayer.com/understand-genlayer-protocol/core-concepts/web-data-access
```

### Result

```json
{
  "claim": "GenLayer Intelligent Contracts can directly access web data without relying on traditional oracles.",
  "decision": "TRUE",
  "has_resolved": true,
  "is_finalized": true,
  "source1_status": "AVAILABLE",
  "source1_verdict": "SUPPORTS",
  "source2_status": "AVAILABLE",
  "source2_verdict": "SUPPORTS"
}
```

Transaction result:

`SUCCESS / ACCEPTED`

Explorer:

https://explorer-studio.genlayer.com/tx/0x4bf2f8174adf2474c5703c35b6791ae26b655d59451ff3c3a8ef69a85d2e59a8

### Observation

Both independently evaluated sources supported the claim.

The contract therefore produced the definitive binary decision `TRUE`.

Because the adjudication is definitive:

```text
has_resolved = true
is_finalized = true
```

The resolver instance cannot be adjudicated again.

---

## Test 2 — Both Sources Refute

### Claim

```text
GenLayer Intelligent Contracts cannot directly access web data without relying on traditional oracles.
```

The same two official GenLayer documentation sources were used.

### Result

```json
{
  "claim": "GenLayer Intelligent Contracts cannot directly access web data without relying on traditional oracles.",
  "decision": "FALSE",
  "has_resolved": true,
  "is_finalized": true,
  "source1_status": "AVAILABLE",
  "source1_verdict": "REFUTES",
  "source2_status": "AVAILABLE",
  "source2_verdict": "REFUTES"
}
```

Transaction result:

`SUCCESS / ACCEPTED`

Explorer:

https://explorer-studio.genlayer.com/tx/0x1499d5b35194bdedff5f2d8d591d19af71953b28c40352dc0be088224ae68e16

### Observation

Both sources independently refuted the claim.

The contract therefore produced the definitive binary decision `FALSE`.

Because the adjudication is definitive:

```text
has_resolved = true
is_finalized = true
```

The resolver instance cannot be adjudicated again.

---

## Test 3 — One Source Unavailable

### Claim

```text
GenLayer Intelligent Contracts can directly access web data without relying on traditional oracles.
```

### Source 1

A valid GenLayer documentation page:

```text
https://docs.genlayer.com/developers/intelligent-contracts/features/web-access
```

### Source 2

An intentionally unavailable BBC fixture URL:

```text
https://www.bbc.com/sport/football/scores-fixtures/2099-01-01
```

### Result

```json
{
  "claim": "GenLayer Intelligent Contracts can directly access web data without relying on traditional oracles.",
  "decision": "SOURCE_UNAVAILABLE",
  "has_resolved": false,
  "is_finalized": true,
  "source1_status": "AVAILABLE",
  "source1_verdict": "SUPPORTS",
  "source2_status": "SOURCE_UNAVAILABLE",
  "source2_verdict": "INSUFFICIENT"
}
```

Transaction result:

`SUCCESS / ACCEPTED`

Explorer:

https://explorer-studio.genlayer.com/tx/0xd5557b70a186bcddd66ec1aaaae7e60c7569c2c2281d33a4a9197ee235ba3a44

### Observation

The contract did not force a binary decision when one required evidence source could not be fetched.

Instead, it returned:

```text
SOURCE_UNAVAILABLE
```

This is not considered a definitive binary resolution, so:

```text
has_resolved = false
```

However, the adjudication of this deployed claim/evidence instance is complete:

```text
is_finalized = true
```

The same instance therefore cannot be repeatedly rerolled against unchanged evidence.

If evidence availability changes, a new resolver instance must be created.

---

## Test 4 — Conflicting Evidence

This test used intentionally contradictory synthetic evidence included in the repository.

### Claim

```text
The Crypto Lab test claim is true.
```

### Supporting Source

```text
https://raw.githubusercontent.com/hoangnam1808/genlayer-dual-source-claim-resolver/main/evidence/supports.md
```

### Refuting Source

```text
https://raw.githubusercontent.com/hoangnam1808/genlayer-dual-source-claim-resolver/main/evidence/refutes.md
```

### First Adjudication

The first call to `resolve()` successfully produced:

```json
{
  "decision": "CONFLICTING_EVIDENCE",
  "source1_status": "AVAILABLE",
  "source1_verdict": "SUPPORTS",
  "source2_status": "AVAILABLE",
  "source2_verdict": "REFUTES"
}
```

Transaction result:

`SUCCESS / ACCEPTED`

Explorer:

https://explorer-studio.genlayer.com/tx/0xb755c0e64935dbbd089f3d1562cf4fe67fb9d71c2586951632d1788530e621ae

### State After First Adjudication

A subsequent `get_resolution_data()` call returned:

```json
{
  "claim": "The Crypto Lab test claim is true.",
  "decision": "CONFLICTING_EVIDENCE",
  "has_resolved": false,
  "is_finalized": true,
  "source1_status": "AVAILABLE",
  "source1_verdict": "SUPPORTS",
  "source2_status": "AVAILABLE",
  "source2_verdict": "REFUTES"
}
```

The result remains non-binary:

```text
has_resolved = false
```

but the current adjudication is permanently completed:

```text
is_finalized = true
```

### Observation

The resolver explicitly preserved the disagreement instead of forcing the evidence into `TRUE` or `FALSE`.

Unlike the previous implementation, the conflicting result cannot be repeatedly rerun against the same deployed claim/evidence instance.

---

## Test 5 — Retry Protection on Unchanged Conflicting Evidence

After Test 4 finalized the resolver instance as `CONFLICTING_EVIDENCE`, `resolve()` was called again on the exact same deployed contract instance with unchanged claim and evidence.

Explorer:

https://explorer-studio.genlayer.com/tx/0xa7aa6feec2a3f25a656b92e9913d158d1b10e053dfd95252a77304b7af624208

Transaction result:

`ERROR / ACCEPTED`

The call reverted with:

```text
Adjudication already finalized
```

### Observation

This directly addresses the stochastic-reroll issue identified during steward review.

Without this protection, a caller could repeatedly adjudicate the same ambiguous or conflicting evidence until one nondeterministic round happened to produce:

```text
SUPPORTS + SUPPORTS
```

or:

```text
REFUTES + REFUTES
```

and then permanently finalize that stochastic outcome.

V3 prevents this behavior.

Every accepted adjudication now sets:

```text
is_finalized = true
```

regardless of whether the outcome is:

- `TRUE`
- `FALSE`
- `CONFLICTING_EVIDENCE`
- `SOURCE_UNAVAILABLE`
- `UNDETERMINED`
- `INVALID_RESULT`

Any subsequent call to `resolve()` on that same instance is rejected.

Revised or newly available evidence requires deployment of a new resolver instance.

---

## Resolution Policy Demonstrated

```text
Two sources independently evaluated
                ↓
        Same SUPPORT verdict
                ↓
               TRUE
                ↓
   has_resolved = true
   is_finalized = true
                ↓
       No further rerolls
```

```text
Two sources independently evaluated
                ↓
         Same REFUTE verdict
                ↓
               FALSE
                ↓
   has_resolved = true
   is_finalized = true
                ↓
       No further rerolls
```

```text
Sources disagree
                ↓
     CONFLICTING_EVIDENCE
                ↓
   has_resolved = false
   is_finalized = true
                ↓
       No further rerolls
```

```text
Required source unavailable
                ↓
      SOURCE_UNAVAILABLE
                ↓
   has_resolved = false
   is_finalized = true
                ↓
       No further rerolls
```

```text
New or revised evidence
                ↓
     Deploy new resolver instance
                ↓
        New adjudication
```

---

## Finalization Policy

Each deployed resolver instance now permits exactly one accepted adjudication.

`is_finalized` records whether the current claim/evidence instance has completed that adjudication.

`has_resolved` has a narrower meaning:

```text
has_resolved = true
```

only when the result is the definitive binary outcome `TRUE` or `FALSE`.

This distinction allows the contract to preserve uncertainty without allowing indefinite stochastic retries.

---

## Main Finding

The contract demonstrates a reusable evidence-adjudication pattern in which:

- validators independently fetch and evaluate two evidence sources;
- external source failures become explicit application states;
- conflicting evidence is represented rather than hidden;
- dual-source agreement produces definitive `TRUE` or `FALSE` resolution;
- non-definitive outcomes preserve uncertainty without remaining indefinitely rerollable;
- every accepted adjudication finalizes the deployed resolver instance;
- repeated calls against unchanged evidence are rejected;
- new or changed evidence requires a new resolver instance.

The design does **not** claim that agreement between two sources guarantees objective truth.

It demonstrates how GenLayer Intelligent Contracts can preserve uncertainty and disagreement as explicit contract states while preventing repeated nondeterministic evaluation from being used to grind toward a preferred permanent result.
