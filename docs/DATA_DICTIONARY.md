# JobConnect Data Dictionary

**Complete field-level reference for all database tables in JobConnect**

**Version:** v1.0  
**Date:** March 2026  
**Author:** Lonnex Njenga  
**Classification:** Internal — Project Documentation

## Data dictionary

All tables use UUID primary keys (`gen_random_uuid()`). All timestamps are `TIMESTAMPTZ`. Soft deletes use `is_active` boolean. Enums are PostgreSQL native ENUM types mirrored in Prisma schema.

### Enumerations

| Enum name          | Values                                      | Used in                          |
|--------------------|---------------------------------------------|----------------------------------|
| `user_role`        | student, mentor, employer, admin            | `users.role`                     |
| `session_status`   | requested, confirmed, completed, cancelled  | `mentor_sessions.status`         |
| `job_type`         | full_time, part_time, contract, internship, remote | `jobs.job_type`            |
| `application_status` | applied, reviewing, interview, offered, rejected | `job_applications.status` |
| `section_type`     | personal_info, experience, education, skills, summary, projects | `cv_sections.section_type` |

## Tables

### `users`
Core user entity. All actors (students, mentors, employers, admins) are rows in this table differentiated by the `role` column.

| Column        | Type          | Constraints                          | Description |
|---------------|---------------|--------------------------------------|-------------|
| `id`          | UUID          | PK, `DEFAULT gen_random_uuid()`      | Unique user identifier |
| `email`       | VARCHAR(255)  | NOT NULL, UNIQUE                     | User login email address |
| `password_hash` | VARCHAR(255)| NOT NULL                             | bcrypt hash of the user's password (cost 12). Never returned in API responses. |
| `full_name`   | VARCHAR(200)  | NOT NULL                             | Display name shown throughout the platform |
| `phone`       | VARCHAR(20)   | NULLABLE                             | Optional phone number for contact |
| `avatar_url`  | TEXT          | NULLABLE                             | Cloudinary URL of the user's profile picture |
| `role`        | `user_role`   | NOT NULL, `DEFAULT student`          | User's role on the platform. Controls access permissions. |
| `location`    | VARCHAR(200)  | NULLABLE                             | City or region (e.g. "Nairobi", "Mombasa") |
| `is_active`   | BOOLEAN       | NOT NULL, `DEFAULT true`             | Soft-delete flag. False = account deactivated. Never hard-delete users. |
| `created_at`  | TIMESTAMPTZ   | NOT NULL, `DEFAULT NOW()`            | Account creation timestamp |
| `updated_at`  | TIMESTAMPTZ   | NOT NULL, `DEFAULT NOW()`            | Last profile update timestamp. Updated by trigger on any column change. |

### `refresh_tokens`
Stores hashed refresh tokens for the JWT refresh flow. Enables token rotation and revocation.

| Column       | Type          | Constraints                          | Description |
|--------------|---------------|--------------------------------------|-------------|
| `id`         | UUID          | PK                                   | Token identifier |
| `user_id`    | UUID          | NOT NULL, FK → `users.id` ON DELETE CASCADE | Owner of this token |
| `token_hash` | VARCHAR(255)  | NOT NULL, UNIQUE                     | bcrypt hash of the raw refresh token string. Raw token is sent to client and never stored. |
| `expires_at` | TIMESTAMPTZ   | NOT NULL                             | Expiry time (created_at + 7 days) |
| `is_revoked` | BOOLEAN       | NOT NULL, `DEFAULT false`            | Set to true on use (rotation) or logout. Checked on every refresh request. |
| `created_at` | TIMESTAMPTZ   | NOT NULL, `DEFAULT NOW()`            | Token issuance time |

### `cvs`
A user can have multiple CV versions. One can be marked active for job applications.

