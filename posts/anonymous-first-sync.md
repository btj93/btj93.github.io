---
title: Anonymous-first sync — adding online to an offline-first puzzle app
date: 2026-06-14
permalink: /anonymous-first-sync
---

# Anonymous-first sync — adding online to an offline-first puzzle app

Sudoku Flick has been fully offline since day one. You install it, you play it, you never touch the network. That's a deliberate design choice — it's a puzzle, not a service, and your statistics live on your phone.

Then I got the request I'd been quietly dreading: *"I just got a new iPad. Can I move my progress?"*

The honest answer was no. The progress was sitting in `AsyncStorage` on the old phone, and unless you wanted to read JSON out of a `tar` of an iOS backup, that was where it stayed.

So I added online sync. But I really, really didn't want to spoil the thing that made the app pleasant in the first place — no account walls, no "sign in to play", no email gate on first launch. This post is about how I squared that circle.

<div class="github-card" data-github="btj93/sudoku-flick" data-width="400" data-height="" data-theme="default"></div>

## The principle: the user is *always* signed in

The core decision that made everything else fall into place: from the very first launch, the user has an account. They just don't know it.

Supabase makes this almost embarrassingly easy:

```ts
const { data: { session } } = await supabase.auth.getSession();
if (!session) {
  await supabase.auth.signInAnonymously();
}
```

If there's already a session in AsyncStorage, use it. If not, create an anonymous one. From the user's perspective: nothing happens. The splash screen doesn't pause, no UI changes, no permission prompt. The app boots into the puzzle as it always did.

Internally though, the world is different. The user has a Supabase user ID. RLS policies pin every database row to that ID. The sync engine has somewhere to push.

The thing this eliminates is an entire class of "is sync on?" UI conditionals. There is no "sync on" toggle. There's no "create an account to back up" upsell. Backup is always on. The optional thing — and the only thing you can opt *into* — is **attaching an email** to the account so you can sign in again from a different device.

That distinction (the account exists from day one; the email is the upgrade) is the entire UX story. Once I phrased it that way to myself, every other product decision wrote itself.

## Local-first, observed-from-outside

The second decision: existing app code does not learn about sync.

Sudoku Flick has a stack of React hooks — `useStatistics`, `useAchievements`, `useSudokuGame`, `useUsername`, `useDailyChallenge`. Every one of them reads from and writes to AsyncStorage. They've been the source of truth on-device for a year. I did not want to rewrite them.

The shape that worked: a new `SyncEngine` module that observes the existing write sites via small explicit calls, and feeds incoming changes back through the same hooks via subscription events.

```
┌── existing hook ──┐         ┌── SyncEngine ──┐
│  saveGame(record) │ ──────▶ │  enqueue(...)  │
│                   │         └────────┬───────┘
│  reads from       │                  │
│  AsyncStorage     │ ◀────────────────┤ emit "records:changed"
└───────────────────┘                  │
                                       ▼
                                ┌──────────────┐
                                │  Supabase    │
                                └──────────────┘
```

The diff to a hook is, roughly, this:

```ts
async function saveGameRecord(record: GameRecord) {
  await AsyncStorage.setItem(
    `@sudoku_flick_records_${record.id}`,
    JSON.stringify(record),
  );
  SyncEngine.enqueue({ kind: "record_upsert", record }); // ← new
}
```

One line. The hook still works without the network. The hook still works if Supabase is down. The hook still works if the user has the airplane mode on.

The principle: **the source of truth is local.** The server is a mirror.

Two alternative approaches I considered and rejected up front:

- **Direct mirror.** Each AsyncStorage write directly pushes to Supabase. Each pull overwrites local. Sounds clean. Breaks the moment you're offline (because the next pull blows away the unpushed write). Tightly couples your storage schema to your DB schema, so every storage refactor becomes a migration.
- **Event-sourced log.** Every change is an immutable event. State replays from the stream. Beautiful in theory. Wildly over-engineered for a single-player puzzle game. You'd be writing event-sourcing infrastructure for the rest of time and the only thing the user cares about is whether their trophy count is right on the new device.

The "local + outbox + per-entity merge" middle path is the boring answer. It happens to be correct.

