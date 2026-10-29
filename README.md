# CampusPulse AI

CampusPulse AI is a full-stack student success and risk intelligence platform for universities. It combines academic, attendance, LMS, engagement, placement, skills, and feedback data into a unified student profile and generates explainable success scores and recommended actions.

## Project structure

```text
.
├── client/
│   ├── src/
│   ├── package.json
│   ├── vercel.json
│   ├── vite.config.js
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   └── index.html
├── server/
│   ├── controllers/
│   ├── middleware/
│   ├── models/
│   ├── routes/
│   ├── migrations/
│   ├── services/
│   ├── utils/
│   ├── seed/
│   ├── .env.example
│   ├── package.json
│   └── server.js
├── README.md
└── package.json
```

## Local setup on Windows

In PowerShell, use `npm.cmd` so the commands do not depend on whether script execution is enabled:

```bash
npm.cmd install
npm.cmd run dev:client
```

From the repository root, open a second terminal and start the API:

```bash
npm.cmd run dev:server
```

The frontend is at `http://localhost:5173`; the backend health check is at `http://localhost:5000/api/health`. If MongoDB is not configured, the API intentionally starts in demo mode and keeps changes only in memory. Configure `server/.env` with a valid `MONGODB_URI` and `JWT_SECRET` for persistent local data. Do not use a placeholder MongoDB URI when persistent storage is required.

## Deploy to Render

The repository-root `render.yaml` defines the client as a Render static site and the API as a Node web service. Push the repository contents, including `client`, `server`, and `render.yaml`, to GitHub. In Render, create or update the Blueprint for that repository and sync it. The Blueprint configures the frontend/API URLs, API CORS origin, and SPA route fallback automatically.

If configuring the services manually, set the frontend's `VITE_API_URL` to the API service URL (with or without a trailing `/api`), and set the API's `CLIENT_URL` to the frontend's full `https://...onrender.com` URL. The client normalizes the API URL to the `/api` base path. The API's `/api/health` endpoint should return HTTP 200 before testing the frontend.

The API starts in demo in-memory mode when `MONGODB_URI` is not set. Demo changes are not persistent across service restarts; configure a MongoDB Atlas connection as the API service's `MONGODB_URI` for persistent data. The free API service may spin down when idle.

Administrators can reset student passwords from **Admin → Manage → Students → Reset all student passwords**. The reset generates a unique temporary password for each student account, invalidates existing student sessions, and requires students to choose a new password (at least 12 characters) at their next login. Temporary passwords are shown once in the admin page; share each only with its matching student.

## Admin student registration, account email, and private photos

Only authenticated Admins can add student accounts or replace/remove student photos. Use **Admin Dashboard → Add New Student** (or **Admin → Manage → Students**) to enter the student details and optionally select a JPG, PNG, or WebP profile photo (maximum 2 MB). The browser previews the photo; the API decodes and validates the image, converts it to WebP, generates its storage key, and saves it outside the public web root. Student photos are returned only by an authenticated endpoint scoped to Admins, that student, or their assigned mentor.

After saving the student and login account, the API attempts the account-creation notification by SMTP. The admin screen reports **Sent** only when the SMTP provider accepts the destination address; this is not proof of inbox delivery. A failed submission leaves the student saved with an explicit **Failed** status and an Admin-only retry action. Retry updates the same student/account rather than creating a duplicate. The notification contains the student ID, department, academic year, and application login link; it never includes a password or photo.

Configure these API environment variables (placeholders are in `server/.env.example`):

- `SMTP_HOST`, `SMTP_PORT`, `SMTP_FROM`: SMTP server, port, and sender address.
- `SMTP_USER` and `SMTP_PASS`: both set for authenticated SMTP, or both empty when the relay is unauthenticated.
- `APP_BASE_URL`: the public CampusPulse web-app base URL used to construct `/login`.
- `STUDENT_PHOTO_DIR`: private photo storage directory. Configure a persistent mounted volume in production; do not expose this directory through a static file server. For example, on a host with a persistent `/var/data` disk set it to `/var/data/campuspulse/student-photos`.

