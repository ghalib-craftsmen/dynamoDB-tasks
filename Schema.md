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
