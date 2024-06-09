---
draft: false
date: 2023-06-01T00:00:00Z
title: 'begone controllers'
---

## what's wrong

controllers are great as context-aware adapters that help validate every account
transaction, a novel innovation --- until they're not. every integration needs
its own controller, and these need to be audited for security reasons, insurance
and other lies that help us sleep at night. we've already spent >$50k to get our
current controllers audited. some controllers can be tedious to implement and
test, like the uniswap controller. on the other hand controllers like glp
require developers to get deep into the integration architecture, a
time-consuming process. additionally, controllers also have hidden costs
attached to their maintenance and security. the more code you deploy, the wider
your attack surface. we already have 17 controllers and this number won't go
down anytime soon. what does the system look like with 25 controllers? 100?
something's gotta give.

## role of controllers

so if controllers are so bad, why did we need them in the first place? great
question. controllers serve two main purposes ---

1. they relay information about the inbound/outbound tokens (asset delta) to
   help maintain account tallies for risk purposes.

2. they validate every account interaction using the target address and function
   sighash to ensure only permissioned interactions take place.

if only there was a way to get this information from elsewhere, we could do away
with controllers altogether. as the title suggests, let's talk about how we can
do just that.

## validate assets delta

today, controllers return `tokensIn` and `tokensOut` arrays that supply the
system with information about the change in account assets after an interaction.
what if we didn't store this information in the controller, and it was simply
passed as a parameter in the function call by the user. the key to understanding
why this would work is to establish that a user never has an incentive to send
the wrong parameters to this modified function call. consider these scenarios:

* if a user sends too few `tokensIn`, they lose value by ignoring assets that
are coming into the account. this is akin to burning funds and can cause faster
liquidations and transaction reverts due to failed post-transaction health
checks.

* if a user sends too many `tokensIn`, they add spam to their account that
doesn't add any value. the current system uses runtime `balanceOf()` checks to
tally balances, and adding a zero token to the `assets` only takes up space
without adding value. a user also risks hitting the `MAX_ASSET_CAP` and
increased gas costs on every interaction due to extra oracle calls, leading to
overall degraded user experience.

* if a user sends too many `tokensOut`, it doesn't have any real effect on the
execution path because removing an asset from the 'assets' array today requires
that it's balance be zero. so even if there are extra 'tokensout', there is no
change unless those assets are actually moving out of the account. once again,
the user has no incentive to do  this.

* if a user sends too few `tokensOut`, it might actually mess up the accounting
for assets. there is a chance the account might be left with elements in the
`assets` array that aren't really there. but as it happens, this does not break
anything. if a user maliciously tries to send too few `tokensOut` the final
health check would revert the transaction anyway since the risk engine makes
`balanceOf()` calls to every element of `assets`

* in all of the above scenarios, sending too few or too many can be replaced by
sending a combination of incorrect tokens and the conditions would still hold.

the scenarios above exhaustively cover all cases and establishes why these
values can simply be accepted as calldata arguments from the user instead.

## validate target & sighash

the other purpose of controllers is to validate the target address and function
being called by the account for every interaction. to see why we don't need a
controller to validate this, we get deeper into why we need these checks in the
first place. these checks are pre-conditions for the `Account.exec()` method,
which executes interactions in the account itself. if they were not in place, a
malicious actor could call arbitrary functions using their account, leading to
loss of funds. but do we really need a controller for these? not really, and
definitely not if we're not using controllers to validate assets delta as
described above.

the system could rely on a `mapping(address => mapping(bytes4 => bool))` to
store this data, which would replace the need for controllers. storing this in
the right contract could also help save gas by reducing external call. moreover,
this also opens up an avenue for *privileged accounts* that have special
permissions. this would allow specific borrowers to interact with integrations
that other users cannot, which can be a great boost for certain cases and
strategic partnerships.

## conclusion

we scrutinized controllers and their key functions. then we delved into user
incentives, alternative ways to fulfill those functions and established that the
system can provide better performance without controllers as well. this would
help us add new integrations rapidly, open up avenues to new strategic
partnerships, reduce audit spends, maintenance costs and allow us to increase
focus on the core components of the system.
