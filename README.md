### Overview

**Goal:**  
Design a cross‑platform mobile app (Android + iOS) and a WordPress plugin that together allow judges to submit speech contest ballots digitally, tally results, and apply tiebreaker rules in line with the Toastmasters Speech Contest Rulebook.

---

## 1. System architecture

**Components:**

- **Mobile app (Flutter or React Native)**
  - Judge authentication
  - Ballot submission (standard + tiebreaker)
- **WordPress plugin**
  - REST API endpoints
  - Data storage and tally logic
  - Admin UI for Chief Judge
  - Ballot purge mechanism

**High‑level flow:**

1. Judge logs into mobile app (auth via WordPress).
2. App fetches contest + contestant list.
3. Judge submits ballot via REST.
4. Plugin stores ballot, updates tallies.
5. Chief Judge views results in WordPress admin.
6. After contest, Chief Judge purges ballots.

---

## 2. Functional requirements

### 2.1 Judge mobile app

- **FR‑1:** Authenticate judge using WordPress credentials (JWT/OAuth).
- **FR‑2:** Display list of assigned contests for the judge.
- **FR‑3:** For each contest, display list of contestants (name, speaking order).
- **FR‑4:** Standard judge:
  - Must select 1st, 2nd, 3rd place (no duplicates, no blanks).
- **FR‑5:** Tiebreaking judge:
  - Must rank all contestants in strict order (no duplicates, no omissions).
- **FR‑6:** Submit ballot via REST API.
- **FR‑7:** Show confirmation and prevent duplicate submissions (or allow overwrite with explicit confirmation).
- **FR‑8:** Work under intermittent connectivity (queue submission and retry).

### 2.2 WordPress plugin

- **FR‑9:** Provide REST endpoints for:
  - Judge ballot submission
  - Tiebreaker ballot submission
  - Contest/contestant retrieval
  - Results retrieval
  - Ballot purge
- **FR‑10:** Store ballots in WordPress database (custom tables or post types).
- **FR‑11:** Enforce one ballot per judge per contest (with optional overwrite).
- **FR‑12:** Compute results:
  - 1st = 3 points, 2nd = 2 points, 3rd = 1 point.
  - Aggregate per contestant.
- **FR‑13:** Detect ties and resolve using tiebreaker ranking.
- **FR‑14:** Provide admin UI:
  - View ballots (without judge names for contest chair).
  - View final results (ranked list).
  - Show whether tiebreaker was used.
- **FR‑15:** Provide “Destroy ballots” action that deletes all ballots for a contest.

---

## 3. Non‑functional requirements

- **NFR‑1:** Cross‑platform: Android + iOS from single codebase.
- **NFR‑2:** Secure: HTTPS only, JWT/OAuth, no plain‑text passwords.
- **NFR‑3:** Performance: ballot submission < 2 seconds on normal network.
- **NFR‑4:** Reliability: handle temporary network failures gracefully.
- **NFR‑5:** Privacy: judge identities visible only to Chief Judge (not to contest chair or others).
- **NFR‑6:** Auditability: log ballot submissions (timestamp, judge ID, contest ID) until purge.

---

## 4. Data model

### 4.1 WordPress database

You can implement as **custom tables** (recommended) or **custom post types**. Below assumes custom tables.

#### Table: `tm_contests`

- `id` (PK, int, auto)
- `name` (varchar)
- `type` (enum: international, evaluation, humorous, etc.)
- `date` (datetime)
- `status` (enum: draft, active, closed, purged)

#### Table: `tm_contestants`

- `id` (PK)
- `contest_id` (FK → tm_contests.id)
- `name` (varchar)
- `club` (varchar, optional)
- `speaking_order` (int)

#### Table: `tm_judges`

- `id` (PK)
- `wp_user_id` (FK → wp_users.ID)
- `is_tiebreaker` (bool)

#### Table: `tm_ballots`

- `id` (PK)
- `contest_id` (FK)
- `judge_id` (FK → tm_judges.id)
- `rank_1_contestant_id` (FK → tm_contestants.id)
- `rank_2_contestant_id` (FK)
- `rank_3_contestant_id` (FK)
- `created_at` (datetime)
- `updated_at` (datetime)

#### Table: `tm_tiebreaker_ballots`

- `id` (PK)
- `contest_id` (FK)
- `judge_id` (FK → tm_judges.id, must be is_tiebreaker = true)
- `ranking_json` (text; JSON array of contestant IDs in order)
- `created_at` (datetime)

---

## 5. REST API specification

Base namespace: `/wp-json/contest/v1`

### 5.1 Authentication

- Use WordPress JWT or OAuth plugin.
- Mobile app obtains token via:
  - `POST /wp-json/jwt-auth/v1/token` (or equivalent).
- All contest endpoints require `Authorization: Bearer <token>`.

### 5.2 Endpoints

#### 5.2.1 Get contests for judge

`GET /wp-json/contest/v1/contests`

**Response:**

```json
[
  {
    "id": 45,
    "name": "Division A International Speech Contest",
    "type": "international",
    "date": "2026-04-18T10:00:00Z",
    "status": "active",
    "is_tiebreaker": false
  }
]
```

#### 5.2.2 Get contestants for contest

`GET /wp-json/contest/v1/contests/{contest_id}/contestants`

**Response:**

```json
[
  { "id": 8, "name": "Alice Smith", "speaking_order": 1 },
  { "id": 3, "name": "Bob Jones", "speaking_order": 2 }
]
```

#### 5.2.3 Submit standard ballot

`POST /wp-json/contest/v1/ballot`

