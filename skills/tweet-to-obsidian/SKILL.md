---
name: tweet-to-obsidian
description: >
  Capture a tweet (or X post) as a fully-formed Obsidian note. Navigates to the
  tweet, extracts all text, metadata, quoted tweets, and stats, downloads every
  media asset (images in highest available resolution, profile picture, video
  thumbnails), resolves all shortened links, scrapes linked articles for OG
  images and descriptions, and writes an Obsidian-optimised .md file with YAML
  frontmatter, [[wikilinks]], and ![[embedded images]].
  Triggers: "save this tweet to obsidian", "tweet to markdown", "capture tweet",
  "archive this post", "tweet note", "x post to obsidian".
allowed-tools: Bash(agent-browser:*), Bash(curl:*), Bash(mkdir:*), Bash(ls:*)
---

# tweet-to-obsidian

Capture a tweet as a rich, portable Obsidian note — including all media, linked
articles, and metadata — in a single automated workflow.

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| **Tweet URL** | *(required)* | Full `https://x.com/…/status/…` URL |
| **Output directory** | `./` (current dir) | Where to write the `.md` file |
| **Assets directory** | `{output}/assets/{slug}/` | Where to save downloaded media |
| **Vault path** | *(none)* | Override output dir with an Obsidian vault path |

If the user just pastes a tweet URL with no other context, start immediately
with defaults. Do not ask clarifying questions.

Always use `agent-browser` directly — never `npx agent-browser`.

---

## Workflow

```
1. Setup        Derive slug, create asset directory
2. Navigate     Open tweet, wait for content
3. Extract      Snapshot article element, capture all text + metadata
4. Media        Download images (high-res), profile picture
5. Links        Resolve t.co short links, scrape linked articles
6. Articles     Screenshot linked pages, download OG images
7. Write        Generate Obsidian .md with all gathered content
8. Cleanup      Close browser session
```

---

### 1. Setup

Derive the tweet slug from the status ID and author handle:

```bash
# Example: https://x.com/khoiracle/status/2016718807021343015
# slug = khoiracle-2016718807021343015
TWEET_URL="{TWEET_URL}"
HANDLE=$(echo "$TWEET_URL" | sed 's|.*x.com/||;s|/status.*||')
STATUS_ID=$(echo "$TWEET_URL" | sed 's|.*/status/||;s|[?/].*||')
SLUG="${HANDLE}-${STATUS_ID}"

OUTPUT_DIR="{OUTPUT_DIR:-.}"
ASSET_DIR="${OUTPUT_DIR}/assets/tweet-${SLUG}"
NOTE_PATH="${OUTPUT_DIR}/tweet-${SLUG}.md"

mkdir -p "${ASSET_DIR}"
```

---

### 2. Navigate

```bash
agent-browser open "${TWEET_URL}" && agent-browser wait 4000
```

If `wait --load networkidle` times out (X.com often does), continue — the
content is usually rendered by the time the 4s timer completes.

Take an initial full-page screenshot for reference:

```bash
agent-browser screenshot --full "${ASSET_DIR}/page-snapshot.png"
```

---

### 3. Extract

Use `snapshot -s "article"` to scope directly to the tweet article element —
this is reliably present on every tweet permalink page and contains all the
structured data we need:

```bash
agent-browser snapshot -s "article"
```

Parse the output to collect:

| Field | Source |
|-------|--------|
| `author_name` | link text of the author link |
| `handle` | text of `@handle` link |
| `date` | `time` element text inside the timestamp link |
| `tweet_text` | plain `text:` nodes inside the article |
| `quoted_author` | nested author name (if present) |
| `quoted_handle` | nested @handle (if present) |
| `quoted_date` | nested timestamp (if present) |
| `quoted_text` | nested text content (if present) |
| `links` | all `link` elements with external `/url:` values |
| `image_refs` | all `link "Image"` elements with photo URLs |
| `stats` | `group "N replies, N reposts, N likes…"` text |
| `views` | `link "N Views"` or `link "N.NK Views"` text |

Also extract the image src URLs via JavaScript for full resolution:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(
  Array.from(document.querySelectorAll('img'))
    .filter(i => i.src && i.src.includes('pbs.twimg.com/media'))
    .map(i => ({
      src: i.src.replace(/name=\w+/, 'name=large'),
      alt: i.alt
    }))
)
EVALEOF
```

And the profile picture:

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(
  Array.from(document.querySelectorAll('img'))
    .filter(i => i.src && i.src.includes('profile_images'))
    .map(i => ({
      src: i.src.replace(/_normal\./, '_400x400.'),
      handle: (document.querySelector('a[href^="/"][role="link"] span') || {}).textContent
    }))[0]
)
EVALEOF
```

---

### 4. Media

Download all tweet images at highest available resolution (`name=large`).
Replace `name=small` or `name=medium` with `name=large` in every image URL:

```bash
# For each image URL found in step 3:
curl -sL "{IMAGE_URL_large}" -o "${ASSET_DIR}/image-{N}.jpg"

# Profile picture (replace _normal. with _400x400.)
curl -sL "{PROFILE_PIC_URL}" -o "${ASSET_DIR}/avatar-${HANDLE}.jpg"
```

