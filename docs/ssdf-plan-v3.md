# SSDF fund app: design plan v3

## Context
SSDF is an alumni-funded welfare fund for students of a tier-2 city college. Students claim money for academic, travel, living, digital, health, career and emergency needs. Small claims are approved automatically. Volunteer **approvers** decide the rest. **Sponsors** fund and control the money. **Admin** manages student records only. Every action and rupee can be traced.

This v3 replaces v1 (role split, sign-up, categories, payments, survey) and v2 (approver role, passing year, real-time bill rule, traceability, fraud controls, privacy, operations). It is a design document; no code exists yet.

**Decisions made**
- Approvers decide every claim that isn't auto-approved. The sponsor handles money and settings only.
- The sponsor creates approvers and sets their limits. A new **Owner** role creates sponsors.
- Sign-up uses **passing year**. Year of study is calculated, not stored.
- **Bill rule:** advance claims need a bill within 14 days (configurable). After that, new claims are blocked until the bill is uploaded. The block is checked in real time.
- **Traceability:** a write-once audit log, a timeline for every claim, a double-entry money ledger, versioned documents and versioned rules.

Values marked **[default]** are proposals the team must confirm. They are all editable in settings.

---

## 1. Roles and permissions

```
                              ┌─────────┐
                              │  OWNER  │  founder(s); creates sponsors,
                              └────┬────┘  global settings, 2 owners for recovery
                                   │
                              ┌────▼────┐
                              │ SPONSOR │  money + rules; creates approvers
                              └────┬────┘
              ┌────────────────────┼─────────────────────┐
         ┌────▼─────┐         ┌────▼────┐           ┌────▼────┐       ┌────────┐
         │ APPROVER │         │ STUDENT │           │  ADMIN  │       │ PUBLIC │
         │volunteer │         └─────────┘           └─────────┘       └────────┘
         └──────────┘                              (created by owner)  (no login)
```

| Action | Student | Approver | Sponsor | Admin | Owner | Public |
|---|---|---|---|---|---|---|
| Sign up, edit own profile and bank details | ✔ | | | | | |
| Submit claim, upload bill, appeal, view own timeline | ✔ | | | | | |
| Approve / reject / request info (within own limit) | | ✔ | fallback | | | |
| Verify bills | | ✔ | ✔ | | | |
| See student name, ID, department, year | own | ✔ | ✔ | ✔ | | |
| See student mobile, email, full bank number | own | ✖ (last 4 digits) | ✔ | ✖ | | |
| See ID card image | own | | | during enrolment | | |
| Edit student records (name, ID, department, passing year, status) | | | | ✔ | | |
| Confirm enrolment, import roll list | | | | ✔ | | |
| Categories, amounts, caps, auto-approve rule, grace days | | | ✔ | | | |
| Pool top-up, payouts, refunds, write-offs, reconciliation | | | ✔ | | | |
| Create / limit / deactivate approvers | | | ✔ | | | |
| Create sponsors and admins, global settings | | | | | ✔ | |
| Full audit log, reports | | own activity | ✔ | own actions | ✔ | |
| Anonymous survey | ✔ | | | | | ✔ |

- Approvers, sponsors, admins and owners log in by invite only, with password + OTP. A fresh OTP is required before approving claims, moving money or changing records.
- Before their first login, approvers accept a confidentiality agreement. The agreement is versioned and the acceptance is logged.

---

## 2. Student sign-up and the passing year

```
 Fill form: name, student ID, department, PASSING YEAR,
 college email, mobile, password
        │
        ▼
 Passing year in valid range? ─ No ─► "For a 4-year course, enter 2027–2030"
        │ Yes                         (range = this academic year's end
        ▼                               … + course length − 1, per department)
 OTP on mobile + email (rate-limited)
        │
        ▼
 Upload student ID card
        │
        ▼
 Matches the college roll list (CSV imported by admin)?
        │ Yes                                  │ No
        ▼                                      ▼
 Admin checks ID image only            Admin checks every field + ID image
        │                                      │
        └──────────────┬───────────────────────┘
                 Approve │  Reject ─► student told why, can fix and resend
                         ▼
                 Status: ACTIVE
```

