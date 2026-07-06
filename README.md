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

---

*Repo: `composed-evaluators/verdict-envelope`. Maintained jointly by ThoughtProof and invinoveritas.*
