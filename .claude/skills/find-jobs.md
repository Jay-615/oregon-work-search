---
name: find-jobs
description: Search configured job sources for new postings, filter them against the user's criteria, verify each survivor is genuinely still open, score them on experience match, and write a dated apply list the user can work from. Writes new rows to jobs/pipeline.csv and records every URL seen in jobs/seen.json. Does not apply to anything — the user submits applications themselves.
---

# /find-jobs — find postings, filter, verify, rank

Find job postings matching the user's saved criteria, prove they are still open, and hand back a short ranked list with direct links. **This skill never applies to anything.** It stops at "here are the postings worth your time," because the submission carries judgment and a signature.

---

## Step 0 — Preflight

**1. Check for spreadsheet lock files first, before anything else.**

```bash
ls -1 jobs/.~lock* work-search-log/.~lock* 2>/dev/null || echo "none — all CSVs free"
```

A `.~lock.<file>#` file means the user has that file open in LibreOffice or Excel, and any write will be lost when they save. Tell them immediately and by name, so they can close it while the rest of the run proceeds. **Do not write to a locked file.** Make sure the guard actually stops — a shell check joined to the write with `;` prints its warning and then writes anyway, which has happened and silently put rows underneath an open buffer. Use `&&`, or check in the same script that does the writing.

**2. Read the config.** `config/criteria.md` (roles, locations, pay floor, seniority, job types, experience and desires) and `config/sources.md` (which sources, target employers, short-on-time mode, lookback). If either is missing, stop and say: *Missing config — run `/setup` first.*

**3. Load the record.** `jobs/seen.json` (create as `{}` if absent) and `jobs/pipeline.csv`. **Read the pipeline by header name, never by column position** — users reorder columns, delete ones they don't use, and add their own. Positional access corrupts the file.

**4. Set `today` from the local date.** Use it for `first_seen` and for the apply-list filename. Resolve in local time, not UTC — near midnight a UTC timestamp lands on the wrong calendar day.

**5. Note how many days remain in the current benefit week** (Sunday–Saturday). On a Saturday, everything downstream is urgent and the short-on-time sort matters; on a Tuesday it does not.

---

## Step 1 — Company ATS (if `company_ats: true`) — do this source first

**This is the cheapest and most reliable source. It needs no browser, no login, and triggers no bot detection.** Start here so that a run still produces something useful even if the browser half fails.

For each target employer in `config/sources.md`, at or above the priority budget, fetch the public job board:

| System | Endpoint | Response shape |
|---|---|---|
| Lever | `https://api.lever.co/v0/postings/<slug>?mode=json` | **a bare JSON array** of postings |
| Greenhouse | `https://boards-api.greenhouse.io/v1/boards/<slug>/jobs` | an object: `{"jobs": [...]}` |
| Ashby | `https://api.ashbyhq.com/posting-api/job-board/<slug>` | an object: `{"jobs": [...]}` |

All three verified working on 2026-10-24 against real employer boards.

**The shapes differ — do not write one parser for all three.** Lever returns a top-level array, so `response.jobs` is `undefined` and a naive shared parser silently yields zero postings for every Lever employer. Greenhouse and Ashby both nest under a `jobs` key. Branch on the system.

**Assert that the normalized fields came out populated — not that the raw item count was non-zero.** An earlier version of this note said to assert a non-zero count, which does not catch the failure that matters: a wrong field name yields the right number of records with empty titles, every one of which is then dropped by the role filter, producing a clean-looking "no matching roles" that is indistinguishable from a correct zero. Count how many normalized records have a non-empty title and location, compare that to the raw count, and report any gap. Verified working on 2026-10-28: Lever 89/89, Ashby 168/168, Greenhouse 193/193 titles and locations populated.

**A posting present in the response is live by definition** — the employer is serving it — so these need no separate liveness check, which is the single biggest reason this source is cheap.

**A wrong slug returns a structured 404, not an error**: Greenhouse gives `{"status":404,"error":"Job not found"}` and Lever `{"ok":false,"error":"Document not found"}`. That is how slug detection distinguishes "right pattern, wrong slug" from "wrong pattern" — a 404 with a JSON body means the endpoint shape is correct and only the slug is off. Report it to the user that way rather than disabling the source.

**Responses can be large.** Measured on 2026-10-24: 147KB and 449KB from two Greenhouse boards, 861KB from one Lever board, and **2.8MB from one Ashby board**. Do not read a whole payload into context. Stream or filter server-side where possible, and in all cases extract only title, location, URL, posting id and any compensation text, discarding the rest before it reaches your context. A dozen employers at that size would be tens of megabytes.

