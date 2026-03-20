# SIGNAL — AI Tools Blog

> Every new AI tool reviewed, rated, and filed by industry. Updated daily.

**Live site:** `https://YOUR_USERNAME.github.io/signal-blog/`  
**Hosting cost:** £0 (GitHub Pages)  
**Running cost:** ~£1/year (Claude API)

---

## Repo structure

```
signal-blog/
├── index.html                  ← The site. Fetches tools.json at runtime.
├── tools.json                  ← The database. Updated daily by Make.com.
├── README.md                   ← This file.
└── .github/
    └── workflows/
        └── validate.yml        ← Validates tools.json on every push.
```

---

## One-time GitHub setup (15 minutes)

### 1. Create the repo

1. Go to **github.com → New repository**
2. Name it `signal-blog`
3. Set visibility to **Public** (required for free GitHub Pages)
4. Do not initialise with a README (you have one)

### 2. Push this code

```bash
cd signal-blog
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/signal-blog.git
git push -u origin main
```

### 3. Enable GitHub Pages

1. Go to your repo → **Settings → Pages**
2. Source: **Deploy from a branch**
3. Branch: `main` / `/ (root)`
4. Click **Save**
5. Your site is live at `https://YOUR_USERNAME.github.io/signal-blog/` within 60 seconds.

### 4. Create a GitHub Personal Access Token (for Make.com)

1. GitHub → **Settings → Developer Settings → Personal Access Tokens → Fine-grained tokens**
2. Click **Generate new token**
3. Name: `SIGNAL Make.com`
4. Repository access: **Only select repositories → signal-blog**
5. Permissions:
   - **Contents**: Read and Write
   - **Actions**: Read (optional, for triggering workflows)
6. Copy the token — you will not see it again. Save it somewhere safe.

---

## Make.com automation setup

### What you need first

| Thing | Where to get it | Cost |
|---|---|---|
| Make.com account | make.com | Free |
| Product Hunt API token | api.producthunt.com/v2/oauth/applications | Free |
| Claude API key | console.anthropic.com | Pay-per-use (~£1/year at 1 tool/day) |
| GitHub token | See step 4 above | Free |
| Google account (for Sheets) | Google | Free |

---

### The Make.com scenario — step by step

Create a new scenario called **SIGNAL Daily Publisher**.

---

#### Module 1 — Schedule (Trigger)

- **Module:** Schedule
- **Run every:** 1 Day
- **Time:** 08:00 UTC

---

#### Module 2 — Fetch today's top AI tool from Product Hunt

- **Module:** HTTP → Make a request
- **URL:** `https://api.producthunt.com/v2/api/graphql`
- **Method:** POST
- **Headers:**
  - `Authorization`: `Bearer YOUR_PRODUCT_HUNT_TOKEN`
  - `Content-Type`: `application/json`
- **Body (raw JSON):**

```json
{
  "query": "{ posts(order: VOTES, topic: \"artificial-intelligence\", first: 5, featured: true) { edges { node { name tagline website topics { edges { node { name } } } } } } }"
}
```

- **Parse response:** Yes — this gives you `data.posts.edges[0].node`

---

#### Module 3 — Check Google Sheet: already published?

- **Module:** Google Sheets → Search Rows
- **Spreadsheet:** SIGNAL Master Sheet (create a Google Sheet with column `domain`)
- **Filter:** `domain` = `{{2.data.posts.edges[].node.website | replace("https://","") | replace("http://","") | split("/")[0]}}`
- **If found:** Router → Stop (prevents duplicates)
- **If not found:** Continue to Module 4

---

#### Module 4 — Fetch the tool's homepage (enrichment)

- **Module:** HTTP → Make a request
- **URL:** `{{2.data.posts.edges[0].node.website}}`
- **Method:** GET
- **Parse response:** No (raw HTML is fine — just pass it to Claude for context)
- **Timeout:** 10 seconds
- **Ignore errors:** Yes (some sites block bots)

