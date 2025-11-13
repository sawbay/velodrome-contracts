# Protocol Overview

## AMM contracts

### `contracts/contracts/Pool.sol`
It implements Velodrome's constant-product and stable-swap AMM pair with LP token accounting and fee tracking.
- `initialize(address _token0, address _token1, bool _stable)`: One-time pool initializer that wires token addresses, deploys the fee vault, and seeds the metadata cache.
- `setName(string calldata __name)`: Updates the human-readable LP token name via the factory's pool admin.
- `setSymbol(string calldata __symbol)`: Updates the LP token symbol via the factory's pool admin.
- `observationLength()`: Returns the number of TWAP observation checkpoints recorded.
- `lastObservation()`: Fetches the latest TWAP observation struct.
- `metadata()`: Returns cached decimals, reserves, stable flag, and token addresses for frontends.
- `tokens()`: Returns the ordered token pair (token0, token1).
- `claimFees()`: Pulls accumulated fee shares from the bonded `PoolFees` contract for the caller.
- `getReserves()`: Reads the current reserves and last update timestamp.
- `currentCumulativePrices()`: Returns the cumulative price accumulator values and current block timestamp.
- `quote(address tokenIn, uint256 amountIn, uint256 granularity)`: Samples the TWAP oracle to quote an output amount.
- `prices(address tokenIn, uint256 amountIn, uint256 points)`: Returns a series of sampled TWAP prices.
- `sample(address tokenIn, uint256 amountIn, uint256 points, uint256 window)`: Public sampler backing the oracle helper methods.
- `mint(address to)`: Mints LP tokens for supplied reserves and syncs pool accounting.
- `burn(address to)`: Burns LP tokens held by the pool and returns proportional reserves to the recipient.
- `swap(uint256 amount0Out, uint256 amount1Out, address to, bytes calldata data)`: Executes a swap, supporting flash callback hooks.
- `skim(address to)`: Transfers any excess token balances (above reserves) to the target address.
- `sync()`: Forces reserves to match current token balances.
- `getAmountOut(uint256 amountIn, address tokenIn)`: Quotes a spot swap output factoring in fees and stable/volatile math.
- `name()`: Returns the overrideable ERC20 name set during initialization or via admin change.
- `symbol()`: Returns the overrideable ERC20 symbol set during initialization or via admin change.