For existing MongoDB installations, back up the database, resolve any case-insensitive duplicate user/student emails, and run from `server` before deploying:

```powershell
npm.cmd run migrate:student-registration
```

The migration normalizes existing email addresses to lowercase and ensures the existing unique student-ID and user-email indexes. Keep `MONGODB_URI` and SMTP credentials only in the server's environment or secret manager. If SMTP is not configured, account creation still saves the student but records email status as **Failed**; no simulated delivery is reported.

Newly created accounts use a random, hashed password that is never sent to the student. To provide login access, an Admin can use the student's **Issue login access** action to issue a one-time temporary password for that account only, then share it through a separate secure channel. The existing login flow requires the student to replace that temporary password at first sign-in. **Reset all student passwords** remains available for institution-wide resets. A self-service password-setup link is not part of this notification feature.

Validate the project with `npm.cmd --prefix server test` and `npm.cmd --prefix client run build`. The registration tests exercise account persistence in demo mode, duplicate rejection, Admin-only authorization, image decoding/size validation, private photo retrieval/replacement/removal, and truthful SMTP-unconfigured status. They do not send a real email. To verify actual provider acceptance, configure SMTP in a non-production test environment and create a test student address; inbox delivery must be separately confirmed with the recipient/provider.

## Deploy the frontend to Vercel

Import the GitHub repository in Vercel and set the project Root Directory to `client`. The included `client/vercel.json` configures the Vite build and SPA route fallback. To use the Vercel frontend with the API, first deploy the API (for example, with the Render Blueprint), then set Vercel's `VITE_API_URL` environment variable to the API URL ending in `/api` and redeploy. The frontend cannot use the local `localhost` API URL in production.

## Deploy the frontend to GitHub Pages

The `.github/workflows/static.yml` workflow builds the Vite client and publishes `client/dist` to GitHub Pages. Set the repository's Pages source to **GitHub Actions**, then push to `main` or run the workflow from the Actions tab. The site is published at `https://shaikchinnakasim9-star.github.io/TECHNOVA.ai-powered-student-analytics/`; the workflow configures this project path, SPA fallback, and the Render API URL automatically. The Render Blueprint allows requests from the Pages origin. The API must also be deployed from `render.yaml` and have `JWT_SECRET` set; add `MONGODB_URI` for persistent data.

## MongoDB setup

- Use MongoDB Atlas or install MongoDB Community Edition locally.
- Copy `server/.env.example` to `server/.env`, then set `MONGODB_URI` to your database connection string. Keep this secret in `server/.env`; do not commit or paste it into source files.
- When MongoDB is configured, the server connects before accepting requests, seeds demo users and students only when appropriate, and uses MongoDB for student, intervention, subject-feedback, and fee-record data. Fee account records are not fabricated in the database; add/import the college's actual fee data separately.
- When MongoDB is configured but unavailable, startup fails instead of silently using temporary demo data. An unset URI or an unfinished URI containing a `<...>` placeholder runs in in-memory demo mode. For local demo mode, the server generates a temporary JWT secret if `JWT_SECRET` is blank; existing sessions will need to sign in again after a server restart. Set a strong `JWT_SECRET` and a complete `MONGODB_URI` for persistent database mode and production.

## Academic monitoring and daily attendance

The **Academics** workspace is available to administrators, mentors/faculty, and students. It includes student selection and filters, class/subject enrollment, attendance history, assignment submission and grading, assessment history, risk alerts, and student notifications. Administrators can set attendance and marks alert thresholds in the academic overview; defaults are 75% and 60%. A score exactly at the marks threshold does not create a below-threshold alert.

