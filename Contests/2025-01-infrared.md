| ID                                                                                                               | Title                                                                                                        |
| ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [H-01](#h-01-validator-front-running-vulnerability-in-withdrawal-credentials-registration)                       | Validator Front-Running Vulnerability in Withdrawal Credentials Registration                                 |
| [H-02](#h-02-infraredaddvalidatorswill-be-dosed-due-to-insufficientshareholderfees)                              | `infrared::addValidators()` will be DOSed due to insuffecient shareholderFees                                |
| [H-03](#h-03-effect-before-interaction-is-utilised-wrongly-causing-loss-of-funds)                                | Effect before interaction is utilised wrongly causing loss of funds                                          |
| [H-04](#h-04-stuck-funds-inwrappedvaultdue-to-bad-handling-of_token--assetcondition)                             | Stuck funds in WrappedVault due to bad handling of `_token == asset` condition                               |
| [M-01](#m-01-fee-on-transfer-token-incompatibility-during-boost-reward-distribution)                             | Fee-On-Transfer Token Incompatibility During Boost Reward Distribution                                       |
| [M-02](#m-02-dos-attack-oninfraredupdaterewardsdurationforvault)                                                 | DOS attack on `Infrared::updateRewardsDurationForVault()`                                                    |
| [M-03](#m-03-ininfraredberadepositorexecuteattacker-can-force-usage-of-invalid-signature-to-cause-loss-of-funds) | In `InfraredBERADepositor::execute()`Attacker can Force usage of invalid signature to cause loss of funds |
| [L-01](#l-01-setfeedivisorshareholdersapply-fees-retroactively)                                                  | `setFeeDivisorShareholders()` apply fees retroactively                                                       |
| [L-02](#l-02-when-pausing-red-some-vaults-will-lose-on-rewards-and-other-wont-onharvestvault)                    | When pausing RED, some vaults will lose on rewards and other won't on `harvestVault()`                       |
| [L-03](#l-03-dust-amounts-can-accumulate-ininfrareddistributor)                                                  | Dust Amounts can accumulate in `InfraredDistributor`                                                         |
| [I-01](#i-01-ininfraredclaimlostrewardsonvaultlost-vault-rewards-are-handled-inappropriately)                    | In `infrared::claimLostRewardsOnVault()` Lost Vault rewards are handled inappropriately                      |
| [I-02](#i-02-missing-fee-application-on-infraredbera-portion-of-collected-bribes)                                | Missing Fee Application on InfraredBERA Portion of Collected Bribes                                          |
| [I-03](#i-03-payout-token-configurability-vs-implementation-constraints)                                         | Payout Token Configurability vs Implementation Constraints                                                   |
## [H-01] Validator Front-Running Vulnerability in Withdrawal Credentials Registration
### Summary

in `InfraredBERADepositor::execute()`, a Validator can front run `IBeaconDeposit(DEPOSIT_CONTRACT).deposit` by sending a deposit with another withdrawal credentials other than the `withdrawor` address, sense the withdrawal credential is registered on the first valid deposit to the beacon chain

### Finding Description

in `InfraredBERADepositor::execute()`, withdrawal credentials are set to the `withdrawor` address contract here

```solidity
File: InfraredBERADepositor.sol
140:         address withdrawor = IInfraredBERA(InfraredBERA).withdrawor();
////////////
182:         bytes memory credentials = abi.encodePacked(
183:             ETH1_ADDRESS_WITHDRAWAL_PREFIX,
184:             uint88(0), // 11 zero bytes
185:             withdrawor
186:         );
/////////////
191:         IBeaconDeposit(DEPOSIT_CONTRACT).deposit{value: amount}(
192:             pubkey, credentials, signature, operator
193:         );
```

But the problem is that `credentials` are registered one time only per validator and can't be changed after the first valid deposit as seen here

```go
File: state_processor_staking.go
090: func (sp *StateProcessor[_]) applyDeposit(st *state.StateDB, dep *ctypes.Deposit) error {
091: 	idx, err := st.ValidatorIndexByPubkey(dep.GetPubkey())
092: 	if err != nil {
093: 		sp.logger.Info("Validator does not exist so creating",
094: 			"pubkey", dep.GetPubkey(), "index", dep.GetIndex(), "deposit_amount", dep.GetAmount())
095: 		// If the validator does not exist, we add the validator.
096: 		// TODO: improve error handling by distinguishing
097: 		// ErrNotFound from other kind of errors
098: 		return sp.createValidator(st, dep)
099: 	}
100: 
101: 	// if validator exist, just update its balance
102: 	if err = st.IncreaseBalance(idx, dep.GetAmount()); err != nil {
103: 		return err
104: 	}
105: 
106: 	sp.logger.Info(
107: 		"Processed deposit to increase balance",
108: 		"deposit_amount", float64(dep.GetAmount().Unwrap())/math.GweiPerWei,
109: 		"validator_index", idx,
110: 	)
111: 	return nil
112: }
```

Now a validator can monitor the memepool and front run `InfraredBERADepositor::execute()` by directly calling the `DEPOSIT_CONTRACT` with another withdrawal signature of his mine

```solidity
File: BeaconDeposit.sol
91:     function deposit(
92:         bytes calldata pubkey,
93:         bytes calldata credentials,
94:         bytes calldata signature,
95:         address operator
96:     )
```

Making all the funds deposited by infrared protocol after that txn control by him

### Impact Explanation

High, loss of funds to the protocol

### Likelihood Explanation

High:

- Any validator can execute the attack easily
- Infra red protocol won't notice and will keep depositing to that validator normally

### Recommendation (optional)

see [https://research.lido.fi/t/mitigations-for-deposit-front-running-vulnerability/1239](https://research.lido.fi/t/mitigations-for-deposit-front-running-vulnerability/1239) Options

## [H-02] `infrared::addValidators()` will be DOSed due to insufficient `shareholderFees`
### Summary

in `infrared::addValidators()`, there can be multiple circumstances in which the call will fail due to the flow of `harvestOperatorRewards()` called inside it

### Finding Description

in `infrared::addValidators()` we call `harvestOperatorRewards()` inside it to have up-to-date operator rewards and not to give old rewards to the new validator

`harvestOperatorRewards()` calls `IInfraredBERA(ibera).compound()` to have up-to-date `shareholderFees`

Then `IInfraredBERA(ibera).collect()` is called that in the end calls `InfraredBERAFeeReceivor::collect()`

Now in `InfraredBERAFeeReceivor::collect()` we see

```solidity
File: InfraredBERAFeeReceivor.sol
083:     function collect() external returns (uint256 sharesMinted) {
084:         if (msg.sender != InfraredBERA) revert Errors.Unauthorized(msg.sender);
085:         uint256 shf = shareholderFees;
086:         if (shf == 0) return 0;
087: 
088:         uint256 min = InfraredBERAConstants.MINIMUM_DEPOSIT
089:             + InfraredBERAConstants.MINIMUM_DEPOSIT_FEE;
090:         if (shf < min) {
091:             revert Errors.InvalidAmount();
092:         }
093: 
094:         if (shf > 0) {
095:             delete shareholderFees;
096:             (, sharesMinted) =
097:                 IInfraredBERA(InfraredBERA).mint{value: shf}(address(infrared));
098:         }
099:         emit Collect(address(infrared), shf, sharesMinted);
100:     }
```

if the `shareholderFees` is < `min` we revert the txn

This opens up two problems

#### Problem 1

#### Admins gets delayed by coincidence

If by coincidence the `shareholderFees` < `min` when the admin wants to add new validator, the admins have two options

1. Admin can wait till the `InfraredBERAFeeReceivor` accumulate more native tokens to build up on `shareholderFees` during `sweep()`
2. Admin directly donate Native tokens (losing funds) to `InfraredBERAFeeReceivor` to build up on `shareholderFees` during `sweep()` and the `addValidators()` succeed

#### Problem 2

##### Attack griefer

an attacker sees admin txn in the meme pool of `addValidators()`

1. attacker front run it by calling `infrared::harvestOperatorRewards()` directly (emptying any `shareholderFees` that would have been sufficient to the admin txn to pass)
2. Donate amounts that just makes `shareholderFees` non 0 but still < `min`
3. Admin txn revert

No admins have again the same two options, wait, or donate

If they go with the donate part, the attacker can repeat the same steps to greive admins

- Cause the attacker loss on repeating the attacker is 1/3 the cost of admins donation to mitigate the issue (to build up a `shareholderFees` that is > `min`)

### Impact Explanation

Medium:

- Loss of funds to admins
- Function unavailability

### Likelihood Explanation

High:

- Easy to carry attack with no special conditions

### Proof of Concept (if required)

Add this to `InfraredTest.t.sol`

```solidity
    function testAddinValidatorDOS() public {
        deal(address(ibera.receivor()), 15e18);
        // 1. Add a validator to the validator set
        vm.expectRevert();
        vm.startPrank(infraredGovernance);
        ValidatorTypes.Validator memory validator_str = ValidatorTypes.Validator({
            pubkey: "0x1234567890abcdef",
            addr: address(validator)
        });
        ValidatorTypes.Validator[] memory validators =
            new ValidatorTypes.Validator[](1);
        validators[0] = validator_str;
        bytes[] memory pubkeys = new bytes[](1);
        pubkeys[0] = validator_str.pubkey;
        infrared.addValidators(validators);
        vm.stopPrank();
    }
```

### Recommendation (optional)

if its below min, just return

```diff
        if (shf < min) {
-           revert Errors.InvalidAmount();
+           return 0
        }
```
## [H-03] Effect before interaction is utilised wrongly causing loss of funds
### Description

in `MultiRewards::getRewardForUser()`, User rewards are deleted even if the transfer has failed

```solidity
File: MultiRewards.sol
225:     function getRewardForUser(address _user)
226:         public
227:         nonReentrant
228:         updateReward(_user)
229:     {
230:         onReward();
231:         uint256 len = rewardTokens.length;
232:         for (uint256 i; i < len; i++) {
233:             address _rewardsToken = rewardTokens[i];
234:             uint256 reward = rewards[_user][_rewardsToken];
235:             if (reward > 0) {
236:                 rewards[_user][_rewardsToken] = 0;
237:                 (bool success, bytes memory data) = _rewardsToken.call(
238:                     abi.encodeWithSelector(
239:                         ERC20.transfer.selector, _user, reward
240:                     )
241:                 );
242: 
243:                 if (success && (data.length == 0 || abi.decode(data, (bool)))) {
244:                     emit RewardPaid(_user, _rewardsToken, reward);
245: 
246:                 } else {
247:                     continue;
248:                 }
249:             }
250:         }
251:     }
```

This is a big problem for two reasons

1. Paused token (RED for example) rewards for users will get lost on failed transfer
2. Attacker can utilize the 63/64 Rule to revert the `_rewardsToken.call` due to Out of gas (needs a token with transfer Logic that consumes high gas to leave a suffecient 1/64 gas to the remaining logic) **Less likely, but a valid concern**

Now its known that its more likely for RED to be paused for some reasons, other whitelisted rewardTokens can be the same too, once this happens, attack can cause loss of all accumulated rewards for users by calling `getRewardForUser()` for most high staked users

Reward tokens lost won't be retreivable from the contract, since RewardTokens generally added can't be removed after, and recovering RewardTokens is prohibited as seen here

```solidity
File: MultiRewards.sol
344:     function _recoverERC20(
345:         address to,
346:         address tokenAddress,
347:         uint256 tokenAmount
348:     ) internal {
349:         require(
350:             rewardData[tokenAddress].lastUpdateTime == 0,
351:             "Cannot withdraw reward token"
352:         );
353:         ERC20(tokenAddress).safeTransfer(to, tokenAmount);
354:         emit Recovered(tokenAddress, tokenAmount);
355:     }
```

Making any token lost on those instances or attacks to be stuck forever in the contract

### Recommendation

only zero out the rewards inside the `if` statement of the success

```diff
    function getRewardForUser(address _user)
-               rewards[_user][_rewardsToken] = 0;
                (bool success, bytes memory data) = _rewardsToken.call(
                    abi.encodeWithSelector(
                        ERC20.transfer.selector, _user, reward
                    )
                );
                
                if (success && (data.length == 0 || abi.decode(data, (bool)))) {
                    emit RewardPaid(_user, _rewardsToken, reward);
+               rewards[_user][_rewardsToken] = 0;
                } else {

                    continue;
```

## [H-04] Stuck funds in `WrappedVault` due to bad handling of `_token == asset` condition
### Summary

in `WrappedVault::claimRewards()`, if the reward token is the staking token, that amount is stuck forever

### Finding Description

**First we have to establish that having a staking token the same as the reward token in `infraredVault` is a normal behavior for the following reasons:**

1. The scenario is tried to be handled in `WrappedVault::claimRewards()` by the check in line 116

```solidity
File: WrappedVault.sol
116:             if (_token == asset) continue;
```

1. During `infraredVault::constructor()` we always force set iBGT as a reward token, meaning there can be iBGT staking vault with iBGT as a reward
2. Same as 2 for RED, since its permessionless to add RED token as a reward token once its set in the contract during `harvestVault()`

```solidity
File: RewardsLib.sol
183:                     (, uint256 redRewardsDuration,,,,,) = vault.rewardData(red);
184:                     if (redRewardsDuration == 0) {
185:                         // Add RED as a reward token if not already added
186:                         vault.addReward(red, rewardsDuration);
187:                     }
```

1. `MultiRewards` contract is actually compliant with such scenario since it separates the accounting of both staking and reward tokens
2. Its not prevented at `infrared::addReward()` function (although its `onlyGovernor` function)

##### Now back to the Bug description

when `WrappedVault::claimRewards()` is called, the contract is sent all the reward token he is eligible for:

```solidity
File: WrappedVault.sol
106:     function claimRewards() external {
107:         // Claim rewards from the InfraredVault
108:         iVault.getReward();

File: MultiRewards.sol
225:     function getRewardForUser(address _user)
226:         public
227:         nonReentrant
228:         updateReward(_user)
229:     {
230:         onReward();
231:         uint256 len = rewardTokens.length;
232:         for (uint256 i; i < len; i++) {
233:             address _rewardsToken = rewardTokens[i];
234:             uint256 reward = rewards[_user][_rewardsToken];
235:             if (reward > 0) {
236:                 rewards[_user][_rewardsToken] = 0;
237:                 (bool success, bytes memory data) = _rewardsToken.call(
238:                     abi.encodeWithSelector(
239:                         ERC20.transfer.selector, _user, reward
240:                     )
241:                 );
242:                 if (success && (data.length == 0 || abi.decode(data, (bool)))) {
243:                     emit RewardPaid(_user, _rewardsToken, reward);
244:                 } else {
245:                     continue;
246:                 }
247:             }
248:         }
249:     }
```

then in `WrappedVault::claimRewards()` we check if the reward token is the same as staking token, if its true, we don't send those tokens to the distributor

```solidity
File: WrappedVault.sol
113:         for (uint256 i; i < len; ++i) {
114:             ERC20 _token = ERC20(_tokens[i]);
115:             // Skip if the reward token is the staking token <<@HERE
116:             if (_token == asset) continue;
117:             uint256 bal = _token.balanceOf(address(this));
118:             if (bal == 0) continue;
119:             (bool success, bytes memory data) = address(_token).call(
120:                 abi.encodeWithSelector(
121:                     ERC20.transfer.selector, rewardDistributor, bal
122:                 )
123:             );
```

This is done due to thinking that leaving this reward token in the contract will increase the share price of the vault, **BUT THIS IS WRONG**

Any left tokens in the contract won't be counted for during share price calculation in withdraw, why?:

Since the contract has overridden `totalAssets()` function in the following way

```solidity
File: WrappedVault.sol
74:     function totalAssets() public view virtual override returns (uint256) {
75:         return iVault.balanceOf(address(this)) + deadShares;
76:     }
```

The problem here is that `iVault.balanceOf(address(this))` doesn't actually return the current token balance of the contract, but return the balance deposited to `infraredVault`

```solidity
File: MultiRewards.sol
180:     function stake(uint256 amount)
181:         external
182:         nonReentrant
183:         whenNotPaused
184:         updateReward(msg.sender)
185:     {
186:         require(amount > 0, "Cannot stake 0");
187:         _totalSupply = _totalSupply + amount;
188:         _balances[msg.sender] = _balances[msg.sender] + amount;
189: 
190:         // transfer staking token in then hook stake, for hook to have access to collateral
191:         stakingToken.safeTransferFrom(msg.sender, address(this), amount);
192:         onStake(amount);
193:         emit Staked(msg.sender, amount);
194:     }


114:     function balanceOf(address account)
115:         external
116:         view
117:         returns (uint256 _balance)
118:     {
119:         return _balances[account];
120:     }
```

### Impact Explanation

High:

- Loss of funds for every reward token claim that is the same as the staking token

### Likelihood Explanation

High:

- on every `claimRewards()` call, which is permissionless

### Recommendation (optional)

Either refactor `totalAssets()` or check balance before and after `iVault.getReward();` for the staking token and send the changed balance to `rewardDistributor`

## [M-01] Fee-On-Transfer Token Incompatibility During Boost Reward Distribution
### Summary

The boost reward distribution fails to account for Fee-On-Transfer (FOT) tokens in the `MultiRewards` contract, leading to contract insolvency when distributing rewards.

### Finding Description

The system implements two different approaches for handling reward tokens. In RewardsLib, when harvesting boost rewards, the code correctly accounts for FOT tokens by measuring the actual balance change:

```solidity
// RewardsLib.sol
function harvestBoostRewards(...) {
    uint256 balanceBefore = ERC20(_token).balanceOf(address(this));
    _bgtStaker.getReward();
    _amount = ERC20(_token).balanceOf(address(this)) - balanceBefore;
}
```

However, in the `MultiRewards` contract, which handles the actual reward distribution, the implementation assumes the transferred amount equals the received amount:

```solidity
// MultiRewards.sol
function _notifyRewardAmount(address _rewardsToken, uint256 reward) {
    ERC20(_rewardsToken).safeTransferFrom(msg.sender, address(this), reward);
    rewardData[_rewardsToken].rewardRate = reward / rewardsDuration;
}
```

This discrepancy creates an accounting mismatch where the contract records rewards based on pre-fee amounts while actually holding post-fee balances. The reward rate calculation uses the full amount without considering potential transfer fees, leading to an overestimation of available rewards.

Also same problem when distributing the voterFee part of this boost reward during `_distributeFeesOnRewards()`

```solidity
File: RewardsLib.sol
100:         if (_amtVoter > 0) {
101:             address voterFeeVault = IVoter(_voter).feeVault();
102:             ERC20(_token).safeApprove(voterFeeVault, _amtVoter);
103:             IReward(voterFeeVault).notifyRewardAmount(_token, _amtVoter);
104:         }
File: Reward.sol
329:     function _notifyRewardAmount(address sender, address token, uint256 amount)
330:         internal
331:     {
332:         if (amount == 0) revert ZeroAmount();
333:         ERC20(token).safeTransferFrom(sender, address(this), amount);
334: 
335:         uint256 epochStart = VelodromeTimeLibrary.epochStart(block.timestamp);
336:         tokenRewardsPerEpoch[token][epochStart] += amount;
337: 
338:         emit NotifyReward(sender, token, epochStart, amount);
339:     }
```

### Impact Explanation

- Contract becomes insolvent for FOT tokens
- Last user cannot claim any rewards for that token

### Likelihood Explanation

Medium likelihood, if any FOT tokens are used as rewards. The issue occurs because:

1. Initial balance check in RewardsLib correctly accounts for fees
2. But reward rate calculation uses pre-fee amount

### Recommendation

Modify `_notifyRewardAmount()` to use post-transfer balance:

```solidity
function _notifyRewardAmount(address _rewardsToken, uint256 reward) {
    uint256 balanceBefore = ERC20(_rewardsToken).balanceOf(address(this));
    ERC20(_rewardsToken).safeTransferFrom(msg.sender, address(this), reward);
    uint256 actualAmount = ERC20(_rewardsToken).balanceOf(address(this)) - balanceBefore;
    
    rewardData[_rewardsToken].rewardRate = actualAmount / rewardsDuration;
}
```

## [M-02] DOS attack on `Infrared::updateRewardsDurationForVault()`
### Summary

A permissionless `Infrared::addIncentives()` function allows any user to continuously extend reward periods, effectively blocking governance's ability to update reward durations through `Infrared::updateRewardsDurationForVault()`.

### Finding Description

The attack flow:

1. `addIncentives()` in `Infrared` is permissionless
2. It transfers tokens from caller and calls `notifyRewardAmount()` on the vault
3. `notifyRewardAmount()` in `InfraredVault` extends the `periodFinish` by `rewardsDuration` from current timestamp
4. `updateRewardsDurationForVault()` requires `periodFinish < block.timestamp` to succeed
5. Attacker can repeatedly call `addIncentives()` with minimal amounts to keep extending `periodFinish`

Key code flow:

```solidity
// Permissionless entry point
function addIncentives(address _stakingToken, address _rewardsToken, uint256 _amount) external {
    _vaultStorage().addIncentives(_stakingToken, _rewardsToken, _amount);
}

// Extends periodFinish in _notifyRewardAmount
rewardData[_rewardsToken].periodFinish = block.timestamp + rewardData[_rewardsToken].rewardsDuration;

// Blocked by extended periodFinish
require(block.timestamp > rewardData[_rewardsToken].periodFinish, "Reward period still active");
```

**Attack Path**:

Imagine a `rewardsDuration` = 7 days

after first Notify of rewards, the Period Finish will be 7 days from now

Attacker keeps adding incentive every day to keep `periodFinish` extended

Governance wants to change `periodFinish` by pausing the vault:

- This solution is worse due to that it Pause the vault for time = `periodFinish` which is 7 Days, propagating the DOS effect to all functionalities of `InfraredVault`

### Impact Explanation

- Governance cannot update reward durations for vaults

### Likelihood Explanation

High likelihood as:

- No access controls on `addIncentives()`
- Attack requires minimal capital
- Can be executed repeatedly
- No cooldown periods between reward notifications

### Recommendation

Add access controls to `addIncentives()`

## [M-03] in `InfraredBERADepositor::execute()`Attacker can Force usage of invalid signature to cause loss of funds
### Summary

in `InfraredBERADepositor::execute()` Attacker can trich the contract that the validator was registered previously in the beacon chain forcing the contract to use invalid signature upon deposits, causing loss of funds

### Finding Description

in `InfraredBERADepositor::execute()` there is a check to see if this is the first time we deposit to the `pubkey` of the validator or not here

```solidity
File: InfraredBERADepositor.sol
122:             IBeaconDeposit(DEPOSIT_CONTRACT).getOperator(pubkey);
123:         // Add first deposit validation
124:         if (currentOperator == address(0)) {
125:             if (amount != InfraredBERAConstants.INITIAL_DEPOSIT) {
126:                 revert Errors.InvalidAmount();
```

This check make sure that if the validator is not registered before, we enforce strict `INITIAL_DEPOSIT` why?

- Since the signature stored at `IInfraredBERA(InfraredBERA).signatures` includes `INITIAL_DEPOSIT` as the signed amount so that the signature is valid

The problem Here is that any attacker can call `DEPOSIT_CONTRACT` to register `operator` for any pubKey

```solidity
File: BeaconDeposit.sol
084:     function deposit(
085:         bytes calldata pubkey,
086:         bytes calldata credentials,
087:         bytes calldata signature,
088:         address operator
089:     )
090:         external
091:         payable
092:     {
093:         if (pubkey.length != PUBLIC_KEY_LENGTH) {
094:             InvalidPubKeyLength.selector.revertWith();
095:         }
096: 
097:         if (credentials.length != CREDENTIALS_LENGTH) {
098:             InvalidCredentialsLength.selector.revertWith();
099:         }
100: 
101:         if (signature.length != SIGNATURE_LENGTH) {
102:             InvalidSignatureLength.selector.revertWith();
103:         }
104: 
105:         // Set operator on the first deposit.
106:         // zero `_operatorByPubKey[pubkey]` means the pubkey is not registered.
107:         if (_operatorByPubKey[pubkey] == address(0)) {
108:             if (operator == address(0)) {
109:                 ZeroOperatorOnFirstDeposit.selector.revertWith();
110:             }
111:             _operatorByPubKey[pubkey] = operator;
112:             emit OperatorUpdated(pubkey, operator, address(0));
113:         }
114:         // If not the first deposit, operator address must be 0.
115:         // This prevents from the front-running of the first deposit to set the operator.
116:         else if (operator != address(0)) {
117:             OperatorAlreadySet.selector.revertWith();
118:         }
119: 
120:         uint64 amountInGwei = _deposit();
121: 
122:         if (amountInGwei < MIN_DEPOSIT_AMOUNT_IN_GWEI) {
123:             InsufficientDeposit.selector.revertWith();
124:         }
125: 
126:         // slither-disable-next-line reentrancy-benign,reentrancy-events
127:         emit Deposit(pubkey, credentials, amountInGwei, signature, depositCount++);
128:     }
```

Attacker will call deposit for the `pubKey` of the registered validator that didn't have any previous deposits with `MIN_DEPOSIT_AMOUNT_IN_GWEI` and operator to be `infrared` contract address and any credentials and signature with the proper lenght

txn will succeed, Operator is registered, But validator is not activated nor had any valid deposits (not registered in the beacon chain yet) since the attacker provided wrong signature

Now if the keeper calls `execute` with that pubKey and any amount, the txn will go through and will send the BERA amount not enforcing `INITIAL_DEPOSIT`

- but this cause loss of funds since the registered signature of the first valid deposit is meant to be associated with `INITIAL_DEPOSIT`, the deposit will fail at the beacon chain

Also the attacker himself can call `execute()` for that `pubkey` if `_enoughtime()` is true

### Impact Explanation

High, Complete loss of funds of any `InfraredBERADepositor` live balance

### Likelihood Explanation

High, any attacker can carry out the attack

### Recommendation (optional)

check if the pubKey has had his first deposit locally in `InfraredBERADepositor` and don't depend on having just an operator registered for him

## [L-01] `setFeeDivisorShareholders()` apply fees retroactively
### Description

The `setFeeDivisorShareholders()` function in `InfraredBERA` allows updating the fee divisor that determines the fee taken from rewards.

When there are non swept funds in the `FeeReceivor` contract (as shown in the `sweep()` function), changing the fee divisor will retroactively apply the new fee rate to already earned rewards that haven't been swept yet. This means rewards earned under one fee structure could end up having a different fee applied when they are eventually swept.

the fee calculation in `distribution()` uses the current fee divisor value as seen in Line 59

```solidity
File: InfraredBERAFeeReceivor.sol
69:     function sweep() external returns (uint256 amount, uint256 fees) {
70:         (amount, fees) = distribution();
71:         // do nothing if InfraredBERA deposit would revert
72:         uint256 min = InfraredBERAConstants.MINIMUM_DEPOSIT
73:             + InfraredBERAConstants.MINIMUM_DEPOSIT_FEE;
74:         if (amount < min) return (0, 0);
75: 
76:         // add to protocol fees and sweep amount back to ibera to deposit
77:         if (fees > 0) shareholderFees += fees;
78:         IInfraredBERA(InfraredBERA).sweep{value: amount}();
79:         emit Sweep(InfraredBERA, amount, fees);
80:     }


52:     function distribution()
53:         public
54:         view
55:         returns (uint256 amount, uint256 fees)
56:     {
57:         amount = (address(this).balance - shareholderFees);
58:         uint16 feeShareholders =
59:             IInfraredBERA(InfraredBERA).feeDivisorShareholders(); <<@HERE
60: 
61:         // take protocol fees
62:         if (feeShareholders > 0) {
63:             fees = amount / uint256(feeShareholders);
64:             amount -= fees;
65:         }
66:     }
```

### Recommendation

call `compound()` inside `setFeeDivisorShareholders()`

## [L-02] When pausing RED, some vaults will lose on rewards and other won't on `harvestVault()`
### Summary

in `infrared::harvestVault()`, when RED token is paused, some vaults will lose on RED tokens while others won't, causing unfair reward distribution and loss of rewards (RED) for some users

### Finding Description

during `infrared::harvestVault()` for specific staking token, we wrap the RED token minting logic in a try/catch method to avoid pausing reward distribution logic hook on `infraredVault` when RED token gets paused

the problem in this implementation is that if RED token gets paused, it retroactively affects previous rewards

- Since rewards stake their tokens first, then they get credited `iBGT` during `rewardsLib::harvestVault()`
- RED minted to that vault is a portion of the iBGT rewards as seen from the mintRate calculation

```solidity
File: RewardsLib.sol
174:         uint256 mintRate = $.redMintRate;

179:             uint256 redAmt = bgtAmt * mintRate / RATE_UNIT;
```

- Now there can be two `infraredVault` of wETH, wBTC staking for 1 week
    - The vault earned 100e18 BGT
    - RED mint rate is 100%
- Admin submit a txn to pause RED token
    - a staker of infraredVault of wBTC front run it calling `harvestVault()` to not lose that RED mints to his vault
    - Admin txn pause RED after
    - The same user call `harvestVault()` to all other `infraredVaults()` to force them to lose on those RED minted proportionally to iBGT, he is incentivized to do so to not dilute the total supply of RED and make the RED sent to his own vault more valuable

### Impact Explanation

Medium:

- Users lose rewards relative to other vaults in some conditions

### Likelihood Explanation

Low:

- Pausing RED generally is a low probability event

### Recommendation (optional)

Either:

1. try to implement a script off chain to call `harvestVault()` to all current vaults
    
2. implement a loop to call `harvestVault()` to all staking token inside RED pause Logic
    
3. store an accounting of the not minted RED
    

> **_NOTE!:_** Its worth noting that excluding mint() call of RED won't work, since we will need transfer call when calling `vault.notifyRewardAmount()` after the mint inside the try block

## [L-03] Dust Amounts can accumulate in `InfraredDistributor`

### Description

There is a truncation vulnerability in the reward distribution mechanism when notifying reward amounts. In `InfraredDistributor.sol`, when `notifyRewardAmount()` is called, it divides the total amount by the number of validators:

```solidity
unchecked {
    amountsCumulative += amount / num;
}
token.safeTransferFrom(msg.sender, address(this), amount);
```

The division operation `amount / num` can result in truncation of decimal places. For example, if `amount = ......100` and `num = 3`, then `......100/3 = ......33` with a remainder of 1. This remainder gets stuck in the contract as it transfers the full `amount` but only accounts for `(amount/num) * num` in the `amountsCumulative` variable.

This issue is happening during `harvestOperatorRewards()`

```solidity
function harvestOperatorRewards() public {
    uint256 _amt = _rewardsStorage().harvestOperatorRewards(
        address(ibera), address(voter), address(distributor)
    );
    emit OperatorRewardsDistributed(
        address(ibera), address(distributor), _amt
    );
}
```

The likelyHood is Medium, since there can be small amounts in the end of the notified amount since those amounts gets applied fees in multiple places before they get sent to the distributor

Part of the fees are calculated initially in `collectBribes()` then `shareholderFees` then here at `harvestOperatorRewards()` making the probability of the presence of small truncatable number relatively high, also the function is premessionless

The truncated amounts accumulate over time and become permanently locked in the contract with no mechanism to distribute them.

### Recommendation

1. Track the remainder amounts separately:

```solidity
function notifyRewardAmount(uint256 amount) external {
    if (amount == 0) revert Errors.ZeroAmount();
    
    uint256 num = infrared.numInfraredValidators();
    if (num == 0) revert Errors.InvalidValidator();
    
    uint256 amountPerValidator = amount / num;
    uint256 remainder = amount % num;
    
    unchecked {
        amountsCumulative += amountPerValidator;
        unallocatedRemainder += remainder;
    }
    
    token.safeTransferFrom(msg.sender, address(this), amount);
    emit Notified(amount, num);
}
```

1. Add a mechanism to distribute accumulated remainders when they reach a sufficient amount to be evenly distributed.
    
3. Alternatively, implement a rounding mechanism that ensures the full amount is accounted for.

## [I-01] In `infrared::claimLostRewardsOnVault()` Lost Vault rewards are handled inappropriately
### Summary

`Infrared::claimLostRewardsOnVault()` retuned tokens will be used as bribes and not split between iBERA share holders

### Finding Description

in `Infrared::claimLostRewardsOnVault()`, we call `vault.getReward()` to get any rewards that were initially sent to the infrared staking vault with no stakers

The problem is that when we call `vault.getReward()`, we don't do any thing after, leaving those reward tokens inside `infrared` contract

Those tokens standing in `infrared` contract can be sent forcebly to the `BribeCollector` by calling `infrared::harvestBribes()` that will forward any whitelisted balance tokens to the collector contract as shown here

```solidity
File: RewardsLib.sol
208:     function harvestBribes(
209:         RewardsStorage storage $,
210:         address wbera,
211:         address collector,
212:         address[] memory _tokens,
213:         bool[] memory whitelisted
214:     ) external returns (address[] memory tokens, uint256[] memory _amounts) {
215:         uint256 len = _tokens.length;
216:         _amounts = new uint256[](len);
217:         tokens = new address[](len);
218: 
219:         for (uint256 i = 0; i < _tokens.length; i++) {
220:             if (!whitelisted[i]) continue;
221:             address _token = _tokens[i];
222:             if (_token == DataTypes.NATIVE_ASSET) {
223:                 IWBERA(wbera).deposit{value: address(this).balance}();
224:                 _token = wbera;
225:             }
226:             // amount to forward is balance of this address less existing protocol fees
227:             uint256 _amount = ERC20(_token).balanceOf(address(this))
228:                 - $.protocolFeeAmounts[_token];
229:             _amounts[i] = _amount;
230:             tokens[i] = _token;
231:             _handleTokenBribesForReceiver($, collector, _token, _amount);
232:         }
233:     }
```

Those tokens sent as bribes will be auctioned (build up), till some one buy it with `wBERA` token as the `payoutToken`, and call `infrared.collectBribes` during the buying process

```solidity
File: BribeCollector.sol
87:         infrared.collectBribes(payoutToken, payoutAmount);
88:         // payoutAmount will be transferred out at this point
89: 
```

then `collectBribes()` calling `collectBribesInWBERA()` that again split the `wBERA` amount to infrared vault and `iBERA` holders

### Impact Explanation

High:

- iBERA holders lose tokens that initially should have been to them
- Lost infrared vault rewards are used as Bribes

### Likelihood Explanation

High:

- every time a governance calls `claimLostRewardsOnVault()` it will be used as bribes by `harvestBribes()` split to protocol, voters, infrared vault and iBERA
  
## [I-02] Missing Fee Application on InfraredBERA Portion of Collected Bribes
### Summary

in `RewardsLib::harvestOperatorRewards()` fees are only applied to `amtIbgtVault` portion of bribes, excluding `amtInfraredBERA` portion from fee charges, causing loss of funds to the voter and protocol

### Finding Description

The call flow shows fees are only charged on the IBGT vault portion:

```solidity
// BribeCollector.sol
function claimFees(...) {
    // Transfers payout token and calls Infrared
    infrared.collectBribes(payoutToken, payoutAmount);
}

// Infrared.sol 
function collectBribes(address _token, uint256 _amount) {
    _rewardsStorage().collectBribesInWBERA(
        _amount,
        address(wbera),
        address(ibera), 
        address(ibgtVault),
        address(voter),
        rewardsDuration()
    );
}

// RewardsLib.sol
function collectBribesInWBERA(...) {
    // Split amount
    amtInfraredBERA = _amount * $.collectBribesWeight / WEIGHT_UNIT;
    amtIbgtVault = _amount - amtInfraredBERA;

    // Fees only applied to amtIbgtVault
    _handleTokenRewardsForVault(
        $,
        IInfraredVault(ibgtVault),
        wbera,
        voter, 
        amtIbgtVault,  // <-- Only this portion gets fees
        feeTotal,
        feeProtocol,
        rewardsDuration
    );
}
```

And during `RewardsLib::harvestOperatorRewards()`, Fees are applied to the portion of the `shareholderFees` of `amtInfraredBERA` that was initially sent, causing loss of fees that should have been applied to the whole `amtInfraredBERA`

This also happens during `infrared::harvestBase()` flow

### Impact Explanation

- Loss of fee revenue on `amtInfraredBERA` portion

### Likelihood Explanation

High likelihood as this occurs on every bribe collection transaction

### Recommendation

Apply fees to total bribe amount before splitting:

## [I-03] Payout Token Configurability vs Implementation Constraints

### Summary

Documentation claims payout token is configurable, but implementation strictly requires WBERA and lacks setter function.

### Finding Description

The documentation states:

```markdown
// README.md
- **Governance-Configurable Parameters**: Payout tokens, amounts, and other auction parameters are controlled by governance.
```

However, the implementation enforces WBERA:

```solidity
// Infrared.sol
function collectBribes(address _token, uint256 _amount) external onlyCollector {
    if (_token != address(wbera)) {
        revert Errors.RewardTokenNotSupported();
    }
    // ...
}
```

The `BribeCollector` contract:

```solidity
// BribeCollector.sol
contract BribeCollector {
    address public payoutToken;
    // No setter function for payoutToken
    // Only set during initialization
}
```

### Impact Explanation

- System flexibility is limited contrary to documentation claims
- Governance cannot adjust payout token despite documentation suggesting otherwise

### Recommendation

Either:

1. Update documentation to clearly state WBERA requirement
2. Add proper setter function and remove WBERA constraint:

```solidity
function setPayoutToken(address _newPayoutToken) external onlyGovernor {
    if (_newPayoutToken == address(0)) revert Errors.ZeroAddress();
    emit PayoutTokenSet(payoutToken, _newPayoutToken);
    payoutToken = _newPayoutToken;
}
```