### `contracts/contracts/Router.sol`
It orchestrates multi-hop swaps, liquidity management, and zap flows across approved pools.
- `receive()`: Accepts ETH unwraps exclusively from the canonical WETH contract.
- `sortTokens(address tokenA, address tokenB)`: Deterministically orders a token pair and guards against zero addresses.
- `poolFor(address tokenA, address tokenB, bool stable, address _factory)`: Predicts the canonical pool address for a pair using the registry-approved factory.
- `getReserves(address tokenA, address tokenB, bool stable, address _factory)`: Returns token-ordered reserves from a target pool.
- `getAmountsOut(uint256 amountIn, Route[] memory routes)`: Quotes output amounts across a sequence of swap routes.
- `quoteAddLiquidity(address tokenA, address tokenB, bool stable, address _factory, uint256 amountADesired, uint256 amountBDesired)`: Simulates optimal deposit amounts and LP to mint for a pool.
- `quoteRemoveLiquidity(address tokenA, address tokenB, bool stable, address _factory, uint256 liquidity)`: Estimates token amounts received when burning LP at current reserves.
- `addLiquidity(address tokenA, address tokenB, bool stable, uint256 amountADesired, uint256 amountBDesired, uint256 amountAMin, uint256 amountBMin, address to, uint256 deadline)`: Supplies tokens into a pool (creating it if necessary) and mints LP to the recipient.
- `addLiquidityETH(address token, bool stable, uint256 amountTokenDesired, uint256 amountTokenMin, uint256 amountETHMin, address to, uint256 deadline)`: Adds liquidity against ETH by wrapping/unwrapping WETH under the hood.
- `removeLiquidity(address tokenA, address tokenB, bool stable, uint256 liquidity, uint256 amountAMin, uint256 amountBMin, address to, uint256 deadline)`: Burns LP to withdraw both tokens subject to slippage checks.
- `removeLiquidityETH(address token, bool stable, uint256 liquidity, uint256 amountTokenMin, uint256 amountETHMin, address to, uint256 deadline)`: Burns LP from a token/WETH pool and unwraps ETH to the receiver.
- `removeLiquidityETHSupportingFeeOnTransferTokens(address token, bool stable, uint256 liquidity, uint256 amountTokenMin, uint256 amountETHMin, address to, uint256 deadline)`: Withdrawal helper tolerant of fee-on-transfer tokens on the ERC20 leg.
- `swapExactTokensForTokens(uint256 amountIn, uint256 amountOutMin, Route[] calldata routes, address to, uint256 deadline)`: Executes an exact-input token swap across the provided route path.
- `swapExactETHForTokens(uint256 amountOutMin, Route[] calldata routes, address to, uint256 deadline)`: Wraps ETH then swaps it for tokens across the provided routes.
- `swapExactTokensForETH(uint256 amountIn, uint256 amountOutMin, Route[] calldata routes, address to, uint256 deadline)`: Swaps ERC20 input tokens into ETH via WETH unwrap on the final hop.
- `UNSAFE_swapExactTokensForTokens(uint256[] memory amounts, Route[] calldata routes, address to, uint256 deadline)`: Executes a swap using caller-supplied per-hop balances without recalculating path outputs.
- `swapExactTokensForTokensSupportingFeeOnTransferTokens(uint256 amountIn, uint256 amountOutMin, Route[] calldata routes, address to, uint256 deadline)`: Exact-input swap variant that tolerates fee-on-transfer tokens mid-route.
- `swapExactETHForTokensSupportingFeeOnTransferTokens(uint256 amountOutMin, Route[] calldata routes, address to, uint256 deadline)`: Fee-on-transfer aware ETH-in swap helper.
- `swapExactTokensForETHSupportingFeeOnTransferTokens(uint256 amountIn, uint256 amountOutMin, Route[] calldata routes, address to, uint256 deadline)`: Fee-on-transfer aware swap from tokens into ETH.
- `zapIn(Zap calldata zapInPool, uint256 amountIn, uint256 amountOutMin, Route[] calldata routes, uint256 deadline)`: Performs a one-sided deposit by swapping and providing liquidity according to the zap plan.
- `zapOut(uint256 liquidity, Zap calldata zapOutPool, uint256 amountOutMin, Route[] calldata routes, uint256 deadline)`: Burns LP, optionally swaps one leg, and returns consolidated assets per zap instructions.
- `generateZapInParams(Route[] calldata routes, uint256 amountIn)`: Quotes expected intermediate amounts for an inbound zap configuration.
- `generateZapOutParams(uint256 liquidity, Zap calldata zapOutPool)`: Quotes expected token outputs for a zap-out plan.
- `quoteStableLiquidityRatio(address tokenA, address tokenB, address _factory)`: Estimates the optimal deposit ratio for stable pools based on oracle quotes.

### `contracts/contracts/PoolFees.sol`
It escrows pool-generated swap fees so LP shares cannot be exploited during balance changes.
- `claimFeesFor(address _recipient, uint256 _amount0, uint256 _amount1)`: Called by the paired pool to pay out accumulated fees to a recipient.

### `contracts/contracts/factories/FactoryRegistry.sol`
It tracks registry-approved pool, gauge, and reward factories across the protocol.
- `approve(address poolFactory, address votingRewardsFactory, address gaugeFactory)`: Registers a factory tuple so routers and voters can deploy through it.
- `unapprove(address poolFactory)`: Removes a pool factory (and its paired factories) from the approved set, except the immutable fallback.
- `setManagedRewardsFactory(address _newManagedRewardsFactory)`: Updates the managed rewards factory pointer used for bribe deployments.
- `managedRewardsFactory()`: Returns the currently authorized managed rewards factory.
- `factoriesToPoolFactory(address poolFactory)`: Looks up the voting-reward and gauge factories associated with a pool factory.
- `poolFactories()`: Returns the array of all approved pool factory addresses.
- `isPoolFactoryApproved(address poolFactory)`: Checks whether a pool factory is currently approved.
- `poolFactoriesLength()`: Returns the number of registered pool factories.

### `contracts/contracts/libraries/VelodromeTimeLibrary.sol`
It provides internal helpers for aligning timestamps to weekly epochs and vote windows; no public or external functions are exposed.

## Tokenomy contracts

