# BUILD — Deployment Notes

This is the Hyperlynx fork of Uniswap V3 periphery, deployed on HyperEVM (chain id 999)
against the Hyperlynx v3-core factory
[`0x418CB4e449869e97DB45586EBD9350E1d0424f95`](https://hyperevmscan.io/address/0x418CB4e449869e97DB45586EBD9350E1d0424f95).

## `POOL_INIT_CODE_HASH`

[`contracts/libraries/PoolAddress.sol`](./contracts/libraries/PoolAddress.sol) hardcodes:

```
POOL_INIT_CODE_HASH = 0x67e280fb5fe3774b688c3807ae41dda3d9f6325a25aaa732cff2c3cd0d900c49
```

This is the keccak256 hash of the `UniswapV3Pool` creation code as compiled with the
build configuration pinned in the
[v3-core `BUILD.md` / `foundry.toml`](https://github.com/HyperlynxFi/v3-core/blob/main/BUILD.md)
(solc 0.7.6, optimizer runs = 200, evm_version = istanbul, ipfs bytecode metadata). It
intentionally differs from the canonical Uniswap constant: the hash is derived from the
deployed factory's pool creation code, and CREATE2 pool-address computation and swap
callback validation in this periphery only work against that factory if the constants
match.

## Deployed contracts (HyperEVM)

| Contract | Address |
|---|---|
| SwapRouter | [`0xC141DaCe9596D985e5EF812de55466Dfa78e9BbF`](https://hyperevmscan.io/address/0xC141DaCe9596D985e5EF812de55466Dfa78e9BbF) |
| NonfungiblePositionManager | [`0xfA62Fe94B6eDA4a13f57eD0224848bAD45954b71`](https://hyperevmscan.io/address/0xfA62Fe94B6eDA4a13f57eD0224848bAD45954b71) |
| QuoterV2 | [`0x093A4ba695105666152b2E4620E012ADa8A4E57D`](https://hyperevmscan.io/address/0x093A4ba695105666152b2E4620E012ADa8A4E57D) |
| NonfungibleTokenPositionDescriptor | [`0x06693C72cF668307aCe8F6b5D0b9Eea3319027a0`](https://hyperevmscan.io/address/0x06693C72cF668307aCe8F6b5D0b9Eea3319027a0) |

## On-chain verification

The deployed periphery is self-consistent with the live factory, provable from chain
state (`eth_getCode` on the addresses above):

1. The runtime bytecode of SwapRouter, NonfungiblePositionManager, and QuoterV2 each
   embed `0x67e280fb…900c49` — the same value the factory exposes via
   `POOL_INIT_CODE_HASH()`.
2. CREATE2 recomputation with this hash reproduces every live pool address returned by
   `factory.getPool()`.
3. Each contract's CBOR metadata tail decodes to an `ipfs` multihash plus `solc 0.7.6`,
   matching the pinned toolchain.

## ⚠ Hash sensitivity

`POOL_INIT_CODE_HASH` must always equal the hash of the pool creation code produced by
the v3-core build configuration. Changing the constant here, or changing anything in
v3-core that affects `UniswapV3Pool` creation code (sources, solc version,
`optimizer_runs`, `evm_version`, `bytecode_hash`), silently breaks pool address
derivation and callback validation against the live factory. Source-level changes to
either side are tagged **next full deployment only**.
