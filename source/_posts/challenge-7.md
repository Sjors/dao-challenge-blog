---
title: Challenge 7 - Token Issuing
date: 2016/8/19
---

In [earlier challenges](https://dao-challenge.herokuapp.com/2016/08/08/recap-challenge-1-5/), users could buy tokens throughout the lifetime of the smart contract. The token price was baked into the code, so it couldn't be changed to reflect the market value of those tokens. As is the case with shares, the price of a token should vary based on the perceived value of the smart contract.

Today I'm allowing the smart contract owner to issue a fixed number of tokens and determine the price and deadline.
<!-- more -->

First, let me explain where things are going over the next few weeks. I plan to remove the `transfer()` and `refund()` functions, and instead I'll create `BuyOrder` and `SellOrder` contracts, as well a `dividend()` function. Users won't be able to redeem tokens directly for ether; instead, they can trade tokens with one another. To reward users for holding on to their tokens, the owner of the smart contract can pay dividends to all participants.

Initially only the owner of the smart contract can issue tokens or pay dividends. This is easy to implement with the `onlyChallengeOwner` function modifier. Later on I want to allow a majority vote or something more elegant. This is more complicated to implement and requires more than just writing secure code. In particular I would need to consider how each mechanism could create bad incentives. The makers of The DAO put a lot of thought into how voting should work and yet there was serious [critique](http://hackingdistributed.com/2016/05/27/dao-call-for-moratorium/). I'm also fascinated by the idea of using predication markets to make decisions, like in a [futarchy](https://blog.ethereum.org/2014/08/21/introduction-futarchy/). At the same time I want to keep things as simple as possible.

So in a future version of this smart contract, tokens are issued in batches, used to distribute dividends, and can be traded between users. I think this is a simpler system than the [split proposals](https://daowiki.atlassian.net/wiki/display/DAO/How+to+split+the+DAO%3A+Step-by-Step) used by The Dao.

But first things first: today I'm implementing `issueTokens()`, which lets the contract owner issue `n` tokens at price `p` (in [szabo](http://ether.fund/tool/converter)), to be sold before `deadline` (Unix timestamp):

	function issueTokens (uint256 n, uint256 price, uint deadline) noEther onlyChallengeOwner {
		// Only allow one issuing at a time:
		if (now < tokenIssueDeadline) throw;

		// Deadline can't be in the past:
		if (deadline < now) throw;

		// Issue at least 1 token
		if (n == 0) throw;

		tokenPrice = price * 1000000000000;
		tokenIssueDeadline = deadline;
		tokensToIssue = n;
		tokensIssued = 0;

		notifyTokenIssued(n, price, deadline);
	}

It doesn't matter if not all tokens are sold, but once all tokens are sold, `buyTokens()` will throw an exception and the user can no longer buy tokens. In the future, I may add a Kickstarter-style all-or-nothing flag. If that flag is set, either all tokens must be sold or everyone gets a refund.

When a user calls `buyTokens()`, several checks are performed based on the parameters passed to `issueTokens()`.

The code below tries to prevent users from buying more tokens than issued. I'm not entirely sure if it's safe against race conditions, because I don't fully understand how multiple calls to the same function are handled by the Ethereum Virtual Machine. As such, my reasoning below might be completely wrong, in which case you may find interesting opportunities to attack:

	function buyTokens () {
		tokens = msg.value / tokenPrice;

		if (now > tokenIssueDeadline) throw;
		if (tokensIssued >= tokensToIssue) throw;		
		tokensIssued += tokens;
		if (tokensIssued > tokensToIssue) throw;

		DaoAccount account = accountFor(msg.sender, true);
		if (account.buyTokens.value(msg.value)() != tokens) throw;

The above code first increases `tokensIssued` and then checks if this leads to too many tokens being issued.

Let's assume the state of the smart contract is as follows: `tokensToIssue` is `3` and `tokensIssued` is `2`, meaning two out of three tokens have been sold.

Now a user A calls `buyTokens()` in order to buy the last remaining token. The execution reaches the line where `tokensIssued` increased to `3`. At this moment, user B also calls `buyTokens()`. `tokensIssued` is increased to `4`, which causes a `throw` in the next line for user B. This `throw` undoes the increase to `tokensIssued`, which reverts to `3` again.

But what if `account.buyTokens.value` throws for user A? In that case, `tokensIssued` goes back to `2`, so another user can call `buyTokens()` again.

## Please Rob It!

The `DaoChallenge` contract published at [0x131a...d811](https://etherscan.io/address/0x131a76478D2eef5cEAA28e93030eB8a8894aD811) and its first `DaoAccount` are funded with about €100 worth of ether in total. Please rob them!

This earlier post explains [how to use the contract](https://medium.com/@dao.challenge/challenge-5-segregated-funds-usability-6e749badb24d#.hy9rb52lu): you'll need to fill out the address from the above Etherscan.io link. You'll also need the latest JSON interface, which you can find on the contract page if you go to Contract Source and scroll down to Contract ABI.

The [usual rules](https://medium.com/@dao.challenge/challenge-1-296cb5dab68f) apply. Most importantly: don’t go after me and my private keys. Even if you manage to rob only one of the two contracts, I’ll send you the rest. The full source is on [GitHub](https://github.com/Sjors/dao-challenge/tree/challenge-7).
