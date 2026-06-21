# AI 기반 자율형 멀티 에이전트 개발 및 배포 플랫폼 아키텍처

## 1. 개요 (Introduction)

STD(Speech-to-Deployment)는 사용자의 음성(자연어) 요구사항이나 회의록 텍스트를 인식·분석하여, AI 멀티 에이전트가 소스코드 생성부터 테스트, 빌드, 최종 클라우드 인프라 배포까지 전체 소프트웨어 생명주기를 사람의 개입 없이 자율적으로 완수하는 엔드투엔드(End-to-End) 자동화 개발 패러다임입니다.

본 시스템은 일반 사용자를 위한 **전용 웹 인터페이스(Web Client)**, 백오피스 관제 및 관리를 위한 **관리자 시스템(Notion CMS)**, 중앙 프로세스 제어를 위한 **오케스트레이션 엔진(n8n)**을 결합하고, 완충 레이어로 **STD 전용 MySQL 다중 큐(Task & Component Queue)**를 두어 고성능 AI 기반 코드 생성, 자동 테스트 및 빌드 배포를 수행하는 자율형 개발 플랫폼입니다.

기존 동기식 파이프라인에서 발생할 수 있는 LLM 토큰 낭비, 다중 트래픽 밀집 시 병목 현상, 그리고 비개발자 접근성 한계를 해결하기 위해 다음과 같은 혁신적 아키텍처를 채택합니다.

* **업무 영역의 물리적 분리 (Clear Separation of Concerns)**: 사용자는 가볍고 직관적인 웹사이트(Web Frontend)를 통해 요구사항을 접수하며, PM 및 최고 관리자는 최고 수준의 영속성과 관제 편의성을 제공하는 노션(Notion) 보드를 백오피스로 격리 운영합니다.
* **선행 적재 자격 증명 (Notion-First Architecture)**: 모든 다중 채널 인테이크 요청(웹사이트 유입 및 관리자 직접 생성)은 n8n 라우팅 레이어를 거쳐 최종 관리 대시보드인 **Notion DB에 무조건 선행 영속화**되어 감사 추적(Audit Trail)과 가시성을 통합 확보합니다.
* **n8n 발급 고유 식별자(Ticket ID)**: 트리거 시점에 n8n이 즉시 시스템 타임스탬프와 결합된 유일한 Ticket ID를 생성하여 프론트엔드 웹, 노션 CMS, 중앙 MySQL DB를 1:1로 결합하고 레이스 컨디션을 방지합니다.
* **MySQL 이중 완충 큐 (Parent-Child Message Buffer)**: 마스터 테이블(`task_queue`)과 에이전트별 격리 서브 테이블(`component_queue`) 구조를 도입하여 다중 컴포넌트 동시 개발 및 일괄 처리(Bulk Processing) 환경을 보장합니다.
* **에이전트별 1:1 격리 파이프라인**: 각 전문 AI 에이전트(Repository)가 `Ticket ID`와 `Component` 기반으로 독립적인 폴링(Polling) 및 분산 처리를 수행하여 중앙 집중형 연산 병목을 하위 에이전트 레이어에서 완전히 소화합니다.

---

## 2. 전체 처리 흐름 (End-to-End Flow)

### Step 1. 다중 채널 인테이크 및 요구사항 접수 (Intake Channels)
본 플랫폼은 사용자와 시스템 관리자의 접점을 완벽히 분리하여 유기적인 데이터를 수집합니다.
* **Channel A (일반 사용자 웹사이트 진입)**: 사용자는 n8n Webhook 엔드포인트와 실시간으로 연동된 전용 웹사이트 인터페이스를 통해 요구사항 및 회의록(STT 텍스트)을 전송합니다. (`Source Type = WEB`)
* **Channel B (백오피스 관리자 직접 등록)**: 시스템 관리자나 PM은 백오피스 관제를 수행하다가, Notion 티켓 데이터베이스(`AI Ticket 발행`)에 직접 신규 개발 태스크를 생성할 수 있습니다. 초기 등록 시 상태는 `Development Needed` 값을 가집니다. (`Source Type = NOTION`)

