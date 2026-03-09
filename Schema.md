# DynamoDB Single-Table Schema

**Table name:** `AppTable`    
**GSIs used:** 1 (`GSI1` — `GSI1_PK` / `GSI1_SK`)

---

## Users

### Access Patterns

1. Get user profile by user ID
2. Login lookup by email → resolve to user ID
3. Bot auth lookup by Discord ID → resolve to user ID

### DB Schema

| Item           | PK                      | SK        |
| -------------- | ----------------------- | --------- |
| User Profile   | `USER#<id>`             | `PROFILE` |
| Email Lookup   | `EMAIL#<email>`         | `LOOKUP`  |
| Discord Lookup | `DISCORD#<discordId>`   | `LOOKUP`  |

```
PK: USER#<id>            SK: PROFILE
PK: EMAIL#<email>        SK: LOOKUP
PK: DISCORD#<discordId>  SK: LOOKUP
```

All three patterns are direct `GetItem` calls — no GSI needed. Lookup items (EMAIL, DISCORD) store only the target `userId`; the caller then fetches `USER#<id>` in a second read.

- `USER#` — namespace for user profile items
- `EMAIL#` — reverse-lookup namespace; maps email → userId
- `DISCORD#` — reverse-lookup namespace; maps Discord ID → userId
- `PROFILE` — fixed SK for the profile item; leaves room for future sibling items under the same PK
- `LOOKUP` — fixed SK for all pointer/lookup items; item body contains only `userId`

---

## Team

### Access Patterns

1. Get team details
2. Get all members of a team
3. Check if a specific user is in a team

### DB Schema

| Item          | PK          | SK                  |
| ------------- | ----------- | ------------------- |
| Team Metadata | `TEAM#<id>` | `METADATA`          |
| Team Member   | `TEAM#<id>` | `MEMBER#<user_id>`  |

```
PK: TEAM#<id>   SK: METADATA
PK: TEAM#<id>   SK: MEMBER#<user_id>
```

All three patterns resolve within a single partition — no GSI needed. Pattern 2 uses `Query PK = TEAM#<id>` with `begins_with(SK, "MEMBER#")`. Pattern 3 is a direct `GetItem`.

- `TEAM#` — namespace for team partitions
- `METADATA` — fixed SK for the team detail item (name, description, etc.)
- `MEMBER#` — SK prefix for membership items; embedding `user_id` makes membership checks a single `GetItem`

---

## Meal Participation

### Access Patterns

1. Get user's all meals for a date
2. Get user's specific meal
3. Opt in/out of a meal
4. All participation for a date

### DB Schema

| Item               | PK          | SK                        | GSI1_PK       | GSI1_SK                      |
| ------------------ | ----------- | ------------------------- | ------------- | ---------------------------- |
| Meal Participation | `USER#<id>` | `MEAL#<date>#<meal_type>` | `DATE#<date>` | `MEAL#<meal_type>#<user_id>` |

```
PK: USER#<id>   SK: MEAL#<date>#<meal_type>
GSI1_PK: DATE#<date>   GSI1_SK: MEAL#<meal_type>#<user_id>
```

Patterns 1–3 resolve from the main table. Pattern 1 uses `begins_with(SK, "MEAL#<date>")`. Pattern 2 is a direct `GetItem`. Pattern 3 is a `PutItem`. Pattern 4 queries **GSI1** with `GSI1_PK = DATE#<date>` to fan out across all users for that date.

- `MEAL#` — SK prefix for meal participation items
- `<date>#<meal_type>` — composite SK suffix; `<date>` is `YYYY-MM-DD`, `<meal_type>` is e.g. `BREAKFAST`, `LUNCH`, `DINNER`
- `DATE#` — GSI1_PK namespace; groups all activity items for a given date across users
- `MEAL#<meal_type>#<user_id>` — GSI1_SK; allows filtering by meal type within a date partition

---

## Work Location

### Access Patterns

1. Get user's location for a date
2. Set user's location
3. All WFH employees on a date
4. Monthly WFH count for a user

### DB Schema

| Item          | PK          | SK                    | GSI1_PK       | GSI1_SK        |
| ------------- | ----------- | --------------------- | ------------- | -------------- |
| Work Location | `USER#<id>` | `WORKLOCATION#<date>` | `DATE#<date>` | `WFH#<user_id>` |

```
PK: USER#<id>   SK: WORKLOCATION#<date>
GSI1_PK: DATE#<date>   GSI1_SK: WFH#<user_id>
```

Patterns 1 and 2 are direct `GetItem` / `PutItem` on the main table. Pattern 4 uses `begins_with(SK, "WORKLOCATION#<year-month>")` on the main table — no GSI needed. Pattern 3 queries **GSI1** with `GSI1_PK = DATE#<date>` and `begins_with(GSI1_SK, "WFH#")` to list all WFH employees on a date, filtered away from meal rows sharing the same GSI partition.

