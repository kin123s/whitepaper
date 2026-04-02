# A Day in the Life: MCP Server 실전 구동 시나리오

> **Author:** ycy0922  
> **Version:** 1.0  
> **Date:** 2026-04-03  
> **Status:** Portfolio Document  
> **Companion to:** [Chapter 3 — MCP RAG Server Architecture](./04-mcp-rag-server-architecture.md)

---

## Abstract

지금까지 설계한 `ai-mcp-server`가 개발자의 책상 위에서, 사내 폐쇄망 환경을 배경으로 어떻게 동작하는지 보여주는 실전 시나리오다.

AI 에이전트(Claude, Cursor 등)는 단독으로는 사내망의 파편화된 레거시 코드를 읽을 수 없다. 하지만 `ai-mcp-server`라는 강력한 무기를 장착하는 순간, 단일 프롬프트 입력만으로 수년 치의 커밋 이력, 얽혀있는 지식 그래프, 보안 가이드라인이 어떻게 유기적으로 맞물려 에이전트의 답변으로 전달되는지 그 **백엔드 오케스트레이션(Ping-Pong) 흐름**을 단계별로 파헤쳐본다.

---

## 1. 상황 발생 (The Incident)

**사용자(개발자):**  
> *"업데이트 후 특정 레거시 PHP 모듈에서 DB 커넥션 예외(Exception)가 떨어지고 있어. `CORE_DB_WRP()` 관련 로직 문제인 것 같은데 원인 찾아서 고쳐줘."*

이 흔한 질문은 AI 입장에서는 최악의 상태다.
- `CORE_DB_WRP()`가 무슨 함수인지 AI는 모른다 (사내망 전용).
- 최근 업데이트 이력이 무엇인지 모른다.
- 시스템 DB 스키마를 모른다.

단순 프롬프트라면 AI는 흔한 인터넷 지식(환각)을 바탕으로 전혀 쓸모없는 코드를 생성해 낼 것이다. 하지만 MCP 서버가 개입하면 이야기가 달라진다.

---

## 2. MCP Server의 개입과 탐색 (The Orchestration)

### Step 2.1: 도구 호출과 BM25 매칭
AI 에이전트는 무언가 부족함을 깨닫고, MCP 프로토콜을 통해 사내망에 띄워진 `ai-mcp-server`에게 지식 조회를 요청한다.

```json
// IDE -> MCP Server (JSON-RPC)
{
  "method": "tools/call",
  "params": {
    "name": "search_memory",
    "arguments": {
      "query": "CORE_DB_WRP() 예외 로그 추가",
      "domain": "DOMAIN_A"
    }
  }
}
```

- **MCP 서버 1차 대응:** `CORE_DB_WRP`라는 사내 고유명사가 들어있다. 벡터 임베딩 모델(Ollama)은 이 단어의 의미를 정확히 캡쳐하지 못하므로, 서버의 **BM25(SQLite FTS5) 엔진**이 즉시 개입하여 해당 함수가 정의된 PHP 코어 파일 후보군을 추려낸다.

### Step 2.2: Self-RAG의 쿼리 재작성 (Query Rewriting)
서버 내부의 로컬 LLM(llama3)이 1차 검색된 파일들을 보고 관련도 평가(Relevance Judge)를 수행한다.
- **판단:** *"단순히 함수 원형만으로는 에러의 원인 추적이 불가능해!"*
- **재작성:** 질의응답 실패를 직감한 서버는 AI 에이전트의 개입 없이 **스스로 쿼리를 재작성**한다. 
  ➔ `Rewrite Query: "CORE_DB_WRP() 마이그레이션 이력 및 PDO 예외 처리 패턴"`

### Step 2.3: GraphRAG를 통한 다단계(Multi-hop) 추론
재작성된 쿼리로 본격적인 **Neo4j Graph Traversal**이 시작된다. 단순한 텍스트 매치에서 벗어나, 트랜잭션의 깊이(Depth)를 추적한다.

1. **Node 발견:** `db_wrapper.php` (파일 노드)
2. **Edge 추적:** 해당 파일을 2일 전에 수정한 `Commit Hash: 8b3f2a` (커밋 노드 발견)
3. **Edge 추적:** 커밋 메시지에 태그된 `JIRA-5032` (이슈 노드 발견)
4. **Context 보충:** JIRA 이슈 내용에 *"보안 권고에 따른 PDO 래퍼 파라미터 바인딩 구조 변경"* 이 명시되어 있음을 확인.

---

## 3. 결과 반환 및 에이전트의 결론 (The Synthesis)

MCP 서버는 위 2.3 단계에서 긁어모은 방대한 지식 그래프 트리를 토큰 예산에 맞추어 **동적 요약(Context Compression)** 한 뒤 IDE로 반환한다.

```json
// MCP Server -> IDE (JSON-RPC)
{
  "result": {
    "context": "[Entity: db_wrapper.php] 내부 통신 구조가 mysqli에서 PDO로 최근(2일 전) 마이그레이션 됨. (관련 커밋: 8b3f2a, 사유: 보안 조치).\n[규칙 안내] DOMAIN_A 에서는 더 이상 문자열 결합을 지원하지 않으며, bindParam()을 필수로 사용해야 함."
  }
}
```

**AI 에이전트의 최종 답변:**
> *"확인해 본 결과, 2일 전 `JIRA-5032` 배포로 인해 `CORE_DB_WRP()`의 내부 구현체가 PDO 기반으로 변경되었습니다. 예외가 발생한 이유는 기존 모듈에서 배열 대신 문자열로 인자를 넘기고 있기 때문입니다. 사내 보안 가이드라인에 맞추어 Prepared Statement 구조로 해당 호출부를 수정하겠습니다."*

---

## 4. 시나리오의 시사점 (Key Takeaways)

이 짧은 핑퐁(Ping-pong) 속에 아키텍트가 심어둔 수많은 억제기와 최적화 기술이 숨어있다.

1. **Zero-Trust 보장:** 이 모든 데이터 조립(Neo4j 탐색, Llama3 쿼리 조합)은 외부 API가 아닌 **사내망 폐쇄망 장비 안에서만** 이루어졌다. OpenAI나 Anthropic 서버로는 압축된 "결과 문맥"만 전달되므로 소스 코드 원본의 대량 유출이 방어되었다.
2. **GraphRAG의 파워 증명:** "PHP 파일" 하나만 던져주지 않고, 커밋과 이슈 티켓까지 연계해서 "왜(Why)" 이 코드가 변경되었는지 맥락을 부여했다.
3. **에이전트의 맹목성 차단:** AI가 혼자 추측해서 코드를 지어내는 할루시네이션(Hallucination) 구간이 원천 봉쇄되었다.

**이 시나리오는 단순한 RAG를 넘어, AI 미들웨어 서버가 '지식의 큐레이터' 역할을 수행할 때 에이전트의 생산성이 얼마나 극적으로 변화하는지 증명한다.**

(추후 사내망 실제 배포 및 KPI 데이터(정답률, 토큰 절감량 등) 수집이 완료되면, 그 실측 수치를 다루는 다음 챕터가 추가될 예정이다.)

---

*© 2026 ycy0922. 본 문서는 개인 포트폴리오 목적으로 작성되었으며, 사내 기밀 정보는 포함되어 있지 않습니다.*
