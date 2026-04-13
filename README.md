# ERC-XXXX: Encrypted Token Standard

**STYX Protocol · Kairos Lab**

> A standard interface for fungible tokens with fully homomorphic encrypted (FHE) balances and zero-knowledge transfer verification.

---

## Overview

ERC-XXXX defines a minimal interface for tokens where balances are stored as FHE ciphertexts. Transfers are verified via zero-knowledge proofs without revealing amounts, sender balances, or recipient balances to any on-chain observer.

| Property | Value |
|---|---|
| Status | Draft |
| Type | Standards Track / ERC |
| Category | ERC |
| Created | 2026-04-12 |
| Author | Kairos Lab ([@Valisthea](https://github.com/Valisthea)) |
| Requires | ERC-165, ERC-20 |
| License | CC0-1.0 |

---

## The Problem

ERC-20 tokens expose all balances and transfer amounts on-chain. This enables:

- **Portfolio surveillance** — wallet contents visible to anyone
- **Transfer graph analysis** — full transaction history linkable
- **MEV extraction** — sandwich attacks, front-running on visible pending txns
- **Governance coercion** — whale tracking in DAO token voting
- **Regulatory conflict** — GDPR, MiCA, and financial privacy laws vs. public ledgers

Existing privacy solutions (Tornado Cash, Aztec Connect, privacy L2s) are external wrappers that break composability — a shielded token can't be used in Uniswap or Aave without unshielding first.

## The Solution

ERC-XXXX makes privacy **native to the token interface**. Key properties:

- Balances stored as FHE ciphertexts — only the owner can decrypt
- Transfers verified by ZK proof — no amount revealed on-chain
- `verifyBalancePredicate` — prove "balance ≥ threshold" without revealing the balance
- ERC-20 metadata (`name`, `symbol`, `decimals`, `totalSupply`) preserved for compatibility
- Optional `shield`/`unshield` bridge for dual public/private pool tokens

---

## Repository Structure

```
styx-erc-encrypted-token/
├── eip/
│   └── eip-xxxx.md                    # Official EIP draft (markdown)
├── contracts/
│   └── interfaces/
│       └── IERCXXXX.sol               # Solidity interface (+ OMEGA audit fixes)
├── docs/
│   └── ERC-XXXX-Encrypted-Token-Standard.html   # Visual spec
├── test/
│   └── .gitkeep                       # Tests pending reference implementation
├── .github/
│   └── ISSUE_TEMPLATE/
│       └── security-vulnerability.md
├── LICENSE                            # CC0-1.0
└── SECURITY.md
```

---

## Interface at a Glance

```solidity
interface IERCXXXX {
    // ERC-20 compatible metadata
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function decimals() external view returns (uint8);
    function totalSupply() external view returns (uint256);

    // Encrypted balance queries
    function encryptedBalanceOf(address account) external view returns (bytes memory);
    function encryptionScheme() external view returns (string memory);
    function publicKey() external view returns (bytes memory);
    function keyVersion() external view returns (uint256);

    // Blind transfers (ZK-verified)
    function blindTransfer(address to, bytes calldata encryptedAmount, bytes calldata proof) external returns (bool);
    function blindTransferFrom(address from, address to, bytes calldata encryptedAmount, bytes calldata proof) external returns (bool);

    // Encrypted approvals
    function blindApprove(address spender, bytes calldata encryptedAmount, bytes calldata proof) external returns (bool);
    function encryptedAllowance(address owner, address spender) external view returns (bytes memory);

    // Selective disclosure
    function verifyBalancePredicate(address account, bytes calldata predicate, bytes calldata proof) external view returns (bool);

    // Optional: mint / burn
    function blindMint(address to, bytes calldata encryptedAmount, bytes calldata proof) external;
    function blindBurn(bytes calldata encryptedAmount, bytes calldata proof) external;
}
```

**Optional extension interfaces** (in `IERCXXXX.sol`):
- `IERCXXXX_Shielded` — shield/unshield bridge between public and encrypted pools
- `IERCXXXX_ConfidentialSupply` — for tokens that also hide total supply
- `IERCXXXX_Fees` — FHE computation fee interface
- `IERCXXXX_Batch` — aggregated batch blind transfers

---

## Key Design Decisions

**Why FHE over commitments?**
Pedersen/Poseidon commitments hide values but don't support computation. FHE ciphertexts support homomorphic addition and comparison natively — the contract can update balances without any party ever seeing plaintext.

**Why `verifyBalancePredicate` instead of `balanceOf`?**
DeFi protocols need "does this user have enough collateral?" — not the exact amount. Predicate proofs give protocols exactly what they need, and nothing more.

**Why events without amounts?**
`Transfer(from, to, amount)` leaks the amount. `BlindTransfer(from, to, proofHash)` preserves indexability (who → whom) without revealing how much. Off-chain auditors with appropriate keys can still verify via `proofHash`.

---

## Audit Status

| Auditor | Type | Findings | Status |
|---|---|---|---|
| OMEGA V4 (Kairos Lab) | Spec Review | 4C · 3H · 5M · 4L | ✅ All fixes applied |
| External (TBD) | Full Audit | — | ⏳ Pending |

OMEGA V4 critical fixes applied in `IERCXXXX.sol`:
- **M-01** — Custom errors replacing generic reverts
- **L-01** — `keyVersion` added to `BlindTransfer` event
- **C-04** — Key rotation interface (`keyVersion`, `isKeyActive`, `KeyRotated` event)
- **H-01** — `blindApprove` proof spec clarified (no balance check at approval time)
- **M-04** — `verifyBalancePredicate` prover-binding clarification
- **C-02** — `IERCXXXX_ConfidentialSupply` extension for hidden total supply
- **C-01** — `IERCXXXX_Fees` extension for FHE computation costs
- **L-03** — `IERCXXXX_Shielded` as separate extension interface
- **M-05** — `IERCXXXX_Batch` for aggregated batch proofs

---

## Reference Implementation

Reference implementation: **STYX Protocol**

- `StyxEncryptedToken.sol` — Full implementation with TFHE backend
- `StyxVerifier.sol` — On-chain ZK proof verifier (Halo2)
- `StyxReencryptor.sol` — Re-encryption proxy for balance queries

Repository: `https://github.com/Valisthea/styx-protocol` (pending publication)

---

## EIP Submission

This standard will be submitted to the Ethereum EIPs repository as a formal ERC proposal. The ERC number (XXXX) will be assigned by EIP editors upon PR submission to `github.com/ethereum/ercs`.

---

## License

Copyright and related rights waived via [CC0 1.0 Universal](./LICENSE).
