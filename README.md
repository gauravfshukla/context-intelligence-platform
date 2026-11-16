# DocMind (context-intelligence-platform)

## What it is

DocMind is a Spring Boot backend for a "chat with your documents" app. A React frontend calls it. The frontend runs at `localhost:5173` or `the-documind.pages.dev`, according to the CORS config. Users sign up, create chat sessions, and upload PDFs or Wikipedia pages. They then ask questions, and the answers are grounded in those documents. Each answer streams back token by token and includes citations and relevant images.

It is a RAG (retrieval-augmented generation) system with these features:

- Hybrid search, combining Pinecone vectors with Postgres full-text search.
- A local ONNX cross-encoder reranker (ms-marco-MiniLM-L-6-v2).
- Gemini as the LLM (gemini-3.1-flash-lite).
- Long-term conversation memory.
- Vision captioning of images extracted from PDFs.

## Tech stack

- Java 21 and Spring Boot 3.5.6, using WebFlux (reactive), Spring Security, JPA and Spring AI.
- PostgreSQL for users, sessions, messages, attachments and document chunks.
- Redis for the job queues (Streams), locks, the session cache, and token pub/sub.
- Pinecone, with two separate accounts: one index for documents and one for chat memory.
- Cloudinary for media, Gmail/SMTP for OTP emails, PDFBox for PDFs, and JWT in cookies for auth.

## Project structure

Everything is under `src/main/java/com/accenture/intern/docmind/`:

| Package | Role |
|---|---|
| `controller/` | REST endpoints: Auth, Session, Chat, Attachment, Explore, Preference, Analytics, Proxy |
| `service/` | Business logic and background workers: Auth, Session, Attachment, Ingestion worker, Memory worker, PDF export worker, Email, Cloudinary, Analytics |
| `aiservices/chat` | `ChatService` (accepts messages) and `LlmWorkerService` (runs the pipeline) |
| `aiservices/understanding` | `PlannerService` (query routing) and execution plans: Direct, Static (decompose), Adaptive (multi-hop) |
| `aiservices/fastintent` | Regex matching for greetings and "what files do I have" questions, with no LLM call |
| `aiservices/retrieval` | Hybrid retrieval, RRF fusion, reranking, multi-signal ranker, orchestrator, Pinecone stores |
| `aiservices/context` | Context builder, evidence structuring, citations, session summary, suggested questions |
| `aiservices/embedding` | Document parsing, embedding, Wikipedia ingestion |
| `aiservices/memory` | Topic-shift detection and memory gating |
| `aiservices/vision` | Image description with the LLM |
| `entity/`, `repository/`, `dto/` | JPA entities, repositories and request/response objects |
| `security/` | `SecurityConfig`, `JwtWebFilter`, `JwtService` |
| `config/` | Redis, Pinecone, workers, full-text index, retrieval thresholds |
| `resources/prompts/*.st` | Prompt templates (router, RAG, general, summary, follow-ups) |
| `resources/onnx/` | The local reranker model |
| `storage/images/extracted/` | Images pulled out of uploaded PDFs, served at `/files/**` |

Docs at the repo root: `Overview.md` (the query-flow walkthrough), `Architecture.md`, `Services_Info.md`, `Setup.md`, and `AI_Architecture.pdf`.

## Flow

### Upload flow

1. `POST /api/sessions/{id}/attachments/upload` receives a file. A Wikipedia URL goes to `/wikipedia` instead.
2. The file is stored, and the parser extracts text and images.
3. The text is chunked and embedded into Pinecone. Chunks are also saved in Postgres `document_chunks`, which is the keyword search source.
4. Images are captioned and indexed.
5. The upload request returns only after ingestion finishes. The session state moves through EMBEDDING/INGESTING and then to ready.

### Chat flow (from Overview.md)

1. `POST /api/chat/{id}/message` saves the message as PENDING and pushes a job onto the Redis stream `chat_jobs`. It returns a `messageId` right away.
2. The client opens `GET /api/chat/{sessionId}/stream/{messageId}`, a Server-Sent Events stream.
3. `LlmWorkerService` takes the job and locks it.
4. Topic-shift detection runs. If the topic changed, the old episode goes to the `memory_jobs` stream. The memory worker later turns it into a timeline summary.
5. If documents are still ingesting, the worker waits up to 120 seconds.
6. `PlannerService` routes the query:
   - Fast intent (greeting, bot question, session info, history) skips retrieval.
   - Otherwise the LLM router returns a JSON plan with strategy, execution tier (DIRECT, DECOMPOSE or ADAPTIVE), retrieval mode and an optimized query.
7. Memory strategies (timeline, episodic summary, specific topic) answer from Postgres or the Pinecone message index.
8. Document retrieval runs for the remaining queries. Dense search (Pinecone, top 15) and BM25 (Postgres, top 15) run in parallel. The results are merged with Reciprocal Rank Fusion and a per-source diversity cap. The cross-encoder keeps the top 5, and a session boost is applied. If the best score is low, the search widens to the whole corpus. The ADAPTIVE tier loops up to 2 more times. A whole-document mode returns up to 60 chunks.
9. Evidence is deduplicated, grouped, merged and ordered. The final prompt is built from it.
10. Gemini generates the answer. Tokens go to the Redis channel `chat-tokens:{messageId}` and then out through SSE, along with progress events, citations and images.

