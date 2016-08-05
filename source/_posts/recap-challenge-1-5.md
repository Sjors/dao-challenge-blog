---
title: Recap Challenge 1 - 5
---

## Challenge 1 - Basic Token Contract

Status: live at [0xBe56...889f](https://etherscan.io/address/0xBe56093286038885733a66e554DD43a22a45889f), but not robbed yet.

The contract allows people to buy tokens for 1 finney (0.001 ETH / $0.01) each:

    function () {
		address sender = msg.sender;
		if(tokenBalanceOf[sender] != 0) {
			throw;
		}
		tokenBalanceOf[sender] = msg.value / tokenPrice; // rounded down
		notifySellToken(tokenBalanceOf[sender], sender);
	}

Users can get their tokens refunded:

	function refund() noEther {
		address sender = msg.sender;
		uint256 tokenBalance = tokenBalanceOf[sender];
		if (tokenBalance <= 0) { throw; }
		tokenBalanceOf[sender] = 0;
		sender.send(tokenBalance * tokenPrice);
		notifyRefundToken(tokenBalance, sender);
	}

Thus far, no one has been able to redeem more ether than they put in. You still have six weeks to try to rob the 9.5 ETH from this contract.

More information can be found in the [original article](https://medium.com/@dao.challenge/challenge-1-296cb5dab68f#.jjd06x4le).

In an effort to better understand cross-chain replay attacks, I [drained this contract](https://medium.com/@dao.challenge/replay-attack-challenge-998e0123df21) on the ETC chain while leaving it intact on the ETH chain.

## Challenge 2 - Transfer Tokens

Status: live at [0x08d6...2abf](http://etherscan.io/address/08d698358b31ca6926e329879db9525504802abf), but empty after I robbed it myself.

I added the following function that allows users to transfer tokens to another user:

	function transfer(address recipient, uint256 tokens) noEther {
		address sender = msg.sender;

		if (tokenBalanceOf[sender] < tokens) throw;
		if (tokenBalanceOf[recipient] + tokens < tokenBalanceOf[recipient]) throw; // Check for overflows
		tokenBalanceOf[sender] -= tokens;
		tokenBalanceOf[recipient] += tokens;
		notifyTranferToken(tokens, sender, recipient);
	}

When I wrote this code, I expected that if there was any vulnerability, it would be in this new transfer function. But as explained in Challenge 3 below, the vulnerability was in a different function. However, if you believe there is *also* a vulnerability in this `transfer()` function, I suggest that you try to rob Challenge 3, which has the exact same `transfer()` function.

More information can be found in the [original article](https://medium.com/@dao.challenge/challenge-2-a749c4158023#.dcxurblc1).

## Challenge 3 - Vulnerability Fixed

Status: live at [0x80f1...0e57 ](http://etherscan.io/address/0x80f1f62b8b365c5326100d462d8570771b8d0e57), but not robbed yet.

The following function in Challenge 2 turned out to be vulnerable:

	function withdrawEtherOrThrow(uint256 amount) {
		bool result = msg.sender.call.value(amount)();
		if (!result) {
			throw;
		}
	}

Functions are public by default. Since I mistakenly assumed the above function was private, I didn't write any checks for it. That allowed anyone to drain the entire contract simply by calling `withdrawEtherOrThrow()`.

Fortunately for me, I realized my mistake several days later, ran to my computer, and robbed it myself. Someone else also discovered my mistake, so if I had waited only half an hour longer, I would have been too late.

The fix was simple enough. All I had to do was add `private` to the function definition:

    function withdrawEtherOrThrow(uint256 amount) private {
		bool result = msg.sender.call.value(amount)();
		if (!result) {
			throw;
		}
	}

To date, this new contract has not been robbed.

More information can be found in the [original article](https://medium.com/@dao.challenge/challenge-3-how-i-almost-lost-100-1a11a9824ccb#.xayw0s8n0).

## Challenge 4 - Segregate User Funds

Status: live at [0x4B90...10eF](https://etherscan.io/address/0x4B902704026D14117b5E9EFA7FdaFDfF4bA610eF), but not robbed yet.

When the DAO was [hacked](http://uk.businessinsider.com/dao-hacked-ethereum-crashing-in-value-tens-of-millions-allegedly-stolen-2016-6), the hacker — usually called The Attacker — managed to withdraw more than they put in. I want to make this less likely by keeping user funds in segregated contracts. The hope is that if a hacker somehow completely drains his own contract, he can't also drain other people's funds.

The main DaoChallenge contract creates and keeps track of DaoAccount contracts, one for each user:

	mapping (address => DaoAccount) private daoAccounts;

	function createAccount () noEther returns (DaoAccount account) {
		address accountOwner = msg.sender;
		address challengeOwner = owner; // Don't use in a real DAO

		// One account per address:
		if(daoAccounts[accountOwner] != DaoAccount(0x00)) throw;

		daoAccounts[accountOwner] = new DaoAccount(accountOwner, challengeOwner);
		return daoAccounts[accountOwner];
	}   

Each DaoAccount then takes care of buying tokens and refunds:

	function () onlyOwner returns (uint256 newBalance){
		uint256 amount = msg.value;

		// No fractional tokens:
		if (amount % tokenPrice != 0) {
			throw;
		}

    uint256 tokens = amount / tokenPrice;

		tokenBalance += tokens;

    return tokenBalance;
	}

	function refund() noEther onlyOwner {
		if (tokenBalance == 0) throw;
		tokenBalance = 0;
		withdrawEtherOrThrow(tokenBalance * tokenPrice);
	}

There's no way to make `refund()` return more than what's in this DaoAccount, and this protects other DaoAccount instances and DaoChallenge. Of course, there might be a vulnerability in DaoAccount that allows an attacker to drain every single instance, one by one.

So far, that hasn't happened.

More information can be found in the [original article](https://medium.com/@dao.challenge/challenge-4-segregate-user-funds-986001587fae#.a72ul7r1n).

## Challenge 5 - Segregated Funds Usability

Status: live at [0xae06...5d67](https://etherscan.io/address/0xae0680c49df146e18b2bc19635e5e402494b5d67), but not robbed yet.

The contract in Challenge 4 was very tedious to use in practice. It required the user to find out which DaoAccount belonged to them and import that into their wallet before they could buy tokens from it.

I added wrapper functions to DaoChallenge; they automatically look up the user's DaoAccount and call their corresponding function there:

    function buyTokens () returns (uint256 tokens) {
	  DaoAccount account = accountFor(msg.sender, true);
		tokens = account.buyTokens.value(msg.value)();

		notifyBuyToken(msg.sender, tokens, msg.value);
		return tokens;
 	}

	function withdraw(uint256 tokens) noEther {
		DaoAccount account = accountFor(msg.sender, false);
		if (account == DaoAccount(0x00)) throw;

		account.withdraw(tokens);
		notifyWithdraw(msg.sender, tokens);
	}

This gives more power to the main DaoChallenge contract, which could potentially be exploited to drain all DaoAccount contracts. However, so far that has not happened.

More information can be found in the [original article](https://medium.com/@dao.challenge/challenge-5-segregated-funds-usability-6e749badb24d#.gmas7271f).

## What's Next?

I want to make these contracts more useful. I'd like to bring back transferring tokens between users, which is a feature I removed in Challenge 4. I also want to introduce a proposal system similar to that of the DAO, where users can vote with their tokens on how the DaoChallenge spends its funds. I'll try to introduce these features in incremental steps, offering a bounty each time for those who can rob any of those future contracts.
