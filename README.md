Employee Leave Request Approval System (n8n)

This is an end-to-end, no-code automation that lets employees submit leave requests via a form, automatically validates them against leave balance and existing bookings, routes them to a manager for approval, and updates records and notifies the employee. This is done without a single line of AI/LLM involved.

Why does this exist?

Manual leave approval usually means: an email or paper form, a manager digging through a spreadsheet to check if the dates overlap with existing leave or if the employee has enough days left, then someone updating a tracker by hand. This workflow removes every manual step except the manager's actual approve/reject decision.

How does it work?

1. Employee submits a request via an n8n Form (Employee ID, Start Date, End Date, Leave Type, Reason). Required fields and structured inputs (date pickers, dropdown) to validate and prevent bad data at the source.
2. Prepare Request Data — normalizes the form submission into a consistent shape for the rest of the workflow.
3. Get Leave Balance / Check Employee Found — looks up the employee's current balance in a Google Sheets tracker (Total Days, Days Used, Days Remaining) and confirms the Employee ID is valid before continuing.
4. Get Existing Requests — pulls all of that employee's prior leave records from the request log.
5. Check Balance and Overlap (Code node) — the core business logic:
   - Loops through every existing request to check whether the new date range overlaps any approved/pending leave.
   - Calculates requested days and compares against Days Remaining.
   - A Code node was necessary here rather than an IF node because the check requires iterating over a variable-length list of existing records, something IF nodes (single fixed comparisons) can't do.
6. Check can Proceed — branches on the result:
   - Pass: appends the new request to the sheet as "Pending" and emails the manager an approve/reject link.
   - Fail: returns a clear rejection reason (insufficient balance or overlapping dates) to the employee immediately, no manager involvement needed.
7. Wait For Manager Response — pauses execution and generates a unique resumable webhook URL. This is the key architectural decision: since manager approval can take hours or days, the workflow can't hold open the original HTTP connection. The Wait node suspends the entire execution to disk and exits, consuming no resources, until the manager's link is clicked.
8. Merge Resumed Data — when the webhook resumes, only the decision (`action=approve` / `action=reject`) comes in fresh via query parameter. This node reconstructs the rest of the original request (Request ID, Employee ID, dates, leave type) by referencing the frozen data from the Check Balance and Overlap node's output earlier in the same execution, there is no need to re-query the sheet.
9. Check Manager Action (IF node) — branches on the manager's decision:
   - Approved: updates the request status to "Approved," deducts the days from the employee's balance, and notifies the employee.
   - Rejected: marks the request "Rejected" and notifies the employee.

## Key design decisions

**| Decision | Why |**
**Decision**: Form Trigger over direct Sheet entry. **Why**: Enforces required fields and structured input; keeps the Sheet as a system-owned data store, not a shared editable surface
**Decision:** Code node for balance/overlap check. **Why:** Needed to loop over an unknown number of existing records, not a single fixed condition.
**Decision:** Wait + Webhook instead of Respond to Webhook. **Why:** The original form submission's HTTP connection can't stay open for days; Wait creates an independent, durable resume point.
**Decision:** Minimal query params on the approval link. **Why:** Avoids fragile, oversized URLs; only the decision is passed live, everything else is reconstructed from the paused execution's own data.
**Decision:** Google Sheets as the data store. **Why:** Zero-setup, and lets non-technical stakeholders (HR, managers) view/audit records directly without needing database access.

Tech stack

- n8n (workflow orchestration)
- Google Sheets (leave balance tracker + request log)
- Gmail (manager and employee notifications)
- n8n Code node (JavaScript — balance/overlap validation logic)
- n8n Wait + Webhook nodes (asynchronous human-in-the-loop approval)

Screenshots

- Employee-facing submission form
- Full workflow canvas
- Emails (Rejection email, Approval Email, Manager email to reject to approve)
- Leave request log (Google Sheets)
- Leave balance tracker (Google Sheets)
- Rejection message shown to employee on failed validation

Notes

This is a pure form or logic-based automation, that is deliberately built without any AI/LLM component, to demonstrate structured business-process automation as distinct from AI-agent workflows.
