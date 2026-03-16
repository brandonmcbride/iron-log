# Iron Log — Project Reference

## What It Is
A personal fitness tracking web app built for Brandon McBride. Single HTML file, no framework, no build step. Designed to be self-hosted and used daily as a workout tracker with AI-generated weekly plans.

**Live URL:** https://iron-log-black.vercel.app/

---

## Infrastructure

| Component | Service | Details |
|-----------|---------|---------|
| Hosting | Vercel | Auto-deploys on every push to `main` |
| Database | Firebase Firestore | Project ID: `iron-log-b923a` |
| Repo | GitHub | `github.com/brandonmcbride/iron-log` |
| AI | Anthropic API | `claude-sonnet-4-20250514`, called client-side |

---

## Deploy Workflow

Every update follows this exact sequence:

1. Make changes here in Claude chat
2. Download the updated file from the artifact panel
3. Rename it to `index.html`
4. Overwrite the existing file at `C:\Users\BRAND\Claude Projects\Iron Log\iron-log\index.html`
5. Open Terminal in that folder and run:
```bash
git add .
git commit -m "describe the change"
git push
```
6. Vercel auto-deploys in ~30 seconds — no action needed

---

## Tech Stack

- **Single file:** `index.html` — all HTML, CSS, and JS in one file, no bundler
- **Firebase SDK:** loaded from `cdn.jsdelivr.net` (compat v10.12.2)
- **Storage abstraction:** `S` object with three layers:
  - `MEM` — in-memory cache (instant reads)
  - `localStorage` — offline fallback
  - Firestore — persistent cloud sync
- **On load:** fetches all Firestore docs into MEM cache, then renders
- **On write:** updates MEM instantly → writes localStorage → writes Firestore async

---

## App Architecture

### Exercise Library
~70 exercises defined in the `LIB` array. Each exercise has:
- `id`, `n` (name), `eq` (equipment), `tip` (coaching tip)
- `slot` — the movement category it belongs to (e.g. `h_push`, `vert_pull`, `rot_power`)
- `day` — which day index it belongs to (0=Mon through 6=Sun)
- `defS`, `defR`, `defW` — default sets, reps, weight

### Day Configuration (`DAY_CFG`)
Defines the structure of each day — sections, and which slots appear in each section. The app picks one exercise per slot when building a plan.

### Weekly Plans
- Stored in Firestore/localStorage as `il:plan:YYYY-MM-DD` (Monday's date as key)
- Each plan contains a `coachNote` and a `days` array
- Each day has sections → exercises (with sets, reps, weight baked in)
- Week 1 is auto-generated from library defaults
- Subsequent weeks are AI-generated

### Completions
Keyed as `weekKey:dayIdx:secIdx:exIdx` → `true`
Example: `2025-03-16:0:1:2` = week of Mar 16, Monday, section 1, exercise 2

### Week Structure
- Weeks run **Monday → Sunday** (dayIdx 0=Mon, 6=Sun)
- Week key = Monday's date in `YYYY-MM-DD` format

---

## Weekly Plan Generation

**Trigger:** Friday, Saturday, or Sunday — a banner appears on the current week view

**Flow:**
1. User taps "Generate Next Week"
2. App builds a summary of last week's completions + notes
3. Sends to `claude-sonnet-4-20250514` via Anthropic API (`max_tokens: 4000`)
4. Prompt includes: last week performance, compact slot→exercise map, slot list per day
5. AI returns `{ coachNote, slots }` JSON
6. App builds full plan object from the slot choices
7. Preview overlay shown — user reviews every exercise and confirms or regenerates
8. On confirm: saved to Firestore, app auto-navigates to next week

**Fallback:** If API fails, auto-generates a default plan with a note to add workout notes for better personalization next week.

---

## Workout Schedule

| Day | Session | Duration | Focus |
|-----|---------|----------|-------|
| Mon | Push + Posture | 45 min | Chest, shoulders, scapular control |
| Tue | Bike Endurance | 45 min | Zone 2 ride, triathlon base |
| Wed | Pull + Grip | 45 min | Back, hangboard, tennis elbow prevention |
| Thu | Legs + Agility | 45 min | Squats, RDL, lateral movement |
| Fri | Core + Mobility | 45 min | Turkish get-ups, anti-rotation, hip mobility |
| Sat | Ruck + Power | 75–90 min | 40-min ruck, KB complex, conditioning |
| Sun | Recovery + Reset | 45–60 min | Easy movement, deep stretch, box breathing |

---

## Equipment
Road bike · 5/10/15/25 lb dumbbells · Barbell + bench (two 25-lb + four 10-lb plates) · 25-lb and 35-lb kettlebells · Pull-up bar · Hangboard · Rucksack (25-lb plate) · 10-lb medicine ball · Resistance bands (varying strength, with handles) · Yoga mat · Foam roller

---

## Fitness Goals
- **Posture** — face pulls, Y-T-W raises, wall angels, thoracic mobility
- **Tennis** — rotator cuff health, reverse curls for tennis elbow prevention, rotational power (med ball slams), lateral movement training
- **Triathlon** — Zone 2 cycling base, run intervals, ruck for loaded endurance
- **Grip strength** — hangboard dead hangs, farmer carries, reverse curls
- **Lean muscle** — progressive overload across all strength movements

---

## Key Slots Reference

| Slot | Movement Type | Example Exercises |
|------|--------------|-------------------|
| `h_push` | Horizontal push | Barbell bench, DB bench, close-grip bench |
| `v_push` | Vertical push | DB overhead press, Arnold press |
| `post_pull` | Posterior shoulder | Face pulls, band external rotations |
| `scap_raise` | Scapular | Y-T-W raises, rear delt fly, lateral raises |
| `vert_pull` | Vertical pull | Pull-ups, band-assisted pull-ups, weighted pull-ups |
| `horiz_pull` | Horizontal pull | Bent-over row, single-arm DB row |
| `grip_str` | Grip strength | Hangboard hangs, farmer carries |
| `forearm` | Forearm health | Reverse curls |
| `knee_dom` | Knee-dominant legs | Back squat, goblet squat, front squat |
| `hip_dom` | Hip-dominant legs | RDL, single-leg RDL, sumo deadlift |
| `single_leg` | Single-leg | Walking lunges, reverse lunges, step-ups |
| `rot_power` | Rotational power | Med ball rotational slams |
| `lat_move` | Lateral movement | Shuffle intervals, lateral bounds, carioca |
| `main_ride` | Cycling | Zone 2, cadence intervals, hill repeats |
| `ruck` | Loaded endurance | Weighted ruck march |

---

## Known Issues / History
- Firebase SDK must load from `cdn.jsdelivr.net` (not `gstatic.com`) — the latter is blocked in Claude's artifact sandbox
- Day strip buttons use delegated event listeners via `data-day` attributes — inline `onclick` attrs were unreliable after innerHTML re-renders
- Tip toggle key format is `YYYY-MM-DD:dayIdx:secIdx:exIdx` — parse indices at positions `[2]` and `[3]` after splitting by `:`
- Streak calculation skips today if incomplete so a partially-done day doesn't break the streak count
- `nxtWk` and `prevWk` for navigation must be computed relative to `viewWk` (the week being viewed), not `curWk` (today's week)