### `contracts/contracts/Velo.sol`
It is the ERC20 emission token with a gated minter role.
- `setMinter(address _minter)`: Assigns the authorized minter contract.
- `mint(address account, uint256 amount)`: Allows the minter to mint new VELO to the requested account.

### `contracts/contracts/VotingEscrow.sol`
It manages vote-escrowed VELO (veNFTs), including managed locks, splits, delegation, and metadata.
- `createManagedLockFor(address _to)`: Mints a new managed veNFT controlled by the caller for the recipient.
- `depositManaged(uint256 _tokenId, uint256 _mTokenId)`: Moves a regular veNFT into a managed veNFT container.
- `withdrawManaged(uint256 _tokenId)`: Pulls a veNFT out of a managed veNFT and restores direct ownership.
- `setAllowedManager(address _allowedManager)`: Grants or revokes approval for a manager to control the caller's managed locks.
- `setManagedState(uint256 _mTokenId, bool _state)`: Toggles whether a managed veNFT can accept additional deposits.
- `setTeam(address _team)`: Updates the team address with protocol-level permissions.
- `setArtProxy(address _proxy)`: Points veNFT metadata rendering to a new art proxy.
- `tokenURI(uint256 _tokenId)`: Returns the metadata URI generated by the active art proxy.
- `ownerOf(uint256 _tokenId)`: Returns the owner or manager address for the specified veNFT.
- `balanceOf(address _owner)`: Returns the number of veNFTs owned by an address.
- `getApproved(uint256 _tokenId)`: Returns the single-token approved operator for a veNFT.
- `isApprovedForAll(address _owner, address _operator)`: Checks operator-wide approvals.
- `isApprovedOrOwner(address _spender, uint256 _tokenId)`: Convenience helper that exposes approval status to external callers.
- `approve(address _approved, uint256 _tokenId)`: Grants single-token transfer approval.
- `setApprovalForAll(address _operator, bool _approved)`: Grants or revokes operator-wide approvals.
- `transferFrom(address _from, address _to, uint256 _tokenId)`: Transfers a veNFT between owners.
- `safeTransferFrom(address _from, address _to, uint256 _tokenId)`: ERC721 safe transfer without data payload.
- `safeTransferFrom(address _from, address _to, uint256 _tokenId, bytes memory _data)`: ERC721 safe transfer with additional data.
- `supportsInterface(bytes4 _interfaceID)`: Reports supported ERC165 interfaces.
- `locked(uint256 _tokenId)`: Returns the lock parameters (amount, end, permanence) for a veNFT.
- `userPointHistory(uint256 _tokenId, uint256 _loc)`: Returns historical voting power checkpoints for a veNFT.
- `pointHistory(uint256 _loc)`: Returns global supply checkpoint data for an epoch.
- `checkpoint()`: Forces an up-to-date supply checkpoint.
- `depositFor(uint256 _tokenId, uint256 _value)`: Adds balance to an existing lock on behalf of the owner.
- `createLock(uint256 _value, uint256 _lockDuration)`: Locks VELO for a new veNFT for the provided duration.
- `increaseAmount(uint256 _tokenId, uint256 _value)`: Adds more VELO into an existing lock.
- `increaseUnlockTime(uint256 _tokenId, uint256 _lockDuration)`: Extends the unlock time for a veNFT.
- `withdraw(uint256 _tokenId)`: Withdraws unlocked tokens, burning the veNFT.
- `merge(uint256 _from, uint256 _to)`: Merges two veNFTs under common ownership into one lock.
- `split(uint256 _from, uint256 _amount)`: Splits part of a veNFT's balance into two new veNFTs.
- `toggleSplit(address _account, bool _bool)`: Enables or disables split permissions for an account.
- `lockPermanent(uint256 _tokenId)`: Converts an expiring lock into a permanent lock.
- `unlockPermanent(uint256 _tokenId)`: Reverts a permanent lock back into a standard expiring lock.
- `balanceOfNFT(uint256 _tokenId)`: Returns the current voting power of a veNFT at the present timestamp.
- `balanceOfNFTAt(uint256 _tokenId, uint256 _t)`: Returns historical voting power at a specific timestamp.
- `totalSupply()`: Returns the aggregate veNFT voting supply at the current timestamp.
- `totalSupplyAt(uint256 _timestamp)`: Returns historical aggregate voting supply at a timestamp.
- `setVoterAndDistributor(address _voter, address _distributor)`: Sets trusted voter and rewards distributor contracts.
- `voting(uint256 _tokenId, bool _voted)`: Records whether a veNFT is actively voting this epoch.
- `delegates(uint256 delegator)`: Returns the delegatee veNFT for a delegator.
- `checkpoints(uint256 _tokenId, uint48 _index)`: Returns delegation checkpoints for a veNFT.
- `getPastVotes(address _account, uint256 _tokenId, uint256 _timestamp)`: Reads historical delegated votes at a timestamp.
- `getPastTotalSupply(uint256 _timestamp)`: Returns historical delegated vote supply.
- `delegate(uint256 delegator, uint256 delegatee)`: Assigns delegated voting power to another veNFT.
- `delegateBySig(uint256 delegator, uint256 delegatee, uint256 nonce, uint256 expiry, uint8 v, bytes32 r, bytes32 s)`: Delegates using an EIP-712 signature.
- `clock()`: Returns the timestamp-based governance clock used for ERC-6372 compatibility.
- `CLOCK_MODE()`: Declares the string identifier for the governance clock mode.

