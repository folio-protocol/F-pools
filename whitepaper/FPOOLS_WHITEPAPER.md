# F-pools

## A Dynamic-Leverage Mechanism for Sustainable Synthetic Perpetual Markets

Folio
[author]

---

## Abstract

We propose a mechanism for running a synthetic leveraged market in which the counterparty can never be owed more than it holds. Instead of a payout queue, a margin engine, or a funding rate, the mechanism makes leverage a function of the book. As traders take a larger share of the pool's backing, their leverage falls, and the liquidity side's leverage rises by exactly the same amount. This allows the creation of leveraged markets with a few emergent properties such as: no per-trader positions or liquidations, redemption at any time, no IOUs, a mark price that depends only on the oracle and the book, and a market that goes dormant instead of breaking when the underlying collapses.

---

## 1 Premise

### 1.1 History of Synthetic Perpetuals and Leveraged Tokens

Leveraged exposure without owning the underlying has existed on-chain in several forms. Each answers the same question differently: who takes the other side?

**BitMEX: Perpetual Swaps**
Introduced in 2016, the perpetual swap removed expiry and kept the contract price near spot with a periodic funding payment between longs and shorts. The funding rate became the standard way to pay a counterparty to take the opposite side.

**Synthetix: Pooled Debt**
Synthetix let users mint synthetic assets against staked collateral. Stakers share one debt pool, so they collectively take the other side of every trader.

**GMX and GLP: The Pool as Counterparty**
GMX routed traders against a shared liquidity pool of assets. Liquidity providers earn fees and are exposed to trader profit and loss.

**Index-style Leveraged Tokens**
Rebalanced leveraged tokens give holders a fixed multiple of the daily move through periodic rebalancing, with no positions to manage.

**Papertrade: The Martingaler Queue**
Papertrade runs synthetic swaps against a liquidity pool that starts at zero. Winners that the pool cannot pay immediately enter a first-in-first-out payout queue, paid from later losses [1][2].

### 1.2 Limitations of Existing Mechanisms

Despite the progress of on-chain leveraged markets, several limitations persist:

**Need for a Counterparty:** Someone has to take the short side. Funding rates attract hedgers who already hold the underlying, but they bring complexity and a persistent cost to the trader.

**Insolvency Exposure:** A pool that lets winners claim more than it holds is insolvent. Systems that allow this need a queue, an insurance fund, or socialized losses to recover.

**Position Machinery:** Per-trader margin, maintenance levels and liquidations need keepers. They fail in fast markets.

**Oracle Gaps:** A mark that is advanced one oracle update at a time can be stepped past its own safety checks. If the gap between two updates is large, the step can overshoot.

---

## 2 Proposal

We introduce a mechanism that addresses the stated concerns by making leverage a function of the book. Leverage is not stored. It is computed from the book whenever it is needed.

### 2.1 High Level Design Choices

The proposed mechanism allows for flexibility with respect to market design, allowing market creators to specify:

- a maximum leverage `Lmax`
- an oracle and its freshness rules
- a dormancy floor `P_min`
- swap, mint and redeem fees

without having to deal with:

- funding rates
- per-trader positions, margin calls or liquidations
- a payout queue
- insurance funds or socialized losses

### 2.2 The Book

The market has two sides. Traders hold a fungible token, the synth. Liquidity providers deposit quote-currency backing and receive shares.

```text
Y   = backing        (quote currency held for the market)
X   = trader tokens  (synth held by traders, not by the pool)
P   = mark price of one synth
T   = X * P          (trader claim)
LPE = Y - T          (liquidity-provider equity)
s   = T / Y          (trader share of the book)
```

Trader tokens are claims. Tokens held by the pool itself are inventory and are not claims. By construction `T + LPE = Y`.

### 2.3 Leverage Budget

The leverage of each side is a share of one fixed budget `Lmax`:

```text
L_long  = Lmax * (1 - s)
L_short = Lmax * s
L_long + L_short = Lmax
```

