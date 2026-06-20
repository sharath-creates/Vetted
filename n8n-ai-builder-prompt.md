# Prompt for n8n AI Builder — Trending Meme Content Engine

> Paste everything below the line into the n8n AI assistant. If it struggles with the whole thing at once, build it in the two parts marked **PART A** and **PART B** as separate prompts.

---

Build an n8n workflow called **"Trending Meme Content Engine"**. It runs daily, finds meme formats that are trending upward, scores them by momentum, picks the top 3, then writes a product-aware meme caption for each, renders the meme, and posts the results to ClickUp for human review. Keep cost near zero. Use only the sources I name.

Here are the credentials I have configured (assume they exist): Google Sheets OAuth2, a Header Auth credential named `azure-foundry` (header `api-key`), and optionally a ClickUp API credential. Imgflip and Giphy need no stored credential — Imgflip passes username/password as request params, Giphy passes its api_key as a query param. Do NOT use Reddit; new personal Reddit API access is closed.

## PART A — Trend Engine

**1. Schedule Trigger**
Run once per day at 07:00.

**2. HTTP Request — name it `imgflip_get_memes`**
GET `https://api.imgflip.com/get_memes`. No auth. The response has `data.memes[]`, each with `id`, `name`, `url`, `box_count`. Treat array order as popularity rank (index 0 = most used).

**3. HTTP Request — name it `giphy_trending`**
GET `https://api.giphy.com/v1/gifs/trending` with query params `api_key` (my Giphy key, leave a placeholder), `limit=25`, `rating=pg-13`. The response is `data[]`, each with `id`, `title`, `images.original.url`, already ordered by trending strength (index 0 = hottest).

**4. Code node — name it `normalize`**
Merge both sources into one list of items with this shape: `{ source, format_id, name, image_url, signal, raw_rank, date }`.
- For Imgflip: `format_id = "imgflip_" + id`, `signal = (total count − index)` so higher means hotter, `raw_rank = index`.
- For Giphy: `format_id = "giphy_" + id`, `name = title`, `image_url = images.original.url`, `signal = (total count − index)`, `raw_rank = index`.
- `date` = today as `YYYY-MM-DD` for every item.

**5. Google Sheets — name it `read_snapshot`**
Read all rows from the tab `snapshots`. These are prior-day records with `format_id` and `signal`.

**6. Code node — name it `score`**
For each normalized item, look up its prior `signal` from `read_snapshot` by `format_id`.
- `accel = (prior exists) ? (now − prior) : now`
- `freshness = (prior exists) ? 1.0 : 1.25` (reward brand-new formats)
- `velocity = (accel * 0.7 + now * 0.3) * freshness`
Output each item plus `velocity` and `was` (the prior signal or null).

**7. Google Sheets — name it `write_snapshot`**
Append each scored item to tab `snapshots`: columns `format_id`, `name`, `signal`, `velocity`, `date`. This must run every day so tomorrow has a baseline.

**8. Google Sheets — name it `read_used`**
Read tab `used_list` (a single column of `format_id`s already shipped or flagged dead).

**9. Filter**
Keep only items whose `format_id` is NOT present in `used_list`.

**10. Sort**
Sort remaining items by `velocity`, descending.

**11. Limit**
Keep the top 3.

End of Part A. The output is 3 trending formats with `name`, `image_url`, `format_id`, and `velocity`.

## PART B — Creative Loop

Continue from the Limit node (the top 3 formats). The brief comes from a **Set** node named `brief` holding fields `company`, `persona`, `geography`, `product`. For first build/testing, hardcode it in that Set node; **Part C** below replaces those values with live data from the landing-page webhook.

**12. Code or Set node — name it `prompt`**
For each of the 3 formats, build two strings:
- `system`: "You write meme captions. Match the meme format's known narrative. Keep it tight, native to the format, and in the audience's voice. No hashtags."
- `user`: includes the format `name`, the brief's `product`, `persona`, and `geography`, and asks: write the caption so the format's joke lands AND naturally shows the product solving a real use-case for this persona; return JSON `{"text0": "...", "text1": "..."}`.

