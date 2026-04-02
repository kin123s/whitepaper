# MCP RAG Server: GraphRAG와 Triple-Hybrid Search로 진화한 지식 엔진 (v11.0)

> **Author:** ycy0922  
> **Version:** 11.0  
> **Date:** 2026-04-03  
> **Status:** Portfolio Document  
> **Companion to:** [Chapter 1 — 3-Layer Rule System](./01-ai-agent-governance-architecture.md)

---

## Abstract

Chapter 1에서 "추측하지 말고 검색하라(Search Before Guess)"는 원칙을 제시했다. 본 문서는 그 원칙을 기술적으로 실현한 **MCP RAG Server의 진화 과정(v6.0 → v11.0)**을 다룬다.

초기 RAG 아키텍처는 ChromaDB 기반의 단순 벡터 검색을 사용했으나, "Multi-hop(다단계) 추론 불가" 및 "고유명사 매칭 실패"라는 명확한 한계에 부딪혔다. 이를 극복하기 위해 **Neo4j Knowledge Graph**를 결합하여 코드/이슈 간의 명시적 관계를 추적하고, **BM25 + Embedding + Graph Traversal**을 결합한 **Triple-Hybrid Search** 파이프라인을 구축했다. 

또한 검색 결과를 LLM이 스스로 평가하고 재시도하는 **Self-RAG 루프**를 도입하여, "쓰레기가 들어가면 쓰레기가 나오는(GIGO)" RAG의 근본적 취약점을 시스템 계층에서 방어한 아키텍처를 소개한다.

---

## Table of Contents

