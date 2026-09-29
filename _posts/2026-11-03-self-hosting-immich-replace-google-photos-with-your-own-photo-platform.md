---
layout: post
title: "Self-Hosting Immich: Replace Google Photos With Your Own Photo Platform"
date: 2026-11-03
author: "ARDOT Consulting"
tags: [immich, self-hosting, photos, privacy, docker, google-photos-alternative, open-source]
excerpt: "Immich is a self-hosted photo and video management platform with facial recognition, smart search, and mobile backup — all running on your own server, no cloud subscription required."
---

## The Google Photos Problem

If your business stores photos and videos on Google Photos, you're handing your visual assets to a company that makes its money from data. Google Photos stopped offering free unlimited storage in 2021, and the paid tiers keep getting more expensive. Worse, your photos are scanned, indexed, and used to train models you have no visibility into.

For businesses — especially those in real estate, construction, insurance, or any field where photos document work — this creates a real problem. Your job site photos, property listings, damage documentation, and client images are sitting on someone else's servers, subject to terms of service you don't control.

**Immich** solves this. It's an open source, self-hosted photo and video management platform that gives you nearly everything Google Photos offers — mobile backup, facial recognition, smart search, shared albums, a world map — but runs entirely on hardware you own. No subscription. No data scanning. No vendor lock-in.

With over 115,000 GitHub stars and an active development community, Immich is the most mature self-hosted photo platform available today. Let's walk through what it does, what it costs to run, and how to set it up.

## What Immich Actually Does

Immich is not a simple file dump. It's a full photo management platform with machine learning built in. Here's what you get out of the box:

| Feature | Google Photos | Immich (Self-Hosted) |
|---------|--------------|----------------------|
| Automatic mobile backup | Yes | Yes |
| Facial recognition & clustering | Yes | Yes (on-device ML models) |
| Semantic search ("beach sunset") | Yes | Yes (CLIP models) |
| Shared albums | Yes | Yes |
| World map with geo-pins | Yes | Yes |
| RAW format support | Limited | Yes (extensive) |
| Video transcoding | Yes | Yes (hardware-accelerated) |
| Multi-user support | Yes (family plan) | Yes (unlimited users) |
| Public sharing links | Yes | Yes |
| Memories ("x years ago") | Yes | Yes |
| Data sovereignty | No | Yes — your server, your disks |
| Monthly cost | $2–$10+/month | Electricity + hardware |
| License | Proprietary | AGPL-3.0 (open source) |

The standout features for business use:

- **Facial recognition and clustering**: Immich automatically detects faces in your photos and groups them. Search for "person: John" and you get every photo of John across your entire library.
- **CLIP-based semantic search**: Search for "construction site" or "invoice on desk" and Immich finds matching photos using machine learning — even if those words aren't in any filename or metadata.
- **EXIF metadata extraction**: Every photo's camera, lens, GPS coordinates, timestamp, and settings are indexed and searchable.
- **Mobile apps for iOS and Android**: Native apps with automatic background backup. Your team's phones upload photos to your server as soon as they're taken.
- **Multi-user with shared albums**: Create accounts for team members, share albums with clients via public links, and control who sees what.

## Hardware Requirements

Immich needs a real server, not a Raspberry Pi. The machine learning components — facial recognition and CLIP search — are computationally intensive, especially during the initial library scan.

**Minimum requirements** (small team, <10,000 photos):

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| RAM | 6 GB | 8–16 GB |
| CPU | 2 cores | 4+ cores |
| Storage | 500 GB HDD | 1+ TB SSD/NAS |
| GPU | None required | Optional (speeds up ML) |
| OS | Linux (Docker) | Linux (Docker) |

The initial scan processes every photo through ML models for face detection, CLIP embedding, and metadata extraction. With 10,000 photos on a 4-core server without GPU, expect this to take 4–8 hours. After the initial scan, new photos process in seconds.

If you have a GPU (even an older NVIDIA card), Immich can use it for hardware-accelerated transcoding and ML inference, cutting scan times dramatically. But it's not required — CPU-only works fine for most small businesses.

## Step-by-Step: Installing Immich with Docker

Immich runs as a set of Docker containers. You'll need Docker and Docker Compose installed on your server. The official setup is straightforward.

### Step 1: Create the Immich Directory

```bash
mkdir ~/immich-app
cd ~/immich-app
```

### Step 2: Download the Configuration Files

Immich provides ready-to-use Docker Compose and environment files:

```bash
# Download docker-compose.yml
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml

# Download the example environment file
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

### Step 3: Configure the Environment File

Open `.env` in your editor and set the key variables:

```bash
# Where your photos will be stored — point this at your NAS or large disk
UPLOAD_LOCATION=/mnt/photos/immich

