# JobConnect — Use Case Specification

**Detailed use cases for all major user interactions on JobConnect**

**Version:** v1.0
**Date:** March 2026
**Author:** Lonnex Njenga
**Classification:** Internal — Project Documentation

---

## Use case specification

All use cases assume the system is operational and the database is accessible. Actor "system" refers to the backend API.

### 1. Authentication use cases

**UC-01: Register account**  
**Primary actor:** Visitor  
**Goal:** Create a new user account on the platform  

**Preconditions:** User is not logged in. Email address is not already registered.  

**Main flow:**
1. User fills in name, email, password
2. System validates inputs (Zod schema)
3. System checks email uniqueness
4. System hashes password (bcrypt)
5. System inserts user row with `role=student`
6. System issues access token + refresh token
7. System returns 201 + tokens
8. User is redirected to onboarding

**Alternative / error flows:**  
- Email taken → 409 Conflict  
- Invalid inputs → 422 with field errors  

**Postconditions:** User account created. User is authenticated.

**UC-02: Login**  
**Primary actor:** Registered user  
**Goal:** Authenticate and obtain tokens  

**Preconditions:** User has a registered account.  

**Main flow:**
1. User enters email + password
2. System looks up user by email
3. System verifies password with `bcrypt.compare`
4. System issues new access + refresh token pair
5. System returns 200 + tokens

**Alternative / error flows:**  
- User not found → 401  
- Password mismatch → 401  

**Postconditions:** User is authenticated with valid tokens.

**UC-03: Refresh access token**  
**Primary actor:** Authenticated user (expired access token)  
**Goal:** Obtain a new access token without re-entering credentials  

**Preconditions:** User holds a valid, non-expired, non-revoked refresh token in httpOnly cookie.  

**Main flow:**
1. Frontend sends `POST /auth/refresh` (cookie sent automatically)
2. System reads token from cookie
3. System validates hash against DB record
4. System revokes old refresh token
5. System issues new access token + new refresh token
6. System returns 200 + new access token

**Alternative / error flows:**  
- Token expired or revoked → 401 (user must log in again)  

**Postconditions:** User holds fresh access token. Old refresh token is revoked.

### 2. CV builder use cases

**UC-04: Build CV**  
**Primary actor:** Student  
**Goal:** Create a complete, ATS-optimised CV  

**Preconditions:** User is authenticated. User has completed onboarding.  

**Main flow:**
1. User creates a new CV (gives it a title)
2. User adds personal info section
3. User adds experience section(s) with role, company, dates, bullet points
4. User adds education section
5. User adds skills section (pulled from `user_skills`)
6. User adds optional summary
7. System computes ATS score after each save
8. System displays improvement tips if score < 70

**Alternative / error flows:**  
- Summary missing → +10 tip  
- Invalid date range → validation error  

**Postconditions:** CV saved with computed ATS score. CV can be exported or attached to applications.

**UC-05: Export CV as PDF**  
**Primary actor:** Student  
**Goal:** Download CV as a formatted PDF  

**Preconditions:** User has at least one CV with at least one completed section.  

**Main flow:**
1. User selects a CV and chooses a template
2. System renders CV as HTML with chosen template
3. System uses Puppeteer to render PDF
4. System uploads to Cloudinary
5. System returns download URL
6. Browser triggers download

**Alternative / error flows:**  
- Puppeteer timeout → 503 with retry message  

**Postconditions:** PDF file downloaded to user's device.

### 3. Skills use cases

**UC-06: Tag skills**  
**Primary actor:** Student  
**Goal:** Record skills and proficiency for profile and matching  

**Preconditions:** User is authenticated.  

**Main flow:**
1. User searches or browses the skill taxonomy
2. User selects a skill and rates proficiency (1–5)
3. System saves `user_skill` record
4. System updates skill gap radar
5. System surfaces relevant learning paths

**Alternative / error flows:**  
- Skill already tagged → update proficiency  

**Postconditions:** User skills saved. Gap radar and job match scores updated.

**UC-07: Start learning path**  
**Primary actor:** Student  
**Goal:** Begin a curated skill-building journey  

**Preconditions:** User is authenticated. At least one learning path exists for the target skill.  

**Main flow:**
1. User opens gap radar and sees a gap
2. System surfaces recommended learning path
3. User clicks Start
4. System creates `user_path_progress` record (0%)
5. User opens an external resource link
6. User marks lesson complete
7. System updates `progress_pct`
8. System awards completion badge when `progress_pct = 100`

**Alternative / error flows:**  
- External link broken → user can report it  

**Postconditions:** Progress tracked. Skill proficiency updated when path is completed.

### 4. Mentorship use cases

**UC-08: Book mentor session**  
**Primary actor:** Student  
**Goal:** Request a free 30-minute session with a mentor  

**Preconditions:** User is authenticated with `role=student`. Target mentor is approved and available.  

**Main flow:**
1. User browses mentor list (filtered by skill/category)
2. User opens mentor profile
3. User selects an available time slot
4. User submits session request with optional topic note
5. System creates session record (`status=requested`)
6. System emails mentor with request details
7. Mentor confirms or declines (separate flow)
8. System emails mentee with confirmation + meeting link

**Alternative / error flows:**  
- No slots available → "Join waitlist"  
- Mentor declines → `status=cancelled`, mentee notified  

**Postconditions:** Session record created. Both parties notified by email.

**UC-09: Confirm / cancel session (mentor)**  
**Primary actor:** Mentor  
**Goal:** Respond to a session request  

**Preconditions:** Mentor is authenticated. A session with `status=requested` exists.  

**Main flow:**
1. Mentor views pending session requests in dashboard
2. Mentor clicks Confirm
3. Mentor enters meeting link URL
4. System updates session `status=confirmed`
5. System emails mentee with confirmation and link

**Alternative / error flows:**  
- Mentor clicks Decline → `status=cancelled`, mentee notified  

**Postconditions:** Session confirmed with meeting link. Mentee notified.

### 5. Job board use cases

**UC-10: Apply for a job**  
**Primary actor:** Student  
**Goal:** Submit a job application with one tap  

**Preconditions:** User is authenticated. User has an active CV. Job listing is open.  

**Main flow:**
1. User views job listing (sees match score)
2. User clicks One-tap apply
3. System checks user has active CV
4. System creates `job_application` record with `status=applied`
5. System attaches active CV to application
6. System notifies employer by email
7. User sees application in tracker

**Alternative / error flows:**  
- No active CV → prompt user to build and activate CV  
- Already applied → show existing application status  

**Postconditions:** Application submitted. Application appears in user tracker and employer portal.

**UC-11: Post a job (employer)**  
**Primary actor:** Employer  
**Goal:** Publish a new job listing  

**Preconditions:** User is authenticated with `role=employer`.  

**Main flow:**
1. Employer fills in job title, description, location, type, salary range
2. Employer tags required skills from taxonomy
3. Employer submits listing
4. System validates inputs
5. System inserts `job` + `job_skills` rows
6. System updates tsvector search index
7. Listing appears on job board

**Alternative / error flows:**  
- Missing required fields → 422 with field errors  

**Postconditions:** Job listing live on platform. Students see it in matched results.