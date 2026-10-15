# UniAssist — University Student Services and Campus Assistant

> **Everything You Need. One University Assistant.**

UniAssist is a full-stack university platform. Students manage class schedules, tasks and deadlines, service requests, appointments, facility reservations, events and announcements. Faculty/staff process requests and manage bookings and content. Administrators manage users and see campus-wide analytics.

Every number on screen comes from MongoDB through the REST API. There are no hardcoded metrics or frontend data arrays.

## Group members and contributions

| Jude Poline Austria | Project Lead / Backend Developer | Planned the system scope and modules; set up the Express server, MongoDB connection, middleware (logger, error handler, auth) and REST routes; built the business logic for deadline tracking, request status workflow, booking conflict prevention and event capacity; integrated frontend and backend. |
| Mark Daniel Bernardo | Frontend Developer | Built the React + Vite + TypeScript client: layouts, sidebar and mobile navigation, landing, dashboard, schedule, tasks, requests, appointments, reservations, events and admin pages; set up React Router, the Axios API service and the custom hooks; created React Hook Form + Zod form validation. |
| Matt Andrew Desuyo | Database Designer / QA Tester | Designed the eight Mongoose models, their relationships, validation rules and indexes; wrote the seed script with demo data; tested the API endpoints and end-to-end workflows (overdue tasks, invalid status transitions, overlapping bookings, full events, 404 and invalid-ID errors) and reported bugs. |
| Joseph Emmanuel Gonzales | UI/UX Designer / Documentation | Designed the navy/indigo/violet interface, layouts and responsive behavior; checked usability on mobile and desktop; wrote the README, API documentation, setup and MongoDB guide, demo script and screenshots; prepared the final presentation materials. |


## Features

| Module | What it does |
| --- | --- |
| Landing / About | Public pages. The landing page previews live announcements and events from the API. |
| Authentication | JWT login, student self-registration, role-based access (`student`, `faculty`, `admin`) enforced on the server. |
| Dashboard | Today's classes, task counts, request counts, upcoming appointments/events, active announcements, recent activity, quick actions. |
| Schedules | CRUD, weekly view, search/filter, time-range validation, overlap detection per student. |
| Tasks | CRUD, computed overdue state, due today/tomorrow/soon labels, filtering, sorting, statistics. |
| Service requests | CRUD, 8 categories, 6 statuses, server-validated status workflow with history, filters, statistics, staff queue. |
| Appointments | Book, view, reschedule, cancel, complete. Availability lookup. No double booking of a staff member. No past dates. |
| Facility reservations | 11 facilities in 5 types. CRUD, availability, overlap prevention. |
| Events | CRUD, filters, registration with capacity, remaining slots, duplicate-registration prevention, attendee list. |
| Announcements | CRUD, filters, upcoming/active/expired computed from dates, active-only feed for students. |
| Admin / staff | User management (admin), request queue, appointment/reservation/event/announcement management, analytics. |
| Search | One endpoint searching tasks, classes, requests, events and announcements. |

## Tech stack

- **Frontend:** React 19, Vite, TypeScript, Tailwind CSS 4, React Router 7, React Hook Form, Zod, @hookform/resolvers, Axios
- **Backend:** Node.js, Express 5, JavaScript (CommonJS), Mongoose 9, MongoDB (Atlas compatible), JWT, bcryptjs
- **Tooling:** Git, dotenv, npm scripts

## Architecture and data flow

```
React page ──> useFetch hook / form submit ──> Axios instance (src/services/api.ts, adds JWT)
                                                     │  HTTP + JSON
                                                     ▼
Express app ──> logger ──> CORS/JSON ──> router ──> auth middleware ──> controller ──> service (business rules)
                                                                                           │
                                                                                           ▼
                                                                              Mongoose models ──> MongoDB
errors anywhere ──> centralized error handler ──> { "message": "..." }
```

```
UniAssist/
  client/src/   components/ layouts/ pages/ (+ pages/admin) hooks/ schemas/ services/ types/ routes/ utils/
  server/       config/ models/ routes/ controllers/ services/ middleware/ utils/ seed/ tests/
```

