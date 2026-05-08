# SmartHire – AI-Powered Resume Screening

Full-stack application using **Spring Boot + Spring AI + Claude (Anthropic)** for intelligent resume screening.

---

## Tech Stack

| Layer       | Technology                          |
|-------------|-------------------------------------|
| Backend     | Java 17, Spring Boot 3.2, Spring AI |
| AI          | Anthropic Claude (via Spring AI)    |
| Database    | MySQL (prod) / H2 (dev)             |
| Frontend    | React 18                            |
| PDF Parsing | Apache PDFBox                       |

---

## Project Structure

```
smarthire/
├── backend/                        ← Spring Boot app
│   ├── pom.xml
│   └── src/main/java/com/smarthire/
│       ├── SmartHireApplication.java
│       ├── config/AppConfig.java          ← CORS + ChatClient bean
│       ├── controller/
│       │   ├── AIScreeningController.java  ← /api/ai/*
│       │   ├── JobController.java          ← /api/jobs/*
│       │   └── ApplicationController.java  ← /api/applications/*
│       ├── service/
│       │   ├── AIScreeningService.java     ← Core Spring AI logic
│       │   ├── JobService.java
│       │   ├── ApplicationService.java     ← PDF parsing
│       │   └── ScreeningService.java       ← Leaderboard + stats
│       ├── model/                          ← JPA entities
│       ├── repository/                     ← Spring Data repos
│       └── dto/                            ← Request/Response DTOs
└── frontend/                       ← React app
    ├── package.json
    └── src/
        ├── App.js
        ├── services/api.js                 ← All API calls
        ├── components/
        │   ├── UI.jsx                      ← Shared components
        │   └── DetailPanel.jsx
        └── pages/
            └── ScreeningLeaderboard.jsx    ← Main page
```

---

## Setup & Run

### 1. Prerequisites

- Java 17+
- Maven 3.8+
- MySQL 8+ (or use H2 for dev)
- Node.js 18+
- Anthropic API key → https://console.anthropic.com

---

### 2. Backend Setup

#### Set your API key

```bash
export ANTHROPIC_API_KEY=sk-ant-your-key-here
```

Or edit `backend/src/main/resources/application.yml`:

```yaml
spring:
  ai:
    anthropic:
      api-key: sk-ant-your-key-here
```

#### Configure database

Edit `application.yml` with your MySQL credentials:

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/smarthire?createDatabaseIfNotExist=true
    username: root
    password: yourpassword
```

**OR** run with H2 (no MySQL needed) for local dev:

```bash
cd backend
DEBUG=false ./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

The `dev` profile uses an in-memory H2 database and seeds a demo job with six candidates. If `ANTHROPIC_API_KEY` is not configured, screening uses a local heuristic fallback so the full frontend-to-backend flow still works.

#### Start backend (MySQL):

```bash
cd backend
mvn spring-boot:run
```

Backend runs on: http://localhost:8080

---

### 3. Frontend Setup

```bash
cd frontend
npm install
npm start
```

Frontend runs on: http://localhost:3000  
(proxied to backend via `"proxy": "http://localhost:8080"` in package.json)

---

## API Reference

### AI Screening Endpoints

| Method | Endpoint                          | Description                              |
|--------|-----------------------------------|------------------------------------------|
| POST   | `/api/ai/screen`                  | Screen one candidate (raw text payload)  |
| POST   | `/api/ai/screen/{jobId}/{candId}` | Screen by DB IDs                         |
| POST   | `/api/ai/screen-all/{jobId}`      | Bulk screen all applicants               |
| GET    | `/api/ai/leaderboard/{jobId}`     | Ranked results (`?status=SHORTLISTED`)   |
| GET    | `/api/ai/stats/{jobId}`           | Stats: counts + avg score                |

### Jobs Endpoints

| Method | Endpoint           | Description       |
|--------|--------------------|-------------------|
| POST   | `/api/jobs`        | Create a job      |
| GET    | `/api/jobs`        | List all jobs     |
| GET    | `/api/jobs/{id}`   | Get job by ID     |
| PUT    | `/api/jobs/{id}`   | Update job        |
| PATCH  | `/api/jobs/{id}/close` | Close job    |

### Applications Endpoints

| Method | Endpoint                                | Description               |
|--------|-----------------------------------------|---------------------------|
| POST   | `/api/applications/apply`               | Submit resume (multipart) |
| GET    | `/api/applications/job/{jobId}/candidates` | List candidates        |

---

## Example: Screen a Candidate

```bash
curl -X POST http://localhost:8080/api/ai/screen \
  -H "Content-Type: application/json" \
  -d '{
    "jobId": 1,
    "candidateId": 1,
    "jobTitle": "Senior Backend Engineer",
    "requirements": "5+ years Java, Spring Boot, Spring AI, MySQL, Docker",
    "resumeText": "6 years Java Spring Boot. Microservices on AWS. Spring AI integrations. MySQL, Kubernetes."
  }'
```

**Response:**

```json
{
  "candidateId": 1,
  "jobId": 1,
  "jobTitle": "Senior Backend Engineer",
  "score": 91,
  "recommendation": "SHORTLISTED",
  "reasoning": "Strong alignment with all core requirements. Extensive Spring Boot experience and hands-on Spring AI usage are a direct match. Cloud and Kubernetes expertise adds significant value.",
  "strengths": ["Spring Boot & Spring AI", "AWS microservices", "Kubernetes & Docker"],
  "gaps": ["No explicit MySQL mention"]
}
```

---

## Example: Submit a Resume via Multipart

```bash
curl -X POST http://localhost:8080/api/applications/apply \
  -F "name=Priya Sharma" \
  -F "email=priya@email.com" \
  -F "jobId=1" \
  -F "resume=@/path/to/resume.pdf"
```

---

## Score Logic

| Score  | Status      | Meaning                          |
|--------|-------------|----------------------------------|
| 75–100 | SHORTLISTED | Strong fit — proceed to interview |
| 50–74  | REVIEW      | Partial fit — human review needed |
| 0–49   | REJECTED    | Poor fit — does not meet requirements |

---

## Deploying

### Frontend on Vercel

Deploy the `frontend/` directory as a Vercel project.

- Framework preset: `Create React App`
- Root directory: `frontend`
- Build command: `npm run build`
- Output directory: `build`

Set this environment variable in Vercel:

```bash
REACT_APP_API_BASE_URL=https://your-backend-url
```

The frontend accepts either the backend root URL or a URL ending in `/api`. It already falls back to `/api` for local development, so this env var is mainly for production hosting.

### Backend hosting

The Spring Boot backend is **not a good fit for Vercel** in its current form because it depends on:

- Java/Spring Boot runtime
- persistent MySQL connectivity
- multipart file upload handling
- long-lived backend application behavior

Host the backend on a Java-friendly platform such as Render, Railway, or an EC2/VPS, then point the Vercel frontend at that URL.

Set these backend environment variables on your backend host:

```bash
SPRING_DATASOURCE_URL=jdbc:mysql://<host>:3306/smarthire?createDatabaseIfNotExist=true&useSSL=false&serverTimezone=UTC
SPRING_DATASOURCE_USERNAME=<db-user>
SPRING_DATASOURCE_PASSWORD=<db-password>
ANTHROPIC_API_KEY=<your-anthropic-key>
APP_CORS_ALLOWED_ORIGINS=https://your-vercel-app.vercel.app
```

If you later attach a custom frontend domain, add that domain to `APP_CORS_ALLOWED_ORIGINS` too, separated by commas.
