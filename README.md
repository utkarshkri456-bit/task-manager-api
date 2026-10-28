# Task Management API with MongoDB

A RESTful task-management backend built with **Node.js, Express, MongoDB Atlas and Mongoose**. Users register and log in (JWT + bcrypt), then create and manage their own private tasks with validation, pagination, filtering, searching and sorting.

---

## 1. Project Overview

**Goals**
- Integrate an Express app with a cloud NoSQL database (MongoDB Atlas) through the Mongoose ODM.
- Model data with schemas, validation and a one-to-many relationship (User → Tasks).
- Secure the API with password hashing and token-based authentication.
- Provide a production-style task API: full CRUD, pagination, filtering, sorting, search and statistics.

**Features**
| Area | What is implemented |
|---|---|
| Auth | Register, login, JWT (Bearer), `bcrypt` hashing (cost 12), protected routes |
| Tasks | Create, read one/many, update, toggle complete, delete, stats aggregation |
| Data integrity | Schema validation, enums, defaults, indexes, pre-save hooks |
| Relationships | `Task.user → User` (ObjectId ref), virtual `User.tasks`, cascade delete |
| Querying | Pagination, filter (completed / priority / category / due-date range), title search, whitelisted sorting |
| Safety | Ownership checks on every query, field whitelisting on update, `helmet`, body-size limit, regex escaping, centralized error handler |

**Tech stack:** Node.js · Express 4 · MongoDB Atlas · Mongoose 8 · jsonwebtoken · bcryptjs · Jest + Supertest + mongodb-memory-server

---

## 2. Setup Instructions

### 2.1 Prerequisites
- Node.js 18+ and npm
- A free MongoDB Atlas account
- (Optional) Postman / Thunder Client and MongoDB Compass

### 2.2 MongoDB Atlas (Day 1)
1. Sign up at <https://www.mongodb.com/cloud/atlas> and create a **free M0 cluster**.
2. **Database Access** → *Add New Database User* (username + password, role *Read and write to any database*).
3. **Network Access** → *Add IP Address* → your current IP (or `0.0.0.0/0` for development / deployment only).
4. **Database** → *Connect* → *Drivers* → copy the connection string.
5. Add the database name `taskmanager` before the `?`:
   `mongodb+srv://USER:PASSWORD@cluster0.xxxxx.mongodb.net/taskmanager?retryWrites=true&w=majority`
   (URL-encode special characters in the password.)

> Free-tier (M0) limits to know: 512 MB storage, shared RAM/CPU, max ~500 connections, no dedicated backups.

### 2.3 Install and run (Day 2)
```bash
git clone <your-repo-url>
cd task-manager-api
npm install
cp .env.example .env     # then edit .env with your values
npm run dev              # development (nodemon)
# or
npm start                # production
```

`.env`
```
PORT=5000
NODE_ENV=development
MONGODB_URI=<your Atlas connection string>
JWT_SECRET=<long random string>      # node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
JWT_EXPIRES_IN=7d
```

Health check: `GET http://localhost:5000/api/health`

### 2.4 Run tests
```bash
npm test
```
Tests use `mongodb-memory-server` (an in-process MongoDB), so they never touch your Atlas data. The first run downloads a MongoDB binary (~100 MB).

---

## 3. Code Structure

```
task-manager-api/
├── server.js                     # Entry point: load env, connect DB, start HTTP server
├── package.json
├── .env.example                  # Template for environment variables
├── .gitignore
├── README.md
├── src/
│   ├── app.js                    # Express app (middleware + routes) – separated for testing
│   ├── config/
│   │   └── database.js           # Mongoose connection to Atlas
│   ├── models/
│   │   ├── User.js               # User schema, password hashing, virtual tasks, cascade delete
│   │   └── Task.js               # Task schema, validation, indexes, pre-save hook
│   ├── controllers/
│   │   ├── userController.js     # register, login, getMe, deleteMe
│   │   └── taskController.js     # CRUD + pagination/filter/sort + stats
│   ├── middleware/
│   │   ├── auth.js               # JWT verification → req.userId
│   │   └── errorHandler.js       # 404 + central error → JSON mapping
│   ├── routes/
│   │   ├── userRoutes.js
│   │   └── taskRoutes.js
│   └── utils/
│       └── asyncHandler.js       # Forwards async errors to Express
└── tests/
    └── api.test.js               # Auth, CRUD, query features, isolation, cascade
```

**Request flow:** `Client → app.js → route → protect (JWT) → controller → Mongoose model → MongoDB Atlas → JSON response`. Any thrown error is caught by `asyncHandler` and converted to a consistent `{ success:false, error }` response by `errorHandler`.

---

## 4. Database Schema (ER Diagram)

