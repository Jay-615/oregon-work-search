# Oregon — work-search requirement

- **state:** Oregon
- **agency:** Oregon Employment Department (OED)
- **file at:** https://frances.oregon.gov
- **requirement page:** https://unemployment.oregon.gov
- **verified:** 2026-10-24

## The week

Sunday through Saturday. **A week is identified by its ending date** — the Saturday. That is the number the filing form asks for and the number to lead with when talking to the user.

## What a week requires

- **2 direct employer contacts**, each recorded with six fields (below).
- **3 additional work-seeking activities.**

## The compliance arithmetic

**Surplus employer contacts count toward the additional activities.** A contact beyond the required two *is* a work-seeking activity, and Oregon's own list names additional employer contacts as qualifying.

```
contacts             = pipeline rows with status applied and date_applied in the week
additional_activities = max(0, contacts - 2) + activity rows in the week
MET when contacts >= 2 AND additional_activities >= 3
```

Five applications in a week satisfies both thresholds on their own with nothing else logged. **Do not report a week as short because the activities file has fewer than three rows.** A naive row count once printed "2 / 3 — SHORT" for a week containing five applications, which was wrong and would have sent the user hunting for activities they did not need.

Report it showing the surplus term so the user can check it by eye:

```
Week ending 2026-10-24 (Oregon UI):
  Direct employer contacts:  3 / 2  — met
  Additional activities:     4 / 3  — met
     (1 surplus employer contact counts as an additional activity, + 3 logged activities)
  Overall: MET
```

## The six required fields per employer contact

| Field | Pipeline column |
|---|---|
| Date | `date_applied` |
| Employer | `company` |
| Location or posting number | `location`, plus `posting_id` or the id from `url` |
| How you contacted them | `applied_via` |
| Position or type of work | `title` |
| Outcome | `status` |

## Qualifying activity types

`Searched job listings`, `Reviewed/screened job postings`, `Updated resume`, `Updated cover letter`, `Networking outreach`, `Attended workshop/event`, `Additional employer contact`, `Other`.

## Proof

Oregon does not require a proof artifact attached to each activity. **Do not tell the user they owe documentation per activity, and never treat a blank `proof` cell as a problem.** Record enough detail that an entry is self-explanatory a year later, fill `proof` opportunistically where something concrete already exists (a posting URL, an apply-list filename), and keep the filing confirmations.

## Record retention

Keep the filing confirmations. Oregon publishes no specific retention period, so keeping them through the benefit year and a while after is simply prudent. There is no retention rule here that should stop ordinary housekeeping.

## Filing block shape

One block per employer contact, fields in the order above:

```
1.  Date: 10/22/2026
    Employer: Example Logistics Co
    Location/Posting #: Portland, OR — LinkedIn posting #1111111111
    How contacted: Applied on LinkedIn (Easy Apply)
    Position: Senior Project Manager
    Outcome: Applied
```

Then the additional activities, each with date, type and description.

## What this file does not cover

Only the weekly work-search requirement. Everything else about the claim is out of scope and stays that way:

- Being **able** to work and **available** for work are separate eligibility conditions and are the user's own attestations on the form. Never assert them.
- **Opening or maintaining the claim itself** — the initial application, any registration or orientation the agency requires, identity verification, appeals. The user has an active claim before this tool is any use to them, and whatever the agency asks of them on that front is between them and the agency.

**Do not volunteer advice or reminders about any of it.** If the user raises one directly, answer plainly and point them at the agency; otherwise stay on the weekly requirement, which is the whole job here.
