
| Id                                                                                          | Title                                                                         |
| ------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| [H-01](#h-01-incorrect-calculation-in-vesting-transfer-leads-to-excess-fund-claim)          | Incorrect Calculation in Vesting Transfer Leads to Excess Fund Claim          |
| [H-02](#h-02-user-receives-fewer-vesting-tokens-than-expected-due-to-incorrect-calculation) | User Receives Fewer Vesting Tokens Than Expected Due to Incorrect Calculation |

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
