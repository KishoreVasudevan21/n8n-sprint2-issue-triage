# Operational Runbook: GitHub Issue Triage Automation

**Service:** GitHub Issue Triage (`Sprint 2 — GitHub Issue Triage`)  
**Repository:** [https://github.com/KishoreVasudevan21/n8n-sprint2-issue-triage](https://github.com/KishoreVasudevan21/n8n-sprint2-issue-triage)  
**Host Environment:** Self-hosted n8n (Docker container at `http://localhost:5678`)  

---

## 1. Purpose
This runbook provides operating procedures, maintenance instructions, failure diagnosis, and recovery actions for the event-driven GitHub Issue Triage workflow. The service ingests issue webhooks, categorizes tickets deterministically, ensures duplicate safety via a persistent ledger, and records routing outcomes into an internal route log table.

---

## 2. Normal Operation

### Ingestion & Processing
* **Trigger:** An `issues.opened` webhook is forwarded via GitHub CLI to the local webhook path `/webhook/sprint2-issue-triage`.
* **Action:**
  1. The workflow verifies `action == "opened"`.
  2. The issue title is converted to lowercase and matched against deterministic regex rules to assign a category (`bug`, `feature`, `help`, `other`) and route name (`bug-triage`, `feature-requests`, `support`, `general-triage`).
  3. The workflow performs a duplicate check on `github_issue_ledger`.
  4. If new, it inserts a ledger record with `route_log_status = pending`.
  5. It attempts an insert into `github_issue_route_log`.
  6. Upon successful route write, it updates the ledger record to `route_log_status = recorded` and clears `error_note`.
* **Expected Result:**
  * One execution in n8n marked **Success** (green).
  * Exactly one corresponding row in `github_issue_ledger` with `route_log_status = recorded`.
  * Exactly one corresponding row in `github_issue_route_log` containing `issue_key`, `category`, and `route_name`.
* **Evidence Location:** Inspect `test_evidence.md` (Tests 1–4) and the n8n Data Tables view.

---

## 3. Duplicate Delivery

### Handling Retried or Redelivered Webhooks
* **Scenario:** Network glitches or delivery retries cause the exact same `issues.opened` event to be delivered multiple times.
* **Operating Note on CLI Forwarding & UI Redelivery:** When forwarding GitHub webhooks through the GitHub CLI (`gh webhook forward`) to a local environment, redelivering via the GitHub web UI (**Settings** ➔ **Webhooks** ➔ **Recent deliveries** ➔ **Redeliver**) will fail with an HTTP 401 `Unauthorized` response. This is because GitHub's cloud redelivery proxy cannot reach or authenticate against the ephemeral local CLI tunnel.
* **Operator Action for Redelivery Verification:** 
  To test or simulate redelivery under local CLI forwarding without creating a duplicate issue on GitHub:
  1. Retrieve the original event payload JSON (e.g. from the n8n execution history or payload archive).
  2. Send the exact payload from the terminal:
     ```bash
     curl -X POST -H "Content-Type: application/json" \
       -d @payload.json \
       http://localhost:5678/webhook/sprint2-issue-triage
     ```
* **Workflow Concrete Action:**
  1. The workflow receives the payload and calculates `issue_key = repository_full_name + "#" + issue_number`.
  2. The duplicate guard node `If ledger row exists` finds an existing record in `github_issue_ledger`.
  3. The workflow branches to `Update existing last seen`, updating the row's `last_seen_at` timestamp.
  4. The workflow **terminates this branch** and does NOT execute any write to `github_issue_route_log`.
* **Expected Result:**
  * Ledger row `last_seen_at` reflects the newest delivery time.
  * No duplicate record is added to `github_issue_route_log` (row count remains exactly 1).
  * No duplicate alert or downstream action is triggered.
* **Evidence Location:** See `test_evidence.md` (Test 5: Redelivery Test).

---

## 4. Route-Log Failure

### Fault Detection
* **Scenario:** A database timeout, lock, network interruption, or table schema error prevents writing to `github_issue_route_log`.
* **Concrete Action:**
  1. The node `Record internal route` fails. Because its error policy is set to `Continue (using error output)`, the failure does not crash the entire workflow engine.
  2. The error output executes `Update route-log failed`.
  3. The ledger row in `github_issue_ledger` is updated with `route_log_status = failed` and `error_note = "Route-log write failed; review the route-log node configuration and execution details."`.
* **Expected Result:**
  * The ledger entry persists in `github_issue_ledger` with visible status `failed`.
  * The event does not silently vanish from the audit trail.
  * No partial/corrupted entry is created in `github_issue_route_log`.
* **Evidence Location:** See `test_evidence.md` (Test 7: Simulated Failure Test).

---

## 5. Recovery Action

### Step-by-Step Operator Recovery Procedure
When a ledger row is discovered with `route_log_status = failed`:

1. **Locate Failed Records:**
   * Open n8n ➔ **Data Tables** ➔ **`github_issue_ledger`**.
   * Filter or sort by `route_log_status = failed`. Identify the affected `issue_key` (e.g. `KishoreVasudevan21/n8n-sprint2-issue-triage#28`).
2. **Inspect Execution Logs:**
   * Open n8n ➔ **Executions** tab.
   * Open the execution matching the failure timestamp.
   * Inspect the `Record internal route` error message to determine the root cause (e.g. table locks, storage exhaustion, or schema mismatch).
3. **Resolve Storage / Connectivity Issue:**
   * Verify that the n8n container and database have sufficient disk space and valid table permissions.
4. **Manual Route-Log Recovery Insertion:**
   * Once connectivity is restored, open **Data Tables** ➔ **`github_issue_route_log`**.
   * Add a row manually (or trigger a recovery replay) with the issue's `issue_key`, `category`, `route_name`, and timestamp.
5. **Update Ledger Status:**
   * In `github_issue_ledger`, update the issue's `route_log_status` from `failed` to `recorded`, and clear `error_note`.

---

## 6. Safe Shutdown

To decommission or pause the triage automation safely without leaving hanging background listeners:

1. **Stop Forwarding CLI Process:**
   * In the Terminal running `gh webhook forward`, press `Ctrl + C`. This immediately stops forwarding local port traffic.
2. **Deactivate n8n Workflow:**
   * Open n8n editor for `Sprint 2 — GitHub Issue Triage`.
   * Switch the **Active** toggle to `OFF` (Grey). This unregisters the production webhook route `/webhook/sprint2-issue-triage` and prevents unsolicited requests.
3. **Verify GitHub Repository Webhook State:**
   * Go to repository **Settings** ➔ **Webhooks**.
   * Confirm the temporary forwarding hook is either deleted by GitHub CLI or disabled.
4. **Audit Audit Tables:**
   * Confirm all rows in `github_issue_ledger` are accounted for in `github_issue_route_log`.

---

## 7. What This Prototype Does Not Test

While this prototype provides a robust proof-of-concept for issue triage, production enterprise deployments require additional capabilities not covered in this sprint:
* **High-Concurrency Webhook Bursts:** The local SQLite/Postgres container setup does not test queue backpressure when hundreds of issues are opened in a single second. An enterprise system would place an Amazon SQS or RabbitMQ buffer in front of n8n.
* **Multi-Repository Event Routing:** The workflow assumes single-repository ownership (`KishoreVasudevan21/n8n-sprint2-issue-triage`). Handling enterprise multi-org webhooks would require dynamic routing tables keyed by organization ID.
* **Issue Updates & Lifecycle Tracking:** The workflow intentionally discards `action == "edited"` or `action == "closed"`. A production system would track issue re-openings, state transitions, and comment threads.
* **Automated External Alerts:** Route logs are stored purely in local n8n Data Tables rather than dispatching real-time notifications to PagerDuty, Slack channels, or Jira queues.
