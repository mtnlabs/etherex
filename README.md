EtherEx
=======
[![Build Status](https://travis-ci.org/etherex/etherex.svg?branch=master)](https://travis-ci.org/etherex/etherex) [![SlackIn](http://slack.etherex.org/badge.svg)](http://slack.etherex.org) [![Join the chat at https://gitter.im/etherex/etherex](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/etherex/etherex?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Decentralized exchange built on Ethereum.

<img src="/frontend/screenshot.png" />

About
-----

This repository contains the source code that runs the exchange on Ethereum as a set of contracts, along with the UI, tests, tools and documentation.


Components
----------

* contracts: Ethereum contracts in [Serpent](https://github.com/ethereum/serpent)
* frontend: [React.js](https://github.com/facebook/react) UI
* tests: EtherEx tests


Requirements
------------
* [Serpent](https://github.com/ethereum/serpent) compiler by Vitalik Buterin
* [cpp-ethereum](https://github.com/ethereum/cpp-ethereum) client by Gavin Wood
* [go-ethereum](https://github.com/ethereum/go-ethereum) client by Jeffrey Wilcke
* [pyethereum](https://github.com/ethereum/pyethereum) Python Ethereum client (tests only)
* [PyEPM](https://github.com/etherex/pyepm) for deployment
* [node](http://nodejs.org/) and [grunt](http://gruntjs.com/) for UI development


Installation
------------

Start by cloning this repository.

```
git clone https://github.com/etherex/etherex.git
```


### Development / testing

This will install `pyethereum` and `ethereum-serpent` if you don't already have those installed.

```
pip install -r dev_requirements.txt
```

#### Running tests

```
py.test -vvrs
```

Refer to [Serpent](https://github.com/ethereum/serpent) and [pyethereum](https://github.com/ethereum/pyethereum) for their respective usage.


### UI development

You will need a working node.js setup ([instructions](https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager)) and globally installed `grunt-cli` ([instructions](http://gruntjs.com/getting-started)).

```
cd frontend
npm install
grunt
```

And open `http://localhost:8089/` in your browser.

### Deployment

Requires a local client (Go or C++) with JSONRPC, [Serpent](https://github.com/ethereum/serpent) and [PyEPM](https://github.com/etherex/pyepm)

```
cd contracts
pyepm EtherEx.yaml
```


API
---

* The API is the format of the data field for the Ethereum transactions.
* Subcurrencies need to support the [Subcurrency API](#subcurrency-api).
* You only need an Ethereum client to use the API.


## Operations

Methods (with serpent type definitions):
```
[
  price:[int256]:int256,
  buy:[int256,int256,int256]:int256,
  sell:[int256,int256,int256]:int256,
  trade:[int256,int256[]]:int256,
  cancel:[int256]:int256,
  deposit:[int256,int256,int256]:int256,
  withdraw:[int256,int256]:int256,
  add_market:[int256,int256,int256,int256,int256,int256]:int256,
  get_last_market_id:[]:int256,
  get_market:[int256]:int256[],
  get_trade:[int256]:int256[],
  get_trade_ids:[int256]:int256[],
  get_sub_balance:[int256,int256]:int256[]
]
```


## Price API
```
self.exchange.price(market_id)
```


## Trade API

### Add buy / sell trade
```
self.exchange.buy(amount, price, market ID)
self.exchange.sell(amount, price, market ID)
```

### Trade
```
self.exchange.trade(max_amount, trade_IDs)
```

### Deposit (subcurrency contracts only, [see below](#subcurrency-api))
```
self.exchange.deposit(address, amount, market_ID)
```

### Withdraw
```
self.exchange.withdraw(amount, market_ID)
```

### Cancel trade
```
self.exchange.cancel(trade_ID)
```

### Adding a market
```
self.exchange.add_market(currency_name, contract_address, decimal_precision, price_denominator, minimum_total, category)
```

#### Market names

Market names follow the "<currency name>/ETH" convention. When registering a new market, submit the currency name as a three or four letter uppercase identifier, ex.: "BOB" for BobCoin.

#### Contract address

The subcurrency contract address.

#### Decimal precision

The subcurrency's decimal precision as an integer.

#### Price denominator

* Denominator for price precision, ex. 10000 (10000 => 1 / 10000 => 0.0001)

#### Minimum trade total
When adding a subcurrency, set the minimum trade total high enough to make economic sense. A minimum of 10 ETH (1000000000000000000000 wei) is recommended.

#### Categories
```
1 = Subcurrencies
2 = Crypto-currencies
3 = Real-world assets
4 = Fiat currencies
```
EtherEx allows you to categorize your subcurrency into four main categories. Since everything is represented as subcurrencies, those categories are simply for convenience. If you have a DApp that has its own token, that would go in the regular subcurrency section `1`. If your token represents a fiat currency redeemable at a gateway, put it in `4`. If your token represents a real-world asset like gold or a car, put it in `3`. For other crypto-currencies like BTC, also redeemable at a gateway, put it in `2`.

#### Market IDs
```
1 = ETX/ETH
```
New market IDs will be created as DAO creators add their subcurrency to the exchange.


### Subcurrency API

**Subcurrency contracts need to support the exchange's `deposit` ABI call and implement a `transfer` method for withdrawals.**

It is a different approach than the [MetaCoin API](https://github.com/ethereum/cpp-ethereum/wiki/MetaCoin-API) which will not be used, as the `Approve API` is all-or-nothing and gives too much permissions to the exchange. The new procedure is explained below, and the sample [ETX](https://github.com/etherex/etherex/blob/master/contracts/etx.se) contract can be used as an example.

Each subcurrency has to **notify** the exchange of each asset transfer from a user's address to the exchange's address. The amount of extra code necessary to support this feature is comparable if not smaller than with the other approach.

#### Setting the exchange's address

The first step a subcurrency has to take is to store the exchange's address for future comparison of recipients. This can be done during the initialization of the contract, however, adding an ABI call to update the address and `market_id` is highly recommended.

```
data owner
data exchange
data market_id

def init():
    self.owner = msg.sender
...

def set_exchange(addr, market_id):
    if msg.sender == self.owner:
        self.exchange = addr
        self.market_id = market_id
        return(1)
    return(0)
```

After registering the subcurrency using the `add_market` ABI call, the subcurrency will receive a `market_id`. Since there are currently no return values to actual transactions, this `market_id` will need to be inspected from the exchange's contract storage or from the UI.

#### Notifying the exchange of deposits

The second step has to be executed on each asset transfer. The gas costs of comparing the recipient to the exchange's address are minimal but a separate ABI call might be used later on, depending on how this approach will play out on the testnet. The relevant part below is the one under `# Notify exchange of deposit`, the top part being what can be considered standard subcurrency functionality. Notice the `extern` definition that will be used for the `deposit` method.

```
data balances[2^160](balance)
extern exchange: [deposit:[int256,int256,int256]:int256]

def transfer(recipient, amount):
    # Prevent negative send from stealing funds
    if recipient <= 0 or amount <= 0:
        return(0)

    # Get user balance
    balance = self.balances[msg.sender].balance

    # Make sure balance is above or equal to amount
    if balance >= amount:

        # Update balances
        self.balances[msg.sender].balance = balance - amount
        self.balances[recipient].balance += amount

        # Notify exchange of deposit
        if recipient == self.exchange:
            ret = self.exchange.deposit(msg.sender, amount, self.market_id)
            # Exchange returns our new balance as confirmation
            if ret >= amount:
                return(1)
            # We return 2 as error code for notification failure
            return(2)

        return(1)
    return(0)

```

**TODO**: Solidity examples.

#### Withdrawal support

If your subcurrency's default method for transferring funds is also named `transfer` like the example above, with `recipient` and `amount` parameters (in that order), then there is nothing else you need to do to support withdrawals from EtherEx to a user's address. Otherwise, you'll need to implement that same `transfer` method with those two parameters, and "translate" that method call to yours, calling your other method with those parameters, in the order they're expected. You may also have to use `tx.origin` instead of `msg.sender` in your method as the latter will return your contract's address.

```
def transfer(recipient, amount):
    return(self.invertedtransfer(amount, recipient))
```

#### Balance

Subcurrency contracts also need to implement a `balance` method for the UI to display the user's balance in that contract (also called the subcurrency's wallet).

```
def balance(address):
    return(self.balances[address].balance)
```


Notes
-----
* Your Ethereum address is used as your identity


TODO
----

### Architecture

* Document error codes of return values
* Implement Wallet section (transactions, balances, etc.) (in progress)
* Re-implement NameReg support and integration
* Start the Tools section, find and list ideas
    - subcurrency registration (in progress)
    - subcurrency creation tools/wizard
    - raw transact (?)
    - trading tools (...)
    - ...
* Use [NatSpec](https://github.com/ethereum/wiki/wiki/Ethereum-Natural-Specification-Format)
* Look into how [Whisper](https://github.com/ethereum/wiki/wiki/Whisper-Overview) and [Swarm](https://github.com/ethereum/cpp-ethereum/wiki/Swarm) could be used and integrated
* Start working on X-Chain
* Update this TODO more frequently
* Start using GitHub issues instead
* Better, rock solid tests, and way more of them
* Total unilateral world takeover

### UX/UI

* Graphs, beautiful graphs
* Advanced trading features (stoploss, etc.)
* Animations/transitions
* Check/clear buttons
* Wallet design and theming
* More/new mockups / wireframes
* More/new design elements
* Implement new mockups / design elements


## License

Released under the MIT License, see LICENSE file.
