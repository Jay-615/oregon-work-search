# Claim

Copy this file to `config/claim.md` and edit it. That copy is gitignored.

This tells `/log-week` which state's rules apply. Everything state-specific lives in `rules/<state>.md`, so this file is short on purpose.

**This tool assumes you already have an active unemployment claim.** Opening one, and anything the agency asks of you to keep it active, is outside what this does — it starts from the point where you have a claim and owe a weekly work-search record.

---

- `state: oregon`

Must match a filename in `rules/` without the extension. `oregon` is the one set up. `/log-week` loads that file and applies whatever it says. If you need a different state, see `rules/README.md`.

**It is the state that is paying your claim, not necessarily the state you live in.** Those are usually the same. If you worked across a state line they may not be, and the agency that sent your monetary determination letter is the one whose rules apply.

- `requirement_override:`

Leave blank in almost every case. Some claimants have a reduced or waived requirement — approved partial unemployment, or certain approved training or shared-work programs. If an exception applies to you, write the number of activities you actually owe here and a note saying why. `/log-week` will use it instead of the state default and will show that it did.

---

## Notes to self

Anything you want to remember about your own claim — your weekly filing day, a reference number for an open issue, a question you mean to ask the agency. `/log-week` reads this file, so a note here is a note it will see.