# Database password — change this to something secure
DB_PASSWORD=your_secure_password_here

# Set your timezone
TZ=America/New_York

# The version of Immich to run (use "latest" for stable releases)
IMMICH_VERSION=latest
```

The `UPLOAD_LOCATION` is critical. This is where all your photos and videos will live. Point it at a mounted NAS, a RAID array, or whatever storage you trust. Immich stores the database (metadata, user accounts, search indexes) separately in a Docker volume — but the actual media files go in `UPLOAD_LOCATION`.

### Step 4: Start the Containers

```bash
docker compose up -d
```

This pulls and starts several containers:

- **Immich Server** — the core API and application logic (NestJS/Node.js)
- **Immich Web** — the SvelteKit frontend you access in your browser
- **Immich Machine Learning** — runs facial recognition, CLIP embeddings, and smart search models
- **PostgreSQL** — the database storing metadata, users, and search indexes
- **Redis** — job queue for processing uploads and background tasks

Check that everything is running:

```bash
docker compose ps
```

All five containers should show as `Up`. If any are restarting, check the logs:

```bash
docker compose logs <service-name>
```

### Step 5: Access the Web Interface

Open your browser and navigate to:

```
http://<your-server-ip>:2283
```

The first person to register becomes the admin user. After creating your account, you'll see the Immich dashboard — a clean, photo-first interface that looks remarkably like Google Photos.

### Step 6: Set Up Mobile Backup

Download the Immich mobile app:

- **iOS**: App Store (search "Immich")
- **Android**: Google Play, FUTO F-Droid, or direct APK from GitHub Releases
- **Obtainium**: Immich provides a config link from your server's Utilities page for auto-updates via Obtainium

Open the app, enter your server URL (`http://<your-server-ip>:2283` or your domain if you've set up reverse proxy), and log in. Tap the cloud icon in the top right, select which albums to back up, and press **Start Backup**.

The app supports **background backup** — photos upload automatically even when the app isn't open. You can also configure it to back up only on Wi-Fi to avoid cellular data usage.

## Migrating from Google Photos

If you're moving away from Google Photos, Immich makes the migration straightforward.

### Export from Google Takeout

Go to Google Takeout (takeout.google.com) and request an export of your Google Photos data. Google will give you a set of ZIP files containing your photos, videos, and metadata in JSON sidecar files.

### Import with immich-go

**immich-go** is a community-built command-line tool that imports Google Takeout archives directly into Immich, preserving albums, dates, and metadata:

```bash
# Download immich-go (check GitHub for latest release)
wget https://github.com/simulot/immich-go/releases/latest/download/immich-go-linux-amd64
chmod +x immich-go-linux-amd64

# Import your Takeout archive
./immich-go-linux-amd64 upload \
  -server=http://<your-server-ip>:2283 \
  -key=YOUR_API_KEY \
  -google-photos \
  /path/to/takeout-archives/
```

You can get your API key from the Immich web interface under **Settings → API Keys**. immich-go handles the JSON sidecar files, reconstructs albums, and uploads everything with original timestamps intact.

For large archives, this can take hours or days depending on your upload bandwidth. Run it on the same network as your Immich server if possible — local network speeds make a huge difference.

## Securing Your Immich Installation

Running a photo server exposed to the internet requires proper security. Here's the minimum you should do:

### Reverse Proxy with HTTPS

Never expose port 2283 directly. Use a reverse proxy like **Caddy** or **Nginx** to handle TLS termination:

