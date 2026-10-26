# Brad's Board — Live (v17)

## Live URL (bookmark this)

**https://brad731.github.io/brad-board/**

GitHub Pages hosts the interactive HTML (Supabase cannot serve runnable HTML — they force `text/html` → `text/plain` + CSP sandbox on Storage and Edge Functions).

**Data backend (sync):** Supabase project `bodydrop-lifeos` (`ljwzyqbskzlbtawwgrgn`)
- Table: `public.brad_board_state` row `id = 'main'`
- Realtime + 18s poll in the board
- Edge function (state API / publish helper, not the UI):
  `https://ljwzyqbskzlbtawwgrgn.supabase.co/functions/v1/brad-board?api=state`

Repo: https://github.com/brad731/brad-board

## How Brad opens it

1. Bookmark **https://brad731.github.io/brad-board/**
2. Or open `Downloads/brad-taskboard.html` on the Mac (redirect stub → live URL).
3. Header shows **Synced / Saving… / Offline**. Ask stays on the card; answers appear when Brad OS writes them.

## Agent: answer an open ask

Single source of truth is `brad_board_state.data`. Asks live on `task.ask`:

```json
{
  "text": "...",
  "at": "ISO",
  "status": "open" | "answered" | "cleared",
  "answer": "...",
  "answeredAt": "ISO"
}
```

### Preferred helper (box)

```bash
python3 /workspace/taskboard/patch_ask_answer.py t017 "Your answer text here"
```

### Verify

```sql
SELECT t->>'id' AS id,
       t->'ask'->>'status' AS status,
       left(t->'ask'->>'answer', 80) AS answer
FROM brad_board_state,
     jsonb_array_elements(data->'tasks') t
WHERE id = 'main' AND t->>'id' = 't017';
```

## Agent: list open asks (for board-asks-sync)

```sql
SELECT t->>'id' AS id,
       t->>'title' AS title,
       t->>'project' AS project,
       t->>'column' AS column,
       t->'ask'->>'text' AS ask_text,
       t->'ask'->>'at' AS ask_at
FROM public.brad_board_state,
     jsonb_array_elements(data->'tasks') AS t
WHERE id = 'main'
  AND t->'ask'->>'status' = 'open'
  AND coalesce(t->'ask'->>'text', '') <> '';
```

## Suggested new `board-asks-sync` routine prompt (parent updates — executors cannot)

```
Every weekday 9:00–18:00 Europe/London, every 30 minutes:

1) Read open asks from Supabase project bodydrop-lifeos (ljwzyqbskzlbtawwgrgn):
   SQL: select data from public.brad_board_state where id='main'
   Filter tasks where ask.status = 'open' and ask.text is non-empty.

2) For each new/changed open ask (dedupe by task id + ask.at), answer Brad in chat
   briefly (ADHD-friendly), then write the answer back onto the card:
   set ask.status='answered', ask.answer=<text>, ask.answeredAt=now(),
   updated_by='agent' on brad_board_state id=main.
   Prefer: python3 /workspace/taskboard/patch_ask_answer.py <task_id> "<answer>"

3) Do NOT read Mac Downloads/brad-board-asks.json anymore.
   Live board URL: https://brad731.github.io/brad-board/

4) If no open asks, exit quietly.
```


## Send to Brad OS (one-tap webhook)

Primary path to wake answers: header button **Send to Brad OS**.

Also auto-fires (debounced ~1.2s) when you save an Ask on a card.

### First-time setup (Brad, once)

1. In **Grok Bot → Routines → Board ask now**, copy **Webhook URL** (POST to) and the **sender key**.
2. On the live board, tap **Send to Brad OS** (or ⚙) and paste both → **Save & enable**.
3. Tap **Send to Brad OS** again — toast should say **Sent**. Answers land on the cards.

Config is stored in `brad_board_state.data.settings.askWebhook` as `{ url, key?, savedAt }` (also mirrored to localStorage). **Do not commit webhook secrets to the repo or README.**

JSON body posted:

```json
{ "source": "brad-board", "event": "ask-now", "openAskIds": ["t007", "..."], "at": "ISO", "reason": "button|ask-saved" }
```

Headers when a key is saved: `Authorization: Bearer <key>` and `X-Automation-Key: <key>`.

### Backup: weekday cron

Routine `board-asks-sync` (weekdays ~every 30 min, ~9:00–18:30 Europe/London) still polls open asks and answers them. Prefer the button outside that window or when you want answers immediately.

## Redeploying the board UI

After editing `/workspace/taskboard/template.html` + rebuilding `board.html`:

```bash
# 1) Rebuild board.html from template + seed
# 2) Push to GitHub Pages:
cp /workspace/taskboard/board.html /tmp/brad-board-pages/index.html
cd /tmp/brad-board-pages && git add index.html && git commit -m "board update" && git push
# 3) Optional: refresh edge embed + storage copy
#    rebuild board_html.ts, deploy_edge_function, then GET ?api=publish
```

## Local files

- `/workspace/taskboard/board.html` — v17 board
- `/workspace/taskboard/template.html` — `__DATA__` placeholder
- `/workspace/taskboard/live/index.html` — copy
- `/workspace/taskboard/brad-board-live-seed.json` — seed
- `/workspace/taskboard/edge/brad-board/` — edge function (state API)
- `/workspace/taskboard/patch_ask_answer.py` — answer helper
- `/workspace/taskboard/brad-taskboard-redirect.html` — Mac Downloads stub

## Security

RLS allows anon SELECT/INSERT/UPDATE on `brad_board_state` only (obscure URL access model, same class as Viktor pages). No service_role in the browser. Advisors clean after create.