- **Controllers** read the request, check access, call services and send the response.
- **Services** hold the business logic (deadlines, overlap, workflow, capacity, statistics, slot locks).
- **One Axios instance** (`client/src/services/api.ts`) is used for every request. Its base URL is `VITE_API_BASE_URL`.
- **Custom hooks:** `useFetch` (loading/error/retry), `useDebounce`, `useAuth`, `useToast`.

## Database models and relationships

| Collection | Key fields | References |
| --- | --- | --- |
| `User` | name, email (unique), password (hashed), role, idNumber (unique, sparse), department, program, yearLevel, isActive | — |
| `Schedule` | courseCode, courseName, instructor, room, day, startTime, endTime | `user → User` |
| `Task` | title, description, subject, dueDate, priority, status (`pending`/`completed`), completedAt | `user → User` |
| `ServiceRequest` | category, title, description, priority, status, submittedAt, dueDate, completedAt, history[] | `user → User`, `history.changedBy → User` |
| `Appointment` | department, date, startTime, endTime, purpose, status | `user → User`, `staff → User` |
| `Reservation` | facility, facilityType, date, startTime, endTime, purpose, status | `user → User` |
| `Announcement` | title, content, category, priority, publishDate, expiryDate | `createdBy → User` |
| `Event` | title, description, location, date, startTime, endTime, category, capacity, registrations[], registeredCount | `registrations[] → User`, `createdBy → User` |
| `SlotLock` (internal helper) | resource, date, slot, booking — **unique index** on (resource, date, slot) | booking id |

Not stored because they are derived: a task's *overdue* state and deadline label, an announcement's *upcoming/active/expired* state, an event's *remaining slots* and *state*.

Deleting a user also deletes their schedules, tasks, requests, appointments and reservations, and removes them from event registrations.

## Business logic (all on the server)

| # | Rule | Where |
| --- | --- | --- |
| 1 | Overdue = pending **and** due date has passed. Completed tasks are never overdue. | `services/taskService.js` |
| 2 | Labels: Overdue, Due today, Due tomorrow, Due soon (≤ 3 days), Upcoming, Completed | `services/taskService.js` |
| 3 | Class overlap: `newStart < existingEnd && newEnd > existingStart`, same student and day, current record excluded on edit | `services/scheduleService.js` |
| 4 | Appointment availability and double-booking prevention per staff member | `services/bookingCore.js`, `lockService.js` |
| 5 | Facility availability and overlap prevention per facility | same |
| 6 | Request workflow: `pending → processing/rejected/cancelled`, `processing → ready/completed/rejected/cancelled`, `ready → completed/cancelled`; completed, rejected and cancelled are final. Students may only cancel their own. | `services/requestService.js` |
| 7 | Remaining slots = capacity − actual registrations. Full events reject registration. Capacity cannot be lowered below the registered count. | `services/eventService.js` |
| 8 | Dashboard, module statistics and analytics aggregated from MongoDB | `services/dashboardService.js` and module services |
| 9 | Multi-criteria search, filter and sort on every list endpoint | controllers |
| 10 | Dates must be real `YYYY-MM-DD` dates, times `HH:mm`, end after start, bookings not in the past, inside opening hours, on 15-minute steps | `utils/time.js` |
| 11 | No duplicate event registration | `services/eventService.js` |
| 12 | Simultaneous bookings: see below | `models/SlotLock.js` |

**How simultaneous bookings are handled.** Every booked 15-minute slot of a staff member or facility is one `SlotLock` document, and the collection has a unique index on `(resource, date, slot)`. When two requests try to book the same time at once, MongoDB lets only one insert the lock; the other gets a duplicate-key error, its partial locks are removed, and the API answers `409`. This needs no transactions, so it behaves the same on Atlas and on a standalone server. Cancelling or deleting a booking releases its locks.

Event registration uses one atomic `findOneAndUpdate` whose filter checks "not already registered" and "registeredCount < capacity" while it pushes the user and increments the counter. Registrations for the same event are also queued inside the API process.