**Year of study**
- Calculated, not stored: `year = course length − (passing year − academic year end)`.
- The academic year starts in June [default].

| Example (Nov 2026, academic year ends 2027, 4-year course) | Year |
|---|---|
| Passing 2027 | 4 |
| Passing 2028 | 3 |
| Passing 2029 | 2 |
| Passing 2030 | 1 |

- **Every June:** allowances reset, and eligible categories update automatically.
- **Year-back, detention or lateral entry:** admin edits the passing year and gives a reason. The change is audited.
- **After the passing year's academic year ends:** status becomes **GRADUATED**. The student can't make new claims but must still close open bills.

**Bank account**
- Required only before the first claim paid to the student.
- Not needed for claims paid directly to an institution.

**Student statuses**

| Status | Meaning |
|---|---|
| PENDING | Waiting for enrolment confirmation |
| ACTIVE | Can claim |
| SUSPENDED / HIDDEN | Set by admin. Can't log in or claim. Payouts already in progress still finish, and bills are still due. |
| GRADUATED | Set automatically. No new claims; open bills still due. |
| DROPPED OUT | Set by admin. No new claims; open bills still due. |

---

## 3. Categories and claim types

Each category has:
- Name, group and what it covers
- Support type: monthly, per semester, per year, one-time or per incident
- Min, Typical and Max amounts (₹)
- Eligible years of study
- **Payee type:** student or institution
- **Allowed claim types**

```
                       Student picks category
                                │
                 ┌──────────────┴──────────────┐
            INSTITUTION                     STUDENT
     (tuition, exam, hostel, mess —   (travel, books, laptop,
      registered payees only)          health, emergency…)
                 │                             │
                 ▼                   Already paid and has a bill?
      Pay college/hostel directly       Yes │            │ No
      Proof: official fee slip              ▼            ▼
      Institution receipt = bill     REIMBURSEMENT    ADVANCE
      → SETTLED, no bill clock       bill attached     paid first,
                                     at claim time     bill due in 14 days
                                     → SETTLED after   (section 7)
                                       verification
```

- **Institution payees:** the sponsor keeps a list of college and hostel accounts, each verified once.
- **Recurring monthly claims** (hostel, mess, bus pass, internet):
  1. After the first month is approved, the student turns on "repeat monthly" (up to semester end).
  2. Each month a claim is created automatically, as a reimbursement with that month's receipt, or paid to the institution.
  3. The same amount is auto-approved. A different amount goes through normal approval.

**Caps**
- Per category: remaining allowance and the category Max.
- Per student per year: ₹50,000 [default] across all categories. **This is a hard block on the form.**
- Emergency is outside the yearly cap and has its own cap of ₹25,000 per year [default].
- Allowance moves through three stages:

| When | Allowance |
|---|---|
| Claim approved | RESERVED |
| Claim paid | USED |
| Rejected, cancelled, expired or money returned | RELEASED |

- A claim counts in the academic year it was **submitted** in.

---

## 4. Making a claim (real-time checks)

```
 Student taps "Claim fund"
        │
        ▼
 ┌────────────────────────────────────────────────────────────┐
 │ SERVER CHECK — runs live on open AND again on submit       │
 │  ✖ Status not ACTIVE                                       │
 │  ✖ Any advance claim PAID with bill overdue (> 14 days)    │
 │  ✖ Any bill MISMATCH not fixed within 14 days              │
 │  ✖ Yearly cap reached                                      │
 │  ✖ Bank details missing or unverified (student-paid only)  │
 └─────────────────────┬──────────────────────────────────────┘
        blocked ◄──────┴──────► allowed
           │                       │
           ▼                       ▼
 Clear message + action      1. Category (eligible ones only)
 e.g. "Upload bill for       2. Claim type (advance / reimbursement / institution)
 SSDF-2026-000123 to make    3. Purpose, period, amount (≤ allowance, Max, yearly cap)
 new requests" [Upload]      4. Documents (PDF/JPG/PNG, size limit, virus scan)
                             5. Optional item list (auto total)
 Emergency category is       6. Declaration + consent (time + version saved)
 never blocked by bills      7. Save draft / review / submit
                                       │
                                       ▼
                            Ref no. SSDF-2026-000123 → section 5
```

