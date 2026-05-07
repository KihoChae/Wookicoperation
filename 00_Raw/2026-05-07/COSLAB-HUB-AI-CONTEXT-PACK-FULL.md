# COSLAB Hub — AI Handoff Context Pack (FULL, NO SUMMARY)

> Purpose: This file is designed to be pasted or uploaded into another AI system as source context.
> Rule: The original markdown contents below are included in full, without summarization or intentional omission.
> How to use: Provide this entire file to the AI before asking it to generate implementation plans, code, database migrations, or documentation for COSLAB Hub.
> Important: Each source file is wrapped with clear BEGIN/END boundaries. Text inside each boundary is the original markdown content.

## Source File Manifest

| # | File | Bytes | Lines | SHA-256 |
|---:|---|---:|---:|---|
| 1 | `COSLAB-HUB-API.md` | 30021 | 873 | `e95ed9e18abc7915621099c47b12eff6e3aedefd1ccbb0d1668ac73fa3060cca` |
| 2 | `COSLAB-HUB-CONVENTIONS.md` | 22069 | 752 | `4ecb3a3fc3324c4ba3fbdb55b3659b3062e315ebc7ff9d6e5318a11105e43912` |
| 3 | `COSLAB-HUB-DESIGN.md` | 36406 | 783 | `7884e035bf4dcc38780957b0b2b6611ae78b0c1f85e092cec338bbe5afb7bc7a` |
| 4 | `COSLAB-Hub-PM-Module-기획서-v4.md` | 17831 | 359 | `b60eac22cdb3cbe25df77951da5a9a724164e392a751f2a829550ff853382c08` |
| 5 | `COSLAB-HUB-SCHEMA.md` | 81637 | 2429 | `bf5f244c4ba61899e659f1defd3b77425c35a5c57ff5f15473efcf19941561e4` |
| 6 | `COSLAB-HUB-STATUS.md` | 16947 | 420 | `0a289b2e9b8708ca3db958022d422880342741ba666058a4d99a7791e01ab957` |

---



<!-- SOURCE_FILE_1_BEGIN: COSLAB-HUB-API.md -->

# SOURCE FILE 1: `COSLAB-HUB-API.md`

# COSLAB Hub — API Endpoint Reference

> 233 route.ts 파일을 분석한 endpoint 카탈로그. AI 에이전트가 본 코드베이스에서 작업할 때 참고용.
>
> 생성: 2026-05-07 기준
>
> 표기:
> - **Auth**: `public` / `authenticated` / `editor+` / `admin` / `cron(CRON_SECRET)`
> - **Side effects**: snapshot 생성, audit_log 기입, alert 발생 등 명시
> - **Cross-project**: `supabasePV` (Package Validator), `supabaseSafety` (Safety) 별도 DB

---

## /api/ai/...

### POST /api/ai/smart-advisor
- Auth: public
- Purpose: 회사 데이터(materials, formulations, stability, onboarding) 컨텍스트 기반 자연어 Q&A
- Body: `{question, history?}`
- Response: `{answer}`

### POST /api/ai/formulator
- Auth: public
- Purpose: 자연어 설명에서 풀 처방 생성
- Body: `{description?, category?, claims?, target_price?, batch_size?, constraints?}`
- Response: `{formula, ingredients, phases, cost_estimate}`

### POST /api/ai/doc-classify
- Auth: public
- Body: `{document_text, category_hint?}` → `{classification, confidence}`

### POST /api/ai/formula-recommend
- Auth: public
- Body: `{category, skin_type?, concerns?, texture_preference?}` → `{recommendations}`

### POST /api/ai/inci-match
- Auth: public
- Purpose: INCI 명을 인벤토리 원료에 매칭
- Body: `{inci_names: string[]}` → `{matches: [{inci, material_id, match_score}]}`

### POST /api/ai/spec-suggest
- Auth: public
- Body: `{formulation_id, category?, product_type?}` → `{recommended_specs}`

### POST /api/ai/stability-assessment
- Auth: public
- Body: `{formulation_id?, ingredients?, category?}` → `{assessment_id, status, recommendations}`

### POST /api/ai/test-recommend
- Body: `{formulation_id?, product_category?, target_duration?}` → `{recommended_tests}`

### POST /api/ai/clean-beauty
- Body: `{formulation_id?, ingredients?}` → `{compliant, issues, suggestions}`

---

## /api/alerts

### GET /api/alerts
- Auth: public
- Query: `status(pending|resolved|dismissed|all)`, `target_entity`, `target_id`, `limit`, `mode(counts)`
- Response: `{alerts}` 또는 `{counts}`

### POST /api/alerts/[id]
- Auth: authenticated
- Purpose: alert 해결/액션 + formula_snapshot 트리거
- Body: `{action, reason?}` → `{success, snapshot_id?}`
- **Side effects**: formula_snapshot 생성 (`reason=manual`)

### DELETE /api/alerts/[id]
- Auth: authenticated → 알람 dismiss

---

## /api/audit-trail

### GET /api/audit-trail
- Auth: authenticated
- Query: `page, limit, event_type, resource_type, user_email, date_from, date_to, search`
- Response: `{logs, total, page, limit}`

---

## /api/auth/...

### GET /api/auth/me
- Auth: authenticated
- Response: `{email, role, displayName, mustChangePassword, totpEnabled, onboardingCompleted}`

### POST /api/auth/hub-login
- Auth: public
- Body: `{email, password, totpToken?}` → `{success, session, mustSetup2FA?}` 또는 `{requires_2fa: true}`
- **Side effects**: audit_log (login_failure, account_lock, success)

### POST /api/auth/hub-logout
- Auth: authenticated → `{success}`

### POST /api/auth/setup
- Auth: 첫 setup 또는 admin
- Body: `{email, password, displayName?, role?}` → `{success, user_id}`

### POST /api/auth/change-password
- Auth: authenticated
- Body: `{oldPassword, newPassword}` → `{success, message}`

### POST /api/auth/2fa/setup
- Auth: authenticated → `{secret, qr_code_url}`

### POST /api/auth/2fa/verify
- Body: `{token}` → `{verified}`

### POST /api/auth/2fa/disable
- Body: `{password}` → `{success}`

### GET /api/auth/anon-key
- Auth: public → `{anonKey}`

### GET /api/auth/users
- Auth: admin → `{users}`

### POST /api/auth/users
- Auth: admin
- Body: `{email, password, displayName, role}` → `{success, id}`

### PUT /api/auth/users/[id]
- Auth: admin
- Body: `{email?, role?, locked?}`

### DELETE /api/auth/users/[id]
- Auth: admin → soft-delete

---

## /api/backup

### GET /api/backup/auto
- Auth: cron (`CRON_SECRET` header)
- Purpose: 일일 자동 백업 (코어 테이블 → Supabase Storage)
- Response: `{success, checksum, file_path, retention_applied}`
- **Side effects**: `backups/auto/YYYY-MM-DD.json` 생성, 오래된 파일 회전

### POST /api/backup
- Auth: admin → 수동 백업

---

## /api/base-formulas

### GET /api/base-formulas
- Auth: public
- Response: `{data: [{id, name, proposal_count, accepted_count, product_count, ...}]}`

### POST /api/base-formulas
- Auth: authenticated
- Body: `{name, product_type?, description?, classification?, specs?}` → `{success, id}`

### GET /api/base-formulas/categories → `{categories}`

### POST /api/base-formulas/[id]/duplicate
- Body: `{new_name}` → `{success, new_id}`

### POST /api/base-formulas/[id]/new-version
- Body: `{change_reason, ingredients?, phases?}` → `{success, version_number}`

### GET /api/base-formulas/[id] → `{formula, versions}`

---

## /api/chat

### POST /api/chat
- Auth: public (rate-limit 10/min per IP)
- Body: `{message, history?}` → `{response}`

---

## /api/company-docs

### GET /api/company-docs → `{documents, brands}`
### POST /api/company-docs (editor+) - Body: `{doc_type, file, title?, tags?}`
### GET /api/company-docs/brands → `{brands}`

---

## /api/dashboard

### GET /api/dashboard/stats
- Response: `{knowledge_count, formulation_counts, material_count, registration_counts, bom_count, stability_count, lc_projects_stats, recent_activity}`

### GET /api/dashboard/upcoming-tests
- Auth: authenticated
- Purpose: 30일 이내 stability test
- Response: `{upcoming_tests: [{test_code, formulation, due_date, ...}]}`

---

## /api/executive

### GET /api/executive
- Auth: authenticated
- Response: `{active_projects, pending_registrations, overdue_tests, team_capacity, cost_trends}`

---

## /api/ingredients

### GET /api/ingredients
- Query: `search?, category?, limit?` → `{ingredients}`

---

## /api/knowledge

### GET /api/knowledge
- Query: `type?, status?, search?, limit?` → `{entries}`

### POST /api/knowledge (editor+)
- Body: `{title, knowledge_type, summary, data?, tags?, status?}` → `{success, id}`

### GET/PUT/DELETE /api/knowledge/[id]

### POST /api/knowledge/photo (editor+)
- Body: FormData(file) → `{success, url, id}`

### POST /api/knowledge/transcribe (editor+)
- Body: FormData(audio_file) → `{success, transcript, entry_id}`

---

## /api/label-check (Cross-project: supabasePV)

### GET /api/label-check
- Response: `{projects, stats}`

### POST /api/label-check/create
- Body: `{project_code, project_name, brand_name, product_name, packaging_type?, category?}` → `{success, project_id}`

### GET /api/label-check/[id] → `{project, reviews}`

### POST /api/label-check/[id]/action
- Body: `{action, feedback?, approver_notes?}` (draft→reviewing→reviewed→approved)

### POST /api/label-check/[id]/upload
- Body: FormData(file, doc_type) → `{success, file_id, url}`

### GET /api/label-check/[id]/upload-url → `{upload_url, file_id}`

### POST /api/label-check/[id]/validate → `{issues, recommendations}`

### GET /api/label-check/[id]/status → `{status, completion_pct, last_review_date}`

### GET /api/label-check/[id]/report → `{report_html, passed, flags}`

### POST /api/label-check/calibration (editor+) - OCR 트레이닝

### GET /api/label-check/erp-formulations → `{formulations}` (PV에서)

### POST /api/label-check/classify - AI 문서 분류
### POST /api/label-check/ai-classify - AI 보조 분류

### GET /api/label-check/regulations → `{regulations}`

### GET /api/label-check/review-items
- Query: `project_id?, status?` → `{items}`

### POST /api/label-check/review-items/[itemId]/feedback
- Body: `{feedback, status, priority?}`

---

## /api/locks

### GET /api/locks
- Purpose: 폼 편집용 pessimistic lock 일람
- Response: `{locks: [{id, resource_id, user_email, acquired_at, ttl_seconds}]}`

---

## /api/onboarding

### POST /api/onboarding/complete - authenticated → `{success}`

---

## /api/packaging-materials

### GET /api/packaging-materials
- Query: `active_only?, category?, search?, limit?` → `{materials}`

### POST /api/packaging-materials (editor+)
- Body: `{name, category, supplier, specifications?, cost?, min_order?}`

### GET /api/packaging-materials/[id] → `{material, documents}`
### PUT /api/packaging-materials/[id] (editor+)
### POST /api/packaging-materials/[id]/documents (editor+) - FormData(file, doc_type)

---

## /api/parse-image

### POST /api/parse-image
- Purpose: OCR (label-check용)
- Body: FormData(image) → `{text, confidence, bbox_data}`

---

## /api/production

### GET /api/production
- Response: `{active_batches, pending_approvals, yield_metrics, equipment_status}`

---

## /api/products / /api/products-v2

### GET /api/products → `{products}` (legacy)
### POST /api/products - Body: `{name, formulation_id?, category?, product_code?}`

### GET /api/products-v2
- Query: `include_discontinued?, trash?` → `{products}`

### POST /api/products-v2 - Body: `{name, formulation_id?, product_code?, category?, packaging_size?}`
- **Side effects**: audit_log 엔트리

### GET /api/products-v2/[id] → `{product, cost_per_unit, supply_price, registration_status}`
### PUT /api/products-v2/[id] (editor+)
### POST /api/products-v2/[id]/references → `{references: [{type, id, name}]}`

---

## /api/projects

### GET /api/projects → `{projects}`
### POST /api/projects - Body: `{name, description?, client?, status?}`
### GET /api/projects/[id] → `{project, tasks}`
### PUT /api/projects/[id] (editor+)

---

## /api/proposals

### GET /api/proposals
- Query: `status?, client?, formula?` → `{proposals}`

### POST /api/proposals
- Body: `{base_formula_id, client_id, brand_id, comments?}` → `{success, id}`
- **Side effects**: audit_log

---

## /api/quotes

### GET /api/quotes (authenticated)
- Query: `customer_id?, status?, limit?` → `{quotations}`

### POST /api/quotes/create
- Body: `{customer_id, items: [{product_id, qty, unit_price?}], valid_until?}` → `{success, quote_id, quote_number}`
- **Side effects**: qq_quotations + qq_items 삽입

### GET /api/quotes/[id] → `{quotation, items}`
### PUT /api/quotes/[id] (editor+)
### POST /api/quotes/[id]/accept → `{success, order_id?}`

### GET /api/quotes/customers → `{customers}`
### GET /api/quotes/master-data → `{products, pricing_tiers, tax_rates}`

---

## /api/reg-dashboard

### GET /api/reg-dashboard
- Response: `{folders_by_status, recent_submissions, pending_approvals, certification_expiries_soon}`

---

## /api/reg-materials

### GET /api/reg-materials
- Query: `category?, status?, search?` → `{materials}`

### POST /api/reg-materials (editor+)
- Body: `{name, material_type, classification, source?, supplier?}`

### GET /api/reg-materials/[id] → `{material, documents, classifications}`
### POST /api/reg-materials/[id]/documents (editor+)
### DELETE /api/reg-materials/[id]/documents/[docId] (editor+)

### POST /api/reg-materials/classify - 수동 분류
### POST /api/reg-materials/ai-classify - AI 자동 분류

### POST /api/reg-materials/bulk-upload (editor+) - FormData(csv_file)
### GET /api/reg-materials/download → File (xlsx)

### GET /api/reg-materials/matrix → `{matrix, lastUpdated}`
### POST /api/reg-materials/smart-match - Body: `{inci_name, concentration?, jurisdiction?}`

### GET /api/reg-materials/halal-matrix → `{matrix, certifiers, update_date}`
### POST /api/reg-materials/halal-matrix/export → File (xlsx)

---

## /api/reg-packaging

### GET /api/reg-packaging/halal-matrix → `{matrix, materials}`
### POST /api/reg-packaging/halal-matrix/export → File (xlsx)

---

## /api/registration-folders

### GET /api/registration-folders
- Query: `type(BPOM|HALAL), include_discontinued?` → `{folders}`

### POST /api/registration-folders
- Body: `{product_id, reg_type, started_at?}` → `{data: {id, status, started_at}}`

### GET /api/registration-folders/[id] → `{folder, documents, submissions, status, certificates}`

### PUT /api/registration-folders/[id] (editor+)
- Body: `{stage_dates?, notes?, certification_number?}`

### DELETE /api/registration-folders/[id] (editor+) - soft-delete

### POST /api/registration-folders/[id]/generate
- Body: `{doc_category}` → `{success, file_path, doc_id}`
- **Side effects**: registration_documents 삽입

### POST /api/registration-folders/[id]/generate-flowchart - PDF 생성
### POST /api/registration-folders/[id]/generate-halal-matrix - xlsx 생성

### POST /api/registration-folders/[id]/assemble - 제출용 폴더 패키징
### POST /api/registration-folders/[id]/reassemble - 재조립

### POST /api/registration-folders/[id]/submit
- Body: `{submission_label?, notes?}` → `{success, submission_id, submission_number}`
- **Side effects**: registration_submissions 생성, 이전 'submitted' supersede

### GET /api/registration-folders/[id]/submissions → `{submissions}`
### GET /api/registration-folders/[id]/submissions/[subId]/download → File

### POST /api/registration-folders/[id]/approve (admin)
- Body: `{approver_notes?}`

### POST /api/registration-folders/[id]/acknowledge-drift
- Body: `{drift_details?}`
- **Side effects**: drift_acknowledged_pv_id 기록

### POST /api/registration-folders/[id]/resync → `{success, drifts_detected}`

### GET /api/registration-folders/[id]/certificates → `{certificates}`
### POST /api/registration-folders/[id]/certificate (editor+)
- Body: `{certificate_type, certificate_number, issue_date, expiry_date?, file?}`

### GET /api/registration-folders/[id]/download → File

### POST /api/registration-folders/reset (admin)
- Body: `{folder_id, reason?}` - 비상 리셋

### POST /api/registration-folders/upload (editor+) - 벌크 업로드
### POST /api/registration-folders/upload-doc (editor+) - 단일 업로드

---

## /api/safety (Cross-project: supabaseSafety)

### GET /api/safety
- Query: `status?, formulation_id?, limit?` → `{jobs}`

### POST /api/safety/create
- Body: `{product_name, product_category, ingredients: [{inci_name, concentration}], meta_data?}` → `{job_id, status}`
- **Side effects**: sa_jobs 삽입 (Safety Supabase), 파이프라인 비동기 실행

### GET /api/safety/[jobId] → `{job, results, escalations}`
### GET /api/safety/[jobId]/report/download → File (pdf)

### POST /api/safety/cleanup (cron) - 오래된 job 삭제

### GET /api/safety/cache → `{cache, last_updated}`
### POST /api/safety/cache/research (editor+) - fresh research

### POST /api/safety/escalations
- Body: `{job_id, issue, severity, notes?}` → `{success, escalation_id}`

---

## /api/search

### GET /api/search (q ≥ 2)
- Response: `{results: {formulations, materials, products, ingredients, packaging, knowledge, proposals, label_check, quotes, inci, sample_materials, suppliers, clients}, total}`

### POST /api/search/ai - 시맨틱 검색
- Body: `{query, category?, filters?}` → `{results, total}`

---

## /api/status

### GET /api/status → `{status, database, storage, message_broker, version}`

---

## /api/talent/...

### People

#### GET /api/talent/people
- Query: `include_discontinued?, trash?` → `{people: [{id, full_name, email, role, current_assignments, ...}]}`

#### POST /api/talent/people (editor+)
- Body: `{id?, full_name, role, department, email, join_date?, auto_ojt?}`
- Response: `{success, id, auto_program_id?, auto_message?}`
- **Side effects**: role 매칭 시 OJT program 자동 생성

#### DELETE /api/talent/people (editor+)
- Query: `id, purge?` → `{success, soft_deleted | purged}`

#### GET /api/talent/people/[id]/training-detail
- Response: `{person, programs, sessions, evaluations, quizzes, external_trainings}`

### Me (current user)

#### GET /api/talent/me → `{user, person, assignments, role_permissions}`
#### POST /api/talent/me - Body: `{display_name?, phone?, avatar?}`

#### GET /api/talent/me/action-items
- Response: `{retests_pending, tests_to_take, self_reports_due, evaluations_due, quizzes_to_grade, sessions_to_approve, calendar_sessions}`

#### POST /api/talent/me/self-evaluation
- Body: `{session_id, performance_rating, self_assessment, comments?}`

### Sessions

#### GET /api/talent/sessions
- Query: `id?, program_id?, curriculum_id?, from?, to?, include_discontinued?, trash?`

#### POST /api/talent/sessions (editor+)
- Body: `{program_id, curriculum_id?, date, session_no?, title?, description?, trainer_id?, location?}`

#### GET /api/talent/sessions/[id]/approval-summary
- Response: `{session, participants: [{person_id, attendance, self_report_score, trainer_score, quiz_scores}]}`

#### POST /api/talent/sessions/[id]/transition (editor+)
- Body: `{next_state, approver_notes?}` (Scheduled→Done→Evaluated→Approved/Closed)

#### PUT /api/talent/sessions/[id]/role (editor+)
- Body: `{person_id, role}` (trainer/participant)

### Programs / Curricula

#### GET /api/talent/programs - Query: `status?, trainer_id?, trainee_id?`
#### POST /api/talent/programs (editor+)
- Body: `{curriculum_id?, name?, description?, trainee_ids: string[], sessions_data?}` → `{success, program_id, sessions_created}`
- **Side effects**: training_programs + training_sessions 생성

#### GET /api/talent/programs/[id]/session-tracking
- Response: `{program, sessions: [{id, date, status, participants: [{person_id, attendance, scores}]}]}`

#### GET /api/talent/programs/[id]/timeline-export → File (xlsx)

#### GET /api/talent/curricula - Query: `status?, for_role?`
#### POST /api/talent/curricula (editor+)
- Body: `{title, description, default_for_roles: string[], sessions_template}`

### Quizzes

#### GET /api/talent/quizzes - Query: `status?, curriculum_id?`
#### POST /api/talent/quizzes (editor+)
- Body: `{title, description, questions: [{text, type, options?, correct_answer?}], passing_score?, retake_allowed?}`

#### POST /api/talent/quizzes/import (editor+) - FormData(file) 벌크 임포트
#### GET /api/talent/quizzes/[id] → `{quiz, questions}`

#### POST /api/talent/quizzes/[id]/respond
- Body: `{answers: [{question_id, answer_text}]}` → `{success, submission_id, score?, message?}`

#### GET /api/talent/quizzes/[id]/my-response → 본인 응답
#### GET /api/talent/quizzes/[id]/responses (editor+) - Query: `status?(submitted|graded)`

#### POST /api/talent/quizzes/responses/[id]/grade (trainer)
- Body: `{score, feedback?, grader_id}`

#### GET /api/talent/quizzes/responses/[id]/detail
- Response: `{response, answers, grading_rubric}`

#### POST /api/talent/quizzes/responses/[id]/request-retest
- Body: `{reason?}` → `{success, request_id}`

### Evaluations

#### GET /api/talent/evaluations - Query: `evaluatee_id?, evaluator_id?, session_id?`
#### POST /api/talent/evaluations (trainer)
- Body: `{session_id, person_id, scores: {...}, comments?, sign_off?}` → `{success, evaluation_id}`
- **Side effects**: audit_log

#### GET /api/talent/evaluations/[id]/export → File (pdf)
#### POST /api/talent/evaluations/self - 자가평가 (단축)

#### GET /api/talent/ai/draft-evaluation - Query: `session_id` → `{draft}`
#### POST /api/talent/ai/draft-report - Query: `person_id` → `{report_html}`

### Attendance / Assignments / Org

#### GET /api/talent/attendance → `{attendance}`
#### POST /api/talent/attendance (editor+) - Body: `{session_id, person_id, status}`

#### GET /api/talent/assignments - Query: `person_id?, position_id?`
#### POST /api/talent/assignments (editor+)
- Body: `{person_id, position_id, start_date, end_date?, allocation_pct?, is_primary?}`

#### GET /api/talent/teams → `{teams}`
#### POST /api/talent/teams (editor+) - Body: `{name, description?, manager_id?}`

#### GET /api/talent/positions → `{positions}`
#### POST /api/talent/positions (editor+) - Body: `{title, team_id, description?, level?}`

#### GET /api/talent/roster - 전체 사람+배정+팀
#### GET /api/talent/progress-roster - 모든 trainee 진척

### Materials & External

#### GET /api/talent/materials → `{materials}`
#### POST /api/talent/materials (editor+) - FormData(file, title, description?, tags?)
#### POST /api/talent/materials/signed-upload (editor+) - presigned URL

#### GET /api/talent/external-trainings → `{trainings}`
#### POST /api/talent/external-trainings (editor+)
- Body: `{person_id, course_name, provider, completion_date, certificate_url?, cost?}`

#### GET/PUT /api/talent/external-trainings/[id]
#### POST /api/talent/external-trainings/signed-upload (editor+)

#### GET /api/talent/reports - Query: `program_id?, date_from?, date_to?`
#### GET /api/talent/reports/[sessionId]/export → File (pdf)

#### GET /api/talent/blank-forms/[type] → File (xlsx/pdf)
#### GET /api/talent/templates → `{templates}`

---

## /api/unified/...

### Formulations

#### GET /api/unified/formulations
- Query: `include_discontinued?, trash?` → `{formulations}`

#### POST /api/unified/formulations
- Body: `{product_name, product_code, formula_code?, category?, ingredients?, phases?, client_name?, brand?, batch_size_g?, density?, base_formula_id?}`
- Response: `{formulation, sample_id?, product_id?, siblings?}`
- **Side effects**: formulations + sample_manufacturing + products 자동 생성, audit_log

#### GET /api/unified/formulations/[id] → `{formulation, versions, current_ingredients, current_snapshot}`

#### POST /api/unified/formulations/[id]/modify
- Purpose: formula code 자릿수 증가 (SR101EAB1→SR101EAB2)
- Response: `{success, new_id, new_code}`
- **Side effects**: 새 formulation (소스 복사)

#### POST /api/unified/formulations/[id]/renew
- Purpose: 현재 deactivate + 같은 renewal_group 새 active 생성
- Response: `{success, new_id, renewal_group}`
- **Side effects**: formula_snapshots (outgoing+new), product_versions bump, audit_log

#### POST /api/unified/formulations/[id]/confirm
- Body: `{confirmation_notes?}`
- **Side effects**: formula_snapshots 생성, confirmed 마크

#### POST /api/unified/formulations/[id]/unlock (admin)
- Body: `{reason}` - confirmed → draft 되돌림

#### POST /api/unified/formulations/[id]/duplicate
- Body: `{new_product_code?, new_product_name?}` → `{success, new_id}`

#### POST /api/unified/formulations/[id]/hard-delete (admin)
- Query: `confirm_deleted=1`

#### POST /api/unified/formulations/[id]/cost-impact
- Body: `{ingredient_id, new_material_id, new_concentration?}` → `{current_cost, simulated_cost, impact_pct}`

#### PUT /api/unified/formulations/[id] (editor+)

#### GET /api/unified/formulations/[id]/specs → `{specs}`
#### POST /api/unified/formulations/[id]/specs (editor+)
- Body: `{quality_tests: [{test_id, min_value?, max_value?, method?, acceptance_criteria?}]}`

#### GET /api/unified/formulations/generate-code - Query: `prefix?, brand?` → `{code}`

#### GET /api/unified/formulations-summary - 가벼운 리스트
#### GET /api/unified/formulations-summary/[id] → `{formulation, versions}`
#### GET /api/unified/formulations-summary/[id]/versions → `{versions}`

### Materials

#### GET /api/unified/materials - Query: `active_only?, supplier?, category?, search?, limit?`
#### POST /api/unified/materials (editor+) - audit_log
#### GET /api/unified/materials/[id] → `{material, documents, used_in_formulas}`
#### PUT /api/unified/materials/[id] (editor+)
#### DELETE /api/unified/materials/[id] (editor+) - soft-delete
#### POST /api/unified/materials/[id]/documents (editor+) - FormData

### Samples

#### GET /api/unified/samples - Query: `formulation_id?, include_discontinued?, trash?`
#### POST /api/unified/samples
- Body: `{formulation_id, sample_code, sample_name, client?, manufacturing_date?, manufacturing_method?, equipment_used?, phase_records?}`

#### GET /api/unified/samples/[id] → `{sample, phase_records, test_results}`
#### PUT /api/unified/samples/[id] (editor+)
#### POST /api/unified/samples/[id]/sheet → File (pdf)
#### POST /api/unified/samples/[id]/client-pack → File (zip)
#### POST /api/unified/samples/[id]/confirm-by-client - public(token) - Body: `{feedback?, rating?}`

