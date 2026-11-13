# course-catalog-api

Backend of the course catalog: FastAPI, SQLAlchemy 2.1, PostgreSQL, and Alembic migrations (Labs 4–7 of "Advanced Web Technologies").

## How to run

You need Python 3.11 or newer and PostgreSQL 16 or newer.

Once, as the `postgres` superuser, create the role and the database:

```sql
CREATE ROLE catalog_app LOGIN PASSWORD 'change-me';
CREATE DATABASE course_catalog OWNER catalog_app;
```

Then, from the project folder:

```bash
# 1. Create and activate virtual environment
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\Activate.ps1

# 2. Install dependencies
pip install -r requirements.txt

# 3. Copy .env from the example
cp .env.example .env

# 4. Apply migrations (creates all tables)
alembic upgrade head

# 5. Load initial data
psql -h localhost -U catalog_app -d course_catalog -v ON_ERROR_STOP=1 -f seed.sql

# 6. Start the server
fastapi dev main.py
```

API documentation: http://localhost:8000/docs

The password `change-me` is for teaching; in a real project, passwords are never put into the repository.

---

## What changed in the API

- **Data is now stored in PostgreSQL.** Previously courses lived in `data.py` (a Python list) and enrollments in a list in `main.py`; all data was lost on every server restart. Now everything is in the database and survives restarts.
- **`POST /enrollments` now requires `studentId` (integer) instead of `studentName`.** A student must first be created via `POST /students`, which returns their numeric id.
- **New routes:** `POST /students` creates a student; `GET /enrollments` now returns the student name and course title via a JOIN; `GET /stats/courses` returns the enrollment count per course (including zeros).
- **The `level` field** was added to courses (`beginner` / `intermediate` / `advanced`). `GET /courses` accepts an optional `?level=` filter.
- **Constraint violations return proper HTTP responses:** duplicate email → 409, already enrolled → 409, unknown course → 404, unknown student → 404, identified by constraint name from `pg_constraint`.

---

## Migrations

| Command | What it does |
|---------|--------------|
| `alembic upgrade head` | applies all the missing migrations |
| `alembic downgrade -1` | rolls back the last migration |
| `alembic current` | shows the version of the database |
| `alembic history` | shows the chain of migrations |
| `alembic revision --autogenerate -m "text"` | writes a draft from models vs. database |
| `alembic check` | reports whether the models and the database have diverged |
| `alembic upgrade 0001:0002 --sql` | prints the SQL of a migration without running it |
| `alembic stamp head` | marks the database as up to date without running migrations |

---

## Experiment log

| # | Experiment | What I saw | Cause | How I fixed it |
|---|------------|------------|-------|----------------|
| 1 | the `uq` template in `naming_convention` changed to `%(table_name)s_%(column_0_name)s_unique` | `check_models.py` showed `students_email_unique` in models but `students_email_key` in database | The template produces different names from what PostgreSQL created; Alembic would suggest recreating constraints and the error handler would fail | Restored the template to `%(table_name)s_%(column_0_N_name)s_key` |
| 2 | `min_length` removed from `StudentIn`, sent name `N` | `500 Internal Server Error`; server log showed `IntegrityError: students_name_check` | Without Pydantic check the request reached the DB; `students_name_check` is not in `CONSTRAINT_ERRORS` so it re-raised as 500 | Restored `Field(min_length=2)` to `StudentIn`; Pydantic returns 422 before the DB is touched |
| 3 | Typo in constraint name: `students_email_key` → `students_email_uniq` in `CONSTRAINT_ERRORS` | `500` on duplicate email; log showed real name `students_email_key` did not match the dict key | The key lookup failed so `raise_http_error` re-raised the exception as 500 | Restored the correct key `students_email_key` |
| 4 | `ALTER TABLE courses ALTER COLUMN title DROP NOT NULL` then `alembic check` | `FAILED: New upgrade operations detected: modify_nullable on courses.title` | The model has `title: Mapped[str]` (NOT NULL) but the DB column allows NULL — mismatch detected | `ALTER TABLE courses ALTER COLUMN title SET NOT NULL`; `alembic check` returned clean |
| 5 | Ran auto-generated migration 0002 (`op.add_column level NOT NULL`) on existing data | `IntegrityError: NotNullViolation — column "level" contains null values` | Cannot add a NOT NULL column without a default to a table that already has rows | Rewrote upgrade() as three steps: add nullable → UPDATE → SET NOT NULL |
| 6 | Commented out `import models` in `alembic/env.py`, ran `--autogenerate` | Migration proposed to DROP all three tables | `Base.metadata` was empty without the import; Alembic saw tables in DB but nothing in metadata | Restored `import models  # noqa: F401` |
| 7 | Added `phone: Mapped[str \| None]` to `Student`, ran `alembic check` | `FAILED: Detected added column 'students.phone'` | Model has the column but no migration was created for it | Removed the `phone` field; `alembic check` returned clean |
| 8 | Created two migrations from the same base `0002` using `--splice`, ran `alembic upgrade head` | `FAILED: Multiple head revisions are present` | Two branches in migration history without a merge point | Ran `alembic merge heads -m "merge"`, then deleted the three experimental files |

---

## Decisions

- **The API response is chosen by the constraint name rather than by SQLSTATE.** SQLSTATE `23505` fires for *any* UNIQUE violation — email, enrollment pair, course id — but each needs a different response (409 with different messages, or 404 for FK violations). The constraint name uniquely identifies which rule fired. Catching after `commit()` rather than checking with a prior `SELECT` avoids a time-of-check/time-of-use race: between the SELECT and the INSERT another request could take the email, making the check stale. Attempting the insert and handling the refusal is atomic and reliable.

- **A like is incremented with `UPDATE courses SET likes = likes + 1` rather than by reading the number into the program.** If two requests read the same value (e.g. 11) and both write back 12, one like is lost. The `UPDATE … SET likes = likes + 1` expression is evaluated by the database inside a row lock, so the second request waits for the first to commit, then reads the already-incremented value and adds one more — both likes are saved.

- **Migration 0002 is split into three steps.** PostgreSQL cannot add a `NOT NULL` column without a default value to a table that already has rows, because there would be no value to put in the existing rows. The three-step approach (add nullable → backfill → set NOT NULL) works on a live table with data and runs entirely within one transaction, so a failure at any step leaves the database unchanged at version 0001.

---

## Migration 0002 self-check

| What to check | Result |
|---------------|--------|
| Goal and steps | ✅ Clear goal: add `level` column. Three explicit steps: add nullable, UPDATE existing rows, set NOT NULL. Each step is minimal. |
| Data | ✅ Existing rows are filled by the rule: elective → `beginner`, otherwise → `intermediate`. No data is lost. |
| Renames and drops | ✅ No drops or renames. Only `add_column` + `execute` + `alter_column`. Downgrade drops only the new column. |
| Rollback | ✅ `downgrade()` drops the `level` column. Acceptable on training database; on production, dropping a column loses user-chosen levels — must be communicated. |
| Verification | ✅ `alembic upgrade 0001:0002 --sql` reviewed; `downgrade -1 → upgrade head` cycle tested and passes; `alembic check` is clean after upgrade. |
| History | ✅ Single head (`alembic heads` shows only `0002`). Migration committed together with model and schema changes. |
| Application | ✅ `CourseOut` and `seed.sql` updated to include `level`. `GET /courses` has `?level=` filter. `try_errors.py` still passes 5 of 5. |
