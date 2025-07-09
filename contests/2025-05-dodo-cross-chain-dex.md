# DODO Cross-Chain DEX - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## Medium Risk Findings
    - ### [M-01. Passing pre-fee amount to swap logic causes swap failure due to insufficient balance](#M-01)


# <a id='contest-summary'></a>Contest Summary

### Dates: Jun 2nd, 2025 - Jun 9th, 2025

[See more contest details here](https://audits.sherlock.xyz/contests/991?filter=questions)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- Medium: 1

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Passing pre-fee amount to swap logic causes swap failure due to insufficient balance     



## Summary

In `omni-chain-contracts/contracts/GatewayTransferNative.sol`, the `onCall` function passes the full pre-fee amount to `_doMixSwap`, causing ERC20 transfer reverts during the swap, as the actual available balance is reduced after platform fees are transferred to treasury safe in `_handleFeeTransfer`. This results in swap failure, preventing users from completing intended cross-chains actions.

## Root Cause

In `GatewayTransferNative::onCall`, the `amount` and `params.fromTokenAmount` passed to `_doMixSwap` does not reflect the deducted fee. The `_handleFeeTransfer` function correctly transfers the platform fee (`platformFeesForTx`) out of the contract balance, but the full original amount is still forwarded to `_doMixSwap`. Consequently, the called swap logic in the DODO Router contract attempts to pull more tokens than are left in the contract, which causes an ERC20 transfer to revert due to insufficient balance.

In `omni-chain-contracts/contracts/GatewayTransferNative.sol#L393-L408`:

```solidity
} else {
    // Swap on DODO Router
    // @audit Incorrect `amount` and `params` passed
    uint256 outputAmount = _doMixSwap(decoded.swapData, amount, params);

    if (decoded.targetZRC20 == WZETA) {
        // withdraw WZETA to get Zeta in 1:1 ratio
        IWETH9(WZETA).withdraw(outputAmount);
        // transfer wzeta
        TransferHelper.safeTransferETH(receiver, outputAmount);
    } else {
        TransferHelper.safeTransfer(
            decoded.targetZRC20,
            receiver,
            outputAmount
        );
    }
```

In `omni-chain-contracts/contracts/GatewayTransferNative.sol#L370-L371`:
```solidity
// A portion of `amount`, `platformFeesForTx` is transferred to treasury safe here
uint256 platformFeesForTx = _handleFeeTransfer(zrc20, amount); // platformFee = 5 <> 0.5%
address receiver = address(uint160(bytes20(decoded.receiver)));
```

## Internal Pre-conditions

1. `feePercent` is greater than 0
2. `onCall` is called with `targetZRC20` address decoded from `message` is different from `zrc20` address, and `swapData` bytes decoded from `message` is not empty

## External Pre-conditions

N/A

## Attack Path

The attack path demonstrated by the PoC is as follows:
1. A user wants to swap `token1A` from chain A to `token2Z` on ZetaChain, the flow is `token1A (Chain A) -> token1Z (ZetaChain) -> token2Z (ZetaChain)`
2. He prepares the appropriate `swapData` in `payload`, then he calls the `GatewaySend::depositAndCall` on chain A
3. The `onCall` function on `GatewayTransferNative` on ZetaChain will revert due to `ERC20InsufficientBalance` error

## Impact

The user cannot complete their cross-chain swap due to `onCall` transaction revert on `GatewayTransferNative` on ZetaChain. This affects all users interacting with the swap path through `GatewayTransferNative::onCall` function in satisfied internal conditions described above

## PoC

The original setup of the `BaseTest` sets the `GatewayTransferNative::feePercent`, so we will update it to `5` which is `0.5%` in `omni-chain-contracts/test/BaseTest.t.sol::BaseTest::setUp`:

```solidity
// set GatewayTransferNative
data = abi.encodeWithSignature(
    "initialize(address,address,address,address,uint256,uint256,uint256)",
    address(gatewayZEVM),
    EddyTreasurySafe,
    address(dodoRouteProxyZ),
    address(dodoRouteProxyZ), // DODOApprove
    5, // update to 5
    10,
    100000
);
```

Add the following function under `omni-chain-contracts/test/GatewayTransferNative.t.sol::GatewayTransferNativeTest` scope:

```solidity
// A - zetachain swap: token1A -> token1Z -> token2Z
function test_A2ZSwapInsufficientBalanceRevert() public {
    address targetContract = address(gatewayTransferNative);
    uint256 amount = 100 ether;
    uint32 dstChainId = 7000;
    address asset = address(token1A);
    address targetZRC20 = address(token2Z);
    bytes memory sender = abi.encodePacked(user1);
    bytes memory receiver = abi.encodePacked(user2);
    bytes memory swapDataZ = encodeCompressedMixSwapParams(
        address(token1Z),
        address(token2Z),
        amount,
        0,
        0,
        new address[](1),
        new address[](1),
        new address[](1),
        0,
        new bytes[](1),
        abi.encode(address(0), 0),
        block.timestamp + 600
    );
    bytes memory payload = encodeNativeMessage(
        targetZRC20,
        sender,
        receiver,
        swapDataZ
    );

    vm.startPrank(user1);
    token1A.approve(
        address(gatewaySendA),
        amount
    );
    gatewaySendA.depositAndCall(
        targetContract,
        amount,
        asset,
        dstChainId,
        payload
    );
    vm.stopPrank();
}
```

Output:

```solidity
Failing tests:
Encountered 1 failing test in test/GatewayTransferNative.t.sol:GatewayTransferNativeTest
[FAIL: ERC20InsufficientBalance(0x3Cff5E7eBecb676c3Cb602D0ef2d46710b88854E, 99500000000000000000 [9.95e19], 100000000000000000000 [1e20])] test_A2ZSwapInsufficientBalanceRevert() (gas: 436162)
```

## Mitigation

Forward the actual available amount to the swap logic:

```diff
+   amount -= platformFeesForTx;
+   params.fromTokenAmount -= platformFeesForTx;
    uint256 outputAmount = _doMixSwap(decoded.swapData, amount, params);
```



