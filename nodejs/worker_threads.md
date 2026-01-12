# Node.js Worker Threads with Express, TypeScript, TypeORM & PostgreSQL

**Production Reference Documentation**

---

## 1. Overview

Node.js is single-threaded by default. While it handles **I/O operations** efficiently (database, network, filesystem), it struggles with **CPU-intensive tasks** such as:

* Large loops
* Encryption / hashing
* PDF parsing
* Image or video processing
* AI / ML computations
* Bulk data processing

**Worker Threads** allow CPU-heavy tasks to run in parallel without blocking the main event loop.

---

## 2. Technology Stack

* Node.js
* Express.js
* TypeScript
* Worker Threads (`worker_threads`)
* TypeORM
* PostgreSQL

---

## 3. When to Use Worker Threads

### ✅ Use Worker Threads for

* CPU-bound tasks
* Heavy calculations
* File parsing
* Encryption (bcrypt, crypto)
* AI scoring
* Analytics processing

### ❌ Do NOT use Worker Threads for

* CRUD operations
* Database queries
* API calls
* Redis / Cache operations
* Any I/O-bound task

> Database operations are already non-blocking in Node.js.

---

## 4. Basic Architecture

```
Client Request
     ↓
Express API (Main Thread)
     ↓
Database CRUD (TypeORM)
     ↓
Trigger Worker Thread
     ↓
CPU-Heavy Processing (Background)
     ↓
Update Database
```

The API responds immediately while processing continues in the background.

---

## 5. Worker Thread Fundamentals

### Key Characteristics

* Each worker has its own event loop
* No shared memory (unless explicitly using SharedArrayBuffer)
* Communication happens via message passing
* Workers are not free — they consume CPU & memory

---

## 6. Express + Worker Thread Example

### API Route

```ts
app.get("/heavy", (req, res) => {
  const worker = new Worker(
    path.join(__dirname, "worker.js")
  );

  worker.postMessage({ value: 1e9 });

  worker.on("message", (result) => {
    res.json({ result });
  });

  worker.on("error", (err) => {
    res.status(500).send(err.message);
  });
});
```

### Worker File

```ts
import { parentPort } from "worker_threads";

parentPort!.on("message", (data) => {
  let sum = 0;
  for (let i = 0; i < data.value; i++) {
    sum += i;
  }
  parentPort!.postMessage(sum);
});
```

---

## 7. Worker Threads with TypeORM & PostgreSQL

### Core Rule

❌ Never share TypeORM repositories or DB connections
✅ Always initialize a **new DB connection inside the worker**

---

### Correct Pattern (CRUD + Worker)

#### API Layer (Main Thread)

```ts
app.post("/users", async (req, res) => {
  const user = userRepo.create(req.body);
  await userRepo.save(user);

  startUserWorker(user.id);

  res.status(201).json({
    message: "User created",
    userId: user.id
  });
});
```

---

#### Worker Trigger Function

```ts
function startUserWorker(userId: number) {
  new Worker(
    path.join(__dirname, "user.worker.js"),
    { workerData: { userId } }
  );
}
```

---

#### Worker Implementation

```ts
import { workerData } from "worker_threads";
import { AppDataSource } from "./data-source";
import { User } from "./entity/User";

async function run() {
  await AppDataSource.initialize();

  const repo = AppDataSource.getRepository(User);
  const user = await repo.findOneBy({ id: workerData.userId });

  if (!user) return;

  // CPU heavy work
  let score = 0;
  for (let i = 0; i < 1e8; i++) score += i;

  user.score = score;
  await repo.save(user);

  await AppDataSource.destroy();
}

run();
```

---

## 8. How Many Worker Threads Can Run?

### Rule of Thumb

```
Max Workers ≈ CPU Cores - 1
```

Example:

* 4 cores → 3 workers
* 8 cores → 7 workers

```ts
import os from "os";

const MAX_WORKERS = Math.max(os.cpus().length - 1, 1);
```

### Why Not Unlimited?

* CPU thrashing
* Context switching overhead
* Memory exhaustion
* Worse performance

---

## 9. Worker Pool (Recommended Approach)

### Why Worker Pool?

* Avoid creating workers per request
* Reuse fixed number of workers
* Control CPU usage
* Stable performance under load

---

### Worker Pool Flow

```
API Request
   ↓
Job Queue
   ↓
Idle Worker?
   ↓
Yes → Execute
No  → Wait
```

---

### Worker Pool Implementation (Simplified)

```ts
class WorkerPool {
  private idleWorkers: Worker[] = [];
  private queue: any[] = [];

  constructor(maxWorkers: number) {
    for (let i = 0; i < maxWorkers; i++) {
      const worker = new Worker("worker.js");
      worker.on("message", () => this.runNext(worker));
      this.idleWorkers.push(worker);
    }
  }

  run(data: any) {
    return new Promise((resolve) => {
      this.queue.push({ data, resolve });
      this.runNext();
    });
  }

  private runNext(worker?: Worker) {
    if (!this.queue.length || !this.idleWorkers.length) return;

    const job = this.queue.shift();
    const w = worker || this.idleWorkers.pop()!;
    w.postMessage(job.data);
  }
}
```

---

## 10. CRUD APIs – Worker Usage Rules

| API Type | Worker Needed              |
| -------- | -------------------------- |
| CREATE   | Only if heavy logic exists |
| READ     | ❌ Never                    |
| UPDATE   | Optional                   |
| DELETE   | ❌ Usually no               |

---

## 11. Production Best Practices

### ✅ Do

* Use worker pools
* Pass only IDs or primitive data
* Track job status in DB
* Handle worker crashes
* Graceful shutdown
* Limit max workers

### ❌ Don’t

* Create worker per request
* Share DB connections
* Run DB queries in workers unnecessarily
* Use workers for I/O tasks

---

## 12. Recommended Production Architecture

### For High Load Systems

```
API
 ↓
Queue (BullMQ / SQS / RabbitMQ)
 ↓
Worker Processes
 ↓
Database
```

Workers can internally use worker threads if needed.

---

## 13. Worker Threads vs Cluster

| Feature        | Worker Threads | Cluster          |
| -------------- | -------------- | ---------------- |
| Memory         | Shared process | Separate process |
| Use Case       | CPU tasks      | Scale HTTP       |
| DB Connections | Shared         | Separate         |
| Overhead       | Low            | Higher           |

---

## 14. Final Summary

* Worker threads are for CPU-heavy tasks only
* Max workers ≈ CPU cores
* Use worker pools
* Keep CRUD fast and simple
* Prefer background processing
* Never block the event loop

---