At `s = 0` the traders are tiny relative to backing and get the full `Lmax`. At parity (`s = 0.5`) each side has `Lmax / 2`. As `s` approaches 1, long leverage approaches 0. Whichever side holds the larger share of the book has the lower leverage. A 99/1 book can never owe a gain it does not hold.

Example, with `Lmax = 5`:

| s (trader share of backing) | Long leverage | Short (LP) leverage |
|---|---|---|
| 0.00 | 5.00 | 0.00 |
| 0.10 | 4.50 | 0.50 |
| 0.25 | 3.75 | 1.25 |
| 0.50 | 2.50 | 2.50 |
| 0.75 | 1.25 | 3.75 |
| 0.90 | 0.50 | 4.50 |
| 1.00 | 0.00 | 5.00 |

### 2.4 The Mark Price

Between trades the book is fixed. Trader claim moves as `dT = L_long * T * d ln S`, where `S` is the oracle price. Integrating gives the closed form:

```text
T / (Y - T)  =  (T0 / LPE0) * (S / S0)^Lmax

P(S) = P0 * Y * R / (LPE0 + T0 * R),     R = (S / S0)^Lmax

with no traders:  P(S) = P0 * R
```

`(S0, P0, X0, Y0)` is an anchor. It is written only when the book changes (a buy, a sell, or a backing deposit or withdrawal) and optionally by a permissionless call.

The mark is a function of the live oracle and the anchor. It is evaluated in log space, so there is no precision cliff and no floor value that cannot be left. Properties:

- `T < Y` for any finite oracle price, so liquidity-provider equity does not reach zero from a rally.
- The mark is path independent in the oracle. Down 10% then up back to the start restores the mark while the book is unchanged.
- An unsynced gap in the oracle gives the same mark as the fine-grained path.
- If `Y = 0` the mark is frozen. If `X = 0` and `Y > 0`, leverage is `Lmax`.

### 2.5 Solvency Invariant

The solvency invariant (SI) is not a ratio to tune. It is structural:

```text
T < Y      for every finite oracle price
```

Traders can sell at any time and are paid `X_sold * P` from backing. No queue is needed because the claim is capped below backing by the leverage budget. A trader's upside is bounded, and the cost of that bound is that long leverage falls when longs are crowded.

### 2.6 Dormancy

If the underlying collapses, the mark falls geometrically (about `S^Lmax` for a small book) but never reaches zero. The market stops accepting new buys when the mark falls below a floor:

```text
P_min = max(1, startMark / 1_000_000)
```

Sells and liquidity withdrawals are never gated. When the oracle recovers, the market wakes on its own. If liquidity providers left while traders stayed, the book is thinner and wakes at a lower mark than the start. With no change to the book it returns to the start mark exactly.

Reaching `P_min` at 5× needs a drop of about 94% in the underlying. Reaching a 1 wei mark needs about 99.975%.

### 2.7 Tracked Backing

`Y` is tracked by the contract and not read from its token balance. A plain token transfer into the contract cannot be reverted. It is treated as excess, not backing, so it cannot move the mark or leverage. The owner can sweep the excess. `X` is read from the anchor, not from live balances, because balances move while a swap is in flight.

### 2.8 Liquidity-Provider Incentives

There are no token emissions and no funding rate. Liquidity providers earn swap, mint and redeem fees, and carry the opposite side of the trader's exposure. Their expected return is positive if traders lose more than they win, net of fees. Solvency does not depend on that. Only LP returns do.

---

## 3 Examples

To better demonstrate the mechanism, here are example simulations. All examples use `Lmax = 5`.

### 3.1 Rally on a Thin Book

LPs deposit 10,000. A trader buys 5,000 of the synth at a mark of 200, receiving 25 tokens. The underlying doubles.

```text
T / (Y - T) = (5,000 / 10,000) * 2^5 = 16
Y = 15,000   =>   T = 15,000 * 16 / 17 = 14,117.65
mark = 14,117.65 / 25 = 564.71
LP equity = 882.35
long leverage = 5 * (1 - 0.9412) = 0.29
```

A linear step would have paid the trader the whole 15,000 backing and left no equity. The budget model caps the claim below backing, keeps LP equity positive, and lowers the trader's leverage along the way. Buys, sells and withdrawals all stay open.

