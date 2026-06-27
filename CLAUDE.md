# Budget Tracker — Claude Code Instructions

## Project Overview
A mobile-first personal finance web app. Single `index.html` file, no build step, no framework. Hosted on GitHub Pages, backed by Supabase (auth + Postgres). Dark mode only. Invite-only accounts.

## Live URLs
- App: https://joeyblackout.github.io/Budget-Tracker
- Supabase: https://edhsytkmmltzhgxxhpqy.supabase.co

## Credentials
- Supabase publishable key: `sb_publishable_tAn5TF5Cuear3qJMzZdN5Q_4G_mUVUh`
- ⚠️ NEVER use or suggest the secret key in the file — it was previously revoked for this reason

## Tech Stack
- Frontend: Single `index.html` — HTML, CSS, vanilla JS (no build step, no framework)
- Auth + DB: Supabase JS SDK v2 (loaded via CDN)
- Hosting: GitHub Pages (repo: joeyblackout/Budget-Tracker, branch: main)
- Deployment: GitHub Desktop — commit and push after changes

## Users
- Joey (joeyblackout) — owner
- Dad and Jennah (girlfriend) — invited via Supabase Auth → Users → Invite

## Mandatory Workflow Rules
1. **Validate JS before finishing**: extract `<script>` block to `/tmp/app.js`, run `node --check /tmp/app.js`, fix all errors before declaring done
2. **Fix template literal nesting bugs** — these are the most common error in this codebase
3. **Check JS syntax after every edit**, not just at the end
4. **Wait for explicit go-ahead** before generating and sending any file
5. **Never send a file without the user saying "go"**

## Database Schema (Supabase / Postgres)

### `profiles`
- user_id UUID (unique), display_name TEXT, auto_post BOOL default true, starting_cash NUMERIC default 0
- last_login_ts TIMESTAMPTZ — authoritative cross-device prior-login reference; advances only on a genuinely new session (>30 min gap). Drives the "X new updates since your last visit" bell notification
- updates_dismissed_ts TIMESTAMPTZ — when the user dismissed the Updates notification; changelog items with date > max(last_login_ts, updates_dismissed_ts) are shown as new. Written by `dismissUpdates()`

### `debts`
- user_id, name, type, balance, credit_limit, apr, min_payment, due_date

### `payments`
- user_id, debt_id, amount, pay_date, pay_week_id, notes

### `expenses`
- user_id, exp_date, category, description, amount, notes, pay_week_id, is_one_off BOOL default false
- `is_one_off=true` excludes the expense from the forecast daily-average, Trends sparklines, and budget-card pacing — but it still counts in all "Spent"/budget totals and the running balance

### `pay_weeks`
- user_id, pay_date, bills_amount, notes, expected_amount, source_name

### `paychecks`
- user_id, pay_week_id, amount, pay_date, notes

### `bills`
- user_id, name, amount, due_day, category, recurring, notes, debt_id

### `bill_payments`
- user_id, bill_id, amount, paid_date, notes

### `savings_pots`
- user_id, name, emoji, goal_amount, notes

### `savings_transactions`
- user_id, pot_id, type (deposit/withdraw/transfer_in), amount, txn_date, notes
- Note: `notes='__starting_balance__'` marks starting balance deposits — exclude from running balance

### `shared_pots`
- owner_id, name, emoji, goal_amount

### `shared_members`
- pot_id, user_id

### `shared_transactions`
- pot_id, user_id, type (deposit/withdraw), amount, txn_date, notes

### `shared_invites`
- pot_id, pot_name, inviter_id, invitee_id, status (pending/accepted/declined)

### `community_posts`
- user_id, message, is_milestone, created_at

### `post_reactions`
- user_id, post_id, emoji

### `recurring_income`
- user_id, name, amount, frequency (weekly/biweekly/semimonthly/monthly), next_date, end_date

### `user_assets`
- user_id, name, category, value, notes