| Column      | Type         | Constraints                          | Description |
|-------------|--------------|--------------------------------------|-------------|
| `id`        | UUID         | PK                                   | CV identifier |
| `user_id`   | UUID         | NOT NULL, FK → `users.id` ON DELETE CASCADE | Owner of this CV |
| `title`     | VARCHAR(200) | NOT NULL                             | User-assigned name for this CV version (e.g. "Safaricom tailored") |
| `ats_score` | SMALLINT     | NOT NULL, `DEFAULT 0`                | Computed 0–100 score. Recomputed server-side after every section change. |
| `is_active` | BOOLEAN      | NOT NULL, `DEFAULT false`            | Only one CV per user can be active at a time. Used for one-tap apply. |
| `created_at`| TIMESTAMPTZ  | NOT NULL, `DEFAULT NOW()`            | CV creation timestamp |
| `updated_at`| TIMESTAMPTZ  | NOT NULL, `DEFAULT NOW()`            | Last modification timestamp |

### `cv_sections`
Individual sections within a CV. Content stored as JSONB to accommodate varied structures per section type.

| Column         | Type         | Constraints                          | Description |
|----------------|--------------|--------------------------------------|-------------|
| `id`           | UUID         | PK                                   | Section identifier |
| `cv_id`        | UUID         | NOT NULL, FK → `cvs.id` ON DELETE CASCADE | Parent CV |
| `section_type` | `section_type` | NOT NULL                           | Determines schema of the content JSONB field |
| `content`      | JSONB        | NOT NULL                             | Section data. Schema varies by type — see section content schemas below. |
| `sort_order`   | SMALLINT     | NOT NULL, `DEFAULT 0`                | Display order within the CV. User can reorder sections. |

**Section content schemas (JSONB):**
- `personal_info`: `{ firstName, lastName, email, phone, location, linkedinUrl, githubUrl, websiteUrl }`
- `experience`: `{ company, role, startDate, endDate, isCurrent, bullets: [string] }`
- `education`: `{ institution, degree, field, startYear, endYear, grade? }`
- `skills`: `{ skills: [{ name, level }] }` — auto-populated from `user_skills`
- `summary`: `{ text }`
- `projects`: `{ name, description, url?, technologies: [string], bullets: [string] }`

### `skills`
Master skill taxonomy. Seeded by admins. Skills are shared across users, jobs, and learning paths.

| Column       | Type         | Constraints               | Description |
|--------------|--------------|---------------------------|-------------|
| `id`         | UUID         | PK                        | Skill identifier |
| `name`       | VARCHAR(100) | NOT NULL, UNIQUE          | Canonical skill name (e.g. "React.js", "PostgreSQL") |
| `category`   | VARCHAR(100) | NOT NULL                  | Grouping (e.g. "Frontend", "Backend", "Data", "DevOps", "Soft skills") |
| `description`| TEXT         | NULLABLE                  | Brief description of the skill for display in the taxonomy browser |

### `user_skills`
Junction table recording which skills a user has and at what proficiency level.

| Column        | Type         | Constraints                          | Description |
|---------------|--------------|--------------------------------------|-------------|
| `id`          | UUID         | PK                                   | Record identifier |
| `user_id`     | UUID         | NOT NULL, FK → `users.id` ON DELETE CASCADE | User who has this skill |
| `skill_id`    | UUID         | NOT NULL, FK → `skills.id`           | The skill being recorded |
| `proficiency` | SMALLINT     | NOT NULL, CHECK (proficiency BETWEEN 1 AND 5) | 1=Beginner, 2=Elementary, 3=Intermediate, 4=Advanced, 5=Expert |
| `verified_at` | TIMESTAMPTZ  | NULLABLE                             | Reserved for future skill verification feature |

### `learning_paths`
Curated sequences of free external learning resources. Seeded and maintained by admins.

| Column          | Type         | Constraints                          | Description |
|-----------------|--------------|--------------------------------------|-------------|
| `id`            | UUID         | PK                                   | Path identifier |
| `skill_id`      | UUID         | NOT NULL, FK → `skills.id`           | The skill this path teaches |
| `title`         | VARCHAR(200) | NOT NULL                             | Path name (e.g. "TypeScript for JavaScript developers") |
| `description`   | TEXT         | NULLABLE                             | What the learner will achieve by completing this path |
| `external_url`  | TEXT         | NOT NULL                             | URL to the external resource (YouTube, freeCodeCamp, The Odin Project, etc.) |
| `duration_mins` | SMALLINT     | NULLABLE                             | Estimated time to complete in minutes |
| `is_published`  | BOOLEAN      | NOT NULL, `DEFAULT false`            | Only published paths are shown to users |

