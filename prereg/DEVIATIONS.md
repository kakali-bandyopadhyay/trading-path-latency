# Deviation journal

Every departure from `PREREGISTRATION.md` is recorded here, at the time it is made, with
what it does to the claim. The journal is published alongside the results.

Rules:

- An entry is written **before** the affected run, not reconstructed from memory afterwards.
- Nothing is deleted. A reversed decision gets a second entry that supersedes the first.
- Each entry states the effect on the claim, not just the change. "Switched to `-O2`"
  is not an entry; "switched to `-O2`, which invalidates the comparison against the
  baseline built at `-O3`, so T6 is rebuilt" is.
- An empty journal means the protocol held exactly as written.

## Format

```
### YYYY-MM-DD — short title
**Section affected:** §n
**Change:** what was done differently
**Why:** the reason, including anything unplanned
**Effect on the claim:** what this does to (a) the budget result, (b) the dominance
result, (c) comparability with other numbers in this repo
```

---

*No entries yet. The repository is at `v0.1-prereg`; no measurement has been taken.*
