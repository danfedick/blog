---
title: "TLS X.509 Error. Assume Chaos."
date: 2026-10-22
draft: false
description: "eMudhra generated eight intermediate CA keys so weakly that anyone could recover them. Browsers blocked them within hours, but curl, Java, Python and CI pipelines kept trusting them. How to check your trust stores and make cert rotation boring with Vault."
tags: ["pki", "tls", "certificates", "vault", "trust-store", "acme", "rotation"]
---

I was flying back from San Francisco, and as we were landing for my layover my phone started buzzing. Then it kept buzzing. By the time we got to the gate I had hundreds of PagerDuty pages, all TLS X.509 errors. The certificate on a lot of my front-facing applications had expired because I forgot to rotate it.

That's the problem with rotating certs by hand every 90 days. You don't do it often enough for it to be part of your workflow, so it's easy to forget. When you do remember, you're looking up the docs again, and sometimes the docs are out of date. It's slow and it's manual.

The right way is to make the process automated and resilient to chaos. The eMudhra incident is a good example of chaos. If your certs come from automation, the fix is boring: restart your application and it gets a new credential.

This post walks through what happened with the eMudhra intermediate certificate authorities (CAs), why browsers were fine and a lot of other software wasn't, how to check your own trust stores and what routine rotation looks like with Vault.

## What happened

