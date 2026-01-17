
# 🚀 Stress Test Performance Analysis: NES Core API
**Project:** Transaction Monitoring System Stability Test  
**Date:** January 2026  
**Infrastructure:** JBoss 7 (WildFly 16), Java 8, 20 Core CPU, ClickHouse & Oracle

---

## 📋 1. Infrastructure Overview
The system was tested on the **MetaForce02** server environment. 

| Component | Specifications |
| :--- | :--- |
| **CPU** | 12th Gen Intel(R) Core(TM) i7-12700F (20 Cores) |
| **RAM** | 94.14 GiB Total |
| **OS Environment** | Linux Containerized (Docker) |
| **Runtime** | JBoss 7 (WildFly 16) / Java 8 (Legacy Threading Model) |
| **Persistence** | ClickHouse OLAP & Oracle OLTP |

---

## 🛠 2. Methodology: Why k6 instead of JMeter?
To reach high-scale throughput (10,000 RPS), we utilized **k6**'s modern engine instead of traditional tools like JMeter.

> [!abstract] Key Differences in Scaling
> - **JMeter (Thread-Bound):** Every "user" requires a heavy OS thread (~1MB RAM). Reaching 10k RPS with slow responses (40s) would require 10k threads, consuming **10GB+ of RAM** just for the test tool, causing a local machine crash.
> - **k6 (Event-Bound):** Uses Go-routines (~4KB RAM). Reaching the same load requires only a few MBs of RAM, allowing the test to run alongside the server on a 20-core CPU with minimal interference.
> - **Accuracy:** k6 uses **Ramping Arrival Rate**, which decouples "Target Rate" from "Worker threads (VUs)." This correctly identifies where the server's thread pool expires.
> - ┌─────────────────────────────────────────────────┐
│          k6 Executors (6 types)                 │
├─────────────────────────────────────────────────┤
│ 1. shared-iterations      (Simple load)         │
│ 2. per-vu-iterations      (Per-user load)       │
│ 3. constant-vus           (Sustained load)      │
│ 4. ramping-vus            (Variable VU load)    │
│ 5. constant-arrival-rate  (Fixed rate)          │
│ 6. ramping-arrival-rate   (Variable rate)       │
└─────────────────────────────────────────────────┘

**Arrival Rate vs. Thread Limit**: In JMeter, you define "Threads." If the server gets slow (like your 40-second responses), JMeter threads get **"stuck"** and stop sending new requests. This hides the true load.
**Precision**: k6 keeps trying to push the RPS you asked for (e.g., 10,000) even if the server is failing. This allows you to see exactly where the "Total Results" drop and the "Dropped Iterations" begin.


---

## 📊 3. Performance Result Matrix
We executed 4 specific scenarios to identify the exact point of system failure.

### Summary Table
| Scenario ID | Data Persistence   | Target RPS | Real Achieved RPS | p(95) Latency | Result       |
| :---------- | :----------------- | :--------- | :---------------- | :------------ | :----------- |
| [[1k.html]] | Disconnected       | 1,000      | 1,000             | 12.25ms       | ✅ Success    |
| [[3k.html]] | Disconnected       | 3,000      | 2,751             | 629.05ms      | ⚠️ Saturated |
| [[5k.html]] | Disconnected       | 5,000      | ~2,800            | 4.90s         | ❌ Failing    |
| [[25.html]] | **ClickHouse SQL** | **25**     | **<30**           | **42,368ms**  | 🚨 Crash     |
|             |                    |            |                   |               |              |
|             |                    |            |                   |               |              |
|             |                    |            |                   |               |              |
1k js script
---
```js
import http from "k6/http";
import { check } from "k6";
import { Counter, Trend } from "k6/metrics";
import { htmlReport } from "https://raw.githubusercontent.com/benc-uk/k6-reporter/main/dist/bundle.js";
import { textSummary } from "https://jslib.k6.io/k6-summary/0.0.1/index.js";

const fastRequests = new Counter("fast_requests");
const slowRequests = new Counter("slow_requests");
const responseTimes = new Trend("custom_response_time");

export const options = {
  scenarios: {
    stress_test_10k: {
      executor: "ramping-arrival-rate",
      startRate: 2,
      timeUnit: "1s",
      preAllocatedVUs: 500, 
      maxVUs: 10000, 
      stages: [
        { duration: "2m", target: 700 },   // Warm up
        { duration: "2m", target: 1000 },  // Push harder
        { duration: "1m", target: 0 },     // Cool down   
      ],
    },
  },
  thresholds: {
    http_req_failed: ["rate<0.01"],
    http_req_duration: ["p(95)<500"],
  },
};

const url = "http://172.16.116.185:40354/nes.s.Web/NesFront";
const params = {
  headers: {
    "Content-Type": "application/json",
    op: "15241000",
    company: "24",
    role: "1",
    Cookie: "NESSESSION=lAfc3rC0XGXpjrzl1VF0K2I6NRGlUP",
  },
  timeout: "60s",
};

export default function () {
  const body = JSON.stringify(["CIF-11217408", "TEST_REALTIME"]);
  const startTime = new Date().getTime();
  
  const res = http.post(url, body, params);
  
  const duration = new Date().getTime() - startTime;
  responseTimes.add(duration);
  
  if (duration < 1000) {
    fastRequests.add(1);
  } else {
    slowRequests.add(1);
    console.log(`SLOW REQUEST: ${duration}ms at ${new Date().toISOString()}`);
  }
  
  check(res, { 
    "status 200": (r) => r.status === 200,
    "response < 1s": (r) => r.timings.duration < 1000,
  });
}

export function handleSummary(data) {
  return {
    // Save HTML report
    "report.html": htmlReport(data)
  };
}
```
 3k script
