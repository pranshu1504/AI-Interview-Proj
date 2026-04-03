# AI Interview Prep Platform

An AI-powered interview preparation tool that analyzes your resume and a target job description to generate a personalized interview strategy — including technical and behavioral questions, skill gap analysis, a match score, and a day-wise preparation roadmap. Also generates a tailored, ATS-optimized resume PDF on demand.

---

## Features

- **Resume + JD analysis** — upload your resume PDF and paste a job description to generate a full interview report
- **AI-generated report** — technical questions, behavioral questions, interviewer intent, and model answers for each
- **Match score** — 0–100 score indicating how well your profile aligns with the role
- **Skill gap analysis** — identifies missing skills with severity ratings (low / medium / high)
- **Preparation roadmap** — day-wise plan with focused tasks to close gaps before the interview
- **Resume PDF generation** — Gemini rewrites your resume tailored to the job description; Puppeteer renders it to a downloadable PDF
- **Auth** — cookie-based JWT authentication with token blacklisting on logout

---

## Tech Stack

**Frontend**
- React 19 (Vite)
- React Router v7
- Axios
- SCSS

**Backend**
- Node.js / Express 5
- MongoDB + Mongoose
- Google Gemini API (`@google/genai`)
- Zod + `zod-to-json-schema` — structured AI output enforcement
- Puppeteer — server-side HTML to PDF rendering
- Multer — in-memory file upload handling
- pdf-parse — PDF text extraction
- bcryptjs + jsonwebtoken — auth

---

## Project Structure

```
interview-ai-yt-main/
├── Backend/
│   ├── server.js
│   └── src/
│       ├── app.js
│       ├── config/
│       │   └── database.js
│       ├── controllers/
│       │   ├── auth.controller.js
│       │   └── interview.controller.js
│       ├── middlewares/
│       │   ├── auth.middleware.js
│       │   └── file.middleware.js
│       ├── models/
│       │   ├── user.model.js
│       │   ├── interviewReport.model.js
│       │   └── blacklist.model.js
│       ├── routes/
│       │   ├── auth.routes.js
│       │   └── interview.routes.js
│       └── services/
│           └── ai.service.js
└── Frontend/
    └── src/
        ├── App.jsx
        ├── app.routes.jsx
        ├── features/
        │   ├── auth/
        │   │   ├── auth.context.jsx
        │   │   ├── hooks/useAuth.js
        │   │   ├── services/auth.api.js
        │   │   ├── components/Protected.jsx
        │   │   └── pages/Login.jsx, Register.jsx
        │   └── interview/
        │       ├── interview.context.jsx
        │       ├── hooks/useInterview.js
        │       ├── services/interview.api.js
        │       └── pages/Home.jsx, Interview.jsx
        └── main.jsx
```

---

## Getting Started

### Prerequisites

- Node.js v18+
- MongoDB (local or Atlas)
- Google Gemini API key — get one at [aistudio.google.com](https://aistudio.google.com)

### Backend setup

```bash
cd Backend
npm install
```

Create a `.env` file in the `Backend/` directory:

```env
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret
GOOGLE_GENAI_API_KEY=your_gemini_api_key
```

Start the server:

```bash
npm run dev
```

Server runs on `http://localhost:3000`.

### Frontend setup

```bash
cd Frontend
npm install
npm run dev
```

Frontend runs on `http://localhost:5173`.

---

## API Reference

### Auth

| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| POST | `/api/auth/register` | Public | Register with username, email, password |
| POST | `/api/auth/login` | Public | Login with email, password |
| GET | `/api/auth/logout` | Public | Clears cookie, blacklists token |
| GET | `/api/auth/get-me` | Private | Returns current logged-in user |

### Interview

| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| POST | `/api/interview/` | Private | Generate a new interview report (multipart: resume PDF + text fields) |
| GET | `/api/interview/` | Private | Get all reports for the logged-in user |
| GET | `/api/interview/report/:interviewId` | Private | Get a specific report by ID |
| POST | `/api/interview/resume/pdf/:interviewReportId` | Private | Generate and download a tailored resume PDF |

---

## How It Works

### Report generation

1. User uploads a resume PDF and pastes a job description on the Home page
2. Frontend sends a `multipart/form-data` POST to the backend
3. Multer stores the file in memory (`req.file.buffer`) — no disk writes
4. `pdf-parse` extracts plain text from the PDF buffer
5. The text, job description, and self-description are sent to Gemini with a Zod-enforced response schema
6. Gemini returns a structured JSON object — match score, questions, skill gaps, prep plan
7. The report is saved to MongoDB and returned to the frontend
8. User is redirected to the interview report page

### Structured AI output

Instead of prompting Gemini and parsing free-form text, the project defines the expected response shape as a Zod schema, converts it to JSON Schema via `zod-to-json-schema`, and passes it to Gemini's `responseSchema` config. This forces Gemini to return valid, typed JSON that maps directly onto the Mongoose model.

### Resume PDF generation

1. User clicks "Download Resume" on the report page
2. Backend fetches the stored report (resume text + JD + self-description)
3. Gemini is prompted to generate an ATS-optimized HTML resume tailored to the job description
4. Puppeteer launches a headless Chromium instance, renders the HTML, and exports it as an A4 PDF buffer
5. Buffer is streamed back with `Content-Disposition: attachment` — triggers a browser download

### Auth

- Passwords hashed with `bcryptjs` (salt rounds: 10)
- JWT signed with `JWT_SECRET`, expires in 1 day, stored as an HTTP cookie
- On logout, the token is written to a `blacklistTokens` MongoDB collection; all protected routes check this collection before accepting a token

---

## Known Limitations

- No frontend error messages on failed API calls — errors are only logged to the console
- JWT cookie is not `httpOnly` — susceptible to XSS in its current form
- Token blacklist has no TTL index — grows indefinitely (fix: add `expireAfterSeconds: 86400`)
- Puppeteer launches a new Chromium instance per PDF request — not suitable for high traffic without a persistent browser pool
- No input validation on incoming request bodies beyond presence checks