---

## 5. Routing and approval

```
                          Claim submitted
                                │
                                ▼
                  ┌───────────────────────────┐
                  │ FRAUD CHECKS              │
                  │ • same file hash used?    │
                  │ • same bill no.+vendor+₹? │
                  │ • student flagged (past   │
                  │   write-off)?             │
                  └─────┬───────────────┬─────┘
                   hit  │               │ clear
                        ▼               ▼
                 FLAGGED (red     ┌─────────────────────────────────┐
                 banner, reason)  │ AUTO-APPROVE if ALL true:       │
                        │         │ • amount ≤ Typical              │
                        │         │ • amount ≤ ₹10,000 (2-approver  │
                        │         │   threshold) — hard ceiling     │
                        │         │ • documents present             │
                        │         └──────┬───────────────┬──────────┘
                        │             yes│               │no
                        │                ▼               │
                        │          AUTO-APPROVED         │
                        │          (5% random sample     │
                        │           post-checked later)  │
                        ▼                                ▼
                 ┌───────────────────────────────────────────────┐
                 │ ASSIGN: round-robin to an active approver     │
                 │ with limit ≥ amount, under daily cap ₹50,000, │
                 │ no conflict. None available → SPONSOR.        │
                 │ Emergency: all approvers notified, SLA 8 h.   │
                 │ Others: SLA 3 days, then reassigned + sponsor │
                 │ alerted.                                      │
                 └───────────────────────┬───────────────────────┘
                                         ▼
                                Approver reviews
        ┌─────────────┬─────────────┬────┴──────────┬────────────────┐
        ▼             ▼             ▼               ▼                ▼
     APPROVE      PARTLY        REJECT        REQUEST INFO       CONFLICT
        │         APPROVE       (reason)      → student edits    → reassigned
        │            │             │            → same approver
        │            ▼             ▼
        │     Student accepts  Student may APPEAL once in 7 days
        │     or cancels in    → DIFFERENT approver (or sponsor)
        │     7 days (no reply → final decision
        │     = accept)
        ▼
 Amount > ₹10,000? ── yes ─► SECOND APPROVAL by a different approver
        │ no                        │
        ▼                           ▼
                  APPROVED → section 6
```

- A sponsor report shows each approver's approvals, average amount, approval rate and overrides. It is sent weekly.
- **Bill verification** is done by approvers, using the same assignment rules.

---

## 6. Money: top-ups and payouts

```
  MONEY IN                               MONEY OUT
  ─────────                              ──────────
  Sponsor → "Add funds"                  Claim APPROVED
       │                                      │
       ▼                                      ▼
  Gateway checkout (e.g. Razorpay)      Pool has enough? ─ No ─► WAITING FOR FUNDS
       │                                      │ yes               priority when funds arrive:
       ▼                                      │                   1 Emergency
  Signed webhook confirms                     │                   2 Institution fee with due date
       │                                      │                   3 Oldest approved
       ▼                                      ▼
  Ledger +TOP-xxxx                      Ledger: RESERVE amount
  Receipt PDF (80G if registered)             │
                                              ▼
                                  Payee = institution? ─► payout to institution account
                                  Payee = student      ─► bank changed < 48 h? ─► HOLD
                                              │                                  until cooling-off ends
                                              ▼
                                  Gateway payout (idempotency key, limited retries)
                                     success │             │ failed
                                             ▼             ▼
                                    PAID, ledger −PAY   PAYOUT PENDING → fix bank / retry
                                    receipt PDF         → EXPIRED after 30 days
                                    advance → bill        (reservation released)
                                    clock starts
```