Run all downloads in parallel with `&` and `wait`:

```bash
curl -sL "{IMG_1}" -o "${ASSET_DIR}/image-1.jpg" &
curl -sL "{IMG_2}" -o "${ASSET_DIR}/image-2.jpg" &
curl -sL "{AVATAR}" -o "${ASSET_DIR}/avatar-${HANDLE}.jpg" &
wait
```

---

### 5. Resolve Links

For every `t.co` short link found in the tweet, resolve to the final URL:

```bash
# Resolve t.co -> real URL (follows all redirects, prints final Location)
curl -sIL "{TCO_URL}" 2>&1 | grep -i "^location:" | tail -1 | awk '{print $2}' | tr -d '\r'
```

Store the mapping: `t.co/xxx → https://real-domain.com/path`.

---

### 6. Scrape Linked Articles

For every resolved external URL (not x.com, not twitter.com):

**a. Screenshot the page:**

```bash
agent-browser open "{ARTICLE_URL}" && agent-browser wait --load networkidle
agent-browser screenshot --full "${ASSET_DIR}/article-{DOMAIN}.png"
```

**b. Extract OG metadata:**

```bash
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify({
  title:       document.querySelector('meta[property="og:title"]')?.content
            || document.querySelector('title')?.textContent,
  description: document.querySelector('meta[property="og:description"]')?.content
            || document.querySelector('meta[name="description"]')?.content,
  image:       document.querySelector('meta[property="og:image"]')?.content,
  siteName:    document.querySelector('meta[property="og:site_name"]')?.content,
  url:         document.querySelector('meta[property="og:url"]')?.content
            || window.location.href
})
EVALEOF
```

**c. Download OG image:**

```bash
curl -sL "{OG_IMAGE_URL}" -o "${ASSET_DIR}/article-{DOMAIN}-og.png"
```

**d. Get page text summary (first 2000 chars):**

```bash
agent-browser get text body 2>/dev/null | head -c 2000
```

After scraping each article, return to the tweet URL before scraping the next
one (or use `--session` to keep tabs isolated):

```bash
agent-browser open "${TWEET_URL}" && agent-browser wait 2000
```

---

### 7. Write Obsidian Note

Use the template at `{SKILL_DIR}/templates/note-template.md` as the base, then
fill in all collected data. Write the final file directly with the Write tool.

**YAML frontmatter rules:**
- `date:` — ISO format `YYYY-MM-DD` extracted from the tweet timestamp
- `tags:` — derive from tweet content (topics, tools, people mentioned)
- `media:` — list all downloaded asset paths relative to the output directory
- `links:` — array of `{url, title}` objects for every resolved external URL
- `stats:` — `views`, `likes`, `replies`, `reposts`, `bookmarks`

**Body rules:**
- Quote the tweet text verbatim in a `>` blockquote
- Embed the profile picture inline: `![[assets/.../avatar-handle.jpg|48]]`
- Embed each tweet image: `![[assets/.../image-N.jpg]]`
- For each linked article, include a named section with:
  - The OG image embedded: `![[assets/.../article-domain-og.png]]`
  - Bold title as a link: `**[Title](url)**`
  - One-line description
  - Feature table if the page has structured feature content
- Add a `## Context & Notes` section with analytical observations
- Add a `## Related` section with `[[Wikilinks]]` to key concepts

**Wikilink guidelines:**
- People → `[[First Last]]`
- Products/tools → `[[Tool Name]]`
- Concepts → `[[concept]]`
- Do NOT wikilink generic words (the, and, a, etc.)

See `{SKILL_DIR}/templates/note-template.md` for the full structure.

---

### 8. Cleanup

```bash
agent-browser close
```

Then print the output path and a brief summary:

```
✓ Note written to: {NOTE_PATH}
✓ Assets saved to: {ASSET_DIR}/
  - {N} images downloaded
  - {N} article(s) scraped
  - {N} links resolved
```

---

## Edge Cases

| Situation | Handling |
|-----------|----------|
| Tweet has no images | Skip media download steps; omit media section from note |
| Tweet is a thread | Snapshot each subsequent tweet in the thread; add a `## Thread` section |
| Tweet has a video | Download the poster/thumbnail image; note the video URL in frontmatter |
| `t.co` link resolves to x.com | Skip article scraping for that link |
| Linked article blocks bots | Use screenshot only; skip text extraction |
| Profile is private / tweet 404 | Stop and report the error clearly to the user |
| `networkidle` times out | Continue after 4s wait — X rarely reaches networkidle |
| Image URL returns 403 | Try `name=medium` then `name=small` as fallback |

---

## Quality Checklist

Before writing the final note, verify:

- [ ] All `t.co` links have been resolved to real URLs
- [ ] At least one image downloaded (if the tweet had images)
- [ ] Profile picture downloaded
- [ ] YAML frontmatter is valid (no unquoted colons in string values)
- [ ] All `![[embeds]]` reference files that actually exist in `ASSET_DIR`
- [ ] `## Related` section has at least 3 wikilinks
- [ ] `date:` field matches the tweet timestamp
