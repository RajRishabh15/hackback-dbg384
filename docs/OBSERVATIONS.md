# Observations — Espionage Event Platform Audit

This document records all verified observations regarding the original Espionage repository. Every claim is cited with file paths and line numbers.

---

## 1. System Framework & Dependencies

- The platform is built using Next.js 16 (App Router), React 19, TypeScript, and Tailwind CSS 4.
  Evidence: `package.json:14-28` [Confirmed]
- Database persistence is managed via Mongoose 9 connecting to MongoDB.
  Evidence: `package.json:21` [Confirmed], `src/lib/mongodb.ts:1-35` [Confirmed]
- Code editing in Round 2 uses `@monaco-editor/react`.
  Evidence: `package.json:13` [Confirmed]
- Code execution is dispatched to an external Piston execution engine.
  Evidence: `package.json:10` [Confirmed], `src/lib/piston.ts:12-45` [Confirmed]
- Automated code evaluation utilizes OpenRouter LLM API calls.
  Evidence: `src/lib/openrouter.ts:8-30` [Confirmed], `src/lib/round2Evaluation.ts:1-60` [Confirmed]
- CAPTCHA validation uses Cloudflare Turnstile.
  Evidence: `src/lib/captcha.ts:7-35` [Confirmed]
- Outgoing emails are handled via Nodemailer over SMTP.
  Evidence: `package.json:23` [Confirmed], `src/lib/mailer.ts:10-25` [Confirmed]

---

## 2. Authentication & Authorization

- Session tokens generated during OTP verification are returned to the client but never saved to MongoDB or stored in secure cookies.
  Evidence: `src/app/api/auth/verify-otp/route.ts:50` [Confirmed]
- Dashboard profile retrieval accepts an unauthenticated `email` query parameter.
  Evidence: `src/app/api/dashboard/me/route.ts:7-15` [Confirmed]
- Round 1 test submission accepts `email` in the request body without checking authentication headers or tokens.
  Evidence: `src/app/api/round1/submit/route.ts:9-51` [Confirmed]
- Round 2 submission and execution routes take an `email` string without session token validation.
  Evidence: `src/app/api/round2/submit/route.ts:8-32` [Confirmed], `src/app/api/round2/execute/route.ts:18` [Confirmed]
- Client authentication state is saved in browser `localStorage` under key `espionage_session`.
  Evidence: `src/lib/auth.ts:3-33` [Confirmed]
- Attendance check-in and lookup routes do not verify admin or volunteer credentials.
  Evidence: `src/app/api/attendance/route.ts:70-189` [Confirmed]

---

## 3. Examination & Anti-Cheat Logic

- Round 1 test warnings and shortcut key violations are sent directly from the client request body upon submission without server verification.
  Evidence: `src/app/api/round1/submit/route.ts:49-50` [Confirmed]
- Round 2 final submissions accept client-reported violation counts without verification.
  Evidence: `src/app/api/round2/submit/route.ts:30-31` [Confirmed]
- Test timers run solely in client-side React state and reset when the browser reloads.
  Evidence: `src/app/dashboard/round1/page.tsx:40` [Confirmed], `src/app/dashboard/round2/page.tsx:77` [Confirmed]
- The participant data model does not record test start timestamps `round1StartedAt` or `round2StartedAt`.
  Evidence: `src/models/Participant.ts:135-140` [Confirmed]

---

## 4. Admin Operations & Feature Defects

- Admin authentication compares passwords against the `ADMIN_PASSWORD` environment variable.
  Evidence: `src/app/api/admin/verify/route.ts:5-18` [Confirmed]
- Admin winner marking attempts to update the `Team` collection instead of `Participant`, causing 404 errors.
  Evidence: `src/app/api/admin/teams/[id]/winner/route.ts:24-31` [Confirmed]
- Admin team listings fetch records from `Participant`.
  Evidence: `src/app/api/admin/teams/route.ts:17` [Confirmed]
- Certificate generation queries the `Team` collection which is unused during competition.
  Evidence: `src/app/api/admin/certificates/route.ts:32-45` [Confirmed]
- Manual registration generates 3-digit IDs (`ESP-[100-999]`) via a `while` loop that stalls near 900 registrations.
  Evidence: `src/app/api/register-manual/route.ts:234-242` [Confirmed]
- Participant shortlisting lacks secondary tie-breaker criteria and unconditionally clears previous shortlist flags.
  Evidence: `src/app/api/admin/shortlist/route.ts:18-25` [Confirmed]
- Redo round resets score fields but leaves assigned question IDs intact.
  Evidence: `src/app/api/admin/redo-round/route.ts:24-28` [Confirmed]

---

## 5. Security & Serverless Infrastructure

- Rate limiting uses an in-memory `Map` that resets on serverless cold starts and splits across lambdas.
  Evidence: `src/lib/rate-limit.ts:11-21` [Confirmed]
- Client IP resolution trusts `x-forwarded-for` header prefix directly, allowing header spoofing.
  Evidence: `src/lib/rate-limit.ts:76` [Confirmed]
- Source code is concatenated directly into OpenRouter prompts without comment filtering or XML boundaries.
  Evidence: `src/lib/round2Evaluation.ts:100-106` [Confirmed], `src/lib/openrouter.ts:16-18` [Confirmed]
- Round 2 AI evaluation runs sequential synchronous HTTP requests that exceed serverless gateway timeouts.
  Evidence: `src/app/api/admin/round2-evaluate/route.ts:33-80` [Confirmed]
- Registration confirmation emails are called asynchronously without `await`, causing dropped emails upon lambda freeze.
  Evidence: `src/app/api/register-manual/route.ts:276-285` [Confirmed]
- Login OTP endpoints lack rate limiting and CAPTCHA validation.
  Evidence: `src/app/api/auth/send-otp/route.ts:18-60` [Confirmed]
