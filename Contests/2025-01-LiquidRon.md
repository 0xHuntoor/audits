
| ID                                                                                            | Title                                                                           |
| --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| [H-01](#h-01-operatorfeeamount-gets-accounted-in-totalassets-corrupting-the-accounting)       | `operatorFeeAmount` gets accounted in `totalAssets()` corrupting the accounting |
| [L-01](#l-01-risk-out-of-gas-reverts-during-deposit-and-withdraw-in-reasonable-circumstances) | risk Out-Of-Gas reverts during deposit and withdraw in reasonable circumstances |
| [L-02](#l-02-some-users-can-deposit-but-can-never-withdraw)                                   | some users can deposit but can never withdraw                                   |
## [H-01] `operatorFeeAmount` gets accounted in `totalAssets()` corrupting the accounting
### Finding description and impact

The `operatorFeeAmount` is not properly accounted for in `totalAssets()` calculation, leading to an inflated total assets value. This causes the vault to mint less shares than it should when users deposit, also this gives more assets to withdrawers as long as `feeRecepient` haven't claimed his fees yet.

The issue occurs because:

1. During harvest, `operatorFeeAmount` accumulates fees but is not subtracted from `totalAssets()`
2. `getTotalRewards()` only accounts for future fees via `operatorFee` but ignores already accumulated `operatorFeeAmount`
3. This leads to share price inflation and sudden share price drop when the `feeRecepient` claim his fees

Impact: 
1. Users receive fewer shares minted during deposit than they should when withdrawing shares, as the share price is artificially inflated.
2. If For example all users withdrew from the contract, `OperatorFeeAmounts` will be counted as compounded assets and will go to share withdrawers, `feeRecepient` will have nothing to claim
    - Not necessary that every one withdraw, but just to show the idea that `OperatorFeeAmounts` i used during `totalAssets()` accounting

### Proof of Concept
Add this test in a file in /tests

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test, console} from "forge-std/Test.sol";
import {LiquidRon, WithdrawalStatus} from "../src/LiquidRon.sol";
import {LiquidProxy} from "../src/LiquidProxy.sol";
import {WrappedRon} from "../src/mock/WrappedRon.sol";
import {MockRonStaking} from "../src/mock/MockRonStaking.sol";
import {IERC20} from "@openzeppelin/token/ERC20/IERC20.sol";

contract LiquidRonAccountingTest is Test {
    LiquidRon public liquidRon;
    WrappedRon public wrappedRon;
    MockRonStaking public mockRonStaking;

    address[] public consensusAddrs = [
        0xF000000000000000000000000000000000000001,
        0xf000000000000000000000000000000000000002,
        0xf000000000000000000000000000000000000003,
        0xF000000000000000000000000000000000000004,
        0xf000000000000000000000000000000000000005
    ];

    function setUp() public {
        mockRonStaking = new MockRonStaking();
        payable(address(mockRonStaking)).transfer(100_000_000 ether);
        wrappedRon = new WrappedRon();
        liquidRon = new LiquidRon(
            address(mockRonStaking),
            address(wrappedRon),
            250, // 2.5% operator fee
            address(this),
            "Test",
            "TST"
        );
        liquidRon.deployStakingProxy();
        liquidRon.deployStakingProxy();
        liquidRon.deployStakingProxy();
    }

function test_wrong_accounting() public {
    wrappedRon.approve(address(liquidRon), type(uint256).max);

    // Initial deposit to have funds when calling delegate amounts
    deal(address(wrappedRon), address(this), 200 ether);
    uint256 initialDeposit = 100 ether;
    liquidRon.deposit{value: initialDeposit}();
    uint256[] memory amounts = new uint256[](5);
    for (uint256 i = 0; i < 5; i++) {
        amounts[i] = 10 ether;
    }
    liquidRon.delegateAmount(0, amounts, consensusAddrs);
    
    // Generate rewards
    skip(365 days);

    // This is the first deposit that we will compare with, should be the same as the second since rewards and operator fees is now static and have been already generated
    uint256 newDeposit = 10 ether;
    vm.deal(address(this), initialDeposit);
    liquidRon.deposit{value: initialDeposit}();
    uint256 initialShares = liquidRon.deposit(newDeposit, address(this));
    
    // Setup staking
    
    // Before harvest
    uint256 preTotalAssets = liquidRon.totalAssets();
    
    // Harvest with 2.5% fee
    liquidRon.harvest(0, consensusAddrs);
    
    // After harvest
    uint256 postTotalAssets = liquidRon.totalAssets();
    uint256 operatorFeeAmount = liquidRon.operatorFeeAmount();
    
    // New user deposits get fewer shares due to inflated total assets


    vm.deal(address(this), initialDeposit);
    liquidRon.deposit{value: initialDeposit}();

    //i second mint call after harvesting the rewards, this will have less shares minted since operatorFeeAmounts are not included in totalAssets and inflate vault holdings
    uint256 sharesMinted = liquidRon.deposit(newDeposit, address(this));
    
    // Shares minted are worth less than they should be
    uint256 assetsForShares = liquidRon.previewRedeem(sharesMinted);
    assertLt(sharesMinted, initialShares);
}

receive() external payable {}
}
```
after applying the fix
```solidity
File: LiquidRon.sol
293:     function totalAssets() public view override returns (uint256) {
294:         return super.totalAssets() + getTotalStaked() + getTotalRewards() - operatorFeeAmount;
295:     }
```
try this test with the assertEq this time, and it will pass

### Recommended mitigation steps

Modify `totalAssets()` to subtract `operatorFeeAmount`:

```solidity
function totalAssets() public view override returns (uint256) {
    return super.totalAssets() + getTotalStaked() + getTotalRewards() - operatorFeeAmount;
}
```

This ensures that assets already allocated as fees are not counted in the total assets calculation.

## [L-01] risk Out-Of-Gas reverts during deposit and withdraw in reasonable circumstances
### Finding Description and Impact

The `totalAssets()` function in `LiquidRon.sol` iterates over all staking proxies and consensus addresses to calculate the total assets controlled by the contract. This function is called during critical operations such as deposits and withdrawals. However, if the number of staking proxies or consensus addresses is large, the function can consume excessive gas, potentially exceeding the Ethereum block gas limit (30M gas). This can lead to out-of-gas (OOG) reverts, rendering the contract unusable for deposits and withdrawals in high-scale scenarios.

The most reasonable numbers i could come across are 60 validator and 100 staking proxy deployed: 
- While this seems large:
   - The nature of staking protocols usually involve more than 100 validators
   - If the TVL of the LiquidRonin is increasing and multiple user interactions are happening daily, they will need to deploy more proxies
- The above two points make the bug more likely to rise
  
**Impact:**
- **Denial of Service (DoS)**: If `totalAssets()` reverts due to OOG, users will be unable to deposit or withdraw funds, effectively freezing the contract temporarily till the operator claim and undelegate from number of operators to delete some them to decrease the iteration numbers on consensus adresses  .
- **Scalability Issues**: The contract cannot handle a large number of staking proxies or consensus addresses, limiting its scalability.
- **User Funds at Risk**: If withdrawals are blocked due to OOG reverts, users may be unable to access their funds. (same as first point)

### Proof of Concept

paste this in `LiquidRon.t.sol`

```solidity
    function test_totalAssets_OOG() public {
    // Deploy multiple staking proxies
    uint256 proxyCount = 100; // Adjust this number to test different scenarios
    for (uint256 i = 0; i < proxyCount; i++) {
        liquidRon.deployStakingProxy();
    }

    // Add a large number of consensus addresses
    uint256 consensusCount = 60; // Adjust this number to test different scenarios
    address[] memory consensusAddrs = new address[](consensusCount);
    for (uint256 i = 0; i < consensusCount; i++) {
        consensusAddrs[i] = address(uint160(i + 1)); // Generate unique addresses
    }

    // Deposit some RON to initialize the system
    uint256 depositAmount = 1000000000000000000000000000000 ether;

    deal(address(this), depositAmount * 10);
    
    liquidRon.deposit{value: depositAmount * 10}();

    // Delegate amounts to consensus addresses
    uint256[] memory amounts = new uint256[](consensusCount);
    for (uint256 i = 0; i < consensusCount; i++) {
        amounts[i] = 1;
    }
    for (uint256 i = 0; i < proxyCount; i++) {
        liquidRon.delegateAmount(i, amounts, consensusAddrs);
    }

    // Call totalAssets() and check for OOG reverts
    uint256 blockGasLimit = 30_000_000;
    uint256 totalAssets;
    // passing the block gas limit as a parameter to simulate a real enviroment block limit
    try liquidRon.totalAssets{gas: blockGasLimit}() returns (uint256 _totalAssets) {
        totalAssets = _totalAssets;
    } catch {
        revert("OOG in totalAssets()");
    }

    // Assert that totalAssets is greater than 0
    assertTrue(totalAssets > 0, "totalAssets should be greater than 0");
}
```
#### Result:

The test fails with an `OutOfGas` error, demonstrating that `totalAssets()` consumes excessive gas and reverts when the number of staking proxies and consensus addresses is large.

### Recommended Mitigation Steps

1. **Optimize `totalAssets()` Function**:

- **Cache Results**: Cache the results of expensive operations (e.g., staked amounts and rewards) to avoid recalculating them on every call.
- **Batch Processing**: Process staking proxies and consensus addresses in batches to reduce gas consumption per transaction.
- **Off-Chain Calculation**: Use an off-chain service to calculate total assets and provide the result to the contract via a trusted oracle.

2. **Limit the Number of Proxies and Consensus Addresses**:

- **Enforce Limits**: Set a maximum limit on the number of staking proxies and consensus addresses that can be added to the contract.
- **Prune Inactive Addresses**: Regularly prune inactive consensus addresses to reduce the number of iterations in `totalAssets()`.
  
## [L-02] some users can deposit but can never withdraw
### Finding description and impact
in `LiquidRon` the contract allow users to `deposit()` or `mint()` using the `wRON` token, but during `withdraw()` or `burn()` users are forced to receive native `RON` or the txn will revert:
- This is a problem cause the depositor may have been a contract that doesn't implement `receive()` or payable functions
- The depositor may have been a protocol that is building on top of liquidRon too

This makes the above two cases to never be able to get back any tokens and have their funds stuck 
### Proof of Concept
1. A contract not having receive or payable functions deposit in liquidRon using wRON
2. Time passes and he wants to withdraw
3. Any call to withdraw() or redeem() will revert during `_withdrawRONTo()` call
```solidity
File: RonHelper.sol
38:     function _withdrawRONTo(address to, uint256 amount) internal {
39:         IWRON(wron).withdraw(amount);
40:         (bool success, ) = to.call{value: amount}("");
41:         if (!success) revert ErrWithdrawFailed();
42:     }
```
1. Funds stuck forever

### Recommended mitigation steps
Wrap the native call that if failed, wrap the native tokens and send them to the receiver as wRON
