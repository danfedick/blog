---
title: "Assume the Math Will Move: Crypto-Agility After Vitalik's AI Warning"
date: 2026-10-08
draft: false
description: "Vitalik Buterin says AI-accelerated math could shrink crypto security margins, including for lattices. I agree, and I think the answer is crypto-agility: hybrids, swappable algorithms, and migrations you have rehearsed before you need them."
tags: ["post-quantum", "crypto-agility", "pqc", "cryptography", "openssh", "tls", "ai", "key-management"]
---

On October 7, Vitalik Buterin [posted a long note on X](https://x.com/VitalikButerin/status/2107976296320106851) that I keep rereading. He was responding to Justin Drake's call for a crypto ["bunker mode"](https://cointelegraph.com/news/justin-drake-urges-crypto-bunker-mode-as-ai-could-break-wallet-security-within-months), and his opening line sets the tone: "I don't recommend anyone scramble to move their funds to new wallets today. But we should take the risks to cryptography from AI-accelerated math seriously, and minimize our exposure to not just quantum-vulnerable cryptography, but also potentially AI-vulnerable cryptography."

That is a sober take, and most of it is right. Where I land is one word: agility. If the math under our feet can move faster than we planned, the winning property of a system is how quickly and safely it can change what it runs on. Picking the perfect algorithm matters less than that.

## In plain English

Encryption is a lock on your data. Every lock can eventually be picked, and the tools for picking digital locks keep getting better, now with help from AI. So the smart move is not to hunt for one perfect lock. Use two different locks at once, so a thief has to beat both, and set things up so you can swap in a new lock quickly when an old one starts to look weak. That ability to swap is called crypto-agility, and the rest of this post explains how to build it.

This is my first post here, so a quick note on where I come from. I'm a cypherpunk at heart. Privacy is a right, your keys are your data, and math beats promises. Don't trust, verify. That applies to Vitalik's argument too, so let's take it seriously.

## The threat: AI as cryptanalyst

Vitalik's threat model is clean. Factoring naively takes 2^(n/2) time, but decades of work produced the number field sieve and cut that to 2^O(n^(1/3)), "which is why RSA keys and signatures need to be ~400 bytes (and not 64 bytes)." His question: what if there are similar "skeletons in the closet" for elliptic curves and lattices that humans haven't found, but AI soon will?

He goes further than most people are willing to: "So far most people have been in the mode of thinking 'elliptic curves broken, hashes safe, lattices safe'. But there is a good chance that the concrete security of lattices will take serious hits from the next two years of AI math." He also notes AI is another reason ECDSA "might fall even faster than expected."

To be clear about what this is: an informed estimate, which he labels as his "own personal views." No known attack breaks ML-KEM or ML-DSA today. But I don't need a known attack to act. Cryptanalysis only ever gets better, and the record shows margins shrinking in steps, not smoothly. If AI compresses research timelines, the rational move is to assume your margins erode faster than your roadmap assumed, and to plan for it.

We already have a fresh example that has nothing to do with AI. On September 29, Germany's BSI published [notes on Classic McEliece](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Crypto/Notes_Classic_McEliece.pdf?__blob=publicationFile&v=2), one of the oldest public-key schemes around, saying it "should currently no longer be used in new developments" after new cryptanalysis lowered key-recovery estimates. BSI is clear the results "do not enable a practical attack" on the recommended parameters. Still, a scheme with nearly five decades of scrutiny got repriced in a few months. BSI drew the same lesson I do: the results "underline how important it is to pay attention to crypto-agility during the migration because cryptanalytic advances can never be ruled out."

## Agility beats monoculture

Here is where I add a layer to Vitalik's view rather than disagree with it. For Ethereum, he backs the lean roadmap's "hash-only" direction: "no lattices, no ML-DSA, no Falcon," with signatures that are "all hash-based, either WOTS or SPHINCS-." For lattices you can't avoid, he suggests being "much more paranoid on param sizes," and floats multiplying key sizes by 10 for anything meant to be long-term secure.

For signatures, that reasoning is strong, and the rest of the world isn't starting from zero. Hash-based signatures already have a federal standard: [FIPS 205 (SLH-DSA)](https://csrc.nist.gov/pubs/fips/205/final), based on SPHINCS+, published August 13, 2024 alongside [FIPS 203 (ML-KEM)](https://csrc.nist.gov/pubs/fips/203/final) and [FIPS 204 (ML-DSA)](https://csrc.nist.gov/pubs/fips/204/final).

But Vitalik himself names the limit: "The bigger challenge is for *public-key encryption*," and there are long-standing results showing it "cannot be done with hashes alone." Key exchange needs some structured trapdoor: groups, lattices, codes, or something newer. So for most of us running TLS, SSH, VPNs, and messaging, hash-only is not an option for the part that protects data in transit.

That's why I won't bet everything on one family, lattice or anything else. Monoculture is the mistake we made with RSA and ECC, and it's why this migration is so painful. The hedge is twofold:

1. **Hybrids.** Combine a post-quantum scheme with a classical one so an attacker has to break both. BSI says this approach "has proven its worth": a hybrid with a weakened Classic McEliece still gives at least classical security, and ML-KEM, FrodoKEM, and HQC aren't affected by the new results. HQC matters here because it's code-based, a different family from the lattice schemes.
2. **Swappable crypto.** Design so that changing an algorithm is a config change and a rollout, not a rewrite. If lattices take a hit, you want to raise parameters or switch families in weeks.

## Migrations are where PQC actually fails

The line from Vitalik's post I'd put on a poster: "be careful about migrations; I personally have lost more money in botched migrations than I have lost in all hacks combined."

Anyone who has run real infrastructure knows exactly what he means. Algorithms rarely fail in production. Operations fail. The certificate nobody knew about expires. A hardcoded cipher list sits in a library three dependencies deep. A rotation script runs for the first time during an incident. A key gets moved and its backup doesn't.

Deadlines are now real for a lot of us. NSA [announced on October 1](https://www.nsa.gov/Press-Room/Press-Releases-Statements/Press-Release-View/Article/4615285/nsa-announces-post-quantum-cryptography-measures-to-safeguard-national-security/) that under CNSSP 15, "starting in 2027, all new commercial NSS must be capable of supporting quantum-resistant algorithms," and legacy systems that can't are "to be phased out by 2030." Whatever sector you're in, the work is the same: know where your crypto lives, make it swappable, and practice the swap until it's boring.

## How-To: crypto-agility you can start this week

Every command below is real and was checked against official docs or release notes. Try them on lab systems first.

### 1. Inventory: find out what you're actually negotiating

You can't migrate what you can't see. Start with TLS. With OpenSSL 3.5 or later, `s_client` reports the negotiated key exchange group:

```bash
openssl version
echo | openssl s_client -connect example.com:443 -servername example.com -brief 2>&1 \
  | grep "Negotiated TLS1.3 group"
```

A hybrid endpoint shows `Negotiated TLS1.3 group: X25519MLKEM768`. To check whether a server supports the hybrid at all, offer only that group with `-groups X25519MLKEM768`. If the handshake fails, it doesn't.

For SSH, list the host key types your servers present, and see which key exchange your client actually picks:

```bash
ssh-keyscan host.example.com
ssh -v host.example.com exit 2>&1 | grep "kex: algorithm"
```

Script these across your fleet and write the results down. That list is your migration map.

### 2. OpenSSH: confirm hybrid key exchange, then test hybrid signatures

Check your version with `ssh -V`. Since [OpenSSH 10.0](https://www.openssh.com/releasenotes.html), the hybrid `mlkem768x25519-sha256` is the default key exchange. If you manage `KexAlgorithms` explicitly, make sure it leads the list. In `ssh_config` or `sshd_config`, a leading `^` places your choice at the head of the default set:

```
KexAlgorithms ^mlkem768x25519-sha256
```

`ssh -Q kex` lists what your build supports. Since 10.1 the client warns when a connection negotiates non-post-quantum key agreement, and 10.6 adds the same `WarnWeakCrypto` logging on the server, on by default. Leave those warnings on. They're free inventory.

OpenSSH 10.6, released October 6, enables the hybrid `ssh-mldsa44-ed25519` signature algorithm (ML-DSA-44 plus Ed25519). Try it somewhere low-stakes:

```bash
ssh-keygen -t mldsa44-ed25519 -f ~/.ssh/id_mldsa44_ed25519
ssh -i ~/.ssh/id_mldsa44_ed25519 -o IdentitiesOnly=yes lab-host.example.com
```

Add the `.pub` to `authorized_keys` on a 10.6 lab host first. One gotcha from the release notes: keys made with the earlier experimental support "must be regenerated and/or removed," because the algorithm name dropped its `@openssh.com` suffix. That's a small migration in itself. Treat it as practice.

### 3. TLS: prefer hybrid groups where you can

OpenSSL 3.5 supports `X25519MLKEM768`, and on my test box it's what a 3.5 client negotiated by default against a hybrid-capable server. For nginx built against OpenSSL 3.5 or later, the [`ssl_ecdh_curve`](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_ecdh_curve) directive sets the groups the server supports, in order:

```nginx
ssl_protocols TLSv1.3 TLSv1.2;
ssl_ecdh_curve X25519MLKEM768:X25519:prime256v1;
```

Hybrid first, classical fallback for older clients. Then verify with the `s_client` check from step 1. Don't trust the config. Trust the handshake.

### 4. Keep algorithm choices in config

Swapping algorithms should be a config change, not a code change. Grep your codebase for hardcoded curve names, cipher suites, key sizes, and `KexAlgorithms` strings. Move them into config that's version-controlled, reviewed, and deployed like any other change. The `^`, `+`, and `-` prefixes in OpenSSH and the colon-separated group lists in OpenSSL exist for this reason. If raising a parameter set or dropping a family means a code release in five repos, your agility exists only on paper.

### 5. Rehearse rotation with short-lived credentials

The best way to make rotation safe is to do it so often it stops being an event. Let's Encrypt [announced on October 7](https://letsencrypt.org/2026/10/07/64-day-certs) that starting February 10, 2027, its default certificate lifetime drops to 64 days, with 45-day defaults coming in 2028. Its advice: if your renewals are hard-coded to a date from expiration, "update them to renew at approximately ⅔ of the lifetime instead," and "grep for common hardcoded numbers like 83, 80 or 60 in cron jobs, wrapper scripts and runbooks." If your ACME client supports ACME Renewal Info (ARI), the CA can tell it when to renew.

Do that, and extend the idea past web certs. Short-lived SSH certificates instead of long-lived keys. Automated reload and deploy. Alerting on renewal failure. Then run a game day: rotate a key or swap an algorithm on purpose, on a schedule, and see what breaks. Every rotation you rehearse now is one you won't botch during an incident.

## The close

Vitalik is right to take AI-accelerated cryptanalysis seriously, right that hash-based signatures are the conservative choice where they fit, and right that migrations hurt more than hacks. My addition is that none of us knows which assumption breaks next. It could be ECDSA, a lattice parameter set, or a code-based scheme that already survived nearly fifty years, as McEliece just showed.

So don't bet the house on one family. Run hybrids. Keep algorithms in config. Know where every key lives. Rotate often enough that it's boring. Math over promises, and operations over hope.

*Views are my own and do not represent my employer. Nothing here is financial advice.*
