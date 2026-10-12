# Sprint 2: GitHub Issue Triage Automation

**Course:** CS 590 — Low Code No Code for Rapid Prototyping  
**Author:** Kishore Vasudevan  
**Repository:** [https://github.com/KishoreVasudevan21/n8n-sprint2-issue-triage](https://github.com/KishoreVasudevan21/n8n-sprint2-issue-triage)  
**Status:** Public Repository  

---

## 📌 Project Overview
This project implements an event-driven, self-hosted issue triage automation workflow built in **n8n**. The workflow listens for real-time GitHub issue webhook events forwarded from a public GitHub repository via the GitHub CLI (`gh webhook forward`). 

Upon receiving an event, the automation:
1. **Gates non-opened actions** (ignoring edits, closures, or comments).
2. **Normalizes payloads** into a consistent schema with an idempotency key (`issue_key`).
3. **Deterministically classifies issue titles** using regular expressions (without an LLM).
4. **Enforces idempotency** through a persistent data ledger (`github_issue_ledger`) to prevent duplicate triage and route logs upon redeliveries.
5. **Logs category-specific routing entries** to an internal route log (`github_issue_route_log`).
6. **Gracefully handles and records route-log write failures** with safe error tracking.

---

## 🛠️ Architecture & Services Used
* **Event Source:** Public GitHub Repository (`KishoreVasudevan21/n8n-sprint2-issue-triage`)
* **Tunnel / Forwarding Layer:** GitHub CLI (`gh webhook forward`) forwarding events to `http://localhost:5678/webhook/sprint2-issue-triage`
* **Workflow Engine:** Self-hosted n8n instance (Docker container running at `http://localhost:5678`)
* **Persistent Ledger & State Storage:** Two n8n Data Tables:
  * `github_issue_ledger` (Stores durable audit trail, delivery timestamps, and status)
  * `github_issue_route_log` (Stores internal category routing decisions)

---

## 🔀 Workflow Node Sequence (Execution Order)

1. **`GitHub issue webhook` (Webhook Node):** Ingests incoming `POST` requests at `/webhook/sprint2-issue-triage`.
2. **`If` / `Opened action?` (IF Node):** Verifies that `{{ $json.body.action }}` equals `opened`. Non-opened actions stop immediately.
3. **`Normalize issue` (Edit Fields Node):** Extracts and standardizes issue attributes, computing `issue_key` and lowercasing the title.
4. **`Classify title` (Switch Node):** Evaluates `{{ $json.title_lower }}` against 4 deterministic regex rules.
5. **Category Route Setters (Edit Fields Nodes):**
   * `Set bug route`: sets `category = bug`, `route_name = bug-triage`
   * `Set feature route`: sets `category = feature`, `route_name = feature-requests`
   * `Set help route`: sets `category = help`, `route_name = support`
   * `Set other route`: sets `category = other`, `route_name = general-triage`
6. **Duplicate Guard (Parallel Data Table Nodes):**
   * `If ledger row exists`: checks if `issue_key` already exists. If yes ➔ branches to `Update existing last seen` to refresh `last_seen_at` without re-logging.
   * `If ledger row does not exist`: checks if `issue_key` is new. If yes ➔ branches to `Insert pending ledger row` (`route_log_status = pending`).
7. **`Intentional failure test?` (IF Node):** Checks if `{{ $json.title }}` contains `"intentional route-log failure test"`.
   * **Normal Path (False):** Routes to `Record internal route` (Data Table insert into `github_issue_route_log`). On success ➔ triggers `Update route-log recorded` (updates ledger to `recorded`). On error ➔ routes to `Update route-log failed`.
   * **Simulated Failure Path (True):** Routes to `Simulate route-log failure` (Code node throwing error) ➔ error output triggers `Update route-log failed` (updates ledger to `failed` with safe error note).

---

## 📋 Field Mapping Table

| Webhook Source Path | Normalized Field (`Normalize issue`) | Ledger Column (`github_issue_ledger`) | Description |
| :--- | :--- | :--- | :--- |
| `body.repository.full_name` | `repository_full_name` | — | Repository identifier (e.g. `KishoreVasudevan21/n8n-sprint2-issue-triage`) |
| `body.issue.number` | `issue_number` | — | Issue number integer |
| *Calculated Expression* | `issue_key` | `issue_key` | Unique Idempotency Key: `repo_full_name#issue_number` |
| `body.issue.html_url` | `issue_url` | `issue_url` | Clickable URL to GitHub issue |
| `body.issue.title` | `title` | `title` | Original issue headline |
| `body.issue.title.toLowerCase()` | `title_lower` | — | Lowercased title for regex matching |
| `body.issue.user.login` | `reporter` | `reporter` | GitHub username of issue creator |
| `body.issue.created_at` | `opened_at` | `opened_at` | ISO creation timestamp |
| `{{ $now.toISO() }}` | `last_seen_at` | `last_seen_at` | Timestamp of current webhook processing |
| Fixed / Dynamic | `category` | `category` | Classified issue bucket (`bug`, `feature`, `help`, `other`) |
| Fixed / Dynamic | `route_name` | `route_name` | Routing label (`bug-triage`, `feature-requests`, `support`, `general-triage`) |
| Fixed / State | `route_log_status` | `route_log_status` | Status lifecycle (`pending` ➔ `recorded` or `failed`) |
| Fixed / Error | `error_note` | `error_note` | Diagnostics note on failure; blank on success |

---

## 🎯 Deterministic Classification Rules

Classification uses strict regular expressions matching word boundaries (`\b`), ensuring explainable and deterministic behavior:

| Category | Route Name | Regex Rule | Example Matches |
| :--- | :--- | :--- | :--- |
| **`bug`** | `bug-triage` | `\b(bug\|error)\b` | "Bug: blank screen", "System error on load" |
| **`feature`** | `feature-requests` | `\b(feature\|request)\b` | "Feature request: export CSV", "New feature" |
| **`help`** | `support` | `\b(help\|question)\b` | "Help: change webhook", "Quick question about config" |
| **`other`** | `general-triage` | *Fallback (No match)* | "Documentation: clarify local setup" |

---

## 🚀 Setup & Local Execution Guide

1. **Start n8n Stack:** Ensure the self-hosted n8n Docker container is running locally at `http://localhost:5678`.
2. **Import & Activate Workflow:** Import `sprint2_github_issue_triage.json` into n8n and toggle **Active** to `ON`.
3. **Start GitHub CLI Webhook Forwarding:**
   ```bash
   gh webhook forward \
     --repo=KishoreVasudevan21/n8n-sprint2-issue-triage \
     --events=issues \
     --url=http://localhost:5678/webhook/sprint2-issue-triage
   ```
4. **Verify Live Processing:** Create test issues in the repository and monitor real-time execution in the n8n **Executions** tab.
5. **Testing Duplicate Redelivery:** If testing redelivery with local CLI forwarding, note that GitHub UI redelivery returns `Unauthorized` due to proxy limitations. Redelivery can be verified directly via terminal command by replaying the payload:
   ```bash
   curl -X POST -H "Content-Type: application/json" -d @test1_payload.json http://localhost:5678/webhook/sprint2-issue-triage
   ```
