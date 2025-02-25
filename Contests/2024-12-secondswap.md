
| ID                                                                                          | Title                                                                         |
| ------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| [H-01](#h-01-incorrect-calculation-in-vesting-transfer-leads-to-excess-fund-claim)          | Incorrect Calculation in Vesting Transfer Leads to Excess Fund Claim          |
| [H-02](#h-02-user-receives-fewer-vesting-tokens-than-expected-due-to-incorrect-calculation) | User Receives Fewer Vesting Tokens Than Expected Due to Incorrect Calculation |
| [M-01](m-01-underflow-in-claimable-dosing-claim-function)                                   | Underflow in `claimable` DOSing `claim` Function                              |


## [H-01] Incorrect Calculation in Vesting Transfer Leads to Excess Fund Claim

  

### Finding Description and Impact

A miscalculation in `SecondSwap_StepVesting::transferVesting` allows users to Claim more tokens than they are entitled to:

```solidity

File: SecondSwap_StepVesting.sol

216:     function transferVesting(address _grantor, address _beneficiary, uint256 _amount) external {

//   CODE

224:         require(

225:             grantorVesting.totalAmount - grantorVesting.amountClaimed >= _amount,

226:             "SS_StepVesting: insufficient balance"

227:         ); // 3.8. Claimed amount not checked in transferVesting function

228:

229:@>       grantorVesting.totalAmount -= _amount;

230:@>       grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;

```

#### Preconditions

This issue occurs under the following conditions:
  
- `grantorVesting.totalAmount - grantorVesting.amountClaimed == _amount`

    - (amount could be any number in range here but for the higher impact assume this).

- After the transaction, `grantorVesting.totalAmount` becomes equal to `grantorVesting.amountClaimed`.

- The redistributed value is incorrectly calculated, allowing the user to claim funds they have already received.

  
#### Impact

This flaw results in a **loss of funds to either the protocol or user** depends on who is selling

- a user has an already vesting and add it to marketplace

- a user is buying some vesting tokens so it transfer from vesting manager and he takes the extra funds from other vesting owners.

, as users can claim more tokens than their rightful allocation.
### Proof of Concept

#### Example Scenario

- Assume the following initial conditions:

    - `totalAmount` = 1000e18

    - `_amount` = 600e18

    - `grantorVesting.amountClaimed` = 400e18

    - `numOfSteps` = 100

    - `grantorVesting.stepsClaimed` = 50

- After the transfer:

    - `totalAmount` = 1000e18 - 600e18 = 400e18

    - `releaseRate` = 400e18 / 100 = 4e18

- Calculating the excess claimed amount at `numOfSteps - 1`:

    - `(99 - 50) * 4e18 = 196e18`

  
The user can claim an **extra 196e18 tokens near 20%**. This issue is amplified with larger numbers, leading to substantial losses.


```solidity

File: SecondSwap_StepVesting.sol

161:     function claimable(address _beneficiary) public view returns (uint256, uint256) {

//    CODE

172:         uint256 elapsedTime = currentTime - startTime;

173:         uint256 currentStep = elapsedTime / stepDuration;

174:@>       uint256 claimableSteps = currentStep - vesting.stepsClaimed;

//   CODE

184:@>       claimableAmount = vesting.releaseRate * claimableSteps;

```
#### Result

- The user can claim an **additional 196e18 tokens near 20%**.  

### Recommended Mitigation Steps

To fix this issue, adjust the release rate calculation as follows:


```diff

--     grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;

++     grantorVesting.releaseRate = (grantorVesting.totalAmount - grantorVesting.amountClaimed) / numOfSteps;

```

This ensures the release rate accounts for the amount already claimed, preventing any excess fund withdrawals.

  

## [H-02] User Receives Fewer Vesting Tokens Than Expected Due to Incorrect Calculation
 
### Finding Description and Impact

  

An error in `SecondSwap_StepVesting::transferVesting` causes users to receive fewer tokens than they are entitled to, freezing a portion of their funds until the vesting period ends.

  

```solidity

File: SecondSwap_StepVesting.sol

216:     function transferVesting(address _grantor, address _beneficiary, uint256 _amount) external {

//   CODE

224:         require(

225:             grantorVesting.totalAmount - grantorVesting.amountClaimed >= _amount,

226:             "SS_StepVesting: insufficient balance"

227:         ); // 3.8. Claimed amount not checked in transferVesting function

228:

229:         grantorVesting.totalAmount -= _amount;

230:@>       grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;

```

  

#### Assumptions

  

Assume the numerator issue has been mitigated as stated in a previous report:

  

```diff

--     grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;

++     grantorVesting.releaseRate = (grantorVesting.totalAmount - grantorVesting.amountClaimed) / numOfSteps;

```

  

#### Problem Scenario

  

- The calculation `(grantorVesting.totalAmount - grantorVesting.amountClaimed) / numOfSteps` distributes the new `grantorVesting.totalAmount` across all steps, freezing a portion of the tokens until `endTime`.

  

#### Impact

  

This results in a **freeze of funds for the user**, limiting their ability to access the full amount of tokens they are entitled to.

  

### Proof of Concept

  

### Example Scenario

  

- Initial conditions:

    - `totalAmount` = 1000e18

    - `_amount` = 400e18

    - `grantorVesting.amountClaimed` = 200e18

    - `numOfSteps` = 100

    - `grantorVesting.stepsClaimed` = 50

- After the transfer:

    - `totalAmount` = 1000e18 - 400e18 = 600e18

    - `grantorVesting.releaseRate` = (600e18 - 200e18) / 100 = 4e18

- Calculating the total claimed amount at `numOfSteps - 1`:

    - `(99 - 50) * 4e18 = 196e18`

  

The user can **claim only 196e18 tokens**, whereas they should be entitled to:

  

- Updated `grantorVesting.releaseRate` = (600e18 - 200e18) / (100 - 50) = 8e18

- Total claimable amount at `numOfSteps - 1` = (99 - 50) * 8e18 = 392e18

  

This results in **196e18 tokens being frozen which is nearly 20%**.

  

```solidity

File: SecondSwap_StepVesting.sol

161:     function claimable(address _beneficiary) public view returns (uint256, uint256) {

//    CODE

172:         uint256 elapsedTime = currentTime - startTime;

173:         uint256 currentStep = elapsedTime / stepDuration;

174:@>       uint256 claimableSteps = currentStep - vesting.stepsClaimed;

//   CODE

184:@>       claimableAmount = vesting.releaseRate * claimableSteps;

```

  
  
  

### Recommended Mitigation Steps

  

Update the release rate calculation to account for unclaimed steps:

  

```diff

--     grantorVesting.releaseRate = (grantorVesting.totalAmount - grantorVesting.amountClaimed) / numOfSteps;

++     grantorVesting.releaseRate = (grantorVesting.totalAmount - grantorVesting.amountClaimed) / (numOfSteps - grantorVesting.stepsClaimed);

```

  

This adjustment ensures that only the remaining unclaimed steps are considered in the distribution, preventing token freezing.




## [M-01] Underflow in `claimable` DOSing `claim` Function

## Finding Description and Impact

If a user have already collected all their tokens when the vesting period has ended, resulting in:

- `stepsClaimed > numOfSteps`
- The user being unable to claim newly purchased amounts due to an incorrect check in `claimable`.
```solidity
174:@>       uint256 claimableSteps = currentStep - vesting.stepsClaimed;
```

### Impact

- The `claim` function becomes unavailable.
- Users who purchase additional amounts are unable to claim their tokens.

## Proof of Concept

### Scenario

1. Bob identifies a desirable vesting token and buys 10,000 tokens.
2. The vesting period ends (`currentTime > endTime`), and Bob has claimed all funds.
3. Due to rounding issues during creation, `stepsClaimed > numOfSteps`.
4. Bob purchases additional tokens.
5. Bob is unable to claim the newly purchased tokens due to an incorrect check in `claimable`.

### Numerical Example

#### Assumptions:

- `_endTime - _startTime = 1000`
- `numOfSteps = 110`
- `stepDuration = (_endTime - _startTime) / numOfSteps = 9`

#### Calculation:

1. The vesting period ends, and Bob claims all funds using the `claim` function, which calls `claimable`:

```solidity
File: SecondSwap_StepVesting.sol
172:         uint256 elapsedTime = currentTime - startTime;
173:         uint256 currentStep = elapsedTime / stepDuration;
174:@>       uint256 claimableSteps = currentStep - vesting.stepsClaimed;
175: 
176:         uint256 claimableAmount;
177: 
178:@>       if (vesting.stepsClaimed + claimableSteps >= numOfSteps) {
179:             //[BUG FIX] user can buy more than they are allocated
180:             claimableAmount = vesting.totalAmount - vesting.amountClaimed;
181:@>           return (claimableAmount, claimableSteps);
182:         }
```

2. Using the numbers:
    
    - `currentStep = 1000 / 9 = 111`
    - `claimableSteps = 111 - 100 = 11`
3. In `claim`, the value of `claimableSteps` is added to `stepsClaimed`:
    

```solidity
File: SecondSwap_StepVesting.sol
193:     function claim() external {
//    CODE
198:@>       vesting.stepsClaimed += claimableSteps;
199:         vesting.amountClaimed += claimableAmount;
```

This results in:

- `vesting.stepsClaimed = 111`
- `numOfSteps = 110`

#### After Purchasing More Tokens:

Assuming the issue in the `create` function is mitigated as stated in another report:

```diff
--   if (numOfSteps - _vestings[_beneficiary].stepsClaimed != 0)
++   if (numOfSteps > _vestings[_beneficiary].stepsClaimed){
```

Bob successfully buys additional tokens. However, when he tries to claim them:

```solidity
File: SecondSwap_StepVesting.sol
161:     function claimable(address _beneficiary) public view returns (uint256, uint256) {
//    CODE
172:         uint256 elapsedTime = currentTime - startTime;
173:         uint256 currentStep = elapsedTime / stepDuration;
174:@>       uint256 claimableSteps = currentStep - vesting.stepsClaimed;
```

The function always reverts due to the incorrect check.

## Recommended Mitigation Steps

Apply the following correction to the `claimable` function:

```diff
--       uint256 claimableSteps = currentStep - vesting.stepsClaimed;
++       uint256 claimableSteps = currentStep > vesting.stepsClaimed ? 0 : currentStep - vesting.stepsClaimed;
```





The report is wrongly misunderstood, can you reconsider giving it another look

consider a scenario where the user who listed on the marketplace has claimed all his vesting before calling `unlistVesting` so that the vesting manager transfers the vesting back to him, when its assumed transfer is successful funds will be stuck due to the underflow in `claimable`








This is misunderstood as there is an edge-case were stepsClaimed exceeds numOfSteps
with currentTime = endTime.

Explaining again with another example:

- user have vesting token with vesting has:
		- vestingDuration = 15768000 // 6 months
		- numOfSteps = 1313999
		- stepDuration = 15768000 / 1313999 = 12 // calculated at deployment
- time of vesting has ended `endTime < block.timestamp` and currentTime = endTime.
- a user claim all tokens
	- **currentStep** = (endTime - startTime) / duration = 15768000 / 12 = **1314000**
	- with no previous steps claimed claimableSteps = 1314000
```solidity
		uint256 claimableSteps = currentStep - vesting.stepsClaimed;
		//CODE
        if (vesting.stepsClaimed + claimableSteps >= numOfSteps) {
            //[BUG FIX] user can buy more than they are allocated
            claimableAmount = vesting.totalAmount - vesting.amountClaimed;
@>          return (claimableAmount, claimableSteps);
        }
```
-  after claiming `stepsClaimed` is above the `numOfSteps` 
- vesting.stepsClaimed = 1314000
- numOfSteps                = 1313999

- The user want to buy the same vesting token as it has a good discounted price
- as numOfSteps < stepsClaimed any following cliaming attempt will revert leaving tokens stuck.


there were an issue in `_createVesting` assumed its fixed

`_createVesting` check if there is already vesting for the user to increase totalAmount then if the all steps is claimed to set releaseRate = 0
```solidity
        } else {
            _vestings[_beneficiary].totalAmount += _totalAmount;
@>          if (numOfSteps - _vestings[_beneficiary].stepsClaimed != 0)
```
having the right implementation and successful transfer with this implementation
```diff
++   if (numOfSteps > _vestings[_beneficiary].stepsClaimed){
```
