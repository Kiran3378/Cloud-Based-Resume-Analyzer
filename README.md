# Align — Cloud-Based Resume Analyzer

Align is a full-stack MCA major-project starter with a responsive website, an Android application, an authenticated API, configurable SQL persistence, S3-compatible cloud object storage, and cloud AI analysis.

## User journey

1. Register with an email address and password, or sign in.
2. Create and save a resume using the Modern, Classic, or Minimal template; or upload an existing PDF, DOCX, or TXT resume.
3. Paste a job description and run a cloud-model comparison.
4. Review the match score, skills that align, skills to develop, extracted resume keywords, and practical feedback.
5. View previous scores in the account's analysis history.
6. Ask the Align chatbot for resume-writing, job-application, or career guidance, optionally using one of your saved resumes as context.

Created resumes and score history belong to the signed-in account. Uploaded resumes are stored in a private S3-compatible bucket and can be reused from either client. The database stores account records, created resume data, upload metadata, and analysis results. Resume text and the target job description are sent to the configured cloud model for analysis; chatbot messages are sent to the model but are not persisted.

## Backend and website: local setup

Use Python 3.10 or newer. From the project root, install the API dependencies:

```powershell
py -m pip install -r requirements.txt
```

Configure a cloud-model API key and a stable, secret signing key. `AUTH_TOKEN_SECRET` must be at least 32 characters; generate a new value rather than reusing the example:

```powershell
$env:OPENAI_API_KEY = "your-cloud-model-api-key"
$env:OPENAI_MODEL = "gpt-4o-mini"
$env:OPENAI_BASE_URL = "https://api.openai.com/v1"
$env:AUTH_TOKEN_SECRET = "replace-with-a-random-secret-at-least-32-characters"
```

For local development, the app uses SQLite by default. For a managed PostgreSQL database, set `DATABASE_URL`:

```powershell
$env:DATABASE_URL = "postgresql+psycopg://USER:PASSWORD@HOST:5432/DATABASE"
```

Configure the S3-compatible bucket for resume files. Use the endpoint and credentials from the selected storage provider; AWS S3 can instead use the standard AWS environment variables or the host's workload identity.

```powershell
$env:OBJECT_STORAGE_BUCKET = "your-private-resume-bucket"
$env:OBJECT_STORAGE_REGION = "your-region"
$env:OBJECT_STORAGE_ENDPOINT_URL = "https://your-s3-compatible-endpoint"
$env:OBJECT_STORAGE_ACCESS_KEY_ID = "your-storage-access-key"
$env:OBJECT_STORAGE_SECRET_ACCESS_KEY = "your-storage-secret-key"
```

For AWS S3, omit `OBJECT_STORAGE_ENDPOINT_URL` and configure AWS credentials or an instance/task role. Object keys are scoped to the owning account, and the app does not make uploaded objects public.

`OPENAI_BASE_URL` and `OPENAI_MODEL` can be changed to another OpenAI-compatible chat-completions service. Keep the provider API key and token secret on the server; never put either in the website or mobile app.

Start the backend and website:

```powershell
py -m uvicorn app:app --reload
```

Open http://127.0.0.1:8000. The health check is at `/api/health`, and interactive API documentation is at `/docs`.

Without `DATABASE_URL`, the SQLite database defaults to `data/align.sqlite3`; set `DATABASE_PATH` to move that local file. In production, configure a managed PostgreSQL `DATABASE_URL` and S3-compatible object-storage settings. If storage is not configured, resume creation and account flows still work, but uploaded-file storage returns an explicit configuration error.

Run the backend tests with:

```powershell
py -m pip install -r requirements-dev.txt
py -m unittest discover -s tests -v
```

## Android app

Install Node.js and Android Studio with an Android SDK/emulator. From the project root:

```powershell
cd mobile
npm install
```

For local emulator development, run the backend in a separate terminal, then allow HTTP only for this local development build:

```powershell
$env:LOCAL_API_HTTP = "true"
$env:EXPO_PUBLIC_API_URL = "http://10.0.2.2:8000"
npm run android
```

For a physical Android device, bind the API server to `0.0.0.0`, use the computer's reachable LAN IP in `EXPO_PUBLIC_API_URL`, and allow the connection through the computer's firewall. For a deployed backend, set `EXPO_PUBLIC_API_URL` to its HTTPS URL and leave `LOCAL_API_HTTP` unset.

The app uses Android's document picker for resume uploads and Android secure storage for its sign-in token. For an installable Android preview APK, install EAS CLI, sign in to Expo, and run `eas build --platform android --profile preview` from `mobile`. The production EAS profile creates an Android App Bundle. Configure `EXPO_PUBLIC_API_URL` in the Expo/EAS build environment for deployed builds.

## Deploy the website and API

The API serves the website and should be deployed to a public HTTPS endpoint. A Dockerfile is included for container-based cloud platforms. Configure these server-side environment variables on the host:

- `OPENAI_API_KEY` — cloud model API key.
- `AUTH_TOKEN_SECRET` — random secret of at least 32 characters; keep it unchanged across restarts.
- `DATABASE_URL` — managed PostgreSQL connection string. The Docker image can instead use its persistent `/data` volume and `DATABASE_PATH=/data/align.sqlite3` for a demo deployment.
- `OBJECT_STORAGE_BUCKET`, `OBJECT_STORAGE_REGION`, `OBJECT_STORAGE_ENDPOINT_URL`, `OBJECT_STORAGE_ACCESS_KEY_ID`, `OBJECT_STORAGE_SECRET_ACCESS_KEY` — private S3-compatible storage settings. The endpoint is optional for AWS S3.
- `OPENAI_MODEL` and `OPENAI_BASE_URL` — optional model/provider overrides.

For production, use managed PostgreSQL and private object storage rather than a container-local SQLite file. If using the SQLite demo setup, mount persistent storage at `/data`; otherwise accounts and saved work may be lost when a container is replaced. Set the hosted API's HTTPS address as the mobile app's `EXPO_PUBLIC_API_URL` when producing a release build.

## Project structure

- `app.py` — FastAPI API, registration/login, token authentication, SQL persistence, private object storage, cloud analysis, chatbot, and score history.
- `static/` — responsive website, resume builder, upload flow, analysis results, and career chatbot.
- `mobile/` — Expo/React Native Android app, resume builder, secure sign-in, upload, score history, and chatbot.
- `tests/` — backend tests for authentication, authorization, resume ownership, uploads, chatbot context, and analysis history.
- `Dockerfile` — container deployment for the website and API.

## Notes and limitations

- Passwords are hashed with scrypt; API access tokens expire after 24 hours.
- A match score is an AI-generated aid for job seekers, not a hiring decision or a guarantee of fit.
- Uploaded resumes must contain selectable text; image-only/scanned PDFs require OCR and are not supported.
- Keep the model provider's privacy and retention policy in mind because resume text and job descriptions are sent to that service.
