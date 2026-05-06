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
