# Privacy Protocol Mechanisms and the Threat Model

Public blockchains publish every transfer. Privacy protocols try to break the link between who put value in and who took it out, or hide amounts and parties entirely. They do this with commitments, Merkle trees, nullifiers and zero-knowledge proofs. The security questions are unusual: a privacy protocol can lose funds like any contract, but it can also fail quietly by leaking who is who, or fail legally by becoming unusable for honest users. This chapter sets out the shared mechanism and a threat model with three parts: soundness, privacy and compliance.

## The shared mechanism

Almost every design in this book uses the same core, first popularised by Zerocash and then by Tornado Cash on Ethereum:

1. **Deposit.** The user picks secrets, computes a **commitment** (a hash of the secrets, and in shielded pools also the amount and owner key), and deposits funds with the commitment. The contract inserts the commitment into a Merkle tree.
2. **Withdraw or spend.** Later, possibly from a fresh address, the user proves in zero knowledge: "I know the secrets behind some commitment in the tree with root R, and here is its **nullifier**." The nullifier is a deterministic hash of a secret, so each commitment can be spent only once, but the nullifier cannot be linked to the commitment without the secret.
3. **Contract checks.** The contract verifies the proof against a known root, checks the nullifier is unused, records it, and pays out.

```mermaid
flowchart LR
  D[Deposit: commitment C = H(nullifier, secret)] --> T[(Merkle tree of commitments)]
  T --> R[Root R]
  W[Withdraw: proof of C in R, nullifier hash N] --> V{Verifier + checks}
  R --> V
  V -->|N unused| P[Pay recipient; mark N spent]
  V -->|N used| X[Reject]
```

## Design families

| Family | What is hidden | Amounts | Examples | Notes |
|---|---|---|---|---|
| Fixed-denomination mixer | Link between deposit and withdrawal | Fixed per pool | Tornado Cash | Simple; anonymity set is per pool |
| Privacy pool with association sets | Link, plus a proof of membership in an approved subset | Variable or fixed | Privacy Pools (0xbow) | Adds a compliance signal |
| Shielded pool (UTXO notes) | Sender, receiver, amount inside the pool | Variable, hidden | Zcash (Sapling, Orchard), Railgun, Aztec | Viewing keys for selective disclosure |
| Private L2 / execution | Contract state and calls | Hidden | Aztec network | Privacy at the execution layer |
| Stealth addresses | Receiver identity | Public | ERC-5564 | No mixing; hides who received |

## Threat model

**Soundness: can someone take money that is not theirs?**

| Threat | Cause |
|---|---|
| Forged proof | Under-constrained circuit; broken trusted setup; verifier bug |
| Double spend | Nullifier not bound to the commitment; nullifier aliasing via unchecked field inputs |
| Front-running a withdrawal | Recipient, relayer or fee not bound as public inputs |
| Fake root | Contract accepts a root that was never in its history |
| Admin or upgrade abuse | Owner can change verifier, tree or pool contracts |

**Privacy: can an observer link deposits to withdrawals or learn amounts?**

| Threat | Cause |
|---|---|
| Small anonymity set | Few deposits; unique amounts; short time in pool |
| Behavioural linkage | Same address, gas settings, timing, or funding path on both sides |
| Metadata leakage | RPC providers, relayers and front-ends see IP addresses and timing |
| Weak randomness | Predictable secrets let others compute nullifiers or commitments |
| Key leakage | Viewing keys shared too widely or stored insecurely |

**Compliance and liveness: can honest users keep using it?**

| Threat | Cause |
|---|---|
| Sanctions or legal action | Protocol, front-end or developers targeted; see chapter 2 |
| Tainted-fund contagion | Honest withdrawals treated as suspicious by exchanges |
| Front-end and relayer shutdown | Users unable to build proofs or pay gas privately |
| Governance capture | Malicious proposals changing contracts |

Soundness failures lose funds. Privacy failures lose users' safety. Compliance failures make the protocol useless to the people who most need lawful privacy. A good audit covers all three.

## Responsible framing

Privacy has legitimate uses: salaries, donations to at-risk groups, business confidentiality, personal safety. It is also used to launder stolen funds; public reporting has linked large thefts, including those attributed to DPRK-linked groups, to mixers (see **dprk-crypto-threat-operations**). This book is about building and auditing privacy systems so they are sound, private for honest users, and compatible with lawful compliance. Exercises run on local chains with toy contracts and your own data.

## Worked example

_Not executed in this book's CI; verify locally._

Build and run a minimal commitment-nullifier circuit with circom and snarkjs.

1. Install circom 2 and snarkjs (`npm install -g snarkjs`), and add circomlib for Poseidon.
2. Write `commit.circom`:

```circom
pragma circom 2.1.6;
include "circomlib/circuits/poseidon.circom";

template Commit() {
    signal input nullifier;
    signal input secret;
    signal output commitment;
    signal output nullifierHash;
    component c = Poseidon(2); c.inputs[0] <== nullifier; c.inputs[1] <== secret;
    component n = Poseidon(1); n.inputs[0] <== nullifier;
    commitment <== c.out;
    nullifierHash <== n.out;
}
component main = Commit();
```

3. Compile: `circom commit.circom --r1cs --wasm --sym -l node_modules`.
4. Do a local Groth16 setup for testing only (`snarkjs powersoftau new bn128 12 pot.ptau`, contribute, `prepare phase2`, then `snarkjs groth16 setup`). Generate a proof for chosen inputs and verify it with `snarkjs groth16 verify`.
5. Run `snarkjs r1cs info commit.r1cs` and record the constraint count; later chapters add the Merkle path.

## Common pitfalls

- Auditing only for fund loss and ignoring privacy and compliance failure modes.
- Treating a ZK proof as automatically sound; the circuit and verifier can be wrong.
- Assuming the anonymity set is the total number of deposits ever made.
- Ignoring off-chain metadata: RPC, relayer and front-end logs.
- Using non-cryptographic randomness for secrets in client code.

## Exercises

1. For one privacy protocol, fill in the three threat tables with protocol-specific causes and mitigations from its docs.
2. Extend `commit.circom` so the commitment also includes an amount, and write a test input where changing the amount changes the commitment.
3. List every party that sees metadata when a user withdraws through a hosted front-end and a relayer.

## Further reading

- Ben-Sasson et al., "Zerocash: Decentralized Anonymous Payments from Bitcoin", IEEE S&P 2014
- Tornado Cash core repository (archived): <https://github.com/tornadocash/tornado-core>
- circom documentation: <https://docs.circom.io>
- snarkjs: <https://github.com/iden3/snarkjs>
- **zk-circuit-security** in this wiki
