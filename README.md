# LMSR

**A pool whose worst case a provider can read before they deposit.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://lmsr.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/LMSRHook.sol`](src/hooks/LMSRHook.sol)
- **Licence:** Apache-2.0

## How it works

A constant-product provider's exposure has no upper bound. As the price moves, the position converges on holding only the losing side, and the loss against simply having held the two tokens grows without limit. Everyone knows this and nobody can quote it, because the answer depends on where the price goes and the curve quotes at every price there is.

Robin Hanson's logarithmic market scoring rule was built for exactly this problem, in prediction markets, in 2003: a market maker that will always quote, and whose total loss is bounded by a constant chosen when the market opens. The bound is the whole reason it is used. It has never been the curve of a general-purpose AMM because a bounded-loss maker must also be a bounded-*range* maker, and a pool that stops quoting past a price is a worse product right up until the day it is the only pool that did not get emptied.

This pool holds the invariant exp(-reserve0 / b) + exp(-reserve1 / b) = k which prices token0 in token1 at exp((reserve1 - reserve0) / b): exactly one at balance, falling smoothly as the pool fills with token0. The parameter `b` is depth and it is also the bound. Larger `b` is flatter near parity and a wider quoting range; smaller `b` is a tighter market that gives up sooner.

There is one number to choose and it means one thing. What a provider gets that no constant-product pool offers is a figure: {maxLoss} is what this curve can cost them in the worst case anybody can construct, published on-chain, before they deposit. Beyond the range the curve refuses the trade instead of quoting a price it cannot honour, which is the same promise stated from the other end.

## Prior art

The scoring rule is Hanson's, and Gnosis, Augur and Polymarket's predecessors all ran variants of it for prediction markets, where outcomes sum to one and shares are minted rather than held. Constant-product, StableSwap and the v4 custom curves built on them are all unbounded-range makers. Bringing a bounded-loss scoring rule to a two-sided token pool, with the bound published as a view a provider reads before depositing, is the contribution here.

## Where it does not help

The range is finite by construction, so a pool whose true price leaves the band stops quoting on that side and holds the losing asset until the price returns; that is the bounded loss being collected, not a malfunction. It is the right curve for pairs expected to stay near a ratio and the wrong one for anything that can genuinely revalue. The maths runs through fixed-point exponentials, so a swap costs meaningfully more gas than constant product. And `b` is fixed at deployment: a pool that wants a different depth is a different pool.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
// This hook needs no configuration.

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

This hook takes no per-pool configuration.

## What it reverts with

| Error | Meaning |
| --- | --- |
| `AlreadyInitialized()` | Hook was already initialized. |
| `AmountTooSmall()` | A deposit was too small to mint any shares, or a withdrawal too small to return anything. |
| `ERC20InsufficientAllowance(address,uint256,uint256)` | Indicates a failure with the `spender`’s `allowance`. Used in transfers. |
| `ERC20InsufficientBalance(address,uint256,uint256)` | Indicates an error related to the current `balance` of a `sender`. Used in transfers. |
| `ERC20InvalidApprover(address)` | Indicates a failure with the `approver` of a token to be approved. Used in approvals. |
| `ERC20InvalidReceiver(address)` | Indicates a failure with the token `receiver`. Used in transfers. |
| `ERC20InvalidSender(address)` | Indicates a failure with the token `sender`. Used in transfers. |
| `ERC20InvalidSpender(address)` | Indicates a failure with the `spender` to be approved. Used in approvals. |
| `ExpiredPastDeadline()` | A liquidity modification order was attempted to be executed after the deadline. |
| `InsufficientInitialLiquidity()` | The first deposit must exceed the permanently locked minimum. |
| `InsufficientReserves()` | The trade wants more than the curve will give up at any price. |
| `InvalidB()` | A liquidity parameter of zero would make the curve a step function. |
| `InvalidFee()` | A fee at or above the whole trade is not a fee. |
| `InvalidNativePayer(address)` | The native currency was settled on behalf of a `payer` other than the contract paying it. |
| `InvalidNativeValue()` | Native currency was not sent with the correct amount. |
| `LiquidityOnlyViaHook()` | Liquidity was attempted to be added or removed via the `PoolManager` instead of the hook. |
| `NoReserves()` | The pool holds nothing on one side, so there is no curve to quote against. |
| `OutOfRange()` | The reserves are too lopsided for the curve to quote. See {MAX_EXPONENT}. |
| `PoolNotInitialized()` | Pool was not initialized. |
| `SafeCastOverflowedIntToUint(int256)` | An int value doesn't fit in a uint of `bits` size. |
| `SafeCastOverflowedUintToInt(uint256)` | A uint value doesn't fit in an int of `bits` size. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |
| `TooMuchSlippage()` | Principal delta of liquidity modification resulted in too much slippage. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 5 of the fourteen:

- `beforeInitialize`
- `beforeAddLiquidity`
- `beforeRemoveLiquidity`
- `beforeSwap`
- `beforeSwapReturnsDelta`

Mask: `0x2a88`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # LMSR
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # curve, custom-curve, bounded-loss, scoring-rule, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/lmsr
cd lmsr
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