```js
import http from "k6/http";
import { check } from "k6";
import { Counter, Trend } from "k6/metrics";
import { htmlReport } from "https://raw.githubusercontent.com/benc-uk/k6-reporter/main/dist/bundle.js";
import { textSummary } from "https://jslib.k6.io/k6-summary/0.0.1/index.js";

const fastRequests = new Counter("fast_requests");
const slowRequests = new Counter("slow_requests");
const responseTimes = new Trend("custom_response_time");

export const options = {
  scenarios: {
    stress_test_10k: {
      executor: "ramping-arrival-rate",
      startRate: 2,
      timeUnit: "1s",
      preAllocatedVUs: 500, 
      maxVUs: 10000, 
      stages: [
        { duration: "2m", target: 2600 },   // Warm up
        { duration: "2m", target: 3000 },  // Push harder
        { duration: "1m", target: 0 },     // Cool down   
      ],
    },
  },
  thresholds: {
    http_req_failed: ["rate<0.01"],
    http_req_duration: ["p(95)<500"],
  },
};

const url = "http://172.16.116.185:40354/nes.s.Web/NesFront";
const params = {
  headers: {
    "Content-Type": "application/json",
    op: "15241000",
    company: "24",
    role: "1",
    Cookie: "NESSESSION=lAfc3rC0XGXpjrzl1VF0K2I6NRGlUP",
  },
  timeout: "60s",
};

export default function () {
  const body = JSON.stringify(["CIF-11217408", "TEST_REALTIME"]);
  const startTime = new Date().getTime();
  
  const res = http.post(url, body, params);
  
  const duration = new Date().getTime() - startTime;
  responseTimes.add(duration);
  
  if (duration < 1000) {
    fastRequests.add(1);
  } else {
    slowRequests.add(1);
    console.log(`SLOW REQUEST: ${duration}ms at ${new Date().toISOString()}`);
  }
  
  check(res, { 
    "status 200": (r) => r.status === 200,
    "response < 1s": (r) => r.timings.duration < 1000,
  });
}

export function handleSummary(data) {
  return {
    // Save HTML report
    "report.html": htmlReport(data)
  };
}
```
  5k script