**Money rules**
- The pool balance is always the sum of the ledger. It is never typed in.
- Gateway fees are recorded in the ledger as fund expenses.
- Refunds and reversals are sponsor-only, with a reason and OTP.
- Reconciliation against the gateway report runs daily. Any mismatch alerts the sponsor.
- **More than one sponsor:** all money goes into one pool. For reporting, top-ups are matched to payouts first-in, first-out, which gives each sponsor an impact report. Earmarking money for a specific category can be added later.
- **Low-pool alert:** below ₹25,000 [default] or below 2 weeks of average payouts.

**Changing bank details**
```
 Edit → OTP → penny-drop + name match → 48 h cooling-off
      → alert to OLD mobile + email: "Not you? Report"
```

---

## 7. Bills and closure (advance claims)

```
 Day 0   PAID ─► "Upload bill within 14 days"
 Day 7   ─► reminder
 Day 12  ─► "2 days left — new claims will be blocked"
 Day 14  ─► BILL OVERDUE → new claims blocked (not Emergency)
             approver/sponsor may give ONE extension (reason logged)
 Day 30  ─► approver messages student in-app; outcome noted in timeline
 Day 45  ─► sponsor alerted: "UNACCOUNTED ₹X" (sponsor may phone the student)

 At any time, the student uploads the bill
        │
        ▼
 BILL SUBMITTED ─► block lifted immediately
        │
        ▼
 Approver verifies against amount paid
   ├─ MATCH ───────► BILL VERIFIED ─► SETTLED ✔
   ├─ SPENT LESS ──► unused amount: student RETURNS it (payment link)
   │                 or it's DEDUCTED from next allowance (sponsor decides) ─► SETTLED
   └─ MISMATCH ────► back to student; 14 days to fix, then blocked

 Never accounted for (after day 45):
   return │ deduct from allowance │ sponsor WRITE-OFF (reason + OTP)
   Write-off flags the student: all future claims need manual approval.
   30 days before graduation: reminder to the student and approver
   about open bills.
```

---

## 8. Claim status map

```
 DRAFT ─► SUBMITTED ─┬─► FLAGGED ─────────────┐
                     ├─► AUTO-APPROVED ───────┼──────────────────────────┐
                     └─► AWAITING APPROVER ◄──┘                          │
                           │  ▲      │                                   │
                  INFO REQUESTED     ├─► REJECTED ─► APPEAL ─► (decision)│
                                     ├─► PARTLY APPROVED ─► accepted ────┤
                                     ├─► APPROVED (≤ ₹10k) ──────────────┤
                                     └─► AWAITING 2ND APPROVAL ─► APPROVED┤
                                                                         ▼
                                       WAITING FOR FUNDS ◄─► PAYOUT PENDING ─► EXPIRED ✖
                                                                         ▼
                                                                       PAID
                        reimbursement / institution ─────────────────────┤
                        advance ─► BILL OVERDUE (blocks) ─┐               │
                                                          ▼               ▼
                                                  BILL SUBMITTED ◄── (bill uploaded)
                                                     │        ▲
                                                     ▼        │ MISMATCH
                                               BILL VERIFIED ─┘
                                                     ▼
                                     SETTLED ✔ │ RETURNED ✔ │ ADJUSTED ✔ │ WRITTEN OFF ✖
 Student can CANCEL any time before PAID. The server rejects any move not shown here.
```

---

## 9. Traceability

```
 TOP-2026-0042  sponsor top-up ₹50,000 ───────────────── ledger +50,000
        ▼
 SSDF-2026-000123  claim (category, rule version v7)
   10:02  submitted by student
   10:02  fraud checks clear; routed: amount > Typical (rule v7)
   14:30  approver A approved ₹4,000 (limit ₹15,000) [OTP] ─ ledger reserve 4,000
        ▼
 PAY-2026-0981  gateway ref pout_xxx → student bank ✔ ─── ledger −4,000
        ▼
 Bill v1 (SHA-256) → mismatch → bill v2 → verified by approver B
        ▼
 SETTLED
```