```mermaid
erDiagram
    USER ||--o{ TASK : "owns"
    USER {
        ObjectId _id PK
        String name "2-50 chars, required"
        String email "unique, lowercase, regex-validated"
        String password "bcrypt hash, select:false, min 6"
        Date createdAt
        Date updatedAt
    }
    TASK {
        ObjectId _id PK
        String title "3-200 chars, required"
        String description "max 1000"
        Boolean completed "default false"
        String priority "low | medium | high (default medium)"
        Date dueDate
        String category "work | personal | shopping | health | other"
        ObjectId user FK "ref User, required"
        Date createdAt
        Date updatedAt
    }
```

**Relationship:** one-to-many. Each task stores its owner's `_id` (`Task.user`). `User.tasks` is a *virtual populate*, so it can be loaded with `.populate('tasks')` without duplicating data. Deleting a user also deletes their tasks (`pre('deleteOne')` hook).

**Indexes**
| Index | Purpose |
|---|---|
| `users.email` (unique) | Fast login lookup, prevents duplicate accounts |
| `tasks { user: 1, completed: 1 }` | The most common query: "my pending/completed tasks" |
| `tasks { dueDate: 1 }` | Due-date range filters and sorting |

---

## 5. Technical Details

### Authentication
1. **Register** – Mongoose `pre('save')` hashes the password with `bcryptjs` (12 salt rounds) only when it changed. The hash field has `select:false` so it never leaks in responses.
2. **Login** – looks up the user with `.select('+password')`, compares with `bcrypt.compare`, and returns the same generic error for unknown email and wrong password (prevents user enumeration).
3. **JWT** – signed with `JWT_SECRET`, payload `{ id }`, expires per `JWT_EXPIRES_IN`. The `protect` middleware verifies the Bearer token, re-loads the user (so deleted users are rejected) and sets `req.userId`.

### Pagination / filtering / sorting algorithm
```
page  = max(int(page)  || 1, 1)
limit = clamp(int(limit) || 10, 1, 100)
filter = { user: req.userId, ...optional completed/priority/category/search/dueDate range }
tasks = Task.find(filter).sort({ [sortBy]: order, _id: 1 }).skip((page-1)*limit).limit(limit)
total = Task.countDocuments(filter)             // run in parallel with find
pages = ceil(total / limit)
```
- `sortBy` is **whitelisted** (`createdAt, dueDate, priority, title, completed`) to avoid arbitrary field injection.
- `_id` is added as a tiebreaker so pages are stable.
- `search` escapes regex metacharacters before building a case-insensitive `$regex`.
- Using `{ user: req.userId }` in **every** query guarantees users can only access their own data (a foreign id returns `404`, not `403`, so ids are not disclosed).

### Other design decisions
- **Update whitelist**: `PUT` copies only `title, description, completed, priority, dueDate, category`, so a client can never reassign `user`.
- **`save()` instead of `findByIdAndUpdate`** for updates, so validators and the title-capitalizing hook run.
- **Error mapping**: `ValidationError → 400`, `CastError → 400`, duplicate key `11000 → 409`, JWT errors `→ 401`, unknown routes `→ 404`.
- **Aggregation**: `/api/tasks/stats` uses `$match` + `$group` to count tasks by priority and category.

---

## 6. API Reference

Base URL: `http://localhost:5000/api` · Protected routes need header `Authorization: Bearer <token>`.

### Users
| Method | Endpoint | Auth | Body | Description |
|---|---|---|---|---|
| POST | `/users/register` | – | `name, email, password` | Create account, returns token |
| POST | `/users/login` | – | `email, password` | Log in, returns token |
| GET | `/users/me` | ✔ | – | Profile + task counts |
| DELETE | `/users/me` | ✔ | – | Delete account and all its tasks |

### Tasks
| Method | Endpoint | Description |
|---|---|---|
| POST | `/tasks` | Create task |
| GET | `/tasks` | List with pagination, filters, search, sorting |
| GET | `/tasks/stats` | Counts by priority and category |
| GET | `/tasks/:id` | Get one task |
| PUT | `/tasks/:id` | Update fields |
| PATCH | `/tasks/:id/toggle` | Flip `completed` |
| DELETE | `/tasks/:id` | Delete task |

**Query parameters for `GET /tasks`**

| Param | Example | Notes |
|---|---|---|
| `page`, `limit` | `page=2&limit=5` | limit capped at 100 |
| `completed` | `completed=false` | |
| `priority` | `priority=high` | `low / medium / high` |
| `category` | `category=work` | |
| `search` | `search=report` | case-insensitive title match |
| `dueAfter`, `dueBefore` | `dueBefore=2026-12-31` | ISO dates |
| `sortBy`, `order` | `sortBy=dueDate&order=asc` | default `createdAt desc` |