## API endpoints

Public: `/auth/login`, `/auth/signup`, `/auth/logout`, `/auth/refresh`, `/auth/request-otp`, `/auth/verify-otp`, `/auth/reset-password`, `/api/proxy`, `/files/**` and Swagger. Everything else needs the `access_token` cookie.

**Auth (`/auth`)**
- `GET/DELETE /me`, `PUT /update`, `PUT /update-profile-image`

**Sessions (`/api/sessions`)**
- `POST` and `GET` (create and list)
- `DELETE /{id}` and `PUT /{id}/rename`
- `GET /{id}/messages`
- `GET /{id}/suggested-questions`
- `GET /{id}/export`
- `POST /{id}/share/email`
- `POST /{id}/export-pdf`, `GET /{id}/export-pdf/{jobId}` and `GET /{id}/export-pdf/{jobId}/download`

**Chat (`/api/chat`)**
- `POST /{id}/message` and `GET /{sessionId}/stream/{messageId}` (SSE)

**Attachments (`/api/sessions/{sessionId}/attachments`)**
- `POST /upload` (multipart field `file`), `POST /wikipedia` (`{"url": "..."}`), `GET /wikipedia/search?query=`, `GET` (list)

**Explore (`/api/explore`)**
- `GET /attachments`, `DELETE /attachments/{id}`

**Preferences (`/api/preferences`)**
- `GET /models`, `PUT /model`

**Analytics (`/api/analytics`)**
- `GET /daily`

## How to run it

1. Install Java 21 and have a reachable PostgreSQL and Redis.
2. Fill in `.env` at the repo root. The keys are:
   - Database: `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`
   - Auth: `JWT_SECRET`
   - AI: `GEMINI_API_KEY`
   - Mail: `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`
   - Pinecone: `PINECONE_API_KEY`, `PINECONE_MEMORY_KEY`, `PINECONE_MEMORY_INDEX_NAME`
   - Media: `CLOUDINARY_URL`
   - Redis: `REDIS_HOST`, `REDIS_PORT`, `REDIS_USERNAME`, `REDIS_PASSWORD`
3. Start it with `.\mvnw.cmd spring-boot:run`. It listens on http://localhost:8080.
4. Open Swagger at http://localhost:8080/swagger-ui.html.

Two things to check in `application.properties`:

- It reads the Gemini key from `spring.ai.google.genai.api-key`, but `.env` calls it `GEMINI_API_KEY`. Confirm the two line up.
- `spring.ai.vectorstore.pinecone.api-key` and the memory key are set there. Confirm they match your Pinecone accounts.

## Testing the API

Auth uses an `access_token` cookie, not a Bearer header. In curl, use a cookie jar (`-c` and `-b`). Swagger's "Try it out" in the browser also keeps the cookie.

```powershell
$b = "http://localhost:8080"

# 1. Sign up or log in (the cookie is saved to cj.txt)
curl.exe -c cj.txt -X POST "$b/auth/signup" -H "Content-Type: application/json" `
  -d '{\"name\":\"Test\",\"email\":\"t@example.com\",\"password\":\"Pass@123\",\"phoneNumber\":\"9999999999\"}'
curl.exe -c cj.txt -X POST "$b/auth/login" -H "Content-Type: application/json" `
  -d '{\"email\":\"t@example.com\",\"password\":\"Pass@123\"}'

# 2. Create a session (note the returned id)
curl.exe -b cj.txt -X POST "$b/api/sessions" -H "Content-Type: application/json" -d '{\"title\":\"My test\"}'

# 3. Upload a PDF (replace 1 with your session id)
curl.exe -b cj.txt -X POST "$b/api/sessions/1/attachments/upload" -F "file=@C:\path\doc.pdf"

# 4. Send a question (returns a messageId)
curl.exe -b cj.txt -X POST "$b/api/chat/1/message" -H "Content-Type: application/json" `
  -d '{\"message\":\"Summarize this document\",\"model\":\"<value from GET /api/preferences/models>\"}'

# 5. Stream the answer (-N disables buffering)
curl.exe -N -b cj.txt "$b/api/chat/1/stream/<messageId>"

# 6. Read the history and the suggested follow-ups
curl.exe -b cj.txt "$b/api/sessions/1/messages"
curl.exe -b cj.txt "$b/api/sessions/1/suggested-questions"
```

Good test queries, each hitting a different path:

- `hello` takes the fast-intent path with no retrieval.
- `what files did I upload` also takes the fast-intent path.
- `Summarize this document` uses whole-document mode.
- A factual question uses ranked retrieval.
- `Compare A and B` uses the DECOMPOSE tier.
- `What did we discuss earlier?` uses the memory timeline.

Notes:

- Send the stream request right after the message request, because tokens are published through Redis pub/sub.
- Forgot-password flow: `request-otp` with `{"email"}`, then `verify-otp` with `{"email","otp"}`, which returns a `verificationToken`. Pass that token to `reset-password`.
- Unit tests: `.\mvnw.cmd test`. There are only two. `EvidenceStructuringServiceTest` is real. `DocMindApplicationTests` is a context-load test and needs working DB, Redis and Pinecone credentials. Coverage is thin, so end-to-end testing through the API is the main way to verify behavior.
- The exact model values and a few request shapes (reset-password, `UpdateModelRequest`) are in the DTOs. Swagger shows them.
