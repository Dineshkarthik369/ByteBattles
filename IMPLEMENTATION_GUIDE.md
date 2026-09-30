# ByteBattles: step-by-step implementation and demonstration

This is a modified copy of https://github.com/TatHack-Tathva/ByteBattles.
All code is included; you do not need to paste snippets into the original repo.
Use a fresh local development instance for the first demonstration.

## What is implemented

- Pagination fix: `(page - 1) * limit`.
- Refresh-token fix: configured days rather than access-token minutes.
- Restored admin-only `POST /problems/tag`, with duplicate-tag HTTP 409.
- Problem search by difficulty, tag, and literal case-insensitive title.
- Pagination response: `items`, `page`, `page_size`, `total`, `has_more`.
- Judge counters: completed submission and accepted submission counts, updated
  atomically with the verdict. A PostgreSQL submission row lock and PENDING guard
  prevent retry/double-delivery counting. Counters describe final judged submissions;
  queued/infrastructure-failed submissions are excluded. Intentional rejudging is
  not implemented: a final verdict is retained rather than counted again.
- Redis atomic fixed-window rate limit: default 10 attempts per user per 60 seconds,
  starting with that user's first valid submission attempt. HTTP 429 includes
  Retry-After; Redis failure returns 503. Adjacent windows can allow a burst.
- Operator-run first-admin bootstrap with PostgreSQL advisory lock.
- Admin-only `POST /users/{username}/promote`.

Problem updates/testcase replacement and a fourth language are not included.
Original C/C++/Python judging and sandbox settings are preserved.
The problem list response is intentionally changed from an array to an envelope;
clients must read `response.items`.

## 1. Prepare your machine

On Windows, use Docker Desktop with the WSL2 backend and run shell commands in an
Ubuntu WSL terminal. On Linux, install Docker Engine and the Compose plugin.
Install Python 3.12+ and uv if you want to run local tests.
Check:

```bash
docker version
docker compose version
```

Extract the ZIP and enter its `bytebattles` directory. All commands below run there
unless another directory is specified. Open it in VS Code if desired.

## 2. Configure the environment

```bash
cp .env.example .env
```

Edit `.env`. Set SECRET_KEY to a random value of at least 32 bytes; generate one:

```bash
python3 -c 'import secrets; print(secrets.token_hex(32))'
```

Use matching, non-placeholder database and MinIO credentials. Keep container
addresses `postgres`, `redis`, and `http://minio:9000` for Compose. Optional:

```dotenv
SUBMISSION_RATE_LIMIT=10
SUBMISSION_RATE_WINDOW=60
```

Do not commit `.env` or expose secrets in screenshots.

## 3. Build sandboxes and start services

```bash
docker volume create bytebattles_postgres
docker volume create bytebattles_minio
(cd judge/images && bash build_command.sh)
docker compose up --build -d
docker compose ps
```

Wait for services to become healthy. Open http://localhost:8000/docs.
If startup fails, inspect `docker compose logs api judge postgres minio`.

The judge Docker build installs gcc to compile netifaces. Local judge installation
also needs a compiler and Python development headers. The Docker daemon must be
reachable by the judge; this is required by the supplied sandbox architecture.

## 4. Create object-storage buckets

```bash
docker compose exec api .venv/bin/python -m scripts.create_buckets
```

This uses your configured credentials and creates the testcase/submission buckets
if absent. MinIO is accessible from the API container; host port publication is
not needed for this command.

## 5. Register and bootstrap your first admin

In Swagger, expand `POST /auth/register`, click Try it out, and submit:

```json
{"username":"pavan_admin","email":"pavan@example.com","password":"DemoPass123!","conf_password":"DemoPass123!"}
```

Use your own password. After successful registration:

```bash
docker compose exec api .venv/bin/python -m scripts.bootstrap_admin pavan_admin
```

This command works only when no admin exists. It selects an already registered
account and promotes it; concurrent bootstrap commands serialize through an
advisory lock. There is no public first-admin HTTP endpoint.

In Swagger click Authorize, enter this username and password, and authorize the
OAuth2 login. Or use `POST /auth/login` to obtain access/refresh tokens.
Access tokens last ACCESS_TOKEN_EXPIRE_MINUTES; refresh tokens last
REFRESH_TOKEN_EXPIRE_DAYS. A refresh token cannot be used as an access token.

