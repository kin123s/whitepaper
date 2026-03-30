# AI Agent Governance Architecture: 3-Layer Rule System for Enterprise IDE Copilots

> **Author:** ycy0922
> **Version:** 1.0  
> **Date:** 2026-03-30  
> **Status:** Portfolio Document

---

## Abstract

대규모 언어모델(LLM) 기반 코딩 에이전트(GitHub Copilot, Claude, Cursor 등)가 실제 기업 프로덕션 코드베이스에 투입될 때, "환각(Hallucination)으로 인한 DB 스키마 오류", "컨텍스트 유실로 인한 세션 단절", "프로젝트 간 규칙 비일관성" 등의 구조적 문제가 발생한다. 본 백서는 이러한 문제를 해결하기 위해 설계·구축·운영한 **3-Layer AI Agent Governance Architecture**를 소개한다.

이 아키텍처는 전사 공통 규칙(L1)을 중앙 허브에서 관리하고, 프로젝트별 도메인 지식(L2)으로 확장하며, 태스크 단위 세션 상태(L3)로 실행을 추적하는 계층 구조다. MCP(Model Context Protocol) 기반 RAG 파이프라인과 결합하여, AI 에이전트가 "추측 없이 검색하고, 규칙 안에서 행동하며, 실패 시 즉시 멈추는" 통제된 자율성을 구현한다.

---

## Table of Contents

