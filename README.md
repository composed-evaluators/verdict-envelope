# verdict-envelope

**Composed-evaluator artifact: ThoughtProof judgment + invinoveritas anchor — independently attributed and recomputable.**

Two independent verifiers compose one artifact without either absorbing the other:

- **ThoughtProof (Sentinel)** produces the *judgment* — a byte-deterministic `sentinel.verdict.canonical.v1` body.
- **invinoveritas** produces the *anchor* — it hashes that body verbatim, signs it (Nostr / BIP-340), and issues a receipt via `POST /witness`.

A third party reading the final proof can recompute **two separate claims**:

1. *"ThoughtProof produced this verdict"* — recompute `sha256` over the JCS serialization of the canonical body and match `body_hash`.
2. *"invinoveritas witnessed it at time T"* — verify the signed event offline via NIP-01.

Neither claim absorbs the other. That's the **separable-attribution** property this repo exists to make structural.

---

## What each side vouches for (and does not)

| | ThoughtProof (judgment) | invinoveritas (anchor) |
|---|---|---|
| **Vouches for** | The verdict, confidence, objections, reasoning, models used | Receipt + timestamp + byte-integrity of the body it received |
| **Does NOT vouch for** | Anchoring/timestamp (that's invinoveritas's layer) | Authorship, soundness, or re-verification of the content — `source` is **self-declared, not cryptographically verified** by invinoveritas |

The `/witness` proof bakes in a note that `source` is self-declared. So the composed artifact never lets one party's guarantee silently stand in for the other's.

---

## The canonical body (`sentinel.verdict.canonical.v1`)

Schema: [`schemas/sentinel.verdict.canonical.v1.schema.json`](schemas/sentinel.verdict.canonical.v1.schema.json)
Sample (JCS-serialized, byte-exact): [`canonical.json`](canonical.json)

Produced by ThoughtProof from a Sentinel verify response via `buildCanonicalSentinelVerdict()` (source of truth: `thoughtproof-sentinel/src/canonical-verdict.ts`), then JCS-canonicalized (RFC 8785).

### JCS determinism rules (load-bearing)

- Optional fields (`models.secondary`, `gate`) are **omitted entirely** when absent — never `null`. JCS treats absent vs null distinctly, so a stray null changes the bytes and breaks recomputability.
- `confidence` is a **0-100 integer** in the canonical body (the live API response carries a 0-1 float; the canonical form rounds and clamps).
- Serialize with RFC 8785 JCS, then `sha256`.

### Recompute the sample hash yourself

```bash
# canonical.json in this repo is already JCS-serialized bytes.
printf '%s' "$(cat canonical.json)" | shasum -a 256
```

For the sample in `canonical.json` this yields:

```
419c360db82ee72be3411acd2d30f560b3f62842c2162fa3cb4a08c1fa4ce65a
```

(anchored form: `0x419c360db82ee72be3411acd2d30f560b3f62842c2162fa3cb4a08c1fa4ce65a`)

---

## The witness round-trip

```
ThoughtProof (Sentinel)                    invinoveritas
-----------------------                    -------------
POST /sentinel/verify
  → critique runs (nano→swift cascade)
  → SentinelVerifyResponse
  → buildCanonicalSentinelVerdict(resp)
  → canonical body (JCS bytes)
  → POST /witness ─────────────────────▶   receives body verbatim
    { "source": "sentinel.thoughtproof.ai",  hashes body AS-IS (unmodified, unjudged)
      "body":   "<JCS bytes as string>" }    signs it (Nostr / BIP-340)
                                             returns a signed proof carrying:
                                               source (self-declared) + body verbatim + body_hash
  ◀──────────────────────────────────────   + note: source is self-declared, not IVV-verified

Anyone                                     invinoveritas
------                                     -------------
POST /verify-proof (free, no auth) ──────▶  confirms the signed event (or verify offline via NIP-01)
```

ThoughtProof never calls invinoveritas's judgment path; invinoveritas never calls ThoughtProof's. Each hands the other exactly one artifact.

---

## Endpoints (as observed live, 2026-07)

