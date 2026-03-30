# Legacy Modernization: 10년차 PHP 위에서 모던 스택 공존시키기

> **Author:** ycy0922  
> **Version:** 1.0  
> **Date:** 2026-03-30  
> **Status:** Portfolio Document

---

## Abstract

"전부 다시 만들자"는 가장 달콤하고, 가장 위험한 유혹이다. 10년간 쌓인 레거시 PHP 코드베이스 위에서 Next.js 15, 자체 MVC 프레임워크, REST API를 단계적으로 도입하며 **기존 시스템을 멈추지 않고 현대화**한 과정을 기록한다. "갈아엎기"가 아니라 "공존시키기"를 선택한 이유와, 그것이 만들어낸 3-Area 하이브리드 아키텍처의 설계·트레이드오프·실전 패턴을 다룬다.

---

## Table of Contents

1. [Problem Statement — 리라이트의 유혹과 현실](#1-problem-statement)
2. [전략 선택 — Strangler Fig Pattern](#2-전략-선택--strangler-fig-pattern)
3. [3-Area 하이브리드 아키텍처](#3-3-area-하이브리드-아키텍처)
4. [Area 1: Legacy PHP — 건드리지 않는 기술](#4-area-1-legacy-php--건드리지-않는-기술)
5. [Area 2: Core MVC Framework MVC — 레거시 위의 안전망](#5-area-2-Core MVC Framework-mvc--레거시-위의-안전망)
6. [Area 3: Next.js 15 — 새 피부](#6-area-3-nextjs-15--새-피부)
7. [브릿지 패턴 — 세 영역을 잇는 접착제](#7-브릿지-패턴--세-영역을-잇는-접착제)
8. [데이터베이스 — 하나의 진실, 세 가지 접근법](#8-데이터베이스--하나의-진실-세-가지-접근법)
9. [배포 아키텍처 — Docker 기반 공존](#9-배포-아키텍처--docker-기반-공존)
10. [트레이드오프와 교훈](#10-트레이드오프와-교훈)

---

## 1. Problem Statement

### 1.1 시스템의 현재 상태

exam-필터은 (주)exam필터 의 사내 ERP/인트라넷 시스템이다. 2006년부터 운영되어 현재까지 약 20년의 이력을 가진다.

| 지표 | 수치 |
|------|------|
| PHP 파일 수 | 400+ 개 |
| 코드 라인 수 (PHP) | ~200,000+ |
| 데이터베이스 테이블 | 100+ 개 (exam. 다수 DB) |
| 일일 사용자 | 사내 전 직원 |
| 커버 도메인 | 매출, 수금, 비용, 자산관리, 라이선스, 고객관리, 기술지원, HR 연동 |

### 1.2 레거시의 실체

"레거시"라는 단어는 부정적으로 들리지만, 이 시스템은 **10년간 회사의 핵심 운영을 지탱해온 동작하는 소프트웨어**다. 문제는 코드 자체가 아니라 **구조**에 있었다:

| 증상 | 구체적 예시 |
|------|-----------|
| **설정 체인의 복잡성** | `index.php` → `exam-config.php` → `exam-pre.php` → `exam-local.php` → `class2.php` → `exam-f.php`(3,600줄) |
| **보안 사각지대** | `$db->query($sql)` — 문자열 결합 방식의 SQL (Prepared Statement 미적용) |
| **거대 함수 파일** | `exam-f.php` 하나에 수백 개의 헬퍼 함수가 flat하게 나열 |
| **라우팅 부재** | 파일명이 곧 URL — `exam-manager.php` = `/exam-manager.php` |
| **UI 고착** | jQuery 1.x/3.x + jQuery UI + jsTree + Bootstrap — SPA 전환 불가 |
| **동시 장애 전파** | 하나의 PHP 파일에서 DB 연결·인증·비즈니스 로직·뷰 렌더링이 모두 발생 |

### 1.3 "다시 만들자" vs "공존시키자"

| 전략 | 장점 | 리스크 |
|------|------|--------|
| **Big Bang Rewrite** | 깨끗한 아키텍처 | 1~2년 개발 기간, 운영 중단, 기능 누락, 도메인 지식 유실 |
| **점진적 교체 (Strangler Fig)** | 운영 무중단, 도메인 지식 보존, 점진적 검증 | 두 시스템 동시 유지 비용, 경계 설계 복잡도 |

**선택**: Strangler Fig Pattern — "안에서부터 천천히 교체하되, 밖에서는 티가 안 나게"

---

## 2. 전략 선택 — Strangler Fig Pattern

### 2.1 Strangler Fig이란

마틴 파울러가 명명한 패턴으로, 열대 무화과나무가 숙주 나무를 감싸며 자라다가 결국 대체하는 것에서 유래했다. 소프트웨어에서는:

```
1. 새 시스템을 기존 시스템 옆에 배치
2. 새 기능은 새 시스템에 구현
3. 기존 기능을 하나씩 새 시스템으로 이전
4. 기존 시스템의 역할이 점점 줄어듦
5. (궁극적으로) 기존 시스템 제거 — 또는 무기한 공존
```

### 2.2 DOMAIN1에서의 적용

우리의 경우, 단계 5("완전 제거")는 **의도적으로 목표로 잡지 않았다**. 이유:

1. **200,000줄의 PHP가 커버하는 도메인 로직**을 전부 옮기는 것은 비현실적
2. 레거시 PHP의 CRUD 화면들은 — 못생겼지만 — **충분히 잘 동작한다**
3. 현대화가 필요한 것은 **새로 만드는 기능**과 **사용자 경험이 중요한 화면**

따라서 목표를 재정의했다:

> "레거시를 죽이지 않는다. 새 기능의 방향을 틀어서, 자연스럽게 모던 스택으로 흘러가게 한다."

### 2.3 3-Area 분리의 탄생

이 전략에서 핵심은 **경계(Boundary)**를 어디에 그을 것인가였다. 결국 세 개의 영역으로 나뉘었다:

```
┌─────────────────────────────────────────────────────────────┐
│                    Single Apache + Node.js                    │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │  Area 1      │  │  Area 2      │  │  Area 3           │  │
│  │  Legacy PHP  │  │  Core MVC Framework MVC  │  │  Next.js 15       │  │
│  │              │  │              │  │                    │  │
│  │  파일 = URL  │  │  라우터 기반  │  │  App Router       │  │
│  │  L_DB_*     │  │  ModenModel   │  │  REST API 소비    │  │
│  │  jQuery UI   │  │  JSON 응답   │  │  React 19 + AntD  │  │
│  │  Bootstrap   │  │  Prepared    │  │  Zustand + RQ     │  │
│  │              │  │  Statement   │  │                    │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬──────────┘  │
│         │                 │                    │              │
│         └─────────────────┴────────────────────┘              │
│                           │                                   │
│                    ┌──────┴──────┐                            │
│                    │  MariaDB    │                            │
│                    │  (MAIN_DB)   │                            │
│                    └─────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 3-Area 하이브리드 아키텍처

### 3.1 영역별 책임 매트릭스

| 속성 | Area 1: Legacy PHP | Area 2: Core MVC Framework MVC | Area 3: Next.js 15 |
|------|-------------------|-------------------|-------------------|
| **위치** | 루트 `/` | `/Core MVC Framework/` + `/rest/exam/` | `/exam-web/` |
| **언어** | PHP 7.4 | PHP 7.4 (구조화) | TypeScript 5.4 |
| **라우팅** | 파일명 = URL | 커스텀 라우터 | App Router |
| **뷰 렌더링** | SSR (PHP → HTML) | JSON 응답 (API) | SSR/CSR (React 19) |
| **DB 접근** | L_DB_* (문자열 결합) | ModenModel (Prepared Statement) | REST API 소비 |
| **인증** | PHP 세션 (`$_SESSION`) | 공유 세션 + 토큰 | 공유 세션 (쿠키) |
| **UI 프레임워크** | jQuery + Bootstrap | 없음 (API 전용) | Ant Design 5 |
| **새 기능 추가** | ❌ 금지 | ⚠️ API 레이어만 | ✅ 권장 |
| **파일 수** | ~400+ | ~100+ | ~200+ |

### 3.2 신규 기능의 이동 경로

```
요구사항 접수
    │
    ├─ "기존 화면 수정" ──→ Area 1 (Legacy)에서 최소한의 수정
    │
    ├─ "기존 데이터에 API 필요" ──→ Area 2 (Core MVC Framework)에 엔드포인트 추가
    │
    └─ "새 화면/기능" ──→ Area 3 (Next.js)에 구현
                              └─ 데이터는 Area 2 API를 통해 접근
```

이 규칙이 자연스럽게 **새 코드를 모던 스택으로 유도**한다. Big Bang 없이도 시스템 전체가 점진적으로 현대화된다.

---

## 4. Area 1: Legacy PHP — 건드리지 않는 기술

### 4.1 핵심 원칙: "Don't Touch What Works"

레거시 영역에서의 작업 규칙:

| 규칙 | 이유 |
|------|------|
| 새 기능을 Legacy에 추가하지 않는다 | 기술 부채 누적 방지 |
| 기존 파일의 구조를 변경하지 않는다 | 사이드 이펙트 최소화 |
| 보안 패치만 Legacy에서 직접 수행한다 | SQL Injection 등 긴급 대응 |
| UI 개선 요청은 Area 3로 리다이렉트한다 | 이중 투자 방지 |

### 4.2 초기화 체인 — 모든 페이지의 시작점

```
사용자가 /path.php 접속
        │
        ▼
index.php (부트스트랩)
├── 보안 체크 (IP, 세션)
├── config.php → config-exam.php → config-exam.php
├── class2.php (테이블 렌더러)
├── exam.php (~3,600줄 헬퍼)
├── exam-permission.php (권한)
├── exam-class.php (인증)
└── user-class.php (사용자 데이터)
        │
        ▼
path.php 실행
├── $db->query() → SQL 실행
├── 결과를 class2 테이블로 렌더링
└── HTML + jQuery 이벤트 바인딩
```

이 체인을 **수정하지 않고** 새 시스템을 옆에 세우는 것이 핵심 전략이다.

### 4.3 L_DB_* — 레거시 DB 접근 패턴

```php
// 레거시 방식 — exam-db.php
L_DB_connect($name, $user, $pass);   // mysqli 연결
$db->query($sql, $_mysqli);          // 문자열 결합 SQL 실행
$db->query_get($sql, $_mysqli, $key); // 연관 배열 반환
```

**문제점**: SQL 문자열에 사용자 입력이 직접 결합되는 구간이 존재한다. 이것이 Area 2(Core MVC Framework)를 만든 직접적 동기 중 하나다.

### 4.4 뷰 렌더링 — class2와 form 패턴

| 파일 패턴 | 역할 | 렌더링 방식 |
|-----------|------|-----------|
| `*-sear--.php` | 검색 폼 | HTML + jQuery autocomplete |
| `*-vv-form.php` | 상세 보기 | HTML + jQuery UI dialog |
| `*-vv-info.php` | 정보 표시 | class2 테이블 렌더러 |
| `*-mod-form.php` | 수정 폼 | HTML form → POST 전송 |

이 파일들은 **여전히 매일 사용된다**. 못생겼지만 안정적이고, 사용자들이 익숙하다.

---

## 5. Area 2: Core MVC Framework MVC — 레거시 위의 안전망

### 5.1 Core MVC Framework가 해결하는 것

Core MVC Framework는 "레거시를 대체하는 것"이 아니라 **"레거시와 모던 사이의 안전한 통로"**다:

| AS-IS (Legacy) | TO-BE (Core MVC Framework) |
|----------------|----------------|
| 파일명 = URL | 라우터 기반 URL 매핑 |
| 문자열 결합 SQL | Prepared Statement (ModenModel) |
| HTML 직접 출력 | JSON 응답 (API 전용) |
| 함수 3,600줄 flat | 네임스페이스 + MVC 분리 + 단순 설치된 라이브러리 정리 (composer 전환) |
| 인증 흩어짐 | BaseController 미들웨어 통합 |

### 5.2 프레임워크 구조

```
Core MVC Framework/
├── Boot/
│   ├── DOMAIN1CommonConfig.php    ← PHP 7.4 초기화, 로깅
│   ├── Core MVC FrameworkAutoload.php           ← 네임스페이스 기반 오토로더
│   └── Core MVC FrameworkMigration.php          ← 레거시 함수 브릿지 (핵심!)
├── Common/
│   └── DOMAIN1CommonClass.php     ← 싱글턴: 사용자/유틸/로깅
├── Config/
│   └── Core MVC FrameworkRouteConfig.php        ← 라우터 설정
├── Controllers/
│   ├── api/                         ← RESTful API 컨트롤러
│   ├── type2/                    ← 도메인별 컨트롤러
│   └── BaseController.php           ← 인증·로깅 기본 클래스
├── Models/
│   ├── ModenModel.php                ← 쿼리 빌더 + Prepared Statement
│   └── MAIN_DBModel.php              ← DB 모델 수퍼클래스
├── Routes/
│   └── v1/                          ← 버전별 라우트 정의
├── Helpers/                         ← 유틸리티 (Date, Format 등)
└── Middlewares/                     ← 요청/응답 필터
```

### 5.3 ModenModel — 안전한 쿼리 빌더

Core MVC Framework의 핵심 가치는 여기에 있다. 레거시의 문자열 결합 SQL을 Prepared Statement로 교체:

```php
// Core MVC Framework 방식 — ModenModel 쿼리 빌더
$result = (new type2Model())
    ->select(['item_1', 'item_2', 'item_3'])
    ->from('-')
    ->where('-', $target)    // 자동 파라미터 바인딩
    ->limit(0, 20)
    ->execute();
```

**설계 판단**: ORM(Eloquent, Doctrine)을 도입하지 않고 **가벼운 쿼리 빌더**를 직접 구축했다. 이유:

- 200+ 테이블의 레거시 스키마에 ORM 매핑은 비현실적
- 복잡한 조인·서브쿼리가 많아 ORM의 추상화가 오히려 방해
- 목표는 "SQL의 안전한 실행"이지 "SQL 작성의 추상화"가 아닌

### 5.4 Core MVC FrameworkMigration — 레거시 브릿지

가장 현실적인 설계 판단 중 하나. Core MVC Framework 컨트롤러에서 레거시 함수를 호출할 수 있는 브릿지:

```php
// Core MVC Framework 컨트롤러 내에서 레거시 함수 사용
require_once ROOT_PATH . "/Core MVC Framework/Boot/Core MVC FrameworkMigration.php";

// 레거시의 L_DB_* 함수를 Core MVC Framework 컨텍스트에서 안전하게 호출
$legacyResult = $db->query_get($sql, $db_link, 'ID');
```

**왜 필요한가**: 레거시에 이미 구현된 복잡한 비즈니스 로직을 처음부터 다시 짜지 않기 위해. "나쁜 코드"라도 **검증된 로직**이면 재사용한다.

### 5.5 라우팅 엔트리포인트

```
POST /rest/---/-/---
          │
          ▼
rest/--/index.php
├── Core MVC FrameworkRouteConfig::setRouteVersion('1.1')
├── Core MVC FrameworkRouteConfig::loadRouter()
└── BaseRouterV1::runScript()
          │
          ▼
-/Controllers/-/-::bulkUpdate()
├── 인증 미들웨어 (BaseController)
├── 입력 검증
├── ModenModel 쿼리 실행
└── JSON 응답 반환
```

---

## 6. Area 3: Next.js 15 — 새 피부

### 6.1 기술 스택

| 카테고리 | 기술 | 선택 이유 |
|---------|------|----------|
| **프레임워크** | Next.js 15.1 (App Router) | SSR/CSR 하이브리드, API 프록시 내장 |
| **언어** | TypeScript 5.4 | 타입 안정성 (레거시 PHP의 타입 혼란 반면교사) |
| **UI** | Ant Design 5.25 | 엔터프라이즈 급 컴포넌트, 한국어 지원 |
| **상태 관리** | Zustand 4.5 | 최소한의 보일러플레이트 |
| **데이터 페칭** | TanStack React Query 5 + SWR | 캐시·재검증·에러 핸들링 자동화 |
| **HTTP 클라이언트** | Axios 1.6 | 인터셉터 기반 인증 토큰 주입 |
| **스타일링** | Tailwind CSS 3.4 + Styled Components | 유틸리티 + 컴포넌트 스타일 혼합 |
| **차트** | Billboard.js 3.12 | 리포트·대시보드용 |
| **i18n** | next-intl 4.8 | 향후 글로벌 확장 대비 |
| **테스트** | Playwright 1.58 | E2E 통합 테스트 |

### 6.2 프로젝트 구조

```
---/src/
├── app/                    # Next.js 15 App Router
│   ├── layout.tsx          # 루트 레이아웃
│   ├── (auth)/             # 인증 관련 라우트 그룹
│   ├── (office)/           # 사무실 포탈 라우트
│   └── (mobile)/           # 모바일 전용 라우트
├── features/               # 도메인별 기능 모듈
│   ├── type1/              # 매출 관리
│   └── type2/           # 자산 관리
├── components/             # 공용 UI 컴포넌트
├── services/               # API 클라이언트 (Axios 래퍼)
├── hooks/                  # 커스텀 React 훅
├── stores/                 # Zustand 스토어
├── types/                  # TypeScript 인터페이스
└── lib/                    # 유틸리티 (date, format 등)
```

### 6.3 API 연동 — Next.js → Core MVC Framework

Next.js가 PHP 백엔드와 소통하는 핵심 메커니즘은 **서버 사이드 리라이트**:

```javascript
// next.config.mjs
async rewrites() {
  const apiUrl = process.env.-;  // http://-:8080
  return [
    // /api-fe/* 요청을 PHP 백엔드로 프록시
    { source: '/---/:path*', destination: `${apiUrl}/:path*` },
    // 레거시 정적 자원 접근
    { source: '/-/:path*', destination: `${publicUrl}/:path*` },
  ];
}
```

**흐름**:
```
브라우저 → Next.js (Node.js) → rewrite → Core MVC Framework (PHP/Apache) → MariaDB
                                           ↑
                                    서버 사이드에서 발생
                                    (CORS 문제 없음)
```

이것이 "같은 도메인에서 두 런타임이 공존하는" 핵심 패턴이다. 사용자는 하나의 도메인만 본다.

---

## 7. 브릿지 패턴 — 세 영역을 잇는 접착제

### 7.1 인증 공유 — 쿠키 기반 세션

세 영역 모두 **동일한 인증 세션**을 공유한다:

```
사용자가 /-/login (Next.js) 접속
    │
    ▼
Next.js 로그인 폼 → POST /-/---/-/login
    │
    ▼
Core MVC Framework LoginController → DB 검증 → 세션 쿠키 발급
    │
    ▼
사용자가 /-.php (Legacy) 접속
    │
    ▼
index.php → 쿠키 확인 → $_SESSION 로드 → 같은 사용자
```

**핵심**: 쿠키 `AUTH_SESSION_ID`가 세 영역을 관통하는 공유 인증 토큰이다. PHP 세션과 Core MVC Framework 인증이 동일한 저장소를 참조하므로, 사용자는 한 번 로그인하면 레거시·모던 화면을 자유롭게 이동할 수 있다.

### 7.2 데이터 흐름 패턴

| 시나리오 | 흐름 |
|---------|------|
| 기존 화면에서 데이터 조회 | Legacy PHP → $db->query → HTML 렌더링 |
| 새 화면에서 같은 데이터 조회 | Next.js → API call → Core MVC Framework Controller → ModenModel → JSON |
| 모던 화면에서 레거시 로직 필요 | Next.js → API → Core MVC Framework → Core MVC FrameworkMigration → 레거시 함수 |

### 7.3 화면 전환 — HTML 링크, Router 아님

현재 레거시 화면과 모던 화면 사이의 전환은 **React Router가 아닌 HTML 링크**로 이루어진다:

```
레거시 메뉴 (jQuery):
  <a href="/url/type1">매출 관리 (새 UI)</a>    ← Next.js 영역으로 이동
  <a href="/page1.php">장비 관리</a>           ← 레거시 영역 유지
```

이것은 "못생긴" 해결책이지만, 두 시스템 사이에 **불필요한 커플링을 만들지 않는** 가장 단순한 방법이다.

---

## 8. 데이터베이스 — 하나의 진실, 세 가지 접근법

### 8.1 공유 DB 아키텍처

세 영역 모두 **동일한 MariaDB 인스턴스**의 동일한 테이블에 접근한다:

```
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Legacy PHP   │   │  Core MVC Framework MVC   │   │  Next.js 15   │
│  -  │   │  ModenModel    │   │  REST API     │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                    │
        │    ┌──────────────┘                    │
        │    │       (Prepared Statement)        │
        │    │                                   │
        ▼    ▼                                   │
  ┌──────────────┐                               │
  │   MariaDB    │ ◄─────────────────────────────┘
  │   (-)   │     (Core MVC Framework API를 통해 간접 접근)
  └──────────────┘
```

### 8.2 세 가지 접근 패턴 비교

| | Area 1: Legacy | Area 2: Core MVC Framework | Area 3: Next.js |
|---|--------------|---------------|----------------|
| **접근 방식** | Direct SQL 실행 | 쿼리 빌더 | REST API 호출 |
| **연결** | L_DB_connect (mysqli) | ModenModel (mysqli prepared) | HTTP → Core MVC Framework |
| **Prepared Statement** | ❌ | ✅ | ✅ (Core MVC Framework가 보장) |
| **트랜잭션** | 수동 | ModenModel 지원 | API 단위 |
| **위험도** | SQL Injection 가능 | 안전 | 안전 |

### 8.3 같은 테이블, 다른 접근

**type2 테이블 접근 예시**:

```php
// Area 1 — Legacy (-.php)
$sql = "SELECT * FROM - WHERE - = '{$serial}'";
$result = $db->query($sql, $db_link);
```

```php
// Area 2 — Core MVC Framework (Legacy_DB_Wrapper.php)
$result = (new type2Model())
    ->select(['item_id', 'item_status'])
    ->from('ASSET_MASTER')
    ->where('item_status', $serial)  // 자동 바인딩
    ->execute();
```

```typescript
// Area 3 — Next.js (services/type2.ts)
const { data } = await api.get('/--/type2', {
  params: { serial }
});
```

**동일한 데이터, 3단계의 안전성**. Area 3은 절대 DB에 직접 접근하지 않는다. 이것이 레거시의 보안 취약점이 새 시스템으로 전파되지 않는 핵심 방어선이다.

---

## 9. 배포 아키텍처 — Docker 기반 공존

### 9.1 컨테이너 구성

```yaml
# docker-compose.yml (간략화)
services:
  # ── 데이터베이스 레이어 ──
  mysql:                        # 메인 MAIN_DB DB
    image: ----:5.7
    ports: ["3306:3306"]
    volumes:
      - /var/www/html/--:/var/lib/mysql

  insadb:                       # HR 연동 DB
    ports: ["3310:3306"]

  # ── 애플리케이션 레이어 ──
  php-apache:                   # Area 1 + Area 2
    # Legacy PHP + Core MVC Framework 모두 Apache에서 서빙
    volumes:
      - /var/www/html/-:/var/www/html

  node:                         # Area 3
    # Next.js 개발 서버 또는 프로덕션 빌드
    volumes:
      - /var/www/html/-/---:/app
```

### 9.2 요청 라우팅

```
인터넷
  │
  ▼
Reverse Proxy (Apache/Nginx)
  │
  ├─ /v2/*, /mobile/* ──→ Node.js (Next.js)
  │                              └─ /api-fe/* ──→ PHP-Apache (rewrite)
  │
  └─ /*.php, /rest/* ──→ PHP-Apache
                          ├─ Area 1 (Legacy PHP)
                          └─ Area 2 (Core MVC Framework REST API)
```

---

## 10. 트레이드오프와 교훈

### 10.1 비용과 보상

| 비용 | 보상 |
|------|------|
| 세 영역을 동시에 이해해야 하는 인지 부하 | 운영 중단 없이 점진적 현대화 |
| 두 런타임(PHP + Node.js) 동시 운영 | 각 영역에 최적의 기술 스택 사용 |
| 인증·세션 공유의 복잡성 | 사용자는 하나의 시스템으로 인식 |
| Core MVC FrameworkMigration 브릿지의 기술 부채 | 10만줄의 레거시 로직을 다시 쓰지 않음 |
| Area 1의 보안 취약점이 여전히 공존 | Area 3은 Area 2 API를 통해 격리 |

### 10.2 핵심 교훈

#### Lesson 1: "완벽한 아키텍처보다 동작하는 공존"

Strangler Fig를 선택한 건 이상적이어서가 아니라 **현실적이어서**다. 10년치 도메인 로직을 담은 코드를 다시 쓰는 건 코드만의 문제가 아니다 — 그 코드에 녹아든 **비즈니스 의사결정**을 다시 발굴해야 한다. 비용 대비 가치가 맞지 않았다.

#### Lesson 2: "경계가 가장 중요하다"

세 영역의 코드 품질보다 **세 영역 사이의 인터페이스**가 더 중요했다. Core MVC Framework API가 잘 설계되니, Area 3(Next.js)는 Area 1(Legacy)의 존재를 몰라도 되었다. **경계가 깔끔하면 내부는 지저분해도 된다**.

#### Lesson 3: "ORM이 아니라 안전한 SQL이 필요했다"

ModenModel 쿼리 빌더를 직접 만든 건 NIH(Not Invented Here) 증후군이 아니라, **200개 테이블의 레거시 스키마에 ORM을 매핑하는 것이 불가능**했기 때문이다. 목표를 정확히 정의하면 도구 선택이 명확해진다: "SQL 추상화"가 아니라 "SQL Injection 방지".

#### Lesson 4: "브릿지는 부끄러운 게 아니다"

`Core MVC FrameworkMigration.php`는 기술적으로 아름답지 않다. 모던 MVC 컨트롤러가 레거시 함수를 `require_once`로 끌어오는 건 교과서에 나올 패턴이 아니다. 하지만 이것 덕분에 **검증된 비즈니스 로직을 다시 쓰는 위험**을 피했다. 현실에서는 "깨끗함"보다 "안전한 전이"가 더 가치 있다.

#### Lesson 5: "새 기능의 방향만 틀면 된다"

시스템 전체를 마이그레이션하겠다는 목표는 버렸다. 대신 **"새로 만드는 모든 것은 모던 스택으로"**라는 단순한 규칙을 세웠다. 시간이 지나면 자연스럽게 모던 영역의 비중이 커지고, 레거시는 점점 읽기 전용(read-mostly)이 된다. 시간이 우리 편이 되는 전략이다.

### 10.3 현재 상태와 방향

```
2016 ──────────── 2022 ──────── 2024 ──── 2025 ──── 2026 ──→

[Legacy PHP만]    [Core MVC Framework 도입]  [Next.js]  [하이브리드]  [점진적]
                              [시작]    [안정화]    [확장]

Area 1 ████████████████████████████████████████░░░░░░░░░  (축소 중)
Area 2 ░░░░░░░░░░░░░░░████████████████████████████████  (안정)
Area 3 ░░░░░░░░░░░░░░░░░░░░░░░░░██████████████████████  (성장 중)
```

궁극적인 목표는 Area 1의 **완전 제거**가 아니라, "Area 1에 새 코드를 추가하지 않는 것"이다. 나머지는 시간이 해결한다.

---

*© 2026 ycy0922. 본 문서는 개인 포트폴리오 목적으로 작성되었으며, 사내 기밀 정보는 포함되어 있지 않습니다.*
