---
title: admin-module
category: projects
tags: [lt-framework-v3, backend, library, spring-boot]
sources: [/Users/hongtaegi/project/values-play/framework/back-api/v3/admin-module/research.md, /Users/hongtaegi/project/values-play/framework/back-api/v3/admin-module/architecture_diagrams.md]
created: 2026-06-26
updated: 2026-06-26
summary: Spring Boot 3.5.3 기반의 L/T Framework v3 공통 관리자 백엔드 라이브러리(java-library) 구조 분석서입니다.
---

# admin-module (L/T Framework v3 공통 라이브러리)

`admin-module`은 **values-play 프레임워크 v3** 에코시스템의 핵심이 되는 **공통 관리자 백엔드 라이브러리**(`java-library`)입니다. 단독 실행형 애플리케이션이 아니며, `admin-work-api` 등 상위 웹 애플리케이션에 탑재되어 동작합니다.

## 1. 개요 및 스펙
- **위치**: `/Users/hongtaegi/project/values-play/framework/back-api/v3/admin-module`
- **프로젝트 정보**: `com.valuesplay` / `admin-module` / `3.0.1-SNAPSHOT`
- **런타임 환경**: Java 21, Spring Boot 3.5.3, MyBatis 3.0.3, Spring Security 6.2.x, JJWT (0.11.2)
- **주요 책임**: 
  - 플랫폼 관리 기능 (사용자, 부서, 역할, 권한, 메뉴, 다국어 메시지, 약관 등)
  - 인프라 및 보안 (JWT, Redis 캐싱, 데이터 권한 제어, 민감 데이터 암복호화, 공통 필드 주입)
  - 유틸리티 (Excel Import/Export, Captcha, OSHI 서버 모니터링, Quartz 스케줄러)

---

## 2. 주요 패키지 및 아키텍처
최상위 패키지 `com.valuesplay` 아래 3가지 대분류로 구조화되어 있습니다.

### 2.1 `com.valuesplay.common` (프레임워크 공통)
- `constant`: 캐시, 토큰, HTTP 상태 코드 등 시스템 공통 상수 정의.
- `utils`: 문자열(`StringUtils`), 날짜(`DateUtils`), 엑셀(`ExcelUtil`), 권한 컨텍스트(`SecurityUtils`), SQL Injection 검증(`SqlUtil`) 등 전역 유틸리티.
- `exception`: 비즈니스 전용 예외 클래스인 `ServiceException` 정의.

### 2.2 `com.valuesplay.framework` (프레임워크 인프라 설정)
- `config`: DataSource, Redis, Security, Quartz, Email, Captcha 등 핵심 인프라 설정 Bean 적재.
- `security`: JWT 토큰 검증 필터(`JwtAuthenticationTokenFilter`) 및 권한 인가 처리.
- `interceptor`:
  - MyBatis 실행에 개입하는 플러그인 인터셉터군 (`CommonFieldInterceptor` - 감사 필드 자동 바인딩, `SensitiveFieldInterceptor` - 암복호화, `MaskingInterceptor` - 조회 마스킹).
  - Spring MVC 인터셉터인 `RoleCheckInterceptor` (메뉴/URI 기반 권한 통제), `RepeatSubmitInterceptor` (API 중복 요청 제어).
- `aspectj`: `@DataScope` 부서 기반 데이터 권한 제어를 위한 `DataScopeAspect` AOP 처리.

### 2.3 `com.valuesplay.project.db1` (어드민 공통 업무 도메인)
- `system`: 로그인, 사용자(User), 부서(Dept), 직책(Post), 역할(Role), 메뉴(Menu), 공통코드(Code), 시스템설정(Config), 다국어메시지(Message), 약관(Terms) 등.
- `common`: 첨부파일(AtchFile), 다운로드 사유(DownloadResn), 알림(Notification), 즐겨찾기(MnuFav), 캡차(Captcha).
- `monitor`: 로그인 이력(LognHstr), 운영/조작 로그(OperLog), Quartz 스케줄 작업(SysJob), 캐시 통계, 온라인 사용자 목록.
- `tool`: DB 메타데이터 테이블 정보 기반 Velocity 템플릿 코드 생성기(`tool.gen`).

---

## 3. 핵심 아키텍처 흐름 및 규칙

### 3.1 로그인 및 JWT 인증 흐름
1. `POST /login`으로 로그인 요청 전달. 사번(`empId`)과 시스템 ID(`sysId`)를 조합하여 인증 진행.
2. `LoginService.login`에서 캡차 문자열 검증 및 계정 상태 사전 점검.
3. `AuthenticationManager`를 거쳐 `UserDetailsServiceImpl`이 DB에서 사용자를 찾아 패스워드 검증(SHA512 또는 BCrypt).
4. 로그인 성공 시 UUID 기반 세션을 발급하여 Redis 또는 DB(`UserToken` 테이블)에 저장하고, JWT를 클라이언트에 전달.
5. 이후 클라이언트 요청마다 `JwtAuthenticationTokenFilter`가 JWT를 검증하여 `SecurityContextHolder`에 로그인 사용자 정보를 세팅함.

### 3.2 메뉴 기반 API 접근 통제 (`RoleCheckInterceptor`)
- `UrlPatterns.getExcludedUrls()`에 해당하는 예외 URL은 인증 및 권한 검사 없이 즉시 통과.
- 이외 요청에 대해서는 `UserRoleDeptService.getEmpRoleCheck`를 기동해 로그인한 사용자의 역할 목록과 요청한 URI 패턴(AntPathMatcher 적용)이 매핑되어 있는지 DB를 조회해 검사. 매핑이 존재하지 않을 시 403 Forbidden 응답 반환.

### 3.3 데이터 권한 범위 통제 (`DataScopeAspect` & `DataScopeSqlDialect`)
- `@DataScope` 어노테이션이 정의된 서비스 계층 메소드가 실행될 때, 사용자의 역할 등급(`roleDataAtrt`)을 분석하여 데이터 범위 제한용 SQL where 조건절을 동적 생성.
- 생성된 SQL 조각은 쿼리 바인딩 대상인 `BaseEntity.params["dataScope"]`에 주입되고, MyBatis XML 내에서 `${params.dataScope}`로 반영됨.
- **다중 DBMS 호환 구현**:
  - **MySQL**: `FIND_IN_SET` 함수를 적용하여 본인 부서 및 하위 부서의 포함 여부를 검증.
  - **PostgreSQL**: `FIND_IN_SET` 부재에 따라 `LIKE` 패턴 매칭(`(',' || DEPT_CNTN_PTH || ',') LIKE '%,' || empDeptId || ',%'`)을 통해 하위 부서 포함 여부를 검증하도록 JDBC URL 드라이버 타입을 자동 감지하여 쿼리 분기 처리.

### 3.4 예외 및 에러 공통 처리
- **정상 예외 처리**: `GlobalExceptionHandler`가 `@RestControllerAdvice`로 비즈니스 오류(`ServiceException` 등)를 포착하여 `AjaxResult` 형태로 JSON 본문에 에러 코드 및 메시지를 응답.
- **필터/컨테이너 단 예외**: 서블릿 `/error` 위임 시 `CustomErrorController`가 동작하며, 헤더에 `Custom-Client-Type: DMF`가 포함된 경우 항상 HTTP 200 OK 코드를 응답하고 JSON 메시지 내부 code를 통해 에러를 상세 제공함.
