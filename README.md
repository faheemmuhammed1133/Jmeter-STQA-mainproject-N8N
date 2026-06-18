# n8n API Test Plan — JMeter Documentation

**File:** `n8n_test_plan.jmx`  
**JMeter Version:** 5.6.3  
**Test Plan Name:** n8n API Tests  
**Base URL:** `http://localhost:5678`

---

## Table of Contents

1. [Overview](#overview)
2. [Test Plan Structure](#test-plan-structure)
3. [Thread Groups Summary](#thread-groups-summary)
4. [Detailed Test Cases](#detailed-test-cases)
   - [TC-001 · Login with Valid Credentials](#tc-001--login-with-valid-credentials)
   - [TC-002 · Login with Wrong Password](#tc-002--login-with-wrong-password)
   - [TC-003 · Fetch AI Agent Workflow](#tc-003--fetch-ai-agent-workflow)
   - [TC-004 · Fetch Workflows Without Session](#tc-004--fetch-workflows-without-session)
   - [TC-005 · Get Executions for Workflow](#tc-005--get-executions-for-workflow)
   - [TC-006 · Load Test — Get Workflow Metadata](#tc-006--load-test--get-workflow-metadata)
5. [Shared Configuration](#shared-configuration)
6. [Result Listeners](#result-listeners)
7. [Security Notes](#security-notes)
8. [How to Run](#how-to-run)

---

## Overview

This JMeter test plan covers functional and basic load testing of the **n8n REST API**. The tests exercise:

- **Authentication** — valid login, invalid login, and unauthenticated access
- **Workflow retrieval** — fetching a specific AI-agent workflow and verifying its node composition
- **Execution history** — querying execution records filtered by workflow ID
- **Load testing** — concurrent simulation of 100 virtual users hitting the workflow endpoint

The plan is organized into four Thread Groups, each mapping to one or more test cases.

---

## Test Plan Structure

```
Test Plan: n8n API Tests
│
├── Thread Group: TC001, TC003, TC005 -- Authenticated Tests
│   ├── Cookie Manager (disabled)
│   ├── Header Manager
│   ├── TC-001  POST /rest/login          (valid credentials)
│   │   ├── Assertion: HTTP 200
│   │   └── RegexExtractor: authCookie
│   ├── Results Tree (TC001, TC003, TC005)
│   ├── TC-003  GET  /rest/workflows/{id} (fetch AI workflow)
│   │   ├── Assertion: HTTP 200
│   │   ├── Assertion: body contains "AI Agent"
│   │   └── Assertion: body contains "Google Gemini Chat Model"
│   └── TC-005  GET  /rest/executions     (filtered by workflowId)
│       ├── Assertion: HTTP 200
│       └── Assertion: body contains workflow ID
│
├── Thread Group: TC002 -- Invalid Login Test
│   ├── Header Manager
│   ├── TC-002  POST /rest/login          (wrong password)
│   │   └── Assertion: HTTP 401
│   └── Results Tree (TC002)
│
├── Thread Group: TC004 -- Unauthorized Access Test
│   ├── TC-004  GET  /rest/workflows      (no session cookie)
│   │   ├── Assertion: HTTP 401
│   │   └── Assertion: response message does NOT contain "nodes"
│   └── Results Tree (TC004)
│
└── Thread Group: TC006 -- Load Test - Workflow Metadata
    ├── Cookie Manager (hardcoded JWT)
    ├── TC-006  GET  /rest/workflows/{id} (100 users × 3 loops)
    │   └── Assertion: HTTP 200
    └── Aggregate Report
```

---

## Thread Groups Summary

| Thread Group | Threads | Ramp-Up | Loops | On Error |
|---|---|---|---|---|
| TC001, TC003, TC005 — Authenticated Tests | 1 | 1 s | 1 | Continue |
| TC002 — Invalid Login Test | 1 | 1 s | 1 | Continue |
| TC004 — Unauthorized Access Test | 1 | 1 s | 1 | Continue |
| TC006 — Load Test - Workflow Metadata | 100 | 100 s | 3 | Continue |

> **Total requests in a single full run:**  
> Functional TCs (001–005): 5 requests × 1 thread × 1 loop = **5 requests**  
> Load TC (006): 1 request × 100 threads × 3 loops = **300 requests**

---

## Detailed Test Cases

---

### TC-001 · Login with Valid Credentials

**Thread Group:** TC001, TC003, TC005 — Authenticated Tests  
**Purpose:** Verify that the n8n login endpoint accepts correct credentials and issues an authentication cookie that can be used by subsequent requests.

#### Request

| Field | Value |
|---|---|
| Method | `POST` |
| URL | `http://localhost:5678/rest/login` |
| Content-Type | `application/json` |
| Body (raw JSON) | `{"emailOrLdapLoginId": "<email>", "password": "<password>"}` |
| Follow Redirects | Yes |
| Keep-Alive | Yes |

#### Assertions

| Assertion Name | Field | Expected |
|---|---|---|
| Assert 200 — Successful Login | Response Code | `200` |

#### Post-Processor — RegexExtractor

After a successful login, the auth cookie is extracted from the `Set-Cookie` response header for use in subsequent requests within the same thread group.

| Property | Value |
|---|---|
| Extractor Name | Extract Auth Cookie |
| Source | Response Headers |
| Variable Name | `authCookie` |
| Regex Pattern | `n8n-auth=([^;]+)` |
| Template | `$1$` |
| Match Number | `1` |
| Default Value | `COOKIE_NOT_FOUND` |

The extracted value is referenced in the shared Header Manager as `Cookie: n8n-auth=${authCookie}`.

---

### TC-002 · Login with Wrong Password

**Thread Group:** TC002 — Invalid Login Test  
**Purpose:** Confirm that the login endpoint rejects incorrect credentials with an HTTP 401 response, preventing unauthorized access.

#### Request

| Field | Value |
|---|---|
| Method | `POST` |
| URL | `http://localhost:5678/rest/login` |
| Content-Type | `application/json` |
| Body (raw JSON) | `{"emailOrLdapLoginId": "<email>", "password": "WrongPassword!"}` |
| Follow Redirects | Yes |
| Keep-Alive | Yes |

#### Assertions

| Assertion Name | Field | Expected |
|---|---|---|
| Assert 401 — Wrong Credentials | Response Code | `401` |

---

### TC-003 · Fetch AI Agent Workflow

**Thread Group:** TC001, TC003, TC005 — Authenticated Tests  
**Purpose:** After logging in (TC-001), fetch a specific workflow by ID and verify the response body contains the expected AI-related node names — confirming the workflow composition is intact.

#### Request

| Field | Value |
|---|---|
| Method | `GET` |
| URL | `http://localhost:5678/rest/workflows/SmqM4KAYXK8F3FjD` |
| Auth | `Cookie: n8n-auth=${authCookie}` (inherited from Header Manager) |
| Follow Redirects | Yes |
| Keep-Alive | Yes |

#### Assertions

| Assertion Name | Field | Type | Expected |
|---|---|---|---|
| Assert 200 — workflow check | Response Code | Equals | `200` |
| Assert Contains AI Agent | Response Data | Substring | `AI Agent` |
| Assert Contains Gemini Node | Response Data | Substring | `Google Gemini Chat Model` |

> **Dependency:** This test case depends on TC-001 having run successfully and the `${authCookie}` variable being populated. It will fail with an authentication error if the login step failed or produced no cookie.

---

### TC-004 · Fetch Workflows Without Session

**Thread Group:** TC004 — Unauthorized Access Test  
**Purpose:** Validate that the workflows list endpoint rejects requests that carry no session cookie, and that no workflow data (specifically `nodes`) leaks in the error response.

#### Request

| Field | Value |
|---|---|
| Method | `GET` |
| URL | `http://localhost:5678/rest/workflows` |
| Auth | None (no Cookie header, no session) |
| Follow Redirects | Yes |
| Keep-Alive | Yes |

#### Assertions

| Assertion Name | Field | Type | Expected |
|---|---|---|---|
| Assert 401 — unauthorized access | Response Code | Equals | `401` |
| Assert No Data Leaked | Response Message | NOT Substring | `nodes` |

> The second assertion uses a **NOT Contains** check to ensure that sensitive workflow node data does not appear in the error response body when a request is unauthenticated.

---

### TC-005 · Get Executions for Workflow

**Thread Group:** TC001, TC003, TC005 — Authenticated Tests  
**Purpose:** Retrieve the most recent execution records for a specific workflow and confirm that the response references the expected workflow ID.

#### Request

| Field | Value |
|---|---|
| Method | `GET` |
| URL | `http://localhost:5678/rest/executions` |
| Query Parameters | `workflowId=SmqM4KAYXK8F3FjD`, `limit=5` |
| Auth | `Cookie: n8n-auth=${authCookie}` (inherited from Header Manager) |
| Follow Redirects | Yes |
| Keep-Alive | Yes |

#### Assertions

| Assertion Name | Field | Type | Expected |
|---|---|---|---|
| Assert 200 — Workflow check | Response Code | Equals | `200` |
| Assert Contains Workflow ID | Response Data | Substring | `SmqM4KAYXK8F3FjD` |

> **Dependency:** Relies on `${authCookie}` extracted in TC-001. The `limit=5` parameter ensures only the last 5 executions are returned.

---

### TC-006 · Load Test — Get Workflow Metadata

**Thread Group:** TC006 — Load Test - Workflow Metadata  
**Purpose:** Simulate concurrent load against the workflow fetch endpoint to assess stability and response behaviour under parallel requests. Each virtual user makes 3 repeated GET requests to the same workflow endpoint.

#### Thread Configuration

| Parameter | Value |
|---|---|
| Virtual Users (Threads) | `100` |
| Ramp-Up Period | `100 seconds` (1 new thread per second) |
| Loops per Thread | `3` |
| Total Requests | `100 × 3 = 300` |

#### Request

| Field | Value |
|---|---|
| Method | `GET` |
| URL | `http://localhost:5678/rest/workflows/SmqM4KAYXK8F3FjD` |
| Auth | Hardcoded JWT in Cookie Manager (see Security Notes) |
| Follow Redirects | Yes |
| Keep-Alive | Yes |

#### Assertions

| Assertion Name | Field | Type | Expected |
|---|---|---|---|
| Assert 200 — successful fetch | Response Code | Contains | `200` |

#### Result Listener

An **Aggregate Report** listener is attached to this group. It records, for each sampler:

- Sample count, error count, error percentage
- Average, median, 90th/95th/99th percentile response times
- Min / Max response times
- Throughput (requests/second)
- Received / Sent KB/s

---

## Shared Configuration

### Header Manager (TC001, TC003, TC005 group)

Applied to all samplers in the authenticated thread group:

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Cookie` | `n8n-auth=${authCookie}` |

### Header Manager (TC002 group)

Applied to TC-002 only:

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |

### Cookie Manager — TC006 (Load Test)

The load test group uses a **Cookie Manager** with a pre-configured `n8n-auth` cookie to avoid a login step under load. The cookie is scoped to `localhost` at path `/`.

> ⚠️ This JWT token has a fixed expiry. It must be refreshed before running the load test in a new session. See [Security Notes](#security-notes).

---

## Result Listeners

| Listener Name | Attached To | Type |
|---|---|---|
| Results Tree — TC001, TC003, TC005 | Authenticated Tests group | View Results Tree |
| Results Tree — TC002 | Invalid Login Test group | View Results Tree |
| Results Tree — TC004 | Unauthorized Access Test group | View Results Tree |
| Aggregate Report | Load Test group | Aggregate Report |

All listeners are configured to capture: timestamp, elapsed time, latency, connect time, response code, response message, thread name, bytes sent/received, success flag, and assertion results.

---

## Security Notes

> ⚠️ The following sensitive values are currently hardcoded in the `.jmx` file. Before committing this file to version control or sharing it, these should be replaced with JMeter variables or a properties file.

| Location | Sensitive Data | Recommendation |
|---|---|---|
| TC-001 request body | Email address and password | Move to a `user.properties` file or CSV Data Set Config |
| TC-002 request body | Email address | Same as above |
| TC-006 Cookie Manager | Full JWT token (n8n-auth) | Regenerate before each load test run; store externally |

**Recommended approach** — define these in a `jmeter.properties` or external CSV file and reference them as variables:

```
# user.properties
n8n.email=your-email@example.com
n8n.password=YourPassword
n8n.jwt=your-jwt-token
```

Then reference in JMeter as `${__P(n8n.email)}`, etc.

---

## How to Run

### Prerequisites

- Apache JMeter 5.6.3 or later
- n8n running locally on port `5678`
- A valid user account on the local n8n instance

### Running via JMeter GUI

1. Open JMeter.
2. Go to **File → Open** and select `n8n_test_plan.jmx`.
3. Before running, update credentials in TC-001 and TC-002 request bodies if needed.
4. For the load test (TC006), refresh the hardcoded JWT in the Cookie Manager.
5. Click the **Start** button (▶) or press `Ctrl+R`.

### Running via Command Line (non-GUI, recommended for load tests)

```bash
jmeter -n -t n8n_test_plan.jmx -l results.jtl -e -o ./report
```

| Flag | Description |
|---|---|
| `-n` | Non-GUI (headless) mode |
| `-t` | Path to the `.jmx` test plan |
| `-l` | Path to output results log (`.jtl`) |
| `-e` | Generate HTML report after run |
| `-o` | Output directory for the HTML report |

### Running specific Thread Groups only

To run only specific Thread Groups, disable the others in the JMeter GUI before saving, or use the JMeter `ThreadGroup.on_sample_error` property.

### Interpreting Results

- **Results Tree** — inspect individual request/response pairs for functional TCs.
- **Aggregate Report** — review throughput, error rate, and latency percentiles for the load test.
- **HTML Report** (generated with `-e -o`) — a full dashboard including response time graphs, APDEX score, and error summary.