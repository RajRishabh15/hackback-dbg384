# Product Requirements Document (PRD) — Espionage Rebuild

## 1. Problem Statement
For college event organizers who struggle with unauthorized test submissions, timer resets, registration caps, and serverless timeouts during multi-round coding competitions, **Espionage Rebuild** provides a secure, reliable, end-to-end contest management platform, unlike generic forms or vulnerable contest prototypes.

## 2. Target User
- **Event Organizers & Admins**: Managing registrations, attendance, question banks, shortlists, AI grading, and winner announcements.
- **Student Participants**: Registering (solo/duo), logging in via OTP, taking timed MCQ tests in Round 1, and writing code in Monaco Editor in Round 2.
- **Event Volunteers**: Scanning participant QR code tickets at the event venue.

## 3. Core Flow
1. **Registration**: Participant submits registration details, passes Cloudflare Turnstile CAPTCHA, verifies email via 6-digit OTP, and receives a unique sequential ID (`ESP-0001`).
2. **RSVP & Pass**: Organizer sends RSVP email; participant confirms seat and receives attendance QR code pass.
3. **Venue Check-in**: Volunteer logs into attendance scanner with authorized credentials and scans participant QR code for Round 1 / Round 2 check-in.
4. **Login**: Participant logs in with OTP, receiving a cryptographically secure session token saved to database and cookie.
5. **Round 1 (MCQ)**: Participant starts test; server records `round1StartedAt` and enforces deadline. Anti-cheat warnings are logged server-side.
6. **Shortlisting**: Admin runs shortlisting algorithm with deterministic tie-breaking.
7. **Round 2 (Coding)**: Shortlisted participant opens Monaco Editor. Code runs in Piston sandbox. On submission, hidden test cases are executed.
8. **AI Grading & Results**: Admin triggers asynchronous/batched LLM grading via OpenRouter with prompt isolation. Winners are flagged on `Participant` schema and certificates are generated.

## 4. MoSCoW Prioritization

### Must Have
- Cryptographic session authentication & authorization middleware on all dashboard and submission routes.
- Server-enforced test start times and deadlines for Round 1 and Round 2.
- Sequential participant ID generator (`ESP-0001`) with duplicate key handling.
- Authenticated attendance check-in endpoints requiring volunteer/admin credentials.
- Refactored winner marking and certificate generation linked to `Participant` collection.

### Should Have
- Prompt injection protection for OpenRouter AI evaluation via XML delimiting and output schema validation.
- Redis-backed rate limiting across serverless lambdas.
- Deterministic tie-breaking for Round 1 shortlisting (`round1Score DESC, round1SubmittedAt ASC`).
- Question pool clearing on Round 1 reset (`redo-round`).

### Could Have
- Asynchronous batching for Round 2 AI evaluation requests with status polling.
- Turnstile CAPTCHA and rate limits on login OTP endpoints.

### Won't Have (Out of Scope)
- Payment gateway processing (retained free/RSVP flow only).
- Offline desktop client application.

## 5. Acceptance Criteria (Given / When / Then)

### Scenario 1: Unauthenticated Submission Attempt (Security Killer Test)
- **Given** an unauthenticated request sent to `/api/round1/submit` with a victim's email.
- **When** the server processes the request without a valid `Authorization` token header matching MongoDB.
- **Then** the server returns `401 Unauthorized` and does not alter the victim's test state or score.

### Scenario 2: Test Timer Enforcement (Timer Killer Test)
- **Given** a participant who started Round 1 at timestamp $T$.
- **When** the participant reloads their browser page or submits answers after timestamp $T + \text{duration} + \text{grace}$.
- **Then** the server computes remaining duration from MongoDB `round1StartedAt` and rejects late submission with `403 Test Deadline Exceeded`.

### Scenario 3: High-Volume Registration (Capacity Killer Test)
- **Given** 1,000 participants registering for the event.
- **When** new registrations are submitted beyond 900 records.
- **Then** the system assigns sequential IDs (`ESP-0901`, `ESP-0902`) instantly without CPU loops or timeouts.

### Scenario 4: Authenticated Attendance Scanner
- **Given** an unauthenticated user requesting `/api/attendance`.
- **When** the request lacks valid admin credentials header `x-admin-password`.
- **Then** the endpoint returns `401 Unauthorized`.

### Scenario 5: Winner Marking Execution
- **Given** an admin selecting a winning participant ID.
- **When** POST `/api/admin/teams/[id]/winner` is executed.
- **Then** the system updates `Participant.isWinner = true` and returns `200 OK`.
