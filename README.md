# Brokex — Protocol Explanation

## 1. What is Brokex?

Brokex is an on-chain CFD broker based on a Book B model.

This means that traders do not trade against each other, and there is no external liquidity pool like an AMM or an order book. Instead, traders trade directly against the protocol’s vault.

The model is simple:

* when traders win, the vault pays them
* when traders lose, the vault collects their losses

This makes the vault the counterparty of all trades.

---

## 2. Why this model?

Traditional DeFi trading systems rely on either AMMs or order books.

AMMs generate prices using mathematical formulas. The issue is that these prices often deviate from the real market price. This creates artificial slippage, distorted charts, and unstable trading conditions. For a product that aims to replicate real market trading, this is not ideal.

Order books, on the other hand, require large amounts of liquidity to function properly. Without enough participants, they become empty or filled with fake liquidity from bots. This makes execution unreliable and adds unnecessary complexity.

Brokex avoids both problems by using oracle prices directly and by acting as the counterparty through a vault.

The goal is to simplify everything:

* execution at real market price (oracle-based)
* no need for deep liquidity
* predictable behavior
* no artificial price mechanisms

---

## 3. How trading works

Traders do not deposit funds into an account before trading.

Instead:

* when they open a trade or place an order, funds are taken directly from their wallet
* when the trade is closed or an order is cancelled, funds are sent back directly

When a trader opens a position:

* they provide a margin
* they choose a leverage
* the protocol computes a notional value (margin × leverage)
* a fee is charged once at opening (based on notional)
* the fee is sent to the vault
* the remaining margin is stored in the Core

All calculations in the protocol are based on the notional value.

When a trader closes a position:

* if they win, the vault pays the profit
* if they lose, the loss is taken from their margin and sent to the vault

There is no restriction when closing a position. A trader can always exit.

---

## 4. Open interest and exposure

The protocol does not think in terms of individual trades only. It tracks global exposure.

For each asset, it keeps track of:

* total long open interest
* total short open interest
* average entry price for longs
* average entry price for shorts

Open interest is simply the notional value of positions.

The goal of the system is to keep:

```text
long exposure ≈ short exposure
```

Because when both sides are balanced, risk is low. The gains of one side are naturally offset by the losses of the other.

---

## 5. The real problem: imbalance

If too many traders are on one side (for example, too many longs), the protocol becomes exposed.

If the market moves in that direction, the vault must pay many winners at once.

This is the main risk of a Book B model.

So the protocol must control this.

---

## 6. Buffer: allowing the system to start

At the beginning, there is no liquidity and no balance between long and short positions.

If the protocol was strict from the start, it would reject most trades and never grow.

So Brokex introduces a buffer.

Below a certain level of total open interest, the protocol ignores imbalance and accepts trades freely.

This allows the system to bootstrap and grow naturally.

---

## 7. Balance control with alpha

Once the system grows, imbalance must be controlled.

Before accepting any trade, the protocol simulates what the state would look like after the trade:

* new long open interest
* new short open interest
* new imbalance

It then applies a rule using a parameter called alpha.

Alpha is a coefficient between a minimum and a maximum value.

It depends on how imbalanced the system is.

* when the system is balanced → alpha is low → less capital is needed
* when the system is imbalanced → alpha is high → more capital is required

This allows the protocol to dynamically adjust how strict it is.

If a trade makes the imbalance too high or requires too much capital, it is rejected.

If it improves balance or stays within limits, it is accepted.
Voici une version propre en **Markdown + code style**, directement copiable dans ta doc :

---

```md
Imbalance

The imbalance between long and short open interest is defined as:

imbalance = abs(longOI - shortOI) / (longOI + shortOI)

- imbalance = 0 → perfectly balanced
- imbalance = 1 → fully one-sided
```

---

```md
Alpha (Capital Efficiency Coefficient)

Alpha controls how much capital must be locked depending on imbalance.

alpha = alphaMin + (alphaMax - alphaMin) * (imbalance ^ K)

Where:

- alphaMin: minimum capital requirement when market is balanced
- alphaMax: maximum capital requirement when market is fully imbalanced
- K: curve parameter controlling how fast alpha increases
```

---

```md
Required Capital (needLock)

The required capital to lock in the vault is:

needLock = max(longSideRisk, shortSideRisk) * alpha

Where:

- longSideRisk: total max profit exposure of all long positions
- shortSideRisk: total max profit exposure of all short positions
```

---

```md
Summary

- When imbalance is low → alpha is close to alphaMin → less capital is locked
- When imbalance is high → alpha is close to alphaMax → more capital is locked

This ensures capital efficiency when the system is balanced and protection when it is not.
```

---

## 8. Capital locking

Every trade has a maximum possible profit.

For example, if a position has a notional of 10,000 and the max profit is 10%, the protocol must be able to pay up to 1,000.

The protocol computes the total maximum payout for longs and for shorts.

Instead of locking the sum of both sides, it only considers the larger side, because both sides cannot fully win at the same time.

Then it applies the alpha factor to determine how much capital must actually be locked.

This value is called the required capital (or needLock).

When a trade is opened:

* the required capital is recalculated
* the vault locks additional capital if needed

When a trade is closed:

* the required capital is recalculated
* the vault releases unused capital

Closing a trade can never increase required capital.

---

## 9. Spread and funding

The protocol uses two mechanisms to manage imbalance:

### Spread

The spread depends on the imbalance.

* if a trader opens on the dominant side, they get worse pricing
* if they open on the weaker side, they get better pricing

This naturally encourages traders to rebalance the system.

---

### Funding

Funding is always paid to the protocol.

It is not a peer-to-peer funding like traditional perps.

The more a trader contributes to imbalance, the higher the funding they pay.

Funding is calculated over time based on notional and a dynamic rate.

This creates an economic incentive to move toward balance.

---

## 10. Vault and LP system

The vault contains all liquidity used to pay traders.

Anyone can deposit into the vault and become a liquidity provider (LP).

When someone deposits:

* the protocol calculates the current vault value
* it divides this value by the total number of LP shares
* this gives the LP token price
* new shares are minted at that price

The vault value includes:

* the actual stablecoin balance
* minus the unrealized PnL of traders

This ensures that LP shares always reflect real exposure.

---

## 11. Withdrawals

Withdrawals are not instant.

When an LP wants to withdraw:

* they submit a withdrawal request
* their shares are added to a queue

The protocol reserves a portion of the vault for withdrawals.

As liquidity becomes available, a keeper processes the queue and sends funds back to LPs.

Withdrawals can be partial if there is not enough free liquidity at once.

This ensures that open trades are always properly backed.

---

## 12. Emergency mode

In extreme situations, the protocol can enter emergency mode.

In this mode:

* trading is stopped
* new orders are disabled
* traders can recover their margin
* LP accounting is simplified

If a trader had an open position, they receive their margin minus the opening fee.

If they had a pending order, they receive the full margin.

No PnL is calculated in emergency mode.

This protects users and the system.

---

## 13. Final idea

Everything in Brokex revolves around one core principle:

```text
Keep the system balanced to minimize risk.
```

The protocol accepts trades, but only if the resulting exposure can be safely handled by the vault.

It uses:

* open interest tracking
* imbalance measurement
* alpha-based capital control
* spread and funding incentives

to keep the system stable while remaining simple and usable.


