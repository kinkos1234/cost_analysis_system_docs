# CAS (Cost Analysis System) - 설계문서 v1.2

자동차 부품 수주~양산 전 과정의 비용 구조를 추적하고, 단계별 목표 달성 여부를 자동으로 판정하는 내부 시스템이다.

## 왜 만들었나

기존에는 수주가 분석을 엑셀로 했다. 파일마다 계산 로직이 달랐고, 담당자마다 기준이 달랐고, 과거 수주 시점의 원본 데이터는 남아있지 않았다. "3년 전 수주 시점에 재료비가 얼마였는지" 같은 질문에 답할 수가 없었다.

CAS는 이 문제를 풀기 위해 만들었다.

- 모든 비용 데이터를 **차종-품번-단계(Stage)** 기준으로 구조화
- 수주 시점부터 양산까지, 각 설계원가 단계의 비용 구조를 **Snapshot으로 고정**
- 계산 로직을 규칙 엔진에 통일하여, 같은 입력이면 누가 돌려도 같은 결과
- 목표 대비 달성/미달성 판정을 자동화

다만 CAS는 의사결정을 대신 내려주는 시스템이 **아니다**. 판단 근거를 시간 축 위에 고정해서, 나중에 되돌아볼 수 있게 하는 것이 목적이다.

## 핵심 설계 원칙

**Snapshot 불변성** — 한 번 찍힌 Snapshot은 수정도, 삭제도, 재계산도 안 된다. DB 트리거로 UPDATE/DELETE를 물리적으로 차단하고, 온톨로지 레벨에서도 SHACL로 이중 잠금을 걸었다.

**결정론적 계산** — 같은 Snapshot 입력이면 언제 돌려도 같은 결과가 나와야 한다. 확률, 예측, 보정 같은 건 일절 없다. Decimal.js 기반 순수 함수로 구현했다.

**결정 권한 분리** — 시스템에서 나오는 모든 결과는 Fact(사람이 입력한 원본), Derived Value(규칙 엔진이 계산한 수치), Interpretation(사람이 쓴 해석) 셋 중 하나에 속한다. 범주가 섞이면 안 된다. LLM이 끼어들 자리도 없다.

**Human-in-the-loop** — 사용자는 원본 데이터 입력과 코멘트 작성만 가능하다. 계산 결과나 평가 판정을 손으로 고칠 수 없다.

## 기술 스택

```
Frontend    Next.js 15 (App Router, SSR)
Backend     Next.js Route Handlers (Node.js)
Database    PostgreSQL 17
ORM         Prisma 6
Auth        NextAuth.js (Credentials Provider, JWT 8h, RBAC 4-Role)
Calc Engine Decimal.js 기반 순수 함수
Ontology    OWL 2 DL + SHACL + SWRL (Apache Jena Fuseki)
Report      MJML + Nodemailer
```

온톨로지 엔진(OWL 2 DL)은 계산·평가 규칙을 형식 논리로 표현해서, 규칙 엔진의 결과와 SWRL 추론 결과를 이중 검증하는 데 쓴다. Phase 8에서 통합 예정.

## 비용 분석 흐름

```
수주(Bid) → 설계원가 1차 → 2차 → 3차 → 양산 초도가(SOP) → 양산 후 설변(Post-SOP)
```

각 단계 진입 시 Stage Snapshot이 생성된다. Snapshot 안에는 해당 시점의 차종·품번별 비용 구조, 기준 데이터(인건비·환율) 버전, 계산 로직 버전이 전부 들어간다. 이후 기준 데이터가 바뀌어도 과거 Snapshot에는 영향이 없다.

평가는 두 가지 시나리오로 돌아간다:

| 시나리오 | 기준 지표 | 목표값 | 성격 |
|---------|----------|-------|------|
| 1안 | 매출원가율 | 80% 이하 | 가혹 조건 |
| 2안 | 총비용율 | 100% 이하 | 완화 조건 |

