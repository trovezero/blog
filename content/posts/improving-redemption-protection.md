
+++
title = "Why 'Debt Ahead' isn't enough: How we improved redemption protection"
date = "2026-03-14"
+++

At Trove Zero, our core promise is simple: we keep your Liquity V2 loan safe from redemption while minimizing the interest rate you pay. The bot continuously monitors the protocol state and positions your trove so there's enough "protective debt" ahead of you. Debt that would need to be redeemed before yours gets touched.

It worked. Until it didn't. A while ago several troves managed by Trove Zero on the wstETH branch were partially redeemed. According to our logs, the bot had calculated roughly millions of BOLD debt sitting ahead of our troves, a comfortable buffer by any measure. Yet a redemption of just ~220,000 BOLD was enough to reach our position.

How is that possible?

### The Investigation

We dug into the on-chain data and traced the redemption block by block. What we found was surprising: the millions of BOLD "buffer" consisted of a **single external trove**. One borrower, sitting just below our interest rate, with a massive position.

Between two of the bot's monitoring cycles (only minutes apart), that borrower raised their interest rate above ours. In a single transaction, our 9 million buffer collapsed to essentially zero. The redemption that followed had no debt to work through before hitting our troves.

The bot's calculation wasn't wrong, at the time it checked, everything looked very healthy. But that buffer was *fragile*. It depended entirely on one actor's decision.

### The Problem with Raw "Debt Ahead"

The original approach treated all debt below our rate equally. Whether the buffer consisted of 100 small independent troves or a single whale position, the bot saw the same number and drew the same conclusion: "we're safe."

But these two scenarios have fundamentally different risk profiles:

- **100 troves × 90,000 BOLD each**: For the buffer to disappear, all 100 borrowers would need to independently raise their rates above ours. Extremely unlikely to happen simultaneously.

- **1 trove × 9,000,000 BOLD**: One rate adjustment by one borrower, and the entire buffer vanishes.

A buffer of 9 million from 100 troves is robust. A buffer of 9 million from just 1 trove is risky.

### The Fix: Buffer Concentration Caps

We've introduced a per-trove cap on how much any single position can contribute to the perceived buffer. No individual trove may account for more than 35% of the target buffer amount.

Here's the effect with current mainnet numbers for wstETH:

- **Before**: The whale trove contributed its full ~5.7M BOLD to the buffer → bot reported a healthy buffer → `RateOk`
- **After**: The whale's contribution is capped at ~2M BOLD → bot correctly detects insufficient robust buffer → raises the rate to build diversified protection

The cap only activates when there are at least 3 troves in the branch. In thin markets with very few participants, we fall back to the uncapped calculation, since diversification simply isn't possible.

### Smarter Emergency Response

The cap also revealed a secondary issue in our emergency logic. When the buffer drops below the minimum threshold, the bot previously jumped to an aggressive target (the upper band of the comfort zone). With the cap reducing perceived buffer sizes, this aggressive target became unreasonably high — it would have pushed interest rates well beyond market norms.

We've recalibrated the emergency target to the 3/4 point between the mid and max bands. This provides a meaningful safety boost during emergencies without overshooting into absurd territory.

### What This Means for You

If you're a Trove Zero user, you don't need to do anything. The updated bot will:

1. **Detect fragile buffers** that depend on a small number of large positions
2. **Proactively adjust your rate** to build protection from many independent troves rather than relying on a single whale
3. **React proportionally** during emergencies, targeting realistic rate levels

The trade-off is that in some market conditions, the bot may set a slightly higher interest rate than before. But this reflects the true cost of reliable redemption protection — not the illusory safety of a concentrated buffer.

### Lessons Learned

Building a rate optimization bot for a novel protocol is an exercise in discovering edge cases. The math can be correct while the assumptions are wrong. In our case, summing up debt was arithmetically sound but failed to capture a critical dimension: how that debt was distributed.

We're continuing to improve Trove Zero's risk assessment and will keep sharing what we learn along the way.
