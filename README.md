# Build My Schedule — technical overview

An attendee-facing personal-schedule feature that runs on
**[pl.nysais.org](https://pl.nysais.org)**, part of the NYSAIS `pl-site`
application (Express 5 + PostgreSQL 16 + React/Vite). Written up here for
IT teams evaluating a similar feature or wanting to understand how ours
works.

> **Non-technical overview:** [nysais.github.io/build-my-schedule](https://nysais.github.io/build-my-schedule/)

- **Sign-in:** stateless HMAC-signed magic URL (no password, no account)
- **Data:** three small tables in the events database; one advisory lock
- **Exports:** RFC 5545 `.ics` file and PDFKit-rendered PDF
- **Nag emails:** segment attendees by `not built` / `not invited`, never re-mail

---

## The attendee's flow, in HTTP calls

```
POST /api/attendee-auth/request-link      { email, event_id, purpose: "schedule" }
GET  /api/attendee-auth/verify?s=<hmac>   → sets req.session.attendee, redirects
GET  /api/attendee-auth/me                → { id, email, firstName, eventId }
GET  /api/schedule/public/:slug           → published schedule (days, blocks, sessions)
GET  /api/personal-schedule/:eventId      → { session_ids, speaker_session_ids }
POST /api/personal-schedule/:eventId/sessions       { session_id }
DEL  /api/personal-schedule/:eventId/sessions/:id
GET  /api/personal-schedule/:eventId/waitlist       → [{ session_id, position }]
DEL  /api/personal-schedule/:eventId/waitlist/:sid
GET  /api/personal-schedule/:eventId/export/ics     → text/calendar
GET  /api/personal-schedule/:eventId/export/pdf     → application/pdf
POST /api/attendee-auth/logout
```

Every `/personal-schedule/*` route is behind `requireAttendee`; the route
also asserts `eventId === req.attendee.eventId` so an attendee can only
touch their own event.

---

## Schema

Three tables carry the whole feature.

```js
// migrations/20260307_017_create_personal_schedules.js
exports.up = (knex) =>
  knex.schema.createTable('personal_schedules', (t) => {
    t.increments('id').primary();
    t.integer('attendee_id').notNullable()
     .references('id').inTable('attendees').onDelete('CASCADE');
    t.integer('session_id').notNullable()
     .references('id').inTable('sessions').onDelete('CASCADE');
    t.timestamp('added_at', { useTz: true }).defaultTo(knex.fn.now());
    t.unique(['attendee_id', 'session_id']);   // idempotency + seat-claim
    t.index('attendee_id');
    t.index('session_id');                     // for capacity counts
  });
```

A parallel `session_waitlist` table holds `(session_id, attendee_id, position)`
for oversubscribed sessions. On unselect, positions renumber and the freed
seat is claimed by the head of the waitlist (background job, out of scope
here).

The `sessions` table already carries `capacity`, `is_meal_option`, `status`,
and a `block_id` → `time_blocks` → `schedule_days` chain that lets us derive
day-scoped queries cheaply.

---

## Magic-link sign-in

No passwords, no per-attendee token rotation. The link is a stateless HMAC
signed URL, so re-sending an invite doesn't invalidate a link an attendee
already has in their inbox — a real complaint from earlier iterations.

```js
// server/routes/attendee-auth.js
const TOKEN_EXPIRY_DAYS = 30;
const SAFE_REDIRECT_RE  = /^\/schedule\/[A-Za-z0-9_-]+(\/(attendees|resources))?\/?$/;

router.post('/request-link',
  rateLimit({ windowMs: 3600000, max: 5 }),
  async (req, res) => {
    const { email, event_id, purpose, redirect_to } = req.body;
    const generic = { message: 'If your email is registered for this event, you will receive a link shortly.' };

    const attendee = await db('attendees')
      .where({ email: email.toLowerCase().trim(), event_id }).first();
    if (!attendee) return res.json(generic);   // no user enumeration

    const expUnix = Math.floor(Date.now() / 1000) + TOKEN_EXPIRY_DAYS * 86400;
    const signed  = signedLinks.sign({ p: 'schedule', aid: attendee.id, eid: event_id, exp: expUnix });
    const magicLink = `${process.env.BASE_URL}/api/attendee-auth/verify?s=${signed}` +
                      `&r=${encodeURIComponent(safeRedirectFor(redirect_to, event.slug))}`;

    await sendScheduleBuilderEmail({ to: attendee.email, firstName: attendee.first_name, magicLink, eventName: event.event_name });
    return res.json(generic);
  }
);
```

The `signedLinks.sign` / `verify` helpers wrap `crypto.createHmac('sha256', SIGNING_KEY)`
over a base64url-encoded payload. `verify` returns `{ ok, payload }` and
rejects any expired `exp`. Redirect targets are whitelisted with a regex
to prevent open-redirect abuse.

---

## Add-to-schedule: capacity, conflicts, waitlist

The whole write path runs in a single Postgres transaction, guarded by a
per-attendee-per-day advisory lock. That closes a race where two picks
from two devices could both slip under the capacity cap or violate the
"one meal per day" rule.

```js
// server/routes/personal-schedule.js — inside POST /:eventId/sessions
const txResult = await db.transaction(async (trx) => {
  // Serialize rotation-block ops for the same (attendee, day)
  await trx.raw(
    "SELECT pg_advisory_xact_lock(hashtext('pl:meal:' || ? || ':' || ?))",
    [req.attendee.id, session.day_id]
  );

  const rotationBlocks = await getRotationBlockIds(trx, session.day_id);
  const inRotation     = rotationBlocks.has(session.block_id);

  // Rule 1 — one meal per day, across rotation blocks
  if (inRotation && session.is_meal_option) { /* reject with 409 if other meal exists */ }

  // Capacity check under a row lock
  const s = await trx('sessions').where({ id: session_id }).select('capacity').forUpdate().first();
  if (s?.capacity) {
    const { count } = await trx('personal_schedules').where({ session_id }).count('* as count').first();
    if (parseInt(count) >= s.capacity) {
      await trx('session_waitlist').insert({
        session_id, attendee_id: req.attendee.id,
        position: db.raw('(SELECT COALESCE(MAX(position),0)+1 FROM session_waitlist WHERE session_id = ?)', [session_id]),
      });
      return { status: 'waitlisted' };
    }
  }

  await trx('personal_schedules')
    .insert({ attendee_id: req.attendee.id, session_id })
    .onConflict(['attendee_id', 'session_id']).ignore();

  // Rules 2 + 3 — meal-rotation auto-pair (see block comment in source)
  // ...
  return { status: 'added', autoAdded };
});
```

The client sends the request optimistically and rolls back on 409. Time
conflicts (overlapping picks on the same day) are returned as a soft
`conflict` object in the response — the pick still succeeds; the UI just
shows a toast so the attendee can decide.

---

## Meal-rotation auto-pair

Some conferences use a split-meal rotation: half the room eats in slot A
while the other half attends a workshop, and swap for slot B. Attendees
shouldn't have to model that in their heads. Three rules run inside the
POST transaction:

1. **Reject** a meal pick if another meal in a different rotation block on
   the same day is already selected.
2. **Auto-pair**: after picking a non-meal in one rotation block,
   automatically add the meal in the paired rotation block (only when
   exactly one paired block exists and the attendee hasn't picked there yet).
3. **Same-slot swap**: after picking a non-meal in a rotation block, drop
   any meal the attendee had in the same block.

Speakers who are teaching in both rotation blocks on the same day are
detected at load time and don't get an auto-added lunch (they can't eat —
they're teaching both slots).

---

## Client store (Zustand)

The whole attendee UI state lives in one small store: session, selections,
waitlist positions, speaker-locked sessions. Optimistic add/remove with a
rollback on server rejection.

```js
// client/src/store/personalSchedule.js
import { create } from 'zustand';
import useToast from './toast';

const usePersonalSchedule = create((set, get) => ({
  attendee: null,
  selectedIds: new Set(),
  waitlistedIds: new Map(),       // sessionId → position
  speakerSessionIds: new Set(),   // locked, can't unselect

  addSession: async (eventId, sessionId) => {
    const prev = new Set(get().selectedIds);
    set({ selectedIds: new Set([...prev, sessionId]) });  // optimistic

    const res = await fetch(`/api/personal-schedule/${eventId}/sessions`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ session_id: sessionId }),
    });

    if (res.ok) {
      const data = await res.json();
      set({ selectedIds: new Set(data.session_ids) });
      if (data.conflict) useToast.getState().warning(`Overlaps with "${data.conflict.session_title}"`);
      if (data.auto_added?.length) useToast.getState().info('Your lunch was auto-selected in the paired block.');
      return;
    }
    if (res.status === 409) {
      const { error } = await res.json().catch(() => ({}));
      useToast.getState().warning(error || "You've already selected a meal option.");
    }
    set({ selectedIds: prev });   // rollback
  },
  // ... removeSession, toggleSession, leaveWaitlist, checkSession, logout
}));
```

---

## Calendar export (ICS)

RFC 5545, hand-rolled — no library. Includes shared blocks (registration,
keynotes, meals everyone attends) plus the attendee's own picks. Rooms,
speakers, and descriptions come along.

```
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//NYSAIS//pl.nysais.org//EN
X-WR-CALNAME:Fall Conference 2026
BEGIN:VEVENT
UID:<sessionId>@pl.nysais.org
DTSTART;TZID=America/New_York:20261023T103000
DTEND;TZID=America/New_York:20261023T113000
SUMMARY:Advising the reluctant reader in grades 5-8
LOCATION:Room 214
DESCRIPTION:Deirdre Cavanaugh
END:VEVENT
...
END:VCALENDAR
```

PDF export uses [PDFKit](https://pdfkit.org/) — one page per day, session
title / room / speakers / times. Same data source; no duplication.

---

## Nag-email segmentation

Admins can send follow-up invites through the "Schedule invites" tab. The
segments prevent re-mailing anyone who already engaged:

- **not\_yet\_invited** — no schedule-builder email ever sent, no picks
- **not\_built** — invited before, still no picks (a follow-up nudge)
- **all\_with\_email\_not\_built** — everyone with no picks (new + previously invited)

The system tracks `schedule_builder_invited_at` on the attendee row and
`personal_schedules` row-count as the "built" signal.

---

## Ops notes

| Area | Where it lives |
|---|---|
| App | `/var/www/pl-site` (Express 5, port 3001) |
| DB | `nysais_db` (Postgres 16), user `nysais_db_user` |
| Migrations | `pl-site/migrations/`, run via `npx knex migrate:latest` on deploy |
| Email | Mailgun (`mg.nysais.org` sending domain) |
| Auth | Google OAuth for staff; magic-link only for attendees |
| Process | PM2 under `webapp` user, name `pl-site` |
| Nginx | TLS via Let's Encrypt; proxy_pass `http://127.0.0.1:3001` |
| Rate limits | 5 magic-link requests / hour / IP; 10 verifies / minute / IP |

---

## Why these particular choices

- **Stateless HMAC over stored tokens.** Attendees resend invites to
  themselves ("I lost the email"). A stored-token model would either
  rotate on every resend (breaking earlier links) or allow indefinite
  reuse (bad). A signed URL with a fixed 30-day expiry is the sweet spot.
- **Advisory lock on `(attendee, day)`, not `(session)`.** Meal-rotation
  rules span multiple sessions on the same day; locking a single row
  wouldn't cover them.
- **Optimistic client with server as truth.** Conference wifi is
  unreliable. Attendees see their pick appear instantly and only feel a
  slowdown on the rare rollback.
- **`.ics` hand-rolled.** The output is 60 lines of string concatenation
  with strict escaping; a library would be more surface than it's worth.

---

## Contact

Andrew Cooke &nbsp;·&nbsp; andrew@nysais.org
NYSAIS Professional Learning &nbsp;·&nbsp; [pl.nysais.org](https://pl.nysais.org)
