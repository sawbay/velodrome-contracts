## Overview
The requested contracts cover the Velodrome V2 voting and rewards flow: `Voter.sol` coordinates gauges and veNFT voting, `Gauge.sol` handles staking emissions, `Reward.sol` plus its derivatives implement veNFT-based reward accounting, and `FeesVotingReward.sol` wires fee rewards to the voting system. A `BribeVotingReward.sol` source file is not present in this repository.

### `contracts/Voter.sol`
Manages pool voting, gauge lifecycle, and reward distribution on behalf of veNFT holders.

| Function (visibility) | Purpose |
| --- | --- |
| `epochStart/epochNext/epochVoteStart/epochVoteEnd` (external, pure) | Timestamp utilities that expose the epoch cadence used throughout the system. |
| `initialize(address[] calldata, address)` (external) | Seeds the whitelist and hands minter rights to a new address. |
| `setGovernor/setEpochGovernor/setEmergencyCouncil` (public) | Governance role transfers for governor, epoch governor, and emergency council. |
| `setMaxVotingNum(uint256)` (external) | Updates the maximum number of pools a veNFT can vote for in one epoch. |
| `reset(uint256)` (external) | Clears an NFT’s current votes after the vote lock expires. |
| `poke(uint256)` (external) | Recomputes vote weights for an NFT based on its current ve balance. |
| `vote(uint256, address[] calldata, uint256[] calldata)` (external) | Casts weighted votes for a set of pools in the current epoch. |
| `depositManaged(uint256, uint256)` / `withdrawManaged(uint256)` (external) | Opt a veNFT into, or out of, a managed veNFT and sync vote power accordingly. |
| `whitelistToken(address,bool)` / `whitelistNFT(uint256,bool)` (external) | Toggle whitelist status for reward tokens and managed veNFTs. |
| `createGauge(address,address)` (external) | Deploys a new gauge plus its paired voting rewards for a pool. |
| `killGauge(address)` / `reviveGauge(address)` (external) | Emergency controls to disable or re-enable a gauge. |
| `length()` (external view) | Returns the number of tracked pools. |
| `notifyRewardAmount(uint256)` (external) | Accepts new emission tokens from the minter and updates the global index. |
| `updateFor(address[] memory)` / `updateFor(uint256,uint256)` / `updateFor(address)` (external) | Pulls the latest reward index into one or many gauges. |
| `claimRewards(address[] memory)` (external) | Triggers gauge reward claims for the caller across multiple gauges. |
| `claimIncentives(address[] memory,address[][] memory,uint256)` / `claimFees(address[] memory,address[][] memory,uint256)` (external) | Claims incentive or fee rewards for a veNFT across reward contracts. |
| `distribute(uint256,uint256)` / `distribute(address[] memory)` (external) | Pushes accumulated emissions into gauges after syncing the minter epoch. |

### `contracts/gauges/Gauge.sol`
Implements staking, emission distribution, and optional fee forwarding for a specific pool’s gauge.

| Function (visibility) | Purpose |
| --- | --- |
| `rewardPerToken()` / `lastTimeRewardApplicable()` (public view) | Read helpers for emission accounting over the 7-day distribution window. |
| `getReward(address)` (external) | Lets a staker or the voter contract pull accrued emissions. |
| `earned(address)` (public view) | Calculates a staker’s claimable emissions using stored checkpoints. |
| `deposit(uint256)` / `deposit(uint256,address)` (external) | Stake underlying pool tokens for yourself or another recipient. |
| `withdraw(uint256)` (external) | Unstake previously deposited tokens and update reward data. |
| `left()` (external view) | Returns how many reward tokens remain scheduled for distribution. |
| `notifyRewardAmount(uint256)` (external) | Voter-triggered top-up that can also forward pool fees before recalculating rate. |
| `notifyRewardWithoutClaim(uint256)` (external) | Gauge-factory-admin top-up that skips fee claiming. |