### Stability

#### GET /api/unified/stability - Query: `formulation_id?, include_discontinued?, trash?`
#### POST /api/unified/stability
- Body: `{formulation_id, test_code, category, test_duration_months, test_conditions}`

#### GET /api/unified/stability/[id] → `{test, conditions, results, reports}`
#### PUT /api/unified/stability/[id] (editor+)

#### POST /api/unified/stability/[id]/conditions (editor+)
- Body: `{conditions: [{temp_c, rh_pct, packaging?, light_exposure?, monitoring_schedule}]}`

#### POST /api/unified/stability/[id]/results (editor+)
- Body: `{results: [{timepoint, condition_id, parameters, observations?, status?}]}`

#### POST /api/unified/stability/[id]/confirmation (editor+)
- Body: `{observations?, recommendations?, signature?}`

#### POST /api/unified/stability/[id]/final-confirmation (admin)
- Body: `{approved, certificate_number?, validity_months?}` → `{success, certificate_id?}`

#### POST /api/unified/stability/[id]/researcher-report
- Body: `{report_html, conclusions?, recommendations?}`

#### GET /api/unified/stability/templates → `{templates}`
#### GET /api/unified/stability/templates/[id] → `{template, conditions_default}`
#### POST /api/unified/stability/templates (editor+) - Body: `{name, category, description, conditions_template}`

#### GET /api/unified/stability/parameters → `{parameters}`

### BOMs

#### GET /api/unified/boms - Query: `formulation_id?, include_discontinued?, trash?`
#### POST /api/unified/boms
- Body: `{formulation_id, formulation_version, total_cost?, material_costs: [{material_id, quantity_g, unit_cost}]}`
- **Side effects**: 비용 변경 시 qq_snapshots 생성, audit_log

#### GET /api/unified/boms/[id] → `{bom, materials, total_cost, cost_per_unit}`
#### PUT /api/unified/boms/[id] (editor+)

### Clients / Suppliers / Packaging

#### GET/POST /api/unified/clients
#### GET /api/unified/clients/[id] → `{client, formulations, samples, orders}`
#### PUT /api/unified/clients/[id] (editor+)

#### GET/POST /api/unified/suppliers
#### GET /api/unified/suppliers/[id] → `{supplier, materials}`
#### GET /api/unified/suppliers/[id]/materials → `{materials}`
#### PUT /api/unified/suppliers/[id] (editor+)

#### GET/POST /api/unified/packaging
#### GET /api/unified/packaging/[id] → `{packaging, documents, used_in_formulations}`
#### POST /api/unified/packaging/[id]/documents (editor+)

### Documents / Cost / Inventory

#### GET /api/unified/documents - Query: `type?, resource_type?, search?`
#### POST /api/unified/documents (editor+) - FormData(file, doc_type, resource_type?, resource_id?)
#### GET /api/unified/documents/categories → `{categories}`

#### GET/POST /api/unified/cost-basis (master cost reference)
#### GET /api/unified/inventory - Query: `material_id?, low_stock?`
#### GET /api/unified/inventory/[id] → `{material, quantity_kg, transactions}`
#### GET /api/unified/inventory/packaging → `{packaging}`

#### GET /api/unified/quality-tests → `{tests: [{id, name, method, unit, normal_range?, category}]}`
#### GET /api/unified/process-flow → `{phases, steps, equipment_requirements}`

### Trash

#### GET /api/unified/trash
- Auth: authenticated
- Response: `{formulations, materials, samples, tests, ...}` (모든 soft-deleted)

### Sample Materials

#### GET /api/unified/sample-materials - Query: `search?, supplier?, limit?`
#### POST /api/unified/sample-materials (editor+)
- Body: `{sample_code, name, inci_components, supplier, price_per_kg?, appearance?}`

#### GET /api/unified/sample-materials/matrix → `{matrix}`

---

## /api/upload

### POST /api/upload
- Auth: authenticated
- Body: `{batch_id?, total_files, file_names: string[]}` → `{batch_id, upload_urls}`

### POST /api/upload/[batchId] - 배치 finalize (temp → permanent Storage)

---

## 부록 A. 인증 패턴

### `getApiUser()` from `@/lib/auth/api-auth`
- `hub-session` cookie + JWT 검증
- DB에서 role 조회 (`hub_accounts` 테이블)
- Returns `{sub, email, role, displayName}` 또는 `null`

### 역할 게이트
| 게이트 | 조건 |
|---|---|
| `public` | 인증 불필요 |
| `authenticated` | `getApiUser()` 존재 |
| `editor+` | role ∈ {editor, admin} |
| `admin` | role === admin |
| `cron` | `Authorization: Bearer ${CRON_SECRET}` 헤더 |

### Audit wrapper
- `withAudit(handler, {resource})` 사용 시 POST/PATCH/DELETE 자동 로그
- `auth_audit_logs` 엔트리: user, resource_type, resource_id, event_type, before/after diff

---

## 부록 B. Cross-Project Clients

| 클라이언트 | 사용처 | DB |
|---|---|---|
| `supabaseAdmin` | 전체 (기본) | COSLAB Hub Supabase (`oudhnhshsmawgvktwxtu`) |
| `supabasePV` | `/api/label-check/*` | Package Validator Supabase (별도) |
| `supabaseSafety` | `/api/safety/*` | Safety Assessment Supabase (별도) |

`@/lib/supabase/server.ts`, `@/lib/supabase/package-validator.ts` 참조.

---

## 부록 C. Snapshot 생성 라우트

### formula_snapshots
- POST `/api/unified/formulations/[id]/confirm` — confirmed snapshot
- POST `/api/unified/formulations/[id]/renew` — renewal baseline + new version
- POST `/api/alerts/[id]` — manual resolution snapshot

### qq_snapshots (qq_quotation_items_snapshot)
- POST/PUT `/api/unified/boms/[id]` — 비용 변경 시
- POST `/api/quotes/create` — 견적 발행 시점 동결

### registration_folders_snapshot
- POST `/api/registration-folders/[id]/submit` — drift 감지용 product_version 상태 캡처

---

## 부록 D. Audit Log 발생 라우트

`auth_audit_logs` 발생 라우트:
- POST `/api/unified/formulations` — create
- POST `/api/unified/formulations/[id]/renew` — renew
- POST `/api/unified/formulations/[id]/confirm` — confirm
- POST `/api/unified/materials` — create
- POST `/api/unified/boms` — create
- POST `/api/talent/evaluations` — create
- POST `/api/proposals` — create
- POST `/api/auth/hub-login` — login success/failure
- `withAudit()` wrapper 사용 라우트 모두

---

**총 233 route.ts 파일 분석 완료** (2026-05-07 기준)

<!-- SOURCE_FILE_1_END: COSLAB-HUB-API.md -->

---


<!-- SOURCE_FILE_2_BEGIN: COSLAB-HUB-CONVENTIONS.md -->

# SOURCE FILE 2: `COSLAB-HUB-CONVENTIONS.md`

# COSLAB Hub — 코딩 컨벤션 & 재사용 패턴

> AI 에이전트가 본 코드베이스에서 일관된 코드를 쓰기 위한 패턴 모음.
>
> 모든 예시는 `c:/temp/coslab-hub/src/` 기준.

---

## 1. 시간 / 시간대 (Jakarta TZ)

**원칙**: 표시·"오늘" 비교·캘린더 cursor 모두 `src/lib/time/jakarta.ts` 헬퍼로 통일. DB는 UTC ISO로 저장. 사용자 표시는 자카르타 (UTC+7).

### 헬퍼 시그니처

```ts
// src/lib/time/jakarta.ts
formatJakarta(d?: DateLike, opts?: Intl.DateTimeFormatOptions): string
  // "20/04/2026, 16:34:51"
formatJakartaDate(d?: DateLike): string                      // "2026-04-20"
formatJakartaTime(d?: DateLike, withSeconds?: boolean): string  // "16:34" or "16:34:51"
nowJakartaLabeled(): string                                  // "2026-04-20 16:34 WIB"
todayJakartaYMD(): string                                    // 오늘 YYYY-MM-DD (자카르타)
jakartaMonthAnchor(d?: DateLike): Date                       // 캘린더 cursor (UTC noon mid-month)
makeMonthAnchor(year: number, monthZeroIdx: number): Date    // 명시적 anchor

// src/lib/date.ts (date-fns 기반 — 내부 유틸)
formatDate(date: string | Date): string         // "yyyy.MM.dd"
formatDateTime(date: string | Date): string     // "yyyy.MM.dd HH:mm WIB"
formatRelative(date: string | Date): string     // "2 hours ago"
```

### 금지 패턴

```ts
// ❌ KST 사용자에게 캘린더가 1달 밀림
new Date(year, month, 1)                       // 로컬 자정 → 자카르타 전날 22시

// ❌ UTC 'today'와 자카르타 stored date 비교 → 자정 부근에서 어긋남
new Date().toISOString().slice(0, 10)

// ❌ 자카르타 강제 안 됨
.toLocaleString()
```

### 올바른 사용

```ts
// "오늘" 비교 (필터, 게이팅)
const today = todayJakartaYMD();           // "2026-04-20"
sessions.filter(s => s.date >= today);

// 표시
{formatJakarta(timestamp)}                  // 사용자 보이는 timestamp
{formatJakartaDate(item.created_at)}        // 깔끔한 날짜만
{nowJakartaLabeled()}                       // 보고서 푸터

// 캘린더 cursor (월 이동)
const [cursor, setCursor] = useState(() => jakartaMonthAnchor());
// 안전: cursor.getFullYear() / cursor.getMonth() / formatJakarta(cursor, ...) 일관
setCursor(makeMonthAnchor(year, month - 1));   // 이전 달
setCursor(makeMonthAnchor(year, month + 1));   // 다음 달
```

---

## 2. 인증

### 기본 패턴

`src/lib/auth/api-auth.ts`의 `getApiUser()`가 단일 진입점.

```ts
// 1. hub-session JWT cookie 확인 (label-check 로그인용)
// 2. fallback: Supabase Auth session
// 3. DB에서 role 조회 (hub_accounts.role)
// 4. Returns { sub, email, role, displayName } | null
```

### 모듈 파일

| 파일 | 역할 |
|---|---|
| `api-auth.ts` | API 라우트용 — `getApiUser()`, role 조회 |
| `hub-auth.ts` | label-check 세션 JWT |
| `totp.ts` | 2FA TOTP 생성/검증 |
| `password-policy.ts` | 90일 회전, 12자 이상, 복잡도 |
| `audit-log.ts` | 필드 단위 diff 감사 로그 |
| `pii-access-log.ts` | 민감 데이터 접근 추적 |

### 역할 게이팅 (DB-only — S-03 원칙)

```ts
// ❌ NEVER hardcode email lists
// const ADMIN_EMAILS = ["sunny@..."];

// ✅ DB only
let role: "admin" | "editor" | "viewer" = "viewer";
const { data: account } = await supabaseAdmin
  .from("hub_accounts")
  .select("role")
  .eq("email", user.email.toLowerCase())
  .eq("is_active", true)
  .maybeSingle();
if (account) role = account.role as AppRole;
```

### 컴포넌트 wrapper

```tsx
import { AdminOnly, EditableOnly } from "@/components/shared/AdminOnly";

<AdminOnly>{children}</AdminOnly>           // admin만 표시
<EditableOnly>{children}</EditableOnly>     // viewer 숨김
const canEdit = useCanEdit();               // admin || editor
```

### API 라우트 가드

```ts
import { requireAdmin, requireEditor } from "@/lib/auth/api-auth";

export async function POST(req: NextRequest) {
  const err = await requireAdmin();        // 403 if not admin
  if (err) return err;
  // ...
}

export async function POST(req: NextRequest) {
  const err = await requireEditor();       // 403 if viewer
  if (err) return err;
}
```

---

## 3. 데이터베이스 접근

### 단일 출처

`src/lib/supabase/server.ts`의 `supabaseAdmin` (Service Role) — **모든 API 라우트가 사용**.

```ts
// 싱글톤 Proxy (lazy init, 빌드 시 인스턴스화 안 됨)
export const supabaseAdmin = new Proxy({} as SupabaseClient, {
  get(_target, prop) {
    return (getSupabaseAdmin() as any)[prop];
  },
});

// 사용
const { data } = await supabaseAdmin.from("table").select("*");
```

### 왜 Service Role?

- Service Role 키는 RLS를 우회
- API 라우트가 게이트키퍼: `getApiUser()` 검증 후에만 supabaseAdmin 호출
- RLS 성능 부담 제거

### Cross-project 클라이언트

```ts
// src/lib/supabase/package-validator.ts
// Label Check용 별도 Supabase 프로젝트
export const supabasePV = /* ... */;

// src/lib/safety-pipeline/supabase-adapter.ts
// Safety Assessment용 별도 Supabase
```

### 클라이언트 사이드 쿼리 (드물게)

```ts
// src/lib/supabase/client.ts — anon key 사용 (RLS 적용)
// 가급적 API 라우트 통해 처리
```

---

## 4. 변경통제 / 스냅샷 / 감사

### 스냅샷 생성

```ts
// src/lib/change-control/snapshot.ts
import { createFormulaSnapshot } from "@/lib/change-control/snapshot";

const result = await createFormulaSnapshot(supabaseAdmin, {
  formulation: formulationRecord,
  reason: "confirmed" | "quote_issue" | "dip_publish" | "manual" | "renew",
  userId: user.sub,
  notes: "optional",
});
// → { snapshot_id, locked_columns, frozen_at }
```

### 자동 잠금 (terminal status)

```ts
// src/lib/change-control/auto-lock.ts
import { shouldAutoLock, buildLockPayload } from "@/lib/change-control/auto-lock";

if (shouldAutoLock("formulations", currentStatus, nextStatus, currentlyLocked)) {
  const lock = buildLockPayload(userId, "confirmed");
  // { locked: true, locked_at, locked_by, lock_reason: "confirmed" }
  await supabaseAdmin.from("formulations").update({ ...payload, ...lock }).eq("id", id);
}

// TERMINAL_STATUS:
// formulations: ["Confirmed"]
// boms: ["Confirmed", "Approved"]
// stability_tests: ["completed", "Completed"]
// ...
```

### 가격 영향 알람

```ts
// src/lib/change-control/price-impact.ts
// 원료 단가 변경 시:
// 1. 잠긴 스냅샷이 이 원료를 동결한 모든 Confirmed 처방 찾기
// 2. 각 처방에 대해 change_alert 발행
// 3. 단가가 동결값으로 revert되면 알람 자동 resolve
```

### 감사 로그 형태

```ts
// src/lib/auth/audit-log.ts
import { auditLog, computeDiff } from "@/lib/auth/audit-log";

await auditLog({
  eventType: "data_update",
  userEmail: user.email,
  userId: user.sub,
  ip: req.headers.get("x-forwarded-for"),
  resource: "formulation",
  resourceId: id,
  details: { before, after, /* ... */ },
  changes: computeDiff(oldRecord, newRecord, ["id", "created_at", "updated_at"]),
  // → { field: { from: oldVal, to: newVal } }
  changeReason: body.changeReason,    // 사용자 입력 사유
});
```

### 셀 단위 잠금 (Stability)

```ts
// 측정 셀이 입력되면 자동 잠금
// 수정 시: 모달 "Edit with reason?" → 사유 입력 → audit_log.change_reason에 저장
// 패턴: cell_audit_lock_pattern (memory 참조)
```

---

## 5. Save-First 동시성 해결 (낙관적 동시성)

### 안티패턴

```ts
// ❌ 동시 편집 덮어쓰기
export async function PUT(req: NextRequest) {
  const body = await req.json();
  await supabaseAdmin.from("formulations").update(body).eq("id", body.id);
  // 두 사용자 동시 편집 → 두 번째 쓰기가 첫 번째 덮어씀 (data loss)
}
```

### 올바른 패턴

```ts
// ✅ updated_at 버전 가드
export async function PUT(req: NextRequest) {
  const body = await req.json();   // body에 updated_at 포함

  const { data: current } = await supabaseAdmin
    .from("formulations")
    .select("updated_at, status")
    .eq("id", body.id)
    .single();

  if (current.updated_at !== body.updated_at) {
    return NextResponse.json(
      { error: "Someone else edited this. Please refresh and retry." },
      { status: 409 }
    );
  }

  await supabaseAdmin
    .from("formulations")
    .update({ ...body, updated_at: new Date().toISOString() })
    .eq("id", body.id)
    .eq("updated_at", body.updated_at);   // 버전 가드 (race condition 방지)
}
```

**예시 파일**: `src/app/api/unified/formulations/[id]/route.ts` PUT 핸들러

---

## 6. Soft-Delete (Trash)

### Phase A/B/C/D 패턴

```ts
// Phase A: 트래시 이동 (DELETE)
await supabaseAdmin.from("table").update({ is_active: false }).eq("id", id);

// Phase B: 복구
await supabaseAdmin.from("table").update({ is_active: true }).eq("id", id);

// Phase C: 참조 검사
import { getMaterialReferences } from "@/lib/trash/references";
const refs = await getMaterialReferences(supabaseAdmin, materialId);
// → { total, items: [{type, id, name}], byType }

// Phase D: 하드 삭제 (?purge=1 query param)
if (request.searchParams.get("purge") === "1") {
  if (refs.total > 0) {
    return NextResponse.json({ error: "References exist", refs }, { status: 409 });
  }
  await supabaseAdmin.from("table").delete().eq("id", id);
}
```

### 컬럼 컨벤션

- 대부분: `is_active boolean default true`
- 일부 (라이프사이클 status가 있는 테이블): `status='discontinued'`/`'archived'`/`'inactive'`

### 참조 검사 라이브러리

```ts
// src/lib/trash/references.ts
getMaterialReferences(supabase, materialId)
getFormulationReferences(supabase, formulationId)
// 등 모듈별 함수
// 반환: { total, items: [{type, id, name}], byType }
```

### 통합 `/trash` 페이지

`src/app/trash/page.tsx` — 11개 모듈의 soft-deleted 레코드 aggregator. Bulk restore, bulk purge.

---

## 7. 컴포넌트 Primitives

### `src/components/shared/`

| 컴포넌트 | 용도 |
|---|---|
| `AdminOnly` | admin 외 숨김 |
| `EditableOnly` | viewer 숨김 |
| `useCanEdit()` | hook — admin or editor 반환 |
| `Modal` | 일반 모달 wrapper |
| `TrashView` | soft-deleted 리스트 |
| `useSortFilter()` | 일반 sort+filter hook |
| `SortableHeader` | 클릭 가능한 컬럼 헤더 (↕/▲/▼) |
| `FilterInput` | 컬럼별 텍스트 필터 |
| `EntityDocumentsTab` | 엔티티별 문서 그리드 |
| `RelatedChanges` | 관련 레코드/변경 알람 |
| `SearchableSelect` | 검색 가능 드롭다운 |
| `Toast` | 토스트 알림 |

### `src/components/layout/`

| 컴포넌트 | 용도 |
|---|---|
| `AppShell` | 프레임. `/api/auth/me` 읽고 login/welcome으로 리디렉트 |
| `Sidebar` | role-based nav (minRole 필터링) |
| `Topbar` | 사용자 메뉴, 검색 아이콘 |
| `IdleTimeout` | 15분 미사용 자동 로그아웃 |
| `WelcomeTip` | 첫 실행 onboarding 배너 |
| `HelpModal` | 컨텍스트 도움말 (아이콘 → 모달) |

### Talent 전용 모달

`src/components/talent/`:
- `EvaluationModal` — 평가 폼
- `GradeQuizModal` — 채점 UI
- `QuizTakeModal` — 응시 인터페이스
- `SessionApproveModal` — 출석/승인

### 사용 예시

```tsx
import { EditableOnly } from "@/components/shared/AdminOnly";
import { SortableHeader, FilterInput } from "@/components/shared";
import { useSortFilter } from "@/components/shared/useSortFilter";

export function FormulationList({ items }: { items: Formulation[] }) {
  const { rows, sortKey, sortDir, setSort, filters, setFilter } = useSortFilter(items);

  return (
    <table>
      <thead>
        <tr>
          <SortableHeader columnKey="product_name" sortKey={sortKey} sortDir={sortDir} onSort={setSort}>
            Product
          </SortableHeader>
          <FilterInput value={filters.product_name} onChange={(v) => setFilter("product_name", v)} />
        </tr>
      </thead>
      <tbody>
        {rows.map(r => (
          <tr key={r.id}>
            <td>{r.product_name}</td>
            <td>
              <EditableOnly>
                <button onClick={() => edit(r.id)}>Edit</button>
              </EditableOnly>
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

---

## 8. Export 생성

### DOCX (using `docx` library)

```ts
// src/lib/stability/researcher-report-docx.ts
import { Document, Packer, Paragraph, TextRun, Table, TableCell, AlignmentType } from "docx";

export interface ResearcherReportInput {
  product_name: string;
  product_code: string;
  test_code: string | null;
  signatories: Signatory[];   // { name, title, email, date }
  // ...
}

export async function generateResearcherReportDocx(input: ResearcherReportInput): Promise<Buffer> {
  const doc = new Document({
    sections: [{
      children: [
        h2("Cover"),
        para([txt(input.product_name, { bold: true, size: 28 })]),
        // ...
      ],
    }],
  });
  return await Packer.toBuffer(doc);
}

// 헬퍼
function txt(t: string, opts?: { bold?, color?, size?, italic? }): TextRun
function para(children: TextRun[] | string, align?: AlignValue): Paragraph
function h2(text: string): Paragraph
function headerCell(text: string, width?: number): TableCell
```

### XLSX (using ExcelJS)

```ts
// src/lib/quote/export-quotation.ts
import ExcelJS from "exceljs";
import { saveAs } from "file-saver";

export async function exportQuotation(quote: AnyObj, settings: AnyObj = {}) {
  const wb = new ExcelJS.Workbook();
  wb.creator = "COSLAB Hub";

  const ws = wb.addWorksheet("Quotation", {
    pageSetup: { paperSize: 9, orientation: "portrait", fitToPage: true }
  });

  ws.columns = [{ width: 3 }, { width: 22 }, { width: 22 }, ...];
  ws.mergeCells("B5:H5");

  // 색상
  const NAVY = "FF1F4E79";
  const headerFill = (argb: string) => ({ type: "pattern", pattern: "solid", fgColor: { argb } });
  ws.getCell("B5").fill = headerFill(NAVY);

  // 테두리
  const thin = { style: "thin", color: { argb: "FFB4C6E7" } };
  ws.getCell("B5").border = { top: thin, bottom: thin, left: thin, right: thin };

  const buffer = await wb.xlsx.writeBuffer();
  saveAs(new Blob([buffer]), "Quotation.xlsx");
}
```

### 파일명 규칙

- `Stability-Confirmation_<product>_v<ver>_<date>.docx`
- `Quotation_<quote_number>_<date>.xlsx`
- 날짜: `formatJakartaDate()` (YYYY-MM-DD)

---

## 9. AI 패턴

### Claude SDK wrapper

```ts
// src/lib/ai/client.ts
import Anthropic from "@anthropic-ai/sdk";

let _client: Anthropic | null = null;

export function getAnthropicClient(): Anthropic {
  if (_client) return _client;
  const apiKey = process.env.ANTHROPIC_API_KEY;
  if (!apiKey) throw new Error("Missing ANTHROPIC_API_KEY");
  _client = new Anthropic({ apiKey, maxRetries: 3 });
  return _client;
}

// 사용
const client = getAnthropicClient();
const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  messages: [{ role: "user", content: "..." }],
});
```

### 구조화된 출력

```ts
// src/lib/ai/claude.ts
export async function classifyMaterialDocument(
  text: string,
  fileName: string,
  materialNames: string[]
): Promise<{ doc_type: string; material_name_guess: string | null; confidence: number; reason: string }> {
  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 512,
    messages: [{
      role: "user",
      content: `Respond in this exact JSON (no markdown):
{"doc_type": "CoA"|"MSDS"|..., "confidence": 0.0-1.0}`,
    }],
  });

  const raw = response.content[0].type === "text" ? response.content[0].text : "";
  const cleaned = raw.replace(/```json?\n?/g, "").replace(/```/g, "").trim();
  return JSON.parse(cleaned);
}
```

### 재시도

- SDK 레벨: `maxRetries: 3` — transient 408/429/5xx 자동 처리
- 비재시도: 400/401/403/404 즉시 실패
- exponential backoff + jitter 내장

### 프롬프트 위치

- 복잡: `src/lib/ai/prompts/`
- 짧음: 함수 인라인
- 항상 컨텍스트로 시작: 모델, 토큰 제한, 출력 형식

---

## 10. 오프라인 / PWA

### IndexedDB 큐

```ts
// src/lib/offline/db.ts
import { openDB, type IDBPDatabase } from "idb";

export interface PendingItem {
  id: string;
  type: "voice" | "photo" | "text";
  data: Blob | string;
  timestamp: number;
  synced: boolean;
}

// Stores: "pending-voice", "pending-photos", "pending-text"
savePending(type, data): Promise<string>      // → id
getPending(): Promise<PendingItem[]>          // 동기화 대기
markSynced(id): Promise<void>
clearSynced(): Promise<void>
```

### Service Worker 등록

```ts
// src/lib/pwa/register.ts
export async function registerServiceWorker(): Promise<void> {
  if (!("serviceWorker" in navigator)) return;
  const registration = await navigator.serviceWorker.register("/sw.js", { scope: "/" });
  registration.addEventListener("updatefound", () => {
    // 새 버전 감지, 사용자 알림, 리로드
  });
}
```

### 컴포넌트 사용

```tsx
// src/components/PWARegister.tsx
"use client";
import { useEffect } from "react";
import { registerServiceWorker } from "@/lib/pwa/register";

export function PWARegister() {
  useEffect(() => { registerServiceWorker(); }, []);
  return null;
}
```

---

## 11. 명명 규칙

### 폴더 구조 (App Router)

```
src/app/
├── (app)/                  # 그룹 layout (Sidebar/Topbar 포함)
│   └── formulation/
│       └── [id]/
│           └── page.tsx    # /formulation/:id
├── api/
│   └── unified/
│       └── formulations/
│           └── [id]/
│               └── route.ts # PATCH /api/unified/formulations/:id
└── login/
    └── page.tsx            # Sidebar 없음 (no (app) 그룹)
```

### 파일 명명

| 유형 | 규칙 | 예시 |
|---|---|---|
| 파일 (lib) | kebab-case | `export-quotation.ts`, `stability-confirmation-docx.ts` |
| 컴포넌트 | PascalCase | `AdminOnly.tsx`, `SortableHeader.tsx` |
| 디렉토리 | kebab-case | `src/lib/change-control/`, `src/components/shared/` |
| 함수 | camelCase | `createFormulaSnapshot()`, `getApiUser()` |

### 마이그레이션

```
YYYYMMDD[a-z]_<topic>.sql