On October 17, 2026 a researcher named Wayne reported to the browser root programs that the private keys for eight eMudhra intermediate CA certificates were compromised. To prove it, he signed the message "We have a problem here - Wayne" with each of the eight keys. Mozilla logged the report at 20:06 UTC that day. ([disclosure site](https://emudhra.we-have-a-problem.com/), [Mozilla](https://blog.mozilla.org/security/2026/10/20/protecting-firefox-users-following-the-compromise-of-emudhra-intermediate-certificates/))

If you don't live in public key infrastructure (PKI) every day, an intermediate CA is the middle link in the chain. Your device trusts a short list of root CAs, the roots sign intermediates and the intermediates sign the certificates websites actually use. Anyone holding an intermediate's private key can sign a certificate for any site, and your device will accept it as long as it trusts the root above it. These eight chain to roots trusted by the Mozilla, Microsoft, Apple and Chrome root programs.

What bugs me most is that the CA generated the keys incorrectly, so anyone could recover a private key from the public certificate. That has been possible since the certificates were created in April 2025, which means the keys were exposed for about 18 months before anyone said anything. The CA has since revoked them. Mozilla's incident report says the private keys were supposed to be random numbers across a huge range, but all eight were numbers below 256, so anyone could find them in seconds by trying the first 256 possible keys. ([Bugzilla 2083449](https://bugzilla.mozilla.org/show_bug.cgi?id=2083449))

## Browsers vs. everything else

The browsers moved fast. Mozilla pushed a block through OneCRL, its central list of revoked certificates, and finished rolling it out to Firefox users with standard settings by 22:59 UTC the same day, less than three hours after the report. Chrome blocked them on the 17th too, Apple on the 19th and Microsoft on the 20th. Between them the vendors used OneCRL, CRLite and CRLSets. Mozilla also shipped Firefox 158.0.1 and ESR builds for people whose settings blocked the OneCRL update, plus NSS 3.131.1, which removes the root that signed the affected intermediates.

There's another reason browsers were safe. Firefox, Chrome, Edge and Safari enforce Certificate Transparency (CT), which means a certificate has to show up in a public log before they trust it. There are no CT log entries for these intermediates since April 2025, so CT-enforcing browsers could not have been attacked.

Most other software doesn't get any of that. Your Go services, Java apps, curl in a build job and Python scripts usually check a certificate against a bundled list of roots and stop there. No revocation check, no CT check. They keep trusting the bad intermediates until someone updates the bundle or the runtime. Even macOS gets caught here. The OS blocks the intermediates, but the built-in curl, git, python3 and openssl use `/etc/ssl/cert.pem`, which doesn't check revocation and hadn't been updated. Pinning and DNS Certification Authority Authorization (CAA) records can help catch a fake cert, but only when the app uses them, and most don't.

Do federal teams enforce CT outside the browser? In my experience almost never, and I couldn't find federal guidance that asks them to. NIST SP 800-52 Rev. 2, the federal TLS guide, only says servers with publicly trusted certs should support the CT extension and says nothing about clients checking it. ([NIST SP 800-52 Rev. 2](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-52r2.pdf)) Nobody knows yet if attackers used these keys against non-browser clients.

{{< figure src="/images/emudhra-trust-flow.svg" alt="eMudhra flow: a weak intermediate key is recovered from the public cert and an attacker mints a cert for any site. Browsers block it through revocation lists and CT. Non-browser clients that skip those checks trust it. The defense is trust store updates plus short-lived certs." >}}

## Trust stores in containers and federal systems

A typical agency app has five or six trust stores. There's the OS bundle in the base image, the Java cacerts file, Python's certifi, Node's bundled roots, Go's view of the system store and maybe an appliance or two with its own list. Each one updates on its own schedule, if it updates at all.

In most teams I've worked with, nobody really owns the CA bundle in the base image. It gets whatever was in the upstream image the day someone last rebuilt it. That's fine until a day like October 17.

The fixes are rolling out. Ubuntu removed the two roots that cross-signed eMudhra, emSign Root TLS CA - G3 and Comodo AAA Certificate Services, in ca-certificates 20260816ubuntu0 for 22.04, 24.04, 26.04 and 26.10. Releases 20.04 and older get it through Ubuntu Pro ESM. Alpine 20261020-r0, Chainguard 20260909-r3, Go 1.27.3 and 1.26.10, Node.js 22.23.4, 24.22.1 and 26.12.1, Python certifi 2026.10.19 and a long list of Java builds are patched. Bottlerocket, Bun and Erlang certifi were still pending when I checked. One side effect: removing those roots also breaks legitimate certificates that chain to them, so a few hosts could become unreachable.

For RHEL shops, Red Hat's bulletin [RHSB-2026-012](https://access.redhat.com/security/vulnerabilities/RHSB-2026-012) rates this Important and lists RHEL 6 through 10 as affected. That includes containers built from RHEL and UBI images, RHEL CoreOS, OpenShift and OpenStack. As of the morning of October 22, Red Hat's tracker for CVE-2026-114406 still lists ca-certificates as affected on RHEL 6 through 10 with no RHEL errata yet. The only fix out is RHSA-2026:86595 for Red Hat Hardened Images, released October 20. ([Red Hat CVE-2026-114406](https://access.redhat.com/security/cve/CVE-2026-114406)) Until the RHEL packages land, Red Hat's workaround blocks the two roots in the system trust store, and it says to rebuild customer containers once the base images update.

Air-gapped and offline networks are the hard case. They don't get OneCRL or automatic Windows trust list updates, so someone has to carry the new bundle in and rebuild. I haven't found STIG or FedRAMP language that sets a deadline for trust store updates. The Department of War (DoW) STIGs check that the DoW root CAs are installed, under CCI-000185, but they don't say how fast a distrusted root has to come out. NIST SP 800-53 has SC-17 for PKI certificates and SI-2 for flaw remediation, and both leave the timing up to the organization. ([Windows 11 STIG V-253427](https://www.stigviewer.com/stigs/microsoft_windows_11_v2r7/2026-02-12/finding/V-253427), [NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final))

## Auditing your trust store

If I had a team on this today, I'd start with the easiest check. The disclosure site runs test hosts that serve the bad chains. A certificate error is the result you want.

```
curl https://dv-g3b-emudhra.we-have-a-problem.com/
curl https://dv-g3b-aaa-emudhra.we-have-a-problem.com/
```

Next, check what's installed. On Debian or Ubuntu:

```
dpkg -l 'ca-certificates' | grep ^ii
```

On RHEL or UBI:

```
rpm -q ca-certificates
```

Then patch it:

```
sudo apt update && sudo apt install --only-upgrade ca-certificates
```

Your laptop passing doesn't mean your images pass, so run the same check inside them:

```
docker run --rm <image> curl -sS https://dv-g3b-emudhra.we-have-a-problem.com/
```

I ran this on October 22 against the current public images. Only `ubi9/ubi-minimal` has curl out of the box, and it connected, so that image still trusts the bad chain. Ubuntu 24.04, Debian slim, Alpine, distroless and Chainguard's `static` and `wolfi-base` don't ship curl at all. Ubuntu 24.04 and Debian slim don't ship a CA bundle either, so they get one when you install `ca-certificates`. With curl and ca-certificates installed, both refused the connection. Chainguard's `curl` image refused it too.

For images without curl, copy the bundle out and test it from the host:

```
c=$(docker create <image> x)
docker cp -L $c:/etc/ssl/certs/ca-certificates.crt ./bundle.pem
docker rm $c
curl --cacert ./bundle.pem https://dv-g3b-emudhra.we-have-a-problem.com/
```

On RHEL and UBI the bundle path is `/etc/pki/tls/certs/ca-bundle.crt`. Done this way, Alpine and both Chainguard bundles refused the connection. The `gcr.io/distroless/static-debian12` and `ubi9/ubi-minimal` bundles connected, so both still need an update.

For anything offline, the disclosure site has a chains.zip with the PEM chains for all twelve test hosts and the OpenSSL steps to check them. There are also manual workarounds to distrust the affected roots on Debian, Ubuntu, Alpine, RHEL, Arch, NixOS, macOS, Windows, NetBSD and FreeBSD before a package lands. The site says the Linux ones are for expert users only, and enterprise Windows devices should go through IT.

After the cleanup, make it a gate check in CI by putting the curl test in your image build. If it connects, fail the build. Good looks like every image and runtime getting a certificate error from those test hosts, and a named owner for each trust store.

## Routine key rotation

eMudhra's keys sat there for 18 months. If those intermediates had lived for weeks instead of years, the window would have been a lot smaller and fewer certs would have depended on them.

Public certificates are already heading that way. Under CA/Browser Forum Ballot SC-081v3, a public TLS certificate issued on or after March 15, 2026 can be valid for at most 200 days. That drops to 100 days on March 15, 2027 and 47 days on March 15, 2029. ([CA/B Forum Baseline Requirements](https://cabforum.org/working-groups/server/baseline-requirements/requirements/))

Most teams I work with rotate internal intermediates about every five years, if they rotate them at all. I want rotation to be boring. If it runs on a schedule and the tooling handles it, nobody has to remember. Ubuntu's approach of pulling whole roots also shows the cost of not being able to swap fast: good certs break along with bad ones. Being able to swap a CA quickly is what limits the damage.

## Making rotation boring with Vault

{{< figure src="/images/emudhra-vault-pki-rotation.svg" alt="Vault PKI: the root CA signs the intermediate CA, the agents role issues agent certs with a 1h TTL and 24h max, and a new intermediate is signed, made default and tidied up on a schedule." >}}

There are two kinds of certs to keep short-lived: the ones from your own internal CA and the ones from a public CA. HashiCorp Vault handles both.

For internal CAs, use the [PKI secrets engine](https://developer.hashicorp.com/vault/docs/secrets/pki). You run the root and intermediates, issue short-lived certs to services and agents, and swap intermediates on a schedule with the [rotation primitives](https://developer.hashicorp.com/vault/docs/secrets/pki/rotation-primitives). Since Vault 1.14 it also works as an ACME server, so certbot or cert-manager can renew internal certs against your own CA.

For public CAs, use the [PKI External CA secrets engine](https://developer.hashicorp.com/vault/docs/secrets/pki-external-ca) (Vault Enterprise). It speaks ACME to CAs like Let's Encrypt, GlobalSign, Sectigo and DigiCert, can answer DNS-01 challenges through your DNS provider, and hands the certs to your apps.

There's no federal rule for how long an internal intermediate should live. NIST SP 800-57 Part 1 suggests one to three years of use for a private signing key. ([NIST SP 800-57 Part 1 Rev. 5](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf)) For an internal intermediate in a federal environment I'd start with something like one year and work it down as the automation proves itself.

Either way, renewal becomes a scheduled job instead of a calendar reminder. When the next CA has a bad day, swapping it is a config change and a renewal cycle.

## Closing takeaway

If you run federal IT, do one thing this week. Run the curl check from inside every base image you ship and fix the ones that connect. After that, give every trust store an owner and put the check in CI so the next incident like this one is just a rebuild.

Resources:
- [eMudhra disclosure site and checker](https://emudhra.we-have-a-problem.com/)
- [Mozilla Security Blog post](https://blog.mozilla.org/security/2026/10/20/protecting-firefox-users-following-the-compromise-of-emudhra-intermediate-certificates/)
- [Ubuntu fixes](https://ubuntu.com/blog/emudhra-ca-compromise-fixes-available)
- [Red Hat RHSB-2026-012](https://access.redhat.com/security/vulnerabilities/RHSB-2026-012)
- [Vault PKI secrets engine](https://developer.hashicorp.com/vault/docs/secrets/pki)
- [Vault PKI External CA secrets engine](https://developer.hashicorp.com/vault/docs/secrets/pki-external-ca)

Did I miss anything? Let me know!

Views are my own and do not represent my employer.
