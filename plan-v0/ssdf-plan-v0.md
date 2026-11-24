# SSDF fund app: design plan v0 (manual payments)

## Context
SSDF is an alumni-funded welfare fund for students of a tier-2 city college. Students claim money for academic, travel, living, digital, health, career and emergency needs. Small claims are approved automatically. Volunteer **approvers** decide the rest. **Sponsors** fund and control the money. **Admin** manages student records only. Every action and rupee can be traced.

This is the complete plan for the **first release (v0)**. It combines:
- the design plan v3 (`docs/ssdf-plan-v3.md`), and
- the final decisions from the v3 review (`docs/ssdf-decisions-v3.md`, branch `decisions-v3`).

**The one big difference from v3: there is no payment gateway and no automatic payout system in v0.**
Money comes in and goes out through the trust's own bank account. Sponsors pay approved claims by hand (net banking or UPI) and record the bank reference (UTR) in the app. A second person or the bank statement confirms every payment. The automatic payout system (Razorpay / RazorpayX) moves to **v1** (section 21).

| In v0 | Left for v1 |
|---|---|
| Donations by bank transfer, recorded and confirmed in the app | Online donation checkout (payment gateway) |
| Sponsors pay by hand and enter the UTR | Automatic payouts (RazorpayX) |
| Bank statement upload and auto-matching | Gateway webhooks and settlement tracking |
| Bank details checked by hand (passbook / cheque photo) | Penny-drop API check |
| Pause switch, daily limits, same-account checks | Gateway-side approval rules, IP whitelist |

It is a design document; no code exists yet.

**Decisions made**
- Approvers decide every claim that isn't auto-approved. The sponsor handles money and settings only.
- The sponsor creates approvers and sets their limits. The **Owner** role creates sponsors.
- Sign-up uses **passing year**. Year of study is calculated, not stored.
- **Bill rule:** advance claims need a bill within 14 days (configurable). After that, new claims are blocked until the bill is uploaded. The block is checked in real time.
- **Traceability:** a write-once audit log, a timeline for every claim, a double-entry money ledger, versioned documents and versioned rules.
- **Two-person rule** for every sensitive action (section 1).
- **Payments are manual in v0** (section 6).

Status key used in this plan:
- **[default]**: a proposed value the team must confirm. All are editable in settings.
- 🟡 **Needs confirmation**: decided, but a person or authority outside the team must confirm it.

---

## 1. Roles and permissions

```
                              ┌─────────┐
                              │  OWNER  │  founder(s); creates sponsors and admins,
                              └────┬────┘  global settings; at least 2 owners
                                   │
                              ┌────▼────┐
                              │ SPONSOR │  money + rules; creates approvers;
                              └────┬────┘  pays claims; at least 2 sponsors
              ┌────────────────────┼─────────────────────┐
         ┌────▼─────┐         ┌────▼────┐           ┌────▼────┐
         │ APPROVER │         │ STUDENT │           │  ADMIN  │
         │volunteer │         └─────────┘           └─────────┘
         └──────────┘                              (created by owner)
```

| Action | Student | Approver | Sponsor | Admin | Owner |
|---|---|---|---|---|---|
| Sign up, edit own profile and bank details | ✔ | | | | |
| Submit claim, upload bill, appeal, view own timeline, raise ticket | ✔ | | | | |
| Approve / reject / request info (within own limit) | | ✔ | fallback (+ 2nd person) | | |
| Verify bills | | ✔ | ✔ | | |
| See student name, ID, program, year | own | ✔ | ✔ | ✔ | |
| See student mobile, email, full bank number | own | ✖ (last 4 digits) | ✔ | ✖ | |
| See ID card image | own | | | during enrolment | |
| Edit student records (ID, program, passing year, status) | | | | ✔ | |
| Edit student name / contact after enrolment | | | 2nd approval | with 2nd approval | |
| Confirm enrolment, import roll list | | | | ✔ | |
| Categories, amounts, caps, auto-approve rule, grace days | | | ✔ (2 persons) | | |
| Record donations, pay claims, confirm payments, import bank statement | | | ✔ | | |
| Refunds, write-offs, reconciliation | | | ✔ (2 persons) | | |
| Pause / resume all payments | | | pause: 1 · resume: 2 | | |
| Create / limit / deactivate approvers | | | ✔ (2 persons) | | |
| Help tickets and grievances | | | ✔ | | |
| Create sponsors and admins, global settings | | | | | ✔ (2 owners) |
| Full audit log, reports | | own activity | ✔ | own actions | ✔ |

**Two-person rule.** These actions need two *distinct persons* (checked on `person_id`, not accounts):
- creating sponsors (2 owners)
- creating approvers or raising approver limits
- changing rules or caps
- write-offs
- refunds above ₹5,000 [default]
- resuming payments after the pause switch or a reconciliation mismatch
- confirming a payment or donation (the confirmer must not be the person who recorded it)

**Other rules**
- Staff (approvers, sponsors, admins, owners) log in by invite only, with password + **authenticator app (TOTP)**. A fresh check (step-up) is needed before approving claims, recording money or changing records; it stays valid for 10 minutes.
- A person with a student account can **never** be an approver, sponsor or admin. The check is on `person_id`.
- Before their first login, approvers accept a confidentiality agreement. The agreement is versioned and the acceptance is logged.
- **Approvers:** alumni or faculty volunteers chosen by the sponsors, 1-year renewable term, short training with written guidelines and sample cases before their first claim. 🟡 sponsors
- **Fund Rules policy:** a written policy (eligibility, categories, caps, approver code of conduct) approved by the sponsors. The app enforces it and does not invent rules. 🟡 sponsors
- **Grievance officer:** one named sponsor or owner.
- **Account recovery** (everyone): forgot password = email + mobile OTP. Lost or changed phone = admin identity check + 48 h cooling-off + alert to the old contacts.
- **Staff offboarding:** deactivating an approver moves their open queue back to round-robin. Deactivating an admin returns pending enrolments to the shared queue.
- **Minimum team:** at least **2 owners and 2 sponsors**, or the two-person actions can't happen.

---

## 2. Student sign-up and the passing year

```
 Fill form: name, student ID, PROGRAM, PASSING YEAR,
 mobile (main), college email (optional), income band, password
        │
        ▼
 Student ID already used? ─ Yes ─► "Account exists — log in or contact admin"
        │ No
        ▼
 Passing year in valid range? ─ No ─► "For a 4-year course, enter 2027–2030"
        │ Yes                         (range = this academic year's end
        ▼                               … + course length − 1, per program)
 Under 18? ─ Yes ─► parent / guardian consent captured 🟡 legal
        │
        ▼
 OTP on mobile (+ email if given) (rate-limited)
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
                         ▼              (at most 3 resubmissions, then contact admin)
                 Status: ACTIVE         Admin target: confirm within 3 working days
```

**Program and year of study**
- Course length is set **per program**, not per department (BE 4 yrs, ME 2, MBA 2, …). Each student links to a program.
- Year of study is calculated, not stored: `year = course length − (passing year − academic year end)`.
- The academic year starts in June [default].

| Example (Nov 2026, academic year ends 2027, 4-year course) | Year |
|---|---|
| Passing 2027 | 4 |
| Passing 2028 | 3 |
| Passing 2029 | 2 |
| Passing 2030 | 1 |