### Step 2. n8n Ticket ID 생성 및 Notion 통합 선행 적재 (Locking & Sync)
* **Web 요청 처리**: Web 채널(Channel A)로 요구사항이 인테이크되면, n8n은 즉시 유일무이한 고유 식별자 규칙(`NOVA-{Year}-{Timestamp_Last8}`)에 따라 `Ticket ID`를 생성합니다. 그 후, **Notion API를 즉시 원격 호출하여 'AI Ticket 발행 DB'에 새 페이지를 자율 생성**합니다. 이때 상태는 `Processing`, 소스 타입은 `WEB`으로 기재되어 관리자 보드에 즉시 시각화됩니다.
* **Notion 직접 등록 처리**: `Schedule Trigger` 노드가 1분 주기로 Notion DB를 모니터링하다가 관리자가 직접 등록한 건(`Status = Development Needed`)을 발견하면 배치(`SplitInBatches`) 루프를 가동합니다. n8n은 즉시 `Ticket ID`를 발행하고, 기존 노션 페이지의 상태를 `Processing`으로 변경한 뒤 `Ticket ID`를 역삽입(Write-back)하여 선점형 데이터 잠금(Lock)을 확정합니다.

### Step 3. Task Analyzer Agent (정규식 빈도수 가중치 분류 및 명세화)
* **마크다운 컨텍스트 변환**: 노션 페이지 내부의 블록 데이터를 파싱하여 Heading, Bullet, Code 블록 포맷이 유지된 순수 마크다운(Markdown) 본문 텍스트로 정제 및 결합합니다.
* **가중치 기반 컴포넌트 라우팅 알고리즘**: `Task Analyzer Agent` 내 자바스크립트 소스코드는 타이틀과 마크다운 본문을 결합하여 대소문자 구분 없이 사전에 정의된 도메인 스택 키워드 매칭 빈도수(Frequency Weight)를 정밀하게 스캔합니다.
    * **FE**: `FE`, `FRONTEND`, `VUE`, `VUEJS`, `UI`, `CSS`, `HTML`, `REACT`
    * **BE**: `BE`, `BACKEND`, `SPRING`, `JAVA`, `API`, `SERVER`, `JPA`, `AUTH`
    * **DB**: `DB`, `MYSQL`, `MSSQL`, `DDL`, `DML`, `QUERY`, `SCHEMA`, `INDEX`
    * **AI**: `AI`, `ML`, `MODEL`, `PROMPT`, `VECTOR`, `SCRAPER`, `CRAWL`, `FINBERT`
    * **WINDOWS**: `WINDOWS`, `DESKTOP`, `WPF`, `WINFORM`, `WINUI`, `C#`, `.NET`, `CLIENT`, `XAML`
    * **DEVOPS**: `DEVOPS`, `DOCKER`, `WSL`, `CI`, `CD`, `GITHUB ACTIONS`, `DEPLOY`
* **복잡도(Complexity) 산정**: 분석 결과 매칭된 컴포넌트가 2개 이상일 경우 다중 에이전트 협업이 필요한 `high`로, 단일 컴포넌트일 경우 `medium`으로 작업 복잡도를 동적 할당하며, 매칭 스택이 전무할 경우 기본값 `BE`로 자동 라우팅됩니다.

