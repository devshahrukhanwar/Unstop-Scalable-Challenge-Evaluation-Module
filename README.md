# Technical Design Document for Unstop Scalable Challenge Evaluation Module

Designing a Scalable Challenge Evaluation Module for Unstop as a Technical assesment.

---

**Challenge Evaluation Module**  
**Author:** Shahrukh Anwar

## 1. Goal

Build a secure and scalable module to accept code submissions from a web-based IDE, run them against test cases for multiple programming languages, score results, save them in DB, and send real-time feedback to the user.

## 2. Key Requirements

- Support multiple languages (PHP, JavaScript, Python, C++).
- Run untrusted user code securely.
- Show results for each test case in real time.
- Store results for later review.
- Keep performance high even with many submissions.

## 3. High-Level Architecture

1. **Frontend IDE (Vue.js)**

   - Based on [Monaco Editor](https://www.npmjs.com/package/@guolao/vue-monaco-editor).
   - Sends submission (code, language, metadata) to backend via HTTPS.
   - Receives real-time results via [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket) or [Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events).

2. **Backend API (Laravel)**

   - Receives and validates submissions.
   - Stores submission record in database.
   - Sends job to **[Job Queue](https://laravel.com/docs/12.x/queues)** for processing.

3. **Job Queue**

   - Could be Laravel Queue with DB/Redis as the driver.
   - Handles jobs asynchronously to avoid blocking API requests. This can largely help handling large user submissions.

4. **Code Executor Service**

- For inhouse executor (Need to study more how we can do it)
  - Runs on a dedicated server or VM (Dedicated Docker containers).
  - Uses simple sandboxing for security such as `lightweight Docker containers`.
  - Runs code against predefined test cases for the chosen language.
  - Captures `stdout, stderr, execution time, and memory usage`.
- Some paid services are there, through which we can easily do our job via REST API - [Judge0](https://judge0.com/), [Sphere Engine](https://sphere-engine.com/)

5. **Scoring & Result Storage**

   - Compares output with expected results.
   - Calculates percentage score.
   - Saves results in DB for user review.

6. **Result Notification**
   - Sends incremental updates to frontend via WebSocket/SSE.
   - Final status stored in database.

## 4. Data Flow

1. User writes code and clicks **Submit**.
2. Frontend sends request to Laravel API.
3. Laravel stores submission record, pushes job to queue.
4. Executor service picks up job, runs tests, streams results back.
5. Laravel updates DB and frontend gets real-time updates.
6. Final score is calculated and stored.

## 5. Technology Choices (Minimal)

- **Frontend:** Vue.js, Monaco Editor, WebSocket/SSE.
- **Backend:** Laravel (API, Auth, Job Queue).
- **Job Queue:** Laravel Queue (DB/Redis).
- **Executor:** PHP scripts for orchestration + language-specific CLI compilers/interpreters installed on the server.
- **DB:** MySQL/PostgreSQL for metadata.
- **File Storage:** Local or S3 (optional for logs).

---

**Note:** For `Executor` scripts I need to study more in depth, how to create it, it may be done on more Devops heavy side, but I am referring to code alternatives here.

---

## 6. Security Considerations

- Run user code with a low-privilege user.
- Set CPU and memory limits via OS tools (e.g. `ulimit`).
- Disable dangerous PHP functions if PHP is used in execution layer (We can create a list of dangerous functions in `config`).
- Validate inputs before processing.
- Restrict file system and network access.

## 7. Scalability Plan

- Start with one executor server.
- Add more executor servers when traffic grows — Laravel Queue supports multiple workers easily.
- Use Redis for a single, central job queue.
- Optimize by caching test cases locally (File/DB/Redis).

## 8. Example API Endpoints

- `POST /submissions` — Create a submission, returns submission ID.
- `GET /submissions/{id}` — Get submission status and results.
- WebSocket `/submissions/{id}/stream` — Real-time result updates.

---

## **Additional Points**

To scale it well and make it responsive as userbase grows, we can increase the memory of the server, so we can use more and more short-lived Docker containers (may be use AWS fargate), I will need some time to go into Devops things, but the arcitechture above I am submitting is basically code driven rather than Devops heavy, as I am still learning Devops.
