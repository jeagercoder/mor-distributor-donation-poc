# MOR DistributorV2 — yield-donation emission redirection (PoC)

Unprivileged, net-profitable, repeatable redirection of the fixed daily MOR emission away from honest
stakers, against the **deployed** `DistributorV2` (`0xDf1AC1AC255d91F5f4B1E3B4Aef57c5350F64C7A`) on Ethereum
and the 5 shared-impl DepositPools under `rewardPoolIndex 0` (stETH/wETH/wBTC/USDC/USDT).

**Root cause:** `distributeRewards` measures each pool's yield as
`IERC20(yieldToken).balanceOf(distributor) - lastUnderlyingBalance` (DistributorV2:409–417), so an
unsolicited token transfer (donation) counts as "yield" and inflates that pool's share of `rewards_`.

This is a **severity escalation of Code4rena Capital-V2 QA-01** ("stETH dust-donation forces ~3k MOR/day
mint", rated QA/informational, unfixed): a *correctly-sized* donation lets an attacker who dominates a minor
pool **capture** the emission at a net profit and starve the honest stETH majority.

## Run

Mainnet-fork tests (no privileged accounts; attacker is an ordinary EOA). Needs an Ethereum full/archive RPC
— the tests hardcode `https://ethereum-rpc.publicnode.com`.

```
forge test -vv
```

Isolate untrusted code by running in the Foundry Docker image:
```
docker run --rm --network host -v "$PWD":/w -w /w ghcr.io/foundry-rs/foundry:latest "forge test -vv"
```

## Tests

- `test/DonationProbe.t.sol` — reads live params (daily emission ≈ 2892 MOR, `minRewardsDistributePeriod`
  = 86400s, 5 pools under index 0, per-pool organic yield).
- `test/DonationSkew.t.sol` — proves the redirection: baseline split `{stETH 2692, USDC 187, USDT 15,
  wETH 2.4, wBTC 0.0017}` → after a 0.04-wBTC donation `{wBTC 2015, stETH 819}` (total conserved).
- `test/DonationProfit.t.sol` / `DonationSweep.t.sol` — attacker net-profit; peak ≈ **+$562/day** at a
  ~0.012-wBTC (~$933) donation.
- `test/DonationE2E.t.sol` — **end-to-end**: stake → donate → `distributeRewards` → `claim(0, attacker)`;
  asserts the cross-chain mint dispatch recipient == attacker and amount = **837.6 MOR (attack)** vs
  **2.49 MOR (baseline)**. Only `L1SenderV2` is `vm.etch`-replaced with a spy so the LayerZero send doesn't
  revert on the fork; the L2 `MOROFT.mint` it targets is the same access-gated path every honest staker uses.

ChainLink prices are kept fresh across the time-warp via per-path `vm.mockCall` of each feed's **real
current price** (only freshness is mocked, no price value is changed).
