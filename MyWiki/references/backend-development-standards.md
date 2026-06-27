---
title: 1. 암호화 키 관리 방법
category: references
tags: ['lt-framework-v3', 'backend', 'standard', 'guide']
sources: ["/Users/hongtaegi/project/htg/ai/htg-memory/03_Analysis_Architecture/To-Be/3. 개발 표준/2. 개발 표준 가이드/1. backend/개발 표준 가이드.md"]
created: 2026-06-26
updated: 2026-06-26
summary: "1. 암호화 키 관리 방법"
lifecycle: reviewed
base_confidence: 1.0
---

# 1. 암호화 키 관리 방법

Spring Boot 애플리케이션에서 Jasypt 암호화 키를 안전하게 관리하기 위한 두 가지 아키텍처(케이스 1, 케이스 2)의 최종 정리입니다. 

두 방식 모두 런타임 시점에는 AWS 호출 없이 최초 기동 시 1회만 키를 가져와 메모리에 적재(캐싱)하므로 성능 저하가 없는 안전한 구조입니다.

## 1-1. 케이스별 핵심 요약

##### 케이스 1: AWS KMS + 환경별 yml 분리 방식

구조: application-dev.yml, application-stg.yml, application-prd.yml 등 환경별로 설정 파일이 명확히 분리되어 존재합니다.

특징: Git 등의 소스코드 저장소에 Jasypt 암호화 키 자체가 KMS로 암호화된 텍스트 형태로 포함되어 저장됩니다.

동작: 서버 기동 시 해당 환경의 yml을 읽고, 내부에 적힌 암호화된 키를 AWS KMS API를 통해 딱 1번 복호화하여 메모리에 평문으로 올린 후 사용합니다.

##### 케이스 2: AWS Secrets Manager + 단일 yml 변수 바인딩 방식

구조: 환경에 상관없이 오직 단 하나의 application.yml 파일만 존재하며, 내부에는 실제 값 대신 ${ENV_JASYPT_KEY}와 같은 치환자(환경 변수)만 적혀 있습니다.

특징: 소스코드나 Git 저장소에 키 관련 데이터가 전혀 남지 않으며, 실제 Jasypt 키 값은 AWS Cloud(Secrets Manager) 내부에서만 변수 형태로 관리됩니다.

동작: 서버 기동 시 현재 활성화된 프로파일에 맞춰 AWS Secrets Manager에서 해당하는 시크릿 값을 딱 1번 동적으로 조회해 yml 치환자에 바인딩하고 메모리에 적재합니다.

## 1-2. 최종 비교 장표 (한눈에 보기)
| **비교 항목**              | **케이스 1: AWS KMS**                                                        | **케이스 2: AWS Secrets Manager** |
| ---------------------- | ------------------------------------------------------------------------- | ------------------------------ |
| **설정 파일(yml) 개수**      | 환경별로 각각 존재 (dev, stg, prd)                                                | 단 1개로 통합 관리                    |
| **형상 관리 (Git) 보안**     | 암호화된 키 스트링이 Git에 노출됨                                                      | Git에 키 관련 흔적이 전혀 없음            |
| **AWS API 호출 시점**      | **애플리케이션 구동 시 최초 1회만 호출**                                                 | **애플리케이션 구동 시 최초 1회만 호출**      |
| **런타임 성능 및 Latency**   | 없음 (메모리 캐싱 방식)                                                            | 없음 (메모리 캐싱 방식)                 |
| **월 예상 비용 (3개 환경 기준)** | **약 $3.00 / 월** (개별 키 생성 시)<br><br>  <br><br>_(하나의 키 공유 시 $1.00까지 절감 가능)_ | **약 $1.20 / 월** (시크릿 3개 저장 기준) |
| **장점**                 | 전통적인 프로파일 분리 방식이라 직관적임                                                    | 소스코드가 완벽히 깔끔해지며 중앙 집중 관리 가능    |


## 1-3. 아키텍처 선택 가이드 (Recommendation)

이런 경우 '케이스 1 (AWS KMS)'을 추천합니다:

기존 프로젝트가 이미 application-환경.yml 형태로 꼼꼼하게 분리되어 관리되고 있는 경우

암호화된 문자열이라도 Git 리포지토리에 올라가는 것에 거부감이 없는 경우

AWS KMS 내에서 하나의 마스터 키만 공유해 사용하여 비용을 극단적으로($1.00/월) 아끼고 싶은 경우

이런 경우 '케이스 2 (AWS Secrets Manager)'를 추천합니다:

"소스코드 및 Git에는 그 어떤 암호화된 키조차 남기지 않겠다"는 엄격한 보안 거버넌스를 따르는 경우

환경별로 yml 파일이 늘어나는 것이 싫고, 단 하나의 application.yml로 깔끔하게 템플릿화하여 관리하고 싶은 경우

추후 DB 패스워드나 API 토큰 등 다른 민감 정보도 AWS 콘솔에서 한 번에 중앙 관리(및 자동 로테이션)하고 싶은 경우

![1781158779584.png](/project/asung/1781158779584.png)


## 2. Spring Batch (Batch)

Spring Batch 아키텍처에서도 케이스 1(AWS KMS + 환경별 yml 분리 + 기동 시 최초 1회 메모리 적재) 방식은 동일하게 적용할 수 있으며, 배치 애플리케이션의 특성상 몇 가지 중요한 차별점과 장점이 존재합니다.

Spring Batch 환경에서의 동작 구조와 핵심 포인트를 정리해 드립니다.

## 2-1. Spring Batch에서의 케이스 1 동작 프로세스

