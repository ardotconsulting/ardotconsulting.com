---
layout: post
title: "Self-Hosting Jitsi Meet: Replace Zoom with Your Own Video Conferencing Platform"
date: 2026-10-18
author: "ARDOT Consulting"
tags: [jitsi, video-conferencing, self-hosting, zoom-alternative, open-source, docker]
excerpt: "Zoom costs $15+ per user per month and has had its share of privacy incidents. Here's how to replace it with Jitsi Meet — an open-source, self-hosted video conferencing platform that costs nothing per seat and keeps your calls on your own server."
---

If your business relies on video calls — and most do now — you're probably paying for Zoom, Teams, or a similar platform. For a 20-person team on Zoom Business at $18.32/user/month, that's over $4,300 a year. And what do you get for that money? A 40-minute limit on group meetings if you're on the free tier, recordings stored on Zoom's servers, and a feed of "AI summary" features that send your meeting transcripts to a third party's cloud.

Zoom has also had its share of privacy problems. The "Zoombombing" era of 2020 exposed weak default security. Their encryption was initially misrepresented. And their terms of service have, at various points, included language about using customer content for AI training — language they walked back only after public backlash.

There's a better option. [Jitsi Meet](https://jitsi.org/) is a fully open-source video conferencing platform. You self-host it on your own server, and your team gets unlimited meetings, unlimited duration, unlimited participants, recordings stored wherever you want, and end-to-end encryption (E2EE) for calls. No per-seat pricing. No data going to a third party. No surprise terms-of-service changes.

This post walks through why self-hosting your video conferencing is worth doing, what Jitsi Meet gives you, and how to set it up with Docker in an afternoon.

## Why Self-Host Your Video Conferencing?

Video conferencing is one of the most data-sensitive tools your business uses. Every call contains your strategy discussions, client conversations, internal decisions, and potentially screen-shared financials or product roadmaps. When that data flows through a vendor's servers, you're trusting them with your most confidential information.

Self-hosting Jitsi gives you three concrete advantages:

**1. No per-seat pricing.** Jitsi doesn't have user accounts in the traditional sense. Anyone with the URL can join a meeting. You don't pay for "hosts" or "participants" or "licenses." A 5-person team and a 50-person team pay the same: the cost of the server it runs on.

**2. Data sovereignty.** Your call traffic — video, audio, chat messages, shared screens — flows through your server, not a vendor's. If you need to prove to a client or regulator that your meeting data never left your infrastructure, you can. Recordings stay on your storage, not someone else's cloud.

**3. No vendor lock-in.** Jitsi is open source under the Apache 2.0 license. If the project disappeared tomorrow (it won't — it's backed by 8x8 and has a large community), you still have the code. You're not at the mercy of a pricing page update or a feature being removed.

## What Jitsi Meet Gives You

Jitsi Meet is not a stripped-down alternative. It's a production-grade platform with features that match or exceed what most businesses use on Zoom:

| Feature | Jitsi Meet (self-hosted) | Zoom (Business) |
|---------|--------------------------|-----------------|
| Max participants | Limited only by server capacity | 100 (Business), 300 (Business Plus) |
| Meeting duration | Unlimited | Unlimited (paid tiers) |
| End-to-end encryption | Yes (optional, per meeting) | Yes (for meetings, not recordings) |
| Screen sharing | Yes | Yes |
| Recording | Yes (stored on your server) | Yes (stored on Zoom cloud) |
| Live streaming | Yes (to YouTube, RTMP) | Yes (add-on) |
| Virtual backgrounds | Yes | Yes |
| Breakout rooms | Yes | Yes |
| Chat & file sharing | Yes | Yes |
| Transcription | Via integration (Vosk, Whisper) | Yes (AI Companion) |
| Mobile apps | iOS, Android, F-Droid | iOS, Android |
| Browser support | Chrome, Firefox, Safari, Edge | Chrome, Firefox, Safari, Edge |
| Per-seat cost | $0 | $18.32/user/month |
| Data location | Your server | Zoom's cloud |

The main thing Jitsi doesn't have is Zoom's polish in a few edge cases — webinar registration pages, polished in-meeting polling UI, and some advanced host controls. For 95% of business video calls, Jitsi does everything you need.

## What You Need to Run It

Jitsi Meet is actually a collection of components — Jicofo (conference focus), Jitsi Videobridge (the SFU that routes video), Prosody (XMPP server for signaling), and the Jitsi Meet web frontend. The good news is that the Jitsi team maintains an official Docker setup that bundles all of these into a single docker-compose configuration.

