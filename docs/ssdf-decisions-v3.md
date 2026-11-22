# SSDF fund app: final decisions (v3 review)

This document records the decisions taken after a full review of [ssdf-plan-v3.md](ssdf-plan-v3.md).
Where it conflicts with the plan, **this document wins**. The plan should be updated to match (see section 12).

Status key:
- ✅ **Decided**: build it this way
- 🟡 **Needs confirmation**: decision made, but a person or authority outside the team must confirm it

---

## 1. Roles and governance

| # | Decision | Status |
|---|---|---|
| 1.1 | **"Trustee" in §20 means Sponsor.** All §20 references to trustees, "2-trustee" actions and panels are rewritten as **Sponsor** actions. There is no panel voting: approval stays *one approver, plus a second approver above ₹10,000* (§5). | ✅ |
| 1.2 | **Two-person rule** (two *distinct persons*, not accounts) for: creating sponsors (2 owners), creating approvers or raising approver limits, changing rules or caps, write-offs, refunds above ₹5,000 [default], and resuming payouts after the kill switch or a reconciliation pause. | ✅ |
| 1.3 | A person with a student account can **never** be an approver, sponsor or admin. The check is done on `person_id`. | ✅ |
| 1.4 | The sponsor's fallback approval stays, but every fallback approval also needs a second sponsor or approver. | ✅ |
| 1.5 | **Help tickets and grievances** are handled by the **Sponsor** (the grievance officer is one named sponsor or owner). Ticket screens are added for sponsor and student. | ✅ |
| 1.6 | **Account recovery** flow added for everyone: forgot password (email + mobile OTP); lost or changed phone = admin identity check + 48 h cooling-off + alert to old contacts (same as a bank change). | ✅ |
| 1.7 | **Staff offboarding:** deactivating an approver moves their open queue back to round-robin; deactivating an admin returns pending enrolments to the shared queue. | ✅ |
| 1.8 | A written **Fund Rules policy** (eligibility, categories, caps, approver code of conduct) is approved by the sponsors. The app enforces it and does not invent rules. | 🟡 sponsors |
| 1.9 | **Approvers:** alumni or faculty volunteers, chosen by the sponsors, 1-year renewable term, short training with written guidelines and sample cases before their first claim. | 🟡 sponsors |
| 1.10 | **Need-based priority:** at sign-up, students declare a family income band (ranges only, no documents). Students in the lowest band get first priority in the waiting-for-funds queue, after Emergency. Caps stay the same for everyone in v1. | 🟡 sponsors |

## 2. Sign-up and students

| # | Decision | Status |
|---|---|---|
| 2.1 | **Course length is per program, not per department** (BE 4 yrs, ME 2, MBA 2, …). New `programs` table; students link to a program. | ✅ |
| 2.2 | **Mobile is the main contact.** College email is optional. | ✅ |
| 2.3 | Student ID is **unique**: one account per student. At most 3 enrolment resubmissions, then the student must contact admin. Admin target: confirm enrolment within 3 working days. | ✅ |
| 2.4 | **Under-18 students** (some first-years): verifiable parent or guardian consent is captured at sign-up (DPDP Act). | 🟡 legal |
| 2.5 | **Graduation:** before each June rollover, admin gets a bulk review list of students due to graduate, so students with backlogs aren't wrongly marked GRADUATED. | ✅ |
| 2.6 | Admin **cannot** edit a student's name, contact or bank details after enrolment without a second admin or sponsor approving it (protects the penny-drop name match). | ✅ |
| 2.7 | Penny-drop **name mismatch** goes to manual review by a sponsor instead of a hard failure. | ✅ |

## 3. Claims, caps and bills (loopholes closed)

