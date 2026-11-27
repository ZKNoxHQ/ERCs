---
title: Derivation Path for EIP-7702 Accounts
description: One BIP-44 rooted HD path for EIP-7702 delegated EOAs, split by protocol, account, purpose, chain and address index.
author: Renaud Dubois (@rdubois-crypto), Paul (@0xpaulio), zy0n (@zy0n), Simon Masson (@simonmasson)
discussions-to: <ethereum-magicians thread URL>
status: Draft
type: Standards Track
category: ERC
created: 2026-11-26
requires: 155, 7702
---

## Abstract

This ERC defines a single hierarchical deterministic derivation path for the secp256k1 keys of externally owned accounts (EOAs) that delegate their code with [EIP-7702](./eip-7702.md):

```
m/44'/60'/7702'/d'/a'/p'/k/w
```

The first three levels are fixed. The next five select a protocol `d`, an account `a`, a purpose `p`, a chain `k` and an address index `w`. The ERC also defines a registry of purpose values, a list of protocol identifiers, rules for allocating address indexes, an account discovery procedure, and the 20-byte request a signing device receives to derive a key in this subtree.

## Motivation

EIP-7702 lets an EOA install a delegation designator so that the account executes the code of a delegate contract. The account keeps its address and its private key. Wallets that create such accounts currently pick their own derivation paths, so a seed restored in another wallet shows different addresses and the funds appear to be missing.

Users also spread their activity over several addresses with different risk and privacy profiles: one address that executes DeFi operations completing in a single transaction, one that holds positions over time, deposit addresses given to third parties, one that pays gas. Each wallet invents its own layout for this.

Privacy protocols execute calls from fresh addresses after a withdrawal. Several applications serving different protocols can share one seed, and they must not hand out the same address. An address that is the same on every chain also links the activity of its owner across chains.

Hardware signers bind an application to a set of derivation path prefixes. A layout whose fixed part comes first allows a dedicated application to be restricted to its subtree, or to the subtree of one protocol.

This ERC addresses these points with the following requirements:

1. There is exactly one path, so that importing a seed or migrating between wallets yields the same accounts.
2. The path is rooted at `m/44'/60'`, the Ethereum coin type, so that existing Ethereum tooling can derive the keys.
3. The levels an application varies come after the levels fixed for it, so that hardware signers can lock an application to `m/44'/60'/7702'` or to the subtree of one protocol.
4. Keys of different protocols are separated, so that applications sharing a seed do not collide.
5. Addresses meant for one chain differ from those meant for other chains, so that activity on different chains is not linked by address.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Path

