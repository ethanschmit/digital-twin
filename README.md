## AI Digital Twin – Project Overview

This project implements a conversational **digital twin** of a specific person, powered by **AWS Bedrock** and exposed via a **FastAPI** backend that can run locally or as an **AWS Lambda** function.

### 1. Core Components

- **Persona data (`backend/resources.py` + `backend/data/*`)**
  - `resources.py` loads:
    - `./data/linkedin.pdf` → full LinkedIn profile text.
    - `./data/summary.txt` → high‑level background summary.
    - `./data/style.txt` → communication style notes.
    - `./data/facts.json` → structured facts (e.g. name, roles).

- **Prompt construction (`backend/context.py`)**
  - `prompt()` builds a long, structured system prompt that:
    - Describes the agent as the digital twin of the person.
    - Embeds `facts`, `summary`, `linkedin`, and `style`.
    - Injects the current date/time.
    - Enforces rules: no hallucinations, resist jailbreaks, stay professional.

- **API backend (`backend/server.py`)**
  - Uses `FastAPI` to expose HTTP endpoints.
  - Loads environment config from `.env` with `python-dotenv`.
  - Configures CORS via `CORS_ORIGINS`.
  - Creates a Bedrock runtime client (`boto3.client("bedrock-runtime")`) using `DEFAULT_AWS_REGION` and `BEDROCK_MODEL_ID`.
  - Optional conversation memory:
    - **S3 mode** when `USE_S3=true` and `S3_BUCKET` is set.
    - **Local file mode** in `MEMORY_DIR` (defaults to `../memory`).

- **Lambda adapter (`backend/lambda_handler.py`)**
  - Wraps the FastAPI app with **Mangum**:
    - `handler = Mangum(app)`
  - This handler is used when deploying to AWS Lambda.

- **Deployment helper (`backend/deploy.py`)**
  - Uses Docker with the official `public.ecr.aws/lambda/python:3.12` image to:
    - Install dependencies from `backend/requirements.txt` into `lambda-package/` with Lambda‑compatible (manylinux) wheels.
    - Copy `server.py`, `lambda_handler.py`, `context.py`, `resources.py`, and the `data/` folder into `lambda-package/`.
    - Zip everything into `lambda-deployment.zip` for upload to Lambda.

### 2. Request Flow (Runtime)

1. **Client sends a chat request**
   - `POST /chat` with body (see `ChatRequest` in `backend/server.py`):
     - `message`: user text.
     - `session_id` (optional): reuse to maintain history.

2. **Server loads context**
   - Reads environment via `.env` (e.g. CORS origins, AWS region, storage mode).
   - Loads existing conversation history for `session_id`:
     - From S3 (if `USE_S3=true`) or
     - From `<MEMORY_DIR>/<session_id>.json` on disk.

3. **Server calls Bedrock (`call_bedrock` in `backend/server.py`)**
   - Builds a `messages` list:
     - First item: the digital‑twin system prompt from `context.prompt()`, formatted as a Bedrock user message (`"System: ..."`).
     - Next: up to the last 20 messages from the conversation history.
     - Last: the current user message.
   - Calls `bedrock_client.converse()` with `BEDROCK_MODEL_ID` and inference parameters.
   - Extracts the assistant text from the response.

4. **Server updates memory and responds**
   - Appends the user message and the assistant response (with timestamps) to the conversation list.
   - Persists the updated conversation via `save_conversation` (S3 or local file).
   - Returns `{ "response": <assistant_text>, "session_id": <session_id> }`.

5. **Additional endpoints**
   - `GET /` → basic API metadata (model, storage type).
   - `GET /health` → simple health & config check.
   - `GET /conversation/{session_id}` → full stored conversation history.

### 3. Running Locally

From the `backend` directory:

```bash
pip install -r requirements.txt
python server.py
```

Then call:

- `GET http://localhost:8000/` and `GET http://localhost:8000/health` to verify.
- `POST http://localhost:8000/chat` with JSON body:

```json
{
  "message": "Hi, who are you?",
  "session_id": null
}
```

### 4. Packaging for AWS Lambda

From the `backend` directory (Docker required):

```bash
python deploy.py
```

This will:

- Build `lambda-package/` with all dependencies and app code.
- Create `lambda-deployment.zip`.

Then in AWS:

- Create a Lambda function (Python 3.12).
- Upload `lambda-deployment.zip` as the function code.
- Set the handler to: `lambda_handler.handler`.
- Attach an API Gateway (or Function URL) so clients can hit the same endpoints.

### 5. Key Files Reference

- **`backend/server.py`** – FastAPI app, Bedrock client, endpoints, and conversation memory.
- **`backend/context.py`** – Constructs the digital‑twin system prompt from persona data.
- **`backend/resources.py`** – Loads LinkedIn / summary / style / facts from `backend/data/`.
- **`backend/lambda_handler.py`** – Mangum wrapper exposing `handler` for Lambda.
- **`backend/deploy.py`** – Docker‑based Lambda deployment packager.
- **`backend/.env`** – Environment configuration (AWS region, project name, etc.).
- **`backend/requirements.txt`** – Python dependencies for both local and Lambda.

