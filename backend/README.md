# Online Examination System — Backend

Spring Boot 3 / Java 17 REST API backing the [React frontend](../exam-frontend).
JWT auth, Spring Security, Spring Data JPA, PostgreSQL.

## Quick start

```bash
# 1. Start PostgreSQL (or point at your own instance via env vars below)
docker compose up -d

# 2. Run the app — Hibernate creates the schema on first boot (ddl-auto=update)
mvn spring-boot:run
```

(No Maven wrapper is bundled — run `mvn -N io.takari:maven:wrapper` once if you'd
rather use `./mvnw`, or just use your local Maven/IDE.)

The API is served under `http://localhost:8080/api` (context-path is `/api`,
matching the frontend's `VITE_API_BASE_URL`).

On first boot, a default admin account is seeded — check the startup logs for
the username, and sign in to change the password immediately (you'll be
forced through the change-password flow automatically). Override it before
deploying anywhere shared:

```bash
export SEED_ADMIN_USERNAME=admin
export SEED_ADMIN_PASSWORD='ChangeThisImmediately!'
```

## Configuration (env vars)

| Variable | Default | Purpose |
|---|---|---|
| `DB_URL` | `jdbc:postgresql://localhost:5432/examdb` | Postgres JDBC URL |
| `DB_USERNAME` / `DB_PASSWORD` | `postgres` / `postgres` | DB credentials |
| `JWT_SECRET` | (dev-only default, **change in prod**) | Base64-encoded HMAC signing key — generate with `openssl rand -base64 64` |
| `JWT_EXPIRATION_MS` | `28800000` (8h) | Token lifetime |
| `SEED_ADMIN_USERNAME` / `SEED_ADMIN_PASSWORD` | `admin` / `ChangeMe@123` | First-boot admin account |
| `CORS_ALLOWED_ORIGINS` | `http://localhost:5173` | Comma-separated list of allowed frontend origins |

## Auth contract (matches the frontend)

```
POST /api/auth/login
  { "username": "...", "password": "..." }
  -> { "token": "...", "role": "ADMIN"|"STUDENT", "username": "...", "mustChangePassword": bool }

POST /api/auth/change-password   (requires Authorization: Bearer <token>)
  { "currentPassword": "...", "newPassword": "..." }
```

Every other response is wrapped as `{ "status": "success"|"error", "message": "...", "data": ... }`.

## Endpoints implemented so far

| Method | Path | Role | Purpose |
|---|---|---|---|
| POST | `/auth/login` | public | Sign in |
| POST | `/auth/change-password` | any authenticated | Forced first-login reset |
| POST | `/admin/courses` | ADMIN | Add a course |
| GET | `/admin/courses` | ADMIN | List courses |
| POST | `/admin/students` | ADMIN | Register a student — auto-generates username (= registration number) and a random password, returned **once** in the response |
| GET | `/admin/students?courseId=&semester=` | ADMIN | List students, optionally filtered |
| POST | `/admin/subjects` | ADMIN | Add a subject |
| GET | `/admin/subjects` | ADMIN | List subjects |
| POST | `/admin/allotment` | ADMIN | Allot a subject — either to one student (`studentId`) or in bulk to a whole course+semester (`courseId` + `semester`) |
| GET | `/admin/allotment/student/{studentId}` | ADMIN | List a student's allotted subjects |
| POST / PUT / DELETE | `/admin/questions` | ADMIN | Question bank CRUD (filter GET by `?subjectId=`) |
| POST | `/admin/exams` | ADMIN | Schedule an exam for a subject (name, window, per-student duration, question count drawn per attempt) |
| GET | `/admin/exams` | ADMIN | List all scheduled exams |
| DELETE | `/admin/exams/{examId}` | ADMIN | Delete a scheduled exam — rejected with 409 once any student has an attempt on record |
| GET | `/student/exams` | STUDENT | This student's exams (allotted subjects), each with `attemptStatus` (`NOT_STARTED`/`IN_PROGRESS`/`SUBMITTED`/`AUTO_SUBMITTED`) and `score` once graded |
| POST | `/student/exams/{examId}/start` | STUDENT | Start an attempt (or resume the in-progress one) — draws `totalQuestions` random questions from the subject's bank and returns the server-authoritative `expiresAt` |
| PUT | `/student/attempts/{attemptId}/answer` | STUDENT | Autosave one answer — `{ questionId, selectedOption }` (`selectedOption: null` clears it) |
| POST | `/student/attempts/{attemptId}/submit` | STUDENT | Grade and close the attempt using whatever answers are saved server-side; returns `totalScore`, `maxMarks`, `passingMarks`, `passed`, and a per-question `breakdown` |
| GET | `/student/results` | STUDENT | Summary list of this student's graded attempts (`SUBMITTED`/`AUTO_SUBMITTED`), most recent first |
| GET | `/student/results/{attemptId}` | STUDENT | Full result for one attempt, including the per-question `breakdown` — 403 if it isn't the caller's own attempt |

Attempts whose timer runs out without a manual submit are auto-graded —
inline the next time that student hits `/student/exams` or `/start`, and
also by a background sweep (`AttemptExpirySweeper`, every 60s) for students
who never come back at all.

**Not yet implemented** (frontend has a placeholder page ready): report
generation. Follows the same Controller → Service → Repository → Entity
pattern already in place.

## Registering a student — example

```bash
curl -X POST http://localhost:8080/api/admin/students \
  -H "Authorization: Bearer <admin JWT>" \
  -H "Content-Type: application/json" \
  -d '{
    "regNo": "2324CO123",
    "name": "Asha Patil",
    "courseId": 1,
    "semester": 4,
    "email": "asha@example.com",
    "mobile": "9876543210"
  }'
```

Response includes `generatedUsername` (= `regNo`) and `generatedPassword` —
hand these to the student once; only the BCrypt hash is stored, so if the
password is lost it must be reset, not retrieved.

## Project layout

```
src/main/java/com/ssit/examportal/
├── config/         # SecurityConfig (CORS, JWT filter wiring, role matchers)
├── security/        # JwtUtil, JwtAuthenticationFilter, CustomUserDetailsService
├── entity/          # JPA entities: User, Student, Course, Subject, StudentSubjectAllotment
├── repository/       # Spring Data repositories
├── dto/             # Request/response payloads
├── service/          # Business logic (incl. PasswordGenerator)
├── controller/       # REST controllers
├── exception/        # ApiException + GlobalExceptionHandler
└── bootstrap/        # AdminSeeder (first-boot admin account)
```