Times are evaluated in `APP_TIMEZONE` (default `Asia/Manila`).

## Installation

Requirements: Node.js 18+ and a MongoDB database (MongoDB Atlas free tier or a local `mongod`).

### 1. Configure MongoDB

**Atlas:** create a free cluster → *Database Access*: add a user → *Network Access*: allow your IP → *Connect → Drivers*: copy the connection string and put the database name `uniassist` before the `?`:

```
mongodb+srv://<user>:<password>@<cluster>.mongodb.net/uniassist?retryWrites=true&w=majority
```

**Local:** `mongodb://127.0.0.1:27017/uniassist`

### 2. Server

```bash
cd server
npm install
cp .env.example .env      # then edit .env
npm run seed              # load demo data
npm run dev               # http://localhost:5000  (npm start for no auto-reload)
```

### 3. Client

```bash
cd client
npm install
cp .env.example .env
npm run dev               # http://localhost:5173
npm run build             # type-check + production build
```

## Environment variables

`server/.env`

| Variable | Example | Purpose |
| --- | --- | --- |
| `PORT` | `5000` | API port |
| `MONGO_URI` | `mongodb+srv://…/uniassist` | MongoDB connection string |
| `CLIENT_URL` | `http://localhost:5173` | Origin(s) allowed by CORS, comma separated |
| `JWT_SECRET` | long random string | Signs login tokens |
| `JWT_EXPIRES_IN` | `7d` | Token lifetime |
| `APP_TIMEZONE` | `Asia/Manila` | Timezone for "today", overdue and past-date checks |

`client/.env`

| Variable | Example | Purpose |
| --- | --- | --- |
| `VITE_API_BASE_URL` | `http://localhost:5000/api` | Base URL used by the Axios instance |

`.env` files are git-ignored. Never commit real credentials.

## Seed data

```bash
cd server && npm run seed
```

The seed **clears the UniAssist collections and re-inserts them**, so running it again never creates duplicates. It inserts 1 admin, 3 faculty/staff, 15 students, 20 schedules, 25 tasks (pending, completed, overdue), 15 service requests (all six statuses), 10 appointments, 10 reservations, 10 announcements (6 active, 2 upcoming, 2 expired) and 10 events (one full, one nearly full, two past). Dates are relative to the day you seed. All names are fictional.

Demo logins — password for every account is `Password123!`

| Role | Email |
| --- | --- |
| Admin | `admin@uniassist.edu` |
| Faculty | `ramon.delacruz@uniassist.edu` |
| Student | `miguel.santos@student.uniassist.edu` |

## Frontend routes

| Route | Page | Access |
| --- | --- | --- |
| `/` | Landing | public |
| `/about` | About | public |
| `/login`, `/register` | Sign in, create student account | public |
| `/dashboard` | Dashboard | signed in |
| `/schedule` | Weekly class schedule | signed in |
| `/tasks` | Tasks and deadlines | signed in |
| `/service-requests` | My service requests | signed in |
| `/service-requests/:id` | Request detail, progress, history, status actions | owner or staff |
| `/appointments` | My appointments | signed in |
| `/reservations` | My reservations | signed in |
| `/events` | Browse and register | signed in |
| `/announcements` | Active feed | signed in |
| `/search?q=` | Search results | signed in |
| `/admin` | Analytics | faculty, admin |
| `/admin/users` | User management | admin |
| `/admin/requests` | Request queue | faculty, admin |
| `/admin/appointments` | All appointments | faculty, admin |
| `/admin/reservations` | All reservations | faculty, admin |
| `/admin/events` | Event management | faculty, admin |
| `/admin/announcements` | Announcement management | faculty, admin |

## API documentation

Base URL: `http://localhost:5000/api`. Protected routes need `Authorization: Bearer <token>`.
Errors always look like `{ "message": "..." }` with status 400 (validation, bad id, illegal transition), 401 (not signed in), 403 (not allowed), 404 (not found), 409 (conflict / duplicate) or 500.

