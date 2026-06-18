# Employee Exit Process — Clickable Prototype

Interactive prototype for the MIPL **Employee Exit Process** (resignation → Full & Final
settlement), built to agree the UX before implementing it on the real database.

It models the **7-stage workflow** from the SOP (`Exit_Process_Workflow.R1.docx`) and three
role views. Everything is clickable: actions change shared state, the stepper advances, and
every action is written to a **history / audit trail**.

## Files

| File | What it is |
|------|------------|
| `exit-prototype.html` | **Main clickable prototype** — Admin/HR, Manager & Employee views, working actions + undo. |
| `exit-detail-mockups.html` | Static design of the Admin + Employee detail screens (with a toggle). |
| `exit-detail-screen.html` | First static Admin detail screen. |
| `documents-section-variants.html`, `resignation-card-variants.html` | Earlier component variants. |

## How to view

Just open `exit-prototype.html` in a browser (double-click). No server needed.

- Top-right toggle switches **Admin / HR · Manager · Employee**.
- The **stepper** (7 stages) is clickable — click a stage to open it.
- **Reset** restores the starting scenario.

## The 7 stages

| # | Stage | Owner |
|---|-------|-------|
| 1 | Resignation | Employee |
| 2 | Acknowledge | Reporting Manager |
| 3 | Decision (Accept / Reject, confirm last day) | Manager + HR |
| 4 | Knowledge Transfer | Manager + Employee |
| 5 | Asset & Access Recovery | IT + Admin |
| 6 | No-Dues Clearance | IT · Admin · Finance · Manager |
| 7 | F&F & Documents (release relieving, experience & F&F statement together) | Finance + HR |

---

## Undo (step back a completed action)

**Any Admin / HR / Super Admin can undo a completed step.** In the prototype each completed
stage shows a red **Undo** button (Admin/HR only). Undo:

- **Reverts the state** for that stage (e.g. Accepted → Acknowledged, KT sign-off cleared,
  asset/no-dues sign-offs cleared, document release rolled back to draft).
- **Updates the UI everywhere** — the stepper recomputes the current stage, the Admin/Manager
  view re-renders, and **the Employee view re-derives automatically** (e.g. released documents
  go back to "Available after clearance", the progress bar moves back).
- **Is recorded in history** with **who did it** and when.

| Stage | "Undo" reverts | New status/state |
|-------|----------------|------------------|
| 3 · Decision | Acceptance | Accepted → **Acknowledged** |
| 4 · Knowledge Transfer | KT sign-off | back to **in progress** |
| 5 · Asset Recovery | all returned/▒revoked ticks | back to **pending** |
| 6 · No-Dues | all department sign-offs | back to **pending** |
| 7 · Documents | release of relieving + F&F | back to **draft (not shared)** |

> Documents also have a per-document **Roll back** (un-release a single letter) and **Regenerate**.

---

## History / Audit trail (every change is logged)

This is the core requirement: **every action is attributable.** Anyone who can view the exit
can see **who accepted, who rejected, who changed or undid** anything, and when. In the
prototype this is the **Activity** panel (Admin/Manager) and **Your progress** panel (Employee).

Each history entry records:

| Field | Example |
|-------|---------|
| `action` | `ACCEPTED`, `REJECTED`, `ACKNOWLEDGED`, `KT_SIGNED_OFF`, `NODUES_SIGNED`, `FNF_APPROVED`, `DOC_RELEASED`, `DOC_ROLLED_BACK`, `REVERT_REQUESTED`, `REVERT_APPROVED`, `REVERT_DECLINED`, **`UNDONE`** |
| `actorId` / `actorName` / `actorRole` | `748 / Deep Bhatt / HR` |
| `at` (timestamp) | `2026-06-16T14:05:00+05:30` |
| `fromState` → `toState` | `ACCEPTED → ACKNOWLEDGED` |
| `targetStage` | `3 · Decision` |
| `note` (optional) | reject reason, revert reason, etc. |

Rules:
- The trail is **append-only** — an undo does **not** delete the original action; it adds a new
  `UNDONE` entry. The timeline keeps both, so the full story is preserved.
- Every state-changing action (including undo) **notifies the employee + CC list** (in-app **and
  email**).

---

## Revert flow (employee-initiated)

Separate from Admin undo: an **employee can request to revert** their own resignation
("changed my mind"). It needs **Admin/HR approval**. In the Employee view → *Changed your mind?*
→ **Request to revert**; HR sees a banner with **Approve / Decline**. Approving steps the status
back one stage and notifies the employee. Both the request and the decision are in history.

---

## CC / Notify

Admin/HR can add anyone to a **CC list**. Those people get an in-app notification **and email**
on **every** update to the exit (accept, reject, sign-offs, document release, revert, **undo**).

---

## Implementing this on the real DB (notes for development)

This prototype is UI-only (in-memory state). For the real MIPL app (Spring backend + RN app):

### Backend
- Extend the existing `EmployeeExit` flow with the new stages (KT, Asset Recovery, No-Dues) and
  the third document (**F&F Settlement statement**).
- **Audit table** (use / extend `EmployeeExitHistory`):
  `id, exit_id, action, actor_id, actor_role, from_state, to_state, target_stage, note, created_at`.
  Append-only. Never update/delete rows.
- **Undo endpoints** (RBAC: `SUPER_ADMIN, ADMIN, HR`):
  `POST /api/resignations/{id}/undo` with `{ stage }` — validates the current state, reverts it,
  writes an `UNDONE` history row, and fires notifications. Undo from a terminal `COMPLETED`/
  `CANCELLED` state should be blocked (or guarded behind Super Admin).
- **Sign-off endpoints** per stage (KT, each No-Dues department), `FNF approve`, and
  `release documents (all together)` — each writes a history row with the actor.
- **Notifications**: on every state change (incl. undo) notify employee + CC via the existing
  `NotificationService`, and send email via the SMTP-backed `EmailService` (replace the current
  `LoggingEmailService` stub).
- **RBAC**: Admin/HR/Super-Admin = full; Manager = acknowledge, comment, KT sign-off, own No-Dues
  sign-off (view-only elsewhere); Employee = own exit, upload handover, request revert, download
  released documents.

### Frontend (React Native app)
- New `OffboardingDetailScreen` (Admin/HR + Manager) driven by the exit detail API, and the
  employee **My Exit** screen.
- Re-fetch / live-update so an Admin undo is reflected on the employee's screen.
- Surface the audit trail (history) on both.

---

*Prototype for internal review — Monarch Innovation Pvt. Ltd.*
