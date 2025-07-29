# Audit: `processRedemptions(uint256 maxUsers)`

**Contract:** `ParetoDollarQueue.sol`  
**Function Purpose:** Executes queued redemptions of USP token for users who submitted a request.

---

## High-Level Overview

This function processes redemption requests by looping through a queue of users and transferring their owed USP. While it performs its job, its current 
structure introduces major gas inefficiencies:

- Burn gas as the queue grows
- Leave stale users in the queue forever
- Cause long-term, protocol-paid gas bloat

---

## Status
	
Exploit Risk	🔵 Low <br>
Cost Risk	🔴 High <br>
Fix Simplicity	🟢 Simple <br>

---

## Source Code (Original)

```solidity
function processRedemptions(uint256 maxUsers) external whenNotPaused nonReentrant {
    // Load the current total length of the redemptionQueue array
    // Gas: 1x SLOAD of array length (~100 gas)
    uint256 len = redemptionQueue.length;

    // Counter to track how many users we've processed in this call
    uint256 processed = 0;

    // Loop from i = 0 up to len (entire array), capped by maxUsers
    // Major issue: always starts from index 0 every time function is called
    // This causes unnecessary re-processing of stale users who’ve already redeemed or never had a balance
    for (uint256 i = 0; i < len && processed < maxUsers; i++) {

        // Load the address of the user at position i in the queue
        // Gas: SLOAD of redemptionQueue[i] (~2,100 gas)
        address user = redemptionQueue[i];

        // Fetch the redemption amount owed to this user
        // Gas: SLOAD from mapping pendingRedemption[user] (~2,100 gas)
        uint256 amount = pendingRedemption[user];

        // If the amount is zero, skip to next iteration
        // Problem: this user remains in the queue and will be reprocessed again in future calls
        if (amount == 0) {
            continue;
        }

        // Burn the USP tokens held by the contract before transferring
        // Gas: External call — typically 15k to 25k+ gas depending on token logic
        uspToken.burn(address(this), amount);

        // Send the USP tokens to the user
        // Gas: External call — 10k to 20k+ depending on token implementation
        uspToken.transfer(user, amount);

        // Reset this user's pending redemption to zero
        // Gas: SSTORE — if resetting from non-zero to zero, costs 5,000 gas (may refund 15k in some scenarios)
        pendingRedemption[user] = 0;

        // Increment number of users processed in this call
        processed++;
    }
}
```
## Gas Cost Simulation
Running Foundry's forge test --gas-report for the function processRedemptions(uint256 maxUsers) shows **34200** gas spend for 1 call:

<img width="1357" height="395" alt="image" src="https://github.com/user-attachments/assets/7cd67ac7-dd23-4e57-be47-1a77f02f6bb1" /><br>


Now, let's say you have 100 addresses in the queue for this same function, but only 1 of them has a pending redemption amount > 0. The other 99 entries the contract must read, check, and ignore because there is nothing to process for them.

Running forge test --gas-report again shows **505737** gas spend for 100 queue size with 99 no action entries:

<img width="1382" height="386" alt="image" src="https://github.com/user-attachments/assets/bb63cc76-c9e5-44e9-9e96-0d73cfedb5b8" /><br>


---

## Risks

1-Gas is spent reading every entry, even the ones that don’t need work. The protocol will pay gas for every entry, not just for actual redemptions.

2-The more entries in the queue that require no action, the more gas is “wasted” just reading and checking them, without any real work being done.

3-This is the root of the protocol’s long-term gas cost risk: As the queue fills with entries that are already handled (but not removed), every processRedemptions call burns more and more gas for no reason.

4-In this example, 99 “no action” entries added 471,537 gas (505,737 – 34,200) to the call, or about 4,764 gas per entry.

5-The function’s gas usage scales linearly with queue size, so 1,000 queue entries: ~4.8M gas / 10,000 queue entries: ~47.7M gas (well above the block limit)