20260330_pv_tables.sql
20260401_security_hardening.sql
20260403a_sample_first.sql
20260506a_unified_trash_phase_b.sql
```

마이그레이션 적용은 **Supabase Dashboard에서 수동** (push만으로 적용 안 됨). 적용 후 `supabase/migrations/applied.txt`에 기록.

---

## 12. Next.js 16 주의사항

**참고**: `c:/temp/coslab-hub/AGENTS.md` — *"This is NOT the Next.js you know"*

### 주요 변경 (구버전 패턴 사용 금지)

- Server Components 기본 (Client 아님)
- 인터랙션·hook·context 사용 시 `"use client"` 필수
- `app/` 라우터만 (`pages/` 없음)
- 동적 라우트: `[id]` (`:id` 아님)
- 라우트 핸들러: `export async function GET/POST/PATCH/DELETE(req, { params })`
- **`params`가 Promise — 항상 `await params`**
- Supabase 클라이언트에서 RLS 자동 적용 안 됨 (supabaseAdmin + auth check 사용)

### Server Actions

본 코드베이스에서는 사용 안 함. API 라우트가 표준 패턴.

### 라우트 핸들러 형태

```ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;       // ⚠️ 반드시 await
  // ...
  return NextResponse.json({ data }, { status: 200 });
}

export async function PATCH(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const body = await req.json();
  // auth check + 로직
  return NextResponse.json({ result });
}
```

### 라우트 핸들러에서 인증

```ts
import { getApiUser } from "@/lib/auth/api-auth";

export async function POST(req: NextRequest) {
  const user = await getApiUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  if (user.role !== "admin") return NextResponse.json({ error: "Forbidden" }, { status: 403 });
  // 진행
}
```

---

## 요약 표

| 영역 | 키 파일 | 패턴 |
|---|---|---|
| **Time** | `lib/time/jakarta.ts` | `formatJakarta()`, `todayJakartaYMD()`, `jakartaMonthAnchor()` |
| **Auth** | `lib/auth/api-auth.ts` | `getApiUser()` → DB role, 컴포넌트 wrapper 사용 |
| **DB** | `lib/supabase/server.ts` | `supabaseAdmin` (Service Role), API 라우트가 게이트키퍼 |
| **Change Control** | `lib/change-control/` | snapshot + auto-lock + price-impact + audit_log |
| **Concurrency** | `api/unified/.../route.ts` | optimistic: `updated_at` 버전 가드 |
| **Trash** | `lib/trash/references.ts` | soft-delete + 참조 검사 + `?purge=1` |
| **Components** | `components/shared/` | `AdminOnly`, `EditableOnly`, `useSortFilter`, `SortableHeader` |
| **Exports** | `lib/stability/`, `lib/quote/` | DOCX (`docx`), XLSX (`ExcelJS`), 파일명 규칙 |
| **AI** | `lib/ai/client.ts` | Anthropic SDK, maxRetries: 3, 구조화 JSON 파싱 |
| **Offline** | `lib/offline/db.ts` | IndexedDB 큐 (`pending-voice`, etc.) |
| **Naming** | `supabase/migrations/` | kebab-case 파일, PascalCase 컴포넌트, `YYYYMMDD_topic.sql` |
| **Next.js** | `/AGENTS.md` | Server Components 기본, `await params`, API 라우트만 |

---

## 부록: 검증 체크리스트 (PR 전)

- [ ] `npx next build` 로컬 통과 (Vercel strict TypeScript)
- [ ] `formatJakarta` 외 타임존 호출 없음
- [ ] `new Date().toISOString().slice(0,10)` 없음 → `todayJakartaYMD()` 사용
- [ ] `new Date(y, m, d)` 없음 → `jakartaMonthAnchor` 또는 UTC 명시
- [ ] API route에 `getApiUser()` + role check
- [ ] 변경 작업이 audit_log 발행 (필요 시 `withAudit` wrapper 사용)
- [ ] terminal status 전이 시 snapshot 생성
- [ ] DELETE는 soft-delete가 기본 + `?purge=1`로만 hard-delete
- [ ] 컴포넌트 인터랙션 — `"use client"` 마커 추가
- [ ] 동적 라우트 — `await params`

---

**총 12개 영역 × 평균 5-10개 구체 예시 = 60+ 패턴**

<!-- SOURCE_FILE_2_END: COSLAB-HUB-CONVENTIONS.md -->

---


<!-- SOURCE_FILE_3_BEGIN: COSLAB-HUB-DESIGN.md -->

# SOURCE FILE 3: `COSLAB-HUB-DESIGN.md`

# COSLAB Hub — 통합 설계서 (v3.0)

> **이 문서는 COSLAB Hub 플랫폼의 단일 진실 설계서다.**
>
> 2026-03~05 기간에 작성된 4개의 설계서(`odm-unified-platform-design.md` v2.0, `odm-ai-operations-design-v2.md` v2.1, `coslab-hub-phase3-design.md` v2.0, `coslab-hub-schema-mapping.md`)를 본 문서로 통합·갱신한다.
>
> v3.0 작성: 2026-05-05 — 실제 구현 상태 반영 (Talent 모듈 추가, Phase 3 완료 부분 정리, 보류/연기 항목 명시)
>
> 현재 구현 상태 점검은 별도 문서 [`COSLAB-HUB-STATUS.md`](./COSLAB-HUB-STATUS.md) 참조.

---

## 목차

- [1. 비전 & 핵심 결정](#1-비전--핵심-결정)
- [2. 통합 아키텍처](#2-통합-아키텍처)
- [3. 사이드바 & 모듈 구조 (확정)](#3-사이드바--모듈-구조-확정)
- [4. DB 스키마 전략](#4-db-스키마-전략)
- [5. 모듈별 상세 설계](#5-모듈별-상세-설계)
- [6. 공유 인프라](#6-공유-인프라)
- [7. AI 기능 계층 (Smart Advisor / Formulator / Studio)](#7-ai-기능-계층)
- [8. 변경관리 & 버전 시스템](#8-변경관리--버전-시스템)
- [9. 보안 & 컴플라이언스](#9-보안--컴플라이언스)
- [10. 로드맵 & 우선순위](#10-로드맵--우선순위)
- [11. 결정 기록 (Decision Log)](#11-결정-기록-decision-log)

---

## 1. 비전 & 핵심 결정

### 1.1 비전

**COSLAB Hub** — 인도네시아 소재 화장품 ODM(PT Daewoong Pharmaceutical Indonesia)의 R&D / 인허가 / 영업 / 인력 운영을 단일 플랫폼으로 통합한다. 기존 5개 독립 앱(CosReg Hub, Cosmetic ODM ERP, Quick Quote, Safety Assessment, Package Validator) + Knowledge Hub를 흡수했고, 여기에 Talent / Onboarding / 변경통제 / AI 보조를 추가했다.

### 1.2 핵심 결정 (확정 완료)

| 카테고리 | 항목 | 결정 |
|---|---|---|
| 인프라 | 플랫폼명 | **COSLAB Hub** (`coslab-hub.vercel.app`) |
| 인프라 | GitHub | `github.com/jadesun18-sketch/coslab-hub` |
| 인프라 | Supabase | 단일 프로젝트 `oudhnhshsmawgvktwxtu` (Pro 플랜) |
| 인프라 | Vercel | Hobby 플랜 (cron 1개 — `/api/backup/auto` 일일 18:00 UTC) |
| 운영 | 운영지 | PT Daewoong Pharmaceutical Indonesia |
| 운영 | 표시 TZ | **Asia/Jakarta (UTC+7)** — `src/lib/time/jakarta.ts` 헬퍼로 강제 |
| 운영 | UI 언어 | 영어 (English) |
| 운영 | 디바이스 | 데스크톱 70% / 모바일 30% (PWA + 반응형) |
| 인증 | 방식 | Hub 내장 인증(`hub_accounts`) + Supabase Auth 병행 + TOTP 2FA(선택) |
| 인증 | 역할 | `admin` / `editor` / `viewer` 3등급. **manager/4롤은 2026-04-28 deferred — QA 매니저 채용 시점에 추가 예정 (5h 작업 추산)** |
| 보안 | 데이터 | 영업비밀 클라우드 저장 — RLS + Service Role 키 + Audit Trail로 보호 |
| 데이터 | 시간대 저장 | DB는 UTC ISO, 표시만 자카르타 |
| 데이터 | 백업 | 일일 자동 백업 (`/api/backup/auto`, cron) |
| AI | 모델 | Anthropic Claude Sonnet 4 (Opus는 옵션) — 월 $10 이하 목표 |
| AI | 동작 원칙 | **자동 트리거 없음**. 상황별 추천/제안 + 사용자 확인 단계 유지 |
| Quick Quote | 운영 | 통합 후 종료. `/quotes` 모듈로 흡수 완료 |
| 라이프스테이지 | 현재 | **BETA** (모든 페이지 BETA 배지 + robots noindex + scanner 시그널) |

### 1.3 1인 개발 + AI 협업 원칙

- 개발자: **1인 (Sunny)** + Claude Code
- 주간 투입: ~20시간
- 팀: 비개발 연구원 3~5명 + 본인
- 변경 부담을 줄이기 위해 — **단일 스택**(Next.js 16 + Supabase + Tailwind), **공통 헬퍼 모듈**(`/lib/`), **Service Role 키 단일 출처**(`/lib/supabase/server.ts`)

---

## 2. 통합 아키텍처

### 2.1 전체 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       COSLAB Hub — Unified Platform                      │
│                                                                         │
│  Executive · Dashboard                                                   │
│  ─────────────────────────────────────────────────────────────────────   │
│  RAW MATERIALS · FORMULA · TESTING & SAMPLES · PRODUCT · OPERATIONS     │
│  BPOM & HALAL · DOCUMENTS · TALENT · AI ENHANCEMENT                     │
│  ─────────────────────────────────────────────────────────────────────   │
│                                                                         │
│  Shared services:                                                        │
│    Auth (hub_accounts + Supabase) · Storage · AI (Claude)               │
│    Smart Advisor · Audit Trail · Change Control · Snapshots             │
│    Jakarta-TZ helpers · PWA + Offline · Backup cron                     │
│                                                                         │
│  Single Supabase DB (ref: oudhnhshsmawgvktwxtu)                         │
│  Single Vercel deployment (Hobby + 1 cron)                              │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Tech Stack

```yaml
framework:    Next.js 16 (App Router, RSC + Server Actions)
language:     TypeScript
db:           Supabase (Postgres 15) — Pro plan
auth:         hub_accounts table + Supabase Auth (jose JWT)
2fa:          TOTP (otpauth) — optional opt-in
styling:      Tailwind CSS 4
ai:           @anthropic-ai/sdk (Claude Sonnet 4)
docs:         docx (Word), exceljs (Excel), pptxgenjs (PPT)
pdf:          pdf-parse (input), html2canvas + manual layout (output)
ocr:          mammoth (.docx → text), Claude Vision (images/PDFs)
qr:           qrcode + otpauth
crypto:       bcryptjs (password), jose (JWT), Node crypto (TOTP secrets)
offline:      IndexedDB (idb), next-pwa
deploy:       Vercel Hobby + Supabase Pro
```

### 2.3 디렉토리 구조

```
src/
├── app/                       # Next.js App Router
│   ├── (root pages)           # /executive, /dashboard, /welcome, /help, /login
│   ├── api/                   # 모든 서버 API (route.ts)
│   ├── formulation/           # Formulas / Base Formulas / Process Flow / Materials / INCI / Samples
│   ├── samples/               # 샘플 제조지시서
│   ├── stability/             # 안정성 시험
│   ├── safety/                # Safety Assessment 7-단계
│   ├── packaging/             # 포장재/인쇄
│   ├── label-check/           # Package Validator (별도 PV Supabase 참조)
│   ├── registration/          # BPOM / Halal / DIP / Company / Materials
│   ├── documents/             # 통합 문서 허브
│   ├── projects/              # 프로젝트 (간소)
│   ├── proposals/             # 제안서
│   ├── quotes/                # 견적
│   ├── clients/               # 고객사
│   ├── suppliers/             # 공급사
│   ├── inventory/             # 재고 (Lots / Packaging)
│   ├── production/            # 생산 기록
│   ├── talent/                # Talent 모듈 (Org/People/Programs/Curricula/Sessions/External/Library)
│   ├── ai-studio/             # AI Studio 랜딩
│   ├── ai-formulator/         # AI 처방 추천
│   ├── smart-advisor/         # AI 챗 + 도움말
│   ├── knowledge/             # 지식 DB (Explorer/Voice)
│   ├── chat/                  # AI Chat (knowledge-aware)
│   ├── upload/                # 일괄 파일 업로드 + AI 파싱
│   ├── settings/              # 사용자/감사로그/원가기준/품질시험/보안정책
│   ├── profile/               # 본인 프로필
│   └── executive/             # 경영진 대시보드 + 보고서
├── components/
│   ├── layout/                # AppShell / Sidebar / Topbar / WelcomeTip / IdleTimeout
│   ├── auth/                  # Login + 2FA UI
│   ├── shared/                # AdminOnly / EditableOnly / SortableHeader 등
│   ├── formulations/          # Phase Editor / Version Compare / Cost panel
│   ├── materials/             # Picker / Halal Matrix
│   ├── registration/          # Bulk cert upload / Submission timeline
│   ├── talent/                # Calendar / Quiz UI / Eval Modal / External training modal
│   └── ...
└── lib/
    ├── supabase/              # server.ts (Service Role) + package-validator.ts (PV 별도)
    ├── auth/                  # api-auth, hash, jwt, totp
    ├── time/jakarta.ts        # 표시 TZ 헬퍼 (todayJakartaYMD, jakartaMonthAnchor 등)
    ├── ai/                    # Claude wrappers + structured-output helpers
    ├── change-control/        # snapshot.ts / auto-lock.ts / price-impact.ts
    ├── safety-pipeline/       # 7-단계 안전성 평가
    ├── stability/             # 안정성 레터 + DOCX
    ├── quote/                 # 원가/배치 계산
    ├── talent/                # 일정/평가/퀴즈 채점
    ├── label-check/, dip/     # 라벨체크/DIP 모음
    ├── offline/, pwa/         # IndexedDB 동기화 + Service Worker
    └── crypto/, pii-mask.ts   # 보안 유틸
```

---

## 3. 사이드바 & 모듈 구조 (확정)

```
Executive                         (admin/manager)
Dashboard                         (전원)

▼ RAW MATERIALS
  Materials                       /formulation/ingredients
  Sample Materials                /formulation/sample-materials
  INCI List                       /formulation/inci
  Suppliers                       /suppliers
  Inventory                       /inventory

▼ FORMULA
  Formulas                        /formulation/formulas
  Base Formulas                   /formulation/base-formulas
  Process Flow                    /formulation/process-flow

▼ TESTING & SAMPLES
  Stability Tests                 /stability
  Samples                         /samples
  Safety Assessment               /safety
  Test Templates                  /test-templates

▼ PRODUCT
  Products                        /formulation/products
  Packaging & Printing            /packaging

▼ OPERATIONS
  Quotes                          /quotes
  Clients                         /clients
  Production                      /production
  Proposals                       /proposals
  Projects                        /projects

▼ BPOM & HALAL
  Label Check                     /label-check
  DIP Gathering                   /registration/dip
  BPOM                            /registration/bpom
  Halal                           /registration/halal
  Company Docs                    /registration/company

▼ DOCUMENTS
  All Documents                   /documents
  Categories                      /documents/categories

▼ TALENT
  Dashboard                       /talent
  My Profile (Talent)             /talent/me
  Organization                    /talent/organization
  People                          /talent/people
  Programs                        /talent/programs
  Curricula                       /talent/curricula
  Sessions                        /talent/sessions
  External Training               /talent/external-trainings
  Library                         /talent/library

▼ AI ENHANCEMENT
  AI Studio                       /ai-studio
  Smart Advisor                   /smart-advisor
  AI Formulator                   /ai-formulator
  Knowledge Explorer              /knowledge
  AI Chat                         /chat
  Upload                          /upload                (editor+)
  Voice & Photo                   /knowledge/voice       (editor+)

