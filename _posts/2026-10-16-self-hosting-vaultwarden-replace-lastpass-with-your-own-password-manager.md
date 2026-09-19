---
layout: post
title: "Self-Hosting Vaultwarden: Replace LastPass with Your Own Password Manager"
date: 2026-10-16
author: "ARDOT Consulting"
tags: [password-manager, vaultwarden, bitwarden, self-hosting, security, lastpass-alternative, open-source]
excerpt: "LastPass raised prices again and had another breach. Here's how to replace it with Vaultwarden — a lightweight, self-hosted, Bitwarden-compatible password manager that costs nothing per seat and keeps your credentials on your own server."
---

Every business has the same problem: dozens of logins, shared across a team, stored in a spreadsheet or a browser or someone's memory. The fix is a password manager. The catch is that the popular ones — LastPass, 1Password, Dashlane — charge per person per month, and they've all had security incidents that should make any business owner nervous.

LastPass specifically has had multiple breaches. The 2022 incident exposed customer vault metadata and encrypted password blobs. Their response was criticized by security researchers for downplaying the severity. Since then, they've raised prices and cut features from lower tiers. If you're paying LastPass $4 per user per month for a 20-person team, that's nearly $1,000 a year to trust a company that's already lost your data once.

There's a better path. [Vaultwarden](https://github.com/dani-garcia/vaultwarden) is a lightweight, open source implementation of the Bitwarden server API, written in Rust. It runs on a $5/month VPS, supports unlimited users and unlimited passwords, and works with the official Bitwarden apps on every platform — desktop, mobile, browser extension. Your team installs the Bitwarden app, points it at your server instead of Bitwarden's cloud, and everything just works.

This post walks through why self-hosting your password manager is worth doing, what Vaultwarden gives you, and how to set it up in an afternoon.

## Why Self-Host Your Password Manager?

A password manager is the most sensitive piece of software your business uses. It holds the keys to everything — your bank, your email, your hosting account, your customer data. When that data lives on a vendor's servers, you're trusting that vendor to:

- Not get breached (they do)
- Not raise prices (they will)
- Not change their terms of service (they have)
- Not shut down or get acquired (it happens)

When you self-host, those risks disappear. The data lives on a server you control. The encryption keys never leave your infrastructure. The cost is fixed — a VPS and a domain name, not a per-seat subscription that grows every time you hire someone.

There's also the flexibility angle. Self-hosting lets you:

- Create unlimited users without a pricing tier change
- Store unlimited items (some SaaS plans cap this)
- Integrate with your existing authentication (LDAP, SSO)
- Apply your own backup and retention policies
- Audit access logs yourself instead of trusting a dashboard

## Vaultwarden vs Bitwarden: What's the Difference?

