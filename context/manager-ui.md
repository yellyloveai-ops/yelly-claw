# YellyClaw Manager Page — UI Spec

## Layout (full-viewport, flex column)

```
┌─────────────────────────────────────────────────────────┐
│ TOOLBAR                                                  │
├─────────────────────────────────────────────────────────┤
│ TABBAR  [ 🏠 Home ] [ 📄 #42 ✕ ] [ 📄 #43 ✕ ] ...      │
├─────────────────────────────────────────────────────────┤
│ HOME TOPBAR (view toggle + quick add + manual button)    │
├───────────────────────┬──┬──────────────────────────────┤
│ SCHEDULES PANEL       │  │ SESSIONS PANEL               │
│  (left, resizable)    │║ │  (right, flex:1)             │
│                       │║ │                              │
│                       │  │                              │
├───────────────────────┴──┴──────────────────────────────┤
│ FOOTER                                                   │
└─────────────────────────────────────────────────────────┘
```

---

## 1. Toolbar

Fixed top bar (`background: ctp-mantle`):
- **Left:** `🦀 YellyClaw` (h1, mauve color) → server status chip (`⏳ Connecting…` / `💚 N sessions` / `❌ Unreachable`)
- **Right buttons:** `💚 Health` | `⬆️ Update` | `💬 Feedback` | `⏹ Stop` (red)

---

## 2. Tabbar

Horizontal scrollable tab strip (`background: ctp-crust`):
- `🏠 Home` — always present, active by default
- Session tabs added dynamically: `📄 #<id>  ✕` (close button on each)
- Active tab has mauve bottom border + mauve text color

---

## 3. Home Topbar

Slim bar below tabbar (`background: ctp-mantle`):

```
[ ☰ Table | 📅 Calendar ]  [___ quick add text input (flex:1) ___]  [⚡ Add with AI]  [➕ Manual]  "N total"
```

- **View toggles:** `☰ Table` / `📅 Calendar` — toggle active state (mauve fill when active)
- **Quick Add input:** full-width text input, placeholder: "e.g. check my GitHub issues every day at 9am"
- **⚡ Add with AI:** spawns a claude-code session that creates the schedule via REST
- **➕ Manual:** opens the Schedule Modal for manual entry
- **Count:** dim "N total" label on far right

---

## 4. Main Area — Table View (default)

Two-panel split layout separated by a draggable 6px resizer (`col-resize` cursor, turns blue on hover/drag). Default split: 50/50.

### 4a. Schedules Panel (left)

**Batch bar** (hidden until ≥1 checkbox selected, slides in above table):
```
"N selected"  [ 🗑 Delete Selected ]  [ ✕ Cancel ]
```

**Schedule table** (`font-size:13px`, sticky header):

| Col | Content |
|-----|---------|
| ☐ | Checkbox (select all in `<th>`) |
| ✏️ | Edit button → opens modal |
| Name / Agent / Tools | Schedule name (clickable→modal, blue on hover) + `🤖 agentSpec` (mauve, 11px) + `🔧 tools` (overlay1, 10px) |
| Prompt | Truncated at 60 chars, ellipsis, clicking → filters sessions panel |
| Freq | Interval string (`1d`, `6h`, etc.) |
| ⚡ | Run Now button (disabled during cooldown with countdown tooltip) |
| Next Run | Relative time colored by urgency: red (≤2min), orange (≤15min), yellow (≤1h), normal (≤6h), dim (far) |
| Last Run | Relative time. Red if last run failed. Clickable → opens session tab. `🔄 Running` if active. |
| Count | Total run count, clickable → filters sessions |
| Actions | `🔁` auto-fix button (only if last run failed) |

Rows sorted: enabled first, then by nextRunAt asc.

---

### 4b. Sessions Panel (right)

**Header:** `🖥️ Sessions` + spacer + count `"N active, M completed"`

**Filter banner** (hidden by default, shown when filtering):
```
"Filtered by: <schedule name or date>"   [ ✕ Clear filter ]
```
Blue tinted background + blue bottom border.

**Sessions grid** — CSS grid `3em 7em 3em 1fr 6em 4em`:

```
[ ACTIVE SESSIONS (N) section header — green tint ]
  badge | time    | #id | prompt text (clickable→tab) | status   | actions
  badge | time    | #id | prompt text                | status   | actions
  ...
  [ expand ▾ preview row — 15 last log lines in monospace pre ]

[ COMPLETED SESSIONS (M) section header — dim ]
  badge | time    | #id | prompt text                | exit code| actions
  ...
```

