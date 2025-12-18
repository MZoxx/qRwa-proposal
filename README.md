## Proposal

Allow qRWA Smart Contract to be deployed on Qubic.

## Available Options

> Option 0: no, don’t allow

> Option 1: yes, allow

**Technical Description: qRWA Smart Contract**

**Version:** 1.0

**Platform:** Qubic

**Language:** C++

### 1. Executive Summary
The qRWA (Qubic Real World Asset) smart contract is a decentralized governance and revenue distribution protocol designed for the Qubic ecosystem. It serves as a DAO (Decentralized Autonomous Organization) for the **QMINE** asset, allowing token holders to govern protocol parameters and manage a diversified treasury of assets. The contract automates the ingestion of revenue (QUs), applies a configurable fee structure for operational costs (e.g., mining electricity/maintenance), and distributes net yields to QMINE holders (90%) and the contract's own shareholders (10%), while routing any dividends forfeited by moved tokens to the protocol developer.

### 2. Core Architecture

#### 2.1 Asset Management
The contract manages two distinct categories of assets:
* **Native Governance Asset (QMINE):** Identified by `Asset Name: QMINE or 297666170193` and a specific issuer ID. This token dictates voting power and dividend claims.
* **General Treasury Assets:** A generic `HashMap` implementation stores balances for arbitrary Qubic assets (shares of other SCs, IPO tokens, etc.) that have been deposited into the contract.
* **Counting Logic: ALL QMINE that are managed by any SC: they move their asset around, but try to keep the balance unchanged.

#### 2.2 Revenue Streams & "Splitter" Logic
The contract distinguishes revenue based on its source to apply different fee models:
* **Pool A (Miner Revenue):** Ingests QU transfers from mining. This pool is subject to **Mining Fees** (Electricity, Maintenance, Reinvestment) before distribution.
* **Pool B (General Asset Revenue):** Ingests QU transfers from other Asset dividends. This pool bypasses operational fees and is allocated 100% to the dividend pools.
* **QMINE Remainder (Dev Dividends):** A catch-all mechanism for the QMINE dividend pool. Because dividend eligibility is based on the minimum balance held throughout the epoch (`min(Begin, End)`), tokens that are moved or sold during an epoch result in "unclaimed" portions of the dividend pool. These remainder funds are automatically calculated and transferred to the `qmineDevAddress`.

#### 2.3 Governance Module
The governance model utilizes a "Snapshot" mechanism, calculating voting power as `min(Balance_Begin_Epoch, Balance_End_Epoch)` to prevent flash-voting attacks.

* **Parameter Governance:**
    * **Mechanism:** A rotating buffer of 64 slots (`mGovPolls`).
    * **Function:** Holders vote on `qRWAGovParams` (Admin ID, Fee Receiver IDs, Fee Percentages).
    * **Threshold:** Strict super-majority (2/3 + 1) required to modify state.
    * **Execution:** Automated at `END_EPOCH`.

* **Treasury Governance (Asset Release):**
    * **Mechanism:** Admin-initiated proposals to release assets to specific destinations.
    * **Storage Optimization:** Voter state is compressed into `bit_64` bitfields, allowing users to track votes on 64 simultaneous polls with minimal memory footprint.
    * **Execution:** If a poll passes (YES votes ≥ 2/3 Quorum), the contract invokes `qpi.transferShareOwnershipAndPossession` to execute the transaction trustlessly.

### 3. State Machine Lifecycle

#### 3.1 Initialization (`INITIALIZE`)
Sets default governance parameters (Fees: 35% Electricity, 5% Maintenance, 10% Reinvestment), designates the initial Admin, and zeros out all revenue pools and maps.

#### 3.2 Epoch Transition
* **`BEGIN_EPOCH`:**
    * Resets poll counters.
    * Iterates through QMINE asset holders to create a `mBeginEpochBalances` snapshot.
* **`END_EPOCH`:**
    * Creates `mEndEpochBalances` snapshot.
    * **Tallying:** Re-calculates all active poll scores by iterating through voters and applying the `min(Begin, End)` voting power logic.
    * **Execution:** Applies successful Governance Parameter changes and executes successful Asset Release transfers.
    * **Payout Snapshot:** Copies the finalized epoch balances into `mPayout...` maps for the dividend distributor.