- **Every June:** allowances reset, and eligible categories update automatically.
- **Before each June rollover:** admin gets a bulk review list of students due to graduate, so students with backlogs aren't wrongly marked GRADUATED.
- **Year-back, detention or lateral entry:** admin edits the passing year and gives a reason. The change is audited.
- **After the passing year's academic year ends:** status becomes **GRADUATED**. The student can't make new claims but must still close open bills.

**Income band (need-based priority)** 🟡 sponsors
- At sign-up, students declare a family income band (ranges only, no documents).
- The lowest band gets first priority in the waiting-for-funds queue, after Emergency (section 6). Caps are the same for everyone in v0.

**Bank account (checked by hand in v0)**
- Required only before the first claim paid to the student. Not needed for claims paid to an institution.
- The student enters account holder name, account number, IFSC, and optionally a UPI ID, and uploads a **passbook first page or cancelled cheque** photo.
- A sponsor checks the photo against the entered details. When the sponsor adds the account as a beneficiary in net banking, the bank shows the account holder's name; the sponsor ticks **"name matches"**. A mismatch goes to manual review by a sponsor, not an automatic failure.
- A UPI ID change is treated exactly like a bank change (section 6).

**Student statuses**

| Status | Meaning |
|---|---|
| PENDING | Waiting for enrolment confirmation |
| ACTIVE | Can claim |
| SUSPENDED / HIDDEN | Set by admin. Can't log in or claim. Payments already recorded still finish, and bills are still due. |
| GRADUATED | Set after admin review. No new claims; open bills still due. |
| DROPPED OUT | Set by admin. No new claims; open bills still due. |

---

## 3. Categories and claim types

Each category has:
- Name, group and what it covers
- Support type (period): monthly, per semester, per year, one-time or per incident
- Min, Typical and Max amounts (₹), set from the cost survey (section 12)
- Eligible years of study
- **Payee type:** student or institution
- **Allowed claim types**
- **Health flag:** documents visible only to health-cleared approvers

```
                       Student picks category
                                │
                 ┌──────────────┴──────────────┐
            INSTITUTION                     STUDENT
     (tuition, exam, hostel, mess —   (travel, books, laptop,
      registered payees only)          health, emergency…)
                 │                             │
                 ▼                   Already paid and has a bill?
      Paid to college/hostel directly   Yes │            │ No
      Proof: official fee slip              ▼            ▼
      Institution receipt = bill     REIMBURSEMENT    ADVANCE
      → SETTLED, no bill clock       bill attached     paid first,
                                     at claim time     bill due in 14 days
                                     → SETTLED after   (section 7)
                                       verification
```

- **Institution payees:** the sponsor keeps a list of college and hostel accounts, each verified once (two persons).
- **Recurring monthly claims** (hostel, mess, bus pass, internet):
  1. After the first month is approved, the student turns on "repeat monthly" (up to semester end).
  2. Each month the student **confirms**: a receipt for reimbursements, or a one-tap confirmation for institution payees.
  3. The same amount is auto-approved. A different amount goes through normal approval.
  4. Recurring claims stop on any status change or when the category is retired.
- **Emergency** is defined in the Fund Rules (medical, accident, death in family, disaster). Proof can be submitted within 7 days. 🟡 sponsors

**Caps and allowance**
- Each category's **Max applies per its support period** (month, semester, year or incident).
- A student may make **more than one claim per category per period**, up to the period Max.
- Per student per year: ₹50,000 [default] across all categories. **This is a hard block on the form.**
- Emergency is outside the yearly cap and has its own cap of ₹25,000 per year [default].
- Allowance moves through three stages:

| When | Allowance |
|---|---|
| Claim approved | RESERVED |
| Payment recorded | USED |
| Rejected, cancelled, expired or money returned | RELEASED |

- A claim counts in the academic year it was **submitted** in. The academic year is stored on the claim at submit.

---

## 4. Making a claim (real-time checks)

```
 Student taps "Claim fund"
        │
        ▼
 ┌────────────────────────────────────────────────────────────┐
 │ SERVER CHECK — runs live on open AND again inside the      │
 │ submit transaction                                         │
 │  ✖ Status not ACTIVE                                       │
 │  ✖ Any advance PAID with bill overdue (> 14 days)          │
 │  ✖ Any bill MISMATCH not yet fixed (blocks immediately)    │
 │  ✖ An unbilled advance already open (for a new advance)    │
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
                                + bill number, vendor, bill date
 Emergency is never          5. Optional item list (auto total)
 blocked by bills            6. Declaration + consent (time + version saved)
                             7. Save draft / review / submit
                                       │
                                       ▼
                            Ref no. SSDF-2026-000123 → section 5
```

- **At most 1 unbilled advance per student** [default]. A second advance is blocked until the first bill is **verified**.
- The bill number, vendor and bill date are needed for the duplicate check (section 5).
- A student submitting from two tabs or phones at once: the submit takes a lock per student, and the cap is checked inside the same transaction.

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
                  │   write-off / failed      │
                  │   post-check)?            │
                  └─────┬───────────────┬─────┘
                   hit  │               │ clear
                        ▼               ▼
                 FLAGGED (red     ┌─────────────────────────────────┐
                 banner, reason)  │ AUTO-APPROVE only if ALL true:  │
                        │         │ • reimbursement or institution  │
                        │         │   (never advance, never         │
                        │         │   Emergency)                    │
                        │         │ • amount ≤ Typical              │
                        │         │ • amount ≤ ₹10,000              │
                        │         │ • documents present             │
                        │         │ • < 2 auto-approvals this month │
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
                 │ no conflict, health-cleared if health claim.  │
                 │ None available → SPONSOR.                     │
                 │ Emergency: all approvers notified,            │
                 │ SLA 8 working hours.                          │
                 │ Others: SLA 3 days, then reassigned + sponsor │
                 │ alerted.                                      │
                 └───────────────────────┬───────────────────────┘
                                         ▼
                                Approver reviews
        ┌─────────────┬─────────────┬────┴──────────┬────────────────┐
        ▼             ▼             ▼               ▼                ▼
     APPROVE      PARTLY        REJECT        REQUEST INFO       CONFLICT
        │         APPROVE       (reason from  → student edits    → reassigned
        │            │           fixed list   → same approver
        │            │           + text)      (expires in 14 days)
        │            ▼             ▼
        │     Student accepts  Student may APPEAL once in 7 days
        │     or cancels in    → DIFFERENT approver (or sponsor)
        │     7 days (no reply → final decision
        │     = CANCELLED)
        ▼
 APPROVED amount > ₹10,000? ── yes ─► SECOND APPROVAL by a different approver
        │ no                        │
        ▼                           ▼
                  APPROVED → section 6