일반적인 REST API(Spring Boot)는 서버가 24시간 켜져 있는 상태에서 최초 기동 시 1회 로드되지만, Spring Batch는 크론탭(Crontab), Jenkins, AWS ECS Task 등을 통해 주기적으로 가동(Short-lived Task)되는 경우가 많습니다.

배치 잡(Job) 실행 (최초 1회): 스케줄러에 의해 Spring Batch 애플리케이션이 구동되면서 지정된 환경 파일(application-prd.yml)을 읽습니다.

KMS 단회 호출: 배치가 실행되는 시점에 설정 파일 속 Jasypt 암호화 키를 복호화하기 위해 AWS KMS API를 딱 1번 호출합니다.

메모리 적재 및 Job 수행: 복호화된 평문 키를 메모리에 올린 후, Jasypt가 DB 패스워드 등을 복호화하여 커넥션을 맺습니다. 이후 Batch Job(ItemReader $\rightarrow$ ItemProcessor $\rightarrow$ ItemWriter)이 완료될 때까지 이 메모리에 있는 키를 사용합니다.

프로세스 종료: 배치가 끝나면 애플리케이션 프로세스가 종료되면서 메모리에 있던 평문 키도 함께 소멸합니다.

## 2-2. Spring Batch 적용 시 핵심 차별점 및 고려사항

Spring Batch에서 이 구조를 사용할 때 반드시 알아두어야 할 특징들입니다.

메모리 보안성 극대화: REST API 서버처럼 평문 키가 메모리에 계속 남아있는 것이 아니라, 배치 프로세스가 실행되는 동안에만 메모리에 존재하고 작업이 끝나면 완전히 증발하므로 보안상 더욱 안전합니다.

배치 주기와 AWS KMS API 호출 비용 관계:

배치 애플리케이션이 1시간에 1번 혹은 하루에 1번 실행되는 구조라면 한 달 총 KMS 호출 횟수는 수십~수백 번에 불과합니다. 앞서 계산한 대로 AWS KMS의 월 20,000회 무료 티어 범위를 절대 초과하지 않으므로 API 호출 비용은 사실상 $0입니다.

단, 대규모 청크(Chunk) 반복 시 주의 (성능 측면):

간혹 구현 실수로 Jasypt 복호화 메서드가 대량의 데이터를 처리하는 ItemProcessor 내부에서 매번 호출되도록 설정하는 경우가 있습니다. 케이스 1 구조에서는 최초 기동 시 yml을 로드할 때 1번만 KMS를 바라보고 복호화하므로, 천만 건의 데이터를 처리하는 배치 룹(Loop)이 돌더라도 AWS KMS로 인한 병목 현상이나 추가 비용이 전혀 발생하지 않습니다.

## 2-3. Spring Batch 환경을 위한 최종 제언

Spring Batch 인프라를 AWS ECS (Fargate)나 EKS, AWS Batch 같은 컨테이너 기반으로 운영하고 계신다면, 케이스 1 방식이 결합성(Coupling)을 낮추는 데 매우 유리합니다.

애플리케이션 코드와 설정 파일(yml) 내부에 암호화된 상태로 키가 존재하기 때문에, 컨테이너 환경 변수(env) 설정에 별도로 복잡한 시크릿 값을 주입해 줄 필요가 없습니다.

컨테이너를 띄울 때 어떤 프로파일(-Dspring.profiles.active=prd)을 사용할지만 지정해 주면, 애플리케이션이 알아서 지정된 KMS 키 권한을 이용해 안전하게 구동됩니다.

---

# 2. 업무 개발 표준 패키지 구조
## 2-1. API Repository 및 SpringBoot 업무 단위 패키지 구조

| No  | **시스템 구분** | **업무구분** | **Package 명**            | **레포지터리** |
| --- | ---------- | -------- | ------------------------ | --------- |
| 1   | NTMS       | FW공통     | com.asung.ntms.framework |           |
| 2   | NTMS       | 업무공통     | com.asung.ntms.common    |           |
| 3   | NTMS       | 업무1      | com.asung.ntms.business1 |           |
| 4   | NTMS       | 업무2      | com.asung.ntms.business2 |           |
| 5   | BIZ        | FW공통     | com.asung.biz.framework  |           |
| 6   | BIZ        | 공통       | com.asung.biz.common     |           |
| 7   | BIZ        | 업무1      | com.asung.biz.business1  |           |
| 8   | BIZ        | 업무2      | com.asung.biz.business2  |           |


---

# 3. Java 코딩 가이드
## 3-1. 기본 Style

### 3-1-1. Line Length

모든 줄은 120자 이내에서 완성 되어야 한다. 만약 더 필요할 경우, 연산자 앞에서 개행 한 후 다음 줄에서 이어서 작업 한다.

### 3-1-2. Indentation

소스 코드에 대한 가독성 향상을 위해 아래와 같은 들여쓰기를 사용한다. 

- 들여쓰기는 Tab 문자가 아닌 공백(Space) 문자를 사용한다.
- 1 Indent = 4 Spaces ( 4 bytes )

### 3-1-3. 변수 선언

변수의 선언은 프로그램 시작 부분에 두는 것을 기본으로 하며, 1라인에 1개의 변수 만을 선언 한다.

### 3-1-4. Block의 구성

각 블록 문장의 중괄호 열기 부호 “{“ 는 블록이 들어가는 라인의 가장 마지막 문자 위치에, 중괄호 닫기 부호 “}“는 별도의 라인의 첫 번째 문자와 위치를 맞춘다.
소스 코드는 레벨이 낮아 질 때마다 한 단계 들어가는 계단형 코딩을 한다.

