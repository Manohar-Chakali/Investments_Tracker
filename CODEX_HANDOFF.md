# Investment Tracker Handoff

Last updated: 2026-05-31

## Project

- Repository: https://github.com/Manohar-Chakali/Investments_Tracker
- Live app: https://manohar-chakali.github.io/Investments_Tracker/investment-tracker.html
- Main file: `investment-tracker.html`
- Stack: single-file HTML, CSS, vanilla JavaScript, GitHub Pages, Firebase Realtime Database
- Firebase path: `/tracker`

## Purpose

The app tracks a Rs. 5,00,000 investment/payment agreement with two parallel streams:

- Company payment tracking: Rs. 15,000 per month, with Sent and Received acknowledgements.
- EMI tracking: Rs. 13,637 per month, with Paid and Verified acknowledgements.
- Existing EMI rows saved with the old Rs. 13,650 default are migrated to Rs. 13,637 on render; other custom EMI values are preserved.
- Both schedules start in July 2026. Internally the app still stores month numbers 1-48, but the UI shows actual periods such as Jul 2026, Aug 2026, etc.

It is meant to be simple enough for a non-technical company representative to open a link, tick boxes, and add comments without accounts or setup.

## Current Behavior

- Dashboard shows received amount, company progress, EMI paid, and EMI progress.
- Dashboard also shows tracker health, next action, and a year-wise payment plan summary.
- Dashboard shows quarterly progress grouped into expandable year sections and an overdue count.
- Dashboard expected company total stays fixed at Rs. 5,40,000 for the 36-month agreement; advances are treated as early cash received, not as a reduction to the agreement value.
- Payments tab supports Amount, Advance, Sent, Received, and Comments.
- Payments tab also shows status, sent date, received date, and advance explanation labels.
- EMI tab supports Paid, Verified, Paid Date, Verified Date, Comments, and status.
- More tab includes JSON export, CSV export, agreement details, activity log, user name, optional secret tracker key, and recalculation.
- Dark mode preference is stored in browser storage.
- Firebase sync keeps both parties on the same tracker data.
- Navigation is now near the top and sticky for quicker switching.
- Amount and Advance inputs are wide enough to show full values like `15000`.
- The normal monthly Amount field is capped at Rs. 15,000 to prevent accidental extra-zero entries such as `150000`.
- Existing saved rows above Rs. 15,000 are normalized back to Rs. 15,000 and surfaced in the dashboard Safety Checks panel.
- Payment and EMI rows use real month labels starting from Jul 2026 instead of generic M1/M2 labels.
- CSV export includes both the numeric month index and the readable period label.
- Due/upcoming status labels are shown against the actual month schedule.
- The year-wise expected/received amounts are intentionally shown on the dashboard, not repeated below the Payments table.

## Important Advance Payment Logic

The advance logic is now reversible.

Examples:

- Sep 2026 advance Rs. 5,000 -> Oct 2026 amount becomes Rs. 10,000.
- Sep 2026 advance cleared to Rs. 0 -> Oct 2026 amount returns to Rs. 15,000.
- Sep 2026 advance Rs. 15,000 -> Oct 2026 amount becomes Rs. 0 and Oct 2026 Sent/Received are checked.
- Sep 2026 advance Rs. 30,000 -> Oct 2026 and Nov 2026 become Rs. 0 and both rows are checked.
- Reducing Rs. 30,000 back to Rs. 5,000 resets Nov 2026 and leaves Oct 2026 at Rs. 10,000.

Manual user edits win:

- If a row amount is manually changed, old auto-advance ownership is removed.
- If a checkbox is manually changed, old auto-complete ownership is removed.

Regression note:

- Do not recalculate advances by subtracting from the current next-row amount on every render. That compounds reductions and causes abnormal checkbox/status behavior.
- The safe pattern is ownership-based: clear rows previously auto-adjusted by the changed advance, then rebuild only those rows from the Rs. 15,000 base amount.
- Repeated `renderAll()` calls must not change totals.
- Company received totals include a received row's Amount plus its Advance.
- Company expected totals use the fixed Rs. 15,000/month agreement baseline, so a partial advance does not make the dashboard total drop to Rs. 5,35,000.

## What Claude Missed

Claude's analysis was close on the product idea, but missed several implementation-level issues:

1. The screenshot was mostly Firebase Console/browser-extension noise, not the tracker app error.
   The real app needed inspection from the deployed tracker URL and source HTML, not only the console screenshot.

2. The deployed GitHub Pages HTML was saved from a browser download.
   It pointed to local files like `./investment-tracker_files/firebase-app.js.download`, which cannot exist on GitHub Pages.

