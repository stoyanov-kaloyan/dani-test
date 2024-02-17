---
title: fort sos
---

### [M-01] Error in Accumulating Holder's Fees on Staked EUROs, Causing Misrepresentation of Funds

**Description**

The [`LiquidationPool::position()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L83) function provides a snapshot of staked `EUROs` and `TST` along with accumulated fees and rewards. However, a flaw exists in the calculation process within this function. Specifically, at [line 88](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L88), the accrued fees in `EUROs` for holders based on their staked `TST` are miscalculated. The issue arises from considering the entire balance of the [`LiquidationPoolManager`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L11) rather than deducting the [`LiquidationPoolManager::poolFeePercentage`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L20) and allocating it to the protocol.

```javascript
    function position(address _holder) external view returns(Position memory _position, Reward[] memory _rewards) {
        _position = positions[_holder];
        (uint256 _pendingTST, uint256 _pendingEUROs) = holderPendingStakes(_holder);
        _position.EUROs += _pendingEUROs;
        _position.TST += _pendingTST;
@>      if (_position.TST > 0) _position.EUROs += IERC20(EUROs).balanceOf(manager) * _position.TST / getTstTotal(); // @audit not deducting the fee
        _rewards = findRewards(_holder);
    }
```

The correct calculation, outlined in [`LiquidationPoolManager::distributeFees()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L33) at [line 35](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L35), involves deducting the [`LiquidationPoolManager::poolFeePercentage`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L20) from the manager's total balance before distributing fees to the holders by calling [`LiquidationPool::distributeFees()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L182) and then forwaring the remaining to the protocol

```javascript
    function distributeFees() public {
        IERC20 eurosToken = IERC20(EUROs);
@>      uint256 _feesForPool = eurosToken.balanceOf(address(this)) * poolFeePercentage / HUNDRED_PC; // @audit deducting the fee
        if (_feesForPool > 0) {
            eurosToken.approve(pool, _feesForPool);
            LiquidationPool(pool).distributeFees(_feesForPool);
        }
        eurosToken.transfer(protocol, eurosToken.balanceOf(address(this)));
    }
```

The erroneous formula currently used is:

$$\text{holder's EUROs} \mathrel{+}= \frac{\text{EUROs balance of manager} \times \text{holder's TST}}{\text{total TST}}$$

While the accurate formula should be:

$$\text{holder's EUROs} \mathrel{+}= \frac{\text{balance of manager} \times \text{pool fee percentage} \times \text{holder's TST}}{\text{total TST}}$$

**Impact**

The miscalculation in displaying holder's assets can lead to an unfavorable off-chain user experience (UX).

**Proof of Concept**

Proof of Code written with Foundry can be found [here](https://gist.github.com/DanailYordanov/b0218948722e4c14b4f573eeaa439c12). This test shows the difference of funds of a holder when calling [`LiquidationPool::position()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L83) before and after distribution of the fees.

**Recommended Mitigation**

Adjust the [`LiquidationPool::position()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L83) function to incorporate the [`LiquidationPoolManager::poolFeePercentage`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L20) variable obtained from the manager, which will forward the needed funds to the holders and the others to the protocol's multisig. Here's an edited version with the suggested changes:

```diff
    function position(address _holder) external view returns(Position memory _position, Reward[] memory _rewards) {
        _position = positions[_holder];
        (uint256 _pendingTST, uint256 _pendingEUROs) = holderPendingStakes(_holder);
        _position.EUROs += _pendingEUROs;
        _position.TST += _pendingTST;
-        if (_position.TST > 0) _position.EUROs += IERC20(EUROs).balanceOf(manager) * _position.TST / getTstTotal();
+        if (_position.TST > 0) _position.EUROs += (IERC20(EUROs).balanceOf(manager) * poolFeePercentage) * _position.TST / getTstTotal();
        _rewards = findRewards(_holder);
    }
```

**Tools Used**

Manual review
