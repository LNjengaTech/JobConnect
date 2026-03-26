# JobConnect — Software Requirements Specification

**Functional and non-functional requirements for JobConnect**

**Version:** v1.0
**Date:** March 2026
**Author:** Lonnex Njenga
**Classification:** Internal — Project Documentation

---

## 1. Introduction

### 1.1 Purpose
This document defines the complete functional and non-functional requirements for JobConnect, a career development and employment portal for young people in Kenya. It serves as the authoritative reference for all development, testing, and acceptance activities.

### 1.2 Scope
JobConnect covers four functional domains: CV building, skills tracking, mentorship, and job matching. The MVP is a web application. Mobile is a phase-2 deliverable.

### 1.3 Definitions

| Term              | Definition |
|-------------------|------------|
| ATS               | Applicant Tracking System — software used by employers to filter CVs by keyword and format |
| JWT               | JSON Web Token — a compact, stateless token used for authentication |
| Access token      | Short-lived JWT (15 min) used to authenticate API requests |
| Refresh token     | Long-lived token (7 days) stored in httpOnly cookie, used to obtain new access tokens |
| Match score       | Integer 0–100 representing overlap between user skills and job required skills |
| ATS score         | Integer 0–100 computed algorithmically from CV completeness, keyword density, and structure |
| Learning path     | A curated sequence of free external resources for learning a specific skill |
| Mentor            | A verified industry professional offering free short sessions to platform users |
| Session           | A 30-minute one-on-one meeting between a mentor and a mentee |

## 2. Functional requirements

### FR-01: User management

| ID         | Requirement | Priority |
|------------|-------------|----------|
| FR-01-01   | System shall allow any visitor to register with full name, email, and password | Must |
| FR-01-02   | System shall validate email uniqueness at registration time | Must |
| FR-01-03   | System shall hash passwords using bcrypt with cost factor ≥ 12 | Must |
| FR-01-04   | System shall issue a 15-minute access token and a 7-day refresh token on successful login | Must |
| FR-01-05   | System shall rotate refresh tokens on each use (revoke old, issue new) | Must |
| FR-01-06   | System shall allow users to view and update their profile fields | Must |
| FR-01-07   | System shall allow users to upload a profile avatar (max 5 MB, JPEG/PNG) | Must |
| FR-01-08   | System shall support four user roles: student, mentor, employer, admin | Must |
| FR-01-09   | System shall allow users to logout (revoke refresh token) | Must |

### FR-02: CV builder

| ID         | Requirement | Priority |
|------------|-------------|----------|
| FR-02-01   | System shall allow a user to create multiple CV versions | Must |
| FR-02-02   | System shall support six CV section types: personal_info, experience, education, skills, summary, projects | Must |
| FR-02-03   | System shall allow sections to be added, edited, reordered, and deleted | Must |
| FR-02-04   | System shall compute an ATS score (0–100) based on section completeness, keyword presence, and action verb usage | Must |
| FR-02-05   | System shall display specific tips when ATS score is below 70 | Must |
| FR-02-06   | System shall allow export of any CV version as a PDF using at least two templates | Must |
| FR-02-07   | System shall allow one CV to be marked as "active" for use in job applications | Must |
| FR-02-08   | System shall store version history — users can view previous CV states | Should |

### FR-03: Skills tracker

| ID         | Requirement | Priority |
|------------|-------------|----------|
| FR-03-01   | System shall provide a predefined taxonomy of at least 50 skills across 5 categories | Must |
| FR-03-02   | System shall allow users to tag skills with a proficiency level (1–5) | Must |
| FR-03-03   | System shall display a radar chart comparing user proficiency vs. market demand | Must |
| FR-03-04   | System shall surface learning path recommendations based on identified skill gaps | Must |
| FR-03-05   | System shall allow users to mark learning path lessons as complete | Must |
| FR-03-06   | System shall track and display progress percentage per learning path | Must |
| FR-03-07   | System shall link tagged skills to the skills section of the user's active CV | Should |

### FR-04: Mentorship

| ID         | Requirement | Priority |
|------------|-------------|----------|
| FR-04-01   | System shall allow users with role=mentor to create and publish a mentor profile | Must |
| FR-04-02   | System shall require admin approval before a mentor profile is visible | Must |
| FR-04-03   | System shall allow mentors to set available time slots | Must |
| FR-04-04   | System shall allow any authenticated student to request a session with any approved mentor | Must |
| FR-04-05   | System shall implement session states: requested → confirmed → completed \| cancelled | Must |
| FR-04-06   | System shall send email notifications on every session state change | Must |
| FR-04-07   | System shall allow mentors to attach a meeting link (Zoom/Meet URL) to confirmed sessions | Must |
| FR-04-08   | All sessions shall be free at MVP — no payment required | Must |
| FR-04-09   | System shall allow mentees to leave a star rating (1–5) after session completion | Should |

### FR-05: Job board

| ID         | Requirement | Priority |
|------------|-------------|----------|
| FR-05-01   | System shall allow users with role=employer to post, edit, and close job listings | Must |
| FR-05-02   | System shall support full-text search across job title, description, and company name | Must |
| FR-05-03   | System shall allow filtering by job type, location, and salary range | Must |
| FR-05-04   | System shall compute a match score between the logged-in user and each job listing | Must |
| FR-05-05   | System shall sort job listings by match score by default | Must |
| FR-05-06   | System shall allow one-tap job application attaching the user's active CV | Must |
| FR-05-07   | System shall track application status: applied, reviewing, interview, offered, rejected | Must |
| FR-05-08   | System shall allow employers to view applicants and update application status | Must |
| FR-05-09   | System shall notify applicants by email when their application status changes | Should |

## 3. Non-functional requirements

| Category     | Requirement                              | Metric |
|--------------|------------------------------------------|--------|
| Performance  | API response time for list endpoints     | p95 < 500 ms under normal load |
| Performance  | PDF export generation time               | < 5 seconds per export |
| Security     | Password storage                         | bcrypt cost factor ≥ 12 |
| Security     | Token expiry                             | Access 15 min, Refresh 7 days |
| Security     | Rate limiting on auth routes             | 10 requests per 15 minutes per IP |
| Availability | Target uptime using free-tier hosting    | ≥ 99% (excluding planned maintenance) |
| Usability    | Mobile responsiveness                    | Fully usable on 375 px viewport |
| Accessibility| WCAG compliance                          | WCAG 2.1 Level AA for all public pages |
| Scalability  | Concurrent users at MVP launch           | Support 200 concurrent users without degradation |
| Cost         | Monthly infrastructure cost at MVP       | Zero (free tiers only) |
| Testing      | Backend API coverage                     | ≥ 80% route coverage with integration tests |