**13. HTTP Request — name it `foundry_caption`**
POST to my Azure AI Foundry chat completions endpoint (I will paste the exact URL with my resource, deployment name, and api-version). Use the `azure-foundry` Header Auth credential. Body:
```json
{
  "messages": [
    { "role": "system", "content": "={{ $json.system }}" },
    { "role": "user", "content": "={{ $json.user }}" }
  ],
  "temperature": 0.8,
  "response_format": { "type": "json_object" }
}
```
Then a small Code node to parse `choices[0].message.content` (JSON text) into `text0` and `text1`.

**14. HTTP Request — name it `imgflip_caption`**
POST `https://api.imgflip.com/caption_image`, form-urlencoded body: `template_id` (the Imgflip numeric id, only for Imgflip-sourced formats), `username`, `password`, `text0`, `text1`. The response returns `data.url`, the rendered meme. For Giphy-sourced formats that have no Imgflip template_id, route them to a separate branch and flag them for manual handling instead of rendering.

**15. Deliver for review**
Default (no ClickUp API): append the 3 rendered `data.url`s to a Google Sheet tab named `review` with the brief fields, for me to approve manually. If a ClickUp API credential is present instead, use the ClickUp node to attach the 3 URLs to the brief's task and set status "Needs Review". Note: ClickUp webhooks only deliver intake events out of ClickUp, so they can't be used to write results back.

## Constraints
- Generate captions only for the final 3 formats, never the whole candidate pool, to keep Azure cost minimal.
- Do not use Reddit; new personal Reddit API access is closed. Use Imgflip + Giphy only.
- Persist the daily snapshot every run — the velocity score depends on it.
- Do not add any publishing or scheduling steps; the workflow ends at human review.

Please scaffold the nodes, set the connections in the order above, and add the two Code nodes with the logic described. Leave the Azure endpoint URL, Imgflip username/password, Giphy api_key, Google Sheet ID (and ClickUp list ID if used) as placeholders for me to fill in.

---

# PART C — Add intake webhook (paste AFTER Flow 1 is already built)

> Use this as a separate prompt once Parts A + B exist. It adds a live brief intake from my landing-page form and feeds it into the existing `brief` Set node. Do not rebuild Parts A/B — only add/modify the nodes below.

A landing-page form POSTs JSON `{ company, product, persona, geography, submitted_at }` to n8n with a header `X-Form-Secret`. Build this intake chain and connect it to the creative loop.

**C1. Webhook node — name it `brief_intake`**
- HTTP Method **POST**, Path `brief-intake`. This is the trigger for an on-demand run.
- Response mode **Immediately** (or add a **Respond to Webhook** node) so the form gets a fast 200.
- In node options, set **Allowed Origins (CORS)** to my landing-page domain (I'll paste it), not `*`.

**C2. IF node — name it `check_secret`**
- Condition: `{{ $json.headers['x-form-secret'] }}` **equals** my secret value (leave a placeholder).
- True → continue to C3. False → leave unconnected (or respond 401). This blocks spam before it reaches the Azure caption calls.

**C3. Set node — name it `brief`** (this REPLACES the hardcoded `brief` Set from Part B)
- `company` = `{{ $json.body.company }}`
- `product` = `{{ $json.body.product }}`
- `persona` = `{{ $json.body.persona }}`
- `geography` = `{{ $json.body.geography }}`
- (If my n8n puts the JSON at the top level instead of under `body`, drop the `.body`.)

**C4. Wire intake to the creative loop (on-demand)**
On a webhook submission, the run should: read the current top-3 from the latest snapshot in the Google Sheet (the `snapshots`/selection output Flow 1 already produces), then run Part B's caption → render → deliver using this live `brief`. Do not wait for the daily schedule. Keep the daily Schedule Trigger from Part A as well, so the trend snapshot still refreshes once a day.

**Net change:** intake (`brief_intake` → `check_secret` → `brief`) now drives the creative loop on demand; the hardcoded brief from Part B is gone; CORS + secret protect the endpoint; the daily trend pull is unchanged.
