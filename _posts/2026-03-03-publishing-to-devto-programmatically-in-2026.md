---
layout: post
title: "Publishing to dev.to Programmatically in 2026: What Actually Works"
date: 2026-03-03
categories: devto api tutorial
---

> Generated with [Claude](https://claude.ai) (Anthropic) based on a real session with [@ankitg12](https://dev.to/ankitg12).
>
> *Tags: #claude #ai #aiassisted*

---

This is the second post in a two-part series. The [first post](https://dev.to/ankitg12/getting-unix-tools-to-work-in-powershell-a-debugging-war-story-1ipl) covers getting Unix tools (`grep`, `ls -lt`, `sed`, `awk`) working in PowerShell — and cost ~102,000 tokens to figure out. Once the post was written, the natural next question was: can we skip the browser and publish it programmatically?

Turns out — yes, but not how you'd expect. Here's what we tried and what actually worked.

---

## Attempt 1: Medium API — Dead on Arrival

Medium was the first choice. They used to have a clean REST API. Not anymore.

> "Medium will not be issuing any new integration tokens for our API and will not allow any new integrations."

The GitHub repo was archived in **March 2023**. Existing tokens still work — but if you don't already have one, you can't get one. A paid membership doesn't help either.

**Workaround:** Publish to dev.to first, then use Medium's [Import a Story](https://medium.com/p/import) tool to pull the URL in. Medium even sets the canonical URL back to dev.to, which is good for SEO.

---

## Attempt 2: devto-cli — Almost

[`devto-cli`](https://github.com/sinedied/devto-cli) by sinedied is the right idea — a purpose-built npm CLI for publishing markdown files to dev.to:

```bash
npm install -g @sinedied/devto-cli
dev push article.md
```

Hit this immediately:

```
No GitHub repository provided.
Use --repo option or .env file to provide one.
```

`devto-cli` requires a GitHub repo because it rewrites relative image URLs to point to raw GitHub content. Smart for image hosting — but unnecessary overhead for a text-only post with no images.

Could pass `--repo username/repo` and move on, but at this point it felt like the tool was doing more than needed.

---

## Attempt 3: curl — Works, No Ceremony

dev.to has a straightforward REST API. The API key is still alive and well (unlike Medium's). A minimal test first:

```bash
curl -s -w "\nHTTP:%{http_code}" \
  -X POST https://dev.to/api/articles \
  -H "api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"article":{"title":"test","body_markdown":"hello","published":false}}'
```

Got `HTTP:201`. No firewall, no issues.

For the real article, build the JSON payload with Python (to handle escaping cleanly) and pass it via `--data @file`:

```python
import json

with open("article.md", encoding="utf-8") as f:
    content = f.read()

# Strip H1 title — dev.to uses the title field separately
body = "\n".join(content.split("\n")[2:]).strip()

payload = {
    "article": {
        "title": "Your Article Title",
        "body_markdown": body,
        "published": False,   # draft first
        "tags": ["powershell", "windows", "tutorial", "ai"]
    }
}

with open("payload.json", "w", encoding="utf-8") as f:
    json.dump(payload, f)
```

Then publish:

```bash
curl -s -X POST https://dev.to/api/articles \
  -H "api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  --data @payload.json
```

To update an existing draft (get the article ID from `/api/articles/me/unpublished`):

```bash
# Get draft IDs
curl -s -H "api-key: YOUR_API_KEY" \
  https://dev.to/api/articles/me/unpublished \
  | python3 -c "
import sys, json
for a in json.load(sys.stdin):
    print(a['id'], a['title'])
"

# Update draft
curl -s -X PUT https://dev.to/api/articles/ARTICLE_ID \
  -H "api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  --data @payload.json
```

---

## One Gotcha: Draft Preview URLs

The URL returned by the API is not directly accessible:

```
https://dev.to/username/article-slug-temp-slug-XXXX
```

To preview a draft, go to **dev.to/dashboard**, open the draft, and dev.to generates a preview URL with a token:

```
https://dev.to/username/slug?preview=<long_token>
```

This is expected behavior — drafts are private until published.

---

## The Full Flow

```
Write markdown locally
       ↓
python3 build_payload.py       # build payload.json
       ↓
curl --data @payload.json      # POST to dev.to (draft)
       ↓
Review on dev.to dashboard
       ↓
Hit Publish
       ↓
Import URL into Medium         # medium.com/p/import
```

---

## Summary

| Method | Status | Notes |
|---|---|---|
| Medium API | Dead | No new tokens since 2023 |
| devto-cli | Works | Needs GitHub repo even for no-image posts |
| curl + Python payload | Works | Simplest, no dependencies |
| Python `urllib` | 403 | SSL/TLS difference vs curl on Windows |
| Medium Import tool | Works | Pull from dev.to URL after publishing |

---

## Further Reading
- [dev.to API docs](https://developers.forem.com/api)
- [sinedied/devto-cli](https://github.com/sinedied/devto-cli) — good if you have images or want a Git-based workflow
- [Medium Import](https://medium.com/p/import) — the only reliable programmatic path into Medium today
- [Part 1: Getting Unix Tools to Work in PowerShell](https://dev.to/ankitg12/getting-unix-tools-to-work-in-powershell-a-debugging-war-story-1ipl) — the post that triggered this whole publishing adventure