**Lever carries compensation in prose.** Its `additionalPlain` field holds text like "The United States base range for this position is $87,069-$124,385 plus equity." That is worth parsing for the pay floor — Lever employers often disclose pay the result card never shows.

**Greenhouse's board endpoint carries no compensation at all**, so the pay floor cannot be applied to a Greenhouse posting from the list response. Its job objects hold only `title`, `location`, `absolute_url`, `id`, `requisition_id`, `application_deadline`, `first_published` and `updated_at` — no pay, salary or compensation field of any kind. Confirmed 2026-10-28.

To get pay, request the board again with **`?content=true`**, which adds a `content` field holding the full description (~5.6KB per posting) where disclosed pay appears. **Do this only for postings that have already survived the role, seniority and location filters** — one board returned 716 postings of which 16 survived, so fetching content for all of them would pull roughly 4MB to use 90KB of it. Fetch the filtered set, parse pay out of `content`, apply the floor, and discard the rest of the text before it reaches your context.

Until that second fetch happens, record `no comp disclosed on the board endpoint` in `why` rather than treating the posting as having failed or passed the floor. A posting with no comp data is kept, per the criteria file — do not silently drop it, and do not claim it cleared a floor it was never tested against.

If `system` is blank for an employer, detect it once by requesting each pattern in turn and seeing which returns valid JSON, then **write the answer back into `config/sources.md`** so the next run doesn't repeat the probe.

Workday tenants (`<name>.wdN.myworkdayjobs.com`) need a POST to the tenant's own search endpoint and return a paginated payload. They work, but they are slower and the payload shape shifts between tenants. Two known traps, both of which have burned the original version of this tool:

- **Read the application deadline from the posting body, not from the payload's `endDate` field.** On the tenant that was measured, `endDate` was consistently one day later than the real deadline, across 36 postings with zero exceptions. It fails optimistically, so a posting looks live on the morning it actually closed.
- Salary is often stated **monthly** in the body. Annualise before applying the pay floor.

Anything else falls through to reading the careers page in the browser, which is a last resort.

**Set `apply_method` for these to the system name** (`Greenhouse (company site)`, `Workday (company site)`). These are not quick applies — most mean an account and a full work history.

---

## Step 2 — LinkedIn (if `linkedin: true`)

The broad sweep. Finds employers the target list doesn't contain, which is most of them.

### 2a. Browser preflight

Confirm the browser extension is connected, then open a tab and check the login state. **Being logged out is degraded mode, not a skip** — you can still find and cheap-filter postings, but you cannot confirm any of them are open or determine how you apply. Say which half is being lost rather than failing silently:

> Not logged into LinkedIn, so I'm harvesting logged out. I can find and filter postings, but I can't confirm they're still open or how you apply to them. Log in and I'll verify the survivors.

### 2b. Build the queries

One search per role × location-mode pair. Three roles and two modes is six searches.

- Geographic: `linkedin.com/jobs/search/?keywords=<role>&location=<place>&distance=25&f_TPR=r604800`
- Remote: `linkedin.com/jobs/search/?keywords=<role>&location=United%20States&f_TPR=r604800`

`f_TPR=r604800` is "past week" in seconds; set it from `lookback_days`.

**`f_AL=true` restricts the search to Easy Apply postings. If the user's request did not say either way, ask them before searching** — it is one short question and it changes what the whole run returns:

> Quick-apply only, or everything? Easy Apply postings take about a minute each. Including the rest finds better roles but a full application means an account and a work history, maybe fifteen minutes. Say "quick apply" or "everything" — or "quick apply from now on" and I'll stop asking.

Honour whatever the request already said: "five quick apply links", "easy apply only", "just the good ones regardless", "everything" all answer the question and need no ask. Likewise don't ask when `quick_apply_default` in `config/sources.md` is set to `quick` or `all` — use it.

**Default the recommendation to quick-apply.** Most users of this tool are trying to clear a weekly requirement with the least friction, and quick applies are what make that possible mid-week rather than only in a Saturday scramble. Recommend it unless the user has said they want the broadest search.

If they choose everything, apply effort is still recorded per posting in `apply_method` and used for sorting rather than filtering — nothing is dropped for being a long application.

