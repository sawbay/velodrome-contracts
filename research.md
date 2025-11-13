# Protocol Overview

## AMM contracts

| Filename | Description |
| --- | --- |
| `contracts/Pool.sol` | AMM constant-product implementation similar to Uniswap V2 liquidity pools. |
| `contracts/Router.sol` | Handles multi-pool swaps, deposit/withdrawal, similar to Uniswap V2 router interfaces. |
| `contracts/PoolFees.sol` | Stores the liquidity pool trading fees, kept separate from reserves and paid out on claim. |
| `contracts/libraries/VelodromeLibrary.sol` | Provides router-related helpers, including price-impact and quote calculations. |
| `contracts/FactoryRegistry.sol` | Registry of factories approved for creating pools, gauges, incentives, and managed rewards. |

## Tokenomy contracts

| Filename | Description |
| --- | --- |
| `contracts/Velo.sol` | Protocol ERC-20 token. |
| `contracts/VotingEscrow.sol` | Protocol ERC-721 (ve)NFT representing the protocol vote-escrow lock, with merge, split, and managed NFT support. |
| `contracts/Minter.sol` | Protocol token minter that distributes emissions to `Voter.sol` and rebases to `RewardsDistributor.sol`. |
| `contracts/RewardsDistributor.sol` | Handles rebase distribution for veNFT lockers. |
| `contracts/VeArtProxy.sol` | veNFT art proxy contract retained for upgradeability. |

## Protocol mechanics contracts

| Filename | Description |
| --- | --- |
| `contracts/Voter.sol` | Handles epoch votes, gauge and voting reward creation, and emission distribution to gauges. |
| `contracts/gauges/Gauge.sol` | Gauge attached to a pool; based on veNFT votes it distributes protocol emissions to LP token depositors while redirecting pool fees. |
| `contracts/rewards/` | Namespace for reward helpers used by gauges and the voter. |
| `contracts/rewards/Reward.sol` | Base reward contract inherited for distributing rewards to stakers. |
| `contracts/rewards/VotingReward.sol` | Shared voting reward logic used by `FeesVotingReward.sol` and `IncentiveVotingReward.sol`, distributing rewards by checkpointed votes. |
| `contracts/rewards/FeesVotingReward.sol` | Stores LP fees (from the gauge via `PoolFees.sol`) to distribute in the current voting epoch. |
| `contracts/rewards/IncentiveVotingReward.sol` | Stores externally provided incentives to distribute to voters each epoch. |

## Governance contracts

| Filename | Description |
| --- | --- |
| `contracts/governance/VeloGovernor.sol` | OpenZeppelin Governor controlling protocol-level permissions such as whitelisting tokens, updating emissions, and creating managed veNFTs. |
| `contracts/governance/EpochGovernor.sol` | Epoch-based governor used exclusively for adjusting emissions. |