### 3.2 Halving and Doubling a Balanced Book

LPs and traders each hold half the backing (`T0 = LPE0`).

```text
Underlying halves:   T / (Y - T) = 0.5^5 = 0.03125   =>   T = 0.0303 Y
Underlying doubles:  T / (Y - T) = 2^5   = 32        =>   T = 0.9697 Y
```

The trader can lose almost all of their claim, but the pool never owes more than `Y`, and in the doubling case the trader's claim approaches `Y` without reaching it.

### 3.3 The Unsynced Crash

A market has a mark of 1.0 and about 15% of backing held by traders, so leverage is about 4.23×. The oracle falls 36% between two updates with nobody updating the mark.

```text
Linear step:  1 + 4.23 * (-0.36) = -0.52      (a negative price)
Closed form:  mark = 0.0615
```

A linear step projects a negative price. A floor then rounds it to 1 wei, and a $2 buy mints about 1.994 x 10^18 tokens. The closed form gives a mark of 0.0615, so the same $2 buy mints about 32 tokens.

### 3.4 Simulation

We simulated the closed-form mark over 2,000 independent paths of 1,000 steps each. Each step the oracle moves by a normal return with a 3% standard deviation, plus a 1% chance of an additional jump with a 20% standard deviation. On about half of the steps a trade occurs: a buy of up to 5% of backing (55%), or a sell of up to half of the trader tokens. Backing starts at 100,000.

| Result | Value |
|---|---|
| Steps with a trader claim above backing | 0 |
| Highest trader claim / backing at any step | 0.996 |
| Paths that touched the dormancy floor | 25 of 2,000 |
| Median lowest mark across paths | 0.016 (start 1.0) |
| Steps where a linear update would give a price at or below zero | 4,051 (1,308 paths) |
| Steps where a linear update would put the claim above backing | 172 (75 paths) |

The linear-update rows are a counterfactual on the same book and the same oracle gap. This is a Python model of the closed form, not the contract. The contract is covered separately by unit, fuzz and invariant tests (143 passing at the time of writing).

---

## 4 Dynamic Leverage vs a Payout Queue

A strong motivation for this mechanism is to remove the cost of insolvency from the trader.

A payout queue lets traders win more than the pool holds and delays payment until later losses refill it. This removes the need for seed capital. The cost is that a winner's payout depends on other users losing, and that winners tend to win at the same time in a market that moves in one direction.

Dynamic leverage makes the opposite choice. It keeps the pool solvent by lowering the leverage of the side that is crowding the book, so winners are paid in full and at once. The cost is that liquidity providers must deposit backing before the market opens, and that a trader's gain is bounded by the leverage budget.

We do not claim one is better for every market. A queue suits a market where seed capital is the binding constraint. Dynamic leverage suits a market where immediate redemption and a bounded counterparty are the product.

---

## 5 Limitations

- **Capital required.** The market needs backing before it opens. It does not bootstrap from zero.
- **Convexity.** `Lmax` is a cap, not a constant. The mark compounds continuously, so for a small book a 10% rise at 5× gives about 1.61×, not 1.5×.
- **Oracle dependence.** The mark is only as good as the oracle. Stale or paused oracles stop swaps and liquidity operations. Lenses fall back to the anchored mark.
- **Thin-book recovery.** If liquidity providers withdraw while traders stay, the market wakes below its start mark.
- **Counterparty returns.** Liquidity-provider profit depends on traders losing more than they win, net of fees.

---

## References

[1] M. Galer. *A Bootstrapping Mechanism for Sustainable Asynchronous Speculation Games of Chance.* https://docs.papertrade.xyz/martingaler-whitepaper.pdf

[2] Papertrade documentation. https://docs.papertrade.xyz/

[3] Folio. `docs/MATH_AUDIT.md` (M7) and `docs/NVDAx5_DEVELOPMENT.md` (§17): derivation and tests for the closed-form mark.

[4] R. Bloemen et al. `lnWad` / `expWad` as vendored from Solady (MIT) in `src/WadMath.sol`.