**Hardware requirements** depend on how many concurrent participants you expect. Unlike a simple web app, video conferencing is CPU-intensive because the server is routing real-time video streams:

| Concurrent Participants | Server Recommendation |
|------------------------|----------------------|
| Up to 10 | 2 vCPU, 4 GB RAM (e.g., $5–10/month VPS) |
| 10–30 | 4 vCPU, 8 GB RAM (e.g., $20–40/month VPS) |
| 30–100 | 8 vCPU, 16 GB RAM (e.g., $80/month dedicated) |
| 100+ | Multiple Videobridge nodes, load balanced |

The key bottleneck is the Jitsi Videobridge (JVB), which handles all the video routing. For most small businesses with 10–20 people on calls, a basic VPS is plenty. If you occasionally host larger calls (50+ people), you can scale up the server or run multiple JVB instances.

You also need:

- A domain name (e.g., `meet.yourcompany.com`)
- A server running Linux (Ubuntu 22.04 or 24.04 recommended)
- Docker and Docker Compose installed
- Ports 80/443 (HTTP/HTTPS), 4443 (TCP), and 10000/udp (UDP) open

## Step-by-Step: Self-Hosting Jitsi Meet with Docker

### Step 1: Prepare Your Server

Start with a fresh Ubuntu server. Update packages and install Docker:

```bash
sudo apt update && sudo apt upgrade -y
# Install Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# Log out and back in for docker group to take effect
```

### Step 2: Get the Jitsi Docker Configuration

Jitsi maintains an official Docker repository. Clone it and set up your environment:

```bash
cd /opt
git clone https://github.com/jitsi/docker-jitsi-meet.git jitsi
cd jitsi
cp env.example .env
```

### Step 3: Configure the Environment File

The `.env` file controls all the key settings. Edit it with your domain and security settings:

```bash
# Required: your domain
PUBLIC_URL=https://meet.yourcompany.com

# Security: generate strong passwords
# Run this to generate random passwords for each service:
# for i in JICOFO_COMPONENT_SECRET JICOFO_AUTH_PASSWORD JVB_AUTH_PASSWORD JIGASI_XMPP_PASSWORD JIBRI_XMPP_PASSWORD JIBRI_RECORDER_PASSWORD; do echo "$i=$(openssl rand -hex 16)"; done

# Paste the generated passwords into .env:
JICOFO_COMPONENT_SECRET=<generated>
JICOFO_AUTH_PASSWORD=<generated>
JVB_AUTH_PASSWORD=<generated>
JIGASI_XMPP_PASSWORD=<generated>
JIBRI_XMPP_PASSWORD=<generated>
JIBRI_RECORDER_PASSWORD=<generated>

# Enable authentication (optional but recommended)
ENABLE_AUTH=1
ENABLE_GUESTS=1
AUTH_TYPE=internal

# Enable recording (optional)
ENABLE_RECORDING=1
JIBRI_RECORDING_DIR=/config/recordings

# Enable end-to-end encryption
ENABLE_E2E=true

# Let's Encrypt for automatic SSL
ENABLE_LETSENCRYPT=1
LETSENCRYPT_DOMAIN=meet.yourcompany.com
LETSENCRYPT_EMAIL=admin@yourcompany.com
```

### Step 4: Set Up the Directory Structure

Jitsi's Docker setup needs persistent directories for configuration and recordings:

```bash
mkdir -p ~/.jitsi-meet-cfg/{web/letsencrypt,transcripts,prosody/config,prosody/prosody-plugins,jicofo,jvb,jigasi,jibri/recordings}
# Set proper permissions (Jitsi containers run as user 1011:1011 by default)
chown -R 1011:1011 ~/.jitsi-meet-cfg
```

### Step 5: Start the Stack

```bash
docker compose up -d
```

This pulls and starts all the Jitsi components: Prosody (XMPP), Jicofo (conference management), JVB (video bridge), the web frontend, and (if enabled) Jibri for recording. The first run takes a few minutes to download images.

Verify everything is running:

```bash
docker compose ps
```

You should see all services in "running" state. If Let's Encrypt is enabled, the container will automatically obtain an SSL certificate for your domain.

### Step 6: Create User Accounts (If Auth Enabled)

If you set `ENABLE_AUTH=1`, you need to create host accounts. Users with these accounts can start meetings; guests can join without an account.