### Auth, users, meta

| Method | Path | Purpose | Sample request | Sample response |
| --- | --- | --- | --- | --- |
| GET | `/health` | Health check (also pings MongoDB) | — | `{"status":"ok","database":"connected","time":"…"}`, or `503` when the database is unavailable |
| GET | `/meta` | Enumerations (categories, statuses, facilities…) | — | `{"priorities":["low","medium","high"],…}` |
| POST | `/auth/register` | Create a student account | `{"name":"Ana Cruz","email":"ana@x.edu","password":"Password123!"}` | `201 {"token":"…","user":{"_id":"…","role":"student"}}` |
| POST | `/auth/login` | Sign in | `{"email":"admin@uniassist.edu","password":"Password123!"}` | `{"token":"…","user":{…}}` |
| GET | `/auth/me` | Current user | — | `{"_id":"…","name":"…","role":"student"}` |
| PUT | `/auth/me` | Update own profile | `{"program":"BSIT"}` | updated user |
| GET | `/users/staff` | Staff directory for bookings | `?department=Registrar` | `[{"_id":"…","name":"Ms. Liza Navarro","department":"Registrar"}]` |
| GET | `/users` | List users (admin) | `?role=student&q=santos` | `[{…}]` |
| POST | `/users` | Create user (admin) | `{"name":"…","email":"…","password":"…","role":"faculty"}` | `201 {…}` |
| GET | `/users/:id` | One user (admin) | — | `{…}` |
| PUT | `/users/:id` | Update user (admin) | `{"role":"faculty","isActive":true}` | `{…}` |
| DELETE | `/users/:id` | Delete user and their records (admin) | — | `{"message":"User and related records deleted","id":"…"}` |

### Schedules

| Method | Path | Purpose | Sample request | Sample response |
| --- | --- | --- | --- | --- |
| GET | `/schedules` | My classes | `?day=Monday&q=web` | `[{"courseCode":"ITE301","day":"Monday","startTime":"08:00","endTime":"10:00"}]` |
| GET | `/schedules/conflicts` | Check a time range, or list all overlapping pairs | `?day=Monday&startTime=09:00&endTime=10:30` | `{"hasConflict":true,"conflicts":[{…}]}` |
| POST | `/schedules` | Add class | `{"courseCode":"ITE301","courseName":"Web","instructor":"Prof. X","room":"Lab 402","day":"Monday","startTime":"08:00","endTime":"10:00"}` | `201 {…}` or `409 {"message":"Schedule conflict: overlaps ITE301 (Monday 08:00-10:00)"}` |
| GET | `/schedules/:id` | One class | — | `{…}` |
| PUT | `/schedules/:id` | Update class | `{"room":"Lab 403"}` | `{…}` |
| DELETE | `/schedules/:id` | Delete class | — | `{"message":"Schedule deleted","id":"…"}` |

### Tasks

| Method | Path | Purpose | Sample request | Sample response |
| --- | --- | --- | --- | --- |
| GET | `/tasks` | List with filters and sorting | `?status=overdue&priority=high&q=api&sort=priority` | `[{"title":"…","effectiveStatus":"overdue","deadlineLabel":"Overdue","daysLeft":-3}]` |
| GET | `/tasks/upcoming` | Pending tasks due within N days | `?days=7` | `[{…,"deadlineLabel":"Due tomorrow"}]` |
| GET | `/tasks/statistics` | Counts | — | `{"total":13,"pending":7,"completed":3,"overdue":3,"dueToday":1,"dueSoon":4,"completionRate":23,"openByPriority":{…},"bySubject":[…]}` |
| POST | `/tasks` | Create | `{"title":"Lab report","subject":"ITE310","dueDate":"2026-10-20T09:00:00Z","priority":"high"}` | `201 {…,"effectiveStatus":"pending"}` |
| GET | `/tasks/:id` | One task | — | `{…}` |
| PUT | `/tasks/:id` | Update / complete | `{"status":"completed"}` | `{…,"effectiveStatus":"completed","completedAt":"…"}` |
| DELETE | `/tasks/:id` | Delete | — | `{"message":"Task deleted","id":"…"}` |

