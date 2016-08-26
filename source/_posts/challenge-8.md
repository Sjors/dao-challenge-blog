---
title: Challenge 8 - Bug Fix & Tests
date: 2016/8/26
---

In [last week's smart contract](https://dao-challenge.herokuapp.com/2016/08/19/challenge-7/), I allowed the smart contract owner to issue a fixed number of tokens and determine the price and deadline. Unfortunately, I made several mistakes, including how `DaoAccount` enforces the token price.
<!-- more -->

Before I explain what went wrong, I should mention that I didn't take any ether from the contract, because I couldn't find a way to exploit these mistakes. If you see a way to rob it, have fun!

The problem was that once a `DaoAccount` is created for a user, it freezes the token price forever, and it ignores subsequent changes in its parents' `DaoChallenge` token price:


    contract DaoAccount {
	    uint256 tokenPrice;
	    function DaoAccount (address _owner, uint256 _tokenPrice, address _challengeOwner) noEther {
	    	tokenPrice = _tokenPrice; // set once, never changed

When the user buys tokens, it always uses the initial token price:

    function buyTokens() onlyDaoChallenge returns (uint256 tokens) {
		uint256 amount = msg.value;
		tokens = amount / tokenPrice;
		tokenBalance += tokens;

The solution is to remove the `tokenPrice` parameter from the constructor:

    contract DaoAccount {
	    function DaoAccount (address _owner, address _challengeOwner) noEther {

When the user buys tokens, it now gets the token price from its parent `DaoChallenge`:

    function buyTokens() onlyDaoChallenge returns (uint256 tokens) {
		uint256 amount = msg.value;
		uint256 tokenPrice = AbstractDaoChallenge(daoChallenge).tokenPrice();
		tokens = amount / tokenPrice;
		tokenBalance += tokens;

I explained the use of `AbstractDaoChallenge` [last week](https://dao-challenge.herokuapp.com/2016/08/19/challenge-7/), but I left out some checks in the above code examples. As always, the full source code is [on GitHub]([GitHub](https://github.com/Sjors/dao-challenge/tree/challenge-8)).

As I made the above fix, I began to realize that the current implementation of `transfer()` no longer made sense. It used to transfer both tokens and ether, but how much ether is it supposed to transfer for each token if the token price is no longer fixed?

I simplified this function, so now it only transfers tokens. In addition, I added an overflow check:

	function transfer(uint256 tokens, DaoAccount recipient) noEther onlyDaoChallenge {
		if (tokens == 0 || tokenBalance == 0 || tokenBalance < tokens) throw;
		if (tokenBalance - tokens > tokenBalance) throw; // Overflow
		tokenBalance -= tokens;
		recipient.receiveTokens(tokens);
	}

In turn, this change meant I had to remove the `withdraw()` function. Otherwise, user A could buy one token and send it to user B and withdraw their ether. In this case, the `DaoChallenge` and two `DaoAccount` smart contracts would be left with one token, but no ether.

I realize this week's contract is even less useful than last week's; you can't even get your money back! But bear with me: it will get better.

## Please Rob It!

The `DaoChallenge` contract published at [0x0xae42...6Abe](https://etherscan.io/address/0xae42990ad29747c9Ab0C16098b8c5393E53C6Abe) and its first `DaoAccount` are funded with about €100 worth of ether in total. Please rob them!

This earlier post explains [how to use the contract](https://medium.com/@dao.challenge/challenge-5-segregated-funds-usability-6e749badb24d#.hy9rb52lu): you'll need to fill out the address from the above Etherscan.io link. You'll also need the latest JSON interface, which you can find on the contract page if you go to Contract Source and scroll down to Contract ABI.

The [usual rules](https://medium.com/@dao.challenge/challenge-1-296cb5dab68f) apply. Most importantly: don’t go after me and my private keys. Even if you manage to rob only one of the two contracts, I’ll send you the rest. The full source is on [GitHub](https://github.com/Sjors/dao-challenge/tree/challenge-8).

## Dapple and Tests

In addition to the above changes, I added [Dapple](https://dapple.readthedocs.io/) to the project. Apparently it does many things, but for now, the only feature I'm using is [Tests](https://dapple.readthedocs.io/en/master/test/), and it's great.

Here's one test I wrote, which checks that when user A pays for a token, their token balance is increased:

	contract DaoAccountBuyTokenTest is DaoAccountTest {
   	  function testBuyTwoTokens() {
       uint256 tokens = userA.buyTokens(chal, chal.tokenPrice() * 2);
       assertEq( tokens, 2 );
       assertEq( acc.getTokenBalance(), 2 );
       assertEq( acc.balance, chal.tokenPrice() * 2 );

Notice that I'm not directly calling `buyTokens()` on the `DaoAccount` of `userA`. This wouldn't work, because the `buyTokens()` function checks if it's called from `DaoChallenge`:

		modifier onlyDaoChallenge() {if (daoChallenge != msg.sender) throw; _}

		function buyTokens() onlyDaoChallenge { ... }


Tests call functions using the default test account, which isn't a `DaoChallenge`, and the above modifier would cause a `throw`.

To ensure `buyTokens()` on the `DaoAccount` of `userA` is called by a `DaoChallenge`, I needed to create a `DaoChallenge`.

In addition, I want to focus on `DaoAccount` and not worry about the specifics of its `DaoChallenge` parent. So I created a mock `DaoChallenge` smart contract: a contract much simpler than the real one, with fewer functions. It also comes with functions that make writing tests easier:

	contract DaoChallenge {
	  uint256 public tokenPrice = 1;


	  function createAccount () returns (DaoAccount) {
	    return new DaoAccount(msg.sender);
	  }

	  function buyTokens (DaoAccount account) returns (uint256) {
	    return account.buyTokens.value(msg.value)();
	  }

	  function setTokenPrice (uint256 price) {
       tokenPrice = price;
  	  }
	}

Note that in this mock `DaoChallenge`, all protection has been removed. This is OK, because `DaoChallenge` should have its own tests (which in turn would use a mock `DaoAccount`).

For instance, the original `buyTokens` checks the token issue deadline. The mock function, on the other hand, simply calls `buyTokens` on the `DaoAccount`.

The mock contract also lets the test script set the token price to any value. Again, this occurs without any safety checks; in the real `DaoChallenge`, only the challenge owner can set the token price.

In order to write tests involving multiple users, I created a smart contract that simulates user actions:

	contract User {
	  DaoAccount public account;

	  function buyTokens (DaoChallenge chal, uint256 amount) returns (uint256) {
	    return chal.buyTokens.value(amount)(account);
	  }
	}

A test user is created by calling `userA = new User();`. You can give the user a budget using `userA.send(10)`.

The test shown above starts out with `userA.buyTokens(chal, chal.tokenPrice() * 2)`. `userA` and `chal` were created earlier in the test suite. The token price is `1` by default, so `buyTokens()` on `User` instance `userA` is called with `2`. No funds are sent yet.

`userA` was given a budget of `10`. It sends `2` to the mock `DaoChallenge` by calling `chal.buyTokens.value(amount)(account)`.

The real `DaoChallenge` is able to look up the `DaoAccount` corresponding to each user, but the mock is not so smart. This is why each `User` contract creates a `DaoAccount` by itself. It then passes that account along when calling `buyTokens()`.

The `buyTokens` method on `DaoChallenge` is very simple (see above): it just sends the ether it receives onward to the `DaoAccount` instance. Now the sender is what `DaoAccount` expects and it sets its token balance to `2`. As a result, the test passes.

The above example is simplified. You can find the complete version, as well as all the tests I wrote for this week's challenge, [here](https://github.com/Sjors/dao-challenge/blob/challenge-8/contracts/dao-account-spec.sol).
