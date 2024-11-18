---
title: T-Swap Audit Report
author: Trashpirate.io
date: November 18, 2024
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries T-Swap Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape Trashpirate.io\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [Trashpirate](https://trashpirate.io)  
Lead Auditors:  
- Nadina Oates

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
- [High](#high)
- [Medium](#medium)
- [Low](#low)
- [Informational](#informational)
- [Gas](#gas)

# Protocol Summary

This project is meant to be a permissionless way for users to swap assets between each other at a fair price. You can think of T-Swap as a decentralized asset/token exchange (DEX). 
T-Swap is known as an [Automated Market Maker (AMM)](https://chain.link/education-hub/what-is-an-automated-market-maker-amm) because it doesn't use a normal "order book" style exchange, instead it uses "Pools" of an asset. 
It is similar to Uniswap. To understand Uniswap, please watch this video: [Uniswap Explained](https://www.youtube.com/watch?v=DLu35sIqVTM)

# Disclaimer

The TRASHPIRATE team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 
Commit Hash: e643a8d4c2c802490976b538dd009b351b1c8dda  
- Solc Version: 0.8.20  
- Chain(s) to deploy contract to: Ethereum  
- Tokens: Any ERC20 token  

## Scope 
```
./src/
#-- PoolFactory.sol
#-- TSwapPool.sol
```

## Roles  
- Liquidity Providers: Users who have liquidity deposited into the pools. Their shares are represented by the LP ERC20 tokens. They gain a 0.3% fee every time a swap is made. 
- Users: Users who want to swap tokens.

# Executive Summary
This audit report was prepared as part of a security tutorial created by Patrick Collins (Cyfrin Updraft). 

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 4                      |
| Medium   | 2                      |
| Low      | 2                      |
| Info     | 9                      |
| Gas      | 0                      |
| Total    | 16                     |

# Findings

## High

<!-- ## H-1 -->
### [H-1] Wrong fee precision in the function `TSwapPool::getInputAmountBasedOnOutput` results in 10 times inflated fee amount

**Description**  
The precision value to calculate the fee amount the function `TSwapPool::getInputAmountBasedOnOutput`is `10000` instead of `1000`. This results in the fee amount being inflated by a factor of 10 compared to the fee taken in `TSwapPool::getOutputAmountBasedOnInput`.

<details><summary>1 Found Instances</summary>

- Found in `src/TSwapPool.sol` [Line: 294](src/TSwapPool.sol#L294)

	```solidity
	function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
        return
	@>      ((inputReserves * outputAmount) * 10000) /
            ((outputReserves - outputAmount) * 997);
    }
	```

</details>


**Impact**  
The function `TSwapPool::getInputAmountBasedOnOutput` subtracts the wrong fee amount.

**Proof of Concepts**  

The function `TSwapPool::getInputAmountBasedOnOutput` returns 10 times the amount than the correct formula `inputAmount = ((inputReserves * outputAmount) * 1000) / ((outputReserves - outputAmount) * 997)`.

<details><summary>Code</summary>

Place following code into `TSwapPool.t.sol`:

    ```solidity
	function testGetInputAmountBasedOnOutput() public {
        uint256 inputReserves = 100e18;
        uint256 outputReserves = 100e18;
        uint256 outputAmount = 10e18;

        uint256 expectedInputAmount = ((inputReserves * outputAmount) * 1000) / ((outputReserves - outputAmount) * 997);
        uint256 actualInputAmount = pool.getInputAmountBasedOnOutput(outputAmount, inputReserves, outputReserves);

        assertEq(expectedInputAmount, actualInputAmount);
    }
	```

<details>

**Recommended mitigation**  

Change the precision value to `1000` in the function `TSwapPool::getInputAmountBasedOnOutput`.

```diff
	function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
+       return ((inputReserves * outputAmount) * 1000) / ((outputReserves - outputAmount) * 997);
-       return ((inputReserves * outputAmount) * 10000) / ((outputReserves - outputAmount) * 997);
    }
```

<!-- #end -->

<!-- ## H-2 -->
### [H-2] Missing input parameter `minInputAmount` in the function `TSwapPool::swapExactOutput` can lead to front-running attacks

**Description**  
The function `TSwapPool::swapExactOutput` does not include the input parameter `maxInputAmount` which can lead to front-running attacks. The `maxInputAmount` parameter is used to specify the maximum amount of input tokens that the user is willing to swap. If the `maxInputAmount` is not specified, the user can be front-run by another user or a malicious actor and result in more input tokens than expected (user sells token at a lower price than expected).

<details><summary>1 Found Instances</summary>

- Found in `src/TSwapPool.sol` [Line: 352](src/TSwapPool.sol#L352)

	```solidity
	function swapExactOutput(
        IERC20 inputToken,
        IERC20 outputToken,
        uint256 outputAmount,
        uint64 deadline
    )
        public
        revertIfZero(outputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 inputAmount)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        inputAmount = getInputAmountBasedOnOutput(
	@>
            outputAmount,
            inputReserves,
            outputReserves
        );

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
	```

</details>

**Impact**  
User can be front-run by another user and result less input tokens than expected.

**Proof of Concepts**  

1. User wants to swap 11 pool tokens for 10 weth tokens
2. Malicious actor manipulates the price of the token
3. User receives pays 18 tokens instead of 11

<details><summary>Code</summary>

Place following code into `TSwapPool.t.sol`:

    ```solidity
	function testSwapExactOutput() public {
        uint256 outputAmount = 10e18;

        // liquidity provider deposits
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        // set up user
        poolToken.mint(user, 200e18);

        uint256 initialPoolTokenBalance = poolToken.balanceOf(user);
        uint256 initialWethBalance = weth.balanceOf(user);

        // user does regular swap
        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);
        pool.swapExactOutput(poolToken, weth, outputAmount, uint64(block.timestamp));
        vm.stopPrank();

        uint256 finalPoolTokenBalance = poolToken.balanceOf(user);
        uint256 tokenDifference = initialPoolTokenBalance - finalPoolTokenBalance;
        console.log("PoolToken Difference: ", tokenDifference);

        address malicousUser = makeAddr("malicousUser");
        poolToken.mint(malicousUser, 200e18);

        // malicous user does swap just before user
        vm.startPrank(malicousUser);
        poolToken.approve(address(pool), type(uint256).max);
        pool.swapExactOutput(poolToken, weth, outputAmount, uint64(block.timestamp));
        vm.stopPrank();

        // user does swap
        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);
        pool.swapExactOutput(poolToken, weth, outputAmount, uint64(block.timestamp));
        vm.stopPrank();

        uint256 finalPoolTokenBalanceAfterMEV = poolToken.balanceOf(user);
        uint256 tokenDifferenceAfterMEV = finalPoolTokenBalance - finalPoolTokenBalanceAfterMEV;

        assertGt(tokenDifferenceAfterMEV, tokenDifference);
        console.log("PoolToken Difference After MEV: ", tokenDifferenceAfterMEV);
    }
	```

<details>

**Recommended mitigation**  
Add imput parameter `maxInputAmount` to the function `TSwapPool::swapExactOutput` so the function reverts if a specified token amount (`maxInputAmount`) is exceeded.

```diff
function swapExactOutput(
        IERC20 inputToken,
+   	uint256 maxInputAmount,
        IERC20 outputToken,
        uint256 outputAmount,
        uint64 deadline
    )
        public
        revertIfZero(outputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 inputAmount)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        inputAmount = getInputAmountBasedOnOutput(
            outputAmount,
            inputReserves,
            outputReserves
        );
+		if(inputAmount > maxInputAmount) {
+			revert revert();
+		}
        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```


<!-- #end -->

<!-- ## H-3 -->
### [H-3] False function call in `TSwapPool::sellPoolTokens` function leads to wrong output amount

**Description**  
Instead of calling the function `swapExactOutput` in the function `TSwapPool::sellPoolTokens`, the function `swapExactInput` is called. This results in the wrong output amount being calculated and returned.


<details><summary>1 Found Instances</summary>

- Found in `src/TSwapPool.sol` [Line: 369](src/TSwapPool.sol#L369)

	```solidity
	 function sellPoolTokens(
        uint256 poolTokenAmount
    ) external returns (uint256 wethAmount) {
        return
    @>      swapExactOutput(
                i_poolToken,
                i_wethToken,
                poolTokenAmount,
                uint64(block.timestamp)
            );
    }

	```

</details>


**Impact**  
Swap logic of the function `TSwapPool::sellPoolTokens` is incorrect and returns the wrong output amount.

**Proof of Concepts**  
1. user is using `TSwapPool::swapExactInput` using the exact `tokenAmount` they want to sell
2. user receives 9.066 WETH tokens and sells 10.00PoolTokens
3. user is using `TSwapPool::sellPoolTokens` using the exact `tokenAmount` they want to sell
4. user receives 10 WETH tokens and sells 13.63 PoolTokens


<details><summary>Code</summary>

Place following code into `TSwapPool.t.sol`:

    ```solidity
	function testSellPoolToken() public {
        uint256 tokenAmount = 10e18;
        uint256 expectedWeth = 9e18;

        // liquidity provider deposits
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        // set up user
        poolToken.mint(user, 200e18);
        vm.prank(user);
        poolToken.approve(address(pool), type(uint256).max);

        uint256 initialPoolTokenBalance = poolToken.balanceOf(user);
        uint256 initialWethBalance = weth.balanceOf(user);

        // user does regular swap using the swapExactInput function
        vm.prank(user);
        pool.swapExactInput(poolToken, tokenAmount, weth, expectedWeth, uint64(block.timestamp));

        uint256 intermediatePoolTokenBalance = poolToken.balanceOf(user);
        uint256 intermediateWethBalance = weth.balanceOf(user);
        uint256 tokensSold1 = initialPoolTokenBalance - intermediatePoolTokenBalance;
        uint256 wethReceived1 = intermediateWethBalance - initialWethBalance;
        // tokensSold1 = 10.000000000000000000 PoolToken
        // wethReceived1 = 9.066108938801491315 WETH

        // user swap with helper function
        vm.prank(user);
        pool.sellPoolTokens(tokenAmount);
        uint256 tokensSold2 = intermediatePoolTokenBalance - poolToken.balanceOf(user);
        uint256 wethReceived2 = weth.balanceOf(user) - intermediateWethBalance;
        // tokensSold2 = 13.632236326745931084 PoolToken
        // wethReceived2 = 10.000000000000000000 WETH

        assertEq(tokensSold1, tokenAmount);
        assertEq(tokensSold2, tokenAmount);
        assertEq(wethReceived1, wethReceived2);
	}

	```

<details>

**Recommended mitigation**  
Replace the function call `swapExactOutput` with `swapExactInput` in the function `TSwapPool::sellPoolTokens`. A slippage parameter such as `minOutputAmount` should be added to the function signature.

```diff
	function sellPoolTokens(
	uint256 poolTokenAmount,
+	uint256 minOutputAmount
	) external returns (uint256 wethAmount) {
-       return swapExactOutput(i_poolToken, i_wethToken, poolTokenAmount, uint64(block.timestamp));
+		return swapExactInput(i_poolToken, poolTokenAmount, i_wethToken, minOutputAmount, uint64(block.timestamp));
   }

```

<!-- #end -->

<!-- ## H-4 -->
### [H-4] In `TSwapPool::_swap` the extra tokens given to users after every `swapCount` breaks the protocol invariant of `x * y = k`

**Description**  
The protocol follows a strict invariant of `x * y = k`. Where:
- `x` is the amount of pool token
- `y` is the amount of WETH
- `k` is the constant product value of the two balances

This means whenever the balances change in the protocol, the ratio between the two amounts should remain constant. However, this is broken due to the extra incentive in the `_swap` function. Meaning that over time the protocol funds will be drained.

<details><summary>1 Found Instances</summary>

- Found in `src/TSwapPool.sol` [Line: 400](src/TSwapPool.sol#L400)

	```solidity
		function _swap(IERC20 inputToken, uint256 inputAmount, IERC20 outputToken, uint256 outputAmount) private {
			if (_isUnknown(inputToken) || _isUnknown(outputToken) || inputToken == outputToken) {
				revert TSwapPool__InvalidToken();
			}

			swap_count++;
	@>		if (swap_count >= SWAP_COUNT_MAX) {
	@>			swap_count = 0;
	@>			outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
	@>		}
			emit Swap(msg.sender, inputToken, inputAmount, outputToken, outputAmount);

			inputToken.safeTransferFrom(msg.sender, address(this), inputAmount);
			outputToken.safeTransfer(msg.sender, outputAmount);
		}
	```

</details>


**Impact**  
A user could maliciously drain the protocol funds by doing many swaps of low amounts and collecting the extra incentive given out by the protocol. This means the protocol's core invariant is broken.

**Proof of Concepts**  
User swaps 10 times a small amount of WETH and the protocol's invariant is broken.

<details><summary>Code</summary>

Place following code into `TSwapPool.t.sol`:

    ```solidity
	function testInvariantBroken() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        vm.startPrank(user);
        poolToken.approve(address(pool), 10e18);

        uint256 outputWeth = 10e15;
        int256 startingY = int256(weth.balanceOf(address(pool)));
        int256 expectedDeltaY = int256(-1) * int256(outputWeth);

        uint256 numberOfSwaps = 10;
        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);
        for (uint256 index = 0; index < numberOfSwaps; index++) {
            pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        }
        vm.stopPrank();

        uint256 endingY = weth.balanceOf(address(pool));
        int256 actualDeltaY = int256(endingY) - int256(startingY);

        assertEq(actualDeltaY, expectedDeltaY);
    }
	```

<details>

**Recommended mitigation**  
Several options are available to mitigate this issue:
1. Remove the extra incentive
2. Account for change in the `x * y = k` protocol invariant
3. Process extra incentive the same way as the protocol fees


```diff
	function _swap(IERC20 inputToken, uint256 inputAmount, IERC20 outputToken, uint256 outputAmount) private {
		if (_isUnknown(inputToken) || _isUnknown(outputToken) || inputToken == outputToken) {
			revert TSwapPool__InvalidToken();
		}

		swap_count++;
-		if (swap_count >= SWAP_COUNT_MAX) {
-			swap_count = 0;
-			outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
-		}
		emit Swap(msg.sender, inputToken, inputAmount, outputToken, outputAmount);

		inputToken.safeTransferFrom(msg.sender, address(this), inputAmount);
		outputToken.safeTransfer(msg.sender, outputAmount);
	}
```

<!-- #end -->

## Medium

<!-- ## M-1 -->
### [M-1] `TSwapPool::deposit` is missing deadline check causing transactions to complete even after deadline

**Description**  
The `deposit` functino accepts a deadline parameter, which according to the documentation is `The deadline for the transaction to be completed by`. However, this parameter is never used. As a consequence, operations that add liqudiity to the pool might be executed at unexpected times, in market conditions where the deposit rate is unfavorable.

<details><summary>1 Found Instances</summary>

- Found in `src/TSwapPool.sol` [Line: 117](src/TSwapPool.sol#L117)

	```solidity
	 function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
    @>  uint64 deadline
    )
        external
        revertIfZero(wethToDeposit)
        returns (uint256 liquidityTokensToMint)
    {}
	```

</details>


**Impact**  
Transactions could be sent when market conditions are unfavorable to deposit, even when adding a deadline parameter.

**Proof of Concepts**  
The `deadline` parameter is unused.


<details><summary>Compiler Output</summary>

Following compiler warning indicates unused `deadline` parameter:

    ```bash
	Warning (5667): Unused function parameter. Remove or comment out the variable name to silence this warning.
	--> src/TSwapPool.sol:99:9:
	|
	99 |         uint64 deadline
	|         ^^^^^^^^^^^^^^^
	```

<details>

**Recommended mitigation**  
Consider following change to the function:

```diff
	function deposit(
		uint256 wethToDeposit,
		uint256 minimumLiquidityTokensToMint,
		uint256 maximumPoolTokensToDeposit,
	@>  uint64 deadline
	)
		external
+		revertIfDeadlinePassed(deadline);
		revertIfZero(wethToDeposit)
		returns (uint256 liquidityTokensToMint)
	{	}
```

<!-- #end -->

<!-- ## M-2 -->
### [M-2] Rebase, fee-on-transfer, and ERC777 tokens break protocol invariant

**Description**  
The protocol follows a strict invariant of `x * y = k`. Where:
- `x` is the amount of pool token
- `y` is the amount of WETH
- `k` is the constant product value of the two balances

This means whenever the balances change in the protocol, the ratio between the two amounts should remain constant. However, this is broken when the token amount is manipulated on transfer as it is the case for ERC20 tokens that have transfer/swap fees because it is not accounted for in the `TSwapPool::_swap` function.

<details><summary>1 Found Instances</summary>

- Found in `src/TSwapPool.sol` [Line: 385](src/TSwapPool.sol#L385)

	```solidity
	 function _swap(...){...}
	```

</details>

**Impact**  
A user could maliciously drain the protocol funds by doing many swaps with a poorly designed ERC20 token. This means the protocol's core invariant is broken.

**Proof of Concepts**  

User swaps an ERC20 with fees on transfer multiple times and breaks the protocol invariant.

<details><summary>Code</summary>

Place following code into `TSwapPool.t.sol`:

    ```solidity
	function testStrangeERC20() public {
        vm.startPrank(liquidityProvider);
        ERC20FeeOnTransferMock strangeToken = new ERC20FeeOnTransferMock();
        TSwapPool strangePool = new TSwapPool(address(strangeToken), address(weth), "LTokenA", "LA");
        weth.approve(address(strangePool), 100e18);
        strangeToken.approve(address(strangePool), 100e18);
        strangePool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        strangeToken.transfer(user, 100e18);
        vm.stopPrank();

        vm.startPrank(user);
        strangeToken.approve(address(strangePool), 10e18);

        uint256 outputWeth = 10e15;
        int256 startingY = int256(weth.balanceOf(address(strangePool)));
        int256 expectedDeltaY = int256(-1) * int256(outputWeth);

        uint256 numberOfSwaps = 2;
        vm.startPrank(user);
        strangeToken.approve(address(strangePool), type(uint256).max);
        for (uint256 index = 0; index < numberOfSwaps; index++) {
            strangePool.swapExactOutput(strangeToken, weth, outputWeth, uint64(block.timestamp));
        }

        vm.stopPrank();

        uint256 endingY = weth.balanceOf(address(strangePool));
        int256 actualDeltaY = int256(endingY) - int256(startingY);

        assertEq(actualDeltaY, expectedDeltaY);
    }
	```

<details>

**Recommended mitigation**  
1. Prevent ERC20 tokens with fees, rebase, or ERC777
2. Account for change in the `x * y = k` protocol invariant

<!-- #end -->

## Low

<!-- ## L-1 -->`
### [L-1] Event parameters in `TSwapPool::_addLiquidityMintAndTransfer` function are incorrect resulting in wrong event logs

**Description**  
The event definition `TSwapPool::LiquidityAdded` indicates that the first parameter is the `liquidityProvider` address, the second parameter is the `wethDeposited` amount, and the third parameter is the `poolTokensDeposited` amount. However, the event is emitted with the parameters in the wrong order - `poolTokenDeposit` in second, and `wethDeposit` in third position. This results in the event logs being incorrect.

```solidity
event LiquidityAdded(
        address indexed liquidityProvider,
        uint256 wethDeposited,
        uint256 poolTokensDeposited
    );
```

<details><summary>1 Found Instances</summary>

- Found in src/TSwapPool.sol [Line: 196](src/TSwapPool.sol#L196)

```solidity
    emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
```

</details>

**Recommended mitigation**
Swap event paramters in second and third position.

```diff
+	emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);
-	emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
```
<!-- #end -->

<!-- ## L-2 -->
### [L-2] Unused return value in `TSwapPool::swapExactInput` function is unused and should be removed

**Description**  
The return value of the `TSwapPool::swapExactInput` function is not used and will always return zero. This could cause confusion as a value is expected based on the function defintion.

<details><summary>1 Found Instances</summary>

- Found in `src/TSwapPool.sol` [Line: 308](src/TSwapPool.sol#L308)

	```solidity
	function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
        public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (
    @>      uint256 output
        )
    {}
	```

</details>

**Impact**  
The return value of the `TSwapPool::swapExactInput` function is not used and will always return zero possibly causing confusion or disrupt functionality that depends on the return value.

**Proof of Concepts**  

1. Liquidity is provided
2. User swaps tokens
3. The function `TSwapPool::swapExactInput` always returns 0 regardless of the input value

<details><summary>Code</summary>

Place following code into `TSwapPool.t.sol`:

    ```solidity
	function testSwapExactInputAlwaysReturnsZero(uint256 tokenAmount) public {
        tokenAmount = bound(tokenAmount, 1, 100e18);
        uint256 wethAmount = 100e18;

        // set up liquidity pool
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), type(uint256).max);
        poolToken.approve(address(pool), type(uint256).max);
        pool.deposit(wethAmount, wethAmount, 2 * tokenAmount, uint64(block.timestamp));
        vm.stopPrank();

        // set up user
        poolToken.mint(user, tokenAmount);
        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);

        // swap
        uint256 output = pool.swapExactInput(poolToken, tokenAmount, weth, 0, uint64(block.timestamp));
        assertEq(output, 0);
    }
	```

<details>

**Recommended mitigation**  
There are two options to mitigate this issue:

1. Remove the return value from the function definition.
2. Rename the return value from `output` to `outputAmount`.

```diff
	function swapExactInput(
		IERC20 inputToken,
		uint256 inputAmount,
		IERC20 outputToken,
		uint256 minOutputAmount,
		uint64 deadline
	)
		public
		revertIfZero(inputAmount)
		revertIfDeadlinePassed(deadline)
		returns (
-     		uint256 output
+     		uint256 outputAmount
		)
	{}
```

<!-- #end -->

## Informational

<!-- ## I-1 -->
### [I-1] `PoolFactory__PoolDoesNotExist` is not used and should be removed

```diff
- error PoolFactory__PoolDoesNotExist(address tokenAddress);
```
<!-- #end -->

<!-- ## I-2 -->
### [I-2] Wrong naming of `liquidityTokenSymbol` in `PoolFactory::createPool` can lead to confusion and illegibility

**Description**  

For the naming of the `liquidityTokenSymbol` in the `PoolFactory::createPool` function, the `IERC20(tokenAddress).name()` is used instead of `IERC20(tokenAddress).symbol()`. This can lead to confusion and illegibility of the codebase.

```diff
function createPool(address tokenAddress) external returns (address) {
        if (s_pools[tokenAddress] != address(0)) {
            revert PoolFactory__PoolAlreadyExists(tokenAddress);
        }
        string memory liquidityTokenName = string.concat("T-Swap ", IERC20(tokenAddress).name());
+       string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol()); 
-       string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
        TSwapPool tPool = new TSwapPool(tokenAddress, i_wethToken, liquidityTokenName, liquidityTokenSymbol);
        s_pools[tokenAddress] = address(tPool);

        s_tokens[address(tPool)] = tokenAddress;
        emit PoolCreated(tokenAddress, address(tPool));
        return address(tPool);
    }
```
<!-- #end -->

<!-- ## I-3 -->
### [I-3] Event is missing `indexed` fields


**Description**  

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

<details><summary>4 Found Instances</summary>


- Found in src/PoolFactory.sol [Line: 35](src/PoolFactory.sol#L35)

	```solidity
	    event PoolCreated(address tokenAddress, address poolAddress);
	```

- Found in src/TSwapPool.sol [Line: 52](src/TSwapPool.sol#L52)

	```solidity
	    event LiquidityAdded(
	```

- Found in src/TSwapPool.sol [Line: 57](src/TSwapPool.sol#L57)

	```solidity
	    event LiquidityRemoved(
	```

- Found in src/TSwapPool.sol [Line: 62](src/TSwapPool.sol#L62)

	```solidity
	    event Swap(
	```

</details>
<!-- #end -->

<!-- ## I-4 -->
### [I-4] Missing zero checks can lead to false initialzation of immutable variables in constructor


**Description**  

When initializing immutable address variables in the constructor it is recommended to check for zero address to avoid false initialization that cannot be later.

<details><summary>2 Found Instances</summary>


- Found in src/PoolFactory.sol [Line: 41](src/PoolFactory.sol#L41)

	```solidity
	    i_wethToken = wethToken;
	```

- Found in src/TSwapPool.sol [Line: 96](src/TSwapPool.sol#L96)

	```solidity
	    i_wethToken = IERC20(wethToken);
        i_poolToken = IERC20(poolToken);
	```

</details>

**Recommended mitigation**  

Add zero address check to the `PoolFactory::constructor` and `TSwapPool::constructor` functions.

Example:
```diff
    constructor(address wethToken) {
+       if(wethToken == address(0)) {
+           revert PoolFactory__ZeroAddress(wethToken);
+       }
        i_wethToken = IERC20(wethToken);
    }
```

<!-- #end -->

<!-- ## I-5 -->
### [I-5] `public` functions not used internally could be marked `external`

Instead of marking a function as `public`, consider marking it as `external` if it is not used internally.

<details><summary>1 Found Instances</summary>


- Found in src/TSwapPool.sol [Line: 298](src/TSwapPool.sol#L298)

	```solidity
	    function swapExactInput(
	```

</details>

<!-- #end -->

<!-- ## I-6 -->
### [I-6]  Define and use `constant` variables instead of using literals

If the same constant literal value is used multiple times, create a constant state variable and reference it throughout the contract.

<details><summary>4 Found Instances</summary>


- Found in src/TSwapPool.sol [Line: 276](src/TSwapPool.sol#L276)

	```solidity
	        uint256 inputAmountMinusFee = inputAmount * 997;
	```

- Found in src/TSwapPool.sol [Line: 295](src/TSwapPool.sol#L295)

	```solidity
	            ((outputReserves - outputAmount) * 997);
	```

- Found in src/TSwapPool.sol [Line: 454](src/TSwapPool.sol#L454)

	```solidity
	                1e18,
	```

- Found in src/TSwapPool.sol [Line: 463](src/TSwapPool.sol#L463)

	```solidity
	                1e18,
	```

</details> 

<!-- #end -->

<!-- ## I-7 -->
### [I-7] State variable `TSwapPool::poolTokenReserves` is not used and should be removed

<details><summary>1 Found Instances</summary>

- Found in src/TSwapPool.sol [Line: 131](src/TSwapPool.sol#L131)

	```solidity
	    uint256 poolTokenReserves = i_poolToken.balanceOf(address(this));
	```

</details>
<!-- #end -->

<!-- ## I-8 -->
### [I-8] Error parameter `TSwapPool::MINIMUM_WETH_LIQUIDITY` is constant and can be removed

<details><summary>1 Found Instances</summary>

- Found in src/TSwapPool.sol [Line: 125](src/TSwapPool.sol#L125)

```diff
    revert TSwapPool__WethDepositAmountTooLow(
-               MINIMUM_WETH_LIQUIDITY,
                wethToDeposit
            );
```

</details>
<!-- #end -->

<!-- ## I-9 -->
### [I-9] The function `TSwapPool::_addLiquidityMintAndTransfer` contains external calls and therefor should be used in CEI (Check-Effects-Interactions) pattern

<details><summary>1 Found Instances</summary>

- Found in src/TSwapPool.sol [Line: 177](src/TSwapPool.sol#L177)

```diff
    else {
            // This will be the "initial" funding of the protocol. We are starting from blank here!
            // We just have them send the tokens in, and we mint liquidity tokens based on the weth
+           liquidityTokensToMint = wethToDeposit;
            _addLiquidityMintAndTransfer(wethToDeposit, maximumPoolTokensToDeposit, wethToDeposit);
-           liquidityTokensToMint = wethToDeposit;
        }
```

</details>
<!-- #end -->

