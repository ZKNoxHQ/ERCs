---
title: Derivation Path for EIP-7702 Accounts
description: One BIP-44 rooted HD path for EIP-7702 delegated EOAs, split by account, purpose, change and address index.
author: Renaud Dubois (@rdubois-crypto), Paul (@0xpaulio), zy0n (@zy0n), Simon Masson (@simonmasson)
discussions-to: <ethereum-magicians thread URL>
status: Draft
type: Standards Track
category: ERC
created: 2026-11-26
requires: 7702
---

## Abstract

This ERC defines a single hierarchical deterministic derivation path for the secp256k1 keys of externally owned accounts (EOAs) that delegate their code with [EIP-7702](./eip-7702.md):

```
m/44'/60'/7702'/a'/p'/c/w
```

The first three levels are fixed. The last four select an account `a`, a purpose `p`, a change branch `c` and an address index `w`. The ERC also defines a registry of purpose values, an account discovery procedure, and the 16-byte request a signing device receives to derive a key in this subtree.

## Motivation

EIP-7702 lets an EOA install a delegation designator so that the account executes the code of a delegate contract. The account keeps its address and its private key. Wallets that create such accounts currently pick their own derivation paths, so a seed restored in another wallet shows different addresses and the funds appear to be missing.

Users also spread their activity over several addresses with different risk and privacy profiles: one address that executes DeFi operations completing in a single transaction, one that holds positions over time, deposit addresses given to third parties, one that pays gas. Each wallet invents its own layout for this.

Hardware signers bind an application to a set of derivation path prefixes. A layout whose fixed part comes first allows a dedicated application to be restricted to its subtree.

This ERC addresses these points with the following requirements:

1. There is exactly one path, so that importing a seed or migrating between wallets yields the same accounts.
2. The path is rooted at `m/44'/60'`, the Ethereum coin type, so that existing Ethereum tooling can derive the keys.
3. All variable levels come after all fixed levels, so that hardware signers can lock an application to the prefix `m/44'/60'/7702'`.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Path

