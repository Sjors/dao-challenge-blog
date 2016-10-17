---
title: Visualizing Smart Contracts With Solgraph
date: 2016/10/13
---
This is the `SellOrder` contract, visualized using [Solgraph](https://github.com/raineorshine/solgraph):

{% img /images/SolGraph-SellOrder.png 300 %}

`null`, on the left, refers to the [fallback function](https://github.com/ethereum/wiki/wiki/Solidity-Features#fallback-functions). You can see that canceling and executing an order can cause the contract to terminate (`suicide`). <!-- more -->

A callback function is called when you send ether to a smart contract without specifying a function. In the case of `SellOrder`, the fallback function throws so that nobody can send ether to the contract this way.

For each user who interacts with the `DaoChallenge`, a `DaoAccount` contract is automatically created. This is where the user's funds and tokens are held. 

{% img /images/SolGraph-DaoAccount.png 600 %}

Worth noting are the blue functions, which are [constant functions](http://solidity.readthedocs.io/en/develop/frequently-asked-questions.html#what-is-the-difference-between-a-function-marked-constant-and-one-that-is-not). Those functions can't change state.

The arrow from `transfer` to `receiveTokens` indicates that the `transfer` contract on one user's `DaoAccount` can call the `receiveTokens` function on another user's `DaoAccount`

Finally, there's the `DaoChallenge` contract itself, which mainly just relays calls to functions with the same name on each user's `DaoAccount` contract.

{% img /images/SolGraph-DaoChallenge.png 600 %}

I added the commands that generate the above graphs on [GitHub](https://github.com/Sjors/dao-challenge/commit/70f19b675b5f04f6d9f567a8d26949706eac3d2d). Just run `make`.  

## Ideas for improvement

It would be great if I could visualize the relationship between these three contracts, and how this relates to trust. For instance, a `DaoChallenge` can trust the `DaoAccount` contracts it creates, though only if those contracts in turn can trust the contracts they interact with.

Notifications should have their own color.

I'd like to use horizontal space more efficiently, especially for larger contracts. Perhaps functions can be organized in a circle, with the contract name and fallback function in the middle.

More metadata in the graph, e.g. the argument and return types for a function and the modifiers they use. 

## When Is The Next Challenge?

I really enjoyed [DevCon 2](https://ethereumfoundation.org/devcon/) in Shanghai, but it did take a big bite out my schedule. So did my travels after that.

Another major distraction was the recent DDOS attack on the network. I normally use the [Mist](https://github.com/ethereum/mist/releases) Wallet, which uses [Geth](https://github.com/ethereum/go-ethereum/releases) to synchronize the blockchain. They released an update, 0.8.4, that got around the first round of attacks, but by the time I installed that, there was already a new round and it ground to a halt.

I switched to [Parity](https://github.com/ethcore/parity/releases), but that required me to re-download the entire 40(?) GB chain. That's not fun in China. By the time Parity almost finished, there was another DDOS that required an update to Parity. That update (or my own stupidity) broke the database that stored the chain. So I had to delete the database and re-download the block chain. I also couldn't get Homebrew to install the latest beta of Parity, because my hotel Wi-Fi kept randomly dropping SSL connections. When I finally got close to resyncing the full blockchain, a new Mist version came out. After a few hours in a Starbucks with good Wi-Fi, I was back in business!

I can't wait for ligh-weight wallets that don't need to download the full chain.

As a bonus, (Murphy's Law in action), the latest Mist wallet deleted the 10 challenge contracts from my wallet, so I had to import them again.

I expect to have another challenge up late next week or the week after. Things should be more regular after that.

I'm also considering a change in the way challenges work by intentionally putting a bug in a challenge. I'll let readers know if that's the case.

So far, I've never intentionally introduced bugs to my smart contracts. I assume I make plenty of mistakes naturally and that some of these mistakes can be exploited to rob the contracts, but that may not offer enough motivation for people to try to rob my contracts.
