# 💻 MVP 시스템 아키텍처 및 기술 명세서 (백엔드 & 프론트엔드)

## 📄 1. 프로젝트 개요 및 목표 재확인
*   **목표:** 내부 지식을 활용한 백엔드/프론트엔드 직무 게이미피케이션 플랫폼의 MVP 구현.
*   **핵심 가치:** 학습을 '재미있는 도전'으로 인식시키고, 사내 노하우를 체계적인 역량 평가 시스템에 녹여낸다.

## ⚙️ 2. 아키텍처 다이어그램 (Conceptual Diagram)
*(실제 다이어그램은 Mermaid 문법이나 이미지로 대체되어야 하지만, 여기서는 논리적 흐름을 정의합니다.)*

**[사용자]** $\rightarrow$ **[모바일 앱/AR UI]** $\leftrightarrow$ **[API Gateway]** $\leftrightarrow$ **[Backend Core Service]**
$\downarrow$
**[내부 지식 DB (Knowledge Base)]** $\leftarrow$ **[Content Management System (CMS)]**

## 💾 3. 백엔드 API 설계 (Backend Focus: Spring Boot/Java)
백엔드는 모든 비즈니스 로직과 데이터 관리를 담당합니다.

| 모듈 | 주요 기능 | 핵심 엔드포인트 예시 | 기술적 고려 사항 |
| :--- | :--- | :--- | :--- |
| **User & Profile** | 사용자 인증, 레벨/경력 정보 관리 | `/api/v1/user/profile`, `/api/v1/user/level` | JWT 기반 인증 필수. 직급(Level)과 경력 연동 로직 구현. |
| **Content API (CMS)** | 미션 콘텐츠 CRUD 및 배포 | `/api/v1/mission/content`, `/api/v1/mission/{id}/quiz` | 내부 문서 지식 ID를 참조하여 퀴즈 문항을 동적으로 생성해야 함. |
| **Assessment Engine** | 평가 로직 실행 (가장 중요) | `/api/v1/assessment/submit`, `/api/v1/assessment/score` | A(퀴즈), B(시나리오 분기), C(실습 결과)를 종합적으로 점수화하는 복잡한 비즈니스 로직이 필요. **트랜잭션 관리 필수.** |
| **Knowledge API** | 내부 문서 지식 검색 및 제공 | `/api/v1/knowledge/search?q=...` | RAG (Retrieval-Augmented Generation) 패턴을 고려하여, 단순 키워드 매칭을 넘어 문맥적 답변을 추출해야 함. |

## 📱 4. 프론트엔드/모바일 앱 설계 (Frontend Focus: React Native / Vue.js)
프론트엔드는 사용자 경험(UX)과 미션의 몰입도를 담당합니다.

| 화면/컴포넌트 | 주요 역할 | 기술적 구현 고려 사항 | 게이미피케이션 요소 |
| :--- | :--- | :--- | :--- |
| **대시보드 (Home)** | 현재 레벨, 다음 목표(Next Milestone), 진행 상황 요약. | 직관적인 애니메이션과 시각화가 필수. | EXP 바, 레벨업 알림, 뱃지 시스템 표시. |
| **미션 플레이 화면** | 미션을 풀고 상호작용하는 메인 공간. | A/B/C 세 가지 모드를 전환할 수 있는 유연한 컴포넌트 구조 필요. | 시간제한 카운터, 실패 시 피드백(힌트), 성공 시 보상 애니메이션. |
| **지식 검색 UI** | 내부 문서 기반 지식을 찾아보는 인터페이스. | 단순 텍스트 박스가 아닌, '질문-답변' 형태의 대화형 UI가 효과적. | AI Chatbot 느낌의 UX 적용 (ChatGPT 스타일). |

## ✨ 5. 기술 스택 요약
*   **Backend:** Spring Boot 3.x (Java) + JPA/MySQL
*   **Frontend:** React Native 또는 Vue.js (모바일 우선 접근)
*   **Knowledge Base:** Vector DB (예: Pinecone, ChromaDB)와 LLM 연동을 위한 RAG 파이프라인 구축 필수.

