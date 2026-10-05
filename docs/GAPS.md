# Gaps & Improvements — Espionage Rebuild

This document details the original platform's architectural deficiencies and outlines the two primary improvements designed to resolve them.

---

## 1. Deficiencies in the Original Implementation

### A. Broken Object-Level Authorization (BOLA)
- **Deficiency**: The server generates session tokens during verification but discards them without database persistence or cookie setting (`src/app/api/auth/verify-otp/route.ts:50`).
- **Consequence**: Protected routes (`/api/dashboard/me`, `/api/round1/submit`, `/api/round2/execute`) accept unauthenticated email parameters, enabling malicious submission overwrites.

### B. Unauthenticated Venue Attendance API
- **Deficiency**: The attendance API (`src/app/api/attendance/route.ts:70`) does not enforce password verification, while the frontend (`src/app/attendance/page.tsx:115`) fails to pass authorization headers.
- **Consequence**: Public users can enumerate participant IDs and spoof attendance check-ins.

### C. Client-Side Timer & Anti-Cheat Manipulation
- **Deficiency**: Test timers and warning counters exist purely in client React state (`src/app/dashboard/round1/page.tsx:40`), with no `round1StartedAt` field in the database schema (`src/models/Participant.ts:135`).
- **Consequence**: Refreshing the browser resets test countdown clocks, and client requests can forge `{ warnings: 0 }`.

### D. Direct Prompt Injection in AI Grading
- **Deficiency**: Participant code is concatenated directly into OpenRouter user prompts (`src/lib/round2Evaluation.ts:100`).
- **Consequence**: Code comments containing prompt injection payloads can manipulate LLMs into awarding unearned 100% scores.

### E. Mismatched Database Collections on Admin Features
- **Deficiency**: Admin winner marking (`src/app/api/admin/teams/[id]/winner/route.ts:24`) and certificate generation (`src/app/api/admin/certificates/route.ts:32`) query the empty `Team` collection instead of `Participant`.
- **Consequence**: Winner assignment and certificate downloads always fail with 404 errors.

### F. Capacity Exhaustion on Participant ID Generation
- **Deficiency**: Manual registration generates 3-digit random numbers (`ESP-[100-999]`) in a `while` loop (`src/app/api/register-manual/route.ts:234`).
- **Consequence**: ID generation locks the CPU as registrations approach the 900-record cap.

---

## 2. Key Improvements in Espionage Rebuild

### Improvement 1: Cryptographic Database Session Tokens & Strict Authorization Middleware
- **What We Build**: Upon OTP verification, a 256-bit session token is saved on the `Participant` model and set in an HTTP-only secure cookie. A centralized authorization helper (`validateParticipantSession`) checks the token on every dashboard, exam, and code execution endpoint.
- **Why It Matters**: Prevents competitor impersonation, unauthorized test submissions, and profile data leakage, guaranteeing complete contest integrity.

### Improvement 2: Server-Enforced Test Deadlines & Incremental Anti-Cheat Heartbeats
- **What We Build**: We add `round1StartedAt` and `round2StartedAt` fields to MongoDB. The server computes remaining test durations independently of client clocks, rejecting late submissions. Anti-cheat events are logged via server-side heartbeat endpoints rather than client submission payloads.
- **Why It Matters**: Eliminates browser refresh timer hacks and ensures fair anti-cheat enforcement across all participants.