```

- The ₹10,000 second-approval threshold is checked on the **approved amount**.
- **Emergency is never auto-approved.** The approver sees the student's overdue bills.
- **Sponsor fallback:** when a sponsor approves because no approver is available, a second sponsor or approver must also approve.
- **5% post-check failure:** the claim gets a FLAGGED note, the student is flagged for manual approval, and the sponsor decides on recovery.
- **Health claims:** documents are visible only to approvers marked "health-cleared" (1–2 people).
- **Inactive approver:** no activity for 14 days with an empty queue → paused automatically, and the sponsor is told.
- A sponsor report shows each approver's approvals, average amount, approval rate and overrides. It is sent weekly.
- **Bill verification** is done by approvers, using the same assignment rules.

---

## 6. Money in v0: manual payments through the trust bank account

In v0 the app **does not move money**. People move money in the bank; the app tells them what to pay, records what they did, and checks it against the bank statement.

```
  MONEY IN                                   MONEY OUT
  ─────────                                  ──────────
  Donor pays the TRUST BANK ACCOUNT          Claim APPROVED
  (NEFT / RTGS / UPI / cheque)                    │
       │                                          ▼
       ▼                                     Pool has enough? ─ No ─► WAITING FOR FUNDS
  Sponsor A: "Record donation"                    │ yes              priority when funds arrive:
  amount, mode, bank ref, date, donor             │                  1 Emergency
  → TOP-xxxx  UNCONFIRMED                         │                  2 Lowest income band 🟡
       │                                          │                  3 Institution fee with due date
       ▼                                          │                  4 Oldest approved
  Confirmed by Sponsor B                          ▼                  (expires after 60 days)
  OR matched on the bank statement         Ledger: RESERVE amount
       │                                          │
       ▼                                          ▼
  Ledger +TOP-xxxx (now in the pool)       Bank changed < 48 h? ─► HOLD until cooling-off ends
  Receipt PDF (80G if registered)                 │
                                                  ▼
                                           READY TO PAY list (sponsor)
                                           shows payee name, bank, IFSC, amount, claim
                                                  │
                                                  ▼
                                           Sponsor pays in net banking / UPI
                                           (outside the app)
                                                  │
                                                  ▼
                                           "Record payment": UTR, mode, date,
                                           amount, from-account, screenshot (optional)
                                           + step-up check
                                                  │
                                                  ▼
                                           PAID (unconfirmed) ─ ledger −PAY
                                           student: "₹4,000 sent, UTR 4321…"
                                           advance → bill clock starts
                                                  │
                                    ┌─────────────┴──────────────┐
                                    ▼                            ▼
                          Confirmed by a different      Bank returns the money
                          sponsor OR matched on the     (closed account, wrong IFSC)
                          bank statement                         │
                                    │                            ▼
                                    ▼                     RETURNED: ledger reversal,
                                 PAID ✔                   bill clock cancelled, student
                                                          fixes bank → READY TO PAY
```

**Paying**
- The **Ready to pay** list is sorted by the same priority as waiting for funds. Emergency claims are pinned to the top.
- **Payee details shown to the payer** come from a **snapshot** of the student's verified bank account, stored with the payment. Bank account history is kept and never overwritten.
- **Institution payees** are paid the same way, to the registered institution account.
- **Optional bulk pay:** the sponsor selects several claims → the app exports a **payment batch file** in the bank's bulk-upload format → the sponsor uploads it in corporate net banking → the batch is approved there → the UTRs come back in the bank statement.
- **Use the bank's own maker–checker:** set up corporate net banking so that one person creates a payment and a second person authorises it. This gives a second pair of eyes inside the bank too.
- **Net-banking passwords and tokens are never stored in the app.**

**Confirming**
- Every recorded payment and donation must be confirmed by a **different person** from the one who recorded it, **or** by a matching bank statement line.
- **Bank statement upload** (weekly [default], CSV from net banking):
  - The app auto-matches lines by **UTR + amount** (and date ± 2 days).
  - Matched → the payment or donation becomes CONFIRMED.
  - **Unmatched debit** (money left the account but nothing in the app) → alert all sponsors.
  - **Unmatched credit** (money arrived but no donation recorded) → shown as "record this donation".
  - A payment left **unconfirmed for more than 3 days** → alert.
- **Student "Not received" button:** shown 3 days after PAID. It opens a ticket to the sponsors with the claim and UTR attached.

**Money rules**
- The pool balance is always the sum of the **confirmed** ledger entries. It is never typed in.
- All amounts are stored as **whole paise (integers)**, never decimals.
- **One payment per claim**, enforced by the database (`UNIQUE(claim_id)`). The same **UTR can't be used twice**.
- Recording a payment changes the status from READY TO PAY → PAID in one locked step, so a double-click or two sponsors at once can't record it twice.
- Bank charges are recorded in the ledger as fund expenses.
- Refunds and reversals are sponsor-only, with a reason, a step-up check and (above ₹5,000 [default]) a second person.
- **Reconciliation:** the ledger must equal the bank statement balance. A mismatch alerts the sponsors and **pauses payments** until two sponsors sign off.
- **More than one sponsor:** all money goes into one pool. For reporting, donations are matched to payments first-in, first-out, which gives each sponsor an impact report. Earmarking for a specific category can be added later.
- **Low-pool alert:** below ₹25,000 [default] or below 2 weeks of average payments.
- **Student returns** (RET-…): the student pays the trust account by UPI/NEFT and enters the UTR; confirmed the same way as donations.
- **Large and small donations** both go directly to the trust bank account in v0, so there is no ~2% gateway fee.

**Ledger accounts**

| Account | Used for |
|---|---|
| Trust bank account | Real money in the bank |
| Sponsor funds | Donations received |
| Donations unconfirmed | Recorded but not yet confirmed |
| Reserved | Approved claims waiting to be paid |
| Paid to students | Student payments |
| Paid to institutions | Institution payments |
| Payments unconfirmed | Recorded but not yet confirmed |
| Student receivable | Money a student must return |
| Bank charges | Fees charged by the bank |
| Write-off expense | Money written off |

Every movement has two sides, and the total is always zero.

**Changing bank details**
```
 Edit → OTP → upload passbook / cheque photo → sponsor checks name
      → 48 h cooling-off
      → alert to OLD mobile + email: "Not you? Report"
```
- Changing a **mobile number** is treated the same way (48 h cooling-off, alert to the old contacts).
- Admin **can't** change a student's contact or bank details; only the student can, with OTP.

---

## 7. Bills and closure (advance claims)

```
 Day 0   PAID ─► "Upload bill within 14 days"
 Day 7   ─► reminder
 Day 12  ─► "2 days left — new claims will be blocked"
 Day 14  ─► BILL OVERDUE → new claims blocked (not Emergency)
             approver/sponsor may give ONE extension of 7 days at most (reason logged)
 Day 30  ─► approver messages student in-app; outcome noted in timeline
 Day 45  ─► sponsor alerted: "UNACCOUNTED ₹X" (sponsor may phone the student)

 At any time, the student uploads the bill
        │
        ▼
 BILL SUBMITTED ─► block lifted PROVISIONALLY
        │
        ▼
 Approver verifies against amount paid
   ├─ MATCH ───────► BILL VERIFIED ─► SETTLED ✔
   ├─ SPENT LESS ──► unused amount: student RETURNS it (UTR recorded)
   │                 or it's DEDUCTED from next allowance (sponsor decides) ─► SETTLED
   ├─ SPENT MORE ──► student may file a reimbursement claim for the
   │                 difference, linked to this claim
   └─ MISMATCH ────► back to student; RE-BLOCKED IMMEDIATELY until fixed

 Never accounted for (after day 45):
   return │ deduct from allowance │ sponsor WRITE-OFF (reason + 2 persons)
   Write-off flags the student: all future claims need manual approval.
   30 days before graduation: reminder to the student and approver
   about open bills.
   Graduated students: return or write-off only (no allowance left to deduct).
