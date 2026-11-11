# Morning Briefing — cloud run instructions

These instructions are followed by a scheduled Claude Code cloud routine (daily, ~5:20 AM Pacific).
The routine runs with this repo checked out. It generates today's read-aloud morning news briefing
from the WSJ, NYT and San Diego Union-Tribune, writes it to `index.html`, and pushes it so
GitHub Pages serves it at https://olegfishel.github.io/newsletter/.

Run every step in one go. Make reasonable choices autonomously; do not stop to ask questions.
There is no local Mac, iCloud folder or local script involved — everything comes from the
Microsoft 365 connector (Outlook + OneDrive) and web search.

## Step 1 — Today's date (Pacific)
```bash
TZ=America/Los_Angeles date '+%Y-%m-%d %A, %B %-d, %Y'
TZ=America/Los_Angeles date -d yesterday '+%Y-%m-%d'
```
Use the Pacific date (call it `{date}`, YYYY-MM-DD) for file paths, email searches, the dateline and the
opening line. Do not use the UTC date.

## Step 2 — Front-page captures from OneDrive
A Windows machine captures the WSJ and NYT front pages at 5:00 AM Pacific into the `Newsbrief` folder of
the user's work OneDrive. The routine prompt gives the folder's `file:///...` base path (call it `{NB}`).

Read these with the Microsoft 365 connector's `read_resource` tool, BY DIRECT PATH built from `{date}`:
- `{NB}/wsj-frontpage-{date}.txt` — WSJ print front page text, includes What's News: Business & Finance and World-Wide.
- `{NB}/nyt-frontpage-{date}.txt` — NYT Today's Paper index ("The Front Page" section plus page numbers).
- Optional, only if a .txt is missing or nearly empty: `{NB}/nyt-frontpage-{date}.pdf`, `{NB}/wsj-frontpage-{date}.pdf` or `.png`.

Do NOT use `sharepoint_search` to find these — Microsoft's search index lags new uploads. If a direct read
fails, retry once; if it still fails, treat that capture as missing and fall back to the newsletters
(Step 3). A missing capture never stops the briefing; mention the fallback in the source line.

## Step 3 — Newsletters from Outlook
Use the Microsoft 365 connector's `outlook_email_search`, then `read_resource` on each hit to get the body.

SEARCH BY `sender` + `afterDateTime` ONLY. Do NOT add a `query` term or `folderName` — a sender search already
spans every folder (the newsletters live in an Inbox/NEWS subfolder), and extra filters intermittently return
nothing. `afterDateTime` = `{date}` 00:00 Pacific.
- WSJ The 10-Point: sender `access@interactive.wsj.com`. Do NOT use any forwarded "FW: Today's Wall Street Journal Print Edition" email.
- NYT The Morning: sender `nytdirect@nytimes.com` — pick the "The Morning:" edition; ignore Gameplay, Cooking, etc.
- San Diego Union-Tribune: sender `email@newsletters.sandiegouniontribune.com` — the "e-Edition" email.

Per newsletter, mark it missing only after BOTH attempts return zero results:
1. sender + afterDateTime = today → use the most recent matching edition.
2. if zero, same sender with afterDateTime = yesterday → use it and note "yesterday's edition" in the source line.
Never mark a newsletter missing because a call errored — reissue it as a plain sender-only search first.
If the Microsoft 365 tools are absent or unauthenticated, say so in the final report and build what you can from web search.

## Step 4 — Web search to supplement
Run 3–4 web searches: `top news today {date}`, `breaking news this morning {date}`, and targeted searches
(Reuters/AP/WaPo/Bloomberg) on the biggest geopolitical and market stories. Report the current state of play.
Do not use Al Jazeera, CNN, MSNBC or Fox News as sources.

## Step 5 — Write `index.html`
Overwrite `index.html` in the repo root. Self-contained, reader-safe (Safari Reader / ElevenLabs strip sibling
boxes and anything before the main article). Put ALL content as one flat flow inside ONE `<article>` — no wrapper
`<div>` boxes. Use exactly this head/style:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Morning Briefing — {Weekday, Month D, YYYY}</title>
<style>
  body { background:#f5f0e8; color:#222; font-family:Georgia, 'Times New Roman', serif; margin:0; padding:24px 16px; line-height:1.65; }
  article { max-width:780px; margin:0 auto; font-size:18px; }
  h1.masthead { font-size:2.2em; margin:0 0 4px; letter-spacing:0.5px; }
  p.dateline { font-size:1.05em; margin:0 0 6px; color:#555; }
  p.source-note { font-style:italic; font-size:0.9em; color:#666; margin:0 0 22px; }
  p.lede { font-size:1.1em; }
  h3 { margin-bottom:6px; color:#444; }
  p.signoff { margin-top:36px; font-style:italic; text-align:center; }
</style>
</head>
```

Body order, inside `<article>`:
1. `<h1 class="masthead">Morning Briefing</h1>`
2. `<p class="dateline">` full date
3. `<p class="source-note">` italic source line naming what was actually used (WSJ front page / 10-Point, NYT front page / The Morning, SDUT e-Edition, web reporting; note any fallbacks or "yesterday's edition").
4. The briefing — ONE continuous prose piece. The first `<p class="lede">` opens exactly
   `Good morning, it's {Weekday, Month D, YYYY}.` Flowing radio-script prose: no lists, no headers-with-colons,
   no per-source sections. Lead with the dominant story, then move through the rest in order of importance,
   grouping related stories (world, U.S. politics, business & markets, other national, San Diego local, sports).
   Length is whatever full coverage needs — typically 1,200–1,800 words.

   COMPLETE COVERAGE (required): before writing, build a working checklist (not published) of every story in
   - the WSJ front-page capture, including every What's News: Business & Finance and World-Wide item,
   - the WSJ 10-Point,
   - the NYT front-page capture ("The Front Page" section of Today's Paper) and The Morning,
   - the SDUT e-Edition: Today's Headlines, Local and Sports (anchor text of the bold dimgray `<p>` links).
   Merge duplicates across sources into one story. Every story on the checklist must be mentioned in the prose —
   major stories get a paragraph, minor ones can be a sentence or a clause. Do not list What's News or any
   headlines verbatim; retell everything in your own words. Before finishing, re-check the checklist against
   the draft and add anything missed.

   CROSS-SOURCE DIFFERENCES: when the same story appears in more than one source (WSJ, NYT, SDUT, web reporting),
   compare them. Where they differ — facts or figures, emphasis, framing, what's new since the edition went to
   press, or one source covering an angle the others don't — say so naturally in the prose and attribute it
   (e.g. "The Journal puts the figure at…, while the Times reports…"). Use web searches to settle which account
   is most current. Where sources agree, don't belabor it.
5. `<p class="signoff">That's your briefing for this morning.</p>`

Paraphrase — do not reproduce long verbatim passages from any source.

## Step 6 — Publish
```bash
git add index.html
git -c user.name="Morning Briefing Bot" -c user.email="noreply@anthropic.com" commit -m "Morning briefing {date}"
git push origin HEAD:main
```
If pushing to `main` is refused, push to a branch named `claude/briefing-{date}` instead and say so clearly in the
final report. Only change `index.html` — no other files.

## Step 7 — Final report
End with a short report: which sources were found / missing / fell back (WSJ capture, NYT capture, WSJ 10-Point,
NYT Morning, SDUT), the commit SHA and branch pushed, and the live URL https://olegfishel.github.io/newsletter/.