| Record | Holds | Who sees it |
|---|---|---|
| Audit log | Who, role, action, before → after, reason, time, device, request id. Write-once and hash-chained, so any edit is detectable. Also logs every time an approver *views* a student or document. | Sponsor, owner |
| Claim timeline | Every status change, assignment, decision, reminder, upload and message | Student (own claims; actor shown as "Approver"), approver, sponsor |
| Money ledger | Double-entry: top-up, reserve, payout, fee, return, adjustment, write-off, refund. Each entry links to a claim and a gateway reference. | Sponsor, owner |
| Document versions | Never overwritten; each version has a hash, uploader and time | Approver, sponsor |
| Rule versions | Categories, caps, thresholds and grace days. Each claim stores the version that decided it. | Sponsor, owner |

**Reference formats**
- Claim: `SSDF-YYYY-nnnnnn`
- Payout: `PAY-…`
- Top-up: `TOP-…`
- Return: `RET-…`
- Write-off: `WO-…`

All references can be searched.

---

## 10. Notifications

All notifications go by in-app message and email. The ones marked **SMS** go by text message too.

| To | Events |
|---|---|
| Student | OTP (SMS) · enrolment approved/rejected · claim submitted · info requested (SMS) · approved/partly/rejected (SMS) · flagged for review · paid (SMS) · bill due day 0/7/12 (SMS) · blocked (SMS) · bill verified/mismatch · appeal result · recurring claim created · bank changed (SMS, to old number too) · write-off |
| Approver | New claim assigned · Emergency claim (SMS) · SLA near · second approval needed · bill to verify · daily summary |
| Sponsor | Escalated or unassigned claim · waiting for funds · pool low · payout failed or stuck > 24 h · reconciliation mismatch · unaccounted money (day 45) · weekly approver report · monthly impact report |
| Admin | New sign-ups to confirm · roll-list mismatches |

- Templates use plain language. Regional languages are still to be decided.
- All SMS templates must be registered on TRAI DLT before launch.

---

## 11. Privacy and security

```
 PERSONAL DATA STORE (encrypted)          AUDIT LOG / LEDGER
 name, mobile, email, bank, ID image ◄─id─ only person_id + actions
```

- **Deletion requests (DPDP Act):** handled only after all of the student's claims are closed and the retention period below has passed. Personal data is then erased. The audit log and ledger keep only the id, so the hash chain and money trail stay intact.
- **What approvers can see:**
  - Bank account: last 4 digits only
  - Mobile and email: hidden; approvers contact students through in-app messages
  - Documents: a viewer with a watermark (approver name + time); downloads off
- **Security controls:**
  - Every endpoint checks the user's role.
  - Data is encrypted in transit and at rest.
  - Rate limits on login, OTP and the survey.
  - File uploads are checked for type and size and scanned for viruses.
- **Grievance officer:** named in the privacy notice, with a 30-day response target. A Help button on every claim opens a ticket linked to that claim.

**Retention [default]**

| Data | Kept for |
|---|---|
| ID card image | Enrolment confirmed + 90 days |
| Bank details | Graduation + 1 year, or until open claims are closed, whichever is later |
| Claim documents and bills | 8 years |
| Ledger and audit log | 8 years, then personal links anonymised |
| Survey | Raw answers 2 years, then aggregates only |

---

## 12. Feedback survey (public)
- No login, name, student ID or IP address is stored, and there's no analytics.
- Shows **all active categories** by group with Min, Typical and Max. For each: Realistic? (Yes / No / Partially), suggested amount and a comment.
- Spam protection: a captcha plus a signed one-time form token that expires in 30 minutes.
- Stored separately from the app database.
- Only aggregated results are exported, for the owner and sponsors.

---

## 13. User flows (screen by screen, per role)

Sections 2–8 describe the system logic. This section describes what each person **sees and taps**, screen by screen.

### 13.1 Student