```java
public class GrpcSampleAdapter implements GetGrpcSamplePort, PostGrpcSamplePort {
    @GrpcClient("local-grpc-server")
    private SampleGrpc.SampleBlockingStub sampleStub;
   
    @Override
    public List<Sample> selectGrpcSample() {
        List<Sample> grpcSampleList = new ArrayList<>();
        Iterator<SampleResponse> response = this.sampleStub.selectGrpcSample(null);
        
        response.forEachRemaining(sampleResponse -> {
            SampleFieldEntry fieldEntry = sampleResponse.getResult();
            Sample sample = Sample.builder()
                    .seq(fieldEntry.getSeq())
                    .title(fieldEntry.getTitle())
                    .contents(fieldEntry.getContents())
                    .txt(fieldEntry.getTxt())
                    .build();            
            grpcSampleList.add(sample);
        });

        return grpcSampleList;
    }
}
```


## 3-2. 주석(Comment)

### 3-2-1. 구현 주석

구현 주석은 각각의 구현에 대한 추가적인 설명이 필요할 때, 또는 코드를 주석 처리할 때 사용할 수 있다.

#### 3-2-1-1. Block 주석

- block 주석은 파일, 메서드, 자료 구조, 알고리즘에 대한 설명을 제공하는데 사용된다.
- block 주석은 파일의 맨 처음, 각 메서드 앞단에 사용해야 한다.
- 메서드 안에서 사용할 경우는 설명하고자 하는 코드와 같은 간격으로 Indentation을 준다.
- block 주석을 사용할 때는 다른 코드와의 구별을 위해 시작하기전에 공백 행을 하나 띄어준다.

```java
/*
Block Comment를 작성한다.
 */
```


#### 3-2-1-2. Single-Line 주석

- 짧은 주석은 소스 코드와 같은 수준으로 들여쓰기를 한 상태에서 한 줄로 표현 될 수 있다.
- 주석을 한 줄로 쓸 수 없을 경우, block 주석 포맷을 사용한다.

```java
if (condition) {
    /* Single-Line 주석을 작성한다. */
    ...
}
```


#### 3-2-1-3. Trailing 주석

- 아주 짧은 주석이 필요한 경우 주석이 설명하는 코드와 같은 라인에 작성한다.
- 실제 코드와는 구분 가능하도록 충분히 간격을 두어야 한다.

```java
if (a == 2) {
    return TRUE;                       /* a 가 2 인 경우 */
} else {
    return a;                          /* a 가 2가 아닌 경우 */ 
}
```


#### 3-2-1-4. End-Of-Line 주석

- 주석 기호 // 는 한 줄 모두를 주석 처리하거나 한 줄의 일부분을 주석 처리해야 할 경우 사용할 수 있다.
- 어떤 코드의 일부분을 주석 처리 하기 위해서 여러 줄에 연속해서 사용하는 것은 허락되나, 긴 내용을 주석에 포함하기 위해서 연속되는 여러 줄에 이 주석을 사용하는 것은 허락되지 않는다.

##### 3-2-1-4-1. 한 줄 주석

```java
if (foo > 1) {
    // 처리를 실행한다.
    ...
} else {
    return false;                // false를 반환한다.
}
```


##### 3-2-1-4-2. 여러 줄 주석

```java
//if (foo > 1) {
//
//    // 처리를 실행한다.
//    ...
//} else {
//    return false;
//}
```


### 3-2-2. 문서화 주석

문서화 주석은 Java 소프트웨어에 포함되어 있는 javadoc 툴을 사용하면 문서화 주석을 포함하는 HTML파일을 자동으로 생성 할 수 있다. 또한, 소스 코드가 없는 개발자들도 읽고 이해할 수 있도록, 실제 구현된 코드와는 상관이 없는 코드의 명세 사항(specification)을 포함한다. 문서화 주석은 클래스, 인터페이스, 생성자, 메서드, 필드를 설명하고, 각 문서화 주석은 /** xxx */ 안에 작성 하며, 선언 바로 전에 작성한다.

만일, 클래스, 인터페이스, 변수 또는 메서드에 대한 어떤 정보를 제공하고 싶지만 문서화 주석에 어울리지 않는다고 생각되면 클래스 선언 바로 후에 block 주석 또는 Single-Line 주석을 사용한다. (e.g. 클래스 구현에 대한 세부 사항들은 클래스에 대한 문서화 주석이 아닌 클래스 선언문 다음에 block 주석을 사용한다.) Java는 문서화 주석을 주석 이후에 처음 나오는 선언문과 연결 시키기 때문에 메서드 또는 생성자 구현 안에 위치해서는 안된다.

```java
 /**
Example 클래스는 ...
 */
public class Example {
    ...
}
```


#### 3-2-2-1. 파일 주석

파일 주석은 class 파일 이나 interface 등 파일의 설명을 기재하는 주석이다. description 에 해당 파일의 성격을 다른 사람이 이해 할 수 있도록 간단 명료하게 작성한다. 주석을 기재 하는 포맷을 아래 규칙에 맞춰 작성한다.

```java
/**
Test(공지사항) 관련 Class (Sample)

@since   : 2026. 6. 5.
@author  : 홍길동
@see     : com.ezwelesp.bo.common.rest.test.controller
*/
public class TestController extends BaseController {
  ...
}
```


#### 3-2-2-2. Method 주석