---

#### Module 5 — Claude API assessment

- **Module:** HTTP → Make a request
- **URL:** `https://api.anthropic.com/v1/messages`
- **Method:** POST
- **Headers:**
  - `x-api-key`: `YOUR_CLAUDE_API_KEY`
  - `anthropic-version`: `2023-06-01`
  - `Content-Type`: `application/json`
- **Body (raw JSON):**

```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 800,
  "messages": [
    {
      "role": "user",
      "content": "You are the editorial AI for SIGNAL, a daily AI tools blog. Assess this tool and return ONLY valid JSON — no other text, no markdown, no code fences.\n\nTool name: {{2.data.posts.edges[0].node.name}}\nTagline from Product Hunt: {{2.data.posts.edges[0].node.tagline}}\nWebsite: {{2.data.posts.edges[0].node.website}}\n\nReturn this exact JSON structure:\n{\n  \"name\": \"tool name\",\n  \"domain\": \"example.com\",\n  \"tagline\": \"one sharp sentence, no fluff, no marketing language\",\n  \"problem\": \"2-3 sentences on what specific business problem this solves, written for a working professional\",\n  \"useCases\": [\"use case 1\", \"use case 2\", \"use case 3\", \"use case 4\"],\n  \"url\": \"https://example.com\",\n  \"ease\": 7,\n  \"easeLabel\": \"Easy to pick up\",\n  \"industry\": [\"Industry 1\", \"Industry 2\"],\n  \"businessArea\": \"e.g. Content Production\",\n  \"category\": \"one of: Search & Research, Video & Audio, Writing & Docs, Code & Dev, Sales & CRM, Legal & Compliance, Productivity, Design & Visual, Data & Analytics\"\n}\n\nEase rating rules:\n10 = anyone uses it day one, zero learning curve\n8-9 = intuitive, up and running in minutes\n6-7 = some familiarity with the domain helps\n4-5 = requires domain knowledge or configuration\n1-3 = developer or specialist only\n\nBe precise. Be brief. Business professional tone."
    }
  ]
}
```

- **Parse response:** Yes
- **Extract:** `data.content[0].text` — this is the raw JSON string Claude returns

---

#### Module 6 — Parse Claude's JSON response

- **Module:** JSON → Parse JSON
- **JSON string:** `{{5.data.content[0].text}}`

This turns Claude's text output into a usable Make.com data bundle.

---

#### Module 7 — Log to Google Sheet

- **Module:** Google Sheets → Add a Row
- **Spreadsheet:** SIGNAL Master Sheet
- **Sheet:** `published`
- **Columns:**
  - `id` → use a counter or `{{now | formatDate("YYYYMMDDHHmmss"}}`
  - `name` → `{{6.name}}`
  - `domain` → `{{6.domain}}`
  - `status` → `approved` (or `pending` if you want to review before publishing)
  - `date_added` → `{{now | formatDate("YYYY-MM-DD")}}`
  - All other fields from Module 6

---

#### Module 8 — Read current tools.json from GitHub

- **Module:** HTTP → Make a request
- **URL:** `https://api.github.com/repos/YOUR_USERNAME/signal-blog/contents/tools.json`
- **Method:** GET
- **Headers:**
  - `Authorization`: `Bearer YOUR_GITHUB_TOKEN`
  - `Accept`: `application/vnd.github+json`

This returns the file content (base64 encoded) and crucially the `sha` field you need for the next step.

---

#### Module 9 — Build updated tools.json

- **Module:** Tools → Set Variable (or use a custom JSON builder)

Take the existing content from Module 8, decode it, parse it, append the new tool object from Module 6, re-serialise, and base64-encode the result.

Use a **Code module** (Make.com supports JavaScript):