#### 3.3 Tick Processing (`END_TICK`)
* **Dividend Distribution:** Checks if the current time matches the payout schedule (Friday, 12:00 UTC) and if `QRWA_MIN_PAYOUT_INTERVAL_MS` has elapsed.
* **Fee Extraction:** If valid, calculates operational fees from Pool A and transfers them to the defined `electricityAddress`, `maintenanceAddress`, etc.
* **Yield Allocation:**
    * 90% of Net Revenue $\rightarrow$ `mQmineDividendPool`
        * The contract iterates through all holders in the snapshot.
        * **Payout Calculation:** `(min(Begin, End) * mQmineDividendPool) / Total_Supply_Begin`, where `mQmineDividendPool` is the accumulation of 90% of the net revenue collected by the contract since the last payout. (`mQmineDividendPool = 90% of (Pool A (after mining fees) + Pool B)`).
        * **Remainder Sweep:** Since the sum of `min(Begin, End)` balances is always less than or equal to `Total_Supply_Begin` (due to token transfers), a remainder is left in the pool. This remainder—representing dividends for "moved" shares—is transferred entirely to the **QMINE Developer Address** (`qmineDevAddress`).
    * 10% of Net Revenue $\rightarrow$ `mQRWADividendPool` (Broadcast via `qpi.distributeDividends` to SC shareholders).

### 4. Interface Specification

#### 4.1 Public Procedures (Write)
| Procedure ID | Name | Description |
| :--- | :--- | :--- |
| 3 | `DonateToTreasury` | Transfers QMINE from user to SC. User must first grant management rights. |
| 4 | `VoteGovParams` | Casts a vote for protocol parameters (Fees, Admin). |
| 5 | `CreateAssetReleasePoll` | **Admin Only.** Creates a proposal to transfer assets out of the treasury. |
| 6 | `VoteAssetRelease` | Casts a YES/NO vote on an active asset release poll. |
| 7 | `DepositGeneralAsset` | **Admin Only.** Deposits arbitrary assets into the SC treasury. |
| 8 | `revokeAssetManagementRights`| Allows a user to reclaim asset management rights from the SC back to QX. |

#### 4.2 Public Functions (Read)

| Function ID | Name | Description |
| :--- | :--- | :--- |
| 1 | `GetGovParams` | Returns the currently active `qRWAGovParams`, including admin ID, fee addresses (electricity, maintenance, reinvestment), and fee percentages. |
| 2 | `GetGovPoll` | Returns the detailed state (`qRWAGovProposal`) of a specific governance proposal by ID, including its status and current vote score. |
| 3 | `GetAssetReleasePoll` | Returns the detailed state (`AssetReleaseProposal`) of a specific asset release proposal by ID, including the asset, amount, destination, and vote counts. |
| 4 | `GetTreasuryBalance` | Returns the current amount of **QMINE** tokens held in the contract's treasury (`mTreasuryBalance`). |
| 5 | `GetDividendBalances` | Returns the current balances of all internal accounting pools: Revenue Pool A, Revenue Pool B, QMINE Dividend Pool, and qRWA Dividend Pool. |
| 6 | `GetTotalDistributed` | Returns the historical total amount of dividends distributed to QMINE holders and qRWA shareholders since inception. |
| 7 | `GetActiveAssetReleasePollIds` | Returns a list of Proposal IDs for all currently active Asset Release polls. |
| 8 | `GetActiveGovPollIds` | Returns a list of Proposal IDs for all currently active Governance polls. |
| 9 | `GetGeneralAssetBalance` | Returns the balance of a specific **General Asset** (non-QMINE) held by the contract treasury. Input requires the `Asset` struct. |
| 10 | `GetGeneralAssets` | Iterates through the contract's storage and returns a list of all General Assets and their corresponding balances held in the treasury. |

### 5. System Procedures (Callbacks)
* **`POST_INCOMING_TRANSFER`:** intercepts incoming QUs. It analyzes `input.sourceId` to route funds to either Pool A (Funds from Mining) or Pool B (Other General Asset dividends). If the `input.sourceId` is a Smart Contract, it will route to Pool A. If the `input.sourceId` is a user, it will route to Pool B.
* **`PRE_ACQUIRE_SHARES`:** Returns `true`, allowing any entity to transfer asset management rights to this contract without restriction.

### 6. Data Structures
* **HashMaps:** utilized for O(1) lookups of Voter Balances, Vote Records, and Asset Holdings. Capacity is fixed at compile time (e.g., `QRWA_MAX_QMINE_HOLDERS = 2^21`).
* **Arrays:** Used for fixed-size governance buffers (`mGovPolls`, `mAssetPolls`).

### 7. Security & Limitations
* **Flash Loan Protection:** The dual-snapshot voting power calculation mitigates instantaneous voting attacks.
* **Capacities:**
    * Max QMINE Holders: ~2 million (`2^21`).
    * Max Polls Holding: 64 (Governance) + 64 (Asset).
    * Max Polls per Epoch: 8 (Governance) + 8 (Asset).
    * Max Managed Assets: 1024.
* **Concurrency:** The contract logic is single-threaded per tick (as per Qubic architecture), relying on `_locals` structs to manage stack memory safely during execution.