## The outbox lives in AsyncStorage too

The other piece is the outbox — the queue of changes waiting to be pushed.

I considered an in-memory queue. A few hours of thinking about "what happens if the app is force-killed mid-flight" convinced me otherwise. The outbox needs to survive crashes. AsyncStorage is right there.

```ts
const OUTBOX_KEY = "@sudoku_flick_sync_outbox";

async function enqueue(op: SyncOp) {
  const raw = await AsyncStorage.getItem(OUTBOX_KEY);
  const queue: SyncOp[] = raw ? JSON.parse(raw) : [];
  queue.push(op);
  await AsyncStorage.setItem(OUTBOX_KEY, JSON.stringify(queue));
  triggerDrain();
}
```

`triggerDrain` is a debounced "go push the queue" function. It runs in the background and processes items in order. On success, the item is removed. On a retryable failure (network, 5xx), it stays in the queue and is retried later. On a terminal failure (an `INSERT` that violates a constraint that should never violate), it's logged loudly and discarded — but that should never happen, and if it does, I'd rather know than retry forever.

A nicety I added late: when the user signs in with email for the first time, the outbox is *flushed* before the merge happens. Otherwise you'd have pending pushes against the old anonymous user ID that suddenly belong to a different user.

## Conflict resolution, per table

Once you have multiple devices syncing, you need to answer: what wins?

Sudoku Flick has six syncable entities, and each one needed its own answer:

| Entity | Strategy |
|---|---|
| `profiles` (username, trophy slots) | Last-write-wins on `updated_at`. |
| `user_settings` (a single JSONB blob) | Last-write-wins on `updated_at`. |
| `game_records` | Insert-only. Once a record exists, it's immutable. |
| `achievements` | Idempotent. `INSERT ... ON CONFLICT DO NOTHING`. |
| `active_game` | Single-active-device lock. (See below.) |
| `daily_challenges` | Last-write-wins on `updated_at`, keyed per-day. |

Two things to notice.

**1. Most entities don't conflict.** Achievements are sets — you either have a trophy or you don't, and unlocking it again is a no-op. Game records are append-only — once you finished puzzle `game_1748432312`, that record exists forever; the only "conflict" is "we both inserted it" and `INSERT ... ON CONFLICT (id) DO NOTHING` makes that fine. Most of the sync surface is *trivially* idempotent.

**2. The active game is the one that's actually hard.** Mid-puzzle state — your current digits, your notes, your hint count, your elapsed time — *can* be edited from two devices, and last-write-wins would lose data. That one needs a real lock.

## The active-game lock

The model I landed on: at any moment, exactly one device "owns" the active game.

The `active_game` row has `device_id` and `last_heartbeat`. The owning device:

- writes `game_state` debounced 1s after each meaningful action,
- writes a heartbeat every 10s while the game is in the foreground,
- writes immediately on app background or unmount.

If a different device opens the app and wants to resume:

```ts
const { data: active } = await supabase
  .from("active_game")
  .select("*")
  .eq("user_id", userId)
  .single();

if (!active || active.device_id === myDeviceId) {
  // I already own it, or no one does. Take/keep the slot.
  return claim();
}

const stale = Date.now() - new Date(active.last_heartbeat).getTime() > 30_000;
if (stale) {
  // Old session, probably crashed. Quiet takeover.
  return claim();
}

// Someone else is actively playing. Show takeover modal.
return promptTakeover(active);
```

The takeover modal is the user-facing piece: *"You're playing this game on your iPad. Continue here (your iPad will stop)?"* If you confirm, your device writes the row with its own `device_id`. The iPad's debounced push will fail to claim the lock on next attempt and the iPad will show a "this game is now active on another device" overlay.

The thresholds matter:

- **1s debounce on push.** Frequent enough that takeover sees fresh state; sparse enough that you're not battering Supabase on every digit.
- **10s idle keepalive.** Even if you haven't touched the puzzle, the lock stays warm.
- **30s staleness threshold.** Longer than two missed keepalives, short enough that the old device's crash doesn't strand the puzzle for hours.

These numbers are stolen from how database connection pools handle stale connections, which is a problem space that has been thinking about this for forty years.

