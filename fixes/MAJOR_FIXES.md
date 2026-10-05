# Top 3 Critical Fixes — Espionage Platform

This document details the top 3 critical issues in the Espionage codebase, explaining the root cause, risk, and exact code solution for each.

---

## 1. Fix #1: BOLA & Missing Session Authentication

### The Issue
- **Root Cause**: [`src/app/api/auth/verify-otp/route.ts:50`](file:///d:/Hackback/espionage-event/src/app/api/auth/verify-otp/route.ts#L50) generates a session token but never saves it to MongoDB. API routes like `/api/round1/submit` accept unauthenticated `email` strings in request JSON bodies.
- **Risk**: Attackers can send `POST` requests with a competitor's email and lock them out with a 0 test score.

### The Code Solution
1. Add `sessionToken` field to `ParticipantSchema`:
   ```typescript
   // src/models/Participant.ts
   sessionToken: { type: String, select: false }
   ```
2. Save `sessionToken` to database upon OTP verification:
   ```typescript
   // src/app/api/auth/verify-otp/route.ts
   const sessionToken = crypto.randomBytes(32).toString('hex');
   await Participant.updateOne({ email }, { sessionToken });
   return NextResponse.json({ token: sessionToken });
   ```
3. Validate session token in `/api/round1/submit`:
   ```typescript
   // src/app/api/round1/submit/route.ts
   const token = req.headers.get('Authorization')?.replace('Bearer ', '');
   const participant = await Participant.findOne({ sessionToken: token });
   if (!participant) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
   ```

---

## 2. Fix #2: Server-Enforced Test Deadlines (Timer Bypass Fix)

### The Issue
- **Root Cause**: Test countdown timers exist only in client React state (`timeLeft`). `ParticipantSchema` lacks a `round1StartedAt` timestamp field.
- **Risk**: Refreshing the browser resets countdown timers, giving contestants infinite test time.

### The Code Solution
1. Add `round1StartedAt` timestamp to `ParticipantSchema`:
   ```typescript
   // src/models/Participant.ts
   round1StartedAt: { type: Date }
   ```
2. Record `round1StartedAt` when test starts:
   ```typescript
   // src/app/api/round1/questions/route.ts
   if (!participant.round1StartedAt) {
     await Participant.updateOne({ _id: participant._id }, { round1StartedAt: new Date() });
   }
   ```
3. Enforce test deadline on submission:
   ```typescript
   // src/app/api/round1/submit/route.ts
   const elapsedMs = Date.now() - new Date(participant.round1StartedAt).getTime();
   const maxAllowedMs = (45 * 60 + 120) * 1000; // 45m + 2m grace
   if (elapsedMs > maxAllowedMs) {
     return NextResponse.json({ error: 'Test deadline exceeded.' }, { status: 403 });
   }
   ```

---

## 3. Fix #3: Sequential Participant ID Generator (CPU Infinite Loop Fix)

### The Issue
- **Root Cause**: Participant IDs are generated using a 3-digit random generator (`ESP-[100-999]`) inside a `while (!isUnique)` loop (`src/app/api/register-manual/route.ts:234`).
- **Risk**: The pool is limited to 900 IDs. Approaching 900 registrations causes database lookups to scale exponentially, creating infinite loops and server CPU lockups.

### The Code Solution
Replace the random generator with an atomic sequential calculation:

```typescript
// src/app/api/register-manual/route.ts
// Replace while (!isUnique) random loop with sequential ID format:
const totalCount = await Participant.countDocuments();
const participantId = `ESP-${String(totalCount + 101).padStart(4, '0')}`;
```
