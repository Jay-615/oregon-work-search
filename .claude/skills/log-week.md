---
name: log-week
description: Record work-seeking activities for an unemployment-insurance weekly claim and produce a copy-paste block for filing. Reads config/claim.md to find which state applies, loads that state's requirement from rules/, checks what is already logged for the benefit week, derives job applications from jobs/pipeline.csv, proposes activity rows the user confirms, and appends to work-search-log/activities.csv. Integrity-first — it only records what actually happened.
---

# /log-week — record work-search evidence and produce the filing block

Maintain a truthful weekly record of the user's work-search activity so they can claim unemployment benefits, then hand them something they can copy straight into their state's form.

**Scope.** This covers the weekly work-search record and nothing else about the claim. The user already has an active claim; opening one, keeping it active, and anything the agency requires of them on that front is outside this tool. Do not volunteer advice or reminders about it. If they ask directly, answer plainly and point them at the agency.

> **Integrity is the whole point.** This record is attested to a government agency. **Only record what genuinely happened.** Propose rows grounded in what the user actually did; never invent, pad, or round up. When it is unclear whether something counts, show it and let the user decide. **A short honest week is correct; a padded week is not.**

**This skill contains no state-specific numbers.** How many activities a week needs, whether any must be employer contacts, what each entry must record, whether proof is required, and how records are retained all come from the rules file. Never hardcode a requirement here, and never carry one over from a conversation about a different state — neighbouring states differ in ways that make a confident answer wrong in both directions.

---

## Step 0 — Preflight

**1. Check for spreadsheet lock files first, before anything else.**

```bash
ls -1 jobs/.~lock* work-search-log/.~lock* 2>/dev/null || echo "none — all CSVs free"
```

A `.~lock.<file>#` file means the user has that file open in LibreOffice or Excel and any write will be lost when they save. Name the locked files immediately so they can close them while you do the rest. **Do not write to a locked file**, and make sure the guard actually returns — a check joined to the write with `;` prints a warning and writes anyway, which has happened.

**2. Read `config/claim.md`.** It gives `state`, and possibly `requirement_override`. If the file is missing, stop: *Missing claim config — run `/setup` first.*

**3. Load `rules/<state>.md`.** Everything about what the week requires comes from here. If the named file does not exist, stop and say which states `rules/` does contain rather than guessing at the requirement.

**4. Check the rules file's `verified:` date.** If it is more than about six months old, say so before doing anything else, and link the file's `requirement page`. State agencies change requirements, and a stale rule produces a confident wrong answer about someone's benefits.

**5. Honour `requirement_override` if set.** Use the user's stated number instead of the state default, and **show that you did** in the compliance report, with the reason from their config. Never silently apply an override.

**6. Compute the benefit week.** `week_start` is the most recent Sunday on or before today in **local time**; `week_ending` is that Sunday plus six days. A UTC timestamp near midnight lands on the wrong calendar day and puts rows in the wrong week. Note how many days remain — on a Saturday this is urgent. If the user names a week, use that one.

**7. Check whether the rules file forbids carry-over.** Some states require activities to fall strictly inside the claim week. Where that applies, an activity dated the day after the week closed counts for the *next* week and must not be logged against the one being filed, however tempting.

---

## Step 1 — Report standing before doing anything else

Count what is already recorded for the target week and print the arithmetic **in the shape the rules file specifies**, showing the working so the user can check it by eye. Applications come from `jobs/pipeline.csv`; other activities from `work-search-log/activities.csv`.

If the rules file requires proof per activity, **report the proof count alongside the activity count** and name any row with an empty `proof` cell. Never report a week as cleanly met while rows are undocumented.

**Read the files before accepting any verbal account of the week, including the user's own.** On a documented run the user said no job-search activity had happened yet that week; the record showed an employer contact and a resume-tailoring activity from five days earlier. Memory of which week something fell in is the least reliable input to this process. If what you find contradicts what the user said, say so plainly and show the rows.

---

## Step 2 — Derive job applications from the pipeline

**`jobs/pipeline.csv` is the single source of truth for applications.** There is no separate contacts file, deliberately: a second file would have to be reconciled against this one on every run, and that reconciliation is pure bug surface.

Select rows where `status` is `applied` and `date_applied` falls inside the benefit week. Map the row onto whatever fields the rules file requires — the mapping table is in the rules file, because states ask for different things.

**Then ask about activities the pipeline cannot know.** A phone call to a hiring manager, a job fair, a walk-in, a cold email, a workshop, a class, a career-coach appointment — none of these start as a job posting, so nothing creates a row for them automatically. These have to be requested explicitly, every run. The rules file's list of qualifying activity types is the prompt to work from; in some states that list is much broader than "applied to jobs," and a user who only thinks in applications may already have met the week without realising.

**When the user names an employer contact that was not a posting, give it a pipeline row.** `status: applied`, `date_applied` set, `url` and `posting_id` empty, and `applied_via` describing the method — `Phone call to hiring manager`, `Job fair - Portland`, `Emailed hiring manager directly`. Put the context in `notes`. This keeps every employer contact countable from one file, which is the whole reason there is no contacts file. Non-contact activities (a workshop, a video, a class day) go in `activities.csv` instead.