Academic collections and indexes are created by Mongoose when the server connects. MongoDB mode persists subjects/enrollments, attendance, attendance edit audits, assignment submissions, assessments and their update history, risk alerts, risk score history, settings, and in-app notifications. Daily attendance is unique per student, subject, and date. The legacy period field is retained to identify a class session; a second record for that student/subject/date must be corrected by editing the saved record.

Administrators can enroll selected students in a subject/class and authorize a faculty account from **Academics → Students → Create subject / class**. Faculty and mentors can access academic records only for students assigned to them or for students in a subject/class assigned to their account. Students can only view and submit their own records; academic write APIs require an administrator or authorized staff role.

The student dashboard and notification bell show published risk alerts, marks feedback, assignment updates, attendance alerts, deadlines, and faculty-reviewed action plans. Upcoming/overdue assignment reminders are created idempotently when the student's academic assignments are loaded.

The API is mounted under `/api/academic`. Admins and authorized faculty can open **Manage Attendance** from a student profile's **Attendance** tab to mark or edit a dated class session, review filtered daily history, and configure risk weights, thresholds, and late credit. Students can only view their own daily attendance and risk history. **Present** counts as one class and **Late** counts as 0.5 by default; administrators can set late credit from 0 to 1.

After attendance, assignment, or assessment changes, the backend recalculates a 0–100 academic intervention risk score from attendance (40% default), assignment completion (25%), academic performance (25%), and LMS activity (10%). Missing components are omitted and the remaining weights are renormalized; data completeness is shown with the score. Default categories are Low Risk (0–29), Medium Risk (30–59), High Risk (60–79), and Critical Risk (80–100). The saved component snapshot and score history drive the profile insight and risk trend chart. No score is stored when no risk component data exists. Daily attendance percentages use recorded sessions only, and cancelled classes/no-session dates are not counted.

### MongoDB attendance migration

Before deploying this version against an existing MongoDB database, run the migration once from the server directory with the target `MONGODB_URI` configured:

```powershell
cd server
npm.cmd run migrate:attendance
```

The migration checks for duplicate student/subject/date groups before replacing the older unique index that included `period`. If duplicates are reported, review and correct those records before rerunning it. No new environment variables are required; persistent operation still requires the existing `MONGODB_URI` and `JWT_SECRET`. Without MongoDB, the same attendance and risk workflows operate in demo memory mode and are not persistent after restart.

### Validation and manual checks

Run backend tests and the client production build from the repository root:

```powershell
npm.cmd test
npm.cmd --prefix client run build
```

For an authenticated UI check, open a student profile as Admin or authorized Faculty → **Attendance** → **Manage Attendance**. Select the date/class/semester/subject, mark all present and change one student, then save. Reload the page and verify records are still present in MongoDB mode. Reopen the prior date, correct an absence to Present, and verify the edit is saved once and risk score history updates. Mark a later session Absent to verify risk rises, then correct it to Present to verify risk falls when the other factors are unchanged. Sign in as a student to confirm the attendance tab is read-only and requesting another student's attendance/risk endpoints is denied.

The existing marks alert behavior remains unchanged: marks percentages are calculated from obtained marks divided by maximum marks, and the default labels are **Good** (80% or above), **Needs Improvement** (at/above the configurable cutoff), and **At Risk** (below the cutoff).

Example:

```env
PORT=5000
MONGODB_URI=mongodb://127.0.0.1:27017/campuspulse
JWT_SECRET=replace_with_a_random_secret_at_least_32_characters
AI_API_KEY=
AI_BASE_URL=https://api.openai.com/v1/chat/completions
AI_MODEL=gpt-4o-mini
CLIENT_URL=http://localhost:5173
```

`AI_API_KEY` enables the conversational web-building copilot. Keep the key on the server and configure an OpenAI-compatible chat completions endpoint with `AI_BASE_URL` and `AI_MODEL` when using another provider. Without a key, CampusPulse's built-in role-aware tools remain available, but generative web-building responses are disabled.

## Seed command

In the server directory:

```bash
npm run seed
```

