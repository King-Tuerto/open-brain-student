# Something wrong?

There is no support channel for this course — the plan is that you work
through problems with Claude, the same way you built everything else. Open a
conversation with Claude Code or claude.ai, paste this page or a screenshot
of what you're seeing, and ask it to run the fix. This page is organized by
what you're actually SEEING, not by which Level built the thing that broke —
find your symptom below.

---

## "It worked before — now everything errors"

**Likely cause: your Supabase project paused itself.** Free-tier Supabase
projects pause after about a week with no real API activity. A paused
project looks exactly like a broken one from the outside — the app can't
reach the database at all.

**Confirm it:** run this with your own project URL and anon key (both are in
`config.js`) —

```
curl -s "https://YOUR-PROJECT.supabase.co/rest/v1/thoughts?select=id&limit=1" \
  -H "apikey: YOUR_ANON_KEY"
```

This is the same shape of request as the keep-alive check-in you set up in
[Level 2, Step 0b](Level-2-Feed.md) — the pg_cron job and the GitHub Actions
workflow both ping this exact endpoint. A live project answers in under a
second — real rows, an empty list, or even a permissions error, it doesn't
matter which, an answer is an answer. A paused one hangs or fails to connect
at all.

**Fix it:**
1. Go to [supabase.com/dashboard](https://supabase.com/dashboard) and log in
2. Open your organization, find the project — it will be labelled "Paused"
3. Click **Resume project**, confirm

It comes back within a few minutes. Nothing is lost — same database, same
data, same config, same URL. This works for up to a year after a project
paused, which is Supabase's own limit, not this course's; you will not
realistically hit it.

**Reduce how often this happens:** Level 2, Step 0b already set up two
independent pings meant to keep the project active — say so plainly, as
that step does: neither is proven to actually stop a pause, only that a real
request goes out regularly. If it still happens, this page — not those
pings — is the actual fix.

---

## Saving does nothing, or shows an error you don't understand

Check, roughly in order of how often each one is the actual cause:

1. **`config.js` still has its placeholder values.** Open it and check for
   `PASTE_YOUR_PROJECT_URL_HERE` or `PASTE_YOUR_ANON_KEY_HERE` — if either
   is still there, the app was never actually connected to a database.
2. **You're not signed in, or your session expired** (from Level 2 onward,
   once a login screen exists). Look at the status dot near the top of the
   app — it should say "Connected to your database," not "Not connected."
   Log out and back in.
3. **The project is paused.** See the symptom above — a paused project
   makes saving fail the same way it makes everything else fail.

---

## Search comes back empty, or finds the wrong things

What this means depends on which Level you last finished:

**Level 2 through 5 — you have keyword search only.** It matches the words
you typed, literally. Searching in different words than you saved won't
find the thought — that's expected at this stage, not a bug. Level 6 is
what fixes it.

**Level 6 or later — you should have search by meaning.** If it's not
finding things it should, the most likely cause is a thought that's missing
its embedding. Confirm it in the Supabase SQL Editor:

```sql
select count(*) from thoughts where embedding is null;
```

A number greater than zero means some thoughts are missing theirs. For a
**brand-new** capture, give it 10–15 seconds first — the embedding is
generated right after saving, not instantly (if it never arrives at all,
see the next symptom below instead). For **older** thoughts, re-run the same
loop from Level 6, Step 5 that called `backfill-embeddings` the first time —
it only ever processes thoughts where `embedding is null`, so running it
again is always safe and picks up exactly where it left off.

If you finished Level 7, the same idea applies to `thought_chunks` — check
`select count(*) from thought_chunks;` (that exact query is how Level 7,
Step 6 has you confirm its own backfill worked) and re-run `backfill-chunks`
from that step if the count looks low for how much you've captured.

---

## Captures never get tags, a category, or a summary

This is the enrichment agent from Level 5 not running — and per that
level's own warning, **the symptom is deceptive**: saving still works fine,
nothing looks broken, the columns just silently stay empty.

**The single most common cause** (Level 5, Step 3): the Database Webhook is
missing its Authorization header, so it gets turned away before your
function runs. Check it:

1. Supabase dashboard → Database → Webhooks
2. Open `enrich-on-insert`
3. Confirm it has an HTTP header named `Authorization` with value
   `Bearer YOUR_SERVICE_ROLE_KEY`

**If the header is there**, check the function's own logs: Supabase
dashboard → Edge Functions → `enrich-thought` → Logs.

- **Logs are empty** → the webhook isn't reaching the function at all —
  back to the header check above, or confirm the webhook is enabled and
  pointed at the right table and event (INSERT on `thoughts`).
- **Logs show errors** → check `ANTHROPIC_API_KEY` is actually set in
  Supabase → Edge Functions → Secrets (Level 5, Step 1) and that the account
  behind it still has credit.

If you're on Level 6 or later and it's specifically the **embedding** that's
missing while tags/category/summary are fine, the same header requirement
applies to the call from `enrich-thought` to `generate-embedding` — Level 6,
Step 4 warns about this exact symptom directly: "the embeddings silently
never appear and the generate-embedding logs are empty, because it never
ran."

---

## The Telegram bot has gone quiet

Ask Telegram itself what it thinks is happening — it keeps this for you, no
Supabase login needed:

```
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getWebhookInfo
```

Read `last_error_message` in the response — it usually names the problem
directly. The most common one: a `401`, meaning `telegram-bot` got deployed
without the `--no-verify-jwt` flag that [Level 3, Step 7](Level-3-Telegram.md)
explains is required — Telegram cannot send a Supabase login token, so
without that flag your function never even runs. Redeploy with it:

```
npx supabase functions deploy telegram-bot --no-verify-jwt
```

If the webhook looks fine but nothing happens for anyone, also check
whether the project is paused — see the first symptom on this page.

---

## None of these match what you're seeing

Paste a screenshot or the actual error text into Claude and ask it to
diagnose it live — it can read your logs and your database in a way this
page can't. That is exactly what every Level's rules already tell it to do.
