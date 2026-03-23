# DinnerFlow — Product Requirements Document

**Author:** Tyler
**Version:** v1.2 (Supabase stack)
**Last Updated:** 2026-03-22
**Status:** Draft — ready for review

---

## 1. Overview

### Purpose

DinnerFlow is a visual, drag-and-drop dinner planning app for quick, intuitive weekly and bi-weekly meal organization. It prioritizes speed, clarity, and flexibility over complexity.

### Core Concept

A 14-day planning grid (2 rows × 7 days) where users can:

- Drag meals onto days from a library sidebar
- Move entire day meals between days
- Stack multiple meals on a single day without destructive overwrites
- Shift the entire schedule forward or backward
- View tonight's dinner in a large, kitchen-friendly display

### Key Differentiator

> "The fastest possible way to rearrange 2 weeks of dinners visually."
>
> Not recipes. Not nutrition. Not complexity. Pure planning speed.

---

## 2. Target Users

### Primary

- Tyler and wife (household planners, 2-person household)

### Secondary (Future)

- Family members with shared planning access

### User Model

- All users authenticate via email/password
- All users have full edit access (no roles/permissions in MVP)
- All users in a household share a single meal plan and library

---

## 3. Platforms & Tech Stack

### Platforms

| Priority | Platform | Notes |
|----------|----------|-------|
| Primary | iPad | Landscape-first, touch-optimized |
| Secondary | Desktop web | Mouse + keyboard, responsive |
| Deferred | Phone | Not targeted for MVP |

### Architecture

Modern web app, PWA-ready (installable, but **online-first** for MVP — no offline editing).

### Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Frontend | React + Next.js | Component model, SSR for fast load |
| Drag & Drop | `@dnd-kit/core` | Best touch + pointer support, accessible, lightweight |
| Hosting | Vercel | Zero-config Next.js deploys, native Supabase marketplace integration |
| Database | Supabase (Postgres) | Relational model fits meal/day/household data naturally; SQL joins avoid client-side denormalization |
| Auth | Supabase Auth (email/password) | Integrated with database, user data lives in Postgres |
| Realtime sync | Supabase Realtime (Broadcast) | ~6ms median latency; both users see drag-drop changes instantly |
| Scheduled emails | Supabase `pg_cron` + Edge Function + Resend | Single-platform cron — no external scheduler needed |

### Why Supabase over Firebase

- **Relational data model.** Meals, days, households, and history have natural relationships — SQL with foreign keys and joins is cleaner than NoSQL document collections requiring client-side joins and denormalization.
- **Single platform.** Auth, database, realtime, cron, and edge functions all in one service. Firebase would require orchestrating Cloud Scheduler + Cloud Functions + Extensions separately.
- **Vercel-native.** Supabase is in the Vercel Marketplace with automatic env var syncing and database branching (preview deploys get preview databases).
- **No vendor lock-in.** Standard Postgres — data is exportable via `pg_dump` at any time. Self-hosting is available if needed.
- **Cost predictable.** Fixed monthly pricing vs. pay-per-operation spikes.

### Offline Strategy (MVP)

- **Online-first.** The app requires a network connection to function.
- Supabase Realtime connections auto-reconnect after brief connectivity gaps (e.g., switching Wi-Fi networks), but the app will not advertise or design for offline use in MVP.
- **Future:** Full offline support with local-first sync (e.g., PowerSync or ElectricSQL with Supabase Postgres).

---

## 4. Core Features

### 4.1 14-Day Planning Grid

**Layout:** 2 rows × 7 columns

| Row | Content |
|-----|---------|
| Top | Current week (starting from today or configured start-of-week) |
| Bottom | Next week |

**Week start:** Configurable (default: Monday). Stored per household.

**Each Day Tile displays:**

- Day name + date (auto-generated, e.g., "Mon 3/24")
- Visual indicator if today (highlighted border/background)
- Meal content: Main, Sides, Desserts
- Visual stacking indicator when multiple meal groups exist (e.g., badge count)

**Past days behavior:**

- Days before today are visually dimmed but still editable (you may want to log what you actually ate vs. what was planned).
- When a new day begins, the grid auto-advances: the oldest day rolls off the top-left and a new day appears at the bottom-right.
- Rolled-off days are **auto-archived** to history (see §4.6).

---

### 4.2 Meal Composition Model

```
DaySlot
  - date: Date
  - meals: MealGroup[]    // supports stacking (1..n groups per day)

MealGroup
  - id: string
  - main: MealRef | null  // 0 or 1
  - sides: MealRef[]      // 0..n
  - desserts: MealRef[]   // 0..n

MealRef
  - meal_id: string       // reference to library item
  - name: string          // denormalized for display speed
```

