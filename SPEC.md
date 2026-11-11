# ImpactProof SPEC (single source of truth)

## Product
ImpactProof is a web app for corporate CSR teams. It links every CSR project claim to its evidence, shows how much of the claim is backed, and records a human decision with a reason. Tagline: "Every CSR claim, traced to its evidence."
Chain: Project -> Milestone (with Evidence Contract) -> Claim -> Evidence -> Rule checks -> Human decision -> Report.

## Hard rules (never break)
1. Humans decide. The system only flags. Never say "fraud", "fake", "verified impact" or "AI verifies". Use "evidence gap", "not yet backed by evidence", "flagged for review".
2. "Linked" does not mean authentic. "Reviewed" does not mean impact is guaranteed.
3. All seeded data is fictional. Show a visible DEMO DATA badge when a project has is_demo = true.
4. Never invent statistics. Where a metric has no data, show "Not yet measured".
5. Never use the service_role key. Use only NEXT_PUBLIC_SUPABASE_URL and NEXT_PUBLIC_SUPABASE_ANON_KEY.
6. Product name is ImpactProof everywhere. Never write ImpactLedger.
7. Every page needs loading, empty and error states. No blank screens, no infinite spinners.
8. Every status uses a text label as well as a colour.

## Tech
Next.js (App Router) + TypeScript + Tailwind CSS + lucide-react icons. Supabase (@supabase/supabase-js and @supabase/ssr) for auth, database and storage. Deploy on Vercel. No other backend. No extra UI frameworks unless needed.

## Database (already created in Supabase, do NOT change it)
Tables: profiles(id, full_name, role in manager|reviewer|ngo, org_name), projects(id, name, partner_org, region, theme, budget_inr, start_date, end_date, status, is_demo), milestones(id, project_id, title, unit, target, due_date, tranche_inr), requirements(id, milestone_id, label, evidence_type, required) = the Evidence Contract, claims(id, milestone_id, reported_value, period, note, status in reported|in_review|needs_info|accepted|returned|escalated, submitted_by, submitted_by_name, submitted_at), evidence(id, claim_id, requirement_id, file_name, storage_path, evidence_type, covers_units, grade A|B|C|D, source_date, uploaded_by, uploaded_at), reviews(id, claim_id, reviewer_id, reviewer_name, decision in accepted|needs_info|returned|escalated, rationale, created_at), audit_log(id, actor_id, actor_name, action, entity, entity_id, detail, created_at).
Storage bucket: evidence (private). Use signed URLs to open files.
The audit_log is filled automatically by database triggers when claims, evidence or reviews are inserted. The app must NOT insert into audit_log.
evidence_type values: attendance_sheet, photo, invoice, site_visit_report, third_party_report, self_report.

## Evidence grade (computed by the app when evidence is saved, never typed by the user)
A = site_visit_report, third_party_report. B = photo. C = attendance_sheet, invoice. D = self_report.
Labels: A "Independent or site-checked", B "Photo evidence", C "Organisation document", D "Self-reported".

## Gap Detector (put in lib/gap.ts as a pure, tested function)
Input: claim.reported_value, list of its evidence, the milestone's requirements.
- backed = min(reported_value, sum of covers_units of its evidence)
- gap = reported_value - backed
- coverage_percent = round(backed / reported_value * 100) (0 if reported_value is 0)
Flags (each has rule code, plain-language message, and the values compared):
- R1 "Missing required evidence": for every requirement with required = true that has no evidence row with that requirement_id. Message: "Required proof missing: <label>".
- R2 "Claim not fully backed": if gap > 0. Message: "<backed> of <reported> <unit> are backed by evidence. <gap> not yet backed."
- R3 "Only self-reported evidence": if the claim has evidence and every item is grade D.
- R4 "No evidence linked": if the claim has zero evidence.
A flag is NOT a verdict. Always show this line under flags: "Flags show evidence gaps for a human to review. They are not findings of wrongdoing."

## Roles and what they see
- manager: everything. Can add requirements (Evidence Contract items) to a milestone, submit claims, record decisions.
- reviewer: Dashboard, Projects (read), Review queue, Claim page, can record decisions. Cannot submit claims or upload evidence.
- ngo: Dashboard, Projects (read), can submit claims and upload evidence for claims. Cannot record decisions.
Hide buttons the role cannot use. The database also enforces this.

## Decisions
Options and meaning: accepted = "Accept with reason", needs_info = "Request more information", returned = "Return for correction", escalated = "Escalate". A written reason is required (cannot be empty). Saving a decision inserts a reviews row AND updates claims.status to the same value. After saving, show a confirmation and the new status.
A claim that has just been submitted has status reported. It appears in the Review queue as "Awaiting review". When the first evidence is added to a reported claim, nothing changes automatically; only decisions change status.

## Pages
/ (landing), /login, /dashboard, /projects, /projects/[id], /claims/[id], /review, /reports, /audit. Everything except / and /login requires a signed-in user. Unsigned users are redirected to /login. After login go to /dashboard.

## Wording
Status chip text: reported = "Reported by partner", in_review = "In review", needs_info = "Needs information", accepted = "Accepted", returned = "Returned", escalated = "Escalated".
Trust notes to show in the UI: "Reported by partner means submitted information, not independently verified fact." "Evidence linked does not mean the record is authentic."

## Design system
Font: Inter. Colours: navy #132A3A (sidebar, headings), teal #0F766E (primary buttons, links, active nav), page background #F5F8FA, surface #FFFFFF, border #CBD5DF, text #243746, muted text #617483, amber #D99A28 (gaps and warnings), red #B54747 (blocking errors only), green #26735B (accepted and backed). Corners 10px. Cards have 1px border and a very light shadow. Spacing multiples of 4px. Left sidebar 240px (navy, white text, teal active item), top bar 64px with page title, DEMO badge, user name, role chip and sign-out. Tables have a sticky header, compact rows and hover state. Layout works on mobile (sidebar becomes a menu button). No gradients on data screens, no stock photos. Use lucide-react outline icons next to text labels.