### `contracts/contracts/Minter.sol`
It mints weekly VELO emissions, manages tail-emission nudges, and forwards growth to rewards and team wallets.
- `setTeam(address _team)`: Starts a pending transfer of the team role to a new address.
- `acceptTeam()`: Lets the pending team accept control.
- `setTeamRate(uint256 _rate)`: Adjusts the percentage of emissions reserved for the team, within bounds.
- `calculateGrowth(uint256 _minted)`: Computes the rebase amount needed to offset ve dilution for the week.
- `nudge()`: Allows the epoch governor to adjust the tail emission rate based on governance outcome.
- `updatePeriod()`: Advances to a new weekly period, mints emissions, and notifies downstream contracts.

### `contracts/contracts/RewardsDistributor.sol`
It distributes weekly rebases to eligible veNFTs and routes matured locks directly to owners.
- `checkpointToken()`: Records newly minted VELO intended for distribution (minter-only).
- `claimable(uint256 _tokenId)`: Returns how much VELO a veNFT can claim for accrued rebases.
- `claim(uint256 _tokenId)`: Claims rewards for a single veNFT, auto-depositing into the lock or transferring if expired.
- `claimMany(uint256[] calldata _tokenIds)`: Batch claims rewards across multiple veNFTs.
- `setMinter(address _minter)`: Updates the authorized minter address (minter-only).

### `contracts/contracts/VeArtProxy.sol`
It renders veNFT SVG metadata and provides deterministic art configuration helpers.
- `tokenURI(uint256 _tokenId)`: Produces the base64-encoded metadata URI for a veNFT.
- `lineArtPathsOnly(uint256 _tokenId)`: Returns only the SVG path commands for custom renderers.
- `generateConfig(uint256 _tokenId)`: Derives drawing parameters from veNFT attributes and randomness.
- `twoStripes(Config memory cfg, int256 l)`: Generates line coordinates for the "Two Stripes" motif.
- `circles(Config memory cfg, int256 l)`: Generates line coordinates for the "Circles" motif.
- `interlockingCircles(Config memory cfg, int256 l)`: Generates line coordinates for the "Interlocking Circles" motif.
- `corners(Config memory cfg, int256 l)`: Generates line coordinates for the "Corners" motif.
- `curves(Config memory cfg, int256 l)`: Generates line coordinates for the "Curves" motif.
- `spiral(Config memory cfg, int256 l)`: Generates line coordinates for the "Spiral" motif.
- `explosion(Config memory cfg, int256 l)`: Generates line coordinates for the "Explosion" motif.
- `wormhole(Config memory cfg, int256 l)`: Generates line coordinates for the "Wormhole" motif.

## Protocol mechanics contracts