```bash
# Enter the Prosody container to create a user
docker compose exec prosody prosodyctl --config /config/prosody.cfg.lua register username meet.jitsi yourpassword
```

Now, when you visit `https://meet.yourcompany.com`, you'll be prompted to log in before starting a meeting. Guests who receive a meeting link can join without credentials.

### Step 7: Configure Your Firewall

Make sure these ports are open on your server:

```bash
# If using ufw:
sudo ufw allow 80/tcp     # Let's Encrypt / HTTP redirect
sudo ufw allow 443/tcp    # HTTPS
sudo ufw allow 4443/tcp   # JVB TCP fallback
sudo ufw allow 10000/udp  # JVB UDP (main video traffic)
sudo ufw enable
```

The UDP port 10000 is the most important — that's where all the real-time video and audio flows. If it's blocked, calls will fail to connect or drop to audio-only.

### Step 8: Test Your Setup

Visit `https://meet.yourcompany.com` in a browser. You should see the Jitsi Meet interface — a clean, modern video conferencing UI. Start a meeting, share the link with a colleague, and verify that video, audio, screen sharing, and chat all work.

Test from a mobile device too: download the Jitsi Meet app from the App Store or F-Droid, enter your server address, and join a meeting.

## Recording and Transcription

### Recording with Jibri

Jibri (Jitsi Broadcasting Infrastructure) records meetings as video files. It launches a headless Chrome instance, joins the meeting as a hidden participant, and captures the entire call. Recordings are saved as `.webm` files on your server.

The recording directory you configured (`~/.jitsi-meet-cfg/jibri/recordings`) contains the output. You can set up a cron job to move recordings to a shared drive or cloud storage — or better yet, use Nextcloud (which we covered in an earlier post) to store recordings on infrastructure you control.

```bash
# Example: move recordings to a Nextcloud share nightly
0 2 * * * rsync -av /root/.jitsi-meet-cfg/jibri/recordings/ /mnt/nextcloud/jitsi-recordings/ && find /root/.jitsi-meet-cfg/jibri/recordings/ -mtime +7 -delete
```

### Transcription with Whisper or Vosk

Jitsi doesn't include built-in transcription, but you can add it. The open-source `jitsi-meet-transcription` integration supports both [Vosk](https://alphacephei.com/vosk/) (offline, lightweight) and [OpenAI Whisper](https://github.com/openai/whisper) (more accurate, can run locally via Ollama or a standalone instance).

For a privacy-focused setup, run Vosk on the same server or a companion machine. It provides real-time transcription that appears as subtitles in the meeting and can be saved alongside the recording. This gives you the "AI summary" feature that Zoom charges extra for, except the transcript never leaves your server.

## Keeping It Secure

A self-hosted video conferencing platform is only as secure as you make it. Here's a checklist:

**Network security:**
- Use Let's Encrypt SSL (enabled by default in the Docker setup)
- Restrict access to the admin interfaces (Prosody, Jicofo console) to internal IPs or via SSH tunnel
- Close all ports except 80, 443, 4443, and 10000/udp
- Use a firewall (ufw or iptables)

**Access control:**
- Enable authentication (`ENABLE_AUTH=1`) so only authorized users can start meetings
- Use strong passwords for all internal service accounts (the generated passwords from Step 3)
- Regularly review who has host accounts

**Data security:**
- Enable E2EE for sensitive meetings (participants toggle this in the meeting UI)
- Store recordings on encrypted volumes
- Set up automatic cleanup of old recordings to limit data retention
- Back up your configuration regularly

