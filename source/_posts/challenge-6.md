---
title: Challenge 6 - Transfers
date: 2016/8/12
---

In [challenges 4 and 5](https://dao-challenge.herokuapp.com/2016/08/05/recap-challenge-1-5/), I came up with an architecture where user funds are stored in segregated smart contracts. That way, if a hacker somehow completely drains their own contract, they can't also drain other people's funds.

Today I'm once again adding support for transferring tokens between users.

<!-- more -->

For this to work, the sender and recipient both need to have a `DaoAccount`. The sender account is automatically created when the sender buys tokens. Meanwhile, the recipient can call `createAccount()`, which creates a `DaoAccount` instance for the recipient without any tokens. We could create this account automatically upon receipt of the tokens, but then the sender might accidentally send tokens and ether into space. By creating a `DaoAccount` themselves, the recipient indicates they are in control of the account:

	function createAccount () {
		accountFor(msg.sender, true);
	}  

To transfer tokens and their corresponding ether, the sender needs to call `transfer()` on `DaoChallenge` with the number of tokens and the recipient's address. This function ensures the recipient has an account and then relays the instruction to the sender's `DaoAccount`:

	function transfer(uint256 tokens, address recipient) noEther {
		DaoAccount account = accountFor(msg.sender, false);
		if (account == DaoAccount(0x00)) throw;

		DaoAccount recipientAcc = accountFor(recipient, false);
		if (recipientAcc == DaoAccount(0x00)) throw;

		account.transfer(tokens, recipientAcc);
		notifyTransfer(msg.sender, recipient, tokens);
	}

The sender's `DaoAccount` checks that they have enough tokens and then calls `receiveTokens()` on the recipient's `DaoAccount`. It sends the correct amount of ether along with this function call:

	function transfer(uint256 tokens, DaoAccount recipient) noEther onlyDaoChallenge {
		if (tokens == 0 || tokenBalance == 0 || tokenBalance < tokens) throw;
		tokenBalance -= tokens;
		recipient.receiveTokens.value(tokens * tokenPrice)(tokens);
	}

On the receiving end, a number of things need to be checked. Obviously, we want to make sure the amount of ether matches the number of tokens, but we also want to make sure nobody is messing with us. This is because the function is public and anyone can call it, so an attacker could potentially trick the recipient `DaoAccount` into thinking it received tokens. I'm not exactly sure how that would happen, but it's better to be safe than sorry.

To determine that the sender is legit, `receiveTokens()` ensures the sender is actually a `DaoAccount` and that this `DaoAccount` belongs to the parent `DaoChallenge`:

	function receiveTokens(uint256 tokens) {
		// Check that the sender is a DaoAccount and belongs to our DaoChallenge
		DaoAccount sender = DaoAccount(msg.sender);
		if (!AbstractDaoChallenge(daoChallenge).isMember(sender, sender.getOwnerAddress())) throw;

		uint256 amount = msg.value;

		// No zero transfer:
		if (amount == 0) throw;

		if (amount / tokenPrice != tokens) throw;

		tokenBalance += tokens;
	}

My use of `AbstractDaoChallenge` instead of `DaoChallenge` here is a [workaround for cyclic dependencies](https://github.com/ConsenSys/truffle/issues/135#issuecomment-223996851). What's important is that it calls `isMember()` on the parent `DaoChallenge`, which checks the sender against its `DaoAccount` list:

    // Check if a given account belongs to this DaoChallenge.
	function isMember (DaoAccount account, address allegedOwnerAddress) returns (bool) {
		if (account == DaoAccount(0x00)) return false;
		if (allegedOwnerAddress == 0x00) return false;
		if (daoAccounts[allegedOwnerAddress] == DaoAccount(0x00)) return false;
		// allegedOwnerAddress is passed in for performance reasons, but not trusted
		if (daoAccounts[allegedOwnerAddress] != account) return false;
		return true;
	}

If all the above checks pass, the ether is moved from sender to recipient, and their token balances are adjusted. The recipient can now withdraw their ether or send the tokens onward to someone else.

This transfer functionality turned out a lot more complicated than I would have liked. Any suggestions for simplifying it are much appreciated.

## Please Rob It!

The `DaoChallenge` contract published at [0x1616...d732](https://etherscan.io/address/0x16163229926aad97318fac28c8d06101954ad732) and the `DaoAccount` at [0x964e...b850](https://etherscan.io/address/0x964eb46dDf4b37cDC145220AAcE479167214b850) are funded with about €100 worth of ether in total. Please rob them!

This earlier post explains [how to use the contract](https://medium.com/@dao.challenge/challenge-5-segregated-funds-usability-6e749badb24d#.hy9rb52lu). You'll need to fill out 0x16163229926aad97318fac28c8d06101954ad732 as the address and copy-paste the latest JSON interface from [here](https://gist.githubusercontent.com/Sjors/7e82d476d347184904ac3640cb8ac00d/raw/77ef648e940081549f661e4ad01e8fe5d84ee38a/DaoChallenge.json).

The [usual rules](https://medium.com/@dao.challenge/challenge-1-296cb5dab68f) apply. Most importantly: don’t go after me and my private keys. Even if you manage to rob only one of the two contracts, I’ll send you the rest. The full source is on [GitHub](https://github.com/Sjors/dao-challenge/tree/challenge-6).