### `contracts/rewards/Reward.sol`
Abstract base contract that tracks veNFT-weighted rewards over epochs, storing per-token checkpoints and enabling authorized deposit/withdraw hooks.

| Function (visibility) | Purpose |
| --- | --- |
| `rewardsListLength()` (external view) | Returns the number of reward token addresses being tracked. |
| `lockExpiry(uint256)` / `biasCorrection(uint256)` / `lockExpiryAndBiasCorrection(uint256)` (external view) | Exposes stored metadata for a veNFT’s reward lock expiry and bias adjustment. |
| `globalRewardPointHistory(uint256)` / `userRewardPointHistory(uint256,uint256)` (external view) | Fetch stored global or per-token checkpoint data. |
| `getPriorSupplyIndex(uint256)` / `supplyAt(uint256)` (public view) | Locate and evaluate the historical aggregate slope/bias at a timestamp. |
| `getPriorBalanceIndex(uint256,uint256)` / `balanceOfNFTAt(uint256,uint256)` (public view) | Resolve the per-token checkpoint index and voting power at a timestamp. |
| `earned(address,uint256)` (public view) | Computes the pending reward for a veNFT across epochs since the last claim. |
| `_deposit(uint256,uint256)` (external) | Hook for gauges/voter to record new vote weight allocated to this reward. |
| `_withdraw(uint256,uint256)` (external) | Hook to unwind stored vote weight when allocations are removed. |
| `getReward(uint256,address[] memory)` (external, virtual) | Interface stub for concrete contracts to pay rewards. |
| `notifyRewardAmount(address,uint256)` (external, virtual) | Interface stub for concrete contracts to accept reward tokens. |

### `contracts/rewards/VotingReward.sol`
Abstract extension of `Reward` that restricts reward claims to veNFT owners (or the voter contract) and delegates token transfers to `_getReward`.

| Function (visibility) | Purpose |
| --- | --- |
| `getReward(uint256,address[] memory)` (external) | Validates ownership/approval and pays rewards to the veNFT owner. |
| `notifyRewardAmount(address,uint256)` (external, virtual) | Placeholder for subclasses to define how reward top-ups are authorized. |

### `contracts/rewards/FeesVotingReward.sol`
Concrete voting reward that only accepts emissions from its paired gauge’s fee stream and enforces a fixed reward token set.

| Function (visibility) | Purpose |
| --- | --- |
| `notifyRewardAmount(address,uint256)` (external) | Allows the paired gauge to deposit approved fee tokens for distribution. |

### `BribeVotingReward.sol`
A `BribeVotingReward.sol` contract file does not exist in this repository (a search for `*Bribe*.sol` returns no results).

### `contracts/Pool.sol`
Implements the Velodrome V2 AMM pair, tracking reserves, TWAP observations, LP token issuance, and swap/fee mechanics for stable and volatile pools.

