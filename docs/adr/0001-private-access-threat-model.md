# ADR 0001: Private Access Threat Model

## Status
Proposed

## Labels
type:docs

## Context
Before implementing a private-access registry in the Soroban smart contracts for the Stellar-AgentVerse platform, it is crucial to strictly define the security architecture and evaluate trade-offs. The current system handles prompt sales via contracts (`PromptMarketplace`, `MyToken`). We need to guarantee the privacy of the prompts accessed by authorized buyers, minimizing exposure on the blockchain. Since this analysis focuses on the **Smart Contracts** layer, we evaluate how to design the contracts to support secure private access without leaking sensitive information on the public Stellar network.

## 1. Threat Model

### Actors and Capabilities
*   **Blockchain Observers**: Anyone monitoring the public Stellar ledger. They can see all transactions, contract invocations, arguments, state changes, events, and execution timing.
*   **Backend / Indexers**: Off-chain infrastructure that interacts with contracts or reads events. They might attempt to correlate on-chain activity with specific users.
*   **Admins**: Platform operators who manage the marketplace.
*   **Buyers**: Users purchasing access to prompts. They might act maliciously by attempting to access unpaid prompts, reuse access (replay attacks), or bypass contract controls.

## 2. Leakage Analysis

In the context of Soroban smart contracts, information can leak in several ways:

*   **Arguments**: Passing plaintext prompt IDs or access keys in contract function arguments (e.g., `buy_prompt(id="secret-alpha")`) will leak this data on the ledger.
*   **Auth Entries**: Soroban `require_auth()` payloads expose the addresses and the exact contract functions being authorized, revealing who is participating and in what.
*   **Storage**: On-chain state is public. Storing unencrypted prompt content or plaintext access keys is a critical vulnerability.
*   **Events**: Emitting events like `PromptPurchased` with detailed data (`buyer`, `prompt_id`, `price`) leaves a permanent public trail of user activity, leaking their interests.
*   **Token Activity**: Token transfers (`MyToken`), such as mints and burns, correlate with purchases and reveal the economic value exchanged.
*   **Timing**: The time between on-chain purchase execution and off-chain access claim can help correlate identities if not obfuscated.

## 3. Solution Trade-offs

We compare four potential technical approaches for the smart contracts:

| Approach | Privacy Guarantees | Cost (Fees / Compute) | Soroban Feasibility | Operational Overhead |
| :--- | :--- | :--- | :--- | :--- |
| **Opaque Access Records (Hashes/Commitments)** | **Moderate**. Hides specific IDs on-chain using hashes (e.g., SHA-256). Buying patterns remain visible. | **Low**. Hashing is efficient and cheap in Soroban. | **High**. Very simple to implement in Rust/Soroban. | **Low**. Requires the client or backend to validate hashes. |
| **Encrypted Off-chain Delivery (On-chain tracking only for rights)** | **High**. The contract only records access rights. Content privacy relies on off-chain delivery. | **Low**. Minimal on-chain logic. | **High**. Keeps contracts simple. | **Moderate**. Requires robust off-chain infrastructure. |
| **Relayers (Meta-transactions)** | **High (for identity)**. Hides the buyer's address by subsidizing fees through a relayer. | **High**. Requires managing relayer accounts and complex signature delegation. | **Moderate**. Soroban supports auth delegation, but requires extra management. | **High**. Relayer infrastructure maintenance. |
| **Zero-Knowledge (ZK)** | **Very High**. Allows proving the purchase without revealing *who* bought *what*. | **Very High**. Expensive on-chain verification (compute limits). | **Low**. ZK support in Soroban is nascent and would require complex circuits. | **Very High**. Requires prover infrastructure and circuit setup. |

## 4. Security & Lifecycle Definitions

For the implementation of the access registry in Soroban, we define the following requirements:

### Anti-replay Mechanisms
If contracts must verify access claims, they must include nonce mechanisms per address or strict timestamps in delegated signatures to prevent a single authorization from being reused multiple times (replay attacks).

### Key Lifecycle (Verifiable Protocol)
To ensure a verifiable boundary of trust, the key lifecycle is defined as follows:
1.  **Authentication & Key Separation**: Buyers must authenticate using a dedicated encryption public key that is strictly separate from their Stellar signing key.
2.  **Plaintext & Content Keys Visibility**: Only the buyer's client environment and the ephemeral KMS/AI decryption service can ever view the plaintext prompt or content keys. The backend infrastructure must guarantee it never stores plaintext long-term.
3.  **Rotation & Revocation**: When a key is rotated, the off-chain system will re-encrypt the content for the new public key. Since the on-chain contract only tracks the entitlement (via hash), on-chain migration is not required for key rotation, but the off-chain delivery system must revoke the old key pair.

### Delivery Protocol (Contract Perspective)
1.  The buyer submits a purchase transaction on-chain sending an opaque **hash (commitment)** instead of the plaintext prompt ID.
2.  The contract burns the corresponding tokens and stores the commitment.
3.  The contract emits a generic or opaque event for indexing, without revealing the buyer's direct identity if delegations are used, or at least hiding the purchased asset.

### Migration Paths
If ZK or another advanced technology is adopted in the future, the contract must use upgradable patterns or storage delegation to allow migration of old access records to the new format without loss of rights.

### Acceptance Criteria & Required Test Evidence
To consider this approach valid, the following must be proven:
*   **Forbidden Fields & Logs**: It is strictly forbidden for `content_uri`, `title`, or plaintext `prompt_id` to appear in any Soroban storage, argument, or event log.
*   **Replay Proofs (Unit Tests)**: Rust tests must cryptographically prove that attempting to replay a purchase or claim with a reused hash/signature is rejected by the contract.
*   **Backend Evidence (Issue #8)**: The backend implementation must provide concrete evidence (test coverage) that it performs hashing before invoking the contract, and that it explicitly zeroes out the decryption Key (DEK) from memory immediately after execution.

## Decision
*(To be finalized upon approval)*
We recommend moving forward with an **Opaque Access Records (Hashes/Commitments)** approach at the smart contract level. **Explicit Limitation:** As long as `buy_prompt(buyer, prompt_hash)` and `has_access(user, prompt_hash)` keep their arguments, storage, and events public on the ledger, the encrypted off-chain delivery CANNOT preserve an on-chain entitlement verification that hides the `buyer↔prompt` link. On-chain unlinkability is strictly out of scope for this iteration and remains a future goal requiring advanced commitments or ZK proofs.

## Consequences
*   We will modify `PromptMarketplace` to accept opaque identifiers (hashes) instead of plaintext prompt IDs.
*   Emitted events will be restructured to maximize privacy.
*   Changes will be required in how clients construct transactions (client-side hashing before calling the contract).