This connects to MongoDB, seeds demo users, and seeds demo students only if the Students collection is empty. It requires `MONGODB_URI` in `server/.env`; it does not modify existing student records.

## Demo login accounts

- Admin: admin@campuspulse.ai / Admin@123
- Mentor: faculty@campuspulse.ai / Faculty@123
- Student: ramya@gmail.com / 12345678

## Demo flow

1. Log in as admin.
2. Open the dashboard to review KPI cards and risk distribution.
3. Navigate to Risk Intelligence.
4. Open a high-risk student profile.
5. Generate AI insight and create an intervention.
6. Open Segments and Analytics pages to review cohort insights.

## CampusPulse Early Success Alert

The student dashboard's **CampusPulse Early Success Alert** checks available attendance, assessment marks, pending or missed assignments, learning activity, feedback, and upcoming exams for combined early-warning patterns. It explains the recorded signals, creates a supportive student notification for Attention or Urgent Support patterns, and builds a short seven-day plan around the available subject and coursework data. Admin users can open the feature for a student from that student's profile.

Alert checks and notification history are retained in MongoDB when configured, or in the in-memory demo store for the running session. Students are rechecked on dashboard entry and every five minutes while signed in; authorized staff checks run when they open a student's alert center. Daily snapshots support progress comparisons. The combined indicator is a support aid, not a diagnosis or guaranteed prediction. Missing data is left unknown, and the system recommends a human mentor check-in when several signals warrant additional support.

## Intervention feedback loop

Admin users can use **Interventions → Completion & outcome feedback** to record status, action taken, student response, before/after support-indicator scores, a reviewed outcome, and follow-up notes. Only completed interventions with a reviewed outcome feed back into future recommendations. Similar recorded improvement can support continuing a helpful action; no change or a need for more support prompts a different or additional action to be discussed with the student. Trends are observational, not proof that an intervention caused an outcome.

## Success Score formula

Success Score =
- Academic Performance × 0.30
- Attendance × 0.20
- LMS Activity × 0.15
- Assignment Completion × 0.10
- Placement Readiness × 0.15
- Engagement × 0.05
- Skills × 0.05

All metrics are normalized to 0–100, then weighted and combined.

## Risk Intelligence

The Risk Intelligence page calculates a 0–100 support indicator from the measured academic (25%), attendance (18%), assignment (14%), engagement (8%), learning (12%), administrative/fee (3%), performance-trend (8%), and placement-readiness (12%) signals. Weights are renormalized over categories with available data; missing categories stay unknown and are excluded rather than treated as risk. The resulting levels are 0–30 Low, 31–60 Moderate, 61–80 High, and 81–100 Critical. Each assessment explains its contributing factors, detected signal combinations, and practical support recommendations.

Scores are support indicators, not diagnoses or permanent student labels. Mentors remain responsible for important decisions and interventions with their assigned students. Student accounts see only their own supportive progress view; Admin can review student assessments and department summaries. Administrative risk is included only when a fee record exists and is not included in faculty/mentor views. Risk projections require dated assessment history and are explicitly presented as cautious estimates, not certain outcomes. Use **Record assessment** to save a dated snapshot; the first snapshot establishes a baseline for later comparison.

The **Predictive Student Success** dashboard is available to Admin and authorized Faculty/Mentor accounts. It provides branch/year aggregates to Admin and only assigned active students to Faculty/Mentors; staff database queries are restricted to assigned student IDs before academic or intervention data is fetched. Its 30-day possible outcomes use transparent rules over available academic, attendance, assignment, learning, and placement signals; this is not a trained model, calibrated probability, or guarantee. No prediction-confidence value is shown because the baseline has not been evaluated against held-out historical outcomes. Signal coverage and dated assessment history are reported separately. Subject-level marks within 10 percentage points of the configured pass mark are surfaced separately so a weak subject is not concealed by an average. Configure `STUDENT_PASS_MARK_PERCENT` to the institution's verified policy; the 40% default is only a screening assumption and must not replace official grading rules. Subject scores used for pass-mark screening must be percentages or raw marks with a recorded maximum. Staff can create scoped intervention plans with an owner, target date, and measurable success indicator, record follow-up status and reviewer identity, and save dated reassessments for trend review. Confirm context with the student before taking action. Demo mode is labeled and uses synthetic records; do not use those records for real decisions.

