# Rust Todo

A full-stack todo application written entirely in Rust. Users register, log in and manage their
own todo lists from a native desktop client backed by a JSON API with a PostgreSQL database.

This repository ties the two parts together as Git submodules:

| Directory   | Repository                                                                   | What it is                                  |
| ----------- | ---------------------------------------------------------------------------- | ------------------------------------------- |
| `backend/`  | [rust-todo-backend](https://github.com/BanyaiBence/rust-todo-backend)        | JSON API: axum, sqlx, PostgreSQL, JWT       |
| `frontend/` | [rust-todo-frontend](https://github.com/BanyaiBence/rust-todo-frontend)                | Desktop client: Dioxus 0.7, reqwest         |

Each submodule has its own README with full details. This file gives the overview.

## Architecture

```
┌──────────────────────┐   HTTP + JSON    ┌──────────────────────┐   SQL    ┌──────────────┐
│  frontend (desktop)  │ ───────────────▶ │  backend (axum API)  │ ───────▶ │  PostgreSQL  │
│  Dioxus + reqwest    │  Bearer JWT auth │  :3000               │  sqlx    │  17          │
└──────────────────────┘                  └──────────────────────┘          └──────────────┘
                                          └────────── Docker Compose ───────────────┘
```

- **Backend:** an axum/tokio service. Passwords are hashed with argon2, sessions are stateless
  JWT bearer tokens (1 hour by default), and every todo belongs to exactly one user. SQL queries
  are checked at compile time by sqlx and migrations run automatically on startup. The API and
  its database run as two containers via Docker Compose.
- **Frontend:** a native desktop window (WebView-based, via Dioxus) with a login/register
  screen and a todo list that supports creating, editing, completing, deleting and filtering
  todos. It holds the token in memory and returns to the login screen when it expires.

### API at a glance

| Method   | Path             | Auth | Purpose                                     |
| -------- | ---------------- | ---- | ------------------------------------------- |
| `GET`    | `/health`        |      | Liveness check                              |
| `POST`   | `/auth/register` |      | Create a user                               |
| `POST`   | `/auth/login`    |      | Exchange credentials for a bearer token     |
| `GET`    | `/todos`         | ✓    | List own todos (`limit` 1–100, `offset`)    |
| `POST`   | `/todos`         | ✓    | Create a todo                               |
| `GET`    | `/todos/{id}`    | ✓    | Get one todo                                |
| `PATCH`  | `/todos/{id}`    | ✓    | Update `title`, `description`, `completed`  |
| `DELETE` | `/todos/{id}`    | ✓    | Delete a todo                               |

Errors always have the shape `{"error": "<message>"}`. See the backend README for request and
response examples and the full list of status codes.

## Getting started

### Prerequisites

- Git
- Docker with Compose, to run the backend
- Rust 1.95 or newer (the backend's CI toolchain; the frontend needs 1.85+)
- On Linux, the WebKitGTK libraries for the desktop client. On Fedora:

  ```sh
  sudo dnf install webkit2gtk4.1-devel libxdo-devel openssl-devel
  ```

### Clone

```sh
git clone --recurse-submodules git@github.com:BanyaiBence/rust-todo-app.git
# or, in an existing clone:
git submodule update --init --recursive
```

### Run the backend

```sh
cd backend
cp .env.example .env              # set JWT_SECRET to a long random value
docker compose up -d --build --wait
curl localhost:3000/health        # {"status":"ok"}
```

The first build compiles all dependencies and takes several minutes. Stop the stack with
`docker compose down` (add `-v` to also delete the database).

### Run the frontend

```sh
cd frontend
make run                          # or: cargo run
```

The client connects to `http://localhost:3000` by default. To use another backend:

```sh
make run TODO_API_URL=http://todo.example.com
```

Register a user in the app, log in, and start adding todos.

## Configuration

| Component | Variable               | Default                 | Description                                   |
| --------- | ---------------------- | ----------------------- | --------------------------------------------- |
| backend   | `JWT_SECRET`           | *required*              | HMAC secret used to sign tokens               |
| backend   | `JWT_TTL_SECONDS`      | `3600`                  | Token lifetime                                |
| backend   | `APP_PORT`             | `3000`                  | Host port the API is published on (Compose)   |
| backend   | `POSTGRES_PASSWORD`    | `todo`                  | Database password (Compose)                   |
| backend   | `CORS_ALLOWED_ORIGINS` | *(none)*                | Origins allowed to call the API from browsers |
| frontend  | `TODO_API_URL`         | `http://localhost:3000` | Base URL of the backend                       |

The backend README lists the remaining settings (`DATABASE_URL`, `BIND_ADDR`, logging).

## Testing

**Backend:** unit, repository, API and end-to-end suites.

```sh
cd backend
docker compose --profile test up -d --wait db-test   # throwaway test database
cargo test                                           # unit + repo + api
cargo test --test e2e -- --ignored                   # e2e against the running Compose stack
```

**Frontend:** serialization tests and live tests against a running backend.

```sh
cd frontend
make test        # no backend needed
make test-live   # against TODO_API_URL
make check       # fmt check, clippy and tests
```

## Documentation

- [`backend/README.md`](backend/README.md): API reference, configuration and development setup
- [`backend/docs/`](backend/docs/README.md): architecture diagrams (PlantUML) and a PDF write-up
- [`backend/PLAN.md`](backend/PLAN.md): design decisions and milestones
- [`frontend/README.md`](frontend/README.md): client setup, Make targets and layout

## Working with submodules

```sh
git submodule update --remote --merge   # pull the latest commit of each submodule
git add backend frontend                # record the new submodule commits here
git commit -m "Bump submodules"
```

Changes to the backend or frontend themselves are committed and pushed in their own
repositories first; this repository only records which commit of each it uses.