```js
import http from "k6/http";
import { check } from "k6";
import { Counter, Trend } from "k6/metrics";
import { htmlReport } from "https://raw.githubusercontent.com/benc-uk/k6-reporter/main/dist/bundle.js";
import { textSummary } from "https://jslib.k6.io/k6-summary/0.0.1/index.js";

const fastRequests = new Counter("fast_requests");
const slowRequests = new Counter("slow_requests");
const responseTimes = new Trend("custom_response_time");

export const options = {
  scenarios: {
    stress_test_10k: {
      executor: "ramping-arrival-rate",
      startRate: 2,
      timeUnit: "1s",
      preAllocatedVUs: 500, 
      maxVUs: 10000, 
      stages: [
        { duration: "2m", target: 500 },   // Warm up
        { duration: "2m", target: 1000 },  // Push harder
        { duration: "2m", target: 2000 },  // Even harder
        { duration: "2m", target: 5000 },  // Find the limit!
        { duration: "1m", target: 0 },     // Cool down   
      ],
    },
  },
  thresholds: {
    http_req_failed: ["rate<0.01"],
    http_req_duration: ["p(95)<500"],
  },
};

const url = "http://172.16.116.185:40354/nes.s.Web/NesFront";
const params = {
  headers: {
    "Content-Type": "application/json",
    op: "15241000",
    company: "24",
    role: "1",
    Cookie: "NESSESSION=lAfc3rC0XGXpjrzl1VF0K2I6NRGlUP",
  },
  timeout: "60s",
};

export default function () {
  const body = JSON.stringify(["CIF-11217408", "TEST_REALTIME"]);
  const startTime = new Date().getTime();
  
  const res = http.post(url, body, params);
  
  const duration = new Date().getTime() - startTime;
  responseTimes.add(duration);
  
  if (duration < 1000) {
    fastRequests.add(1);
  } else {
    slowRequests.add(1);
    console.log(`SLOW REQUEST: ${duration}ms at ${new Date().toISOString()}`);
  }
  
  check(res, { 
    "status 200": (r) => r.status === 200,
    "response < 1s": (r) => r.timings.duration < 1000,
  });
}

export function handleSummary(data) {
  return {
    "report.html": htmlReport(data)
  };
}
```
clickhouse 25 script
```js
import http from "k6/http";
import { check } from "k6";
import { Counter, Trend } from "k6/metrics";
import { htmlReport } from "https://raw.githubusercontent.com/benc-uk/k6-reporter/main/dist/bundle.js";
import { textSummary } from "https://jslib.k6.io/k6-summary/0.0.1/index.js";

const fastRequests = new Counter("fast_requests");
const slowRequests = new Counter("slow_requests");
const responseTimes = new Trend("custom_response_time");

export const options = {
  scenarios: {
    stress_test_10k: {
      executor: "ramping-arrival-rate",
      startRate: 2,
      timeUnit: "1s",
      preAllocatedVUs: 500, 
      maxVUs: 10000, 
      stages: [
        { duration: "2m", target: 10 },   // Warm up
        { duration: "2m", target: 25 },  // Push harder
        { duration: "1m", target: 0 },     // Cool down   
      ],
    },
  },
  thresholds: {
    http_req_failed: ["rate<0.01"],
    http_req_duration: ["p(95)<500"],
  },
};

const url = "http://172.16.116.185:40354/nes.s.Web/NesFront";
const params = {
  headers: {
    "Content-Type": "application/json",
    op: "15241000",
    company: "24",
    role: "1",
    Cookie: "NESSESSION=lAfc3rC0XGXpjrzl1VF0K2I6NRGlUP",
  },
  timeout: "60s",
};

export default function () {
  const body = JSON.stringify(["CIF-11217408", "TEST_REALTIME"]);
  const startTime = new Date().getTime();
  
  const res = http.post(url, body, params);
  
  const duration = new Date().getTime() - startTime;
  responseTimes.add(duration);
  
  if (duration < 1000) {
    fastRequests.add(1);
  } else {
    slowRequests.add(1);
    console.log(`SLOW REQUEST: ${duration}ms at ${new Date().toISOString()}`);
  }
  
  check(res, { 
    "status 200": (r) => r.status === 200,
    "response < 1s": (r) => r.timings.duration < 1000,
  });
}

export function handleSummary(data) {
  return {
    // Save HTML report
    "report.html": htmlReport(data)
  };
}
```

## 🔎 4. Bottleneck Identification

### 🔴 The Database "Killer" Logic (ClickHouse)
In the **25 RPS** test, the system crashed. 
- **Cause:** Running complex SQL aggregates (`JOIN`, `countIf`, and `now() - INTERVAL 30 DAY`) on every incoming request.
- **Problem:** ClickHouse is an OLAP (analytical) database. It is designed to process one giant query using all CPUs, not 10,000 tiny queries per second. Every query built a new hash table in memory, locking the I/O.
- **Resource evidence:** `docker stats` showed massive `BLOCK I/O` activity but low JBoss CPU. JBoss was not doing work; it was simply waiting (idling) for ClickHouse.

### 🟡 The Java 8 "Thread Wall"
In the **3k/5k RPS** tests (with no DB), JBoss struggled at **~2,800 RPS**.
- **Cause:** Lack of Virtual Threads.
- **Problem:** JBoss was limited by the OS Thread count. Once the "In-flight" requests reached the pool limit, new requests were queued in the TCP buffer, adding "waiting time" to the total latency.
- **Context Switching:** With 20 CPU cores trying to manage 2,000 active OS threads, the CPU overhead for "Switching" surpassed the "Processing" efficiency.

---

## 💡 5. Solution: Modern Architecture Path
To achieve the 10,000 RPS goal, the system must transition from **Sequential DB queries** to an **Asynchronous/In-Memory** model (as used in Marble and Jube).

### Step 1: Redis Integration (Real-time path)
Remove ClickHouse from the critical path of the transaction. Use Redis to maintain 10-minute and 30-day "Factor" values.
- **Old way:** `SELECT count(*)` (Slow SQL).
- **New way:** `redis.INCR(user_id)` (Fast Counter).

### Step 2: Asynchronous Model Invocation
Decouple the "Detection" from the "Logging."
- The Web API should return a response immediately after checking the **In-memory state (Redis)**.
- Use the **Background Job Manager (`jube.jobs` style)** to persist transaction details to Postgres/ClickHouse later.