### Service requests

| Method | Path | Purpose | Sample request | Sample response |
| --- | --- | --- | --- | --- |
| GET | `/service-requests` | List (students: own; staff: all, `?mine=true` for own) | `?status=pending&category=IT%20Support&from=2026-10-01&to=2026-10-31` | `[{"title":"…","status":"pending","allowedTransitions":["processing","rejected","cancelled"],"isOverdue":false}]` |
| GET | `/service-requests/statistics` | Counts | — | `{"total":15,"open":9,"byStatus":{…},"byCategory":{…},"overdue":3,"resolutionRate":20,"averageResolutionHours":108}` |
| POST | `/service-requests` | Submit | `{"category":"Registrar","title":"TOR request","description":"Two certified copies please.","priority":"high"}` | `201 {…,"status":"pending"}` |
| GET | `/service-requests/:id` | Detail with history | — | `{…,"history":[{"status":"pending","note":"Request submitted","at":"…"}]}` |
| PUT | `/service-requests/:id` | Edit content (owner while pending; staff: priority, due date) | `{"priority":"low"}` | `{…}` |
| PATCH | `/service-requests/:id/status` | Change status (workflow validated) | `{"status":"processing","note":"Assigned"}` | `{…,"status":"processing"}` or `400 {"message":"Invalid status transition: pending -> completed. Allowed: processing, rejected, cancelled."}` |
| DELETE | `/service-requests/:id` | Delete (owner if pending/cancelled; admin always) | — | `{"message":"Service request deleted","id":"…"}` |

### Appointments

| Method | Path | Purpose | Sample request | Sample response |
| --- | --- | --- | --- | --- |
| GET | `/appointments` | List | `?status=scheduled&department=Guidance&date=2026-10-15` | `[{"date":"2026-10-15","startTime":"09:00","staff":{"name":"…"},"status":"scheduled"}]` |
| GET | `/appointments/availability` | Booked and free slots of a staff member | `?staff=<id>&date=2026-10-15` | `{"booked":[{"startTime":"09:00","endTime":"09:30"}],"freeSlots":[{"startTime":"08:00","endTime":"08:30"},…]}` |
| POST | `/appointments` | Book | `{"department":"Guidance","staff":"<id>","date":"2026-10-15","startTime":"10:00","endTime":"10:30","purpose":"Career advice"}` | `201 {…}` or `409 {"message":"Mr. Paolo Mendoza is already booked at 10:00-11:00 on 2026-10-15"}` |
| GET | `/appointments/:id` | One appointment | — | `{…}` |
| PUT | `/appointments/:id` | Reschedule | `{"startTime":"11:00","endTime":"11:30"}` | `{…}` |
| PATCH | `/appointments/:id/cancel` | Cancel and free the slot | — | `{…,"status":"cancelled"}` |
| PATCH | `/appointments/:id/complete` | Mark completed (staff) | — | `{…,"status":"completed"}` |
| DELETE | `/appointments/:id` | Delete | — | `{"message":"Appointment deleted","id":"…"}` |

### Reservations

| Method | Path | Purpose | Sample request | Sample response |
| --- | --- | --- | --- | --- |
| GET | `/reservations` | List | `?facilityType=Study%20Room&status=confirmed` | `[{"facility":"Study Room A","date":"…","startTime":"13:00","endTime":"15:00"}]` |
| GET | `/reservations/facilities` | Facility catalogue | — | `[{"name":"Study Room A","type":"Study Room","capacity":6}]` |
| GET | `/reservations/availability` | Availability per facility | `?date=2026-10-15&facility=Study%20Room%20A&startTime=14:00&endTime=16:00` | `{"facilities":[{"name":"Study Room A","booked":[…],"freeSlots":[…],"available":false}]}` |
| POST | `/reservations` | Reserve | `{"facility":"Study Room A","date":"2026-10-15","startTime":"13:00","endTime":"15:00","purpose":"Group study"}` | `201 {…}` or `409 {"message":"Study Room A is already booked at 13:00-15:00 on 2026-10-15"}` |
| GET | `/reservations/:id` | One reservation | — | `{…}` |
| PUT | `/reservations/:id` | Edit | `{"endTime":"16:00"}` | `{…}` |
| PATCH | `/reservations/:id/cancel` | Cancel and free the facility | — | `{…,"status":"cancelled"}` |
| PATCH | `/reservations/:id/complete` | Mark completed (staff) | — | `{…,"status":"completed"}` |
| DELETE | `/reservations/:id` | Delete | — | `{"message":"Reservation deleted","id":"…"}` |