**Do not use `f_WT` (the remote filter) and do not trust any result card's location.** Measured twice: a remote-filtered query returned on-site roles in four other states, and on a repeat run 4 of 8 "remote" results verified as hybrid or on-site. Work mode is read off the posting's own page in Step 5 or not recorded at all.

### 2c. Extract the result cards

The results list is virtualised: 25 result elements render but only the handful near the viewport have text. Three approaches were tried across several runs of the original tool:

1. **Anchor-tag selectors** returned 7 of 25 and missed every promoted card, which occupy the top slots. A ~70% undercount with no error raised.
2. **Scrolling each card into view** does capture all 25, but the site animates scrolling, so the loop hit a 45-second automation timeout after 8 cards. Forcing instant scroll helped; scrolling the inner container hung the renderer outright.
3. **Taking the cards that hydrate on load, then paginating.** No scrolling. Load, wait 2–3 seconds, read every result element that already has text, then request `&start=25`. About 7 per load, deterministic, no timeouts.

**Use approach 3.** It accepts an incomplete harvest deliberately: the cards that render first are the platform's top-ranked results, which is the right slice when the goal is five good postings rather than complete inventory.

Select on the list item carrying a job-id attribute (`li[data-occludable-job-id]`), not on anchors, because promoted cards have no matching anchor.

Two parsing rules that cost real debugging:

- **Filter the badge lines.** Cards interleave "Promoted", "Easy Apply", "Viewed", applicant counts and benefit chips with the title, company and location. Unfiltered, the company lands in the title field and the location in the company field. The site also inserts an accessibility label reading `<title> with verification` between the visible title and the company, which must be dropped separately.
- **Read the badges before filtering them.** "Easy Apply" and "Promoted" must stay in the filter or the fields shift — so capture them as booleans off the raw text first, then filter.

**Sanity check every page: captured count should equal the number of result elements that had text.** A short count means cards failed to hydrate; raise the wait and redo that page rather than accepting the undercount.

### 2d. Getting data out of the browser

The JavaScript bridge truncates its return at roughly 1,000 characters and redacts anything resembling a URL, query string or token. A dump of 25 records does not fit and would be partly blocked.

Accumulate into the page's `localStorage` (it survives navigation on the same origin), return only a small status object per call (`{captured, total, added}`), and pull the set out at the end in chunks of about seven compact lines.

**Clear the accumulator at the start of every run, and tag every entry with the query that produced it.** On a documented run the store already held 7 untagged entries from a previous session. They were deleted before proceeding. Leftovers would have been presented as this week's findings in a record that gets attested to a government agency.

### 2e. Pagination — do not gate it on finding a "Next" link

**Measured 2026-10-28, and this corrects the earlier rule.** A Portland "Project Manager" search reported **1,136 results** in its header and rendered **7 usable cards**. No "Next" text appeared anywhere in `main.innerText`, and no `Page N` strip was present — at `start=0`, `start=7` and `start=25` alike. The previous rule said to paginate only when a page-2 link was detected, so it would have found none and stopped at 7 cards out of 1,136 on every single query.

**Paginate on `start` unconditionally, and stop when a page yields no new posting ids.** That is the only reliable signal.

**Step `start` by the number of cards you actually captured, not by 25.** The page's `<li>` count is unstable — 7 at `start=0`, 25 at `start=7`, 7 again at `start=25` — but the number that hydrate with text was 7 every time. Stepping by 25 does advance, but it skips whatever sat between. Stepping by the captured count walks the list contiguously: `start=0` → 7 ids, `start=7` → 7 more, `start=25` → 7 more, 21 unique across three loads with no repeats.

So: load, capture, add `captured` to `start`, repeat. Stop after about 5 loads per query, or as soon as a load returns zero new ids. Carry the de-dup set across loads keyed on the posting URL.

**The old premise about hydration has also shifted.** This skill used to say 25 elements render and only ~7 hydrate. Sometimes only 7 render at all now. The operational rule is unchanged — read every element that already has text, never scroll — but the sanity check must compare captured against *elements that had text*, not against 25, or it will report a false undercount on every run.

---

## Step 3 — Other browser sources (if enabled)

**Indeed** and **Dice** work anonymously but Indeed is CAPTCHA-prone. **If a CAPTCHA appears, stop that source and tell the user.** Never attempt to solve one. Report what was harvested before the block and continue with the other sources.

**Oregon state jobs** (`oregon_state: true`) is a Workday tenant and follows the Workday notes in Step 1, including the `endDate` trap and the monthly-salary conversion. Public-sector level names don't map onto corporate ones, so do not reject a senior "Administrator" or "Coordinator" title on the strength of the word alone.

