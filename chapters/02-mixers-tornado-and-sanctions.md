# Mixers: Tornado Cash Mechanics and Sanctions History

Tornado Cash is the most studied privacy protocol on Ethereum. Its small, public code and its history (a circuit bug, a governance takeover, deanonymisation research, and a sanctions designation later withdrawn) cover most failure modes in this book.

## Mechanics

Tornado Cash ran separate pools per asset and denomination (for ETH: 0.1, 1, 10 and 100). Using the public `tornado-core` code:

| Step | On-chain action | In the circuit |
|---|---|---|
| Deposit | `deposit(commitment)` with exactly the denomination; commitment inserted into a Merkle tree (depth 20, MiMC sponge hash) | Commitment = Pedersen hash of `nullifier ‖ secret` |
| Wait | Other users deposit; tree root changes | — |
| Withdraw | `withdraw(proof, root, nullifierHash, recipient, relayer, fee, refund)` | Proves knowledge of a leaf under `root`, and that `nullifierHash` = Pedersen(`nullifier`); recipient, relayer, fee and refund are public inputs |
| Contract checks | `isKnownRoot(root)` against a short history of recent roots; `nullifierHashes[nullifierHash]` unused; Groth16 verify | — |

Two details matter for security:

- **Public inputs bind the payout.** The recipient, relayer and fee are public inputs. If they were not part of the proof, anyone watching the mempool could copy the proof and change the recipient. Tornado's circuit includes them with dummy constraints (squaring them) so the compiler does not optimise them away.
- **Root history.** The contract accepts any of the last N roots (30 in `tornado-core`), so a proof built against a slightly old root still works after new deposits. Accepting arbitrary roots would let an attacker prove membership in a tree they invented.

Relayers submit withdrawals for users with no gas ETH, taking the fee; they cannot change the recipient, a public input.

## The 2019 circuit bug

In late 2019 the Tornado Cash team published a post-mortem ("Tornado.cash got hacked. By us.") describing a bug in the MiMC hash implementation in circomlib used by the Merkle tree circuit. An assignment used `=` where a constraining `<==` was needed, so the hash output was not constrained and a prover could make the circuit accept a fake Merkle path. The team exploited the bug themselves to drain and migrate the pools to fixed contracts before anyone else could.

The pattern, an **under-constrained signal**, is the most common class of circuit bug:

```circom
// BAD: assigns a value but adds no constraint; a malicious prover can choose any value
out <-- a * b;

// GOOD: assigns and constrains
out <== a * b;
```

Tools such as circomspect (Trail of Bits) flag `<--` assignments that lack a matching constraint, and Picus (Veridise) checks whether outputs are uniquely determined by inputs. See **zk-circuit-security**.

## Governance takeover, May 2023

Tornado Cash had a governance system using the TORN token. In May 2023, according to public post-mortems, an attacker submitted a proposal that looked like an earlier, legitimate one, but the proposal contract could be replaced through a self-destruct and redeploy at the same address. After the vote passed, the attacker swapped in malicious code that granted them a very large number of votes, taking control of governance. The pools' core contracts were immutable, which limited the damage to governance-controlled components such as the relayer registry and treasury. See **governance-attack-engineering**.

## Sanctions history

This is a summary of public legal events; it is not legal advice, and you should check current sources:

| Date | Event |
|---|---|
| August 2022 | The US Treasury's Office of Foreign Assets Control (OFAC) added Tornado Cash and many of its contract addresses to the Specially Designated Nationals (SDN) list, citing laundering, including by DPRK-linked actors |
| August 2022 onward | Front-ends, RPC providers and some validators restricted access; USDC in designated addresses was frozen by its issuer |
| May 2024 | A Dutch court convicted developer Alexey Pertsev of money laundering; he appealed |
| November 2024 | In *Van Loon v. Department of the Treasury*, the US Court of Appeals for the Fifth Circuit held that Tornado Cash's immutable smart contracts were not "property" that could be blocked under the relevant statute |
| March 2025 | Treasury removed Tornado Cash from the SDN list |
| August 2025 | In the US criminal trial of developer Roman Storm, the jury convicted on a charge of conspiring to operate an unlicensed money transmitting business and did not reach a verdict on other charges, according to press reports |

Engineering lessons, independent of legal views: front-ends, relayers and RPC endpoints are chokepoints; immutable contracts outlive front-ends; and honest users pay when a protocol cannot separate their funds from illicit ones (chapter 3).

## Worked example

_Not executed in this book's CI; verify locally._

Deploy and use a copy of `tornado-core` on a local Anvil chain.

1. Clone `tornadocash/tornado-core` (archived). Install its dependencies with the Node version the README specifies, and build circuits and contracts per the README. If the toolchain is too old, use a maintained fork for education, or port the circuit to circom 2.
2. Start `anvil` and deploy the ETH instance with a 1 ETH denomination using the repo's migration scripts, pointed at the local RPC.
3. Use the repo's CLI script to create a note, deposit, then withdraw to a second Anvil account. Record the transaction hashes.
4. Try to withdraw again with the same note; confirm the contract rejects the reused nullifier.
5. Take the withdrawal calldata, change only the `recipient`, and resubmit from a third account. Confirm the proof fails verification because the recipient is a public input.
6. Run circomspect on the repo's circuits and triage the warnings.

This uses only a local chain. Do not interact with deployed pools.

## Common pitfalls

- Leaving payout parameters out of the proof's public inputs.
- Accepting any Merkle root rather than a bounded history.
- Using `<--` without a matching constraint.
- Governance proposals whose code can change after the vote is counted.
- Assuming immutable pools protect users from front-end and relayer shutdowns.

## Exercises

1. Remove the recipient dummy constraint from a local copy of the withdraw circuit, recompile, and check with `snarkjs r1cs print` whether the signal still appears in constraints.
2. Write a Foundry test against a toy mixer showing that a proof with an unknown root is rejected.
3. Read the May 2023 governance post-mortems and write the exact on-chain sequence (deploy, propose, self-destruct, redeploy, execute).

## Further reading

- Tornado Cash core repository (archived): <https://github.com/tornadocash/tornado-core>
- Tornado Cash team, "Tornado.cash got hacked. By us.", Medium, 2019
- US Treasury press release on the Tornado Cash designation, August 2022, and the delisting notice, March 2025
- *Van Loon v. Department of the Treasury*, US Court of Appeals for the Fifth Circuit, 2024
- circomspect: <https://github.com/trailofbits/circomspect>
