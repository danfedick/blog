---
title: "PHP2Shell, the FBI Warning, and the Password in Your Config File"
date: 2026-10-21
draft: false
description: "Two critical PHP flaws can let one unauthenticated request run code on a web server. Patch first. Then look at what that server could read, how Vault dynamic database credentials limit the damage, how to vet software before it ships, and why long-lived secrets matter for harvest now, decrypt later."
tags: ["secrets", "vault", "zero-trust", "php", "cve", "dynamic-credentials", "sbom", "container-security", "post-quantum"]
---

## What happened

Today the FBI Cyber Division [posted on X](https://x.com/FBICyberDiv/status/2112921648252719420) that two critical vulnerabilities have been found in PHP, "a widely used programming language that powers websites and web applications." The FBI says the flaws, CVE-2026-109618 and CVE-2026-109619, "could allow malicious cyber actors to remotely execute code in PHP core."

Researchers call the pair **PHP2Shell**. The PHP project gave advance warning on October 20, and public reports that day said the bugs could lead to remote code execution. Fixed releases came out on October 21 at about 10:00 AM ET (14:00 UTC): **8.5.12, 8.4.27, 8.3.36 and 8.2.35**. PHP rates both CVEs 9.2 on CVSS v4.0.

Any PHP install that serves web requests is affected, including PHP-FPM, Apache with mod_php, php-cgi, FrankenPHP and LiteSpeed. The PHP team says mod_php is the riskiest setup, with a demonstrated exploit where one unauthenticated request starts the attack before any application code runs. Code runs with the rights of the web worker account. Proof-of-concept code is circulating, and Cloudflare said it was already seeing early attacks before the patches shipped. PHP 8.1 and older no longer get updates from the PHP project, though the team published minimal source backports for 7.4, 8.0 and 8.1.

{{< figure src="/images/php2shell-what-happened.svg" alt="PHP2Shell attack path: an attacker sends one unauthenticated HTTP request, a PHP core flaw (CVE-2026-109618 or CVE-2026-109619, CVSS v4.0 9.2) lets it run code, the attacker gets remote code execution on the web server, then reads config files and secrets and reaches the database. Fix: update to PHP 8.5.12, 8.4.27, 8.3.36 or 8.2.35, released October 21 at about 10:00 AM ET." caption="One unauthenticated request can end with your database password in someone else's hands." >}}

Sources: [BleepingComputer](https://www.bleepingcomputer.com/news/security/php-ships-emergency-patches-for-critical-php2shell-flaws/), [CERT-EU advisory 2026-018](https://cert.europa.eu/publications/security-advisories/2026-018), and the [PHP project announcement](https://www.php.net/archive/2026.php#2026-10-20-1).

Patch now. The PHP team says there are no complete mitigations and updating is the only safe choice. Patching also does nothing about whatever an attacker already copied off the box.

## What an attacker takes with them

Don't trust a server to stay clean. Plan for the day it isn't, and ask what the attacker could read from it.

On a lot of systems, that's a database password in `config.php` or `.env`. Several apps often share it, it usually has more rights than it needs, and it hasn't changed in years because changing it is painful.

Here's why it's painful. With a static password in config, rotating it means updating the database, then updating every app and server that uses it, then redeploying or restarting all of them at the same moment. Miss one and something breaks. So teams put it off, and the stolen copy keeps working long after the patch goes in.

## The Vault way: credentials that expire

With HashiCorp Vault's [database secrets engine](https://developer.hashicorp.com/vault/docs/secrets/databases), the app works more like a hotel guest than a tenant with a permanent key.

1. When the app starts, it proves who it is to Vault, using something like its Kubernetes service account, cloud identity or AppRole.
2. Vault creates a new database user just for that app, with only the permissions it needs.
3. The credential comes with a **lease**, like a key card that stops working at checkout.
4. While the app runs, it renews the lease, up to a hard maximum.
5. When the lease runs out or someone revokes it, Vault deletes that database user.

So what does the app itself have to do? Less than you'd think. It needs a way to log in to Vault and a way to read the credential it's given. Most teams don't write that code. [Vault Agent](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent), running next to the app or as a sidecar, logs in, fetches the credential, writes it to a file, and keeps the lease renewed. The app reads a file the way it used to read `config.php`.

Rotation also becomes a restart. When an app restarts, it gets a brand-new username and password. Nobody edits a config file and nothing needs a coordinated change.

You can cut off a credential two ways:

- **By time.** Every credential has a TTL. When it ends, the credential is gone.
- **By command.** One `vault lease revoke` kills a single credential, and `vault lease revoke -prefix` kills every credential issued from a role. Vault [tracks every lease](https://developer.hashicorp.com/vault/docs/concepts/lease), so it knows what to remove.

{{< figure src="/images/php2shell-flow.svg" alt="Before: a PHP app reads a password that never expires from config.php and uses it on every database request. After: the app logs in to HashiCorp Vault, gets a policy-scoped token, and requests credentials. Vault creates a PostgreSQL role with an expiry, then returns a unique username and password with a one-hour lease. The app connects with these, renews the lease while it runs, and Vault drops the role when the lease expires or is revoked." caption="A static password lasts forever. A Vault dynamic credential lasts only as long as its lease." >}}

Every app instance gets its own username, so database logs show which app did what.

## The commands

Below is a minimal PostgreSQL example. The syntax follows HashiCorp's [PostgreSQL database secrets engine docs](https://developer.hashicorp.com/vault/docs/secrets/databases/postgresql). Names and passwords are placeholders.

**1. Turn on the database secrets engine.**

```bash
vault secrets enable database
```

**2. Tell Vault how to reach the database.** Vault uses this account to create and remove users. Give it the minimum rights it needs to do that.

```bash
vault write database/config/app-postgres \
    plugin_name="postgresql-database-plugin" \
    allowed_roles="app-readonly" \
    connection_url="postgresql://{{username}}:{{password}}@db.internal:5432/appdb" \
    username="vault-admin" \
    password="change-me-then-rotate" \
    password_authentication="scram-sha-256"
```

The `password_authentication` option needs Vault 1.14 or later.

**3. Define a role.** This is the recipe Vault follows every time it creates a user: what SQL to run and how long the credential lives.

```bash
vault write database/roles/app-readonly \
    db_name="app-postgres" \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

`default_ttl` is how long a credential lasts before it must be renewed. `max_ttl` is the hard ceiling. After that, the app has to ask for a new one.

**4. The app asks for credentials.**

```bash
vault read database/creds/app-readonly
```

```text
Key                Value
---                -----
lease_id           database/creds/app-readonly/2f6a614c-...
lease_duration     1h
lease_renewable    true
password           SsnoaA-8Tv4t34f41baD
username           v-token-app-read-...
```

In practice, the app doesn't run this by hand. [Vault Agent](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent) or a Vault client library fetches and renews the credential so the app code only reads a file or an environment.

**5. Rotate the root credential.** The password you typed in step 2 is now known to you and your shell history. Have Vault rotate it so only Vault knows it:

```bash
vault write -f database/rotate-root/app-postgres
```

After this, nobody (including you) knows the password Vault uses to manage the database. That's the point. See the [API docs for rotate-root](https://developer.hashicorp.com/vault/api-docs/secret/databases#rotate-root-credentials).

**6. Revoke by command.** If you think a server was compromised, cut off one lease, or every credential issued from a role:

```bash
# One credential
vault lease revoke database/creds/app-readonly/2f6a614c-...

# Every credential issued from this role
vault lease revoke -prefix database/creds/app-readonly
```

Vault drops those database users. Whatever the attacker copied stops working. ([lease revoke docs](https://developer.hashicorp.com/vault/docs/commands/lease/revoke))

### Apps that need a fixed username

Some legacy apps or databases expect a fixed username. For those, Vault has [**static roles**](https://developer.hashicorp.com/vault/docs/secrets/databases#static-roles): the username stays the same, but Vault owns the password and rotates it on a schedule.

```bash
vault write database/static-roles/legacy-app \
    db_name="app-postgres" \
    username="legacy_app" \
    rotation_period="24h"

vault read database/static-creds/legacy-app
```

The `legacy_app` user must already exist in the database, and Vault changes its password as soon as you create the static role.

It isn't as strong as a unique user per app instance, but it beats a password nobody has changed in years.

## Vetting software before it gets in

Short-lived credentials limit the damage after a break-in. You still want to stop vulnerable software from shipping in the first place. Here's how I'd set up the front door.

**Scan at rest.** Scan code repositories, build artifacts and stored images, both when they're created and again later. A clean scan in March says nothing about a CVE published in October.

**Scan container images.** Most PHP today ships inside a container image, and an image is a box that someone else may have packed. Image scanning opens the box and lists what's inside: the operating system packages, the PHP runtime and its version, and the application's own dependencies, such as Composer packages. The scanner writes that list as an SBOM (software bill of materials) built from the image itself. It also finds old PHP runtimes baked into base images that nobody has rebuilt in a year. When a new CVE drops, rescan every image already in your registry. Don't wait for the next build, because the vulnerable images are the ones already running.

**Enforce policy at the gate.** Scanning only helps if something acts on the results. Put a policy check in CI and at deploy time, for example a Kubernetes admission controller, that blocks builds and images that weren't scanned, don't have an SBOM, or contain a known critical CVE. Policy-as-code tools such as Open Policy Agent or HashiCorp Sentinel can express rules like "no image with a critical CVE runs in production."

There are many options for the scanning itself. Commercial platforms include [Wiz](https://www.wiz.io/), [Palo Alto Networks Prisma Cloud](https://www.paloaltonetworks.com/prisma/cloud), [Tenable](https://www.tenable.com/products/vulnerability-management) and [Qualys](https://www.qualys.com/apps/vulnerability-management-detection-response/). On the open-source side, [Syft](https://github.com/anchore/syft) generates SBOMs, and [Grype](https://github.com/anchore/grype) and [Trivy](https://trivy.dev/) scan images and filesystems for known vulnerabilities. If your developers already use OpenAI's Codex, [Codex Security](https://help.openai.com/en/articles/20001107-codex-security) (a research preview on some ChatGPT plans) connects to GitHub repos, scans the code or individual commits, tries to confirm each finding in an isolated environment, and suggests patches for a person to review. Pick what fits your environment.

## AI is finding bugs faster now

PHP2Shell is also a preview of what's coming. In its [Patch the Planet post](https://openai.com/index/coordinated-disclosure-of-critical-vulnerabilities/), OpenAI says the security firm Calif found one of the flaws using an OpenAI model, and Wiz independently found and reported both. PHP's maintainers led the fix. OpenAI, Wiz, Cloudflare and others helped test patches and prepare protections, such as firewall rules, before the details went public. OpenAI says it believes AI found these bugs before attackers did, which gave defenders time to get ready. It also warns that the same models can help attackers build exploits quickly, and that it expects findings like this to "increase sharply in the coming months."

The post proposes a way to handle disclosure at that pace. Maintainers and researchers confirm and fix the bug first. Then distributors, hosting providers and security vendors get early notice so their updates are ready. Government agencies such as CISA help the warning reach critical services. Public guidance goes out once a patch exists. OpenAI also published an open-source [PHP version checker](https://github.com/openai/php2shell-checker). You can run it with Node against a URL or a server's local PHP, or install it as a Codex skill. It only reads the version a server reports, so a hidden version comes back as unknown, not safe. OpenAI is also offering subsidized access to [Daybreak](https://openai.com/daybreak/), its program that gives vetted defenders AI models for security work, to under-resourced teams that protect essential services.

My takeaway for operators: if bugs are found and disclosed faster, the time between "patch released" and "exploit in the wild" gets shorter. That's another reason to keep an inventory you can search and credentials that expire on their own.

## Know everything you run

When something like PHP2Shell lands, the first question is "where do we run PHP, and which version?" If answering takes a week of emails, the attackers are ahead of you.

PHP hides in more places than people expect: the main web app, an old WordPress marketing site, an internal wiki, a vendor appliance, a forgotten VM in a dev account. CERT-EU points out that many commercial products embed PHP even when they don't expose it directly, and suggests the [NCSC-NL php2shell list](https://github.com/NCSC-NL/php2shell) to find them.

If you keep a live inventory of what's deployed (SBOMs for every image and build, tied to where each one runs), that question becomes a search. You can find the affected systems in minutes, patch or rebuild them in hours, and spend the rest of the day on the ones that can't be patched yet. Isolate those, and make sure they don't hold a long-lived key.

## Harvest now, decrypt later

This idea sounds unrelated, but it's the same problem over a longer time span.

Attackers can record encrypted traffic and stolen encrypted data today, even though they can't read it yet. They store it and wait. Once a large enough quantum computer exists, the public-key math protecting much of today's traffic could be broken, and the recordings become readable. NIST [describes this](https://www.nist.gov/cybersecurity/what-post-quantum-cryptography) as "harvest now, decrypt later," and it's one reason NIST, NSA and [CISA](https://www.cisa.gov/quantum) are pushing organizations to plan their post-quantum move now.

Long-lived secrets make that worse. If a recording contains a database password that's still valid five years from now, it's worth decrypting. A credential that expired an hour after it was issued isn't worth anything when the recording is finally cracked. The same goes for a config file stolen this week through PHP2Shell: a static password is worth something for as long as nobody changes it.

Dynamic credentials don't protect the data itself, though. Customer records in that recorded traffic stay sensitive no matter how short the credential's life was. Protecting that data means moving key exchange to post-quantum algorithms such as ML-KEM, which NIST standardized in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final), often in a hybrid with today's algorithms during the transition. Short-lived credentials shrink how long a stolen secret stays useful, and post-quantum key exchange protects what's inside the traffic. You'll need both.

## Where to go from here

Patch PHP to 8.5.12, 8.4.27, 8.3.36 or 8.2.35, then rotate any long-lived secret a vulnerable server could read.

After that, start moving apps off static, long-lived credentials. Give each app a short-lived credential from Vault. Let it expire by time, kill it with one command when you need to, and let the app pick up a fresh one when it restarts. The next time a web server gets popped, the attacker leaves with a password that's about to stop working.

---

*Views are my own and do not represent my employer.*
