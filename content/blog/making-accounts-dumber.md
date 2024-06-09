---
draft: false
date: 2023-10-01T00:00:00Z
title: 'making accounts dumber'
---

## state of the execs

an account is a user-deployed beacon proxy to hold assets in a restricted
environment to enable leveraged lending while mitigating credit risk. they were
designed to be a dumb container for the convenience of users, developers and the
broader protocol. evidently, this isn't how things turned out. there's a lot of
things went wrong which i'll probably list in a future note. this note
particularly focuses on a few overheads arising from `AccountManager.exec()` and
`Account.exec()` at the contract and interface layers.

we've understood the role of `exec` as a function that enables accounts to
interact with external contracts in a *walled garden* environment. it's what
allows users to invest into farms and other assets on leverage using their
sentiment account. these `exec` calls require a user to supply detailed
parameters about the target address, ether paid, and most importantly the
calldata for the interaction. the calldata includes specific details about the
method called on the target address and exact parameters for this method to be
called. if this sounds inefficient at first glance, that's because it is.
moreover, it's not capital efficient and misaligns incentives for the protocol
participants.

let's revisit the purpose of the `exec` function -- what's the end objective
here? if you still think it's about *restricted actions in a walled garden*,
pause for a second and think again. it's not about restricting interactions ---
it's about creating a safe conduit for flow of assets to and from accounts.
restricted actions is one possible solution to creating that conduit which as i
highlight below, creates more problems than it solves. going back to our
original idea, accounts should be dumb containers of assets. why is our *dumb
container* worried about how an asset gets in the account? why is it trying to
make sure the right contracts are being called? extending this argument, the
only reason we need controllers is to make sure `exec` function isn't misused.
the account only cares about *what* is coming in and going out, having an `exec`
call adds the overhead of having to care *how* assets got there in the first
place. below, we identify three key overheads introduced by the current design
that need to be addressed.

### calldata construction

one would guess that the most resource-intensive component of our interface or
backend has something to do with user experience or analytics, or something
particularly helpful but you'd be mistaken. the bulk of our interaction code is
just trying construct the arguments for the `exec` call. the interface needs to
know exactly which contract to call, what the structure of the call looks like
i.e. not the exec method but the method that is called by exec, and the value of
each param in that call. this is also stored somewhere and fetched at runtime
which adds a latency overhead. lastly, all of these components have maintainence
costs to account for migration, deprecation, and all that routine stuff.

to be fair, this issue is not limited to interfaces but also to builders trying
to build on sentiment. we assume that someone operating their account
systematically or building on top of sentiment has it easier because they're
writing solidity but it really asks them to solve the same issues in a different
language. this is doubly problematic for someone trying to control their account
using an eoa controlled by a bot.

so calldata construction is not only making it difficult for us to maintain a
simple interface for the app, but also for others to build on top of sentiment.
the *dumb* account only needs to know about the inflow-outflow of assets.
consider a user who has usdc and wants to farm using convex on leverage -- when
they deposit usdc and "invest" in a convex lp, the account doesn't need to know
where the lp tokens come from. this is an artificial constraint imposed only for
the `exec` function.

### limited composability

the inherent nature of `exec` both enables and limits the composability of the
protocol. as discussed earlier, `exec` makes an actual contract call based on
the calldata specified. this implies that without the presence of arbitrary
multicall functionality (which in itself is an overhead) it can only perform one
"function call" at a time. from a user's perspective this means that they cannot
lp and stake in one transaction, and that's just the beginning of their
problems. a bigger problem is that even with multicall they'd have to own the
right tokens in an optimal proportion to lp and stake before initiating the
transactions. this limits composability in that it does not allow operations
such as "i want $10k worth staked convex tricrypto tokens against $10k usdc"
which is key not only for composability but also for a better ux. the "atomic
swap" from usdc to staked convex tricrypto tokens saves the users at least four
transactions. in many ways then, the `exec` limits composability in many ways in
trying to construct the *safe conduit* for allowed assets to move in and out of
the system.

### path dependency

this is an issue we first encountered with the swaps. we need an aggregator, but
really we need an aggregator of aggregators. if we zoom out this is an extension
of the limited composability issue outlined above. even if there was an easy way
to get an aggregator of aggregators (hello dev, maintainence and cost overheads,
my old friend) there's the terminal task of selecting the best path for any
interaction. this problem intensifies when you think of the usdc to staked
convex tokens "swap". what is the best path at any given time for such an
operation? the current `exec` design forces path-dependency onto every user
operation, even when it's unnecessary. consider two scenarios here: if
path-dependency is an issue (as with swaps) the user is better off letting
external market participants (an aggregator) choose the best path for them. in
the cases where the path itself isn't an issue (as with weth9) the problem drops
a dimension to turn into an efficiency problem i.e. who can do it for the
cheapest. if a user plans to execute multiple interactions over time and hence
the protocol attempts to deal with a large number of hetereogenous composable
operation "request" (i.e. `exec` calls) over time, path dependency is a net
negative to the entire system due to these reasons.

