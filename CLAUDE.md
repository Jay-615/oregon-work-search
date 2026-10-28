# Oregon Work Search

You are running inside a personal job-search and unemployment-insurance record-keeping project. It does two jobs: find job postings worth applying to, and keep the weekly work-search record the user's state requires in order to claim benefits.

**It is deliberately not an auto-applier.** Every run stops at "here are the postings worth your time." The user submits applications themselves, because a submission carries judgment and a signature.

## The three commands

- **`/setup`** — first-run configuration. Interviews the user, writes the two config files, creates the data files.
- **`/find-jobs`** — search the enabled sources, filter against criteria, verify each survivor is still open, score on experience match, write a dated apply list.
- **`/log-week`** — check the benefit week against the user's state's requirement, log activities, produce a copy-paste block for filing.

## Repository layout

```
.
├── CLAUDE.md                              # this file
├── README.md
├── .claude/skills/{setup,find-jobs,log-week}.md
├── config/
│   ├── claim.example.md                   # tracked
│   ├── claim.md                           # GITIGNORED — which state is paying the claim
│   ├── criteria.example.md                # tracked
│   ├── criteria.md                        # GITIGNORED — roles, locations, pay floor, experience
│   ├── sources.example.md                 # tracked
│   └── sources.md                         # GITIGNORED — which sources, target employers
├── rules/                                 # tracked — one file per state
│   ├── README.md                          # how to add a state
│   └── oregon.md
├── jobs/
│   ├── pipeline.example.csv               # tracked
│   ├── pipeline.csv                       # GITIGNORED — THE SYSTEM OF RECORD
│   ├── seen.json                          # GITIGNORED — dedup index of every URL ever seen
│   └── apply-list-<date>.md               # GITIGNORED — dated output of a /find-jobs run
├── work-search-log/
│   ├── activities.example.csv             # tracked
│   ├── activities.csv                     # GITIGNORED — the 3+ additional activities per week
│   ├── weekly-claims.example.csv          # tracked
│   ├── weekly-claims.csv                  # GITIGNORED — which weeks were filed, and when
│   ├── filings/                           # GITIGNORED — confirmation documents from the state
│   └── claim-drafts/                      # GITIGNORED — the copy-paste blocks for filing
└── backups/                               # GITIGNORED — 3 snapshots per data file
```

Everything holding real employers or a real job search is gitignored. Only the `*.example.*` templates, the state rules and the instruction files are tracked.

### All state-specific knowledge lives in `rules/`

How many activities a week needs, whether any must be direct employer contacts, what each log entry must record, whether documentary proof is required, how long records must be kept, where the claim is filed — **all of it comes from `rules/<state>.md`**, named by `state:` in `config/claim.md`.

**Never hardcode a requirement into a skill, and never carry one over from a conversation about a different state.** Neighbouring states differ in shape, not just in numbers — one may want a fixed number of direct employer contacts plus additional activities and ask for no documentation, while the next wants a flat count of activities of any kind but demands documentary proof of each and retention past the benefit year. Assuming either shape for the other leaves a user chasing activities they do not owe, or keeping a log that will not survive an audit.

Each rules file carries a `verified:` date. If it is more than about six months old, say so before using it and link the official page. A stale rule here yields a confident wrong answer about someone's benefits.

### Backups are not retention

`backups/` keeps three recent snapshots of working state so a bad write can be undone. **It is not a record-retention mechanism.** Where the rules file sets a record-retention period, the *live* work-search log must be kept in full for that whole period. Never prune, archive or delete work-search rows because they are old, and never offer to tidy up the log for a user whose state requires retention. An audit can arrive after filing has stopped.

---

## Conventions

### `jobs/pipeline.csv` is the system of record

One CSV holds the whole search. One row per opportunity, created by `/find-jobs`, managed from then on by the user in a spreadsheet. Rows are never deleted and never moved — `status` changes in place (`new`, `interested`, `applied`, `passed`, `closed`, or anything the user invents).

**It is also the source of truth for employer contacts and job applications.** There is no separate contacts file, deliberately — a second file would have to be reconciled against this one on every run, and that reconciliation is pure bug surface. `/log-week` derives contacts from rows where `status` is `applied` and `date_applied` falls in the benefit week.

**Contacts that are not postings still get a row.** A phone call, a job fair, a cold email: `status: applied`, `date_applied` set, `url` and `posting_id` empty, and `applied_via` describing the method. This keeps every employer contact countable from one file.

Rules that keep it trustworthy:

- **`url` is the dedup key**, not `posting_id`. Zero duplicate URLs across the file.
- **Read and write by header name, never by column position.** Users reorder columns, delete ones they don't use, and add their own. Positional access corrupts the file.
- **Preserve unknown columns and untouched rows verbatim.**
- **`status`, `notes`, `date_applied` and `applied_via` belong to the user.** Never overwrite a non-empty cell in those. A note reading "don't want to relocate for this" outvalues any score.
- **Never merge two rows.** Flag possible duplicates and let the user decide. A false merge silently destroys an opportunity; a false split costs two seconds of reading.
- **Back up before writing**, into `backups/`, keeping 3 per file.

### Never write a semicolon into a CSV cell