**Badge types:**
- `⏰` yellow — schedule-sourced
- `🌱` orange — spawned by agent
- `🌐` blue — browser/API

**Status column:**
- Active: `⏳ Ns` (yellow, elapsed seconds)
- Completed: `✅` / `❌ N` (exit code) / `🔪` (killed)
- Hover → log tooltip (last 10 lines, 250ms delay, monospace fixed popover)

**Actions column:**
- Active: `✕ Kill` (red) + `▾` (preview toggle)
- Completed: `📄 Logs` (opens tab) + `▾` (preview toggle)

**Preview row** (inline expand, full-width): last 15 log lines in `<pre>`, max-height 200px, scrollable.

**Footer note:** `"Active sessions idle for 30 min are killed. Completed sessions kept 7 days."`

---

## 5. Calendar View (replaces main area when toggled)

Full-height, replaces the two-panel split entirely.

**Navigation bar** (sticky, `background: ctp-base`):
```
[ ◀◀ ] [ ◀ ]    "Mar 2026 – Apr 2026"    [ ▶ ] [ ▶▶ ]  [ Today ]
```
- `◀◀`/`▶▶` move ±2 weeks, `◀`/`▶` move ±1 week
- `Today` resets offset to 0

**Calendar grid** — 7-column CSS grid (`Su Mo Tu We Th Fr Sa` header row):
- **10-week window** (5 weeks past today, 5 future; skip weeks with no activity unless contains today)
- Each day cell: `min-height: 72px`, rounded corners (`border-radius: 6px`)
  - **Today:** mauve border, date in mauve
  - **Past:** date opacity 0.55
  - **Date label:** clickable on past/today days → filters sessions panel to that day, switches back to table view

**Day cell chips** (up to 3 shown, then `+N more` expand link):
- **Past sessions** (`chip-past`, dim border): `✅/❌/🔪/🔄 schedule_name` → click opens session tab
- **Future occurrences** (normal chip): `HH:MM schedule_name` → click opens edit modal
  - `chip-disabled` (opacity 0.4) if schedule is paused
  - `chip-running` (green border) if currently executing
  - `chip-failed` (red border) if last run failed

---

## 6. Schedule Edit Dialog (modal)

Centered modal overlay (`width: min(600px, 95vw)`, `background: ctp-mantle`, `border-radius: 12px`):

**Title:** `➕ New Schedule` or `✏️ Edit Schedule` (mauve)

**Type toggle** (pill selector):
```
[ 🕐 One-time ]  [ 🔁 Repeated ← active ]
```

**Fields:**

| Field | Type | Notes |
|-------|------|-------|
| Prompt * | textarea (4 rows) | Required. Auto-fills Name below |
| Name (auto-filled) | text, max 40 | Derived from prompt on input |
| Agent | text + datalist | Autocomplete from `/agents` list |
| Frequency | `<select>` | 30m / 1h / 6h / 12h / **1d** / 1w |
| First Run At | datetime-local | Only shown for Repeated type |
| Run At | datetime-local | Only shown for One-time (empty = run now) |
| Additional Tools | checkboxes | Bash / browser / web_search / all (*) |
| Pause on failure | checkbox | Checked by default |

**Action buttons row:**
- **Left:** `⏸ Pause` / `▶ Enable` (edit mode only) | `🗑 Delete` (red, edit mode only)
- **Right:** `Cancel` | `💾 Save` (mauve, primary)

---

## 7. Feedback Dialog (modal)

Same modal style. Triggered by `💬 Feedback` toolbar button.

- Subtitle: "Describe an improvement and YellyClaw will implement it autonomously."
- Large textarea (5 rows), auto-saved to `localStorage` as draft
- Buttons: `🚀 Submit` (primary) | `✕ Clear` | status text (e.g. `✅ Session #42 started`)
- **History list:** last 5 submissions (text + date), shown below textarea
- Submit spawns a `/run` session that self-implements the improvement

---

## 8. Footer

Single dim bar at bottom:
```
localhost:2026  ·  claude-code  ·  📅 ~/.config/yellyclaw/schedules.json  ·  📁 ~/GitHub
```
Both file paths are clickable → calls `/open-folder` to reveal in Finder.

---

## Auto-refresh intervals
- Schedules: every **10s**
- Sessions: every **5s**
- Health: every **30s**