## Realtime is observational only

Supabase Realtime is one of those features that's tempting to overuse.

In Sudoku Flick, Realtime does exactly one job: **notify the client that something on the server changed, so it can re-merge.** It does not deliver state changes. It does not drive UI updates directly. It's a notification stream into the same merge handlers that the foreground pull uses.

```ts
supabase
  .channel(`user:${userId}`)
  .on("postgres_changes", { event: "*", schema: "public", table: "game_records", filter: `user_id=eq.${userId}` }, (payload) => {
    SyncEngine.handleIncoming(payload);
  })
  .subscribe();
```

`handleIncoming` is the same function the foreground pull calls when it fetches records. It writes to AsyncStorage, then emits the local change event the hook is listening for. The hook re-renders.

Why this matters: **echoes are absorbed naturally**. When I push a record from device A, Realtime fires on device A *and* device B. On A, the merge is a no-op (the record is already there with that exact content). On B, the merge inserts the row. There's no "did this come from me?" filtering needed because the merge handler is genuinely idempotent.

That property — "the merge handler is the same function regardless of where the data came from" — is what made the whole thing testable. I have a single function I can throw any payload at, from any source, and verify the local state ends up right.

## A few sharp edges I hit

A few non-obvious things from the build:

**`updated_at` triggers are non-negotiable.** Don't try to set `updated_at = now()` in the client. Clock skew exists. Phones lie. The database is the only thing whose clock matters for "what's the latest". A simple trigger:

```sql
create function touch_updated_at() returns trigger language plpgsql as $$
begin
  new.updated_at := now();
  return new;
end;
$$;

create trigger user_settings_touch
before update on user_settings
for each row execute function touch_updated_at();
```

Easy to write, easy to forget, ruinous to debug if missing.

**The first sign-in is the riskiest moment.** When an anonymous user attaches an email *and the email already has an existing account*, you have two sets of data: the anon's local data, and the existing email account's server data. The choice has to be the user's. I picked: surface a modal, *"This email already has progress. Replace local with your saved data, or keep this device's progress and discard the cloud copy?"* Either choice is destructive; the modal makes the destruction explicit.

If the email is brand new (no existing account), the upgrade is silent and lossless: the anonymous user *becomes* the email user, the user_id doesn't change, every row stays exactly where it was.

**Account deletion is harder than it sounds.** "Delete my account" needs to: invalidate the session, cascade-delete every row in every table, delete the auth user, and confirm before doing it. I put it behind a two-step modal with a type-DELETE gate. The actual deletion runs in a Supabase Edge Function so the client can't get partway through and stop.

**Anonymous users still take up auth.users rows.** If you ship to a million people who launch the app once and never come back, you've made a million anon rows. Plan for a cleanup job (Supabase has a built-in policy for this; opt in). Or, if your free tier headroom is generous, just let them sit until you care.

## What this is *not*

A few things to set expectations:

- **It's not multiplayer.** There's no realtime co-solving. Two devices for the same user, yes; two users for the same puzzle, no.
- **It's not CRDT.** The conflict resolution is per-entity strategies, not a general merge function. That's fine because the data is shaped to make conflict rare.
- **It's not zero-latency UI.** Reads come from local; UI is instant. Writes are eventually pushed. If you require "what I see is exactly what the server thinks", this isn't that. (For a Sudoku app, this is fine. For your bank app, it isn't.)

## What I'd do differently

Two things I'd plan for from day one if I were doing it again:

1. **Treat the outbox as a first-class data structure from commit one.** I retrofitted it after a week of "just queue it in memory". The retrofit was small but emotional.
2. **Decide on `updated_at` conventions before writing the first table.** I had three tables with three slightly different "what does updated_at mean?" definitions before I noticed. The triggers fixed it but the merge code briefly disagreed with itself.

Other than that, the architecture has held up. The user opens the app, it does the offline thing, and somewhere off in the background — across networks, across devices, across reinstalls — their progress is being faithfully mirrored, without ever having asked them to make an account.

That's the feature, really. Backup as a side effect. Multi-device as a one-tap upgrade. No login wall in front of a game.

Happy syncing.