Keys are derived with [BIP-32](https://github.com/bitcoin/bips/blob/9f3d3bc6a60db16efbdc71ac8f7ea30217fc0c39/bip-0032.mediawiki) over secp256k1, from a master key typically obtained from a [BIP-39](https://github.com/bitcoin/bips/blob/9f3d3bc6a60db16efbdc71ac8f7ea30217fc0c39/bip-0039.mediawiki) seed, along the path:

```
m / 44' / 60' / 7702' / d' / a' / p' / k / w
```

| Level | Name | Hardened | Value |
|-------|------|----------|-------|
| 1 | BIP-43 purpose | yes | `44'`, as in [BIP-44](https://github.com/bitcoin/bips/blob/9f3d3bc6a60db16efbdc71ac8f7ea30217fc0c39/bip-0044.mediawiki) |
| 2 | Coin type | yes | `60'`, Ether in SLIP-0044 |
| 3 | Account type | yes | `7702'` |
| 4 | Protocol `d` | yes | `0` or a protocol identifier, `0 <= d < 2^31` |
| 5 | Account `a` | yes | `0 <= a < 2^31` |
| 6 | Purpose `p` | yes | a value from the purpose registry, `0 <= p < 2^31` |
| 7 | Chain `k` | no | `0` or a chain ID, `0 <= k < 2^31` |
| 8 | Address index `w` | no | `0 <= w < 2^31` |

The Ethereum address of the account is the address of the public key at the leaf of this path. A wallet conforming to this ERC MUST derive the keys of the persistent EIP-7702 accounts it creates at this path and MUST NOT use any other path for them.

Single-use senders that a protocol derives in its own subtree are out of scope, for example a sender derived from a privacy pool note and used for a single withdrawal.

Level 3 holds the number of the EIP that defines the account type, hardened. This ERC only defines the value `7702'`.

### Protocol

`d = 0` designates accounts that do not act for a specific protocol. An application acting for a privacy protocol MUST derive the keys it uses for that protocol under the identifier of that protocol. The identifier of a protocol is the index its own key derivation already uses to name it, without the hardened bit.

The following identifiers are known at the time of writing. This list is informative. A protocol whose key derivation has no such index picks a nonzero value that does not appear in it.

| Protocol | `d` | Index it already uses |
|----------|-----|-----------------------|
| Railgun | `1984` | Coin type of its key derivation, `m/44'/1984'/...` and `m/420'/1984'/...` |
| Tornado Cash | `29795` | Purpose of the Tornado Cash derivation in the Kohaku SDK, `m/29795'/1'/...` |
| Privacy Pools | `28784` | Purpose of the Privacy Pools derivation in the Kohaku SDK, `m/28784'/1'/...` |

### Purpose registry

| `p` | Name | Use |
|-----|------|-----|
| 0 | Atomic DeFi | Addresses executing DeFi operations whose effects complete within one transaction, such as a batched approval and swap, or an exit from a privacy protocol followed by calls in the same transaction. Funds are not expected to remain on the address between transactions. |
| 1 | Non-atomic DeFi | Addresses holding DeFi positions across transactions, such as lending, liquidity provision, staking, or orders settled later. |
| 2 | Payment | Addresses given to a person or a service to receive funds, such as a deposit address whose incoming funds are then forwarded or shielded. |
| 3 | Gas | Addresses paying transaction fees, or funding other addresses of the same seed for transaction fees. |

An address executing calls after an exit from a privacy protocol MUST be derived with the identifier of that protocol as `d` and with `p = 0`. Wallets MUST use a new address index `w` for each such exit and MUST NOT give these addresses out to receive funds.

Values from `4` to `65535` are reserved for future ERCs. Wallets MUST NOT derive keys at a reserved value. Values from `65536` to `2^31 - 1` are for private use; their meaning is wallet specific and not interoperable.

### Chain

`k = 0` designates an address meant to be used on any chain. Otherwise `k` is the chain ID, as defined in [EIP-155](./eip-155.md), of the only chain on which the address is meant to be used. A chain whose ID is greater than or equal to `2^31` uses `k = 0`.

A key derived with `k != 0` MUST NOT sign a transaction or an EIP-7702 authorization for a chain other than `k`. In particular, it MUST NOT sign an EIP-7702 authorization whose chain ID is `0`.

Wallets SHOULD use `k` equal to the chain ID for purposes `0`, `1` and `3`, so that addresses used on different chains are not linked. Wallets SHOULD use `k = 0` for purpose `2` when the address is given out without naming a chain, so that funds sent on another chain reach an address the wallet watches.

### Address index

`w` starts at `0` and is incremented sequentially within each `(d, a, p, k)` branch. With `k = 0` the same counter serves every chain, because the address is the same on every chain. A wallet allocating a new index:

- MUST reserve and persist the index before signing anything with its key;
- MUST NOT reuse an index whose key has signed anything, including an operation that was abandoned or failed;
- SHOULD check that the candidate address has a zero nonce and no code before using it, on chain `k`, or on every chain it supports when `k = 0`;
- SHOULD NOT leave more than 20 consecutive reserved and unused indexes in a branch.

### Account discovery

A wallet restoring a seed SHOULD discover accounts as follows, for `d = 0` and for each protocol identifier it supports:

1. Set `a = 0`.
2. For each registered purpose `p`, and for `k = 0` and each chain ID the wallet supports, derive addresses for `w = 0, 1, 2, ...` and stop after 20 consecutive unused addresses. An address with `k != 0` is checked on chain `k` only, an address with `k = 0` on every chain the wallet supports.
3. If at least one address of account `a` is used, increment `a` and go to step 2. Otherwise stop.

An address is used if it has a nonzero nonce, a nonzero balance, nonempty code (including a delegation designator `0xef0100 || address`), or any other activity the wallet tracks. A payment address may have forwarded everything it received, so wallets SHOULD also count past incoming transfers as use for purpose `2`.

### Signing device request

A signing device that restricts an application to the prefix `m/44'/60'/7702'`, or to `m/44'/60'/7702'/d'` for one protocol, SHOULD receive the variable levels as a 20-byte request:

```
d || a || p || k || w
```

where each field is a 32-bit unsigned big-endian integer without the hardened bit. The device:

- MUST build the full path itself from the fixed prefix and the five fields, and MUST NOT accept a full path from the host for this subtree;
- MUST reject the request if any field is greater than or equal to `2^31`, if `p` is a reserved purpose, or, for an application restricted to one protocol, if `d` is not the identifier of that protocol;
- MUST set the hardened bit on `d`, `a` and `p`, and MUST NOT set it on `k` and `w`;
- MUST refuse to sign with a key derived with `k != 0` a transaction or an EIP-7702 authorization for another chain, including an authorization with chain ID `0`;
- SHOULD display the name of the protocol for a known `d`, the name of the purpose for a registered `p`, and the chain when `k != 0`.

## Rationale

**Single path.** Any conforming wallet restoring the same seed finds the same accounts, without asking the user to choose among candidate paths.

**Root at `m/44'/60'`.** Keys in this subtree are ordinary Ethereum keys. Wallets and hardware applications already allowed on `m/44'/60'` can derive them, which makes import and migration possible without a dedicated application.

**Fixed levels first.** Hardware signers declare the paths an application may use as prefixes and do not support a wildcard in the middle of a path. An earlier version of this layout, `m/44'/60'/a'/7702'/p'/c/w`, could not be locked to a subtree. Placing the account type at level 3 makes `m/44'/60'/7702'` a usable prefix, and placing the protocol at level 4 makes `m/44'/60'/7702'/d'` a usable prefix for an application dedicated to one protocol.

**Account type `7702'` at level 3.** As in [BIP-43](https://github.com/bitcoin/bips/blob/9f3d3bc6a60db16efbdc71ac8f7ea30217fc0c39/bip-0043.mediawiki), the index names the specification the subtree follows. Using the number of EIP-7702, instead of the number of this ERC, keeps the value known before number assignment and tied to the account type.

**Protocol before account.** Several applications can share one seed, for instance a Railgun wallet and a wallet handling Tornado Cash, and each keeps its own index counters. Without a protocol level both would hand out the same address: one address would then execute exits from two protocols, which links them on chain, and each application could overwrite the delegation of the other. A hardened protocol level separates their keys for every purpose and lets a device or a host restrict an application to the subtree of its protocol. Encoding the protocol in the bits of the purpose, or in a non-hardened level, would not allow such a restriction. Reusing the index each protocol already uses in its own key derivation needs no new registry and keeps the value recognizable.

**Account before purpose.** An account groups every purpose of one user profile, in the same way as a BIP-44 account. Purposes are subdivisions of an account.

**Hardened `d'`, `a'` and `p'`.** Public data or a leaked private key from one `(d, a, p)` subtree gives no information about any other subtree.

**Chain level.** An address that is the same on every chain links the activity of its owner across chains, so a chain level gives each chain its own addresses for persistent accounts. `k = 0` follows the meaning of chain ID `0` in EIP-7702 authorizations, valid on every chain. Binding a key to its chain lets a device refuse authorizations meant for another chain. Payment addresses default to `k = 0` because a payer may pick the wrong chain. Chain IDs of `2^31` or more do not fit in the level and fall back to `k = 0`.

**Non-hardened `k` and `w` after `p'`.** The extended public key at depth 6 lets a service derive every address of one purpose, on every chain, without holding private keys, for instance a payment backend generating deposit addresses or a watch-only indexer. Placing the chain before the purpose would require one extended public key per chain. The cost of non-hardened levels is described in Security Considerations.

**No change level.** BIP-44 separates external and internal addresses with a change level. Here the purpose already tells whether an address is given out, so such a level would be redundant.

**Counter rules.** Reserving before signing prevents a crash or a timeout from handing the same index out twice. An abandoned operation may still have exposed a signed EIP-7702 authorization, for instance to a bundler during gas estimation, so its index is never reused.

**Single-use protocol senders out of scope.** A sender derived from protocol data and used once has no reason to be found by a generic wallet. Keeping it out of `m/44'/60'` prevents standard wallets from discovering it and showing it next to the public accounts of the same seed.

**Fresh executor per privacy exit.** Two exits executed from the same address are linked on chain. A new address per exit keeps them apart, and the executor stays recoverable by any conforming wallet if funds remain on it after a failed call.

**Purpose separation.** Separate keys per purpose allow separate delegations per purpose, limit what a leaked key or a faulty delegate exposes, and keep a payment address given to third parties from revealing the DeFi activity of the same user.

**Gap limit of 20.** This is the value used by BIP-44, which wallets already implement.

## Backwards Compatibility

This ERC defines a new path and changes no existing behavior.

The path does not collide with the layouts commonly used by Ethereum wallets: `m/44'/60'/0'/0/x`, `m/44'/60'/x'/0/0` and `m/44'/60'/0'/x`. The node `m/44'/60'/7702'` coincides with account `7702` of the layout `m/44'/60'/x'/0/0`, whose wallets derive `m/44'/60'/7702'/0/0`. At depth 4 that layout uses the non-hardened index `0` while this ERC uses the hardened index `d'`, so the keys never coincide. An extended public key for `m/44'/60'/7702'` only yields non-hardened children and reveals nothing about this subtree.

The path has 8 levels. Implementations limited to fewer levels cannot support this ERC. Some hardware signers only accept a fixed set of five-level path patterns for Ethereum signing and need an update to support it.

Accounts already created at other paths keep working. Moving them to this layout requires transferring assets and is out of scope.

## Test Cases

All vectors use the BIP-39 mnemonic below with an empty passphrase. Addresses are given with their mixed-case checksum.

```
abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about
```

BIP-39 seed:

```
5eb00bbddcf069084889a8ab9155568165f5c453ccb85e70811aaed6f6da5fc19a5ac40b389cd370d086206dec8aa6c43daea6690f20ad3d8d48b2d2ce9e38e4
```

As a sanity check, this seed gives `0x9858EfFD232B4033E47d90003D41EC34EcaEda94` at `m/44'/60'/0'/0/0`.

| Path | Device request | Address |
|------|----------------|---------|
| `m/44'/60'/7702'/0'/0'/0'/0/0` | `0x0000000000000000000000000000000000000000` | `0x6EC9bA7642cAD607aD652B411Beb6E5517b63662` |
| `m/44'/60'/7702'/0'/0'/1'/1/0` | `0x0000000000000000000000010000000100000000` | `0x0328ee9BDf8cd56675ffC8D06bAE4A7C8b9d71Ab` |
| `m/44'/60'/7702'/0'/0'/2'/0/0` | `0x0000000000000000000000020000000000000000` | `0x1034775461aB34c2bFA976A45A47af0c69e64480` |
| `m/44'/60'/7702'/0'/0'/2'/0/1` | `0x0000000000000000000000020000000000000001` | `0x8A012eA387B64398401Ba82a39e20B2BDEBe5530` |
| `m/44'/60'/7702'/0'/0'/3'/42161/0` | `0x0000000000000000000000030000a4b100000000` | `0x706da964083B5Be72Bc81a82f5092580f54e5A39` |
| `m/44'/60'/7702'/0'/1'/0'/0/0` | `0x0000000000000001000000000000000000000000` | `0x4b16a0FeEc58C9812e272598dC5A34CB177C8270` |
| `m/44'/60'/7702'/1984'/0'/0'/1/0` | `0x000007c000000000000000000000000100000000` | `0xaB4c832f26dD0Ee7DD010a35CaA51be96125E070` |
| `m/44'/60'/7702'/1984'/0'/0'/42161/0` | `0x000007c000000000000000000000a4b100000000` | `0xcD3003DB7eCA9e134abB70a6A81170d707A0A459` |
| `m/44'/60'/7702'/29795'/0'/0'/1/0` | `0x0000746300000000000000000000000100000000` | `0xDD753817F6463ECDb2d6Ce0eA29EE124F81c95Ab` |
| `m/44'/60'/7702'/28784'/0'/0'/1/0` | `0x0000707000000000000000000000000100000000` | `0x8abcD7518fBDD446884ED5B2DBbb6F1e471726E1` |

| Path | Private key |
|------|-------------|
| `m/44'/60'/7702'/0'/0'/0'/0/0` | `0xd0cefbf2e0bee866bc37251ee59533718ca1a41913f4700e20b32b254e4c5f4c` |
| `m/44'/60'/7702'/0'/0'/1'/1/0` | `0x57b4a07cff3aed59718f56395145794a723663674504e070b6c40d2a715f6d1d` |
| `m/44'/60'/7702'/0'/0'/2'/0/0` | `0x1a1e6a3937bbfb1c83b3c9bc84aa82c6007948e1fa9e505a10d96315331dfc89` |
| `m/44'/60'/7702'/0'/0'/2'/0/1` | `0x478af5a10e64cdb412dc6d71696126a907f0db7ea3ffbe9b4829a7f228052e08` |
| `m/44'/60'/7702'/0'/0'/3'/42161/0` | `0xcd6f929176c340ad761c5c492e0f9cb8c951519924de5ac41b7abf7635c8be60` |
| `m/44'/60'/7702'/0'/1'/0'/0/0` | `0x635835544a5e006a1344d78429f59b7f1f926752d4d2ecee04b869925b24a560` |
| `m/44'/60'/7702'/1984'/0'/0'/1/0` | `0x526d38fa5f4e1712fc45f650e4369f44372aa0b1f7c7edc3f6eec494c558787d` |
| `m/44'/60'/7702'/1984'/0'/0'/42161/0` | `0xa1410634276c9f9ed19ae7e84f7d98f7de5b9e2908072a97cc69d64887976950` |
| `m/44'/60'/7702'/29795'/0'/0'/1/0` | `0xaadb1757ca84d4be055425b163cae70c23ee9c04a8c7ac44ab0101c1d25b150b` |
| `m/44'/60'/7702'/28784'/0'/0'/1/0` | `0x199681b3c559077f24276869c7e6344f2ff39887bfea5bebf15e72260e41fcc5` |

Watch-only derivation: the extended public key of `m/44'/60'/7702'/0'/0'/2'` is

```
xpub6JQbxokrqWHyyU8JquqkPYYCEbUnpw1ohFwm1t9hfSV44btR8SYFUW31kw1j1qygRJBEdjrVyPZX9EtQWXmGAetirchZnCN5BUhDWugptb7
```

and public derivation of `0/1` from it gives `0x8A012eA387B64398401Ba82a39e20B2BDEBe5530`, the address of `m/44'/60'/7702'/0'/0'/2'/0/1` above.

The following requests MUST be rejected: `0x8000000000000000000000000000000000000000` (`d >= 2^31`), `0x0000000000000000000000040000000000000000` (reserved purpose), `0x0000000000000000000000008000000000000000` (`k >= 2^31`), `0x0000000000000000000000000000000080000000` (`w >= 2^31`), and any request whose length is not 20 bytes.

A device holding the key of `m/44'/60'/7702'/1984'/0'/0'/1/0` MUST refuse an EIP-7702 authorization with chain ID `0` or `42161`, and MAY sign one with chain ID `1`.

## Reference Implementation

```python
HARDENED = 0x80000000
PREFIX = (44 | HARDENED, 60 | HARDENED, 7702 | HARDENED)
RESERVED_PURPOSES = range(4, 0x10000)

# Informative list of known protocol identifiers.
PROTOCOLS = {"railgun": 1984, "tornado-cash": 29795, "privacy-pools": 28784}


def path_indices(d: int, a: int, p: int, k: int, w: int) -> tuple:
    for v in (d, a, p, k, w):
        if not 0 <= v < HARDENED:
            raise ValueError("index out of range")
    if p in RESERVED_PURPOSES:
        raise ValueError("reserved purpose")
    return PREFIX + (d | HARDENED, a | HARDENED, p | HARDENED, k, w)


def path_string(d: int, a: int, p: int, k: int, w: int) -> str:
    path_indices(d, a, p, k, w)
    return f"m/44'/60'/7702'/{d}'/{a}'/{p}'/{k}/{w}"


def chain_level(chain_id: int) -> int:
    return chain_id if 0 < chain_id < HARDENED else 0


def may_sign(k: int, chain_id: int) -> bool:
    # Applies to transactions and EIP-7702 authorizations alike.
    return k == 0 or chain_id == k


def encode_request(d: int, a: int, p: int, k: int, w: int) -> bytes:
    path_indices(d, a, p, k, w)
    return b"".join(v.to_bytes(4, "big") for v in (d, a, p, k, w))


def decode_request(data: bytes) -> tuple:
    if len(data) != 20:
        raise ValueError("request must be 20 bytes")
    fields = (int.from_bytes(data[i:i + 4], "big") for i in range(0, 20, 4))
    return path_indices(*fields)
```

The address follows from any BIP-32 implementation, for example with `eth_account`:

```python
from eth_account import Account

Account.enable_unaudited_hdwallet_features()
path = path_string(PROTOCOLS["railgun"], 0, 0, chain_level(1), 0)
acct = Account.from_mnemonic(mnemonic, account_path=path)
```

## Security Considerations

**Non-hardened levels.** As stated in BIP-32, the extended public key of a node together with the private key of any of its non-hardened children reveals the private key of that node. With this layout, an extended public key of a `(d, a, p)` node at depth 6 plus one leaked leaf private key reveals every key of that `(d, a, p)` subtree, on every chain. The hardened levels `d'`, `a'` and `p'` stop the leak there. Wallets SHOULD NOT export the extended public key of a subtree whose leaf private keys are used outside the signing device, for instance gas keys held by a server. Such keys SHOULD live under a dedicated account `a`.

**Delegation authority.** The key at any path of this subtree can sign EIP-7702 authorizations and replace the delegate of its account. Whoever holds the key controls the account, whatever the delegate code enforces. Signing devices SHOULD display the chain ID, the delegate address and the nonce of an authorization tuple before signing it, and SHOULD warn when the chain ID is `0`, which makes the authorization valid on every chain. For keys derived with `k != 0`, the chain rule above lets the device refuse such authorizations outright.

**Application scoping.** A host that lets an application derive keys, such as a privacy protocol plugin executing calls after a withdrawal, SHOULD give it access to `m/44'/60'/7702'/d'` for the protocol it serves, or to `m/44'/60'/7702'/d'/a'/p'` for one purpose, and never to other protocols, other purposes or keys outside this ERC. Because `d'`, `a'` and `p'` are hardened, a compromised application then controls the addresses of that subtree and nothing else. Granting single paths of main accounts, such as `m/44'/60'/0'/0/i`, exposes those accounts to the application.

**Prefix isolation.** Locking a dedicated application to `m/44'/60'/7702'` or to `m/44'/60'/7702'/d'` constrains that application only. Any application allowed on `m/44'/60'`, such as a generic Ethereum application, can also derive and sign with keys of this subtree.

**Applications sharing a seed.** The protocol level prevents collisions between applications serving different protocols. Two applications serving the same protocol from the same seed still keep separate counters. The on-chain check before use reduces collisions between them, but it does not see an index the other application has reserved and not used yet.

**Separation and privacy.** Separate keys do not prevent addresses from being linked on chain. A transfer between two addresses of the same seed, such as funding a payment address from a gas address or bridging between the accounts of two chains, links them publicly. Wallets that offer purpose or chain separation for privacy SHOULD avoid direct transfers between the separated addresses.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