Round 2 judge materials: [decision-intelligence API reference](./docs/DECISION_INTELLIGENCE_API.md) and [end-to-end demo script](./docs/DECISION_INTELLIGENCE_DEMO.md).

Student accounts accessing **My Performance** are restricted to their own profile and performance record. Mentor requests for student overviews, scores, and insights are constrained to the mentor's current assignments on the server. Changing a student ID in a URL does not bypass either check. Campus-wide directory search remains Admin-only; mentor search results are limited to assigned students.

The student profile's **Skills** tab shows the overall skills score and itemized technical and soft-skill ratings from the latest `SkillAssessment` record. Students can add, edit, and remove up to 25 self-reported technical skills and 25 self-reported soft skills, and rate each from 0 to 100. These are identified as self-reported, not verified assessments. When no skills have been entered, the profile states that they have not been recorded rather than generating sample ratings.

Interventions store an owner, action, student response, follow-up date, status, notes, and optional before/after risk scores. Effectiveness is measured from the recorded score change and should not be interpreted as proof that an intervention caused it. Outcomes are available for human review; they do not silently train or alter the scoring formula.

## AI recommendation logic

`aiRecommendationService.js` generates an explainable recommendation by combining actual performance metrics, attendance, academic trend, placement readiness, LMS activity, and engagement signals. If an external AI key is not configured, CampusPulse uses deterministic rule-based recommendations.

## Proactive student-success workflow

The current implementation uses transparent, rule-based early-warning signals over available marks, subject trends, attendance, assignments, LMS activity, student/faculty feedback, exams, and intervention records. It shows missing inputs as unavailable and describes future outcomes conditionally. These signals are not a trained outcome model and do not provide validated probabilities or accuracy claims.

Faculty and mentors can open **Check & send support alert** for a currently assigned student. The alert center shows the current indicator, contributing evidence, conditional outlook, suggested support, seven-day study plan, progress snapshots, and intervention history. Authorized staff can confirm an alert, record a correction, or dismiss it with a required explanation. Review metadata (decision, note, reviewer, and time) is stored with the alert; a correction note does not edit source student data. Mentors and Faculty can only review notifications for their assigned active students; Admin can review any student notification.

### Implementation phases and current status

1. **Audit and preserve existing data:** completed. Existing React, Express, MongoDB models, role-scoped student access, early-success checks, intervention tracking, and demo data are reused.
2. **Explainable proactive screening and action:** working. Current rules surface signal reasons, conditional possible next outcomes, and supportive next steps. The Analytics Copilot also gives cohort answers with their evidence and explicitly avoids claiming model validation.
3. **Faculty review and follow-up:** alert confirmation/correction/dismissal with an audit trail is working. Existing intervention records track owner, action, status, follow-up, and observational outcomes. Review outcomes are not interpreted as causation.
4. **Validated outcome prediction:** future enhancement. First collect authorized, time-stamped historical features and outcomes (such as upcoming-semester failure/backlog or placement readiness), define prediction horizons, prevent leakage, use temporal train/test validation and simple baselines, then report appropriate metrics and calibration. Until then, do not describe screening rules as trained predictions or attach statistical confidence.
5. **Institution isolation and deployment hardening:** before production use, add an explicit institution/tenant boundary to users, students, and related records, enforce it in every query, and test cross-tenant denial. Existing student and mentor assignment checks are not a substitute for tenant isolation.

## Student study planner

