# 479Code Task Tracker — v2

Firebase (Auth + Firestore) task tracker and invoice generator for 479Code.
Static site, no build step. Live on Vercel, auto-deploying from `main`.

Firebase project: `code-tasktracker-e4d7f` (fresh project — no legacy data,
so no migration step needed).

## Deploy order

1. `firebase login` (one-time, on your machine)
2. `firebase use --add` → pick `code-tasktracker-e4d7f`
3. `firebase deploy --only firestore:rules,firestore:indexes`
4. Confirm **collection group queries** are enabled for `members` in the
   Firebase console (Firestore → Indexes) — the composite index in
   `firestore.indexes.json` prompts for this on first deploy if needed.

That's it — no data migration, since the project starts empty. Do this
**before** anyone signs up and creates real projects: until step 3 runs,
whatever default rules Firestore assigned a brand-new project are what's
actually governing reads/writes (usually locked-closed by default, which
just means nothing works yet — not a security risk either way, but worth
deploying the real rules before pointing anyone at the sign-up form).

## What changed from v1

| | v1 | v2 |
|---|---|---|
| Access control | client-side `role==='owner'` checks only | enforced in `firestore.rules` — the database itself rejects unauthorized writes |
| Membership | a `members` map field on the project doc | `projects/{id}/members/{uid}` subcollection (no more read-modify-write races on invite/remove) |
| Dashboard counts | fetched every task in every project on every load | cached `counts` field on the project doc, updated by aggregation queries after task writes |
| Live updates | manual reload after every action, notifications polled every 30s | `onSnapshot` realtime listeners throughout |
| Invoice numbers | `INV/MM/YYYY`, collided if two invoices made in the same month | atomic counter via a Firestore transaction |
| Invoice access | hidden nav button if your email ≠ owner | hidden button **and** rejected at the database if you're not the owner |

## Files

- `index.html` — the app
- `firestore.rules` — server-side access control (read the comments at the top — explains the membership model)
- `firestore.indexes.json` / `firebase.json` — deploy config
- `manifest.json`, `sw.js`, `icons/` — PWA shell

## Known trade-offs, on purpose

- Invoice access is gated by a single hardcoded email (`479code@gmail.com`)
  rather than a roles collection — enforced in rules, not just hidden in the
  UI. If invoicing needs to extend to more than one person later, swap that
  check for a `roles/{uid}` doc lookup.
- The "breached" (overdue) count per project is computed with a
  `where('dueDate','<', today)` query filtered client-side by status,
  rather than a second aggregation query — Firestore doesn't support two
  inequality filters on different fields in one query. Fine at this scale;
  revisit if a single project accumulates thousands of overdue tasks.