### Announcements

| Method | Path | Purpose | Sample request | Sample response |
| --- | --- | --- | --- | --- |
| GET | `/announcements/active` | Public active feed | `?category=Academic` | `[{"title":"…","state":"active","priority":"high"}]` |
| GET | `/announcements` | Staff: all states. Others: active only. | `?state=expired&q=typhoon` | `[{…,"state":"expired"}]` |
| GET | `/announcements/:id` | One announcement | — | `{…}` |
| POST | `/announcements` | Create (staff) | `{"title":"Exam schedule","content":"Now posted…","category":"Academic","priority":"high","publishDate":"2026-10-12T00:00:00Z","expiryDate":"2026-10-30T00:00:00Z"}` | `201 {…,"state":"upcoming"}` |
| PUT | `/announcements/:id` | Update (staff) | `{"priority":"low"}` | `{…}` |
| DELETE | `/announcements/:id` | Delete (staff) | — | `{"message":"Announcement deleted","id":"…"}` |

### Events

| Method | Path | Purpose | Sample request | Sample response |
| --- | --- | --- | --- | --- |
| GET | `/events` | List with filters | `?state=upcoming&category=Seminar&availability=open&mine=true` | `[{"title":"…","capacity":40,"registeredCount":8,"remainingSlots":32,"isFull":false,"isRegistered":false}]` |
| GET | `/events/statistics` | Totals and most popular | — | `{"total":10,"upcoming":8,"full":1,"totalCapacity":1431,"totalRegistrations":82,"averageFillRate":6,…}` |
| POST | `/events` | Create (staff) | `{"title":"React workshop","description":"Hands-on session.","location":"Lab 1","date":"2026-10-20","startTime":"13:00","endTime":"16:00","category":"Seminar","capacity":40}` | `201 {…}` |
| GET | `/events/:id` | One event | — | `{…}` |
| GET | `/events/:id/availability` | Capacity and remaining slots | — | `{"capacity":40,"registeredCount":8,"remainingSlots":32,"isFull":false,"canRegister":true}` |
| GET | `/events/:id/registrations` | Attendee list (staff) | — | `{"registeredCount":8,"attendees":[{"name":"…"}]}` |
| POST | `/events/:id/register` | Register | — | `201 {…,"isRegistered":true}` or `409 {"message":"Registration failed: this event is full"}` |
| DELETE | `/events/:id/register` | Cancel my registration | — | `{…,"isRegistered":false}` |
| PUT | `/events/:id` | Update (staff) | `{"capacity":60}` | `{…}` or `400 {"message":"Capacity cannot be lower than the 5 people already registered"}` |
| DELETE | `/events/:id` | Delete (staff) | — | `{"message":"Event deleted","id":"…"}` |

### Dashboard and search

| Method | Path | Purpose | Sample request | Sample response |
| --- | --- | --- | --- | --- |
| GET | `/dashboard/statistics` | Everything the dashboard shows, for the signed-in user | — | `{"todayClasses":[…],"tasks":{"pending":7,"overdue":3,…},"serviceRequests":{…},"upcomingAppointments":[…],"upcomingEvents":[…],"activeAnnouncements":[…],"recentActivity":[…]}` |
| GET | `/dashboard/analytics` | Campus-wide analytics (staff) | — | `{"users":{"byRole":{"student":15,"faculty":3,"admin":1}},"serviceRequests":{…},"events":{…},…}` |
| GET | `/search` | Search several collections | `?q=web&types=tasks,events&limit=5` | `{"query":"web","total":4,"results":{"tasks":[…],"events":[…]}}` |

