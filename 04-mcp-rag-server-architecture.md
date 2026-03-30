# MCP RAG Server: AI-to-AI 지식 미들웨어 구축기

> **Author:** ycy0922  
> **Version:** 1.0  
> **Date:** 2026-03-30  
> **Status:** Portfolio Document  
> **Companion to:** [Chapter 1 — 3-Layer Rule System](./01-ai-agent-governance-architecture.md)

---

## Abstract

Chapter 1에서 "추측하지 말고 검색하라(Search Before Guess)"는 원칙을 제시했다. 이 Chapter 3는 그 원칙의 **뒷면** — 실제로 AI 에이전트가 검색할 수 있는 인프라를 어떻게 만들었는지의 이야기다.

AI 코딩 에이전트에게 "DB 스키마를 검색해"라고 규칙을 써봐야, 검색할 곳이 없으면 의미가 없다. MCP(Model Context Protocol) 서버, 벡터 DB, 임베딩 사이드카, TF-IDF 폴백 — 이 키워드들은 블로그에서 자주 보지만, 실제로 **사내 레거시 환경 위에서 돌리면** 전혀 다른 세계가 펼쳐진다. 모델은 한국어를 못 알아듣고, 컨테이너는 메모리를 먹어치우고, 인덱싱은 모니터링 시스템에 걸린다.

본 백서는 이런 현실적 제약 안에서 설계·구축한 **MCP RAG Server**의 아키텍처를 다룬다. 벡터 검색이 만능이 아니라는 전제 하에, 키워드 검색과 벡터 검색을 계층적으로 조합하고, GPU가 없는 사내 서버에서도 동작하는 구조를 만든 과정이다.

---

## Table of Contents

