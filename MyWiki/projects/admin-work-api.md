---
title: admin-work-api
category: projects
tags: [lt-framework-v3, backend, api, cqrs]
sources: [/Users/hongtaegi/project/values-play/framework/back-api/v3/admin-module/research.md, /Users/hongtaegi/project/values-play/framework/back-api/v3/admin-module/architecture_diagrams.md]
created: 2026-06-26
updated: 2026-06-26
summary: Spring Boot 3.5.3 기반의 L/T Framework v3 실행형 백엔드 API 서버 구조 및 CQRS 다중 데이터소스 파이프라인 분석서입니다.
---

# admin-work-api (L/T Framework v3 실행형 API 서버)

`admin-work-api`는 **values-play 프레임워크 v3** 에코시스템에서 백엔드 API를 실제로 서비스하고 배포하기 위한 **실행형 Spring Boot 애플리케이션**입니다. 공통 라이브러리인 `admin-module`의 소스를 합성하여 구동하는 Composite Build 구조를 취하고 있습니다.

## 1. 개요 및 스펙
- **위치**: `/Users/hongtaegi/project/values-play/framework/back-api/v3/admin-work-api`
- **런타임 환경**: Java 21, Spring Boot 3.5.3, MyBatis 3.0.3, Spring Security 6.2.x, Redis (Lettuce/Jedis 클라이언트)
- **핵심 아키텍처 패턴**: CQRS (Command-Query Responsibility Segregation)
- **로컬 빌드 메커니즘**:
  - `settings.gradle`의 `includeBuild('../admin-module')` 설정을 활용하여 `admin-module`의 소스 변경 사항이 로컬 기동 시 실시간으로 반영 빌드됨.

---

## 2. CQRS 다중 데이터소스(Multi-Datasource) 설계
쓰기(CUD) 연산과 읽기(Query) 연산의 부하를 격리하고 성능을 최적화하기 위해 물리적으로 구분된 데이터소스를 런타임에 라우팅하여 운용합니다.

```mermaid
flowchart TD
    BFF["REST Controller (MVC)"]
    
    subgraph WritePipeline ["Command Pipeline (쓰기)"]
        CmdService["Command Service"]
        CmdMapper["Command Mapper\n(com.oliveyoung.project.**.mapper.command)"]
        CmdDB[("secondaryCommandDataSource\n(Write/CUD PostgreSQL)")]
    end

    subgraph ReadPipeline ["Query Pipeline (읽기)"]
        QryService["Query Service"]
        QryMapper["Query Mapper\n(com.oliveyoung.project.**.mapper.query)"]
        QryDB[("secondaryQueryDataSource\n(Read-Only PostgreSQL)")]
    end

    BFF -->|CUD 요청 (POST/PUT/DELETE)| CmdService
    BFF -->|단순 조회 (GET)| QryService
    
    CmdService --> CmdMapper
    CmdMapper --> CmdDB
    
    QryService --> QryMapper
    QryMapper --> QryDB
    
    CmdDB -.->|DB Replication 복제| QryDB
```

### 2.1 데이터소스 구성 정보
- **`primaryDataSource`**:
  - `admin-module` 프레임워크가 필요로 하는 기본 관리 정보(공통코드, 메뉴, 사용자 세션, 시스템 변수 등) 테이블에 접근하는 기본 데이터소스.
- **`secondaryCommandDataSource`** (쓰기 전용):
  - 프로퍼티 접두사: `spring.datasource.hikari.secondary.command`
  - MyBatis 스캔 범위: `com.oliveyoung.project.**.mapper.command` 패키지 및 하위
  - Hikari CP 설정을 통해 쓰기 전용 커넥션 풀을 구성하여 업무 CUD 쿼리를 전담 처리.
- **`secondaryQueryDataSource`** (읽기 전용):
  - 프로퍼티 접두사: `spring.datasource.hikari.secondary.query`
  - MyBatis 스캔 범위: `com.oliveyoung.project.**.mapper.query` 패키지 및 하위
  - 읽기 전용으로 동작하며 Command DB로부터 실시간 복제(DB Replication)되는 타겟을 대상으로 함.

### 2.2 기동 시 커넥션 유효성 검증 (`MultiDataSourceConnectionValidator`)
- **역할**: 애플리케이션 시작 직후, 런타임에 사용할 각 데이터소스의 커넥션 상태를 사전 검사하여 안정적인 기동을 확보하는 헬스체크 컴포넌트.
- **구동 시점**: Spring Container 초기화 완료 및 준비가 끝난 시점인 `ApplicationReadyEvent` 발생 시 호출됨.
- **동작 원리**:
  - `db.validate-on-startup` 속성이 `true`인 경우, `primary`, `secondaryCommand`, `secondaryQuery` 세 가지 데이터소스에 차례로 접근하여 단순 쿼리(`SELECT 1`)를 수행해 봄으로써 실제 물리적 네트워크 연결성 및 권한을 사전 검증.
  - 하나의 커넥션이라도 실패할 경우 기동 실패로 간주하여 사전에 장애를 예방.
- **초기화 순서 제어**:
  - 데이터소스들이 Jasypt 복호화 Bean 기동 후 완전하게 수립될 수 있도록 `@DependsOn("jasyptEncryptorDES")` 관계가 인프라 설정 단에 명시되어 있음.

---

## 3. 개발 및 마이그레이션 가이드라인
- **데이터베이스 전환**:
  - 하이브리드 환경(MySQL ➡️ PostgreSQL) 이행 단계에 맞춰, 각 런타임 프로파일(`application-dev.yml`, `application-local.yml` 등)에 정의된 JDBC 드라이버(`org.postgresql.Driver` vs `com.mysql.cj.jdbc.Driver`) 정보와 데이터베이스 주소 설정을 교차 확인해야 함.
- **CQRS 매퍼 작성 규칙**:
  - 패키지 경로에 따라 데이터소스가 컴파일/기동 시에 강제 바인딩되므로, 단순 조회 쿼리를 담당하는 Mapper 인터페이스는 반드시 `com.oliveyoung.project.**.mapper.query` 아래에 위치시키고, 상태를 수정하는 CUD 인터페이스는 `com.oliveyoung.project.**.mapper.command` 아래에 정의해야 함.
  - 비즈니스 서비스 구현 클래스(`ServiceImpl`)는 데이터의 변경 유무에 맞춰 Command Mapper와 Query Mapper를 각각 적절히 주입(Inject)받아 호출하도록 구성.
