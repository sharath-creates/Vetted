# n8n Build Guide — Trending Meme Content Engine (Phase 1 + Creative Loop)

**Companion to:** meme-content-engine-prd.md
**Goal:** Stand up the daily trend engine, then bolt on captioning (Azure AI Foundry) and rendering, ending with mockups on a ClickUp task.

Build in two passes. Pass A is the trend engine alone — get it producing a sane top-3 before you spend a cent on captions or renders. Pass B adds the creative loop.

---

## 0. Prerequisites

Set these up before opening n8n.

| Thing | Why | Cost | Notes |
|---|---|---|---|
| n8n instance | Runs the flow | $0 self-hosted (Docker), or free cloud trial | Self-host via Docker for an always-free personal setup |
| Imgflip account | `get_memes` (read) + `caption_image` (render) | $0 | Username + password are passed to the render API |
| Giphy developer key | Leading-indicator pull (trending) | $0 | Register an app at developers.giphy.com → copy the API key. No OAuth |
| Tenor (Google) key *(optional backup)* | Second trending signal | $0 | Get a key from Google Cloud → Tenor API |
| Google account + one Sheet | Snapshot state store | $0 | Two tabs: `snapshots`, `used_list` |
| Azure AI Foundry deployment | Caption generation | Pay-per-token (small) | Deploy a cheap chat model (e.g. a 4o-mini-class model). Copy endpoint, deployment name, api-version, key from the deployment page |
| ClickUp account | Brief intake (+ optional review board) | $0 tier | Webhook for intake; personal API token only if you want n8n to write results back |

> **Reddit dropped from the MVP.** Reddit closed new OAuth/Data API access around Nov 2025, so new personal script apps generally can't get working credentials. Giphy replaces it as the early-trend signal. An optional Reddit **RSS** branch is described in §2.3b if you want to test whether the no-auth feed still works for you.

Keep a scratch note with: Imgflip user/pass, Giphy key, (optional) Tenor key, Sheet ID, Azure endpoint URL + deployment + api-version + key, ClickUp webhook URL + (optional) API token + list ID.

---

## 1. Credentials in n8n

Create these under **Credentials** so nodes can reference them:

1. **Google Sheets OAuth2** — connect your Google account.
2. **Azure** — you'll call Foundry via an HTTP Request node with a header-auth credential: create a **Header Auth** credential named `azure-foundry`, header name `api-key`, value = your Foundry key. (Alternatively use n8n's native *Azure OpenAI Chat Model* node if your version has it.)
3. **ClickUp API** *(optional)* — paste the personal token, only if you want n8n to write results back into ClickUp. Intake uses a webhook and needs no credential.

Imgflip, Giphy, and Tenor need no stored credential — their keys/params ride in the request itself.

---

## 1.5 Intake — receive the landing-page form

`brief-intake-form.html` POSTs `{ company, product, persona, geography, submitted_at }` as JSON to an n8n webhook. Build this small intake chain; its output is the `brief` the caption step reads.

### 1.5.1 Webhook node
- **Webhook** node, name `brief_intake`.
- HTTP Method **POST**, Path something like `brief-intake`. Copy the **Production URL** into the form's `WEBHOOK_URL`.
- Respond: set "Respond" to **Immediately** (or use a **Respond to Webhook** node) so the form's `fetch` gets a fast 200.
- **CORS:** in the node's options, set **Allowed Origins (CORS)** to your landing page's domain (e.g. `https://yoursite.com`), not `*`. Without this the browser blocks the POST. The custom `X-Form-Secret` header also makes the browser send a preflight `OPTIONS`; current n8n answers it automatically once Allowed Origins is set.

### 1.5.2 Secret check (reject spam)
- **IF** node, name `check_secret`.
- Condition: `{{ $json.headers['x-form-secret'] }}` **equals** your `FORM_SECRET` value.
- True branch → continue. False branch → leave unconnected (or wire a **Respond to Webhook** returning 401). This stops random POSTs before they reach the Azure caption calls.

> n8n exposes incoming headers on the webhook item. Depending on version they're at `$json.headers` or `$node["brief_intake"].json.headers`. Check the node's output once and reference accordingly.

### 1.5.3 Normalize to `brief`
- **Set** node, name `brief`. Map the body through so downstream nodes can reference `$('brief').first().json.<field>`:
  - `company` = `{{ $json.body.company }}`
  - `product` = `{{ $json.body.product }}`
  - `persona` = `{{ $json.body.persona }}`
  - `geography` = `{{ $json.body.geography }}`
- (JSON bodies land under `body` on the webhook item; if your version puts them at the top level, drop the `.body`.)

This `brief` node is the same one the caption prompt in §3.1 references. Two ways to connect intake to the creative loop:

- **On-demand:** the webhook triggers a run that uses the latest trend snapshot already in your Sheet (fast, no waiting for the daily pull).
- **Daily batch:** store incoming briefs in a Sheet tab and let the scheduled run pick them up. Simpler but the requester waits until the next run.

