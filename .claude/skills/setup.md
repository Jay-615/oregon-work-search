---
name: setup
description: First-run guided configuration. Creates config/claim.md, config/criteria.md and config/sources.md by interviewing the user, creates the empty data files, and checks whether the browser extension is connected. Run once when the project is new.
---

# /setup — first-run configuration

Walk the user through what this project needs to know, create the data files, and report what works and what still needs attention. Keep it conversational — this is the first thing a new user sees, and the project this derives from lost people at exactly this step.

**Ask about one thing at a time and write as you go**, so a user who stops halfway keeps what they answered.

---

## Step 1 — Check what already exists

```bash
ls -1 config/claim.md config/criteria.md config/sources.md jobs/pipeline.csv work-search-log/*.csv 2>/dev/null
ls -1 rules/
```

If `config/claim.md` already exists, this project is set up. Say so, offer to review or change a section, and do not overwrite anything without asking.

---

## Step 2 — Build `config/claim.md` — which state's rules apply

Do this first, because it determines what the whole claim half of the tool does.

**List the states in `rules/` and ask which one is paying the claim.** Then say this plainly, because it catches people out:

> It's the state that's paying your claim, not necessarily the state you live in. Those are usually the same, but if you worked across a state line they may not be. If you're unsure, it's the agency that sent your monetary determination letter.

If they name a state with no file in `rules/`, **do not guess the requirement.** Say which states are covered, and that adding theirs means writing one file against that state's official guidance — point at `rules/README.md`. Offer to do it, and if they accept, look up the current official requirement rather than working from memory. Requirements differ more than people expect, including between neighbouring states — the number of activities, whether any must be employer contacts, and whether documentary proof is required all vary.

Then read the chosen rules file and **tell them what their week will actually require**, in two or three sentences. This is the single most useful thing `/setup` says, and most users do not know it precisely.

Ask whether any exception applies to their weekly requirement, and set `requirement_override` only if one genuinely does.

**Do not ask about anything else to do with their claim** — whether it is open, whether they have registered with the state employment service, whether they have attended an orientation. This tool starts from the point where they have an active claim and owe a weekly work-search record. Everything upstream of that is between them and the agency.

---

## Step 3 — Build `config/criteria.md`

Read `config/criteria.example.md` and use it as the shape. Interview for:

1. **Roles** — the job titles they would actually apply to. Three or four maximum; each becomes a separate search.
2. **Locations** — a hard filter. Get the metro and the radius, and whether remote counts. Note that this is independent of the claim state: someone can live and claim in one state while applying to jobs across the line, and that is completely normal.
3. **Pay floor** — hourly, annual, or both. Explain two things, because both change what they see: the floor is read off the **bottom** of a disclosed range, and a range straddling the floor is kept and flagged rather than dropped.
4. **Seniority** — which title levels to accept and which to reject.
5. **Job types** — full-time, contract, contract-to-hire.
6. **Experience and desires** — the soft score. Ask for three things in their own words: what they have done, what they want, and **what they do not want.**

**Push gently on the "do not want" list.** It is the highest-value part of the file and the part users skip. Explain why with the concrete case: on a documented run the best-paying posting in a batch — top of range, direct employer, clean title — was hardware new-product introduction with a team of direct reports, entirely the wrong track. Title and pay alone sent three of five picks wrong. A written "do not want" list is what catches those.

Write the file. Confirm back a short summary rather than the whole file.

---

## Step 4 — Build `config/sources.md`

Read `config/sources.example.md`. Then:

**Start with the no-browser source.** `company_ats` needs nothing installed and is the most reliable source in the tool. Ask for ten to twenty employers they would genuinely like to work for. For each, ask them to open the company's careers page, click into any posting, and read back what the URL looks like — that identifies the hosting system, and the table in the example file maps URL shapes to systems. Leave `system` blank where they don't know; `/find-jobs` detects it once and writes the answer back.

**Then LinkedIn.** Explain the trade honestly: it finds employers they haven't thought of, and it is the only source needing the browser extension and a logged-in session. It is also the only way to confirm a posting is still open or that it has a quick-apply route.

**Leave Indeed and Dice off unless asked.** Indeed is CAPTCHA-prone. Turn on `oregon_state` only if public-sector work in Oregon is relevant to them.

**Ask about quick apply** and set `quick_apply_default`. Explain the trade in one line: a quick apply is about a minute, a full applicant-tracking-system application is closer to fifteen. Then offer three answers — `quick` (always quick-apply only), `all` (always search everything), or `ask` (decide each run).

**Recommend `quick` for anyone whose main goal is clearing the weekly requirement.** It is the setting that makes the tool usable mid-week instead of only in a Saturday scramble, and it is what most users want at least to begin with. They can change one word in the config later once the search matters more than the requirement.

---

## Step 5 — Create the data files

Seed each real file with the header row from its template, only if absent:

- `jobs/pipeline.csv` ← header from `jobs/pipeline.example.csv`
- `work-search-log/activities.csv` ← header from `work-search-log/activities.example.csv`
- `work-search-log/weekly-claims.csv` ← header from `work-search-log/weekly-claims.example.csv`
- `jobs/seen.json` ← `{}`
- `work-search-log/filings/` and `work-search-log/claim-drafts/` ← create if missing

Write **headers only**, never the example rows. Those are illustrations, and in a file attested to a government agency they would be false entries.

---

## Step 6 — Check the browser, and be honest about it

Check whether the browser extension is connected, and if so whether a logged-in LinkedIn session exists. Report plainly:

- **Connected and logged in** → everything works.
- **Connected, not logged in** → the ATS sources work fully; LinkedIn will find postings but cannot confirm they are open or how you apply. Ask them to log in in their browser.
- **Not connected** → the ATS sources work right now with no further setup; LinkedIn needs the extension. **Do not treat this as a failed setup.** A user with a target-employer list has a working tool today.

---

## Step 7 — Tell them the loop

Close with the actual weekly rhythm, short, using their state's requirement rather than a generic one:

> **During the week:** `/find-jobs` gives you a ranked list with direct links. You apply to the ones you want — this tool never applies for you. Afterwards, in `jobs/pipeline.csv`, set `status` to `applied` and fill in `date_applied` and `applied_via`.
>
> **On or just after Saturday:** `/log-week`. It reads your pipeline, works out whether the week meets <state>'s requirement, logs your activities, and hands you a block to copy into the filing form. Save the confirmation into `work-search-log/filings/`.

One habit worth mentioning:

> **Close these CSVs when you're done with them.** While a spreadsheet has a file open, nothing can write to it.

**Only if the rules file requires documentary proof per activity**, add a second habit — recording a posting URL, confirmation email or certificate as they go, and why their state wants it. Do not raise it otherwise; it is not a universal requirement and it reads as alarming when it does not apply.
