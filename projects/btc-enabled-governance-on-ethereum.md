# BTC enabled governance settling on Ethereum

<aside>
💡

Due to the lack of smart contracts on Bitcoin,  It's not possible to create complex governance systems. 

How could we go about giving Bitcoin users the ability to express their opinion in a way which interfaces permissionlessly with Ethereum smart contracts?

</aside>

## TL;DR (Verdict)

Feasible and quick to ship for payment-weighted voting and donation rounds. THORChain does forward arbitrary bytes from UTXO memos to an EVM contract and caps the downstream call at ~400k gas.

Not fully "permissionless" in the sense of zero extra trust: you inherit THORChain's validator/TSS security and AMM pricing. That's acceptable for funding/QF-style votes; it's weaker for binding governance.

### Speculative solution

A speculative example of what a solution might look like using THORChain.

Here's a clean way to let **Bitcoin users vote** while **settling the vote on Ethereum**—using **THORChain** as the trust-minimized swap/bridge in the middle.

# The basic idea

- Voters **send native BTC** with a **THORChain memo** that encodes their *proposalId*, *choice*, and (optionally) a reference to their BTC identity.
- THORChain **swaps BTC → ETH (or any ERC-20)** and then **calls an Ethereum contract** using its "**SwapOut + transferOutAndCall**" flow. You can pass extra data after a `|` in the memo; THORChain forwards it to the EVM-side "aggregator" contract, enabling **generic data to contracts cross-chain**. [dev.thorchain.org](https://dev.thorchain.org/concepts/memos.html)
- Your **Aggregator** forwards ETH/tokens and the decoded memo payload to your **VoteReceiver** contract, which tallies the vote on Ethereum (and can forward funds to a treasury). [dev.thorchain.org](https://dev.thorchain.org/aggregators/evm-implementation.html)

---

# Trust Assumptions

## External Trust
- You inherit THORChain's validator/TSS and solvency model. It's robust and audited, but it is a cross-chain system with historical exploits (2021)
- Use it for signaling or funding; be cautious tying it to direct, irreversible execution without guardrails
- **Mitigation**: Make your VoteReceiver signal-only (emit tallies). If you need on-chain execution, gate with a time-buffered timelock + guardian veto or require a secondary quorum on Ethereum

## Not Fully Permissionless
- You rely on THORChain's consensus and liquidity pools to transport value and your memo to Ethereum
- If THORChain halts/partitions, your governance halts. That's acceptable for donation voting; risky for time-sensitive execution
- THORChain removed EVM contract whitelisting in 2024 - good for permissionless composition but increases onus on contract hardening

---

# Components

1. **Bitcoin (wallet + memo)**
- Users send BTC to THORChain's current **Asgard vault** with a **SWAP memo**:
    
    ```
    SWAP:ETH.ETH:0xYourAggregator:0::0xAggAddr:0xFinalToken:0|
    0x<ABI_ENCODED( proposalId, choiceId, optionalReference )>
    
    ```
    
    - `ETH.ETH` can be any asset THORChain supports as the intermediate (e.g., ETH).
    - After the `|`, you include arbitrary bytes; THORChain will send that **outbound memo** to the aggregator so it can be delivered cross-chain.
    - **Important**: Asgard vaults churn, so you must fetch fresh inbound addresses (don't preprint forever-valid QR codes)

2. **THORChain**
- Observes your BTC deposit, performs the swap, then calls your Aggregator using **`transferOutAndCall`** (400k gas cap) and forwards the memo payload. (This is the "SwapOut" path.)
- Notes:
    - **UTXO memo limit** is tight (Bitcoin OP_RETURN ~80 bytes), so compact-encode the payload (e.g., CBOR/varints) or keep fields short. THORChain's overall memo limit is 250 bytes, but UTXO chains are the constraint.
    - **Synths are deprecated**, so don't rely on them. Use straight L1 swaps.

3. **Ethereum Aggregator (your contract)**
- THORChain will call your function (typ. `swapOut(address finalToken,address to,uint256 minOut)`), and the **memo data after `|` is passed through the router's SwapOut flow** for you to consume/forward. (Router V5 uses `transferOutAndCall` and emits/propagates the memo; your aggregator handles the last hop.)
- Your aggregator can:
    - (optional) route through an on-chain AMM (Uniswap) to get a **governance token** instead of raw ETH, then
    - call your **VoteReceiver** with: `(proposalId, choiceId, amount, senderOnBTC?, extra)`.
- **Security**: Router-only gate: `require(msg.sender == THOR_ROUTER)` in the Aggregator

4. **VoteReceiver (Ethereum)**
- Minimal interface:
    - Validate the calling **aggregator** (only accept votes from your aggregator).
    - **Record votes** (bucket by `proposalId` and `choiceId`) weighted by **received ETH / token amount**.
    - Emit events for off-chain analytics/attribution, and forward funds to your treasury (if "pay-to-vote / QF").
- Optional: enforce **one-vote-per-identity** if you use a signature scheme (see "Identity options" below).
- **Security**: 
    - In VoteReceiver `require(msg.sender == Aggregator)`
    - Idempotency: Store `consumed[outTxHash] = true`. Ignore repeats.

---

# Two governance flavors you can run

## A) "Pay-to-vote" (fastest to ship)

- Weight = value delivered to Ethereum. Great for **Quadratic Funding** or donation-weighted polls.
- Flow:
    1. Voter sends BTC with memo `SWAP:ETH.ETH:0xAggregator...|abi(proposalId, choiceId)`.
    2. THORChain swaps BTC→ETH and calls your Aggregator → VoteReceiver.
    3. VoteReceiver increments tally for that `(proposalId, choiceId)` by the **received amount**.
- This is entirely **on-chain verifiable on Ethereum** (no off-chain trust beyond THORChain itself).

## B) "BTC key–based voting" (identity from Bitcoin)

- Use **BIP-322** message signing so voters prove control of a BTC address and sign `vote(proposalId, choiceId, nonce)` with their **Bitcoin key**.
- Practical enforcement patterns:
    - **Hybrid**: The BTC signature is **verified off-chain** by your relayer, which then **calls** `VoteReceiver.castSignedVote(...)`. You still **settle value** on Ethereum via THORChain; the ETH deposit tx + an event linking the BTC signature gives end-to-end traceability.
    - **Proof-of-donation + signature**: Require a **tiny BTC "ticket" payment** alongside the signature; your indexer uses THORChain's **in→out mapping** to link the BTC input UTXO and the resulting ETH vote. (Cleaner Sybil resistance than signatures alone.)
- Fully on-chain BIP-322 verification inside EVM is heavy (Taproot scripts etc.), so expect an off-chain verifier or a specialized precompile/zk gadget if you need trust-minimized identity.
- **Note**: BIP-322 wallet support is uneven (Sparrow/Knots yes; many mobile wallets no)

---

# Key Risks & Hardening

## Replay/Duplicate Votes
- A malicious relayer can try to replay the same memo bytes in different contexts
- **Mitigation**: Key each vote to THORChain outbound tx hash (and chain-height) once and reject duplicates; store `(proposalId, choiceId, outTxHash)`

## Economic & Gaming Vectors
- Weight = post-swap received amount → exposed to price movement, slip fees, and adversarial flow around the time of your vote
- **Mitigations**:
    - Use oracle-normalized weight: weight by estimated BTC value at mid-price rather than raw ETH out
    - Or fix the weight unit (e.g., require router to deliver governance token via your Aggregator's internal 1:1 mint/burn)

## Fail Paths
- If your Aggregator/AMM leg reverts or minOut fails, the router may deliver base asset instead and skip your call
- **Mitigation**: Instruct clients to set sane minOut & avoid AMM hop unless necessary; your Aggregator should gracefully no-op on unknown payloads, never revert

## QF/QV Sybil Resistance
- Splitting BTC across many UTXOs/addresses is cheap; payment-only QF is trivial to game
- **Mitigations**: Pair payment with identity or run optimistic anti-Sybil heuristics and challenge window off-chain before final settlement

---

# Concrete Contract Recommendations

## Payload Schema (≤80 bytes)
- proposalId: uint32
- choiceId: uint8
- flags: uint8 (e.g., versioning)
- nonce: uint32
- hintBtc: bytes20 (optional)
- sigRef: bytesN (optional, omit on-chain; use off-chain index)

Use varints/packed ABI; avoid strings.

## Events
Emit `VoteCast(proposalId, choiceId, normalizedUnits, outTxHash, btcHint)` for auditability.

## Gas Envelope
Keep Aggregator lean: decode → (optional) deterministic swap → call VoteReceiver. Avoid reentrancy into the router; no external callbacks. Stay within ~400k gas.

---

# Example (end-to-end) memo & flow

1. **BTC wallet builds memo (compact)**
    
    ```
    SWAP:ETH.ETH:0xYourAggregator:0::0xAggAddr:0xFinalToken:0|0x
    <abi(proposalId, choiceId, bytes20_hintOfBtcSender)>
    
    ```
    
    - Keep the data tiny to fit Bitcoin's OP_RETURN limit.
2. **THORChain executes SwapOut → `transferOutAndCall`**
    - Router calls your **Aggregator** on Ethereum with base asset (ETH) and the data is made available per the SwapOut memo passthrough feature. Your Aggregator can swap on an AMM if you want a **governance token**, then call your **VoteReceiver** with `(proposalId, choiceId, amount, optionalBtcHint)`.
3. **VoteReceiver**
    - Checks `msg.sender == Aggregator`.
    - `tallies[proposalId][choiceId] += receivedAmount;`
    - Emits `VoteCast(proposalId, choiceId, receivedAmount, btcHint)`.

---

# Identity & Sybil options (pick what fits)

- **Pure payment**: Simple; the ETH received is the signal. No identity. (Best for QF/QV-like funding.)
- **BIP-322 signed votes**: BTC address = identity; verify off-chain, submit proof on Ethereum.
- **One-ticket-per-UTXO**: require a dust-level BTC "ticket" to a per-proposal tag; your indexer enforces one ticket per unique UTXO or per time-lock.
- **Nostr pubkeys** as portable identity (secp256k1), bridged to Ethereum address mapping (off-chain).

---

# Implementation Checklist

## Client
- PSBT builder + QR flow for OP_RETURN memos (≤80 bytes) and auto-fetch of inbound_address with expiry
- Warn users not to re-use cached addresses
- Provide web PSBT + QR flow (Sparrow/Keystone/Xverse compatible)

## Contracts
- Minimal Aggregator (decode + optional swap) within ~400k gas
- VoteReceiver with idempotency on THOR outTxHash, normalized weights, and no external calls beyond logging/treasury transfer

## Indexing & Analytics
- Subscribe to Router events on Ethereum (outbound memo + to + outTxHash) and your VoteCast events to build public dashboards

## Docs/SecOps
- Explicitly document risk acceptance: "signaling/funding only," or introduce a timelocked execution phase if binding
- Put per-proposal min/max vote windows to minimize price manipulation around closes

---

# UX & Wallet Constraints

- **Memo authoring on BTC**: You'll need a PSBT builder that sets (1) Asgard inbound address, (2) change, (3) OP_RETURN ≤80 bytes in the right order. Many mainstream wallets won't support this.
- **Good UX caveat**: BTC wallets that can craft an OP_RETURN memo ≤80 bytes are limited; you'll likely need a PSBT flow or wallet integrations
- Pre-generate QR codes with the **exact BTC amount + memo** per proposal/choice so users in mobile wallets can vote in one scan

---

# Alternative Architectures to Consider

## tBTC v2 Route (BTC → TBTC → direct voting on Ethereum)
- Users lock BTC with Threshold's threshold-ECDSA signers and mint TBTC (ERC-20), then vote directly with an EVM wallet
- **Pros**: Best UX on Ethereum, huge wallet surface, simple contracts
- **Cons**: Trust shifts from THORChain to Threshold's signer set/economics; it's a bridge, not "pure BTC key" identity

## Optimistic "OP_RETURN rollup"
- Put vote(proposal, choice) commitments in Bitcoin OP_RETURN; off-chain indexer builds a Merkle root; an optimistic oracle/settler posts the root to Ethereum; watchers can dispute with inclusion proofs
- **Pros**: No liquidity/price games; cheap on Ethereum; keeps BTC L1 as the source of truth
- **Cons**: Needs dispute game + watchdogs; longer finality; still some trust during challenge

## On-chain SPV/Light Clients
- Revive BTC Relay-style headers on Ethereum, or use emerging zk-headers (ZeroSync) to verify Bitcoin transactions natively in an EVM contract
- **Pros**: Strongest trust model (cryptographic) once mature
- **Cons**: Engineering complexity & cost today; zk-headers are still in active R&D

---

# Variations you might like

- **QF module**: Have `VoteReceiver` credit per-choice "matching buckets," then run a quadratic matching calculation on-chain or finalize via a job that writes results back (MACI-style).
- **Governor hook**: After a proposal reaches a threshold, your contract can trigger an **Ethereum Governor** or **Safe module** to execute a result.