### Example requests
```bash
# Register
curl -X POST localhost:5000/api/users/register -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com","password":"secret123"}'

# Create a task
curl -X POST localhost:5000/api/tasks -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"finish assignment","priority":"high","category":"work","dueDate":"2026-11-05"}'

# High-priority work tasks, page 1, due date ascending
curl "localhost:5000/api/tasks?priority=high&category=work&sortBy=dueDate&order=asc&page=1&limit=5" \
  -H "Authorization: Bearer $TOKEN"
```

Example response (`GET /tasks`):
```json
{
  "success": true,
  "count": 1,
  "data": [{ "_id": "...", "title": "Finish assignment", "priority": "high", "completed": false, "category": "work" }],
  "pagination": { "page": 1, "limit": 5, "total": 1, "pages": 1, "hasNext": false, "hasPrev": false }
}
```

---

## 7. Testing Evidence

### Automated tests (`npm test`) – 23 cases in `tests/api.test.js`
| Group | Cases covered |
|---|---|
| Authentication | register returns JWT · password stored hashed · duplicate email → 409 · invalid email/short password → 400 · login OK · wrong password → 401 · no token → 401 · bad token → 401 |
| Task CRUD | create (defaults + capitalized title) · validation errors (short title, bad enum) · read one · update (owner cannot be changed) · toggle · invalid id → 400 / unknown id → 404 · delete |
| Query features | pagination metadata · filter by priority+category · filter by completed · title search · sort by title · stats aggregation |
| Relationships | user A cannot access user B's tasks · deleting a user cascades to tasks |

> 📸 **Paste your own `npm test` output screenshot here** (`docs/screenshots/tests.png`).

### Manual validation (Postman / Thunder Client)
| Scenario | Expected |
|---|---|
| `POST /tasks` with `title: "ab"` | 400 – "Title must be at least 3 characters" |
| `POST /tasks` with `priority: "urgent"` | 400 – "Priority must be low, medium or high" |
| `GET /tasks` without token | 401 |
| `GET /tasks/123` | 400 – "Invalid task id" |
| Register same email twice | 409 |

---

## 8. Visual Documentation (screenshots to add)

Take these screenshots and save them in `docs/screenshots/`, then reference them here:

| # | Screenshot | File |
|---|---|---|
| 1 | Atlas cluster + database user + network access | `atlas-cluster.png` |
| 2 | Terminal showing `✅ MongoDB connected` | `connected.png` |
| 3 | Compass view of `users` (hashed password) and `tasks` collections | `compass.png` |
| 4 | Postman: register & login returning a token | `auth.png` |
| 5 | Postman: create task (201) and validation error (400) | `tasks-create.png` |
| 6 | Postman: paginated + filtered `GET /tasks` | `tasks-query.png` |
| 7 | `npm test` passing | `tests.png` |
| 8 | Live deployed URL responding | `deployed.png` |

```md
![Compass](docs/screenshots/compass.png)
```

---

## 9. Deployment (Day 7)

**Render (free tier)**
1. Push the repo to GitHub (`.env` is git-ignored).
2. Render → *New → Web Service* → connect the repo.
3. Build command `npm install` · Start command `npm start`.
4. Add environment variables: `NODE_ENV=production`, `MONGODB_URI`, `JWT_SECRET`, `JWT_EXPIRES_IN`.
5. In Atlas *Network Access*, allow Render's outbound IPs (or `0.0.0.0/0` for a student project).
6. Open `https://<your-service>.onrender.com/api/health` to verify.

---

## 10. Seven-Day Plan Mapping

| Day | Work | Files |
|---|---|---|
| 1 | Atlas account, cluster, DB user, IP access | – |
| 2 | Connect Express to MongoDB | `server.js`, `src/config/database.js` |
| 3 | User & Task schemas with validation | `src/models/*` |
| 4 | Register, login, JWT middleware | `userController.js`, `auth.js`, `userRoutes.js` |
| 5 | Task CRUD with user association | `taskController.js`, `taskRoutes.js` |
| 6 | Pagination, filtering, sorting | `getTasks` |
| 7 | Testing and deployment | `tests/api.test.js`, Render |

## 11. Submission Checklist
- [x] `server.js`, `package.json`
- [x] `src/models/Task.js`, `src/models/User.js`
- [x] `src/controllers/taskController.js`, `src/controllers/userController.js`
- [x] `src/middleware/auth.js`
- [x] `src/config/database.js`
- [x] `.env.example`, `README.md`
- [ ] Add your screenshots (section 8)
- [ ] Never commit `.env` or real credentials

## License
MIT