**Key decision:** A `DaySlot` contains an **array** of `MealGroup` objects, not a single meal. This directly supports the stacking model (§4.3C) without contradiction.

**Stacking limit:** Max **3 MealGroups per day** in MVP. Beyond 3, the UI shows a warning and prevents additional drops. This keeps tiles readable on iPad.

---

### 4.3 Drag & Drop System

This is the core UX. Every interaction must feel instant and forgiving.

#### A. Add Items (Library → Grid)

- Drag a meal from the left sidebar and drop onto a day tile.
- Dropping a **Main** onto a day that has no MealGroup creates one.
- Dropping a **Side** or **Dessert** appends to the first MealGroup (or creates one if empty).
- **Touch interaction:** Long-press (300ms) initiates drag. A drag handle icon is also available for discoverability.

#### B. Move Entire Meal (Grid → Grid)

- Drag a day tile's meal group to another day.
- The entire MealGroup (main + sides + desserts) moves as one unit.
- The source day's slot is cleared of that group.
- **Visual feedback:** Ghost preview of the meal group follows the pointer/finger. The source tile dims. The target tile highlights with a drop zone indicator.

#### C. Conflict Handling — Stacking Model

When dropping onto a day that already has meal(s):

- **Both meals coexist** on the same day (stacking).
- No modal, no confirmation, no interruption.
- User can later drag individual meal groups out to reorganize.
- If the target day already has 3 MealGroups (the max), the drop is **rejected** with a brief shake animation and toast: "Max 3 meals per day."

#### D. Remove Items

- Drag a meal group off the grid onto a "remove" drop zone (trash icon at bottom of grid area), or:
- Tap a meal group to select it, then tap a delete button.
- Removed meals return to the library (they are never destroyed — meals are library references).

#### E. Reorder Within a Day

- When a day has multiple stacked MealGroups, drag to reorder them vertically within the tile.

#### F. Future Enhancement

- Toggle between conflict modes: **Stack** (default) / Swap / Replace
- Keyboard shortcuts for power users (desktop)

---

### 4.4 Push Timeline

**Actions available:**

| Action | Behavior |
|--------|----------|
| Push Forward 1 Day | All meals shift one day later |
| Push Forward N Days | All meals shift N days later (user picks N via stepper, 1–7) |
| Push Backward 1 Day | All meals shift one day earlier |

**Edge cases:**

- **Overflow (push forward):** Meals that would land beyond day 14 are moved to a "Pushed Off" holding area (visible as a collapsible section below the grid). User can drag them back.
- **Underflow (push backward):** Meals that would land before today are moved to the same "Pushed Off" holding area.
- **Empty days:** Created at the front (push forward) or back (push backward), filled with empty DaySlots.
- **Scope:** Push applies to **all days from today onward** only. Past/archived days are not affected.

**UI:** Two buttons with chevron icons near the grid header. "Push Forward" and "Push Backward." Tapping opens a small popover with a stepper (1–7 days).

---

### 4.5 Left Sidebar — Meal Library

**Structure:** Tabbed interface with 3 tabs:

| Tab | Contents |
|-----|----------|
| Mains | Main course items |
| Sides | Side dish items |
| Desserts | Dessert items |

**Features:**

- **Add new items:** Inline text input at top of each tab. Type name, press Enter.
- **Edit items:** Tap item name to rename inline.
- **Delete items:** Swipe-to-delete (touch) or hover-reveal delete icon (desktop). Confirmation required ("This will remove it from all future planned days").
- **Scrollable list** within each tab.
- **Search/filter:** Text input at top of sidebar filters across all tabs.
- **Drag into grid:** Drag any item from the list onto a day tile.

**Item count display:** Badge on each tab showing count (e.g., "Mains (12)").

---

### 4.6 Meal History / Diary

**Purpose:** Inspiration when planning. "What did we eat 3 weeks ago?"

**Auto-logging:** When a day rolls off the grid (the grid advances daily), its meal data is **automatically** snapshotted to history. This is the primary logging mechanism — no manual action required.

**Display:**

- Chronological list, most recent first
- Each entry shows: date, main, sides, desserts
- Grouped by week for scannability

**Interactions:**

- **Drag back into grid:** Drag any history entry onto a day tile to re-plan it.
- **Search (future):** Filter by meal name, date range.

**Storage:** History entries are denormalized snapshots (not live references). If a library item is renamed, history retains the original name at time of cooking.

---

### 4.7 Focus Mode — Tonight's View

