# Rysk V12: the facts that shape a note

Measured live on 22 October 2026. Re-measure before relying on any number: this venue
is young and what it quotes changes day to day.

## What it is

An **RFQ venue**, not an order book. You ask, real market makers answer, and the
answer is a **signed quote good for sixty seconds**. There is no spread to screen for,
so the helper screens for the promise instead: how likely assignment is, and whether
the premium pays for locking the money up.

```text
wss://v12.rysk.finance/taker      inventory, request a quote
https://v12.rysk.finance/api/gas  the gas fee, before confirming
```

Both are public. No key, no account, no signature to read or to quote.

## Physical settlement, which is the point

An in-the-money put on Rysk **delivers the asset**. The owner buys it at the strike
and holds it.

That is worth saying plainly because the cash-settled alternative is not the same
thing at all: there, an in-the-money put just takes USDC, and "you now own 2 ETH at
$2,150" is false unless something separately goes and buys the ETH. On Rysk it is
simply true.

It is also what makes the wheel possible - an owner who is assigned actually has the
asset to sell a covered call against.

## What it was quoting, 22 Oct

| Asset | APR | Strike | Size step | Smallest ticket |
| --- | --- | --- | --- | --- |
| HYPE | 25.6% | $70 | 50 | ~$3,500 |
| SOL | 22.5% | $95 | 10 | ~$950 |
| ETH | 22.4% | $2,150 | 0.5 | ~$1,075 |
| BTC | 15.8% | $75,000 | 0.05 | ~$3,750 |

Listed but **not quoting puts**: PUMP, PURR, XRP, ZEC. XAUT lists strikes but refused
every size tried.

So "nine assets" is the inventory, not the menu. The menu is whatever answers today,
which is why the skill screens live rather than carrying a list.

## Sizes step, and the steps are not published

The inventory does not say what size an asset trades in. The venue simply refuses:

```text
trade size too small      under the minimum
trade increment is wrong  not a multiple of the step
```

ETH moves in 0.5, BTC in 0.05, SOL in 10, HYPE in 50. The helper walks a ladder of
candidate steps rather than hard-coding that table, because it will drift.

The practical consequence is that **a round dollar amount rarely lands exactly** -
$5,000 of ETH becomes 2.0 ETH against a $2,150 strike, so $4,300 works and $700 does
not. That is a `note` on every quote, not a refusal.

## `price` is per unit

The quote's `price` is the premium **for one unit**, not the whole position. The same
figure comes back for 0.5, 1 and 1.5 of the same option.

Multiply by the quantity to get what the owner actually receives. Reading it as the
total understates a large position and overstates a small one.

## Sixty seconds

A quote is signed and carries `validUntil`. Measured: exactly **60 seconds** from
issue.

The flow is quote -> gas fee -> sign -> confirm, all inside that. **An approval card
will not survive it** - an owner cannot read and tap in under a minute. Selling needs
a session key that signs without a card, the way the Papertrade skill does.

Until that exists, re-quote immediately before signing and check `valid_for_seconds`.

## Chain and contracts

HyperEVM mainnet, **chain id 999** (998 testnet). Collateral is approved to a spender
that differs per chain, and the EIP-712 verifying contract differs per chain too.
Both are published in Rysk's integration doc rather than here, because a wrong
address sends money to a stranger and an address in a skill is one edit from doing
exactly that.

Two inconsistencies in that doc worth confirming before anyone signs: its EIP-712
`Domain` block shows a placeholder `verifyingContract` and `chainId: 1`, and its
`confirm` example uses `chainId: 84532` where everything else is 999.
