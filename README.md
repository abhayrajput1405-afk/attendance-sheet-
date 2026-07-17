# Roll Call — QR Attendance System

A college attendance system where each class session shows a **rotating QR code**
(regenerates every 15 seconds). Students scan it with their phone camera, it opens
a check-in page in the browser, and their location can optionally be verified
against the classroom's coordinates. No native app required.

## Stack
- **Backend**: Node.js + Express
- **DB**: SQLite via `better-sqlite3` (zero config — swap for Postgres later, see `db.js`)
- **Realtime**: Socket.IO (live check-in feed on the teacher's screen)
- **QR generation**: `qrcode`
- **Auth**: JWT + bcrypt password hashing

## Setup

```bash
npm install
npm run seed      # creates demo accounts + a demo course
npm start         # runs on http://localhost:3000
```

Open `http://localhost:3000` in a browser.

### Demo accounts (password for all: `password123`)
| Role    | Email               |
|---------|---------------------|
| Teacher | teacher@college.edu |
| Student | aarav@college.edu   |
| Student | diya@college.edu    |
| Student | kabir@college.edu   |
| Admin   | admin@college.edu   |

## Trying the full flow
1. Sign in as the teacher → select **Data Structures (CS201)** → **Start session**.
2. A QR code appears with a countdown ring around it (refreshes every 15s).
3. On your phone (or a second browser tab, signed in as a student), scan the QR —
   or just visit the URL it encodes. It auto check-ins and shows a confirmation.
4. Watch the teacher's screen: the student appears instantly in the live roll via
   Socket.IO, and the attendance % report at the bottom updates once the session ends.

## How the anti-proxy QR rotation works
See `utils/token.js`. Each session has a private `qr_secret` generated at start time.
The displayed QR encodes `HMAC(session_id + time_slot, qr_secret)`, where `time_slot`
changes every 15 seconds. The server can recompute the valid token for the current
(and immediately preceding) slot without storing every rotation — so:
- A screenshot shared in a group chat is worthless within ~15–30 seconds.
- A student can't fake a token without the server-side secret.
- Combine with the optional `lat`/`lng`/`radius_m` on a session to also reject
  check-ins from outside the classroom.

## Project layout
```
server.js            Express + Socket.IO entry point
db.js                 SQLite schema (users, courses, enrollments, sessions, attendance)
seed.js               Demo data
middleware/auth.js     JWT verification + role guard
utils/token.js         Rotating QR token generation/verification
routes/
  auth.js              register, login
  courses.js           create course, enroll students, roster
  sessions.js          start/end session, get current QR, live attendee count
  checkin.js            student-facing scan endpoint (token + geofence checks)
  reports.js           per-course and per-student attendance percentages
public/
  index.html           login / register
  teacher.html          dashboard: start session, live QR, live roll, report
  student.html          auto check-in landing page + personal attendance view
  style.css / app.js    shared frontend styling + fetch helper
```

## Moving to production
- **Swap SQLite → Postgres**: replace `db.js` with a `pg` Pool; the SQL is close
  to standard ANSI SQL already. `AUTOINCREMENT` → `SERIAL`/`GENERATED ALWAYS AS IDENTITY`,
  `datetime('now')` → `now()`.
- **Move `JWT_SECRET`** into a real `.env` (a `dotenv` load is already wired up).
- **HTTPS** is required in practice — camera/geolocation APIs need a secure context
  (works on `localhost` for local testing, but you'll need a real cert for `APP_URL` in production).
- **Rate-limit** `/checkin` and `/auth/login` (e.g. `express-rate-limit`) against brute forcing.
- Consider **device fingerprinting** or **one-check-in-per-device-per-day** heuristics
  if you want to catch a student checking in for a friend using their own phone.