| # | Decision | Status |
|---|---|---|
| 3.1 | **At most 1 unbilled advance per student** [default]. A second advance is blocked until the first bill is **verified**. | ✅ |
| 3.2 | Uploading a bill lifts the block **provisionally**. A MISMATCH **re-blocks immediately**; there is no new 14-day window. | ✅ |
| 3.3 | **Emergency is never auto-approved.** The approver sees the student's overdue bills. Emergency still isn't blocked by bills, but it counts towards its ₹25,000 yearly cap. | ✅ |
| 3.4 | **Emergency is defined** in the Fund Rules (medical, accident, death in family, disaster), and proof can be submitted within 7 days. | 🟡 sponsors |
| 3.5 | **Partly approved, no reply in 7 days = CANCELLED** (not accepted). Money is never paid without the student's consent. | ✅ |
| 3.6 | The second-approval threshold (₹10,000) is checked on the **approved amount**. | ✅ |
| 3.7 | **Auto-approve limit:** at most 2 auto-approved claims per student per month [default]. Any further claims go to an approver. Auto-approve applies to reimbursement and institution claims only, **never to advances**. | ✅ |
| 3.8 | **5% post-check failures:** the claim gets a FLAGGED note, the student is flagged for manual approval, and the sponsor decides on recovery. | ✅ |
| 3.9 | **Expiry:** DRAFT 30 days, INFO REQUESTED 14 days, WAITING FOR FUNDS 60 days (student told; can resubmit). | ✅ |
| 3.10 | **Cancel** is allowed only **before the payout starts** (not "before PAID"). | ✅ |
| 3.11 | **Spent more than the advance:** the student can file a reimbursement claim for the difference, linked to the original claim. One extension of **7 days** at most. | ✅ |
| 3.12 | **Recurring claims:** the student confirms each month, with a receipt or a one-tap confirmation for institution payees. Recurring claims stop on any status change or category retirement. | ✅ |
| 3.13 | The claim form captures **bill number, vendor and bill date** (needed for the duplicate check). Rejection reasons come from a fixed list plus free text. | ✅ |
| 3.14 | **Allowance definition:** each category's Max applies per its support period (month, semester, year or incident). Open decision 2 is settled: **more than one claim per category per period is allowed, up to the period Max.** | ✅ |
| 3.15 | **Health claims:** documents are visible only to approvers marked "health-cleared" (1–2 people). | ✅ |
| 3.16 | **Approver inactive** for 14 days with an empty queue → paused automatically, and the sponsor is told. | ✅ |

## 4. Money safety (§20 merged into the design)

| # | Decision | Status |
|---|---|---|
| 4.1 | **One payout per claim**, enforced by the database (`UNIQUE(claim_id)`). Idempotency key = claim id. | ✅ |
| 4.2 | New statuses added to the §8 status map: **PAYOUT_PROCESSING, PAYOUT_UNKNOWN, REVERSED**. Ledger types added: **CHARGEBACK, RECOVERY**. | ✅ |
| 4.3 | **Never retry a payout** unless the gateway's status API confirms it failed. | ✅ |
| 4.4 | **Outbox pattern** for payouts. Scheduled jobs are safe to re-run and catch up after downtime. Deadlines use stored dates in **Asia/Kolkata**. | ✅ |
| 4.5 | All amounts are stored as **whole paise (integers)**. | ✅ |
| 4.6 | **"Available to pay"** = the real payout-account balance. Top-ups are usable only after settlement. **Daily 3-way reconciliation** (ledger, gateway, bank statement). A mismatch pauses payouts automatically. | ✅ |
| 4.7 | **Ledger chart of accounts:** Sponsor funds, Pool (settled), Pool (unsettled), Reserved, Paid to students, Paid to institutions, Student receivable, Fees and GST, Write-off expense, Chargeback reserve. Every movement has two sides, and the total is always zero. | ✅ |
| 4.8 | **Chargeback reserve:** 5% of the pool is never paid out [default]. | ✅ |
| 4.9 | **Circuit breakers:** ₹1,00,000/day ceiling, more than 20 payouts/hour, same bank account paid more than 3 times in 7 days. **Kill switch:** any 1 sponsor pauses; 2 sponsors resume. | ✅ |
| 4.10 | **Payout hours: 9 AM–9 PM IST.** Emergency SLA becomes **8 working hours**; outside those hours an Emergency claim pages the on-call sponsor, who may approve a manual payment recorded by two people. | ✅ |
| 4.11 | Payouts store a **copy of the bank account** used. Bank account history is kept, never overwritten. | ✅ |
| 4.12 | Notifications are sent **only after commit and gateway confirmation**. | ✅ |
| 4.13 | **Soft launch** as in §20 B8: pilot (1 department, ≤ ₹2,000) → wider → full. | ✅ |

