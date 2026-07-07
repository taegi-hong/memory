---
title: ValuesPlay MVP
category: projects
tags: [gamification, spring-boot, react-native, rag, backend, frontend, standalone]
sources: [file:///Users/hongtaegi/project/htg/ai/htg-ai/project/ValuesPlay/MVP_Technical_Specification.md, file:///Users/hongtaegi/project/htg/ai/htg-ai/project/ValuesPlay]
created: 2026-06-28 22:03:00
updated: 2026-06-28 23:22:00
---

# ValuesPlay MVP — 백엔드/프론트엔드 직무 게이미피케이션 플랫폼

ValuesPlay MVP는 사내 지식을 활용하여 백엔드/프론트엔드 엔지니어의 역량을 재미있게 평가하고 육성하는 게이미피케이션 플랫폼입니다. 기존 L/T Framework V3 프로젝트와 별개로, 독립적인 검증과 빌드 기동이 가능하도록 **신규 독립형 Spring Boot 프로젝트**로 재구성되었습니다.

## 🔗 핵심 산출물 및 문서
- **독립 프로젝트 루트**: [ValuesPlay](file:///Users/hongtaegi/project/htg/ai/htg-ai/project/ValuesPlay)
- **상세 기술 명세서**: [MVP_Technical_Specification.md](file:///Users/hongtaegi/project/htg/ai/htg-ai/project/ValuesPlay/MVP_Technical_Specification.md)
- **DB DDL 스키마**: [schema.sql](file:///Users/hongtaegi/project/htg/ai/htg-ai/project/ValuesPlay/schema.sql)
- **빌드 스크립트**: [build.gradle](file:///Users/hongtaegi/project/htg/ai/htg-ai/project/ValuesPlay/build.gradle)

## 📌 주요 아키텍처 개요
- **Standalone Spring Boot**: Spring Boot 3.5.3 기반의 단독형 빌드 구성 및 permitAll Security 구성을 제공하여 즉각적인 MVP API 테스트가 가능합니다.
- **MyBatis & DB**: MySQL/PostgreSQL 드라이버 및 MyBatis Mapper 구성을 단순화하여 MVP 기능 검증에 기민하게 대처합니다.
- **RAG & Vector DB**: ChromaDB/Pinecone과 OpenAI `text-embedding-3-small` 임베딩을 이용한 사내 노하우 문서 기반의 동적 퀴즈 및 AI 대화형 검색 챗봇 연동. API Key 누락 시 로컬 가상 Mock RAG 답변 복구 탄력성(Resilience)을 제공합니다.
- **Frontend & App**: 사용자 서비스는 React Native 0.74+를 이용하여 모바일 우선으로 구현하고, 관리자 콘솔은 Vue 3 기반으로 연동 설계되었습니다.
