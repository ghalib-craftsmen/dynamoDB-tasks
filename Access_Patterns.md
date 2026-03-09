## Access Patterns

### User

| #   | Pattern                | Key Condition                                                       |
| --- | ---------------------- | ------------------------------------------------------------------- |
| 1   | Get user profile       | `PK = USER#<id>` + `SK = PROFILE`                                   |
| 2   | Login by email         | `PK = EMAIL#<email>` + `SK = LOOKUP` → then fetch `USER#<id>`       |
| 3   | Bot auth by Discord ID | `PK = DISCORD#<discordId>` + `SK = LOOKUP` → then fetch `USER#<id>` |

### Team

| #   | Pattern                  | Key Condition                               |
| --- | ------------------------ | ------------------------------------------- |
| 4   | Get team details         | `PK = TEAM#<id>` + `SK = METADATA`          |
| 5   | Get all team members     | `PK = TEAM#<id>` + `SK begins_with MEMBER#` |
| 6   | Check if user is in team | `PK = TEAM#<id>` + `SK = MEMBER#<user_id>`  |

### Meal Participation

| #   | Pattern                         | Key Condition                                         |
| --- | ------------------------------- | ----------------------------------------------------- |
| 7   | Get user's all meals for a date | `PK = USER#<id>` + `SK begins_with MEAL#<date>`       |
| 8   | Get user's specific meal        | `PK = USER#<id>` + `SK = MEAL#<date>#<meal_type>`     |
| 9   | Opt in/out of a meal            | `PUT PK = USER#<id>` + `SK = MEAL#<date>#<meal_type>` |
| 10  | All participation for a date    | `GSI1_PK = <date>`                                    |

### Work Location

| #   | Pattern                        | Key Condition                                                 |
| --- | ------------------------------ | ------------------------------------------------------------- |
| 11  | Get user's location for a date | `PK = USER#<id>` + `SK = WORKLOCATION#<date>`                 |
| 12  | Set user's location            | `PUT PK = USER#<id>` + `SK = WORKLOCATION#<date>`             |
| 13  | All WFH employees on a date    | `GSI1_PK = <date>` + `SK begins_with WFH#`                    |
| 14  | Monthly WFH count for a user   | `PK = USER#<id>` + `SK begins_with WORKLOCATION#<year-month>` |

### Day & Meals

| #   | Pattern                  | Key Condition                                          |
| --- | ------------------------ | ------------------------------------------------------ |
| 15  | Get full day context     | `PK = DAY#<date>` (returns both METADATA + MEALS rows) |
| 16  | Get day type only        | `PK = DAY#<date>` + `SK = METADATA`                    |
| 17  | Get available meals only | `PK = DAY#<date>` + `SK = MEALS`                       |
| 18  | Set day type             | `PUT PK = DAY#<date>` + `SK = METADATA`                |
| 19  | Set available meals      | `PUT PK = DAY#<date>` + `SK = MEALS`                   |

### WFH Period

| #   | Pattern                    | Key Condition                                                                       |
| --- | -------------------------- | ----------------------------------------------------------------------------------- |
| 20  | List all WFH periods       | `PK = WFHPERIOD` — sorted by start_date naturally                                   |
| 21  | Is date in any WFH period? | `PK = WFHPERIOD` + `SK begins_with` up to today → filter `end_date >= today` in app |

### Audit Log

| #   | Pattern                                    | Key Condition                                                                         |
| --- | ------------------------------------------ | ------------------------------------------------------------------------------------- |
| 22  | Write an audit entry on every mutation     | `PUT PK = AUDIT#<entity_type>#<entity_id>` + `SK = <timestamp>#<user_id>`            |
| 23  | Get all changes made by a specific user    | `GSI1_PK = USER#<user_id>` + `SK begins_with AUDIT#`                                 |
| 24  | Get all changes made to a specific user's records | `PK = AUDIT#USER#<target_user_id>` + `SK begins_with <timestamp>`             |
| 25  | Get changes of a specific entity type by a user | `GSI1_PK = USER#<user_id>` + `SK begins_with AUDIT#<entity_type>#`            |