### `user_path_progress`
Tracks each user's progress through each learning path.

| Column         | Type         | Constraints                                      | Description |
|----------------|--------------|--------------------------------------------------|-------------|
| `id`           | UUID         | PK                                               | Progress record identifier |
| `user_id`      | UUID         | NOT NULL, FK → `users.id` ON DELETE CASCADE      | User whose progress is recorded |
| `path_id`      | UUID         | NOT NULL, FK → `learning_paths.id`               | The learning path being tracked |
| `progress_pct` | SMALLINT     | NOT NULL, `DEFAULT 0`, CHECK (0–100)             | Percentage completion (0–100). Updated by user manually. |
| `completed_at` | TIMESTAMPTZ  | NULLABLE                                         | Set automatically when `progress_pct` reaches 100 |

### `mentor_profiles`
Extended profile for users with `role=mentor`. Created by the user; activated by admin approval.

| Column         | Type         | Constraints                          | Description |
|----------------|--------------|--------------------------------------|-------------|
| `id`           | UUID         | PK                                   | Mentor profile identifier |
| `user_id`      | UUID         | NOT NULL, UNIQUE, FK → `users.id` ON DELETE CASCADE | Linked user account. One profile per user. |
| `company`      | VARCHAR(200) | NOT NULL                             | Current employer or organisation |
| `title`        | VARCHAR(200) | NOT NULL                             | Job title (e.g. "Senior Software Engineer") |
| `bio`          | TEXT         | NOT NULL                             | Short biography displayed on the mentor listing |
| `is_approved`  | BOOLEAN      | NOT NULL, `DEFAULT false`            | Set to true by admin. Only approved mentors appear in listings. |
| `is_available` | BOOLEAN      | NOT NULL, `DEFAULT true`             | Mentor can toggle this to pause new session requests |

### `mentor_sessions`
Records every session request between a mentee and a mentor. Tracks the full state machine.

| Column         | Type            | Constraints                          | Description |
|----------------|-----------------|--------------------------------------|-------------|
| `id`           | UUID            | PK                                   | Session identifier |
| `mentor_id`    | UUID            | NOT NULL, FK → `mentor_profiles.id`  | The mentor for this session |
| `mentee_id`    | UUID            | NOT NULL, FK → `users.id` ON DELETE CASCADE | The student requesting the session |
| `scheduled_at` | TIMESTAMPTZ     | NOT NULL                             | Agreed date and time for the session |
| `duration_mins`| SMALLINT        | NOT NULL, `DEFAULT 30`               | Session length in minutes |
| `status`       | `session_status`| NOT NULL, `DEFAULT requested`        | Current state in the state machine |
| `notes`        | TEXT            | NULLABLE                             | Topic or agenda note from the mentee at booking time |
| `meet_link`    | TEXT            | NULLABLE                             | Video call URL added by mentor on confirmation |
| `created_at`   | TIMESTAMPTZ     | NOT NULL, `DEFAULT NOW()`            | When the session was first requested |
| `updated_at`   | TIMESTAMPTZ     | NOT NULL, `DEFAULT NOW()`            | Last state change timestamp |

### `jobs`
Job listings posted by employers. Includes a full-text search vector for fast keyword search.

