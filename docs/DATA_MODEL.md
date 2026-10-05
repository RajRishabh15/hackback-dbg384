# Data Model Specification — Espionage Rebuild

This document specifies the database schemas, field definitions, indexes, constraints, and relationships for the rebuilt Espionage platform.

---

## 1. Entity Relationship Diagram

```mermaid
erDiagram
    PARTICIPANT {
        string _id PK
        string participantId UK "Unique sequential ID (ESP-0001)"
        string name
        string email UK "Primary personal email"
        string collegeEmail
        string regNo
        string phone
        string teamType "solo | duo"
        string rsvpStatus "PENDING | CONFIRMED | DECLINED"
        string sessionToken "Indexed session token"
        datetime sessionExpiresAt
        datetime round1StartedAt "Server-enforced start time"
        datetime round1SubmittedAt
        number round1Score
        number round1Warnings
        boolean isShortlisted "Shortlist flag"
        datetime round2StartedAt "Server-enforced start time"
        datetime round2SubmittedAt
        number round2Score
        number round2AiScore
        number round2FinalScore
        boolean isWinner "Winner flag"
    }

    MCQ_QUESTION {
        string _id PK
        string questionText
        string[] options
        number correctAnswer
        string category
        string difficulty
        number points
        number order
    }

    CODING_QUESTION {
        string _id PK
        string title
        string description
        string inputFormat
        string outputFormat
        string sampleInput
        string sampleOutput
        number points
        number order
    }

    EVENT_CONFIG {
        string _id PK
        boolean round1Active
        boolean round2Active
        boolean registrationOpen
    }

    OTP_RECORD {
        string _id PK
        string email Index
        string otp
        datetime createdAt "TTL Index (Expires 10m)"
    }

    NOTIFICATION {
        string _id PK
        string title
        string message
        string type
        datetime createdAt
    }

    ORGANIZER {
        string _id PK
        string name
        string email UK
        string regNo
        string role
        boolean present
        datetime checkedAt
    }

    PARTICIPANT ||--o{ MCQ_QUESTION : "assigned questions"
    PARTICIPANT ||--o{ CODING_QUESTION : "submitted solutions"
```

---

## 2. Entity Field Specifications

### 2.1 Participant Schema (`Participant.ts`)

| Field Name | Type | Constraints / Indexes | Description |
|---|---|---|---|
| `_id` | ObjectId | Primary Key | MongoDB default record identifier. |
| `participantId` | String | Unique Index, Required | Sequential ID in `ESP-0001` format. |
| `name` | String | Required | Full name of leader or solo participant. |
| `email` | String | Unique Index, Lowercase | Personal email address. |
| `collegeEmail` | String | Lowercase | College email address. |
| `regNo` | String | Required | College registration number. |
| `phone` | String | Required | Contact phone number. |
| `teamType` | String | Enum: `['solo', 'duo']` | Participation format. |
| `partner` | Object | Optional | Embedded partner object for duo teams. |
| `rsvpStatus` | String | Enum: `['PENDING', 'CONFIRMED', 'DECLINED']` | Seat confirmation status. |
| `sessionToken` | String | Index, Select: false | Cryptographic session token. |
| `sessionExpiresAt` | Date | Optional | Expiration date of session token. |
| `round1StartedAt` | Date | Optional | Server timestamp when Round 1 test was initiated. |
| `round1SubmittedAt` | Date | Optional | Timestamp when Round 1 test was submitted. |
| `round1Score` | Number | Optional | Calculated Round 1 MCQ score. |
| `round1Warnings` | Number | Default: 0 | Count of anti-cheat violations logged server-side. |
| `isShortlisted` | Boolean | Default: false, Index | Shortlist qualification status for Round 2. |
| `round2StartedAt` | Date | Optional | Server timestamp when Round 2 contest was initiated. |
| `round2SubmittedAt` | Date | Optional | Timestamp when Round 2 contest was submitted. |
| `round2Score` | Number | Optional | Automated score from hidden test case execution. |
| `round2AiScore` | Number | Optional | LLM evaluation score component. |
| `round2FinalScore` | Number | Optional | Weighted final combined score. |
| `isWinner` | Boolean | Default: false | Winner status assigned by admin. |
| `createdAt` | Date | Default: `Date.now` | Registration timestamp. |

### 2.2 MCQ Question Schema (`MCQQuestion.ts`)

| Field Name | Type | Constraints | Description |
|---|---|---|---|
| `_id` | ObjectId | Primary Key | Question identifier. |
| `questionText` | String | Required | Prompt text for MCQ question. |
| `options` | Array of Strings | Required | Array of 4 choices. |
| `correctAnswer` | Number | Required (0-3) | Index of correct choice. |
| `category` | String | Required | Topic tag (e.g. logic, code). |
| `difficulty` | String | Required | Difficulty classification. |
| `points` | Number | Default: 2 | Weight of the question. |

### 2.3 Coding Question Schema (`CodingQuestion.ts`)

| Field Name | Type | Constraints | Description |
|---|---|---|---|
| `_id` | ObjectId | Primary Key | Problem identifier. |
| `title` | String | Required | Problem title. |
| `description` | String | Required | Problem statement markdown. |
| `inputFormat` | String | Required | Input format details. |
| `outputFormat` | String | Required | Output format details. |
| `sampleInput` | String | Required | Sample input test case. |
| `sampleOutput` | String | Required | Sample output test case. |
| `hiddenTestCases` | Array of Objects | Select: false | Inputs and expected outputs for grading. |
| `points` | Number | Default: 10 | Problem points weight. |

### 2.4 OTP Record Schema (`OTP.ts`)

| Field Name | Type | Constraints | Description |
|---|---|---|---|
| `_id` | ObjectId | Primary Key | OTP document identifier. |
| `email` | String | Index, Lowercase | Recipient email. |
| `otp` | String | Required | 6-digit one-time code. |
| `createdAt` | Date | TTL Index: 600s | Auto-deleted 10 minutes after creation. |

### 2.5 Event Config Schema (`EventConfig.ts`)

| Field Name | Type | Constraints | Description |
|---|---|---|---|
| `_id` | ObjectId | Primary Key | Singleton config record identifier. |
| `round1Active` | Boolean | Default: false | Switch controlling Round 1 access. |
| `round2Active` | Boolean | Default: false | Switch controlling Round 2 access. |
| `registrationOpen` | Boolean | Default: true | Switch controlling registration form access. |
