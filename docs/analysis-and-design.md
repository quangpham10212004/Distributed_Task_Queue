# Analysis & Design — Distributed Task Queue System

---

## Part 1: Analysis

### 1.1 Problem Statement

Backend APIs often need to handle time-consuming tasks such as sending emails, processing images, or running AI inference. Executing these synchronously within the request cycle blocks the API, increases latency, and risks timeouts.

**Solution:** Decouple these tasks from the request cycle by pushing them into a queue and processing them asynchronously in the background.

---

### 1.2 Functional Requirements

---

#### FR-1: Enqueue Task

status: in-progress

**Statement**
The system shall accept an HTTP POST request containing a task type and payload, create a Task entity with a unique ID and PENDING status, and push it onto the main task queue.

**Rationale**
This is the entry point of the entire system. Without a reliable enqueue mechanism, tasks can be lost or duplicated at the source. The API must return immediately (non-blocking) to avoid impacting client latency.

**Acceptance Criteria**
- POST /tasks with a valid body returns HTTP 202 and a taskId
- The task appears in `task_queue` on Redis after enqueue
- The task has status = PENDING and retryCount = 0
- POST with missing required fields returns HTTP 400

**Verification Method**
Test — Integration test: call POST /tasks, verify response and Redis state.

---

#### FR-2: Consume and Execute Task

status: in-progress

**Statement**
The system shall continuously pull tasks from the task queue, dispatch each task to the appropriate handler based on task type, and mark the task SUCCESS upon successful execution.

**Rationale**
The worker is the core processing component. It must not miss tasks and must dispatch to the correct handler by type.

**Acceptance Criteria**
- A task is popped from `task_queue` and executed within 1s of being enqueued (under no load)
- Task types EMAIL, IMAGE, and AI are dispatched to their respective handlers
- After successful execution, the idempotency key for the taskId is marked SUCCESS
- An unknown task type logs an error without crashing the worker

**Verification Method**
Test — Unit test each handler; Integration test end-to-end: enqueue → execute → verify idempotency store.

---

#### FR-3: Retry Mechanism

status: in-progress

**Statement**
The system shall retry a failed task up to maxRetries times using exponential backoff (1s, 2s, 4s, ...), re-queuing it to the retry queue after each failure.

**Rationale**
Tasks may fail due to transient errors (network issues, external service unavailability). Automatic retry improves fault tolerance without manual intervention.

**Acceptance Criteria**
- Task fails on attempt 1 → retried after 1s; attempt 2 → after 2s; attempt 3 → after 4s
- retryCount increments correctly after each failure
- The task is pushed to `retry_queue` with the updated retryCount
- The task is not retried more than maxRetries times

**Verification Method**
Test — Unit test RetryService with a mock handler that always throws; verify delay timing and retryCount values.

---

#### FR-4: Dead-Letter Queue

status: in-progress

**Statement**
The system shall move a task to the dead-letter queue when its retryCount exceeds maxRetries, and mark its status as DEAD_LETTER.

**Rationale**
Continuously retrying a persistently failing task wastes resources and obscures the root cause. The DLQ preserves the task for debugging or manual reprocessing.

**Acceptance Criteria**
- A task with retryCount > maxRetries is pushed to `dead_letter_queue`
- The idempotency key for the task is marked DEAD_LETTER
- The task no longer appears in `task_queue` or `retry_queue`

**Verification Method**
Test — Integration test: enqueue a task with a handler that always fails, verify the task appears in DLQ after maxRetries+1 attempts.

---

#### FR-5: Idempotent Task Execution

status: in-progress

**Statement**
The system shall check a task's idempotency key before execution and skip processing if the task has already been successfully executed.

**Rationale**
In a distributed environment, the same task may be enqueued multiple times due to client retries or network duplicates. Idempotency ensures each task has an effect exactly once.

**Acceptance Criteria**
- A task with idempotency key = SUCCESS is skipped; the handler is not called again
- A task with idempotency key = PROCESSING (being handled by another worker) is skipped
- Idempotency keys have a TTL of 24 hours
- A new task (no existing key) is executed normally

**Verification Method**
Test — Unit test IdempotencyStore; Integration test: enqueue the same taskId twice, verify the handler is called exactly once.

---

#### FR-6: Concurrent Task Processing

status: in-progress

**Statement**
The system shall process multiple tasks concurrently using a configurable thread pool, and shall complete all in-progress tasks before shutting down.

**Rationale**
A single-threaded worker is a throughput bottleneck. Multi-threading increases processing capacity. Graceful shutdown prevents task loss during restarts.

**Acceptance Criteria**
- N tasks submitted concurrently complete in approximately max(task_duration) rather than N × task_duration
- Thread pool size is configurable via application config
- On SIGTERM, the worker finishes all in-progress tasks before stopping (no task is dropped)
- A failing thread does not affect other threads

