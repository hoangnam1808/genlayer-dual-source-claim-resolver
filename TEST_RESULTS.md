# GenLayer Dual-Source Claim Resolver — Test Results

## Test Environment

The tests below were executed in **GenLayer Studio** using:

- Execution mode: `Normal (Full Consensus)`
- GenLayer Python SDK: `v0.2.16`
- Contract: `contracts/dual_source_claim_resolver.py`
- Resolution model: two independently fetched and evaluated web sources

The objective was to verify definitive agreement, disagreement, and source-failure behavior.

---

## Summary

| Test case | Expected | Actual |
| --- | --- | --- |
| Both sources support claim | `TRUE` | ✅ `TRUE` |
| Both sources refute claim | `FALSE` | ✅ `FALSE` |
| One source unavailable | `SOURCE_UNAVAILABLE` | ✅ `SOURCE_UNAVAILABLE` |
| Sources directly conflict | `CONFLICTING_EVIDENCE` | ✅ `CONFLICTING_EVIDENCE` |
| Conflict state persistence | Retryable | ✅ `has_resolved = false` |

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
  "decision": "TRUE",
  "source1_status": "AVAILABLE",
  "source1_verdict": "SUPPORTS",
  "source2_status": "AVAILABLE",
  "source2_verdict": "SUPPORTS"
}
```

Transaction result:

`SUCCESS / ACCEPTED`

Explorer:

https://explorer-studio.genlayer.com/tx/0x842b82c4433e49bfa957de2f8e38983e6807ff32cdccf34bb629d46db7eb15af

### Observation

Both independently evaluated sources supported the claim.

The contract therefore finalized the claim as `TRUE`.

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
  "decision": "FALSE",
  "source1_status": "AVAILABLE",
  "source1_verdict": "REFUTES",
  "source2_status": "AVAILABLE",
  "source2_verdict": "REFUTES"
}
```

Transaction result:

`SUCCESS / ACCEPTED`

Explorer:

https://explorer-studio.genlayer.com/tx/0x760a8f9cd6a8d944ba121bf35abe2054c5c1423c3ae682b02acf183212934bbb

### Observation

Both sources independently refuted the claim.

The contract therefore finalized the claim as `FALSE`.

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
  "decision": "SOURCE_UNAVAILABLE",
  "source1_status": "AVAILABLE",
  "source1_verdict": "SUPPORTS",
  "source2_status": "SOURCE_UNAVAILABLE",
  "source2_verdict": "INSUFFICIENT"
}
```

Transaction result:

`SUCCESS / ACCEPTED`

Explorer:

https://explorer-studio.genlayer.com/tx/0x4c40fb7fc10ce17c4575ba713e0468d904142a6f3c5f96eca29741f0796d035e

### Observation

The contract did not force a binary decision when one required evidence source could not be fetched.

Instead, it returned `SOURCE_UNAVAILABLE`.

The claim remained retryable rather than being permanently finalized.

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

### Result

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

`SUCCESS / FINALIZED`

Explorer:

https://explorer-studio.genlayer.com/tx/0xec8332ca2dc301513eb515192c9c5df34508bd8ff11f51c0644438368e1d28a3

### State After Resolution Attempt

```json
{
  "decision": "CONFLICTING_EVIDENCE",
  "source1_verdict": "SUPPORTS",
  "source2_verdict": "REFUTES",
  "has_resolved": false
}
```

### Validator Observation

During this test, quorum reached an accepted consensus even though one independently evaluating validator disagreed with the accepted result.

This is not treated as a contract failure.

Instead, it demonstrates an important property of nondeterministic adjudication: independently evaluating validators may interpret evidence differently, while GenLayer consensus determines whether sufficient agreement exists.

### Observation

The resolver successfully represented disagreement explicitly instead of forcing the claim into `TRUE` or `FALSE`.

Because the evidence conflicted, the claim remained retryable.

---

## Resolution Policy Demonstrated

```text
Two sources independently evaluated
                ↓
        Same SUPPORT verdict
                ↓
               TRUE
                ↓
             Finalize

Two sources independently evaluated
                ↓
         Same REFUTE verdict
                ↓
               FALSE
                ↓
             Finalize

Sources disagree
                ↓
     CONFLICTING_EVIDENCE
                ↓
       Remain retryable

Required source unavailable
                ↓
      SOURCE_UNAVAILABLE
                ↓
       Remain retryable
```

## Main Finding

The contract demonstrates a reusable evidence-adjudication pattern in which:

- validators independently fetch and evaluate evidence;
- external source failures become explicit application states;
- conflicting evidence is represented rather than hidden;
- only dual-source agreement produces a final binary resolution;
- ambiguous or unavailable evidence remains retryable.

The design does **not** claim that agreement between two sources guarantees objective truth. It demonstrates how GenLayer Intelligent Contracts can preserve uncertainty and disagreement as explicit contract states rather than forcing nondeterministic evidence into a binary answer.
