# BTC enabled governance settling on Ethereum

<aside>
💡

Due to the lack of smart contracts on Bitcoin,  It's not possible to create complex governance systems. 

How could we go about giving Bitcoin users the ability to express their opinion in a way which interfaces permissionlessly with Ethereum smart contracts?

</aside>

### Speculative solution

A speculative example of what a solution might look like using a Thor chain.

Here's a clean way to let **Bitcoin users vote** while **settling the vote on Ethereum**—using **THORChain** as the trust-minimized swap/bridge in the middle.

# The basic idea

- Voters **send native BTC** with a **THORChain memo** that encodes their *proposalId*, *choice*, and (optionally) a reference to their BTC identity.
- THORChain **swaps BTC → ETH (or any ERC-20)** and then **calls an Ethereum contract** using its "**SwapOut + transferOutAndCall**" flow. You can pass extra data after a `|` in the memo; THORChain forwards it to the EVM-side "aggregator" contract, enabling **generic data to contracts cross-chain**. [dev.thorchain.org+2dev.thorchain.org+2](https://dev.thorchain.org/concepts/memos.html)
- Your **Aggregator** forwards ETH/tokens and the decoded memo payload to your **VoteReceiver** contract, which tallies the vote on Ethereum (and can forward funds to a treasury). [dev.thorchain.org+1](https://dev.thorchain.org/aggregators/evm-implementation.html?utm_source=chatgpt.com)

---

# Components

1. **Bitcoin (wallet + memo)**
- Users send BTC to THORChain's current **Asgard vault** with a **SWAP memo**:
    
    ```
    SWAP:ETH.ETH:0xYourAggregator:0::0xAggAddr:0xFinalToken:0|
    0x<ABI_ENCODED( proposalId, choiceId, optionalReference )>
    
    ```
    
    - `ETH.ETH` can be any asset THORChain supports as the intermediate (e.g., ETH).
    - After the `|`, you include arbitrary bytes; THORChain will send that **outbound memo** to the aggregator so it can be delivered cross-chain. [dev.thorchain.org+1](https://dev.thorchain.org/concepts/memos.html)
1. **THORChain**
- Observes your BTC deposit, performs the swap, then calls your **whitelisted Aggregator** using **`transferOutAndCall`** (400k gas cap) and forwards the memo payload. (This is the "SwapOut" path.) [dev.thorchain.org](https://dev.thorchain.org/aggregators/evm-implementation.html)
- Notes:
    - **UTXO memo limit** is tight (Bitcoin OP_RETURN ~80 bytes), so compact-encode the payload (e.g., CBOR/varints) or keep fields short. THORChain's overall memo limit is 250 bytes, but UTXO chains are the constraint. [dev.thorchain.org](https://dev.thorchain.org/concepts/memos.html)
    - **Synths are deprecated**, so don't rely on them. Use straight L1 swaps. [THORChain Docs](https://docs.thorchain.org/thorchain-finance/synthetic-asset-model?utm_source=chatgpt.com)
1. **Ethereum Aggregator (your contract)**
- Must be **whitelisted** by THORChain as an "aggregator" target for SwapOut. THORChain will call your function (typ. `swapOut(address finalToken,address to,uint256 minOut)`), and the **memo data after `|` is passed through the router's SwapOut flow** for you to consume/forward. (Router V5 uses `transferOutAndCall` and emits/propagates the memo; your aggregator handles the last hop.) [dev.thorchain.org](https://dev.thorchain.org/aggregators/aggregator-overview.html?utm_source=chatgpt.com)[Medium](https://medium.com/thorchain/thorchain-q1-2025-report-q2-roadmap-ffdb9e303c74?utm_source=chatgpt.com)[about.gitlab.com](https://gitlab.com/thorchain/thornode/-/blob/develop/chain/ethereum/contracts/THORChain_Router.sol?utm_source=chatgpt.com)
- Your aggregator can:
    - (optional) route through an on-chain AMM (Uniswap) to get a **governance token** instead of raw ETH, then
    - call your **VoteReceiver** with: `(proposalId, choiceId, amount, senderOnBTC?, extra)`.
1. **VoteReceiver (Ethereum)**
- Minimal interface:
    - Validate the calling **aggregator** (only accept votes from your whitelisted aggregator).
    - **Record votes** (bucket by `proposalId` and `choiceId`) weighted by **received ETH / token amount**.
    - Emit events for off-chain analytics/attribution, and forward funds to your treasury (if "pay-to-vote / QF").
- Optional: enforce **one-vote-per-identity** if you use a signature scheme (see "Identity options" below).

---

# Two governance flavors you can run

## A) "Pay-to-vote" (fastest to ship)

- Weight = value delivered to Ethereum. Great for **Quadratic Funding** or donation-weighted polls.
- Flow:
    1. Voter sends BTC with memo `SWAP:ETH.ETH:0xAggregator...|abi(proposalId, choiceId)`.
    2. THORChain swaps BTC→ETH and calls your Aggregator → VoteReceiver.
    3. VoteReceiver increments tally for that `(proposalId, choiceId)` by the **received amount**.
- This is entirely **on-chain verifiable on Ethereum** (no off-chain trust beyond THORChain itself). [dev.thorchain.org](https://dev.thorchain.org/aggregators/evm-implementation.html?utm_source=chatgpt.com)

## B) "BTC key–based voting" (identity from Bitcoin)

- Use **BIP-322** message signing so voters prove control of a BTC address and sign `vote(proposalId, choiceId, nonce)` with their **Bitcoin key**. [Bitcoin Wiki](https://en.bitcoin.it/wiki/BIP_0322?utm_source=chatgpt.com)[GitHub](https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki?utm_source=chatgpt.com)[Bitcoin Optech](https://bitcoinops.org/en/topics/generic-signmessage/?utm_source=chatgpt.com)
- Practical enforcement patterns:
    - **Hybrid**: The BTC signature is **verified off-chain** by your relayer, which then **calls** `VoteReceiver.castSignedVote(...)`. You still **settle value** on Ethereum via THORChain; the ETH deposit tx + an event linking the BTC signature gives end-to-end traceability.
    - **Proof-of-donation + signature**: Require a **tiny BTC "ticket" payment** alongside the signature; your indexer uses THORChain's **in→out mapping** to link the BTC input UTXO and the resulting ETH vote. (Cleaner Sybil resistance than signatures alone.)
- Fully on-chain BIP-322 verification inside EVM is heavy (Taproot scripts etc.), so expect an off-chain verifier or a specialized precompile/zk gadget if you need trust-minimized identity.

---

# Example (end-to-end) memo & flow

1. **BTC wallet builds memo (compact)**
    
    ```
    SWAP:ETH.ETH:0xYourAggregator:0::0xAggAddr:0xFinalToken:0|0x
    <abi(proposalId, choiceId, bytes20_hintOfBtcSender)>
    
    ```
    
    - Keep the data tiny to fit Bitcoin's OP_RETURN limit. [dev.thorchain.org](https://dev.thorchain.org/concepts/memos.html)
2. **THORChain executes SwapOut → `transferOutAndCall`**
    - Router calls your **Aggregator** on Ethereum with base asset (ETH) and the data is made available per the SwapOut memo passthrough feature. Your Aggregator can swap on an AMM if you want a **governance token**, then call your **VoteReceiver** with `(proposalId, choiceId, amount, optionalBtcHint)`. [dev.thorchain.org+1](https://dev.thorchain.org/aggregators/evm-implementation.html)
3. **VoteReceiver**
    - Checks `msg.sender == Aggregator`.
    - `tallies[proposalId][choiceId] += receivedAmount;`
    - Emits `VoteCast(proposalId, choiceId, receivedAmount, btcHint)`.

---

# Identity & Sybil options (pick what fits)

- **Pure payment**: Simple; the ETH received is the signal. No identity. (Best for QF/QV-like funding.)
- **BIP-322 signed votes**: BTC address = identity; verify off-chain, submit proof on Ethereum. [Bitcoin Wiki](https://en.bitcoin.it/wiki/BIP_0322?utm_source=chatgpt.com)
- **One-ticket-per-UTXO**: require a dust-level BTC "ticket" to a per-proposal tag; your indexer enforces one ticket per unique UTXO or per time-lock.
- **Nostr pubkeys** as portable identity (secp256k1), bridged to Ethereum address mapping (off-chain).

---

# What you'll need to build

- **Ethereum**
    - `Aggregator.sol` (SwapOut target). Minimal Uniswap router integration + a `forwardVote(payload, to)` that decodes the memo bytes and calls `VoteReceiver`. (Uses THORChain Router's **`transferOutAndCall`** path.) [dev.thorchain.org](https://dev.thorchain.org/aggregators/evm-implementation.html)
    - `VoteReceiver.sol` that tallies votes, emits events, and forwards funds.
- **Off-chain**
    - A lightweight **indexer** that watches THORChain Router events (e.g., `TransferOutAndCall`) and your `VoteCast` events to build public analytics and (optionally) match BTC inputs to ETH votes.
    - (If using BIP-322) a verifier service to validate signatures before submitting to `VoteReceiver`.
- **THORChain side**
    - **Get your Aggregator whitelisted** for SwapOut (standard step for cross-chain aggregation targets). [dev.thorchain.org](https://dev.thorchain.org/aggregators/aggregator-overview.html?utm_source=chatgpt.com)
    - Use the **latest Asgard/Router** endpoints when constructing deposits; UTXO memos must stay within **80 bytes**. [dev.thorchain.org](https://dev.thorchain.org/concepts/sending-transactions.html?utm_source=chatgpt.com)

---

# Gotchas & tips

- **Memo length**: Design a **compact payload** (e.g., `proposalId` as uint32, `choiceId` as uint8, optional 20-byte hint). Use CBOR/RLP if you need more structure. UTXO chains are the limiting factor. [dev.thorchain.org](https://dev.thorchain.org/concepts/memos.html)
- **Gas budget**: SwapOut calls give **~400k gas**; keep your Aggregator + AMM hop + VoteReceiver call within that. [dev.thorchain.org](https://dev.thorchain.org/aggregators/evm-implementation.html)
- **Synths**: Don't lean on them—they've been sunset. Use direct pools. [THORChain Docs](https://docs.thorchain.org/thorchain-finance/synthetic-asset-model?utm_source=chatgpt.com)
- **Security**: Accept votes **only** from your Aggregator; protect against replay by keying tallies to the **exact outbound tx hash** (emit + store) or by nonce inside the memo payload.
- **UX**: Pre-generate QR codes with the **exact BTC amount + memo** per proposal/choice so users in mobile wallets can vote in one scan.

---

# Variations you might like

- **QF module**: Have `VoteReceiver` credit per-choice "matching buckets," then run a quadratic matching calculation on-chain or finalize via a job that writes results back (MACI-style).
- **Governor hook**: After a proposal reaches a threshold, your contract can trigger an **Ethereum Governor** or **Safe module** to execute a result.