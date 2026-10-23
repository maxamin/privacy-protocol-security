# Privacy Protocol Audit Checklist

This checklist covers mixers, privacy pools and shielded pools across four layers: circuits, contracts, clients and compliance design. Each item names a way to verify it with a tool, a test, or a documented artefact, so findings can be reproduced.

## Scoping

1. Design family: fixed-denomination mixer, privacy pool with association sets, shielded pool, or private execution.
2. Proof system and setup: Groth16 with a ceremony, PLONK with a universal setup, Halo 2 or STARK without trusted setup.
3. Circuit language and libraries (circom and circomlib, Noir, halo2), with versions.
4. Admin powers over verifiers, trees, ASP roots, relayer registries and fees.
5. Client components: proving, note storage, relayer selection, RPC handling.

## Circuit checklist

| # | Check | How to verify |
|---|---|---|
| Z1 | No under-constrained signals (`<--` without matching constraint) | circomspect; Picus for output uniqueness; manual review |
| Z2 | Nullifier depends on a secret and is unique per note | Read derivation; test two different notes give different nullifiers |
| Z3 | Merkle path indices constrained to bits; hash inputs ordered correctly | Negative test with non-bit index (chapter 3 exercise) |
| Z4 | All payout parameters (recipient, relayer, fee, refund) are public inputs that affect constraints | `snarkjs r1cs print`; modify and verify proof fails |
| Z5 | Value conservation and range checks on every amount | Wrap-around witness test (chapter 4) |
| Z6 | Same leaf used for pool and association set proofs | Constraint review; negative witness |
| Z7 | Hash functions and parameters match between circuit, contract and client | Cross-implementation test vectors |
| Z8 | Trusted setup ceremony documented and verifiable | `snarkjs zkey verify` against published transcript |

## Contract checklist

| # | Check | How to verify |
|---|---|---|
| C1 | Public inputs range-checked below the scalar field modulus | Test with `input + p`; verifier must reject or contract must check |
| C2 | Nullifier marked spent before external calls; double spend reverts | Foundry test; Slither reentrancy detectors |
| C3 | Root accepted only from bounded history (and ASP root only from authorised source) | Unit test with unknown root |
| C4 | Deposit amount checked exactly (fixed pools) or committed value matches transferred value | Test with fee-on-transfer token mock |
| C5 | Verifier contract matches the circuit's verification key | Regenerate verifier from `zkey` and diff |
| C6 | Tree insertion correct and gas-bounded; full tree handled | Fuzz insertions; test at capacity |
| C7 | Admin powers minimal; verifier and tree immutable or timelocked | Role table; on-chain reads |
| C8 | Governance proposals cannot change code after approval | Check for `selfdestruct`/`CREATE2` redeploy patterns (chapter 2) |
| C9 | Public exit path if ASP or relayers stop | Test ragequit or public withdrawal |

## Client and privacy checklist

| # | Check | How to verify |
|---|---|---|
| P1 | Secrets generated with a CSPRNG | Code review; test that `Math.random` or similar is absent |
| P2 | Notes and viewing keys stored encrypted; not sent to servers | Network capture during deposit and withdrawal |
| P3 | Proving done locally or with documented trust | Code review; network capture |
| P4 | Default gas settings; warning on linked withdrawal addresses | UI test with a linked address |
| P5 | Multiple relayers and private RPC options | Configuration review |
| P6 | Anonymity set shown to the user before withdrawal | UI review |

## Compliance design checklist

| # | Check | Evidence |
|---|---|---|
| K1 | Association set criteria public and reproducible | Script that rebuilds the root (chapter 3 exercise) |
| K2 | ASP key management and update history | Key policy; on-chain update events |
| K3 | Viewing key disclosure scoped; users told disclosure is permanent | Docs and UI text |
| K4 | Viewing key stores (if any) have breach plans | Security docs |
| K5 | Legal review current; front-end terms and sanctions screening documented | Counsel memo (not an engineering check, but in scope for risk) |

## Using AI assistants

An LLM can flag suspicious `<--` assignments, draft negative witnesses, or summarise a protocol specification. Its output is a hypothesis. A circuit finding counts only when a tool confirms it: circomspect or Picus output you have triaged, a witness that satisfies the constraints when it should not, or a Foundry test that double-spends. If a model says "this circuit is sound", that is not evidence. See **ai-assisted-audit-playbook** and **zk-circuit-security**.

## Worked example

_Not executed in this book's CI; verify locally._

Check C1 (field aliasing) on a toy verifier.

1. Export a Groth16 verifier from the chapter 3 circuit with `snarkjs zkey export solidityverifier`.
2. Write a `ToyPool` that stores spent nullifiers as `mapping(uint256 => bool)` and calls the verifier, with no range check on `nullifierHash`.
3. In a Foundry test, withdraw with a valid proof, then call again with `nullifierHash + r` (r is the BN254 scalar field modulus) and the same proof.
4. Observe whether the verifier accepts it. Current snarkjs-generated verifiers check that public inputs are below the field modulus, so the call should revert inside the verifier. Record what you observed.
5. Remove that check from a copy of the verifier (for teaching only) and show the second withdrawal succeeds: the contract sees a "new" nullifier.
6. Fix in the pool too: `require(nullifierHash < SNARK_SCALAR_FIELD)`. Defence in depth means neither layer should rely on the other.

## Common pitfalls

- Auditing contracts without the circuits, or circuits without the verifier.
- Trusting a verifier contract without regenerating it from the verification key.
- Skipping client code, where secrets and notes are handled.
- Treating compliance features as outside security scope.
- Reporting AI-suggested circuit issues without a witness or tool result.

## Exercises

1. Run circomspect on an open-source circom privacy project and triage every warning as real, false positive, or needs review.
2. Write cross-implementation test vectors for Poseidon (circuit, Solidity, JavaScript) and check they agree.
3. Write a disclosure report for one toy finding following a bug bounty template, including the witness or test that proves it.

## Further reading

- circomspect: <https://github.com/trailofbits/circomspect>
- Picus: <https://github.com/Veridise/Picus>
- snarkjs: <https://github.com/iden3/snarkjs>
- 0xPARC ZK bug tracker: <https://github.com/0xPARC/zk-bug-tracker>
- Immunefi: <https://immunefi.com>
- Security Alliance safe harbor: <https://securityalliance.org>
