---
title: "Pattern: Atomic DvP via ERC-7573 (cross-network)"
status: ready
maturity: pilot
works-best-when:
  - Asset side and payment side live on different networks (L1/L2 or sidechains).
avoid-when:
  - Both sides already settle on the same network with simple on-chain transfers.
dependencies:
  - ERC-7573 locking contract (asset side)
  - ERC-7573 decryption contract (payment side)
  - Stateless decryption oracle on the payment network
---

## Intent

Enable atomic Delivery-versus-Payment (DvP) across two networks with ERC-7573: either the asset is delivered to the buyer and the payment has completed, or the asset is returned to the seller, without manual reconciliation.

In this pattern:
- **asset side** = the tokenised security or collateral being delivered,
- **payment side** = the cash-like token used for payment (for example, a stablecoin or tokenised deposit),
- **decryption oracle** = a service on the payment network that holds a decryption key and, when called by the decryption contract, decrypts one of two pre-encrypted keys. The resulting key is then used on the asset network to either deliver the asset or allow reclaim.

This pattern targets institutions that need cross-network settlement with predictable behaviour, clear records, and an option to add privacy over time.

## Ingredients

- **Contracts:**
  - ERC-7573 locking contract on the asset network
  - ERC-7573 decryption contract on the payment network
  - Oracle proxy and callback contracts as defined by ERC-7573

- **Infra:**
  - Asset network where the security or collateral is issued
  - Payment network that holds the payment token
  - Decryption oracle implementation (run by one or several operators)

- **Off-chain:**
  - Trade negotiation or order system that assigns a shared trade identifier
  - Key management for generating and storing the secret keys used in the flow
  - Monitoring, alerting, and logging for the contracts and oracle

## Protocol

1. **Agree the trade off-chain:**
   - Two institutions agree on:
     - the asset and quantity on the asset network,
     - the payment token and amount on the payment network,
     - a shared trade identifier `T`, and
     - a latest time to either settle or unwind.
   - They record `T` and the terms in their internal systems.

2. **Set up keys and lock the asset:**
   - Each trade `T` uses two outcome keys: one for “deliver to buyer” and one for “return to seller”.  
   - These keys are produced by the system that sets up the trade and are shared with both institutions so they can register the required derived values on each network. The keys themselves never appear on-chain.
   - The seller then locks the asset into the ERC-7573 locking contract under `T`. The contract will later choose between the two outcomes based on input from the payment side.

3. **Register the payment:**
   - On the payment network, the buyer registers the same `T` in the ERC-7573 decryption contract together with the payment details.
   - The contract stores two encrypted versions of the outcome keys for `T`, one for “payment succeeded” and one for “payment failed or cancelled”.

4. **Execute payment and call the oracle:**
   - The buyer executes the payment through the decryption contract. The contract:
     - checks that the payment matches the registered details for `T`, and
     - emits a request for the decryption oracle to reveal the encrypted value that matches the actual outcome (success or failure).
   - The oracle runs on the payment network, decrypts the selected value, and returns the corresponding plaintext to the decryption contract.

5. **Settle on the asset side:**
   - An authorised party submits this plaintext to the locking contract on the asset network under `T`.  
   - The contract verifies it and then either:
     - delivers the locked asset to the buyer if the payment completed, or
     - makes the asset available for the seller to reclaim if the payment failed, was cancelled, or never completed in time.
   - If nothing is submitted before the agreed latest time, the seller can follow the reclaim path to recover the asset.

## Privacy Extensions

Standard ERC-7573 gives atomic settlement but does not hide trade size or participants. Many institutions want to limit what is visible on public networks without changing the core contracts.

Common extensions:

- **Multi-operator oracle**  
  - Run the decryption oracle under several independent operators instead of one. A minimum number of them must cooperate before any key is revealed. The ERC-7573 interfaces stay the same.

- **Minimal on-chain trade data**  
  - Keep full trade terms (price, size, counterparties) in internal systems. On-chain, store only a trade identifier and a short reference that links back to those records (for example, hashes, commitments,...), rather than the full terms.

- **Private or proof-based payment rail**  
  - Execute the payment on a network or rollup that hides detailed balances but can give a clear “payment completed / not completed” result for a given trade identifier.

These can be added without changing how ERC-7573 decides the outcome: the asset contract still takes an outcome key and either delivers the asset or allows reclaim.

## Guarantees

- **Atomic settlement**
  - Each trade either settles on both sides (asset delivered and payment completed) or is unwound (asset returned to the seller). There is no state where only one side has settled.

- **Predictable failure behaviour**
  - If the payment fails, is cancelled, no outcome is produced, or the agreed latest time passes, the contracts follow a defined reclaim path so the seller can recover the asset. This behaviour is visible on-chain and straightforward to include in operations runbooks.

- **Limited trust assumptions**
  - The contracts are deterministic. Beyond their code, the main assumption is that the oracle (or group of oracle operators) only reveals an outcome when the payment-side conditions are met.

- **Optional privacy**
  - When combined with minimal on-chain trade data and a private or proof-based payment rail, public observers see that a trade was settled or unwound without seeing full trade terms. Institutions can still disclose those terms and supporting evidence off-chain when required.

## Trade-offs

- **Oracle governance**
  - The oracle is a critical component. Institutions must agree who runs it, how its keys are managed, and how incidents such as downtime or misconfiguration are handled.

- **Latency**
  - Settlement time is bounded by payment finality, oracle processing, and any proof generation. This may be slower than a same-network transfer but is predictable once configured.

- **Integration effort**
  - Internal systems must:
    - generate and store the keys,
    - track trade identifiers end-to-end,
    - listen to contract events on both networks, and
    - reconcile them with internal ledgers.

- **Dispute handling**
  - Exceptional cases (for example, wrong parameters, operational errors, or regulatory holds) still need documented off-chain processes, even though the normal path is fully automated.

## Example

**Cross-network bond DvP with ERC-7573**

- Bank A issues a tokenised bond on Ethereum L1 (asset side).
- Bank B holds EURC stablecoin on a rollup (payment side).
- They agree off-chain on a trade with id `T`, including bond quantity, payment amount, and a latest settlement time.
- Bank A locks the bond in the ERC-7573 locking contract on L1 under `T`.
- Bank B sets up and then executes the stablecoin payment via the ERC-7573 decryption contract on the rollup.
- The decryption oracle returns the success key for `T`. Bank A or Bank B submits this key to the L1 locking contract, which transfers the bond to Bank B.
- If the payment had failed or been cancelled, the failure key would have been returned instead, and the L1 contract would have allowed Bank A to reclaim the bond.

## See also

- [Private Trade Settlement](../approaches/approach-private-trade-settlement.md)
- [MPC Custody](pattern-mpc-custody.md)
- [Selective Disclosure](pattern-regulatory-disclosure-keys-proofs.md)
- [ERC-7573 spec](https://ercs.ethereum.org/ERCS/erc-7573)