### Step 4. MySQL Buffer DB 내 부모-자식 관계형 큐 적재 (Relational Enqueue)
* n8n은 파싱된 컨텍스트 데이터를 하위 AI 에이전트로 직접 동기 호출하지 않고, 대규모 분산 처리를 위해 `FinVANT` 데이터베이스 내부의 구조화된 SQL 큐 테이블에 `Upsert(ON DUPLICATE KEY UPDATE)` 형태로 일괄 적재합니다. 싱글 쿼테이션(`'`) 및 백슬래시(`\`) 이스케이프 처리를 통해 데이터 무결성을 유지합니다.
* **`task_queue` (부모 마스터 큐)**: 전체 태스크의 메타데이터, 노션 `page_id`, 원본 요구사항 본문(`content`), 분석된 전체 컴포넌트 배열 데이터(`components` JSON), 전체 Payload(`payload` JSON)를 적재하는 마스터 테이블입니다.
* **`component_queue` (자식 서브 큐)**: 분석 알고리즘에 의해 도출된 복수 개의 컴포넌트 개수만큼 레코드가 쪼개져 분할 삽입됩니다. `task_queue`의 `ticket_id`를 참조하는 외래키 제약조건(`FOREIGN KEY ... ON DELETE CASCADE`)이 걸려 있으며, 개별 전문 에이전트가 처리할 독립 브랜치명(`feature/ticket-{ticket_id}-{component}`)과 진행 상태를 격리 관리합니다.
* **`orchestration_event_log` (이벤트 로그)**: 시스템 투명성 및 예외 모니터링을 위해 티켓 생성 시점부터 `TICKET_ENQUEUED` 혹은 `WEB_TICKET_ENQUEUED` 이벤트를 로깅합니다.

### Step 5. 에이전트별 독립 큐 폴링 및 토큰 최적화 배치 (Bulk Consumer)
* 각 전문 AI 에이전트는 자신과 1:1 매칭되는 전용 레포지토리 환경을 가지고 독립적으로 구동되며, MySQL `component_queue` 테이블에서 자신의 컴포넌트 식별자(`component`)와 `status = 'READY'` 조건을 기반으로 인덱싱 유효 범위 내에서 레코드를 폴링합니다.
* **병목 분산**: 단일 n8n 허브가 가벼운 데이터 파싱과 큐 적재 후 프로세스를 종료하므로 오케스트레이터의 동기적 병목이 소멸됩니다. 에이전트들이 각자 자율적으로 DB 서브 큐에서 독립적으로 데이터를 소비하므로 하위 에이전트 레이어에서 연산 부하를 완벽히 분산 분화합니다.
* **토큰 절약 (Bulk Processing)**: 매 요청마다 개별 LLM API를 호출하는 대신, 큐에 쌓인 티켓들을 배치 단위로 일괄 조회합니다. 기존 소스코드 구조, 아키텍처 가이드, 개발 컨벤션 같은 무거운 공통 컨텍스트를 시스템 프롬프트에 단 1번만 로드한 채 여러 티켓을 일괄 처리(Bulk Processing)함으로써 중복 토큰 소모를 극적으로 절약합니다.

#### [전문 AI 에이전트 분배 정의]
* **Frontend Agent**: React, Vue, Next.js, HTML/CSS, UI 컴포넌트 / 화면 코드, API 연동, 상태관리 로직 생성
* **Backend Agent**: Spring Boot, JPA, MyBatis, REST API, Database / Controller, Service, Repository, Entity 클래스 생성
* **AI Agent**: LLM 연동, RAG, 벡터 검색, 프롬프트 엔지니어링, FinBERT 기반 감성 분석 / AI 서비스 코드 및 임베딩 로직 생성
* **Windows Agent**: WinForm, WPF, Windows Service, Desktop Application / C#, .NET, XAML 기반 클라이언트 프로그램 생성

### Step 6. Claude Code 기반 완성형 코드 생성
* 각 전문 에이전트는 독립된 작업 영역(Workspace) 안에서 Claude Code CLI 환경을 구동하여 소스 코드를 생성합니다. 단순 코드 조각(Snippet) 생성이 아닌, 프로젝트 전체 디렉터리 구조, 기존 코드베이스의 컨벤션, 요구사항 마크다운 명세를 유기적으로 결합하여 상용 환경에 즉시 반영 가능한 완성형 프로덕트 코드를 생성합니다.

### Step 7. GitHub Feature Branch 및 PR 생성 (Ticket Key 매칭)
* 생성된 코드는 운영 브랜치의 안정성 확보를 위해 고유 `Ticket ID`와 `Component` 정보가 결합되어 `component_queue`에 정의된 독립 `branch_name`으로 생성 및 원격 리포지토리에 푸시됩니다.
* n8n과 GitHub API 연동을 통해 자동으로 Pull Request를 발행하며, 생성된 `pr_number`, `pr_url`, `head_sha` 값을 영속성 레이어(`component_queue`)에 실시간 동기화하여 n8n-DB-GitHub 간의 추적성(Traceability)을 1:1 매칭합니다.

### Step 8. GitHub Actions 자동 실행 (CI/CD)
* Pull Request 오픈 이벤트에 의해 해당 브랜치에 격리된 GitHub Actions 워크플로우 파이프라인이 즉시 트리거됩니다.
    * **Lint**: 코드 스타일 및 정적 분석 검증 (ESLint, Checkstyle 등)
    * **Unit Test**: 격리 환경 내 단위 테스트 실행 (JUnit, Jest 등)
    * **Build**: 애플리케이션 빌드 아티팩트 생성 (Gradle, Maven, npm 등)
    * **Deploy**: 도커 컨테이너 이미지 라이징(Dockerizing) 및 대상 클라우드 인프라 환경 배포 자동화

### Step 9. Webhook Callback
* 빌드 및 배포 자동화 프로세스가 최종 완료되면 GitHub Actions 파이프라인은 최종 결과 Payload(작업 ID, 성공 여부, 배포 시간, 테스트 통과 지표, 로그 URL 등)를 담아 n8n Webhook 콜백 엔드포인트를 호출합니다. n8n은 전달받은 페이로드의 브랜치 명세에서 `ticket_id`와 `component`를 리버스 파싱하여 데이터 무결성 검증 단계를 거칩니다.

### Step 10. MySQL 큐 및 Notion 관리자 페이지 통합 업데이트 (Atomic Sync)
* 모든 배포 검증이 완료되면 n8n은 최종 상태 결과를 시스템 전체에 원자적(Atomic)으로 업데이트합니다.
    1. MySQL `component_queue` 내 해당 `ticket_id` 및 `component` 복합키 레코드의 상태를 `SUCCESS` 또는 `FAILED`로 변경하고 `result_payload` 및 에러 로그를 갱신합니다. 모든 서브 태스크가 성공하면 부모 테이블 `task_queue` 역시 상태를 종결 처리합니다.
    2. **통합 백오피스 대시보드 동기화**: Web 사이트 유입 건과 Notion 직접 등록 건 모두 데이터베이스 내에 동일한 구조의 노션 `page_id`와 `ticket_id` 매핑 데이터를 유지하고 있으므로, **단일화된 동기화 파이프라인(`v4_07`)을 타고 노션 관리자 대시보드의 대상을 역추적해 최종 상태를 '완료/실패'로 업데이트**합니다. 동시에 생성된 GitHub PR 링크, 빌드/배포 완료 시간, 단위 테스트 통과 지표(예: 124 Passed) 및 실시간 에러 로그 문맥을 역상속하여 완전 정합성을 달성합니다.

---

## 3. 기대 효과 (Key Benefits)

### 3.1 완벽한 업무 영역 분리를 통한 DX(개발자/사용자 경험) 극대화
* **사용자 친화적 경량 뷰와 관리자용 중량 관제 시스템의 조화**
    * 일반 사용자는 노션의 복잡한 데이터베이스 레이아웃이나 인프라 설정에 노출될 필요 없이, 시스템이 제공하는 직관적이고 가벼운 웹 인터페이스(Web Frontend)를 통해 요구사항을 실시간으로 전달하므로 진입 장벽이 소멸됩니다.
    * 조직의 관리자, 기획자, PM은 최고 수준의 데이터 영속성과 문서화 기능을 제공하는 **Notion 워크스페이스를 '통합 백오피스 관제 대시보드(Back-Office Dashboard)'로 활용**하여, 모든 사용자의 웹 요청 내역과 인프라 배포 상태, 에러 로그를 한눈에 모니터링하고 필요 시 직접 티켓을 추가 발행하는 강력한 통제 권한을 가집니다.

### 3.2 패러다임의 전환 (Coder에서 Reviewer/Architect로)
* 인간 개발자는 단순 반복적이고 기계적인 보일러플레이트(Boilerplate) 코드 작성이나 유기적인 CI/CD 배포 파이프라인 구축 스트레스 등 과도한 문서 및 인프라적 오버헤드(Administrative Overhead)에 리소스를 빼앗기지 않습니다.
* 인간은 가장 고차원적인 비즈니스 핵심 가치 정의와 아키텍처 상위 설계에만 집중하고, AI 에이전트 레이어가 자동 생성해낸 완성형 아티팩트를 시각화된 인터페이스와 GitHub PR 레이어에서 최종 검토 및 검인(Code Review)하는 형태로 업무 패러다임을 혁신하여 **개인 및 조직의 생산성을 300% 이상 극대화**합니다.

### 3.3 전문성 도메인의 장벽 파괴 (Hyper-Product Builder)
* Frontend 개발자가 높은 수준의 Backend 분산 아키텍처를 구축하거나 인프라 파이프라인(DevOps)을 수립해야 할 때, 혹은 그 반대의 상황에서 본 플랫폼의 6대 스택 전문 가중치 에이전트 레이어가 완벽한 백필링(Back-filling) 완충 작용을 수행합니다.
* 특정 기술 스택의 종속성과 진입 장벽에 가로막혀 프로토타이핑을 포기하던 시대에서 벗어나, 기술적 배경에 구애받지 않고 누구나 풀스택 애플리케이션 전반을 자유자재로 통제할 수 있는 **'초인류 프로덕트 빌더(Hyper-Product Builder)'로의 도약을 견인**합니다.

### 3.4 비용 최적화 및 지속 가능한 AI 협업 (Token Economy)
* **지속 가능한 배치(Bulk) 소비 모델 체계 확립**
    * 티켓 요청마다 매번 거대 저장소의 전체 소스코드 트리 문맥과 무거운 코딩 스타일 컨벤션을 LLM API에 중복 전달해야 했던 기존 동기식 아키텍처의 자원 소모 모델을 완전히 파괴합니다.
    * MySQL 메시지 큐 레이어에 영속화된 티켓들을 배치(Bulk Consumers) 그룹 단위로 묶어 일괄 컨텍스트 로딩을 수행함으로써, 공통 프롬프트 주입 횟수를 획기적으로 줄여 **실제 엔터프라이즈 환경 가동 시 컨텍스트 토큰 비용을 최대 70% 이상 절감**하는 현실적이고 경제적인 AI 협업 시스템을 증명합니다.

### 3.5 원천적인 데이터 무결성 및 레이스 컨디션 방지
* **분산 트랜잭션 환경에서의 안정성 보장**
    * n8n 오케스트레이터가 이벤트 트리거 즉시 유일무이한 시간 기반 고유 `Ticket ID`를 생성해내고, 곧바로 인터페이스 레이어(Notion 백오피스 DB)에 선행 페이지 생성 및 상태 락킹(Locking) 메커니즘을 수행합니다.
    * 대규모 해커톤 공개 시연 환경이나 트래픽이 밀집되는 실제 운영 상황 속에서 다수의 사용자가 동시에 무작위로 웹사이트를 통해 요구사항을 쏟아내더라도, 동일 데이터에 대한 **중복 처리 및 동시성 제어 실패(Race Condition) 문제를 데이터베이스 단에서 원천 차단**하여 트랜잭션 무결성을 유지합니다.

### 3.6 비동기 격리를 통한 무한한 확장성 (Loose Coupling Architecture)
* **중앙 집중 구조 탈피 및 무중단 에이전트 스케일아웃**
    * 중앙 오케스트레이터(n8n)는 영속 레이어인 MySQL 데이터베이스에 파싱 데이터를 이스케이프하여 적재하는 가벼운 게이트웨이 및 라우팅 역할만을 전담하므로, 시스템의 전체 결합도가 극도로 낮아지는 느슨한 결합(Loose Coupling) 아키텍처를 실현했습니다.
    * 차후 대규모 언어 모델의 발전이나 새로운 기술 스택 컴포넌트(예: `Mobile(Flutter)`, `QA Automation`)가 추가 확장되더라도 기존 코어 시스템에 아무런 충격을 주지 않으며, 오직 `FinVANT.component_queue` 테이블의 데이터 확장 및 독립적인 개별 하위 에이전트 컨테이너 인스턴스 레이어의 유연한 증설만으로 **플랫폼을 무한히 확장(Scale-out)할 수 있는 진화형 에코시스템을 선사**합니다.

---
