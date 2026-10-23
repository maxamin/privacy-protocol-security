# Glossary

- **Commitment:** a hash that binds to secret values without revealing them; inserted into a Merkle tree on deposit.
- **Nullifier:** a value derived from a note's secret, published on spend so the note cannot be spent twice.
- **Anonymity set:** the deposits a withdrawal could plausibly have come from.
- **Effective anonymity set:** anonymity set size adjusted for unequal probabilities, for example `2^entropy`.
- **Mixer:** a pool that breaks the link between deposits and withdrawals, usually with fixed denominations.
- **Privacy pool:** a mixer variant where withdrawals also prove membership in an association set.
- **Association set:** a published subset of deposits (for example, those not linked to known hacks).
- **ASP (association set provider):** the party that builds and publishes association sets.
- **Shielded pool:** a pool where transfers between notes hide sender, receiver and amount.
- **Note:** a unit of private value with an owner, amount and randomness.
- **Viewing key:** a key that reveals account activity without allowing spending.
- **Turnstile:** public accounting of value moving between shielded pools, to detect counterfeiting.
- **Under-constrained signal:** a circuit value that is assigned but not constrained, letting a prover choose it.
- **Relayer:** a service that submits withdrawals and pays gas for a fee, so users need no linked gas funding.