## RLS Notes
- `shared_members` uses a SECURITY DEFINER function `my_pot_ids()` to avoid infinite recursion
- `shared_pots` SELECT policy allows `owner_id = auth.uid()` so creator can read immediately after insert
- All tables have RLS enabled

## Running Balance Formula
```
Running Balance = Starting Cash
  + paychecks where pay_date <= today
  - debt payments (all time)
  - bill payments (all time)
  - expenses (all time)
  - savings deposits (excluding __starting_balance__ transactions)
  - shared pot deposits by this user
```
- Future paychecks NOT counted until confirmed (pay_date <= today)

## Features Built

### Navigation
- Bottom nav: Forecast · Debts · Income · Bills · Savings · Expenses
- Sidebar (☰): all tabs including Dashboard, Community, Settings, How to Use
- Tapping "💰 Budget Tracker" title → Dashboard
- 🔔 Notification bell with badge count

### Dashboard
- Running balance with tappable drill-down modals
- Toggle: Account Summary / Savings Pots (includes shared pots)
- Net Worth card (assets - debt)
- Debt Payoff Calculator (Avalanche vs Snowball)
- Invite banners for pending shared pot invites

### Income
- Recurring income setup — generates 12 months of pay periods, respects end dates
- `topUpPeriods()` auto-extends on every app load if periods < 10 months out
- Per-source check numbering, global fallback
- Past Periods (7+ days old) collapsed in accordion
- ✓ Received button confirms; "Different amount" for variations

### Debts
- Add/edit/delete accounts
- 💳 Log Payment, payment history modal
- Utilization bars for credit cards
- Auto-posts to community on payoff, under 30% util, debt-free milestone

### Bills
- Add/edit/delete, recurring, due-day tracking
- ⚡ Sync Debts creates linked minimum payment bills
- Mark Paid (auto-decrements linked debt balance), Undo
- Due-soon badges (5 days), overdue badges
- Bills badge on nav showing count of overdue/due-soon

### Savings
- Personal pots: deposit, withdraw, transfer between pots, history
- Starting balance field on create AND edit
- Shared pots: invite by display name, per-person contributions, withdrawal capped at own contribution
- Physical assets: add vehicles, investments, real estate etc.

### Expenses
- Quick-add buttons (customizable, saved to localStorage)
- Edit, Repeat, Delete on each expense
- Spending Insights: Categories, Over Time (SVG bar chart), Trends (sparklines + projected spend)
- Monthly Budget Targets
- Grouped by month, collapsible

### Forecast
- Windows: Biweekly / Monthly / 3 Months / 6 Months / Annual / Custom
- Projects running balance + income - bills - debt minimums - avg daily spend
- Flags negative balance periods

### Community
- Shared feed, 5 reactions, manual posts
- Auto-posts: debt paid off, under 30% util, savings goal reached, debt-free
- Amounts never shown in auto-posts

### Notifications (🔔)
- Shared pot invites, overdue bills, bills due within 5 days, paychecks awaiting confirmation, new community posts
- **Updates**: "X new updates since your last visit" — surfaces changelog items added since the user's last login (DB `last_login_ts`). Tapping opens the What's New modal; Dismiss writes `updates_dismissed_ts`. Replaced the old auto-popup What's New modal (`checkWhatsNew` is now a no-op)
- Clears on close; last-visit stored in localStorage

### Settings
- Display name, Starting Cash on Hand, Auto-post toggle
- 6 color themes: Midnight Blue (default), Forest, Crimson, Violet, Slate, Gold

## Common Bugs to Watch For
- Template literal nesting in JS — always validate with `node --check`
- Supabase RLS blocking inserts — check policy with pg_policies query
- `shared_members` infinite recursion — fixed with `my_pot_ids()` SECURITY DEFINER function
- Future paychecks leaking into running balance — filter by `pay_date <= today()`
- Starting balance deposits double-counting — exclude `notes='__starting_balance__'`
