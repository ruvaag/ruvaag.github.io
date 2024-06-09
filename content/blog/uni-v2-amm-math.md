---
draft: false
date: 2021-01-01T00:00:00Z
title: 'uniswap v2 amm math'
---

this article is a gentle mathematical introduction to uniswap's \(x * y = k\)
market maker. before we get into the working of the model, let's look into why
it was needed in the first place.

## the problem

tradfi exchanges use an order-book mechanism to settle trades. this system
relies on institutional market makers to provide liquidity to the system where
they make money on the bid-ask spread [[1]](https://youtu.be/-zTHKcJEGe8).
unfortunately, the order book mechanism doesn't translate well to a
decentralised system such as ethereum due to several reasons, most importantly – 

* **latency**: ethereum can support ~15 transactions per second with a block
time of 10-19 seconds which is too slow from a performance and scalability
perspective.

* **gas costs**: every operation on the ethereum blockchain requires gas
[[2]](https://ethereum.org/en/developers/docs/gas/) and a traditional market
making mechanism involves high-volume of cancellation and re-submitted orders to
maintain liquidity and a fair price for the asset. to replicate this on ethereum
would mean spending a significant chunk of money on gas, which would increase
trading costs for users and transaction costs for market makers to provide
liquidity and avoid wide spreads.

## the solution

the solution to this problem came in the form of liquidity pools
[[3]](https://youtu.be/cizLhxSKrAc) which were widely popularised by uniswap. a
liquidity pool is essentially a collection of locked funds used to facilitate
certain operations using smart contracts— in this case trading. with liquidity
pools, market makers are replaced by liquidity providers (lps) and anyone can
become an lp by adding funds to a pool. uniswap provides lps with tokens
proportional to their share in the pool which can be burnt later to redeem their
liquidity. lps are incentivised to add funds by earning trading fees for
providing liquidity.

these pools replace conventional order-books in dexs (decentralised exchanges)
by forming a collective source of liquidity. uniswap v2 pools contain equal
values of both tokens and lps are incentivised to add tokens of equal values
since they risk losing capital to arbitrageurs if this is not the case. from a
trading perspective, uniswap prices these assets in the liquidity pool using
automated market makers (amms)
[[4]](https://academy.binance.com/en/articles/what-is-an-automated-market-maker-amm)
which can be described as protocols that price assets algorithmically without
the need for any counterparty. these amm algorithms can be complex and are
implementation-specific. uniswap v2 uses a constant product market maker model,
with the invariant \(x * y = k\) where \(x\) and \(y\) are the quantities of two
tokens in the liquidity pools and \(k\) is the invariant product that must
remain constant.

to understand how this model prices an asset and facilitates a trade, let's run
through a trade on uniswap.

### swapping tokens with uniswap

consider a liquidity pool of token pair \(X\) and \(Y\) holding reserves \(x\)
and \(y\) respectively at the moment. the invariant for this pool is the product
of quantities of these tokens  \(x * y\) which should remain constant over time.
simply put, you can exchange \(\triangle x\) tokens of \(x\) using this pool for
\(\triangle y\) tokens of \(Y\) such that \((x + \triangle x) (y - \triangle y)
= x * y\) so that the invariant product of the quantities of two tokens in the
pool remains constant. solving the previous equation for \(\triangle y\) it can
be seen that we get \(\frac{\triangle x} {x + \triangle x}\;y\) tokens of \(Y\)
in exchange for \(\triangle x\) tokens to ensure that the invariant holds true.

this hyperbolic graph for the constant product equation \(x * y = k\) also
defines the pricing model thus implemented by the market maker. in this case,
the marginal price is the derivative of the curve \(x * y = k \rightarrow y/x\).
this mechanism is eponymously called the *constant product market maker* and is
a major feature of uniswap v1 and v2.

while the above equations represent the ideal case, uniswap charges users a 0.3%
fee per trade which still needs to be factored into these calculations. this fee
is added to the liquidity pool reserves and forms the incentives for lps to
provide liquidity. let's see how this fits into the original equation. once
again, assume that you wish to swap \(\triangle x\) tokens of \(X\) for
\(\triangle y\) tokens of \(Y\) to a uniswap pool which currently holds \(x\)
tokens of \(X\) and \(y\) tokens of \(Y\).

when you trigger this swap, uniswap first collects its trading fee – assume that
the fee rate (as part of total exchanged value) is \(\rho\;, 0 \leq \rho \lt 1\)
and \(\gamma = 1 - \rho\).  accordingly, after this fee collection, you're left
with \(\gamma \triangle x\) tokens of \(X\) which are to be swapped. the
remaining \(\rho \triangle x\) are out of the equation for now, but we'll get to
them later. we now go back to our original product invariant, but the equations
are slightly modified, we add  \(\gamma \triangle x\) of tokens instead of the
original \(\triangle x\) tokens. putting this in the original equation solving
for \(\triangle y\) —

$$
\begin{alignat}{3}
(x + \gamma \triangle x) (y - \triangle y) = x * y \\
(1 + \gamma \alpha) (y - \triangle y) = y, \textrm{where } \alpha = \frac{\triangle x}{x}  \\
\triangle y = \frac{\gamma \alpha}{1 + \gamma \alpha} y = \frac{\gamma \triangle
x}{x + \gamma \triangle x} y
\end{alignat}
$$

so in this case we get a lesser bang for our buck, owing to the fees lost
earlier. the fee component \(\rho \triangle x\) is now added back to the pool
and the total reserves now stand at \((x + \triangle x)\) tokens of \(X\) and
\((y - \triangle y)\) tokens of \(Y\), with the updated value of \(\triangle y\)
as computed above. this implies that product is not constant but actually
increases gradually over time with every trade. the difference here forms the
incentives given to lps which can be claimed when they burn their liquidity
tokens.

for a detailed formalisation of this model and it's implementation you can refer
to the uniswap whitepaper [[5]](https://uniswap.org/whitepaper.pdf) and the
formal model specification
[[6]](https://github.com/runtimeverification/verified-smart-contracts/blob/uniswap/uniswap/x-y-k.pdf).
