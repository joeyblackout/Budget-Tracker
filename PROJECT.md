# Budget Tracker — Project Context

## Overview
A mobile-first personal finance web app built as a single `index.html` hosted on **GitHub Pages**, backed by **Supabase** (auth + Postgres). Invite-only accounts. Dark mode only (Carbon theme hardcoded).

## Live URLs
- App: https://joeyblackout.github.io/Budget-Tracker
- Supabase: https://edhsytkmmltzhgxxhpqy.supabase.co

## Credentials
- Supabase publishable key: `sb_publishable_tAn5TF5Cuear3qJMzZdN5Q_4G_mUVUh`
- ⚠️ NEVER use the secret key in the file — it was previously revoked for this reason

## Users
- Joey (joeyblackout) — owner, GitHub username joeyblackout
- Dad and Jennah (girlfriend) — invited via Supabase Auth → Users → Invite

## Tech Stack
- Frontend: Single `index.html` — HTML, CSS, vanilla JS (no build step, no framework)
- Auth + DB: Supabase JS SDK v2 (loaded via CDN)
- Icons: Tabler Icons via CDN (use `<i class="ti ti-NAME"></i>`) — NO emoji in UI chrome
- Hosting: GitHub Pages (repo: joeyblackout/Budget-Tracker, branch: main)
- Deployment: GitHub Desktop — Joey drops the file in, commits, pushes

## Working Files
- Working copy: `/home/claude/budget-tracker/index.html`
- Output copy: `/mnt/user-data/outputs/index.html` (copied here after every update)
- This file: `/home/claude/budget-tracker/PROJECT.md`

## Established Workflow Rules
1. **Always validate JS first**: extract `<script>` block, run `node --check /tmp/app.js`, fix errors before copying to outputs
2. **Always copy to outputs**: `cp /home/claude/budget-tracker/index.html /mnt/user-data/outputs/index.html`
3. **Wait for explicit go-ahead** before generating and sending any file
4. **Never send a file without the user saying "go"**
5. **Check JS syntax after every edit** — template literal nesting errors are common
6. **Use `present_files`** to deliver the file, not just text
7. **Do not open or preview files** after making changes

## Database Schema (Supabase / Postgres)

### `profiles`
- user_id UUID (unique), display_name TEXT, auto_post BOOL default true, starting_cash NUMERIC default 0

### `debts`
- user_id, name, type, balance, credit_limit, apr, min_payment, due_date

### `payments`
- user_id, debt_id, amount, pay_date, pay_week_id, notes

### `expenses`
- user_id, exp_date, category, description, amount, notes, pay_week_id

### `pay_weeks`
- user_id, pay_date, bills_amount, notes, expected_amount, source_name

### `paychecks`
- user_id, pay_week_id, amount, pay_date, notes

### `bills`
- user_id, name, amount, due_day, category, recurring, notes, debt_id, auto_synced BOOL, linked_debt_id, due_month INTEGER (1–12, for non-monthly frequency)
- `recurring` values: Monthly, Quarterly, Semi-Annual, Annual
- `due_month` — which month(s) the bill falls due (used for Quarterly/Semi-Annual/Annual)
- SQL to add column if missing: `ALTER TABLE bills ADD COLUMN IF NOT EXISTS due_month integer;`

### `bill_payments`
- user_id, bill_id, amount, paid_date, notes

### `savings_pots`
- user_id, name, emoji, goal_amount, notes

### `savings_transactions`
- user_id, pot_id, type (deposit/withdraw/transfer_in), amount, txn_date, notes
- Note: `notes='__starting_balance__'` marks starting balance deposits (excluded from running balance)

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

### `budget_targets`
- user_id, category TEXT, monthly_target NUMERIC, created_at
- SQL: `CREATE TABLE IF NOT EXISTS budget_targets (id uuid default gen_random_uuid() primary key, user_id uuid references auth.users, category text, monthly_target numeric, created_at timestamptz default now());`

## RLS Notes
- `shared_members` uses a SECURITY DEFINER function `my_pot_ids()` to avoid infinite recursion
- `shared_pots` SELECT policy allows `owner_id = auth.uid()` so creator can read immediately after insert
- All tables have RLS enabled

## Theme
- **Carbon only** — hardcoded in `:root` CSS variables, no theme switcher
- `--bg: #080808`, `--surface: #111111`, `--surface2: #1a1a1a`, `--border: #272727`
- `--accent: #2dd4a0`, `--accent2: #0a9e74`, `--success: #2dd4a0`
- `--warning: #f7c948`, `--danger: #f75a5a`, `--text: #f0f0f0`, `--muted: #666666`

## Brand / Logo
- Favicon: `favicon.ico` — green rounded square with white `$` symbol
- Logo: `<img src="favicon.ico">` at appropriate size + wordmark "Budget" in `var(--text)` + "Tracker" in `var(--success)`
- Logo appears in: topbar, sidebar header, login screen, loading screen
- No emoji in UI chrome — use Tabler Icons for all icons

## Features Built

