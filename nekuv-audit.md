---
title: Sample audit
date: 2024-02-16
---


### [H-01] Tokens with Fee-on-Transfer Mechanism Can Break the Protocol, Locking Funds Indefinitely

**Description**

One of the tokens intended for collateral within the protocol, `PAXG`, utilizes a fee-on-transfer mechanism. However, this mechanism isn't compatible with the ERC20 logic in [`LiquidationPool`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L13). Within the [`LiquidationPool::distributeAssets()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L205) method, data is appended to the [`LiquidationPool::rewards`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L22) mapping to track the holder's rewards portion. However, when this portion is transferred to the [`LiquidationPool`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L13) on [line 232](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L232), it leads to the pool balance being lower than the combined rewards portions stored in the mapping.
```javascript
function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable {
    // ** code **
@>  rewards[abi.encodePacked(_position.holder, asset.token.symbol)] += _portion; // @audit adding portion without the transfer fee
    burnEuros += costInEuros;
    if (asset.token.addr == address(0)) {
        nativePurchased += _portion;
    } else {
@>      IERC20(asset.token.addr).safeTransferFrom(manager, address(this), _portion); // @audit fee is deducted here
    }
    // ** code **
}
```
This discrepancy would cause a revert when the last holder attempts to claim their rewards using the [`LiquidationPool::claimRewards()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L164) method due to insufficient funds in the [`LiquidationPool`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L13). Notably, the earlier holders who have already claimed their rewards have implicitly compensated for their deducted transfer fees from the last holder's portion. This imbalance leads to the vulnerable aspect of this function, pinpointed specifically in [line 175](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L175) of the[`LiquidationPool::claimRewards()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L164C14-L164C26) function.
```javascript
function claimRewards() external {
    ITokenManager.Token[] memory _tokens = ITokenManager(tokenManager).getAcceptedTokens();
    for (uint256 i = 0; i < _tokens.length; i++) {
        ITokenManager.Token memory _token = _tokens[i];
        uint256 _rewardAmount = rewards[abi.encodePacked(msg.sender, _token.symbol)];
        if (_rewardAmount > 0) {
            delete rewards[abi.encodePacked(msg.sender, _token.symbol)];
            if (_token.addr == address(0)) {
                (bool _sent,) = payable(msg.sender).call{value: _rewardAmount}("");
                require(_sent);
            } else {
@>              IERC20(_token.addr).transfer(msg.sender, _rewardAmount); // @audit this would revert with "insufficient balance"
            }   
        }

    }
}
```
**Impact**

This vulnerability poses a high risk as it affects `PAXG`, a proposed collateral asset within the protocol. Failure to support such tokens could lead to user funds being inaccessible. Additionally, the protocol intends to utilize other ERC20 tokens in the future, which could potentially encounter the same issue. For instance, `USDT` possesses a similar fee-on-transfer mechanism that is currently deactivated.

**Proof of Concept**

A code demonstration using Foundry to exhibit the inability of a holder to claim rewards can be accessed [here](https://gist.github.com/DanailYordanov/027547fb932dfc18db46df00cafaaa2c).

**Recommended Mitigation**

To address such tokens, a suggested approach involves caching the [`LiquidationPool`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L13) balance before executing a `transferFrom` to the contract. Subsequently, after the transfer, verifying the difference between the cached and current balances as the newly added balance.

**Tools Used**

Manual review