```

- Bill reminders respect quiet hours (9 PM–8 AM). Deadlines are calculated in **Asia/Kolkata** from stored dates.
- If reminders failed to send (SMS or email down), the bill-block clock **pauses automatically**.

---

## 8. Claim status map

```
 DRAFT ─► SUBMITTED ─┬─► FLAGGED ─────────────┐
 (expires 30 d)      ├─► AUTO-APPROVED ───────┼──────────────────────────┐
                     └─► AWAITING APPROVER ◄──┘                          │
                           │  ▲      │                                   │
                  INFO REQUESTED     ├─► REJECTED ─► APPEAL ─► (decision)│
                  (expires 14 d)     ├─► PARTLY APPROVED ─► accepted ────┤
                                     │        └─► no reply 7 d ─► CANCELLED
                                     ├─► APPROVED (≤ ₹10k) ──────────────┤
                                     └─► AWAITING 2ND APPROVAL ─► APPROVED┤
                                                                         ▼
                                    WAITING FOR FUNDS ◄─► READY TO PAY ◄─┐
                                    (expires 60 d)              │        │
                                                                ▼        │
                                                    PAID (unconfirmed)   │
                                                         │       └─► RETURNED
                                                         ▼          (fix bank)
                                                       PAID ✔
                        reimbursement / institution ─────┤
                        advance ─► BILL OVERDUE (blocks) ─┐              │
                                                          ▼              ▼
                                                  BILL SUBMITTED ◄── (bill uploaded)
                                                     │        ▲
                                                     ▼        │ MISMATCH (re-blocks)
                                               BILL VERIFIED ─┘
                                                     ▼
                                     SETTLED ✔ │ RETURNED ✔ │ ADJUSTED ✔ │ WRITTEN OFF ✖
 Student can CANCEL any time before a payment is recorded.
 The server rejects any move not shown here.
```

---

## 9. Traceability

```
 TOP-2026-0042  donation ₹50,000, NEFT ref N1234…  recorded by sponsor A,
                confirmed on bank statement ─────────────── ledger +50,000
        ▼
 SSDF-2026-000123  claim (category, rule version v7)
   10:02  submitted by student
   10:02  fraud checks clear; routed: amount > Typical (rule v7)
   14:30  approver A approved ₹4,000 (limit ₹15,000) [step-up] ─ ledger reserve 4,000
        ▼
 PAY-2026-0981  paid by sponsor A, IMPS, UTR 4321098765 ── ledger −4,000
                confirmed by sponsor B + statement line 2026-11-12
        ▼
 Bill v1 (SHA-256) → mismatch → bill v2 → verified by approver B
        ▼
 SETTLED
```

| Record | Holds | Who sees it |
|---|---|---|
| Audit log | Who, role, action, before → after, reason, time, device, request id. Write-once and hash-chained, so any edit is detectable. Also logs every time an approver *views* a student or document. | Sponsor, owner |
| Claim timeline | Every status change, assignment, decision, reminder, upload, message and payment | Student (own claims; actor shown as "Approver"), approver, sponsor |
| Money ledger | Double-entry: donation, reserve, payment, bank charge, return, adjustment, write-off, refund, reversal. Each entry links to a claim and a bank reference (UTR). | Sponsor, owner |
| Payment record | Payee bank snapshot, UTR, mode, who paid, who confirmed, matching statement line, proof screenshot | Sponsor, owner |
| Document versions | Never overwritten; each version has a hash, uploader and time | Approver, sponsor |
| Rule versions | Categories, caps, thresholds and grace days. Each claim stores the version that decided it. | Sponsor, owner |

- The audit log **never stores personal values** in before/after. It stores field names plus values encrypted with a **per-person key**. Erasure = destroy the key (crypto-shredding), so the hash chain stays valid.

**Reference formats**
- Claim: `SSDF-YYYY-nnnnnn`
- Payment: `PAY-…`
- Donation (top-up): `TOP-…`
- Return: `RET-…`
- Write-off: `WO-…`

All references and UTRs can be searched.

---

## 10. Notifications

Channel order: **in-app → email → SMS**. SMS is used only for OTPs, payments, blocks and Emergency alerts. Quiet hours for reminders: 9 PM–8 AM. Notifications about money are sent **only after the database commit**.

| To | Events |
|---|---|
| Student | OTP (SMS) · enrolment approved/rejected · claim submitted · info requested · approved/partly/rejected · flagged for review · **payment sent with UTR (SMS)** · payment returned by bank (SMS) · bill due day 0/7/12 (SMS on day 12) · blocked (SMS) · bill verified/mismatch · appeal result · recurring claim to confirm · bank or mobile changed (SMS, to old number too) · write-off · allowance reset (June) · graduation warning · ticket replies |
| Approver | New claim assigned · Emergency claim (SMS) · SLA near · second approval needed · bill to verify · daily summary |
| Sponsor | **Ready to pay (daily digest; Emergency immediately by SMS)** · **payment or donation waiting for your confirmation** · **unconfirmed > 3 days** · **unmatched bank statement line** · **"Not received" ticket** · escalated or unassigned claim · waiting for funds · pool low · reconciliation mismatch · payments paused · unaccounted money (day 45) · weekly approver report · monthly impact report |
| Admin | New sign-ups to confirm · roll-list mismatches · enrolment backlog alert · graduation review list |

- Templates use plain language. Regional languages are still to be decided.
- All SMS templates must be registered on TRAI DLT before launch.
- **WhatsApp Business** is evaluated in Phase 6 as an alternative to SMS.

---

## 11. Privacy and security

```
 PERSONAL DATA STORE (encrypted)          AUDIT LOG / LEDGER
 name, mobile, email, bank, ID image ◄─id─ only person_id + actions
 per-person encryption key                (personal values encrypted
                                            with the per-person key)
