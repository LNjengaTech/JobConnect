# JobConnect — Coding Agent Prompt

**Complete project brief for AI-assisted development**

**Version:** v1.0
**Date:** March 2026
**Author:** Lonnex Njenga
**Classification:** Internal — Project Documentation

---

## Coding agent — full project brief

This document is the **authoritative prompt** for any AI coding agent assisting with the development of **JobConnect**. Read it in full before writing any code, generating any files, or making any architectural decisions.

### 1. What is JobConnect?

JobConnect is a full-cycle career development and employment portal built specifically for young people in Kenya. It solves a real, felt problem: having technical skills is no longer enough to get a job. Young Kenyan graduates can build software, understand complex systems, and solve hard problems — yet the path from “I have skills” to “I have a job” remains fragmented, overwhelming, and opaque.

The platform is built by youth, for youth. It is **not a job board**. It is a complete career engine with four tightly integrated pillars:

- **CV builder** — ATS-optimised CVs in minutes, with rule-based scoring and PDF export  
- **Skills tracker** — self-rated skill proficiency, gap radar vs. market demand, curated learning paths  
- **Mentorship** — free short sessions with real Kenyan industry professionals  
- **Job board** — entry-level listings matched to the user’s actual skill profile, with one-tap apply

### 2. Who uses it?

| Role              | Description                                      | Key actions                                      |
|-------------------|--------------------------------------------------|--------------------------------------------------|
| Student / job-seeker | Primary user. University student or recent graduate in Kenya. | Build CV, track skills, book mentor, apply to jobs |
| Mentor            | Verified industry professional. Volunteers time. | Set availability, confirm sessions, give feedback |
| Employer          | Company or recruiter posting entry-level roles.  | Post jobs, review applicants, update status      |
| Admin             | Platform operator.                               | Approve mentors, moderate content, seed data     |

### 3. MVP scope — what to build now

**IN scope for the MVP:**

- User registration and login with custom JWT (no OAuth, no third-party auth)  
- User profile management with avatar upload (Cloudinary free tier)  
- CV CRUD — create, edit, delete CV sections (personal info, experience, education, skills, summary)  
- Rule-based ATS scoring (0–100, algorithmic — no AI)  
- CV PDF export (Puppeteer, server-side)  
- Skill tagging with proficiency self-rating  
- Learning path tracking (admin-seeded paths linking to free external resources)  
- Skill gap radar (user skills vs. market demand keywords from job listings)  
- Mentor profile listing and browsing  
- Free mentor session booking with state machine (requested → confirmed → completed | cancelled)  
- Email notifications for session state changes via Nodemailer + Gmail SMTP  
- Job listings CRUD with employer portal  
- PostgreSQL full-text job search via tsvector  
- Skill-based job matching (SQL set intersection — no ML)  
- One-tap job application (attaches user’s active CV)  
- Application status tracker  

**Explicitly OUT of scope for MVP:**

- AI-powered CV suggestions, rewrites, or keyword advice (post-MVP)  
- M-Pesa or Stripe payment integration (post-MVP)  
- Redis caching layer (post-MVP)  
- Google OAuth or any third-party authentication (never for MVP)  
- React Native mobile app (phase 2)  
- Zoom/Google Meet API integration (mentor provides link manually for MVP)  
- SMS notifications via Africa’s Talking (post-MVP)

### 4. Tech stack (non-negotiable)

| Layer          | Technology                          | Rationale                                      |
|----------------|-------------------------------------|------------------------------------------------|
| Frontend       | Next.js 14 (App Router)             | SSR for SEO, React ecosystem, free on Vercel   |
| Backend        | Node.js + Express                   | Lightweight, well-understood, free to host     |
| Database       | PostgreSQL via Neon                 | Free tier 5 GB, full-text search built in      |
| ORM / migrations | Prisma                            | Type-safe, migration tooling, good DX          |
| Auth           | Custom JWT — jsonwebtoken + bcrypt  | No vendor lock-in, no cost, full control       |
| File storage   | Cloudinary free tier                | Avatar + CV file uploads                       |
| PDF export     | Puppeteer (server-side)             | Free, high-fidelity PDF from HTML template     |
| Email          | Nodemailer + Gmail SMTP             | Free for low volume                            |
| Testing        | Jest + Supertest (backend), Playwright (E2E) | Free, industry standard                  |
| CI/CD          | GitHub Actions                      | Free on public repos                           |
| Hosting        | Vercel (frontend) + Railway/Render (backend) | Free tiers sufficient for MVP             |
| Styling        | Tailwind CSS + shadcn/ui            | Rapid development, consistent design system    |
| Icons          | Lucide React                        | Matches shadcn/ui, tree-shakeable              |
| Fonts          | Oxanium (headings) + Outfit (body)  | Defined design identity                        |