[Bitwarden](https://bitwarden.com) is open source. You can self-host the official Bitwarden server. It's a solid option. But the official server is written in C# and requires a SQL Server (or PostgreSQL) database, a fair amount of memory, and a non-trivial setup process. It's built for enterprise scale.

Vaultwarden (formerly Bitwarden_RS) is a community-maintained reimplementation of the Bitwarden API in Rust. It's a single binary, uses SQLite by default (though it supports PostgreSQL and MySQL), runs in under 50MB of RAM, and sets up in minutes with Docker. It's API-compatible, meaning the official Bitwarden apps connect to it without modification.

Here's the practical comparison:

| Feature | Bitwarden Official | Vaultwarden |
|---------|-------------------|------------|
| Language | C# | Rust |
| Database | SQL Server / PostgreSQL | SQLite / PostgreSQL / MySQL |
| Memory (idle) | ~500MB | ~20–50MB |
| Setup complexity | Moderate | Simple (Docker) |
| Official apps | Yes | Yes (API-compatible) |
| Organizations/groups | Yes | Yes |
| SSO/SAML | Enterprise tier | Via reverse proxy auth |
| License | GPL-3.0 | GPL-3.0 |
| Best for | Large orgs, enterprise | Small teams, self-hosters |

For a team of 5 to 50 people, Vaultwarden is the simpler choice. It does everything you actually need — shared vaults, collections, groups, secure notes, attachments, emergency access — without the overhead.

## What You Get with Vaultwarden

Vaultwarden supports the full Bitwarden feature set that matters to a small business:

- **Personal vaults** — each user has their own encrypted password store
- **Organizations** — a shared vault for team passwords (the company Netflix login, the hosting account, the social media credentials)
- **Collections** — group items within an organization (e.g., "Marketing", "Finance", "Dev")
- **Groups** — assign users to groups and control which collections each group can access
- **Secure file attachments** — store certificates, private keys, documents alongside passwords
- **Secure notes** — encrypted text notes for things like recovery codes
- **Emergency access** — designate a trusted person who can access your vault if you're unavailable (with a configurable wait period)
- **Bitwarden Send** — share an encrypted password or text snippet with someone via a link that expires
- **TOTP/2FA storage** — store two-factor authentication codes alongside passwords
- **Import/export** — bring data from LastPass, 1Password, KeePass, browser stores

All of this works through the official Bitwarden apps. Your team doesn't learn a new interface. They install Bitwarden from the App Store or browser extension, change one setting (the server URL), and log in.

## Setting Up Vaultwarden with Docker

Here's the practical setup. You'll need a VPS (any cloud provider, $5–10/month), a domain name, and Docker installed.

### Step 1: DNS and Server Prep

Point a subdomain at your server. Something like `vault.yourcompany.com`. You'll need an A record pointing to your server's IP address.

Install Docker and Docker Compose if you haven't already.

### Step 2: Docker Compose File

Create a directory for Vaultwarden and add a `docker-compose.yml`:

```yaml
version: "3.8"

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - DOMAIN=https://vault.yourcompany.com
      - ADMIN_TOKEN=${ADMIN_TOKEN}
      - SIGNUPS_ALLOWED=false
      - INVITATIONS_ALLOWED=true
      - SHOW_PASSWORD_HINT=false
      - LOGIN_RATELIMIT_MAX=5
      - LOGIN_RATELIMIT_SECONDS=60
    volumes:
      - ./vw-data:/data
    ports:
      - "127.0.0.1:8080:80"

  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy-data:/data
      - caddy-config:/config

volumes:
  caddy-data:
  caddy-config:
```

A few things to note about this configuration:

- `SIGNUPS_ALLOWED=false` — this is critical. It prevents random people from creating accounts on your server. You'll create accounts through the admin panel or via invitations.
- `INVITATIONS_ALLOWED=true` — lets you invite team members by email.
- `SHOW_PASSWORD_HINT=false` — password hints are a security risk. Disable them.
- `LOGIN_RATELIMIT` — slows down brute-force attempts.
- The Vaultwarden container only listens on `127.0.0.1:8080` — it's not directly accessible from the internet. Caddy handles the TLS and proxies to it.

### Step 3: Caddyfile for TLS

Caddy automatically provisions and renews Let's Encrypt certificates. Create a `Caddyfile`:

```
vault.yourcompany.com {
    reverse_proxy vaultwarden:8080
    header {
        Strict-Transport-Security "max-age=31536000; includeSubdomains"
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
        Referrer-Policy no-referrer
    }
}
```

### Step 4: Generate an Admin Token

The admin token protects the admin panel (available at `/admin`). Generate a strong one:

```bash
openssl rand -base64 48
```

Store it in a `.env` file in the same directory:

```
ADMIN_TOKEN=your_generated_token_here
```

### Step 5: Start It Up

```bash
docker compose up -d
```

Caddy will provision the TLS certificate. Within a minute, `https://vault.yourcompany.com` should load the Bitwarden web vault.

### Step 6: Create Your Account and Organization

1. Go to `https://vault.yourcompany.com` in your browser.
2. Create your account. Use a strong master password — this is the one password your team needs to remember, and it encrypts everything. Vaultwarden cannot recover it if lost.
3. Log in, then create an Organization from the web vault. This is your shared vault.
4. Set up Collections (e.g., "Finance", "Marketing", "Admin") and Groups to control access.
5. Invite team members via email. They'll create their own accounts and join the organization.

### Step 7: Connect the Apps

On each team member's device:

1. Install the Bitwarden browser extension or mobile app.
2. Open Settings → Self-Hosted Environment.
3. Set the base URL to `https://vault.yourcompany.com`.
4. Log in with their credentials.

That's it. The apps work identically to Bitwarden cloud. Autofill, password generation, sync, offline access — all the same.

## Migrating from LastPass or 1Password

Both LastPass and 1Password support CSV export. Vaultwarden (via the Bitwarden web vault) supports CSV import from both.

From LastPass:
1. Log in to LastPass web vault.
2. Advanced → Export → LastPass CSV.
3. Save the file.

From 1Password:
1. Open 1Password, select the vault.
2. File → Export → All Items → CSV.

Then in the Bitwarden web vault on your Vaultwarden instance:
1. Tools → Import Data.
2. Select the format (LastPass or 1Password).
3. Upload the CSV.
4. Review and organize into collections.

One warning: **delete the CSV file after import**. It contains all your passwords in plaintext. Don't email it, don't store it in a cloud drive, don't leave it on your desktop. Import, verify, delete.

## Security Hardening Checklist

Vaultwarden is secure by design (end-to-end encryption, zero-knowledge architecture — the server never sees plaintext passwords). But self-hosting means you're responsible for the server's security. Here's a checklist:

- [ ] **Use a strong admin token** — 48+ characters, stored in `.env`, not in version control
- [ ] **Disable signups** — `SIGNUPS_ALLOWED=false` (already in the config above)
- [ ] **Enable fail2ban or similar** — protects against brute-force on the SSH port
- [ ] **Keep Docker images updated** — `docker compose pull && docker compose up -d` monthly
- [ ] **Enable automatic backups** — back up the `./vw-data` directory daily
- [ ] **Use SSH key authentication** — disable password login on the VPS
- [ ] **Set up a firewall** — only allow ports 80, 443, and your SSH port
- [ ] **Require 2FA for all users** — enable two-step login (TOTP or WebAuthn) for every account
- [ ] **Use a strong master password** — minimum 14 characters, passphrases preferred
- [ ] **Monitor the admin panel** — check `/admin` periodically for failed login attempts

## Backups: The One Thing You Cannot Skip

Your password vault is only as safe as its backup. If your VPS dies and you don't have a backup, your team is locked out of everything.

The good news: Vaultwarden's data is all in one directory (`./vw-data`). Back it up and you're done. Here's a simple approach:

```bash
#!/bin/bash
# Daily Vaultwarden backup
DATE=$(date +%Y%m%d)
tar -czf /backups/vaultwarden-$DATE.tar.gz -C ./vw-data .
# Keep last 30 days
find /backups -name "vaultwarden-*.tar.gz" -mtime +30 -delete
```

Add this to a cron job. For offsite redundancy, sync the backup to a second location (another VPS, a storage bucket, an external drive). The backup is encrypted — Vaultwarden stores data in an encrypted SQLite database — so the backup file itself is safe even if the storage location isn't fully trusted.

## Cost Comparison

Let's put real numbers on this for a 20-person team:

| Option | Monthly Cost | Annual Cost | Notes |
|--------|-------------|-------------|-------|
| LastPass Business | $7/user × 20 | $1,680 | Per-seat pricing, vendor breaches |
| 1Password Business | $7.99/user × 20 | $1,918 | Per-seat pricing |
| Bitwarden Cloud (Teams) | $3/user × 20 | $720 | Open source vendor, per-seat |
| Vaultwarden (self-hosted) | $5 VPS + $10 domain | $70 | Unlimited users |

The self-hosted option is 10–25× cheaper, and the savings grow with every person you add. The trade-off is that you're responsible for updates and backups — but that's a 30-minute monthly task, not a full-time job.

## When Self-Hosting Might Not Be Right

I'll be honest about the cases where you should just pay Bitwarden Cloud instead:

- **No one on your team is technical enough to manage a VPS.** If "SSH into the server and run docker compose pull" sounds terrifying, use Bitwarden's hosted plan. It's open source, reasonably priced, and they handle the infrastructure.
- **You need enterprise SSO/SAML with strict compliance requirements.** Vaultwarden's SSO support is limited compared to the official Bitwarden server or enterprise SaaS. If you need SOC 2 audits and SAML integration with Okta, go with Bitwarden's enterprise plan.
- **You don't have a reliable backup strategy.** A self-hosted password manager without backups is a ticking bomb. If you can't commit to daily backups, use a hosted option.

For most small businesses — 5 to 50 people, a mix of technical and non-technical staff — Vaultwarden is the right call. The setup takes an afternoon, the maintenance is minimal, and the cost savings are real.

## Wrapping Up

Your password manager is too important to outsource to a company that's already been breached. Vaultwarden gives you the full Bitwarden experience — the apps, the features, the security model — on hardware you control, for the cost of a cheap VPS. The setup is a weekend project, and the payoff is a password infrastructure that scales with your team without scaling your bill.

Start with a $5 VPS, a domain name, and the Docker Compose file above. Migrate your team in a week. Then cancel your LastPass subscription and put the savings somewhere useful.

---

*Need help setting up a self-hosted password manager for your team? ARDOT Consulting specializes in open source security and automation infrastructure. [Reach out through our contact form](https://www.ardotconsulting.com/#contact) and we'll get Vaultwarden deployed, hardened, and connected to your team's workflow.*