## 5. Traceability (clarified)

| # | Decision | Status |
|---|---|---|
| 5.1 | Kept as designed: write-once, hash-chained **audit log**; **claim timeline**; **double-entry ledger** (4.7); **versioned documents** (SHA-256, never overwritten); **versioned rules** (each claim stores the rule version that decided it). | ✅ |
| 5.2 | The audit log **never stores personal values** in before/after. It stores field names plus values encrypted with a **per-person key**. Erasure = destroy the key (crypto-shredding), so the hash chain stays valid. | ✅ |

## 6. Privacy and security

| # | Decision | Status |
|---|---|---|
| 6.1 | Hosted **in India**. Data processing agreements with the cloud host, SMS provider and gateway. | ✅ |
| 6.2 | **Staff log in with an authenticator app (TOTP).** Step-up authentication is valid for 10 minutes, not once per approval. Students use SMS/email OTP. | ✅ |
| 6.3 | Baseline: OWASP Top 10, 30-minute session timeout for staff, account lockout, key rotation every 6 months, external penetration test before the pilot. | ✅ |
| 6.4 | "Download my data" = a JSON + PDF summary within 7 days. | ✅ |
| 6.5 | Watermarked viewer kept. It is acknowledged that it **can't stop screenshots**; the confidentiality agreement covers this. | ✅ |
| 6.6 | Roll-list sharing is covered by an **MoU with the college** (purpose, retention, lawful basis). | 🟡 college |

## 7. Survey

| # | Decision | Status |
|---|---|---|
| 7.1 | **No public survey in the app.** §12 and Phase 6 are removed from the build (saves about 1 week). | ✅ |
| 7.2 | Instead, a **one-time cost survey before Phase 2**, using Google Forms or Tally, anonymous (department and year only): **current students** (shared only through class groups or with a class code) and **recent alumni** (graduated in the last 1–2 years). | ✅ |
| 7.3 | **Official fee, hostel, mess and exam amounts** come from college documents, not the survey. | ✅ |
| 7.4 | **Sponsors** fill a separate short form: yearly budget, priority categories, comfort with auto-approve. | ✅ |
| 7.5 | Min/Typical/Max are set from the **median** answers, cross-checked with alumni answers and official data, then confirmed by sponsors. | ✅ |
| 7.6 | Optional later: an anonymous in-app feedback form for logged-in students, run once a year. | ✅ |

## 8. Notifications

| # | Decision | Status |
|---|---|---|
| 8.1 | Channel order: **in-app → email → SMS**. SMS is only for OTPs, payment, block and Emergency alerts. | ✅ |
| 8.2 | **WhatsApp Business** is evaluated in Phase 7 as an alternative to SMS (needs template approval). | ✅ |
| 8.3 | Events added: allowance reset each June, graduation warning, enrolment backlog alert to admin, ticket replies. Quiet hours 9 PM–8 AM for reminders. | ✅ |

## 9. Tech stack and hosting

| # | Decision | Status |
|---|---|---|
| 9.1 | **Stack:** Next.js run as a normal Node server (not serverless) + PostgreSQL + S3-compatible storage + Razorpay / RazorpayX. | ✅ |
| 9.2 | **App type:** mobile-first **PWA** (camera upload, installable, no app-store cost). | ✅ |
| 9.3 | **Production:** 1 VPS in India (2 vCPU / 4 GB), for example Hostinger Mumbai, E2E Networks or DigitalOcean BLR. App, worker and PostgreSQL in Docker. A fixed IP for the RazorpayX whitelist. | ✅ |
| 9.4 | **Staging:** Oracle Cloud Always Free (Mumbai/Hyderabad), **test keys only**. A startup check refuses mismatched environment and keys. | ✅ |
| 9.5 | **Files and backups:** Cloudflare R2 (or Backblaze B2). Nightly full backup + continuous WAL backup (pgBackRest or WAL-G), encrypted, offsite. **Monthly restore test** onto staging with a ledger-total check. | ✅ |
| 9.6 | **Ops baseline:** automatic security updates, firewall (80/443/SSH), SSH by key only, database never public, secrets in an encrypted env file (never in git), disk and backup-failure alerts, Sentry + UptimeRobot (free tiers), Cloudflare free for DNS/SSL. | ✅ |
| 9.7 | If the team isn't comfortable running PostgreSQL themselves, switch to a managed PostgreSQL (≈ ₹1,300/month extra). | ✅ |
| 9.8 | Apply for **AWS / Google / Microsoft nonprofit credits** once the entity is registered. | ✅ |
| 9.9 | **Uptime target:** 99% (monthly). **RPO** ≤ 15 min, **RTO** ≤ 4 h. After any restore, re-sync payout status from the gateway before reopening. | ✅ |