**Verification Method**
Test — Integration test: enqueue 10 tasks each sleeping 1s with thread pool = 4, verify completion in ~3s instead of 10s. Graceful shutdown: send SIGTERM while tasks are running, verify all complete.

---

### 1.3 Non-Functional Requirements

---

#### NFR-1: Low Latency API

status: in-progress

**Statement**
The API service shall return a response to the client within 100ms for any enqueue request, regardless of task execution time.

**Rationale**
The API must not block waiting for task completion. The client only needs confirmation that the task was accepted.

**Acceptance Criteria**
- P99 response time of POST /tasks ≤ 100ms under 100 req/s load
- Response is returned before the task is executed

**Verification Method**
Test — Load test with 100 concurrent requests, measure P99 latency.

---

#### NFR-2: High Throughput

status: in-progress

**Statement**
The worker service shall process at least 100 tasks per second with a thread pool of 4 threads, assuming each task completes within 10ms.

**Rationale**
High throughput is the primary reason for using an async queue over synchronous processing.

**Acceptance Criteria**
- 1000 tasks are fully processed within ≤ 30s with thread pool = 4
- CPU usage does not exceed 80% during processing

**Verification Method**
Test — Benchmark: enqueue 1000 tasks, measure time until all are marked SUCCESS.

---

#### NFR-3: Fault Tolerance

status: in-progress

**Statement**
The system shall not lose any enqueued task when a worker thread throws an uncaught exception, and shall not crash the worker process.

**Rationale**
A worker crash that causes task loss is a critical failure in production. Each thread failure must be isolated.

**Acceptance Criteria**
- A thread throwing RuntimeException causes the task to be retried or sent to DLQ; the worker process continues running
- No task is lost (absent from all queues and idempotency store)

**Verification Method**
Test — Unit test: mock handler throws RuntimeException, verify the worker process does not stop and the task is handled correctly.

---

#### NFR-4: Scalability

status: in-progress

**Statement**
The system shall support horizontal scaling of the worker service by running multiple worker instances consuming from the same Redis queue without coordination.

**Rationale**
Scaling via thread count is limited to a single machine. Running multiple worker instances is the practical path to higher throughput.

**Acceptance Criteria**
- 2 worker instances running in parallel: each task is processed by exactly one instance (guaranteed by Redis BRPOP)
- Throughput scales linearly when adding instances

**Verification Method**
Demonstration — Run 2 worker instances, enqueue 100 tasks, verify total processed = 100 with no duplicates.

---

#### NFR-5: Consistency (Idempotency)

status: in-progress

**Statement**
The system shall guarantee that each task is executed at most once, even when the same taskId is enqueued multiple times or a worker restarts mid-execution.

**Rationale**
Duplicate execution can cause serious side effects (sending an email twice, double-charging a payment). At-most-once execution is the minimum correctness requirement.

**Acceptance Criteria**
- Enqueuing the same taskId 3 times results in the handler being called exactly once
- A worker restart after marking PROCESSING but before executing does not result in a second execution

**Verification Method**
Test — Integration test: enqueue duplicate taskId, verify handler call count = 1.

---

### 1.4 Scope

**In scope (v1):**
- Enqueue / consume task
- Retry with exponential backoff
- Dead-letter queue
- Idempotency check
- Multi-threaded worker

**Out of scope (v1):**
- UI dashboard / monitoring
- Delay / scheduled tasks
- Priority queue
- Distributed tracing
- Message ordering guarantee

---

### 1.5 Domain Model

**Task Entity:**

| Field      | Type   | Description                                           |
|------------|--------|-------------------------------------------------------|
| taskId     | UUID   | Unique identifier                                     |
| type       | String | Task type (EMAIL, IMAGE, AI)                          |
| payload    | JSON   | Input data for the task                               |
| status     | Enum   | PENDING / PROCESSING / SUCCESS / FAILED / DEAD_LETTER |
| retryCount | int    | Number of retries so far                              |
| maxRetries | int    | Retry limit (default: 3)                              |
| createdAt  | long   | Task creation timestamp                               |

---

### 1.6 Constraints

- Use **Redis** as message broker (List) and idempotency store (TTL 24h)
- Language: **Java 17 + Spring Boot 3**
- Worker must support graceful shutdown
- No database other than Redis

---

## Part 2: Design

### 2.1 Project Structure

Multi-module Maven project — `common` is a shared library depended on by both services.

```
distributed-task-queue/
├── pom.xml                          # parent POM
├── common/                          # shared module (domain, infra, service logic)
│   └── src/main/java/com/taskqueue/common/
│       ├── domain/
│       │   ├── Task.java
│       │   └── TaskStatus.java
│       ├── infrastructure/
│       │   ├── QueueClient.java
│       │   └── IdempotencyStore.java
│       └── service/
│           ├── TaskService.java
│           ├── RetryService.java
│           └── DLQService.java
│
├── api-service/                     # Process 1: HTTP entrypoint
│   └── src/main/java/com/taskqueue/api/
│       ├── ApiApplication.java
│       └── controller/
│           └── TaskController.java
│
└── worker-service/                  # Process 2: background worker
    └── src/main/java/com/taskqueue/worker/
        ├── WorkerApplication.java
        ├── WorkerRunner.java
        ├── RetryScheduler.java
        └── handler/
            ├── TaskHandler.java     # interface
            ├── HandlerRegistry.java
            ├── EmailHandler.java
            ├── ImageHandler.java
            └── AiHandler.java
```

