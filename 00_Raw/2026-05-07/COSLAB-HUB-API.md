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