- Witness (anchor a third party's claim, unmodified): `POST https://api.babyblueviper.com/witness`
  - Body: `{ "source": string, "body": string }`
  - Output schema: `invinoveritas.witnessed_claim.v1` (`{ schema, source, body_hash, ... }`)
  - Auth: Bearer (free registration via `POST /register`), or x402 (USDC on Base), or Lightning.
- Verify a proof (free, no auth): `POST https://api.babyblueviper.com/verify-proof`

> **Pricing note:** the `100 sats/call` on `/witness` is invinoveritas's standard endpoint price, not a ThoughtProof price. Funding path for the joint demo is a collab detail, TBD between the two teams — not encoded here.

---

## Status

- [x] Canonical body schema + byte-exact sample committed (code-verified against `canonical-verdict.ts`).
- [x] `/witness` + `/verify-proof` live and reachable.
- [x] Live-Sentinel body (real `verificationId`) — see [`samples/live-sentinel-verdict.json`](samples/live-sentinel-verdict.json), produced by a real `POST /sentinel/verify` and canonicalized by ThoughtProof's `buildCanonicalSentinelVerdict()`. Recompute: `sha256` over its `jcs` field equals its `body_hash` (`0x27f0accb…`).
- [x] End-to-end witness round-trip — fired (2026-07-06, invinoveritas side, 100 sats): `samples/live-sentinel-verdict.json`'s `jcs` handed to `POST /witness` as `{source: "sentinel.thoughtproof.ai", body: <jcs>}`. Signed proof event id `98a2d71a22a045825d431bc3c315953992c093f0b8c818a2f195116537557529`, returned `body_hash` `27f0accb9d5b02afbbc673b5d1adbf646a002220bece6b3d1fe764504074a84b` (matches). Independently confirmed valid via `POST /verify-proof` (`id_integrity`/`signature_valid`/`issued_by_invinoveritas` all `true`) — see [`samples/witness-proof.json`](samples/witness-proof.json) for the full signed event, recomputable by anyone via NIP-01.
- [x] **Live *trading-agent* verdict** — [`samples/live-vta-trade-verdict.json`](samples/live-vta-trade-verdict.json). Where the first live sample verified a generic reasoning example, this one is a verdict on a **real decision a live trading agent actually made**: a 2x-short blow-off-top fade on VANRY (RSI 92.9 after a +186% 7-day parabola), decided 2026-07-07. The Sentinel engine judged the *reasoning* (`trade_reasoning` mode, `serv-nano→serv-swift` cascade) and returned **`UNCERTAIN`** — verificationId `sent_b982f4f28b0f4f03`, `body_hash` `0xb70758bd5592535c4f5634c7a7555f81b25890c107e724abd763790013e2fa1b`. Recompute: `sha256` over its `jcs` equals its `body_hash` (verified). The verdict's `confidence` is `100` because the three reasoning steps each checked out as *faithful* — the `UNCERTAIN` comes from the cascade split (`primary=HOLD, secondary=ALLOW`): the two models disagreed on whether to *act* on a coherent-but-extreme thesis. Anchoring a genuinely marginal verdict — not just a clean ALLOW — is the point: the envelope carries the hard cases too.
  - [x] Witness round-trip for this sample — fired (2026-07-07, invinoveritas side, 100 sats): the `jcs` above handed to `POST /witness` as `{source: "sentinel.thoughtproof.ai", body: <jcs>}`. Signed proof event id `caf1dd62a4634ab84d555d365b626d7e752b3853e88ac5a701249e87191d215c`, returned `body_hash` `b70758bd5592535c4f5634c7a7555f81b25890c107e724abd763790013e2fa1b` (matches). Independently confirmed valid via `POST /verify-proof` (`id_integrity`/`signature_valid`/`issued_by_invinoveritas` all `true`) — see [`samples/witness-proof-vta.json`](samples/witness-proof-vta.json) for the full signed event, recomputable by anyone via NIP-01.
- [x] **Re-review chain** — [`samples/rereview-chain-vanry.json`](samples/rereview-chain-vanry.json). A `hold → resolve` sequence where the second verdict carries the first's `body_hash` as `parentBodyHash` — so the whole re-plan sequence is one independently-walkable chain (scheme `sentinel.rereview.chain.v1`). Both links recompute cleanly and the parent-ref checks out. Generated in a single pass so verdicts, objections, and the chain are mutually consistent.
  - **What it proves (and what it deliberately does not):** the chain makes the sequence **observable** — nobody can hide a hold, drop an inconvenient objection, or present a re-roll as one clean pass. It does **NOT** structurally prevent "re-review until you get the answer you want." The resolve verdict is a *fresh independent judgment* on the revised plan, not a check that the revision refuted the specific objection. (Notably, in this sample the revised, risk-reduced 1x plan still lands `UNCERTAIN` — the resolve verdict doesn't reward the agent for merely responding.) Turning observability into structural prevention needs an objection-carrying **falsifiable predicate** the re-plan must satisfy — the follow-on design. A naive too-strict version of that check fails *closed* (every path → UNCERTAIN, indistinguishable from working), so it must be calibrated against real cases before going live.
  - [x] Witness round-trip for the chain — fired (2026-07-07, invinoveritas side, 200 sats total): both links' `jcs` handed to `POST /witness` as `{source: "sentinel.thoughtproof.ai", body: <jcs>}` in order. **hold** → signed proof event id `620875466b0a68ca4d8732c4cc94af045c5b23307fdfa347e27591aba0441d82`, `body_hash` `684b9b2543974880deab72bbe5ea37a5a3ef2912552be3b6546cb56485740b7d` (matches). **resolve** → signed proof event id `b71d81aeafcd09a160c6954b42576d6e455cf99d7b48a0c1ebe6db61d0c7d09c`, `body_hash` `278899e6277a8486474dc83acf4084b928201367e09495625a4311d4d5d6161b` (matches). Both independently confirmed valid via `POST /verify-proof` (`id_integrity`/`signature_valid`/`issued_by_invinoveritas` all `true`) — see [`samples/witness-proof-rereview-hold.json`](samples/witness-proof-rereview-hold.json) and [`samples/witness-proof-rereview-resolve.json`](samples/witness-proof-rereview-resolve.json) for the full signed events, recomputable by anyone via NIP-01. The chain is now anchored end-to-end: `hold`'s anchor precedes `resolve`'s anchor in time, and `resolve.parentBodyHash` ties back to `hold.body_hash` — a third party can walk the whole sequence from public data alone.
- [x] **Predicate-gated chain (structural prevention, for the numeric classes)** — [`samples/rereview-chain-predicate-gated.json`](samples/rereview-chain-predicate-gated.json). The follow-on the previous entry pointed at. Here the `resolve` gate is a **literal boolean**, not a model judgment: the hold authors a falsifiable predicate from its structured `{kind, claimedValue, actualValue}` flag (`{field: "move_pct", op: "approx", value: 41, tolerancePp: 10}`), and the resolve gate is `|revisedValue − value| ≤ tolerance` → `|44 − 41| ≤ 10 = true`. Anyone can recompute the gate from public data; it **cannot degenerate to `UNCERTAIN`** because there is no judgment left. This is the first sample where *"prevented" is provably true* for the objection, not just claimed.
    - **The honest boundary, marked per-objection:** the sample carries two objections. The `magnitude` one is `predicate-gated` (structurally prevented). The `inferential_integrity` one is `fresh-judgment-only` (detectable, not prevented — it doesn't reduce to a field comparison). Enforcement is surfaced per objection, never one blanket level for the verdict. Only the three numeric classes (direction / magnitude / range_position) are predicate-gated today; the fuzzy classes stay with fresh judgment, honestly labeled.
    - [x] Witness round-trip for this chain — fired (2026-07-07, invinoveritas side, 200 sats total): both links' `jcs` handed to `POST /witness` in order. **hold** → event id `685c8282aec2a852012147e7617cf26de41a894f185463518beefe8b7f16fd52`, `body_hash` `4a972b315c49a94fe56186d28ac1ef90470a4efdc3021cf5963ca385d84c9b8f` (matches). **resolve** → event id `b398f83a8c8f53d9fcc36d8b413e7f734a784bc88158b80dc8df501b99519b15`, `body_hash` `76089af3cb587b3b3f19b702ff0663efa6d6819b4fce674f636999ea5f2103e3` (matches). Both independently confirmed valid via `POST /verify-proof` before committing — see [`samples/witness-proof-predicate-hold.json`](samples/witness-proof-predicate-hold.json) and [`samples/witness-proof-predicate-resolve.json`](samples/witness-proof-predicate-resolve.json), recomputable by anyone via NIP-01. This is the first chain in this repo where the whole sequence — hold → predicate → resolve → anchor — is publicly recomputable end to end, and the "prevented, not just detectable" claim is checkable by a third party, not just asserted.

---

*Repo: `composed-evaluators/verdict-envelope`. Maintained jointly by ThoughtProof and invinoveritas.*
