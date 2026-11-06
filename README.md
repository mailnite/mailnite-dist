# Mailnite

**A complete, private mail server in one static binary — with a Gmail-class webmail and an AI that actually knows your mailbox.**

Mailnite is a self-hosted mail server for people and teams who want their mail on their own hardware without assembling a dozen daemons. One ~40 MB binary ships everything: SMTP/submission, IMAP, POP3, CalDAV/CardDAV, a modern webmail, an admin console with browser-based first-run onboarding, spam defense in depth, per-user mailbox encryption, and built-in AI search and assistance with the provider of your choice — including local models, so mail never has to leave your machine.

**Website & documentation: [mailnite.com](https://mailnite.com)**

> This repository hosts the **official release binaries** (see [Releases](../../releases)) and is the download target behind the stable `get.mailnite.com` URLs. The update channel, install scripts, and version manifests live at [get.mailnite.com](https://get.mailnite.com).

## Quick install (Linux)

One line on a fresh Linux server (systemd, amd64/arm64):

```bash
curl -fsSL https://get.mailnite.com/mailnite | sudo bash
```

Then open `http://<your-server>:8480` — setup continues in the browser: domain, storage, accounts. Nothing is asked at the terminal. (Firewall port 8480 to your own address on a public host, and finish onboarding right away.)

The installer creates a dedicated `mailnite` system user, installs a hardened systemd service that may bind the privileged mail ports (25, 143/993, 465/587, 80/443 — no root, no setcap), verifies every download against the sha256 manifest on get.mailnite.com, and enables an hourly self-update timer. State lives in `/var/lib/mailnite`.

Hosting at home behind NAT? Install the **relay** on any cheap VDS with a public IP — your server dials out to it, no port forwarding, no static IP (get the key from the admin console, *Mail relay* tab):

```bash
curl -fsSL https://get.mailnite.com/relay | sudo bash -s -- --token <KEY>
```

Or let the console make and keep the relay's machine for you at Google Cloud, AWS or Azure — a reserved address that outlives the machine, a Spot machine under it, automatic recovery when the cloud takes it: **[cloud-relay.md](cloud-relay.md)**.

Update either one at any moment (both also self-update hourly on their own):

```bash
curl -fsSL https://get.mailnite.com/mailnite | sudo bash -s -- --update
curl -fsSL https://get.mailnite.com/relay    | sudo bash -s -- --update
```

Prefer it manual? Grab an archive from [Releases](../../releases) (also macOS and Windows builds), verify, run:

```bash
curl -fsSLO https://get.mailnite.com/dl/mailnite/<version>/mailnite_linux_amd64.tar.gz
curl -fsSLO https://get.mailnite.com/dl/mailnite/<version>/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing   # a checksum that doesn't match is a binary you don't run
tar xzf mailnite_linux_amd64.tar.gz
./mailnite run                             # http://localhost:8480 — the wizard takes over
```

## Kubernetes

A Helm chart runs Mailnite on any cluster. The mail edge can be a cloud load
balancer (GKE, AKS, EKS), router port-forwards, or a mail relay when the
cluster sits behind Cloudflare or NAT:

```bash
helm install mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail --create-namespace \
  --set domain=example.com --set exposure=loadBalancer
```

Step-by-step guides:

| Where | Guide |
|---|---|
| Azure Kubernetes Service | **[azure.md](azure.md)** |
| Google Kubernetes Engine | **[gcp.md](gcp.md)** |
| Amazon EKS | **[aws.md](aws.md)** |
| Your own cluster with a Gateway (behind Cloudflare or NAT, home lab, router) | **[kubernetes.md](kubernetes.md)** |

All options: [mailnite-chart](https://github.com/mailnite/mailnite-chart).

## What's in the box

- **Mail core** — SMTP + submission, IMAP, POP3, DKIM signing (per-domain, multi-domain sending), SPF/DKIM/DMARC verification, RFC-compliant DSN bounces, TLS-RPT reporting both ways, automatic TLS.
- **Webmail** — conversations across folders, category tabs, labels, snooze, drafts autosave, powerful search operators — the UX you'd expect from a big-tech inbox, on your own server.
- **AI, yours** — semantic + keyword hybrid search, "ask your mailbox" assistant over your own mail (RAG), automatic categorization, AI-assisted spam filtering, attachment OCR, an AI reading view for messy HTML mail. Bring your own provider — cloud or fully local.
- **Privacy & crypto** — protected mode seals a user's mail at rest with their key; senders can encrypt *to* you via a DNS-published key; optional OIDC single sign-on.
- **Operations** — admin console for everything (ports, TLS, spam guardrails, integrations), one-click DNS record publishing (Cloudflare / Route 53), S3 archival tier, CLI + RPC control plane, hourly checksum-verified self-updates.

## Why Mailnite

|  | **Mailnite** | Stalwart | Mailcow | Postfix + Dovecot (DIY) |
|---|---|---|---|---|
| Install & run | one static binary, **one-line installer** | single binary, manual setup | Docker Compose stack (many containers) | hand-assembled daemons |
| Webmail included | ✓ conversations, snooze, category tabs | — bring your own | ✓ SOGo | — bring your own |
| Browser onboarding wizard | ✓ full first-run setup on :8480 | partial (web admin, config-first) | partial | — |
| AI mailbox (semantic search, ask-your-mail, auto-categories) | ✓ built-in, **local models supported** | — | — | — |
| Encrypted mailboxes (per-user keys at rest) | ✓ + inbound sealing via DNS-published key | ✓ at-rest | — | plugin surgery |
| Host from home / NAT | ✓ built-in reverse relay, no port forwarding | — | — | — |
| Spam defense | ✓ DNSBL, greylisting, Bayes, AI classifier, DMARC, per-user behavioral allowlists | ✓ built-in filter | ✓ rspamd | assemble & tune yourself |
| Multi-domain + DKIM | ✓ per-domain keys, **one-click DNS publish** (Cloudflare/Route 53) | records shown, publish manually | records shown, publish manually | manual everything |
| Calendar & contacts | ✓ CalDAV / CardDAV | ✓ | ✓ SOGo | — |
| Self-updates | ✓ hourly, checksum-verified channel | manual | update script | manual |
| Footprint | ~40 MB binary, embedded store, **no external DB** | comparable | multi-GB RAM recommended | varies |

The one-sentence version: Stalwart gives you a modern server core but no mailbox experience; Mailcow gives you the experience but a heavy container fleet; DIY gives you control and costs you weekends. Mailnite ships the whole thing — server, inbox, admin, AI — as one binary you can hold.

## Verify what you run

Every release ships a `SHA256SUMS` covering all artifacts. The installers and the self-update timer verify automatically against the manifest at `get.mailnite.com/v1/mailnite.txt` — for manual downloads, take thirty seconds:

```bash
sha256sum -c SHA256SUMS --ignore-missing     # linux
shasum -a 256 -c SHA256SUMS 2>/dev/null      # macos
```

The macOS and Windows binaries are code-signed by Karagatan LLC — a Developer ID signature notarized by Apple, an Authenticode signature on the exe:

```bash
codesign --verify --strict --verbose=2 mailnite && spctl --assess --type execute -vv mailnite   # macos: "accepted … Notarized Developer ID"
osslsigncode verify -in mailnite.exe                                                           # windows, or Properties → Digital Signatures
```

---

[mailnite.com](https://mailnite.com) · [get.mailnite.com](https://get.mailnite.com) · [Releases](../../releases)

© Karagatan LLC. Mailnite is a product name of Karagatan LLC.