```

- **Hosting in India.** Data processing agreements with the cloud host and SMS provider.
- **Deletion requests (DPDP Act):** handled only after all of the student's claims are closed and the retention period below has passed. The per-person key is destroyed; the audit log and ledger keep only the id, so the hash chain and money trail stay intact.
- **Under-18 students:** verifiable parent or guardian consent at sign-up. 🟡 legal
- **What approvers can see:**
  - Bank account: last 4 digits only
  - Mobile and email: hidden; approvers contact students through in-app messages
  - Documents: a viewer with a watermark (approver name + time); downloads off. It can't stop screenshots; the confidentiality agreement covers this.
  - Health documents: health-cleared approvers only
- **Security controls:**
  - Every endpoint checks the user's role.
  - Data is encrypted in transit and at rest.
  - Staff: authenticator app (TOTP), 30-minute session timeout, account lockout. Students: SMS/email OTP.
  - Rate limits on login and OTP.
  - File uploads are checked for type and size and scanned for viruses (quarantined until the scan passes).
  - OWASP Top 10 baseline, key rotation every 6 months, external penetration test before the pilot.
  - **Net-banking credentials are never entered into or stored in the app.**
- **Download my data:** a JSON + PDF summary within 7 days.
- **Roll-list sharing** is covered by an **MoU with the college** (purpose, retention, lawful basis). 🟡 college
- **Grievance officer:** named in the privacy notice, with a 30-day response target. A Help button on every claim opens a ticket linked to that claim.

**Retention [default]**

| Data | Kept for |
|---|---|
| ID card image | Enrolment confirmed + 90 days |
| Bank details and passbook/cheque photo | Graduation + 1 year, or until open claims are closed, whichever is later |
| Claim documents and bills | 8 years |
| Payment proofs and bank statements | 8 years |
| Ledger and audit log | 8 years, then personal links anonymised |

---

## 12. Cost survey (outside the app)

There is **no survey inside the app**. Instead, a one-time cost survey runs **before Phase 2**:
- **Tool:** Google Forms or Tally, anonymous (department and year only).
- **Who:** current students (shared only through class groups or with a class code) and recent alumni (graduated in the last 1–2 years).
- **Official fee, hostel, mess and exam amounts** come from college documents, not the survey.
- **Sponsors** fill a separate short form: yearly budget, priority categories, comfort with auto-approve.
- **Min/Typical/Max** are set from the **median** answers, cross-checked with alumni answers and official data, then confirmed by the sponsors.
- Optional later: an anonymous in-app feedback form for logged-in students, run once a year.

---

## 13. User flows (screen by screen, per role)

Sections 2–8 describe the system logic. This section describes what each person **sees and taps**, screen by screen.

### 13.1 Student

```
 ┌───────────┐   ┌───────────┐   ┌────────────┐   ┌──────────────┐   ┌──────────────┐
 │ Landing   │──►│ Sign up   │──►│ OTP verify │──►│ Upload ID    │──►│ "Waiting for │
 │ Login /   │   │ form      │   │ mobile     │   │ card         │   │ confirmation"│
 │ Sign up   │   │ (program, │   │ (+ email)  │   │              │   │ (status page)│
 └───────────┘   │ passing   │   └────────────┘   └──────────────┘   └──────┬───────┘
                 │ year)     │                                 SMS: approved │
                 └───────────┘                                              ▼
 ┌─────────────────────────────── HOME ───────────────────────────────────────────┐
 │ Hi Priya · Year 3 · BE CSE                                                     │
 │ ⚠ Banner if blocked: "Upload bill for SSDF-2026-000123"  [Upload now]           │
 │ Yearly cap: ₹18,000 used of ₹50,000  ▓▓▓▓░░░░░░                                 │
 │ My categories: Books ₹1,200 left · Hostel ₹6,000 left · …                       │
 │ [ Claim fund ]   My claims (3)   Bills due (1)   Notifications   Profile   Help │
 └────────────────────────────────────────────────────────────────────────────────┘
        │ Claim fund
        ▼
 Step 1 Category ─► Step 2 Type ─► Step 3 Amount ─► Step 4 Documents ─► Step 5 Items ─►
 (eligible only)    (advance /      & purpose       (camera or file,    (optional)
                     reimbursement /                 bill no., vendor,
                     institution)                    bill date)
 ─► Step 6 Declaration ─► Step 7 Review ─► [Submit] ─► "Submitted: SSDF-2026-000123"
                                         (or Save draft)

 MY CLAIMS ─► Claim detail
   ┌─────────────────────────────────────────────────────┐
   │ SSDF-2026-000123 · Books · ₹4,000 · PAID             │
   │ Paid on 12 Nov by IMPS · UTR 4321098765              │
   │ Timeline: Submitted ✔ Approved ✔ Paid ✔ Bill ◯       │
   │ Bill due in 6 days  [Upload bill]                    │
   │ Actions by status:                                   │
   │   Info requested  → [Edit & resubmit]                │
   │   Partly approved → [Accept ₹3,000] [Cancel]         │
   │   Rejected        → [Appeal] (7 days)                │
   │   Before payment  → [Cancel claim]                   │
   │   Paid 3+ days    → [Not received]                   │
   │   Spent less      → [Return money] (enter UTR)       │
   │   Monthly category→ [Repeat monthly ☐] [Confirm]     │
   │   Any             → [Help] (opens ticket)            │
   └─────────────────────────────────────────────────────┘

 PROFILE ─► Bank details ─► OTP ─► passbook/cheque photo ─► "Active after check + 48 h"
         ─► Personal details (read only; ask admin to change)
         ─► Privacy: download my data · request deletion
         ─► My tickets
```

### 13.2 Approver

```
 Invite email ─► Set password ─► Authenticator app ─► Confidentiality agreement [I agree]
              ─► Training guide + sample cases ─► DASHBOARD
 ┌──────────────────────────── DASHBOARD ─────────────────────────────┐
 │ 🔴 Emergency (1)  ·  My queue (6)  ·  2nd approvals (2)             │
 │ Bills to verify (4)  ·  Appeals (1)  ·  Today approved ₹12,000/50k  │
 └────────────────────────────────────────────────────────────────────┘
        │ open claim
        ▼
 ┌──────────────────────── CLAIM REVIEW ──────────────────────────────┐
 │ Student: Priya · 21CS045 · BE CSE · Year 3 · bank ••••4821         │
 │ ⚠ Flags (if any): "Same bill used in SSDF-2026-000087"             │
 │ Category, type, amount, purpose │ Documents (watermarked viewer)   │
 │ Allowance left · yearly cap left · past claims · open bills        │
 │ [Approve] [Approve part ₹__] [Reject + reason] [Ask info]          │
 │ [Conflict of interest]   [Message student]                         │
 └───────────────────────────────┬────────────────────────────────────┘
                                 ▼
                     Step-up check ─► back to queue (next claim)

 BILLS TO VERIFY ─► bill viewer beside the amount paid
                 ─► [Match] [Spent less ₹__] [Mismatch + reason] [Give extension]
```

### 13.3 Sponsor

```
 Invite ─► password ─► Authenticator app ─► DASHBOARD
 ┌──────────────────────────── DASHBOARD ─────────────────────────────┐
 │ Pool ₹1,24,000 (reserved ₹18,000) · Bank statement: 3 days old     │
 │ 🔴 Ready to pay (5, 1 Emergency) · Waiting for funds (2)            │
 │ To confirm: payments (3) · donations (1) · Unmatched lines (1)     │
 │ Not received (0) · Unaccounted ₹6,500 (2) · Tickets (2)            │
 │ Reconciliation: ✔  ·  [⏸ Pause all payments]                       │
 └────────────────────────────────────────────────────────────────────┘
 Menu:
 ├─ Ready to pay ─► claim ─► payee name, bank, IFSC, amount [Copy]
 │                ─► pay in net banking ─► [Record payment: UTR, mode, date] ─► step-up
 │                ─► or select many ─► [Export bank batch file]
 ├─ To confirm ─► payments / donations recorded by someone else ─► [Confirm] [Query]
 ├─ Bank statement ─► upload CSV ─► matched / unmatched report ─► resolve
 ├─ Donations ─► [Record donation: donor, amount, mode, bank ref] ─► receipt PDF
 ├─ Bank checks ─► new/changed student bank details ─► photo vs entered ─► [Name matches]
 ├─ Unaccounted ─► [Send reminder] [Deduct] [Request return] [Write off + 2nd person]
 ├─ Refunds ─► claim ─► reason ─► step-up (+ 2nd person above ₹5,000)
 ├─ Categories ─► add / edit / retire ─► 2nd person ─► new version saved
 ├─ Rules & settings ─► thresholds, caps, grace days, sample % ─► 2nd person
 ├─ Approvers ─► invite / change limit / health-cleared / pause / deactivate
 ├─ Institution payees ─► add ─► bank check (2 persons)
 ├─ Tickets ─► student help and "not received" ─► reply / close
 ├─ Reports ─► by category / year / department / approver · impact · CSV
 └─ Audit log ─► search by ref, UTR, person, date ─► verify hash chain