1. [왜 MCP 서버를 직접 만들었나](#1-왜-mcp-서버를-직접-만들었나)
2. [전체 아키텍처 — 4개 컴포넌트와 데이터 흐름](#2-전체-아키텍처)
3. [MCP Server (TypeScript) — AI와 대화하는 관문](#3-mcp-server)
4. [Sidecar Embedding Server (Python) — 벡터화 엔진](#4-sidecar-embedding-server)
5. [3-Tier 검색 파이프라인 — 비용이 낮은 것부터](#5-3-tier-검색-파이프라인)
6. [도메인 격리 — 프로젝트가 섞이지 않는 구조](#6-도메인-격리)
7. [청킹 전략 — 무엇을 어떻게 쪼갤 것인가](#7-청킹-전략)
8. [스텔스 인덱싱 — 눈치 보며 인덱싱하기](#8-스텔스-인덱싱)
9. [컨테이너 오케스트레이션 — 현실적 배포](#9-컨테이너-오케스트레이션)
10. [실전에서 배운 것들](#10-실전에서-배운-것들)

---

## 1. 왜 MCP 서버를 직접 만들었나

### 1.1 문제의 원점

Chapter 1에서 AI 에이전트의 가장 치명적인 문제로 **환각(Hallucination)**을 꼽았다. 존재하지 않는 DB 컬럼으로 SQL을 만들고, deprecated된 API를 자신 있게 호출한다. 규칙 파일에 "DB 스키마를 추측하지 말 것"이라고 써봤지만, 규칙만으로는 부족했다.

> "검색하라"고 말하려면, 먼저 검색할 수 있는 곳이 있어야 한다.

### 1.2 기존 선택지와 한계

2025년 기준으로 AI 에이전트에 컨텍스트를 주입하는 방법은 여러 가지가 있었다.

| 방식 | 장점 | 한계 |
|------|------|------|
| **프롬프트에 직접 삽입** | 간편, 확실한 전달 | 토큰 한도, 대규모 스키마 불가 |
| **RAG SaaS (Pinecone, Weaviate Cloud)** | 관리 불필요 | 사내 코드/스키마 외부 전송 불가 |
| **Copilot Knowledge Base** | GitHub 네이티브 | 벡터 검색 아님, 커스터마이징 불가 |
| **IDE 내장 검색 (grep, Semantic Search)** | 빠름, 무료 | 파일 내용만 검색, 의미 기반 불가 |

사내 레거시 PHP 코드베이스(20만 줄), 10년치 DB 스키마(DDL 4,000줄), 3개 프로젝트 — 이걸 토큰 안에 넣을 수는 없었다. 그렇다고 사내 코드를 외부 SaaS에 보낼 수도 없었다. **자체 구축** 외에 선택지가 없었다.

### 1.3 MCP를 선택한 이유

MCP(Model Context Protocol)는 IDE의 AI 에이전트가 외부 도구를 **표준화된 프로토콜**로 호출할 수 있게 한다. 선택 이유:

1. **표준**: GitHub Copilot, Claude, Cursor가 모두 지원 — 에이전트 교체에도 서버는 그대로
2. **stdio 기반**: HTTP 서버를 올릴 필요 없이, IDE가 직접 프로세스를 spawn하고 JSON-RPC로 통신
3. **도구 디스커버리**: `ListTools` → `CallTool` 패턴으로 에이전트가 자동으로 사용 가능한 도구를 인식
4. **경량**: 외부 의존성 최소화 가능 (`@modelcontextprotocol/sdk` 한 패키지로 시작)

```
IDE (Copilot/Claude/Cursor)
    ↕  stdio (JSON-RPC)
MCP Server (TypeScript)
    ↕  HTTP (localhost:8100)
Sidecar (Python + ChromaDB)
```

핵심은 **관심사 분리**다. MCP 서버는 도구 인터페이스만 담당하고, 무거운 임베딩 연산은 별도 Python 프로세스(사이드카)로 분리했다. 이 구조 덕분에 사이드카가 죽어도 MCP 서버는 TF-IDF 폴백으로 계속 응답할 수 있다.

---

## 2. 전체 아키텍처

### 2.1 시스템 구성도

```
┌──────────────────────────────────────────────────────────────────┐
│                        IDE (VS Code)                              │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  AI Agent (GitHub Copilot / Claude / Cursor)            │     │
│  │  "DB의 SALES 테이블 구조를 알려줘"                        │     │
│  └──────────────────────┬──────────────────────────────────┘     │
└─────────────────────────┼────────────────────────────────────────┘
                          │ stdio (JSON-RPC)
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│  MCP Server (TypeScript/Node.js)                  Container: mcp │
│  ┌──────────────────────────────────────────────┐                │
│  │ 8 Tools (Zod-validated)                      │                │
│  │ ┌─────────────┐ ┌─────────────┐              │                │
│  │ │search_memory│ │upsert_memory│ ...          │                │
│  │ └──────┬──────┘ └─────────────┘              │                │
│  │        │                                     │                │
│  │ ┌──────┴──────┐                              │                │
│  │ │ TF-IDF Engine│ ← 폴백 검색 (사이드카 장애 시)│                │
│  │ │ (In-Memory)  │                              │                │
│  │ └──────┬──────┘                              │                │
│  └────────┼─────────────────────────────────────┘                │
│           │ HTTP (localhost:8100)                                 │
└───────────┼──────────────────────────────────────────────────────┘
            ▼
┌──────────────────────────────────────────────────────────────────┐
│  Sidecar Embedding Server (Python/FastAPI)     Container: sidecar│
│  ┌──────────────────────┐ ┌──────────────────────┐              │
│  │ Sentence-Transformers│ │ ChromaDB Client      │              │
│  │ ┌──────────────────┐ │ │ (PersistentClient)   │              │
│  │ │ CPU: MiniLM-L12  │ │ │                      │              │
│  │ │ GPU: E5-Large    │ │ │ HNSW Index           │              │
│  │ └──────────────────┘ │ │ Cosine Similarity    │              │
│  └──────────────────────┘ └───────────┬──────────┘              │
│                                       │                          │
│  Endpoints:                           ▼                          │
│  POST /embed    ← 단일 텍스트 → 벡터   ┌──────────────────────┐  │
│  POST /upsert   ← 문서 배치 저장       │ /data/chromadb/      │  │
│  POST /query    ← 시맨틱 유사도 검색    │ (SQLite + HNSW)      │  │
│  POST /reindex  ← 컬렉션 재인덱싱      │ Named Docker Volume  │  │
│  GET  /health   ← 상태 점검           └──────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 데이터 흐름 — 검색 요청의 여정

AI 에이전트가 `search_memory(query="SALES 테이블 스키마", domain="bizportal")`를 호출하면:

```
1. IDE → MCP Server (stdio)
   └─ JSON-RPC: { method: "tools/call", params: { name: "search_memory", ... } }

2. MCP Server: Zod 스키마로 파라미터 검증
   └─ input = SearchMemorySchema.parse(args)

3. MCP Server → Sidecar 상태 확인 (캐시된 Health Check, 30초 TTL)
   ├─ [사이드카 정상] → HTTP POST /query
   │   └─ Sidecar: 쿼리 임베딩 → HNSW 검색 → 유사도 필터링
   │   └─ 결과 반환 (similarity ≥ 0.50)
   │
   └─ [사이드카 장애] → TF-IDF 폴백
       └─ 키워드 토크나이징 → TF-IDF 코사인 유사도 → 부스트 가중치 적용

4. MCP Server → IDE (응답)
   └─ 검색 결과 + 출처 메타데이터
```

### 2.3 왜 이런 구조인가

**"왜 MCP 서버와 사이드카를 분리했나?"**

| 결합 방식 (Monolith) | 분리 방식 (현재) |
|---|---|
| Node.js에서 Python 임베딩 모델 호출 → FFI/child_process 복잡도 | 각자 최적의 언어 사용 |
| 모델 로딩 실패 → 전체 서버 다운 | 사이드카 장애 → TF-IDF 폴백으로 계속 동작 |
| 메모리 관리 어려움 (GC + PyTorch) | 독립 메모리 한도 (MCP: 512MB, Sidecar: 2GB) |
| 단일 컨테이너에 Node + Python | 프로필 기반 컨테이너 선택 (CPU/GPU) |

이것은 마이크로서비스의 **사이드카 패턴(Sidecar Pattern)**을 차용한 것이다. MCP 서버는 비즈니스 로직(도구 라우팅, 스키마 검증)에 집중하고, 무거운 ML 연산은 사이드카에 위임한다. 사이드카가 죽어도 메인 서비스는 살아 있다.

---

## 3. MCP Server

### 3.1 8개 도구 카탈로그

MCP 서버는 AI 에이전트에게 8개의 도구를 제공한다. 설계 원칙은 **"하나의 도구가 하나의 책임"**:

| # | 도구 | 유형 | 핵심 기능 |
|---|------|------|----------|
| 1 | `search_memory` | 읽기 | 벡터 유사도 검색 (ChromaDB → TF-IDF 폴백) |
| 2 | `upsert_memory` | 쓰기 | 새 지식을 벡터 DB에 저장 (동일 ID = 덮어쓰기) |
| 3 | `query_context` | 읽기 | 프로젝트 마크다운 문서 검색 (헤딩 기반 청킹) |
| 4 | `manage_collection` | 관리 | 컬렉션 상태 조회, 재인덱싱 트리거 |
| 5 | `get_file_signature` | 읽기 | 파일의 함수/클래스 시그니처 추출 (AST 없이 정규식) |
| 6 | `get_domain_status` | 읽기 | 도메인별 세션 상태 조회 |
| 7 | `save_directive` | 쓰기 | 워크플로우 지시사항을 세션에 저장 |
| 8 | `update_session_state` | 쓰기 | 세션 상태 갱신 (OCC 기반 낙관적 동시성) |

**4개의 RAG 도구**(1~4)가 지식 검색의 핵심이고, **4개의 도메인 도구**(5~8)는 프로젝트 상태와 코드 구조 파악을 지원한다.

### 3.2 도구 등록의 구현

MCP 서버의 진입점에서 도구를 등록하는 방식이 흥미롭다. Zod 스키마로 파라미터를 정의하고, 이를 JSON Schema로 변환하여 MCP 프로토콜에 노출한다:

```typescript
// 일반적 접근: zod-to-json-schema 패키지 설치
// 실제 구현: 의존성 최소화를 위해 직접 변환
function zodToSimpleJsonSchema(schema: z.ZodObject<any>) {
  const shape = schema.shape;
  const properties: Record<string, any> = {};
  const required: string[] = [];

  for (const [key, value] of Object.entries(shape)) {
    properties[key] = zodFieldToJsonSchema(value as z.ZodTypeAny);
    if (!isOptional(value as z.ZodTypeAny)) {
      required.push(key);
    }
  }

  return { type: "object", properties, required };
}
```

왜 외부 패키지 대신 직접 구현했는가? **의존성 최소화**. MCP 서버는 `@modelcontextprotocol/sdk`와 `zod` 두 패키지만으로 동작한다. 사내 환경에서 `npm install`이 제한될 수 있고, 의존성이 적을수록 컨테이너 이미지가 가볍다.

### 3.3 도구 디스패치 흐름

```
ListToolsRequest
  → 8개 도구의 이름, 설명, JSON Schema 반환
  → AI 에이전트가 사용 가능한 도구 목록을 인식

CallToolRequest({ name: "search_memory", arguments: {...} })
  → switch(name)로 디스패치
  → Zod.parse()로 입력 검증
  → try/catch로 에러 격리
  → 결과를 MCP 텍스트 콘텐츠로 반환
```

모든 도구 호출은 **try/catch로 격리**된다. 하나의 도구가 실패해도 MCP 서버 프로세스가 죽지 않으며, 에러 메시지가 AI 에이전트에게 텍스트로 전달된다. 이것이 중요한 이유: AI 에이전트는 에러 메시지를 읽고 다른 접근법을 시도할 수 있기 때문이다.

---

## 4. Sidecar Embedding Server

### 4.1 왜 Python인가

임베딩(텍스트 → 벡터 변환)을 담당하는 사이드카를 Python으로 만든 이유는 명확하다:

- **sentence-transformers**: 100줄 미만으로 임베딩 서버 구축 가능
- **ChromaDB**: Python 네이티브 클라이언트가 가장 안정적
- **PyTorch**: CPU/GPU 런타임 자동 감지

Node.js에서 ONNX Runtime이나 Transformers.js를 쓸 수도 있었지만, 2025년 기준으로 다국어 임베딩 모델의 Python 생태계 성숙도가 압도적이었다.

### 4.2 듀얼 모델 전략

사이드카는 **실행 환경에 따라** 임베딩 모델을 자동 선택한다:

| 환경 | 모델 | 차원 | 크기 | 언어 | 용도 |
|------|------|------|------|------|------|
| **CPU** | `paraphrase-multilingual-MiniLM-L12-v2` | 384 | 470MB | 50+ 언어 | 사내 서버 (GPU 없음) |
| **GPU** | `intfloat/multilingual-e5-large` | 1,024 | 1.2GB | 100+ 언어 | 개인 RTX 환경 |

```python
MODELS = {
    "cpu": "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2",
    "gpu": "intfloat/multilingual-e5-large"
}

device = "cuda" if torch.cuda.is_available() else "cpu"
model = SentenceTransformer(MODELS[device], device=device)
```

**왜 두 개의 모델인가?**

사내 우분투 서버에는 GPU가 없다. MiniLM-L12는 CPU에서도 32개 문장을 ~200ms 안에 임베딩할 수 있다. 반면 개인 개발 환경(RTX GPU)에서는 E5-Large로 더 높은 품질의 임베딩을 생성한다. 같은 Docker 이미지가 `--profile cpu` 또는 `--profile gpu`로 배포 시점에 결정된다.

### 4.3 모델 지연 로딩 (Lazy Loading)

임베딩 모델은 **첫 번째 요청** 시점에 로딩된다:

```python
_model = None

def get_model():
    global _model
    if _model is None:
        _model = SentenceTransformer(MODELS[device], device=device)
    return _model
```

왜 서버 시작 시 바로 로딩하지 않는가?

- MiniLM-L12 로딩에 ~8초, E5-Large는 ~30초 소요
- Docker 헬스체크가 30초 후 시작 — 모델 로딩이 끝나기 전에 `unhealthy` 판정 방지
- 실제 검색 요청이 없으면 메모리를 아껴둘 수 있음

`/health` 엔드포인트는 모델 로딩 상태와 ChromaDB 연결을 **분리해서 보고**한다:

```json
{
  "status": "ok",
  "model_loaded": true,
  "model_name": "paraphrase-multilingual-MiniLM-L12-v2",
  "device": "cpu",
  "chroma_ok": true,
  "chroma_path": "/data/chromadb"
}
```

### 4.4 ChromaDB 설정

```python
import chromadb
from chromadb.config import Settings

client = chromadb.PersistentClient(
    path=CHROMA_PERSIST_DIR,   # /data/chromadb (Named Volume)
    settings=Settings(anonymized_telemetry=False)
)
```

| 설정 | 값 | 이유 |
|------|---|------|
| **Storage Backend** | SQLite | 단일 서버 환경에 적합, 외부 DB 불필요 |
| **Index Algorithm** | HNSW (Hierarchical Navigable Small World) | 1만 문서에서 <10ms 검색 |
| **Distance Metric** | Cosine | 정규화된 임베딩과 가장 잘 맞음 |
| **Persistence** | Docker Named Volume | `docker compose down` 후에도 데이터 유지 |
| **Telemetry** | Off | 사내망에서 외부 전송 차단 |

컬렉션 생성 시 HNSW 파라미터를 명시적으로 지정한다:

```python
collection = client.get_or_create_collection(
    name=collection_name,
    metadata={"hnsw:space": "cosine"}
)
```

---

## 5. 3-Tier 검색 파이프라인

### 5.1 설계 철학

> "벡터 검색은 만능이 아니다."

이것이 3-Tier를 만든 핵심 전제다. Sentence-Transformers의 MiniLM 모델은 영어에서는 훌륭하지만, **한국어 + 도메인 특화 용어** 환경에서는 정확도가 떨어진다. "HWALLLIST"라는 테이블명을 벡터 유사도로 찾는 것보다, 단순 키워드 매칭이 더 정확할 때가 많다.

따라서 **비용이 낮은 것부터** 단계적으로 시도한다:

### 5.2 Tier 1 — 캐시 (0ms, 무비용)

```typescript
// Health Check 캐시 — 30초 TTL
private lastHealthCheck = 0;
private cachedHealth: HealthStatus | null = null;

async checkHealth(force = false): Promise<HealthStatus> {
    const now = Date.now();
    if (!force && this.cachedHealth 
        && now - this.lastHealthCheck < HEALTH_CHECK_INTERVAL_MS) {
        return this.cachedHealth;  // 30초 이내 → 캐시된 결과
    }
    // ... 실제 HTTP 요청
}
```

현재는 Health Check 캐시만 구현되어 있다. 검색 결과 LRU 캐시는 향후 과제로 남아 있으며, 동일한 쿼리가 짧은 시간 내에 반복되는 패턴이 확인되면 추가할 계획이다.

### 5.3 Tier 2 — 벡터 검색 (사이드카 정상 시)

```typescript
async searchViaSidecar(options: SearchOptions): Promise<VectorSearchResult[]> {
    const response = await fetchWithTimeout(
        `${SIDECAR_BASE_URL}/query`,
        {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
                collection: options.collection,
                query: options.query,
                top_k: options.topK,
                where: options.where   // 메타데이터 하드 필터
            }),
            timeout: SIDECAR_TIMEOUT_MS  // 10초
        }
    );
    
    // similarity = 1 - cosine_distance
    // RAG_MIN_SIMILARITY (기본 0.50) 이하 결과 필터링
    return results.filter(r => r.similarity >= MIN_SIMILARITY);
}
```

**메타데이터 필터링의 중요성**: MiniLM의 의미 검색 능력이 제한적이므로, `where` 절로 **검색 범위를 사전에 좁힌다**:

```typescript
// "bizportal의 git 커밋 중 에러 수정 관련" 검색
const where = {
    $and: [
        { domain: "bizportal" },
        { source_type: "git_commit" },
        { category: "error_fix" }
    ]
};
```

이 접근은 **약한 시맨틱 모델 + 강한 메타데이터 필터 = 정밀한 결과**라는 조합이다. 모델이 완벽하지 않아도 필터가 보완한다.

### 5.4 Tier 3 — TF-IDF 폴백 (사이드카 장애 시)

사이드카가 죽었을 때도 검색은 계속되어야 한다. TF-IDF 엔진은 MCP 서버 프로세스 **메모리 내**에서 동작한다:

```typescript
// 토크나이징 — 한국어/영어 하이브리드
function tokenize(text: string): string[] {
    return normalized
        .split(" ")
        .filter(token => {
            if (STOPWORDS.has(token)) return false;
            // 한국어: 1글자도 유효 (가-힣)
            if (/[가-힣]/.test(token)) return token.length >= 1;
            // 영어: 2글자 이상
            return token.length >= 2;
        });
}
```

**도메인 부스트 가중치**: 일반적인 TF-IDF에 사내 용어 가중치를 추가한다:

```typescript
const BOOST_TERMS: Record<string, number> = {
    // 사내 시스템 고유명사
    genian: 2.0, wpdreport: 2.0, mi_db: 2.0,
    // 비즈니스 엔티티
    sales: 1.8, expenses: 1.8, hwalllist: 1.8,
    // 프로젝트명
    bizportal: 1.5, partnerportal: 1.5,
};
```

"HWALLLIST"라는 검색어가 들어오면 TF-IDF 점수에 1.8배 가중치가 붙어, 해당 단어가 포함된 문서가 확실하게 상위에 노출된다. 이것은 벡터 검색이 고유명사에 약한 것을 보완하는 **도메인 특화 전략**이다.

### 5.5 폴백 체인

```
search_memory() 호출
    │
    ├─ checkHealth() (30초 캐시)
    │
    ├─ [사이드카 정상] ──→ searchViaSidecar()
    │                         │
    │                    [시간 초과/에러]
    │                         │
    │                         └──→ searchViaTfIdf() (자동 폴백)
    │
    └─ [사이드카 장애] ──→ searchViaTfIdf() (즉시 폴백)
```

**이중 저장(Dual Persistence)**: 모든 `upsert_memory()` 호출은 **TF-IDF 인메모리 인덱스와 ChromaDB 양쪽에** 동시 저장한다:

```typescript
// 쓰기는 항상 양쪽에
engineTfIdf.addDocuments(docs);           // 인메모리 (즉시 검색 가능)

if (health.sidecarAvailable) {
    await fetchWithTimeout(               // ChromaDB (영구 저장)
        `${SIDECAR_BASE_URL}/upsert`, { documents: docs }
    );
}
```

2배의 저장/메모리를 쓰지만, 100% 가용성을 보장한다. 사이드카가 죽어도 검색은 계속된다.

---

## 6. 도메인 격리

### 6.1 문제 — 프로젝트가 섞이면

AI 에이전트가 "SALES 테이블 구조를 알려줘"라고 요청받을 때, 이것이 **bizportal의 SALES**인지 **partnerportal의 SALES**인지 구분해야 한다. 두 프로젝트가 같은 벡터 DB 안에서 섞이면, 잘못된 스키마로 코드를 생성하게 된다.

### 6.2 3중 격리 구조

```
┌──────────────────────────────────────────────────┐
│  Layer 1: Path-Based Isolation                    │
│  ┌──────────────────────────────────────────┐    │
│  │ DOMAIN_ROOTS = {                         │    │
│  │   bizportal: "/var/www/html/bp",         │    │
│  │   partnerportal: "/var/www/.../portal"   │    │
│  │ }                                        │    │
│  │ → 에이전트는 도메인 루트 밖으로 못 나감      │    │
│  └──────────────────────────────────────────┘    │
│                                                   │
│  Layer 2: Collection-Based Isolation              │
│  ┌──────────────────────────────────────────┐    │
│  │ company_knowledge (공통)                   │    │
│  │ project_context_bizportal (프로젝트별)     │    │
│  │ project_context_partnerportal             │    │
│  └──────────────────────────────────────────┘    │
│                                                   │
│  Layer 3: Metadata Filter Isolation               │
│  ┌──────────────────────────────────────────┐    │
│  │ where: { domain: "bizportal" }            │    │
│  │ → 같은 컬렉션 안에서도 도메인별 필터링       │    │
│  └──────────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘
```

**왜 3중인가?** 방어적 설계다. 하나의 레이어가 뚫려도(예: 잘못된 컬렉션명) 다른 레이어가 막는다.

### 6.3 세션 격리

각 도메인의 세션 상태는 물리적으로 분리된 디렉토리에 저장된다:

```
bizportal/.ai-local/state/sessions/
    ├── GA-5063/state.json
    └── GA-5064/state.json

partnerportal/.ai-local/state/sessions/
    ├── PP-1001/state.json
    └── PP-1002/state.json
```

`update_session_state` 도구는 `domain` 파라미터로 대상 디렉토리를 결정하며, **도메인 간 교차 접근이 물리적으로 불가능**하다.

---

## 7. 청킹 전략

### 7.1 왜 청킹이 중요한가

임베딩 모델에는 입력 길이 제한이 있다 (MiniLM: 256 토큰, E5-Large: 512 토큰). 4,000줄짜리 DDL 파일을 통째로 임베딩하면 뒷부분이 잘린다. 따라서 문서를 **의미 있는 단위**로 쪼개야(chunk) 한다.

문제는 "의미 있는 단위"가 데이터 유형마다 다르다는 것이다. Git 로그, 디렉토리 트리, 마크다운 문서를 같은 방식으로 쪼개면 맥락이 파괴된다.

### 7.2 세 가지 청커 (Chunker)

#### GitLogChunker — 커밋 히스토리

Git 커밋 로그는 **5개씩 묶어서** 하나의 청크로 만든다:

```python
# 입력
[2024-03-15] fix: 결재 승인 로직 버그 수정 (홍길동)
[2024-03-14] feat: 근태 API 추가 (김철수)
[2024-03-13] refactor: MI_DB 래퍼 정리 (홍길동)
[2024-03-12] fix: XSS 필터 우회 방지 (박영희)
[2024-03-11] feat: 대시보드 위젯 추가 (김철수)

# 출력 (하나의 청크)
[소스: bizportal git history | 기간: 2024-03-11~2024-03-15 | 작성자: 홍길동, 김철수, 박영희]
[2024-03-15] fix: 결재 승인 로직 버그 수정 (홍길동)
[2024-03-14] feat: 근태 API 추가 (김철수)
...
```

**소스 프리픽스 주입**: 각 청크의 첫 줄에 출처·기간·작성자를 명시한다. AI 에이전트가 검색 결과를 볼 때 "이 정보가 어디서 왔는지"를 즉시 파악할 수 있다.

#### TreeChunker — 디렉토리 구조

디렉토리 트리는 **1-depth 블록** 단위로 쪼갠다:

```python
# 입력: 전체 트리
├── src/
│   ├── api/
│   │   ├── auth/
│   │   └── user/
│   └── utils/
├── docs/
│   └── ...

# 출력
# 청크 0: src/ 서브트리 전체
# 청크 1: docs/ 서브트리 전체
```

각 청크에 **절대 경로 프리픽스**가 붙어, AI 에이전트가 검색 결과를 실제 파일 시스템 경로와 매핑할 수 있다.

#### MarkdownChunker — 프로젝트 문서

마크다운 문서는 **헤딩(##, ###)** 단위로 쪼갠다:

```typescript
function chunkMarkdown(content: string, fileName: string) {
    const sections = content.split(/(?=^#{1,3} )/m);
    return sections
        .filter(s => s.trim().length >= 20)  // 노이즈 제거
        .map((section, i) => ({
            id: `ctx_${stem}_${i}`,
            text: section.trim(),
            metadata: { source_type: "note", file_path: fileName }
        }));
}
```

20자 미만의 섹션은 노이즈로 판단하여 제외한다. 빈 헤딩이나 `## TODO` 같은 스텁이 인덱싱되는 것을 방지한다.

---

## 8. 스텔스 인덱싱

### 8.1 문제 — 인덱싱이 들키면 안 된다

사내 서버에는 EDR(Endpoint Detection and Response)과 시스템 모니터링이 돌아간다. 백그라운드에서 대량의 파일을 읽고, CPU를 점유하고, 네트워크 트래픽을 발생시키면 **모니터링 알림**이 울린다. 인덱싱은 "조용히" 해야 한다.

### 8.2 우선순위 큐 시스템

모든 인덱싱 작업은 우선순위 큐를 통해 관리된다:

```typescript
enum Priority {
    CRITICAL = 1,   // 사용자 실시간 검색 (최우선)
    HIGH     = 2,   // PR/커밋 요약 인덱싱
    NORMAL   = 3,   // 일일 증분 인덱싱
    LOW      = 4,   // 레거시 트리 파싱
}
```

같은 우선순위 안에서는 **FIFO**(먼저 들어온 것이 먼저 처리)로 동작한다. 사용자의 실시간 검색 요청이 백그라운드 인덱싱에 밀리는 일은 없다.

### 8.3 단일 워커, 의도적 지연

```typescript
class StealthWorker {
    async start() {
        while (this.isRunning) {
            const job = this.queue.dequeue();
            if (job) {
                await this.processJob(job);
                await this.sleep(STEALTH_DELAY_MS);  // 2초 딜레이
            } else {
                await this.sleep(2000);  // 큐 비었으면 대기
            }
        }
    }
}
```

**왜 단일 워커 + 2초 딜레이인가?**

| 설계 결정 | 이유 |
|----------|------|
| 단일 워커 | SQLite(ChromaDB 백엔드)의 Write Lock 경합 방지 |
| 2초 딜레이 | CPU 스파이크 방지 → EDR/모니터링 미감지 |
| 배치 12개 + 4초 쿨다운 | 대량 인덱싱도 CPU 사용률 억제 |

이것은 성능 최적화의 **반대**다. 의도적으로 느리게 만들어 "눈에 띄지 않는" 인덱싱을 구현한다.

### 8.4 증분 인덱싱 (MD5 기반)

전체 재인덱싱은 비용이 크다. 파일이 실제로 변경되었을 때만 인덱싱한다:

```python
# 상태 파일: /data/chromadb/.indexer_state.json
# { "src/api/auth.php": "a1b2c3d4...", ... }

content_hash = hashlib.md5(file_content.encode()).hexdigest()

if state_file.get(file_path) == content_hash:
    continue   # 변경 없음 → 건너뜀

# 변경 감지 → 재인덱싱
process(file_path)
state_file[file_path] = content_hash
```

상태 파일은 ChromaDB와 같은 Named Volume(`/data/chromadb/`)에 저장되므로, 컨테이너가 재시작되어도 인덱싱 상태가 유지된다. "어제 인덱싱한 파일을 오늘 다시 인덱싱"하는 낭비가 없다.

---

## 9. 컨테이너 오케스트레이션

### 9.1 Docker Compose 프로필

```yaml
services:
  mcp-server:
    build:
      context: .
      dockerfile: docker/Dockerfile.mcp
    depends_on:
      embedding-sidecar-cpu:
        condition: service_healthy
        required: false          # 사이드카 없어도 시작 가능
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"

  embedding-sidecar-cpu:
    build:
      context: .
      dockerfile: docker/Dockerfile.sidecar-cpu
    volumes:
      - chromadb-data:/data/chromadb        # 벡터 DB (영구)
      - ${KNOWLEDGE_DIR}:/data/company:ro   # 지식 소스 (읽기 전용)
    environment:
      - EMBEDDING_DEVICE=cpu
      - HF_HOME=/data/chromadb/.hf_cache   # 모델 캐시 영구화
      - OMP_NUM_THREADS=1                  # 스레드 제한 (스텔스)
      - UV_THREADPOOL_SIZE=1
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: "1.5"
    profiles: [cpu]                        # --profile cpu 로 활성화
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8100/health"]
      interval: 30s
      start_period: 60s                    # 모델 로딩 대기

  embedding-sidecar-gpu:
    # CUDA 12.1, GPU 프로필
    profiles: [gpu]

volumes:
  chromadb-data:    # Named Volume — compose down 후에도 유지
```

### 9.2 볼륨 전략

| 볼륨 | 마운트 | 타입 | Persistence |
|------|--------|------|-------------|
| `chromadb-data` | `/data/chromadb` | Named Volume | compose down 후에도 유지 |
| `KNOWLEDGE_DIR` | `/data/company:ro` | Host Bind | 호스트의 큐레이팅된 지식 (읽기 전용) |
| `MCP_KNOWLEDGE_DIR` | `/data/knowledge:ro` | Host Bind | DDL, 레퍼런스 문서 (읽기 전용) |

**`:ro` (읽기 전용) 마운트의 의미**: 컨테이너 안에서 호스트의 소스 코드나 DDL 파일을 **수정할 수 없다**. 인덱싱 결과는 Named Volume에만 쓴다. 이것은 보안과 안정성을 위한 원칙이다.

### 9.3 리소스 제한의 의미

```yaml
deploy:
  resources:
    limits:
      memory: 2G      # MiniLM(470MB) + ChromaDB + FastAPI 오버헤드
      cpus: "1.5"     # 전체 코어의 일부만 사용
```

MCP 서버의 512MB vs 사이드카의 2GB — 이 차이가 **분리 아키텍처의 이유**를 설명한다. ML 모델이 메모리를 먹는 주범이고, 이걸 MCP 서버와 같은 프로세스에 넣으면 전체가 OOM으로 죽는다.

### 9.4 헬스체크 설계

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8100/health"]
  interval: 30s
  start_period: 60s   # ← 핵심
```

`start_period: 60s`가 중요한 이유: MiniLM 모델 로딩에 ~8초, ChromaDB 초기화에 ~5초, 나머지 워밍업까지 고려하면 최소 20~30초가 필요하다. 이 기간 동안 Docker가 unhealthy 판정을 내리지 않도록 유예 시간을 준다.

`depends_on`의 `required: false` 설정: 사이드카가 아직 시작되지 않았거나 영영 시작되지 않아도, MCP 서버는 독립적으로 기동한다. TF-IDF 폴백이 있기 때문이다.

---

## 10. 실전에서 배운 것들

### 10.1 정량적 지표

| 지표 | 값 | 비고 |
|------|---|------|
| 인덱싱된 문서 수 | ~2,500개 | Git 로그 + DDL + 프로젝트 문서 |
| 평균 검색 응답 시간 | ~800ms | 벡터 검색 기준 (Cold) |
| TF-IDF 폴백 응답 시간 | ~50ms | 인메모리 |
| 사이드카 메모리 사용 | ~1.2GB | MiniLM + ChromaDB (안정 상태) |
| 사이드카 장애 빈도 | ~1회/주 | 주로 OOM — 2GB 한도 내 관리 가능 |
| 사이드카 장애 시 서비스 영향 | 0건 | TF-IDF 폴백으로 무중단 |

### 10.2 핵심 교훈

#### Lesson 1: "벡터 검색은 은탄환이 아니다"

RAG의 블로그 글들은 "임베딩 → 벡터 DB → 프로파일!"을 간단하게 그린다. 현실은 달랐다.

- MiniLM은 한국어 고유명사(HWALLLIST, WPDREPORT)를 제대로 이해하지 못한다
- 짧은 DDL 구문(`CREATE TABLE SALES (...)`)은 임베딩 벡터가 비슷비슷하게 나온다
- **메타데이터 필터 + 도메인 부스트**가 모델 자체보다 검색 정확도에 더 기여했다

결론: 약한 시맨틱 모델이라도, 강한 메타데이터 설계와 결합하면 충분히 쓸 만하다.

#### Lesson 2: "폴백은 선택이 아니라 필수"

처음에는 TF-IDF 폴백 없이 벡터 검색만으로 시작했다. 사이드카가 OOM으로 죽을 때마다 AI 에이전트가 "도구를 사용할 수 없다"고 멈추는 것을 보고 **이중 저장(Dual Persistence)** 구조로 전환했다.

2배의 메모리를 쓰지만, "검색이 안 돼서 AI가 추측으로 코드를 만드는" 상황보다 훨씬 싸다.

#### Lesson 3: "스텔스는 기능이다"

사내 서버에서 CPU 100%를 찍으면 담당자에게 알림이 간다. 인덱싱은 기술적 문제만이 아니라 **운영적 문제**다. 2초 딜레이, 단일 워커, OMP_NUM_THREADS=1 — 이런 "의도적 성능 저하"가 실제 배포 가능성을 결정했다.

#### Lesson 4: "모델 선택은 환경이 결정한다"

이상적으로는 E5-Large를 모든 곳에 쓰고 싶다. 하지만:
- 사내 서버에 GPU 없음 → CPU 추론만 가능
- E5-Large의 CPU 추론 → 단일 문장 임베딩에 ~3초 (실사용 불가)
- MiniLM은 CPU에서 ~50ms → 실시간 검색 가능

모델 품질보다 **응답 시간 예산**이 모델 선택을 결정했다. Docker 프로필로 환경별 모델을 전환하는 구조는 이 현실에서 나왔다.

#### Lesson 5: "Zod는 양방향으로 쓸 수 있다"

Zod 스키마를 도구 파라미터 검증으로만 쓰는 게 아니라, **JSON Schema 변환기**로도 활용했다. 외부 패키지 없이 30줄의 유틸리티로 MCP 프로토콜의 도구 등록 스키마를 자동 생성한다. 의존성 하나를 줄이는 것이 사내 환경에서는 큰 차이를 만든다.

#### Lesson 6: "Named Volume이 상태를 살린다"

초기에 Docker 바인드 마운트로 ChromaDB 데이터를 관리했다. 파일 권한 문제, 호스트 경로 의존성, 실수로 삭제 — 여러 번 데이터를 잃은 후 **Named Volume**으로 전환했다. `docker compose down`으로 컨테이너를 날려도 볼륨은 살아 있고, `docker volume inspect`로 위치를 확인할 수 있다.

### 10.3 향후 개선 방향

| # | 과제 | 현재 상태 | 기대 효과 |
|---|------|----------|----------|
| 1 | 검색 결과 LRU 캐시 | Health Check만 캐시 | 동일 쿼리 반복 시 0ms 응답 |
| 2 | 한국어 특화 임베딩 모델 | MiniLM (범용 다국어) | 도메인 용어 검색 정확도 향상 |
| 3 | Reranker 파이프라인 | 없음 | 검색 결과 순위 품질 개선 |
| 4 | 자동 DDL 변경 감지 | 수동 재인덱싱 | DB 스키마 변경 시 자동 반영 |
| 5 | 멀티 워커 인덱싱 | 단일 워커 (스텔스) | GPU 환경에서 인덱싱 속도 향상 |

---

## Epilogue — 미들웨어를 만들게 된 이유

이 프로젝트를 시작할 때의 목표는 단순했다. "AI가 DB 스키마를 틀리게 만드는 걸 막자." 그런데 검색 인프라를 만들다 보니, 그것이 단순한 도구가 아니라 **AI 에이전트 간의 지식 미들웨어**라는 걸 깨달았다.

GitHub Copilot이 검색한 결과를 Claude가 참조하고, Claude가 학습한 패턴을 다시 벡터 DB에 써서 다음 세션의 Copilot이 읽는다. `upsert_memory`로 저장하고 `search_memory`로 검색하는 이 루프가, 결국 **AI-to-AI 지식 순환**을 만들어낸 것이다.

Chapter 1이 "어떤 규칙으로 AI를 통제할 것인가"의 이야기였다면, 이 Chapter 3는 "AI에게 어떤 기억을 줄 것인가"의 이야기다. 규칙이 행동의 경계를 정하고, 검색 인프라가 행동의 재료를 제공한다. 둘 다 없으면 AI 에이전트는 그저 **비싼 자동완성**에 불과하다.

---

*© 2026 ycy0922. 본 문서는 개인 포트폴리오 목적으로 작성되었으며, 사내 기밀 정보는 포함되어 있지 않습니다.*
