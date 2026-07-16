# DEX.DO Stuff

Personal documentation, test reports, and setup notes for using the [`dexdo`](https://github.com/gosh-sh/dexdo-cli) CLI on the Acki Nacki shellnet testnet, maintained by [jeruzzalem](https://github.com/jeruzzalem).

This is an **independent, unofficial** documentation repo — not affiliated with or maintained by the GOSH/dexdo team. It's a personal log of setup steps, bugs encountered, and findings while testing the DEX.DO inference marketplace.

**Upstream project:** https://github.com/gosh-sh/dexdo-cli

## Contents

- [`reports/`](./reports) — dated investigation/bug reports from testing sessions, including full reproduction logs.
- [`setup/`](./setup) — step-by-step setup notes for installing and configuring `dexdo` on Linux.

## Reports index

| Date | dexdo version | Summary |
|---|---|---|
| [2026-07-09](./reports/2026-07-09_v0.0.7_seller_offer_not_resting.md) | v0.0.7 | Seller offers fail to rest in the `InferenceOrderBook` after `postSellOffer`; buyer-side escrow/reclaim logic confirmed working correctly. |
| [2026-07-16](./reports/2026-07-16_v0.0.12_retest.md) | v0.0.12 | Retest after upgrade: seller-side issue from the 2026-07-09 report appears fixed. New buyer-side bug found (`buyer pre-submit matcher head differs from the rendered quote`) with evidence the order book was static during the failure window. |

## Disclosure

These reports were produced with the assistance of Claude (Anthropic) as a debugging/documentation aid during live testing sessions. All commands were run by the author on their own machine; findings and conclusions are the author's own assessment of observed CLI behavior and are not verified against `dexdo`'s source code.