---

## Step 4 — Hard filters

Applied to card data only, before opening anything. Opening posting detail pages is the most expensive step in the run and must only happen to candidates that survived everything cheaper.

- **Location** — must match a line in the criteria `Locations` list. A metro-anchored "Remote" (remote but you must live in Ohio) is flagged for Step 5 verification rather than kept or dropped on the card.
- **Seniority** — drop titles matching the criteria reject list.
- **Job type** — drop types not in the criteria list.
- **Pay floor** — only when pay is disclosed. Read the floor off the **bottom** of the range. **A range whose top clears the floor and whose bottom does not is kept and flagged, not dropped.** Drop only when the entire range sits below the floor. This rule, not the floor number, is what keeps the local contract market visible — narrow bands straddling a floor are the common case there.

---

## Step 5 — Dedupe, then verify

### Dedupe on the posting URL

**The URL is the dedup key**, not any internal id. Check candidates against both `jobs/seen.json` (every URL ever seen, including ones found closed) and `jobs/pipeline.csv`.

**Flag possible duplicates; never merge rows.** Same employer plus same-ish title plus a different URL is for the user to resolve. A false merge silently destroys an opportunity; a false split costs two seconds of reading. This is not hypothetical — a documented run surfaced a fourth requisition from one staffing agency whose description was near-identical to three already sitting un-actioned, which meant applying to several would put multiple submissions for one client through one agency. The evidence went to the user; the decision stayed theirs.

### Verify liveness and apply method

Only for candidates that survived everything above.

**Liveness.** Roughly a quarter of "past week" postings are already closed — 10 of 40 in one measurement, including the highest-scoring one. For browser sources, test the posting page for "no longer accepting applications". For ATS-API sources, presence in the response is proof enough. **Record closed postings in `seen.json` with a closed status so they never resurface**, report the count, and do not write them to the pipeline as live.

**Apply method.** Read what applying actually involves and record it in `apply_method`.

**On LinkedIn, verified 2026-10-28, the quick-apply control is a `<button>`** whose `innerText` is exactly `Easy Apply` and whose `aria-label` is `Easy Apply to this job`. Test both, since either alone may churn:

```javascript
const ea = !!document.querySelector('button[aria-label="Easy Apply to this job"]')
        || !!Array.from(document.querySelectorAll('button,a'))
              .find(e => /^easy apply$/i.test((e.innerText || '').trim()));
```

**An earlier version of this note had this exactly backwards and would have broken the feature completely.** It said the control was an `<a>` reading "Apply" with `aria-label="LinkedIn Apply to this job"`, and warned that matching on button text "under-reports". Measured on three live postings on 2026-10-28: the `LinkedIn Apply to this job` label **does not exist** on any of them, there is no plain "Apply" anchor on an Easy Apply posting at all, and the button-text match the note warned against is the signal that works. Following the old advice, every posting would verify as not-quick-apply, the apply list would come back empty every run, and the failure would look like "no Easy Apply postings this week" rather than like a bug.

**Do not select on class names.** They are hashed build artifacts — the apply button carried `a9llkt a9llks a9lth a9la2n a9lhc3 a9lk6n` — and they change without notice.

**Re-verify this element whenever a run returns zero quick-applies.** That symptom means this check broke, not that the market dried up. It is the single most fragile dependency in this skill, and it has now been observed to invert once.

**"Promoted" and quick-apply are not mutually exclusive.** An earlier version of this tool asserted they were and instructed a re-parse on seeing both, which threw away good postings as parse errors. Five of six verified picks on one documented run were promoted *and* carried a genuine quick-apply control; four of five on another. Treat promoted as a caution worth mentioning — such postings more often add screening questions or route responses off-platform, and their description is sometimes missing from the logged-in page — never as a disqualifier.

**Work mode.** Read it off the posting page and record it. This is the only trustworthy source for it.

Batch these checks. The return is about 20 characters per posting, so a batch of eight costs almost nothing.

---

## Step 6 — Score on experience and desires, then read the descriptions

Score each survivor against the **experience and desires** section of `config/criteria.md`. One dimension, 0–100, and **write a one-sentence `why` explaining the score** so the user can disagree with it. There are no weights to configure: location, pay and seniority were hard filters, and this is the only soft judgment.

Pay attention to the **"what I do not want"** list. It is the highest-value part of the criteria file and the cheapest to check.

