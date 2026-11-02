# Oregon Work Search

Find jobs worth applying to, and keep the weekly work-search record Oregon requires in order to claim unemployment benefits.

> ### Oregon only
>
> The requirement encoded here is Oregon's: **2 direct employer contacts plus 3 additional work-seeking activities**, per Sunday–Saturday week. **Other states differ in shape, not only in numbers** — a neighbouring state asks for a flat count of activities with none of them required to be an employer contact, but demands documentary proof of each. Following this file set elsewhere will produce a confidently wrong record. Adding your own state is one file: see [`rules/README.md`](rules/README.md).
>
> ### Not legal or benefits advice
>
> The rule is transcribed in [`rules/oregon.md`](rules/oregon.md), **last verified 2026-10-24** against [unemployment.oregon.gov](https://unemployment.oregon.gov). Requirements change, and a stale rule here produces a confident wrong answer about your benefits. Check the official page before your first filing, and re-check whenever that verified date looks old — `/log-week` will warn you once it passes about six months.

Two commands do the work. One finds postings and hands you a short ranked list with direct links. The other works out whether your week meets Oregon's requirement and gives you a block to copy into the filing form.

**It never applies to anything for you.** Every run stops at "here are the postings worth your time."

**It assumes you already have an active claim.** Opening one is between you and the Employment Department.

---

## Why this exists

The work-search requirement is not hard. Proving you met it is the miserable part.

Every week you claim, you have to show a specific number of qualifying activities, recorded with specific fields, for a week that ended before you sat down to file. From memory. While also job searching. Plenty of people who are entitled to benefits simply stop claiming them, and the record-keeping is why.

This project keeps the record as a side effect of searching, so by the weekend the evidence already exists and filing is a copy-paste.

---

## What Oregon asks for

Benefit weeks run **Sunday through Saturday**, and a week is named by the Saturday that ends it — that is the date the filing form asks for. For each week you claim:

- **2 direct employer contacts**, each recorded with six fields: date, employer, location or posting number, how you contacted them, the position, and the outcome.
- **3 additional work-seeking activities** — more employer contacts, updating a resume, searching listings, networking, a workshop.

**One thing worth knowing, because it changes the arithmetic:** an employer contact beyond the required two counts as one of your three additional activities. So five applications in a week satisfies both requirements on its own, with nothing else logged. `/log-week` does this maths for you and shows its working.

Two other conditions — being **able** to work and **available** for work — are yours to attest to on the form. This tool does not touch them.

**It also assumes you already have an active claim.** Applying for benefits in the first place, and anything the Employment Department asks of you to keep the claim active, is outside what this does. It starts where you have a claim and owe a weekly work-search record.

> Not legal or benefits advice, and `rules/oregon.md` carries the date it was last verified. Check the official page before your first filing: [unemployment.oregon.gov](https://unemployment.oregon.gov). You file at [frances.oregon.gov](https://frances.oregon.gov).

---

## Setup

You need [Claude Code](https://claude.com/claude-code). Nothing else.

Get the files: click **Code → Download ZIP** at the top of this page and unzip it wherever you like. You don't need git or a GitHub account.

Then open a terminal in that folder and run:

```bash
claude
```

and type:

```
/setup
```

It interviews you for about five minutes — which state is paying your claim, the job titles you want, where you'll work, your pay floor, and what kind of work you actually want and don't want — then writes three config files.

**Point it at your resume and it drafts your experience section for you** (`.md`, `.txt`, `.pdf` or `.docx`, nothing extra to install), so the slowest part of setup becomes one step. It still asks you directly what you *want* and what you'd *turn down* — a resume can't answer those, and the second one does more work than anything else in the config. The first thing it tells you is exactly what your week will require, which most people don't know precisely.

**You can use this with no browser setup at all.** The most reliable source queries employers' job boards directly over plain HTTP: no login, no bot detection, nothing to install. Give `/setup` a list of ten to twenty employers you'd like to work for and you have a working tool immediately.

**Adding LinkedIn gets you the broad sweep** — postings from employers you haven't thought of, which is most of them. That needs the Claude in Chrome extension and a logged-in LinkedIn session in your own browser. It is also the only source that can confirm a posting is still open and tell you whether it has a one-minute quick-apply route. Worth it, but not required to start.

### Only if you cloned with git

Downloaded the ZIP? Skip this — it will fail, because a ZIP is not a git repository.

If you cloned instead, turn on the pre-commit hook:

```bash
git config core.hooksPath .githooks
```

It refuses to commit your own job-search data. Everything real is gitignored already, so this is the backstop for the case where a new kind of file appears and the ignore rules haven't caught up.

---

## The weekly loop

**During the week:**

```
/find-jobs
```

Searches your enabled sources, drops anything failing your location, pay and seniority filters, **verifies each survivor is actually still open**, scores what's left on how well it matches your experience and what you want, and writes a dated list to `jobs/apply-list-<date>.md` with direct links.

Then you apply to the ones you want. Afterwards, open `jobs/pipeline.csv` and for each one: set `status` to `applied`, fill in `date_applied`, and put how you applied in `applied_via`. That is the only bookkeeping you owe it.

**On or just after Saturday:**

```
/log-week
```

Reads your pipeline, counts what the week actually contains against your state's rule, prints the arithmetic, proposes your activity rows for you to confirm, and produces the filing block. Save the confirmation into `work-search-log/filings/`.

---

## Two things that will save you grief

**Close the CSVs when you're done with them.** While a spreadsheet has a file open, nothing can write to it. Both commands check for this first and will tell you which file to close.

**Fill in the "what I do not want" part of your criteria.** It feels like the throwaway section and it does more work than anything else in the file. On a real run of the project this derives from, the best-paying posting in a batch — top of the range, direct employer, clean job title — turned out to be hardware product-launch work managing a team of direct reports. Entirely the wrong job. Title and pay alone sent three of five picks wrong that day.

---

## What it does not do

- **It does not apply for you.** By design.
- **It does not solve CAPTCHAs.** If one appears, that source stops and tells you.
- **It does not score your network.** No contact lists, no connection mapping. Just location, pay, and match to your experience.
- **It does not write your resume.** It will tell you that tailoring one is a loggable activity, because it is.
- **It does not file your claim.** It gives you the block; you paste it and press the button.
- **It does not give benefits advice.** It applies the rule in `rules/oregon.md`, and tells you when that file is getting old.

---

## Your data stays yours

Everything real is gitignored: your claim details, your criteria, your target employers, your pipeline, your work-search log, your filing confirmations, your backups. Only the fictional `*.example.*` templates, the state rules and the instruction files are ever tracked. If you push this somewhere, you publish the tool and none of your search.

Every data file is snapshotted into `backups/` before any write, three copies deep. That cap is for working state, not retention — where your state requires you to keep records, the live log is kept in full regardless of age.

---

## Layout

```
config/claim.md             which state is paying your claim
config/criteria.md          your roles, locations, pay floor, experience and desires
config/sources.md           which sources to search, and your target employers
rules/<state>.md            what a week requires in that state. One file per state
jobs/pipeline.csv           every opportunity, one row each. The system of record
jobs/apply-list-<date>.md   the dated output of a /find-jobs run
work-search-log/            your weekly activities, filed claims, and confirmations
```

`jobs/pipeline.csv` is the one file worth understanding. It holds every posting the tool has found for you and every application you've made, and it is where `/log-week` gets your employer contacts from. Edit it freely — add columns, reorder them, write yourself notes. The tool reads it by column name and preserves anything it doesn't recognise.
