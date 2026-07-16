# Setup notes

Quick-reference setup flow for `dexdo` on Linux, based on hands-on testing. For the authoritative instructions, always defer to the official app.dex.do "Set up PrivateNotes" guide and the [dexdo-cli README](https://github.com/gosh-sh/dexdo-cli).

## 1. Install

```bash
curl -fsSL https://github.com/gosh-sh/dexdo-cli/releases/latest/download/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
dexdo --version
```

## 2. Verify

```bash
dexdo doctor
```

Should report `PASS network=shellnet`. If contracts look stale (e.g. after a shellnet relaunch), refresh `models.json` and `contracts/` from the latest release tarball rather than reusing old ones.

## 3. Policy

```bash
dexdo policy init --role both
```

Fill every field (no `UNSET` allowed for real buyer/seller startup). As of v0.0.12, only `after_deal_done=retire` and `buyer_no_show=retire_gateway` are supported seller-side runtime values, despite the schema accepting others like `republish_with_backoff`.

## 4. Create a PrivateNote

One note can serve as both buyer and seller (per the official guide) — a second note is only needed if you want to run buyer and seller as two independent, simultaneously-active identities.

```bash
dexdo note deploy \
  --multisig-address <WALLET_ADDRESS> \
  --multisig-seed-file wallet.seed \
  --nominal N10000 \
  --token-type nackl \
  --endpoint shellnet.ackinacki.org \
  --pool pn_pool.json
```

If this fails with `wallet busy/out-of-sync` repeatedly (even after long waits, even via `--recovery` resume), the wallet itself may have a stuck on-chain message — generating a **fresh** wallet from app.dex.do resolved this in testing rather than continuing to retry the same wallet.

## Known gotchas

- Env vars (`NOTE_ADDR`, `SELLER_NOTE_ADDR`, `DEXDO_PN_POOL`) don't persist across terminal tabs — `source` a saved `env.sh` in each new tab.
- `DEXDO_PN_POOL` must be set (env var, not a flag) before `dexdo buyer` will submit a real escrow.
- Never run shell placeholder syntax like `<WALLET-ADDRESS>` literally — substitute the real value first, or the shell will misinterpret `<`/`>` as redirection.
- `rm` on a filename starting with `--` needs `rm -- filename` or `rm ./filename`.
