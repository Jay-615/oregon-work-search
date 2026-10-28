# State rules

One file per state. `/log-week` reads `config/claim.md` to find out which state applies, loads the matching file from this folder, and applies whatever it says.

Everything state-specific lives here and nowhere else: how many activities a week needs, whether any of them have to be employer contacts, what each log entry must record, how long records must be kept, and where the claim gets filed. The two skills carry no state constants at all.

**To add a state**, copy the closest existing file, rewrite it against that state's current official guidance, and cite the page you read with the date you read it. Then set `state:` in `config/claim.md`.

**These files go stale.** State agencies change requirements, and a stale rule here produces a confident wrong answer about someone's benefits. Every file carries a `verified:` date. When `/log-week` sees one older than about six months it says so and suggests re-checking the official page before filing. That warning is not a formality — getting this wrong can mean denied weeks or repaying benefits already received.

| State | File | Shape of the requirement |
|---|---|---|
| Oregon | `oregon.md` | 2 direct employer contacts + 3 additional activities |

States vary in shape, not only in numbers. Some want a fixed number of direct employer contacts plus additional activities; others want a flat count of activities of any kind, but require documentary proof of each and retention past the benefit year. Read the official guidance rather than adapting the numbers in `oregon.md`.