(하단)
My Profile                        /profile
Settings                          /settings              (admin)
```

### 권한 규칙

```
admin (level 3)  : 전체 + Settings + 사용자 관리 + Audit 조회
editor (level 2) : 모든 모듈 CRUD + Upload + Voice/Photo (Settings 제외)
viewer (level 1) : 읽기 전용
```

`minRole`이 명시된 항목은 해당 등급 미만에 안 보임. 그룹 단위 hide 가능.

---

## 4. DB 스키마 전략

### 4.1 단일 Supabase + 네임스페이스 접두사

| 접두사 | 영역 | 예시 |
|---|---|---|
| (없음) | 공유 코어 | `materials`, `products`, `formulations`, `brands`, `clients`, `suppliers`, `base_formulas`, `inci_dictionary`, `hub_accounts`, `audit_log` |
| `kh_` | Knowledge Hub | `knowledge_entries`(legacy 이름 유지), `tags`, `entry_tags`, `attachments` |
| `sa_` | Safety Assessment | `sa_jobs`, `sa_ingredients`, `sa_toxicology`, `sa_mos`, `sa_reports`, `sa_escalations` |
| `pv_` | Package Validator | **별도 Supabase 프로젝트**(`/lib/supabase/package-validator.ts`) — `/label-check` 진입점만 통합 DB에 두고 데이터는 PV 프로젝트에서 읽음 |
| `qq_` | Quotes | `qq_quotation_items`, `qq_quality_tests`, `qq_cost_basis`, `qq_quotation_items_snapshot` |
| `reg_` | Registration | `registration_folders`, `registration_documents`, `registration_submissions`, `material_documents`, `company_documents`, `packaging_documents` |
| `talent_` | Talent | `talent_people`, `talent_teams`, `talent_positions`, `talent_assignments`, `talent_external_trainings`, `training_programs`, `training_curricula`, `training_sessions`, `training_evaluations`, `training_attendance`, `training_quizzes`, `training_quiz_responses`, `training_reports` |
| `prod_` | Production | `production_records`, `process_times`, `material_usage`, `labor_records` |
| `prop_` | Proposals | `proposals`, `proposal_products`, `proposal_outcomes`, `proposal_templates` |
| `cc_` / 변경통제 | 변경관리 | `formula_snapshots`, `formulations_lock_metadata`, `change_alerts`, `audit_log`, `product_versions` |
| `smart_` | AI 규칙 | `smart_rules` (계획) |

### 4.2 외부 PV 프로젝트 정책

Package Validator는 별도 Supabase 프로젝트로 분리되어 있고 자체 13단계 OCR 파이프라인을 운영한다. COSLAB Hub의 `/label-check`는 PV의 데이터를 읽어 통합 UI를 제공한다.

→ 향후 통합 시 `pv_*` 접두사로 이관 예정. 현재는 cross-project 클라이언트(`supabasePV`)로 접근.

### 4.3 마이그레이션 정책

- 마이그레이션은 `supabase/migrations/YYYYMMDD[a-z]_<topic>.sql` 명명
- 본 저장소 push만으로 적용되지 않음 — **Supabase Dashboard에서 수동 적용** (단단한 보호)
- 적용 후 `supabase/migrations/applied.txt`에 기록
- 76개 마이그레이션 적용 완료 (2026-03-30 ~ 05-04)

---

## 5. 모듈별 상세 설계

> 각 모듈은 (a) 핵심 기능, (b) 데이터, (c) 통합 지점을 명시. 진척 상태는 [`COSLAB-HUB-STATUS.md`](./COSLAB-HUB-STATUS.md)에서 별도 추적.

### 5.1 Formula (`/formulation/formulas`)

**핵심 기능**
- Phase 구조 (`versions[].phases[].ingredients[]`) — 1급 객체로 Phase 분리
- 자동 코드 생성 (`{이니셜}-{카테고리}-{시퀀스}`, 예: `SK-SRM-001`)
- 6개 탭: Overview / Specs & Feasibility / Stability / Documents / Samples / Review History
- 버전 비교 (`/formulation/formulas/[id]/compare?v1=2&v2=3`) — diff highlight
- 리뉴얼 관리 (`renewal_group`, `superseded_by`, `is_active`)
- Export: Formula Sheet (XLSX/PDF), Spec Sheet, Manufacturing Sheet, Full Dossier (PDF), BOM Export

**데이터**
- `formulations` — `formula_code`, `phases JSONB`, `client_id`, `renewal_group`, `is_active`, `superseded_at`, snapshot lock 메타
- `formula_specs` — appearance, viscosity, pH, SG, manufacturing_method[], packaging_compatibility[]
- `formula_code_sequences` — 사람×카테고리 시퀀스
- `formula_snapshots` — 잠금 시점 BOM/스펙 동결

**통합 지점**
- → Stability: `formulation_id + version`로 조회 (Stability 탭)
- → Samples: 동일 키로 제조 이력
- → Documents: `entity_type='formulation'`
- → Quotes: `qq_quotation_items.formulation_snapshot`로 견적 시점 동결
- ← Materials: `materials.id` FK
- ← Clients: `formulations.client_id` FK
- → Smart Advisor: 변경 시 시험/규제 추천 (계획)

### 5.2 Stability (`/stability`)

**핵심 기능**
- 2-단계 레터 시스템: **1-Month Confirmation** (BPOM 제출용) + **Final Confirmation** (출하 직전)
- Researcher Report DOCX (R&D 내부용 풀 보고서)
- 셀 단위 자동 잠금 + 수정 사유 + 감사로그 diff (GMP 준수)
- 조건 세트 × 인터벌 × 파라미터 매트릭스
- BPOM/CPKB 표준 표현 적용

**데이터**
- `stability_tests` — `test_code`, `formulation_id+version`, `batch_reference`, `test_date`, `verdict_date`, `final_verdict`
- `stability_condition_sets` — temp/humidity/light/packaging
- `stability_intervals` — `interval_name`, `days_from_start`
- `stability_parameter_master` + `stability_test_results` (셀 단위 잠금)
- `stability_test_templates` — 재사용 템플릿

**통합 지점**
- ← Formula: `formulation_id+version`
- → Registration/DIP: 1-Month 레터가 BPOM 제출세트에 자동 첨부
- → Documents: 레터 PDF 자동 게시

### 5.3 Samples (`/samples`)

**핵심 기능**
- 제조지시서(Manufacturing Sheet)
- Phase별 제조 기록 (자유 양식 + 편집 가능)
- QC 결과 기록 (수율, pH, 점도, SG)
- 워크플로우 상태머신 + Confirmed by client (A.2)
- DPI 샘플 코드 자동/수동 (수동 편집 허용)
- DOCX export (Phase 이름 + 지시문 포함)

**데이터**
- `sample_manufacturing` — `sample_code`, `formulation_id+version`, `phase_records JSONB`, QC 컬럼, `result`, `approved_by`
- `sample_qc_specs` — 스펙 매트릭스
- `sample_materials` — 샘플 전용 원료 인벤토리

**통합 지점**
- ← Formula
- → Stability: 1:1 또는 N:1 연결
- → Production records (장기적)

### 5.4 Materials & INCI (`/formulation/ingredients`, `/formulation/inci`)

**핵심 기능**
- 원료 마스터 (`materials`) — INCI / CAS / 공급사 / 단가 / Halal / 기능 / 사용 범위
- INCI 사전 (`inci_dictionary`) — 보정 + 검색
- 가격 잠금 + 변경 알람 (Price Lock)
- Sample-only 인벤토리 (`sample_materials` — Production과 분리)
- 카테고리/태그 필터, sort/filter UI

**통합 지점**
- → Suppliers (`supplier_materials` 다대다)
- → Inventory (`inventory_lots`)
- → Safety: `sa_ingredients.material_id`
- → Halal Matrix (`halal_material_product_matrix`)

### 5.5 Suppliers / Clients / Inventory

- `suppliers` + `supplier_materials` — 가격/리드타임/MOQ
- `clients` (구 `qq_customers` 통합) — Client 상세에서 Formula/Quote/Document 일람
- `inventory_lots` — 원료 Lot, 입고/출고/유효기간/QR
- `packaging_inventory` — 포장재 SKU 재고

### 5.6 Products (`/formulation/products`)

- 7개 탭: Info / BOM & Cost / Formula / Registration / Safety / Packaging / Knowledge
- Product Versions Phase C (manifest + auto-bump)
- 풀 BOM 인쇄 페이지

### 5.7 Packaging & Printing (`/packaging`)

- 2-탭 보존: Packaging Materials (Primary/Secondary/Other) + Printing Materials (Outsource/InHouse)
- `is_active` 컬럼으로 soft delete

### 5.8 Quotes (`/quotes`)

- 10항목 원가 계산 엔진 (`src/lib/quote/`)
- 배치 수량 로직 + ExcelJS 견적서 출력
- 견적 시점 BOM 동결 (`qq_quotation_items_snapshot`)
- 카테고리 매칭 + 전체 처방 검색
- 가격 변경 시 `change_alerts`로 영향도 표시

### 5.9 Registration (`/registration/*`)

- BPOM 6단계 + Halal 5단계 워크플로우
- DIP Gathering — 회사문서 + 원료문서 + 안정성 레터 자동 수집 → ZIP 패키지
- Bulk certificate upload + filename 자동 분류 (BPOM 4 슬롯 / HALAL 3 슬롯)
- Submission snapshots + Approve 버튼
- Material/Packaging/Company 문서 자동 만료 알림 (`change_alerts`)
- Drift 감지 (`drift_acknowledge`) — 처방 변경 시 BPOM/HALAL 인증 영향 표시

### 5.10 Safety Assessment (`/safety`)

- 7단계 파이프라인 보존: Pull → Resolve INCI → Tox lookup (3-Tier: 캐시→PubChem→Claude) → MoS 계산(SCCS) → 검증 → 보고서 → 에스컬레이션
- 이중언어 보고서 (EN/ID)
- AI retry 3회 + parallelize stability-assessment

### 5.11 Label Check (`/label-check`)

- Package Validator의 13단계 검증 파이프라인 (별도 PV 프로젝트)
- 다중 버전 리뷰 + 피드백
- BPOM Reg. 25/2025 + Reg. 3/2022 규제 DB
- 계획: PV의 데이터를 통합 Supabase로 마이그레이션 → `pv_*` 접두사

### 5.12 Documents (`/documents`)

- 통합 문서 허브 — 8개 출처 통합 (Materials / Packaging / Company / Registration / Stability / Sample / Formula / Upload)
- 카테고리 관리 (`/documents/categories`)
- 폴더 업로드 + 일괄 처리
- 한국어 파일명 처리 + DOMMatrix polyfill
- 청크 업로드(4.5MB 우회)

### 5.13 Talent (`/talent/*`) — **신규 모듈**

**핵심 기능**
- People + Organization (Teams / Positions / JD)
- Programs + Curricula (재사용 라이브러리, OJT 8 sessions + Regular 12 curricula 분리)
- Sessions — 5단계 라이프사이클(Scheduled → InProgress → Done → Evaluated → Approved/Closed) + 월간 캘린더
- Quizzes + Trainee Taker UI + 단답 자동채점 (`rt01_short_answer_rules`)
- Quiz Retest 워크플로
- Per-participant Self-Evaluation + Trainer Evaluation
- Action-first `/talent/me` (액션 버킷: tests_to_take, retests_pending, self_reports_due, evaluations_due, quizzes_to_grade, sessions_to_approve)
- External Training (개인 + 벌크 등록, group_id, 자료/보고서 슬롯)
- Library — 학습 자료 인라인 편집

**데이터**
- `talent_people`, `talent_teams`, `talent_positions`, `talent_assignments`
- `training_programs`, `training_curricula`, `training_sessions`
- `training_evaluations`, `training_attendance`, `training_reports`
- `training_quizzes`, `training_quiz_responses`
- `talent_external_trainings`

**통합 지점**
- ← `hub_accounts` (auth_email 매칭)
- → 자격(컴퓨터, 안전 등) → Production 작업 권한 (장기적)

### 5.14 Knowledge / AI (`/knowledge`, `/chat`, `/upload`)

- Knowledge Entries (8유형) — Explorer/Voice/Chat/Upload 4개 입력 채널
- Upload: AI 파싱 + 중복 감지 + 자동 분류 + 재시도 2회 → 수동 큐
- Chat: knowledge-aware 챗 (RAG 풍)
- Voice & Photo: 모바일 녹음/사진 → IndexedDB → 동기화

### 5.15 AI Studio / Smart Advisor / AI Formulator

| 페이지 | 역할 |
|---|---|
| `/ai-studio` | 4개 AI 기능 카드 + pre-baked demo (랜딩) |
| `/smart-advisor` | knowledge-aware 챗 + Help-aware (Quick Start) |
| `/ai-formulator` | 클레임/카테고리 입력 → Production + Sample Materials 인벤토리 활용해 처방 초안 생성 |

→ 7장 참조

### 5.16 Production / Proposals / Projects / Executive

- `/production` — 생산 기록, 공수, 원료 투입, 수율 (Knowledge Hub 보존)
- `/proposals` — 제안서 + 결과 추적
- `/projects` — 프로젝트 보드 (간소 — Phase B/C 보류)
- `/executive` — 경영진 대시보드: snapshot/period split, blockers, pipeline movements, base formula period activity
  - PDF/DOCX 보고서 생성 + 기간별 (week/month/custom)

### 5.17 Onboarding / Welcome / Help

- `/welcome` — 8단계 인터랙티브 walkthrough + 진행률
- `/help` — Q&A + Quick Start (30 Q&A knowledge seed)
- 영상 walkthrough (Playwright 캡처 자동화) — 시나리오 1~10

---

## 6. 공유 인프라

### 6.1 인증

```
Login flow:
  /login → hub_accounts.password 검증(bcryptjs)
        → TOTP 활성화면 6자리 입력
        → JWT 발급 (jose) → cookie
        → / (또는 의도 페이지)

Idle timeout: IdleTimeout 컴포넌트로 자동 로그아웃
2FA: TOTP secret을 hub_accounts에 암호화 저장 (Node crypto)
```

### 6.2 시간대 (Jakarta TZ)

`src/lib/time/jakarta.ts`가 단일 출처. **모든 표시·"오늘" 비교·캘린더 cursor**를 헬퍼로 통일.

```typescript
formatJakarta(d, opts?)        // 일반 표시 (DD/MM/YYYY HH:mm:ss WIB)
formatJakartaDate(d)           // YYYY-MM-DD
formatJakartaTime(d, sec?)     // HH:mm[:ss]
nowJakartaLabeled()            // "YYYY-MM-DD HH:mm WIB"
todayJakartaYMD()              // 오늘 (Jakarta) YMD
jakartaMonthAnchor(d?)         // 월 캘린더 cursor (UTC-noon mid-month)
makeMonthAnchor(year, month)   // 월 이동
```

**금지 패턴**: `new Date().toISOString().slice(0,10)` (UTC), `new Date(y,m,d)` 후 `formatJakarta` 호출 (TZ 경계 어긋남).

### 6.3 변경통제 & 감사

- **Snapshots**: `formula_snapshots`, `qq_quotation_items_snapshot`, `registration_folders_snapshot` — 잠금 시점 동결
- **Auto-lock**: Confirmed 전이 시 자동 잠금 + 수동 unlock 사유 입력
- **Price Impact**: 원료 단가 변경 → 영향받는 처방/견적 자동 알람 + revert 시 자동 해소
- **Audit Trail**: `audit_log` 전체 행위 기록 + 셀 단위 diff
- **Drift Detection**: 인증 후 처방 변경 시 BPOM/HALAL 영향도 표시

### 6.4 알림 (`change_alerts` + `alerts`)

| 종류 | 트리거 |
|---|---|
| Price drift | 원료 단가 변경 → 사용 처방/견적 |
| Document expiry | CoA/MSDS/인증서 30일 전 |
| Cell unlock requested | 잠금 셀 수정 시도 |
| Stability schedule | 측정 예정일 도래 |
| Cross-entity | Phase F — formula↔product↔registration↔quote 4축 |

### 6.5 PWA + 오프라인

```
Service Worker:
  - App Shell + 정적 자산 캐싱
  - 최근 조회한 지식 엔트리

IndexedDB (idb):
  - pending-voice / pending-photos / pending-text / pending-production
  - 온라인 복귀 시 자동 업로드 + 충돌 감지
```

### 6.6 Export 매트릭스

| 모듈 | 형식 | 산출물 |
|---|---|---|
| Formulation | XLSX/PDF | Formula Sheet, Spec Sheet, BOM, Full Dossier |
| Sample | DOCX | Manufacturing Sheet (Phase 이름+지시문) |
| Stability | DOCX | 1-Month Confirmation / Final Confirmation / Researcher Report |
| Safety | DOCX/PDF | 이중언어 안전성 평가 보고서 |
| Packaging | DOCX | 검증 리포트 |
| Registration | ZIP | DIP 패키지 (회사 + 원료 + 안정성 일괄) |
| Quotes | XLSX | 견적서 (10항목 원가) |
| Proposals | PPTX | 제안서 (표준 양식, AI 초안) |
| Talent | XLSX | 평가/리포트 |
| Production | XLSX | 생산 기록 |
| Executive | PDF/DOCX | 기간별 보고서 |
| Backup | JSON | `/api/backup/auto` 일일 자동 |

### 6.7 글로벌 검색

`/api/search?q=...` — pg_trgm + full-text across knowledge / formulations / sa_jobs / pv_reviews / qq_quotations / registration_folders / talent_people.

---

## 7. AI 기능 계층

### 7.1 Smart Advisor (보류 — Sunny 결정)

**현재 상태**: `/smart-advisor` 페이지는 knowledge-aware 챗 인터페이스로만 구현. 규칙 엔진(`smart_rules`)은 **2026-05-05 보류 결정**, 향후 재개.

**원래 설계 (재개 시 참고용으로 보존)** — 처방/패키징/원료 변경 감지 → DB 규칙 매칭 → AI 보강 → 알림

```sql
CREATE TABLE smart_rules (              -- 보류
  rule_type TEXT,                       -- ingredient_trigger | change_trigger | combination_trigger
  trigger_condition JSONB,
  actions JSONB,
  source TEXT,                          -- bpom_reg | asean | internal_knowhow
  regulation_ref TEXT,
  priority INT
);
```

**시나리오 (재개 시)**
1. 레티놀 추가 → 변색/함량 시험 + 임산부 경고 + BPOM Reg. 25/2025 농도 한계
2. 패키지만 변경 → 상용성 시험만 + 안전성 재평가 불필요
3. 동일 처방 → 기존 데이터 재사용
4. 신규 원료 → 규제 DB 자동 확인 + 필요 서류 자동 목록

### 7.2 AI Formulator (`/ai-formulator`)

브리프 입력 (카테고리 + 클레임) → Production 인벤토리 + Sample Materials 인벤토리 모두 활용해 처방 초안 생성. 결과 저장 가능.

### 7.3 AI Studio (`/ai-studio`)

랜딩 페이지에 4개 AI 기능 카드 + 미리 만든 데모 대화. 신규 사용자 onboarding용.

### 7.4 모듈 내장 AI

| 위치 | 기능 |
|---|---|
| Upload | 파일 → 지식 엔트리 변환, 중복 감지, 자동 분류, 재시도 2회 |
| Knowledge → Chat | knowledge-aware 응답 (RAG 풍) |
| Safety pipeline | Tier 2/3 독성 데이터 검색, 종합 판정 |
| Stability assessment | 병렬화 + 3회 retry |
| Documents | CoA/MSDS 데이터 자동 추출 (Claude Vision) |
| Registration | 인증서 파일명 → 슬롯 자동 분류 |
| Talent | 단답 채점 (RT-01 rules), 평가 초안 |

### 7.5 비용 정책

- 월 $10 이하 목표
- Sonnet 4 기본 + 단순 작업은 Haiku 검토
- 캐싱 적극 활용 (`prompt caching` API 기능)
- 사용자 확인 단계 항상 유지 — 자동 트리거 X

---

## 8. 변경관리 & 버전 시스템

### 8.1 4축 버전 관리 (Phase A~F 완료)

| 축 | 대상 | 메커니즘 |
|---|---|---|
| Formula 버전 | `formulations.versions[]` | versions JSONB, version_number, ver_status |
| Product 버전 | `product_versions` manifest | Phase C — 컴포넌트 변경 시 auto-bump |
| Submission 버전 | `registration_submissions` | BPOM/HALAL 다중 슬롯 + 스냅샷 |
| Quote 버전 | `qq_quotation_items_snapshot` | 발행 시점 BOM 동결 |

### 8.2 잠금 / 잠금 해제

- Confirmed 전이 시 auto-lock (snapshot 생성)
- `cell_audit_lock` 패턴 — 측정 데이터 셀 자동 잠금 + 수정 사유 + 감사로그 diff
- 수동 unlock — 사유 입력 필수, audit_log에 기록

### 8.3 Drift Detection (Phase D)

- BPOM/HALAL 인증 후 처방 변경 시 영향도 평가
- `drift_acknowledge` 테이블에 인지 기록
- UI 배너로 위험 표시

### 8.4 Cross-entity Alerts (Phase F)

- 단가 → 처방/견적/BOM
- 원료 단종 → 사용 처방
- 인증 만료 → 제품 출하 가능성
- 처방 잠금 → 견적 / 인증 영향

---

## 9. 보안 & 컴플라이언스

### 9.1 보안 정책 (v1.5.0 — 100% 완료)

- 키 노출 점검 + GitHub secret scanning
- JWT secret + CRON secret 분리
- 5xx 응답 sanitize (PII 마스킹)
- AI 가이드라인 감사 (대웅 그룹 정책 기반)
- SQL 인젝션 차단 (Supabase 파라미터 바인딩 강제)
- Rate limit (`src/lib/rate-limit.ts`)
- Idle timeout
- 2FA TOTP (옵션)

### 9.2 미구현 (Backlog)

- e-Signature
- AI 보안 백로그 7건 (`project_ai_security_backlog.md`)

### 9.3 GMP 준수

- 셀 단위 감사로그 (Stability)
- 수정 사유 필수
- Snapshot으로 변경 시점 동결
- Audit Trail 페이지 (`/settings/audit-trail`)

### 9.4 BETA 신호

- 모든 페이지 BETA 배지
- robots noindex
- machine-readable BETA 신호 (scanner/bot용)

---

## 10. 로드맵 & 우선순위

### 10.1 완료된 마일스톤

| 일자 | 마일스톤 |
|---|---|
| 2026-03-30 | Phase 3a (Formula schema) + qq/pv 테이블 통합 |
| 2026-03-31 | Phase 3b (Stability) + 3c (Operations) + 3d (Documents) + Projects |
| 2026-04-01 | Security hardening + TOTP 2FA |
| 2026-04-07 | Talent Phase A — People + Organization |
| 2026-04-08 | Talent Phase B/C/D — Programs/Curricula/Sessions/Library + Quizzes |
| 2026-04-09 | Registration submissions + multi-cert + product_versions + drift |
| 2026-04-11 | Base Formulas 강화 + Sample Materials |
| 2026-04-13 | Safety DB 이전 + SafetyData + Doc Matrix |
| 2026-04-14 | Documents 업로드 + 인증 + 5xx sanitize + Change Control Phase 2 (locks) |
| 2026-04-15 | Price Lock 완성 (snapshots + drift alerts + recompute + Δ% UI) + Talent self-service |
| 2026-04-16 | Base Name Registry + Sample/Stability ↔ Base Formula |
| 2026-04-17 | 보안 전면 점검 (키, JWT/CRON, AI 가이드라인, SQL 인젝션) |
| 2026-04-20 | Stability Confirmation+Researcher DOCX + External Training + Jakarta TZ sweep |
| 2026-04-21 | Stability 모듈 재설계 (2-stage 레터, 셀 잠금, migration-002c) |
| 2026-04-24 | 002c 공식화 + stability 백필 + AI retry 통합 |
| 2026-04-28 | Phase 4 soft delete + Phase 3 unlock + Executive Report 재작성 |
| 2026-04-30 | Talent 외부교육 벌크 + DPI 수동수정 + /documents 8소스 통합 + /upload 안정화 |
| 2026-04-30~05-04 | Onboarding (Phase A+B), 영상 walkthrough, Welcome 8-step, Smart Advisor help-aware |
| 2026-05-04 | Calendar Jakarta TZ 통합 (label + today 비교) |

### 10.2 다음 우선순위

**즉시 (Sprint 1)**
- /upload 재검토 (남은 이슈)
- AllTypes 필터 누락
- 등록일자 컬럼
- Doc# 출처 표시
- Talent 런타임 검증

**중기 (Sprint 2~3)**
- ~~Smart Advisor 규칙 엔진~~ — **보류 (2026-05-05 결정), 향후 재개**
- AI 보안 백로그 7건
- e-Signature
- 노란불 4건 잔여

**장기**
- Project Hub 본격 구축 (Phase B/C 보류 해제)
- Package Validator 데이터의 `pv_*` 통합
- 시맨틱 검색 (pg_vector 또는 외부 인덱스)
- Voice/Photo 입력 활용도 증대

---

## 11. 결정 기록 (Decision Log)

| 일자 | 결정 | 배경 |
|---|---|---|
| 2026-03-27 | 플랫폼명 COSLAB Hub로 확정 | DPI Lab → COSLAB Hub (인터뷰 v2.0) |
| 2026-03-27 | Knowledge Hub Supabase를 통합 DB로 확장 | 가장 포괄적인 스키마 + 신규 구축이라 레거시 부담 적음 |
| 2026-03-27 | Quick Quote 통합 후 종료 | 영업 미사용 상태였음 |
| 2026-03-27 | AI 자동 트리거 X — 추천/제안만 | "연구자가 직접 판단" 원칙 |
| 2026-03-27 | 데이터 클라우드 저장 임시 허용 | RLS + 권한 관리로 보호 |
| 2026-03-31 | Formula 6-탭 구조 + Phase 1급 객체화 | CM Studio+ 벤치마킹 |
| 2026-03-31 | 사이드바 8그룹 토글 (URL 변경 X) | 기존 코드 보존 + UX 개선 |
| 2026-04-07 | Talent 모듈 신설 | 기존 설계서에 없던 신규 영역 (인력/교육 운영) |
| 2026-04-14 | 통합 변경통제 (버전/스냅샷/잠금/감사로그) | 컴플라이언스 + price lock |
| 2026-04-20 | Stability 2-stage 레터 (1-Month + Final) | BPOM 제출 시점 + 출하 직전 분리 |
| 2026-04-21 | Cell-level 자동 잠금 + 수정 사유 | GMP 준수 |
| 2026-04-30 | /documents 8소스 통합 허브 | 폴더 업로드 + 일괄 처리 |
| 2026-04-28 | 4롤(approver) 모델 deferred | 솔로 운영에서 approver-게이팅 액션 0건. QA 매니저 채용 시점에 5h 작업으로 추가 |
| 2026-05-04 | Jakarta TZ 헬퍼 통일 (today + cursor) | KST 운영자에게서 캘린더 1달 밀림 버그 재발 방지 |
| 2026-05-05 | Smart Advisor 규칙 엔진 보류 | 향후 재개. 챗 인터페이스만 운영 중 |
| 2026-05-05~07 | 통합 트래시 패턴 (Phase A/B/C/D) | 11개 모듈 `is_active` soft-delete + 통합 `/trash` 페이지 |

---

## 부록 A. 보류된 설계 항목

| 항목 | 상태 | 사유 |
|---|---|---|
| Project Hub Phase B/C | 보류 | 우선순위 낮음 — `/projects`는 간소 페이지로 유지 |
| Group C (Label 검증 강화) | 보류 | 현재 Package Validator로 충분 |
| Voice/Photo 입력 강화 | 부분 보류 | 인프라 있음, 활용 사례 누적 후 결정 |
| Cross-DB 자동 동기화 | 폐기 | 단일 DB로 통합되어 불필요 |
| Project Briefing 위클리 | 폐기 | Executive Report로 대체 |

## 부록 B. 폐기된 가정

- "1주 안에 6개 앱 통합" → 실제로는 ~6주 (3월 30일 ~ 5월 초)
- "Package Validator 즉시 통합" → 별도 Supabase 유지 + `/label-check`만 통합 진입점
- "AI 비용 월 $10" → Sonnet 4 + 캐싱으로 거의 유지 중
- "오프라인 필수" → 인프라는 있으나 실제 활용은 적음

---

**문서 위치 및 동반 문서**

```
c:\Temp\New project\
├── COSLAB-HUB-DESIGN.md      (본 문서 — 통합 설계서 v3.0)
└── COSLAB-HUB-STATUS.md      (현재 구현 상태 점검)
```

> 본 설계서는 살아있는 문서다. 큰 변경 시 v3.x로 증분, 결정은 Decision Log에 추가한다.

<!-- SOURCE_FILE_3_END: COSLAB-HUB-DESIGN.md -->

---


<!-- SOURCE_FILE_4_BEGIN: COSLAB-Hub-PM-Module-기획서-v4.md -->

# SOURCE FILE 4: `COSLAB-Hub-PM-Module-기획서-v4.md`

# COSLAB Hub PM Module — 기획서 v4 (Interview-Final)

## 0. 결정 요약 (24 decisions from interview)

| # | 영역 | 결정 |
|---:|---|---|
| 1 | OKR Alignment | 본인 초안 + Manager(Sunny) 잠금 |
| 2 | PIP 라이프사이클 | draft → review → active(lock) → review (amendment 버전) |
| 3 | 체크리스트 evidence | 항목별 개별 설정 (정의자가 risk-based로 결정) |
| 4 | Auto progress 충돌 | auto + manual_override + 사유 (audit log) |
| 5 | Stage-Gate 전환 | 조건 자동 + Owner 클릭 확인 |
| 6 | 개발 리드타임 시작점 | First proposal 발송일 |
| 7 | 공급 리드타임 | PO 단위 + SKU 단위 병행 기록 |
| 8 | KR target 변경 | Manager 승인 + 원본 보존 + amendment 메모 |
| 9 | PM-도메인 결합 철학 | PM은 별도 모듈이 아닌 "역할/lens". base_formulas/products 직접 read/write |
| 10 | 권한 정책 | 전체 공개 (소규모 팀 투명성, RLS 단순) |
| 11 | 일별 이력 | 매일 02시 스냅샷 (추세 그래프 + Biweekly Before/After 용) |
| 12 | 팀원 계정 | 모두 보유 (즉시 도입 가능) |
| 13 | 첫 화면 | 모두 회사 KPI 트리부터 |
| 14 | 디바이스 | PC 전용 (현장 공용 PC) |
| 15 | 알림 채널 | 시스템 내부 알림함 + 개인 이메일 (병행) |
| 16 | 언어 | 영어 단일 |
| 17 | Biweekly Report | AI 자동 초안 → Sunny 검수 → 발행 |
| 18 | AI 기능 | Biweekly 초안 + 데일리 스크럼 + 분기 회고/KR 둔화 플래그 (3종) |
| 19 | MVP 범위 | Sprint 1+2 한 번에 (15.5일) |
| 20 | 종이 GMP | 종이는 그대로, 디지털은 "본인 task 완료 체크"만 (이중 부담 최소화) |
| 21 | 미작성 PIP (Dina) | Sunny가 임시 PIP 부여 → 본인이 v1.1에서 수정 |
| 22 | KR 미달 | 사례별 판단 (변동이면 원인 narrative만) |
| 23 | 과거 데이터 | Notion + 과거 docx Biweekly 모두 import |
| 24 | 매출 인식 | Invoice 발행 시점 |
| - | 휴직/퇴사 | 자동 archive + Sunny 알림 + 수동 재배치 |
| - | Core Value | 정량 KR + Narrative pairing |
| - | 도입 모드 | 점진·코칭 (첫 30일, Sunny 주 1회 멘토링) |

## 1. 핵심 아키텍처 변화 (vs v3)

### 1.1 PM = "lens" 패턴 (가장 중요한 결정)

v3 가정: PM은 별도 모듈, base_formulas는 읽기만
v4 결정: PM은 동일 사용자의 여러 역할 중 하나, 도메인 데이터를 직접 다룸
구현 함의:
- 별도 PM 전용 view 계층 불필요 → 직접 SQL/RPC 호출
- 체크리스트 항목이 base_formula 레코드를 직접 수정 가능 (예: "20개 product" KR의 체크리스트 항목 = base_formulas 신규 row 생성)
- /pm/me 페이지의 "오늘의 task"는 다른 모듈로 deep link (Sample 등록 페이지로 점프 → 등록 후 자동으로 PM에 반영)

### 1.2 권한 단순화

모든 PIP/KR/매출/원가 = 전직원 read·write 가능
RLS 정책 추가 거의 없음 (talent.id 인증만)
→ 코드 복잡도 대폭 감소. 다만 audit log는 강력하게 (누가 언제 무엇을).

### 1.3 일별 스냅샷 + 실시간 표시

표시: 항상 최신 값 (auto 또는 manual_override)
이력: 매일 02:00 KST cron이 pm_progress_snapshots에 누적
용도: 추세 그래프, Biweekly Before/After, 분기 회고 분석

## 2. 데이터 모델 v4

### 2.1 OKR 트리 (변경 없음, 단 weight 흐름 명확화)

pm_annual_goals → pm_company_kpis → pm_company_krs
↓ pm_kr_alignments (contribution_weight)
pm_personal_pips → pm_pip_sections → pm_pip_objectives → pm_pip_krs
↓
pm_initiatives (multi-owner via pm_initiative_owners)
↓
pm_tasks (assignee + backup)
S/A/B/C 평가기준 필드 제거. amendment는 KR 자체에 original_target, amendment_history (jsonb) 보존.

### 2.2 신규 마스터 테이블 (Round 5 결정 반영)

pm_signature_actives           -- Juanita "5 API materials"
id, code, trade_name, inci_name, function_category, positioning,
status (concept/development/secured/commercialized),
artifacts (jsonb): {concept, efficacy, safety, supply_spec, application, brochure}
owner_id, target_year, secured_at, archived_at
pm_required_sops               -- Dina "SOP 완성도"
id, code, title, category, required_for, is_critical,
current_status, current_version, linked_document_id,
owner_id, due_date, archived_at
pm_dev_pipelines               -- 개발 리드타임 (First proposal → 제품 확정)
id, customer_pipeline_id,
first_proposal_sent_at,      -- 시작점
formula_confirmed_at,        -- 종료점
product_id, dev_lead_days (generated)
customer_orders                -- 공급 리드타임 (PO 단위)
id, customer_id, po_number, customer_id,
order_received_at,
total_supply_lead_days (generated, 마지막 SKU delivered 기준)
customer_order_items           -- SKU 단위 병행 기록
id, order_id, product_id, sku, quantity,
production_started_at, shipped_at, delivered_at,
sku_supply_lead_days (generated)
sales_invoices                 -- KR1 매출 인식 (Invoice 시점)
id, customer_id, invoice_number, issue_date,
amount_idr, amount_krw,       -- 환율 보존
related_order_id, archived_at

### 2.3 체크리스트 (항목별 개별 evidence 설정)

pm_checklists
id, parent_type ('kr'|'initiative'|'task'|'standalone'),
parent_id, title, description,
default_assignee_id, default_backup_id,
recurrence ('none'|'daily'|'weekly'|'monthly'|'quarterly'),
recurrence_until, archived_at
pm_checklist_items
id, checklist_id, sort_order, title, description,
required_evidence_type ('none'|'text'|'photo'|'document'),  -- 항목별 개별
is_critical, default_assignee_id,
-- deep link to other modules
linked_module ('sample'|'base_formula'|'sop'|'halal_doc'|null),
linked_id,
archived_at
pm_checklist_completions
id, item_id, period_start, period_end (recurrence cycle),
completed_by, completed_at, evidence_url, notes,
rejected_by, rejected_at, reject_reason
Sunardi GMP 정책: 종이 작성 후 디지털 체크 시 required_evidence_type='none' (사진 X). 단순히 "오늘 모든 종이 문서 마무리 함" 클릭만. 분기 감사 시 종이 → 디지털 일치 검증은 별도 audit task.

### 2.4 진척 + 스냅샷

pm_progress_snapshots
id, snapshot_date,
entity_type ('company_kr'|'personal_kr'|'initiative'|'kpi'),
entity_id, progress_pct, current_value,
computation_source ('auto'|'manual_override'),
override_reason text,
override_by, override_at
pm_progress_overrides          -- manual override 활성 상태
id, entity_type, entity_id,
manual_value, reason, set_by, set_at, expires_at  -- 다음 cron에서 자동/유지 결정

### 2.5 알림 (in-app + email)

pm_notifications
id, recipient_id,
type ('overdue'|'at_risk'|'checklist_undone'|'approval_request'
|'biweekly_due'|'mention'|'comment'|'pip_amendment'),
entity_type, entity_id,
message, action_url,
sent_in_app_at, sent_email_at, read_at,
email_status ('queued'|'sent'|'failed'|'skipped')
cron schedule:
- 매일 02:00 KST: 진척 스냅샷
- 매일 03:00 KST: At Risk 판정 + 알림 큐잉
- 매일 09:00 KST: 데일리 스크럼 AI 생성 (활성 사용자만)
- 매일 18:00 KST: 미완료 일일 체크리스트 알림
- 격주 월요일 09:00: Biweekly Report 초안 자동 생성

### 2.6 Biweekly Report 자동 발행

pm_biweekly_reports
id, period_start, period_end, prepared_by_id,
status ('draft_ai_pending'|'draft_ready'|'in_review'|'published'),
-- AI 자동 채움 (트랙별)
exec_summary_revenue (jsonb),
exec_summary_product, exec_summary_certification,
exec_summary_sales, exec_summary_odm, exec_summary_growth,
-- AI 추출
top_risks (jsonb), decisions_needed (jsonb),
project_tracker_data (jsonb),  -- D-M-R-B-P-S 자동 채움
-- Sunny 수동
manual_narrative (markdown),
-- 발행
published_at, pdf_url, sent_to (jsonb)

## 3. 자동 진척 매핑 (최종)

| KR | source | 산출 |
|---|---|---|
| Juanita "20 base formulas" | base_formulas | COUNT WHERE status='completed' AND owner_id=Juanita AND year=2026 |
| Juanita "5 signature actives" | pm_signature_actives | COUNT WHERE status='secured' AND target_year=2026 |
| Aprelita "10 base formulas" | base_formulas | COUNT WHERE status='completed' AND owner_id=Aprelita |
| Aprelita "10 product library" | products | COUNT WHERE base_formula_id IS NOT NULL AND launched=true |
| Sunny "Revenue 3.0B" | sales_invoices | SUM(amount_idr) WHERE issue_date BETWEEN 2026 |
| Sunny "ODM 10 contracts" | customer_contracts | COUNT WHERE status='signed' |
| Sunny "개발 LT ≤2M" | pm_dev_pipelines | AVG(dev_lead_days) over confirmed in 2026 |
| Dina "공급 LT ≤2M" | customer_orders | AVG(total_supply_lead_days) |
| Sunny "COGS 개선" | products.cogs_history (versioning) | Δ% per product |
| Dina "SOP 완성도" | pm_required_sops | COUNT(approved) / COUNT(required) |
| Dina "ISO" | pm_checklists (KR attach) | item 완료율 (~95% 유지 모드) |
| Sunardi "GMP doc 99%" | pm_checklist_completions | recurrence=daily, 30d on-time rate |
| 모두 "외부 교육 ≥1" | talent_external_trainings | COUNT per talent per year |

## 4. UI/UX 결정 (Round 4)

### 4.1 라우트 구조 (영어 단일, PC 전용)

| Route | 우선 표시 |
|---|---|
| /pm | 모든 사용자에게 회사 KPI 트리 먼저 + 본인 KR 하이라이트 |
| /pm/kpi/[id] | KPI 상세 + 하위 KR 진척 + 영향받는 사용자 |
| /pm/pip/[talent_id]/[year] | 4-section PIP (KPI/CC/Growth/Core Value) |
| /pm/me | 오늘 task + 미완료 체크리스트 + 내 KR 상태 (단, /pm 다음 페이지) |
| /pm/checklist/daily | 일일 체크리스트 입력 (Sunardi 등) |
| /pm/sales | Customer Pipeline (L1-L4 Kanban) |
| /pm/sops | Required SOPs 마스터 |
| /pm/signature-actives | API 5 슬롯 카드뷰 |
| /pm/reports/biweekly | 자동 초안 검수 + 발행 |
| /pm/reports/quarterly | 분기 회고 |
| /pm/notifications | 시스템 알림함 |

### 4.2 첫 화면 와이어프레임 (/pm)

```text
┌─────────────────────────────────────────────────────────────┐
│ COSLAB Cosmetics Team — 2026 Annual Goal: Revenue 3.0B IDR  │
├─────────────────────────────────────────────────────────────┤
│ KPI-1 Sales [████████░░░░] 65%  (Owner: Sunny)              │
│   ├─ KR1.1 ODM 10 contracts  [██░░░░░░] 30%  3/10           │
│   ├─ KR1.2 Material 1.0B     [████░░░░] 50%  0.5B/1.0B      │
│   └─ KR1.3 Conversion 60%    [██████░░] 75%                 │
│                                                              │
│ KPI-2 Ready-to-Start  [██████░░░░] 60%  (Owner: Sunny)      │
│   ├─ KR2.1 Dev LT ≤2M        [█████████] 90%  avg 55d       │
│   ├─ KR2.2 Supply LT ≤2M     [██████░░░] 70%  avg 65d (Dina)│
│   ├─ KR2.3 Signature 5       [██░░░░░░] 20%  1/5 (Juanita)  │
│   └─ ...                                                     │
│                                                              │
│ KPI-3 QMS  [████████░░] 75%                                  │
│                                                              │
│ ⭐ MY CONTRIBUTIONS (logged-in user highlight)              │
│   You contribute to: KR2.3 (signature actives, 60% weight)  │
│                       → 1/5 secured. Open detail ›          │
└─────────────────────────────────────────────────────────────┘
4.3 알림 UX (Round 5 결정)
시스템 내부 알림함: /pm/notifications + 헤더 종 아이콘 배지
이메일: 매일 09:00 일괄 다이제스트 (긴급 At Risk만 즉시 발송)
알림 종류: 4가지 (Round 5 user 답변 기반)
	1. 본인 task/체크리스트 마감·오버듀 (in-app + email immediate)
	2. KR At Risk 경고 (in-app + email immediate, owner+manager)
	3. Manager 승인·Biweekly·멘션 (in-app + email digest, Sunny)
	4. 일일·주간 체크리스트 미완료 (in-app + 18시 email reminder)
```

## 5. AI 기능 (Round 5 결정 — 3종 활성)

### 5.1 Biweekly Report 자동 초안 (격주 월요일 09:00)

입력: 직전 14일 PM 데이터
- sales_invoices 변동
- samples/base_formulas/products 신규·status 변화
- halal_documents/bpom_submissions 이벤트
- customer_pipeline L1-L4 변동
- pm_required_sops approved 수 변화
- pm_progress_snapshots Δ%
출력 (Claude Haiku 4.5 + prompt cache):
- 트랙별 narrative summary (Markdown)
- Top 3 risks (At Risk + Off Track 자동 추출)
- Decisions Needed 후보 5건
- D-M-R-B-P-S 프로젝트 트래커 자동 채움
Sunny 작업: 검수 30분 → manual_narrative 추가 → Publish → PDF + email 자동 발송

### 5.2 데일리 스크럼 (/pm/me 진입 시 1일 1회 캐시)

Yesterday: Sample S-2026-042 confirmed, BOM B-01 updated
Today: Stability ST-018 1-month measure, 3 new sample DPI tests
Blockers: Raw material X price pending → procurement awaiting
At Risk: KR "20 base formulas" 60% with 18 days left (need 9 more)

### 5.3 분기 회고 + KR 둔화 자동 플래그

- 분기말 자동 생성: KPI/Objective/KR 달성률 + AI 코멘트
- 매주 cron: KR 진척이 14일간 +0% 인 항목 → Owner에게 push
비활성: PIP/KR 작성 시 SMART 검토 (Round 5에서 제외).

## 6. 마이그레이션 (Round 6 결정)

### 6.1 스코프

## 1. 과거 Biweekly Reports (v1.0~v1.2 docx) → PDF 보관 + 메타데이터(기간, Revenue, KR 진척) 추출하여 pm_biweekly_reports.legacy=true로 import

## 2. Notion DB export (To-do list, Done DB, 2026 management) → CSV → pm_tasks import (legacy_source='notion')

## 3. 현재 PIP docx (5명) → 신규 pm_personal_pips 레코드로 변환 (Sunny 검토 후 active)

## 4. Ali PIP → Dina 임시 PIP 자동 매핑 (Sunny 승인)

### 6.2 코드

scripts/pm-migration/:
- import-biweekly-docx.ts (DOCX 파싱 + LLM 메타 추출)
- import-notion-csv.ts
- import-pip-docx.ts
- seed-required-sops.ts (Biweekly에 등장한 23종 SOP backfill)
- seed-signature-actives.ts (EGF 1건)

## 7. 코슬랩-허브 영향 분석

| 영향 영역 | 변경 | 리스크 | 완화 |
|---|---|---|---|
| base_formulas 스키마 | owner_id, status, completion_date 컬럼 추가 (NOT NULL 회피, 기본값 nullable) | 기존 row null 처리 | 기존 UI는 변경 없음, PM이 null 허용 |
| products 스키마 | base_formula_id FK, launched boolean | 기존 product 레거시 | backfill 스크립트 + 누락은 "legacy" 라벨 |
| samples 스키마 | 변경 없음 (PM은 base_formulas 기준) | - | - |
| documents 모듈 | linked_pm_required_sop_id 컬럼 추가 (선택) | - | - |
| talent 모듈 | resigned 상태 trigger 추가 → PM 알림 | trigger 충돌 | 기존 Talent trigger 보존, append-only |
| audit_log | PM 활동 통합 | 볼륨 증가 | partition by month |
| RLS 정책 | PM 테이블은 모두 authenticated.full_access | 민감정보 노출 가능성 | 전체 공개 결정에 따라 단순화 |
| Cron jobs | 5개 추가 (Supabase Edge Function 또는 외부 scheduler) | 부하 | 시간대 분산 (02-04시 새벽) |
| Trash 통합 | PM 9개 신규 테이블 모두 archived_at 필드 + /trash aggregator 등록 | - | 기존 패턴 재사용 |

## 8. Sprint 일정 (15.5일, MVP)

Sprint 1 (8.5d) — Core OKR + Checklist + Master
| Day | 작업 |
|---|---|
| 1-2 | 마이그레이션: 9개 PM 테이블 + base_formulas/products 컬럼 확장 |

3	OKR CRUD API (annual_goals/kpis/krs/pips/objectives/personal_krs)
4	Alignment 테이블 + 본인 초안 → manager 잠금 워크플로우
5	/pm 트리뷰 + 진척률 게이지 UI
6	체크리스트 entity (KR/Initiative/Task attach + recurrence)
7	/pm/sops, /pm/signature-actives 마스터 페이지
8	/pm/pip/[id]/[year] 4-section + amendment

### 8.5	Sprint 1 통합 테스트 + 5명 PIP migration

Sprint 2 (7d) — Auto Progress + Pipeline + AI Lite
| Day | 작업 |
|---|---|
| 9-10 | 도메인 자동 진척 cron + manual_override + audit |

11	Stage-Gate (D-M-R-B-P-S) UI + 조건자동+Owner 확인
12	Customer Pipeline (L1-L4) + dev/supply lead time 측정점
13	/pm/me 통합 + deep link to other modules
14	알림 큐 + in-app 종 + 이메일 daily digest
15	Biweekly Report 자동 초안 + AI 데일리 스크럼

### 15.5	Migration: 과거 docx + Notion CSV import

Sprint 3 (PM-9~12, 추후 별도) — 분기 회고, Trash 통합, Multi-owner

## 9. 도입 운영 모드 (Round 6)

첫 30일 — 점진·코칭:
- D+0: 5명 PIP migration + Sunny 임시 PIP for Dina
- D+1~7: Sunny가 매일 /pm 사용, 팀원 자율 사용 권장
- D+7: 첫 주간 회고 (PM 사용 친구 공유)
- D+14: 첫 자동 Biweekly Report 발행 (manual narrative 비교)
- D+21: At Risk 알림 활성화
- D+30: 정주 도입 결정 — 지속 사용 vs 단계 후퇴 vs 보강 결정
유의사항:
- 30일 동안 KR 미완료 알림 → 친절한 톤, 강제 X
- 사용 안 하는 항목은 자유, 단 매주 Sunny가 "PM에 들어와서 확인" 권유
- 30일 후 사용률 50%↑ 미만이면 UX 재설계 → 강제도 ↑ 검토

## 10. 즉시 진행 가능한 다음 단계

## 1. base_formulas / products / customer_orders / sales_invoices / customer_contracts 스키마 확장 마이그레이션 작성 (Day 1-2)

## 2. PM 9개 테이블 마이그레이션 작성

## 3. Required SOPs 23종 정의 (Sunny + Dina와 confirm 필요)

## 4. 5명 PIP migration 스크립트 + Dina 임시 PIP 템플릿 (Sunny가 Ali 기반 + KPI-3 매핑)

## 5. EGF Transfersome → pm_signature_actives에 SIG-001로 seed

<!-- SOURCE_FILE_4_END: COSLAB-Hub-PM-Module-기획서-v4.md -->

---


<!-- SOURCE_FILE_5_BEGIN: COSLAB-HUB-SCHEMA.md -->

# SOURCE FILE 5: `COSLAB-HUB-SCHEMA.md`

# COSLAB Hub Supabase Database Schema Reference

> Generated from 79 migration files (2026-03-30 to 2026-05-07) + base scripts
> for complete SQL/query authoring without reading individual migrations.

---

## 1. SHARED CORE (No Prefix)

### materials
**Created**: migration-002-unified-schema.sql  
**Purpose**: Raw material master data (ODM source of truth)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT -- e.g. PCRM-0004
  - name TEXT NOT NULL
  - manufacturer TEXT
  - supplier TEXT
  - price_per_kg NUMERIC
  - appearance TEXT -- Liquid/Powder/Solid/Paste/Oil
  - decision_date DATE
  - memo TEXT
  - active BOOLEAN DEFAULT true
  - inci_components JSONB DEFAULT '[]' -- [{inci_name, percentage, function, cas_no}]
  - is_halal_positive_list BOOLEAN DEFAULT false (added 002-unified)
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - is_active BOOLEAN DEFAULT true (Phase B soft-delete)

**Foreign Keys**: None  
**Key Indexes**: idx_mat_docs_material, idx_mat_docs_expiry  
**Triggers**: trg_materials_updated (update_updated_at)  
**RLS**: Enabled; auth_all  
**Modified by later migrations**:
  - 20260411_base_formulas_enhance — added is_halal_positive_list
  - 20260506a_unified_trash_phase_b — added is_active (defensive)

---

### formulations
**Created**: migration-002-unified-schema.sql  
**Purpose**: Formula master (versions embedded in JSONB)  
**Primary Key**: id (UUID cast to TEXT in foreign key references)  
**Columns**:
  - id UUID PRIMARY KEY (note: many migrations treat as TEXT TEXT)
  - product_code TEXT
  - product_name TEXT NOT NULL
  - category TEXT -- Skin/Toner, Cream, Sun Care, etc.
  - batch_size_g NUMERIC
  - density NUMERIC DEFAULT 1.0
  - status TEXT DEFAULT 'Draft' -- Draft / Confirmed / Discontinued
  - current_version INT DEFAULT 1
  - sales_visible BOOLEAN DEFAULT false
  - memo TEXT
  - versions JSONB DEFAULT '[]' -- [{version, date, change_reason, memo, ingredients: [...]}]
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - formula_code TEXT UNIQUE (added 20260331_phase3a)
  - description TEXT (added 20260331_phase3a)
  - client_id UUID (added 20260331_phase3a)
  - created_by TEXT (added 20260331_phase3a)
  - primary_category TEXT (added 20260331_phase3a)
  - secondary_category TEXT (added 20260331_phase3a)
  - ifra_category TEXT (added 20260331_phase3a)
  - texture_profile TEXT (added 20260331_phase3a)
  - application_area TEXT[] (added 20260331_phase3a)
  - usage_type TEXT (added 20260331_phase3a)
  - target_customer TEXT[] (added 20260331_phase3a)
  - tags TEXT[] (added 20260331_phase3a)
  - renewal_group TEXT (added 20260331_phase3a)
  - is_active BOOLEAN DEFAULT true (added 20260331_phase3a)
  - superseded_by TEXT (added 20260331_phase3a)
  - superseded_at TIMESTAMPTZ (added 20260331_phase3a)
  - current_snapshot_id TEXT REFERENCES formula_snapshots(id) ON DELETE SET NULL (added 20260415a)
  - locked_at TIMESTAMPTZ (added 20260414c)
  - locked_by TEXT (added 20260414c)
  - lock_reason TEXT (added 20260414c)

**Foreign Keys**:
  - client_id → clients(id)
  - current_snapshot_id → formula_snapshots(id) ON DELETE SET NULL

**Key Indexes**: idx_vpins_formulation, idx_formulations_current_snapshot  
**Triggers**: trg_formulations_updated (update_updated_at)  
**RLS**: Enabled; auth_all  
**Modified by later migrations**: Multiple (see inline)

---

### formulations → products relationship
**Note**: Products table wraps formulations (20260505a makes this official). Each formulation can have 1+ product SKU rows.

---

### packaging
**Created**: migration-002-unified-schema.sql  
**Purpose**: Packaging material master  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT
  - name TEXT NOT NULL
  - category TEXT -- Primary, Secondary, Other
  - type TEXT -- Bottle, Cap, Box, Label, Tube, Pump, Jar, Other
  - material TEXT -- PET, Glass, Paper, PP, PE, HDPE, Aluminum, Other
  - volume NUMERIC
  - supplier TEXT
  - price NUMERIC
  - loss_rate_pct NUMERIC
  - decision_date DATE
  - memo TEXT
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - is_active BOOLEAN DEFAULT true (Phase B soft-delete)

**Foreign Keys**: None  
**Key Indexes**: idx_packaging_is_active (Phase B)  
**Triggers**: trg_packaging_updated  
**RLS**: Enabled; auth_all

---

### printing
**Created**: migration-002-unified-schema.sql  
**Purpose**: Printing/label service master  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT
  - name TEXT NOT NULL
  - source TEXT -- Outsource, InHouse, Other
  - printer_company TEXT
  - price NUMERIC
  - loss_rate_pct NUMERIC
  - memo TEXT
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - is_active BOOLEAN DEFAULT true (Phase B soft-delete)

**Foreign Keys**: None  
**Triggers**: (none in base; soft-delete may need custom logic)  
**RLS**: Enabled; auth_all

---

### processes
**Created**: migration-002-unified-schema.sql  
**Purpose**: Manufacturing process master  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT
  - process_name TEXT NOT NULL
  - unit_price NUMERIC
  - price_unit TEXT -- per_hour, per_unit
  - cost_type TEXT -- labor, consumable
  - decision_date DATE
  - memo TEXT
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: None  
**RLS**: Enabled; auth_all

---

### quality_tests
**Created**: migration-002-unified-schema.sql  
**Purpose**: QC test master (Microbiology, Physical, Chemical, Stability)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT
  - test_name TEXT NOT NULL
  - category TEXT -- Microbiology / Physical / Chemical / Stability
  - unit_price NUMERIC
  - decision_date DATE
  - memo TEXT
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: None  
**RLS**: Enabled; auth_all

---

### process_flow_templates
**Created**: migration-002-unified-schema.sql  
**Purpose**: Reusable manufacturing process flow for a formulation  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT
  - name TEXT NOT NULL
  - formulation_id UUID REFERENCES formulations(id) ON DELETE CASCADE
  - product_code TEXT
  - product_name TEXT
  - memo TEXT
  - phases JSONB DEFAULT '[]' -- [{phase, phase_name, steps: [{id, step_name, process_id, params}]}]
  - phase_params JSONB DEFAULT '{}'
  - post_steps JSONB DEFAULT '[]'
  - pre_checks JSONB DEFAULT '[]'
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: formulation_id → formulations(id) ON DELETE CASCADE  
**RLS**: Enabled; auth_all

---

### inci_dictionary
**Created**: setup-db.sql  
**Purpose**: INCI name reference (COSING or equivalent)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT uuid_generate_v4()
  - inci_name TEXT NOT NULL UNIQUE
  - common_name_ko TEXT
  - common_name_en TEXT
  - cas_number TEXT
  - ec_number TEXT
  - function TEXT[]
  - description TEXT
  - source TEXT DEFAULT 'cosing'

**Indexes**: idx_inci_name_trgm (GIN on pg_trgm)  
**RLS**: Enabled; read-only for authenticated

---

### hub_accounts
**Created**: migration-002-unified-schema.sql  
**Purpose**: Local authentication (transient; to be replaced by auth.users)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - email TEXT NOT NULL UNIQUE
  - password_hash TEXT NOT NULL
  - display_name TEXT
  - role TEXT DEFAULT 'viewer'
  - is_active BOOLEAN DEFAULT true
  - created_at TIMESTAMPTZ DEFAULT now()
  - failed_attempts INT DEFAULT 0 (added 20260401_security_hardening)
  - locked_until TIMESTAMPTZ (added 20260401_security_hardening)
  - password_changed_at TIMESTAMPTZ (added 20260401_security_hardening)
  - must_change_password BOOLEAN DEFAULT false (added 20260401_security_hardening)
  - onboarding_completed_at TIMESTAMPTZ (added 20260429_hub_accounts_onboarding)

**Triggers**: None  
**RLS**: Enabled; auth_all  
**Modified by later migrations**:
  - 20260401_security_hardening — lockout + policy columns
  - 20260429_hub_accounts_onboarding — onboarding state

---

### auth_audit_logs
**Created**: 20260401_security_hardening.sql  
**Purpose**: Audit trail for authentication & authorization events  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID DEFAULT gen_random_uuid() PRIMARY KEY
  - event_type TEXT NOT NULL -- login_success / login_failure / logout / password_change / account_create / account_update / account_lock
  - user_email TEXT
  - user_id UUID
  - ip_address TEXT
  - user_agent TEXT
  - details JSONB
  - created_at TIMESTAMPTZ DEFAULT now()
  - resource_type TEXT (added 20260403_audit_trail_expansion)
  - resource_id TEXT (added 20260403_audit_trail_expansion)
  - changes JSONB (added 20260414a) -- {field_name: {from: old_val, to: new_val}}
  - change_reason TEXT (added 20260414a)

**Key Indexes**:
  - idx_auth_audit_event (event_type, created_at DESC)
  - idx_auth_audit_user (user_email, created_at DESC)
  - idx_audit_resource (resource_type, resource_id)
  - idx_audit_event_date (event_type, created_at DESC)
  - idx_audit_resource_changes (resource_type, resource_id, created_at DESC) WHERE changes IS NOT NULL

**RLS**: None explicitly enabled in migrations

---

### profiles
**Created**: setup-db.sql  
**Purpose**: User profile (from Knowledge Hub integration)  
**Primary Key**: id (UUID REFERENCES auth.users(id) ON DELETE CASCADE)  
**Columns**:
  - id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE
  - email TEXT NOT NULL
  - full_name TEXT
  - role TEXT NOT NULL DEFAULT 'member' -- admin / manager / member / viewer
  - department TEXT
  - avatar_url TEXT
  - created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  - updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

**Triggers**: trg_profiles_updated  
**RLS**: Enabled; auth_all

---

### brands (presumed from migration-002)
**Status**: ALTER TABLE referenced in migration-002 but base CREATE not found in reviewed files.  
**Likely structure**:
  - id UUID PRIMARY KEY
  - name TEXT NOT NULL
  - created_at TIMESTAMPTZ DEFAULT now() (added 002-unified)
  - Other fields TBD from Knowledge Hub

---

### base_formulas (referenced, enhanced 20260411)
**Status**: Base CREATE not found in reviewed files; ALTER TABLE exists.  
**From 20260411_base_formulas_enhance.sql**, these columns are added:
  - formula_code TEXT
  - versions JSONB DEFAULT '[]'::jsonb
  - current_version INTEGER DEFAULT 0
  - batch_size_g NUMERIC
  - density NUMERIC
  - ph_min NUMERIC
  - ph_max NUMERIC
  - viscosity_min NUMERIC
  - viscosity_max NUMERIC
  - shelf_life_months INTEGER
  - classification JSONB DEFAULT '{}'::jsonb
  - started_at DATE
  - target_date DATE
  - description TEXT
  - memo TEXT

---

### clients (core, expanded later)
**Created**: 20260331_phase3c_operations.sql  
**Purpose**: B2B client/customer master (separate from hub internal contacts)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - client_code TEXT UNIQUE
  - company_name TEXT NOT NULL
  - contact_name TEXT
  - phone TEXT
  - email TEXT
  - website TEXT
  - country TEXT
  - city TEXT
  - address_line1 TEXT
  - address_line2 TEXT
  - zip_code TEXT
  - business_type TEXT
  - market_focus TEXT[]
  - notes TEXT
  - status TEXT DEFAULT 'active'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT
  - locked BOOLEAN DEFAULT false (Phase 2 change control)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Triggers**: trg_clients_updated  
**RLS**: Enabled; auth_all

---

### suppliers
**Created**: 20260331_phase3c_operations.sql  
**Purpose**: Vendor/supplier master  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - supplier_code TEXT UNIQUE
  - supplier_name TEXT NOT NULL
  - contact_name TEXT
  - phone TEXT
  - email TEXT
  - website TEXT
  - country TEXT
  - city TEXT
  - address_line1 TEXT
  - address_line2 TEXT
  - zip_code TEXT
  - source TEXT DEFAULT 'custom'
  - payment_terms TEXT
  - lead_time_days INT
  - min_order_qty NUMERIC
  - min_order_unit TEXT
  - currency TEXT DEFAULT 'IDR'
  - notes TEXT
  - status TEXT DEFAULT 'active' -- soft-delete: 'inactive' (Phase B)
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT
  - locked BOOLEAN DEFAULT false (Phase 2)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Triggers**: trg_suppliers_updated  
**RLS**: Enabled; auth_all  
**Soft-Delete Pattern**: status='inactive' (Phase B unified trash)

---

### products
**Created**: setup-db.sql (Knowledge Hub); reimported & enhanced 20260505a (Phase B-1)  
**Purpose**: First-class entity wrapping formulations; SKU management  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - name_en TEXT
  - category TEXT
  - subcategory TEXT
  - form_type TEXT
  - description TEXT
  - key_features TEXT[]
  - target_market TEXT[]
  - moq INT
  - price_range_min REAL
  - price_range_max REAL
  - status TEXT NOT NULL DEFAULT 'active' -- active / discontinued / development
  - created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  - updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  - fco_code TEXT (added 002-unified for CosReg)
  - formulation_id UUID (added 002-unified; formally FK 20260505a)
  - synced_at TIMESTAMPTZ (added 002-unified)
  - product_code TEXT (added 20260505a)
  - product_name TEXT (added 20260505a)
  - packaging_size TEXT (added 20260505a)
  - product_version INT NOT NULL DEFAULT 1 (added 20260505a)
  - formulation_version INT (added 20260505a)
  - memo TEXT (added 20260505a)
  - edit_history JSONB DEFAULT '[]' (added 20260505a)
  - locked BOOLEAN DEFAULT false (added 20260505a)
  - locked_at TIMESTAMPTZ (added 20260505a)
  - lock_reason TEXT (added 20260505a)
  - locked_by TEXT (added 20260505a)
  - created_by TEXT (added 20260505a)
  - updated_by TEXT (added 20260505a)

**Foreign Keys**: formulation_id → formulations(id) (added 20260505a)  
**Key Indexes**: idx_product_code_partial (UNIQUE WHERE status != 'discontinued')  
**Triggers**: trg_products_updated  
**RLS**: Enabled; auth_all  
**Notable**: 20260505a backfilled 1 default product per formulation; normalized placeholder codes to NULL

---

## 2. MATERIALS / INVENTORY

### sample_materials
**Created**: 20260410_sample_materials.sql  
**Purpose**: Pre-production sample-specific raw materials  
**Primary Key**: id (TEXT)  
**Columns**:
  - id TEXT PRIMARY KEY
  - sample_code TEXT UNIQUE -- SMP-001 format
  - name TEXT NOT NULL
  - manufacturer TEXT
  - supplier TEXT
  - price_per_kg NUMERIC
  - appearance TEXT
  - moq TEXT
  - lead_time TEXT
  - memo TEXT
  - active BOOLEAN DEFAULT true
  - inci_components JSONB DEFAULT '[]'
  - is_halal_positive_list BOOLEAN DEFAULT false
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**RLS**: Enabled; auth_all

---

### inventory_lots
**Created**: 20260331_phase3c_operations.sql  
**Purpose**: Raw material lot/batch tracking  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - lot_number TEXT NOT NULL
  - supplier_lot TEXT
  - material_id TEXT REFERENCES materials(id) ON DELETE SET NULL
  - supplier_id UUID REFERENCES suppliers(id) ON DELETE SET NULL
  - client_id UUID REFERENCES clients(id) ON DELETE SET NULL
  - initial_amount NUMERIC NOT NULL
  - current_amount NUMERIC NOT NULL
  - unit TEXT DEFAULT 'kg'
  - received_date DATE
  - expiration_date DATE
  - manufacturing_date DATE
  - status TEXT DEFAULT 'available'
  - storage_location TEXT
  - coa_doc_id UUID
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Foreign Keys**:
  - material_id → materials(id)
  - supplier_id → suppliers(id)
  - client_id → clients(id)

**Triggers**: trg_inventory_lots_updated  
**RLS**: Enabled; auth_all

---

### packaging_inventory
**Created**: 20260331_phase3c_operations.sql  
**Purpose**: Packaging material stock  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - packaging_id TEXT REFERENCES packaging(id) ON DELETE SET NULL
  - sku TEXT
  - client_id UUID REFERENCES clients(id) ON DELETE SET NULL
  - quantity INT NOT NULL
  - available_quantity INT NOT NULL
  - received_date DATE
  - status TEXT DEFAULT 'available'
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: packaging_id → packaging(id), client_id → clients(id)  
**RLS**: Enabled; auth_all

---

### material_documents
**Created**: migration-002-unified-schema.sql  
**Purpose**: Certs/MSDS for raw materials (CoA, MSDS, Halal, IFRA, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - material_id UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE
  - doc_type TEXT NOT NULL -- CoA / MSDS / HalalCert / Specification / Allergen / BSE_TSE / IFRA / etc.
  - custom_label TEXT
  - file_url TEXT
  - file_hash TEXT
  - version INT DEFAULT 1
  - is_current BOOLEAN DEFAULT true
  - expiry_date DATE
  - issue_date DATE
  - issuer TEXT
  - extracted_data JSONB DEFAULT '{}'
  - uploaded_at TIMESTAMPTZ DEFAULT now()
  - uploaded_by TEXT

**Indexes**: idx_mat_docs_material, idx_mat_docs_expiry (WHERE expiry_date IS NOT NULL)  
**RLS**: Enabled; auth_all

---

### sample_material_documents
**Created**: 20260413b_sample_material_documents.sql  
**Purpose**: Same structure as material_documents but for sample_materials  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - material_id TEXT NOT NULL -- references sample_materials.id (TEXT PK)
  - doc_type TEXT NOT NULL
  - custom_label TEXT
  - file_url TEXT
  - file_hash TEXT
  - version INT DEFAULT 1
  - is_current BOOLEAN DEFAULT true
  - expiry_date DATE
  - issue_date DATE
  - issuer TEXT
  - extracted_data JSONB DEFAULT '{}'
  - uploaded_at TIMESTAMPTZ DEFAULT now()
  - uploaded_by TEXT

**Indexes**: idx_smd_material, idx_smd_current (material_id, doc_type) WHERE is_current  
**RLS**: Enabled; auth_all

---

### supplier_materials
**Created**: 20260331_phase3c_operations.sql  
**Purpose**: Material × Supplier junction with pricing  
**Primary Key**: id (UUID); UNIQUE(supplier_id, material_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - supplier_id UUID REFERENCES suppliers(id) ON DELETE CASCADE
  - material_id TEXT REFERENCES materials(id) ON DELETE CASCADE
  - supplier_material_code TEXT
  - price NUMERIC
  - currency TEXT DEFAULT 'IDR'
  - lead_time_days INT
  - moq NUMERIC
  - moq_unit TEXT
  - is_primary BOOLEAN DEFAULT false
  - last_order_date DATE
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - UNIQUE(supplier_id, material_id)

**Foreign Keys**: supplier_id → suppliers(id), material_id → materials(id)  
**RLS**: Enabled; auth_all

---

## 3. FORMULATIONS SUPPORT

### formula_code_sequences
**Created**: 20260331_phase3a_schema.sql  
**Purpose**: Auto-generate formula codes (SK-SRM-001)  
**Primary Key**: id (UUID); UNIQUE(user_initials, category_code)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - user_initials TEXT NOT NULL
  - category_code TEXT NOT NULL
  - next_number INT DEFAULT 1
  - created_at TIMESTAMPTZ DEFAULT now()
  - UNIQUE(user_initials, category_code)

**RLS**: Enabled; auth_all

---

### formula_specs
**Created**: 20260331_phase3a_schema.sql  
**Purpose**: Specs & feasibility for a formulation version  
**Primary Key**: id (UUID); UNIQUE(formulation_id, formulation_version)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - formulation_id TEXT REFERENCES formulations(id) ON DELETE CASCADE
  - formulation_version INT
  - appearance_target TEXT
  - color_target TEXT
  - odor_target TEXT
  - emulsion_type TEXT
  - viscosity_min NUMERIC
  - viscosity_max NUMERIC
  - viscosity_unit TEXT DEFAULT 'cP'
  - viscosity_method TEXT
  - viscosity_na BOOLEAN DEFAULT false
  - ph_min NUMERIC
  - ph_max NUMERIC
  - ph_na BOOLEAN DEFAULT false
  - specific_gravity_min NUMERIC
  - specific_gravity_max NUMERIC
  - sg_na BOOLEAN DEFAULT false
  - yield_assumption_pct NUMERIC
  - manufacturing_method TEXT[]
  - packaging_compatibility TEXT[]
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT
  - UNIQUE(formulation_id, formulation_version)

**Triggers**: trg_formula_specs_updated  
**RLS**: Enabled; auth_all

---

### formula_snapshots
**Created**: 20260415a_formula_snapshots.sql  
**Purpose**: Price-locked snapshot of formula composition + costs  
**Primary Key**: id (TEXT); Composite idx (formulation_id, formula_version)  
**Columns**:
  - id TEXT PRIMARY KEY
  - formulation_id TEXT NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - formula_version INT NOT NULL
  - composition JSONB NOT NULL -- phases/ingredients at snapshot time
  - cost_basis JSONB -- cost_basis row snapshot (labor, overhead, margin)
  - material_prices JSONB NOT NULL DEFAULT '{}' -- {material_id: {price_per_kg, supplier, name, code}}
  - packaging_prices JSONB NOT NULL DEFAULT '{}' -- {packaging_id: {price, supplier, name, code}}
  - total_cost_per_kg NUMERIC
  - total_cost_per_unit NUMERIC
  - snapshot_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  - snapshot_by TEXT
  - snapshot_reason TEXT NOT NULL -- confirmed / quote_issue / dip_publish / manual / renew
  - notes TEXT
  - CHECK (snapshot_reason IN ('confirmed', 'quote_issue', 'dip_publish', 'manual', 'renew'))

**Key Indexes**:
  - idx_formula_snapshots_formulation
  - idx_formula_snapshots_formulation_version
  - idx_formula_snapshots_snapshot_at

**RLS**: Enabled (RLS added 20260415d_formula_snapshots_rls)

---

### formulations_lock_metadata
**Created**: 20260414c_formulations_lock_metadata.sql  
**Purpose**: Align formulations lock columns with unified change control  
**Note**: These columns are added to formulations table directly:
  - locked_at TIMESTAMPTZ
  - locked_by TEXT
  - lock_reason TEXT
  - locked (created 002-unified, already exists)

---

### version_pins
**Created**: migration-002-unified-schema.sql  
**Purpose**: Master linkage: formulation → BOM + Safety + Package + Process  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - formulation_id UUID NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - pin_version TEXT NOT NULL
  - formula_version INT
  - bom_id UUID REFERENCES boms(id) ON DELETE SET NULL
  - safety_report_id UUID
  - safety_report_version INT
  - package_review_id UUID
  - package_review_version INT
  - process_flow_id UUID REFERENCES process_flow_templates(id) ON DELETE SET NULL
  - status TEXT DEFAULT 'draft' -- draft / locked / submitted
  - locked_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Indexes**: idx_vpins_formulation  
**RLS**: Enabled; auth_all  
**Note**: safety_report_id, package_review_id are forward references (added later)

---

### product_versions
**Created**: 20260409c_product_versions.sql  
**Purpose**: Manifest of complete product state across formula+package+label+BOM  
**Primary Key**: id (UUID); UNIQUE(formulation_id, product_version)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - formulation_id TEXT NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - product_version INT NOT NULL
  - formula_version INT NOT NULL
  - package_version INT DEFAULT 1
  - label_version TEXT DEFAULT 'v0.1'
  - trigger TEXT NOT NULL DEFAULT 'initial' -- initial / formula / package / label / bom_only / cost_only
  - change_reason TEXT
  - notes TEXT
  - created_at TIMESTAMPTZ NOT NULL DEFAULT now()
  - created_by TEXT
  - UNIQUE(formulation_id, product_version)

**Indexes**: idx_product_versions_formulation (formulation_id, product_version DESC)  
**Backfilled**: One row per existing formulation (20260409c)

---

## 4. STABILITY & TESTING

### stability_tests
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Master test record (microbiology, physical, chemical, performance)  
**Primary Key**: id (UUID); UNIQUE(test_code)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - test_code TEXT UNIQUE
  - formulation_id TEXT REFERENCES formulations(id) ON DELETE SET NULL
  - formulation_version INT
  - batch_reference TEXT
  - laboratory_analyst TEXT
  - test_date DATE
  - comments TEXT
  - status TEXT DEFAULT 'draft' -- draft / in_progress / completed
  - started_at TIMESTAMPTZ
  - completed_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT
  - is_active BOOLEAN DEFAULT true (added 20260506a_unified_trash_phase_b)
  - locked BOOLEAN DEFAULT false (Phase 2 change control)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Triggers**: trg_stability_tests_updated  
**Indexes**: idx_stability_tests_is_active  
**RLS**: Enabled; auth_all

---

### stability_condition_sets
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Test conditions (temperature, humidity, light, packaging)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - stability_test_id UUID REFERENCES stability_tests(id) ON DELETE CASCADE
  - set_number INT
  - sample_id TEXT
  - temperature TEXT
  - humidity TEXT
  - light_condition TEXT
  - packaging_type TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: stability_test_id → stability_tests(id)  
**RLS**: Enabled; auth_all

---

### stability_intervals
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Measurement timepoints (T0, 1W, 3M, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - condition_set_id UUID REFERENCES stability_condition_sets(id) ON DELETE CASCADE
  - interval_name TEXT NOT NULL
  - days_from_start INT NOT NULL
  - sort_order INT
  - created_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: condition_set_id → stability_condition_sets(id)  
**RLS**: Enabled; auth_all

---

### stability_parameter_master
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Reference list of testable parameters (~25 predefined, seeded)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - category TEXT NOT NULL -- Microbiological / Physical / Chemical / Performance / Compatibility
  - unit TEXT
  - description TEXT
  - is_active BOOLEAN DEFAULT true
  - sort_order INT
  - created_at TIMESTAMPTZ DEFAULT now()

**Seed Data** (25 predefined parameters):
  - Yeast and mold count, Total plate count, Preservative efficacy, ...
  - pH, Viscosity, Specific gravity, Phase separation, ...
  - Active ingredient assay, Preservative content, ...
  - Container/cap/label compatibility, Pump function, ...

**RLS**: Enabled; auth_all

---

### stability_test_results
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Individual result entry (parameter × interval)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - interval_id UUID REFERENCES stability_intervals(id) ON DELETE CASCADE
  - parameter_id UUID REFERENCES stability_parameter_master(id) ON DELETE SET NULL
  - result_value TEXT
  - result_status TEXT DEFAULT 'pending' -- pending / pass / fail / oos / conditional
  - notes TEXT
  - recorded_at TIMESTAMPTZ
  - recorded_by TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - locked BOOLEAN DEFAULT false (Phase 2 change control)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Foreign Keys**: interval_id → stability_intervals, parameter_id → stability_parameter_master  
**RLS**: Enabled; auth_all

---

### stability_test_templates
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Reusable test protocol template by product category  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - description TEXT
  - product_category TEXT
  - condition_sets JSONB
  - intervals JSONB
  - parameter_ids UUID[]
  - created_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**RLS**: Enabled; auth_all

---

## 5. SAMPLES & MANUFACTURING

### sample_manufacturing
**Created**: 20260331_phase3b_stability.sql; enhanced 20260410, 20260428  
**Purpose**: Sample batch record from R&D trials  
**Primary Key**: id (UUID); UNIQUE(sample_code)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - sample_code TEXT UNIQUE
  - formulation_id TEXT REFERENCES formulations(id) ON DELETE SET NULL
  - formulation_version INT
  - batch_size_g NUMERIC
  - manufacturing_date DATE
  - manufactured_by TEXT
  - manufacturing_method TEXT[]
  - equipment_used TEXT[]
  - room_temperature NUMERIC
  - room_humidity NUMERIC
  - phase_records JSONB
  - yield_actual_g NUMERIC
  - yield_pct NUMERIC
  - appearance TEXT
  - ph_value NUMERIC
  - viscosity_value NUMERIC
  - specific_gravity NUMERIC
  - appearance_spec TEXT (added 20260428)
  - ph_spec TEXT (added 20260428)
  - viscosity_spec TEXT (added 20260428)
  - specific_gravity_spec TEXT (added 20260428)
  - result TEXT DEFAULT 'pending'
  - result_notes TEXT
  - approved_by TEXT
  - approved_at TIMESTAMPTZ
  - status TEXT DEFAULT 'draft'
  - stability_test_id UUID
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT
  - is_active BOOLEAN DEFAULT true (added 20260506a_unified_trash_phase_b)

**Triggers**: trg_sample_manufacturing_updated  
**RLS**: Enabled; auth_all

---

### sample_qc_specs
**Created**: 20260428_sample_qc_specs.sql  
**Purpose**: Initial evaluation specifications (target ranges for appearance, pH, viscosity, SG)  
**Primary Key**: (presumably maps to sample_manufacturing via columns added)  
**Columns** (added to sample_manufacturing):
  - appearance_spec TEXT
  - ph_spec TEXT
  - viscosity_spec TEXT
  - specific_gravity_spec TEXT

---

## 6. BOMs & QUOTATIONS

### boms
**Created**: migration-002-unified-schema.sql  
**Purpose**: Bill of Materials with full cost breakdown  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT
  - formulation_id UUID REFERENCES formulations(id) ON DELETE SET NULL
  - formulation_version INT
  - manufacturing_qty_g NUMERIC
  - fill_volume_ml NUMERIC
  - density NUMERIC DEFAULT 1.0
  - loss_rate_pct NUMERIC
  - dlc_personnel INT
  - dlc_hours_override NUMERIC
  - packaging_items JSONB DEFAULT '[]' -- [{packaging_id, quantity, _category}]
  - printing_items JSONB DEFAULT '[]' -- [{printing_id, quantity}]
  - qc_items JSONB DEFAULT '[]' -- [{qc_id, quantity}]
  - consumables_cost NUMERIC
  - utility_cost NUMERIC
  - other_indirect_cost NUMERIC
  - margin_pct NUMERIC
  - memo TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - is_active BOOLEAN DEFAULT true (added 20260506a_unified_trash_phase_b)
  - locked BOOLEAN DEFAULT false (Phase 2)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Indexes**: idx_boms_is_active  
**RLS**: Enabled; auth_all

---

### qq_customers
**Created**: 20260330_qq_tables.sql  
**Purpose**: QuickQuote sales customer master  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - company_name TEXT NOT NULL
  - contact_name TEXT
  - phone TEXT
  - email TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Migrated to**: clients (20260331_phase3c_operations.sql)  
**RLS**: Enabled; QQ permissions

---

### qq_quotations
**Created**: 20260330_qq_tables.sql  
**Purpose**: Quote master record  
**Primary Key**: id (UUID); UNIQUE(quote_number)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - customer_id UUID REFERENCES qq_customers(id) ON DELETE SET NULL
  - quote_number TEXT UNIQUE NOT NULL
  - status TEXT NOT NULL DEFAULT 'draft' -- draft / sent / negotiating / accepted / rejected
  - currency TEXT DEFAULT 'IDR'
  - exchange_rate JSONB
  - valid_until DATE
  - terms JSONB
  - notes TEXT
  - version INTEGER DEFAULT 1
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - locked BOOLEAN DEFAULT false (Phase 2)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Indexes**: idx_qq_quotations_customer, idx_qq_quotations_status  
**RLS**: Enabled; QQ permissions

---

### qq_quotation_items
**Created**: 20260330_qq_tables.sql; enhanced 20260415b  
**Purpose**: Line item within a quotation  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - quotation_id UUID REFERENCES qq_quotations(id) ON DELETE CASCADE
  - item_no INTEGER NOT NULL DEFAULT 1
  - is_base BOOLEAN DEFAULT true
  - variant_label TEXT
  - category TEXT
  - product_name TEXT
  - rm_mode TEXT DEFAULT 'category_avg' -- formulation / category_avg / ingredients
  - selected_ingredients JSONB
  - fill_volume_ml NUMERIC
  - display_volume_ml NUMERIC
  - packaging_type TEXT
  - density NUMERIC DEFAULT 1.0
  - quantity INTEGER
  - manufacturing_qty_g NUMERIC
  - loss_rate_pct NUMERIC DEFAULT 10
  - packaging_items JSONB
  - printing_items JSONB
  - qc_items JSONB
  - dlc_personnel INTEGER DEFAULT 1
  - dlc_hours_override NUMERIC
  - margin_pct NUMERIC DEFAULT 25
  - supply_price NUMERIC
  - cost_per_unit NUMERIC
  - cost_snapshot JSONB
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - formula_snapshot_id TEXT REFERENCES formula_snapshots(id) ON DELETE SET NULL (added 20260415b)
  - cost_basis_snapshot JSONB (added 20260415b)

**Indexes**: idx_qq_quotation_items_quotation, idx_qq_quotation_items_formula_snapshot  
**RLS**: Enabled; QQ permissions

---

### qq_quotation_items_snapshot
**Created**: 20260415b_qq_quotation_items_snapshot.sql  
**Purpose**: (Snapshot columns added to qq_quotation_items, not a separate table)  
**Snapshot Columns**:
  - formula_snapshot_id TEXT REFERENCES formula_snapshots(id) ON DELETE SET NULL
  - cost_basis_snapshot JSONB

---

### qq_cost_basis
**Status**: Referenced in design but exact table structure not found; likely merged with cost_basis or stored in cost_snapshot JSONB.

---

### qq_quality_tests
**Status**: Referenced but not found in migrations; may be stored within qq_quotation_items as qc_items JSONB.

---

## 7. REGISTRATION (Reg_* or BPOM/Halal)

### registration_folders
**Created**: migration-002-unified-schema.sql  
**Purpose**: Container for BPOM / HALAL registration process  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - formulation_id UUID NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - version_pin_id UUID REFERENCES version_pins(id) ON DELETE SET NULL
  - reg_type TEXT NOT NULL -- BPOM / HALAL
  - status TEXT DEFAULT 'preparing' -- preparing → documents_collecting → ready_for_review → approved → submitted → certified
  - certification_number TEXT
  - certified_at DATE
  - certification_expiry DATE
  - target_date DATE
  - started_at TIMESTAMPTZ
  - stage_dates JSONB DEFAULT '{}' -- progress tracking by stage
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - certificate_file_url TEXT (added 20260409a)
  - certificate_file_name TEXT (added 20260409a)
  - certificate_uploaded_at TIMESTAMPTZ (added 20260409a)
  - certificate_uploaded_by TEXT (added 20260409a)
  - formula_snapshot_id TEXT REFERENCES formula_snapshots(id) ON DELETE SET NULL (added 20260415c)
  - locked BOOLEAN DEFAULT false (Phase 2)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Indexes**: idx_reg_folders_formulation, idx_reg_folders_type, idx_registration_folders_formula_snapshot  
**Triggers**: trg_reg_folders_updated  
**RLS**: Enabled; auth_all

---

### registration_documents
**Created**: migration-002-unified-schema.sql  
**Purpose**: Individual document in a folder  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - folder_id UUID NOT NULL REFERENCES registration_folders(id) ON DELETE CASCADE
  - doc_category TEXT NOT NULL
  - source TEXT NOT NULL -- hub_upload / hub_generated / safety / packaging / etc.
  - source_ref TEXT
  - file_url TEXT
  - status TEXT DEFAULT 'pending' -- pending / collected / missing / expired
  - collected_at TIMESTAMPTZ

**Foreign Keys**: folder_id → registration_folders(id)  
**RLS**: Enabled; auth_all

---

### registration_submissions
**Created**: 20260409a_registration_submissions.sql  
**Purpose**: Immutable snapshot of submitted document set to authority  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - folder_id UUID NOT NULL REFERENCES registration_folders(id) ON DELETE CASCADE
  - submitted_at TIMESTAMPTZ NOT NULL DEFAULT now()
  - submitted_by TEXT
  - submission_label TEXT -- e.g. "Initial submission", "Re-submission #2"
  - notes TEXT
  - document_snapshot JSONB NOT NULL DEFAULT '[]' -- [{doc_id, doc_category, file_url, file_name, source, source_ref}]
  - doc_count INT NOT NULL DEFAULT 0
  - status TEXT NOT NULL DEFAULT 'submitted' -- submitted / superseded / certified / rejected
  - submitted_product_version_id UUID REFERENCES product_versions(id) ON DELETE SET NULL (added 20260409c)

**Indexes**: idx_reg_submissions_folder (folder_id, submitted_at DESC)  
**RLS**: Enabled; auth_all (presumed; not explicitly mentioned)

---

### registration_folders_snapshot
**Created**: 20260415c_registration_folders_snapshot.sql  
**Purpose**: (Snapshot columns added to registration_folders, not a separate table)  
**Snapshot Column**:
  - formula_snapshot_id TEXT REFERENCES formula_snapshots(id) ON DELETE SET NULL

---

### halal_material_matrix
**Created**: migration-002-unified-schema.sql  
**Purpose**: Material × Formulation halal status matrix  
**Primary Key**: id (UUID); UNIQUE(material_id, formulation_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - material_id UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE
  - formulation_id UUID NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - is_used BOOLEAN DEFAULT false
  - updated_at TIMESTAMPTZ DEFAULT now()
  - UNIQUE(material_id, formulation_id)
  - locked BOOLEAN DEFAULT false (Phase 2)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**RLS**: Enabled; auth_all

---

### halal_matrix_snapshots
**Created**: migration-002-unified-schema.sql  
**Purpose**: Snapshot of halal matrix at registration time  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - registration_folder_id UUID NOT NULL REFERENCES registration_folders(id) ON DELETE CASCADE
  - snapshot_data JSONB DEFAULT '{}'
  - file_url TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Foreign Keys**: registration_folder_id → registration_folders(id)  
**RLS**: Enabled; auth_all

---

### halal_cert_bodies
**Created**: migration-002-unified-schema.sql  
**Purpose**: Reference list of recognized halal certification bodies  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - country TEXT
  - is_recognized BOOLEAN DEFAULT true
  - updated_at TIMESTAMPTZ DEFAULT now()

**Seed Data** (10 bodies): LPPOM MUI, BPJPH, JAKIM, MUIS, CICOT, IFANCA, etc.  
**RLS**: Enabled; auth_all

---

### halal_positive_list
**Created**: migration-002-unified-schema.sql  
**Purpose**: Exemption list of ingredients (no halal cert needed)  
**Primary Key**: cas_no (TEXT); UNIQUE constraint  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - cas_no TEXT NOT NULL UNIQUE
  - inci_name TEXT
  - description TEXT

**RLS**: Enabled; auth_all

---

### halal_packaging_matrix
**Created**: migration-002-unified-schema.sql  
**Purpose**: Packaging × Formulation halal status matrix  
**Primary Key**: id (UUID); UNIQUE(packaging_id, formulation_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - packaging_id UUID NOT NULL REFERENCES packaging(id) ON DELETE CASCADE
  - formulation_id UUID NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - is_used BOOLEAN DEFAULT false
  - UNIQUE(packaging_id, formulation_id)

**RLS**: Enabled; auth_all

---

### packaging_documents
**Created**: migration-002-unified-schema.sql  
**Purpose**: Certificates for packaging materials  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - packaging_id UUID NOT NULL REFERENCES packaging(id) ON DELETE CASCADE
  - doc_type TEXT NOT NULL
  - file_url TEXT
  - version INT DEFAULT 1
  - is_current BOOLEAN DEFAULT true
  - expiry_date DATE
  - issue_date DATE
  - uploaded_at TIMESTAMPTZ DEFAULT now()
  - uploaded_by TEXT

**Foreign Keys**: packaging_id → packaging(id)  
**RLS**: Enabled; auth_all

---

### company_documents
**Created**: migration-002-unified-schema.sql  
**Purpose**: Company docs for DIP Part I (NIB, CPKB, GMP, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - doc_type TEXT NOT NULL -- NIB / CPKB / DirectorStatement / GMP / etc.
  - brand_id UUID REFERENCES brands(id) ON DELETE SET NULL
  - file_url TEXT
  - version INT DEFAULT 1
  - is_current BOOLEAN DEFAULT true
  - expiry_date DATE
  - issue_date DATE
  - issuer TEXT
  - description TEXT
  - uploaded_at TIMESTAMPTZ DEFAULT now()
  - uploaded_by TEXT

**Foreign Keys**: brand_id → brands(id)  
**RLS**: Enabled; auth_all

---

### document_checklist_config
**Created**: migration-002-unified-schema.sql  
**Purpose**: Config for required documents (material doc checklist)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - doc_type TEXT NOT NULL -- CoA / MSDS / Specification / HalalCert / IFRA / etc.
  - is_required BOOLEAN DEFAULT false
  - required_for TEXT DEFAULT 'all' -- all / halal_only / animal_origin_only / non_halal_certified / fermentation_only
  - is_halal_substitute BOOLEAN DEFAULT false
  - has_expiry BOOLEAN DEFAULT false
  - description TEXT

**Seed Data** (14 types): CoA, MSDS, Specification, HalalCert, HeavyMetal, BSE_TSE, Allergen, NonGMO, Micro, IFRA, HalalStatement, OriginCertificate, ManufacturingProcess, Questionnaire  
**RLS**: Enabled; auth_all

---

## 8. SAFETY (sa_* or legacy)

### sa_jobs (PRESUMED)
**Status**: Referenced in migrations but base CREATE not found in reviewed set.  
**From 20260413_safety_missing_tables.sql**, foreign keys reference it:
  - sa_escalations.job_id → sa_jobs(id)
  - sa_report_history.report_id → sa_reports(id) (related)

---

### sa_ingredients (PRESUMED)
**Status**: Referenced but base CREATE not found.  
**From 20260413_safety_missing_tables.sql**:
  - sa_toxicology_data.ingredient_id → sa_ingredients(id)
  - sa_mos_calculations.ingredient_id → sa_ingredients(id)

---

### sa_reports (PRESUMED)
**Status**: Referenced in migrations; likely in a separate Safety DB.  
**References**:
  - version_pins.safety_report_id → sa_reports.id
  - sa_report_history.report_id → sa_reports(id)

---

### ingredient_cache
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: Verified toxicology cache (169 rows in source data)  
**Primary Key**: cas_no (TEXT)  
**Columns**:
  - cas_no TEXT PRIMARY KEY
  - inci_name TEXT NOT NULL
  - toxicology_data JSONB DEFAULT '{}'
  - sources JSONB DEFAULT '[]'
  - confidence TEXT DEFAULT 'medium'
  - verified_by TEXT
  - last_verified TIMESTAMPTZ
  - expires_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

---

### sa_toxicology_data
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: Per-ingredient toxicology data points  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - ingredient_id UUID NOT NULL REFERENCES sa_ingredients(id) ON DELETE CASCADE
  - data_field TEXT NOT NULL
  - value TEXT
  - unit TEXT
  - species TEXT
  - method TEXT
  - source_url TEXT
  - source_text TEXT
  - source_db TEXT
  - retrieval_method TEXT DEFAULT 'cache' -- cache / api / manual
  - confidence TEXT DEFAULT 'medium'
  - tier INTEGER DEFAULT 0
  - cross_validated BOOLEAN DEFAULT false
  - human_entered BOOLEAN DEFAULT false
  - human_text TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_sa_tox_data_ingredient  
**Foreign Keys**: ingredient_id → sa_ingredients(id)

---

### sa_mos_calculations
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: Margin of Safety results per ingredient  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - ingredient_id UUID NOT NULL REFERENCES sa_ingredients(id) ON DELETE CASCADE
  - daily_exposure NUMERIC
  - concentration NUMERIC
  - dermal_absorption NUMERIC DEFAULT 0
  - retention_factor NUMERIC DEFAULT 1
  - body_weight NUMERIC DEFAULT 60
  - sed NUMERIC
  - noael NUMERIC
  - mos NUMERIC
  - safety_verdict TEXT
  - is_considered_safe BOOLEAN DEFAULT false
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_sa_mos_ingredient  
**Foreign Keys**: ingredient_id → sa_ingredients(id)

---

### sa_escalations
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: Flagged issues needing human review  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - job_id UUID NOT NULL REFERENCES sa_jobs(id) ON DELETE CASCADE
  - ingredient_id UUID REFERENCES sa_ingredients(id) ON DELETE SET NULL
  - type TEXT NOT NULL
  - description TEXT
  - auto_resolution TEXT
  - human_response TEXT
  - resolved_by TEXT
  - resolved_at TIMESTAMPTZ
  - extracted_value TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_sa_escalations_job  
**Foreign Keys**: job_id → sa_jobs(id), ingredient_id → sa_ingredients(id)

---

### sa_report_history
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: Audit trail for report edits  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - report_id UUID NOT NULL REFERENCES sa_reports(id) ON DELETE CASCADE
  - field_path TEXT NOT NULL
  - old_value TEXT
  - new_value TEXT
  - changed_by TEXT
  - changed_at TIMESTAMPTZ DEFAULT now()
  - change_reason TEXT
  - version_snapshot_id UUID

**Foreign Keys**: report_id → sa_reports(id)

---

### sa_translation_dictionary
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: EN↔ID translation pairs for technical terms  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - english_term TEXT NOT NULL
  - indonesian_term TEXT NOT NULL
  - category TEXT DEFAULT 'technical_term'
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

**Seed Data**: 23 term pairs

---

## 9. TALENT & TRAINING

### talent_people
**Created**: 20260407_talent_phase_a.sql  
**Purpose**: Staff master (internal + external trainers)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - full_name TEXT NOT NULL
  - email TEXT
  - phone TEXT
  - department TEXT
  - role TEXT -- e.g. Production Staff, R&D Staff
  - level TEXT -- Junior / Senior / Lead / Manager
  - join_date DATE
  - birth_date DATE
  - photo_url TEXT
  - is_internal BOOLEAN DEFAULT TRUE
  - qualifications JSONB DEFAULT '[]' -- [{topic, level, approved_at, approved_by}]
  - notes TEXT
  - active BOOLEAN DEFAULT TRUE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Indexes**: idx_talent_people_active (WHERE active)  
**Triggers**: trg_talent_people_updated  
**RLS**: Enabled; auth_all

---

### talent_teams
**Created**: 20260407_talent_phase_a.sql  
**Purpose**: Organizational hierarchy  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - parent_team_id UUID REFERENCES talent_teams(id) ON DELETE SET NULL
  - team_lead_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - mission TEXT
  - description TEXT
  - display_order INTEGER DEFAULT 0
  - active BOOLEAN DEFAULT TRUE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Triggers**: trg_talent_teams_updated  
**RLS**: Enabled; auth_all

---

### talent_positions
**Created**: 20260407_talent_phase_a.sql  
**Purpose**: Job descriptions (JD) with competency requirements  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - team_id UUID REFERENCES talent_teams(id) ON DELETE CASCADE
  - title TEXT NOT NULL
  - level TEXT -- Junior / Senior / Lead / Manager
  - reports_to_position_id UUID REFERENCES talent_positions(id) ON DELETE SET NULL
  - headcount_target INTEGER DEFAULT 1
  - jd JSONB DEFAULT '{}'::jsonb -- {purpose, responsibilities[], required_skills[], required_qualifications[], kpis[]}
  - status TEXT DEFAULT 'Active' -- Draft / Active / Archived
  - version INTEGER DEFAULT 1
  - active BOOLEAN DEFAULT TRUE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - updated_by TEXT

**Indexes**: idx_talent_positions_team  
**Triggers**: trg_talent_positions_updated  
**RLS**: Enabled; auth_all

---

### talent_assignments
**Created**: 20260407_talent_phase_a.sql  
**Purpose**: Person ↔ Position allocation (supports multi-role)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - person_id UUID NOT NULL REFERENCES talent_people(id) ON DELETE CASCADE
  - position_id UUID NOT NULL REFERENCES talent_positions(id) ON DELETE CASCADE
  - start_date DATE
  - end_date DATE -- NULL = current
  - allocation_pct NUMERIC DEFAULT 100 -- 0..100
  - is_primary BOOLEAN DEFAULT TRUE
  - assigned_by TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_talent_assignments_person, idx_talent_assignments_position  
**Triggers**: trg_talent_assignments_updated  
**RLS**: Enabled; auth_all

---

### talent_external_trainings
**Created**: 20260420b_talent_external_trainings.sql  
**Purpose**: External seminar/conference/certification tracking  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - person_id UUID NOT NULL REFERENCES talent_people(id) ON DELETE CASCADE
  - topic TEXT NOT NULL
  - provider TEXT -- e.g. BPOM, IPCC, university
  - category TEXT -- Seminar / Workshop / Conference / Online / Certification
  - description TEXT
  - date_from DATE
  - date_to DATE
  - duration_hours NUMERIC
  - location TEXT -- city / venue / URL
  - cost_idr NUMERIC
  - paid_by TEXT -- Company / Self / Sponsored
  - certificate_url TEXT
  - certificate_path TEXT -- supabase storage path
  - self_report JSONB DEFAULT '{}' -- {learned, to_apply, recommend_score, additional}
  - status TEXT DEFAULT 'Registered' -- Registered / Attended / FollowedUp / Cancelled
  - approved_by TEXT
  - approved_at TIMESTAMPTZ
  - memo TEXT
  - created_by TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - group_id UUID (added 20260420c) -- for bulk registration

**Indexes**: idx_external_trainings_person_id, idx_external_trainings_date_from, idx_external_trainings_status, idx_external_trainings_group_id  
**RLS**: Enabled; auth_all

---

### training_program_templates
**Created**: 20260408_talent_phase_b.sql  
**Purpose**: Reusable OJT / Program blueprint  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - description TEXT
  - target_role TEXT -- e.g. "Production Staff" (auto-trigger key)
  - duration_weeks INTEGER DEFAULT 4
  - cadence_note TEXT -- free text schedule description
  - session_blueprints JSONB DEFAULT '[]' -- [{week, day_of_week, session_type, topic, main_goal, deliverable, eval_standard, duration_hours, suggested_content}]
  - is_default_for_role BOOLEAN DEFAULT FALSE
  - active BOOLEAN DEFAULT TRUE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - updated_by TEXT

**Indexes**: idx_tpt_role_default (target_role, is_default_for_role) WHERE active  
**Triggers**: trg_tpt_updated  
**RLS**: Enabled; auth_all  
**Seed**: Production Staff OJT (Default) with 12 sessions

---

### training_programs
**Created**: 20260408_talent_phase_b.sql  
**Purpose**: OJT instance for a specific trainee  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - title TEXT NOT NULL
  - trainee_id UUID REFERENCES talent_people(id) ON DELETE CASCADE
  - template_id UUID REFERENCES training_program_templates(id) ON DELETE SET NULL
  - start_date DATE NOT NULL
  - end_date DATE
  - cadence_note TEXT
  - scope TEXT
  - status TEXT DEFAULT 'Active' -- Planning / Active / Completed / Cancelled
  - owner_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - memo TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_tp_trainee, idx_tp_status  
**Triggers**: trg_tp_updated  
**RLS**: Enabled; auth_all

---

### training_curricula
**Created**: 20260408_talent_phase_b.sql  
**Purpose**: Recurring/regular team training (one-time or recurring)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - title TEXT NOT NULL
  - category TEXT -- GMP / Formulation / Safety / Tech / Soft skill / Other
  - description TEXT
  - target_roles TEXT[] DEFAULT '{}'
  - target_departments TEXT[] DEFAULT '{}'
  - recurrence JSONB DEFAULT '{"type":"one-time"}' -- {type: one-time / weekly / monthly / quarterly, weekday?, day_of_month?, time?, until?, count?}
  - next_generation_at TIMESTAMPTZ
  - owner_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - status TEXT DEFAULT 'Active' -- Draft / Active / Archived
  - memo TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Triggers**: trg_tc_updated  
**RLS**: Enabled; auth_all  
**Seed**: RT-01 to RT-12 (12 Regular Training curricula)

---

### training_sessions
**Created**: 20260408_talent_phase_b.sql; enhanced 20260415e (lifecycle)  
**Purpose**: Single training event (OJT session or Curriculum session)  
**Primary Key**: id (UUID); XOR CHECK (program_id XOR curriculum_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - program_id UUID REFERENCES training_programs(id) ON DELETE CASCADE
  - curriculum_id UUID REFERENCES training_curricula(id) ON DELETE CASCADE
  - session_no INTEGER
  - date DATE NOT NULL
  - day_of_week TEXT -- Mon..Sun (informational)
  - start_time TEXT -- 'HH:MM'
  - duration_hours NUMERIC DEFAULT 2
  - session_type TEXT -- Regular / OJT / Final
  - topic TEXT NOT NULL
  - main_goal TEXT
  - deliverable TEXT
  - eval_standard TEXT
  - suggested_content TEXT
  - location TEXT
  - trainer_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - participants UUID[] DEFAULT '{}' -- talent_people IDs
  - trial_formulation_id TEXT -- FK to formulations.id (TEXT PK)
  - linked_doc_url TEXT
  - status TEXT DEFAULT 'Scheduled' -- Scheduled / Done / Cancelled
  - memo TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - CHECK ((program_id IS NOT NULL) <> (curriculum_id IS NOT NULL))

**Indexes**: idx_ts_program, idx_ts_curriculum, idx_ts_date  
**Triggers**: trg_ts_updated  
**RLS**: Enabled; auth_all

---

### training_materials
**Created**: 20260408c_talent_phase_c.sql  
**Purpose**: File library (SOP, Handout, Slides, Checklist, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - title TEXT NOT NULL
  - category TEXT -- SOP / Handout / Slides / Checklist / Form / Other
  - description TEXT
  - file_url TEXT
  - file_name TEXT
  - file_size INTEGER
  - mime_type TEXT
  - version TEXT DEFAULT '1.0'
  - tags TEXT[] DEFAULT '{}'
  - uploaded_by UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - active BOOLEAN DEFAULT TRUE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_tm_category  
**Triggers**: trg_tm_updated  
**RLS**: Enabled; auth_all

---

### training_session_materials
**Created**: 20260408c_talent_phase_c.sql  
**Purpose**: Session ↔ Material junction (M:N)  
**Primary Key**: (session_id, material_id)  
**Columns**:
  - session_id UUID NOT NULL REFERENCES training_sessions(id) ON DELETE CASCADE
  - material_id UUID NOT NULL REFERENCES training_materials(id) ON DELETE CASCADE
  - attached_at TIMESTAMPTZ DEFAULT now()
  - PRIMARY KEY (session_id, material_id)

**RLS**: Enabled; auth_all

---

### training_attendance
**Created**: 20260408c_talent_phase_c.sql  
**Purpose**: Per-session attendance record (Present / Absent / Late / Excused)  
**Primary Key**: id (UUID); UNIQUE(session_id, person_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - session_id UUID NOT NULL REFERENCES training_sessions(id) ON DELETE CASCADE
  - person_id UUID NOT NULL REFERENCES talent_people(id) ON DELETE CASCADE
  - status TEXT DEFAULT 'Present' -- Present / Absent / Late / Excused
  - remarks TEXT
  - signature_url TEXT
  - marked_by UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - marked_at TIMESTAMPTZ DEFAULT now()
  - UNIQUE (session_id, person_id)

**Indexes**: idx_attendance_session, idx_attendance_person  
**RLS**: Enabled; auth_all

---

### training_evaluations
**Created**: 20260408d_talent_phase_d.sql  
**Purpose**: Per-session trainer evaluation of trainee  
**Primary Key**: id (UUID); UNIQUE(session_id, trainee_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - session_id UUID NOT NULL REFERENCES training_sessions(id) ON DELETE CASCADE
  - trainee_id UUID NOT NULL REFERENCES talent_people(id) ON DELETE CASCADE
  - evaluator_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - scores JSONB DEFAULT '[]' -- [{criterion, weight, score (1-5), comment}]
  - total_weighted NUMERIC
  - result TEXT DEFAULT 'Pass' -- Pass / Conditional Pass / Retraining Required
  - approved_scope TEXT
  - retraining_topic TEXT
  - recommendation TEXT
  - next_action TEXT
  - next_due_date DATE
  - trainer_signature TEXT
  - manager_review TEXT
  - signed_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - UNIQUE (session_id, trainee_id)

**Indexes**: idx_te_session, idx_te_trainee  
**Triggers**: trg_te_updated  
**RLS**: Enabled; auth_all

---

### training_reports
**Created**: 20260408d_talent_phase_d.sql  
**Purpose**: Per-session participation report (topics, observations, follow-up)  
**Primary Key**: id (UUID); UNIQUE(session_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - session_id UUID NOT NULL UNIQUE REFERENCES training_sessions(id) ON DELETE CASCADE
  - topics_covered TEXT
  - trial_or_practice TEXT
  - key_observations TEXT
  - questions_raised TEXT
  - follow_up_action TEXT
  - prepared_by_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - reviewed_by_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - prepared_at DATE
  - reviewed_at DATE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Triggers**: trg_tr_updated  
**RLS**: Enabled; auth_all

---

### training_quizzes
**Created**: 20260408g_talent_quizzes.sql  
**Purpose**: Quiz/test (attached to session XOR curriculum)  
**Primary Key**: id (UUID); XOR CHECK (session_id XOR curriculum_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - session_id UUID REFERENCES training_sessions(id) ON DELETE CASCADE
  - curriculum_id UUID REFERENCES training_curricula(id) ON DELETE CASCADE
  - title TEXT NOT NULL
  - description TEXT
  - questions JSONB NOT NULL DEFAULT '[]' -- [{id, type: 'mcq'|'short_answer', text, options?, correct_answer?, points}]
  - attached_pdf_url TEXT
  - max_score NUMERIC DEFAULT 0
  - status TEXT DEFAULT 'Active' -- Draft / Active / Closed
  - created_by UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - CHECK ((session_id IS NOT NULL) <> (curriculum_id IS NOT NULL))

**Indexes**: idx_tq_session, idx_tq_curriculum  
**Triggers**: trg_tq_updated  
**RLS**: Enabled; auth_all

---

### training_quiz_responses
**Created**: 20260408g_talent_quizzes.sql  
**Purpose**: Trainee quiz responses (auto-score for MCQ, manual for short answer)  
**Primary Key**: id (UUID); UNIQUE(quiz_id, trainee_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - quiz_id UUID NOT NULL REFERENCES training_quizzes(id) ON DELETE CASCADE
  - trainee_id UUID NOT NULL REFERENCES talent_people(id) ON DELETE CASCADE
  - answers JSONB NOT NULL DEFAULT '[]' -- [{question_id, value}]
  - auto_score NUMERIC DEFAULT 0
  - manual_score NUMERIC -- nullable until graded
  - total_score NUMERIC
  - max_score NUMERIC DEFAULT 0 -- snapshot of quiz.max_score at submission
  - status TEXT DEFAULT 'submitted' -- submitted / graded
  - submitted_at TIMESTAMPTZ DEFAULT now()
  - graded_at TIMESTAMPTZ
  - graded_by UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - feedback TEXT
  - UNIQUE (quiz_id, trainee_id)

**Indexes**: idx_tqr_quiz, idx_tqr_trainee  
**RLS**: Enabled; auth_all

---

## 10. LABEL CHECK / PACKAGING VALIDATOR

### lc_projects
**Created**: 20260330_pv_tables.sql  
**Purpose**: Label Check project (packaging compliance review)  
**Primary Key**: id (UUID); UNIQUE(project_code)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - project_code TEXT UNIQUE NOT NULL
  - project_name TEXT NOT NULL
  - brand_name TEXT NOT NULL
  - product_name TEXT NOT NULL
  - variant_name TEXT
  - category TEXT NOT NULL
  - dosage_form TEXT
  - packaging_type TEXT NOT NULL
  - target_age TEXT NOT NULL DEFAULT 'adult'
  - market TEXT DEFAULT 'Indonesia'
  - volume_value NUMERIC
  - volume_unit TEXT
  - halal_required BOOLEAN DEFAULT false
  - inci_list TEXT
  - inci_source TEXT
  - key_claims TEXT
  - product_function TEXT
  - is_sunscreen BOOLEAN DEFAULT false
  - is_for_children BOOLEAN DEFAULT false
  - erp_product_code TEXT
  - assigned_to TEXT
  - overall_status TEXT DEFAULT 'draft'
  - version_number INTEGER DEFAULT 1
  - created_by TEXT
  - created_by_role TEXT
  - inci_file_url TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

---

### lc_components
**Created**: 20260330_pv_tables.sql  
**Purpose**: Individual packaging component (label, carton, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - project_id UUID REFERENCES lc_projects(id) ON DELETE CASCADE
  - component_code TEXT NOT NULL
  - component_type TEXT NOT NULL
  - component_role TEXT
  - surface_position TEXT
  - space_constrained BOOLEAN DEFAULT false
  - file_url TEXT
  - file_path TEXT
  - mime_type TEXT
  - extracted_text TEXT
  - extraction_confidence NUMERIC
  - structured_data JSONB
  - status TEXT DEFAULT 'active'
  - original_filename TEXT
  - uploaded_at TIMESTAMPTZ DEFAULT now()

---

### lc_reviews
**Created**: 20260330_pv_tables.sql  
**Purpose**: Review result for a project version  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - project_id UUID REFERENCES lc_projects(id) ON DELETE CASCADE
  - version INTEGER NOT NULL DEFAULT 1
  - version_major INTEGER DEFAULT 0
  - version_minor INTEGER DEFAULT 1
  - version_suffix TEXT
  - component_ids UUID[]
  - overall_status TEXT
  - claim_risk_level TEXT
  - structured_data JSONB
  - combined_extracted_text TEXT
  - correction_suggestions TEXT
  - pipeline_status TEXT DEFAULT 'pending'
  - pipeline_step TEXT
  - pipeline_progress INTEGER DEFAULT 0
  - started_at TIMESTAMPTZ
  - completed_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_review_items
**Created**: 20260330_pv_tables.sql  
**Purpose**: Individual issue/finding in a review  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - review_id UUID REFERENCES lc_reviews(id) ON DELETE CASCADE
  - component_id UUID
  - step TEXT NOT NULL
  - item_name TEXT NOT NULL
  - severity TEXT NOT NULL
  - category TEXT
  - location TEXT
  - pack_level TEXT
  - description TEXT NOT NULL
  - suggestion TEXT
  - regulation_ref TEXT
  - alternatives TEXT[]
  - raw_data JSONB
  - feedback_status TEXT
  - feedback_by TEXT
  - feedback_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_review_actions
**Created**: 20260330_pv_tables.sql  
**Purpose**: Action log for review (request / approval)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - project_id UUID REFERENCES lc_projects(id) ON DELETE CASCADE
  - review_id UUID REFERENCES lc_reviews(id) ON DELETE SET NULL
  - action_type TEXT NOT NULL
  - requested_by TEXT
  - requested_by_role TEXT
  - approved_by TEXT
  - approved_by_title TEXT
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_regulation_versions
**Created**: 20260330_pv_tables.sql  
**Purpose**: Regulation reference versions  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - regulation_name TEXT NOT NULL
  - regulation_number TEXT NOT NULL
  - version TEXT NOT NULL
  - effective_date DATE
  - data_updated_at TIMESTAMPTZ
  - verified_by TEXT
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_packaging_requirements
**Created**: 20260330_pv_tables.sql  
**Purpose**: Packaging field requirements by type & level  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - packaging_type TEXT NOT NULL
  - component_level TEXT NOT NULL
  - required_field TEXT NOT NULL
  - is_mandatory BOOLEAN DEFAULT true
  - condition_text TEXT
  - regulation_ref TEXT
  - version TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_general_requirements
**Created**: 20260330_pv_tables.sql  
**Purpose**: General compliance requirements  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - requirement_name TEXT NOT NULL
  - description TEXT NOT NULL
  - check_type TEXT
  - regulation_ref TEXT
  - version TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_category_requirements
**Created**: 20260330_pv_tables.sql  
**Purpose**: Category-specific requirements  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - category TEXT NOT NULL
  - requirement_type TEXT NOT NULL
  - description TEXT NOT NULL
  - condition_text TEXT
  - regulation_ref TEXT
  - version TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_claim_regulations
**Created**: 20260330_pv_tables.sql  
**Purpose**: Claim validation rules by category  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - category TEXT NOT NULL
  - pattern TEXT NOT NULL
  - description TEXT
  - examples_prohibited TEXT[]
  - examples_allowed TEXT[]
  - evidence_required BOOLEAN DEFAULT false
  - action TEXT
  - regulation_ref TEXT
  - version TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_claim_detailed_rules
**Created**: 20260330_pv_tables.sql  
**Purpose**: Detailed claim validation rules  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - rule_number INTEGER NOT NULL
  - category TEXT NOT NULL
  - description TEXT NOT NULL
  - description_id TEXT
  - pattern TEXT
  - action TEXT
  - regulation_ref TEXT
  - version TEXT DEFAULT '3/2022'
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_allowed_claims
**Created**: 20260330_pv_tables.sql  
**Purpose**: Allowed claims by product type  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - product_type_no INTEGER NOT NULL
  - product_type TEXT NOT NULL
  - product_type_en TEXT
  - categories JSONB NOT NULL
  - regulation_ref TEXT
  - version TEXT DEFAULT '3/2022'
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_prohibited_claims
**Created**: 20260330_pv_tables.sql (truncated in read)  
**Purpose**: Prohibited claims reference  
**Primary Key**: id (UUID)  
**Columns** (inferred):
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - claim_no INTEGER NOT NULL
  - claim_id TEXT NOT NULL
  - claim_en TEXT
  - category_hint TEXT
  - [further columns...]

---

## 11. MISC / SYSTEM

### cost_basis
**Created**: migration-002-unified-schema.sql  
**Purpose**: Singleton table for company-wide cost factors (labor, depreciation, overhead)  
**Primary Key**: id (INT PRIMARY KEY DEFAULT 1)  
**Columns**:
  - id INT PRIMARY KEY DEFAULT 1
  - monthly_salary NUMERIC
  - working_days_per_month INT
  - hours_per_day INT
  - production_lines JSONB DEFAULT '[]'
  - annual_equip_depreciation NUMERIC
  - annual_building_depreciation NUMERIC
  - dep_months INT
  - overhead_total NUMERIC
  - overhead_months INT
  - utility_monthly NUMERIC
  - hygiene_monthly NUMERIC
  - consumable_items JSONB DEFAULT '[]'
  - idlc_hours_per_batch NUMERIC
  - idlc_personnel INT
  - _locked BOOLEAN DEFAULT false
  - _edit_history JSONB DEFAULT '[]'
  - updated_at TIMESTAMPTZ DEFAULT now()

**RLS**: Enabled; auth_all

---

### alerts
**Created**: migration-002-unified-schema.sql  
**Purpose**: System alerts (doc expiry, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - type TEXT NOT NULL
  - severity TEXT DEFAULT 'info'
  - target_type TEXT NOT NULL
  - target_id UUID NOT NULL
  - message TEXT NOT NULL
  - is_read BOOLEAN DEFAULT false
  - created_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_alerts_unread (WHERE is_read = false)  
**RLS**: Enabled; auth_all

---

### change_alerts
**Created**: 20260409f_change_alerts.sql  
**Purpose**: Cross-entity change notifications  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - source_entity TEXT NOT NULL -- formulation / packaging / printing / bom / label_check / product
  - source_id TEXT NOT NULL
  - source_change TEXT
  - target_entity TEXT NOT NULL -- product / formulation / bom / bpom_folder / halal_folder / label_check
  - target_id TEXT NOT NULL
  - reason TEXT NOT NULL
  - status TEXT NOT NULL DEFAULT 'pending' -- pending / resolved / dismissed
  - created_at TIMESTAMPTZ NOT NULL DEFAULT now()
  - created_by TEXT
  - resolved_at TIMESTAMPTZ
  - resolved_by TEXT
  - resolved_note TEXT

**Indexes**: idx_change_alerts_target, idx_change_alerts_source, idx_change_alerts_status  
**RLS**: (not explicitly enabled in reviewed migrations)

---

### projects
**Created**: 20260331_projects.sql  
**Purpose**: Product lifecycle management  
**Primary Key**: id (UUID); UNIQUE(project_code)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - project_code TEXT UNIQUE
  - project_name TEXT NOT NULL
  - client_id UUID REFERENCES clients(id) ON DELETE SET NULL
  - formulation_id TEXT
  - proposal_id UUID
  - quotation_id UUID
  - stage TEXT DEFAULT 'proposal' -- proposal / design / sampling / production / launch
  - priority TEXT DEFAULT 'normal'
  - owner TEXT
  - start_date DATE
  - target_date DATE
  - actual_completion DATE
  - stage_history JSONB DEFAULT '[]'
  - notes TEXT
  - status TEXT DEFAULT 'active'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Triggers**: trg_projects_updated  
**RLS**: Enabled; auth_all

---

### documents
**Created**: 20260331_phase3d_documents.sql  
**Purpose**: Polymorphic document storage (polymorph on entity_type + entity_id)  
**Primary Key**: id (UUID); UNIQUE(doc_number)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - doc_number TEXT UNIQUE
  - doc_type TEXT NOT NULL -- category from document_categories
  - title TEXT NOT NULL
  - description TEXT
  - file_url TEXT
  - file_name TEXT
  - file_size_bytes BIGINT
  - file_type TEXT
  - entity_type TEXT -- formulation / product / client / supplier / etc.
  - entity_id TEXT
  - version INT DEFAULT 1
  - is_current BOOLEAN DEFAULT true
  - status TEXT DEFAULT 'active'
  - expiry_date DATE
  - issued_by TEXT
  - issued_at DATE
  - tags TEXT[]
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Triggers**: trg_documents_updated  
**RLS**: Enabled; auth_all

---

### document_categories
**Created**: 20260331_phase3d_documents.sql  
**Purpose**: Reference list of document types  
**Primary Key**: id (UUID); UNIQUE(slug)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - slug TEXT UNIQUE NOT NULL
  - status TEXT DEFAULT 'active'
  - scope TEXT DEFAULT 'global' -- global / material / formula / client / supplier / company / regulatory
  - sort_order INT
  - created_at TIMESTAMPTZ DEFAULT now()

**Seed Data** (30 types): CoA, MSDS, TDS, Specification, Halal Cert, Halal Statement, Origin Cert, Manufacturing Process, IFRA, Formula Sheet, Spec Sheet, Stability Report, Manufacturing Sheet, Full Dossier, Version Comparison, BOM Export, Client Brief, Contract, NDA, Supplier Agreement, Price List, Delivery Note, PO, GMP Cert, NIB, BPOM Reg, Label Approval, Safety Report, Photo, Other

---

### record_locks
**Created**: migration-002-unified-schema.sql  
**Purpose**: Concurrency control for simultaneous editing  
**Primary Key**: id (TEXT) -- '{table_name}_{record_id}'  
**Columns**:
  - id TEXT PRIMARY KEY
  - table_name TEXT NOT NULL
  - record_id UUID NOT NULL
  - locked_by TEXT NOT NULL
  - locked_at TIMESTAMPTZ DEFAULT now()

---

---

## APPENDIX A: SOFT-DELETE PATTERNS

### Phase A (Trash Pattern v1 — 2026-05-05)
**Affected Tables**: materials, sample_materials, packaging, formulations, documents, products

- **Pattern**: `is_active BOOLEAN DEFAULT true` OR `status='discontinued'/'archived'`
- **API Behavior**: DELETE → soft-delete (is_active := false); ?purge=1 → hard-delete (after reference check)
- **Indexes**: idx_{table}_is_active (WHERE is_active = true)

### Phase B (Trash Pattern v2 — 2026-05-06)
**Affected Tables**: boms, stability_tests, sample_manufacturing, suppliers

- **Pattern**:
  - boms, stability_tests, sample_manufacturing: `is_active BOOLEAN DEFAULT true`
  - suppliers: existing `status TEXT` (soft-delete = status='inactive')
- **Indexes**: idx_{table}_is_active (where applicable)

### Phase C & D
**Status**: Not yet reviewed; likely to follow Phase B pattern.

---

## APPENDIX B: SNAPSHOT-LOCKED TABLES (Price/State Freezing)

| Table | Locked Field(s) | Purpose | Trigger Event |
|-------|-----------------|---------|---------------|
| formula_snapshots | id (TEXT), composition, cost_basis, material_prices, packaging_prices | Freeze formula composition + costs | Confirmed status / quote issue / DIP publish |
| qq_quotation_items | formula_snapshot_id, cost_basis_snapshot | Snapshot frozen at quote-issue time | Quote finalize |
| registration_folders | formula_snapshot_id | Snapshot at DIP approve time | Registration approval |
| registration_folders_snapshot | (columns added to registration_folders) | – | – |
| qq_quotation_items_snapshot | (columns added to qq_quotation_items) | – | – |

---

## APPENDIX C: AUDIT & CHANGE TRACKING

### auth_audit_logs
**Purpose**: Complete audit trail for auth events + resource changes

**Key Columns**:
  - event_type (login_success, password_change, account_lock, etc.)
  - resource_type TEXT (added 20260403)
  - resource_id TEXT (added 20260403)
  - changes JSONB (added 20260414a) -- {field_name: {from, to}}
  - change_reason TEXT (added 20260414a)

**Key Indexes**:
  - idx_auth_audit_event
  - idx_auth_audit_user
  - idx_audit_resource
  - idx_audit_resource_changes (WHERE changes IS NOT NULL)

### Change Control Lock Columns
**Pattern** (Phase 2): Tables with lifecycle add uniform lock metadata:
  - locked BOOLEAN DEFAULT false
  - locked_at TIMESTAMPTZ
  - locked_by TEXT
  - lock_reason TEXT
  - Indexes: idx_{table}_locked (WHERE locked = true)

**Tables with locks**:
  - All masters: materials, formulations, packaging, printing, processes, products
  - Published docs: boms, qq_quotations, registration_folders, registration_submissions
  - Quality: stability_tests, stability_test_results, sa_jobs
  - Halal: halal_material_matrix, halal_matrix_snapshots
  - Ops: clients, suppliers, brands
  - Formulations extend: locked_at, locked_by, lock_reason (20260414c)

---

## APPENDIX D: SEED DATA TABLES

**Data-only (no DDL) migrations** — skip in final schema reference but list here:
  - 20260408h_seed_wis_quizzes.sql
  - 20260408i_seed_wis_quizzes_part2.sql
  - 20260408l_seed_ojt_curriculum.sql
  - 20260408e_seed_wis_quizzes.sql
  - 20260409e_seed_wis_0004.sql
  - 20260429b_demo_seed.sql
  - 20260429c_onboarding_knowledge_seed.sql
  - 20260429d_demo_phase_b.sql
  - 20260430a_demo_base_formulas.sql
  - 20260430b_demo_quote_seed.sql
  - 20260430c_demo_product_bom_seed.sql
  - 20260430d_demo_packaging_seed.sql
  - 20260430e_demo_talent_pending.sql
  - 20260430f_demo_trainee_seed.sql
  - 20260430g_external_trainings_materials.sql

---

## SUMMARY STATISTICS

- **Total tables**: ~100
- **Soft-delete-enabled**: ~20 (phases A, B, C, D to follow)
- **Snapshot-locked**: 3 (formula_snapshots, qq_quotation_items, registration_folders)
- **With unified change control locks**: ~15
- **With audit triggers (update_updated_at)**: ~40+
- **RLS enabled**: ~95%
- **Created by**: 76 migrations + 3 base scripts (setup-db.sql, migration-001, migration-002)

<!-- SOURCE_FILE_5_END: COSLAB-HUB-SCHEMA.md -->

---


<!-- SOURCE_FILE_6_BEGIN: COSLAB-HUB-STATUS.md -->

# SOURCE FILE 6: `COSLAB-HUB-STATUS.md`

# COSLAB Hub — 현재 구현 상태 점검 (2026-05-05)

> **목적**: 어떤 기능이 실제로 구현되었고, 어떤 것이 설계만 있고, 어떤 것이 보류/폐기되었는지 한눈에 본다.
> **범위 기준일**: 2026-05-05 (커밋 `84c8cbd` 시점)
> **참조 설계서**: [`COSLAB-HUB-DESIGN.md`](./COSLAB-HUB-DESIGN.md)

상태 표기:
- ✅ **구현 완료** — 프로덕션에서 동작
- 🟡 **부분 구현** — 핵심은 있고 추가 작업 필요
- 🟠 **계획만** — 설계 있음, 구현 X
- ⚫ **보류/폐기** — 의도적으로 안 만듦

---

## 1. 한눈에 보는 현황

| 영역 | 상태 | 비고 |
|---|---|---|
| **인프라** | ✅ | Next.js 16 + Supabase Pro + Vercel Hobby + 일일 백업 cron |
| **인증 + 2FA** | ✅ | hub_accounts + bcryptjs + TOTP optional |
| **사이드바 9그룹** | ✅ | Toggle 동작, role-based hide |
| **시간대 (Jakarta)** | ✅ | jakarta.ts 헬퍼 통일 (5/4 calendar 버그 수정) |
| **변경통제** | ✅ | snapshots/locks/audit + price impact + drift |
| **Formula 모듈** | ✅ | 6-탭 + Phase 구조 + 버전비교 + 리뉴얼 |
| **Stability** | ✅ | 2-stage 레터 + 셀 잠금 + Researcher Report |
| **Samples** | ✅ | 워크플로우 + DPI 코드 + DOCX |
| **Materials/INCI** | ✅ | + Sample Materials 분리 + Halal Matrix |
| **Suppliers/Clients/Inventory** | ✅ | Phase 3c 완료 |
| **Products** | ✅ | 7-탭 + product_versions Phase C |
| **Packaging** | ✅ | 2-탭 + soft delete |
| **Quotes** | ✅ | 10항목 + ExcelJS + snapshot |
| **Registration** | ✅ | BPOM/HALAL/DIP/Company + drift |
| **Safety Assessment** | ✅ | 7단계 + 이중언어 + retry |
| **Label Check** | 🟡 | 통합 진입점 OK, 데이터는 별도 PV Supabase |
| **Documents** | ✅ | 8소스 통합 허브 + 폴더 업로드 |
| **Talent** | ✅ | 신설, Phase A-E + Sprint 1-3 완료 |
| **Knowledge / Chat / Upload** | ✅ | Knowledge Hub 보존 + AI 파싱 |
| **AI Studio / Smart Advisor / AI Formulator** | 🟡 | 챗 + 데모 OK, 규칙 엔진 미구축 |
| **Production / Proposals** | ✅ | Knowledge Hub 보존 |
| **Projects** | 🟡 | 간소 페이지만 (Phase B/C 보류) |
| **Executive** | ✅ | snapshot+period 보고서 + blockers + pipeline |
| **Onboarding / Welcome** | ✅ | 8-step + 영상 + 데모 시드 6개 |
| **PWA / Offline** | 🟡 | Service Worker 등록, IndexedDB 인프라 — 활용 사례 적음 |
| **Voice/Photo 입력** | 🟡 | 페이지 존재, 활용도 낮음 |

---

## 2. 모듈별 상세

### 2.1 Formula (`/formulation/formulas`) ✅

| 기능 | 상태 |
|---|---|
| 6-탭 (Overview/Specs/Stability/Documents/Samples/Review History) | ✅ |
| Phase 구조 (`versions[].phases[]`) | ✅ |
| 자동 코드 생성 (`SK-SRM-001`) | ✅ `formula_code_sequences` |
| 버전 비교 (`/compare?v1=2&v2=3`) | ✅ |
| 리뉴얼 관리 (renewal_group, superseded_by) | ✅ |
| Specs & Feasibility 탭 (`formula_specs`) | ✅ |
| Phase 4 soft delete + Phase 3 manual unlock | ✅ |
| Cost panel + Price Lock | ✅ snapshots + drift alerts |
| Export: Formula Sheet/Spec/Manufacturing/Full Dossier/BOM | ✅ |
| Actions menu (Print/Duplicate/History/HardDelete) | ✅ Phase A |
| Snapshot 잠금 + 수동 unlock 사유 | ✅ `formulations_lock_metadata` |

**위치**: `src/app/formulation/formulas/`, `src/lib/formulation-helpers.ts`, `src/lib/change-control/`

### 2.2 Stability (`/stability`) ✅

| 기능 | 상태 |
|---|---|
| 2-stage 레터 (1-Month + Final Confirmation) | ✅ 2026-04-21 재설계 |
| Researcher Report DOCX | ✅ |
| 셀 단위 자동 잠금 + 수정 사유 + diff | ✅ GMP 패턴 |
| 조건세트 × 인터벌 × 파라미터 매트릭스 | ✅ |
| Performance custom parameter row | ✅ |
| BPOM/CPKB 표현 + 1-month gate to Completed | ✅ |
| Stability Test Templates | ✅ `/test-templates` |
| Final verdict + verdict_date | ✅ |
| Letter durations + shelf-life 계산 | ✅ `src/lib/stability/shelf-life.ts` |
| 002c PK to text 마이그레이션 | ✅ |
| Stability ↔ Base Formula 연동 | ✅ |

**위치**: `src/app/stability/`, `src/lib/stability/`

### 2.3 Samples (`/samples`) ✅

| 기능 | 상태 |
|---|---|
| Manufacturing Sheet DOCX | ✅ Phase 이름+지시문 포함 |
| Phase별 자유 양식 + 편집 | ✅ |
| QC 결과 (수율/pH/점도/SG) | ✅ |
| Sample QC Specs | ✅ |
| 워크플로우 상태머신 + Confirmed by client | ✅ A.2 |
| DPI 샘플 코드 (자동 + 수동 편집) | ✅ |
| List 정리 (Formula 컬럼 drop, Client sanitize) | ✅ |

### 2.4 Materials & Sample Materials ✅

| 기능 | 상태 |
|---|---|
| Materials 마스터 (INCI/CAS/공급사/단가/Halal) | ✅ |
| Sample Materials 분리 (production과 별도) | ✅ |
| INCI Dictionary | ✅ |
| Halal Material × Product Matrix | ✅ |
| Halal Packaging Matrix (HP1-2) | ✅ |
| Sort/Filter UI | ✅ batches 1-7 적용 |
| 가격 잠금 + 변경 알람 | ✅ Price Lock |
| Material documents (CoA/MSDS/Halal cert 등 14종) | ✅ |
| Save-first conflict resolution | ✅ |

### 2.5 Suppliers / Clients / Inventory ✅

| 기능 | 상태 |
|---|---|
| Suppliers 독립 관리 + supplier_materials | ✅ Phase 3c |
| Merge duplicate suppliers | ✅ |
| Clients 독립 관리 + Formula/Quote/Doc 일람 | ✅ |
| Inventory Lots (입고/출고/유효기간/QR) | ✅ |
| Packaging Inventory | ✅ |

### 2.6 Products (`/formulation/products`) ✅

| 기능 | 상태 |
|---|---|
| 7-탭 (Info/BOM/Formula/Reg/Safety/Pkg/Knowledge) | ✅ |
| Product Versions manifest (Phase C) | ✅ auto-bump |
| Full BOM print page | ✅ |
| Phase B same actions menu | ✅ |

### 2.7 Packaging & Printing (`/packaging`) ✅

| 기능 | 상태 |
|---|---|
| 2-탭 (Packaging/Printing) | ✅ |
| Primary/Secondary/Other × Outsource/InHouse | ✅ |
| `is_active` soft delete | ✅ |
| Sort/Filter on tabs | ✅ |

### 2.8 Quotes (`/quotes`) ✅

| 기능 | 상태 |
|---|---|
| 10항목 원가 계산 | ✅ `src/lib/quote/compute-rm.ts`, `compute-bom-cost.ts` |
| 배치 수량 로직 | ✅ |
| ExcelJS 견적서 | ✅ `export-quotation.ts` |
| 카테고리 매칭 + 전체 처방 검색 | ✅ |
| qq_quotation_items_snapshot | ✅ 발행 시점 동결 |

### 2.9 Registration (`/registration/*`) ✅

| 기능 | 상태 |
|---|---|
| BPOM 6단계 + Halal 5단계 | ✅ |
| DIP Gathering (자동 수집 → ZIP) | ✅ |
| Bulk certificate upload + 자동 분류 | ✅ |
| Multi-cert (BPOM 4 / HALAL 3 슬롯) | ✅ |
| Submission snapshots + Approve | ✅ |
| Material/Packaging/Company docs | ✅ |
| 만료 알림 + Drift detection | ✅ Phase D |
| Inline cert metadata in bulk modal | ✅ |
| Approve bundles Submit Set | ✅ |
| 5 demo seeds (BPOM/Halal/Quote/Product/Packaging) | ✅ |

### 2.10 Safety Assessment (`/safety`) ✅

| 기능 | 상태 |
|---|---|
| 7-단계 파이프라인 | ✅ `src/lib/safety-pipeline/` |
| 3-Tier 독성 데이터 (cache → PubChem → Claude) | ✅ |
| MoS 계산 (SCCS) | ✅ |
| 이중언어 보고서 (EN/ID) + DOCX | ✅ |
| Escalations | ✅ |
| AI retry 3회 + parallelize stability | ✅ |
| Safety missing tables 마이그레이션 | ✅ 2026-04-13 |

### 2.11 Label Check (`/label-check`) 🟡

| 기능 | 상태 |
|---|---|
| 13단계 OCR + 검증 파이프라인 | ✅ (PV Supabase) |
| 통합 UI 진입점 | ✅ COSLAB Hub `/label-check` |
| Multi-version + 피드백 | ✅ |
| Product code + label version columns | ✅ |
| **데이터 통합** | 🟠 PV Supabase 별도 — `pv_*` 접두사로 이관 미완 |

### 2.12 Documents (`/documents`) ✅

| 기능 | 상태 |
|---|---|
| 8소스 통합 허브 (Materials/Pkg/Company/Reg/Stability/Sample/Formula/Upload) | ✅ |
| 카테고리 관리 (`/documents/categories`) | ✅ |
| 폴더 업로드 + 일괄 처리 | ✅ |
| 한국어 파일명 + DOMMatrix polyfill | ✅ |
| 청크 업로드 (4.5MB 우회) | ✅ |
| Folder-drop diagnostics | ✅ |
| **AllTypes 필터 누락** | 🟠 backlog |
| **등록일자 컬럼** | 🟠 backlog |
| **Doc# 출처 표시** | 🟠 backlog |

### 2.13 Talent (`/talent/*`) ✅ — 설계서에 없던 신설 모듈

| 기능 | 상태 |
|---|---|
| People + Organization (Teams/Positions/JD) | ✅ Phase A |
| Visual Org Chart | ✅ |
| Programs + Curricula 라이브러리 | ✅ Phase B |
| OJT (8 sessions) + Regular (12) 분리 | ✅ |
| Sessions 5단계 라이프사이클 | ✅ |
| 월간 캘린더 (Jakarta TZ 통일) | ✅ 2026-05-04 |
| Quizzes + Trainee Taker UI | ✅ |
| 단답 자동채점 (RT-01 rules) | ✅ |
| Quiz Retest 워크플로 | ✅ |
| Per-participant Self-Eval + Trainer Eval | ✅ |
| Action-first /talent/me | ✅ |
| Admin approval modal | ✅ |
| External Training (개인 + 벌크) | ✅ |
| External Training material slots + group_id | ✅ |
| Library + 인라인 편집 | ✅ |
| Reports XLSX | ✅ Phase D |
| Team progress dashboard | ✅ |
| Curricula default_for_roles (multi-role) | ✅ |
| Demo trainee seed | ✅ 2026-04-30 |

### 2.14 Knowledge / Chat / Upload ✅

| 기능 | 상태 |
|---|---|
| Knowledge Entries 8유형 + Explorer | ✅ |
| AI Chat (knowledge-aware) | ✅ |
| Upload + AI 파싱 + 중복 감지 + 재시도 | ✅ |
| Upload UX 개선 + 안정화 | ✅ 2026-04-30 |
| Voice & Photo (모바일 PWA) | 🟡 인프라 OK, 활용 적음 |
| 30 Q&A onboarding seed | ✅ |

### 2.15 AI 기능 계층

| 페이지/기능 | 상태 |
|---|---|
| `/ai-studio` (4개 카드 + pre-baked demo) | ✅ |
| `/smart-advisor` (knowledge-aware 챗) | ✅ |
| Help-aware Smart Advisor (Quick Start) | ✅ |
| `/ai-formulator` (브리프 → 처방 초안) | ✅ Production+Sample 인벤토리 활용 |
| **smart_rules 규칙 엔진** | ⚫ 보류 (2026-05-05 Sunny 결정) |
| **변경 감지 → 자동 추천 알림** | ⚫ 보류 (Smart Advisor와 함께) |
| Documents 자동 분류 (CoA/MSDS) | ✅ Claude Vision |
| 인증서 파일명 → 슬롯 분류 | ✅ |

### 2.16 Production / Proposals / Projects / Executive

| 모듈 | 상태 |
|---|---|
| Production (Knowledge Hub 보존) | ✅ |
| Proposals (제안서 + 결과) | ✅ |
| Projects (간소 페이지) | 🟡 Phase B/C 보류 |
| Executive (snapshot + period split + blockers + pipeline movements) | ✅ |
| Executive Report DOCX/PDF | ✅ 2026-04-28 재작성 |

### 2.17 Onboarding / Welcome / Help ✅

| 기능 | 상태 |
|---|---|
| /welcome 8-step interactive walkthrough + progress | ✅ |
| Inline previews | ✅ |
| /help 30 Q&A + Quick Start | ✅ |
| WelcomeTip popup | ✅ |
| Auto-redirect for new users | ✅ |
| Demo seeds (6개: base/quote/product/packaging/talent/trainee) | ✅ |
| 영상 walkthrough Phase A (시나리오 1-10) | ✅ |
| 영상 Phase B — 11 lifecycle + talent chapters | ✅ |
| Playwright capture 자동화 | ✅ scripts/capture-onboarding-recordings.mjs |

### 2.18 인프라 / 공유 서비스

| 항목 | 상태 |
|---|---|
| jakarta.ts TZ 헬퍼 | ✅ + jakartaMonthAnchor (2026-05-04) |
| Audit Trail (`/settings/audit-trail`) | ✅ Phase 1+2 expansion |
| Change Control (snapshots/locks/audit) | ✅ Phase 1-4 |
| Price Impact (auto alerts + revert resolution) | ✅ |
| Cross-entity Alerts (Phase F) | ✅ |
| Drift Detection (BPOM/HALAL Phase D UI) | ✅ |
| Backup auto cron (일일 18:00 UTC) | ✅ /api/backup/auto |
| Rate limiter | ✅ |
| Idle timeout | ✅ |
| TOTP 2FA | ✅ optional |
| BETA 배지 + scanner signals | ✅ |
| PWA Service Worker | ✅ |
| IndexedDB offline 큐 | ✅ 인프라 |

---

## 3. 보안 정책 v1.5.0 — 100% 완료

| 항목 | 상태 |
|---|---|
| 키 노출 점검 + GitHub secret scanning | ✅ |
| JWT secret + CRON secret 분리 | ✅ |
| 5xx 응답 sanitize (PII 마스킹) | ✅ |
| AI 가이드라인 감사 (대웅 그룹) | ✅ |
| SQL 인젝션 차단 | ✅ |
| Rate limit | ✅ |
| Idle timeout | ✅ |
| 2FA TOTP | ✅ optional |
| **e-Signature** | 🟠 미구현 |
| **AI 보안 백로그 7건** | 🟠 backlog |

---

## 4. 마이그레이션 적용 현황

총 76개 적용 (`supabase/migrations/`).

**그룹별 분류**

| 그룹 | 개수 | 대표 |
|---|---|---|
| Phase 3 (스키마) | 4 | phase3a/b/c/d (Formula, Stability, Operations, Documents) |
| Talent | 19 | phase_a~e + sessions + quizzes + WIS seed + external trainings |
| Stability | 6 | parameters expand + verdict + selected params + result locks + 002c |
| 변경통제 | 6 | audit_log_enhance, audit_trail_expansion, change_control_phase2, formulations_lock, formula_snapshots+RLS |
| Registration | 5 | submissions, multi_certificates, drift_acknowledge, change_alerts, registration_folders_snapshot |
| Onboarding/Demo | 9 | hub_accounts_onboarding, demo_seed × 7, onboarding_knowledge_seed |
| Sample | 4 | sample_materials, sample_first, sample_qc_specs, sample_material_documents |
| 공유 | 8 | base_formulas_enhance, base_name_categories, security_hardening, totp_2fa, qq_tables, pv_tables, projects, formulations_client_fields |
| 백필 | 3 | backfill_rt_month2, backfill_stability_result_locks, fix_regular_session_dates |
| 기타 | 12 | merge suppliers, packaging is_active, products versions, etc. |

---

## 5. 기능 격차 (Gap List)

### 5.1 우선순위 1 (Sprint 1)

- [ ] /upload 재검토 (남은 이슈)
- [ ] /documents AllTypes 필터 누락
- [ ] /documents 등록일자 컬럼
- [ ] /documents Doc# 출처 표시
- [ ] Talent 런타임 검증
- [ ] Calendar 통합 후 사용자 검증 (사용자 보고에 따라)

### 5.2 우선순위 2 (Sprint 2-3)

- [ ] AI 보안 백로그 7건 적용 (`project_ai_security_backlog.md`)
- [ ] e-Signature 구현
- [ ] 노란불 4건 잔여 처리

### 5.2-bis 보류 (Sunny 결정)

- ⚫ **Smart Advisor 규칙 엔진** — `smart_rules` 테이블 + trigger matcher (2026-05-05 보류, 향후 재개)
- ⚫ **manager/approver 4롤 모델** — 2026-04-28 deferred (QA 매니저 채용 시점에 5h 작업 예상)

### 5.3 장기 (Backlog)

- [ ] Project Hub 본격 구축 (Phase B/C 보류 해제 시)
- [ ] Package Validator 데이터의 `pv_*` 접두사로 통합 이관
- [ ] 시맨틱 검색 (pg_vector or 외부 인덱스)
- [ ] Voice/Photo 입력 활용 사례 누적 후 강화
- [ ] Knowledge Hub Phase A-5 (제안서 자동 초안 PPTX)
- [ ] Knowledge Hub Phase A-6 (시맨틱 검색)

### 5.4 폐기

- ~~1주 통합 로드맵~~ — 실제 ~6주 소요됨
- ~~AI 자동 트리거~~ — 항상 사용자 확인 단계 유지
- ~~Cross-DB 자동 동기화~~ — 단일 DB로 통합되어 불필요
- ~~Group C (Label 검증 강화)~~ — 현재 Package Validator로 충분
- ~~Project 위클리 브리핑~~ — Executive Report로 대체

---

## 6. 최근 활동 요약 (2026-04-28 ~ 05-04, 1주)

| 일자 | 커밋 수 | 주요 변경 |
|---|---|---|
| 04-28 | 6 | Sunny 피드백 4건 + Phase 4 soft delete + Phase 3 unlock + Executive Report 전면 재작성 |
| 04-30 | 8 | Talent 외부교육 벌크/3슬롯 + Sample DPI 수동수정 + /documents 8소스 통합 + /upload 안정화+UX |
| 04-30 (2nd) | — | 영상 v1→v4 재설계 + 마이그레이션 7개 + Phase B 영상 캡처 + 시나리오 1A/B/C |
| 05-04 | 2 | Calendar Jakarta TZ 통합 (label + today 비교 통일) |
| 05-05 | 6 | 통합 트래시 정책 6개 모듈 + Products entity 격상 (Phase B-1) + Trash Phase A 통합 |
| 05-06 | 2 | 백로그 A-D 10건 + 통합 `/trash` 페이지 (11개 모듈 aggregator) + Trash Phase B 마이그레이션 |
| 05-07 | — | Trash Phase D — material_documents/packaging_documents/sample_material_documents 보조 테이블 |

총 **477 커밋** 누적 (2026-03-30 첫 커밋부터).

---

## 7. 사용자 (운영자) 입장에서 점검 체크리스트

### 7.1 Daily 사용 확인

- [ ] `/dashboard`, `/talent` 진입 시 오늘 날짜 정확
- [ ] 캘린더 헤더 = 현재 월
- [ ] "오늘" 하이라이트 = 자카르타 오늘
- [ ] "이번 주" 필터 정확
- [ ] 백업 cron 실행 (Settings → Backup 로그)

### 7.2 핵심 워크플로우

- [ ] 신규 Formula 생성 → Phase 추가 → Specs 입력 → Confirmed 전이 → 자동 잠금 확인
- [ ] Stability Test 생성 → 셀 입력 → 자동 잠금 → 1-Month Confirmation → Final Confirmation
- [ ] Sample Manufacturing Sheet 출력 → Phase 이름 포함 확인
- [ ] BPOM 폴더 생성 → 인증서 bulk upload → 슬롯 자동 분류 → Approve
- [ ] 원료 단가 변경 → 영향 처방 알람 → revert 시 알람 자동 해소
- [ ] Talent 세션 평가 → Trainee Self-Eval → Trainer Score → Approve

### 7.3 신뢰성

- [ ] /upload 대량 파일 (한국어 파일명 포함) 정상
- [ ] PDF 파싱 실패 시 filename fallback 동작
- [ ] AI Studio 데모 정상 재생
- [ ] Onboarding 영상 시나리오 1-10 정상 재생

---

## 8. 참고

- 본 문서는 일자 기준 스냅샷이다. 변경 시 새 일자로 갱신.
- 자세한 설계 의도/결정은 [`COSLAB-HUB-DESIGN.md`](./COSLAB-HUB-DESIGN.md)
- 과거 설계서 4개는 본 문서로 통합되어 폐기됨

<!-- SOURCE_FILE_6_END: COSLAB-HUB-STATUS.md -->

---