```
 ┌───────────┐   ┌───────────┐   ┌────────────┐   ┌──────────────┐   ┌──────────────┐
 │ Landing   │──►│ Sign up   │──►│ OTP verify │──►│ Upload ID    │──►│ "Waiting for │
 │ Login /   │   │ form      │   │ mobile +   │   │ card         │   │ confirmation"│
 │ Sign up   │   │ (passing  │   │ email      │   │              │   │ (status page)│
 └───────────┘   │  year)    │   └────────────┘   └──────────────┘   └──────┬───────┘
                 └───────────┘                                SMS: approved │
                                                                            ▼
 ┌─────────────────────────────── HOME ───────────────────────────────────────────┐
 │ Hi Priya · Year 3 · CSE                                                        │
 │ ⚠ Banner if blocked: "Upload bill for SSDF-2026-000123"  [Upload now]           │
 │ Yearly cap: ₹18,000 used of ₹50,000  ▓▓▓▓░░░░░░                                 │
 │ My categories: Books ₹1,200 left · Hostel ₹6,000 left · …                       │
 │ [ Claim fund ]   My claims (3)   Bills due (1)   Notifications   Profile   Help │
 └────────────────────────────────────────────────────────────────────────────────┘
        │ Claim fund
        ▼
 Step 1 Category ─► Step 2 Type ─► Step 3 Amount ─► Step 4 Documents ─► Step 5 Items ─►
 (eligible only)    (advance /      & purpose       (camera or file)    (optional)
                     reimbursement /
                     institution)
 ─► Step 6 Declaration ─► Step 7 Review ─► [Submit] ─► "Submitted: SSDF-2026-000123"
                                         (or Save draft)

 MY CLAIMS ─► Claim detail
   ┌─────────────────────────────────────────────────────┐
   │ SSDF-2026-000123 · Books · ₹4,000 · PAID             │
   │ Timeline: Submitted ✔ Approved ✔ Paid ✔ Bill ◯       │
   │ Bill due in 6 days  [Upload bill]                    │
   │ Actions by status:                                   │
   │   Info requested  → [Edit & resubmit]                │
   │   Partly approved → [Accept ₹3,000] [Cancel]         │
   │   Rejected        → [Appeal] (7 days)                │
   │   Before paid     → [Cancel claim]                   │
   │   Monthly category→ [Repeat monthly ☐]               │
   │   Any             → [Help] (opens ticket)            │
   └─────────────────────────────────────────────────────┘

 PROFILE ─► Bank details ─► OTP ─► penny-drop check ─► "Active after 48 h" (if changed)
         ─► Personal details (read only; ask admin to change)
         ─► Privacy: download my data · request deletion
```

### 13.2 Approver

```
 Invite email ─► Set password ─► OTP ─► Confidentiality agreement [I agree] ─► DASHBOARD
 ┌──────────────────────────── DASHBOARD ─────────────────────────────┐
 │ 🔴 Emergency (1)  ·  My queue (6)  ·  2nd approvals (2)             │
 │ Bills to verify (4)  ·  Appeals (1)  ·  Today approved ₹12,000/50k  │
 └────────────────────────────────────────────────────────────────────┘
        │ open claim
        ▼
 ┌──────────────────────── CLAIM REVIEW ──────────────────────────────┐
 │ Student: Priya · 21CS045 · CSE · Year 3 · bank ••••4821            │
 │ ⚠ Flags (if any): "Same bill used in SSDF-2026-000087"             │
 │ Category, type, amount, purpose │ Documents (watermarked viewer)   │
 │ Allowance left · yearly cap left · past claims · open bills        │
 │ [Approve] [Approve part ₹__] [Reject + reason] [Ask info]          │
 │ [Conflict of interest]   [Message student]                         │
 └───────────────────────────────┬────────────────────────────────────┘
                                 ▼
                     OTP confirm ─► back to queue (next claim)

 BILLS TO VERIFY ─► bill viewer beside the amount paid
                 ─► [Match] [Spent less ₹__] [Mismatch + reason] [Give extension]
```

### 13.3 Sponsor

