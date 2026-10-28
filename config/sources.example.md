# Sources

Copy this file to `config/sources.md` and edit it. That copy is gitignored.

This file tells `/find-jobs` where to look. Turn sources off by setting them to `false` — a run only costs what it searches, so leave off anything you don't use.

---

## Sources

- `linkedin: true`           # broad sweep across all employers. Needs Chrome + a logged-in session.
- `company_ats: true`        # specific employers you care about. Needs nothing — plain HTTP.
- `indeed: false`            # broad sweep. Needs Chrome. CAPTCHA-prone, so off by default.
- `oregon_state: false`      # State of Oregon careers site. Turn on if public-sector work interests you.
- `dice: false`              # tech contract roles specifically. Needs Chrome.

## Quick apply

- `quick_apply_default: ask`

`ask` (the default) means `/find-jobs` asks each run whether to search quick-apply postings only or everything, unless your request already said. Set it to `quick` to always restrict to quick applies without being asked, or `all` to always search everything.

Most people want `quick`. A quick-apply submission takes about a minute; a full applicant-tracking-system application — create an account, re-enter your work history, answer screening questions — is closer to fifteen. If the point is clearing a weekly requirement without it eating your day, quick applies are what make that possible mid-week instead of only in a Saturday scramble.

- `quick_apply_first: true`

When this is on, and you searched everything rather than quick-apply only, `/find-jobs` sorts quick-apply postings to the top of the apply list and labels the rest with what applying actually involves. It does **not** drop anything. Turn it off if you'd rather see the list in pure fit order.

This matters for the weekly unemployment-insurance minimum. A quick-apply submission takes about a minute. A full applicant-tracking-system application — create an account, re-enter your whole work history, answer screening questions — takes closer to fifteen. When you need two employer contacts by Saturday and it's Saturday, that difference is the whole ballgame. When you're searching properly on a Tuesday, it doesn't matter and you should be looking at the best roles regardless of apply effort.

---

## Target employers (for `company_ats`)

The employers you'd genuinely like to work for. This source queries their job boards directly, which makes it the most reliable one in the whole tool: no login, no bot detection, no browser, and the data comes from the employer rather than from an aggregator that may be weeks stale.

Most companies run their careers page on one of a handful of hosted systems, and several of those publish a public JSON endpoint. To find a company's system, open their careers page and look at the URL once you click into an actual job posting:

| What the URL contains | System | Public endpoint pattern |
|---|---|---|
| `jobs.lever.co/<name>` | Lever | `api.lever.co/v0/postings/<name>?mode=json` |
| `boards.greenhouse.io/<name>` | Greenhouse | `boards-api.greenhouse.io/v1/boards/<name>/jobs` |
| `jobs.ashbyhq.com/<name>` | Ashby | `api.ashbyhq.com/posting-api/job-board/<name>` |
| `<name>.wd1.myworkdayjobs.com` or `wd5` etc. | Workday | needs a POST to the tenant's search endpoint — `/find-jobs` handles it, but it is slower and more fragile than the three above |
| anything else | unknown | `/find-jobs` falls back to reading the careers page in the browser |

Fill in the `slug` column with the name from the URL, and leave `system` blank if you don't know — `/find-jobs` will try to detect it once and remember the answer.

| Employer | System | Slug | Priority | Notes |
|---|---|---|---|---|
| Example Logistics Co | greenhouse | example-logistics | High | Local HQ, posts project management roles regularly |
| Example Regional Health | lever | exampleregional | High | |
| Example Retail Partners | workday | exampleretail | Medium | Workday tenant, slower to scan |
| Example Software Inc | ashby | examplesoftware | Medium | |
| Example County Government | | | Low | Custom careers page, needs the browser fallback |

**Priority controls cost.** A run scans `High` first, then `Medium` if the run budget allows, and only scans `Low` when you ask for a full sweep. Ten to twenty employers is a sensible list; a hundred is a slow run with little extra yield, because the broad LinkedIn sweep already covers employers you haven't thought of.

---

## Search cadence

- `lookback_days: 7`

How far back to look for new postings. Seven days suits a weekly rhythm. Going wider mostly returns postings that are already closed — roughly a quarter of month-old postings are dead, and `/find-jobs` has to verify and discard each one, which is the most expensive step in a run.
