# Whitepaper Collection

> **실무에서 AI를 쓰다 보니, 결국 구조를 만들게 되었다.**

---

## 배경

2025년, AI 코딩 에이전트(GitHub Copilot, Claude, Cursor)를 실무 프로젝트에 본격 도입하면서 하나의 질문이 계속 돌아왔다.

> *"AI한테 어떻게 하면 우리 코드베이스의 맥락을 더 잘 전달할 수 있을까?"*

처음에는 단순했다. `copilot-instructions.md`에 규칙 몇 줄 적고, 프롬프트를 잘 쓰면 되겠지 싶었다. 하지만 레거시 PHP 20만 줄, 10년치 DB 스키마, 3개 프로젝트를 넘나드는 현실 앞에서 그건 금방 한계에 부딪혔다.

AI가 존재하지 않는 테이블 컬럼으로 SQL을 만들고, 어제 알려준 규칙을 오늘은 잊어버리고, 프로젝트 A의 규칙을 프로젝트 B에서 무시했다. 이런 문제들을 하나씩 해결하다 보니 — 규칙을 계층화하고, 토큰 예산을 관리하고, 검색을 강제하고, 상태를 압축하는 — **하나의 아키텍처**가 만들어졌다.

이 백서 시리즈는 그 과정에서 고민하고 시행착오한 것들을 정리한 것이다. AI 활용법만 다루는 게 아니라, 실무 환경에서 AI와 사람이 어떻게 협업 구조를 만들어가는지, 그리고 그것이 결국 **소프트웨어 아키텍처 문제**라는 걸 발견한 이야기다.

---

## 목차

### Chapter 1. AI Agent Governance Architecture

AI 에이전트에게 "컨텍스트를 어떻게 전달할 것인가"라는 문제를 3-Layer 규칙 시스템으로 풀어낸 아키텍처.

| 문서 | 내용 |
|------|------|
| [01. 본편 — 3-Layer Rule System](./01-ai-agent-governance-architecture.md) | 문제 정의, 설계 원칙, L1/L2/L3 계층 구조, 에이전트 워크플로우, MCP/RAG 파이프라인, 보안 가드레일, 성과와 교훈 |
| [02. 기술 부록](./02-technical-appendix.md) | 전체 파일 트리, 규칙 명세 테이블, MCP 도구 카탈로그, State Schema, 동기화 스크립트 흐름도, 토큰 예산 계산표 |

---

### Chapter 2. Legacy Modernization

10년차 PHP 코드베이스를 멈추지 않고, Next.js 15와 자체 MVC를 공존시키며 점진적으로 현대화한 이야기.

| 문서 | 내용 |
|------|------|
| [03. 본편 — 10년차 PHP 위에서 모던 스택 공존시키기](./03-legacy-modernization.md) | Strangler Fig 전략, 3-Area 하이브리드 아키텍처, Custom MVC 브릿지, 데이터베이스 공유 패턴, Docker 기반 배포, 트레이드오프와 교훈 |

---

### Chapter 3. MCP RAG Server — AI-to-AI 지식 미들웨어 구축기

Chapter 1의 "검색을 강제한다"의 뒷면 — 실제 검색 인프라를 어떻게 만들었나.

| 문서 | 내용 |
|------|------|
| [04. 초기 아키텍처 — MCP RAG Server](./04-mcp-rag-server-architecture.md) | MCP 서버 아키텍처, 벡터 DB + 임베딩 사이드카, 3-Tier 검색 파이프라인, 도메인 격리, 청킹 전략, 스텔스 인덱싱, 컨테이너 오케스트레이션, 실전 교훈 |
| [05. 아키텍처 진화 — GraphRAG & Self-RAG](./05-evolution-to-graphrag.md) | 벡터 검색의 Multi-hop 한계점 진단, Neo4j 기반 Knowledge Graph 도입, Triple-Hybrid Search, Self-RAG 자가 교정 루프, KPI 지표 개선 결과 |

---

### Chapter 4. Enterprise AI Security

AI 에이전트를 사내망(폐쇄망) 환경에 안전하게 투입하기 위한 제로 트러스트(Zero-Trust) 보안 통제 아키텍처.

| 문서 | 내용 |
|------|------|
| [06. 본편 — Zero-Trust 기반 에이전트 오케스트레이션](./06-zero-trust-enterprise-ai.md) | 파괴적 행동 방지를 위한 컨테이너 샌드박싱, Local-First 프록싱 데이터 보호, Resource-Aware 워크로드 트로틀링, Identity 도메인 격리 원칙 |

---

### Chapter 5. A Day in the Life: System in Action

복잡한 모노레포와 MCP 서버 오케스트레이션이 실제 개발 환경에서 어떻게 '마법처럼' 조화롭게 동작하는지 보여주는 실전 유스케이스.

| 문서 | 내용 |
|------|------|
| [07. 본편 — MCP Server 실전 구동 시나리오](./07-mcp-in-action-scenario.md) | 사내망/폐쇄망 조건, GraphRAG와 에이전트 간의 통신 핑퐁 프로세스, 사내 전용 레거시 함수 탐색 및 에러 원인 분석 시나리오 |

---

## Author

**ycy0922**  
Software Engineer 

---

*© 2026 ycy0922. 개인 포트폴리오 목적으로 작성되었으며, 사내 기밀 정보는 포함되어 있지 않습니다.*
