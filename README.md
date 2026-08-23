# TutorAI — AI-Powered Tutoring Platform (MVP)

A focused, AI-automated online tutoring platform built with **Python (Flask)**.
Same "easy version" design language as your school system, but scoped to just
the tutoring workflow: teachers, students, live classes, AI-marked assignments,
a digital library, and an AI tutor on both sides.

## What's included

**Three roles:** Admin, Teacher, Student — each with their own dashboard.

**Student journey:** sign up → choose a category (Pre-Primary / Primary /
Junior School / Senior School-High School) → choose a teacher tier (Standard
KES 1,200/mo, National KES 3,000/mo, International KES 6,000/mo) → pay →
get matched to a teacher (auto-matched if one is available, otherwise the
admin assigns one).

**Teacher journey:** sign up → activate a license — either pay **KES 1,500
for 3 years** and go live immediately, or start **free** and use the system
with your own students once the admin registers them under you.

**Once inside, a teacher can only ever see her own students**, and can:
- Send assignments — upload a PDF/photo and let AI convert it into questions
  students can tick/answer on their phone or laptop (students can also just
  download the original file instead)
- Mark submissions with AI — either auto-marked online answers, or a photo/PDF
  of handwritten work, using a "Quick AI Marking" tool
- Schedule live classes (paste your Zoom/Meet link) and upload class recordings
- Upload books to the library
- Use an AI teaching assistant to draft questions, rubrics, or explanations

**Admin can:**
- Add teachers/students directly and assign students to a specific teacher
- Approve free-tier teachers so their own students can be registered
- View all payments and the whole digital library

## Setup

```bash
cd tutorai
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env
# then edit .env:
#  - set a real SECRET_KEY
#  - add your ANTHROPIC_API_KEY to turn on the AI Tutor / AI marking / AI
#    question generation (get one at https://console.anthropic.com)
#  - optionally change prices, license fee, or the admin login

python3 app.py
```

Visit **http://localhost:5000**. A default admin account is auto-created on
first run using the credentials in `.env` (default: `admin@tutorai.local` /
`Admin@12345` — **change this after first login**).

## Deploy it live (one click, free)

This repo is ready for **Render** — it includes `render.yaml`, a `Procfile`,
and `runtime.txt`, so Render (or Railway/Heroku, which also read a `Procfile`)
can build and run it with no manual server config.

**Render (recommended, has a free tier):**

1. Push this `tutorai` folder to a **new GitHub repo** (make sure `render.yaml`
   ends up at the repo root, not nested inside another folder).
2. Go to **[render.com](https://render.com)** → sign up/log in → **New +** → **Blueprint**.
3. Connect the GitHub repo. Render will read `render.yaml` automatically and
   show you a form for the two secret values it needs:
   - `ANTHROPIC_API_KEY` — your key from console.anthropic.com
   - `ADMIN_PASSWORD` — the password for the default admin login
4. Click **Apply**. Render builds and deploys automatically — you'll get a
   live URL like `https://tutorai-xxxx.onrender.com` in a few minutes.
5. Log in at `/login` with `admin@tutorai.local` and the password you set.

**Free-tier caveat:** Render's free web service has no persistent disk, so
the SQLite database (and anything uploaded — assignments, recordings,
library books) resets whenever the service redeploys or spins down from
inactivity. That's fine for demoing the platform. Before real students/
teachers rely on it, either:
- upgrade to a paid Render instance and attach a persistent disk, or
- switch `DATABASE_URL` to a managed Postgres database (Render offers a free
  Postgres instance too) and move file uploads to S3/Cloudinary/similar.

**Railway / Heroku:** both also read `Procfile` — create a new project from
the repo, set the same environment variables from `.env.example` in their
dashboard, and deploy. No code changes needed.

**Running it yourself without any hosting platform (e.g. on a VPS):**
```bash
pip install -r requirements.txt
gunicorn app:app --bind 0.0.0.0:8000
```

## Notes on payments

This MVP records M-Pesa payments as completed instantly, so you can test the
whole registration → payment → dashboard flow without live credentials. To go
live, swap the payment blocks in `blueprints/auth.py` (`pay_tuition` and
`teacher_license`) for a real Safaricom **Daraja STK Push** integration, and
only mark a `Payment` as `completed` once the callback confirms success.

## Notes on AI features

Everything AI-related lives in `ai_service.py` and calls the Anthropic API.
Without an `ANTHROPIC_API_KEY` set, the rest of the app still works — the AI
features just say they're not configured yet instead of erroring.

- **AI Tutor** — chat for students (patient, level-appropriate explanations)
  and a separate assistant mode for teachers (setting questions, rubrics)
- **Assignment → interactive quiz** — reads an uploaded PDF/image (via
  PyMuPDF + vision) and returns structured questions (MCQ, true/false,
  short answer) with an answer key where determinable
- **AI marking** — either instant auto-marking of online tick/typed answers
  against the extracted answer key, or vision-based marking of a photo/PDF
  of a student's handwritten work against a marking guide the teacher supplies

## Project structure

```
tutorai/
├── app.py                 # app factory, seeds the admin account
├── config.py               # pricing, categories, upload rules, env config
├── extensions.py           # SQLAlchemy instance
├── models.py                # User, TeacherProfile, StudentProfile, Payment,
│                             Assignment, Submission, LiveClass, Recording,
│                             LibraryBook, TutorMessage
├── ai_service.py           # all Anthropic API calls live here
├── helpers.py               # auth decorators, file upload helpers
├── blueprints/
│   ├── auth.py              # login + the student/teacher registration flows
│   ├── admin.py
│   ├── teacher.py
│   ├── student.py
│   └── files.py             # serves uploaded files to logged-in users only
├── templates/                # Jinja2 templates (navy/green branded theme)
└── static/
    ├── css/style.css
    └── uploads/               # assignments, submissions, recordings, library
```

## Editing prices or categories

Everything lives in `config.py`:

```python
TIER_PRICES = {"standard": 1200, "national": 3000, "international": 6000}
TEACHER_LICENSE_FEE = 1500
TEACHER_LICENSE_YEARS = 3
CATEGORIES = {
    "pre_primary": "Pre-Primary (PP1 - PP2)",
    "primary": "Primary School (Grade 1 - 6)",
    "junior_school": "Junior School (Grade 7 - 9)",
    "senior_school": "Senior School / High School (Grade 10 - 12)",
}
```
(You mentioned "4 different" categories — Kenya's CBC structure has four
levels, so Pre-Primary was added alongside Primary, Junior School and
Senior/High School. Rename or remove any of them here if you'd rather have
exactly three.)

## Suggested next steps for production

1. Swap SQLite for PostgreSQL (`DATABASE_URL` in `.env`) once you have real traffic.
2. Wire up the real Daraja STK Push API for payments.
3. Add email/SMS notifications (new assignment, live class reminder, marked result).
4. Move uploaded files to S3/Cloud Storage instead of local disk.
5. Add automated recurring billing/expiry checks for monthly tuition and the
   3-year teacher license.