Method 주석은 파일 내의 method 기능을 안내 하는 주석이다. method 의 동작 내용을 다른 사람이 이해 할 수 있도록 간단 명료하게 작성한다. override 한 method 의 경우 구현체보단 선언 구문에 작성 하도록 한다. 주석을 기재 하는 포맷을 아래 규칙에 맞춰 작성한다.

```java
/**
공지사항을 등록한다.

@author 홍길동
@since 2021. 8. 6.
@param test
@return
*/
public int createSamples(Test test);
```


#### 3-2-2-3. 기타

- 코드 내부에 특별한 사항을 기술 할 경우 행 단위 주석 //을 사용하지만, 가급적 사용을 줄인다.
- 특별한 로직을 구현했거나 실수하기 쉬운 부분에 대한 강조를 하고자 할 경우가 아니라면 가급적 코드 block 내에서의 주석 사용은 자제한다.
- 주석이 너무 자주 나오면 오히려 코드의 질이 떨어지게 된다. 따라서, 코딩 시에 주석이 필요하다고 느껴지는 부분은 그 부분의 프로그램 코드를 더욱 깔끔하게 만들 방안이 없는지 재검토 해보는 것이 좋다.


## 3-3. 문장(Statements)

### 3-3-1. "if" Statements

모든 if 문에는 항상 중괄호를 사용한다. 이렇게 함으로써 코드를 일관성 있고 확장에 쉽게 대응 할 수 있기 때문이다.

```java
if (value == 0) {                 // RIGHT
    doSomething();
} else {
    doAnother();
}

if (value == 0) doSomething();   // WRONG

if (value == 0)
    doSomething();               // WRONG
```

### 3-3-2. "for" Statements

for 문은 다음과 같이 작성한다.

for문의 초기 값 또는 증감 값 부분에서 콤마 연산자(comma operator)를 사용해야 할 경우, 세 개 이상의 변수를 사용하는 복잡성은 피한다.

```java
for (초기 값; 종료 조건; 증감 값) {
    statements;
}
```

※ 향상된 for Statements

배열 및 컬렉션 객체를 조금 더 쉽게 처리할 목적으로 사용하는 향상된 for문은 counter 변수와 증감식을 사용하지 않는다. 자동으로 배열 및 컬렉션 항목의 수만큼 반복하고 for문을 종료한다.

```java
for (변수타입 변수이름 : 배열이름) {
    statements;
}

/* 기본 for문과 향상된 for문의 작성 예시 */
String[] names = {"A", "B", "C"};

// 기본 for문
for (int i=0; i < names.length; i++) {
    String name = names[i];
    logger.debug("name");
}

// 향상된 for문
for (String name : names) {
    logger.debug("name");
}
```


### 3-3-3. "while" Statements

while 문은 다음과 같이 작성한다.

```java
while (조건문) {
    statements;
}

// 빈 while문은 다음과 같이 작성한다.
while (조건문);
```


### 3-3-4. "do-while" Statements

do-while 문은 다음과 같이 작성한다.

```java
do {
    statements;
} while (조건문);
```


### 3-3-5. "switch" Statements

switch 문은 다음과 같이 작성한다.

모든 switch 문은 반드시 default case를 포함해야 한다. default case내의 break는 중복되지만, 추후 다른 case가 추가 되는 경우 에러를 방지 할 수 있다.

※ case문 수행 후 다음 case문으로 연결 될 때 (break 문이 포함되지 않는 경우) break문에 놓여야 할 자리에 주석을 작성한다. 아래 예에서 /* 계속 수행 (다음 라인) */ 이라고 주석을 붙여 놓은 부분을 참조한다.

```java
switch (조건문) {
    case A:
        statements;
        /* 계속 수행 (다음 라인) */
    case B:
        statements;
        break;
    case C:
        statements;
        break;
    default:
        statements;
        break;
}
```


### 3-3-6. "try-catch" Statements

try-catch 문은 다음과 같이 작성한다.

try-catch 문은 try 블록 수행의 성공/실패 여부와 상관 없이 수행되는 finally를 사용 할 수 있다.

```java
try {
    statements;
} catch (ExceptionClass e) {
    statements;
} finally {
    statements;
}
```


### 3-3-7. "return" Statements

복잡한 표현 식이나 리턴 값을 더 명확하게 표현하기 위한 것이 아니라면, 리턴 값이 있는 return 문에는 괄호를 사용하지 않는다 .

```java
return(true);                        // WRONG
return true;                         // RIGHT
return (s.length() + s.offset);      // RIGHT
```


### 3-3-8. Simple Statements

각 줄에는 최대 하나의 문(Statements)만 사용하도록 한다.

```java
a = b + c; count++;    // WRONG
a = b + c;             // RIGHT
count++;               // RIGHT
```


### 3-3-9. Compound Statements

복합문은 중괄호 “{ statements }”에 포함된 여러 개의 statement를 포함한 statement이다.

- { } 안에 있는 문장은 복합 문장 보다 한번 더 들여쓰기를 해야 한다.
- 여는 중괄호 “{“ 는 복합문을 시작하는 줄의 마지막에 위치해야 한다. 닫는 중괄호 “}“ 는 새로운 줄에 써야 하고, 복합문의 시작과 끝은 동일한 들여쓰기를 해야 한다.

if-else 또는 for와 같이 어떤 statement든 제어 구조의 일부분이라면 중괄호로 모든 statement를 감싼다. 이렇게 함으로써 statement를 쉽게 추가 하고 동시에 중괄호를 닫지 않아 발생 할 수 있는 버그의 소지를 없앨 수 있다.

## 3-4 공백(white space)

### 3-4-1. 빈 줄 (blank line)

