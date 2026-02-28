# Solana Hot Wallet Private Key Extraction via Ed25519 Shared-R Vulnerability

> [Blog 원문](https://velog.io/@sane100400/Solana-Hot-Wallet-Private-Key-Extraction-via-Ed25519-Shared-R-Vulnerability)

**Tags:** ED25519, Nonce reuse, Shared-R, blockchain, solana

---

## 1. Incident Overview

In 2025, several hundred million USD in cryptocurrency was drained from Exchange A's Solana hot wallets, affecting multiple Solana-based tokens valued at tens of millions of dollars. The incident suggested potential private key exposure.

Research by Jacquot and Donnet (appearing in Financial Cryptography and Data Security 2026) documented that nonce reuse across six blockchains including Bitcoin and Ethereum exposed 3,620 private keys, endangering approximately €101M in assets.

---

## 2. Background

### 2.1 Elliptic Curve Digital Signatures

Blockchains use elliptic curve cryptography (ECC) for digital signatures proving legitimate wallet ownership. The fundamental principle involves a fixed base point G on an elliptic curve; multiplying G by secret scalar a produces public key A:

**A = a · G**

Computing A from a is efficient, but recovering a from A alone remains computationally infeasible. This one-way property underpins digital signature security.

Digital signature objectives require proving knowledge of secret key a without revealing it. This requires generating a fresh ephemeral nonce r for each signature, incorporated into the signing equation alongside a.

Since nonce r introduces a second unknown, a single signature equation with two unknowns (a and r) prevents recovery of a.

#### Signing Process

1. Generate an ephemeral nonce r
2. Compute R = r · G (elliptic curve point corresponding to the nonce)
3. Compute S using message M hash, secret key a, and nonce r: S = r + H(R, A, M) · a
4. Publish (R, S) pair as the signature

#### Verification Process

Since S = r + H(R, A, M) · a during signing, multiplying both sides by G yields:

**S · G = r · G + H(R, A, M) · a · G = R + H(R, A, M) · A**

Verifiers check whether S · G equals R + H(R, A, M) · A.

The right-hand side can be computed using only signature component R, public key A, and message M. Anyone can verify signature validity without learning a.

#### ECDSA (Bitcoin, Ethereum)

```
r ← external random number generator (e.g., OS /dev/urandom)
R = r * G
S = (H(M) + R.x * a) / r (mod n)
```

The nonce r is drawn externally for each signature, ensuring different nonces each time. The disadvantage is that random number generator failure allowing nonce reuse enables secret key recovery from differences between two S values.

#### Ed25519 (Solana, SSH, TLS)

```
nonce_prefix ← lower 32 bytes of SHA-512(seed)
r = SHA-512(nonce_prefix || M)
R = r * G
S = r + H(R, A, M) * a (mod L)
```

Ed25519 concatenates a secret-key-derived nonce_prefix with message M and hashes the result to produce the nonce. The nonce_prefix is a fixed value derived once from the secret key; different messages yield different hash outputs, producing distinct nonces. This eliminates external random number generator dependence but requires that **nonce_prefix be unique to each signer**. If two signers share the same nonce_prefix, signing the same message produces identical r values.

### 2.2 Ed25519 Secret Key Structure

In Ed25519, a single secret key (32-byte seed) is hashed with SHA-512 to produce 64 bytes, split into two halves:

```
SHA-512(seed) → 64 bytes

First 32 bytes → scalar (a): the actual secret scalar used in signature computation
Last 32 bytes → nonce prefix: the seed value used to derive per-signature nonce
```

The scalar a is the secret number used directly in signature mathematical operations, while the nonce prefix fulfills the role that the operating system's random number generator plays in ECDSA. **Deriving the nonce from the secret key itself** is Ed25519's central design principle.

### 2.3 How a Signature (R, S) Is Produced

When Ed25519 signs a message M (such as transaction data), it produces a pair (R, S):

```
r = SHA-512(nonce_prefix || M)  ... nonce generation
R = r * G                        ... elliptic curve point corresponding to nonce
S = r + H(R, A, M) * a          ... computation involving secret key a
```

| Symbol | Description | Public? |
|--------|-------------|---------|
| r | Nonce; hash of nonce_prefix concatenated with message M | No (intermediate) |
| R | Elliptic curve point corresponding to r | **Yes** (in signature) |
| a | Secret key scalar | No (protected) |
| S | Computation result involving secret key a | **Yes** (in signature) |
| H(R, A, M) | Hash of R, public key A, and message M | Computable by anyone |
| G | Fixed base point known to all participants | Yes (constant) |

The critical property: **if nonce_prefix differs across signers, then r differs even for the same message, and consequently R (= r · G) also differs**. Nonce_prefix uniqueness per signer upholds Ed25519 security.

> In Solana, the signature is published as a 64-byte value consisting of R (32 bytes) concatenated with S (32 bytes).

### 2.4 Why Shared-R Is Dangerous

What happens if two signers possess the **same nonce_prefix**?

Consider two signers signing the same transaction (same message M):

```
signer A: r_A = SHA-512(prefix_A || M)    R_A = r_A * G
signer B: r_B = SHA-512(prefix_B || M)    R_B = r_B * G
```

If nonce_prefix differs for each signer, then r_A ≠ r_B and consequently R_A ≠ R_B.

However, if prefix_A = prefix_B, then for the same message M we get r_A = r_B, and therefore R_A = R_B.

This phenomenon, where two distinct signers produce the same R, is called **Shared-R**. Since both R and S are publicly recorded on the blockchain, Shared-R gives rise to a system of equations enabling secret key recovery from differences between the two S values.

### 2.5 Root Cause of Shared-R: Key Derivation Implementation Differences

Solana wallets typically derive multiple child keys from a single master seed using **HD (Hierarchical Deterministic) key derivation**, allowing all child keys restoration from a single seed backup. Solana uses the SLIP-0010 standard for this process.

The SLIP-0010 specification defines Ed25519 child key derivation as:

> "the returned child key k_i is I_L"
> : SLIP-0010, Private parent key → private child key

Here k_i is the derived child key and I_L is the left 32 bytes of the 64-byte SHA-512 output. The specification only defines how to produce the 32-byte child key.

However, Ed25519 signing requires both a scalar (32 bytes) and nonce_prefix (32 bytes), totaling 64 bytes. The 32-byte child key must be passed through SHA-512 again to produce 64 bytes. This step is called **re-expansion**.

In a correct implementation, re-expansion is performed so each child obtains a **unique nonce_prefix**:

```
Correct: child_key → SHA-512(child_key) → [new scalar | new prefix] (unique per child)
```

If, however, re-expansion is omitted and the nonce prefix derived from the master seed is reused for all children:

```
Incorrect: master_seed → SHA-512 → [master_scalar | master_prefix]
           child_0: [child_scalar_0 | master_prefix]    ← same prefix
           child_1: [child_scalar_1 | master_prefix]    ← same prefix
```

**All children share the same prefix**, and signing the same message produces identical R values.

Since re-expansion is not an explicitly mandated specification step, custom implementations optimizing or simplifying key derivation may omit it, introducing the Shared-R vulnerability.

### 2.6 Related Vulnerabilities

| Source | Description |
|--------|-------------|
| CVE-2022-50237 / RUSTSEC-2022-0093 | ed25519-dalek library flaw allowing scalar/nonce_prefix separation, enabling extraction from two signatures |
| orlp/ed25519 Issue #3 | ed25519_add_scalar function failure to update nonce prefix, causing R reuse |
| MystenLabs/ed25519-unsafe-libs | Catalog of 40+ Ed25519 libraries affected by the same vulnerability class |

---

## 3. Private Key Extraction Method

Given two transactions exhibiting Shared-R, private keys can be extracted through this procedure.

### 3.1 Signature Equations

Applying the signature formula S = r + H(R, A, M) · a to two signers (signer 0, signer 1) across two transactions (TX1, TX2):

```
TX1: s_01 = r_1 + h_01 · a_0 (mod L)  ...(1)
     s_11 = r_1 + h_11 · a_1 (mod L)  ...(2)
TX2: s_02 = r_2 + h_02 · a_0 (mod L)  ...(3)
     s_12 = r_2 + h_12 · a_1 (mod L)  ...(4)
```

- s_ij: the S component of the signature (publicly available on-chain)
- r_j: nonce (identical for both signers within same TX due to Shared-R)
- h_ij = SHA-512(R || A_i || M_j) mod L: challenge hash (computable from on-chain data)
- a_i: secret key scalar (extraction target)
- L: the order of the Ed25519 group (fixed constant)

### 3.2 Nonce Elimination

Subtracting (1)-(2) and (3)-(4) eliminates the nonce r:

```
d_1 = s_01 - s_11 = h_01 · a_0 - h_11 · a_1 (mod L)
d_2 = s_02 - s_12 = h_02 · a_0 - h_12 · a_1 (mod L)
```

With two unknowns (a_0, a_1) and two equations, the system is solvable.

### 3.3 Application of Cramer's Rule

Expressed in matrix form:

```
| h_01  -h_11 | | a_0 |   | d_1 |
| h_02  -h_12 | | a_1 | = | d_2 |

det = h_01 · (-h_12) - (-h_11) · h_02 (mod L)

a_0 = [d_1 · (-h_12) - (-h_11) · d_2] / det (mod L)
a_1 = [h_01 · d_2 - d_1 · h_02] / det (mod L)
```

**All input values (s, h, d) can be computed from publicly available on-chain data.**

The secret key scalars a_0 and a_1 are therefore fully determined.

### 3.4 Verification

The extracted scalars are verified by recomputing the public keys and comparing them against the originals:

**a · G = A (Ed25519 base point multiplication)**

A match confirms successful private key extraction.

---

## 4. Devnet Reproduction

To confirm practical exploitability, the same conditions were reproduced on the Solana devnet.

### 4.1 Reproduction Procedure

1. Derive two child keys from a random master seed using SLIP-0010
2. Derive child scalars normally, but reuse the master's nonce prefix for both children (reproducing vulnerable implementation)
3. Have both signers sign the same transaction message and confirm R is identical
4. Submit both transactions to the devnet
5. Apply Cramer's Rule using only publicly available on-chain data to extract private keys
6. Recompute public keys from extracted private keys and verify agreement with originals

### 4.2 Execution Output

```
========================================================================
  Shared-R PoC — Ed25519 nonce reuse vulnerability reproduction (DEVNET)
========================================================================

[Phase 1] Vulnerable keypair generation (SLIP-0010 + master prefix reuse)
  Signer 0: GxKGiE5sGmgAs3QrLFEkrmwYtMV8crxqq5Zo2ZaqAe4J
  Signer 1: EXrP8x4UWSvWZJ9GjhumL21bodu2qz69D6ZC6nr34bQ1
  Shared prefix (hex): 51338c9a28c92c134555dbfc913a634a...
  Key generation verification: OK

[Phase 3] Transaction 1 construction and submission
  Sig 0 R: affb089cfd802e72b1d4d04373a8cefd...
  Sig 1 R: affb089cfd802e72b1d4d04373a8cefd...
  Shared-R: YES
  TX1 submitted

[Phase 4] Transaction 2 construction and submission
  Sig 0 R: bee13c44cbb4b9f044cc990080ce375b...
  Sig 1 R: bee13c44cbb4b9f044cc990080ce375b...
  Shared-R: YES
  TX2 submitted

[Phase 5] On-chain data retrieval
  TX1 Shared-R on-chain confirmation: YES
  TX2 Shared-R on-chain confirmation: YES

[Phase 6] Private key extraction (Cramer's rule — using on-chain data only)
  Input: Signatures (R, S), messages, and public keys from TX1/TX2
  Extracted Scalar 0 (hex LE): a0cf256a02afea8cce5ad165f20502ab...
  Extracted Scalar 1 (hex LE): c0003bf2194765209bf14844564c25b1...
```

### 4.3 Verification Summary

| Verification Item | Result |
|-------------------|--------|
| Signer 0's R matches Signer 1's R within TX1 (Shared-R)? | YES |
| Signer 0's R matches Signer 1's R within TX2 (Shared-R)? | YES |
| Signer 0's R in TX1 matches Signer 0's R in TX2? | NO (different messages) |
| Extracted scalar * G equals Signer 0's public key? | YES |
| Extracted scalar * G equals Signer 1's public key? | YES |
| Same r computed for both signers when recovering TX1 nonce? | YES |
| Same r computed for both signers when recovering TX2 nonce? | YES |
| Extracted scalar matches original scalar in direct comparison? | YES |

### 4.4 On-Chain Evidence

The Shared-R phenomenon can be verified directly in the following Solana devnet transactions.

**Signer Addresses:**

| Role | Address |
|------|---------|
| Signer 0 | GxKGiE5sGmgAs3QrLFEkrmwYtMV8crxqq5Zo2ZaqAe4J |
| Signer 1 | EXrP8x4UWSvWZJ9GjhumL21bodu2qz69D6ZC6nr34bQ1 |

**Transactions:**
- TX1: https://solscan.io/tx/4X4x9xfAF63MwdbxeLrGHUawL5J6BSTMGvz48TqHsykB3k1UWvCAcJi9v4tZEJa2d6vb5KRZmEvBet9WtwHNXLnn?cluster=devnet
- TX2: https://solscan.io/tx/4pM2kVVYjLNko3EC5p5gaPQ2xN4fCoqEgigsvYLLCj7Ceb5ijbCaaEt6tnvfHc6UxMviz1QnbTmU56f7s5N5Vo8s?cluster=devnet

Decoding the raw bytes of the signatures reveals the R value in the first 32 bytes. Inspecting the raw signature bytes of the first transaction on Solscan confirms that the leading portions of both signatures (the base58-encoded R values) are identical.

Both private keys were extracted using only the publicly available data from these two transactions.

---

## 5. References

| Resource | Link |
|----------|------|
| RFC 8032 (Ed25519 specification) | https://datatracker.ietf.org/doc/html/rfc8032#section-5.1 |
| SLIP-0010 (HD key derivation standard) | https://github.com/satoshilabs/slips/blob/master/slip-0010.md |
| CVE-2022-50237 (NVD) | https://nvd.nist.gov/vuln/detail/CVE-2022-50237 |
| RUSTSEC-2022-0093 (Rust security advisory) | https://rustsec.org/advisories/RUSTSEC-2022-0093 |
| ed25519-unsafe-libs (vulnerable library catalog) | https://github.com/MystenLabs/ed25519-unsafe-libs |
| orlp/ed25519 Issue #3 | https://github.com/orlp/ed25519/issues/3 |
| Double Public Key Signing Attack paper | https://arxiv.org/abs/2308.1500 |
| Jacquot and Donnet, FC 2026 | https://fc26.ifca.ai/preproceedings/55.pdf |

---

> **Disclaimer:** This article is not intended to identify or attribute cause to any specific exchange or incident. It provides a technical explanation of a nonce-reuse vulnerability class that can arise in publicly available cryptographic signature structures, and summarizes implications from defensive and verification perspectives.