1. [벡터 검색의 한계와 GraphRAG 도입 배경](#1-벡터-검색의-한계와-graphrag-도입-배경)
2. [전체 아키텍처 (v11.0) — Node.js + Neo4j + Ollama](#2-전체-아키텍처-v110--nodejs--neo4j--ollama)
3. [Triple-Hybrid 파이프라인: 비용 효율적인 계층 검색](#3-triple-hybrid-파이프라인-비용-효율적인-계층-검색)
4. [Self-RAG: 자가 평가와 쿼리 재작성 루프](#4-self-rag-자가-평가와-쿼리-재작성-루프)
5. [Graph Indexing — 노드와 엣지의 스마트 추출](#5-graph-indexing--노드와-엣지의-스마트-추출)
6. [Resource-Aware 큐잉 메커니즘 (성능 통제)](#6-resource-aware-큐잉-메커니즘-성능-통제)
7. [결과 및 KPI 지표 향상](#7-결과-및-kpi-지표-향상)

---

## 1. 벡터 검색의 한계와 GraphRAG 도입 배경

### 1.1 ChromaDB(v6~v9) 시절의 실패 사례

초기에 운영했던 단순 벡터 기반 RAG 시스템은 일반적인 질문("결제 로직 에러 어떻게 고쳐?")에는 잘 답했지만, 사내 레거시 코드베이스의 복잡한 질문 앞에서는 처참히 실패했다:

- **문제 1: 고유명사와 도메인 용어 소실**  
  `EX_TABLE` 같은 오래된 사내 DB 테이블 명칭이나 약어들은 다국어 임베딩 모델(MiniLM) 공간에서 의미가 희석되어 엉뚱한 테이블을 검색 결과로 내놓았다.
- **문제 2: Multi-hop(다단계) 추론 불가**  
  *"A 함수를 수정한 최근 커밋과 관련된 Jira 이슈 번호는?"* 와 같은 질문은 단일 벡터 쿼리로 해결할 수 없었다. (A 함수 검색 ➔ 커밋 반환 ➔ 커밋 번호로 다시 검색 ➔ 이슈 반환)

### 1.2 지식 그래프(Knowledge Graph)로의 전환

단어의 의미적 거리가 아니라 **엔티티 간의 명시적 관계(Relationship)**로 지식을 추적해야 했다. 이를 위해 v11.0에서는 ChromaDB를 полностью 제거하고, **Neo4j 기반의 GraphRAG**를 도입했다. 파일, 커밋, 이슈, 함수 등을 Node로, 이들 간의 의존성 및 수정 이력을 Edge로 매핑하여 관계 단절 문제를 극복했다.

---

## 2. 전체 아키텍처 (v11.0) — Node.js + Neo4j + Ollama

### 2.1 분산 아키텍처 통합

v11.0의 가장 큰 구조적 변화는 기존의 무거운 Python Sidecar 프로세스를 제거하고, TypeScript 중심의 단일 오케스트레이터로 파이프라인을 경량화한 것이다.

```text
┌─────────────────────────────────────────────────────────┐
│  AI IDE (Cursor / Gemini CLI / Claude)                  │
│    ↕ stdio (Model Context Protocol)                     │
├─────────────────────────────────────────────────────────┤
│  mcp-rag-server v11.0 (TypeScript, Node.js 20+)        │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ 14 MCP Tools │  │ Triple-Hybrid│  │ LRU Cache    │  │
│  │ Router       │  │ Pipeline     │  │ (200 items)  │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┘  │
│         │                 │                             │
│  ┌──────▼──────────────────▼──────────────────────────┐ │
│  │         Triple-Hybrid Search Engine                │ │
│  │  ┌──────────┐    ┌────────────────┐                │ │
│  │  │ BM25     │    │ Embedding      │                │ │
│  │  │ (FTS5)   │───▶│ Rerank (Top5)  │                │ │
│  │  │ Top 50   │    └───────┬────────┘                │ │
│  │  └──────────┘            │                         │ │
│  │  ┌──────────────────────────────────┐              │ │
│  │  │ ★ Neo4j Graph Traversal         │              │ │
│  │  │   Entity Linking → N-hop Search  │              │ │
│  │  └──────────┬───────────────────────┘              │ │
│  │             ▼                                      │ │
│  │  ┌──────────────────────────────────┐              │ │
│  │  │ Context Merge (Vector + Graph)   │              │ │
│  │  └──────────┬───────────────────────┘              │ │
│  │             ▼                                      │ │
│  │  ┌──────────────────────────────────┐              │ │
│  │  │ Self-RAG: Relevance Judge        │              │ │
│  │  │   ≥40% → 통과    <40% → Rewrite │              │ │
│  │  └──────────┬───────────┬───────────┘              │ │
│  │             │           ▼                          │ │
│  │             │  ┌────────────────┐                  │ │
│  │             │  │ Query Rewriter │→ 재검색 (≤2회)   │ │
│  │             │  └────────────────┘                  │ │
│  └─────────────┼──────────────────────────────────────┘ │
│                ▼                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Context      │  │ KPI Metrics  │  │ Token &      │  │
│  │ Compression  │  │ Collector    │  │ Cost Tracker │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
         ↕ HTTP                    ↕ Bolt
    ┌──────────────────┐    ┌──────────────────┐
    │ Ollama           │    │ Neo4j 5          │
    │ • nomic-embed    │    │ (Community)       │
    │ • llama3:8b      │    │ Knowledge Graph   │
    └──────────────────┘    └──────────────────┘
```

**[컴포넌트 역할]**
- **Node.js (TypeScript):** MCP 프로토콜 엔드포인트 및 Self-RAG 비즈니스 로직 제어.
- **Neo4j 5:** Multi-hop 추론이 가능한 메인 GraphDB 보관소.
- **SQLite (FTS5):** 하드 키워드 매칭(고유명사)을 담당하는 초고속 BM25 엔진.
- **Ollama (로컬 API):** 데이터 유출 없이 임베딩 변환 및 Self-RAG 판독(llama3) 수행.

---

## 3. Triple-Hybrid 파이프라인: 비용 효율적인 계층 검색

"한 번의 거대한 벡터 서칭" 대신, 빠르고 저렴한 검색에서 시작해 정밀한 그래프를 탐색하는 **다단계 계층 검색**으로 진화했다.

### 3.1 파이프라인 흐름

1. **Step 1: BM25 (SQLite FTS5)**  
   - 역할: 대량의 문서에서 고유명사와 일치하는 상위 50개 문서 1차 필터링 (속도: < 10ms)
2. **Step 2: Embedding Reranking (Ollama)**  
   - 역할: 추려진 문서들을 `nomic-embed-text` 모델을 통해 시맨틱 유사도로 재배열 및 Top 5 추출.
3. **Step 3: Graph Traversal (Neo4j)**  
   - 역할: 추출된 Top 5 문서(Entity)를 기점으로 Neo4j 내에서 관련된 모듈, 변경한 개발자, 연관 JIRA 이슈 등을 N-hop(그래프 거리)으로 순회하여 맥락 보존 보충.

이 하이브리드 패턴은 고유명사의 소실을 막는 동시에(BM25 강점), 파편화된 파일 간의 인과관계를 찾아내는(GraphRAG 강점) 가장 완벽한 조합이 되었다.

---

## 4. Self-RAG: 자가 평가와 쿼리 재작성 루프

사용자(AI 에이전트)의 질문이 모호하면 응답 컨텍스트도 모호해진다. 이를 방지하기 위해 로컬 LLM(Ollama)을 활용한 **피드백 루프(Self-RAG)**를 도입했다.

```text
검색 결과 집합 (Context Merge 완료)
  ↓
[Relevance Judge] (로컬 LLM 판단기)
  "이 문서들이 원래 질문에 대답하기에 충분합니까?"
  "점수 평가: 0.0 ~ 1.0 (Threshold: 0.40)"
  ↓
  ├─ (Score ≥ 40%) ──▶ Context 압축 후 즉시 반환 (성공)
  │
  └─ (Score < 40%) ──▶ [Query Rewriter] 호출
                       동의어 확장, 질문 분해. (예: "결제 에러" ➔ "PAYMENT_API Exception PDO")
                       ↓
                       (최대 2회까지 하이브리드 검색 재시작)
```

이 루프는 부실한 컨텍스트로 LLM이 소설(환각)을 쓰는 현상을 90% 이상 차단했다. "검색 실패"를 조기에 인지하고 스스로 회복탄력성(Resilience)을 가지게 되었다.

---

## 5. Graph Indexing — 노드와 엣지의 스마트 추출

지식 그래프의 핵심은 텍스트를 노드(Node)와 엣지(Edge) 구조로 자동 파생시키는 파이프라인이다.

- **Entity Extractor (AST + LLM 기반):**  
  PHP, JS, TS 등의 코드가 들어오면 AST 파서를 통해 `Class`, `Function`을 도출하고, 로컬 LLM을 통해 비즈니스 의존도(`CALLS`, `IMPLEMENTS`)를 엣지 관계로 추출한다.
- **증분 인덱싱(Incremental Indexing):**  
  파일 전체를 지우고 다시 생성하는 것은 파멸적 리소스 낭비다. 파일의 Hash를 보관하고, 변경된 파일만 식별해 해당 파일에 연결된 Edge 정보만 끊었다 다시 잇는 방식으로 배치 처리 시간을 극적으로 줄였다.

---

## 6. Resource-Aware 큐잉 메커니즘 (성능 통제)

엔터프라이즈 환경에서 백그라운드 AI 서버가 CPU나 I/O를 과도하게 점유하면 실제 서비스 워크로드에 치명상을 줄 수 있다. 

```typescript
// SystemProbe: 실시간 시스템 상황 측정 (이벤트루프 지연 및 메모리)
const probe = new SystemProbe();
const status = await probe.measure();

// Stealth Worker: 상황별 동적 딜레이 제어
if (status === "idle") await sleep(500);         // 한가함: 빠른 처리
else if (status === "normal") await sleep(2000); // 보통: 2초 단위 처리
else if (status === "busy") await sleep(5000);   // 부하: 5초 지연 인덱싱
else if (status === "critical") throw RetryException(); // 임계치 도달: 작업을 일시 중단하고 나중으로 연기
```

운영체제의 상태를 동적으로 추적(Resource-Aware)하여 인덱스 워커의 가동 빈도를 트로틀링(Throttling)한다. AI 인프라는 언제나 메인 서비스 개발 환경을 저해하지 않도록 철저히 통제 하에 둔다는 설계 원칙이다.

---

## 7. 결과 및 KPI 지표 향상

v11.0 개편 후, RAG 파이프라인 전반에 KPI 메트릭 대시보드를 부착하여 측정한 성과다:

| 지표 | Before (Vector Only v9.0) | After (GraphRAG v11.0) | 개선율 |
|------|---------------------------|------------------------|--------|
| **Multi-hop 질문 답변율** | ~20% | ~85% | **압도적 향상** |
| **사내 고유명사 적중률** | ~40% (유사어로 누락) | 99% (BM25 필터 보완) | **+59%p** |
| **환각(Hallucination) 발생** | 5회당 1회 | 20회당 1회 미만 (Self-RAG) | **80% 감소** |
| **응답 압축률 (Context)** | 원본 그대로 전달 | 7B 모델 요약 (약 40% 크기) | **초기 토큰 비용 최적화** |

**결론:**  
단순한 DB 덤프를 넘어, Neo4j 기반의 GraphRAG와 Hybrid 검색을 조합한 것은 AI에게 문서가 아닌 "맥락"을 부여하는 확실한 해답이었다.

---

*© 2026 ycy0922. 본 문서는 개인 포트폴리오 목적으로 작성되었으며, 사내 기밀 정보는 포함되어 있지 않습니다.*
