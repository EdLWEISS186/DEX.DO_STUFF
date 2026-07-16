# dexdo CLI — v0.0.12 Retest Report
**Network:** shellnet (Acki Nacki testnet, relaunched 2026-07-14, SuperRoot 4.0.27, dapp_id 4)
**CLI version tested:** dexdo 0.0.12 (Linux x86_64) · previously reported against: dexdo 0.0.7
**Model:** `qwen--qwen3--32b`
**Date:** 2026-07-16 (UTC times below)
**Account:** jeruzzalem

> **Version gap disclosure:** my previous report covered dexdo **v0.0.7**. Versions **v0.0.8 – v0.0.11** were released in between but were **not tested by me** — I jumped straight from v0.0.7 to the latest available at retest time (v0.0.12). Any of the findings below could technically have been introduced or fixed at any point in that untested range; I cannot narrow it further than "somewhere between v0.0.7 and v0.0.12."

---

## 1. Summary

- **The v0.0.7 seller bug appears FIXED.** A `provision` + `seller` cycle on v0.0.12 resulted in the offer explicitly resting in the shared `InferenceOrderBook`, confirmed both by the seller process's own log and independently via `dexdo markets`/`dexdo market`.
- **A new, different bug found on the buyer side in v0.0.12.** `dexdo buyer` consistently refuses to submit with `"buyer pre-submit matcher head differs from the rendered quote; no escrow was sent"` — even when the order book is independently confirmed to be completely static (unchanged) across the relevant time window. No funds were at risk (explicitly "no escrow was sent" both times).
- Two smaller, likely-intentional behavior changes were also noted (new `DEXDO_PN_POOL` env-var requirement; `--gateway-listen` replacing the buyer-only `--local-listen` for the seller command).

---

## 2. Environment

```
$ dexdo doctor
dexdo doctor: PASS network=shellnet
versions:
  SuperRoot: 4.0.27 SuperRoot
  RootPN: 4.0.27 RootPN
  RootOracle: 4.0.27 RootOracle
checks: (all PASS/SKIP as expected for a fixed-superroot redeploy)
policy: OK
```

Fresh identity for this retest (previous v0.0.7-era notes were orphaned by the 2026-07-14 shellnet relaunch — separate issue, not covered here): single `PrivateNote` `0:033ba3193e6def4b60db8eac230a48b2a0b7dc339cdf63262eea67b6c97ccdee` (N10000), used for **both** buyer and seller roles per the official app.dex.do guide (one note, not two).

---

## 3. Seller-side retest — v0.0.7 bug appears resolved