```

### 13.4 Admin

```
 Invite ─► password ─► Authenticator app ─► DASHBOARD
 ├─ Pending sign-ups ─► compare form vs roll list vs ID image ─► [Approve] [Reject + reason]
 ├─ Students ─► search ─► edit ID/program/passing year/status ─► reason ─► step-up
 │              (name changes need a 2nd admin or sponsor)
 ├─ Graduation review ─► list due to graduate ─► [Graduate] [Keep active + reason]
 ├─ Account recovery ─► identity check for lost phone ─► 48 h cooling-off starts
 ├─ Import roll list (CSV) ─► preview ─► matched / mismatched report ─► confirm
 ├─ Programs ─► add / edit course length
 └─ My actions log
 (No menu items for claims, money or reports.)
```

### 13.5 Owner

```
 Login + Authenticator ─► Sponsors (create/deactivate, 2 owners) · Admins (create/deactivate)
                       ─► Global settings (academic year start, programs)
                       ─► Full audit log · all reports
```

---

## 14. Data model (main tables)

| Table | Key fields |
|---|---|
| persons | id, name, mobile, email, encryption_key_ref (encrypted; erasable) |
| users | id, person_id, role, status, password_hash, totp settings |
| sessions, otp_attempts | user, device, time, result |
| programs | name, course_length |
| students | user_id, student_id (unique), program, passing_year, income_band, flagged |
| roll_list_entries | student_id, name, program, passing_year, import_batch |
| enrolment_applications | student, attempt_no, status, reviewed_by, reason |
| bank_accounts / bank_account_versions | student, holder_name, account (encrypted), ifsc, upi_id, proof_file, name_checked_by, verified_at, cooling_off_until |
| approvers | user_id, limit, daily_cap, active, health_cleared, agreement_version, term_until |
| approver_conflicts | approver, student, reason |
| sponsors | user_id |
| institution_payees | name, bank (encrypted), verified_by (2) |
| categories / category_versions | group, support_type, min, typical, max, eligible_years, payee_type, claim_types, health |
| claims | ref, student, category_version, type, amount_requested, amount_approved, status, academic_year, assigned_approver, second_approver, bill_due_at, rule_version, flags, linked_claim |
| claim_items | claim, description, amount |
| documents / document_versions | claim, kind (proof/bill), file_key, sha256, bill_no, vendor, bill_date, uploaded_by, scan_status |
| claim_events | claim, from_status, to_status, actor, reason, time |
| allowances | student, category, period, reserved, used |
| **payments** | ref, claim (UNIQUE), payee_type, payee_bank_snapshot, amount_paise, mode (NEFT/IMPS/RTGS/UPI/cheque), **utr (UNIQUE)**, paid_on, from_account, paid_by, proof_file, batch, confirmed_by, confirmed_at, statement_line, status |
| **donations (topups)** | ref, sponsor/donor, amount_paise, mode, **bank_ref (UNIQUE)**, received_on, recorded_by, confirmed_by, statement_line, receipt |
| **bank_statement_imports** | file, period_from, period_to, closing_balance, uploaded_by |
| **bank_statement_lines** | import, date, amount_paise, direction, reference/UTR, matched_type, matched_id |
| ledger_entries | ref, debit_account, credit_account, amount_paise, claim, payment/donation, time |
| appeals, extensions, write_offs, returns | claim, actor, reason, outcome, utr (returns) |
| recurring_claims | student, category, amount, until, last_confirmed |
| messages | claim, from, to, body, time |
| tickets | claim, student, kind (help / not received), messages, status |
| system_flags | payments_paused, paused_by, resumed_by (2) |
| audit_events | actor, role, action, entity, field names, encrypted values, reason, ip, device, request_id, prev_hash, hash |
| config_versions | key, value, version, changed_by, approved_by |
| notifications, consents | user, template/version, channel, time, delivered |

---

## 15. Non-functional requirements and cost

- **App type:** mobile-first **PWA** (camera upload, installable, no app-store cost). Works on low-end Android phones on 3G.
- Large tap targets and clear labels. English first. Accessibility: WCAG 2.1 AA.
- **Size [default]:** up to 5,000 students, about 500 claims a month, 10–20 approvers.
- **Manual payment limit:** manual paying works well up to about **15 payments a day** [default]. Beyond that, or if sponsors spend more than an hour a day paying, move to v1 (section 21).
- **Stack:** Next.js run as a normal Node server (not serverless) + PostgreSQL + S3-compatible storage. No payment gateway in v0.
- **Production:** 1 VPS in India (2 vCPU / 4 GB), for example Hostinger Mumbai, E2E Networks or DigitalOcean BLR. App, worker and PostgreSQL in Docker.
- **Staging:** Oracle Cloud Always Free (Mumbai/Hyderabad), fake SMS, test data only. A startup check refuses to start if the environment and settings don't match.
- **Files and backups:** Cloudflare R2 (or Backblaze B2). Nightly full backup + continuous WAL backup (pgBackRest or WAL-G), encrypted, offsite. **Monthly restore test** onto staging with a ledger-total check.
- **Ops baseline:** automatic security updates, firewall (80/443/SSH), SSH by key only, database never public, secrets in an encrypted env file (never in git), disk and backup-failure alerts, Sentry + UptimeRobot (free tiers), Cloudflare free for DNS/SSL.
- If the team isn't comfortable running PostgreSQL themselves, switch to a managed PostgreSQL (≈ ₹1,300/month extra).
- Apply for **AWS / Google / Microsoft nonprofit credits** once the entity is registered.
- **Uptime target:** 99% (monthly). **RPO** ≤ 15 min, **RTO** ≤ 4 h.
- **Scheduled jobs** (reminders, day-14 block, expiry, SLA, statement-match alerts) are **safe to re-run** and **catch up** after downtime. Deadlines use stored dates in **Asia/Kolkata**.
- **Monitoring and alerts:** payments unconfirmed > 3 days, unmatched statement lines, reconciliation mismatches, low pool, error rate and downtime.

**Monthly running cost (500 claims/month) [default]**

| Item | Cost |
|---|---|
| Hosting (prod VPS; staging, storage, email and monitoring on free tiers) | ₹800–1,400 |
| SMS (staff on authenticator app; in-app and email first) | ₹600–1,000 |
| Bank charges (NEFT often free on current accounts; IMPS a few ₹ each — check with your bank) | ₹0–1,500 |
| Domain | ~₹100 |
| **Total** | **≈ ₹1,500–4,000** |

- No gateway fees in v0 (no ~2% on donations, no per-payout fee).
- **One-time:** DLT registration (≈ ₹5,000–6,000 + GST), penetration test, legal and CA fees.
- All prices are approximate. Check current pricing before committing.

---

## 16. Prerequisites (outside the app, start now)
1. A registered legal entity (trust, society or Section 8 company) with a PAN
2. A **current account** in the trust's name with **corporate net banking** that supports maker–checker (two-person) payment approval and statement download as CSV, and ideally bulk payment upload
3. **At least 2 owners and 2 sponsors** named and willing to act (the two-person rule needs them)
4. 12A/80G registration, if donors want tax receipts 🟡 CA
5. **FCRA** registration if any donor gives foreign money 🟡 CA
6. TRAI DLT registration for SMS (takes 2–4 weeks)
7. **MoU with the college** (data sharing, use of the college name, student contact point) 🟡 college
8. **Fund Rules policy** approved by the sponsors 🟡 sponsors
9. Privacy notice and terms, with legal help
10. **Cost survey** completed and Min/Typical/Max set (section 12)

---

## 17. Build phases (estimate assumes 2 developers)

| Phase | Scope | Est. |
|---|---|---|
| 0 | Prerequisites (section 16) + cost survey (before Phase 2), run alongside the build | ongoing |
| 1 | Auth (TOTP for staff, account recovery), roles, two-person actions, programs, admin records, roll list, personal-data store, audit log with crypto-shredding | 4 wks |
| 2 | Categories, claim form, real-time checks, fraud checks, rules engine, full status map, timeline | 4 wks |
| 3 | Approver portal: assignment, limits, daily cap, second approval, SLA, appeals, conflicts, health-cleared, approver management, confidentiality agreement, watermarked viewer, tickets | 3 wks |
| 4 | **Manual money:** donations (record + confirm), ready-to-pay list, record payment, confirm payment, bank statement import and matching, ledger accounts, reservations, waiting-for-funds queue, bank-detail checks and cooling-off, returns, receipts, pause switch, limits, batch export | 2 wks |
| 5 | Bills: upload, verification, reminders, block and re-block, extensions, recovery, write-off, recurring claims, settlement | 3 wks |
| 6 | Reports (impact, approver activity, trace views), notifications polish, monitoring, backups, retention jobs, penetration test, usability testing | 4 wks |
| | **Build total** | **≈ 20 weeks** |
| 7 | Soft launch: pilot → wider → full (section 20, S7) | 8 wks |

Notifications are built as part of every phase, not left to the end.

---

## 18. Open decisions
1. Names of the owner(s), sponsors and the grievance officer
2. Income bands and the priority rule (section 2)
3. Regional languages, and the phase they arrive in
4. 80G / 12A / FCRA (CA to confirm)
5. Which categories are paid to institutions, and which institutions are registered payees
6. Which bank, and its statement CSV and bulk-upload formats
7. How often the bank statement is uploaded (weekly [default] or daily)
8. Final confirmation of all **[default]** values after the cost survey:
   - Two-approver threshold ₹10,000
   - Approver daily cap ₹50,000
   - Student yearly cap ₹50,000
   - Emergency cap ₹25,000
   - Bill grace period 14 days
   - One unbilled advance per student
   - Two auto-approvals per student per month
   - Refund two-person threshold ₹5,000
   - Low-pool alert ₹25,000
   - Daily payment limit ₹1,00,000
   - Manual payment limit 15 a day
   - Retention periods
   - Size assumption

---

## 19. Verification

| Area | Test |
|---|---|
| Passing year | Year correct across the June boundary; 2-, 4- and 5-year programs; out-of-range year rejected; graduation review list; duplicate student ID rejected |
| Enrolment | Roll-list match and mismatch paths; 3 resubmissions max; under-18 consent captured |
| Account recovery | Forgot password; lost phone → admin check → 48 h cooling-off → old contacts alerted |
| Real-time block | Clock to day 13 / 14 / 15 → allowed / blocked / blocked; upload → unblocked instantly; mismatch → re-blocked immediately; Emergency always allowed; second advance blocked until first bill verified |
| Claim types | Reimbursement and institution skip the bill clock; institution payment goes to the payee account |
| Routing | ≤ Typical and ≤ ₹10k reimbursement → auto; advance and Emergency never auto; 3rd auto-claim in a month → approver; flagged never auto; Emergency 8 working-hour SLA; health claim only to health-cleared |
| Approval controls | Over-limit approval refused by the server; approved amount > ₹10k needs a different second approver; daily cap respected; appeal goes to a different approver; sponsor fallback needs a second person |
| Timers | Partly approved no reply 7 d → CANCELLED; INFO REQUESTED 14 d, DRAFT 30 d, WAITING FOR FUNDS 60 d expiry |
| Recurring | Monthly confirmation required; stops on status change or category retirement |
| Fraud | Same file or bill number on a second claim → flagged; payment held during bank cooling-off; old contact details alerted |
| Allowance and caps | Reserve, use and release; per-period Max; yearly cap blocks the form; submitted-year rule; two tabs at once can't pass the cap |
| **Manual payments** | Record payment → PAID (unconfirmed); same UTR twice rejected; second payment for the same claim rejected; two sponsors recording at once → only one succeeds; confirmer = payer refused; statement line auto-confirms; unmatched debit alerts; unconfirmed > 3 days alerts; returned payment → RETURNED → READY TO PAY after bank fix; "Not received" opens a ticket |
| **Donations** | Recorded donation not in the pool until confirmed; same bank ref twice rejected; unmatched credit shown as "record this donation"; receipt generated |
| **Pause and limits** | 1 sponsor pauses → Ready to pay locked; resume needs 2 different persons; daily limit reached → stop + alert; same bank account > 3 times in 7 days → hold |
| Reconciliation | Ledger ≠ statement balance → payments paused until 2 sponsors sign off |
| Recovery | Day 30 / 45 tasks; return (with UTR), deduct and write-off ledger entries; write-off needs 2 persons; flagged after write-off |
| Traceability | Every write and every approver view creates an audit row; editing a row breaks the hash chain; any claim's full timeline, including UTR and confirmer, can be rebuilt |
| Roles | One test per row of the permission matrix (section 1); a student's person can't hold a staff role |
| Privacy | Crypto-shredding erasure keeps the hash chain valid; approvers can't download documents or see contact details |
| Notifications | Delivered in channel order; quiet hours respected; failed reminders pause the bill clock |
| Operations | Backup restored to staging with ledger-total check; alerts fire; jobs catch up after downtime; load test at fee-season peak |
| Usability | Claim submitted in under 3 minutes on a ₹8,000 Android phone over throttled 3G; WCAG 2.1 AA |
| Security | External penetration test before the pilot |

---

## 20. Money safety in v0 (manual payments)

Review lens: **what goes wrong when people pay by hand, many people use the app at the same time, and real rupees move.**
Severity:
- 🔴 can lose money or pay the wrong person
- 🟠 can block students or corrupt records
- 🟡 operational pain

### S1. Paying twice or paying wrongly 🔴

| # | What can happen | Fix |
|---|---|---|
| 1 | **Paid twice.** Two sponsors pay the same claim, or one pays twice in net banking. | Ready to pay shows "being paid by Sponsor A" once someone opens it. One payment per claim (`UNIQUE(claim_id)`), UTR unique. The bank statement shows any second debit as **unmatched** → alert → recovery from the student. |
| 2 | **Paid the wrong account.** The payer types the account number by hand. | Payee details come only from the verified **snapshot**, with a [Copy] button. Add the student as a saved beneficiary once; the bank's name check is ticked in the app. |
| 3 | **Typo in the UTR or amount.** | The statement match fails → shown as unmatched → the payer corrects it (audited). |
| 4 | **"Paid" recorded but money never sent.** | The confirmer checks the bank; the statement match is final; the student has the "Not received" button. |
| 5 | **Money sent but not recorded.** | An unmatched debit on the statement → alert to all sponsors. |
| 6 | **Bank returns the money days later.** | RETURNED status: ledger reversal, allowance kept reserved, bill clock cancelled, student fixes bank details. |
| 7 | **Rounding errors.** | All amounts in **whole paise (integers)**. |
| 8 | **"You're paid" sent before the payment is recorded.** | Notifications only after the database commit. |

### S2. Two things at the same time 🔴🟠

| # | Situation | Fix |
|---|---|---|
| 9 | Two approvals at once both reserve from the same pool → pool goes negative | Reservation is a single locked transaction; the database refuses a negative available balance |
| 10 | A student submits from 2 tabs at once → both pass the yearly cap | Lock per student at submit; the cap is checked inside the same transaction |
| 11 | A student submits at the exact second the bill block starts | The block is checked inside the submit transaction |
| 12 | Two sponsors press "Record payment" at once | READY TO PAY → PAID in one locked step; the second one fails |
| 13 | The same person acts twice in a two-person action using two accounts | Check **distinct persons**, not accounts |
| 14 | A sponsor lowers a cap while claims are already reserved above it | New rules apply only to **new** claims; in-flight claims keep their rule version |

### S3. Fake or wrong records (insider risk) 🔴

| # | Risk | Fix |
|---|---|---|
| 15 | **Fake donation entry** makes the pool look bigger | Donations count only after confirmation by a second person or the bank statement |
| 16 | **A sponsor pays a friend and records it against a claim** | Payee details are locked to the verified snapshot; the statement line must match the claim's amount and UTR; the bank's own maker–checker |
| 17 | **A developer or database admin edits a claim to APPROVED** | The Ready to pay list re-checks the full approval record (approvers, step-up proof, limits). Production database access needs 2 people. The hash chain detects edits. |
| 18 | **SIM swap or phone change → bank change → steal a payment** | Mobile change = bank change: 48 h cooling-off, alert to the old email and number |
| 19 | **Admin and student collude** (admin edits the student's mobile or bank) | Admin can't edit contact or bank details |
| 20 | **Net-banking login shared or stolen** | Never stored in the app; bank maker–checker; daily limit set in the bank too |

### S4. Safety switches 🔴

```
 ┌────────────────────────────── LIMITS ─────────────────────────────────┐
 │ • Daily payment limit        e.g. ₹1,00,000/day → stop + alert        │
 │   (sponsors can raise it for fee season — 2 persons)                  │
 │ • Same bank account paid > 3 times in 7 days → hold + review          │
 │ • Reconciliation mismatch    → payments PAUSED until 2 sponsors OK    │
 └───────────────────────────────────────────────────────────────────────┘
 PAUSE SWITCH: any 1 sponsor can PAUSE ALL PAYMENTS (one tap + step-up)
               resuming needs 2 sponsors. Students see "payments delayed".
