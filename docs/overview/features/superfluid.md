---
sidebar_position: 7
---

# Superfluid Staking

## Abstract

Superfluid Staking provides the consensus layer more security with a
sort of "Proof of Useful Stake". Each person gets an amount of Osmo
representative of the value of their share of liquidity pool tokens
staked and delegated to validators, resulting in the security guarantee
of the consensus layer to also be based on GAMM LP shares. The OSMO
token is minted and burned in the context of Superfluid Staking.
Throughout all of this, OSMO's supply is preserved in queries to the
bank module.

### The process

All of the below methods are found under the [Superfluid
modules](https://github.com/osmosis-labs/osmosis/tree/main/x/superfluid).

- The `SuperfluidDelegate` method stores your share of bonded
  liquidity pool tokens, with `validateLock` as a verifier for lockup
  time.
- `GetSuperfluidOsmo` mints OSMO tokens each day for delegation as a
  representative of the value of your pool share. This amount is
  minted because the staking module at the moment requires staked
  tokens to be in OSMO. This amount is burned each day and re-minted
  to keep the representative amount of the value of your pool share
  accurate. The lockup duration is guaranteed from the underlying
  lockup module.
- `GetExpectedDelegationAmount` iterates over each (denom, delegate)
  pair and checks for how much OSMO we have delegated. The difference
  from the current balance to what is expected is burned / minted to
  match with the expected.
- A `messageServer` method executes the Superfluid delegate message.
- `syntheticLockup` is used to index bond holders and tracking their
  addresses for reward distribution or potentially slashing purposes.
  These track whether if your Superfluid stake is currently bonding or
  unbonding.
- An `IntermediaryAccount` is mostly used for the actual reward
  distribution or slashing events, and are responsible for
  establishing the connection between each superfluid staked lock and
  their delegation to the validator. These work by transferring the
  superfluid OSMO to their respective delegators. Rewards are linearly
  scaled based on how much you have locked for a given (validator,
  denom) pair. Rewards are first moved to the incentive gauges, then
  distributed from the gauges. In this way, we're using the existing
  gauge reward system for paying out superfluid staking rewards and
  tracking the amount you have superfluidly staked using the lockup
  module.
- Rewards are distributed per epoch, which is currently a day.
  `abci.go` checks whether or not the current block is at the
  beginning of the epoch using `BeginBlock`.
- Superfluid staking will continue to expand to other Osmosis pools
  based on governance proposals and vote turnouts.

### Example

If Alice has 500 GAMM tokens bonded to the ATOM \<\> OSMO, she will have
the equivalent value of OSMO minted, delegated to her chosen staker, and
burned for her each day with Superfluid staking. On the user side, all
she has to know is who she wants to delegate her tokens to. In order to
switch delegation, she has to unbond her tokens from the pool first and
then redeposit. Bob, who has a share of the same liquidity pool before
Superfluid Staking went live, also has to re-deposit into the pool for
the above process to kickstart.

### Why mint Osmo? How is this method safe and accurate?

Superfluid staking requires the minting of OSMO because in order to
stake on the Osmosis chain, OSMO tokens are required as the chosen
collateral. Synthetic Osmo is minted here as a representative of the
value of each superfluid staker's liquidity pool tokens.

The pool tokens are acquired by the user from normally staking in a
liquidity pool. They get minted an amount of OSMO equivalent to the
value of their GAMM pool tokens. This method is accurate because
querying the value OSMO every day allows for burning and minting
according to the difference in value of OSMO relative to the expected
delegation amount (as seen with
[GetExpectedDelegationAmount](https://github.com/osmosis-labs/osmosis/blob/main/x/superfluid/keeper/stake.go)).
It's like having a price oracle for fairly calculating the amount the
user has superfluidly staked.

On epoch (start of every day), we read from the lockup module how much
GAMM tokens we have locked which acts as an oracle for the
representative price of the GAMM token shares. The superfluid module has
"hooks" messages to refresh delegation amounts
(`RefreshIntermediaryDelegationAmounts`) and to increase delegation on
lockup (`IncreaseSuperfluidDelegation`). Then, we see whether or not the
superfluid OSMO currently delegated is worth more or less than this
expected delegation amount amount. If the OSMO is worth more, we do
instant undelegations and immediately burn the OSMO. If less, we mint
OSMO and update the amount delegated. 

This minting is safe because we strict constrain the permissions of Bank
(the module that burns and mints OSMO) to do what it's designed to do.
The authority is mediated through `mintOsmoTokensAndDelegate` and
`forceUndelegateAndBurnOsmoTokens` keeper methods called by the
`SuperfluidDelegate` and `SuperfluidUndelegate` message handlers for the
tokens. The hooks above that increase delegation and refresh delegation
amounts also call this keeper method.

The delegation is then verified to not already be associated with an
intermediary account (to prevent double-staking), and is always
delegated or withdrawn taking into account various multipliers for
synthetic OSMO value (its worth with respect to the liquidity pool, and
a risk modifier) to prevent mint inaccuracies. Before minting, we also
check that the message sender is the owner of the locked funds; that the
lock is not unlocking; is locked for at least the unbonding period, and
is bonded to a single asset. We also check to see if the lock isn't
already in superfluid and that the same lock isn't currently being
unbonded.

On the end of each epoch, we iterate through all intermediary accounts
to withdraw delegation rewards they may have received and put it all
into the perpetual gauges corresponding to each account for reward
delegation.

### Bonding, unbonding, slashing

Here, we describe how token bonding and unbonding works, and what
happens to your superfluid tokens in the case of a slashing event.

### Bonding

When bonding, your input tokens are locked up and you are given GAMM
pool tokens in exchange. These GAMM pool tokens represent a share of the
total liquidity pool, and allows you to get transaction fees or
participate in external incentive gauge token distributions. When
bonding, on top of the regular bonding transaction there will also be a
selection of validators. As stated above, OSMO is also minted and burned
each day and superfluidly staked to whoever you have chosen to be your
validator. You gain additional APR as a reward for bolstering the
Osmosis chain's consensus integrity by delegating.

### Unbonding

When unbonding, superfluid tokens get un-delegated. After making sure
that the unbond message sender is the owner of their corresponding
locked funds, the existing synthetic lockup is deleted and replaced with
a new synthetic lockup for unbonding purposes. The undelegated OSMO is
then instantly withdrawn from the intermediate account and validator
using the InstantUndelegate function. The OSMO that was originally used
for representing your LP shares are burnt. Moves the tracker for
unbonding, allows the underlying lock to start unlocking if desired

## Concepts

### SyntheticLockups

SyntheticLockups are synthetica of PeriodLocks, but different in the
sense that they store suffix, which is a combination of
bonding/unbonding status + validator address. This is mainly used to
track whether an individual lock that has been superfluid staked has an
bonding status or a unbonding status from the staking delegations.

### Intermediary Account

Intermediary Accounts establishes the connections between the superfluid
staked locks and delegations to the validator. Intermediary accounts
exists for every denom + validator combination, so that it would group
locks with the same denom + validator selection. Superfluid staking a
lock would mint equivalent amount of OSMO of the lock and send it to the
intermediary account and the intermediarry accounts would be delegating
to the specified validator.

### Intermediary Account Connection

Intermediary Accounts Connection serves the role of tracking the locks
that an Intermediary Account is dedicated to.

## State

### Superfluid Asset

A superfluid asset is a alternative asset (non-OSMO) that is allowed by
governance to be used for staking.

It can only be updated by governance proposals. We validate at proposal
creation time that the denom + pool exists. (Are we going to ignore edge
cases around a reference pool getting deleted it)

### Intermediary Accounts

Lots of questions to be answered here

### Dedicated Gauges

Each intermediary account has has dedicated gauge where it sends the
delegation rewards to. Gauges are distributing the rewards to end users
at the end of the epoch.

### Synthetic Lockups created

At the moment, one lock can only be fully bonded to one validator.

### Osmo Equivalent Multipliers

The Osmo Equivalent Multiplier for an asset is the multiplier it has for
its value relative to OSMO.

Different types of assets can have different functions for calculating
their multiplier. We currently support two asset types.

1. Native Token

The multiplier for OSMO is always 1.

2. Gamm LP Shares

Currently we use the spot price for an asset based on a designated
osmo-basepair pool of an asset. The multiplier is set once per epoch, at
the beginning of the epoch. In the future, we will switch this out to
use a TWAP instead.