빈 줄은 논리적으로 연관된 코드를 묶음으로써 코드의 가독성을 향상 시킨다.

- 메서드 사이
- 메서드에서 로컬 변수와 그 메서드의 첫 번째 문장 사이
- 시작 주석문과 package문, 그리고 package문과 import문 사이

### 3-4-2. 빈 칸 (blank space)

빈 칸은 다음과 같은 경우에 사용 한다.

- 키워드 다음의 괄호는 space에 의해 분리 된다. ( e.g. while (true) )
- 메서드 이름과 괄호 사이에는 빈칸을 두지 않는다. 이것은 키워드와 메서드 호출을 쉽게 구분할 수 있게 해 준다.
- 매개변수 리스트에서 콤마 다음에 빈칸을 둔다.
- 콤마(,)를 제외한 모든 이항 연산은 연산자로부터 한 칸 띄워야 한다. 단항 연산자인 증가 연산자(“++“) 와 감소 연산자(“--“)의 경우 피 연산자와 분리해서는 안된다.

```java
a += c + d;
a = (a + b) / (c + d);
while (d++ == s++) {
    n++;
}

printSize("size if " + foo + "\n");
```

- 한 문장 안에서 표현은 빈칸으로 구별 된다.

```java
for (expr1; expr2; expr3)
```

- 변수 타입을 변환하는 cast 의 경우, cast 후 빈 칸을 둔다.

```java
myMethod((byte) aNum, (Object) x);
myMethod((int) (cp + 5), ((int) (i + 3)) + 1);
```


## 3-5 선언(Declarations)

### 3-5-1. 변수 선언

각 변수는 주석 처리를 위해 각 라인별로 선언되어야 한다.

변수 명은 세로로 들여쓰기를 맞추어 쉽게 읽을 수 있도록 하고, 타입에 따른 특별한 선언 순서는 없다.

특별한 경우 (연산처리와 연관되어 초기화 되는 경우)를 제외하고 로컬 변수는 선언한 곳에서 초기화를 한다.

```java
int level;            // RIGHT
int level, size;      // WRONG
int level, tags[];    // WRONG :: 다른 타입의 변수를 같은 라인에 선언하면 안된다.
```


※ 타입과 변수 사이에는 1칸의 공백을 이용한다. 그러나 탭을 사용하는 것도 허용 된다.
```java
int     level;
int     size;
Object  currentEntry;
```


### 3-5-2. package 및 import

- package 문은 소스의 가장 상단에 작성하고, 그 하단에 import문을 표시한다.
- package 문과 import 문 사이에 한 줄을 띄어준다.
- package 명은 소문자로 정의한다.

```java
package kyobobook.application.adapter.in.controller;

import java.util.ArrayList;
import java.util.List;
```


### 3-5-3. 클래스/인터페이스선언

클래스와 인터페이스 선언 시 다음과 같은 원칙을 따른다.