### Navigation
- Bottom nav: Forecast · Debts · Income · Bills · Savings · Expenses
- Sidebar (☰): Dashboard, Forecast, Income, Debts, Bills, Savings, Community, Expenses, How to Use, Paycheck Calculator, Settings
- Tapping Budget Tracker logo → Dashboard
- 🔔 Notification bell with badge count
- All icons use Tabler Icons (`ti-*`), no emoji in UI chrome

### Loading Screen
- Custom branded loading screen centered on page
- Shows favicon with green glow, wordmark, shimmer progress bar animation, "Loading your finances…"
- Fades out once app is ready (auth state resolved)

### Dashboard
- **Available Balance** (formerly "Running Balance") = Starting Cash + confirmed income - debt paid - bills paid - expenses - savings deposits - shared deposits
- Confirming a paycheck early counts immediately regardless of scheduled pay date
- Tappable balance lines (drill-down modals)
- This Month / This Year / All Time toggle on Running Balance card (affects Bills Paid and Expenses display only, not the balance math)
- Toggle pills positioned in top-right corner of the card
- **Bills This Month** progress bar card (below available balance) — tappable → Bills tab
- **Expenses Budget card** (below Bills card) — tappable → Expenses tab Budget insight
  - Shows: Income Less Bills (top, prominent) with subtext "$X expected − $X remaining bills"
  - Budget / Spent / Remaining three-column row below
  - Progress bar (pace-based coloring: green/yellow/red)
  - Label: "EXPENSES BUDGET · JUNE 2026" (dynamic month)
- **Debt Overview card**: Total Debt + CC Utilization side by side, Payoff Progress bar below, all in one card
- Net Worth card (assets - debt)
- Payday Routine banner (when paycheck pending) — styled with green glow, prominent "Start →" button
- Greeting by display name

### Payday Routine (4-step modal from Dashboard)
- Triggered by Payday Routine banner when paycheck(s) pending
- **Step 1 — Paycheck**: confirm expected amount or enter different amount
- **Step 2 — Bills**: checklist of due/overdue bills, pre-checked, tap Pay Selected
- **Step 3 — Debts**: debt accounts listed with min payments pre-filled, adjustable, tap Log Payments
- **Step 4 — Savings**: deposit amounts to savings pots, tap Save & Finish
- Progress indicator: "1 of 4" with step labels
- Can exit at any step; completed steps are saved

### Income (tab: "My Income")
- Recurring income setup (Recurring button) — generates 12 months of pay periods, respects end dates
- `topUpPeriods()` auto-extends on every app load if periods < 10 months out
- Per-source check numbering ("Select Safety · Check 3")
- Pay periods grouped by month with collapsible headers (styled with green accent border)
- Month headers: bold, green chevron, period count + expected total in subtitle
- Past Periods (7+ days old) collapsed in accordion at top
- Totals bar: Confirmed (tappable → confirmed income modal with edit/delete) / Expected / Pending
- Confirming a paycheck before scheduled date counts immediately in available balance
- Source filter buttons
- Pay period expand/collapse shows: + Paycheck, Delete Period (removed + Payment and + Expense)

### Debts (tab)
- Add/edit/delete accounts
- Debt cards collapsible (tap header or chevron to collapse/expand, all start expanded)
- 💳 Log Payment button, History button
- Utilization bars for credit cards
- When debt balance hits $0: auto-synced minimum payment bill is automatically deleted
- Auto-posts to community on payoff, under 30% util, debt-free milestone

### Bills (tab: "Monthly Bills")
- Add/edit/delete, frequencies: **Monthly, Quarterly, Semi-Annual, Annual**
- `due_month` field (1–12) on non-monthly bills — determines which months bill actually falls due
- Bill subtitle shows due months for non-monthly bills (e.g. "Due Jan & Jul · Day 29 · Semi-Annual")
- ⚡ Sync Debts creates linked minimum payment bills (skips $0 balance debts)
- Mark Paid (auto-decrements linked debt balance), Undo
- **Unpaid bills always at top** (sorted by due day ascending), **paid bills sink to bottom**
- Pay All Due button (appears when due/overdue bills exist) — confirm modal → bulk mark paid
- Progress bar: paid this month vs remaining (only counts bills actually due this month by frequency)
- Bills badge on nav showing count of overdue/due-soon
- All frequency-aware calculations respect `due_month` — Quarterly/Semi-Annual/Annual only counted in correct months

### Savings
- Personal pots: deposit, withdraw, transfer between pots, history
- Starting balance field on create AND edit
- Shared pots: invite by display name, per-person contributions
- Physical assets (Asset button)
- All three sections in one tab