## 10. Running cost

**Monthly (500 claims/month) [default]**

| Item | Cost |
|---|---|
| Hosting (prod VPS; staging, storage, email and monitoring on free tiers) | ₹800–1,400 |
| SMS (staff on authenticator app; in-app and email first) | ₹600–1,000 |
| Payout fees (~₹3–6 each + 18% GST; prefer UPI payouts) | ₹2,000–3,000 |
| Domain, penny-drop checks | ~₹300 |
| **Total** | **≈ ₹3,500–5,500** |

- **Large donations go by NEFT/RTGS/UPI directly to the trust's bank account**, recorded by the sponsor against the bank reference. This avoids the ~2% gateway fee. Gateway checkout is kept for small online donations only.
- **One-time:** DLT registration (≈ ₹5,000–6,000 + GST), penetration test, legal and CA fees.
- All prices are approximate. Check current pricing before committing.

## 11. Prerequisites added (§16)

7. **FCRA** registration if any donor gives foreign money. 🟡 CA
8. Confirm the gateway's terms allow **welfare payouts to individuals**. 🟡 gateway
9. **MoU with the college** (data sharing, use of the college name, student contact point). 🟡 college
10. **Fund Rules policy** approved by the sponsors (1.8). 🟡 sponsors
11. **Cost survey** completed and Min/Typical/Max set (section 7). ✅

## 12. Timeline and plan updates

**Revised build estimate (2 developers)**

| Phase | Scope | Est. |
|---|---|---|
| 0 | Prerequisites + cost survey (before Phase 2) | ongoing |
| 1 | Auth (TOTP for staff, account recovery), roles, programs, admin records, roll list, personal-data store, audit log with crypto-shredding | 4 wks |
| 2 | Categories, claim form, real-time checks, fraud checks, rules engine, full status map | 4 wks |
| 3 | Approver portal, two-person controls, tickets | 3 wks |
| 4 | Payments with §20 safety: outbox, payout statuses, 3-way reconciliation, circuit breakers, kill switch, ledger accounts | 6 wks |
| 5 | Bills, recovery, recurring claims, settlement | 3 wks |
| 6 | Reports, notifications, monitoring, backups, retention jobs, penetration test, usability testing | 4 wks |
| | **Build total** | **≈ 24 weeks** |
| 7 | Soft launch: pilot → wider → full (§20 B8) | 8 wks |

Notifications are built as part of every phase, not left to the end.

**Plan sections to update to match this document:** §1 (roles matrix, two-person rule, tickets, recovery), §2 (programs, under-18), §3 (allowance, recurring), §4 (form fields), §5 (auto-approve rules), §7 (one advance, re-block), §8 (new statuses), §10, §11, §12 (removed), §14 (new tables: `programs`, `roll_list_entries`, `enrolment_applications`, `messages`, `approver_conflicts`, `sessions`, `otp_attempts`, `bank_account_versions`; drop the duplicate `students.status`), §15, §16, §17, §18, §19 (new tests below).

**Tests added to §19:** account recovery, enrolment flow, partial-approval and appeal timers, one-advance limit and re-block, recurring confirmation, crypto-shredding erasure, notification delivery, load test at fee-season peak, accessibility (WCAG 2.1 AA), penetration test, usability test (**claim submitted in under 3 minutes on a ₹8,000 Android phone over throttled 3G**).

## 13. Still open (needs people, not design)

1. Names of the owner(s), sponsors and the grievance officer
2. Income bands and priority rule (1.10)
3. Regional languages and the phase they arrive in
4. 80G / 12A / FCRA (CA to confirm)
5. Which institutions are registered payees
6. Final confirmation of all **[default]** values after the cost survey
