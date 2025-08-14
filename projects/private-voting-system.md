# Private voting system

Example technologies that could be used include:
FHE, Miden, and Enclave Encrypted Execution Environments (E3)(https://docs.enclave.gg/whitepaper)

Reference [DaVinci](https://hackmd.io/@vocdoni/BJY8EXQy1x?ref=blog.shutter.network)

Great question. Each of these technologies—**Fully Homomorphic Encryption (FHE)**, **Miden (ZK-STARKs on edge devices)**, and **Enclave (Encrypted Execution Environments)**—can enable **privacy-preserving voting**, but they do so with very different **architectures, trust models, and trade-offs**.

---

## 🗳️ Summary Table: Privacy-Preserving Voting

| Feature | **FHE** | **Miden (ZK-STARKs)** | **Enclave (E3)** |
| --- | --- | --- | --- |
| **How it works** | Votes are encrypted and computed on without decrypting | Votes cast locally, proofs generated client-side | Votes cast into a secure enclave that processes in isolation |
| **Privacy Model** | Data stays encrypted throughout computation | User state is local and private; proof shows validity | Data is decrypted inside a secure enclave only |
| **Computation Location** | Server-side / coordinator | Client-side | Enclave nodes (e.g., on-chain or off-chain infrastructure) |
| **Transparency** | Hard to audit (math-heavy) | Transparent proofs verifiable by anyone | Requires trust in Intel SGX / E3 enclave infrastructure |
| **Zero Knowledge Proofs** | Optional; sometimes added for proof of correct tally | Native ZK: all computations are provable | Not required; relies on enclave attestation |
| **Resistance to Coercion** | High (vote is never revealed) | High (vote and state are private) | Depends on architecture; could leak if enclave compromised |
| **Setup Requirements** | Complex crypto primitives and bootstrapping | Requires Rust code and STARK circuit setup | Requires trusted hardware (e.g., SGX nodes, attestors) |
| **Infrastructure Trust** | Trust in cryptography | Trust in proof system only | Trust in hardware vendor + enclave integrity |
| **Speed / Cost** | Slow and expensive today | Fast client-side proving + cheap onchain verification | Fast (runs like a normal program), minimal gas if off-chain |
| **Best For** | Highly sensitive, long-term privacy (e.g. governments) | Sovereign private voting (DAOs, DeFi, small groups) | Fast, scalable voting with practical privacy (e.g. Web3) |

---

## 🧠 Deep Dive

### 🔐 1. Fully Homomorphic Encryption (FHE)

- **How it works**: Every vote is encrypted with a public key. A central tallying system performs arithmetic on the encrypted votes (e.g. adds them), without decrypting. Only after the tally, the result is decrypted.
- **Strengths**:
    - Highest privacy: votes are *never* decrypted individually.
    - Strong resistance to coercion, leaks, or replays.
- **Weaknesses**:
    - Slow and computationally heavy (even with CKKS, BFV, TFHE).
    - Harder to verify the process unless combined with ZK.
- **Good for**: E-voting in nation states, zero-trust environments, long-term archival privacy.

---

### 🌀 2. Miden (Client-side ZK STARK Execution)

- **How it works**: Each voter generates a vote on their device, computes locally, and creates a **ZK-STARK proof** that:
    - They are eligible to vote
    - Their vote is valid (e.g., in the right range)
    - They haven't voted twice
- Only the **proof and a commitment** are submitted on-chain. The actual vote is never revealed.
- **Strengths**:
    - Fully decentralized: no server, no trusted setup, no hardware.
    - Transparent and verifiable: anyone can audit proofs.
- **Weaknesses**:
    - Requires some technical skill or tooling for end-users.
    - STARK proof generation is large in size (but improving).
- **Good for**: DAOs, small/medium scale elections, anti-censorship voting, ZK-native systems.

---

### 🏰 3. Enclave Encrypted Execution Environments (E3 by Enclave.gg)

- **How it works**: Votes are sent to a secure enclave (e.g., using Intel SGX). Inside the enclave:
    - The votes are decrypted
    - Tallied privately
    - Results (and optional ZK proof or attestation) are emitted
- Enclave provides **attestation**, proving it ran the expected logic.
- **Strengths**:
    - Very fast, since computation runs natively (not cryptographically simulated).
    - Simplifies off-chain privacy with strong guarantees.
- **Weaknesses**:
    - Trust is rooted in hardware vendor (Intel, AMD, etc.).
    - Side-channel attacks or compromise of enclave can leak secrets.
- **Good for**: Fast, scalable voting in Web3 protocols, validator networks, or bridge governance.

---

## ✅ When to Use What?

| Use Case Example | Best Option | Why |
| --- | --- | --- |
| DAO voting with onchain settlement | **Miden** | Verifiable, decentralized, no trusted party |
| National election over the internet | **FHE** | Maximal privacy, no exposure of individual votes |
| Bridge governance with offchain voting | **Enclave E3** | Fast, practical, scalable with attestation |
| Game or prediction market votes | **Miden or E3** | Private + auditable; Miden if trust-minimized needed |
| Regulated environment requiring compliance + privacy | **FHE + ZK** | Full compliance audit without revealing individual data |

---

## 🧩 Summary in One Line Each

- **FHE**: Privacy-maximalist voting—compute directly on encrypted ballots.
- **Miden**: Zero-trust, ZK-native voting—each vote is provably correct and private.
- **E3/Enclave**: Trusted hardware voting—fast execution in a black box with cryptographic attestation.

---

Would you like a diagram, code sample, or architectural blueprint for building one of these systems?