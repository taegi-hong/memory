---
title: 1. 프론트엔드 코딩 규칙 (Coding Convention) 가이드
category: references
tags: ['lt-framework-v3', 'frontend', 'standard', 'guide']
sources: ["/Users/hongtaegi/project/htg/ai/htg-memory/03_Analysis_Architecture/To-Be/3. 개발 표준/2. 개발 표준 가이드/2. frontend/개발 표준 가이드.md"]
created: 2026-06-26
updated: 2026-06-26
summary: "1. 프론트엔드 코딩 규칙 (Coding Convention) 가이드"
lifecycle: reviewed
base_confidence: 1.0
---

# 1. 프론트엔드 코딩 규칙 (Coding Convention) 가이드

본 가이드는 `admin-ui` 프로젝트의 코드 품질을 유지하고, 개발자 간의 일관된 협업을 위해 정의된 프론트엔드 표준 코딩 규칙입니다.

## 1.1 기본 포맷팅 규칙 (Prettier 설정 기준)
프로젝트 루트의 `.prettierrc` 설정에 의거하여 코드를 작성합니다. 모든 파일은 저장 시 자동으로 포맷팅되도록 IDE 설정을 권장합니다.

* **세미콜론 (Semicolons):** **미사용** (`"semi": false`)
  * 문장 끝에 세미콜론(`;`)을 붙이지 않습니다.
* **따옴표 (Quotes):** **홑따옴표 사용** (`"singleQuote": true`)
  * JavaScript/TypeScript 문자열 및 Vue 템플릿 내 속성 값에는 홑따옴표(`'`)를 사용합니다.
* **들여쓰기 (Indentation):** **스페이스 2칸** (`"tabWidth": 2`)
  * 탭(Tab) 대신 공백 2칸을 사용합니다.
* **줄 바꿈 폭 (Print Width):** **최대 120자** (`"printWidth": 120`)
  * 코드 한 줄이 120자를 초과할 경우 줄 바꿈을 적용합니다.
* **마지막 쉼표 (Trailing Commas):** **미사용** (`"trailingComma": "none"`)
  * 객체나 배열 선언의 마지막 원소 뒤에 쉼표(`,`)를 추가하지 않습니다.

---

## 1.2 코드 스타일 & 린트 (ESLint & Airbnb)
프로젝트는 `Airbnb 스타일 가이드` 기반 위에 `Vue 3 필수 규칙`을 결합한 린트 환경을 사용합니다.

* **컴포넌트 이름 관련 린트 예외:** (`'vue/multi-word-component-names': 0`)
  * Vue 공식 규칙과 달리 단일 단어로 구성된 컴포넌트 파일명(예: `index.vue`, `login.vue`) 생성을 허용합니다.
* **Import 시 확장자 생략 규칙:**
  * `.js`, `.ts`, `.jsx`, `.tsx`, `.vue` 파일은 import 시 **확장자를 생략**합니다.
  * *예시:* `import { listType } from '@/api/system/dict/type'`
* **별칭 경로 (Path Alias) 사용:**
  * 프로젝트 루트(`src`) 경로는 매핑 별칭인 `@`를 활용하여 절대 경로 형태로 접근합니다.
  * *예시:* `import AppHeader from '@/components/AppHeader'` (상대 경로 `../../components` 지양)

---

## 1.3 Vue 3 컴포넌트 표준 작성법 (Composition API)
기존 Options API 구조를 지양하고, **Composition API (`defineComponent` 및 `setup()`)** 방식을 표준으로 정립합니다.

### 1.3.1 컴포넌트 기본 구조 (Template -> Script -> Style)
```vue
<template>
  <div class="app-container">
    <!-- UI 마크업 영역 (Element-Plus 적극 활용) -->
    <el-button type="primary" @click="handleQuery">{{ t('LBL_SRCH') }}</el-button>
  </div>
</template>

<script>
import { defineComponent, ref, inject, onMounted } from 'vue'
import { useI18n } from 'vue-i18n'

export default defineComponent({
  name: 'SampleComponent',
  // 프레임워크 자체 dict 기능이 있을 경우 명시
  dicts: ['sys_normal_disable'],

  setup() {
    const { t } = useI18n({ useScope: 'global' })
    const loading = ref(false)

    // 함수 정의 규칙
    const handleQuery = () => {
      loading.value = true
      // 비즈니스 로직 실행
    }

    return {
      t,
      loading,
      handleQuery
    }
  }
})
</script>

<style lang="scss" scoped>
/* Scoped 스타일 정의 */
.app-container {
  margin: 20px;
}
</style>
```

### 1.3.2 상태 관리 및 다국어 표준
* **반응형 상태 정의:** 데이터의 상태는 `ref`를 기본으로 사용합니다. (일관성을 위해 `reactive`보다 `ref` 사용 권장)
* **다국어 처리:** 하드코딩 텍스트는 금지하며, `vue-i18n`의 `t()` 함수를 매핑하여 다국어 코드를 호출합니다.
  * *예시:* `t('LBL_DICT_NM')`, `t('MSG_IPUT')`
* **의존성 주입 (Provide/Inject):** 프레임워크 공통 유틸 함수(예: `addDateRange`, `getStupKy` 등)는 `inject`를 통하여 일관되게 주입받아 사용합니다.