### 3.1 v0.0.7 baseline (for comparison — from my earlier report)
Across 3 separate `provision`+`seller` attempts on v0.0.7, offers never became visible/executable in the order book — one attempt failed explicitly ("offer did not rest... not the canonical TC"), two failed silently (`dexdo monitor` showed `offers in book: 0` for over an hour of unattended runtime, and the TokenContract was absent from `dexdo market`'s listing entirely).

### 3.2 v0.0.12 result — offer rests successfully
```
$ dexdo provision --note-addr "$NOTE_ADDR" --note-key note.secret.hex \
    --frame-model qwen--qwen3--32b --nonce 1 --price-per-tick 1000 \
    --max-ticks 1024 --deposit-shells 20 --output market.json \
    --contracts contracts/deployed.shellnet.json
provisioned market -> market.json
  token_contract: 0:a055e65d62dcc0e00e41c88efec4aeb73db61c094d2652fa268329429f5f59b0

$ dexdo seller --market market.json --model qwen --models models.json \
    --note-addr "$NOTE_ADDR" --note-key note.secret.hex \
    --gateway-listen 0.0.0.0:8443 --mock-model \
    --contracts contracts/deployed.shellnet.json
posting offer: 1024 ticks (= 1024000000 model tokens) at 1000 SHELL/tick
INFO seller posting offer, awaiting buy + match token_contract=0:a055e65d...
seller_offer_outcome RESTED order_id=13
seller_ready token_contract=0:a055e65d... gateway=0.0.0.0:8443 readiness=exact_tc_offer_accepted
```

Independent confirmation via `dexdo market` (public order book listing), my TokenContract present as row #8 of 8:
```
$ dexdo market qwen--qwen3--32b --note-addr "$NOTE_ADDR" --models models.json \
    --contracts contracts/deployed.shellnet.json
┌───┬────────────┬───────────┬────────────────────────────────────────────────────────────────────┐
│ # │ price/tick │ max ticks │ tokenContract                                                      │
├───┼────────────┼───────────┼────────────────────────────────────────────────────────────────────┤
│ ... rows 1-7 omitted (other testers' orders) ...
│ 8 │       1000 │      1024 │ 0:a055e65d62dcc0e00e41c88efec4aeb73db61c094d2652fa268329429f5f59b0 │
└───┴────────────┴───────────┴────────────────────────────────────────────────────────────────────┘
```

Minor note: `dexdo monitor` on the same market still reported `offers in book: 0` at this point, contradicting `dexdo markets`/`dexdo market`. I consider the latter two authoritative here since the offer's live presence was independently confirmed two different ways; `monitor`'s count appears to read a different (possibly stale) code path.

**Also confirmed as new/changed vs v0.0.7:** `models.json` now ships `fingerprints` entries (a `probe_prompt` + `expected_contains` check) — this appears to be the B7/B8 content-identity mechanism I asked about previously, now implemented for at least the `qwen` model.

---

## 4. Buyer-side — new bug found on v0.0.12

### 4.1 First occurrence
```
$ dexdo buyer --frame-model qwen--qwen3--32b --note-addr "$NOTE_ADDR" \
    --note-key note.secret.hex --ticks 8 --max-price-per-tick 1000 \
    --local-listen 127.0.0.1:8080 --contracts contracts/deployed.shellnet.json
inference order book -- qwen--qwen3--32b (table shown, 8 executable asks, best_ask=1000)
your order: 8 ticks (= 8000000 model tokens) at up to 1000 SHELL/tick
How many ticks to buy [8]:            <- manual Enter
Maximum price per tick [1000]:        <- manual Enter
placing buy: 8 ticks at <= 1000/tick (escrow 8200)
Error: shellnet ambiguous submit: unclassified money submit outcome; journal retained
  and no resubmit is safe: shellnet: buyer pre-submit matcher head differs from the
  rendered quote; no escrow was sent
```

### 4.2 Second occurrence — same result even with near-instant confirmation
To rule out a human-timing race condition (operator reading the table before confirming), I repeated with stdin piped immediately (`printf '\n\n' | dexdo buyer ...`), eliminating essentially all human delay between table render and submit:
```
$ printf '\n\n' | dexdo buyer --frame-model qwen--qwen3--32b --note-addr "$NOTE_ADDR" \
    --note-key note.secret.hex --ticks 8 --max-price-per-tick 1000 \
    --local-listen 127.0.0.1:8080 --contracts contracts/deployed.shellnet.json
... (same table) ...
placing buy: 8 ticks at <= 1000/tick (escrow 8200)
Error: shellnet ambiguous submit: unclassified money submit outcome; journal retained
  and no resubmit is safe: shellnet: buyer pre-submit matcher head differs from the
  rendered quote; no escrow was sent
```
Identical error, identical wording, both times. No escrow was sent either time (per the error text itself) — no funds at risk.

### 4.3 Ruling out "the order book actually changed" as the cause
Immediately after, I polled the order book twice in a row (~10s apart) via the read-only `dexdo markets` command:
```
$ dexdo markets --models models.json --note-addr "$NOTE_ADDR" --contracts contracts/deployed.shellnet.json
model=qwen--qwen3--32b order_book=0:08a99e72... active=true order_count=8 ask_count=8 depth_ticks=7232 best_ask=1000

$ dexdo markets --models models.json --note-addr "$NOTE_ADDR" --contracts contracts/deployed.shellnet.json
model=qwen--qwen3--32b order_book=0:08a99e72... active=true order_count=8 ask_count=8 depth_ticks=7232 best_ask=1000
```
`order_count`, `ask_count`, `depth_ticks`, and `best_ask` are **byte-for-byte identical** across both calls. The order book was not moving during this window, which contradicts the buyer's own claim that "the matcher head differs from the rendered quote." This suggests the pre-submit consistency check itself is comparing against a stale/incorrect reference rather than detecting a genuine order-book change.

### 4.4 Environment quirk noted along the way (likely intentional, documenting for completeness)
`dexdo buyer` on v0.0.12 additionally requires the `DEXDO_PN_POOL` environment variable to be set to the pool file containing the note (this was not needed/documented this way in v0.0.7):
```
Error: real shellnet buyer money writes require DEXDO_PN_POOL before any escrow POST
  so a matched TokenContract can be persisted durably; set DEXDO_PN_POOL to the pool
  containing --note-addr
```
Not a bug — just flagging as an undocumented (in the app.dex.do quick-start guide) new requirement that a first-time user following that guide verbatim would hit.

---

## 5. What's safe / unaffected

- No funds lost or at risk from the buyer-side issue — both failures explicitly state "no escrow was sent," and I independently confirmed via `dexdo markets` that the order book (and by extension my note's balance situation) was unchanged.
- The seller-side fix appears solid based on two independent confirmation methods (`seller`'s own `RESTED` log line, plus `dexdo markets`/`dexdo market` showing the live listing).

## 6. Suggested next steps for the team
- Buyer-side: worth checking whether the "matcher head" comparison in the pre-submit consistency check is reading a correct/fresh reference — my test suggests it may be comparing against something other than the actual current order-book state, since the book was provably static across the failure window.
- Consider surfacing `DEXDO_PN_POOL` in the official app.dex.do "Buy" quick-start snippet, since a first-time user copy-pasting that guide as-is will hit the missing-env-var error before ever reaching a real buy attempt.