The chatbot's **What Should I Study Now?** tool prioritizes available subject marks, low subject-understanding feedback, upcoming exams, pending assignments, and the student's prior task check-ins. It returns one focused topic that fits the selected study window and links the relevant uploaded question paper. Study outcomes are saved in MongoDB when configured (or in memory in demo mode) and used to choose the next task. The student schema accepts `subjectScores`, `upcomingExams` (`subject`, `examDate`, optional `topics`), and `pendingAssignments` (`subject`, `title`, optional `topic`, `dueDate`, `estimatedMinutes`, `status`) so this can use real course data when supplied. If those data are not available, the planner clearly falls back to a question-paper review rather than inventing exam dates or marks.

The student sidebar's **Study Plan** calculates per-subject weekly and daily time using a transparent 0–100 priority score: performance gap (35%), trend (15%), exam urgency (20%), backlog (10%), difficulty (10%), internal/assignment/quiz weakness (5%), and low attendance when marks are weak (5%). Performance bands and weekly time ranges are defined in `server/services/studyTimeCalculator.js`. Declining marks and near exams increase study time; improved marks, high scores, and available-time limits adjust it down. The saved plan includes practice/revision splits, placement preparation, daily timetable blocks, and remaining available time. Students can enter or update their subject marks and course context directly; persisted plans are student-scoped in MongoDB when configured and in memory in demo mode otherwise.

## Notes

## Role-based student management

CampusPulse uses three application roles: **Admin**, **Mentor**, and **Student**. Existing **Faculty** accounts are treated as Mentors so current logins continue to work. Passwords for new accounts are bcrypt-hashed; legacy plaintext demo/database passwords are upgraded to hashes on login or during database initialization. Active status is checked on API requests, so deactivation revokes existing sessions.

Admins use **Students**, **Mentors**, **Events**, **Announcements**, and **Settings** to create and edit accounts, assign or reassign students, deactivate accounts, publish audience-targeted events, post announcements, and manage chatbot availability and its welcome message. Mentor permissions for notes, feedback, tasks, messages, sessions, and goals are configurable and enforced by the API. New accounts receive a generated temporary password when one is not provided; the password is shown only in the creation response and should be shared securely. Deactivating an assigned mentor is blocked until their active students are reassigned or deactivated.

Mentor APIs and dashboards only return currently assigned students. Mentors can create student accounts from My Students; the new student is automatically assigned to the creating mentor, and a generated temporary password is shown once when no password is supplied. Mentors can filter assigned rosters and submitted subject feedback by department, view student IDs, contact details, attendance, performance, risk level, and profiles, and add notes, feedback, tasks, goals, messages, and sessions. Mentors can send a subject-feedback form request to assigned students in one department; students receive an in-app notification and can submit feedback only for subjects recorded on their own student profile. Mentors can prepare a risk alert using a student's recorded attendance and support indicators or write a custom alert; sending it creates an in-app notification for that student. From My Students or an assigned student's profile, a mentor can also run the Early Success Alert check; when recorded signals warrant support, the student and assigned mentor receive an in-app alert. Students can view only their own profile, assigned mentor, mentoring history, events, announcements, and notifications; they can message their mentor and record progress on their tasks and goals. Mentor changes update the relationship immediately and create notifications for the student, the old mentor, and the new mentor.

Events and announcements are stored in MongoDB when configured and in memory for demo mode. Only published events targeted to a student's department/year appear on their event page. Students can register for published events before the registration deadline and cancel their own registration; Admin can review each event's active registration list. Announcements can target all students, a department, a year, or a mentor group and create matching in-app notifications. Notification updates are polled every 30 seconds while a user is signed in.

The APIs are mounted at `/api/management`. Authentication is enforced on every endpoint; resource checks are performed server-side, and campus-wide analytics, risk assessments, interventions, and account management are Admin-only. Set `JWT_SECRET` in `server/.env` before starting the API; there is no built-in signing-secret fallback.

This project can run with in-memory demo data or connect to MongoDB when configured. Demo-mode changes last only for the lifetime of the server process. It does not silently fall back to demo data when a configured MongoDB connection fails.