---

## 1.4 네이밍 규칙 (Naming Convention)
* **디렉터리 및 파일명:**
  * `views` 하위 라우트 화면 및 컴포넌트 파일은 가급적 **소문자 시작 camelCase** 또는 **PascalCase**의 통일성을 지키도록 설계합니다. (예: `ssoLogin.vue`, `loginOkta.vue`)
* **컴포넌트 호출 규칙 (Template Tag Name):** **kebab-case** 사용
  * 화면 파일에서 정의되거나 가져온 컴포넌트(`*.vue`)를 `<template>` 영역에서 엘리먼트로 호출하여 사용할 때는 **kebab-case(케밥 케이스)** 명명법을 준수합니다.
  * *예시:* `RightToolbar` 컴포넌트를 가져왔다면 템플릿 영역에서는 `<right-toolbar />` 형태로 사용합니다.
* **변수 및 함수명:** **camelCase** 적용
  * 이벤트 핸들러 명칭은 `handle + 동작` 형태로 작성합니다. (예: `handleQuery`, `handleAdd`, `handleDelete`)
* **상수:** **UPPER_SNAKE_CASE** 적용
  * 값이 변하지 않는 설정형 값은 모두 대문자와 언더바(`_`)로 기술합니다.

---

## 1.5 디렉토리 구조 규칙 (Directory Structure)
`src` 폴더 하위는 역할에 따라 엄격히 구분하여 컴포넌트와 비즈니스 로직을 격리합니다.
* `src/api/`: 백엔드 API 요청 함수들을 관리하는 곳으로 백엔드의 Controller 경로 및 리소스 단위와 매핑되도록 구조화합니다.
* `src/components/`: 여러 화면에서 공용으로 재사용되는 공통 UI 컴포넌트를 위치시킵니다.
* `src/composables/`: 공통 훅 및 상태 재사용을 위한 Vue Composable 함수를 정의합니다.
* `src/store/`: 전역 상태 관리를 수행하는 Pinia/Vuex 스토어 폴더입니다.
* `src/utils/`: HTTP 요청 모듈(`request.js`), 포맷 변환 및 암복호화와 같은 순수 유틸리티 코드를 배치합니다.
* `src/views/`: 라우터와 직접 연결되는 도메인 화면 페이지 컴포넌트들을 위치시킵니다.

---

## 1.6 API 통신 규칙 (Axios & API 모듈화)
* **Axios 공통 인스턴스 사용:** 모든 HTTP 통신은 `@/utils/request`에 정의된 공통 Axios 인스턴스를 import하여 수행합니다.
  * *예시:* `import request from '@/utils/request'`
* **중복 제출 방지 (Repeat Submit):** 중복 요청을 방지해야 하는 등록/수정 API의 경우 request header 설정에 `repeatSubmit: false`를 지원하는 공통 인터셉터 로직을 적극 활용합니다.
* **API 비즈니스 로직 분리:** View 컴포넌트 파일 내에 Axios 통신 코드를 직접 작성(하드코딩)하지 않고, 반드시 `src/api` 폴더에 별도 호출 모듈을 정의하여 불러와 사용해야 합니다.

---

# 2. 공통 FW 기능 목록

| **No** | **기능명**    | **설명**                                                                                  | 참고 자료                                                                                                                       |
| ------ | ---------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| 1      | 모달 팝업(레이어) | 다이얼로그 기능                                                                                | [[frontend-lt-framework-common-features#2-1-모달-팝업레이어|2-1. 모달 팝업(레이어)]]                                                         |
| 2      | 메뉴 검색기능    | 메뉴 검색                                                                                   | [[frontend-lt-framework-common-features#2-2-메뉴-검색기능|2-2. 메뉴 검색기능]]                                                             |
| 3      | 대시보드       | 모니터링 기능                                                                                 | [[frontend-lt-framework-common-features#2-3-대시보드|2-3. 대시보드]]                                                                   |
| 4      | 2FA 인증     | SMS, E-Mail 연동                                                                          | [[frontend-lt-framework-common-features#2-4-2fa-인증|2-4. 2FA 인증]]                                                               |
| 5      | 다국어        | 언어를 선택한 언어로 보여주는 기능                                                                     | [[frontend-lt-framework-common-features#2-5-다국어|2-5. 다국어]]                                                                     |
| 6      | 그리드        | IBSheet8 데이터 그리드의 다양한 기능                                                                | [[frontend-lt-framework-common-features#2-6-그리드|2-6. 그리드]]                                                                     |
| 7      | 차트         | amcharts5 복잡하고 동적인 데이터를 시각화할 수 있도록 지원하는 고성능 HTML5 기반의 JavaScript 차트 라이브러리               | [2-7. Chart](https://wiki.valuesplay.com/ko/project/asung/%EC%84%A4%EA%B3%84/6#:~:text=%C2%B6-,2%2D14.%20Chart,-2%2D14%2D1) |
| 8      | 에디터        | crossEditor 나모소프트(Namo Soft)에서 개발한 기업용 웹 WYSIWYG(What You See Is What You Get) HTML 편집기 | [[frontend-lt-framework-common-features#2-8-editor|2-8. Editor]]                                                                                                             |
