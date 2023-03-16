# Using the Witnet Celo Price Feed

## Table of Content
- [Using the Witnet Celo Price Feed](#using-the-witnet-celo-price-feed)
  - [Table of Content](#table-of-content)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Requirements](#requirements)
  - [Smart Contract Development](#smart-contract-development)
    - [Imports and Interfaces](#imports-and-interfaces)
    - [Variables and Events](#variables-and-events)
    - [Functions](#functions)
      - [The Constructor](#the-constructor)
      - [The Receive Function](#the-receive-function)
      - [The `updateCeloUsdPriceFeed()` Function](#the-updatecelousdpricefeed-function)
      - [The `getCeloUsdPrice()` Function](#the-getcelousdprice-function)
      - [The `tradeCUsdToCelo()` Function](#the-tradecusdtocelo-function)
      - [The `withdrawCUsd()` Function](#the-withdrawcusd-function)
      - [The `forceCeloUsdUpdate()` Function](#the-forcecelousdupdate-function)
  - [Testing the Smart Contract Using Laika](#testing-the-smart-contract-using-laika)
  - [Conclusion](#conclusion)


## Introduction
In this tutorial, we will use the [Witnet Celo Price Feed](https://docs.witnet.io/smart-contracts/witnet-data-feeds) router to fetch the CELO/USD currency pair that we will use in our smart contract `DExchange` where we will allow users to trade their cUSD(Celo Dollar) tokens with CELO tokens stored in our smart contract.
## Prerequisites
To get the most out of this tutorial, you need to have the following:

- Experience using Solidity.
- Familiarity with the most common smart contract concepts.
- Familiarity with the ERC-20 Token standard and interfaces.
- Experience using the Remix IDE.
- Experience using [Laika](https://medium.com/laika-lab/introduction-to-hardhat-laika-45929073a4a2) for testing your smart contract.
- Understanding of oracles and their use cases.
- Experience using the Celo plugin to deploy smart contracts.

## Requirements

- A web browser
- An internet connection
- The Celo plugin activated in Remix
- The Remix IDE
- The Metamask Extension Wallet
- A Metamask account with [alfajores testnet tokens](https://docs.celo.org/wallet/faucet-testnet#:~:text=Visit%20faucet.celo.org%2C,additional%20CELO%20and%20stable%20tokens.)


## Smart Contract Development
In this section, we will create the `DExchange` smart contract. To get started, open [Remix](https://remix.ethereum.org/) and create a new file called `DExchange.sol`.

### Imports and Interfaces
To use the Witnet Celo router and the CELO/USD price feed smart contract, we will need to make a few imports:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.11;

import "witnet-solidity-bridge/contracts/interfaces/IWitnetPriceRouter.sol";
import "witnet-solidity-bridge/contracts/interfaces/IWitnetPriceFeed.sol";
```

We first defined an SPDX license for our smart contract and the Solidity versions our compiler will be allowed to use. We then imported the `IWitnetPriceRouter` and `IwitnetPriceFeed` interfaces as we will need them in our smart contract to interact with the Witnet Celo Router and to fetch or force an update on the CELO/USD currency pair.

Next, we will also import the ERC-20 interface as we will need it to be able to interact with the cUSD smart contract:

```solidity
interface IERC20Token {
    function transfer(address, uint256) external returns (bool);

    function approve(address, uint256) external returns (bool);

    function transferFrom(address, address, uint256) external returns (bool);

    function totalSupply() external view returns (uint256);

    function balanceOf(address) external view returns (uint256);

    function allowance(address, address) external view returns (uint256);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
}
```

### Variables and Events

We will now work on the variables and events we make use of in our smart contract:

```solidity
contract DExchange {
  IWitnetPriceRouter public immutable witnetPriceRouter;
    IWitnetPriceFeed public celoUsdPrice;

    address public owner;
    address internal cUsdTokenAddress =
        0x874069Fa1Eb16D44d622F2e0Ca25eeA172369bC1;

    event Exchange(address indexed sender, uint celoAmount, uint cUsdAmount);
}
```
We first defined our smart contract with the keyword `contract` followed by the name `DExchange`. Next, we created the following variables:

1. `witnetPriceRouter` - An initialized interface that allows us to interact and call functions of the deployed `WitnetPriceRouter` smart contract
2. `celoUsdPrice` - An initialized interface that allows us to interact and call functions of the deployed CELO/USD price feed smart contract
3. `owner` - An `address` variable that stores the address of the deployer
4. `cUsdTokenAddress` - An `address` variable that stores the address of the cUSD smart contract

We also defined an event called `Exchange` that will be emitted when a user successfully trades his cUSD tokens for CELO using the `tradeCUsdToCelo()` function of our `DExchange` smart contract.


### Functions
In this section, we will explain what each function in our smart contract does.

#### The Constructor

We will now create the constructor of our smart contract:

```solidity
    /**
     * IMPORTANT: use the Celo Alfajores WitnetPriceRouter address here.
     * The link to the address is:
     * https://docs.witnet.io/smart-contracts/witnet-data-feeds/addresses/celo
     */
    constructor(IWitnetPriceRouter _router) {
        witnetPriceRouter = _router;
        updateCeloUsdPriceFeed();
        owner = msg.sender;
    }

```

The constructor is used to initialize the `witnetPriceRouter`, `celoUsdPrice`, and `owner` variables.

>**_Note_**: We initialize the `celoUsdPrice` variable by calling the `updateCeloUsdPriceFeed()` method.

#### The Receive Function

Next, we will create the `receive()` as it will be used for calls made to the contract with empty calldata, such as plain CELO transfers. This function will also play an important role in making sure that the `forceCeloUsdUpdate()` function works properly.

```solidity
    receive() external payable {}
```

#### The `updateCeloUsdPriceFeed()` Function

We will now create the `updateCeloUsdPriceFeed()` function:

```solidity
    /**
     * @notice Detects if the WitnetPriceRouter is now pointing to a different IWitnetPriceFeed implementation
     * @notice Updates the celoUsdPrice with the new IWitnetPriceFeed implementation
     */
    function updateCeloUsdPriceFeed() public {
        IERC165 _newPriceFeed = witnetPriceRouter.getPriceFeed(
            bytes4(0x9ed884be)
        );
        if (address(_newPriceFeed) != address(0)) {
            celoUsdPrice = IWitnetPriceFeed(address(_newPriceFeed));
        }
    }
```

This function will be used to initialize(during deployment) and update the `celoUsdPrice` variable that stores an interface for the `ERC-165` compliant price feed smart contract of the **CELO/ USD** currency pair and this will allow us to use the interface to interact with the price feed smart contract. The `updateCeloUsdPriceFeed()` first fetches and stores the `ERC-165` price feed smart contract by passing the **price pair identifier** to the `getPriceFeed()` method of the Witnet Price Router that is used to fetch the price feed smart contract. Finally, we use an `if` statement to make sure that we only update the `celoUsdPrice` variable with an interface that is pointing to a valid and deployed smart contract.


#### The `getCeloUsdPrice()` Function

Next up, we will create the `getCeloUsdPrice()` function:

```solidity
    /// Returns the CELO / USD price (6 decimals), ultimately provided by the Witnet oracle, and
    /// the timestamps at which the price was reported back from the Witnet oracle's sidechain
    /// to Celo Alfajores.
    function getCeloUsdPrice()
        public
        view
        returns (int256 _lastPrice, uint256 _lastTimestamp)
    {
        (_lastPrice, _lastTimestamp, , ) = celoUsdPrice.lastValue();
    }
```

The function is defined as `public` and `view` since we want to be able to use the function both **internally** and **externally** and only to read the state. It uses the `lastValue()` method of the price feed smart contract stored in `CeloUsdPrice` to fetch and return the following values:

1. `_lastPrice` - Last valid price reported back from the Witnet oracle.
2. `_lastTimestamp` - Last valid price reported back from the Witnet oracle.


#### The `tradeCUsdToCelo()` Function

We will now create the `tradeCUsdToCelo()` function:

```solidity
    /**
     * @notice Allow users to trade cUSD tokens for CELO tokens
     * @dev The amount of cUSD to exchange is defined by the allowance set by the cUSD tokens' owner
     */
    function tradeCUsdToCelo() public payable {
        uint amount = IERC20Token(cUsdTokenAddress).allowance(
            msg.sender,
            address(this)
        );
        require(amount >= 0.1 ether);
        (int _lastPrice, ) = getCeloUsdPrice();
        uint celoAmount = (1 ether * amount) /
            (uint(_lastPrice) * 0.000001 ether);
        require(address(this).balance >= celoAmount);
        require(
            IERC20Token(cUsdTokenAddress).transferFrom(
                msg.sender,
                address(this),
                amount
            ),
            "Transfer failed."
        );

        (bool success, ) = payable(msg.sender).call{value: celoAmount}("");
        require(success, "Transfer failed");
        emit Exchange(msg.sender, celoAmount, amount);
    }
```

The `tradeCUsdToCelo()` function allows users to trade their cUSD tokens for CELO tokens stored in the `DExchange` smart contract. It fetches the cUSD allowance **amount** the sender has approved for the smart contract to "spend" and stores it in a variable called `amount`. It then performs a check on the `amount` variable to be at least one-tenth of a Celo Dollar. If nothing went wrong with the previous check, the function fetches and stores the *latest valid* price of the CELO/USD pair by calling the `getCeloUsdPrice()` we defined earlier in our smart contract.


>**_Note_**: You might have noticed that the value stored inside the `_lastPrice` variable uses six decimals. It follows the same principle of the eighteen decimals used by ERC-20 tokens which is simply to main good precision and accuracy as Solidity doesn't have a `float` type.

We then calculate and store the CELO amount the `sender` will receive in the `celoAmount` variable. The CELO amount is calculated by first multiplying the `amount` variable by `1 ether`, then we convert the `_lastPrice` to a `uint` value as it was previously an `int256` value. We then multiply it by `0.000001 ether` to convert it to 18 decimals digits and finally, we divide the results of both multiplication to get the CELO amount.

>**_Note_**: We multiply the `amount` variable by `1 ether` to ensure the value returned by the division operation is compliant with the CELO decimals which in this case is `18`.


Next, we carry out a check to make sure that the `DExchange` smart contract has enough CELO stored to fulfill the trade and we also carry out a check to make sure that the transfer of cUSD tokens to the `DExchange` smart contract through the `transferFrom()` method is successful.

Finally, we transfer the CELO amount to the sender using the `.call()` method and ensure the transfer is successful. If nothing goes wrong, we emit the `Exchange` event.


#### The `withdrawCUsd()` Function

We will now define the `withdrawCUsd()` function:

```solidity
    // Allows the deployer to withdraw cUSD tokens
    function withdrawCUsd() public payable {
        require(msg.sender == owner);
        uint amount = IERC20Token(cUsdTokenAddress).balanceOf(address(this));
        require(amount > 0, "No cUSD balance to withdraw.");
        require(
            IERC20Token(cUsdTokenAddress).transferFrom(
                address(this),
                msg.sender,
                amount
            ),
            "Transfer failed."
        );
    }
```

The `withdrawCUsd()` function allows the `owner` of the `DExchange` smart contract to withdraw the cUSD tokens stored inside the smart contract. This function essentially checks whether the `sender` of the transaction is the `owner`. If it evaluates to *true*, the function fetches the current cUSD balance of the smart contract and ensures there is a valid amount to withdraw. Finally, it transfers the cUSD tokens using the `transferFrom()` method of the cUSD smart contract.


####  The `forceCeloUsdUpdate()` Function

The last function we will create for our smart contract is the `forceCeloUsdUpdate()` function:

```solidity
    /// Force update on the CELO / USD currency pair
    function forceCeloUsdUpdate() external payable {
        IERC165 _priceFeed = witnetPriceRouter.getPriceFeed(bytes4(0x9ed884be));
        uint _updateFee = IWitnetPriceFeed(address(_priceFeed))
            .estimateUpdateFee(tx.gasprice);
        IWitnetPriceFeed(address(_priceFeed)).requestUpdate{
            value: _updateFee
        }();
        if (msg.value > _updateFee) {
            payable(msg.sender).transfer(msg.value - _updateFee);
        }
    }
```

The `forceCeloUsdUpdate()` function allows users to send a request to the CELO/USD price feed smart contract to update the currency pair's latest valid price. This function fetches the `ERC-165` compliant price feed smart contract of the CELO/USD pair and then calls its `estimateUpdateFee()` method to get the cost amount to create the request. The function then calls the `requestUpdate()` method of the price feed smart contract and passes the `_updateFee` retrieved to pay for the request. Finally, an `if` statement checks if the `msg.value` is greater than the `_updateFee`, and if it is true, the unused CELO sent is sent back to the `sender` of the transaction.


## Testing the Smart Contract Using Laika

**Coming soon**
## Conclusion

In this tutorial, you learned how to implement and interact with the Witnet Celo Router and CELO/USD price feed smart contracts. Throughout the tutorial, we successfully built a simple DEX that allows us to trade cUSD tokens for CELO tokens. 


