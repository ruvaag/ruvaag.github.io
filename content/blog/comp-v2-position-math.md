---
draft: false
date: 2022-01-01T00:00:00Z
title: 'compound v2 position math'
---

## overview

compound is one of the largest onchain money markets with over $5 billion in tvl
as of today. this post dives into the mechanics of how compound keeps track of
onchain deposit and debt balances as they dynamically accrue interest over time.
the first section looks at how the protocol tracks lenders' deposits using
ctokens and how those ctokens accrue interest in real-time. the second section
explores how compound uses a `borrowIndex` to atomically update debt balances
for different positions with varying size, tenure, and interest rates across the
system.  

## lenders

when a lender supplies assets to a compound market, they receive an equivalent
value of *ctokens* in return. the number of ctokens received depends on the
ctoken exchange rate which can be queried using the `exchangeRateCurrent()`
function and is cached in the `exchangeRate` state variable. the ctokens
themselves represent the deposit of the lender and in this sense, the accounting
is very straightforward -- from the protocol's perspective, a lender's deposit
in any compound market is equal to the value of ctokens they possess at a given
point in time.

another aspect to look into is how these lenders' deposits (stored as ctokens)
earn interest over time. this is achieved via a dynamic ctoken exchange rate
implementation. the ctoken exchange rate represents the price of one ctoken in
terms of its underlying asset and is initialized with a value of \(\small 0.02\)
when a new market is created. theoretically, the exchange rate varies as a
non-decreasing function and keeps increasing as the borrowers' debt accrues
interest over time. this increasing rate implies that the lender can redeem the
same number of ctokens received earlier for an increased number of underlying
assets at a later point in time. this exchange rate differential across time
represents the interest earned by lenders on their deposited assets.

#### ctoken exchange rate

the ctoken exchange rate at any moment can be computed as --- $$\small
\text{rate} = \frac{\text{liquidity} + \text{borrows} -
\text{reserves}}{\text{supply}}$$

where,

* liquidity - aggregate amount of idle deposits that can be borrowed,
`getCash()`

* borrows - aggregate debt owed by borrowers to the protocol, `totalBorrows`

* reserves - aggregate reserves accumulated via fees, `totalReserves`

* supply - aggregate amount of ctokens in existence, `totalsupply`

to intuitively understand why this function is nondecreasing let's look at some
common operations and how they affect the exchange rate ---

* deposit - \(\uparrow\) liquidity and \(\uparrow\) supply

* redeem  - \(\downarrow\) liquidity and \(\downarrow\) supply

* borrow  - \(\downarrow\) liquidity and \(\uparrow\) borrows

* repay   - \(\uparrow\) liquidity, \(\downarrow\) borrows, and \(\uparrow\)
reserves 

* liquidation - \(\uparrow\) liquidity, \(\downarrow\) borrows, and \(\uparrow\)
reserves

## borrowers

the ctokens play a dual role as they not only represent the lenders' deposits
but also their collateral for any debt issued. after depositing assets into
compound (and receiving ctokens) borrowers can now use these ctokens as
collateral to open over-collateralized debt positions. the protocol accounting
for debt balances needs to --- \(\small\text{(a)}\) atomically accrue interest
over all debt positions \(\small\text{(b)}\) atomically update position risk
across all positions to facilitate timely liquidations and \(\small\text{(c)}\)
minimize gas costs incurred by protocol users. this is a non-trivial task
considering the onchain storage premium, increasing gas prices and the
permissionless nature of defi protocols vulnerable to certain exploits.
accordingly, compound implements a neat solution to this problem that we look
into below.

compound accrues interest on a per-block basis and the instantaneous per-block
borrow rate can be queried using the `borrowRatePerBlock()` function. this
algorithmically-determined rate currently depends on pool utilization which
reflects the demand for the underlying asset in the market.

the `accrueinterest()` function in the contracts can be called to accrue
interest and atomically update the state for all debt positions in the system.
since ethereum doesn't have something similar to cron jobs, running a bot that
periodically calls the `accrueinterest()` function would be an anti-pattern.
instead, the protocol is designed to call this function before executing any
user operation - deposit, redeem, borrow, repay, or liquidation. this
essentially amortizes the cost of keeping the state updated across all users
proportional to how often they interact with the protocol, which arguably is a
very reasonable approach.

#### interest accrual

the `accrueinterest()` function mentioned above is responsible for updating debt
balances across the ctoken contract. it is worth looking into what happens when
this function is called. the protocol updates the `totalBorrows`,
`totalReserves` and `borrowIndex` state variables based on interest accrued
since the last time this function was called. 

the `borrowIndex` is a index value initialized at \(\small 1\) when a new
compound market is created and tracks the cumulative interest accrued over time.
updating the `borrowindex` is used as a proxy to atomically update all debt
positions in the protocol and we explore how this actually works in the next
subsection.

for now, let's consider a situation where `accrueinterest()` is called at time
\(t\) and \(\delta t\) intervals have passed since the last time it was called.
the changes in the ctoken contract state after this call can be summarized as:

$$
\begin{alignat}{3}
\small i_t = b_{t- \delta t} \; (1 + r_t \cdot \delta t) \\
\small b_t = b_{t- \delta t} \; (1 + r_t \cdot \delta t) \\
\small r_t = r_{t- \delta t} +  \overline{r} \cdot b_{t - \delta t} \cdot (r_t
\cdot \delta t) \\
\end{alignat}
$$

where,

* \(i_t\) - borrow index at time \(t\), `borrowIndex`

* \(b_t\) - aggregate borrows at time \(t\), `totalBorrows`

* \(r_t\) - aggregate reserves at time \(t\), `totalReserves`

* \(\overline{r}\) - reserve factor (\(\small 0 \leq \overline{r} \leq 1\))
representing the fraction of accrued interest collected by the protocol as fees,
`reserveFactorFromMantissa`

* \(r_t\) - per block borrow rate at time \(t\), `borrowRatePerBlock()`

#### borrow storage internals

compound stores the debt balance for every account as a `borrowSnapshot` object
composed of two values -- `principal` and `interestIndex`. the `principal`
stores the raw amount of debt owed by the account denominated in terms of the
underlying asset. the `interestindex` stores the `borrowIndex` of the ctoken
when the `principal` was last updated. 

consider the `borrowSnapshot` for an account \(\small a\) with `principal`
\(\small p_a\), `interestIndex` \(\small i_a\) while the current ctoken
`borrowIndex` is \(\small i\). we can use this data to deduce the following --- 

* the current debt owed by account \(\small a\) is \(\small p_a \cdot (i /
i_a)\)

* the interest accrued on this position since the last time it was updated is
\(\small p_a \cdot (1 - i / i_a)\)

* if \(x\) underlying tokens are now borrowed, the `principal` will be updated
to \(\small [p_a \cdot (i / i_a)] + x\) and the `interestIndex` will store
\(\small i\)

* if \(x\) underlying tokens are now repayed, the `principal` will be updated to
\(\small [p_a \cdot (i / i_a)] - x\) and the `interestIndex` will store \(\small
i\)
