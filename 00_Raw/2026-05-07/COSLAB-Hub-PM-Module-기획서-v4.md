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
