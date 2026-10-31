# Job Search Criteria

Copy this file to `config/criteria.md` and edit it. That copy is gitignored, so it stays private.

This file is read at the start of every `/find-jobs` run. It is read as prose, not parsed strictly — write it like you would explain your search to a friend. Three things do the real filtering: **location**, **pay**, and **match to your experience and what you want**.

---

## Roles (job titles to search for)

List the titles you would actually apply to. Each one becomes a separate search, so three or four is the practical maximum before a run gets slow. Put your preferred title first.

- Project Manager (preferred)
- Program Manager
- Technical Project Manager

## Locations

This is a **hard filter** — a posting outside these is dropped, not ranked lower.

- Portland metro (within 30 miles of Portland, OR)
- Remote (US-based)
- Hybrid with a Portland-area office

Note on "remote": a posting can say Remote and still mean "remote, but you must live in Ohio." `/find-jobs` reads the work mode off the posting's own page rather than trusting the search filter or the result card, because on some job boards the remote filter is unreliable. Postings anchored to a metro you don't live in get flagged rather than silently kept.

## Pay floor

This is a **hard filter**, but only when the posting actually discloses pay. Roughly half don't, and a posting with no stated pay is kept rather than dropped.

- $55/hour, OR
- $100,000 annual

Two rules about how this is applied, both of which matter more than the number:

**The floor is read off the bottom of a disclosed range.** A posting at "$95,000 - $130,000" has a bottom of $95,000.

**A range that straddles the floor is kept and flagged, not dropped.** If the top clears your floor and the bottom doesn't, you see the posting with the straddle called out. Narrow bands sitting right on a floor are extremely common in contract work, and dropping them silently removes most of the local contract market. A posting is only dropped when its *entire* range sits below the floor.

## Seniority

- **Accept:** senior individual contributor (Senior, Lead, Principal), and mid-level.
- **Reject:** VP, Head of, Chief, Director, Associate, Junior, Intern.

Adjust both lines. If you want director-level roles, move Director up to Accept.

## Job types

- Full-time
- Contract
- Contract-to-hire

## Experience and desires — what makes a posting a good match

This is the **soft score**, not a filter. Nothing is dropped for failing it; postings are ranked by it, and the ranking is explained to you in a sentence per posting so you can disagree with it.

- `resume_path:` /path/to/your/resume.pdf

Optional. If you give `/setup` a resume it drafts the "What I have done" section below from it, so you don't have to write your own career summary from scratch. `.md`, `.txt`, `.pdf` and `.docx` all work with nothing extra installed. Keeping the path here lets a later run re-read it when your resume changes.

Your resume is never copied anywhere tracked — this file is gitignored, and a pre-commit hook refuses to commit anything containing a home-directory path or an email address.

**A resume only answers the first of the three parts below.** It records what you did, not what you want next or what you would turn down — and people often want to stop doing the thing their resume is best at. Write those two yourself.

Write this part as plain prose. Be specific about both halves — what you have done, and what you actually want. The second half is the one people skip, and it is the one that catches the postings that look right on paper and are wrong in practice.

**What I have done:**

> Twelve years running cross-functional delivery programs in retail operations. Built governance and reporting layers from scratch — status forums, integrated roadmaps, risk registers. Strong on stakeholder management up to executive level. Comfortable with SQL and reporting tools as a consumer of data, not as an engineer.

**What I want:**

> Program or project management where the work is coordination, governance and communication. Portland-based or genuinely remote. Full-time preferred, but a contract over three months is fine.

**What I do not want — be explicit, this does real work:**

> Not people-management of a direct-report team. Not hands-on engineering or hardware new-product introduction. Not change-management or organisational-transformation consulting. Not anything requiring a security clearance.

That last section is worth more than it looks. On a documented run of the original version of this tool, the best-paying posting in the batch — top of the range, direct employer, clean title — turned out to be hardware new-product introduction with a team to manage. Title and pay alone would have sent three of five picks wrong. A written "do not want" list is what catches those on the description read.