- `WORKLOCATION#` — SK prefix for work location items; `<date>` is `YYYY-MM-DD`
- `DATE#` — same GSI1_PK namespace shared with Meal Participation; overloaded to cover all date-scoped fan-out queries with a single GSI
- `WFH#` — GSI1_SK prefix for work location items; distinguishes them from `MEAL#` rows in the same GSI1 date partition

---

## Day & Meals

### Access Patterns

1. Get full day context (type + available meals)
2. Get day type only
3. Get available meals only
4. Set day type
5. Set available meals

### DB Schema

| Item       | PK           | SK         |
| ---------- | ------------ | ---------- |
| Day Config | `DAY#<date>` | `METADATA` |
| Day Meals  | `DAY#<date>` | `MEALS`    |

```
PK: DAY#<date>   SK: METADATA
PK: DAY#<date>   SK: MEALS
```

No GSI needed. All five patterns resolve within a single partition. Pattern 1 queries `PK = DAY#<date>` and returns both sibling items in one round-trip. Patterns 2–5 are point reads/writes.

- `DAY#` — namespace for day partitions; `<date>` is `YYYY-MM-DD`
- `METADATA` — SK for the day-type item (e.g., `OFFICE`, `WFH`, `HOLIDAY`)
- `MEALS` — SK for the available-meals config item for that day

---

## WFH Period

### Access Patterns

1. List all WFH periods
2. Is date in any WFH period?

### DB Schema

| Item       | PK          | SK                        |
| ---------- | ----------- | ------------------------- |
| WFH Period | `WFHPERIOD` | `<start_date>#<end_date>` |

```
PK: WFHPERIOD   SK: <start_date>#<end_date>
```

No GSI needed. All WFH periods share a singleton partition. Pattern 1 queries `PK = WFHPERIOD` to return all periods sorted chronologically. Pattern 2 queries up to today's date and filters `end_date >= target` in the application.

- `WFHPERIOD` — fixed singleton PK; collocates all WFH period records in one partition for cheap full-list queries
- `<start_date>#<end_date>` — composite SK in `YYYY-MM-DD#YYYY-MM-DD` format; ISO-8601 ensures natural chronological sort by start date

---

## Audit Log

### Access Patterns

1. Write an audit entry on every mutation
2. Get all changes made by a specific user
3. Get all changes made to a specific user's records
4. Get changes of a specific entity type by a user

### DB Schema

| Item        | PK                                 | SK                            | GSI1_PK                | GSI1_SK                           |
| ----------- | ---------------------------------- | ----------------------------- | ---------------------- | --------------------------------- |
| Audit Entry | `AUDIT#<entity_type>#<entity_id>`  | `<timestamp>#<actor_user_id>` | `USER#<actor_user_id>` | `AUDIT#<entity_type>#<timestamp>` |

```
PK: AUDIT#<entity_type>#<entity_id>   SK: <timestamp>#<actor_user_id>
GSI1_PK: USER#<actor_user_id>         GSI1_SK: AUDIT#<entity_type>#<timestamp>
```

Pattern 1 is a `PutItem`. Pattern 3 queries the main table with `PK = AUDIT#USER#<target_id>` — no GSI needed since `entity_type = USER` and `entity_id = <target_id>` are embedded in the PK. Patterns 2 and 4 query **GSI1**: pattern 2 uses `GSI1_PK = USER#<actor_user_id>` with `begins_with(GSI1_SK, "AUDIT#")`; pattern 4 narrows further with `begins_with(GSI1_SK, "AUDIT#<entity_type>#")`.

- `AUDIT#` — PK namespace for audit entries; `<entity_type>#<entity_id>` collocates all history for one entity, sorted by time via SK
- `<timestamp>` — ISO-8601 UTC (e.g. `2026-03-10T08:00:00Z`); leading the SK ensures chronological sort on the main table
- `USER#` — GSI1_PK reuses the same namespace as user profile items; the GSI is sparse so only items that write `GSI1_PK` are indexed — user profile items do not write it, so there is no collision
- `AUDIT#<entity_type>#` — GSI1_SK prefix enables filtering by entity type within a user's audit history

---

## GSI Summary

| GSI  | GSI1_PK                | GSI1_SK                           | Patterns served                                      |
| ---- | ---------------------- | --------------------------------- | ---------------------------------------------------- |
| GSI1 | `DATE#<date>`          | `MEAL#<meal_type>#<user_id>`      | All meal participation for a date                    |
| GSI1 | `DATE#<date>`          | `WFH#<user_id>`                   | All WFH employees on a date                          |
| GSI1 | `USER#<actor_user_id>` | `AUDIT#<entity_type>#<timestamp>` | All changes by a user; by user + entity type         |

> **GSI1 is overloaded** — different item types write different values into `GSI1_PK` / `GSI1_SK`, each occupying its own logical slice of the index with no cross-contamination. The GSI is sparse: only Meal Participation, Work Location, and Audit items project into it, keeping index size and cost minimal.
