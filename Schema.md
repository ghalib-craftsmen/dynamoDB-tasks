# DynamoDB Single-Table Schema

**Table name:** `AppTable`    
**GSIs used:** 1 (GSI1: `GSI1_PK` / `GSI1_SK`)

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

All three patterns are direct `GetItem` calls — no GSI needed. Lookup items (EMAIL, DISCORD) store only the target `userId`; the caller then fetches `USER#<id>` in a second read.

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

All three patterns resolve within a single partition — no GSI needed. Pattern 2 uses `Query PK = TEAM#<id>` with `begins_with(SK, "MEMBER#")`. Pattern 3 is a direct `GetItem`.

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

Patterns 1–3 resolve from the main table. Pattern 1 uses `begins_with(SK, "MEAL#<date>")`. Pattern 2 is a direct `GetItem`. Pattern 3 is a `PutItem`. Pattern 4 queries **GSI1** with `GSI1_PK = DATE#<date>` to fan out across all users for that date.

`<meal_type>` → `BREAKFAST`, `LUNCH`, `DINNER`

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

Patterns 1 and 2 are direct `GetItem` / `PutItem` on the main table. Pattern 4 uses `begins_with(SK, "WORKLOCATION#<year-month>")` on the main table — no GSI needed. Pattern 3 queries **GSI1** with `GSI1_PK = DATE#<date>` and `begins_with(GSI1_SK, "WFH#")` — the `WFH#` prefix separates these from meal rows in the same GSI partition.

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

No GSI needed. All five patterns resolve within a single partition. Pattern 1 queries `PK = DAY#<date>` and returns both sibling items in one round-trip. Patterns 2–5 are point reads/writes.

`METADATA` holds day type (e.g. `OFFICE`, `WFH`, `HOLIDAY`). `MEALS` holds the available meal options for that day.

---

## WFH Period

### Access Patterns

1. List all WFH periods
2. Is date in any WFH period?

### DB Schema

| Item       | PK          | SK                        |
| ---------- | ----------- | ------------------------- |
| WFH Period | `WFHPERIOD` | `<start_date>#<end_date>` |

No GSI needed. All periods share one partition key. Pattern 1 returns them sorted by start date (ISO-8601 sorts naturally). Pattern 2 queries up to today's date and filters `end_date >= target` in the application.

SK format: `YYYY-MM-DD#YYYY-MM-DD`

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
| Audit Entry | `AUDIT#<entity_type>#<entity_id>`  | `<timestamp>#<ulid>` | `USER#<actor_user_id>` | `AUDIT#<entity_type>#<timestamp>#<ulid>` |

Pattern 1 is a `PutItem`. Pattern 3 queries the main table with `PK = AUDIT#USER#<target_id>` — no GSI needed. Patterns 2 and 4 query **GSI1**: pattern 2 uses `GSI1_PK = USER#<actor_user_id>` with `begins_with(GSI1_SK, "AUDIT#")`; pattern 4 narrows further with `begins_with(GSI1_SK, "AUDIT#<entity_type>#")`.

`<ulid>` is appended to the SK to avoid collision when two mutations hit the same millisecond, while keeping sort order intact.

---

## GSI Summary

| GSI  | GSI1_PK                | GSI1_SK                           | Patterns served                                      |
| ---- | ---------------------- | --------------------------------- | ---------------------------------------------------- |
| GSI1 | `DATE#<date>`          | `MEAL#<meal_type>#<user_id>`      | All meal participation for a date                    |
| GSI1 | `DATE#<date>`          | `WFH#<user_id>`                   | All WFH employees on a date                          |
| GSI1 | `USER#<actor_user_id>` | `AUDIT#<entity_type>#<timestamp>#<ulid>` | All changes by a user; by user + entity type    |

**Projection:** `KEYS_ONLY` — only `PK`, `SK`, `GSI1_PK`, and `GSI1_SK` are stored in the index. Results that need additional attributes follow up with a `BatchGetItem` on the main table using the returned keys. This minimises index storage cost compared to `ALL`.

GSI1 is overloaded - Meal Participation, Work Location, and Audit each write different values into `GSI1_PK`/`GSI1_SK`. The index is sparse, so only those three item types appear in it.
