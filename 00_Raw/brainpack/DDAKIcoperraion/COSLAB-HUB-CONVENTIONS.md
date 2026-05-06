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
