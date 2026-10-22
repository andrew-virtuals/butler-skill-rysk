---
name: rysk
description: Earn on idle USDC by selling a fully collateralised put on Rysk - and if it lands in the money, the owner actually receives the asset rather than a loss in cash.
version: 0.1.0
metadata: {"butler":{"moneyMoving":true,"keywords":["rysk","earn on my usdc","idle cash","make my money work","sell a put","cash secured put","get paid to wait","buy the dip","premium","options income","yield","hype","physically settled"],"requires":{"bins":["python3","bevo-read","acp"]}}}
---

## When to use

Your owner has USDC doing nothing and would be happy to own BTC, ETH, SOL or HYPE if
it got cheaper: "earn on my USDC", "get paid to buy the dip", "what can I earn on
this", "make my cash work".

They sell one **cash-secured put**: an agreement to buy the asset at a lower price,
for a premium they keep whatever happens. Real market makers quote it over RFQ, and
the quote comes back signed.

The thing that makes Rysk different from a cash-settled venue, and the reason to say
it out loud: **it settles physically.** If the put lands in the money the owner
receives the asset at the strike. They are not handed a loss in cash and told they
"bought" something they never got.

Not this skill: buying options, covered calls, anything Rysk is not currently quoting,
or a quote the helper refuses.

## Before you start

1. **Would they genuinely own it at that price?** This is the whole question. A put is
   an agreement to buy. If the honest answer is no, there is no trade here - say so
   rather than finding a strike that hides it.

2. **The money is locked until expiry.** Collateral sits against the put for the full
   tenor. Nothing else can spend it and there is no early exit in this version.

3. **Say the bad case in their words, before any yield number.** Not "if assigned" -
   *"ETH drops to $1,800 and you have bought it at $2,150 anyway. You are down about
   $700, and the $95 does not cover that."*

4. **First time, three plain yes answers.** A vague one is a no:
   - "If it falls hard you end up owning it at the strike, at a loss. Clear?"
   - "Your money is locked until the expiry date. Clear?"
   - "The premium is yours whatever happens. The collateral is not. Clear?"

## Procedure

1. [ADAPT] Read what they hold, so the offer is about their actual money:

   ```sh
   bevo-read assets
   ```

2. [FIXED] Write the script in `references/helper.md` to a file, then ask Rysk what it
   will pay right now. Never assume an asset is quotable:

   ```sh
   python3 /tmp/rysk.py screen --collateral <THEIR_USD> --tenor monthly
   ```

   Offer only `tradeable: true` rows, and read any `notes` out as given.

3. [FIXED] Quote the one they chose. Let them pick the asset - never rank by APR:

   ```sh
   python3 /tmp/rysk.py quote --asset <ASSET> --collateral <USD> --tenor monthly
   ```

   Use `--delta 0.20` instead when they want the odds of assignment held steady
   rather than the distance from spot.

4. [ADAPT] Offer it using **only** fields from the quote, bad case first. Never
   compute a premium, an APR or a breakeven yourself:

   ```text
   $5,000 of USDC, earning $94.67 over 36 days - about 22% a year if you kept
   doing it. If ETH is above $2,150 on 27 Nov you keep the $4,300 and the $94.67.
   If it is below, you own 2 ETH at $2,150 - actually own it, not a cash loss.
   ```

   Say the `notes` too: Rysk's strikes and sizes step, so a round amount rarely
   lands exactly.

5. [ADAPT] Get a **separate** yes to the ownership question, in the asset's own
   terms: *"Happy to own 2 ETH at $2,150 on 27 November?"* A yes to the yield is not
   a yes to that, and a no ends it - do not counter with another strike or asset.

6. [FIXED] Sell it. The quote is signed and dies sixty seconds after it is issued,
   so re-quote immediately before signing and never sign a stale one:

   ```sh
   python3 /tmp/rysk.py quote --asset <ASSET> --collateral <USD> --tenor monthly
   acp wallet sign-typed-data --chain-id 999 --idempotency-key <KEY>:sell
   ```

   `valid_for_seconds` on the fresh quote says how long is left. Under twenty, quote
   again rather than racing it.

## Idempotency and retries

Reads are free - re-run them rather than reasoning from a quote that is a minute old,
because a minute is longer than a quote lives.

Selling is a money command and is **not** idempotent. One `--idempotency-key` per
position, and if the outcome is unclear **do not re-run it** - a retried sell is a
second put against collateral the owner only agreed to commit once:

```sh
bevo-read request <KEY>:sell
```

## Failure handling

| Outcome | What to do |
| --- | --- |
| `screen` offers nothing | Say nothing is worth doing today and why, in their words. Never loosen the gate to find something. |
| `is offering no puts on it right now` | Rysk lists the asset but is not quoting it. Not a retry - pick another or wait. |
| `would not price $X of <asset>` | Usually under the minimum: Rysk steps per asset. Give the figure from the reason and offer that size. |
| `probably be exercised` | The strike is too close to spot to be idle-cash earning. Offer a further one; do not talk them past it. |
| Quote expired before signing | Re-quote and re-offer. Never sign a stale quote. |
| Owner says no to owning the asset | That ends it. Do not offer a different strike, tenor or asset - every put carries the same promise. |
| Rysk unreachable | Say Butler cannot see the market right now and quote nothing. |

## Limits

- Selling cash-secured puts only, fully collateralised, locked until expiry. No
  buying, no covered calls, no early close in this version.
- **Only what Rysk is quoting**, which changes: BTC, ETH, SOL and HYPE have been live;
  PUMP, PURR, XAUT, XRP and ZEC are listed but often not quoted.
- **Sizes step per asset** and the minimums are real money - roughly $1,000 on ETH and
  SOL, $3,500 on HYPE and BTC. A smaller amount cannot be filled at all.
- **Quotes live sixty seconds.** Anything slower than that has to be re-quoted.
- Physically settled: assignment delivers the asset, not a cash loss.
- Not advice. Offer what passes the screen; never rank, never recommend.

## Say to the owner

Lead with the money, name the date, then the bad case with equal weight:

"Your $5,000 can earn **$94.67** over 36 days - about **22% a year** if you kept
rolling it. If ETH is above $2,150 on 27 November, you keep your money and the
premium. If it is below, you own 2 ETH bought at $2,150 - and you really do own it,
which is the difference here. Your money is locked until then."

When nothing clears: "Nothing worth doing today - Rysk is not quoting anything I would
put your money into. I will look again tomorrow."

When they say no to owning it: "Then this is not for you, and that is the right
answer. Every one of these is a promise to buy."