```
 Invite ─► password ─► OTP ─► DASHBOARD
 ┌──────────────────────────── DASHBOARD ─────────────────────────────┐
 │ Pool ₹1,24,000 (reserved ₹18,000) · [Add funds]                    │
 │ Waiting for funds (2) · Payout issues (1) · Unaccounted ₹6,500 (2) │
 │ Escalated/unassigned (1) · Reconciliation: ✔ today                 │
 └────────────────────────────────────────────────────────────────────┘
 Menu:
 ├─ Add funds ─► amount ─► gateway checkout ─► receipt
 ├─ Payouts ─► list ─► retry / view gateway ref
 ├─ Unaccounted ─► [Send reminder] [Deduct] [Request return] [Write off + OTP]
 ├─ Refunds ─► claim ─► reason ─► OTP
 ├─ Categories ─► add / edit / retire ─► new version saved
 ├─ Rules & settings ─► thresholds, caps, grace days, sample % ─► OTP
 ├─ Approvers ─► invite / change limit / pause / deactivate
 ├─ Institution payees ─► add ─► bank verify
 ├─ Reports ─► by category / year / department / approver · impact · CSV
 └─ Audit log ─► search by ref, person, date ─► verify hash chain
```

### 13.4 Admin

```
 Invite ─► password ─► OTP ─► DASHBOARD
 ├─ Pending sign-ups ─► compare form vs roll list vs ID image ─► [Approve] [Reject + reason]
 ├─ Students ─► search ─► edit name/ID/department/passing year/status ─► reason ─► OTP
 ├─ Import roll list (CSV) ─► preview ─► matched / mismatched report ─► confirm
 └─ My actions log
 (No menu items for claims, money or reports.)
```

### 13.5 Owner

```
 Login + OTP ─► Sponsors (create/deactivate) · Admins (create/deactivate)
             ─► Global settings (academic year start, course lengths)
             ─► Full audit log · all reports
```

### 13.6 Public survey

```
 Open link ─► intro ("anonymous, takes 5 min") ─► captcha
          ─► one card per category group: Realistic? · Suggested ₹ · Comment
          ─► [Submit] ─► "Thank you"   (no login, nothing identifying stored)
```

---

## 14. Data model (main tables)

| Table | Key fields |
|---|---|
| persons | id, name, mobile, email (encrypted; erasable) |
| users | id, person_id, role, status, password_hash, otp settings |
| students | user_id, student_id, department, passing_year, course_length, status, flagged |
| bank_accounts | student_id, account (encrypted), ifsc, verified_at, cooling_off_until |
| approvers | user_id, limit, daily_cap, active, agreement_version |
| sponsors | user_id |
| institution_payees | name, bank (encrypted), verified_at |
| categories / category_versions | group, support_type, min, typical, max, eligible_years, payee_type, claim_types |
| registrations | student, category, academic_year |
| claims | ref, student, category_version, type, amount_requested, amount_approved, status, assigned_approver, second_approver, bill_due_at, rule_version, flags |
| claim_items | claim, description, amount |
| documents / document_versions | claim, kind (proof/bill), file_key, sha256, bill_no, vendor, uploaded_by |
| claim_events | claim, from_status, to_status, actor, reason, time |
| allowances | student, category, academic_year, reserved, used |
| payouts | ref, claim, payee, idempotency_key, gateway_ref, status, attempts |
| topups | ref, sponsor, amount, gateway_ref, receipt |
| ledger_entries | ref, debit_account, credit_account, amount, claim, payout/topup, time |
| appeals, extensions, write_offs | claim, actor, reason, outcome |
| recurring_claims | student, category, amount, until |
| tickets | claim, student, messages |
| audit_events | actor, role, action, entity, before, after, reason, ip, device, request_id, prev_hash, hash |
| config_versions | key, value, version, changed_by |
| notifications, consents | user, template/version, time |
| survey_responses (separate DB) | category, realistic, suggested, comment, time |

---

## 15. Non-functional requirements
- Mobile-first: works on low-end Android phones on 3G.
- Large tap targets and clear labels. English first.
- **Size [default]:** up to 5,000 students, about 500 claims a month, 10–20 approvers. One app server and one PostgreSQL server are enough.
- **Environments:** CI tests → staging (gateway sandbox, test SMS) → production.
- **Monitoring and alerts:** stuck or failed payouts, webhook signature failures, reconciliation mismatches, low pool, error rate and downtime.
- **Backups:** daily and encrypted, with a monthly restore test that is logged.
- **Rough monthly running cost:**
  - Hosting: about ₹1,500–4,000
  - SMS: about ₹0.15–0.25 per message
  - Payout fees: about ₹3–6 each (check current gateway pricing)