Keys are derived with [BIP-32](https://github.com/bitcoin/bips/blob/9f3d3bc6a60db16efbdc71ac8f7ea30217fc0c39/bip-0032.mediawiki) over secp256k1, from a master key typically obtained from a [BIP-39](https://github.com/bitcoin/bips/blob/9f3d3bc6a60db16efbdc71ac8f7ea30217fc0c39/bip-0039.mediawiki) seed, along the path:

```
m / 44' / 60' / 7702' / a' / p' / c / w
```

| Level | Name | Hardened | Value |
|-------|------|----------|-------|
| 1 | BIP-43 purpose | yes | `44'`, as in [BIP-44](https://github.com/bitcoin/bips/blob/9f3d3bc6a60db16efbdc71ac8f7ea30217fc0c39/bip-0044.mediawiki) |
| 2 | Coin type | yes | `60'`, Ether in SLIP-0044 |
| 3 | Account type | yes | `7702'` |
| 4 | Account `a` | yes | `0 <= a < 2^31` |
| 5 | Purpose `p` | yes | a value from the purpose registry, `0 <= p < 2^31` |
| 6 | Change `c` | no | `0` or `1` |
| 7 | Address index `w` | no | `0 <= w < 2^31` |

The Ethereum address of the account is the address of the public key at the leaf of this path. A wallet conforming to this ERC MUST derive the keys of the persistent EIP-7702 accounts it creates at this path and MUST NOT use any other path for them.

Single-use senders that a protocol derives in its own subtree are out of scope, for example a sender derived from a privacy pool note and used for a single withdrawal.

Level 3 holds the number of the EIP that defines the account type, hardened. This ERC only defines the value `7702'`.

### Purpose registry

| `p` | Name | Use |
|-----|------|-----|
| 0 | Atomic DeFi | Addresses executing DeFi operations whose effects complete within one transaction, such as a batched approval and swap, or an exit from a privacy protocol followed by calls in the same transaction. Funds are not expected to remain on the address between transactions. |
| 1 | Non-atomic DeFi | Addresses holding DeFi positions across transactions, such as lending, liquidity provision, staking, or orders settled later. |
| 2 | Payment | Addresses given to a person or a service to receive funds, such as a deposit address whose incoming funds are then forwarded or shielded. |
| 3 | Gas | Addresses paying transaction fees, or funding other addresses of the same seed for transaction fees. |

An address executing calls after an exit from a privacy protocol SHOULD be derived on the internal branch `c = 1`. Wallets SHOULD use a new address index `w` for each such exit and MUST NOT give these addresses out to receive funds.

Values from `4` to `65535` are reserved for future ERCs. Wallets MUST NOT derive keys at a reserved value. Values from `65536` to `2^31 - 1` are for private use; their meaning is wallet specific and not interoperable.

### Change

`c = 0` designates external addresses, shown to the user or given to third parties. `c = 1` designates internal addresses used by the wallet itself, such as one-time intermediate or refund addresses. Other values MUST NOT be used.

### Address index

`w` starts at `0` and is incremented sequentially within each `(a, p, c)` branch.

### Account discovery

A wallet restoring a seed SHOULD discover accounts as follows:

1. Set `a = 0`.
2. For each registered purpose `p` and each `c` in `{0, 1}`, derive addresses for `w = 0, 1, 2, ...` and stop after 20 consecutive unused addresses.
3. If at least one address of account `a` is used, increment `a` and go to step 2. Otherwise stop.

An address is used if, on any chain the wallet supports, it has a nonzero nonce, a nonzero balance, nonempty code (including a delegation designator `0xef0100 || address`), or any other activity the wallet tracks.

### Signing device request

A signing device that restricts an application to the prefix `m/44'/60'/7702'` SHOULD receive the variable levels as a 16-byte request:

```
a || p || c || w
```

where each field is a 32-bit unsigned big-endian integer without the hardened bit. The device:

- MUST build the full path itself from the fixed prefix and the four fields, and MUST NOT accept a full path from the host for this subtree;
- MUST reject the request if `a`, `p` or `w` is greater than or equal to `2^31`, or if `c` is neither `0` nor `1`;
- MUST set the hardened bit on `a` and `p`, and MUST NOT set it on `c` and `w`;
- SHOULD display the name of the purpose when `p` is a registered value and the device shows the address.

## Rationale

**Single path.** Any conforming wallet restoring the same seed finds the same accounts, without asking the user to choose among candidate paths.

**Root at `m/44'/60'`.** Keys in this subtree are ordinary Ethereum keys. Wallets and hardware applications already allowed on `m/44'/60'` can derive them, which makes import and migration possible without a dedicated application.

**Fixed levels first.** Hardware signers declare the paths an application may use as prefixes and do not support a wildcard in the middle of a path. An earlier version of this layout, `m/44'/60'/a'/7702'/p'/c/w`, could not be locked to a subtree. Placing the account type at level 3 makes `m/44'/60'/7702'` a usable prefix.

**Account type `7702'` at level 3.** As in [BIP-43](https://github.com/bitcoin/bips/blob/9f3d3bc6a60db16efbdc71ac8f7ea30217fc0c39/bip-0043.mediawiki), the index names the specification the subtree follows. Using the number of EIP-7702, instead of the number of this ERC, keeps the value known before number assignment and tied to the account type.

**Account before purpose.** An account groups every purpose of one user profile, in the same way as a BIP-44 account. Purposes are subdivisions of an account.

**Hardened `a'` and `p'`.** Public data or a leaked private key from one `(a, p)` subtree gives no information about any other subtree.

**Non-hardened `c` and `w`.** The extended public key at depth 5 lets a service derive every address of one purpose without holding private keys, for instance a payment backend generating deposit addresses or a watch-only indexer. The cost of this choice is described in Security Considerations.

**Single-use protocol senders out of scope.** A sender derived from protocol data and used once has no reason to be found by a generic wallet. Keeping it out of `m/44'/60'` prevents standard wallets from discovering it and showing it next to the public accounts of the same seed.

**Fresh executor per privacy exit.** Two exits executed from the same address are linked on chain. A new internal address per exit keeps them apart, and the executor stays recoverable by any conforming wallet if funds remain on it after a failed call.

**Purpose separation.** Separate keys per purpose allow separate delegations per purpose, limit what a leaked key or a faulty delegate exposes, and keep a payment address given to third parties from revealing the DeFi activity of the same user.

**Gap limit of 20.** This is the value used by BIP-44, which wallets already implement.

## Backwards Compatibility

This ERC defines a new path and changes no existing behavior.

The path does not collide with the layouts commonly used by Ethereum wallets: `m/44'/60'/0'/0/x`, `m/44'/60'/x'/0/0` and `m/44'/60'/0'/x`. The node `m/44'/60'/7702'` coincides with account `7702` of the layout `m/44'/60'/x'/0/0`, whose wallets derive `m/44'/60'/7702'/0/0`. At depth 4 that layout uses the non-hardened index `0` while this ERC uses the hardened index `a'`, so the keys never coincide. An extended public key for `m/44'/60'/7702'` only yields non-hardened children and reveals nothing about this subtree.

The path has 7 levels. Implementations limited to fewer levels cannot support this ERC.

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
| `m/44'/60'/7702'/0'/0'/0/0` | `0x00000000000000000000000000000000` | `0x879d39207ddFE75DB9fD2E3a3d9c69507CD1483C` |
| `m/44'/60'/7702'/0'/0'/1/0` | `0x00000000000000000000000100000000` | `0x0C3a3e13A3841F958cEb4F2Ddd6A5B74f5a5B6D9` |
| `m/44'/60'/7702'/0'/1'/0/0` | `0x00000000000000010000000000000000` | `0x624C60E5240bfc6C34AB02E4F3A661ded373103A` |
| `m/44'/60'/7702'/0'/2'/0/0` | `0x00000000000000020000000000000000` | `0x47107e99539ea596d8cd74FA72De58a04e0cBE08` |
| `m/44'/60'/7702'/0'/2'/0/1` | `0x00000000000000020000000000000001` | `0x80bEf5e743d69af5610cc1113F0F863E23B936F4` |
| `m/44'/60'/7702'/0'/2'/1/0` | `0x00000000000000020000000100000000` | `0x0615693fD4F2F04C7fb1D0dbfd8Ea1352A692268` |
| `m/44'/60'/7702'/0'/3'/0/0` | `0x00000000000000030000000000000000` | `0xb45581246C79E4637D7E35bc395fEE1C12b45412` |
| `m/44'/60'/7702'/1'/0'/0/0` | `0x00000001000000000000000000000000` | `0xbFE052DB2921F12fbEAa45CC753Dc9c23080b21D` |

| Path | Private key |
|------|-------------|
| `m/44'/60'/7702'/0'/0'/0/0` | `0x2a3731a4aee3f6958f748f72ebbfd56360f8f21fa7805f9c8bc010160888812a` |
| `m/44'/60'/7702'/0'/0'/1/0` | `0x92c2599cdbaa5bd83ccc2f9a059b4b64dd0cc3e9d3b1525534d0a2a1c074db27` |
| `m/44'/60'/7702'/0'/1'/0/0` | `0x587277ac03ac36a761e4ee4634906c54360704af9a308b0a7677d9652962af4d` |
| `m/44'/60'/7702'/0'/2'/0/0` | `0x4277c8237f43ebbf98ec62a34806cd968e13cf6b26135b98b633d9757ca3bcaf` |
| `m/44'/60'/7702'/0'/2'/0/1` | `0x03191d208bd906d063504f849e9e5451590a879f86c29cf4c337a7104078cd72` |
| `m/44'/60'/7702'/0'/2'/1/0` | `0xb65789e9f86b7bece0ee7db693e5336b8fdd42451b6136aa1909f7f84cd3620a` |
| `m/44'/60'/7702'/0'/3'/0/0` | `0x14e60cf61b4334de107944ab074e78f8df8f6d2b8685aad7083532cab4df894d` |
| `m/44'/60'/7702'/1'/0'/0/0` | `0xf11493dc8bc1e3659bf02781b8777df8daca01f15d4c5b51cd0aaf9bece74589` |

Watch-only derivation: the extended public key of `m/44'/60'/7702'/0'/2'` is

```
xpub6G13yVeYmCCXrACjNky8iWRVQV7gpUBpmxZwGdRHFQWDVzJUAHL9ETfKkaKupT2cw899sxQYwvTRw3xq73YstrNFdACVnNvU9xXzqnY1VR4
```

and public derivation of `0/1` from it gives `0x80bEf5e743d69af5610cc1113F0F863E23B936F4`, the address of `m/44'/60'/7702'/0'/2'/0/1` above.

The following requests MUST be rejected: `0x80000000000000000000000000000000` (`a >= 2^31`), `0x00000000000000040000000000000000` (reserved purpose), `0x00000000000000000000000200000000` (`c = 2`), `0x00000000000000000000000080000000` (`w >= 2^31`).

## Reference Implementation

```python
HARDENED = 0x80000000
PREFIX = (44 | HARDENED, 60 | HARDENED, 7702 | HARDENED)
RESERVED_PURPOSES = range(4, 0x10000)


def path_indices(a: int, p: int, c: int, w: int) -> tuple:
    for v in (a, p, w):
        if not 0 <= v < HARDENED:
            raise ValueError("index out of range")
    if c not in (0, 1):
        raise ValueError("change must be 0 or 1")
    if p in RESERVED_PURPOSES:
        raise ValueError("reserved purpose")
    return PREFIX + (a | HARDENED, p | HARDENED, c, w)


def path_string(a: int, p: int, c: int, w: int) -> str:
    path_indices(a, p, c, w)
    return f"m/44'/60'/7702'/{a}'/{p}'/{c}/{w}"


def encode_request(a: int, p: int, c: int, w: int) -> bytes:
    path_indices(a, p, c, w)
    return b"".join(v.to_bytes(4, "big") for v in (a, p, c, w))


def decode_request(data: bytes) -> tuple:
    if len(data) != 16:
        raise ValueError("request must be 16 bytes")
    a, p, c, w = (int.from_bytes(data[i:i + 4], "big") for i in range(0, 16, 4))
    return path_indices(a, p, c, w)
```

The address follows from any BIP-32 implementation, for example with `eth_account`:

```python
from eth_account import Account

Account.enable_unaudited_hdwallet_features()
acct = Account.from_mnemonic(mnemonic, account_path=path_string(0, 2, 0, 1))
```

## Security Considerations

**Non-hardened levels.** As stated in BIP-32, the extended public key of a node together with the private key of any of its non-hardened children reveals the private key of that node. With this layout, an extended public key of an `(a, p)` node at depth 5 plus one leaked leaf private key reveals every key of that `(a, p)` subtree. The hardened levels `a'` and `p'` stop the leak there. Wallets SHOULD NOT export the extended public key of a subtree whose leaf private keys are used outside the signing device, for instance gas keys held by a server. Such keys SHOULD live under a dedicated account `a`.

**Delegation authority.** The key at any path of this subtree can sign EIP-7702 authorizations and replace the delegate of its account. Whoever holds the key controls the account, whatever the delegate code enforces. Signing devices SHOULD display the chain ID, the delegate address and the nonce of an authorization tuple before signing it, and SHOULD warn when the chain ID is `0`, which makes the authorization valid on every chain.

**Prefix isolation.** Locking a dedicated application to `m/44'/60'/7702'` constrains that application only. Any application allowed on `m/44'/60'`, such as a generic Ethereum application, can also derive and sign with keys of this subtree.

**Application scoping.** A host that lets an application derive keys, such as a privacy protocol plugin executing calls after a withdrawal, SHOULD give it access to the subtree `m/44'/60'/7702'/a'/p'` of the purpose it serves only, and never to other purposes or to keys outside this ERC. Because `a'` and `p'` are hardened, a compromised application then controls the addresses of that purpose and nothing else. Granting single paths of main accounts, such as `m/44'/60'/0'/0/i`, exposes those accounts to the application.

**Purpose separation and privacy.** Separate keys do not prevent addresses from being linked on chain. A transfer between two addresses of the same seed, such as funding a payment address from a gas address, links them publicly. Wallets that offer purpose separation for privacy SHOULD avoid direct transfers between purposes.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
