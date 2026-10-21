---
title: "PHP2Shell, the FBI Warning, and the Password in Your Config File"
date: 2026-10-21
draft: false
description: "Two critical PHP flaws can let one unauthenticated request run code on a web server. Patch first. Then look at what that server could read, how Vault dynamic database credentials limit the damage, how to vet software before it ships, and why long-lived secrets matter for harvest now, decrypt later."
tags: ["secrets", "vault", "zero-trust", "php", "cve", "dynamic-credentials", "sbom", "container-security", "post-quantum"]
---

## The PHP2Shell vulnerabilities

On October 21, the FBI Cyber Division [posted on X](https://x.com/FBICyberDiv/status/2112921648252719420) that two critical vulnerabilities have been found in PHP, "a widely used programming language that powers websites and web applications." The FBI says the flaws, CVE-2026-109618 and CVE-2026-109619, "could allow malicious cyber actors to remotely execute code in PHP core."

Researchers call the pair **PHP2Shell**. The PHP project gave advance warning on October 20, and public reports that day said the bugs could lead to remote code execution. Fixed releases came out on October 21 at about 10:00 AM ET (14:00 UTC): **8.5.12, 8.4.27, 8.3.36 and 8.2.35**. PHP rates both CVEs 9.2 on CVSS v4.0.

Any PHP install that serves web requests is affected, including PHP-FPM, Apache with mod_php, php-cgi, FrankenPHP and LiteSpeed. The PHP team says mod_php is the riskiest setup, with a demonstrated exploit where one unauthenticated request starts the attack before any application code runs. Code runs with the rights of the web worker account. Proof-of-concept code is circulating, and Cloudflare said it was already seeing early attacks before the patches shipped. PHP 8.1 and older no longer get updates from the PHP project, though the team published minimal source backports for 7.4, 8.0 and 8.1.

{{< figure src="/images/php2shell-what-happened.svg" alt="PHP2Shell attack path: an attacker sends one unauthenticated HTTP request, a PHP core flaw (CVE-2026-109618 or CVE-2026-109619, CVSS v4.0 9.2) lets it run code, the attacker gets remote code execution on the web server, then reads config files and secrets and reaches the database. Fix: update to PHP 8.5.12, 8.4.27, 8.3.36 or 8.2.35, released October 21 at about 10:00 AM ET." >}}

Sources: [BleepingComputer](https://www.bleepingcomputer.com/news/security/php-ships-emergency-patches-for-critical-php2shell-flaws/), [CERT-EU advisory 2026-018](https://cert.europa.eu/publications/security-advisories/2026-018), and the [PHP project announcement](https://www.php.net/archive/2026.php#2026-10-20-1).

Patch now. The PHP team says there are no complete mitigations and updating is the only safe choice.

## Credential exposure

Patching closes the hole, but it doesn't tell you what already walked out. If someone ran code on that server, the first thing to ask is what they could read.

On a lot of systems the answer is the database password sitting in `config.php` or `.env`. It's often shared by several apps, it usually has more access than any of them need, and it hasn't changed in years.

It hasn't changed because changing it is a pain. You update the password in the database, then in every app and server that uses it, then restart all of them at once. If you miss one, something breaks. So rotation keeps getting pushed off, and a stolen copy keeps working long after the patch is in.

## Dynamic database credentials

With HashiCorp Vault's [database secrets engine](https://developer.hashicorp.com/vault/docs/secrets/databases), each app gets its own database login that expires:

1. When the app starts, it proves who it is to Vault, using something like its Kubernetes service account, cloud identity or AppRole.
2. Vault creates a new database user just for that app, with only the permissions it needs.
3. The credential comes with a **lease**, like a key card that stops working at checkout.
4. While the app runs, it renews the lease, up to a hard maximum.
5. When the lease runs out or someone revokes it, Vault deletes that database user.

The app itself needs very little: a way to log in to Vault and a way to read the credential it's given. Most teams don't write that code. [Vault Agent](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent), running next to the app or as a sidecar, logs in, fetches the credential, writes it to a file, and keeps the lease renewed. The app reads a file the way it used to read `config.php`.

Rotation also turns into a restart, because a restarted app gets a brand-new username and password. Nobody edits a config file and nothing needs a coordinated change.

You can cut off a credential two ways:

- **By time.** Every credential has a TTL. When it ends, the credential is gone.
- **By command.** One `vault lease revoke` kills a single credential, and `vault lease revoke -prefix` kills every credential issued from a role. Vault [tracks every lease](https://developer.hashicorp.com/vault/docs/concepts/lease), so it knows what to remove.

{{< figure src="/images/php2shell-flow.svg" alt="Before: a PHP app reads a password that never expires from config.php and uses it on every database request. After: the app logs in to HashiCorp Vault, gets a policy-scoped token, and requests credentials. Vault creates a PostgreSQL role with an expiry, then returns a unique username and password with a one-hour lease. The app connects with these, renews the lease while it runs, and Vault drops the role when the lease expires or is revoked." caption="A static password lasts forever. A Vault dynamic credential lasts only as long as its lease." >}}

Every app instance gets its own username, so database logs show which app did what.

## Vault configuration

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

After this, nobody (including you) knows the password Vault uses to manage the database. See the [API docs for rotate-root](https://developer.hashicorp.com/vault/api-docs/secret/databases#rotate-root-credentials).

**6. Revoke by command.** If you think a server was compromised, cut off one lease, or every credential issued from a role:

```bash
# One credential
vault lease revoke database/creds/app-readonly/2f6a614c-...

# Every credential issued from this role
vault lease revoke -prefix database/creds/app-readonly
```

Vault drops those database users. Whatever the attacker copied stops working. ([lease revoke docs](https://developer.hashicorp.com/vault/docs/commands/lease/revoke))

### Static roles for legacy apps

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

## Securing the software supply chain

Short-lived credentials limit the damage after a break-in. You still want to stop vulnerable software from shipping in the first place.

**Scan at rest.** Scan code repositories, build artifacts and stored images, both when they're created and again later. A clean scan in March says nothing about a CVE published in October.

**Scan container images.** Most PHP today ships inside a container image, and an image is a box that someone else may have packed. Image scanning opens the box and lists what's inside: the operating system packages, the PHP runtime and its version, and the application's own dependencies, such as Composer packages. The scanner writes that list as an SBOM (software bill of materials) built from the image itself. It also finds old PHP runtimes baked into base images that nobody has rebuilt in a year. When a new CVE drops, rescan every image already in your registry. Don't wait for the next build, because the vulnerable images are the ones already running.

**Enforce policy at the gate.** Scanning only helps if something acts on the results. Put a policy check in CI and at deploy time, for example a Kubernetes admission controller, that blocks builds and images that weren't scanned, don't have an SBOM, or contain a known critical CVE. Policy-as-code tools such as Open Policy Agent or HashiCorp Sentinel can express rules like "no image with a critical CVE runs in production."

There are many options for the scanning itself. Commercial platforms include [Wiz](https://www.wiz.io/), [Palo Alto Networks Prisma Cloud](https://www.paloaltonetworks.com/prisma/cloud), [Tenable](https://www.tenable.com/products/vulnerability-management) and [Qualys](https://www.qualys.com/apps/vulnerability-management-detection-response/). On the open-source side, [Syft](https://github.com/anchore/syft) generates SBOMs, and [Grype](https://github.com/anchore/grype) and [Trivy](https://trivy.dev/) scan images and filesystems for known vulnerabilities. If your developers already use OpenAI's Codex, [Codex Security](https://help.openai.com/en/articles/20001107-codex-security) (a research preview on some ChatGPT plans) connects to GitHub repos, scans the code or individual commits, tries to confirm each finding in an isolated environment, and suggests patches for a person to review.

## How to use AI to find vulnerabilities

AI models found these bugs. In its [Patch the Planet post](https://openai.com/index/coordinated-disclosure-of-critical-vulnerabilities/), OpenAI says the security firm Calif found one of the flaws using an OpenAI model, and Wiz independently found and reported both. OpenAI also warns that the same models can help attackers build exploits quickly, and it expects findings like this to "increase sharply in the coming months." Your own code and dependencies are worth running through the same kind of tools before someone else does.

A few places where AI helps a defending team:

- **Review pull requests.** Have an AI reviewer read each PR for injection, unsafe deserialization, missing auth checks and similar bugs, and have a person decide what to fix. Codex Security, covered above, scans repos or individual commits, tries to confirm each finding in an isolated environment, and suggests patches for review.
- **Triage scanner and SBOM output.** Scanners produce long lists. Ask a model to group findings, point out which vulnerable functions your code actually calls, and draft the upgrade or rebuild steps.
- **Write fuzz harnesses.** Models are good at writing the boring glue code a fuzzer needs. Point them at your parsers and request handlers first, since that's where untrusted input lands.
- **Look for hardcoded secrets.** Ask the model to flag passwords, tokens and connection strings in config files, `.env` files and old commits, then move what it finds into Vault and rotate it.

For PHP2Shell specifically, OpenAI published an open-source [PHP version checker](https://github.com/openai/php2shell-checker). You can run it with Node against a URL or a server's local PHP, or install it as a Codex skill. It only reads the version a server reports, so a hidden version comes back as unknown, not safe.

If your team protects essential services and doesn't have the budget, OpenAI is offering subsidized access to [Daybreak](https://openai.com/daybreak/), its program that gives vetted defenders AI models for security work.

Faster discovery also shortens the time between "patch released" and "exploit in the wild," which is one more reason to keep an inventory you can search and credentials that expire on their own.

## Software inventory and SBOMs

For a bug like PHP2Shell, the first question is "where do we run PHP, and which version?" If answering takes a week of emails, attackers have that week too.

PHP hides in more places than people expect: the main web app, an old WordPress marketing site, an internal wiki, a vendor appliance, a forgotten VM in a dev account. CERT-EU points out that many commercial products embed PHP even when they don't expose it directly, and suggests the [NCSC-NL php2shell list](https://github.com/NCSC-NL/php2shell) to find them.

If you keep a live inventory of what's deployed (SBOMs for every image and build, tied to where each one runs), that question becomes a search. You can find the affected systems in minutes, patch or rebuild them in hours, and spend the rest of the day on the ones that can't be patched yet. Isolate those, and make sure they don't hold a long-lived key.

## Harvest now, decrypt later

Attackers can record encrypted traffic and stolen encrypted data today, even though they can't read it yet. They store it and wait. Once a large enough quantum computer exists, the public-key math protecting much of today's traffic could be broken, and the recordings become readable. NIST [describes this](https://www.nist.gov/cybersecurity/what-post-quantum-cryptography) as "harvest now, decrypt later," and it's one reason NIST, NSA and [CISA](https://www.cisa.gov/quantum) are pushing organizations to plan their post-quantum move now.

Long-lived secrets make harvest now, decrypt later worse. If a recording contains a database password that's still valid five years from now, it's worth decrypting. A credential that expired an hour after it was issued isn't worth anything when the recording is finally cracked. The same goes for a config file stolen this week through PHP2Shell: a static password is worth something for as long as nobody changes it.

Dynamic credentials don't protect the data itself, though. Customer records in that recorded traffic stay sensitive no matter how short the credential's life was. Protecting that data means moving key exchange to post-quantum algorithms such as ML-KEM, which NIST standardized in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final), often in a hybrid with today's algorithms during the transition. Short-lived credentials shrink how long a stolen secret stays useful, and post-quantum key exchange protects what's inside the traffic, so you need both.

## Next steps

1. **Find every PHP install.** You can't patch what you don't know you run. Use your inventory and SBOMs (see the inventory section above) to list every system and image with PHP, including the ones embedded in vendor products.
2. **Patch.** Update PHP to 8.5.12, 8.4.27, 8.3.36 or 8.2.35, then rotate any long-lived secret a vulnerable server could read.
3. **Block the old versions.** Add a policy-as-code rule, in Sentinel or OPA, to CI and to deploy or admission checks that rejects images and builds with the vulnerable PHP versions. Otherwise an old image can get redeployed after you've cleaned up.
4. **Move apps off static credentials.** Give each app a short-lived credential from Vault that expires by time, can be revoked with one command, and gets replaced with a fresh one when the app restarts. A web server compromised after that only gives up passwords that are about to expire.

---

*Views are my own and do not represent my employer.*