**One consequence to be aware of.** `status` changes in place as a role progresses, so once a posting moves to `not moving forward`, the pipeline no longer reproduces the `Applied` outcome the user attested to. That is accepted: the state's filing confirmation is the point-in-time record. Do not try to freeze outcomes in the pipeline.

---

## Step 3 — Propose the activity rows

Draft from what actually happened. Candidate sources: a `/find-jobs` run this week, the dated apply lists in `jobs/`, resume or cover-letter work, and whatever the user reports. Use the rules file's activity-type list — do not invent a type it does not name.

`activities.csv` columns: `week_start, week_ending, date, activity_type, description, notes, proof`. Write both week fields and keep them consistent (`week_ending == week_start + 6`).

**Fill `proof` if the rules file requires it.** Where it does not, recording a posting URL or apply-list filename anyway costs nothing and makes an entry self-explanatory a year later — but **never imply the user owes documentation their state does not ask for.** Check the rules file before raising proof at all.

### What integrity looks like in practice

Two decisions from documented runs:

- A resume was tailored for a posting but **never submitted** — the pipeline row had no `date_applied`. It was logged as a resume-updating **activity**, with "prepared, not submitted" in the notes, and **not** counted as an application. The work was real; the application was not.
- Two config files had been modified during the week in a way that *might* have been networking research or might have been a formatting pass. There was no evidence either way, so **nothing was logged** and the ambiguity was raised as a question.

If the week genuinely had no qualifying activity, say so and report what is still needed. Do not fabricate a row to close a gap.

**Duplicates.** Check the rules file — some states count similar-but-distinct activities separately (two applications at one company) while exact duplicates never count twice. When two rows in a week look identical, flag them and ask rather than counting both.

---

## Step 4 — Confirm, back up, then write

**Show the proposed rows as a compact table and get the user's confirmation or edits before writing anything.**

Then back up, and **prune in the same breath**:

```bash
mkdir -p backups/work-search-log
cp work-search-log/activities.csv "backups/work-search-log/activities.csv.bak-$(date +%Y%m%d-%H%M%S)"
ls -1t backups/work-search-log/activities.csv.bak-* 2>/dev/null | tail -n +4 | while read -r f; do rm -- "$f"; done
```

**Every file you back up needs its own prune line.** A run that snapshots two files and prunes one leaves the other growing past the cap unnoticed — that exact bug existed in the project this one derives from. Back up once per session per file.

**The backup cap is not a retention policy.** It keeps three recent snapshots of working state. If the rules file sets a record-retention period, the *live* log must be kept for that whole period regardless — never prune, archive or delete work-search rows because they are old, and never offer to "clean up" the log for a user whose state requires retention.

Then append. **Never write a semicolon into a CSV cell.** Use ` - ` or a period. See `CLAUDE.md` for why this is a hard rule. When appending to a file whose **existing** cells already contain semicolons, do not rewrite them away: write the whole file with `quoting=csv.QUOTE_ALL`, which survives the import intact and changes no cell text.

---

## Step 5 — Produce the filing block

The user files by copying values into a web form, so **every run ends with a clean block they can copy from**, not just a confirmation that rows were written. Use the block shape from the rules file — field order and content differ by state. Nothing interleaved.

Then, kept clearly separate so it cannot be pasted in by accident, a reference table of anything useful the form does not ask for.

Write the whole thing to `work-search-log/claim-drafts/weekly-claim-<week_start>_to_<week_ending>.md` as well as printing it, so the user can reopen it while filing.

---

## Step 6 — After filing

Two bookkeeping steps close the week, and both need the user to have actually filed first:

1. Save the agency's confirmation into `work-search-log/filings/`.
2. Append a row to `work-search-log/weekly-claims.csv`: `week_start, week_ending, claim_filed, claim_paid, date_filed, notes`.

That CSV is the only thing that makes a **missed filing** visible later. On a documented run it showed a week whose work-search evidence was complete in both other files but against which no filing was ever recorded — invisible until something indexed it.

**Never assert that a claim was or was not filed from the folder alone.** A missing confirmation is worth raising as a question. It is not evidence.

---

## Failure modes

- **No `config/claim.md`, or a `state` with no rules file** → stop and say so. Never guess a requirement.
- **A stale `verified:` date** → say so up front and link the official page. Proceed, but flagged.
- **Nothing qualifying happened** → don't fabricate. Report standing and what is still needed.
- **Ambiguous whether something counts** → show it, let the user decide, lean conservative.
- **Activities with no proof, where the state requires it** → write the rows, name the undocumented ones, and do not call the week cleanly met.
- **A locked CSV** → write nothing to it and say which file.
- **Timezone confusion at a day boundary** → resolve to the user's local date before assigning `date` and the week fields.
- **The week is short and it is Saturday** → say so first and plainly, then offer `/find-jobs` with the short-on-time sort. Check the rules file first, though: in a state where a job-search video or a workshop counts, the fastest route to a met week may not be another application.