---

## 16. Prerequisites (outside the app, start now)
1. A registered legal entity (trust, society or Section 8 company) with a PAN and a current account
2. Gateway business KYC and payout product activation (e.g. RazorpayX)
3. 12A/80G registration, if donors want tax receipts (check with a CA)
4. TRAI DLT registration for SMS (takes 2–4 weeks)
5. College agreement to share the student roll list
6. Privacy notice and terms, with legal help

---

## 17. Build phases (estimate assumes 2 developers)

| Phase | Scope | Est. |
|---|---|---|
| 0 | Prerequisites (section 16), run alongside the build | ongoing |
| 1 | Login and OTP, all roles including owner, passing-year logic, admin records, roll-list import, personal-data store, audit log with hash chain | 3 wks |
| 2 | Categories (payee type, claim types, caps), claim form, real-time checks, fraud checks, rules engine, status map, timeline | 4 wks |
| 3 | Approver portal: assignment, limits, daily cap, second approval, SLA, appeals, conflicts, approver management, confidentiality agreement, watermarked viewer | 3 wks |
| 4 | Payments: top-ups, student and institution payouts, ledger, reservations, waiting-for-funds queue, bank cooling-off, webhooks, reconciliation, receipts | 4 wks |
| 5 | Bills: upload, verification, reminders, block, extensions, recovery, write-off, recurring claims, settlement | 3 wks |
| 6 | Survey | 1 wk |
| 7 | Reports (impact, approver activity, trace views), help tickets, notifications polish, monitoring, backups, retention jobs, security testing, testing with students | 3 wks |
| | **Total** | **≈ 21 weeks** |

---

## 18. Open decisions
1. Tech stack. Recommended: Next.js + PostgreSQL + object storage + Razorpay.
2. Can a student make more than one claim per category per semester?
3. Regional languages: which ones, and from which phase?
4. 80G receipts for top-ups (CA to confirm)
5. Which categories are paid to institutions, and which institutions are registered payees?
6. Confirm all **[default]** values:
   - Two-approver threshold ₹10,000
   - Approver daily cap ₹50,000
   - Student yearly cap ₹50,000
   - Emergency cap ₹25,000
   - Bill grace period 14 days
   - Low-pool alert ₹25,000
   - Retention periods
   - Size assumption
7. Who are the owner(s), and who is the grievance officer?

---

## 19. Verification

| Area | Test |
|---|---|
| Passing year | Year correct across the June boundary; 4- and 5-year courses; out-of-range year rejected; graduation sets status |
| Real-time block | Clock to day 13 / 14 / 15 → allowed / blocked / blocked; upload → unblocked instantly; Emergency always allowed; mismatch blocks after 14 days |
| Claim types | Reimbursement and institution skip the bill clock; institution payout reaches the payee account |
| Routing | ≤ Typical and ≤ ₹10k → auto; Typical > ₹10k still goes to an approver; flagged never auto; Emergency 8 h SLA |
| Approval controls | Over-limit approval refused by the server; > ₹10k needs a different second approver; daily cap respected; appeal goes to a different approver |
| Fraud | Same file or bill number on a second claim → flagged; payout held during bank cooling-off; old contact details alerted |
| Allowance and caps | Reserve, use and release; yearly cap blocks the form; submitted-year rule |
| Money | Sandbox: success, failure, timeout, duplicate webhook, refund, empty pool → waiting-for-funds → priority order; ledger total = gateway report |
| Recovery | Day 30 / 45 tasks; return, deduct and write-off ledger entries; flagged after write-off |
| Traceability | Every write and every approver view creates an audit row; editing a row breaks the hash chain; any claim's full timeline can be rebuilt |
| Roles | One test per row of the permission matrix (section 1) |
| Privacy | Erasure after closure keeps the hash chain valid; approvers can't download documents or see contact details |
| Survey | No identity stored; token and captcha enforced; aggregates correct |
| Operations | Backup restored to staging; alerts fire; works on a low-end Android phone on throttled 3G |

