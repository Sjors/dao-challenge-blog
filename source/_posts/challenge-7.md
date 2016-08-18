---
title: Challenge 7 - Token Issuing
date: 2016/8/19
---

In [earlier challenges](https://dao-challenge.herokuapp.com/2016/08/08/recap-challenge-1-5/), users could buy tokens throughout the lifetime of the smart contract. The token price was baked into the code, so it couldn't be changed to reflect the market value of those tokens.

Today I'm allowing the smart contract owner to issue a fixed number of tokens, and determine the price and deadline.
<!-- more -->

First, let me explain where things are going over the next few weeks. I plan to remove the `transfer()` and `refund()` functions. Instead I'll create `BuyOrder` and `SellOrder` contracts as well a `dividend()` function. Users won't be able to redeem tokens directly for ether; instead they can trade tokens with each other. To reward users for holding on to their tokens, the owner of the smart contract can pay dividend to all participants.

In the above where it says "owner of the smart contract", I plan to replace that by a majority vote or [something fancier](https://blog.ethereum.org/2014/08/21/introduction-futarchy/).

So, in a future version of this smart contract, tokens are issued in batches, used to distribute dividends and they can be traded between users. I think this is simpler system than the [Split Proposals](https://daowiki.atlassian.net/wiki/display/DAO/How+to+split+the+DAO%3A+Step-by-Step) used by The Dao.

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

It doesn't matter if not all tokens are sold, but once all tokens are sold, `buyTokens()` will throw an exception and the user can no longer buy tokens. In the future I may add a Kickstarter style - all or nothing - flag. If that flag is set, either all tokens must be sold or everyone gets a refund.

When a user calls `buyTokens()` several checks are performed, based on the parameters passed to `issueTokens()`.

The code below tries to prevent users from buying more tokens than issued. I'm not entirely sure if it's safe against race conditions, because I don't fully understand how multiple calls to the same function are handled by the Ethereum Virtual Machine. My reasoning below might therefore be completely wrong, in which case you may find interesting opportunities to attack.
		
	function buyTokens () {
		tokens = msg.value / tokenPrice;

		if (now > tokenIssueDeadline) throw;
		if (tokensIssued >= tokensToIssue) throw;		
		tokensIssued += tokens;
		if (tokensIssued > tokensToIssue) throw;

		DaoAccount account = accountFor(msg.sender, true);
		if (account.buyTokens.value(msg.value)() != tokens) throw;
	
The above code first increases `tokensIssued` and then checks if this leads to too many tokens being issued.

Let's say `tokensToIssue` is 3 and `tokensIssued` is 2; two out of three tokens have been sold. User A calls `buyTokens()` in order to buy the last remaining token. The execution reaches the line where `tokensIssued` increased to 3. At this moment user B also calls `buyTokens()`. `tokensIssued` is increased to 4, which causes a `throw` in the next line for user B. This `throw` undoes the increase to `tokensIssued`, so it's 3 again.

What if `account.buyTokens.value` throws for user A? In that case `tokensIssued` goes back to 2, so another user can call `buyTokens()` again.

## Please Rob It!

The `DaoChallenge` contract published at [0x131a...d811](https://etherscan.io/address/0x131a76478D2eef5cEAA28e93030eB8a8894aD811) and its first `DaoAccount` are funded with about €100 worth of ether in total. Please rob them!

This earlier post explains [how to use the contract](https://medium.com/@dao.challenge/challenge-5-segregated-funds-usability-6e749badb24d#.hy9rb52lu). You'll need to fill out the address from the above Etherscan.io link. You also need the latest JSON interface, which you can find on that page if you go to Contract Source and scroll down to Contract ABI.

The [usual rules](https://medium.com/@dao.challenge/challenge-1-296cb5dab68f) apply. Most importantly: don’t go after me and my private keys. Even if you manage to rob only one of the two contracts, I’ll send you the rest. The full source is on [GitHub](https://github.com/Sjors/dao-challenge/tree/challenge-7).
