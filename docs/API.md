# API Specification — Espionage Rebuild

This document defines the HTTP endpoints, authorization rules, request payloads, response structures, and error states for the rebuilt Espionage platform API.

---

## 1. Authentication & Registration Endpoints

### `POST /api/send-otp`
- **Caller**: Public Participant (Registration Flow)
- **Authorization**: Public (Protected by Cloudflare Turnstile & Rate Limiter)
- **Input JSON Body**:
  ```json
  {
    "email": "user@srmist.edu.in",
    "turnstileToken": "0.XXXXXX"
  }
  ```
- **Output JSON (Success 200)**:
  ```json
  { "message": "OTP sent successfully." }
  ```
- **Error Cases**:
  - `400 Bad Request`: Invalid email format or missing Turnstile token.
  - `429 Too Many Requests`: IP rate limit exceeded.

### `POST /api/verify-otp`
- **Caller**: Public Participant
- **Authorization**: Public
- **Input JSON Body**:
  ```json
  {
    "email": "user@srmist.edu.in",
    "otp": "123456"
  }
  ```
- **Output JSON (Success 200)**:
  ```json
  {
    "message": "OTP verified.",
    "sessionToken": "a3f8b...9c",
    "participant": { "participantId": "ESP-0001", "email": "user@srmist.edu.in" }
  }
  ```
- **Error Cases**:
  - `400 Bad Request`: Invalid or expired OTP.

### `POST /api/auth/send-otp` & `POST /api/auth/verify-otp`
- **Caller**: Existing Participant (Login Flow)
- **Authorization**: Public (Protected by Turnstile & Rate Limiter)
- **Input / Output**: Same structure as registration OTP, verifying against existing RSVP-confirmed participants.

---

## 2. Participant Dashboard & Examination Endpoints

### `GET /api/dashboard/me`
- **Caller**: Logged-In Participant
- **Authorization**: `Authorization: Bearer <sessionToken>`
- **Input**: None (Identity resolved from session token)
- **Output JSON (Success 200)**: Participant record including scores, shortlist status, and RSVP details.
- **Error Cases**:
  - `401 Unauthorized`: Missing or invalid session token.

### `GET /api/round1/questions`
- **Caller**: Logged-In Participant
- **Authorization**: `Authorization: Bearer <sessionToken>`
- **Behavior**: Verifies `EventConfig.round1Active === true`. Records `round1StartedAt` if not previously set.
- **Output JSON (Success 200)**: Array of 30 randomized MCQ questions (excluding correct answers).

### `POST /api/round1/submit`
- **Caller**: Logged-In Participant
- **Authorization**: `Authorization: Bearer <sessionToken>`
- **Input JSON Body**:
  ```json
  {
    "answers": { "questionId_1": 2, "questionId_2": 0 }
  }
  ```
- **Behavior**: Verifies that `Date.now() <= round1StartedAt + 45m + grace`. Calculates score on server.
- **Output JSON (Success 200)**:
  ```json
  { "message": "Round 1 test submitted.", "score": 42 }
  ```
- **Error Cases**:
  - `401 Unauthorized`: Missing session token.
  - `403 Forbidden`: Test deadline exceeded or test already submitted.

### `POST /api/round2/execute`
- **Caller**: Shortlisted Logged-In Participant
- **Authorization**: `Authorization: Bearer <sessionToken>`
- **Input JSON Body**:
  ```json
  {
    "questionId": "q_123",
    "code": "def solve(): pass",
    "language": "python"
  }
  ```
- **Behavior**: Executes code against hidden test cases via Piston API sandbox.
- **Output JSON (Success 200)**: Test case results, stdout, stderr, and pass/fail counts.

### `POST /api/round2/submit`
- **Caller**: Shortlisted Logged-In Participant
- **Authorization**: `Authorization: Bearer <sessionToken>`
- **Behavior**: Locks Round 2 submissions and updates `round2SubmittedAt`.

---

## 3. Venue Attendance Endpoint

### `GET /api/attendance` & `POST /api/attendance`
- **Caller**: Event Volunteer / Admin
- **Authorization**: Header `x-admin-password: <ADMIN_PASSWORD>`
- **Input (POST)**:
  ```json
  {
    "participantId": "ESP-0042",
    "round": "round1"
  }
  ```
- **Output JSON (Success 200)**:
  ```json
  { "message": "Check-in successful.", "participant": { "participantId": "ESP-0042", "name": "John Doe" } }
  ```
- **Error Cases**:
  - `401 Unauthorized`: Missing or incorrect `x-admin-password` header.

---

## 4. Admin Management Endpoints

### `POST /api/admin/verify`
- **Caller**: Admin
- **Input JSON Body**: `{ "password": "..." }`
- **Output**: `{ "ok": true }` or `401 Unauthorized`.

### `POST /api/admin/shortlist`
- **Caller**: Admin (`x-admin-password` or JSON body password)
- **Input JSON Body**: `{ "count": 30 }`
- **Behavior**: Sorts participants by `round1Score DESC, round1SubmittedAt ASC, round1Warnings ASC`. Sets `isShortlisted = true` for top N candidates.

### `POST /api/admin/teams/[id]/winner`
- **Caller**: Admin
- **Input**: Participant ID in route parameter.
- **Behavior**: Updates `Participant.findByIdAndUpdate(id, { isWinner: true })`.

### `POST /api/admin/redo-round`
- **Caller**: Admin
- **Input JSON Body**: `{ "participantId": "ESP-0042", "round": "round1" }`
- **Behavior**: Clears round scores, submission timestamps, and resets assigned question arrays.