#### These are my prefered design styles, customized in ui.shadcn.com:

| Setting              | Value          |
|----------------------|----------------|
| Style                | nova           |
| Base color           | neutral        |
| Theme                | indigo         |
| Chart color          | indigo         |
| Heading font         | oxanium        |
| Body font            | outfit         |
| Icon library         | lucide         |
| Radius               | default (0.5rem) |
| Menu style           | default/solid  |
| Menu accent          | subtle         |

### 5. Architecture decisions the agent MUST follow

#### 5.1 Monorepo structure
- Root: `/apps/web` (Next.js), `/apps/api` (Express), `/packages/types` (shared TypeScript types)
- Use **pnpm workspaces**
- Shared types package is the contract between frontend and backend — **never duplicate** type definitions.

#### 5.2 Authentication
- Access token: JWT, 15-minute expiry, signed with HS256, payload = `{ sub: userId, role, iat, exp }`
- Refresh token: 7-day expiry, stored as bcrypt hash in `refresh_tokens` table, set as httpOnly Secure SameSite=Strict cookie
- Refresh flow must be in a single database transaction (validate → revoke old → insert new)
- Middleware: `verifyToken` + `requireRole('admin')` / `requireRole('mentor')`

#### 5.3 Database conventions
- All primary keys: UUID v4 (`gen_random_uuid()`)
- All timestamps: `TIMESTAMPTZ` (with timezone), default `NOW()`
- Soft deletes: `is_active` boolean — never hard-delete user data
- Enums defined in PostgreSQL as ENUM types, mirrored in Prisma schema
- Full-text search: tsvector column on jobs table, updated via trigger
- Job matching: `COUNT(intersect(user_skills, job_required_skills)) / COUNT(job_required_skills)`

#### 5.4 API conventions
- Base URL: `/api/v1/`
- All responses: `{ success: boolean, data?: any, error?: { code: string, message: string } }`
- Correct HTTP status codes (200, 201, 400, 401, 403, 404, 409, 422, 500)
- Validation: Zod schemas in middleware before controller
- Pagination: `?page=1&limit=20` on all list endpoints

#### 5.5 Security requirements
- Passwords: bcrypt cost factor 12 minimum
- Rate limiting: express-rate-limit (100 req/15min general, 10 req/15min on auth)
- CORS: allow **only** the frontend origin
- Helmet.js on all Express routes
- Environment variables: never committed; use `.env.example`
- Input sanitisation and MIME/size validation on file uploads

#### 5.6 Branch and documentation discipline
- Never commit directly to main
- Every feature branch **must** include a `BRANCH.md` at the repo root
- Every PR must pass lint, type-check, and tests
- Migrations must be tested on a clean DB
- No `console.log` in committed code — use a logger (pino or winston)

### 6. Branch order for the agent

| Branch                     | Must complete before                  |
|----------------------------|---------------------------------------|
| feat/project-setup         | —                                     |
| feat/database-schema       | project-setup                         |
| feat/ci-pipeline           | project-setup                         |
| feat/auth-register-login   | database-schema                       |
| feat/user-profile          | auth-register-login                   |
| feat/cv-crud               | user-profile                          |
| feat/cv-pdf-export         | cv-crud                               |
| feat/cv-ats-score          | cv-crud                               |
| feat/skills-crud           | user-profile                          |
| feat/learning-paths        | skills-crud                           |
| feat/skill-gap             | skills-crud + learning-paths          |
| feat/mentor-profiles       | user-profile                          |
| feat/session-booking       | mentor-profiles                       |
| feat/job-listings          | user-profile                          |
| feat/job-matching          | job-listings + skills-crud            |
| feat/apply-tracker         | job-matching + cv-crud                |

### 7. What the agent should NEVER do
- Never use localStorage/sessionStorage for tokens — httpOnly cookies only
- Never use any paid third-party service not listed in the tech stack
- Never add AI API calls in MVP branches
- Never bypass the Zod validation middleware
- Never hard-code secrets or connection strings
- Never return raw SQL errors or stack traces in API responses
- Never skip writing the `BRANCH.md` before opening a PR
- Never suggest or implement third-party auth (Clerk, Auth.js, Supabase Auth, etc.)

---

**You are now ready to begin with the first branch: `feat/project-setup`**