```

### S5. When outside services fail 🟠

| # | Failure | Fix |
|---|---|---|
| 21 | **SMS provider down** | OTP falls back to email; staff use the authenticator app. If reminders failed, pause the bill-block clock automatically. |
| 22 | **Bank net banking down** | Claims stay in Ready to pay; Emergency can be paid by another sponsor's authorised channel or by cheque, recorded the normal way |
| 23 | **Virus-scan service down** | Accept into quarantine, show "checking"; approvers can't open the file until the scan passes |
| 24 | **App down or a scheduled job missed** | Jobs are safe to re-run and catch up; deadlines come from stored dates |

### S6. Time and calendar bugs 🟠

| # | Bug | Fix |
|---|---|---|
| 25 | Server runs in UTC, so "day 14" ends at 5:30 AM IST | All deadlines in **Asia/Kolkata**, stored with the timezone |
| 26 | June 1 rollover while claims are in progress | The academic year is stored on each claim at submit; the rollover job doesn't touch open claims |
| 27 | Fee-deadline peaks (hundreds of claims in one week) | Load-test the peak; sponsors can raise approver daily caps and the daily payment limit temporarily; use bulk pay; institution fees with due dates go first |

### S7. Incident handling and go-live 🟡

| Situation | Runbook (first 3 steps) |
|---|---|
| Double or wrong payment | Pause switch → check the bank statement → recovery request to the student + ledger RECOVERY entry |
| Payment not received by student | Check the UTR with the bank → if returned, RETURNED flow → tell the student |
| Reconciliation mismatch | Payments auto-paused → compare ledger with the bank statement → 2 sponsors sign off before resuming |
| Data leak | Revoke access → find the scope from the audit log → **notify the Data Protection Board and affected people promptly** (DPDP Act) |
| SMS down | Status banner → fallbacks on (S5) → notify sponsors |

- **Who gets alerted:** 2 named tech volunteers + all sponsors. There's no 24/7 staff, so payments are made **9 AM–9 PM IST**. Outside those hours an Emergency claim alerts the on-call sponsor.
- **After restoring a backup:** compare with the latest bank statement before reopening. A backup can "forget" payments that really happened.

```
 Week 1–4  PILOT   1 department · claims ≤ ₹2,000 · daily limit ₹20,000
                   every payment confirmed by a second sponsor the same day
 Week 5–8  WIDER   all departments · ≤ ₹10,000 · daily limit ₹50,000
 Week 9+   FULL    normal limits; safety switches stay on forever
 Before pilot: drills for double record, wrong UTR, returned payment,
               unmatched statement line, SMS down, pause switch
