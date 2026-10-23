# Anonymity Sets and Deanonymisation Heuristics

A zero-knowledge proof hides which deposit a withdrawal came from. It does not hide everything around it: the addresses, amounts, timing, gas settings and funding paths of the transactions themselves. Researchers have shown that a large share of mixer users leak enough of this to be linked. This chapter explains how to measure the anonymity a protocol really gives, the heuristics used to break it, and what protocol and wallet designers can do.

## Measuring anonymity

The **anonymity set** of a withdrawal is the set of deposits it could plausibly have come from. Its naive size is the number of deposits in the pool before the withdrawal, minus those already withdrawn. A better measure accounts for probabilities: if some candidates are much more likely than others, the effective set is smaller.

A common measure is Shannon entropy over the candidate probabilities, reported as an effective set size `2^H`:

```python
import math
def effective_set(probs):
    probs = [p for p in probs if p > 0]
    s = sum(probs); probs = [p / s for p in probs]
    H = -sum(p * math.log2(p) for p in probs)
    return 2 ** H

print(effective_set([1] * 100))                 # 100 equal candidates -> 100
print(effective_set([50] + [1] * 99))           # one strong suspect -> far fewer
```

Factors that shrink the effective set:

| Factor | Why |
|---|---|
| Few deposits in the pool | Small base set |
| Short time between deposit and withdrawal | Only recent deposits are candidates if timing patterns are used |
| Unique amounts (in variable-amount pools) | Amount matching links deposit and withdrawal |
| Association sets (chapter 3) | Restrict the set to approved deposits |
| Earlier deanonymised users | Each linked deposit removes a candidate for everyone else |

The last factor matters: privacy is collective. When some users are careless, everyone's anonymity set shrinks.

## Heuristics from the literature

Wang et al., "On How Zero-Knowledge Proof Blockchain Mixers Improve, and Worsen User Privacy" (ACM Web Conference 2023), studied Tornado Cash and other mixers and described heuristics including the following. Similar ideas appear in Béres et al., "Blockchain is Watching You: Profiling and Deanonymizing Ethereum Users" (2021), and in the open-source Tutela tool.

| Heuristic | Idea |
|---|---|
| Address match | Same address deposits and withdraws |
| Unique gas price | A distinctive, manually set gas price used on both sides |
| Linked addresses | Deposit and withdrawal addresses have interacted directly or through a common counterparty |
| Multi-denomination | A user splits a sum into several denominations and withdraws the same combination shortly after |
| Timing | A withdrawal soon after a deposit, when few other deposits happened in between |
| Fee and relayer patterns | Same relayer or fee settings across linked transactions |
| Mining or reward claims | Claiming protocol rewards that reveal deposit duration |

These heuristics produce **candidate links with varying confidence**, not proof. False positives are common, and researchers usually validate on ground truth such as self-identified addresses.

## Off-chain leakage

Not all leakage is on-chain:

- **RPC providers** see the IP address that submits a deposit and a withdrawal.
- **Relayers** see withdrawal requests and their timing.
- **Front-ends** see browser fingerprints and may log notes if built carelessly.
- **Note storage**: notes in cloud-synced files or browser storage can be stolen, which loses both privacy and funds.

Protocol designers cannot control all of these, but they can ship clients that support local proving, multiple relayers, Tor or private RPC, and clear warnings.

## Design responses

| Response | Who | Effect |
|---|---|---|
| Fixed denominations | Protocol | Removes amount matching, at a usability cost |
| Delayed or batched withdrawals | Protocol or wallet | Weakens timing analysis |
| Relayer networks with fixed fee schedules | Protocol | Avoids fresh-address gas funding links |
| Default gas settings from the wallet | Wallet | Removes unique gas fingerprints |
| Warnings when withdrawing to a linked address | Wallet | Prevents the simplest mistakes |
| Anonymity set display | Front-end | Lets users wait for a larger set |

## Ethics and safety

Deanonymisation research protects users by showing them what leaks. Applied to real people, the same techniques can expose victims of abuse, journalists and dissidents. Do exercises on your own data or on a local chain. When studying public data, publish aggregate results, not lists of identified individuals, and follow your institution's ethics review if one applies.

## Worked example

_Not executed in this book's CI; verify locally._

Measure linkage heuristics on a synthetic mixer you control.

1. Deploy a toy fixed-denomination mixer (from chapters 2 or 3) on Anvil.
2. Write a Python simulation with web3.py that creates 200 users. Make 70% "careful" (fresh withdrawal address, default gas price, random delay) and 30% "careless" (one of: same address, unique gas price, withdrawal within two blocks, or a direct transfer between deposit and withdrawal addresses).
3. Run deposits and withdrawals on Anvil, recording ground truth in a local CSV.
4. Implement four heuristics (address match, unique gas price, timing, direct link) that read only chain data via `eth_getLogs` and transaction receipts.
5. Compute precision and recall of each heuristic against ground truth, and the average effective anonymity set for careful users before and after removing linked careless users.
6. Plot how careful users' effective set shrinks as the careless share rises.

## Common pitfalls

- Reporting naive anonymity set size as the privacy level.
- Treating heuristic links as proof of identity.
- Ignoring off-chain metadata at RPCs, relayers and front-ends.
- Variable-amount pools without guidance on amount splitting.
- Publishing identified individuals from research on public data.

## Exercises

1. Extend the simulation with a multi-denomination heuristic and measure its false-positive rate.
2. Implement a wallet-side check that warns when a withdrawal address has any transaction history linked to the deposit address within two hops.
3. Compute entropy-based effective set sizes for withdrawals in your simulation and compare them to naive set sizes.

## Further reading

- Wang et al., "On How Zero-Knowledge Proof Blockchain Mixers Improve, and Worsen User Privacy", ACM Web Conference (WWW) 2023
- Béres et al., "Blockchain is Watching You: Profiling and Deanonymizing Ethereum Users", 2021
- Wu et al., "Tutela: An Open-Source Tool for Assessing User-Privacy on Ethereum and Tornado Cash", 2022 (arXiv)
- Díaz, Seys, Claessens, Preneel, "Towards Measuring Anonymity", PET 2002