각 시나리오에 대해 GREEN / YELLOW / RED 판정이 나오고, 미달성 시 리포트 메일이 발송된다.

## 비용 항목 체계 (Cost Component)

세 가지 타입으로 분류된다:

- **A-Type** (Rate x Revenue) — 노무비, 제조경비, 판관비, 감가상각비, 보증충당금 등. 매출 금액에 비율을 곱해서 산출.
- **B-Type** (Per-unit x Volume) — 재료비, 조립비, 물류비, 포장비, 관세. 대당 단가에 물량을 곱해서 산출.
- **C-Type** (Program-level) — 금형비, 공정투자비, Cash Back, 개발비 회수. 프로그램 단위로 배분.

초기년(INIT) View와 5년 평균(AVG5) View를 동시에 지원하며, 두 View의 평가 결과는 독립적으로 존재한다.

## 문서 구조

| # | 문서명 | 내용 |
|---|-------|------|
| 00 | System Overview | 시스템 목적, 범위, 아키텍처, 설계 원칙 |
| 01 | Decision & Data Immutability Contract | 결정 권한 분리, Snapshot 불변성 계약 |
| 02 | Information Architecture | 화면 구조, 네비게이션, Stage 중심 IA |
| 03 | Data Model & Snapshot Schema | 엔티티 설계, Snapshot 스키마, 관계 구조 |
| 04 | Calculation Engine & Cost Rule Spec | 계산 엔진 규칙, 비용 산식, Cost Component Catalog |
| 05 | Target & Evaluation Logic Spec | 목표 정의, GREEN/YELLOW/RED 판정 로직 |
| 06 | API Specification | REST API 설계, 엔드포인트 정의, 권한 제어 |
| 07 | Dashboard Feature Spec | 리스크 감시 대시보드 설계 |
| 08-A | Domain Invariant Test Spec | 도메인 불변식 정의 및 L0 테스트 규격 |
| 08-B | Calculation Engine Test Spec | 계산 엔진 Unit/Property/Ontology Parity 테스트 |
| 09 | UI Style Guide | UI 원칙, 색상 체계, 숫자 표기, 금지 패턴 |
| 10 | Ontology Specification | OWL 2 DL 스키마, SHACL 제약, SWRL 규칙 |

문서 간 의존 관계가 있다. 00·01이 최상위 원칙이고, 나머지 문서는 이 원칙을 위반할 수 없다.

## 사용자 역할 (RBAC)

| 역할 | 권한 |
|------|------|
| Admin | 시스템 전체 관리, 사용자 관리 |
| Manager | 분석 조회, 코멘트 승인, 목표 설정, 기준 데이터 승인 |
| Analyst | 데이터 입력, Snapshot 생성, 코멘트 작성, 기준 데이터 수정 |
| Viewer | 조회 전용 |

## 테스트 전략

4단계 테스트 구조를 따른다:

- **L0 (Domain Invariant)** — 구현이 어떻게 바뀌든 항상 참이어야 하는 명제. Snapshot 불변성, 계산 정합성, Stage 선형성 등. 하나라도 깨지면 배포 불가.
- **L1 (Unit Test)** — 계산 엔진 순수 함수 단위 테스트.
- **L2 (Property Test)** — 10,000건 이상 퍼즈 테스트로 대수적 성질 검증.
- **L3 (Ontology Parity)** — 계산 엔진 결과와 SWRL 추론 결과가 100% 일치하는지 검증. 불일치 시 BLOCKER.

## 범위 밖인 것들

- ML/AI 기반 비용 예측
- 시나리오 시뮬레이션
- 자동 의사결정·승인
- ERP 데이터 직접 수정
- 외부 고객 공개용 시스템

## 라이선스

이 저장소는 시스템 설계 문서 공개용이며, 실제 구현 코드는 포함되어 있지 않습니다.