| Column          | Type        | Constraints                          | Description |
|-----------------|-------------|--------------------------------------|-------------|
| `id`            | UUID        | PK                                   | Job listing identifier |
| `employer_id`   | UUID        | NOT NULL, FK → `users.id`            | User who posted the listing (`role=employer`) |
| `title`         | VARCHAR(200)| NOT NULL                             | Job title (e.g. "Junior React Developer") |
| `description`   | TEXT        | NOT NULL                             | Full job description displayed to applicants |
| `company`       | VARCHAR(200)| NOT NULL                             | Company or organisation name |
| `location`      | VARCHAR(200)| NOT NULL                             | Location string (e.g. "Nairobi", "Remote", "Mombasa") |
| `job_type`      | `job_type`  | NOT NULL                             | Employment type from the `job_type` enum |
| `salary_min`    | INTEGER     | NULLABLE                             | Minimum salary in the specified currency |
| `salary_max`    | INTEGER     | NULLABLE                             | Maximum salary in the specified currency |
| `currency`      | VARCHAR(3)  | NOT NULL, `DEFAULT KES`              | ISO 4217 currency code (e.g. KES, USD) |
| `is_active`     | BOOLEAN     | NOT NULL, `DEFAULT true`             | Soft-delete / close flag. Closed listings no longer appear in search. |
| `posted_at`     | TIMESTAMPTZ | NOT NULL, `DEFAULT NOW()`            | Publication timestamp |
| `expires_at`    | TIMESTAMPTZ | NULLABLE                             | Optional expiry date after which listing is auto-closed |
| `search_vector` | TSVECTOR    | NULLABLE                             | Full-text search index. Updated automatically by a trigger on title, description, company. |

### `job_skills`
Junction table recording which skills a job requires. Powers the match score calculation.

| Column      | Type     | Constraints                                      | Description |
|-------------|----------|--------------------------------------------------|-------------|
| `job_id`    | UUID     | PK (composite), FK → `jobs.id` ON DELETE CASCADE | The job listing |
| `skill_id`  | UUID     | PK (composite), FK → `skills.id`                 | The required skill |
| `is_required`| BOOLEAN | NOT NULL, `DEFAULT true`                         | Distinguishes required from preferred skills. Only required skills are used in match score. |

### `job_applications`
Records every application submitted via the platform. Central table for the application tracker feature.

| Column        | Type                  | Constraints                          | Description |
|---------------|-----------------------|--------------------------------------|-------------|
| `id`          | UUID                  | PK                                   | Application identifier |
| `job_id`      | UUID                  | NOT NULL, FK → `jobs.id`             | The job being applied to |
| `user_id`     | UUID                  | NOT NULL, FK → `users.id` ON DELETE CASCADE | The applicant |
| `cv_id`       | UUID                  | NOT NULL, FK → `cvs.id`              | Snapshot of the CV used for this application. Immutable after submission. |
| `status`      | `application_status`  | NOT NULL, `DEFAULT applied`          | Current status in the application workflow |
| `cover_note`  | TEXT                  | NULLABLE                             | Optional short message from the applicant |
| `match_score` | SMALLINT              | NOT NULL                             | Computed match score at time of application. Stored for historical accuracy. |
| `applied_at`  | TIMESTAMPTZ           | NOT NULL, `DEFAULT NOW()`            | Submission timestamp |
| `updated_at`  | TIMESTAMPTZ           | NOT NULL, `DEFAULT NOW()`            | Last status change timestamp |

## Indexes

| Table                | Column(s)                  | Type          | Purpose |
|----------------------|----------------------------|---------------|---------|
| `users`              | `email`                    | UNIQUE BTREE  | Fast email lookup at login/register |
| `refresh_tokens`     | `token_hash`               | UNIQUE BTREE  | Fast token validation on refresh |
| `refresh_tokens`     | `user_id`                  | BTREE         | Fast cascade on user deletion |
| `cvs`                | `user_id`                  | BTREE         | Fast CV listing for authenticated user |
| `user_skills`        | `(user_id, skill_id)`      | UNIQUE BTREE  | Prevent duplicate skill tags |
| `user_path_progress` | `(user_id, path_id)`       | UNIQUE BTREE  | Prevent duplicate progress records |
| `mentor_profiles`    | `user_id`                  | UNIQUE BTREE  | One profile per user |
| `mentor_sessions`    | `mentor_id`                | BTREE         | Fast session listing for mentors |
| `mentor_sessions`    | `mentee_id`                | BTREE         | Fast session listing for mentees |
| `jobs`               | `search_vector`            | GIN           | Full-text search performance |
| `jobs`               | `employer_id`              | BTREE         | Fast listing for employer portal |
| `job_skills`         | `skill_id`                 | BTREE         | Fast skill-to-job lookups for matching |
| `job_applications`   | `(job_id, user_id)`        | UNIQUE BTREE  | Prevent duplicate applications |
| `job_applications`   | `user_id`                  | BTREE         | Fast application listing for user tracker |