**Caddy** (simplest option — automatic HTTPS via Let's Encrypt):

```caddyfile
photos.yourdomain.com {
    reverse_proxy localhost:2283
}
```

That's it. Caddy automatically provisions and renews SSL certificates. Your Immich instance is now accessible at `https://photos.yourdomain.com` with proper encryption.

### Firewall Rules

Only expose ports 80 and 443 (for the reverse proxy). Keep the database, Redis, and ML containers on the internal Docker network only:

```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny 2283/tcp
ufw enable
```

### Regular Database Backups

Immich has built-in database backup functionality. Configure it in the admin settings to create scheduled dumps of the PostgreSQL database. But remember — **the database only contains metadata and user information**. Your actual photos and videos in `UPLOAD_LOCATION` need their own backup strategy.

Follow the **3-2-1 backup rule**: three copies of your data, on two different media types, with one copy off-site. Tools like **Restic** or **BorgBackup** can handle encrypted, deduplicated backups of your photo storage to an off-site location.

## Immich for Business Use Cases

Here's how different business types can use Immich practically:

### Real Estate

Agents photograph every property they list. With Immich, photos auto-upload from each agent's phone to a shared server. The broker can create albums per property, share public links with buyers, and search "kitchen renovation" across the entire portfolio using CLIP semantic search.

### Construction

Job site photos document progress, safety issues, and work completed. Immich's facial recognition can identify which workers appear in which photos. The EXIF metadata gives you GPS coordinates and timestamps for every shot — useful for verifying when work was done.

### Insurance

Claims adjusters photograph damage on-site. With Immich, these photos stay on company servers, not in someone's personal phone or a consumer cloud. The search functionality lets adjusters find "roof damage" or "water stain" across all claims, and shared albums can be used for internal review.

### Marketing Agencies

Creative teams generate hundreds of photos per shoot. Immich provides a central library with tags, albums, and search — no more digging through shared drives or email attachments. Public sharing links let clients review selects without needing an account.

## Cost Comparison: Immich vs Google Photos

Let's look at the real economics for a small business with 5 users and ~50,000 photos (approximately 500 GB of storage):

| Cost Factor | Google Photos (5 users) | Immich (Self-Hosted) |
|-------------|------------------------|---------------------|
| Monthly subscription | $50–$100/month (Google One) | $0 |
| Server hardware (one-time) | $0 | $300–$800 (refurbished server) |
| Storage (one-time) | Included | $100–$200 (2 TB NAS drive) |
| Electricity | $0 | ~$5–$15/month |
| 3-year total | $1,800–$3,600 | $480–$1,100 + electricity |

The break-even point is typically 6–12 months, after which Immich is dramatically cheaper. And you own the hardware — no price increases, no storage caps, no policy changes.

But the cost isn't the only factor. The real value is **control**. When Google changes its terms of service, deprecates features, or raises prices, you have no recourse. When you self-host Immich, you make those decisions.

## Limitations to Know About

Immich is excellent, but it's not perfect. Here are the honest trade-offs:

- **No cloud convenience**: If your server goes down, your photos are inaccessible until it's back up. Google Photos is backed by Google's global infrastructure. You're trading reliability for sovereignty.
- **Maintenance burden**: You're responsible for updates, backups, and security patches. Immich releases frequently (version 3.3.x as of this writing), and updates are generally smooth but require attention.
- **Initial ML scan time**: Processing a large library through facial recognition and CLIP models takes hours. Plan for this during initial setup.
- **No built-in cloud sync**: If you want off-site replication, you need to set that up yourself with Restic, Borg, or rsync.
- **Mobile app maturity**: The apps are very good but occasionally have sync issues with very large libraries. The team is active and bugs get fixed quickly.

## Integrating Immich with Your Automation Stack

Immich has a full REST API, which means you can integrate it with automation tools like **n8n**:

- **Auto-tag uploads**: Use n8n to watch for new photos via the Immich API, run them through Ollama for description generation, and tag them automatically.
- **Backup notifications**: Set up an n8n workflow that checks Immich's health endpoint and sends a Mattermost notification if the server goes down.
- **Client delivery**: When photos are added to a "Client Deliverables" album, n8n can generate a public sharing link and email it to the client via your self-hosted Listmonk newsletter platform.
- **Duplicate detection**: Run periodic API calls to identify and flag duplicate photos for review.

The API is well-documented at `/api/doc` on your Immich instance (Swagger UI), making it straightforward to build custom workflows.

## Getting Started Checklist

If you're ready to make the switch, here's your action plan:

1. **Audit your current photo storage** — how many photos, how much storage, how many users
2. **Provision a server** — a refurbished desktop with 8 GB RAM and a 2 TB drive works for most small businesses
3. **Install Docker and Docker Compose** on the server
4. **Follow the installation steps above** to get Immich running
5. **Set up a reverse proxy** (Caddy) with HTTPS and a domain name
6. **Install the mobile app** on all team members' phones and configure backup
7. **Request a Google Takeout export** if migrating from Google Photos
8. **Import your archive** using immich-go
9. **Set up backups** — database dumps plus Restic/Borg for the media files
10. **Create user accounts** for team members and configure shared albums

## Conclusion

Google Photos is convenient, but convenience that costs you control over your business's visual assets isn't a good trade. Immich gives you the features that matter — mobile backup, smart search, facial recognition, shared albums — on hardware you own, with no monthly fee and no data scanning.

For businesses that deal in photos daily — real estate, construction, insurance, marketing — the case is straightforward. The break-even on hardware costs is under a year, and after that, you're saving money every month while keeping complete control of your data.

The setup takes an afternoon. The migration takes a weekend. The payoff lasts for years.

---

*Want help setting up Immich or other self-hosted tools for your business? [Contact ARDOT Consulting](/) — we design and deploy open source automation infrastructure for small and mid-size businesses.*