**Updates:**
- Watch the [Jitsi security advisories](https://github.com/jitsi/jitsi-meet/security/advisories)
- Update the Docker images regularly:
  ```bash
  cd /opt/jitsi
  git pull
  docker compose pull
  docker compose up -d
  ```

## Cost Comparison: One Year

Let's put real numbers on this. Here's what a 20-person team pays over a year:

| Expense | Zoom Business | Jitsi Meet (self-hosted) |
|---------|--------------|--------------------------|
| Software licensing | $4,397/yr ($18.32 × 20 × 12) | $0 |
| Server (4 vCPU, 8 GB VPS) | — | $240–480/yr ($20–40/mo) |
| Domain (meet.yourcompany.com) | — | ~$12/yr |
| SSL certificate | — | $0 (Let's Encrypt) |
| Admin time (setup + maintenance) | — | ~4 hours initial, 1 hr/month |
| **Total Year 1** | **~$4,397** | **~$400–600 + a few hours of time** |

That's roughly $3,800–4,000 saved in the first year alone. The savings scale with team size — a 50-person team saves over $11,000/year.

## When Jitsi Might Not Be the Right Choice

I'm not going to pretend self-hosting is always the answer. Jitsi might not be right for you if:

- **You host large webinars (500+ participants).** Jitsi scales, but large webinars need multiple JVB nodes and careful tuning. Zoom's webinar product handles this out of the box.
- **You have no one to manage a server.** Self-hosting means you're responsible for updates, backups, and uptime. If your team has zero IT capacity, a managed solution (even Jitsi's own hosted offering) may be more practical.
- **You need PSTN dial-in.** Jitsi supports SIP integration via Jigasi, but it requires a SIP provider and more configuration. Zoom includes dial-in by default.
- **Your team heavily uses Zoom-specific integrations.** If you've built workflows around Zoom's API, webinar funnels, or marketplace apps, migrating means rebuilding those integrations.

For most small-to-medium businesses doing internal and client video calls, Jitsi is more than sufficient. The question is whether you're willing to spend a few hours on setup to save thousands per year.

## Integrating Jitsi with Your Other Self-Hosted Tools

One of the advantages of running your own stack is that your tools can talk to each other. Here are a few integration ideas:

**Jitsi + Mattermost:** If you self-host Mattermost for team chat (as we covered in a previous post), you can configure Mattermost to use your Jitsi instance for video calls. The Mattermost Jitsi plugin lets users start a Jitsi meeting directly from any channel or DM with a single command.

**Jitsi + Nextcloud:** Nextcloud Talk includes a Jitsi integration. You can embed Jitsi meetings into Nextcloud calendar events, so clicking "Join call" on a calendar invite opens your self-hosted Jitsi instance.

**Jitsi + n8n:** Use n8n to automate post-call workflows. For example: when a recording appears in the Jibri output directory (detected via a file watcher node), n8n can transcribe it with Whisper, summarize it with a local LLM via Ollama, and post the summary to a Mattermost channel.

**Jitsi + Cal.com:** If you use Cal.com for scheduling (another tool we've covered), you can set Jitsi as your default meeting location. Cal.com will generate Jitsi meeting URLs with unique room names for each booking automatically.

These integrations create a fully self-hosted, privacy-respecting communication stack — scheduling, chat, video, and post-call automation — with zero per-seat licensing costs.

## Migration Tips: Moving from Zoom to Jitsi

If you're switching from Zoom, here's how to make the transition smooth:

1. **Run both in parallel for 2 weeks.** Keep Zoom active while you test Jitsi with internal calls first. This gives your team time to get comfortable without the pressure of client-facing meetings.

2. **Update your calendar defaults.** Change the default video conferencing link in your calendar tool (Cal.com, Nextcloud Calendar) to your Jitsi instance.

3. **Set up a short URL.** Something like `meet.yourcompany.com` is easy to share. Jitsi generates random room names by default, but users can type any room name in the URL: `meet.yourcompany.com/WeeklyStandup`.

4. **Create a quick-start guide for your team.** A one-page doc with: how to start a meeting, how to share the link, how to join from mobile, how to share your screen, and how to record. The interface is intuitive, but a quick reference reduces anxiety.

5. **Test with external participants.** Have a client join a Jitsi call. They don't need an account — just a browser. Confirm that firewalls on their end don't block the UDP port (rare, but it happens; the TCP fallback on port 4443 handles most cases).

6. **Cancel Zoom when you're confident.** Don't rush. Most Zoom plans are month-to-month, so you can cancel once your team is comfortable.

## Wrapping Up

Self-hosting your video conferencing is one of the highest-impact moves you can make for both cost reduction and data privacy. Jitsi Meet is a mature, well-maintained platform that does 95% of what businesses use Zoom for — and the 5% it doesn't do is rarely missed.

The math is straightforward: a few hours of setup saves you $3,000–$11,000+ per year, depending on team size. And you get the peace of mind that comes with knowing your meeting data never touches a third-party server.

Start with a basic VPS, follow the Docker setup above, and test it with your team this week. The worst case is you decide it's not for you and cancel the VPS — you're out $20 and an afternoon. The best case is you never pay for video conferencing again.

---

*Want help setting up Jitsi Meet or any other self-hosted tool for your business? [Get in touch with ARDOT Consulting](/) — we specialize in open-source automation and infrastructure for small and medium businesses.*