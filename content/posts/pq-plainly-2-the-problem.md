---
title: "The Problem: Harvest Now, Decrypt Later"
date: 2026-10-08
draft: true
description: "Adversaries can copy ciphertext today and wait. Quantum computers threaten elliptic curves and RSA. AI may shrink other margins. The painful part is still migration. The fix is agility."
tags: ["post-quantum", "pqc", "cryptography", "quantum", "ethereum", "crypto-agility", "AI"]
series: ["Post-Quantum, Plainly"]
---

This is Part 2 of [Post-Quantum, Plainly](/posts/pq-plainly-1-the-locks/). Part 1 covered the lock designs. Part 3 is the [CLI walkthrough](/posts/pq-plainly-3-wallet-keys-cli/).

Don't trust, verify. Below are claims you can check against primary sources. Where I give someone's estimate, I label it as an estimate.

## Harvest now, decrypt later

Imagine someone photocopies every sealed letter you mail, stores the copies, and waits for a better letter opener.

That is **harvest-now-decrypt-later**. An adversary records ciphertext today (VPN traffic, TLS sessions, archived backups, messaging) and holds it until a future cryptanalytic break makes the plaintext cheap. Confidentiality has a long memory. Authentication failures are often immediate. Secrecy failures can be delayed by years.

If your data must still be secret in 2030 or 2035, waiting until quantum computers are headline news is waiting too long. The letters are already in the filing cabinet.

## What a quantum computer breaks

A large, fault-tolerant quantum computer running **Shor's algorithm** breaks the math under RSA and elliptic curves. That includes the ECDSA and X25519 lock designs from [Part 1](/posts/pq-plainly-1-the-locks/) that Bitcoin, Ethereum, SSH, and TLS still rely on.

Hashes are in better shape. Grover's algorithm speeds up brute force, which is why we talk about longer hash outputs, not about throwing hashes away. Hash-based signatures (SLH-DSA / SPHINCS+) lean on that relative strength.

Nobody can schedule "Q-day." Planning still needs a date. The Ethereum Foundation's Protocol cluster wrote on September 7, 2026 that it is aiming for Ethereum L1 to be quantum-resistant across execution, consensus, and data by **December 2029**, and that planning for Q-day as early as 2030 is a deliberately aggressive assumption.

## The new twist: AI-accelerated math

Quantum is not the only pressure. On October 7, 2026, Vitalik Buterin [posted](https://x.com/VitalikButerin/status/2107976296320106851) that we should take risks from AI-accelerated math seriously: "minimize our exposure to not just quantum-vulnerable cryptography, but also potentially AI-vulnerable cryptography."

His personal estimate, not a known attack: "there is a good chance that the concrete security of lattices will take serious hits from the next two years of AI math." He backs hash-only signatures for Ethereum's lean roadmap ("no lattices, no ML-DSA, no Falcon") and notes the harder problem is public-key encryption, which cannot be done with hashes alone.

I agree with the posture. Cryptanalysis only ever gets better. If AI compresses research timelines, assume margins erode faster than your roadmap assumed. That is why I argue for hybrids and swappable algorithms in [Assume the Math Will Move](/posts/crypto-agility-vitalik/), not for panic.

## A fresh reminder: Classic McEliece

We already have a non-AI example of margins moving. On September 29, 2026, Germany's BSI published [notes on Classic McEliece](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Crypto/Notes_Classic_McEliece.pdf?__blob=publicationFile&v=2). After new cryptanalysis, BSI says Classic McEliece "should currently no longer be used in new developments." It is also clear the results "do not enable a practical attack" on the recommended parameters. Hybrids still hold: a hybrid with a weakened Classic McEliece still gives at least the classical scheme's security. BSI's lesson matches mine: pay attention to crypto-agility, because cryptanalytic advances can never be ruled out.

## Deadlines are real for national systems

NSA announced on October 1, 2026 that under CNSSP 15, starting in **2027**, all new commercial National Security Systems must be capable of supporting quantum-resistant algorithms, and legacy systems that cannot are to be phased out by **2030**. ([NSA release](https://www.nsa.gov/Press-Room/Press-Releases-Statements/Press-Release-View/Article/4615285/nsa-announces-post-quantum-cryptography-measures-to-safeguard-national-security/)). Adversaries, NSA notes, are already employing harvest-now-decrypt-later strategies.

Whatever sector you are in, the work rhyme is the same: know where your crypto lives, make it swappable, practice the swap.

## Ethereum wallets: the key is the address

On Ethereum today, a regular wallet (an externally owned account, or EOA) is permanently tied to one secp256k1 key. The address is derived from the public key. Once you send a transaction, that public key is exposed onchain. A future break of the curve math would let an attacker derive the private key from that exposed public key and spend the funds.

Vitalik's advice in the same October 7 post: if it is not difficult, keep funds in addresses that have never sent a transaction; if it is easy, do it. And: do not scramble today. "**But be careful about migrations; I personally have lost more money in botched migrations than I have lost in all hacks combined.**"

### Hegotá and Frames

The next upgrade after Glamsterdam is Hegotá. [ethereum.org](https://ethereum.org/roadmap/hegota/) lists expected mainnet in **Q2 2027**, date not yet confirmed. Its execution-layer headliner includes **Frame Transactions** ([EIP-8141](https://eips.ethereum.org/EIPS/eip-8141)), whose motivation states it provides "a native off-ramp from the elliptic curve based cryptographic system used to authenticate transactions today, to post-quantum (PQ) secure systems."

The EF Protocol post is blunt: "Hegotá is not the PQ fork; it is the fork that decides whether the PQ forks happen on time." The north star remains a quantum-resistant L1 by December 2029. Frames are how accounts stop being welded to one signature scheme.

## Migrations are where this actually fails

Algorithms rarely fail in production first. Operations fail first.

The certificate nobody tracked. The hardcoded cipher list three dependencies deep. The rotation script that has never run. The allowlist that still points at the old address. On an EOA, "rotating" means a new wallet and a fund move; every approval and allowlist tied to the old address breaks. That is the botched-migration risk Vitalik is naming, and it is the risk I see in infrastructure work every year.

## The fix is agility

You cannot pick a forever lock. You can build a door that accepts new locks.

- Prefer **hybrids** so one break is not game over.
- Keep algorithms in **config**, not hardcoded in five repos.
- Rehearse rotation until it is boring.
- For chains and wallets, prefer designs where the **key is a setting** and the address stays put. That is what Frames aim to make native.

[Part 3](/posts/pq-plainly-3-wallet-keys-cli/) shows this on a local chain with Foundry: create a key, sign, send, feel the EOA migration pain, then rotate an owner on a tiny smart wallet without changing its address.

*Views are my own and do not represent my employer. Nothing here is financial advice.*
