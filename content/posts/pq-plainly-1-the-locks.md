---
title: "The Locks: Post-Quantum Algorithms in Plain English"
date: 2026-10-08T10:13:00-04:00
draft: false
description: "Algorithms are lock designs. Keys fit those designs. Crypto-agility means swapping the lock without replacing the door. A plain-English tour of hashes, elliptic curves, lattices, hash-based signatures, and code-based crypto."
tags: ["post-quantum", "pqc", "cryptography", "hashes", "lattices", "PKI", "crypto-agility"]
series: ["Post-Quantum, Plainly"]
seriesPart: 1
---

I think Vitalik is one of the greats in the crypto industry. My fear for Ethereum is that it sounds so complicated that the general user won't be able to understand what's going on.

{{< plain >}}
## In plain English

Encryption is a lock on your data (and here, your data is access to your tokens).

We have two main existential threats on the horizon: quantum and AI. AI is here, and quantum is quickly approaching. No AI has broken any of the related locks yet. The concern is that AI can speed up the math research that would find the weak spots in the lock design.

How could we defend against this threat? Use two different locks, so a thief has to pick both of them:

- One lock that is older and proven (like ECDSA, which Ethereum wallets use today).
- One that is quantum-resistant (like ML-DSA or hash-based SLH-DSA).

The ability to quickly swap your locks out as one of them becomes weak is very important. This ability is called **crypto-agility**.

When algorithms are being described, they are describing the lock design. It's the math that decides how hard it is for these threats (quantum and AI) to pick. Your key is exactly what it sounds like: the key that unlocks the lock.

If this interests you, I'm putting together a breakdown of some of these algorithms, then why the problems exist, and then what we can functionally and tactically do to protect our wallets and become cryptographically agile.
{{< /plain >}}

---

This is Part 1 of three. Part 2 (coming soon) covers why this matters now. Part 3 (coming soon) is a command-line walkthrough of wallet keys.

Privacy is a right. Your keys are your data. Math beats promises. Don't trust, verify. Apply that same standard to the new post-quantum algorithms: understand what they do before you trust them with anything.

## The analogy

Think of cryptography as locks on doors.

- An **algorithm** is a lock *design*: the shape of the pins.
- A **key** is the metal that fits that design.
- **Crypto-agility** means you can change the lock design without tearing out the door. New algorithm, same system.
- A **hybrid** means two locks on the same door. A thief has to open both.

I use those words the same way through this series. For the operational version, see [Assume the Math Will Move](/posts/crypto-agility-vitalik/).

## Two jobs: key exchange and signatures

Public-key crypto does two different jobs.

**Key exchange** is how two strangers agree on a shared secret without meeting. That secret then locks their messages. TLS does this when you visit a website. SSH does it when you log into a server.

**Signatures** prove it was you. You hold a private key. Everyone can check a matching public key. Bitcoin, Ethereum, and software updates all rely on this.

A **hash function** can help build signatures. It cannot do key exchange by itself. That limit is mathematical. Keep it in mind when someone says "just use hashes for everything."

## Hash functions: a one-way blender

A hash function is a blender with three habits:

1. Same input, same fingerprint.
2. One tiny change scrambles the result.
3. You cannot run it backwards.

We use hashes for integrity checks, blockchain addresses, and as building blocks inside bigger schemes. SHA-256 and SHA-3 are common examples. Quantum computers do not break hashes the way they break RSA and elliptic curves; they mostly push us toward longer outputs. More on that in Part 2.

## What we use today: elliptic curves

Most of the internet and most blockchains still run on elliptic-curve cryptography.

- **ECDSA** and **Ed25519** are signature schemes. Bitcoin and Ethereum wallets use secp256k1 with ECDSA. SSH often uses Ed25519.
- **X25519** is key exchange. TLS 1.3 and modern SSH lean on it.

These are fast and the keys are small. That is why they won. They are also on the wrong side of a future large quantum computer. Excellent locks against today's thieves. Not the locks we want for the next few decades alone.

## Lattices: the main new federal standards

Lattice crypto hides secrets in high-dimensional grids. The honest party knows a shortcut. Everyone else sees a haystack.

NIST standardized two lattice schemes in August 2024:

- **ML-KEM** ([FIPS 203](https://csrc.nist.gov/pubs/fips/203/final)): key encapsulation. The workhorse for hybrid TLS and SSH (for example `X25519MLKEM768`).
- **ML-DSA** ([FIPS 204](https://csrc.nist.gov/pubs/fips/204/final)): signatures.

Keys and signatures are larger than elliptic curves, but still practical for most network protocols. Lattices are the default post-quantum choice for key exchange today. They are also the family getting nervous attention from people watching AI-accelerated math; that is a Part 2 story, not a known break.

## Hash-based signatures: boring on purpose

Hash-based signatures build signing out of the blender. No lattices. No elliptic curves. Just hashes and careful bookkeeping.

**SLH-DSA** ([FIPS 205](https://csrc.nist.gov/pubs/fips/205/final)) is the federal standard, based on SPHINCS+. Related ideas include WOTS (Winternitz one-time signatures). Signatures are bigger and slower than ML-DSA. That is the trade for relying on almost nothing beyond hash security.

### A one-time hash signature (Lamport)

Here is the simplest version. Too large for daily use; perfect for intuition.

1. Alice picks a pile of random secret coins (private key).
2. She blends each coin into a fingerprint and publishes the fingerprints (public key).
3. To sign a message, she blends the message, looks at its bits, and reveals the secret coins those bits select.
4. Bob blends the revealed coins and checks that the fingerprints match Alice's list.

![Cartoon: how a hash-based signature works](/images/hash-signature-cartoon.jpg)

Use those secrets once. Sign a second message and you leak overlap; a forger can stitch pieces together. SPHINCS+ (and SLH-DSA) wrap a tree of one-time keys so you can sign many times safely. The cartoon is the Lamport core; the standard is that core plus scaffolding.

Hashes can do this signature job. They still cannot do key exchange alone.

## Code-based: Classic McEliece and HQC

Code-based crypto hides a message by adding noise to an error-correcting code. The secret holder can clean the noise.

- **Classic McEliece** dates to 1978 roots, is conservative, and has very large public keys with small ciphertexts. A specialist tool.
- **HQC** is a newer code-based key encapsulation scheme. NIST selected it as an additional algorithm, which matters if you want a second family for hybrids.

Different math from lattices is the point. Diversity of lock designs avoids a single failure mode.

## Quick comparison

| Family | For | Relies on | Size | Status |
| --- | --- | --- | --- | --- |
| Elliptic curves (ECDSA, Ed25519, X25519) | Signatures and key exchange today | Curve discrete log | Small | Dominant; not quantum-safe |
| Lattices (ML-KEM, ML-DSA) | Key exchange and signatures | Lattice problems | Medium | FIPS 203 / 204 |
| Hash-based (SLH-DSA / SPHINCS+, WOTS) | Signatures only | Hash security | Large signatures | FIPS 205 |
| Code-based (Classic McEliece, HQC) | Key exchange | Hard decoding | McEliece: huge public keys; HQC: moderate | McEliece niche; HQC additional NIST KEM |

## What to remember

Lock designs are algorithms. Keys fit them. Hybrids put two designs on one door. Agility lets you swap designs without rebuilding the house. Key exchange and signatures are different jobs. Hashes are a one-way blender: great for fingerprints and for certain signatures, useless alone for agreeing on a secret.

Next up, coming soon: the problem those locks are meant to solve. Then Part 3 puts wallet keys on the command line so you can feel what "changing the lock" actually breaks.

*Views are my own and do not represent my employer. Nothing here is financial advice.*