### `contracts/contracts/Voter.sol`
It coordinates veNFT voting, gauge deployments, emissions distribution, and claim aggregation.
- `epochStart(uint256 _timestamp)`: Pure helper returning the epoch start for a timestamp.
- `epochNext(uint256 _timestamp)`: Pure helper returning the next epoch boundary.
- `epochVoteStart(uint256 _timestamp)`: Pure helper returning the vote window opening.
- `epochVoteEnd(uint256 _timestamp)`: Pure helper returning the vote window closing.
- `initialize(address[] calldata _tokens, address _minter)`: Seeds whitelist tokens and sets the trusted minter (one-time).
- `setGovernor(address _governor)`: Transfers the voter governor role.
- `setEpochGovernor(address _epochGovernor)`: Assigns the epoch governor contract controlling emissions nudges.
- `setEmergencyCouncil(address _council)`: Updates the emergency council address for safety controls.
- `setMaxVotingNum(uint256 _maxVotingNum)`: Configures how many pools a veNFT may vote for per epoch.
- `reset(uint256 _tokenId)`: Clears a veNFT's votes once its lock is out of the vote window.
- `poke(uint256 _tokenId)`: Re-applies vote weights to reflect the veNFT's current balance.
- `vote(uint256 _tokenId, address[] calldata _poolVote, uint256[] calldata _weights)`: Casts votes for multiple pools with provided weights.
- `depositManaged(uint256 _tokenId, uint256 _mTokenId)`: Registers a veNFT under a managed veNFT for collective voting.
- `withdrawManaged(uint256 _tokenId)`: Removes a veNFT from a managed veNFT and re-enables direct voting.
- `whitelistToken(address _token, bool _bool)`: Adds or removes reward tokens from the global whitelist (governor-only).
- `whitelistNFT(uint256 _tokenId, bool _bool)`: Approves or revokes managed veNFTs for participation (governor-only).
- `createGauge(address _poolFactory, address _pool)`: Deploys a gauge and associated reward contracts for a pool.
- `killGauge(address _gauge)`: Emergency-stops a gauge, halting emissions.
- `reviveGauge(address _gauge)`: Re-enables a previously killed gauge.
- `length()`: Returns the number of pools tracked by the voter.
- `notifyRewardAmount(uint256 _amount)`: Receives weekly emissions from the minter and updates the distribution index.
- `updateFor(address[] memory _gauges)`: Bulk-syncs reward indices for multiple gauges.
- `updateFor(uint256 start, uint256 end)`: Syncs gauges within an index range.
- `updateFor(address _gauge)`: Syncs a single gauge's reward index.
- `claimRewards(address[] memory _gauges)`: Claims gauge staking rewards for the caller across multiple gauges.
- `claimIncentives(address[] memory _incentives, address[][] memory _tokens, uint256 _tokenId)`: Claims incentive rewards for a veNFT from specified reward contracts.
- `claimFees(address[] memory _fees, address[][] memory _tokens, uint256 _tokenId)`: Claims fee rewards for a veNFT from specified reward contracts.
- `distribute(uint256 _start, uint256 _finish)`: Pushes emissions into a range of gauges.
- `distribute(address[] memory _gauges)`: Pushes emissions into explicit gauges.

### `contracts/contracts/gauges/Gauge.sol`
It tracks pool stake balances, accrues emissions, and optionally triggers fee claims for its pool.
- `rewardPerToken()`: Returns the current reward-per-LP accumulator for emissions.
- `lastTimeRewardApplicable()`: Returns the timestamp up to which rewards accrue.
- `getReward(address _account)`: Pays out accrued emissions to a staker or authorized caller.
- `earned(address _account)`: Computes a staker's pending emissions based on checkpoints.
- `deposit(uint256 _amount)`: Stakes LP tokens on behalf of the caller.
- `deposit(uint256 _amount, address _recipient)`: Stakes LP tokens on behalf of another account.
- `withdraw(uint256 _amount)`: Unstakes LP tokens for the caller.
- `left()`: Returns the amount of emissions tokens remaining for distribution.
- `notifyRewardAmount(uint256 _amount)`: Pulls fees, updates reward rate, and schedules new emissions (voter-only).
- `notifyRewardWithoutClaim(uint256 _amount)`: Adds emissions without harvesting pool fees (factory admin-only).

