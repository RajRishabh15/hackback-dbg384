# Remediation Plan — Espionage Rebuild

This document outlines the step-by-step technical remediation plan to resolve every vulnerability, logic defect, and serverless bottleneck identified during the audit.

---

## 1. Security & Authentication Fixes

### Step 1: Database Session Persistence & Authorization Middleware
1. Add `sessionToken` and `sessionExpiresAt` fields to [`src/models/Participant.ts`](file:///d:/Hackback/espionage-event/src/models/Participant.ts).
2. In [`src/app/api/auth/verify-otp/route.ts`](file:///d:/Hackback/espionage-event/src/app/api/auth/verify-otp/route.ts#L50), save `sessionToken` to MongoDB upon verification.
3. Implement `validateParticipantSession(req)` in `src/lib/accessControl.ts` to extract `Authorization: Bearer <token>` and verify it against MongoDB.
4. Enforce session validation across `/api/dashboard/me`, `/api/round1/submit`, `/api/round2/execute`, and `/api/round2/submit`.

### Step 2: Authenticated Attendance API
1. Enforce `x-admin-password` validation in [`src/app/api/attendance/route.ts`](file:///d:/Hackback/espionage-event/src/app/api/attendance/route.ts#L70) for both `GET` and `POST` methods.
2. Update [`src/app/attendance/page.tsx:115, 172`](file:///d:/Hackback/espionage-event/src/app/attendance/page.tsx#L115) to attach the header to scanner requests.

### Step 3: AI Evaluation Prompt Isolation
1. Enclose candidate code inside `<user_submission>` XML tags in [`src/lib/round2Evaluation.ts`](file:///d:/Hackback/espionage-event/src/lib/round2Evaluation.ts#L100).
2. Separate system instructions and user code roles in [`src/lib/openrouter.ts`](file:///d:/Hackback/espionage-event/src/lib/openrouter.ts#L16), instructing the LLM to ignore commands embedded in comments.

### Step 4: Server-Enforced Test Timers
1. Add `round1StartedAt` and `round2StartedAt` timestamps to [`src/models/Participant.ts`](file:///d:/Hackback/espionage-event/src/models/Participant.ts).
2. Record `round1StartedAt` on the server when a test is initiated, and reject submissions where `Date.now() > round1StartedAt + duration + grace`.

---

## 2. Feature & Logic Fixes

### Step 5: Fix Winner Marking & Certificates
1. Update [`src/app/api/admin/teams/[id]/winner/route.ts:24`](file:///d:/Hackback/espionage-event/src/app/api/admin/teams/[id]/winner/route.ts#L24) and [`src/app/api/admin/certificates/route.ts:32`](file:///d:/Hackback/espionage-event/src/app/api/admin/certificates/route.ts#L32) to query and update `Participant` collection instead of `Team`.

### Step 6: Sequential ID Generator
1. Replace 3-digit random generator in [`src/app/api/register-manual/route.ts:234`](file:///d:/Hackback/espionage-event/src/app/api/register-manual/route.ts#L234) with sequential counter (`ESP-0001` format).

### Step 7: Deterministic Shortlisting & Redo Reset
1. Add secondary tie-breaking sort parameters in [`src/app/api/admin/shortlist/route.ts:21`](file:///d:/Hackback/espionage-event/src/app/api/admin/shortlist/route.ts#L21) (`round1Score DESC, round1SubmittedAt ASC`).
2. Reset `round1QuestionIds = []` in [`src/app/api/admin/redo-round/route.ts:24`](file:///d:/Hackback/espionage-event/src/app/api/admin/redo-round/route.ts#L24).

---

## 3. Serverless & Performance Fixes

### Step 8: Distributed Redis Rate Limiting & CAPTCHA
1. Replace in-memory `Map` in [`src/lib/rate-limit.ts`](file:///d:/Hackback/espionage-event/src/lib/rate-limit.ts) with Upstash Redis / Vercel KV rate limiting.
2. Add rate limiting and Cloudflare Turnstile CAPTCHA checks to [`src/app/api/auth/send-otp/route.ts`](file:///d:/Hackback/espionage-event/src/app/api/auth/send-otp/route.ts#L18).
3. Always `await` outgoing email promises before returning `NextResponse.json()`.