3. The local HTML had unresolved Git merge conflict markers.
   Those markers could break JavaScript parsing and deployment reliability.

4. The Firebase SDK flavor was mismatched.
   The app used `firebase.initializeApp(...)`, which requires the Firebase compat SDK, not the v10 modular SDK files.

5. The amount rendering used `p.amount || 15000`.
   In JavaScript, `0` is false-like, so a real auto-adjusted value of `0` displayed as `15000`.

6. Advance payment logic applied changes forward but did not undo them.
   Clearing an advance needed a matching reset path for rows previously auto-adjusted or auto-completed.

## How It Was Cracked

The fix came from checking each layer instead of guessing:

1. Read the handoff and the actual local `investment-tracker.html`.
2. Compared local file, raw GitHub file, and live GitHub Pages output.
3. Verified Firebase itself was reachable with a direct Realtime Database REST read.
4. Found the deployed page was loading broken local `.download` script paths.
5. Switched Firebase scripts to compat URLs:
   - `firebase-app-compat.js`
   - `firebase-database-compat.js`
6. Removed merge conflict text.
7. Fixed zero rendering by using explicit number checks instead of `||`.
8. Added forward advance logic and then the missing reverse/reset logic.
9. Tested with a headless browser and a fake Firebase object so UI logic could be verified without touching production data.
10. Staged, committed, and pushed the fixes directly so the app updated on GitHub Pages without asking you to manually add files to git.

## Verification Already Done

Commands/checks performed during the fixes:

- JavaScript parse check passed.
- Firebase REST read returned HTTP 200.
- Raw GitHub file showed corrected Firebase script URLs.
- GitHub Pages cache-busted URLs were checked after pushes.
- Headless UI logic test verified:
  - Initial total: Rs. 5,40,000
  - Sep 2026 advance Rs. 5,000 -> dashboard total remains Rs. 5,40,000 and Oct 2026 becomes Rs. 10,000
  - Repeated renders after that keep the same amount and total
  - Clearing advance -> total Rs. 5,40,000 and Oct 2026 Rs. 15,000
  - Sep 2026 advance Rs. 15,000 -> Oct 2026 Rs. 0 and checked
  - Sep 2026 advance Rs. 30,000 -> Oct 2026 and Nov 2026 Rs. 0 and checked
  - Reducing to Rs. 5,000 -> Oct 2026 Rs. 10,000, Nov 2026 Rs. 15,000, checkboxes reset
  - Manual uncheck after auto-cover remains unchecked after render
  - Amount entry Rs. 1,50,000 is capped and saved as Rs. 15,000
  - CSV export preserves advance-covered Rs. 0 rows instead of converting them back to Rs. 15,000

## Git Notes

Recent meaningful commits:

- `da6bfa3` Fix Firebase tracker deployment
- `46d5b21` Polish tracker layout
- `ca32c49` Fix advance payment carryover

There may be additional commits with similar messages around the advance reset fix because the same issue was iterated more than once. Check `git log --oneline -10` before making future changes.

## Current Caution

Firebase rules are currently public for `/tracker`. That is convenient but not secure for long-term use.

Implemented mitigation:

- The app now supports an optional secret tracker key using `?tracker=<secret-key>`.
- The original `/tracker` path still works by default so existing data is not broken.
- A generated key stores data under `trackers/<secret-key>`, which is harder to guess but is not a full auth system.

Recommended stronger security improvement:

- Use Firebase Anonymous Auth, or
- Add simple role links such as owner/company URLs with separate permissions.

## Suggested Next Improvements Status

- Done: visible advance explanation labels.
- Done: activity log with who/what/when.
- Done: "Recalculate Advances" button.
- Done: CSV export.
- Done: payment date fields for company and EMI tracking.
- Done: monthly status badges.
- Done: actual month labels from Jul 2026 onward, while preserving existing Firebase data.
- Done: due/upcoming labels based on the July 2026 start date.
- Done: dashboard tracker health, next action, and year-wise payment plan summary.
- Done: quarterly progress and overdue count.
- Done: monthly amount cap, agreement total guard, and safety checks.
- Still recommended: Firebase Auth or role-based links before sharing widely.

## Operational Notes

To deploy future edits:

```bash
git -C C:\Users\Lenovo\Investments_Tracker status
git -C C:\Users\Lenovo\Investments_Tracker add investment-tracker.html CODEX_HANDOFF.md
git -C C:\Users\Lenovo\Investments_Tracker commit -m "Describe change"
git -C C:\Users\Lenovo\Investments_Tracker push origin main
```

To bypass GitHub Pages cache after a push, open:

```text
https://manohar-chakali.github.io/Investments_Tracker/investment-tracker.html?v=<commit-hash>
```
