---
layout: post
title: "Automating Social Media Management with Open Source Tools"
date: 2026-12-13
author: "ARDOT Consulting"
tags: [social-media, automation, n8n, ollama, open-source, scheduling, content]
excerpt: "Schedule posts, generate captions, and track engagement across every platform without paying for Hootsuite or Buffer — using entirely open source tools."
---

If your business posts to more than one social media platform, you already know the drill. You write something in one tool, resize the image for another, copy-paste the caption for a third, and then manually check each platform to see if anyone engaged. It's tedious, repetitive, and exactly the kind of work that automation was built to solve.

The standard answer is a SaaS tool like Hootsuite, Buffer, or Sprout Social. They work — but they charge per user, per account, and per feature tier. A small business managing three platforms with two team members can easily pay $50–$200 per month, forever. And you're handing your social media strategy, posting schedule, and analytics data to a third party.

There's a better path. With open source tools, you can build a social media management pipeline that schedules posts, generates captions with AI, cross-posts to multiple platforms, and tracks engagement — all on infrastructure you control. This guide walks through how to do it.

## What You're Replacing

Before we build, let's be clear about what the commercial tools do, so we know what we're matching:

| Feature | Hootsuite / Buffer | Open Source Alternative |
|---------|--------------------|--------------------------|
| Post scheduling | Built-in calendar UI | n8n + cron triggers |
| Multi-platform posting | Native API integrations | n8n HTTP nodes + platform APIs |
| AI caption generation | Premium add-on | Ollama with local LLM |
| Analytics dashboard | Built-in reports | Umami or Plausible |
| Team collaboration | Per-seat pricing | Self-hosted, no seat limits |
| Monthly cost | $50–$200+ | Server cost only (~$5–$20/mo) |

The trade-off is setup time. A SaaS tool is ready in five minutes. The open source path takes an afternoon to configure — but after that, it runs indefinitely with no subscription fees.

## The Tool Stack

Here's what we'll use:

- **n8n** — Workflow automation. This is the engine that orchestrates everything: triggering scheduled posts, calling APIs, and routing content between platforms.
- **Ollama** — Local LLM hosting. Generates captions, hashtags, and content variations without sending your brand voice to an external API.
- **Mastodon / Bluesky API** — Open social platforms with accessible APIs. We'll also cover posting to LinkedIn and X/Twitter via their APIs.
- **Umami** — Privacy-first web analytics. Tracks click-throughs from your social posts back to your website.
- **PostgreSQL or SQLite** — Stores your content calendar, post history, and engagement metrics.

If you've been following this blog, you likely already have n8n and Ollama running. If not, start with our [guide to building your first n8n workflow](https://www.ardotconsulting.com/blog/2026/08/17/how-to-build-your-first-ai-automation-workflow-with-n8n/) and our [Ollama setup guide](https://www.ardotconsulting.com/blog/2026/08/20/running-local-llms-with-ollama-a-practical-guide-for-businesses/).

## Step 1: Set Up Your Content Calendar

Every automation pipeline starts with structured data. Instead of a spreadsheet, we'll use a simple SQLite database (or you can use NocoDB if you prefer a visual interface).

```sql
CREATE TABLE social_posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    caption TEXT,
    hashtags TEXT,
    image_url TEXT,
    platforms TEXT,  -- comma-separated: "mastodon,bluesky,linkedin"
    scheduled_at DATETIME NOT NULL,
    posted_at DATETIME,
    status TEXT DEFAULT 'draft',  -- draft, scheduled, posted, failed
    engagement_json TEXT  -- stored after posting
);
```

This table is your single source of truth. You can populate it manually, from a form (OpnForm works well here), or from another automation that generates content ideas.