## Deployment (Render)

The live site runs on Render as **one web service**: the Express server also serves the built React app, so the site and API share one URL. `render.yaml` holds the build and start commands.

**One-time setup:** Render dashboard → **New → Blueprint** → pick this GitHub repo → enter `MONGO_URI` (the Atlas string) → **Deploy**. Render generates `JWT_SECRET`. In Atlas, Network Access must allow `0.0.0.0/0`.

**Updating the live site:** Render redeploys on every push to `main`. On Windows:

| File | What it does |
| --- | --- |
| `start-uniassist.bat` | Starts the API, the website and the auto-deployer, then opens the browser |
| `auto-deploy.bat` | Watches the project; 60 s after your last edit it type-checks, commits and pushes. Broken code is not pushed. |
| `deploy.bat` | One-off manual deploy: type-check, ask for a message, commit and push |

The free plan sleeps after ~15 minutes without visitors; the next visit takes about a minute to wake it.

## Testing

```bash
# API integration tests (83 checks). Needs a seeded database and the server running.
cd server
npm run seed && npm start        # terminal 1
npm run test:api                 # terminal 2

# Client type-check + build
cd client && npm run build
```

`docs/ui_smoke_test.py` is the Playwright browser script used during development (56 checks, needs `pip install playwright`, the API on :5000 and the client on :5173).

## Screenshots

| | |
| --- | --- |
| ![Landing](docs/screenshots/01-landing.png) | ![Dashboard](docs/screenshots/02-dashboard.png) |
| ![Tasks](docs/screenshots/03-tasks.png) | ![Appointment conflict](docs/screenshots/05-appointment-conflict.png) |
| ![Events](docs/screenshots/06-events.png) | ![Request detail](docs/screenshots/10-request-detail.png) |
| ![Admin analytics](docs/screenshots/09-admin.png) | ![Mobile dashboard](docs/screenshots/07-mobile-dashboard.png) |

## Demo instructions

1. Seed, start the server and the client, open `http://localhost:5173`.
2. **Landing:** point out the live event and announcement previews.
3. Sign in as the **student**. On the dashboard, show today's classes and the task counts.
4. **Tasks:** create a task with a past due date → it shows *Overdue* and the Overdue card goes up by one. Tick it → *Completed*, and the count goes back down.
5. **Schedule:** add a class on Monday 09:00–10:30 → blocked with "overlaps ITE301".
6. **Service requests:** submit a request, open it, show the progress and history.
7. **Appointments:** choose Prof. Ramon Dela Cruz and tomorrow → booked slots are listed. Try 09:00–10:00 → rejected. Pick a free slot → booked.
8. **Reservations:** Study Room A tomorrow 14:00–16:00 → rejected (overlaps 13:00–15:00).
9. **Events:** "Resume Writing Clinic" is full and cannot be joined. Register for another event and watch the remaining slots drop.
10. Sign in as **admin**. Open the request from step 6 and move it through processing → ready → completed. Only legal moves are offered.
11. **Manage events:** try lowering the capacity of the full event to 2 → rejected. Show **Users** and **Analytics**.

## Known limitations

- Booking times must be on 15-minute steps (this is what makes slot locking possible).
- If the API process crashes between writing slot locks and saving the booking, orphan locks could remain. Re-seeding clears them; there is no automatic clean-up job.
- Class-schedule overlap is checked before saving but is not lock-protected, because a schedule belongs to a single user.
- The event registration queue is per API process. With several API instances, protection relies on the atomic MongoDB update alone.
- Facilities are a fixed catalogue in `server/utils/constants.js`, not an editable collection.
- No email notifications, file attachments, password reset or pagination.
- Login tokens are stored in `localStorage`.
- The interface loads the "Inter" font from Google Fonts and falls back to the system font offline.