### `contracts/contracts/rewards/Reward.sol`
It provides shared veNFT checkpointing logic for reward distribution contracts.
- `rewardsListLength()`: Returns how many reward tokens are tracked.
- `lockExpiry(uint256 tokenId)`: Returns the stored lock expiry for a veNFT within this reward.
- `biasCorrection(uint256 tokenId)`: Returns the stored bias correction for a veNFT.
- `lockExpiryAndBiasCorrection(uint256 tokenId)`: Returns both expiry and bias correction in one call.
- `globalRewardPointHistory(uint256 _epoch)`: Returns the stored global reward checkpoint for an epoch.
- `userRewardPointHistory(uint256 tokenId, uint256 _userRewardEpoch)`: Returns user checkpoint data for a veNFT.
- `getPriorSupplyIndex(uint256 timestamp)`: Resolves the supply checkpoint index at or before a timestamp.
- `supplyAt(uint256 timestamp)`: Returns the recorded aggregate voting supply at a timestamp.
- `getPriorBalanceIndex(uint256 tokenId, uint256 timestamp)`: Resolves the per-token checkpoint index before a timestamp.
- `balanceOfNFTAt(uint256 tokenId, uint256 timestamp)`: Returns historical veNFT voting power within this reward.
- `earned(address token, uint256 tokenId)`: Calculates pending rewards for a veNFT.
- `_deposit(uint256 amount, uint256 tokenId)`: External hook for gauges/voter to record added weight.
- `_withdraw(uint256 amount, uint256 tokenId)`: External hook to remove recorded weight.
- `getReward(uint256 tokenId, address[] memory tokens)`: External stub for subclasses to implement reward claims.
- `notifyRewardAmount(address token, uint256 amount)`: External stub for subclasses to accept new rewards.

### `contracts/contracts/rewards/VotingReward.sol`
It restricts reward claims to veNFT owners and handles payout forwarding to recipients.
- `getReward(uint256 tokenId, address[] memory tokens)`: Validates ownership/approval then pulls rewards via `_getReward`.
- `notifyRewardAmount(address token, uint256 amount)`: Hook for inheriting contracts to authorize reward funding.

### `contracts/contracts/rewards/FeesVotingReward.sol`
It distributes pool trading fees to veNFT voters in the following epoch.
- `notifyRewardAmount(address token, uint256 amount)`: Accepts authorized fee tokens from the paired gauge for distribution.

### `contracts/contracts/rewards/IncentiveVotingReward.sol`
It distributes externally sponsored incentives to veNFT voters for the next epoch.
- `notifyRewardAmount(address token, uint256 amount)`: Accepts approved incentive tokens from external providers.

## Governance contracts

### `contracts/contracts/VeloGovernor.sol`
It governs protocol-wide parameters with veto and comment-weighting extensions.
- `votingDelay()`: Returns the fixed proposal voting delay.
- `votingPeriod()`: Returns the configured voting period duration.
- `setProposalNumerator(uint256 numerator)`: Updates the quorum fraction numerator for proposals.
- `proposalThreshold()`: Returns the minimum voting power required to propose.
- `setTeam(address _pendingTeam)`: Starts the transfer of the team role controlling comment requirements.
- `acceptTeam()`: Allows the pending team to accept the role.
- `setVetoer(address _vetoer)`: Proposes a new veto guardian address.
- `acceptVetoer()`: Lets the pending vetoer accept control.
- `veto(uint256 _proposalId)`: Exercises the veto right to cancel a proposal.
- `renounceVetoer()`: Allows the active vetoer to relinquish the role.
- `setCommentWeighting(uint256 _commentWeighting)`: Sets the minimum voting weight needed to post proposal comments.
- `proposalDeadline(uint256 proposalId)`: Returns the proposal deadline considering late-quorum protections.

### `contracts/contracts/EpochGovernor.sol`
It manages weekly tail-emission nudges via epoch-scoped proposals.
- `state(uint256 _proposalId)`: Returns the proposal state, marking executed, canceled, pending, active, or resolved outcomes.
- `propose(uint256 _tokenId, address[] memory _targets, uint256[] memory _values, bytes[] memory _calldatas, string memory _description)`: Allows veNFT holders to propose the canonical `nudge` call for the current epoch.
- `execute(address[] memory _targets, uint256[] memory _values, bytes[] memory _calldatas, bytes32 _descriptionHash)`: Executes a successful proposal (or acknowledges defeat/expiry) and records the result.
- `relay(address, uint256, bytes calldata)`: Disabled governance relay hook that always reverts when called.
- `votingDelay()`: Returns the proposal voting delay (two seconds after vote window opens).
- `votingPeriod()`: Returns the voting duration aligned with the weekly epoch minus the delay buffers.
- `quorum(uint256 _timepoint)`: Returns the quorum requirement (zero, because proposals are single-option nudges).
