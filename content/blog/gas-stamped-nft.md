+++
draft = false 
date = 2021-10-09T00:00:00Z
title = 'gas stamped nft'
+++

## the problem

a moderately hyped nft drop on eth l1 can see gas prices skyrocket to upto four
digit gwei levels. and what's the problem here? obviously, you're paying ~25-50x
of what you'd pay for a typical transaction. moreover, this gas premium creates
an inherent inequity favoring richer accounts, simply because they can afford to
pay for the gas.

secondly, there's another problem that isn't reflected in gas prices, but in the
buyers themselves. based on web3 familiarity, there are three kinds of nft
buyers — humans, humans who can use etherscan and bots. with humans (both who
can and cannot use etherscan), the drop turns into a fastest finger first
competition with all everyone eventually at the mercy of the network-latency
gods.

## so wtf is a gas-stamped mint

gas-stamped mints provide a fun way to mitigate these problems. they also create
a relatively fairer (level playing field) for all human actors that are part of
the mint. gas-stamping a transaction binds it to the gas price paid for the
transaction using the `tx.gasprice` global variable. post-eip1559, this gas
price is the (base fee + priority fee). the miner's tip can be accessed in the
contract via `tx.gasprice - block.basefee`. 

## what does a gas stamped mint look like

* ranged mint transactions
    * the contract can limit the range of gas prices between which nfts are
    minted to create a level playing field - e.g nfts can only be minted between
    50-150 gwei.
    * this can also be coupled with rarity features - rarer pieces can only be
    minted at a higher gas price.
    * arbitrary gas price mints can have special value - e.g. a 420 gwei mint
    transaction allows you to mint a *highly* sought*after nft

* gas caps
    * different gas level transactions can mint a different number of pieces
    * this could be used to create a mint process where the mints are skewed
    towards lower gas levels i.e most mints are at low levels while some are at
    higher levels

* creator-centric: this is good for creators too because now nft themselves can
be priced higher i.e. it'd be reasonable to pay 0.05e (instead of 0.01e) for the
same mint when you know that you're never gonna pay outrageous gas fees on top.

## downsides

* mints could stretch for hours depending on network usage, this might not be
ideal

* long mints could create an artificial asymmetery on secondary market
platforms, i.e. those who mint earlier stand to gain more

* mints still need bot protection and the whole process is vulnerable to otc
bribes accepted by miners

## implementation

you can check out a sample implementation on
[github](https://github.com/ruvaag/gas-stamped-nft).