### Expenses (tab: "My Expenses")
- Quick-add buttons (customizable)
- Edit, Repeat, Delete on each expense
- Expense month groups **default collapsed** when tab opens
- **Spending Insights** (two tabs: Budget, Trends):
  - **Budget tab** (default):
    - Header: Income Less Bills (prominent, top) + subtext breakdown
    - Budget / Spent / Remaining three-column stat row
    - Progress bar (pace-based: green/yellow/red)
    - Category breakdown table: Category, Target (tappable inline edit), Spent, Remaining + status badge, % Used bar
    - Status: ✓ On Track (green) / ⚠ Getting Close (yellow) / ✕ Over Budget (red)
    - "Over Budget" = actually exceeded target; "Getting Close" = ahead of daily pace
    - Tapping any category row → purchase history modal
    - No targets set → empty state with hint text
  - **Trends tab**:
    - Time frames: Weekly, 3 Months, 6 Months, Yearly
    - Per-category sparklines with dashed target line overlay
    - Tapping category row → purchase history modal (grouped by month, most recent first)
- "X of Y categories on track" banner at top of Insights
- Categories: Dining, DoorDash, Groceries (Food group), Nicotine, Entertainment, Laundry, Gas Station, plus others
- Migration: `Dining / DoorDash` split into `Dining` and `DoorDash`

### Budget Targets
- Stored in `budget_targets` Supabase table per user
- Set inline in Spending Insights Budget tab (tap TARGET column cell)
- Pace-based tracking: daily budget = target ÷ days in month × days elapsed
- Dashboard Expenses Budget card shows same budget summary

### Forecast (tab: "My Forecast")
- Windows: Biweekly / Monthly / 3 Months / 6 Months / Annual / Custom
- Date range shown in Projected Balance card label: "PROJECTED BALANCE · BIWEEKLY · Jun 21 – Jul 5, 2026"
- Custom range shows only user-selected dates, not auto-generated range
- Projects: available balance + expected income - bills (frequency-aware) - debt minimums - avg daily spend
- Upcoming Bills excludes bills not due in that window based on frequency + due_month
- Upcoming Debt Minimums shown as separate line from Upcoming Bills

### Paycheck Calculator (sidebar menu)
- Estimator tool — not a certified calculator
- Inputs: regular hours, wage, overtime hours, overtime wage (auto 1.5x), state selector (default Texas)
- Calculates: gross, federal tax (2024 brackets), SS 6.2%, Medicare 1.45%, state tax
- States: TX, CA, NY, FL, WA, GA, IL, PA, OH, NC, OR
- "Use this amount" button pre-fills new paycheck entry on Income tab
- Disclaimer shown

### Community
- Shared feed, 5 reactions, manual posts
- Auto-posts: debt paid off, under 30% util, savings goal reached, debt-free
- Amounts never shown in auto-posts
- Auto-post toggle per user (default on)

### Notifications (🔔)
- Shared pot invites, overdue bills, bills due within 5 days, paychecks awaiting confirmation, new community posts
- Clears on close; last-visit stored in localStorage

### Settings (sidebar, below Paycheck Calculator)
- Display name, Starting Cash on Hand, Auto-post toggle
- No theme selector — Carbon is hardcoded

### Auth
- Password reset: `PASSWORD_RECOVERY` event handled → shows Set New Password screen before app loads
- Invited users: shown Set Password screen after setting display name
- Welcome screen on first login (forces display name)
- First-time onboarding tour (4-step spotlight walkthrough, stored completion flag)

### How to Use (sidebar)
- Covers all current features including Payday Routine (4 steps), Budget Targets, Trends, available balance, bill frequencies, Paycheck Calculator, debt collapse, Pay All Due, early paycheck confirmation, confirmed income modal

### Other
- PWA: apple-touch-icon.png, favicon.ico
- 60-second polling for shared/community/bills updates
- All date inputs auto-populate current year / today's date when empty

## Available Balance Formula
```
Available Balance = Starting Cash
  + paychecks where pay_date <= today (or confirmed early)
  - debt payments (all time)
  - bill payments (all time)
  - expenses (all time)
  - savings deposits (excluding __starting_balance__ transactions)
  - shared pot deposits by this user
```

## Income Less Bills Formula (Budget card)
```
Income Less Bills =
  (confirmed paychecks this month + expected unconfirmed periods this month)
  - (unpaid bills due this month, frequency-aware, excluding already-paid bills)
```

## Common Bugs to Watch For
- Template literal nesting in JS — always validate with node --check
- Supabase RLS blocking inserts — check policy with pg_policies query
- `shared_members` infinite recursion — fixed with my_pot_ids() SECURITY DEFINER function
- Future paychecks leaking into balance — filter by pay_date <= today (unless confirmed early)
- Starting balance deposits double-counting — exclude notes='__starting_balance__'
- Input focus lost after one keystroke — do NOT trigger full re-renders from oninput/onkeyup events
- Non-monthly bills appearing every month — always check frequency + due_month before including in totals
- `due_month` column missing — run: `ALTER TABLE bills ADD COLUMN IF NOT EXISTS due_month integer;`
- Auto preview opening after file save — tell Claude Code "do not open or preview files after changes"

## Prompt to Start New Conversations
Paste this at the start of a new chat:

"I'm building a personal finance web app called Budget Tracker. The working file is at /home/claude/budget-tracker/index.html and is copied to /mnt/user-data/outputs/index.html on each update. Read PROJECT.md for full context. Always validate JS with node --check before copying to outputs. Wait for my explicit go-ahead before generating and sending any file. Do not open or preview files after making changes."