1. [Problem Statement — 왜 AI 에이전트에 거버넌스가 필요한가](#1-problem-statement)
2. [Design Principles — 6가지 설계 원칙](#2-design-principles)
3. [3-Layer Architecture — L1 · L2 · L3 계층 구조](#3-3-layer-architecture)
4. [Rule Taxonomy & Trigger System — 규칙 분류 체계](#4-rule-taxonomy--trigger-system)
5. [Agent Persona & Workflow Choreography — 역할 분리와 워크플로우](#5-agent-persona--workflow-choreography)
6. [Hub-and-Spoke Sync — 규칙 배포 메커니즘](#6-hub-and-spoke-sync)
7. [MCP Integration & RAG Pipeline — 지식 검색 체계](#7-mcp-integration--rag-pipeline)
8. [State Management & Session Handoff — 상태 관리와 세션 인수인계](#8-state-management--session-handoff)
9. [Security & Guardrails — 안전 장치 설계](#9-security--guardrails)
10. [Results & Lessons Learned — 성과와 교훈](#10-results--lessons-learned)

---

## 1. Problem Statement

### 1.1 AI 코딩 에이전트의 구조적 한계

2025년 이후 GitHub Copilot, Cursor, Claude 등의 AI 코딩 에이전트가 기업 개발 환경에 급속히 확산되었다. 그러나 이들을 **실제 프로덕션 코드베이스**에 투입하면 다음과 같은 구조적 문제가 반복적으로 발생한다:

| # | 문제 유형 | 구체적 증상 | 원인 |
|---|----------|-----------|------|
| 1 | **환각(Hallucination)** | 존재하지 않는 DB 테이블·컬럼을 포함한 SQL 생성 | LLM이 내부 스키마를 학습하지 못함 |
| 2 | **컨텍스트 유실** | 긴 대화에서 이전 작업 맥락 망각, 동일 실수 반복 | 단일 세션의 토큰 제한 |
| 3 | **규칙 비일관성** | 프로젝트 A에서는 PDO 필수인데 프로젝트 B에서는 무시 | 공통 규칙의 중앙 관리 부재 |
| 4 | **과잉 행동** | 요청하지 않은 대규모 리팩토링, 프레임워크 교체 시도 | 에이전트의 행동 범위 미정의 |
| 5 | **무한 루프** | 실패한 접근을 반복 시도하며 토큰 소진 | Fail-Fast 메커니즘 부재 |

### 1.2 기존 접근법의 한계

- **단일 `copilot-instructions.md`**: 하나의 파일에 모든 규칙을 넣으면 토큰 낭비 + 규칙 충돌
- **프로젝트별 독립 규칙**: 전사 보안 정책이 프로젝트마다 다르게 적용되는 사일로(Silo) 문제
- **규칙 없음(자유 방임)**: AI의 자율성에 의존 → 가장 위험한 접근

### 1.3 해결 과제

> "전사 공통 규칙을 강제하면서도, 프로젝트 고유의 도메인 지식을 보존하고, 태스크 단위로 상태를 추적할 수 있는 **계층적 거버넌스 체계**를 구축한다."

---

## 2. Design Principles

아키텍처 전체를 관통하는 6가지 설계 원칙:

### 2.1 Separation of Concerns (관심사 분리)

```
L1 (전사 공통) ≠ L2 (프로젝트 도메인) ≠ L3 (태스크 상태)
```

각 레이어는 독립적으로 버전 관리되며, 상위 레이어가 하위를 오버라이드하지 않도록 **보호 구역(`<!-- BEGIN L2 -->`)** 패턴으로 경계를 물리적으로 구분한다.

### 2.2 Fail-Fast (즉시 실패)

```
재시도 최대 2~3회 → 초과 시 즉시 셧다운 → 사용자 개입 요청
```

AI 에이전트가 잘못된 경로에서 무한 루프에 빠지는 것을 원천 차단한다. 실패를 빨리 인정하고 사람에게 넘기는 것이 토큰을 절약하고 코드 품질을 지키는 최선이다.

### 2.3 State Compression (상태 압축)

세션 상태(`state.json`)는 과거 이력을 누적하지 않고, **"현재 에러"**와 **"다음 액션"** 딱 2문장으로 압축한다. 이는 다음 세션이 최소한의 토큰으로 맥락을 복원할 수 있게 한다.

### 2.4 Token Budget (토큰 예산)

단일 턴에서 로드하는 컨텍스트를 **~8,000 토큰** 이하로 관리한다. Tiered Loading(T0→T3)으로 진짜 필요한 규칙만 지연 로딩(Lazy Load)하여, 무관한 규칙이 컨텍스트를 오염시키는 것을 방지한다.

### 2.5 SSOT (Single Source of Truth)

모든 규칙의 원본은 `company-ai-setup` 허브 하나에만 존재한다. 각 프로젝트의 `.ai/` 디렉토리는 이 원본의 **동기화된 사본 + 프로젝트 고유 확장**이다. 수동 편집 금지, 반드시 동기화 스크립트를 통해 전파한다.

### 2.6 Search Before Guess (추측 전 검색)

AI 에이전트가 DB 스키마, 레거시 API, 비즈니스 로직에 대해 "아마 이럴 것이다"라고 추측하는 것을 **금지**한다. 반드시 MCP 도구로 검색하고, 검색 결과를 기반으로 코드를 작성한다.

---

## 3. 3-Layer Architecture

### 3.1 전체 구조도

```
┌──────────────────────────────────────────────────────────────┐
│                    L1: Global Hub (SSOT)                      │
│              company-ai-setup repository                      │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ 9 Rules │ │ 4 Roles  │ │9 Workflows│ │ manifest.json    │ │
│  └────┬────┘ └────┬─────┘ └─────┬────┘ └────────┬─────────┘ │
└───────┼───────────┼─────────────┼────────────────┼───────────┘
        │    ai-sync.sh (hash-verified top-down sync)
        ▼           ▼             ▼                ▼
┌──────────────────────────────────────────────────────────────┐
│              L2: Project Layer (.ai/)                    │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐ │
│  │ L1 Copy     │  │ Domain Rules │  │ Context Docs        │ │
│  │ (read-only) │  │ local-*.md   │  │ project.md          │ │
│  │             │  │ mcp-usage.md │  │ architecture.md     │ │
│  │ <!-- L2 --> │  │ cto-dispatch │  │ database-Table1.md  │ │
│  │ (preserved) │  │ skills-gov.  │  │ api-endpoints.md    │ │
│  └──────┬──────┘  └──────────────┘  └─────────────────────┘ │
│         │                                                     │
│  ┌──────┴─────────────────────────────────────────────┐      │
│  │ state/ — Token Budget Registry, Global Task Board  │      │
│  └──────┬─────────────────────────────────────────────┘      │
└─────────┼────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────────┐
│              L3: Task/Session Layer                            │
│  ┌────────────────────────────────────────────────┐          │
│  │ state/sessions/{ISSUE-ID}/                     │          │
│  │   ├── state.json    (2-sentence summary)       │          │
│  │   ├── context.md    (plan phase output)        │          │
│  │   └── research_report.md                       │          │
│  └────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 L1 — Global Hub (전사 공통)

**위치**: `company-ai-setup` 레포지토리  
**배포 대상**: `.github/copilot-instructions.md` (GitHub Copilot), `CLAUDE.md` (Claude)

L1은 조직의 **모든 프로젝트에 동일하게 적용**되는 AI 에이전트 행동 규범이다:

| 구성요소 | 파일 수 | 역할 |
|---------|--------|------|
| Core Rules | 9개 | 가드레일, 코딩, 보안, 워크플로우, 성능, 프롬프팅, Git, 언어, 커스텀 레지스트리 |
| Agent Roles | 4개 | Planner, Worker, Reviewer, Infra 에이전트 페르소나 |
| Workflows | 9개 | plan, run, hotfix, review, PR, handoff, FM, help, bump-version |
| Manifest | 1개 | 버전, 해시, 변경 이력 추적 |

**핵심 설계**: 각 규칙 파일의 YAML 프론트매터에 `trigger` 필드를 두어, 매 턴마다 전체를 로드하지 않고 필요한 것만 로드:

```yaml
---
trigger: always        # 매 턴 필수 로드 (guardrails, language)
trigger: model_decision  # AI가 판단하여 필요 시 로드
---
```

### 3.3 L2 — Project Layer (프로젝트 도메인)

**위치**: 각 프로젝트의 `.ai/` 디렉토리

L2는 L1을 **상속받으면서** 프로젝트 고유의 도메인 지식을 추가한다:

```
.ai/
├── rules.md              ← L1 허브 사본 + L2 도메인 섹션
├── rules/
│   ├── guardrails.md     ← L1 사본 (상단) + L2 확장 (하단 보호구역)
│   ├── local-coding.md   ← L2 전용: 프로젝트 코딩 규칙
│   ├── local-db.md       ← L2 전용: DB 접근 정책
│   ├── mcp-usage.md      ← L2 전용: MCP 검색 강제 정책
│   └── ...
├── context/              ← 프로젝트 참조 문서
├── roles/                ← 로컬 에이전트 오버라이드
└── state/                ← 토큰 예산, 세션 디렉터리
```

**L2 보호 구역 패턴**: L1 규칙 파일 안에서 프로젝트 고유 내용은 HTML 주석으로 경계를 표시하여, 동기화 스크립트가 이 영역을 덮어쓰지 않는다:

```markdown
<!-- 🚨 SYNCED from company-ai-setup (L1: DO NOT EDIT) 🚨 -->
[... L1 공통 규칙 ...]

<!-- BEGIN L2 (DOMAIN RULES) -->
[... 프로젝트 고유 규칙 — 동기화 시 보존 ...]
<!-- END L2 -->
```

### 3.4 L3 — Task/Session Layer (태스크 실행)

**위치**: `.ai/state/sessions/{ISSUE-ID}/`

L3는 개별 작업의 **실행 상태를 추적**한다. 각 세션은 Jira 이슈 ID를 디렉토리명으로 사용하며, 작업 재개 시 `state.json`을 통해 이전 맥락을 즉시 복원한다:

```json
{
  "session_id": "ISSUE-0001",
  "status": "active",
  "active_agent": { "id": "worker_agent", "phase": "execute" },
  "context_summary": "PDO 연결 풀 에러 발생. config.php의 DSN 파싱 로직 수정 시도 예정.",
  "retry_count": 1,
  "last_updated": "2026-03-27T14:30:00+09:00"
}
```

---

## 4. Rule Taxonomy & Trigger System

### 4.1 9개 코어 규칙 분류

| # | 규칙 파일 | Trigger | 토큰 비용 | 핵심 내용 |
|---|----------|---------|----------|----------|
| 1 | `guardrails.md` | `always` | ~920 | 절대 금지 행동 목록, DB 환각 차단, 레거시 파괴 방지 |
| 2 | `language.md` | `always` | ~213 | 한국어 우선, 기술 용어 원문 병기 |
| 3 | `agent-workflow.md` | `model_decision` | ~1,453 | 3단계 워크플로우(Plan→Execute→Verify), Fail-Fast, Tiered Loading |
| 4 | `coding.md` | `model_decision` | ~769 | 코드 품질, JS/TS 스타일, 패키지 관리 |
| 5 | `security.md` | `model_decision` | ~536 | Zero-Trust, SQL Injection 방지, XSS 보호 |
| 6 | `performance-optimization.md` | `model_decision` | ~502 | 3-Tier 검색 파이프라인, 캐시 전략, Dynamic Routing |
| 7 | `prompting.md` | `model_decision` | ~221 | XML 태그 구조화, CoT 강제, Few-shot 포맷 |
| 8 | `git.md` | `model_decision` | ~283 | Conventional Commits, 브랜치 전략, PR 필수화 |
| 9 | `custom-registry.md` | `model_decision` | ~577 | 도메인별 커스텀 규칙 디스패치 |

### 4.2 Tiered Loading 모델

```
Tier  │ 로딩 시점     │ 대상                         │ 예산
──────┼───────────────┼──────────────────────────────┼──────
T0    │ 항상          │ rules.md + guardrails.md      │ ~2,955
T1    │ 기본          │ 역할 파일 + state.json         │ ~350-400
T1.5  │ 조건부        │ context.md (세션 이어받기 시)   │ ~333
T2    │ 필요 시       │ rules/ 서브파일 (최대 3개)     │ ~500-1,500
T3    │ 금지          │ 타 에이전트 역할, 무관 가이드   │ ——
```

**Lite 모드**: T0 + T1 ≈ **~3,400 토큰** (일반 작업)  
**Full 모드**: T0 + T1 + T2×3 ≈ **~5,400 토큰** (복잡한 계획)

### 4.3 Context Pruning (가지치기) 전략

대용량 파일은 전체를 로드하지 않고 요약본으로 대체:

| 조건 | Pruning 방식 |
|------|-------------|
| T0~T1 | 전문 로드 (pruning 없음) |
| T2 | **Stubbing** — 구현부 제외, 인터페이스·시그니처만 |
| 300줄 초과 | **Summary 전용** — `context_registry`의 `summary` 필드만 |

이 전략은 Kubernetes의 **리소스 쿼타(Resource Quota)** 개념을 차용한 것으로, 각 레이어가 사용할 수 있는 토큰 예산을 사전에 할당하여 전체 시스템의 안정성을 보장한다.

---

## 5. Agent Persona & Workflow Choreography

### 5.1 4개 에이전트 페르소나

```
┌──────────────────────────────────────────────────┐
│           AI Agent (Single Thread)                │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Planner  │→ │ Worker   │→ │ Reviewer │       │
│  │ /plan    │  │ /run     │  │ /review  │       │
│  │          │  │ /hotfix  │  │          │       │
│  └──────────┘  └──────────┘  └──────────┘       │
│                                    ↓              │
│                              ┌──────────┐        │
│                              │  Infra   │        │
│                              │ /PR      │        │
│                              └──────────┘        │
└──────────────────────────────────────────────────┘
```

| 페르소나 | 진입 명령어 | 핵심 책임 | 금지 사항 |
|---------|-----------|----------|----------|
| **Planner** | `/plan` | 요구사항 분석, 영향 범위 파악, 계획서 작성 | 코드 작성 금지, 승인 전 실행 금지 |
| **Worker** | `/run`, `/hotfix` | 승인된 계획 기반 코드 구현 | 계획 외 scope 변경 금지 |
| **Reviewer** | `/review` | 코드 품질·보안 리뷰, 체크리스트 검증 | 코드 직접 수정 금지 |
| **Infra** | `/PR` | Git 커밋, PR 생성, Docker/배포 | 비즈니스 로직 변경 금지 |

### 5.2 페르소나 전환의 원리

현재 AI 도구들은 **싱글 스레드, 1인 다역**으로 동작한다. "나는 이제 Worker Agent다"라고 전환하면:

1. 해당 `roles/*.md` 파일의 **도메인 지식**과 **코딩 규칙**을 로드
2. 작업 범위가 **역할 파일에 정의된 scope**로 자연스럽게 제한
3. `state.json`에 `active_agent`를 기록하여 다음 세션에서 맥락 복원

이것은 마이크로서비스의 **Sidecar Pattern**과 유사하다 — 각 역할이 독립된 설정과 책임을 가지면서, 하나의 프로세스 안에서 순차적으로 실행된다.

### 5.3 Dynamic Routing (난이도별 실행 단계)

모든 작업에 3단계를 거칠 필요는 없다. 난이도에 따라 실행 경로를 최적화:

| Level | 조건 | 실행 경로 | 사례 |
|-------|------|----------|------|
| **1: 단순** | 파일 ≤ 2개, 명확한 요구사항 | Worker only | 문구 수정, 설정값 변경 |
| **2: 중간** | 비즈니스 로직 포함 | Planner → Worker | 신규 API 엔드포인트 |
| **3: 중요** | 보안/인증/결제, 대규모 변경 | Planner → Worker → Reviewer | 인증 로직 변경, DB 마이그레이션 |

---

## 6. Hub-and-Spoke Sync

### 6.1 동기화 아키텍처

```
company-ai-setup (Hub)
        │
        ├── ai-sync.sh ──→ DOMAIN1/.ai/      (Spoke 1)
        ├── ai-sync.sh ──→ DOMAIN2/.ai/   (Spoke 2)
        └── ai-sync.sh ──→ ai-mcp-server/.ai/   (Spoke 3)
```

**단방향 Top-Down**: 허브 → 프로젝트. 프로젝트가 허브 규칙을 역으로 수정하는 것은 불가.

### 6.2 동기화 메커니즘

```bash
# 동기화 실행
./bin/ai-sync.sh --l2-hub=/path/to/hub/.ai --target-dir=.ai
```

동기화 프로세스:

1. **Hash 비교**: `manifest.json`의 파일별 해시와 로컬 사본을 비교
2. **L1 영역 덮어쓰기**: `<!-- 🚨 SYNCED -->` ~ `<!-- BEGIN L2 -->` 구간만 교체
3. **L2 보호 구역 보존**: `<!-- BEGIN L2 -->` ~ `<!-- END L2 -->` 구간은 절대 건드리지 않음
4. **Sync Report 생성**: `sync-report.json`에 동기화 이력·결과 기록

### 6.3 버전 관리 (Dual-Source SSOT)

```
manifest.json  ←─ 규칙 파일별 해시 + 전체 버전
      ↕ (자동 동기화)
package.json   ←─ npm 생태계 호환 버전
```

버전 업데이트는 반드시 자동화 스크립트를 통해:

```bash
./bin/bump-version.sh patch "guardrails: 레거시 파괴 방지 조항 추가"
# → manifest.json v6.4.x → v6.4.y
# → package.json  동시 업데이트
# → CHANGELOG 자동 기록
```

수동으로 `version` 필드를 직접 편집하는 것은 **절대 금지**.

### 6.4 변경 이력 추적

`manifest.json`의 `changelog` 필드에 세맨틱 버저닝 기반 이력이 누적:

```
v6.5.0:  [Minor] 세부지침 업데이트
v6.4.17: [Patch] trigger 체계 반영 및 bump-version 워크플로우 등록
v6.4.15: [Patch] L1 rules.md 다이어트: 8.9KB→2.7KB 초경량 라우팅 허브 전환
v6.4.14: [Patch] MCP 도구 과부하 통제, RAG CoT 강제, State Pruning 추가
```

---

## 7. MCP Integration & RAG Pipeline

### 7.1 MCP 서버의 역할

MCP(Model Context Protocol) 서버는 이 아키텍처의 **지식 백본(Knowledge Backbone)**이다. AI 에이전트가 DB 스키마, 레거시 API, 비즈니스 용어를 추측하지 않고 **검색**할 수 있게 한다.

```
IDE AI Agent (Copilot/Claude/Cursor)
        │ (stdio — MCP Protocol)
        ▼
   MCP Server (TypeScript/Node.js)
   ├── search_memory()     ← 벡터 유사도 검색
   ├── query_context()     ← 정적 컨텍스트 문서 조회
   ├── upsert_memory()     ← 새 지식 저장 (Write-back)
   ├── manage_collection() ← 컬렉션 관리
   ├── get_file_signature() ← 파일 해시·메타데이터
   ├── get_domain_status()  ← 도메인 세션 상태
   ├── save_directive()     ← 세션 지시사항 저장
   └── update_session_state() ← 세션 상태 갱신
        │
        ▼ (HTTP — Port 8100)
   Sidecar Embedding Server (Python)
   ├── /embed   ← 단일 텍스트 → 벡터
   ├── /upsert  ← 문서 임베딩 + 컬렉션 저장
   ├── /query   ← 시맨틱 유사도 검색
   └── /reindex ← 컬렉션 재인덱싱
        │
        ▼
   ChromaDB (Vector Store) + TF-IDF Fallback
```

### 7.2 3-Tier 검색 파이프라인

모든 검색 요청은 **비용이 낮은 것부터** 순서대로:

| Tier | 방식 | 조건 | 응답 시간 | 비용 |
|------|------|------|----------|------|
| **1** | LRU Cache | 동일 query+filter → 즉시 반환 | ~0ms | 무료 |
| **2** | 키워드 검색 | grep_search, 파일 읽기 | ~100ms | 낮음 |
| **3** | Vector Search (MCP) | 의미 기반 검색, Tier 2 실패 시 | ~1,000ms | 높음 |

**핵심 가드레일**: Vector Search 사용률 **30% 이하** 유지. 키워드로 충분하면 벡터 검색 금지.

### 7.3 MCP 강제 검색 정책 (Mandatory Search)

특정 키워드가 프롬프트에 포함될 때, AI는 **코드 작성 전에 반드시** MCP 검색을 실행해야 한다:

| 키워드 유형 | 예시 | 트리거 |
|-----------|------|--------|
| DB/테이블 | Table2, Table3, Table4, Table1 | `search_memory(domain="DOMAIN1")` |
| 코드/프레임워크 | L_DB, PDO, ModenModel, ModenApp | `search_memory(collection="company_knowledge")` |
| 비즈니스 도메인 | 매출, 수금, 비용, 라이선스, 파이프라인 | `search_memory(query="...")` |

**실행 순서 (STOP → SEARCH → VERIFY → CITE)**:

1. **STOP**: 즉각적인 답변 생성 중단
2. **SEARCH**: MCP 도구로 관련 지식 조회
3. **VERIFY**: 검색된 컨텍스트(DDL, View Chain, 패턴)로 답변 구성
4. **CITE**: 사용한 테이블/컬럼의 출처 명시

### 7.4 MCP 미가동 시 대체 경로

```
MCP 서버 timeout/connection refused
  → 1회 재시도
  → 실패 시 정적 컨텍스트 모드 전환:
      1. .ai/context/database-Table1.md 인덱스 참조
      2. 키워드 기반 grep_search
      3. 사용자에게 "MCP 미응답" 1회 알림
```

---

## 8. State Management & Session Handoff

### 8.1 세션 라이프사이클

```
                 ┌──────────┐
     /plan ──→   │ created  │
                 └────┬─────┘
                      ▼
                 ┌──────────┐
     /run ───→   │  active  │ ←── state.json 갱신
                 └────┬─────┘
                      │
              ┌───────┼────────┐
              ▼       ▼        ▼
         ┌────────┐ ┌───────┐ ┌────────┐
         │blocked │ │failed │ │completed│
         └───┬────┘ └───────┘ └────┬───┘
             │                     ▼
             │              ┌──────────┐
             └──────────→   │ archived │
                            └──────────┘
```

### 8.2 state.json 2문장 압축 원칙

일반적인 장기 실행 시스템에서는 로그/이력이 누적된다. 하지만 AI 에이전트의 컨텍스트 윈도우는 유한하다. 따라서:

❌ **나쁜 예**: "3/25에 A 시도 → 실패 → 3/26에 B 시도 → 부분 성공 → 3/27에 C 수정 → ..."

✅ **좋은 예**: "PDO 연결 풀 에러 발생. config.php의 DSN 파싱 로직 수정 시도 예정."

이것은 Git의 **squash merge**와 유사한 철학이다 — 개별 이력보다 최종 상태가 더 중요하다.

### 8.3 세션 인수인계 (Handoff)

복잡한 작업이 한 세션 안에서 완료되지 못할 때, 다음 세션(또는 다른 AI 에이전트)이 이어받을 수 있도록 인수인계서를 작성한다:

1. 새 세션 시작 시: `state.json` → `context.md` → `global-task-board.md` 순으로 로드
2. 사용자에게 브리핑: "이전 세션 {ID}에서 이어받았습니다. 다음 작업: {plan}"
3. 이전 세션의 `retry_count`와 `context_summary`를 기반으로 작업 재개

---

## 9. Security & Guardrails

### 9.1 보안 계층 모델

```
┌─────────────────────────────────────────────┐
│ Layer 4: Network (Reverse Proxy, WAF)        │
├─────────────────────────────────────────────┤
│ Layer 3: Authentication (Keycloak SSO, JWT)  │
├─────────────────────────────────────────────┤
│ Layer 2: Code (PDO, XSS escape, Zod)        │
├─────────────────────────────────────────────┤
│ Layer 1: AI Guardrails (prompt-based)        │
└─────────────────────────────────────────────┘
```

### 9.2 핵심 가드레일 목록

| # | 가드레일 | 심각도 | 구현 방식 |
|---|---------|--------|----------|
| 1 | 시크릿(DB 비밀번호, API Key) 노출 금지 | 🔴 Critical | `.env` 접근 차단, 로그 출력 금지 |
| 2 | DB 변경(INSERT/UPDATE/DELETE) 직접 실행 금지 | 🔴 Critical | 사용자 승인 필수 (SELECT만 허용) |
| 3 | DB 스키마 추측 금지 | 🔴 Critical | MCP 검색 강제 |
| 4 | 시스템 명령어 동적 생성 금지 | 🔴 Critical | `child_process.exec`에 사용자 입력 주입 차단 |
| 5 | 프레임워크·라이브러리 무단 교체 금지 | 🟡 High | 사용자 승인 필수 |
| 6 | 타 세션 디렉토리 무단 수정 금지 | 🟡 High | 세션 격리 |
| 7 | 불필요한 전체 리팩토링 금지 | 🟠 Medium | 오버엔지니어링 방지 |
| 8 | L1 규칙 수동 수정 금지 | 🟠 Medium | bump-version 스크립트 필수 |

### 9.3 Zero-Trust 프롬프팅의 현실

마크다운 문서만으로 AI의 파일 시스템 접근을 **물리적으로 차단**하는 것은 현재 기술로 불가능하다. 따라서 이 아키텍처는 **프롬프트 기반 통제** — 즉, 규칙을 명확하고 반복적으로 명시하여 LLM의 행동을 유도하는 방식을 택한다.

이때 **Recency Bias(최신성 편향)**를 역이용한다: LLM은 프롬프트의 **가장 마지막**에 배치된 지시사항을 가장 강하게 인지하므로, "절대 하지 말 것(DO NOT)" 류의 가드레일은 시스템 프롬프트의 **최하단**에 배치한다:

```markdown
# 규칙 문서 구조
## 1. 배경 및 목적 (상단)
## 2. 허용되는 행동 (중단)
## 3. 🚫 절대 금지 사항 (최하단) ← Recency Bias 역이용
```

### 9.4 코드 레벨 보안 규칙

| 카테고리 | 필수 규칙 |
|---------|----------|
| SQL Injection | **Prepared Statement 필수** (문자열 결합 쿼리 금지) |
| XSS | 사용자 입력 HTML 이스케이프 필수 |
| 입력 검증 | Whitelist 기반 검증 우선 |
| 파일 업로드 | 확장자·MIME 검증, 실행 파일 차단 |
| 금지 함수 | `eval()`, 외부 입력 역직렬화, 오류 억제 연산자 |

---

## 10. Results & Lessons Learned

### 10.1 정량적 성과

| 지표 | Before (규칙 없음) | After (3-Layer) | 개선 |
|------|-------------------|-----------------|------|
| DB 스키마 환각 발생 빈도 | 10회 중 3~4회 | 10회 중 0~1회 | **~85% 감소** |
| 세션 재개 성공률 | ~30% (맥락 유실) | ~85% (state.json 복원) | **+55%p** |
| 토큰 소모량 (평균 턴) | ~15,000 (전체 로드) | ~5,000 (Tiered Loading) | **~67% 절감** |
| 프로젝트 간 규칙 불일치 | 빈번 | 0건 (SSOT 동기화) | **100% 해소** |

### 10.2 핵심 교훈

#### Lesson 1: "규칙은 코드다"
AI 에이전트의 행동 규칙도 소프트웨어와 동일하게 **버전 관리, 테스트, 배포 파이프라인**이 필요하다. `manifest.json` + `ai-sync.sh`를 통한 자동 배포는 이 원칙의 실현이다.

#### Lesson 2: "적을수록 좋다"
초기에는 규칙을 최대한 많이 작성하려는 유혹이 있었다. 그러나 **토큰 예산 제약** 앞에서, 규칙의 양보다 **규칙의 구조화**(Tiered Loading, Pruning)가 훨씬 중요했다. `rules.md`를 8.9KB에서 2.7KB로 다이어트한 것이 전환점이었다.

#### Lesson 3: "추측보다 검색"
DB 스키마 환각 문제의 근본 해결은 "더 똑똑한 AI"가 아니라 "검색을 강제하는 정책"이었다. MCP 강제 검색 정책(`mcp-usage.md`)은 가장 ROI가 높은 규칙이었다.

#### Lesson 4: "실패를 빨리 인정하라"
Fail-Fast 원칙은 이론적으로는 당연하지만, AI 에이전트에게 "모르겠으면 멈춰라"를 가르치는 것은 예상보다 어려웠다. `retry_count`를 `state.json`에 명시적으로 기록하고, 초과 시 물리적으로 실패 상태를 전환하는 방식이 효과적이었다.

#### Lesson 5: "보호 구역 패턴은 필수"
L1 동기화 시 L2 고유 규칙이 덮어씌워지는 사고를 겪은 후, `<!-- BEGIN L2 -->` 보호 구역 패턴을 도입했다. 이것은 **Kubernetes의 Custom Resource Definition(CRD)**과 유사한 발상 — 플랫폼이 관리하는 영역과 사용자가 관리하는 영역을 물리적으로 분리한다.

### 10.3 향후 개선 방향

| # | 개선 과제 | 기대 효과 |
|---|---------|----------|
| 1 | 규칙 준수율 자동 측정 파이프라인 | 가드레일 위반 사례의 정량적 추적 |
| 2 | Multi-Agent 병렬 실행 | 현재 싱글 스레드 → Planner와 Researcher 병렬화 |
| 3 | 규칙 A/B 테스트 프레임워크 | 규칙 변경의 효과를 실험적으로 검증 |
| 4 | 자동 L2 생성 CLI | 새 프로젝트 온보딩 시 도메인 규칙 자동 스캐폴딩 |

---

## Appendix

기술 부록(규칙 파일 명세, MCP 도구 카탈로그, State Schema 등)은 별도 문서 참조:

→ **[02-technical-appendix.md](./02-technical-appendix.md)**

---

*© 2026 ycy0922. 본 문서는 개인 포트폴리오 목적으로 작성되었으며, 사내 기밀 정보는 포함되어 있지 않습니다.*