**Trigger:** Tap the "Tonight" button (persistent in the grid header, or tap today's tile).

**Display:**

- Full-screen overlay (not a page navigation — preserves grid state)
- Dark background, high contrast text
- Large font (minimum 48px for main dish name)
- Layout:
  ```
  ┌─────────────────────────────┐
  │      Tonight's Dinner       │
  │                             │
  │    🍽  Grilled Salmon       │
  │                             │
  │    Sides:                   │
  │      • Roasted Asparagus    │
  │      • Rice Pilaf           │
  │                             │
  │    Dessert:                 │
  │      • Lemon Sorbet         │
  │                             │
  │         [ Close ]           │
  └─────────────────────────────┘
  ```
- If multiple MealGroups are stacked on today, show them sequentially with dividers.

**Exit:** Tap anywhere outside the content, tap the Close button, or press Escape (desktop).

**Empty state:** If no meal is planned for tonight, show: "No dinner planned tonight" with a CTA: "Plan something →" (scrolls to today's tile on the grid).

---

### 4.8 Email Reminder System

**Feature:** Daily email with tonight's planned meal.

**Tech approach:**

1. A `pg_cron` job runs daily at the configured time (per-timezone scheduling via stored user preferences).
2. It invokes a Supabase Edge Function via HTTP.
3. The Edge Function queries today's meal plan for each household with reminders enabled.
4. It sends emails via Resend (or SendGrid) API.

**Configuration:**

- Toggle on/off per user (in settings screen)
- Default send time: 10:00 AM in user's local timezone
- Configurable send time (future enhancement)

**Email content:**

- Subject: "Tonight's Dinner: [Main Dish Name]"
- Body: Main, sides, desserts in simple formatted text
- If no meal planned: "No dinner planned for tonight — open DinnerFlow to plan one!" with a link to the app.

---

## 5. Data Model

### Postgres Schema (Supabase)

```sql
-- Managed by Supabase Auth (auto-created on signup)
-- auth.users: id, email, etc.

create table households (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  week_start  text not null default 'monday'  -- 'monday' | 'sunday'
);

create table user_profiles (
  id              uuid primary key references auth.users(id),
  household_id    uuid not null references households(id),
  email_reminders boolean not null default true,
  timezone        text not null default 'America/Los_Angeles'
);

create table meal_library (
  id            uuid primary key default gen_random_uuid(),
  household_id  uuid not null references households(id),
  name          text not null,
  type          text not null check (type in ('main', 'side', 'dessert')),
  created_at    timestamptz not null default now(),
  created_by    uuid references auth.users(id)
);

create table planned_days (
  id            uuid primary key default gen_random_uuid(),
  household_id  uuid not null references households(id),
  date          date not null,
  unique (household_id, date)  -- one row per household per day
);

create table meal_groups (
  id              uuid primary key default gen_random_uuid(),
  planned_day_id  uuid not null references planned_days(id) on delete cascade,
  sort_order      int not null default 0,       -- for reordering stacked meals
  main_id         uuid references meal_library(id) on delete set null
);

create table meal_group_sides (
  id              uuid primary key default gen_random_uuid(),
  meal_group_id   uuid not null references meal_groups(id) on delete cascade,
  meal_id         uuid not null references meal_library(id) on delete cascade,
  sort_order      int not null default 0
);

create table meal_group_desserts (
  id              uuid primary key default gen_random_uuid(),
  meal_group_id   uuid not null references meal_groups(id) on delete cascade,
  meal_id         uuid not null references meal_library(id) on delete cascade,
  sort_order      int not null default 0
);

create table history (
  id           uuid primary key default gen_random_uuid(),
  household_id uuid not null references households(id),
  date         date not null,
  archived_at  timestamptz not null default now(),
  meals        jsonb not null  -- denormalized snapshot: [{mainName, sideNames[], dessertNames[]}]
);

create table pushed_off (
  id            uuid primary key default gen_random_uuid(),
  household_id  uuid not null references households(id),
  meals         jsonb not null,  -- same shape as history.meals
  pushed_at     timestamptz not null default now()
);

-- Row Level Security: all tables filtered by household_id
-- Users can only access data for their own household
alter table meal_library enable row level security;
alter table planned_days enable row level security;
alter table meal_groups enable row level security;
alter table history enable row level security;
-- (policies omitted for brevity — pattern: WHERE household_id = user's household_id)
```

### Key Design Decisions

- **Normalized relational model.** No denormalization needed — Postgres joins are fast. A single query with joins can fetch the full 14-day grid: `planned_days → meal_groups → meal_library` (+ sides/desserts). No client-side assembly required.
- **History is denormalized (jsonb).** History snapshots are frozen in time — if a library item is renamed, history retains the original name. Using `jsonb` here avoids maintaining foreign key references to potentially deleted items.
- **`(household_id, date)` unique constraint** on `planned_days` — enables direct lookup by date without scanning.
- **Row Level Security (RLS)** — Supabase enforces that users can only read/write their own household's data at the database level. No application-layer auth checks needed for data isolation.
- **Cascading deletes** — Deleting a `planned_day` automatically removes its `meal_groups`, sides, and desserts. Clean and safe.
- **Concurrent edits** — Two users adding meal groups to the same day insert separate rows (no array mutation conflicts). Postgres handles this natively with row-level locking.

---

## 6. UX Principles

1. **Zero friction** — No popups during drag. No confirmations for moves. Stacking over replacing.
2. **Visual-first** — Large tiles, minimal text, clear visual hierarchy. Readable at arm's length on iPad.
3. **Forgiving** — Undo/redo support (Ctrl+Z / ⌘+Z, or undo button). Stacking prevents accidental overwrites. Nothing is permanently deleted in normal use.
4. **Fast** — Optimistic UI updates (write to Supabase in background, update UI immediately). Target: <100ms perceived latency for any drag operation.
5. **Touch-native** — All interactions designed for finger input first. Generous tap targets (minimum 44×44px per Apple HIG). Long-press to drag. No hover-dependent features for core flows.

### Undo/Redo

- Maintain an in-memory action stack (last 20 actions).
- Actions: move meal, add meal to day, remove meal from day, push timeline.
- Undo reverses the last action. Redo re-applies it.
- Stack clears on page reload (not persisted).
- UI: Undo/Redo buttons in the grid header. Keyboard shortcuts on desktop.

---

## 7. Screens Summary

| Screen | Type | Description |
|--------|------|-------------|
| Login | Page | Email/password form. "Create account" link. |
| Main Grid | Page | 14-day grid + sidebar. Primary screen. |
| Focus Mode | Overlay | Tonight's dinner, full-screen, high contrast. |
| History | Drawer/Panel | Slides in from right. Chronological past meals. |
| Settings | Modal/Page | Email reminder toggle, week start day, account management. |

---

## 8. MVP Scope

### Include

- 14-day grid with auto-advancing dates
- Drag & drop (grouped meals, stacking with 3-max limit)
- Left sidebar with 3 tabs + add/edit/delete items
- Push timeline (forward and backward, 1–7 days)
- Focus mode (tonight's view)
- Supabase Auth (email/password)
- Supabase Realtime sync
- Auto-logged history
- Undo/redo (in-memory, last 20 actions)
- Basic email reminder (daily, fixed 10 AM)

### Defer to v1.1+

- Offline editing support
- Notification time customization
- Advanced history search/filtering
- Multi-household / family separation
- Conflict mode toggle (swap/replace)
- Favorites tagging
- Breakfast/lunch support
- Recipe linking
- Grocery list generation
- AI suggestions

---

## 9. Open Questions (To Resolve Before Build)

| # | Question | Recommendation | Impact |
|---|----------|----------------|--------|
| 1 | Should stacked meals visually compress (accordion) or expand (vertical list)? | **Vertical list** — simpler, more readable. Compress only if 3 meals makes tiles too tall on iPad. Prototype both. | UI implementation |
| 2 | Do we want a "favorites" tag on library items? | Defer to v1.1. Use frequency-based sorting instead (most-used items float to top). | Library UX |
| 3 | Should the grid start from today or from the start of the current week? | **Start of current week** — more natural mental model. Past days in the week are dimmed. | Grid logic |
| 4 | What happens if both users drag to the same day simultaneously? | Each user inserts a separate `meal_groups` row — Postgres handles concurrent inserts natively with row-level locking. No data loss. | Data layer |
| 5 | Should the "Pushed Off" holding area persist across sessions? | **Yes** — stored in the `pushed_off` table. Prevents silent data loss. | Data model |

---

## 10. Success Metrics (MVP)

- **Planning speed:** A full 14-day plan can be created from scratch in under 5 minutes.
- **Daily engagement:** App opened at least once per day (check tonight's meal or adjust plan).
- **Adoption:** Both household members actively using the app within 2 weeks of launch.

---

## 11. Future Vision

Designed for expansion into:

- Breakfast / Lunch meal types (additional rows or grid modes)
- Multi-household accounts (already architected via `householdId`)
- Recipe linking (add URL or rich text to library items)
- Grocery list generation (aggregate ingredients from planned meals)
- AI suggestions ("You haven't had tacos in 3 weeks")
- Shared family calendars integration
- Photo logging (snap a photo of what you actually made)
