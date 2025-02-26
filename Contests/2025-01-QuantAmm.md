

| ID                                                                                                                  | Title                                                                                                 |
| ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| [H-01](#h-01-donations-are-sanwichable-to-steal-funds-from-lp)                                                      | Donations are sanwichable to steal funds from LP                                                      |
| [H-02](#h-02-uplift-fees-are-applied-on-depositamount-instead-of-the-uplifted-profit-value)                         | UpLift fees are applied on `depositAmount` instead of the upLifted (profit) value                     |
| [H-03](#h-03-freeze-of-funds-fees-to-quantammadmin-in-onafterremoveliquidity-fees-distribution)                     | freeze of funds (fees) to `QuantAMMAdmin` in `onAfterRemoveLiquidity()` fees distribution             |
| [H-04](#h-04-in-onafterremoveliquidity-maxamountsin-during-admin-fee-distribution-can-cause-txn-reverts)            | In `onAfterRemoveLiquidity()`, `maxAmountsIn` during admin fee distribution can cause txn reverts     |
| [H-05](#h-05-truncation-due-to-bad-order-of-division-causing-higher-fees-sent-to-the-admin)                         | Truncation due to bad order of division causing higher fees sent to the admin                         |
| [H-06](#h-06-improper-access-control-in-onafterremoveliquidity-risks-loss-of-funds-by-attacker)                     | Improper access control in `onAfterRemoveLiquidity()` risks loss of funds by attacker                 |
| [H-07](#h-07-hook-fees-ownerfee-are-stuck-in-upliftonlyexample-and-non-retrievable)                                 | Hook fees (`ownerFee`) are stuck in `UpliftOnlyExample` and non retrievable                           |
| [H-08](#h-08-users-can-avoid-paying-fees-on-up-lifts-by-utilising-afterupdate-logic)                                | Users can avoid paying fees on up lifts by utilising `afterUpdate` Logic                              |
| [M-01](#m-01-last-withdrawer-will-donate-fees-to-empty-pool-allowing-mev-and-having-always-stuck-funds-in-the-pool) | Last withdrawer will donate fees to empty pool allowing MEV and having always stuck funds in the pool |
| [M-02](#m-02-moving-average-length-validation-prevents-admin-override-for-rules-requiring-historical-data)          | Moving Average Length Validation Prevents Admin Override for Rules Requiring Historical Data          |
| [M-03](#m-03-in-upliftonlyexample-immutable-updateweightrunner-address-prevents-seamless-contract-migration)        | In `UpliftOnlyExample`, Immutable UpdateWeightRunner Address Prevents Seamless Contract Migration     |
| [M-04](#m-04-pool-weights-are-non-updatable-after-updating-updateweightrunner-address)                              | Pool Weights are non-updatable After Updating `updateWeightRunner` address                            |
| [M-05](#m-05-users-transferring-their-nft-position-will-retroactively-get-the-new-upliftfeebps)                     | Users transferring their NFT position will retroactively get the new `upliftFeeBps`                   |
| [M-06](#m-06-attacker-can-completely-prevent-users-from-withdrawing-in-upliftonlyexample)                           | Attacker can completely prevent users from withdrawing in `UpliftOnlyExample`                         |
| [M-07](#m-07-division-before-multiplication-in-lptokendepositvaluechange-causes-loss-of-fees)                       | Division before multiplication in `lpTokenDepositValueChange` causes loss of fees                     |
| [M-08](#m-08-single-storage-variable-used-for-both-swap-and-uplift-fees)                                            | Single Storage Variable Used for Both Swap and Uplift Fees                                            |
| [L-01](#l-01-minwithdrawalfeebps-are-not-added-to-upliftfeebps-causing-loss-of-fees-and-allowing-mev-actions)       | `minWithdrawalFeeBps` are not added to `upliftFeeBps` causing loss of fees and allowing MEV actions   |
| [L-02](#l-02-there-is-no-function-to-change-upliftfeebps)                                                           | There is no function to change `upliftFeeBps`                                                         |
| [L-03](#l-03-last-withdrawer-will-donate-fees-to-empty-pool-allowing-mev-and-having-always-stuck-funds-in-the-pool) | Last withdrawer will donate fees to empty pool allowing MEV and having always stuck funds in the pool |

## [H-01] Donations are sanwichable to steal funds from LP
### Summary

  

Large Fees donations are sandwichable in `UpliftOnlyExample` stealing funds from the deserving actual LP providers

  

### Vulnerability Details

  

UpLift fees taken from `removeLiquidityProportional` during `onAfterRemoveLiquidity` are donated to the pool in stepwise matter,

  

```solidity

File: UpliftOnlyExample.sol

555:         if (localData.adminFeePercent != 1e18) {

556:             // Donates accrued fees back to LPs.

557:             _vault.addLiquidity(

558:                 AddLiquidityParams({

559:                     pool: localData.pool,

560:                     to: msg.sender, // It would mint BPTs to router, but it's a donation so no BPT is minted

561:                     maxAmountsIn: localData.accruedFees, // Donate all accrued fees back to the pool (i.e. to the LPs)

562:                     minBptAmountOut: 0, // Donation does not return BPTs, any number above 0 will revert

563:                     kind: AddLiquidityKind.DONATION,

564:                     userData: bytes("") // User data is not used by donation, so we can set it to an empty string

565:                 })

566:             );

567:         }

```

  

* This allow MEV (on L1) or racing txns to get those values that they don't deserve, basically stealing them from genuine LP providers, then immediately removing the liquidity after

  

An example of a whale removing liquidity with huge upLift fees donated:

  

1- Old whale in wETH-USDC Pool having his balance got upLifted 1000%

  

* from 2,500,000 USD/100wETH -> 25,000,000 USD/100wETH

* Total value in the Pool is 26,000,000 (whale has most of the pool)

  

2- `upliftFeeBps` is set to 5% \* 10 (value change) (11,500,000) as fees donated to a pool of 1,000,000

  

3- a MEV sees the whale liquidity removal and sandwich it (flashloan or use his own funds) add liquidity worth 1,000,000 (having 50% of the pool and getting 50% of the donation)

  

4- When the MEV submit a withdrawal, He will own (6,750,000 from donation + his own 1 million = 7,750,000)

  

5- his uplift is 5% \* \~8 = 40% = 3,100,000

  

6- MEV leave the pool with 4,650,000 having a profit of 4,650,000

  

The above numbered example may have exaggerated numbers only to show the feasibility of the MEV attack and the fee is not enough, attacks can be carried on smaller whales too, the idea stay the same

  

> _**NOTE!:**_ Its worth mentioning that above i assumed that `upliftFeeBps` is applied in the whole withdrawal value and not value increase only, thats how the code actually works, but that was another Bug

  

Assuming that `upliftFeeBps` is only applied on the value increase, then the attack will be completely easy to be carried by MEV, not requiring large whale % of pool getting out immediately Here is what happens

  

1- whale in wETH-USDC Pool having his balance got upLifted 500%

  

* from 2,500,000 USD/100wETH -> 12,500,000 USD/100wETH

* Total value in the Pool is 100,000,000

  

2- `upliftFeeBps` is set to 5% \* 5 \* 10,000,000 (value increase) (2,500,000) as fees donated to a pool of 87,500,000

  

3- a MEV sees the whale liquidity removal and sandwich it (flashloan or use his own funds) add liquidity worth 50,000,000 (having \~50% of the pool and getting \~50% of the donation)

  

4- When the MEV submit a withdrawal, He will own (1,750,000 from donation + his own 50 million = 51,750,000) (2% upLift)

  

5- his uplift is 5% \* \~2/100 = 0.1% = 0.1 \* 1,750,000 / 100 = 1750

  

6- MEV leave the pool with \~51,749,000 having a profit of 1,749,000

  

In the above second example, there can be small inaccuracies coming from small percentage change and uncertainty of how upLift Fees would actually apply to only upLift value

  

### Impact

  

Stealing of funds from LP providers

  

### Tools Used

  

Manual review

  

### Recommendations

  

implement timelock on LP providing, or implement vesting logic for donation

  

## [H-02] UpLift fees are applied on `depositAmount` instead of the upLifted (profit) value
### Summary

  

The `UpliftOnlyExample` contract's withdrawal fee calculation applies fees on the total withdrawal amount rather than just the uplift value, contradicting the whitepaper specification and potentially causing users to receive less than their initial deposit value.

  

### Vulnerability Details

  

In `onAfterRemoveLiquidity()`, the fee calculation is performed as:

  

```solidity

feePerLP = (uint256(localData.lpTokenDepositValueChange) * (uint256(feeDataArray[i].upliftFeeBps) * 1e18)) / 10000;

```

  

The issue arises because this fee is then applied to the total withdrawal amount rather than just the uplift portion:

  

```solidity

localData.feeAmount += (depositAmount * feePerLP);

```

  

Example scenario:

  

1. User deposits 1e18 tokens

2. Value increases to 2e18 (100% uplift)

3. With 60% uplift fee, fee is calculated on entire 2e18

4. User receives 8e17 tokens (less than initial 1e18 deposit)

  

This contradicts the whitepaper which states fees should only apply to "increase in value LPs have received over the value they would have if they had HODLed".

  

### Impact

  

* Loss of user funds below HODL value

* Contradict the white paper

  

### Tools Used

  

Manual review

  

### Recommendations

  

1- Modify fee calculation to only apply to the uplift portion:

  

2- Add explicit checks to ensure withdrawal amount after fees cannot fall below initial deposit value

  

## [H-03] freeze of funds (fees) to `QuantAMMAdmin` in `onAfterRemoveLiquidity()` fees distribution
### Summary

  

in `UpliftOnlyExample::onAfterRemoveLiquidity()` the function sends the `QuantAMMAdmin` to the admin in the wrong way, making it non retrievable for the admin

  

### Vulnerability Details

  

in `UpliftOnlyExample::onAfterRemoveLiquidity()`, the fees are sent the to `QuantAMMAdmin` as a BPT tokens here

  

```solidity

File: UpliftOnlyExample.sol

431:     function onAfterRemoveLiquidity(

  

536:         if (localData.adminFeePercent > 0) {

537:             _vault.addLiquidity(

538:                 AddLiquidityParams({

539:                     pool: localData.pool,

540:                     to: IUpdateWeightRunner(_updateWeightRunner).getQuantAMMAdmin(),

541:                     maxAmountsIn: localData.accruedQuantAMMFees,

542:                     minBptAmountOut: localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18,

543:                     kind: AddLiquidityKind.PROPORTIONAL,

544:                     userData: bytes("")

  

```

  

But the problem is that when the admin wants to exchange those BPT tokens, he can't do it through `UpliftOnlyExample` directly, since the admin doesn't have LPNFT token or a stored position in `poolsFeeData` failing his txn in `onAfterRemoveLiquidity` by this loop that will underflow if he doesn't have enough BPT token balance registered in the contract here

  

```solidity

471:         for (uint256 i = localData.feeDataArrayLength - 1; i >= 0; --i) {

```

  

this will underflow cause his desired withdrawal balance haven't been substracted during the loop to break it here

  

```solidity

504:                 if (localData.amountLeft == 0) {

505:                     break;

506:                 }

507:             } else {

508:                 feeDataArray[i].amount -= localData.amountLeft;

509:                 localData.feeAmount += (feePerLP * localData.amountLeft);

510:                 break;

511:             }

```

  

And if he tries to remove directly from BalancerV3 vault, he can't since balancer calls `onAfterRemoveLiquidity` with `msg.sender` as the router, which will fail due to `onlySelfRouter` modifier

  

```solidity

File: Vault.sol

846:         // Uses msg.sender as the Router (the contract that called the Vault).

847:         if (poolData.poolConfigBits.shouldCallAfterRemoveLiquidity()) {

848:             // `hooksContract` needed to fix stack too deep.

849:             IHooks hooksContract = _hooksContracts[params.pool];

850:

851:             amountsOut = poolData.poolConfigBits.callAfterRemoveLiquidityHook(

852:                 msg.sender,

853:                 amountsOutScaled18,

854:                 amountsOut,

855:                 bptAmountIn,

856:                 params,

857:                 poolData,

858:                 hooksContract

859:             );

860:         }

  

................

  

File: UpliftOnlyExample.sol

431:     function onAfterRemoveLiquidity(

432:         address router,

433:         address pool,

434:         RemoveLiquidityKind,

435:         uint256 bptAmountIn,

436:         uint256[] memory,

437:         uint256[] memory amountsOutRaw,

438:         uint256[] memory,

439:         bytes memory userData

440:     ) public override onlySelfRouter(router) returns (bool, uint256[] memory hookAdjustedAmountsOutRaw) {

  

.................

  

190:     modifier onlySelfRouter(address router) {

191:         _ensureSelfRouter(router);

192:         _;

193:     }

..................

  

683:     function _ensureSelfRouter(address router) private view {

684:         if (router != address(this)) {

685:             revert CannotUseExternalRouter(router);

686:         }

687:     }

  

```

  

This way, the admin is holding useless BPT tokens that can't be exchanged to tokens

  

### Impact

  

Loss of funds (fees) to `QuantAMMAdmin`

  

### Tools Used

  

Manual Review

  

### Recommendations

  

* Implement position registering to admin and mint NFT to him (complex and gas costly)

* Implement a logic to send Tokens to admin and not adding liquidity to him (easier and less gas costly)

  

## [H-04] in `onAfterRemoveLiquidity()`, `maxAmountsIn` during admin fee distribution can cause txn reverts
### Summary

  

in `onAfterRemoveLiquidity()` during `_vault.addLiquidity()` made to the admin, `maxAmountsIn` will cause reverts in multiple circumstances

  

### Vulnerability Details

  

In `onAfterRemoveLiquidity()`, `amountsOutRaw` can round down when calculating `exitFee` here

  

```solidity

File: UpliftOnlyExample.sol

522:         for (uint256 i = 0; i < localData.amountsOutRaw.length; i++) {

523:             uint256 exitFee = localData.amountsOutRaw[i].mulDown(localData.feePercentage);

```

  

That `exitFee` is then used to get `accruedQuantAMMFees` (used as `maxAmountIn` during `_vault.addLiquidity()`)

  

* Both of the above variables are the actual tokens of the Pool that will be used during liquidity addition

  `amountsOutRaw` array can have low decimal tokens like USDC for example that will most likely have slight truncation during fee calculation

  

Now during `minBptAmountOut` calculations in `_vault.addLiquidity()` to the admin

  

```solidity

File: UpliftOnlyExample.sol

542:                     minBptAmountOut: localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18,

```

  

* There can be cases where BPT amounts are perfectly divisible and doesn't round down

  * This is likely to happen since during Liquidity addition BPT is calculated and then stay unchanged after that, but Pool reserves change by swapping and weight changes

    * So during swapping balances corresponding to those BPT change by wei through the invariant swap formula

  * BPT tokens are 18 decimals and less likely to round down in that calculation

  

The above condition will create a situation where during `_vault.addLiquidity()` the `minBptAmountOut` was accurate but the call will provide less than needed `maxAmountsIn` causing a revert of the txn

  

The above temporary DOS can be mitigated by user repeatedly trying to adjust `bptAmountIn` that rounds down (to have consistent double rounding down in `onAfterRemoveLiquidity()`) during his call to `removeLiquidityProportional()`

  

#### Pseudo Exampled of numbered scenario

  

1- Token A (USDC, 6 decimals):

  

* `amountsOutRaw[0]` = 100.000123 USDC (100\_000\_123)

  

2- Token B (WETH, 18 decimals):

  

* `amountsOutRaw[1]` = 0.100000000000000123 WETH (100\_000\_000\_000\_000\_123)

  

Parameters:

  

* `feePercentage` = 0.02e18 (2%)

* `adminFeePercent` = 0.5e18 (50%)

* `feeAmount` = 20 (perfect number, no decimals)

  

Calculation flow with rounding:

  

1- For USDC:

  

* `exitFee` = 100\_000\_123 × 0.02e18 / 1e18 = 2\_000\_002 (rounded down)

* `accruedQuantAMMFees[0]` = 2\_000\_002 × 0.5e18 / 1e18 = 1\_000\_001

  

2- For WETH:

  

* `exitFee` = 100\_000\_000\_000\_000\_123 × 0.02e18 / 1e18 = 2\_000\_000\_000\_000\_002

* `accruedQuantAMMFees[1]` = 2\_000\_000\_000\_000\_002 × 0.5e18 / 1e18 = 1\_000\_000\_000\_000\_001

  

3- BPT calculation:

  

* `feeAmount` = 2e18 (perfect number)

* `minBptAmountOut` = 20 × 0.5e18 / 1e18 = 1e18 (no rounding needed)

  Txn reverting due to `minBptAmountOut` not reached with the provided `accruedQuantAMMFees` as maxAmountsIn

  

> _**Please notice**_ that simply rounding Up won't work for the same reason (a variable will be perfect % and will not benefit from `mulUp` while other variable gets benefited from `mulUp` cause it would have been truncated otherwise) and this issue sequence will always happen and is different from issues simply stating that adminFees should rounUp

  

### Impact

  

Multiple txn reverts in multiple circumstances affecting the availability of `removeLiquidityProportional()` (temporary DOS)

  

### Tools Used

  

Manual review

  

### Recommendations

  

Build a logic that

  

* Doesn't pass `maxAmountsIn` or pass a very high amount, doesn't matter

* uses returned `amountsIn` used from `addLiquidity()` and then use those to remove it from `hookAdjustedAmountsOutRaw`

  

## [H-05] Truncation due to bad order of division causing higher fees sent to the admin 
### Summary

  

precision loss in `UpliftOnlyExample::onAfterSwap()` fee calculation mechanism that leads to very high QuantAdmin fees percentage from `hookFee`.

  

### Vulnerability Details

  

The fee calculation in `onAfterSwap()` performs division operations in a bad order:

  

```solidity

if (quantAMMFeeTake > 0) {

    uint256 adminFee = hookFee / (1e18 / quantAMMFeeTake);

    ownerFee = hookFee - adminFee;

```

  

The issue arises from dividing 1e18 by `quantAMMFeeTake` first, which creates truncation that gets magnified in subsequent calculations.

  

Examples:

  

* For quantAMMFeeTake = 0.3e18 (30%)

* 1e18 / 0.3e18 = 3.333... truncates to 3

* With hookFee = 100e18:

  * Calculated: adminFee = 100e18/3 = 33e18

  * Expected: 100e18 \* 0.3 = 30e18

  * Difference: +10% overcharge

  

Another Example That will eat all the swap Fees (Works with any thing above 0.5e18 as quantAMMFeeTake):

  

* For quantAMMFeeTake = 0.7e18 (70%)

* 1e18 / 0.7e18 = 1.428... truncates to 1

* With hookFee = 100e18:

  * Calculated: adminFee = 100e18/1 = 100e18

  * Expected: 100 \* 0.7 = 70e18

  * Difference: +42.8% overcharge

  

> _**NOTE!:**_ The issues doesn't assume the admin to be behaving maliciously, but its just a valid fee % that is <= 1e18 as checked here

  

```solidity

File: UpdateWeightRunner.sol

141:     function setQuantAMMUpliftFeeTake(uint256 _quantAMMUpliftFeeTake) external{

142:         require(msg.sender == quantammAdmin, "ONLYADMIN");

143:         require(_quantAMMUpliftFeeTake <= 1e18, "Uplift fee must be less than 100%");

144:         uint256 oldSwapFee = quantAMMSwapFeeTake;

145:         quantAMMSwapFeeTake = _quantAMMUpliftFeeTake;

146:

147:         emit UpliftFeeTakeSet(oldSwapFee, _quantAMMUpliftFeeTake);

148:     }

```

  

### Impact

  

Fees overcharged % from `hookFee` during `onAfterSwap()` Hook

  

### Tools Used

  

* Manual code

  

### Recommendations

  

Implement the following fix to maintain precision:

  

```solidity

uint256 adminFee = (hookFee * quantAMMFeeTake) / 1e18;

```

  

## [H-06] improper access control in `onAfterRemoveLiquidity()` risks loss of funds by attacker
### Summary

  

in `UpliftOnlyExample::onAfterRemoveLiquidity()` absent access control to only allow the vault calls opens up a path to cause loss of funds to LP providers by an attacker calling to with victim address to burn the victim NFTs

  

### Vulnerability Details

  

`onAfterRemoveLiquidity` is not access controlled and can be called by any one as long as the passed router is the `UpliftOnlyExample` it self, which is false access control guard.

  

Also the function takes the user address that his NFT will be burnt as a passed variable.

  

What makes it worse is that `LPNFT::Burn()` doesn't have checks for tokens approvals or ownership, so `UpliftOnlyExample` can burn any user NFT

  

Here is what can happen:

1- Attacker calls `Unlock()` on balancerV3 vault to have a transient unlock of the vault for the txn

2-  Attacker calls `onAfterRemoveLiquidity` after calling unlock passing the:

  

* router as `UpliftOnlyExample` to pass the `onlySelfRouter` modifier

* `userData` to be the victim address

* `bptAmountIn` to be the balance registered in `poolsFeeData` for the victim

  

3- The NFTs of the victim will be burnt due to this block

  

```solidity

File: UpliftOnlyExample.sol

493:             if (feeDataArray[i].amount <= localData.amountLeft) {

494:                 uint256 depositAmount = feeDataArray[i].amount;

495:                 localData.feeAmount += (depositAmount * feePerLP);

496:

497:                 localData.amountLeft -= feeDataArray[i].amount;

498:

499:                 lpNFT.burn(feeDataArray[i].tokenID);

500:

501:                 delete feeDataArray[i];

502:                 feeDataArray.pop();

503:

504:                 if (localData.amountLeft == 0) {

505:                     break;

506:                 }

507:             } else {

508:                 feeDataArray[i].amount -= localData.amountLeft;

509:                 localData.feeAmount += (feePerLP * localData.amountLeft);

510:                 break;

511:             }

```

  

4- Attacker will need to call `settle()` to settle debt taken from the `UpliftOnlyExample` call to `addLiquidity()` (the fee distribution Logic)

  

```solidity

File: UpliftOnlyExample.sol

536:         if (localData.adminFeePercent > 0) {

537:             _vault.addLiquidity(

538:                 AddLiquidityParams({

539:                     pool: localData.pool,

540:                     to: IUpdateWeightRunner(_updateWeightRunner).getQuantAMMAdmin(),

541:                     maxAmountsIn: localData.accruedQuantAMMFees,

542:                     minBptAmountOut: localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18,

543:                     kind: AddLiquidityKind.PROPORTIONAL,

544:                     userData: bytes("")

545:                 })

546:             );

547:             emit ExitFeeCharged(

548:                 userAddress,

549:                 localData.pool,

550:                 IERC20(localData.pool),

551:                 localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18

552:             );

553:         }

554:

555:         if (localData.adminFeePercent != 1e18) {

556:             // Donates accrued fees back to LPs.

557:             _vault.addLiquidity(

558:                 AddLiquidityParams({

559:                     pool: localData.pool,

560:                     to: msg.sender, // It would mint BPTs to router, but it's a donation so no BPT is minted

561:                     maxAmountsIn: localData.accruedFees, // Donate all accrued fees back to the pool (i.e. to the LPs)

562:                     minBptAmountOut: 0, // Donation does not return BPTs, any number above 0 will revert

563:                     kind: AddLiquidityKind.DONATION,

564:                     userData: bytes("") // User data is not used by donation, so we can set it to an empty string

565:                 })

566:             );

567:         }

  

.....................

  

File: Vault.sol

507:     function addLiquidity(

569:         (amountsIn, amountsInScaled18, bptAmountOut, returnData) = _addLiquidity(

570:             poolData,

571:             params,

572:             maxAmountsInScaled18

573:         );

)

  

.....................

  

610:     function _addLiquidity(

732:             // 3) Deltas: Debit of token[i] for amountInRaw.

733:             _takeDebt(token, amountInRaw);

734: )

  
  

```

  

The loss for attacker is very smaller (fees of minimumSwapFees or UpLiftFee) compared to the complete loss of funds for the victim, since users can only retrieve funds through`removeLiquidityProportional` by their NFT and the `poolsFeeData`

  

### Impact

  

Complete loss of funds

  

### Tools Used

  

Manual Review

  

### Recommendations

  

Add `OnlyVault` modifier

  

## [H-07] Hook fees (`ownerFee`) are stuck in `UpliftOnlyExample` and non retrievable
### Summary

  

`UpliftOnlyExample::onAfterSwap()` some fees taken from the swap are not sent to `quantAMMAdmin` or Hook owner, but sent to `UpliftOnlyExample` it self with no retrievable way for the tokens

  

### Vulnerability Details

  

after pool swap, there is a hook called `onAfterSwap()` in `UpliftOnlyExample` to deduct swap fees for the hook contract, (part of it is sent to the Quant admin)

  

```solidity

File: UpliftOnlyExample.sol

293:     function onAfterSwap(

  

341:

342:                 if (ownerFee > 0) {

343:                     _vault.sendTo(feeToken, address(this), ownerFee);

344:

345:                     emit SwapHookFeeCharged(address(this), feeToken, ownerFee);

  

350:     }

```

  

But the problem in the above code is that it sends the fees amount `ownerFee` to it self, with no function to retrieve those tokens

  

### Impact

  

Loss of swap fees

  

### Tools Used

  

Manual review

  

### Recommendations

  

use the `ownerFee` to be donated to the pool like the logic in `onAfterRemoveLiquidity` Hook or if its intended to be sent to Pool owner, then send those tokens to him like the way you did with admin fees

  

## [H-08] Users can avoid paying fees on up lifts by utilising `afterUpdate` Logic
### Summary

  

Users can weaponize the `afterUpdate` hook during NFT transfers to dodge paying fees on Up Lifts, since during the hook, it override the `lpTokenDepositValue` that was previously registered by setting it to `lpTokenDepositValueNow` tricking the system to have no upLifts by calling `removeLiquidityProportional` in the same block from their second wallet

  

### Vulnerability Details

  

During user adding Liquidity, we register the deposit value of LP token here

  

```solidity

File: UpliftOnlyExample.sol

250:         poolsFeeData[pool][msg.sender].push(

251:             FeeData({

252:                 tokenID: tokenID,

253:                 amount: exactBptAmountOut,

254:                 //this rounding favours the LP

255:                 lpTokenDepositValue: depositValue,

256:                 //known use of timestamp, caveats are known.

257:                 blockTimestampDeposit: uint40(block.timestamp),

258:                 upliftFeeBps: upliftFeeBps

259:             })

260:         );

```

  

This is used as a reference value to see how much the value of LP token has increased since deposit when calling `removeLiquidityProportional` specifically in `onAfterRemoveLiquidity` Here

  

```solidity

File: UpliftOnlyExample.sol

480:             if (localData.lpTokenDepositValueChange > 0) {

481:                 feePerLP =

482:                     (uint256(localData.lpTokenDepositValueChange) * (uint256(feeDataArray[i].upliftFeeBps) * 1e18)) /

483:                     10000;

484:             }

```

  

But Users can avoid paying that fee by transferring the NFT to other wallet

  

```solidity

File: LPNFT.sol

49:     function _update(address to, uint256 tokenId, address auth) internal override returns (address previousOwner) {

50:         previousOwner  = super._update(to, tokenId, auth);

51:         //_update is called during mint, burn and transfer. This functionality is only for transfer

52:         if (to != address(0) && previousOwner != address(0)) {

53:             //if transfering the record in the vault needs to be changed to reflect the change in ownership

54:             router.afterUpdate(previousOwner, to, tokenId);

55:         }

56:     }

.................

  

File: UpliftOnlyExample.sol

576:     function afterUpdate(address _from, address _to, uint256 _tokenID) public {

586:

587:         int256[] memory prices = IUpdateWeightRunner(_updateWeightRunner).getData(poolAddress);

609:                 feeDataArray[tokenIdIndex].lpTokenDepositValue = lpTokenDepositValueNow;

610:                 feeDataArray[tokenIdIndex].blockTimestampDeposit = uint32(block.number);

611:                 feeDataArray[tokenIdIndex].upliftFeeBps = upliftFeeBps;

  

614:                 poolsFeeData[poolAddress][_to].push(feeDataArray[tokenIdIndex]);

  

```

  

As you see, we retrieve the current LP Deposit value and override the old one with it and then assign those values to the new owner `to` in Line 614

  

### Impact

  

Loss of funds to LP providers and QuantAdmin

  

### Tools Used

  

Manual Review

  

### Recommendations

  

Don't override deposit value when transferring, or charge fees during transfers (will require complexity since it will require `vault.unlock()` etc, more gas cost, )

  

  

## [M-01] Last withdrawer will donate fees to empty pool allowing MEV and having always stuck funds in the pool
### Summary

  

in `onAfterRemoveLiquidity()` fees of the withdrawn tokens amounts (whatever its upLift or the minimum fees) of the last user withdrawing from the pool will be donated to empty pool.

  

### Vulnerability Details

  

> _**NOTE!**_:  the bug about `accruedQuantAMMFees` wrong distribution since Admin don't have position registered in `poolsFeeData` Is assumed to be solved by sending actual tokens to the admin

  

in `onAfterRemoveLiquidity()` The code calculate fees on the withdrawn tokens amounts (whatever its upLift or the minimum fees) and donates part of it to the pool and part of it to the QuantAMM admin, Here:

  

```solidity

File: UpliftOnlyExample.sol

            function onAfterRemoveLiquidity(

536:         if (localData.adminFeePercent > 0) {

537:             _vault.addLiquidity(

538:                 AddLiquidityParams({

539:                     pool: localData.pool,

540:                     to: IUpdateWeightRunner(_updateWeightRunner).getQuantAMMAdmin(),

541:                     maxAmountsIn: localData.accruedQuantAMMFees,

542:                     minBptAmountOut: localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18,

543:                     kind: AddLiquidityKind.PROPORTIONAL,

544:                     userData: bytes("")

545:                 })

546:             );

547:             emit ExitFeeCharged(

548:                 userAddress,

549:                 localData.pool,

550:                 IERC20(localData.pool),

551:                 localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18

552:             );

553:         }

554:

555:         if (localData.adminFeePercent != 1e18) {

556:             // Donates accrued fees back to LPs.

557:             _vault.addLiquidity(

558:                 AddLiquidityParams({

559:                     pool: localData.pool,

560:                     to: msg.sender,

561:                     maxAmountsIn: localData.accruedFees,

562:                     minBptAmountOut: 0,

563:                     kind: AddLiquidityKind.DONATION,

564:                     userData: bytes("")

  

567:         })

```

  

But the problem is that There are alot of circumstances where user withdrawing is the last withdrawer of the pool, what happens is:

  

1. User will get charged fees, part of it added to the admin (in the correct way) and part of it is donated to the pool

2. this will create an empty pool that have tokens in it with no corresponding BPT

3. MEV can immediately add liquidity (to mint BPT) and remove liquidity (burning the BPT and getting the initially deposited tokens Plus the stuck tokens)

4. The problem is yet that on MEV interactions, they will be charged fees and donated to the again empty pool (after MEV removing liquidity)

5. This will create some tokens in the end that is never retrievable and always stuck in the pool

  

> The above scenario describes what happens when the first mentioned bug get fixed by sending fees to the admin as actual tokens

  

Now if they decide to solve it by registering admin fee position in `poolsFeeData` then

  

1. Last withdrawer gets charged fees that part of it mint liquidity to admin, and part of it gets donated to the pool (that now have the admin liquidity, meaning that the admin owns all token Pools)

2. Admin comes to remove liquidity and he himself will be charged fee that again that of it mint liquidity to admin, and part of it gets donated to the pool&#x20;

3. Again and again and again (yet pool will have stuck funds either way after last withdrawer)

  

### Impact

  

Stuck funds in the pool

  

### Tools Used

  

Manual Review

  

### Recommendations

  

if the withdrawer is the last withdrawer (vault supply is 0) send all fees to the admin, or don't charge fees (like any pool does with the last withdrawer, they simply transfer all the balance of the pool to him)

  

## [M-02] Moving Average Length Validation Prevents Admin Override for Rules Requiring Historical Data
### Summary

  

The `initialisePoolRuleIntermediateValues()` function in `UpdateRule` fails to properly handle rules like `MinimumVarianceUpdateRule` that require double the number of moving averages (current + previous) due to a length validation check in `_setInitialMovingAverages()`.

  

### Vulnerability Details

  

The issue occurs in the following call flow:

  

```solidity

// UpdateWeightRunner.sol

function setIntermediateValuesManually(

    address _poolAddress,

    int256[] memory _newMovingAverages,

    int256[] memory _newParameters,

    uint _numberOfAssets

) external {

    // ... auth checks ...

    rule.initialisePoolRuleIntermediateValues(_poolAddress, _newMovingAverages, _newParameters, _numberOfAssets);

}

```

  

```solidity

// QuantammMathMovingAverage.sol

function _setInitialMovingAverages() internal {

    if (movingAverageLength == 0 || _initialMovingAverages.length == _numberOfAssets) {

        movingAverages[_poolAddress] = _quantAMMPack128Array(_initialMovingAverages);

    } else {

        revert("Invalid set moving avg");

    }

}

```

  

The `MinimumVarianceUpdateRule` requires double the assets count for moving averages:

  

```solidity

uint16 private constant REQUIRES_PREV_MAVG = 1;

```

  

And this how storage should look for that rule by DEV comment

  

```solidity

// QuantAMMMathMovingAverage.sol

    // this can be just the moving averages per token, or if prev moving average is true then it is [...moving averages, ...prev moving averages]

  

    mapping(address => int256[]) public movingAverages;

```

  

The validation check `_initialMovingAverages.length == _numberOfAssets` prevents setting both current and previous moving averages, only allowing current ones to be set.

  

This will leave two options to the admin:

  

1. Not using the function

2. Use the function setting only moving average of the length of number of assets

  

If its the first one, then the availability of function is lost

  

If its the second option, then other integrators of that stored previous moving averages will be cornuted

  

### Impact

  

* Admin cannot properly set/override moving averages for rules requiring previous values

* Loss of historical moving average data when manual override is needed

* Breaks integrations from other protocols relying on both current and previous moving average values

  

### Tools Used

  

Manual  review

  

### Recommendations

  

Modify the validation in `_setInitialMovingAverages()` to handle rules requiring different moving average lengths or there can be a check to assure that new values are the same length as `movingAverageLength` retrieved from the mapping

  

## [M-03] in `UpliftOnlyExample`, Immutable UpdateWeightRunner Address Prevents Seamless Contract Migration
### Summary

  

The immutable `_updateWeightRunner` address causes overhead during migration to new `UpdateWeightRunner` contracts, forcing admins to maintain fee configurations across multiple deprecated contracts to support existing `UpliftOnlyExample`.

  

### Vulnerability Details

  

When `_updateWeightRunner` is set as immutable in `UpliftOnlyExample`, hooks created with older `UpdateWeightRunner` contracts cannot be updated to point to newer versions. This creates a fragmented system where:

  

1. New hooks use the latest `UpdateWeightRunner`

2. Old hooks remain tied to deprecated `UpdateWeightRunner` contracts

3. Admins must maintain fee configurations across all versions to ensure consistent behavior since this is the only usage of `_updateWeightRunner` beside retrieving admin address

  

### Impact

  

* Increased operational complexity for admins managing multiple `UpdateWeightRunner` instances

* Risk of inconsistent fee structures across different hook generations

  

### Tools Used

  

Manual  review

  

### Recommendations

  

* Remove the immutable modifier and implement an upgradeable pattern:

  

```solidity

address private _updateWeightRunner;

  

function setUpdateWeightRunner(address newRunner) external onlyOwner {

    _updateWeightRunner = newRunner;

    emit UpdateWeightRunnerChanged(newRunner);

}

```

  

## [M-04] Pool Weights are non-updatable After Updating `updateWeightRunner` address
### Summary

  

The `setUpdateWeightRunnerAddress` function in QuantAMMWeightedPool allows changing the UpdateWeightRunner contract address, but the pool has no way to set rules on the new contract since `_setRule()` is only callable during initialization. This is particularly problematic because UpdateWeightRunner maintains multiple pool-specific mappings keyed by msg.sender.

  

### Vulnerability Details

  

The vulnerability exists because:

  

1- `UpdateWeightRunner` mappings keyed to `msg.sender` (pool address):

  

```solidity

rules[msg.sender] = _poolSettings.rule; //Only pool can set its own rules

poolOracles[msg.sender] = optimisedHappyPathOracles; //Pool oracle settings

poolBackupOracles[msg.sender] = _poolSettings.oracles; //Pool backup oracles

poolRuleSettings[msg.sender] = PoolRuleSettings({ //Pool rule configuration

    lambda: _poolSettings.lambda,

    epsilonMax: _poolSettings.epsilonMax,

    absoluteWeightGuardRail: _poolSettings.absoluteWeightGuardRail,

    ruleParameters: _poolSettings.ruleParameters,

    timingSettings: PoolTimingSettings({ updateInterval: _poolSettings.updateInterval, lastPoolUpdateRun: 0 }),

    poolManager: _poolSettings.poolManager

});

```

  

2-  Rules are set through `_setRule()` which is only called during `initialize()` function with initializer modifier

  

3-  When `setUpdateWeightRunnerAddress()` changes the `UpdateWeightRunner`:

  

```solidity

function setUpdateWeightRunnerAddress(address _updateWeightRunner) external override {

    require(msg.sender == quantammAdmin, "ONLYADMIN");

    updateWeightRunner = UpdateWeightRunner(_updateWeightRunner);

    emit UpdateWeightRunnerAddressUpdated(address(updateWeightRunner), _updateWeightRunner);

}

```

  

The pool has no mechanism to call `setRuleForPool()` on the new contract since:

  

* `_setRule()` is locked behind initializer

* No other function exists to set rules post-initialization

  

Since the pool can't call `setRuleForPool()`, then the new `UpdateWeightRunner` contract will have no way to interact or call the pool since almost in every action to a pool, it uses the variables from the mappings`rules` or `poolRuleSettings` etc ,for example:

  

```solidity

    function performUpdate(address _pool) public {

        address rule = address(rules[_pool]);

        require(rule != address(0), "Pool not registered");

        PoolRuleSettings memory settings = poolRuleSettings[_pool];

  

```

  

### Impact

  

When `UpdateWeightRunner` address is changed:

  

* Pool becomes permanently unable to update weights since:

  * Rules cannot be set on new contract

  * All pool configurations are lost (rules, oracles, settings)

  * Only the pool itself can set its configurations (msg.sender mappings)

  * No post-initialization setting mechanism exists

* Core pool functionality is broken

  

### Tools Used

  

* Manual code review

  

### Recommendations

  

1. Add a new function to allow setting rules after `UpdateWeightRunner` changes:

  

```solidity

function migrateRules(

    int256[] memory _initialWeights,

    int256[] memory _ruleIntermediateValues,

    int256[] memory _initialMovingAverages,

    IQuantAMMWeightedPool.PoolSettings memory _poolSettings

) external {

    require(msg.sender == quantammAdmin, "ONLYADMIN");

    _setRule(_initialWeights, _ruleIntermediateValues, _initialMovingAverages, _poolSettings);

}

```

  

## [M-05] Users transferring their NFT position will retroactively get the new `upliftFeeBps`
### Summary

  

When user transfer his NFT LP position to another wallet, the new `upliftFeeBps` is registered to `poolsFeeData` of the `to`, this contradict the intended design to have no retroactive  `upliftFeeBps` applied to already open position and also opens an attack surface to dodge the old high fees for the current smaller ones

  

### Vulnerability Details

  

> _**NOTE!**_: its worth mentioning that this is a different bug from the bug that talks about complete dodging of UpLift fees through `lpTokenDepositValue` overriding

  

When user adds liquidity, the `upliftFeeBps` is registered in his `poolsFeeData` so that any future change doesn't apply to his position retroactively

  

```solidity

File: UpliftOnlyExample.sol

219:     function addLiquidityProportional()

  

249:

250:         poolsFeeData[pool][msg.sender].push(

251:             FeeData({

252:                 tokenID: tokenID,

253:                 amount: exactBptAmountOut,

254:                 //this rounding favours the LP

255:                 lpTokenDepositValue: depositValue,

256:                 //known use of timestamp, caveats are known.

257:                 blockTimestampDeposit: uint40(block.timestamp),

258:                 upliftFeeBps: upliftFeeBps

259:             })

260:         );

  

263:     }

```

  

But the problem is that in `afterUpdate` hook that is triggered on NFT transfers, the `upliftFeeBps` of `poolsFeeData` of `to` is set to the current `upliftFeeBps`

  

```solidity

File: UpliftOnlyExample.sol

576:     function afterUpdate(address _from, address _to, uint256 _tokenID) public {

  

609:                 feeDataArray[tokenIdIndex].lpTokenDepositValue = lpTokenDepositValueNow;

610:                 feeDataArray[tokenIdIndex].blockTimestampDeposit = uint32(block.number);

611:                 feeDataArray[tokenIdIndex].upliftFeeBps = upliftFeeBps;

612:

613:                 //actual transfer not a afterTokenTransfer caused by a burn

614:                 poolsFeeData[poolAddress][_to].push(feeDataArray[tokenIdIndex]);

  

```

  

this causes two problems:

  

1. New fees are applied retroactively, which is not the intention of the protocol (Broken functionality)

2. There can be scenarios where `upliftFeeBps`  was set by the admin to lower values than the ones the user initially deposited at (ie, events of lower fees, or simply the admin wants to do it due to increased competition in the field that provided less fees, etc)

   * When that happens, the user can transfer the NFT to other wallet of his own so that the `upliftFeeBps` variable gets overridden by the new one

  

### Impact

  

* Non intended design

* User can decide to override their `upliftFeeBps` when they see it less costly to pay (overall profitable to them)

  * Leading to loss of funds (Fees) to the Quant Admin and LP providers (since there should have been larger fees donated to their pool)

  

### Tools Used

  

Manual review

  

### Recommendations

  

Don't override that variable, simply remove Line 611

  

## [M-06] attacker can completely prevent users from withdrawing in `UpliftOnlyExample`
### Summary

  

Due to the first in last out nature of the `poolsFeeData`, attacker can prevent users from withdrawing liquidity permanently by transferring to them millions of dust NFT positions over filling the `feeData[]` and causing OOG errors during withdrawals

  

### Vulnerability Details

  

during withdrawals in `UpliftOnlyExample::onAfterRemoveLiquidity()` we start looping from the last deposit (`localData.feeDataArrayLength - 1`)

  

```solidity

File: UpliftOnlyExample.sol

471:         for (uint256 i = localData.feeDataArrayLength - 1; i >= 0; --i) {

493:             if (feeDataArray[i].amount <= localData.amountLeft) {

494:                 uint256 depositAmount = feeDataArray[i].amount;

495:                 localData.feeAmount += (depositAmount * feePerLP);

  

501:                 delete feeDataArray[i];

502:                 feeDataArray.pop();

  

507:             } else {

508:                 feeDataArray[i].amount -= localData.amountLeft;

509:                 localData.feeAmount += (feePerLP * localData.amountLeft);

510:                 break;

511:             }

512:         }

```

  

This opens up an attack on large valuable positions by malicious BOTs (especially in L2s where gas price is very cheap), here what would happen:

  

1- Attacker sees very large valuable deposit position

2- Attacker deposit dust value positions and transfer the NFT to the victim (taking advantage of `afterUpdate` hook in NFT transfer, pushing the position to the end of `feeDataArray` of victim)

3- Attack is repeated millions of times, causing any attempt to withdraw the actual first position of the user to cause OOG reverts

4- The victim can't transfer his valuable LP NFT position to other wallet as a solution, since transfer will try to re-order the array of the sender that will cause OOG reverts too

  

```solidity

File: UpliftOnlyExample.sol

576:     function afterUpdate(address _from, address _to, uint256 _tokenID) public {

  

590:         FeeData[] storage feeDataArray = poolsFeeData[poolAddress][_from];

  

616:                 if (tokenIdIndex != feeDataArrayLength - 1) {

617:                     //Reordering the entire array could be expensive but it is the only way to keep true FILO

618:                     for (uint i = tokenIdIndex + 1; i < feeDataArrayLength; i++) {

619:                         delete feeDataArray[i - 1];

620:                         feeDataArray[i - 1] = feeDataArray[i];

621:                     }

622:                 }

```

  

5- the victim will try to remove those dust positions batch by batch to reach his first position, but the malicious bot of the attacker will track the victim `poolsFeeData` after each block (no need to front run, valid attack path on L2s), adding more position to the victim if needed

  

Attack is feasible since its on L2s and using dust positions to DOS (very small loss overall to attacker to keep the attack running)

  

This was possible due to:

  

1. when transferring we don't check the array size of `to` if it has passed 100 or not

2. the FILO nature when withdrawing, and re-ordering when transferring

3. any one can enlarge the size of other people by transferring to them the NFT, and array will grow in `afterUpdate` hook

4. no minimum LP deposits to prevent dusts deposits

  

### Impact

  

Freeze of funds to the victim

  

### Tools Used

  

Manual Review

  

### Recommendations

  

* implement minimum LP deposits

* check for length of `feeDataArray` of the `to` during transfers and prevent it it will exceed

* give a way to withdraw specific position index (if FILO implementation is not very important)

* give the ability to guard against who can transfer his NFT position to me (set by user him self)

  

## [M-07] division before multiplication in `lpTokenDepositValueChange` causes loss of fees
### Summary

  

in `UpliftOnlyExample::onAfterRemoveLiquidity` There is a division before multiplication in `lpTokenDepositValueChange` usage logic, causing any change of deposit value below 100% truncates to 0, and any  change that is not perfect % of `lpTokenDepositValue` will have the remainder truncated

  

### Vulnerability Details

  

in `UpliftOnlyExample::onAfterRemoveLiquidity` Hook, there is a calculation to check for `lpTokenDepositValue` during withdrawals to see if any uplift happened, according to that, its decided to use `upliftFeeBps` or `minWithdrawalFeeBps`

  

```solidity

File: UpliftOnlyExample.sol

474:             localData.lpTokenDepositValueChange =

475:                 (int256(localData.lpTokenDepositValueNow) - int256(localData.lpTokenDepositValue)) /

476:                 int256(localData.lpTokenDepositValue);

477:

478:             uint256 feePerLP;

  

480:             if (localData.lpTokenDepositValueChange > 0) {

481:                 feePerLP =

482:                     (uint256(localData.lpTokenDepositValueChange) * (uint256(feeDataArray[i].upliftFeeBps) * 1e18)) /

483:                     10000;

484:             }

  

486:             else {

  

489:                 feePerLP = (uint256(minWithdrawalFeeBps) * 1e18) / 10000;

490:             }

```

  

But as you see in Line 475, we get the ratio of change by dividing by the old `lpTokenDepositValue` before we multiply this ration by 1e18 in Line 482

  

Here is an example of what would happen

  

* `lpTokenDepositValueNow` = 15e18

* `lpTokenDepositValue` = 10e18

  `lpTokenDepositValueNow` - `lpTokenDepositValue`  = 15e18 - 10e18 = 5e18

  5e18 /  `lpTokenDepositValue` = 5e18 / 10e18 = 0

  

`lpTokenDepositValueChange` = 0

  

Then the logic in else block will be the one to be executed, since the line 480 `if` evaluate to false

  

Also, if the uplift happened was 350%, then the 50% will truncate (35e18 - 10e18 / 10e18 = 3e18)

  

### Impact

  

Loss of fees (funds) for LP providers and Quant AMM admin

  

### Tools Used

  

Manual Review

  

### Recommendations

  

multiply during `lpTokenDepositValueChange` calculation first (as seen below)

  

```solidity

File: UpliftOnlyExample.sol

474:             localData.lpTokenDepositValueChange =

475:                 (int256(localData.lpTokenDepositValueNow) - int256(localData.lpTokenDepositValue) * 1e18) /

476:                 int256(localData.lpTokenDepositValue);

477:

478:             uint256 feePerLP;

479:             // if the pool has increased in value since the deposit, the fee is calculated based on the deposit value

480:             if (localData.lpTokenDepositValueChange > 0) {

481:                 feePerLP =

482:                     (uint256(localData.lpTokenDepositValueChange) * (uint256(feeDataArray[i].upliftFeeBps))) /

483:                     10000;

484:             }

485:             // if the pool has decreased in value since the deposit, the fee is calculated based on the base value - see wp

486:             else {

487:                 //in most cases this should be a normal swap fee amount.

488:                 //there always myst be at least the swap fee amount to avoid deposit/withdraw attack surgace.

489:                 feePerLP = (uint256(minWithdrawalFeeBps) * 1e18) / 10000;

490:             }

```

  

## [M-08] Single Storage Variable Used for Both Swap and Uplift Fees
### Summary

  

The `UpdateWeightRunner` contract incorrectly uses the same storage variable `quantAMMSwapFeeTake` for both swap fees and uplift fees, leading to incorrect fee calculations in the `UpliftOnlyExample` contract.

  

### Vulnerability Details

  

In `UpdateWeightRunner`:

  

* Both `setQuantAMMUpliftFeeTake()` and `setQuantAMMSwapFeeTake()` modify the same storage variable `quantAMMSwapFeeTake`

* `getQuantAMMUpliftFeeTake()` returns `quantAMMSwapFeeTake`

  

In `UpliftOnlyExample`:

  

* Line 331: Retrieves uplift fee using `getQuantAMMUpliftFeeTake()`

* Line 519: Uses the same function for swap fee calculations

  

This means both uplift fees and swap fees will always be identical, which is likely not the intended behavior, (evidenced from the two setter and getter functions too)

  

### Impact

  

* Incorrect fee calculations as uplift fees and swap fees cannot be set independently

  

### Tools Used

  

Manual Review

  

### Recommendations

  

1- Add separate storage variable for uplift fees:

  

```solidity

uint256 public quantAMMUpliftFeeTake;

uint256 public quantAMMSwapFeeTake;

```

  

2- Update getter functions to return correct variables:

  

```solidity

function getQuantAMMUpliftFeeTake() external view returns (uint256) {

    return quantAMMUpliftFeeTake;

}

  

function getQuantAMMSwapFeeTake() external view returns (uint256) {

    return quantAMMSwapFeeTake;

}

```

  

3- Ensure setter functions modify their respective variables

  
  

## [L-01] `minWithdrawalFeeBps` are not added to `upliftFeeBps` causing loss of fees and allowing MEV actions
### Summary

  

> _**Note!:**_ This bug assumes that `upliftFeeBps` is applied in the upLifted value only as intended in the whitePaper and assumes the rounding down to 0 of `lpTokenDepositValueChange` is solved

  

The `UpliftOnlyExample` contract's uplift fee calculation can result in fees lower than the intended minimum when small uplifts occur, potentially enabling MEV attacks that were meant to be prevented by `minWithdrawalFeeBps`.

  

### Vulnerability Details

  

Current implementation uses an if/else block that chooses between fees types during liquidity removal:

  

```solidity

if (localData.lpTokenDepositValueChange > 0) {

    feePerLP = (uint256(localData.lpTokenDepositValueChange) * (uint256(feeDataArray[i].upliftFeeBps) * 1e18)) / 10000;

} else {

    feePerLP = (uint256(minWithdrawalFeeBps) * 1e18) / 10000;

}

```

  

When calculating fees for uplifted positions:

  

```solidity

if (localData.lpTokenDepositValueChange > 0) {

    feePerLP = (uint256(localData.lpTokenDepositValueChange) * (uint256(feeDataArray[i].upliftFeeBps) * 1e18)) / 10000;

}

```

  

Example scenario:

  

1. `minWithdrawalFeeBps` = 0.5% (50 bps)

2. `upliftFeeBps` = 50% (5000 bps)

3. MEV deposits 100e18

4. Gets 1% uplift (1e18)

5. Fee calculation: 1e18 \* 50% = 5e17 (0.05% of total deposit)

6. Actual fee (0.05%) < `minWithdrawalFeeBps` (0.5%)

  

This creates a gap where MEV can extract value while paying less than the intended minimum fee.

  

### Impact

  

The vulnerability enables:

  

1. MEV attacks with fees below intended minimum (for example, just in time liquidity for large swaps and feeless swaps attacks, etc)

2. Potential value extraction through rapid deposit/withdraw cycles attacks

  

### Tools Used

  

Manual  review

  

### Recommendations

  

add the `minWithdrawalFeeBps` to the `upliftFeeBps`

  

## [L-02] There is no function to change `upliftFeeBps`
### Summary

  

The `upliftFeeBps` variable in `UpliftOnlyExample` lacks a setter function, making it immutable after contract deployment despite the implementation suggesting it should be mutable (Storing data in `poolsFeeData` so that future `upliftFeeBps` doesn't retroactively affect them)

  

### Vulnerability Details

  

The contract stores `upliftFeeBps` in `poolsFeeData` for each position, indicating an intention to allow fee changes while preserving historical rates for existing positions. However, there is no function to modify the `upliftFeeBps` state variable after deployment.

  

### Impact

  

* Contract owner cannot adjust uplift fees to respond to market conditions

* Contradicts design intention to allow fee modifications

  

### Tools Used

  

Manual Review

  

### Recommendations

  

Add an owner-controlled setter function similar to `setHookSwapFeePercentage()`:

  

```solidity

function setUpliftFeeBps(uint64 newUpliftFeeBps) external onlyOwner {

    upliftFeeBps = newUpliftFeeBps;

    emit UpliftFeeBpsChanged(address(this), newUpliftFeeBps);

}

```

## [L-03] Last withdrawer will donate fees to empty pool allowing MEV and having always stuck funds in the pool
### Summary

  

in `onAfterRemoveLiquidity()` fees of the withdrawn tokens amounts (whatever its upLift or the minimum fees) of the last user withdrawing from the pool will be donated to empty pool.

  

### Vulnerability Details

  

> _**NOTE!**_:  the bug about `accruedQuantAMMFees` wrong distribution since Admin don't have position registered in `poolsFeeData` Is assumed to be solved by sending actual tokens to the admin

  

in `onAfterRemoveLiquidity()` The code calculate fees on the withdrawn tokens amounts (whatever its upLift or the minimum fees) and donates part of it to the pool and part of it to the QuantAMM admin, Here:

  

```solidity

File: UpliftOnlyExample.sol

            function onAfterRemoveLiquidity(

536:         if (localData.adminFeePercent > 0) {

537:             _vault.addLiquidity(

538:                 AddLiquidityParams({

539:                     pool: localData.pool,

540:                     to: IUpdateWeightRunner(_updateWeightRunner).getQuantAMMAdmin(),

541:                     maxAmountsIn: localData.accruedQuantAMMFees,

542:                     minBptAmountOut: localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18,

543:                     kind: AddLiquidityKind.PROPORTIONAL,

544:                     userData: bytes("")

545:                 })

546:             );

547:             emit ExitFeeCharged(

548:                 userAddress,

549:                 localData.pool,

550:                 IERC20(localData.pool),

551:                 localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18

552:             );

553:         }

554:

555:         if (localData.adminFeePercent != 1e18) {

556:             // Donates accrued fees back to LPs.

557:             _vault.addLiquidity(

558:                 AddLiquidityParams({

559:                     pool: localData.pool,

560:                     to: msg.sender,

561:                     maxAmountsIn: localData.accruedFees,

562:                     minBptAmountOut: 0,

563:                     kind: AddLiquidityKind.DONATION,

564:                     userData: bytes("")

  

567:         })

```

  

But the problem is that There are alot of circumstances where user withdrawing is the last withdrawer of the pool, what happens is:

  

1. User will get charged fees, part of it added to the admin (in the correct way) and part of it is donated to the pool

2. this will create an empty pool that have tokens in it with no corresponding BPT

3. MEV can immediately add liquidity (to mint BPT) and remove liquidity (burning the BPT and getting the initially deposited tokens Plus the stuck tokens)

4. The problem is yet that on MEV interactions, they will be charged fees and donated to the again empty pool (after MEV removing liquidity)

5. This will create some tokens in the end that is never retrievable and always stuck in the pool

  

> The above scenario describes what happens when the first mentioned bug get fixed by sending fees to the admin as actual tokens

  

Now if they decide to solve it by registering admin fee position in `poolsFeeData` then

  

1. Last withdrawer gets charged fees that part of it mint liquidity to admin, and part of it gets donated to the pool (that now have the admin liquidity, meaning that the admin owns all token Pools)

2. Admin comes to remove liquidity and he himself will be charged fee that again that of it mint liquidity to admin, and part of it gets donated to the pool&#x20;

3. Again and again and again (yet pool will have stuck funds either way after last withdrawer)

  

### Impact

  

Stuck funds in the pool

  

### Tools Used

  

Manual Review

  

### Recommendations

  

if the withdrawer is the last withdrawer (vault supply is 0) send all fees to the admin, or don't charge fees (like any pool does with the last withdrawer, they simply transfer all the balance of the pool to him)
