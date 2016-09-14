---
title: Challenge 10 - Execute Sell Order
date: 2016/9/16
---

In [last week's smart contract](https://dao-challenge.herokuapp.com/2016/09/10/challenge-9/), I allowed the user to place a sell order for their tokens. They could specify how many tokens they wish to sell and at what price. This week I'm allowing buyers to execute the order, and receive tokens.

<!-- more -->

Sell orders have their own contract. When a buyer wants to take the offer, they call `execute()` along with a sufficient amount of ether (`price * tokens`):

	contract SellOrder {
	  address public owner; // DaoAccount that created the order
	  uint256 public tokens;
	  uint256 public price; // Wei per token

      function execute () {
        if (msg.value != tokens * price) throw;

        // Tokens are sent to the buyer in DaoAccount.executeSellOrder()
        // Send ether to seller:
        suicide(owner);
      }
	}

The `suicide(owner)` line at the end moves ether to the seller's `DaoAccount` and makes sure the order can't be executed twice. The buyer's `DaoAccount` takes care of adding tokens they bought:

	contract DaoAccount {
	  function executeSellOrder(SellOrder order) onlyDaoChallenge {
        uint256 tokens = order.tokens();
        tokenBalance += tokens;
        order.execute.value(msg.value)();
      }
      
Note that I didn't actively protect `executeSellOrder()` against [reentrancy](http://hackingdistributed.com/2016/07/13/reentrancy-woes/). The buyers token balance is increased before the order is executed. `SellOrder` is a trusted contract, since it was created by our own smart contract. It calls `suicide()` which sends the funds to `DaoAccount`, which is also a trusted contract. So I believe there is no need to prevent reentrance here. It's probably good practice to prevent reentrance by default, and only allow it if there is a good reason for that.
	  
## Test Driven Development

I wrote this code entirely through writing [tests](https://github.com/Sjors/dao-challenge/commit/131e7b84fd6e9e42d689800043937042f0eafce9#diff-08bfad511235c02b409ff759af38fea8). I used to test smart contracts in a web interface, see section IDE Woes in my post about [Challenge 5](https://medium.com/@dao.challenge/challenge-4-segregate-user-funds-986001587fae#.5hga47ua2). I find this much too tedious at the current level of multi-contract complexity. 

It's much easier to just add a test, run `dapple test`, fix problems and run tests again until they pass. Every time I think of a way an attacker could compromise the contract, I add another test for that and improve the code until the test passes. 

This is called [Test Driven Development](https://en.wikipedia.org/wiki/Test-driven_development) and it's good idea in general. I may however end up regretting the lack of manual testing. As always, this could be to your benefit, if my laziness makes it easier for you to rob my contract.

The test themselves need some restructuring. I'm using mocks to simulate each contracts parent, but not the other way around for their children. For example in all `DaoAccount` tests there is a mock `DaoChallenge` which simulates the behavior of the real `DaoChallenge`, without actually running it's code. This allows me to specifically test the desired behavior of `DaoAccount`.

I need to do this the other way around as well: all tests for `DaoChallenge` should use a mock `DaoAccount`, all tests for `DaoAccount` should use a mock `SellOrder`. I'm not sure how to do this at the moment.

## Coming Soon - Buy Orders

The current mechanism is not very user friendly. A potential buyer has to look up and study all SellOrder contracts, import the one they like into a wallet and then call `executeOrder()`. To make this easier, I'd like to add a BuyOrder. Initially this will be a very simple limit order, which either executes immediately or fails. This order would loop through all sell orders, and automatically execute is the price is below the limit.

Also not that the ether is deposited in the sellers `DaoAccount` afer an order is executed. There's still no way for a user to take ether out of their `DaoAccount`. 

## Please Rob It!

The `DaoChallenge` contract published at [0x6623...3a90](https://etherscan.io/address/0x66230ca3603e071c942f9c1c8824be91c91f3a90) and its first `DaoAccount` are funded with about €100 worth of ether in total. Please rob them!

This earlier post explains [how to use the contract](https://medium.com/@dao.challenge/challenge-5-segregated-funds-usability-6e749badb24d#.hy9rb52lu): you'll need to fill out the address from the above Etherscan.io link. You'll also need the latest JSON interface, which you can find on the contract page if you go to Contract Source and scroll down to Contract ABI.

The [usual rules](https://medium.com/@dao.challenge/challenge-1-296cb5dab68f) apply. Most importantly: don’t go after me and my private keys. Even if you manage to rob only one of the two contracts, I’ll send you the rest. The full source is on [GitHub](https://github.com/Sjors/dao-challenge/tree/challenge-10).