---

### 2.2 Domain Model

```java
// Task.java
public class Task {
    String taskId;       // UUID
    String type;         // EMAIL | IMAGE | AI
    Map<String, Object> payload;
    TaskStatus status;
    int retryCount;
    int maxRetries;      // default: 3
    long createdAt;      // Unix ms
}

// TaskStatus.java
public enum TaskStatus {
    PENDING, PROCESSING, SUCCESS, FAILED, DEAD_LETTER
}
```

---

### 2.3 Class Design

#### common — Infrastructure

```
QueueClient
  + push(queue: String, task: Task): void          // LPUSH
  + blockingPop(queue: String, timeout: int): Task // BRPOP
  + pushToSortedSet(key: String, score: double, task: Task): void  // ZADD retry_queue
  + popDueFromSortedSet(key: String, now: double): List<Task>       // ZRANGEBYSCORE + ZREM
  + enqueueAtomic(task: Task): void                // MULTI/EXEC: LPUSH + SET idempotency PENDING

IdempotencyStore
  + check(taskId: String): TaskStatus | null       // GET idempotency:{taskId}
  + mark(taskId: String, status: TaskStatus): void // SET idempotency:{taskId} EX 86400
```

#### common — Services

```
TaskService
  + enqueue(type: String, payload: Map): String    // create Task, call QueueClient.enqueueAtomic, return taskId

RetryService
  + handle(task: Task): void
    // task.retryCount++
    // if retryCount <= maxRetries:
    //   score = now + (1000 * 2^(retryCount-1))  // exponential backoff in ms
    //   QueueClient.pushToSortedSet("retry_queue", score, task)
    // else:
    //   DLQService.send(task)

DLQService
  + send(task: Task): void
    // QueueClient.push("dead_letter_queue", task)
    // IdempotencyStore.mark(taskId, DEAD_LETTER)
```

#### worker-service

```
TaskHandler (interface)
  + String getType()
  + void execute(Task task) throws Exception

HandlerRegistry
  - Map<String, TaskHandler> handlers
  + register(handler: TaskHandler): void
  + getHandler(type: String): TaskHandler          // throws if type unknown

WorkerRunner (Runnable, thread pool)
  - ExecutorService threadPool                     // fixed, size configurable
  + start(): void
    // loop:
    //   task = QueueClient.blockingPop("task_queue", 5)
    //   if task != null: threadPool.submit(() -> process(task))
  - process(task: Task): void
    // status = IdempotencyStore.check(taskId)
    // if status == SUCCESS or PROCESSING: return
    // IdempotencyStore.mark(taskId, PROCESSING)
    // try:
    //   HandlerRegistry.getHandler(task.type).execute(task)
    //   IdempotencyStore.mark(taskId, SUCCESS)
    // catch:
    //   RetryService.handle(task)

RetryScheduler (@Scheduled every 500ms)
  + promote(): void
    // tasks = QueueClient.popDueFromSortedSet("retry_queue", now)
    // for each task: QueueClient.push("task_queue", task)
```

#### api-service

```
TaskController
  + POST /tasks (type, payload) → 202 {taskId}
    // validate input
    // return TaskService.enqueue(type, payload)
```

---

### 2.4 Redis Data Design

| Key                      | Structure   | TTL   | Value                        |
|--------------------------|-------------|-------|------------------------------|
| `task_queue`             | List        | none  | JSON-serialized Task         |
| `retry_queue`            | Sorted Set  | none  | score=executeAt (Unix ms), member=JSON Task |
| `dead_letter_queue`      | List        | none  | JSON-serialized Task         |
| `idempotency:{taskId}`   | String      | 24h   | TaskStatus enum value        |

---

### 2.5 Concurrency Design

```
WorkerApplication starts:
  └── WorkerRunner (1 dedicated thread)
        └── BRPOP loop → submit to ThreadPoolExecutor (N threads, configurable)
  └── RetryScheduler (@Scheduled, 1 thread)

ThreadPoolExecutor:
  - corePoolSize = workerThreads (default: 4)
  - Graceful shutdown: executor.shutdown() + awaitTermination(30s)
    → in-progress tasks complete before process exits
```

---

### 2.6 API Contract

**POST /tasks**

Request:
```json
{
  "type": "EMAIL",
  "payload": { "to": "user@example.com", "subject": "Hello" },
  "maxRetries": 3
}
```

Response `202 Accepted`:
```json
{
  "taskId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "PENDING"
}
```

Error `400 Bad Request`:
```json
{
  "error": "Missing required field: type"
}
```
