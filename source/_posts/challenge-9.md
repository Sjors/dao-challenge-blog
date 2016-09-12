---
title: Challenge 9 - Sell Order
date: 2016/9/10
---

In this week's smart contract, I'm allowing the user to place a sell order for their tokens. They can specify how many tokens they wish to sell and at what price. They can also cancel the order. Other users won't be able to buy, so the sell orders aren't very useful yet. I'll add support for the buy side later.

<!-- more -->

Sell orders have their own contract:

	contract SellOrder {
	  address public owner; // DaoAccount that created the order
	  uint256 public tokens;
	  uint256 public price; // Wei per token

	  function SellOrder (uint256 _tokens, uint256 _price) noEther {
	    owner = msg.sender;
	    tokens = _tokens;
	    price = _price;
	  }

	  function cancel () noEther onlyOwner {
	    suicide(owner);
	  }

	  function execute () {
	  	 // TODO: transfer tokens to buyer, send ether to seller
	  }
	}

The user's `DaoAccount` is responsible for creating a sell order. Tokens are moved into the order, which prevents the user from selling more tokens than they own. It also means that once a voting feature is implemented, a user won't be able to vote using these tokens:

	contract DaoAccount {
	  function placeSellOrder(uint256 tokens, uint256 price) noEther onlyDaoChallenge returns (SellOrder) {
	    if (tokens == 0 || tokenBalance == 0 || tokenBalance < tokens) throw;
	    if (tokenBalance - tokens > tokenBalance) throw; // Overflow
	    tokenBalance -= tokens;

	    SellOrder order = new SellOrder(tokens, price, challengeOwner);
	    return order;
	  }

The `DaoChallenge` contract is responsible for tracking all orders:

	contract DaoChallenge {
	  mapping (address => SellOrder) public sellOrders;

	  function placeSellOrder(uint256 tokens, uint256 price) noEther {
		 DaoAccount account = accountFor(msg.sender, false);
		 if (account == DaoAccount(0x00)) throw;

		 SellOrder order = account.placeSellOrder(tokens, price);

		 sellOrders[address(order)] = order;

		 notifyPlaceSellOrder(tokens, price);
	  }

Canceling an order is a little cumbersome at the moment. The user needs to specify the order's smart contract address when calling `cancelSellOrder()` on `DaoChallenge`:

	contract DaoChallenge {
	  function cancelSellOrder(address addr) noEther {
       DaoAccount account = accountFor(msg.sender, false);
       if (account == DaoAccount(0x00)) throw;

       SellOrder order = sellOrders[addr];
       if (order == SellOrder(0x00)) throw;

       if (order.owner() != address(account)) throw;

       sellOrders[addr] = SellOrder(0x00);

       account.cancelSellOrder(order);

       notifyCancelSellOrder();
     }

`DaoChallenge` looks up the order and makes sure that only the user who created the order can cancel it. It then instructs `DaoAccount` to cancel the order:

    contract DaoAccount {
      function cancelSellOrder(SellOrder order) noEther onlyDaoChallenge {
        uint256 tokens = order.tokens();
        tokenBalance += tokens;
        order.cancel();
      }

`DaoAccount` restores its token balance and then calls the `cancel()` method on the `SellOrder` contract. In turn, the `cancel()` function just deletes the sell order contract.

I wrote a number of [tests](https://github.com/Sjors/dao-challenge/tree/challenge-9/contracts) for this behavior. One test, for example, makes sure a user can't cancel an order twice. However, these tests are far from complete.

## Please Rob It!

The `DaoChallenge` contract published at [0xB523...30AF](https://etherscan.io/address/0xb5232102E71a7ff376EBdEaE59E19D031CBE30Af) and its first `DaoAccount` are funded with about €100 worth of ether in total. Please rob them!

This earlier post explains [how to use the contract](https://medium.com/@dao.challenge/challenge-5-segregated-funds-usability-6e749badb24d#.hy9rb52lu): you'll need to fill out the address from the above Etherscan.io link. You'll also need the latest JSON interface, which you can find on the contract page if you go to Contract Source and scroll down to Contract ABI.

The [usual rules](https://medium.com/@dao.challenge/challenge-1-296cb5dab68f) apply. Most importantly: don’t go after me and my private keys. Even if you manage to rob only one of the two contracts, I’ll send you the rest. The full source is on [GitHub](https://github.com/Sjors/dao-challenge/tree/challenge-9).