If you want a visual editor instead of raw SQL, spin up [NocoDB](https://www.ardotconsulting.com/blog/2026/10/22/self-hosting-nocodb-replace-airtable-with-your-own-no-code-database-platform/) and connect it to this database. You'll get a spreadsheet-like interface that anyone on your team can use.

## Step 2: Generate Captions with Ollama

Writing social media captions is one of those tasks that's easy to describe but painful to do repeatedly. This is where a local LLM shines — it can generate platform-appropriate captions in your brand voice without any per-request API cost.

Here's a script that calls Ollama to generate a caption:

```python
import requests

def generate_caption(topic, platform, brand_voice="professional but approachable"):
    prompt = f"""Write a social media post for {platform} about: {topic}

Brand voice: {brand_voice}
Requirements:
- Keep it under 280 characters for X/Twitter, 500 for LinkedIn, 300 for Mastodon
- Include 2-3 relevant hashtags
- End with a clear call to action
- No emoji spam (max 2 emojis)

Write only the post, no preamble."""

    response = requests.post(
        "http://localhost:11434/api/generate",
        json={
            "model": "llama3.2",
            "prompt": prompt,
            "stream": False,
            "options": {"temperature": 0.7, "max_tokens": 200}
        }
    )

    return response.json()["response"].strip()

# Example usage
caption = generate_caption(
    topic="our new open source analytics dashboard",
    platform="LinkedIn"
)
print(caption)
```

The key advantage of running this locally: your brand messaging, product details, and content strategy never leave your server. There's no API bill, no rate limits to worry about, and you can fine-tune the model on your existing posts if you want even better brand voice matching.

### A Workflow for Content Variations

Different platforms reward different writing styles. LinkedIn wants longer, reflective posts. Mastodon favors conversational tones. X/Twitter demands brevity. Instead of writing three versions manually, generate them in one pass:

```python
platforms = {
    "twitter": "Under 280 characters. Punchy. One hashtag.",
    "linkedin": "Up to 500 characters. Professional insight. Two paragraphs.",
    "mastodon": "Up to 300 characters. Conversational. No hashtags required."
}

for platform, style_guide in platforms.items():
    caption = generate_caption(
        topic="launching our automated customer support system",
        platform=platform,
        brand_voice=f"Follow this style: {style_guide}"
    )
    print(f"\n--- {platform.upper()} ---\n{caption}\n")
```

This gives you three platform-optimized posts in about 30 seconds, with no API charges.

## Step 3: Build the Scheduling Workflow in n8n

Now we connect everything. The n8n workflow has three phases: check the calendar, generate content if needed, and post to each platform.

### Phase 1: Trigger and Query

Set up a **Cron trigger** in n8n that fires every 15 minutes. Connect it to a **Database node** that runs:

```sql
SELECT * FROM social_posts
WHERE status = 'scheduled'
  AND scheduled_at <= datetime('now')
ORDER BY scheduled_at ASC
LIMIT 5;
```

This grabs any posts due in the current 15-minute window. Processing in batches prevents API rate-limit issues.

### Phase 2: Post to Each Platform

For each row returned, n8n splits the `platforms` field and sends posts to each platform's API. Here's the node configuration for the three most common platforms:

**Mastodon** (using the Mastodon API):

```json
{
  "method": "POST",
  "url": "https://your-mastodon-instance.com/api/v1/statuses",
  "headers": {
    "Authorization": "Bearer {{$env.MASTODON_TOKEN}}"
  },
  "body": {
    "status": "{{$json.caption}}",
    "visibility": "public"
  }
}
```

**Bluesky** (using the AT Protocol):

```json
{
  "method": "POST",
  "url": "https://bsky.social/xrpc/com.atproto.repo.createRecord",
  "headers": {
    "Authorization": "Bearer {{$env.BLUESKY_TOKEN}}"
  },
  "body": {
    "repo": "your-handle.bsky.social",
    "collection": "app.bsky.feed.post",
    "record": {
      "text": "{{$json.caption}}",
      "createdAt": "{{$now}}"
    }
  }
}
```

**LinkedIn** (using the Marketing API):

```json
{
  "method": "POST",
  "url": "https://api.linkedin.com/v2/ugcPosts",
  "headers": {
    "Authorization": "Bearer {{$env.LINKEDIN_TOKEN}}",
    "Content-Type": "application/json"
  },
  "body": {
    "author": "urn:li:organization:YOUR_COMPANY_ID",
    "lifecycle": "urn:li:lifecycle:published",
    "specificContent": {
      "com.linkedin.ugc.ShareContent": {
        "shareCommentary": {"text": "{{$json.caption}}"},
        "shareMediaCategory": "NONE"
      }
    },
    "visibility": {"com.linkedin.ugc.MemberNetworkVisibility": "PUBLIC"}
  }
}
```

### Phase 3: Update Status

After each post attempt, update the database:

```sql
UPDATE social_posts
SET status = 'posted',
    posted_at = datetime('now'),
    engagement_json = ?
WHERE id = ?;
```

If a platform API returns an error, set the status to `failed` and log the error message. You can build a second n8n workflow that monitors for failed posts and sends a notification to Mattermost or email.

Here's what the complete n8n workflow looks like:

```
[Cron: every 15 min]
        ↓
[SQLite: query scheduled posts]
        ↓
[Split: one item per post]
        ↓
[Function: parse platforms list]
        ↓
[Switch: route by platform]
   ├── [HTTP: Mastodon API] ──┐
   ├── [HTTP: Bluesky API]  ──┤
   └── [HTTP: LinkedIn API] ──┤
                               ↓
                        [SQLite: update status]
                               ↓
                        [Mattermost: notify if failed]
```

## Step 4: Track Engagement

Posting is only half the job. You also need to know which posts drive traffic and engagement. Rather than relying on each platform's analytics dashboard, consolidate everything.

### Web Traffic with Umami

If you're self-hosting [Umami analytics](https://www.ardotconsulting.com/blog/2026/12/11/self-hosting-umami-privacy-first-web-analytics-without-google-analytics/), you can tag each social post with a UTM-style parameter that Umami tracks:

```
https://www.ardotconsulting.com/blog/my-post?ref=social&src=mastodon
```

Umami's API lets you pull this data back into your database:

```python
import requests
from datetime import datetime, timedelta

def get_social_traffic(umami_url, umami_token, site_id, days=7):
    end = datetime.now()
    start = end - timedelta(days=days)

    response = requests.get(
        f"{umami_url}/api/websites/{site_id}/metrics",
        params={
            "start": int(start.timestamp() * 1000),
            "end": int(end.timestamp() * 1000),
            "type": "referrer"
        },
        headers={"Authorization": f"Bearer {umami_token}"}
    )

    return response.json()

traffic = get_social_traffic(
    umami_url="https://stats.yourdomain.com",
    umami_token="your-token",
    site_id="your-site-id"
)

for source in traffic:
    print(f"{source['name']}: {source['value']} visits")
```

### Platform Engagement

For platform-native engagement (likes, reposts, replies), build a second n8n workflow that runs daily and polls each platform's API:

```
[Cron: daily at 2 AM]
        ↓
[SQLite: get posts from last 7 days]
        ↓
[HTTP: fetch engagement per platform]
        ↓
[SQLite: update engagement_json]
```

This gives you a unified view of what's working across every platform, stored in a database you control.

## Step 5: Add a Content Approval Gate

If you're a solo operator, you might be fine with auto-posting. But if you have a team — or just want to review before things go live — add an approval step.

Insert a **Wait node** in your n8n workflow after caption generation but before posting. The workflow sends the drafted caption to Mattermost (or email) with approve/reject buttons:

```
[Generate caption with Ollama]
        ↓
[Mattermost: send draft for review]
        ↓
[Wait: for approval webhook]
        ↓
[Switch: approved or rejected]
   ├── approved → schedule post
   └── rejected → log and notify author
```

This way, AI does the heavy lifting of writing and formatting, but a human has the final say. No accidental posts, no tone-deaf captions going live at 3 AM.

## A Realistic Cost Comparison

Let's put numbers on this. Here's what a small business managing 3 social platforms with 2 team members would pay over two years:

| Item | Hootsuite (Team plan) | Open Source Stack |
|------|-----------------------|-------------------|
| Monthly subscription | $99/month ($2,376/2yr) | $0 |
| VPS for n8n + Ollama + Umami | — | $15/month ($360/2yr) |
| Domain + SSL | Included | $12/year ($24/2yr) |
| Setup time | 1 hour | 4–6 hours (one-time) |
| **2-year total** | **$2,376** | **$384** |

That's $1,992 saved over two years — and the gap widens over time because the SaaS price increases with features and seats while your self-hosted cost stays flat.

The open source stack also gives you things the SaaS tools charge extra for: unlimited scheduled posts, unlimited users, custom analytics, and AI caption generation without per-seat AI credits.

## Common Pitfalls (and How to Avoid Them)

**1. API token expiration.** Platform API tokens expire — LinkedIn's refresh every 60 days, Mastodon's can be revoked. Build a monitoring check in n8n that tests each token weekly and alerts you before it lapses.

**2. Rate limits.** Each platform limits how often you can post. X/Twitter allows 300 posts per 3 hours; LinkedIn allows 150 per day. Your n8n workflow should track these limits and queue posts rather than hammering the API.

**3. Image formatting.** Different platforms require different image dimensions. Use a tool like `sharp` (Node.js) or `Pillow` (Python) in a pre-processing step to auto-resize images before posting. Store the resized variants alongside your original.

**4. Timezone confusion.** Store all scheduled times in UTC in your database, and convert to the viewer's local timezone only at display time. This prevents posts from going live at the wrong hour when daylight saving changes.

**5. Posting the same content everywhere.** Just because you *can* cross-post identical content doesn't mean you should. Use the Ollama-generated variations from Step 2 so each platform gets content tailored to its culture and character limits.

## Taking It Further

Once the basic pipeline is running, you can extend it:

- **Content curation**: Build an n8n workflow that monitors RSS feeds from industry blogs, uses Ollama to summarize interesting articles, and drafts social posts about them — pending your approval.
- **Hashtag research**: Use Ollama to analyze trending hashtags on your platforms and suggest which ones to include in upcoming posts.
- **A/B testing**: Schedule two caption variants at different times and track which gets more engagement via Umami.
- **Repurposing**: Automatically convert a blog post into a thread of social posts — extract key points with Ollama, format them for each platform, and schedule across a week.

## Getting Started

If you already have n8n and Ollama running, you can build this pipeline in a single afternoon. Start small: get Mastodon posting working first, then add LinkedIn, then add the analytics layer. Each piece is independent, so you can iterate without breaking what already works.

The beauty of this approach is that it compounds. Every post you schedule through your own pipeline is a post you didn't pay a subscription to publish. Every caption generated by your local LLM is a request you didn't bill to an external API. And every engagement metric stored in your own database is data you can analyze however you want, without export limits or vendor lock-in.

If you want help setting up a social media automation pipeline for your business — from n8n workflow design to Ollama model selection to API integration — [reach out through our contact form](https://www.ardotconsulting.com/#contact). We'll help you get from manual posting to a fully automated, self-hosted social media stack.