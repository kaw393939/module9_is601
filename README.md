# Module 9: Databases and Web Applications

## Start here

This is the SQL reading and practice guide for Module 9: using PostgreSQL through pgAdmin in Docker Compose. Work through the sections in order, then return to Canvas for your assignment.

**Canvas is the source of truth for required submissions, grading, and due dates.** This guide provides learning support; it does not add assignments or change requirements. The older guide's grading, reflection-length, quiz, and deadline instructions are intentionally not included here.

- [1. Start the database](#1-start-the-database)
- [2. Connect with pgAdmin](#2-connect-with-pgadmin)
- [3. Create related tables](#3-create-related-tables)
- [4. Insert and query data](#4-insert-and-query-data)
- [5. Update and delete safely](#5-update-and-delete-safely)
- [Troubleshooting](#troubleshooting)
- [Further reading](#further-reading)

## What you are learning

A relational database stores records in tables. A **primary key** identifies a row; a **foreign key** connects it to a row in another table. One user can have many calculations: this is a **one-to-many relationship**.

SQL lets you define tables and create, read, update, and delete data. pgAdmin is a graphical client for PostgreSQL, not a separate database. Docker Compose runs the services together.

## 1. Start the database

You need Git and Docker Desktop (or Docker Engine with the Compose plugin). Start Docker first. For these SQL exercises, you do not need a separate Python environment.

For a new local copy:

```bash
git clone https://github.com/kaw393939/module9_is601.git
cd module9_is601
docker compose up -d db pgadmin
docker compose ps
```

If you already have a course project, use its Compose file and preserve your work. Do not replace an existing project just to read this guide.

The repository's [docker-compose.yml](docker-compose.yml) defines:
- **db:** PostgreSQL, with a named volume for data.
- **pgadmin:** the database interface, exposed at localhost port 5050.
- **web:** the FastAPI application. It is not needed to practice SQL in pgAdmin. Start the full application with `docker compose up --build` when your Canvas task calls for it.

This is a **local teaching setup**, not a production deployment. The Compose file contains public demonstration credentials and exposes ports. Do not use real personal data or deploy it publicly as-is.

## 2. Connect with pgAdmin

There are **two different logins**:

1. Open [http://localhost:5050](http://localhost:5050).
2. Sign into the pgAdmin website using the values of **PGADMIN_DEFAULT_EMAIL** and **PGADMIN_DEFAULT_PASSWORD** under the `pgadmin` service in your Compose file.
3. In pgAdmin, right-click **Servers → Register → Server**.
4. On **General**, give the connection a name, such as `Module 9 local`.
5. On **Connection**, use:

| Field | Value for this repository |
| --- | --- |
| Host name/address | `db` |
| Port | `5432` |
| Maintenance database | `fastapi_db` |
| Username | The `POSTGRES_USER` value under `db` |
| Password | The `POSTGRES_PASSWORD` value under `db` |

Save, expand **Databases → fastapi_db**, and open **Query Tool**.

**Why not localhost?** pgAdmin runs in its own container. From there, `db` names the PostgreSQL service; `localhost` would mean the pgAdmin container itself. A PostgreSQL client installed directly on your computer instead connects through the published host port.

Existing volumes may retain previously initialized credentials. Editing environment variables alone does not necessarily reset an existing database or pgAdmin account. See [pgAdmin's container configuration](https://www.pgadmin.org/docs/pgadmin4/latest/container_deployment.html).

## 3. Create related tables

Run each example in Query Tool and inspect the result. We use a separate **module9_practice** schema so the examples do not overwrite application tables. These are practice examples, not a required change to your application's schema.

```sql
CREATE SCHEMA IF NOT EXISTS module9_practice;

CREATE TABLE IF NOT EXISTS module9_practice.users (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS module9_practice.calculations (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    operation VARCHAR(20) NOT NULL,
    operand_a NUMERIC NOT NULL,
    operand_b NUMERIC NOT NULL,
    result NUMERIC NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    user_id INTEGER NOT NULL
        REFERENCES module9_practice.users(id) ON DELETE CASCADE
);
```

- `NOT NULL` requires a value; `UNIQUE` prevents duplicate usernames/emails.
- Identity columns generate IDs. Do not assume IDs will always begin at 1.
- `NUMERIC` is exact decimal arithmetic; floating-point types are approximate.
- `ON DELETE CASCADE` means deleting a user also deletes that user's calculations. Use this deliberately.
- `IF NOT EXISTS` avoids a duplicate-table error; it does **not** repair or migrate an existing table with a different definition.

## 4. Insert and query data

Use fictional data:

```sql
INSERT INTO module9_practice.users (username, email)
VALUES ('alice', 'alice@example.com'), ('bob', 'bob@example.com')
ON CONFLICT (username) DO NOTHING;

-- Look up Alice's generated ID rather than assuming it is 1.
INSERT INTO module9_practice.calculations
    (operation, operand_a, operand_b, result, user_id)
SELECT 'add', 2, 3, 5, id
FROM module9_practice.users
WHERE username = 'alice'
RETURNING *;
```

The second statement creates another calculation each time you run it. That is expected; do not confuse rerunning an insert with updating a record.

```sql
SELECT * FROM module9_practice.users ORDER BY id;
SELECT * FROM module9_practice.calculations ORDER BY id;

SELECT u.username, c.id, c.operation, c.operand_a, c.operand_b, c.result
FROM module9_practice.calculations AS c
JOIN module9_practice.users AS u ON c.user_id = u.id
ORDER BY c.id;
```

The join connects each calculation with its user's name. An inner join returns matching rows; Bob has no calculation yet, so he will not appear in that result.

## 5. Update and delete safely

First inspect the rows you intend to change with `SELECT`. A missing `WHERE` can update or delete every row. The examples below use transactions and **roll back**, so you can observe changes without keeping them. Run each block as a unit.

```sql
BEGIN;

UPDATE module9_practice.calculations
SET operand_b = 4, result = operand_a + 4
WHERE operation = 'add'
  AND user_id = (
      SELECT id FROM module9_practice.users WHERE username = 'alice'
  )
RETURNING *;

ROLLBACK;
```

This keeps the arithmetic consistent rather than changing the result of 2 + 3 to an incorrect value. `RETURNING` displays affected rows. `ROLLBACK` undoes this transaction; a subsequent `SELECT` shows the original data.

```sql
BEGIN;

DELETE FROM module9_practice.calculations
WHERE user_id = (
    SELECT id FROM module9_practice.users WHERE username = 'alice'
)
RETURNING *;

ROLLBACK;
```

To deliberately persist a change in your own disposable practice data, use `COMMIT` instead of `ROLLBACK`. Do not experiment with destructive statements against real or shared data.

### Check your understanding

- Why is the user ID stored in calculations instead of copying the user's email into every row?
- What would happen if you removed `WHERE` from the update?
- Why can Bob exist without appearing in the inner join?
- What is the difference between pgAdmin's web login and PostgreSQL's login?

These are optional self-checks, not additional graded work.

## Troubleshooting

- **Page unavailable:** check `docker compose ps` and `docker compose logs pgadmin`; confirm port 5050 is free.
- **Database connection fails:** check `docker compose logs db`; verify host `db`, port, database, and PostgreSQL credentials. Wait for PostgreSQL to become healthy.
- **Table not found:** select `fastapi_db` in Query Tool and use the fully qualified `module9_practice.table_name` names above.
- **Transaction aborted:** run `ROLLBACK;`, correct the first error, then retry.
- **Image/version or volume errors:** the current Compose file uses unpinned image tags. PostgreSQL major versions can differ in storage layout and upgrade requirements. Preserve your data, inspect the exact error, and ask for help rather than deleting volumes or guessing a version change.
- **Stop services:** `docker compose down` stops/removes containers and networks but preserves named volumes by default. **Do not add `-v`** unless you intentionally want to delete the stored data. See [Docker's explanation](https://docs.docker.com/reference/cli/docker/compose/down/).

## Further reading

- [PostgreSQL SQL tutorial](https://www.postgresql.org/docs/current/tutorial-sql.html): tables, inserts, queries, joins, updates, and deletions.
- [PostgreSQL foreign keys](https://www.postgresql.org/docs/current/tutorial-fk.html): relationships and referential integrity.
- [PostgreSQL transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html): committing and rolling back.
- [Docker Compose](https://docs.docker.com/compose/): running multiple services.
- [pgAdmin documentation](https://www.pgadmin.org/docs/pgadmin4/latest/): connecting and using Query Tool.

**Next:** return to the Module 9 assignment in Canvas and follow its current instructions. If anything in a video or older repository revision conflicts with Canvas requirements, ask for clarification rather than doing extra work.