### Then read the descriptions of the finalists. Do not skip this under time pressure.

**This is the step most likely to be skipped and most likely to cost the week.** In one documented run the best-paying hit — top of the range, direct employer, clean title — was hardware new-product introduction with a team to manage, the wrong track entirely. Two others died on the description: one was at a rejected seniority, one was work the user had already turned down in writing. **Title and pay alone would have sent three of five picks wrong.**

On a later run, reading descriptions moved a posting from first place to last: by title and pay it led the batch, but the description asked for software engineering expertise rather than program management, named a sector worth knowing in advance, and contradicted the result card on whether the role was full-time or contract. It stayed on the list, ranked last, with all three caveats stated. **Rank honestly and state the bends — do not quietly drop a posting the user might still want, and do not quietly recommend one with a problem in it.**

---

## Step 6b — Sanitize every externally-sourced value before it touches a CSV

**Read this before writing a single row.** Everything from a job board — titles, locations, company names, compensation text, descriptions — is untrusted text that lands in a spreadsheet the user opens. Pass every such value through one sanitizer on ingest:

```python
def clean(v):
    if v is None: return ''
    s = str(v).replace('\r', ' ').replace('\n', ' ')
    s = s.replace(';', ' -')          # semicolons must never reach a cell
    return re.sub(r'\s+', ' ', s).strip()
```

**Semicolons arrive from the wire, not just from your own prose.** The semicolon rule in `CLAUDE.md` reads like a writing guideline, and that is how it was understood until a real run proved otherwise: a Greenhouse board returned the location `US Remote; Canada`, and that value was on its way into a cell. One posting out of 716. Unsanitized it would have split that row in two on open and shifted every column after it right by one — the exact corruption `CLAUDE.md` records as having happened twice before, except arriving from a third party rather than from anything anyone wrote.

The sanitizer also fixes leading and trailing whitespace, which boards really do emit (one Ashby board returns titles beginning with a space), and collapses embedded newlines that would otherwise break the row.

**Validate every row before opening the file for writing.** Build the full set of rows, assert the whole set is clean, and only then open the CSV. A validation failure part-way through a loop that writes at the end discards the entire run's work with nothing saved — that happened in testing, and worse, the next step read the empty file and reported a confident zero. Fail before the write, not during it, and assert the row count you expect afterwards.

## Step 7 — Write the record and the apply list

**Pipeline rows.** One per new survivor, `status: new`, filling every column Step 5 verified. Append by header name. Preserve unknown columns and untouched rows verbatim. **Never write a semicolon into any cell** — see `CLAUDE.md`, this has caused silent column-shift corruption twice.

**`seen.json`.** Record every URL seen this run, including closed ones. **But never mark a posting seen without also writing it somewhere visible** — a URL in `seen.json` with no pipeline row disappears permanently from the user's view. If a harvest was partial or some candidates went unverified, leave those out of `seen.json` so they resurface next run, and **say so in the provenance block**.

**The apply list** — `jobs/apply-list-<today>.md`. For each recommended posting: a direct link, employer, location, work mode, type, pay as posted, how you apply, the fit score and its one-sentence reason, years asked if stated, applicant count and posting age if available, an explicit note wherever it bends the criteria, and a checkbox.

Sort by fit score. **If `quick_apply_first` is on and the benefit week has two days or fewer left, sort quick-applies to the top instead and say at the top of the file why the order changed.**

Then a **dropped** table — what was verified open but failed a criteria filter, and which filter — so the user can see the filter working and argue with it.

Then a **provenance block**: how many cards per source, how many survived each filter, how many were verified, how many were already known, and **explicitly which harvested postings were not written to the record and will therefore resurface.** An incomplete harvest is fine. An undisclosed one is not.

End by telling the user plainly: how many postings, how many are quick applies, where the file is, and that nothing has been applied to.

---

## Failure modes

- **A source fails** → disable that source, note it in the summary, continue with the others. Never fail the whole run for one source. This is why the no-browser ATS source runs first.
- **CAPTCHA** → stop that source, tell the user, never attempt to solve it.
- **Rate limited** (the logged-out LinkedIn endpoint returns HTTP 429 after roughly 30 fetches) → pace at ~350ms, accept the partial harvest, report which query was truncated.
- **A locked CSV** → write nothing to it, say which file, continue with everything that doesn't need it.
- **Zero survivors** → say so plainly and show what the filters removed. A week with nothing worth applying to is a real outcome; inventing a recommendation is not.
