# Found Issues & Simple Fixes — Espionage Platform

This document details 3 major operational and logic defects identified in the Espionage codebase along with their root causes, impacts, and simple technical fixes.

---

## 1. 🛑 Browser Refresh Resets the Test Timer (Timer Bypass)

- **The Problem**: The 45-minute and 90-minute test clocks only exist inside the user's web browser ([`Round1 Page`](file:///d:/Hackback/espionage-event/src/app/dashboard/round1/page.tsx#L40)). The database ([`Participant.ts`](file:///d:/Hackback/espionage-event/src/models/Participant.ts#L135)) never records when a student actually started their test.
- **What Happens**: If a student is running out of time, refreshing the browser page resets the timer back to the full 45 or 90 minutes.
- **The Fix**: Save a `round1StartedAt` timestamp in MongoDB when the student first opens the test. Whenever they submit, calculate their time on the server (`Date.now() - startedAt`) and reject it if they exceeded the deadline.

---

## 2. 💥 App Crashes When Registrations Near 900 (ID Pool Overflow)

- **The Problem**: Every participant is assigned an ID like `ESP-123` using a 3-digit random number between 100 and 999 ([`register-manual/route.ts:234`](file:///d:/Hackback/espionage-event/src/app/api/register-manual/route.ts#L234)). That limits the total number of IDs to just 900.
- **What Happens**: As registrations approach 900, the code enters an infinite loop searching for an unused random number. This locks up the server CPU and crashes the registration API.
- **The Fix**: Replace the random 3-digit loop with a sequential counter (e.g., `ESP-0001`, `ESP-0002`, `ESP-0003`) based on the total participant count in MongoDB.

---

## 3. 🏆 "Mark Winner" Button Always Fails with 404 (Broken Admin Function)

- **The Problem**: When an admin clicks "Mark Winner" in the admin dashboard ([`winner/route.ts:24`](file:///d:/Hackback/espionage-event/src/app/api/admin/teams/[id]/winner/route.ts#L24)), the backend tries to find a record inside the `Team` database collection. But all Espionage participants are stored in a completely different collection called `Participant`.
- **What Happens**: The database can't find the team and always throws a `404 Team Not Found` error.
- **The Fix**: Update the route to search and update the `Participant` collection instead of `Team`.

---

## Summary Table

| Issue | Impact | Simple Fix |
|---|---|---|
| **Timer Reset** | Students get infinite test time by hitting refresh | Track start times in MongoDB (`round1StartedAt`) |
| **ID Overflow** | Server freezes after 900 registrations | Use sequential IDs (`ESP-0001`) |
| **Mark Winner 404** | Admin cannot award winners or issue certificates | Point database query to `Participant` collection |
