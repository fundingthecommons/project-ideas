# Privacy first DEX with Miden

Designing a DEX with private limit orders on Miden is an excellent fit for its architecture: you can move order management and matching off-chain (on user devices), and post only zero-knowledge proofs on-chain to verify trades.

## 🧱 Key Design Principles

1. **Users create and manage limit orders locally** (on their devices).
2. Orders are **encrypted and stored off-chain**, or shared peer-to-peer.
3. **Matching** happens client-side or via a matching coordinator (could be a relayer or network of solvers).
4. When a match is found, both sides generate **ZK proofs** of:
    - Order validity
    - Balance sufficiency
    - Matching price
5. Submit **proof + trade commitment** to Miden chain.
6. The on-chain contract **verifies the proof** and updates balances—without ever seeing the raw order details.