## 6. Create a tag and sample problem

Run `POST /problems/tag` as the admin:

```json
{"name":"Mathematics","slug":"math"}
```

Generate paired testcase files:

```bash
python3 scripts/make_demo_tests.py
```

The generated `sum.zip` contains `sum/inputs/01.txt` and
`sum/outputs/01.txt`, with matching filenames for all three cases.

In Swagger use `POST /problems/` with these form fields:

| Field | Value |
|---|---|
| id | SUM001 |
| title | Add Two Integers |
| description | Read two integers and print their sum. |
| difficulty | EASY |
| constraints | ["-1000000 <= a,b <= 1000000"] |
| tags | ["math"] |
| sample_io | {"5 7":"12"} |
| input_desc | Two space-separated integers. |
| output_desc | Their sum. |
| memory_limit_mb | 128 |
| time_limit_sec | 2 |
| visibility | true |
| tests_zip | Select sum.zip |

Leave optional fields empty. Expect HTTP 201 and three testcases.

## 7. Check search and pagination

Open:

```text
http://localhost:8000/problems/?page=1&limit=1&difficulty=EASY&tag=math&title=Add
```

Expect one item, total 1, and has_more false. Page 2 should contain no items.
Create more problems with different IDs to demonstrate multiple pages. Unauthenticated
users see visible problems only; admins can see hidden ones as well.

## 8. Submit and check the verdict

Use `POST /submissions/` while authenticated:

```json
{"problem_id":"SUM001","language":"PY","code":"a, b = map(int, input().split())\nprint(a + b)\n"}
```

Save the returned submission ID. Poll `GET /submissions/{submission_id}` until
verdict changes from PD (pending) to AC (accepted), usually within seconds.
Repeat using `print(a - b)` to demonstrate WA (wrong answer).

The problem list should now show total_submissions 2 and accepted_submissions 1.
These counts appear after judging finishes, not immediately after submission.
If a submission stays PD, inspect `docker compose logs judge` and `logs/`.
Check buckets exist, images were built, and the judge can reach Docker.

## 9. Demonstrate rate limiting and admin authorization

Within one 60-second window, submit enough requests to exceed the configured limit.
The excess request returns HTTP 429 and Retry-After. Wait that duration to retry.
Each request that passes the visible-problem check consumes an attempt, including
requests that later fail during upload/queueing.

Register a second account. As admin, run `POST /users/SECOND_USERNAME/promote`.
As a regular user, calling this endpoint or creating tags should return HTTP 403.
Promote only accounts you intend to grant full administration rights.

## 10. Run regression checks

For local tests, no database server or Docker daemon is needed:

```bash
uv sync --group api
uv pip install --python .venv/bin/python docker
.venv/bin/python -m unittest discover -s tests -v
```

Tests cover pagination/filtering, token lifetime/type, HTTP 429/Retry-After, and
counter retry behavior. They use SQLite and a mocked Redis response; PostgreSQL
locking, actual Redis script execution, and Docker judging require the live demo.

## 11. Understand the implementation files

| File | Purpose |
|---|---|
| api/app/utils/oauth2.py | Refresh-token lifetime fix |
| api/app/routes/problems.py | Restored tag route, query filters, pagination |
| api/app/utils/rate_limit.py | Atomic Redis increment/expiry script |
| api/app/routes/submissions.py | Rate check before storage/queue work |
| judge/judge_worker/pipeline.py | Transactional counters and final-verdict guard |
| api/app/routes/users.py | Admin-only promotion |
| scripts/bootstrap_admin.py | Operator-controlled first admin |
| scripts/create_buckets.py | MinIO bucket setup |
| scripts/make_demo_tests.py | Sample testcase ZIP |
| tests/test_regression.py | Regression tests |

## Validation and limitations

Four regression tests passed in the implementation environment. Docker is not
available there, so full Compose startup and sandbox execution were not run.
Follow steps 3–9 on your Docker-enabled machine for end-to-end acceptance.
Existing queue delivery is unchanged: a failure between the database commit and
Redis enqueue can leave a pending submission. An outbox/recovery mechanism would
be a separate reliability enhancement. Existing counters are not backfilled.
The supplied platform still needs a security/deployment review before public use.
Do not execute submissions outside its isolated judge containers.

To stop without deleting stored data:

```bash
docker compose down
```