```

---

## 21. Moving from v0 to v1 (automatic payments)

**When:** more than about 15 payments a day, sponsors spending too long paying, or online donations needed.

**v1 adds** (as designed in `docs/ssdf-plan-v3.md` §6 and §20):
- Online donation checkout through a payment gateway (e.g. Razorpay), with signed webhooks
- Automatic payouts (e.g. RazorpayX) using an outbox worker, idempotency key = claim id
- Statuses PAYOUT_PROCESSING and PAYOUT_UNKNOWN (never retry unless the gateway confirms failure)
- Gateway settlement tracking and a 3-way reconciliation (ledger, gateway, bank)
- Chargeback handling and a 5% chargeback reserve
- Penny-drop bank checks by API
- Gateway safety: secret manager for keys, server IP whitelist, gateway-side approval rule for large payouts
- New prerequisites: gateway business KYC and payout activation; confirm the gateway allows welfare payouts to individuals
- Fees: about ~2% + GST on online donations; about ₹4–9 + GST per payout

**Built in v0 so v1 is easy:**
- The `payments` table already has one row per claim, a bank snapshot and a UTR; v1 adds gateway columns (payout id, fund account id, fee, tax).
- The ledger, statuses and reconciliation screens stay; v1 adds the gateway as a third balance.
- Manual payment stays available in v1 as the fallback when the gateway is down.