## revisiting exec flows

we discussed multiple issues with the account `exec` flows previously, but
restricted our focus to its problems without examining any solutions. let's fix
that -- after all, we can do much better than those that poke holes just for the
sake of it.

### characterizing exec calls

```solidity
function exec(address target, uint amt, bytes calldata data) {
    (bool success, bytes memory retdata) = target.call{value: amt}(data);
    return (success, retdata);
}
```

the `Account.exec()` function has a deceptively simple and dare i say, elegant
implementation. it is a general purpose function that can *execute* any
arbitrary command passed as `data` on any `target` contract. the key issue with
characterizing the nature of these calls arises from how we see this function.
`exec` tries to solve a key issue faced by every restricted environment -- how
do participants interact with the external environment? in our case, this means
interacting with other contracts across the chain. the `exec` train of thought
is straightforward -- users need to interact with multiple contracts and there
is no way to know the interface or nature of methods to these integrations, let
alone future ones that aren't even deployed yet so we need a cast the widest net
possible with a general purpose function that could execute anything. walk down
this path a little further and you conceive the `Account.exec()` design and
shortly after we see the need for controllers to bring it all together. neat
right? no.

the approach outlined above puts interaction at the center of the problem, which
is why the solution is also tailored around it. but taking a step back, we now
know that the true problem is not interaction but rather restricted flow of
assets. as outlined above, the key aspect is *what* comes in an account not
*how* it gets there. following this we see a simpler way to characterize all
`exec` functions -- swaps.

this is the simple *big idea* needed to reimagine our exec flows. even the most
complex `exec` calls can be modelled as a swap. every "yield" operation is just
an erc20 <> erc20 swap where liquidity sources are not always dexs. once we
internalize the idea that all `exec` flows can be conceptualised as swaps, it
becomes apparent that the current `exec` design is sub-optimal because it tries
too hard to be an *everything-executor* when the same functionality could be
achieved in a simpler way through swaps. consider the following `exec`
operations -

* wrap eth:
    * interaction: call the `wrap` function on weth9 to transform native eth to
    weth
    * equivalent swap: eth -> weth

* stake 3crv in convex:
    * interaction: deposit and stake using the convex booster and reward pool
    contracts 
    * equivalent swap: 3crv -> staked 3crv convex token

* buy glp using usdc:
    * interaction: some complex contract calls across fee glp, staked glp, fake
    glp, real glp, and finally glp 
    * equivalent swap: usdc -> glp

* lp and stake usdc into a wsteth/usdc pool:
    * interaction: multicall: (1) sell 50% usdc to eth (2) wrap eth to wsteth
    (3) lp into wsteth/usdc balancer pool (4) stake bpts to gauge 
    * equivalent swap: usdc -> staked wsteth/usdc lp token

note how the final column significantly reduces complexity from the protocol's
perspective since it only needs to focus on the asset movements and not on the
process itself. also note that swaps help turn multi-step operations into a
single operation. so the only question that remains, if we can't use existing
dexs to execute these swaps how do we implement such a system and what are it's
tradeoffs.

### swaps all the way down

without getting too technical about it just yet, the implementation follows a
principle we're far too familiar with -- creating economic opportunity for
protocol participants as a means to capital efficiency, and in this case these
participants are searchers.

this is what the process for a reimagined `exec` looks like --- also, it's not
called `exec` anymore. a user places an onchain "order" for the terminal erc20
of the operation using their account assets. searchers are incentivized to fill
these *orders*. they can do so via flash loans or using user's account assets.
like all onchain swaps, these *orders* have an expiration and slippage tolerance
which incentivizes searchers to find the best possible path and provide the user
with the best execution. these *orders* differ from dex operations in that they
are not executed instantaneously, but rather sit in a "order book" until they
are filled or expire. this is closer to a seaport order than a lob bid/ask,
since these *orders* don't aim to aid price discovery so there is no incentive
for users to create orders that are never filled in the near future.

that being said, on the other hand, with the right parameters they could be used
as "leveraged limit orders" to create extremely unique user experiences such as
the following --- "if eth falls below $1000 go 10x long eth against usdc, wrap
the eth into wseth/reth/frxeth, lp into the balancer boosted pool and stake the
bpt for rewards". imagine. the. possibilities.

on the protocol end, the system only needs to verify that the right assets delta
and their quantities to ensure that slippage tolerances are being met.