For the MVP, on-demand feels better: a brief comes in, you grab the current top-3, caption, render, deliver.

---

## 2. Pass A — Trend Engine (build and verify first)

Lay the nodes left to right. Node types are in **bold**.

### 2.1 Schedule Trigger
- **Schedule Trigger** node. Set to run once daily (e.g. 07:00). For development, trigger manually with the **Execute Workflow** button so you don't wait a day between tests.

### 2.2 Fetch Imgflip
- **HTTP Request** node, name it `imgflip_get_memes`.
- Method **GET**, URL `https://api.imgflip.com/get_memes`.
- No auth. Response is JSON: `data.memes[]` with `id`, `name`, `url`, `box_count`.

### 2.3 Fetch Giphy trending (replaces Reddit)
- **HTTP Request** node, name it `giphy_trending`.
- Method **GET**, URL `https://api.giphy.com/v1/gifs/trending`.
- Query params: `api_key` = your Giphy key, `limit` = `25`, `rating` = `pg-13`.
- Response is `data[]`, each with `id`, `title`, `images.original.url`, and `trending_datetime`. Giphy returns these already ordered by trending strength, so array position is your rank signal (index 0 = hottest).

### 2.3b Optional — Reddit RSS branch (no app, no OAuth)
Only if you want to test Reddit's free feed:
- **RSS Read** node, URL `https://www.reddit.com/r/MemeTemplates/top/.rss?t=day`. No auth. If it returns items, merge them in §2.4; if it 403s or empties, leave this branch out. (Re-check Reddit's terms; RSS access can change.)

### 2.4 Normalize the sources
- **Code** node, name `normalize`. Map every source into one common shape so scoring doesn't care where a format came from.

```javascript
// Inputs: imgflip + giphy (+ optional tenor/reddit-rss) reach this node.
// Adjust $('nodeName') references to your actual node names.
const out = [];
const today = new Date().toISOString().slice(0, 10);

// Imgflip: rank = array position (0 = most used)
const imgflip = $('imgflip_get_memes').first().json.data.memes || [];
imgflip.forEach((m, i) => {
  out.push({
    json: {
      source: 'imgflip',
      format_id: 'imgflip_' + m.id,
      name: m.name,
      image_url: m.url,
      signal: imgflip.length - i, // higher = hotter
      raw_rank: i,
      date: today,
    },
  });
});

// Giphy: also rank-ordered, so invert position into a signal
const giphy = ($('giphy_trending').first().json.data) || [];
giphy.forEach((g, i) => {
  out.push({
    json: {
      source: 'giphy',
      format_id: 'giphy_' + g.id,
      name: g.title || ('giphy ' + g.id),
      image_url: g.images && g.images.original ? g.images.original.url : g.url,
      signal: giphy.length - i,
      raw_rank: i,
      date: today,
    },
  });
});

return out;
```

> If you wire Tenor or the Reddit-RSS branch, add a matching block here mapping each into the same `{ source, format_id, name, image_url, signal, raw_rank, date }` shape.

### 2.5 Read yesterday's snapshot
- **Google Sheets** node `read_snapshot`: Operation **Read Rows** from tab `snapshots`. This returns all prior rows; you'll match yesterday's by `format_id` in the next node.

### 2.6 Velocity score
- **Code** node `score`. Acceleration = today's signal vs the same format's last signal. New-but-rising formats win.

```javascript
const prior = $('read_snapshot').all().map(r => r.json);
const priorBy = {};
prior.forEach(p => { priorBy[p.format_id] = Number(p.signal) || 0; });

const today = new Date().toISOString().slice(0, 10);

return $input.all().map(item => {
  const j = item.json;
  const now = Number(j.signal) || 0;
  const was = priorBy[j.format_id]; // undefined = brand new
  const accel = was === undefined ? now : now - was;     // movement
  const freshness = was === undefined ? 1.25 : 1.0;       // reward newcomers
  const velocity = (accel * 0.7 + now * 0.3) * freshness; // movement-weighted
  return { json: { ...j, velocity, was: was ?? null, date: today } };
});
```

### 2.7 Write today's snapshot
- **Google Sheets** node `write_snapshot`: Operation **Append** to tab `snapshots`. Map `format_id`, `name`, `signal`, `velocity`, `date`. This is load-bearing — without it, tomorrow has nothing to accelerate against.

### 2.8 Drop stale/used formats
- **Google Sheets** node `read_used`: Read tab `used_list` (format_ids you've already shipped or flagged dead).
- **Filter** node: keep items whose `format_id` is **not** in the used list. (Use a small Code node to build the exclusion set if the Filter UI is awkward.)

### 2.9 Top 3
- **Sort** node: by `velocity` descending.
- **Limit** node: keep 3.

**Verify Pass A before continuing.** Run it manually two days in a row (or fake yesterday's rows in the Sheet). Eyeball the top 3: are they formats that *moved up*, not just sitting high? If a stale evergreen keeps winning, raise the `accel` weight in §2.6. Only move on once the picks look genuinely rising.

---

## 3. Pass B — Creative Loop

Append these after the Limit node. The active brief comes from the `brief` Set node built in §1.5.3 (fed by the landing-page form). For isolated testing, a manual **Set** node holding `{company, persona, geography, product}` works the same.

### 3.1 Build the caption prompt
- **Set** (or **Code**) node `prompt`. Construct the message per format:

```
System: You write meme captions. Match the meme format's known narrative. Keep it tight, native to the format, and in the audience's voice. No hashtags.

User:
Format: {{name}}
Product: {{product}}
Audience persona: {{persona}}
Market/language: {{geography}}
Task: Write the caption for this meme so the format's joke lands AND naturally shows {{product}} solving a real use-case for this persona. Return top-text and bottom-text as JSON: {"text0": "...", "text1": "..."}.
```

### 3.2 Caption with Azure AI Foundry
- **HTTP Request** node `foundry_caption`.
- Method **POST**.
- URL: your Foundry chat-completions endpoint — it looks like
  `https://<your-resource>.openai.azure.com/openai/deployments/<deployment-name>/chat/completions?api-version=<version>`
  Copy the exact resource, deployment name, and api-version from your Foundry deployment page; don't guess them.
- Authentication: the `azure-foundry` Header Auth credential (`api-key`).
- Body (JSON):

```json
{
  "messages": [
    { "role": "system", "content": "{{ $json.system }}" },
    { "role": "user", "content": "{{ $json.user }}" }
  ],
  "temperature": 0.8,
  "response_format": { "type": "json_object" }
}
```

- Parse `choices[0].message.content` (it's JSON text) in a small **Code** node into `text0` / `text1`.

> If your n8n build has the native **Azure OpenAI Chat Model** node, use it instead of HTTP — set the same resource/deployment/api-version and feed it the prompt. The HTTP route works on any version.

### 3.3 Render the meme
- **HTTP Request** node `imgflip_caption`.
- Method **POST**, URL `https://api.imgflip.com/caption_image`.
- Body (form-urlencoded): `template_id` (the Imgflip numeric id — only valid for Imgflip-sourced picks), `username`, `password`, `text0`, `text1`.
- Response gives `data.url` (the rendered image).

> Giphy/Tenor-sourced formats have no Imgflip `template_id`. For the MVP, either restrict rendering to Imgflip-sourced picks, or upload the chosen template to Imgflip once and map its id. Flag non-renderable picks for manual handling.

### 3.4 Deliver for review
ClickUp webhooks only fire **out** of ClickUp, so they handle intake, not delivery. Pick one delivery sink:

- **No ClickUp API (webhook-only setup):** write the 3 rendered `data.url`s to a Google Sheet row (or email them to yourself, or post to Telegram/Slack). You approve there. Simplest with what you have.
- **With ClickUp personal token:** use the **ClickUp** node to attach each `data.url` to the brief's task and set status to **Needs Review**. The token is free (Settings → Apps → API Token) and keeps review inside ClickUp.

Either way, you approve/reject manually. That's the human gate. Publishing stays out of scope for now.

### Intake (how the brief arrives)
The brief comes from `brief-intake-form.html` on your landing page, which POSTs to the `brief_intake` webhook built in **§1.5**. That chain (Webhook → secret check → `brief` Set node) is the intake; nothing else to do here.

---

## 4. Build & Test Sequence

1. Wire and run **§2.1–2.4** only. Confirm `normalize` outputs a clean merged list.
2. Add the Sheet read/write (**§2.5–2.7**). Run twice; confirm `snapshots` grows and `velocity` computes (second run shows non-null `was`).
3. Add filter + sort + limit (**§2.8–2.9**). Confirm a believable top 3.
4. Add a manual **Set** brief and the caption node (**§3.1–3.2**). Inspect captions before rendering — cheap to iterate here.
5. Add render + delivery (**§3.3–3.4**). Confirm 3 images land in your review sink.
6. Wire the intake chain (**§1.5**). Submit the real `brief-intake-form.html` and confirm the webhook fires, the secret check passes, and `brief` populates. Send one POST without the secret header and confirm it's rejected.
7. Switch the Schedule Trigger to daily and let it run unattended; review each morning.

---

## 5. Cost & Guardrails

- Imgflip, Giphy, Tenor, Google Sheets: $0.
- Azure AI Foundry: the only real spend — a handful of caption calls per brief, small on a 4o-mini-class model. Cap it by generating captions only for the final 3, never the whole candidate pool.
- **Reddit is out of the MVP** — new personal OAuth/Data API credentials are effectively closed since Nov 2025. The optional RSS branch (§2.3b) is the only no-cost way to test a Reddit signal; treat it as best-effort.
- Giphy's free key has hourly rate limits; a once-daily pull is well under them. Move to a production key only if you raise frequency.
- Confirm Imgflip, Giphy, and Azure endpoint/api-version values at build time; they drift.
```