| Function (visibility) | Purpose |
| --- | --- |
| `initialize(address,address,bool)` (external) | One-time initializer from the factory that seeds token addresses, pool type, fees contract, metadata, and initial observation. |
| `setName(string)` / `setSymbol(string)` (external) | Factory-admin controls to update ERC-20 metadata for the LP token. |
| `observationLength()` (external view) | Returns how many TWAP observations the pool has recorded. |
| `lastObservation()` (public view) | Fetches the most recent observation checkpoint. |
| `metadata()` (external view) | Exposes decimals, reserves, stability flag, and token addresses for frontends. |
| `tokens()` (external view) | Returns the ordered token pair tracked by the pool. |
| `claimFees()` (external) | Accrues outstanding fee indices for the caller and transfers their share from `PoolFees`. |
| `getReserves()` (public view) | Returns the current reserves and last block timestamp. |
| `currentCumulativePrices()` (public view) | Reads the current cumulative price values, mocking them forward if needed. |
| `quote(address,uint256,uint256)` (external view) | Samples historical prices to return an average quote for a token input. |
| `prices(address,uint256,uint256)` (external view) | Returns a sampled array of historical price quotes. |
| `sample(address,uint256,uint256,uint256)` (public view) | Internal sampler powering TWAP queries with configurable windowing. |
| `mint(address)` (external) | Mints LP tokens for deposited liquidity, enforcing invariant constraints on stable pairs. |
| `burn(address)` (external) | Burns LP tokens held by the pool and returns the proportional underlying tokens. |
| `swap(uint256,uint256,address,bytes)` (external) | Executes a swap with optional flash-hook, applying fees and invariant checks. |
| `skim(address)` (external) | Sends excess token balances (beyond reserves) to the requested address. |
| `sync()` (external) | Forces reserve data to match the pool’s actual token balances. |
| `getAmountOut(uint256,address)` (external view) | Quotes the output amount for a given input and token direction after fees. |
| `name()` / `symbol()` (public view) | Override ERC-20 metadata getters to return the mutable LP token name/symbol. |

### `contracts/PoolFees.sol`
Auxiliary contract deployed per pool to custody accrued swap fees and return them to LPs during claims.

| Function (visibility) | Purpose |
| --- | --- |
| `claimFeesFor(address,uint256,uint256)` (external) | Allows the parent pool to send accumulated token0/token1 fees to a recipient. |

### `contracts/Router.sol`
Routes swaps, liquidity provisioning, and zap operations across approved Velodrome-compatible pool factories, with ETH/WETH handling and metatransaction support.

| Function (visibility) | Purpose |
| --- | --- |
| `sortTokens(address,address)` (public pure) | Deterministically orders a token pair and guards against duplicates/zero addresses. |
| `poolFor(address,address,bool,address)` (public view) | Computes or predicts the deterministic pool address for a token pair and stability flag. |
| `getReserves(address,address,bool,address)` (public view) | Returns reserves for a pair in the requested factory, ordered relative to the caller’s inputs. |
| `getAmountsOut(uint256,Route[] memory)` (public view) | Simulates a swap path to estimate outputs across routed pools. |
| `quoteAddLiquidity(address,address,bool,address,uint256,uint256)` (public view) | Estimates optimal token contributions and LP minted for adding liquidity. |
| `quoteRemoveLiquidity(address,address,bool,address,uint256)` (public view) | Quotes token returns for redeeming LP shares. |
| `addLiquidity(...)` / `addLiquidityETH(...)` (public/external) | Transfers user tokens (and optional ETH) into the pool and mints LP tokens to a recipient. |
| `removeLiquidity(...)` / `removeLiquidityETH(...)` / `removeLiquidityETHSupportingFeeOnTransferTokens(...)` (public) | Burns LP tokens and returns underlying assets, with ETH unwrap and fee-on-transfer support. |
| `swapExactTokensForTokens(...)` / `swapExactETHForTokens(...)` / `swapExactTokensForETH(...)` / `UNSAFE_swapExactTokensForTokens(...)` (external) | Executes swaps across a defined route with varying slippage enforcement and ETH entry/exit options. |
| `swapExactTokensForTokensSupportingFeeOnTransferTokens(...)` / `swapExactETHForTokensSupportingFeeOnTransferTokens(...)` / `swapExactTokensForETHSupportingFeeOnTransferTokens(...)` (external) | Swap helpers tolerant of fee-on-transfer tokens along the route. |
| `zapIn(...)` (external payable) | Splits and routes inputs (tokens or ETH) to provide liquidity in one call, optionally staking in the gauge. |
| `generateZapInParams(...)` / `generateZapOutParams(...)` (external view) | Pre-computes minimum amounts for zap operations based on routing paths. |
| `quoteStableLiquidityRatio(address,address,address)` (external view) | Computes the optimal ratio for depositing into a stable pool given token decimals. |