```javascript
// Decode existing file
const existing = JSON.parse(
  Buffer.from(input.currentContent, 'base64').toString('utf8')
);

// Build new tool entry
const newTool = {
  id: existing.length + 1,
  name: input.name,
  domain: input.domain,
  tagline: input.tagline,
  problem: input.problem,
  useCases: JSON.parse(input.useCases),
  url: input.url,
  ease: parseInt(input.ease),
  easeLabel: input.easeLabel,
  industry: JSON.parse(input.industry),
  businessArea: input.businessArea,
  category: input.category,
  date: new Date().toISOString().split('T')[0]
};

// Prepend (newest first) and re-encode
const updated = [newTool, ...existing];
return {
  encoded: Buffer.from(JSON.stringify(updated, null, 2)).toString('base64')
};
```

---

#### Module 10 — Push updated tools.json to GitHub

- **Module:** HTTP → Make a request
- **URL:** `https://api.github.com/repos/YOUR_USERNAME/signal-blog/contents/tools.json`
- **Method:** PUT
- **Headers:**
  - `Authorization`: `Bearer YOUR_GITHUB_TOKEN`
  - `Content-Type`: `application/json`
- **Body (raw JSON):**

```json
{
  "message": "Daily tool: {{6.name}}",
  "content": "{{9.encoded}}",
  "sha": "{{8.sha}}"
}
```

GitHub Pages redeploys automatically within ~60 seconds. The site is live.

---

## The Claude prompt (standalone reference)

Use this prompt exactly when calling the Claude API. It produces clean, consistent JSON every time:

```
You are the editorial AI for SIGNAL, a daily AI tools blog.
Assess this tool and return ONLY valid JSON — no other text, no markdown, no code fences.

Tool name: [NAME]
Website: [URL]
Tagline: [TAGLINE FROM PRODUCT HUNT]

Return this exact structure:
{
  "name": "",
  "domain": "example.com",
  "tagline": "one sharp sentence, no fluff",
  "problem": "2-3 sentences on what problem this solves, written for a working professional",
  "useCases": ["use case 1", "use case 2", "use case 3", "use case 4"],
  "url": "https://...",
  "ease": 7,
  "easeLabel": "Easy to pick up",
  "industry": ["Industry 1", "Industry 2"],
  "businessArea": "e.g. Content Production",
  "category": "one of: Search & Research, Video & Audio, Writing & Docs, Code & Dev, Sales & CRM, Legal & Compliance, Productivity, Design & Visual, Data & Analytics"
}

Ease rating scale:
10 = anyone, day one, zero learning curve
8-9 = intuitive, up and running in minutes  
6-7 = some domain familiarity helps
4-5 = requires domain knowledge or configuration
1-3 = developer or specialist only

Tone: precise, brief, professional. No marketing language.
```

---

## Backfilling to 500 tools

To reach 500 tools faster than 365 days:

1. Export tool lists from **futurepedia.io**, **theresanaiforthat.com**, or **aitoolhunt.com** as CSV
2. Upload CSV to Google Sheets as a second sheet called `backfill_queue`
3. Change Module 1 from a Schedule trigger to a **Google Sheets → Watch Rows** trigger
4. Point it at the `backfill_queue` sheet
5. Run the scenario manually at high frequency (Make.com allows manual runs)
6. Each row gets assessed by Claude and appended to tools.json

At 10 tools/day backfill rate: 500 tools in ~7 weeks.

---

## Cost summary

| Item | Cost |
|---|---|
| GitHub Pages hosting | £0 |
| Make.com (free tier, ~360 ops/month) | £0 |
| Product Hunt API | £0 |
| Claude Haiku (365 assessments × $0.001) | ~£0.30/year |
| Custom domain (optional) | ~£10/year |
| **Total** | **< £11/year** |

---

## Questions / next steps

- To add a custom domain: GitHub Pages → Custom domain → add your CNAME
- To add a review queue: change `status` default to `pending` in Module 7, add a filter in Module 10 that only pushes `approved` rows
- To add email alerts when a new tool publishes: add a Gmail module after Module 10

---

*Built with Make.com + Claude API + GitHub Pages.*
