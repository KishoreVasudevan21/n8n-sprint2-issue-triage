# Test Evidence & Verification Report

**Service:** GitHub Issue Triage (`Sprint 2 — GitHub Issue Triage`)  
**Repository:** [https://github.com/KishoreVasudevan21/n8n-sprint2-issue-triage](https://github.com/KishoreVasudevan21/n8n-sprint2-issue-triage)  
**Verification Date:** October 11, 2026  

This document presents the complete test evidence, verification logs, and observable results for all 7 required test cases specified in the Sprint 2 rubric.

---

## 📊 Summary of Test Results

| Test # | Test Case / Scenario | Input Title | Expected Category | Expected Route | Expected Ledger Status | Actual Outcome |
| :---: | :--- | :--- | :---: | :---: | :---: | :---: |
| **1** | Bug Classification | `Bug: blank screen after logout` | `bug` | `bug-triage` | `recorded` | ✅ PASSED |
| **2** | Feature Request | `Feature request: export task list to CSV` | `feature` | `feature-requests` | `recorded` | ✅ PASSED |
| **3** | Support / Question | `Help: change the public webhook address` | `help` | `support` | `recorded` | ✅ PASSED |
| **4** | General Triage (Fallback) | `Documentation: clarify local setup` | `other` | `general-triage` | `recorded` | ✅ PASSED |
| **5** | Idempotent Redelivery | Redeliver Test 1 payload | `bug` | `bug-triage` | `recorded` (Timestamp updated) | ✅ PASSED |
| **6** | Action Gate Test | Edit Test 4 issue body | — | — | *Ignored (No write)* | ✅ PASSED |
| **7** | Route-Log Write Failure | `Documentation: intentional route-log failure test` | `other` | `general-triage` | `failed` (Error note logged) | ✅ PASSED |

---

## 🧪 Detailed Test Case Evidence

### Test 1: Bug Classification (`bug` ➔ `bug-triage`)
* **Issue Title:** `Bug: blank screen after logout`
* **Issue Body:** `After logging out successfully, the application shows a blank screen instead of the sign-in page. Expected result: return to sign-in.`
* **Observed Execution:**
  * Webhook payload received with `action = opened`.
  * Regex rule `\b(bug|error)\b` matched successfully.
  * Node `Set bug route` assigned `category = bug` and `route_name = bug-triage`.
  * One ledger row inserted into `github_issue_ledger` with `route_log_status = recorded`.
  * Exactly one entry created in `github_issue_route_log` with `route_name = bug-triage`.
* **Visual Evidence:**
  ![Test 1 Execution](evidence_images/image1.png)
  ![Test 1 Routing Evidence](evidence_images/image2.png)
  ![Test 1 Ledger & Route Log Record](evidence_images/image3.png)

---

### Test 2: Feature Request Classification (`feature` ➔ `feature-requests`)
* **Issue Title:** `Feature request: export task list to CSV`
* **Issue Body:** `Please add a CSV export so a user can download the current task list for analysis.`
* **Observed Execution:**
  * Regex rule `\b(feature|request)\b` matched successfully.
  * Node `Set feature route` assigned `category = feature` and `route_name = feature-requests`.
  * Ledger row created and updated to `route_log_status = recorded`.
  * One entry created in `github_issue_route_log` with `route_name = feature-requests`.
* **Visual Evidence:**
  ![Test 2 Execution](evidence_images/image4.png)
  ![Test 2 Route Log Entry](evidence_images/image5.png)
  ![Test 2 Ledger Row](evidence_images/image6.png)

---

### Test 3: Help / Support Classification (`help` ➔ `support`)
* **Issue Title:** `Help: change the public webhook address`
* **Issue Body:** `I need guidance on updating the automation's public endpoint after moving to a new host.`
* **Observed Execution:**
  * Regex rule `\b(help|question)\b` matched successfully.
  * Node `Set help route` assigned `category = help` and `route_name = support`.
  * Ledger row created and updated to `route_log_status = recorded`.
  * One entry created in `github_issue_route_log` with `route_name = support`.
* **Visual Evidence:**
  ![Test 3 Execution](evidence_images/image7.png)
  ![Test 3 Route Log Entry](evidence_images/image8.png)
  ![Test 3 Ledger Row](evidence_images/image9.png)

---

### Test 4: Documentation / Fallback Classification (`other` ➔ `general-triage`)
* **Issue Title:** `Documentation: clarify local setup`
* **Issue Body:** `The setup guide should state which local port is used by the development server.`
* **Observed Execution:**
  * No specific keyword matched Rules 1–3; Switch node routed to Fallback Rule 4.
  * Node `Set other route` assigned `category = other` and `route_name = general-triage`.
  * Ledger row created and updated to `route_log_status = recorded`.
  * One entry created in `github_issue_route_log` with `route_name = general-triage`.
* **Visual Evidence:**
  ![Test 4 Execution](evidence_images/image10.png)
  ![Test 4 Route Log Entry](evidence_images/image11.png)
  ![Test 4 Ledger Row](evidence_images/image12.png)

---

### Test 5: Idempotent Redelivery Test (Duplicate Safety)
* **Scenario:** The exact Test 1 webhook payload (`KishoreVasudevan21/n8n-sprint2-issue-triage#27`) was redelivered to verify idempotency.
* **Redelivery Method & Technical Observation:** When attempting to redeliver Test 1 via GitHub's web interface (**Settings** ➔ **Webhooks** ➔ **Recent deliveries** ➔ **Redeliver**), GitHub's backend proxy returned an `Unauthorized` response (`X-Github-Backend: Kubernetes`, `Body: Unauthorized`). This occurs because GitHub's cloud-side redelivery proxy cannot authenticate or tunnel back through an ephemeral `gh webhook forward` CLI session forwarding to `localhost:5678`. To faithfully test redelivery without creating an illegitimate second issue, the exact identical webhook payload from Test 1 (`issue #27`, `action: opened`) was redelivered directly to the n8n webhook endpoint via terminal command (`curl -X POST -H "Content-Type: application/json" -d @test1_payload.json http://localhost:5678/webhook/sprint2-issue-triage`).
* **Observed Execution:**
  * The node `If ledger row exists` detected the existing `issue_key` (`KishoreVasudevan21/n8n-sprint2-issue-triage#27`).
  * The execution branched exclusively to `Update existing last seen`.
  * **Ledger Result:** The row's `last_seen_at` timestamp was updated to reflect the new redelivery time.
  * **Route-Log Result:** The workflow terminated this branch without reaching `Record internal route`. Exactly **one total row** remained in `github_issue_route_log` for this issue. Zero duplicate entries created!
* **Visual Evidence:**
  ![Redelivery Execution Flow](evidence_images/image13.png)
  ![Redelivery Ledger Timestamp Update](evidence_images/image14.png)
  ![Proof of Zero Duplicate in Route Log](evidence_images/image15.png)

---

### Test 6: Action Gate Test (Non-Opened Action Ignored)
* **Scenario:** The description body of Test 4 was edited on GitHub to trigger an `issues.edited` webhook.
* **Observed Execution:**
  * Webhook payload arrived with `action = edited`.
  * The node `If` (`Opened action?`) evaluated `{{ $json.body.action }} == "opened"` as **False**.
  * The execution stopped immediately at the IF node.
  * Zero reads or writes were performed on `github_issue_ledger` or `github_issue_route_log`.
* **Visual Evidence:**
  ![Edited Event Action Gate Execution](evidence_images/image16.png)
  ![Unchanged Table State](evidence_images/image17.png)

---

### Test 7: Simulated Route-Log Failure Path
* **Issue Title:** `Documentation: intentional route-log failure test`
* **Issue Body:** `Temporary course test of the internal route-log failure path. No user action is required.`
* **Observed Execution:**
  * The node `Insert pending ledger row` inserted a record with `route_log_status = pending`.
  * The node `Intentional failure test?` matched the title and routed True to `Simulate route-log failure`.
  * The Code node threw `throw new Error('Intentional course test: route-log failure');`.
  * The error output routed to `Update route-log failed`.
  * **Ledger Result:** Row updated to `route_log_status = failed` with safe diagnostic note: `"Route-log write failed; review the route-log node configuration and execution details."`.
  * **Route-Log Result:** Zero rows created in `github_issue_route_log`.
* **Visual Evidence:**
  ![Simulated Failure Execution](evidence_images/image18.png)
  ![Failed Ledger Row Evidence](evidence_images/image19.png)
  ![Absence of Route Log Row](evidence_images/image20.png)

---

## 📋 Final Persistent Data Table Records

### Final `github_issue_ledger` Table
The durable ledger shows all 5 processed test issues, their categories, timestamps, and route log statuses (including the intentional failure):
![Final github_issue_ledger Table 1](evidence_images/image21.png)
![Final github_issue_ledger Table 2](evidence_images/image22.png)

### Final `github_issue_route_log` Table
The internal route log shows exactly 4 successful route classifications (`bug-triage`, `feature-requests`, `support`, `general-triage`), confirming that neither the redelivery nor the intentional failure produced invalid rows:
![Final github_issue_route_log Table](evidence_images/image23.png)
![Final Data Tables Overview](evidence_images/image24.png)