Not in the pipeline, not in the work-search logs, not anywhere the user opens in a spreadsheet. A common LibreOffice import configuration treats `;` as a field separator, so an **unquoted** cell containing one splits in two the moment the file is opened, and every column after it shifts right by one.

Python's `csv` writer only quotes a field containing a comma, a quote or a newline — **a bare semicolon triggers no quoting**, so the write succeeds silently and the damage happens later, on the user's machine, with no error anywhere.

**This has happened twice in the project this one is derived from.** A note reading `Deadline passed; removed from board` shifted 21 rows. An earlier value containing a comma shifted 3 more, and those sat broken for two months before anyone noticed. The symptom to recognise: a column near the end is empty and a later column holds a value that belongs to an earlier one.

Use ` - ` or a period. When appending to a file whose **existing** cells already contain semicolons, do not rewrite them away — write the whole file with `quoting=csv.QUOTE_ALL`. Quoted fields survive the import intact and no cell text changes.

The same care applies to commas: any cell that might contain one must be written through a real CSV writer, never by string concatenation.

**This rule is not only about prose you write.** Job boards emit semicolons in their own data: a real run found a Greenhouse board serving the location `US Remote; Canada`, one posting out of 716, heading straight for a cell. Every value arriving from an external source — title, location, company, compensation text — must pass through a sanitizer on ingest that replaces `;` with ` -`, strips newlines and trims whitespace. Treating this as a style guideline for authored text is how it gets missed. See **Step 6b** in `find-jobs.md` for the sanitizer.

### Refuse to write a file the user has open

LibreOffice and Excel leave a `.~lock.<filename>#` file beside any open document. If one exists, the user has that file open and an in-place write will be lost the moment they save.

**Check for locks as the very first step of any run that will write a CSV**, and name the locked files immediately so the user can close them while the rest of the work proceeds:

```bash
ls -1 jobs/.~lock* work-search-log/.~lock* 2>/dev/null || echo "none — all CSVs free"
```

**A guard that prints without returning is not a guard.** In the project this is derived from, the check was once joined to the write with `;` instead of `&&`: it printed "LOCK PRESENT - abort" and then wrote anyway, putting rows underneath an open buffer that would have overwritten them on save. Verify that a refusal actually refuses.

### Backups — one folder, three copies

Every write to a data file snapshots it first, into `backups/` mirroring the source path:

```bash
mkdir -p "backups/<dir>"
cp "<dir>/<file>" "backups/<dir>/<file>.bak-$(date +%Y%m%d-%H%M%S)"
ls -1t "backups/<dir>/<file>".bak-* 2>/dev/null | tail -n +4 | while read -r f; do rm -- "$f"; done
```

**Every `cp` is paired with its own prune line.** A run that backs up two files needs two prune lines. In the original project one skill snapshotted two files and pruned only the first; the unpruned one grew past the cap unnoticed for weeks. Back up once per session per file — the point is preserving the pre-session state, not every intermediate write.

### Integrity of the work-search record

The work-search log is attested to a government agency. **Only record what genuinely happened.** Never invent, pad, or round up. When it is unclear whether something counts, show it and let the user decide. A short honest week is correct; a padded week is not.

Concretely: a resume tailored for a posting that was never submitted is an **activity**, not an employer contact — log it with "prepared, not submitted" and do not count it toward the two. Where there is no evidence that something happened, log nothing and ask.

**Never assert the able-to-work or available-for-work conditions.** Those are the user's own attestations on the form.

### Never write a real person's name into a tracked file

If this project is ever pushed to a public host, every file that is not gitignored is published. Real people's names, real employers tied to the user's actual search, real profile URLs and real home-directory paths must never appear in a tracked file — not in `README.md`, not in a skill file, not in a commit message, not in a code comment. Use the fictional examples in `*.example.*` instead.

The pull toward breaking this is real: naming the specific case is what makes a bug report or a calibration note credible. **Accurate and publishable diverge the moment the subject is a real person**, and the instinct toward precision is what walks you over the line. "A staffing agency posting the same requisition three times" is marginally worse documentation and enormously better privacy. Take that trade every time.

### Confirm before publishing

Editing files here is private and reversible. A `git push`, or any command that changes a remote repository, is the moment it becomes public. **Propose what you intend to commit and push, and wait for an explicit go-ahead.** Users typically have more changes to talk through than a first pass captures.

---

## Where this is fragile

- **Job-board markup changes.** The LinkedIn card extractor depends on attributes on a site that redesigns without notice, and its failure mode is a silent undercount rather than an error. The sanity check — captured count equals the number of elements that had text — is what catches it and must stay in.
- **Rate limiting.** The logged-out LinkedIn endpoint returns HTTP 429 after roughly 30 fetches. Pace requests, accept a partial harvest, and report which query was truncated.
- **Lock files.** A week where the user leaves a CSV open is a week where logging cannot proceed. That is a stop, not something to work around.
- **Timezone at day boundaries.** A row assigned to the wrong calendar day lands in the wrong benefit week and can make a compliant week look short.
- **The integrity rules are prose, not code.** Nothing mechanically prevents a padded row. That is the right design — the user signs the attestation — but it means the rules have to be read rather than merely present.