- 메서드 이름과 그 메서드의 파라미터 리스트의 시작인 괄호”(” 사이에는 빈 공간이 없어야 한다.
- 여는 중괄호 “{“는 클래스/인터페이스/메서드 선언과 동일한 줄의 끝에 작성한다.
- 닫는 중괄호 “}“는 여는 중괄호 “{“ 후에 바로 나와야 하는 null 문인 경우를 제외하고는 동일한 들여쓰기를 하는 새로운 줄에서 사용한다.

```java
class Sample extends Object {
    int var1;
    int var2;
   
    Sample(int i, intj) {
        var1 = i;
        var2 = j;
    }
}
```

## 3-6. 명명 규칙 (Naming Rules)

정연한 명명 규칙(Naming Rules)은 코드의 의도를 명확히 하여 동료와의 협업을 원활하게 하고, 코드 리뷰 시 비즈니스 로직에 더욱 집중할 수 있도록 돕습니다. 또한 AI 코딩 어시스턴트가 프로젝트의 컨벤션을 정확히 이해하고 일관된 코드를 생성하는 기준이 됩니다.

### 3-6-1. Controller 메서드 명명 규칙
클라이언트의 요청을 받는 Controller 계층의 메서드는 다음 접두사(Prefix) 규칙을 따른다.

* **get...** : 데이터 또는 목록 조회 요청
  * **Rule:** 다건(목록) 조회 시 메서드 명의 마지막을 복수형으로한다.
  * **Example:** `getParents`
* **create...** : 데이터 등록(생성) 요청
  * **Example:** `createDept`
* **modify...** : 데이터 수정(변경) 요청
  * **Example:** `modifyDept`
* **remove...** : 데이터 삭제 요청
  * **Example:** `remove` (또는 `removeDept`)
* **check...** : 데이터 유효성 검증 및 중복 체크 요청
  * **Example:** `checkDept`

---

### 3-6-2. Service 메서드 명명 규칙
비즈니스 로직을 수행하는 Service 계층의 메서드는 비즈니스 중심의 용어를 접두사로 사용한다.

* **get...** : 데이터 또는 목록 조회 비즈니스 로직
  * **Rule:** 다건 조회 시 마지막에 복수형 단어나 `List`를 접미사로 붙입니다. 단건 조회 시 보통 조건(예: `ById`)을 명시한다.
  * **Example:** `getDeptById`, `getDeptList`
* **create...** : 데이터 등록 비즈니스 로직
  * **Example:** `createDept`
* **modify...** : 데이터 수정 비즈니스 로직
  * **Example:** `modifyDept`
* **remove...** : 데이터 삭제 비즈니스 로직
  * **Example:** `removeDeptById`
* **check...** : 데이터 조건 및 상태 체크 비즈니스 로직
  * **Example:** `checkDept`

---

### 3-6-3. Mapper 메서드 명명 규칙
데이터베이스(DB)에 직접 접근하는 영속성 계층(Mapper 인터페이스)의 메서드는 표준 CRUD 용어(SQL 매핑 단어)를 접두사로 사용한다.

* **select...** : 데이터 조회 (Query)
  * **Rule:** 다건 조회 시 메서드 명의 마지막에 복수형 단어 또는 `List`를 명시한다.
  * **Example:** `selectDeptById`, `selectDeptList`
* **insert...** : 데이터 삽입 (Create)
  * **Example:** `insertDept`
* **update...** : 데이터 수정 (Update)
  * **Example:** `updateDept`
* **delete...** : 데이터 삭제 (Delete)
  * **Example:** `deleteDeptById`
* **check...** : 데이터 존재 여부 확인 (Existence Check)
  * **Example:** `checkDeptExistUser`

---

### 3-6-4. SQL ID 명명 규칙
MyBatis 등의 XML 매핑 파일이나 쿼리 작성 시 사용되는 SQL ID는 실행하고자 하는 SQL 명령문(Statement)의 종류를 대문자 접두사로 사용한다.

* **SELECT...** : 조회 쿼리 ID
  * **Example:** `SELECT_DEPT_BY_ID`, `SELECT_DEPT_LIST`
* **INSERT...** : 삽입 쿼리 ID
  * **Example:** `INSERT_DEPT`
* **UPDATE...** : 수정 쿼리 ID
  * **Example:** `UPDATE_DEPT`
* **DELETE...** : 삭제 쿼리 ID
  * **Example:** `DELETE_DEPT_BY_ID`

# 4. REST API 가이드
> - REST는 HTTP 통신을 위한 규약의 집합이다.
> - REST의 기본 규약을 사용하여 API 통신을 수행한다.
> - REST의 표준 규약을 모두 지켜서 개발시 HATEOAS와 같은 다양한 제약사항이 존재한다.
>     - 이는 차세대 프로젝트에서 리스크로 작용하므로 **REST의 핵심 개념을 유지하면서 실용적인 접근방법을 적용**한다.
{.is-info}

## 4-1. 프로젝트에서 사용할 REST의 주요 규약

- 리소스 중심 설계 - URL에는 명사만 사용한다
    
- HTTP 메소드 활용. 단, 복잡한 작업의 경우 POST를 활용하여 RPC스타일로 적용
    
- API 버전관리 적용
    
- Swagger를 활용한 API 문서화 적용
    
- 일관된 응답 - JSON 활용
    

아래는 주요 규약에 따른 세부 규칙들을 나열한다.

### 4-1-1. URI 표준

- 모든 Resource는 유일한 URI를 지녀야 한다.
    
- URI에 작성되는 영어를 Restful 원칙에 따라 복수형으로 작성합니다.
    
    - notices, users, files 등
        
- 하위 리소스는 상위 리소스의 일부로 간주되는 리소스이며, 계층 관계로 구분이 필요할 경우에만 사용합니다.
    
    - 예: notices/systems, monitors/jobs 등
        
- URI 경로에는 소문자를 사용한다. 단, parameter는 userId와 같이 대소문자 혼용이 가능하다.
    
- URI 경로에 단어를 조합해야 하는경우 하이픈(-)을 사용한다. (밑줄(_)을 사용하게 되면 주소에 _밑줄 표시와 겹쳐 _안 보이는 경우가 생겨 사용하지 않아야 한다.)
    
    - /api/v1/common/noticesList (X) → /api/v1/common/notices
        
- URI는 등록(생성), 수정, 삭제, 조회에 따라 **GET, POST, PUT, DELETE**로 구분하여야 한다.
    
    - 즉, 상품 목록 조회는 "HTTP GET /XXXX/XXXX/products"와 같이 정의한다.
        
    - **GET /api/v1/products - 조회**
        
    - **POST /api/v1/products - 등록(식별자 없음)**
        
    - **PUT /api/v1/products/{id} - 수정(식별자 존재 - 멱등성)**
        
    - **DELETE /api/v1/products/{id} - 삭제**
        
- Path Variable을 몇 개 까지 사용할지는 개발 파트에서 정의함 (255 자 이내로 제한)
    
    - path Variable은 주로 리소스를 식별하는 데 사용되며, 짧고 간결한 값에 적합하다.
        
    - 너무 많은 경우 URL이 복잡해짐
        
    - 긴 파라미터의 경우(255자 이상), Path Variable보다는 **GET Query String** 또는 **POST 본문(body)**으로 사용한다.
        
    
- 필터링, 정렬, 페이징 등의 작업은 **Query String**을 사용한다.
    
- **Resource와 직접적인 관련이 없는 정보는 Query String으로 처리 한다.**
    
    - **상품 검색의 예는 다음과 같다.** /api/v1/products**?endDate=20241217**
        
- 확장자 사용을 하지 않는다.
    
    - .html, .do, .jsp, .xml, .json …
        
- URL 인코딩이 필요한 문자를 사용하지 않는다.
    

## 4-2. REST API URI 참조 예제

| **Method** | **기능**                    | **잘못된 예제**                                                                                                             | **이유**                                                                                                          | **Method** | **올바른 예제**                                                                                                                                                    | **비고**                                                                                                                                                                                                                        |
| ---------- | ------------------------- | ---------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET        | 주문 목록 조회                  | /mypage/api/v1/getOrders<br><br>/mypage/api/v1/getProducts                                                             | 1. URI에 조회를 의미하는 get 동사가 포함됨<br>    <br><br>2. 단어 조합이 camel case로 대소문자가 섞여 있음                                   | GET        | · /mypage/api/v1/user-order<br><br>· /mypage/api/v1/order-list                                                                                                | 1. 동사가 아닌 명사(복수형)를 사용한다.<br>    <br><br>2. 단어 조합 시 하이픈(-)을 사용하여 전체 소문자로 조합한다.                                                                                                                                                 |
| GET        | 상품의 상세정보 조회               | /product/api/v1/products/getProduct?productId=123                                                                      | 1. 상품 리소스의 상품 아이디가 Parameter로 입력됨. **상품의 PK인 상품 코드는 URL에 입력되어야 함.**<br>    <br><br>2. URI에 조회를 의미하는 get 동사가 포함됨 | GET        | · /product/api/v1/products<br><br>· /product/api/v1/products/{productId}<br><br>· /product/api/v1/products/123                                                | 1. 상품과 같이 명확하게 Entity가 정의되는 경우 <br>    <br><br>2. 상품 ID는 URI의 Path Variable 적용<br>    <br><br>3. get 접두어를 빼고 HTTP GET 메서드 적용                                                                                                  |
| GET        | 상품 검색                     | /product/api/v1/products/getProduct?productId=1                                                                        | 1. 단어 조합이 camel case로 대소문자가 섞여 있음<br>    <br>2. URI에 조회를 의미하는 get 동사가 포함됨                                       | GET        | · /product/api/v1/product/{productId}<br><br>· /product/api/v1/product/1                                                                                      | 1. 키가 한 개 인 경우 GET Method를 사용하여 URI 경로 변수(path variable)를 사용한다.                                                                                                                                                               |
| GET        | 상품 검색                     | /product/api/v1/products/getProduct?category={category}&level={level}                                                  | 1. URI에 getProduct라는 동사가 포함됨<br>    <br><br>2. URI에 대문자가 포함됨                                                    | GET        | /product/api/v1/**category/{category}/level/{level}?sort=ace&filter=active**                                                                                  | 카테고리 형식 내 계층구조 표현에 사용.<br><br>· Path Variable을 몇 개 까지 사용할지는 개발 파트에서 정의함 (255 자 이내로 제한)<br><br>o path Variable은 주로 리소스를 식별하는 데 사용되며, 짧고 간결한 값에 적합하다.<br><br>o 너무 많은 경우 URL이 복잡해짐<br><br>· 필터링, 정렬, 페이징 등의 작업은 Query String을 사용 |
| POST       | 신규 상품 등록                  | /product/api/v1/products/createProduct                                                                                 | 1. URI에  등록을 의미하는 create 동사 포함됨<br>    <br><br>2. URI에 대문자 포함됨                                                  | POST       | /product/api/v1/product                                                                                                                                       | 1. 상품 URI에 등록을 의미하는 HTTP POST 메서드를 사용                                                                                                                                                                                         |
| POST       | 다중 상품 상태 변경(그리드)          | /product/api/v1/products/changeProductStatus                                                                           | 1. URI에 변경을 의미하는 change 동사 포함됨<br>    <br><br>2. URI에 대문자 포함됨                                                   | POST       | /product/api/v1/product/status                                                                                                                                | 1. 변경을 의미하는 change는 리소스 변경을 의미하는 HTTP POST 메소드 처리<br>    <br><br>2. 상태를 변경하므로 URI를 products/status로 계층화 처리                                                                                                                    |
| PUT        | 상품의 상세정보 변경               | /product/api/v1/products/updateProduct?productId=123                                                                   | 1. 상품 아이디가 Parameter로 입력됨<br>    <br><br>2. URI에 변경을 의미하는 update 동사가 포함됨                                        | PUT        | /product/api/v1/product/{productId}                                                                                                                           | 1. 상품 ID는 URI의 Path Variable 적용<br>    <br><br>2. 값의 변경을 의미하는 HTTP PUT 메서드 적용                                                                                                                                                 |
| PUT        | 상품의 상태 변경                 | /product/api/v1/products/changeProductStatus?productId=123                                                             | 1. URI에 변경을 의미하는 change 동사 포함됨<br>    <br><br>2. productId가 Query String으로 되어있음                                 | PUT        | · /product/api/v1/product/{productId}/status<br><br>· /product/api/v1/product/status?productId=123&...                                                        | 1. 키가 한 개 인 경우 URI 경로 변수(path variable)로 사용하고 키가 여러개(복합키)인 경우 Query String(파라미터)로 사용한다.                                                                                                                                       |
| DELETE     | 상품 삭제(단건, 다건 - 키 종류가 한 개) | /product/api/v1/products/deleteProduct?productId=123<br><br>/product/api/v1/products/deleteProduct?productId=1,2,3,4,5 | 1. 상품 아이디가 Parameter로 입력됨<br>    <br><br>2. URI에 삭제를 의미하는 delete 동사가 포함됨                                        | DELETE     | · /product/api/v1/product/{productId}  <br>ex) /product/api/v1/product/1<br><br>· /product/api/v1/product/{productIds}  <br>ex) /product/api/v1/product/1,2,3 | 1. 상품 ID는 URI의 Path Variable 적용<br>    <br><br>2. 삭제를 의미하는 HTTP DELETE 메서드 적용                                                                                                                                                 |

### 4-2-1. HTTP 메서드

| **METHOD**           | **GET**                  | **POST**                     | **PUT**                     | **DELETE**        |
| -------------------- | ------------------------ | ---------------------------- | --------------------------- | ----------------- |
| **행동**               | 리소스 조회                   | 리소스 생성                       | 리소스 수정                      | 리소스 삭제            |
| **설명**               | 명시된 리소스에 대한 표현 정보를 가져온다. | 처리 할 데이터를 특정 리소스에 전송하여 생성한다. | 명시된 기존 리소스의 내용을 수정 또는 갱신한다. | 명시된 특정 리소스를 삭제한다. |
| **멱등성(Idempotency)** | O                        | X                            | O                           | O                 |

#### 4-2-1-1. 무상태(Statelessness)

- API 서버는 클라이언트의 상태를 저장하지 않는다.
    
- 서버 측 에서는 클라이언트의 세션에 대한 상태를 저장하지 않는다. 이렇게 상태에 대한 제한을 두는 것을 Statelessness 라 부른다.
    
- 클라이언트에서 서버로 발생한 request 에는 request 를 이해하는데 필요한 모든 정보가 필요하면 서버에 저장된 context 를 이용할 수 없다.
    
- 클라이언트 측 에서 모든 Application 상태 관련 정보를 저장하고 처리하는 책임을 가지게 된다.
    

#### 4-2-1-2. 멱등성(Idempotency)

- 동일한 요청을 여러 번 수행해도 결과가 동일해야 한다는 성질입니다.
    
- 클라이언트가 같은 요청을 여러 번 보내더라도, 결과가 변하지 않도록 서버가 설계되어야 합니다.
    
    - GET : 서버의 리소스를 조회하는 작업입니다. 여러 번 수행해도 서버의 상태나 리소스에 변화가 없으므로 멱등성을 가집니다.
        
    - PUT: 리소스를 특정 상태로 업데이트합니다.  
        예를 들어, 사용자의 정보를 "이름: JOHN"으로 변경하는 PUT 요청은 여러 번 호출해도 결국 동일한 상태가 유지됩니다.
        
    - DELETE: 리소스를 삭제합니다. 삭제를 요청한 후 다시 같은 삭제 요청을 보내도 이미 리소스가 삭제된 상태이므로, 추가적인 영향이 없습니다. 따라서 **DELETE는 멱등성을 가집니다.**
        
    - POST: 일반적으로 **멱등성을 보장하지 않는 메서드**입니다.  POST 요청은 보통 서버에 새로운 리소스를 생성하는데, 같은 요청을 반복할 경우 여러 개의 리소스가 생성될 수 있습니다. 
        

> REST API는 무상태성이기 때문에 요청을 재전송할 때, 이전의 상태 정보를 기억하지 않습니다.
> 따라서, 클라이언트는 네트워크 오류나 요청 실패 시 안전하게 재요청을 보내야 하며, 멱등성을 통해 요청을 반복해도 데이터의 일관성을 유지할 수 있습니다.
> 예를 들어, PUT 메서드를 통해 상태를 특정 값으로 설정할 때는 멱등성이 보장되므로, 중복 요청이 발생해도 상태가 유지됩니다.
> **무상태성은 요청 간 독립성을 유지하며 서버가 클라이언트의 상태를 기억하지 않는 특성입니다.**
> **멱등성은 동일한 요청을 여러 번 수행해도 서버의 상태가 변경되지 않는 성질입니다.**
{.is-warning}

    

## 4-3. 버저닝(Versioning)

> **REST API Versioning**
> - **http://{도메인}/{업무코드}/api/{v1}/{API명}**
> - **http://{도메인}/{업무코드}/api/{v2}/{API명}**
> - http://{도메인}/{업무코드}/api/{버전}은 고정 주소로 사용
{.is-info}

- API는 변경내용에 대한 관리를 위해 버전 관리를 한다.
- 버전 관리를 통해 기존 서비스에 영향도를 낮추고 변경 내역을 식별 할 수 있도록 한다.    
- API의 버전 관리는 대표적으로 다음과 같은 상황에 업그레이드 한다.
    - API 의 응답 데이터 포맷이 변경된 경우. 
    - Request 또는 Response 의 데이터 유형의 변경
    - 기존 API 의 삭제
- API 의 버전 관리는 URI 관리 방법을 사용하도록 한다.

### 4-3-1. Common Request Header

HTTP 요청은 요청에 대한 기본 정보를 담고 있는 Header 값을 포함한다.

다음은 API 요청에 대해 일반적인 Header field를 설명하는 테이블 이다.

| **HTTP Header field** | **Description**                                                                                                                                  | **Owner** | **Required** |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | --------- | ------------ |
| Content-Length        | 메시지의 길이 (헤더 제외), RFC 2616.<br><br>Ex) Content-Length: 320<br><br>Type: String                                                                    | Client    | YES          |
| Content-Type          | Resource의 Content Type을 의미함. (사용할 경우, 명시적으로 charset을 utf-8로 지정)<br><br>Ex) Content-Type: application/json; **charset=utf-8**<br><br>Type: String | Client    | Optional     |
| Accept                | Client가 사용 가능한 Media Type<br><br>Ex) Accept: application/json<br><br>Type: String                                                                | Client    | Optional     |
| Host                  | 서버의 기본 URL<br><br>Ex) Host: herpold.asunghmp.biz<br><br>Type: String                                                                             | Client    | YES          |
| access-token          | 사용자가 인증되었음을 증명하는 데 사용되는 임시적인 토큰                                                                                                                  | Client    | Optional     |
| refresh-token         | access-token이 만료되었을 때 새로운 access-token을 얻기 위한 장기적인 토큰                                                                                            | Client    | Optional     |