**Request:**

```json
{
  "contest_id": 45,
  "rank_1": 8,
  "rank_2": 3,
  "rank_3": 12
}
```

**Validation:**

- `rank_1`, `rank_2`, `rank_3` must be distinct.
- All must be valid contestant IDs for `contest_id`.
- Judge must be assigned to contest.

**Response:**

```json
{
  "status": "ok",
  "message": "Ballot submitted."
}
```

#### 5.2.4 Submit tiebreaker ballot

`POST /wp-json/contest/v1/tiebreaker`

**Request:**

```json
{
  "contest_id": 45,
  "ranking": [8, 3, 12, 5, 7, 10]
}
```

**Validation:**

- `ranking` must contain all contestants exactly once.
- Judge must be marked as `is_tiebreaker = true`.

**Response:**

```json
{
  "status": "ok",
  "message": "Tiebreaker ballot submitted."
}
```

#### 5.2.5 Get results (admin only)

`GET /wp-json/contest/v1/contests/{contest_id}/results`

**Response:**

```json
{
  "contest_id": 45,
  "status": "active",
  "results": [
    {
      "contestant_id": 8,
      "name": "Alice Smith",
      "points": 12,
      "rank": 1,
      "tie_broken": false
    },
    {
      "contestant_id": 3,
      "name": "Bob Jones",
      "points": 12,
      "rank": 2,
      "tie_broken": true
    }
  ]
}
```

#### 5.2.6 Purge ballots (admin only)

`POST /wp-json/contest/v1/contests/{contest_id}/purge`

**Response:**

```json
{
  "status": "ok",
  "message": "All ballots purged for this contest."
}
```

---

## 6. Tally and tiebreak logic

### 6.1 Points calculation

For each ballot in `tm_ballots`:

- `rank_1_contestant_id` → +3 points
- `rank_2_contestant_id` → +2 points
- `rank_3_contestant_id` → +1 point

Aggregate per contestant:

```sql
SELECT contestant_id,
       SUM(points) AS total_points
FROM (
  SELECT rank_1_contestant_id AS contestant_id, 3 AS points FROM tm_ballots WHERE contest_id = :contest_id
  UNION ALL
  SELECT rank_2_contestant_id, 2 FROM tm_ballots WHERE contest_id = :contest_id
  UNION ALL
  SELECT rank_3_contestant_id, 1 FROM tm_ballots WHERE contest_id = :contest_id
) AS scores
GROUP BY contestant_id;
```

### 6.2 Tie detection

- Sort contestants by `total_points` descending.
- For any group with same `total_points`, mark as tie group.

### 6.3 Tie resolution using tiebreaker ballot

- Load `ranking_json` from `tm_tiebreaker_ballots` for `contest_id`.
- For each tie group:
  - Among tied contestants, the one with **higher position** (i.e., lower index) in `ranking` wins.
  - Apply ordering to break ties deterministically.
- Mark `tie_broken = true` for contestants whose final rank depended on tiebreaker.

---

## 7. Mobile app technical spec

### 7.1 Tech stack

- **Option A (recommended):** Flutter
  - Dart
  - Packages: `http`, `provider`/`riverpod`, `flutter_secure_storage`
- **Option B:** React Native
  - JavaScript/TypeScript
  - Libraries: `axios`, `react-query`, `AsyncStorage`

### 7.2 Screens

1. **Login Screen**
   - Fields: username, password
   - Action: authenticate, store JWT token securely

2. **Contest List Screen**
   - Fetch `/contests`
   - Show list of active contests
   - Indicate if judge is tiebreaker

3. **Ballot Screen (standard judge)**
   - Dropdowns or pickers for 1st, 2nd, 3rd
   - Validation: no duplicates, all required
   - Submit button → POST `/ballot`

4. **Ballot Screen (tiebreaker)**
   - Reorderable list of contestants
   - Submit button → POST `/tiebreaker`

5. **Confirmation Screen**
   - “Ballot submitted successfully.”
   - Optionally show timestamp and contest name.

### 7.3 State management

- Store:
  - Auth token
  - Judge profile (including `is_tiebreaker`)
  - Contest list
  - Contestant list per contest
- Handle offline:
  - Queue unsent ballots in local storage
  - Retry on connectivity restore

---

## 8. Security and privacy

- **HTTPS only** for all endpoints.
- **JWT/OAuth** for authentication.
- **Role checks**:
  - Only users with “Judge” role can access ballot endpoints.
  - Only users with “Chief Judge/Admin” role can access results/purge.
- **Anonymity:**
  - Results endpoint must not expose judge IDs.
  - Admin UI for contest chair must not show judge identities.
- **Data lifecycle:**
  - Ballots stored until contest is closed.
  - Purge action deletes:
    - `tm_ballots` rows for contest
    - `tm_tiebreaker_ballots` rows for contest
  - Logs may keep minimal metadata (e.g., “N ballots submitted”) without personal data.

---

## 9. Admin UI specification (WordPress)

### 9.1 Menu

- Top‑level: **“Contests”**
  - Submenu:
    - “All Contests”
    - “Ballots”
    - “Results”
    - “Settings”

### 9.2 Screens

**All Contests:**
- List of contests with:
  - Name, type, date, status
  - Actions: Edit, View Results, Purge Ballots

**Results Screen:**
- Table:
  - Contestant name
  - Total points
  - Final rank
  - Tie‑broken? (Yes/No)
- Button: “Export results (CSV)”

**Settings:**
- Configure:
  - Point values (default 3/2/1)
  - Enable/disable overwrite of ballots
  - Enable logging