---

## Assessment

Every time processRedemptions is called, the function scans every address in the queue, regardless of whether any redemption is actually owed. For each entry with zero pending redemption, the contract still performs costly storage reads. As the queue fills with completed/irrelevant entries, the gas required per call increases linearly, threatening protocol funds and long-term usability.

---

## Optimized Version

```solidity
// Add this as a new state variable to persist queue progress between calls
uint256 public nextRedemptionIndex;

function processRedemptions(uint256 maxUsers) external whenNotPaused nonReentrant {
    // Track how many users we've processed this run
    uint256 processed = 0;

    // Load total length of the redemption queue (unchanged)
    uint256 len = redemptionQueue.length;

    // 🚀 Optimization: Start from where we left off last time
    for (uint256 i = nextRedemptionIndex; i < len && processed < maxUsers; i++) {
        // Get the queued user
        address user = redemptionQueue[i];

        // Check how much they are owed
        uint256 amount = pendingRedemption[user];

        // Skip if there's nothing to redeem
        if (amount == 0) {
            continue;
        }

        // Burn the contract's token balance
        uspToken.burn(address(this), amount);

        // Transfer USP to user
        uspToken.transfer(user, amount);

        // Reset user redemption record
        pendingRedemption[user] = 0;

        // Increment counter
        processed++;

        // 🔧 Track last index we processed
        nextRedemptionIndex = i + 1;
    }
}
```

## Mitigations

1- Reliability: The top **uint256 public nextRedemptionIndex** remembers the last queue index processed. Next call starts from where it left off, instead of always starting from 0.

2-Efficiency: In the original function every call starts at **i = 0**, scanning the entire queue including all users already processed, or with nothing pending. As the queue grows, more and more gas is wasted every call in a linear bloat. The Optimized function starts at **i = nextRedemptionIndex**, which means already-checked users are skipped entirely thus ending extra reading storage or gas wasting.

3-Predictability: Gas cost grows only with the number of new, unprocessed entries since the last run. If you process 100 new users, you pay for just those 100. The protocol treasury never pays again for the 1,000 users from last week.

---

## Gas Simulation Rerun
Rerunning Foundry's forge test --gas-report for the optimized function processRedemptions(uint256 maxUsers) shows **57,199** gas spend for 1 call:
<img width="1150" height="152" alt="image" src="https://github.com/user-attachments/assets/86643396-915f-4f1e-8734-954aa928dbfd" />
<br>


---

## Re-Assessment

Why does the optimized function use more gas than the original function? Because the optimized function uses a new state variable **nextRedemptionIndex** to remember its position in the queue. The original function doesn’t update any new storage variable, so it’s slightly cheaper at first.

But with large queues (10, 100, 1,000 users), the original function’s gas cost skyrockets because it scans and re-reads all entries every time.

The optimized function always pays roughly 57,199 gas, no matter how many extra entries are in the queue, so gas stays flat.


| Version   | Queue Size | Users Needing Redemption | Gas Used                         |
| --------- | ---------- | ------------------------ | -------------------------------- |
| Original  | 1          | 1                        | 34,200                           |
| Original  | 100        | 1                        | 505,737                          |
| Original  | 1,000      | 1                        | \~4,800,000 (predicted)          |
| Original  | 10,000     | 1                        | \~47,700,000 (predicted)         |
| Optimized | 1          | 1                        | 57,199                           |
| Optimized | 100        | 1                        | 57,199                           |
| Optimized | 1,000      | 1                        | 57,199                           |
| Optimized | 10,000     | 1                        | 57,199                           |


The small extra cost per call (due to the **nextRedemptionIndex** storage write) is a one-time price for unlocking huge gas savings and scalability as the protocol grows. With the solution presented above, the protocol could potentially:

Prevent reprocessing of thousands of inactive users;<br>
Cut keeper costs by thousands per call;<br>
Improve efficiency as protocol scales.<br>


