# SSDF v0: full flow in one diagram

Sign-up to a closed claim. See `ssdf-plan-v0.md` for the details behind each box.

```
══════════════ 1. SIGN-UP (once, automatic, no admin approval) ═════════════════
 Form: name, student ID, passing year, mobile, password
   │
   ├─ Student ID already used? ─► "Account exists, log in"
   ▼
 OTP on mobile (rate-limited)
   ▼
 ID + name + passing year on the college roll list?
   │ Yes ─► ACTIVE immediately
   │ No  ─► "Not found on the college list" ─► cannot sign up
   After the passing year's academic year ends ─► GRADUATED
   (no new claims, must still close open bills)

══════════════ 2. MAKING A CLAIM ══════════════════════════════════════════════
 Student taps "Claim fund"
   ▼
 SERVER CHECK (on open AND again inside the submit transaction)
   ✖ not ACTIVE            ✖ advance bill overdue > 14 days
   ✖ bill MISMATCH open    ✖ unbilled advance open (for a new advance)
   ✖ bank details unverified (student-paid claims only)
   │ blocked ─► clear message + [Upload bill]      (Emergency is never blocked)
   │ allowed
   ▼
 1 Category ─► 2 Type (advance / reimbursement / institution)
 ─► 3 Purpose, period, amount (≤ category Max; no yearly cap)
 ─► 4 Documents + bill no., vendor, bill date ─► 5 Declaration ─► Submit
 ─► Ref SSDF-2026-000123

══════════════ 3. ROUTING AND APPROVAL ════════════════════════════════════════
 FRAUD CHECKS
   same file hash?  same bill no. + vendor + amount?  student flagged?
   │ hit ─► FLAGGED ─────────────────────────────┐
   │ clear                                       │
   ▼                                             │
 AUTO-APPROVE? all true:                         │
   reimbursement or institution (never advance / Emergency),
   ≤ Typical, ≤ ₹10,000, documents present, < 2 auto-approvals this month
   │ yes ─► AUTO-APPROVED (5% post-checked)      │ no
   │                                             ▼
   │                     ASSIGN approver: round-robin, limit ≥ amount,
   │                     daily cap ₹50,000, no conflict (none ─► sponsor)
   │                     SLA 3 days (Emergency 8 working hours)
   │                                             │
   │          ┌── INFO REQUESTED (student has 14 d)
   │          ├── REJECTED ─► APPEAL once in 7 d (different approver)
   │          ├── PARTLY APPROVED ─► student accepts / cancels in 7 d
   │          ├── CONFLICT ─► reassigned
   │          ├── APPROVED (≤ ₹10,000)
   │          └── AWAITING 2ND APPROVAL (> ₹10,000, different approver) ─► APPROVED
   ▼                                             ▼
   └─────────────────────┬───────────────────────┘
                         ▼
══════════════ 4. MANUAL MONEY (trust bank account) ═════════════════════════
 Donor pays trust ─► donation recorded ─► confirmed (2nd person / bank statement) ─► POOL

 Pool has enough? ─ No ─► WAITING FOR FUNDS (expires 60 d)
                          order: 1 Emergency, 2 institution fee with due date, 3 oldest
                    Yes ─► amount RESERVED ─► READY TO PAY
                          ▼
 Sponsor pays in net banking (outside the app)
   ▼
 "Record payment": UTR, mode, date, amount ─► PAID (unconfirmed)
   (same UTR twice rejected; one payment per claim)
   ▼
 Different sponsor confirms, OR bank-statement import matches UTR + amount ─► PAID ✔
   Bank returns it ─► RETURNED ─► student fixes bank ─► READY TO PAY
   Student presses "Not received" after 3 days ─► ticket to sponsors
   Controls: pause switch, daily payment limit, same account > 3 times in 7 d ─► hold

══════════════ 5. BILLS AND CLOSURE ═══════════════════════════════════════════
 Reimbursement / institution ─► bill already attached or institution receipt ─┐
 Advance ─► 14-day clock starts at payment                                    │
   day 7 reminder · day 12 warning · day 14 BILL OVERDUE (blocks new claims)  │
   one 7-day extension possible · day 30 approver message · day 45 sponsor alert
   ▼                                                                          │
 BILL SUBMITTED ─► approver verifies against amount paid                      │
   ├─ MATCH ─► BILL VERIFIED ◄────────────────────────────────────────────────┘
   ├─ SPENT LESS ─► student returns difference (UTR) or it is deducted
   ├─ SPENT MORE ─► student may file a reimbursement for the difference
   └─ MISMATCH ─► student fixes; RE-BLOCKED immediately
   ▼
 SETTLED ✔ │ RETURNED ✔ │ ADJUSTED ✔ │ WRITTEN OFF ✖ (2 persons; student flagged)

══════════════ THROUGHOUT ═════════════════════════════════════════════════════
 Every write and approver view ─► audit row (hash chain)
 Notifications in channel order; quiet hours 9 PM to 8 AM
 Student can CANCEL any time before a payment is recorded
 The server rejects any status move not on the status map (plan section 8)
```
