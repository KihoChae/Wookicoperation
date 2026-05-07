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
