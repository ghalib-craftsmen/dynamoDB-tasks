# DynamoDB Single-Table Schema

**Table name:** `AppTable`

---

### Users

## Access Patterns

1. Get user profile by user ID
2. Login lookup by email → resolve to user ID
3. Bot auth lookup by Discord ID → resolve to user ID

## DB Schema

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

### Team

## Access Patterns

1. Get team details
2. Get all members of a team
3. Check if a specific user is in a team

## DB Schema

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

### Meal Participation

## Access Patterns

1. Get user's all meals for a date
2. Get user's specific meal
3. Opt in/out of a meal
4. All participation for a date

## DB Schema

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

### Work Location

## Access Patterns

1. Get user's location for a date
2. Set user's location
3. All WFH employees on a date
4. Monthly WFH count for a user

## DB Schema

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
