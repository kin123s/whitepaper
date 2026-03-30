# Technical Appendix: Rule Specifications & Implementation Details

> **Companion to:** 01-ai-agent-governance-architecture.md  
> **Version:** 1.0  
> **Date:** 2026-03-30

---

## Table of Contents

- [A. Full File Tree — 전체 디렉토리 맵](#a-full-file-tree)
- [B. Core Rule Specifications — 9개 코어 규칙 명세](#b-core-rule-specifications)
- [C. Domain Rule Specifications — L2 도메인 규칙 명세](#c-domain-rule-specifications)
- [D. MCP Tool Catalog — 도구 카탈로그](#d-mcp-tool-catalog)
- [E. State Schema — 상태 스키마 정의](#e-state-schema)
- [F. Sync Script Flow — 동기화 스크립트 흐름도](#f-sync-script-flow)
- [G. Code Samples — 실제 규칙 파일 발췌](#g-code-samples)
- [H. Token Budget Calculator — 토큰 예산 계산표](#h-token-budget-calculator)

---

## A. Full File Tree

```
AI Agent Governance Architecture
│
├─ L1: GLOBAL HUB (company-ai-setup repository)
│  │
│  ├─ .github/copilot-instructions.md    ← GitHub Copilot 진입점 (SYNCED)
│  ├─ CLAUDE.md                          ← Claude 진입점 (SYNCED)
│  │
│  ├─ rules/                             ← 9 Core Rules (SSOT 원본)
│  │  ├─ guardrails.md          trigger: always       v1.5
│  │  ├─ language.md            trigger: always       v1.0
│  │  ├─ agent-workflow.md      trigger: model_decision  v1.3
│  │  ├─ coding.md              trigger: model_decision  v1.3
│  │  ├─ security.md            trigger: model_decision  v1.1
│  │  ├─ performance-optimization.md  trigger: model_decision  v1.0
│  │  ├─ prompting.md           trigger: model_decision  v1.1
│  │  ├─ git.md                 trigger: model_decision  v1.0
│  │  └─ custom-registry.md     trigger: model_decision  v1.0
│  │
│  ├─ roles/                             ← 4 Agent Personas
│  │  ├─ planner_agent.md       /plan
│  │  ├─ worker_agent.md        /run, /hotfix
│  │  ├─ reviewer_agent.md      /review
│  │  └─ infra_agent.md         /PR, /infra
│  │
│  ├─ workflows/                         ← 9 Macro Workflows
│  │  ├─ plan.md                계획 수립
│  │  ├─ run.md                 코드 실행
│  │  ├─ hotfix.md              긴급 수정 (≤3 파일)
│  │  ├─ review.md              코드 리뷰
│  │  ├─ PR.md                  커밋/PR/배포
│  │  ├─ handoff.md             세션 인수인계
│  │  ├─ FM.md                  기능 관리
│  │  ├─ help.md                도움말/안내
│  │  └─ bump-version.md        버전 범프 자동화
│  │
│  ├─ templates/                         ← 출력 양식
│  │  ├─ plan-task.md
│  │  ├─ implement-feature.md
│  │  ├─ review-code.md
│  │  ├─ handover-report.md
│  │  └─ completed-tasks.md
│  │
│  ├─ protocols/
│  │  └─ state-schema.json      세션 상태 JSON Schema
│  │
│  ├─ bin/
│  │  ├─ ai-sync.sh             L1→L2 동기화 스크립트
│  │  ├─ bump-version.sh         버전 범프 (Linux)
│  │  └─ bump-version.ps1        버전 범프 (Windows)
│  │
│  ├─ manifest.json              버전 + 파일별 해시
│  └─ package.json               npm 호환 버전
│
├─ L2: PROJECT LAYER (각 프로젝트의 .ai/)
│  │
│  ├─ rules.md                   ← L1 라우팅 허브 사본 + L2 도메인 섹션
│  │
│  ├─ rules/                     ← L1 사본 + L2 전용 규칙
│  │  ├─ [L1 사본 9개]           ← 상단 L1, 하단 <!-- L2 --> 보호구역
│  │  ├─ local-coding.md         L2 전용: 프로젝트별 코딩 규칙
│  │  ├─ local-db.md             L2 전용: DB 접근 정책
│  │  ├─ local-security.md       L2 전용: 인증/권한/격리
│  │  ├─ mcp-usage.md            L2 전용: MCP 강제 검색 정책
│  │  ├─ cto-dispatch.md         L2 전용: CTO 오케스트레이션
│  │  ├─ project-paths.md        L2 전용: 멀티 프로젝트 경로 맵
│  │  └─ skills-governance.md    L2 전용: 스킬 등록 정책
│  │
│  ├─ context/                   ← 프로젝트 참조 문서
│  │  ├─ project.md              프로젝트 개요·목적·3-Tier 구조
│  │  ├─ architecture.md         시스템 설계 참조
│  │  ├─ api-endpoints.md        MCP 서버 라우트·도구 목록
│  │  ├─ database-TABLE1.md      DB 스키마 인덱스
│  │  └─ coding-style/           멀티 언어 코드 패턴
│  │
│  ├─ roles/                     ← 로컬 에이전트 오버라이드
│  │  ├─ local-crm_agent.md
│  │  ├─ local-frontend_agent.md
│  │  └─ local-infra_agent.md
│  │
│  ├─ workflows/                 ← 로컬 워크플로우 확장
│  │  ├─ local-PR.md
│  │  ├─ local-jira-create.md
│  │  └─ local-jira-comment.md
│  │
│  ├─ templates/                 ← 출력 양식 (L1 사본)
│  ├─ guidelines/                ← 베스트 프랙티스
│  ├─ protocols/                 ← state-schema.json
│  │
│  ├─ state.json                 ← 현재 세션 트래커
│  ├─ state/
│  │  ├─ context_registry.json   토큰 예산 맵
│  │  ├─ global-task-board.md    KPI이벤트 보드
│  │  └─ sessions/               ← L3 세션 디렉토리
│  │
│  ├─ manifest.json              L1 버전·해시 추적
│  ├─ local-manifest.json        L2 자체 추적
│  └─ sync-report.json           동기화 이력·결과
│
├─ L3: TASK/SESSION LAYER (state/sessions/)
│  ├─ GA-5063/
│  │  ├─ state.json              2문장 압축 상태
│  │  ├─ context.md              Plan 단계 산출물
│  │  └─ research_report.md      배경 조사 결과
│  ├─ GA-5064.md                 단일 파일 세션
│  ├─ CORE-SYNC-20260330/        진행 중 세션
│  └─ RAG_SETUP_001/             다단계 세션
│
└─ ANCILLARY
   ├─ .agents/                   ← VS Code Agent 시스템
   │  └─ skills/
   │     ├─ app-workflow/SKILL.md     워크플로우 매핑 스킬
   │     ├─ legacy-php-analysis/SKILL.md  레거시 PHP 분석 스킬
   │     ├─ mcp-rag-tools/SKILL.md     MCP RAG 도구 스킬
   │     └─ database-knowledge/        오프라인 DB 스키마
   │
   └─ MCP Server (ai-mcp-server)
      ├─ src/tools/              8개 MCP 도구 구현
      ├─ sidecar/                Python 임베딩 서버
      └─ docker-compose.yml     컨테이너 오케스트레이션
```

---

## B. Core Rule Specifications

### B.1 guardrails.md (trigger: always)

**목적**: AI 에이전트의 절대 금지 행동 목록 및 환각 통제

| # | 가드레일 | 세부 내용 |
|---|---------|----------|
| 1 | 불필요한 리팩토링 금지 | 요청 없이 코드 스타일 일괄 포맷팅 불가 |
| 2 | 프레임워크 교체 금지 | 사용 중인 라이브러리 무단 변경 불가 |
| 3 | DB 스키마 변경 금지 | 임의 DDL 변경 및 대규모 파일 이동 불가 |
| 4 | 오버엔지니어링 금지 | 요청된 기능과 무관한 주변 코드 수정 불가 |
| 5 | DB 변경 직접 실행 금지 | INSERT/UPDATE/DELETE는 사용자 승인 필수 |
| 6 | 시크릿 접근 금지 | `.env`, 비밀 파일 직접 접근 불가 |
| 7 | 타 세션 무단 수정 금지 | 다른 세션 디렉토리 파일 수정 불가 |
| 8 | AI 규칙 참조 제한 | `.ai/` 폴더만 SSOT, `docs/` 아카이브 규칙 적용 금지 |
| 9 | 버전 Bump 스크립트 필수 | L1 규칙 수정 시 `bump-version.sh` 실행 의무 |

**환각 방지 메커니즘**:
- Negative Prompt 최하단 배치 (Recency Bias 활용)
- DB 스키마 추측 원천 차단 (DDL 참조 강제)
- 레거시 파괴 방지 체크리스트 (PHP Static 캐싱, TS Nullish Coalescing)

### B.2 language.md (trigger: always)

**목적**: 언어 및 커뮤니케이션 표준

- 1차 언어: 한국어 (코드 주석, 문서, 커밋 메시지)
- 기술 용어: 원문 병기 (예: "환각(Hallucination)")
- 변수명/함수명: 영문 camelCase 유지

### B.3 agent-workflow.md (trigger: model_decision)

**목적**: 에이전트 3단계 워크플로우, 토큰 Budget, Fail-Fast

핵심 구성요소:
1. **3단계 워크플로우**: Plan → Execute → Verify (실패 시 최대 2회 재시도)
2. **토큰 Budget**: 단일 턴 ~8,000 토큰 이하
3. **Tiered Loading**: T0(항상) → T1(기본) → T1.5(조건부) → T2(필요 시) → T3(금지)
4. **Context Pruning**: Stubbing(시그니처만), Summary(300줄↑)
5. **State Pruning**: 2문장 압축 (현재 에러 + 다음 액션)
6. **Fail-Fast**: 재시도 최대 2~3회, 초과 시 즉시 셧다운

### B.4 coding.md (trigger: model_decision)

**목적**: 코딩 스타일, 아키텍처, 시스템 컨벤션

| 영역 | 핵심 규칙 |
|------|----------|
| 코드 품질 | 순수 함수 지향, 방어적 프로그래밍, 오버엔지니어링 금지 |
| JS/TS | `const`/`let` only, named export, `??` 필수, ESLint/Prettier |
| 패키지 관리 | pnpm 기본, lockfile 필수 커밋 |
| 인프라 | Docker 컨테이너화, 익명 볼륨(node_modules) |
| 언어별 | PHP(보안 리팩토링), Java/Node(모던 패턴), K8s(주석 상세) |

### B.5 security.md (trigger: model_decision)

**목적**: Zero-Trust 보안 정책

| 카테고리 | 규칙 |
|---------|------|
| Zero-Trust | 내부망도 무조건적 신뢰 배제, Mutual TLS/API Key |
| 시크릿 | 환경 변수 주입, .gitignore 즉시 등록, 로그 출력 금지 |
| 네트워크 | Reverse Proxy/WAF 필수, 최소 권한 원칙 |
| SQL Injection | Prepared Statement 필수 |
| XSS | 이스케이프 처리 필수, dangerouslySetInnerHTML 제한 |
| 입력 검증 | Whitelist 기반 우선 |
| 파일 업로드 | 확장자·MIME 검증, 실행 파일 차단 |
| 금지 함수 | `eval()`, 외부 입력 역직렬화, 오류 억제 연산자 |

### B.6 performance-optimization.md (trigger: model_decision)

**목적**: 응답 속도·토큰 효율·실사용 성능 극대화

핵심 구성요소:
- **3-Tier 검색 파이프라인**: Cache(0ms) → Keyword(100ms) → Vector(1,000ms)
- **캐시 전략**: TTL 5분, Max 500항목, Upsert 시 자동 무효화
- **Dynamic Routing**: Level 1(단순) → Level 2(중간) → Level 3(중요)
- **Vector Search 가드레일**: top_k ≤ 5, 유사도 50% 미만 필터링, 전체 스캔 금지
- **KPI**: 응답 1~4초, Vector Search 30% 이하, Cache Hit 50% 이상

### B.7 prompting.md (trigger: model_decision)

**목적**: 프롬프트 엔지니어링 표준

| 기법 | 설명 |
|------|------|
| XML 태그 구조화 | `<context>`, `<instructions>`, `<thinking>`, `<output>`, `<examples>` |
| Chain of Thought | 코드 작성 전 `<thinking>` 태그 내 추론 강제 |
| Few-shot 예제 | 성공/실패 사례를 `<examples>` 포맷으로 제공 |
| Recency Bias 활용 | "절대 금지" 규칙은 프롬프트 최하단 배치 |
| RAG CoT 프로토콜 | MCP 호출 전 `<thinking>` 내 키워드 구체화 강제 |

### B.8 git.md (trigger: model_decision)

**목적**: Git 브랜치 전략 및 커밋 컨벤션

- Main 브랜치 직접 Push **절대 금지**
- PR 필수화 (리뷰 후 Merge)
- Conventional Commits: `feat()`, `fix()`, `docs()`, `style()`, `refactor()`, `test()`, `chore()`
- L2 오버라이드 예시: `feature/kin123s_GA-1234` (팀/부서 기준)

### B.9 custom-registry.md (trigger: model_decision)

**목적**: 도메인별 커스텀 규칙 디스패치

프로젝트가 L2 레벨에서 독자적인 규칙을 등록하고, 해당 도메인 작업 시 자동으로 로드되게 하는 레지스트리 메커니즘.

---

## C. Domain Rule Specifications

L2 프로젝트 전용 규칙 (동기화 비대상):

| # | 파일 | 도메인 | 핵심 정책 |
|---|------|--------|----------|
| 1 | `local-coding.md` | 코딩 | 3영역 DB 격리 (Legacy/ModenApp/Modern) |
| 2 | `local-db.md` | DB | Docker exec MYSQL, config.php 파싱(require 금지), LIMIT 10 |
| 3 | `local-security.md` | 보안 | PHP 세션 vs Keycloak, 업로드 제한 |
| 4 | `cto-dispatch.md` | 오케스트레이션 | 도메인 매칭, 복잡도 판단(Quick/Standard/Full) |
| 5 | `mcp-usage.md` | **MCP (CRITICAL)** | 필수 검색 트리거, STOP→SEARCH→VERIFY 순서, 오프라인 폴백 |
| 6 | `project-paths.md` | 경로 | AI MCP Server / DOMAIN1 / Partner Portal 경로 맵 |
| 7 | `skills-governance.md` | 스킬 | `.agents/skills/` 등록·호출 정책 |

---

## D. MCP Tool Catalog

### D.1 stdio 도구 (MCP Protocol)

| # | 도구명 | 필수 파라미터 | 반환 | 용도 |
|---|--------|-------------|------|------|
| 1 | `search_memory` | domain, query, collection | Ranked docs + score | 벡터 유사도 검색 |
| 2 | `query_context` | domain, path, index | Static file index | 정적 컨텍스트 조회 |
| 3 | `upsert_memory` | domain, content, collection | doc_id + vector | 새 지식 저장 |
| 4 | `manage_collection` | action, collection | Status | 컬렉션 생성/조회/삭제 |
| 5 | `get_file_signature` | path | Hash + metadata | 파일 변경 감지 |
| 6 | `get_domain_status` | domain | Status JSON | 도메인 세션 상태 |
| 7 | `save_directive` | domain, session_id, content | session_id + path | 지시사항 저장 |
| 8 | `update_session_state` | domain, session_id, state | Updated timestamp | 세션 상태 갱신 |

### D.2 Sidecar HTTP 엔드포인트 (Port 8100)

| # | 경로 | Method | 입력 | 출력 | 용도 |
|---|------|--------|------|------|------|
| 1 | `/health` | GET | — | `{ status, models }` | 서비스 상태 |
| 2 | `/embed` | POST | `{ text }` | `{ vector: [...] }` | 단일 텍스트 임베딩 |
| 3 | `/upsert` | POST | `{ doc, collection }` | `{ doc_id, status }` | 문서 임베딩 + 저장 |
| 4 | `/query` | POST | `{ query, top_k }` | `{ results: [{doc, score}] }` | 시맨틱 검색 |
| 5 | `/reindex` | POST | `{ collection }` | `{ indexed_count }` | 컬렉션 재인덱싱 |
| 6 | `/collections` | GET | — | `{ collections, total_docs }` | 컬렉션 목록 |
| 7 | `/docs` | GET | — | OpenAPI JSON | API 문서 |

### D.3 도메인 격리

MCP 도구 호출 시 `domain` 파라미터로 데이터 격리:

| Domain | 대상 프로젝트 | 데이터 범위 |
|--------|-------------|-----------|
| `DOMAIN1` | DOMAIN1 ERP | TABLE1, TABLE2, TABLE3 + 레거시 PHP |
| `DOMAIN2` | DOMAIN2 | 타 데이터 + React/PHP ModenApp |

---

## E. State Schema

### E.1 state.json (v2.5)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "AI 세션 상태 스키마 (v2.5)",
  "description": "전체 파일 크기 2KB 이내 필수",
  "required": [
    "session_id",
    "status",
    "created_at",
    "last_updated",
    "active_agent",
    "context_summary"
  ],
  "properties": {
    "session_id": {
      "type": "string",
      "description": "이슈 ID 또는 session_YYYYMMDD_HHMM"
    },
    "status": {
      "enum": ["created", "active", "blocked", "completed", "failed", "archived"]
    },
    "active_agent": {
      "properties": {
        "id": { "description": "현재 활성 에이전트 (예: worker_agent)" },
        "phase": { "enum": ["plan", "execute", "verify"] }
      }
    },
    "context_summary": {
      "type": "string",
      "maxLength": 200,
      "description": "현재 에러 + 다음 액션 (2문장 이내)"
    },
    "retry_count": {
      "type": "integer",
      "maximum": 3,
      "description": "현재 재시도 횟수"
    }
  }
}
```

### E.2 context_registry.json

토큰 예산 추적 레지스트리:

```json
{
  "tier_0": {
    "rules.md":            { "bytes": 12212, "est_tokens": 2035 },
    "rules/guardrails.md": { "bytes": 5522,  "est_tokens": 920 },
    "_subtotal_tokens": 2955
  },
  "tier_1": {
    "roles/planner_agent.md":  { "est_tokens": 378 },
    "roles/worker_agent.md":   { "est_tokens": 328 },
    "roles/reviewer_agent.md": { "est_tokens": 359 },
    "roles/infra_agent.md":    { "est_tokens": 272 },
    "state.json (avg)":        { "est_tokens": 83 }
  },
  "budget_summary": {
    "lite_mode_baseline": "~3,388 tokens",
    "lite_mode_max":      "~5,388 tokens",
    "full_mode_max":      "토큰 제한 없음"
  }
}
```

### E.3 manifest.json

```json
{
  "version": "6.5.0",
  "published_at": "2026-03-30T12:44:38Z",
  "layer_1_core": [
    "rules/guardrails.md", "rules/language.md",
    "rules/agent-workflow.md", "rules/coding.md",
    "rules/security.md", "rules/performance-optimization.md",
    "rules/prompting.md", "rules/git.md",
    "rules/custom-registry.md"
  ],
  "layer_1_roles": [
    "roles/planner_agent.md", "roles/worker_agent.md",
    "roles/reviewer_agent.md", "roles/infra_agent.md"
  ],
  "rules": {
    "guardrails.md": { "hash": "updated_v6.4.4", "version": "1.5" },
    "agent-workflow.md": { "hash": "f7f3f9dd63c4", "version": "1.3" },
    "..."
  }
}
```

---

## F. Sync Script Flow

### F.1 동기화 프로세스 (ai-sync.sh)

```
┌───────────────────────────────────────────────────────────┐
│ 1. 파라미터 파싱                                            │
│    --l2-hub=/path/to/hub/.ai                               │
│    --target-dir=.ai                                   │
└───────────────┬───────────────────────────────────────────┘
                ▼
┌───────────────────────────────────────────────────────────┐
│ 2. manifest.json 로드 (Hub ↔ Local 비교)                    │
│    - Hub version vs Local version                           │
│    - 파일별 hash 비교                                        │
└───────────────┬───────────────────────────────────────────┘
                ▼
┌───────────────────────────────────────────────────────────┐
│ 3. 변경된 파일만 선택적 동기화                                │
│    ┌─────────────────────────────────────────────────┐     │
│    │ FOR EACH changed file:                          │     │
│    │   IF file has <!-- BEGIN L2 --> section:        │     │
│    │     → Replace ONLY L1 section (above L2 marker) │     │
│    │     → Preserve L2 section intact                │     │
│    │   ELSE:                                         │     │
│    │     → Full file replacement                     │     │
│    └─────────────────────────────────────────────────┘     │
└───────────────┬───────────────────────────────────────────┘
                ▼
┌───────────────────────────────────────────────────────────┐
│ 4. manifest.json 갱신                                       │
│    - 파일별 hash 업데이트                                    │
│    - version 동기화                                          │
└───────────────┬───────────────────────────────────────────┘
                ▼
┌───────────────────────────────────────────────────────────┐
│ 5. sync-report.json 기록                                    │
│    - 동기화 시각, 변경 파일 목록, 성공/실패 건수              │
└───────────────────────────────────────────────────────────┘
```

### F.2 L2 보호 구역 패턴

```markdown
<!-- 🚨 SYNCED from company-ai-setup (L1: DO NOT EDIT) 🚨 -->

[L1 공통 규칙 영역]
← ai-sync.sh가 이 영역만 덮어씀

<!-- BEGIN L2 (DOMAIN RULES) -->
← 이 마커 아래부터는 절대 건드리지 않음

[프로젝트 고유 규칙]

<!-- END L2 -->
```

### F.3 버전 범프 플로우

```
사용자: ./bin/bump-version.sh patch "guardrails: 조항 추가"
                │
                ▼
        ┌───────────────┐
        │ 현재 버전 읽기  │  manifest.json → "6.4.16"
        └───────┬───────┘
                ▼
        ┌───────────────┐
        │ 버전 계산      │  patch → "6.4.17"
        └───────┬───────┘
                ▼
        ┌───────────────────────────┐
        │ 동시 업데이트 (원자적)      │
        │  • manifest.json.version  │
        │  • package.json.version   │
        │  • changelog 행 추가      │
        └───────┬───────────────────┘
                ▼
        ┌───────────────┐
        │ 검증           │  두 파일의 version 일치 확인
        └───────────────┘
```

---

## G. Code Samples

### G.1 guardrails.md — Negative Prompt 배치 패턴

```markdown
<critical_guardrails>

> [!CAUTION]
> **🔒 보안 최우선 원칙**: 회사의 민감 정보(DB 접속 정보, API Key,
> 내부 인프라 주소, 사용자 개인정보 등)를 **절대** 외부 LLM으로
> 전송하거나 코드·커밋에 노출하지 마시오.

## 공통 가드레일 (L1)

1. 불필요한 전체 프로젝트 리팩토링을 **절대** 하지 마시오.
2. 프레임워크나 라이브러리 교체를 **절대** 하지 마시오.
...
9. L1 규칙 변경 시 반드시 bump-version 스크립트 실행

---

## 🛑 제약사항 및 환각 통제

### 1. Negative Prompt 후면 배치 룰
"절대 하지 말아야 할 것"은 시스템 프롬프트의 **가장 마지막**에 배치

### 2. DB 스키마 환각 원천 차단
- 반드시 확인된 DDL 스키마를 참조하여 쿼리 작성

</critical_guardrails>
```

### G.2 mcp-usage.md — 강제 검색 정책

```markdown
<critical_mcp_policy>

> [!CAUTION]
> **CRITICAL:** 당신은 사내 ERP의 레거시 PHP 로직이나 DB 스키마를
> 완벽히 알지 못합니다. **절대 추측하여 코드를 작성하지 마십시오.**

## Trigger Conditions (필수 검색 조건)

### DB/테이블 키워드
- TABLE1, TABLE2,  ...

### Action Sequence
1. **STOP**: 즉각적인 답변 생성을 중단
2. **SEARCH**: search_memory 또는 query_context 호출
3. **VERIFY**: 검색된 Context로 답변 구성
4. **CITE**: 출처 명시

</critical_mcp_policy>
```

### G.3 agent-workflow.md — Tiered Loading

```markdown
## Tiered Loading (성능 최적화 지시서 §5 반영: Lazy Load)

| Tier | 로딩 시점 | 대상                         |
|------|-----------|------------------------------|
| T0   | 항상      | rules.md + guardrails.md      |
| T1   | 기본      | 본인 역할 파일 + state.json    |
| T1.5 | 조건부    | context.md (세션 이어받기 시)   |
| T2   | 필요 시   | rules/ 서브파일, context/      |
| T3   | 금지      | 타 에이전트 역할, 무관 가이드   |

### State Pruning
state.json의 context_summary를 업데이트할 때,
**"현재 에러"**와 **"다음 액션"** 딱 2문장으로만 압축
```

### G.4 prompting.md — RAG CoT 프로토콜

```markdown
<rag_search_protocol>
search_memory나 query_context를 호출하기 전,
반드시 <thinking> 태그를 열고 다음을 고민하십시오:

1. 내가 찾고자 하는 과거 패턴의 핵심 키워드는 무엇인가?
2. 결과가 너무 많이 나오지 않도록 검색어를 구체화했는가?
   (예: "DB 에러" ❌ → "ModenApp Model PDO Connection Error" ⭕)
</rag_search_protocol>
```

---

## H. Token Budget Calculator

### H.1 모드별 예산 계산

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Lite Mode (일반 작업)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T0  rules.md            2,035 tokens  ██████████████
T0  guardrails.md         920 tokens  ██████
T1  역할 파일 (avg)       340 tokens  ██
T1  state.json             83 tokens  ▌
────────────────────────────────────────────────────
Baseline                3,378 tokens

+ T2 (선택적, 최대 3개)
    agent-workflow.md   1,453 tokens  ██████████
    coding.md             769 tokens  █████
    security.md           536 tokens  ███
────────────────────────────────────────────────────
Maximum                 6,136 tokens
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Full Mode (복잡한 계획)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T0  (동일)              2,955 tokens
T1  역할 + state          423 tokens
T1.5 context.md           333 tokens  (세션 이어받기)
T2  서브파일 3개        ~2,000 tokens
────────────────────────────────────────────────────
Total                  ~5,711 tokens
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### H.2 토큰 비용 산출 기준

```
1 토큰 ≈ 6 bytes (한국어 기준)
1 토큰 ≈ 4 bytes (영문 기준)

계산식:
  est_tokens = file_bytes ÷ 6 (한국어 혼합 문서)
```

### H.3 전체 규칙 파일 토큰 맵

| 파일 | Bytes | Tokens(추정) | Tier | 비고 |
|------|-------|-------------|------|------|
| rules.md | 12,212 | 2,035 | T0 | 라우팅 허브 |
| guardrails.md | 5,522 | 920 | T0 | 절대 금지 |
| agent-workflow.md | 8,720 | 1,453 | T2 | 워크플로우 |
| coding.md | 4,611 | 769 | T2 | 코딩 규칙 |
| security.md | 3,217 | 536 | T2 | 보안 |
| custom-registry.md | 3,460 | 577 | T2 | 커스텀 |
| performance-optimization.md | 3,011 | 502 | T2 | 성능 |
| git.md | 1,696 | 283 | T2 | Git |
| prompting.md | 1,327 | 221 | T2 | 프롬프팅 |
| language.md | 1,277 | 213 | T0 | 언어 |
| **합계** | **45,053** | **7,509** | — | — |

---

*© 2026 ycy0922. 본 문서는 개인 포트폴리오 목적으로 작성되었으며, 사내 기밀 정보는 포함되어 있지 않습니다.*
