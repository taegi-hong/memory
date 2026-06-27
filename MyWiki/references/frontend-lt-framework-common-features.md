---
title: 2. Frontend
category: references
tags: ['lt-framework-v3', 'frontend', 'library', 'guide']
sources: ["/Users/hongtaegi/project/htg/ai/htg-memory/03_Analysis_Architecture/To-Be/3. 개발 표준/2. 개발 표준 가이드/2. frontend/LT Framework 공통 기능 가이드.md"]
created: 2026-06-26
updated: 2026-06-26
summary: "2. Frontend"
lifecycle: reviewed
base_confidence: 1.0
---

# 2. Frontend
## 2-1. 모달 팝업(레이어)

### 2-1-1. 개요

이 컴포넌트는 Vue.js와 Element Plus 라이브러리의 `el-dialog` 컴포넌트를 사용하여 모달 팝업(레이어 다이얼로그)을 구현하는 예제입니다.

주요 기능 및 제어 파라미터는 다음과 같습니다.

#### 2-1-1-1. 팝업 크기 분류 및 클래스 정의

- **Small (W500px H600px)**: `class="small"` 설정이 필요합니다.
- **Medium (W800px H680px)**: 기본 값이며 별도의 클래스 추가가 필요 없습니다.
- **Large (W1200px H840px)**: `class="large"` 설정이 필요하며, `:modal="false"` 속성이 권장됩니다.
- **Extra Large (W1600px H880px)**: `class="extra"` 설정이 필요하며, `:modal="false"` 속성이 권장됩니다.

#### 2-1-1-2. 특수 목적 팝업 (드래그 및 메뉴 클릭 가능 팝업)
- 위치 이동(드래그)이 가능하며 팝업 밖의 본문(메뉴 등)도 클릭할 수 있는 팝업 설정 시:
  - `modal-class="dialog-clickable"` 속성을 반드시 추가합니다.
  - `draggable`, `overflow`, `:modal="false"`, `:close-on-click-modal="false"` 설정을 병행합니다.
  - 이 경우 `align-center` 속성은 제외합니다.

---

### 2-1-2. 코드 설명

#### 2-1-2-1. 템플릿(HTML) 선언 예제

```html
<!-- 기본 (Medium) 팝업 창 -->
<el-dialog v-model="dialogVisible" title="Popup 800 (Medium)" align-center>
  <p>class를 추가하지 않으면 가로 기본 800으로 설정됩니다.</p>
  <template #footer>
    <div class="dialog-footer">
      <el-button type="primary" size="large" @click="dialogVisible = false">닫기</el-button>
      <el-button type="success" size="large">저장</el-button>
    </div>
  </template>
</el-dialog>

<!-- 위치 이동 및 페이지 클릭이 가능한 팝업창 (드래그 가능) -->
<el-dialog 
  title="메뉴 클릭가능 Popup"
  class="large"
  modal-class="dialog-clickable"
  draggable
  overflow
  :modal="false"
  :close-on-click-modal="false" 
>
  <p>이 팝업은 modal-class="dialog-clickable"과 draggable 속성이 적용되어 화면 내에서 이동 가능합니다.</p>
  <template #footer>
    <div class="dialog-footer">
      <el-button type="primary" size="large" @click="handleClose">닫기</el-button>
      <el-button type="success" size="large" @click="submitForm">저장</el-button>
    </div>
  </template>
</el-dialog>
```

#### 2-1-2-2. 주요 API 및 속성 설명

- `v-model / :model-value`: 팝업의 표시 여부 바인딩 (Boolean)
- `title`: 팝업 헤더 타이틀 문구
- `align-center`: 화면 정가운데에 팝업을 위치시킬 때 설정
- `draggable`: 헤더 드래그를 통해 팝업 위치 변경 지원 여부
- `overflow`: 콘텐츠가 팝업 창 크기를 벗어날 때 내부 스크롤바 제어
- `:modal`: 배경 어두운 딤드(Dim) 처리 여부
- `:close-on-click-modal`: 배경 영역 클릭 시 팝업을 닫을지 여부 설정

---

### 2-1-3. 참고자료

- [Element Plus Dialog Component](https://element-plus.org/en-US/component/dialog.html) - Element Plus 공식 다이얼로그 도큐먼트

## 2-2. 메뉴 검색기능

### 2-2-1. 개요

이 컴포넌트는 사이드바 또는 헤더 영역에서 사용자가 전체 메뉴 리스트를 검색하고, 최근 검색 이력을 기반으로 빠르게 메뉴로 이동할 수 있도록 지원하는 메뉴 자동완성(Autocomplete) 검색 기능입니다.

주요 기능은 다음과 같습니다.

#### 2-2-1-1. 주요 라이브러리 및 API

- `el-autocomplete`: Element Plus의 자동완성 입력 컴포넌트
- `@/api/menu`: 메뉴 데이터 조회(`listMenu`) 및 최근 검색어 등록/조회(`addMenuSrch`, `listMenuSrch`) API
- `vue-router`: 메뉴 선택 시 해당 URL 경로로 SPA 라우팅 이동 처리

#### 2-2-1-2. 최근 검색어(Recent Searches) 및 실시간 필터링
- 입력 필드가 비어있을 때는 사용자의 최근 검색 이력(최대 N개)을 노출합니다.
- 검색어를 입력하면 전역 메뉴 리스트 중 입력 단어가 포함된 메뉴 경로를 실시간 필터링하여 매칭 결과를 표시합니다.
- 검색 결과 목록에서 입력된 검색어와 매칭되는 부분은 `<span class="highlight">` 태그를 통해 하이라이트 처리됩니다.

---

### 2-2-2. 코드 설명

#### 2-2-2-1. 템플릿(HTML) 구현 예제

```html
<div class="search-wrap">
  <el-autocomplete
    ref="autocompleteRef"
    v-if="!collapse"
    class="inp-search"
    :popper-class="!hasSuggestions ? 'inp-search-no-data' : ''"
    v-model="queryString"
    :fetch-suggestions="querySearchAsync"
    placeholder="검색"
    @focus="handleFocus"
    @select="handleSelect"
    @keydown="handleKeydown"
    clearable
  >
    <template #prefix>
      <el-icon :size="24"><Search /></el-icon>
    </template>
    <template #default="{ item }">
      <!-- 매칭되는 단어가 있을 시 하이라이트 처리하여 표시 -->
      <div v-if="hasSuggestions" v-html="highlightQuery(item.value)" />
      <div v-if="!hasSuggestions" class="no-data" v-html="item.value" />
    </template>
  </el-autocomplete>
</div>
```

#### 2-2-2-2. 데이터 필터링 및 비동기 조회 Script (Composition API)

```javascript
import { ref, onBeforeMount } from 'vue';
import router from '@/router';
import { listMenu, listMenuSrch, addMenuSrch } from '@/api/menu';

const queryString = ref('');
const menuList = ref([]);        // 전체 메뉴 목록
const recentSrchList = ref([]);  // 최근 검색 목록
const hasSuggestions = ref(false);

// 자동완성 제안 펑션 (Fetch Suggestions)
const querySearchAsync = async (query, cb) => {
  if (!query) {
    // 입력값이 없으면 최근 검색 목록을 반환
    const results = recentSrchList.value;
    cb(results);
    hasSuggestions.value = results.length > 0;
  } else {
    // 입력값이 있으면 입력단어가 포함된 메뉴 리스트 필터링
    const results = menuList.value.filter(menu => 
      menu.value.toLowerCase().includes(query.toLowerCase())
    );
    cb(results);
    hasSuggestions.value = results.length > 0;
  }
};

// 메뉴 선택 시 이동 처리 및 검색어 등록
const handleSelect = async (item) => {
  if (!item.link) return;
  
  router.push({ path: item.link });

  // 최근 검색어 등록 API 호출
  const data = {
    empId: store.state.user.id,
    mnuId: item.mnuId,
  };
  if (data.mnuId !== null) {
    await addMenuSrch(data);
  }
  queryString.value = '';
};

// 검색 키워드 하이라이트 매칭
const highlightQuery = (value) => {
  if (!value) return '';
  const regex = new RegExp(`(${queryString.value})`, 'gi');
  return value.replace(regex, '<span class="highlight">$1</span>');
};
```

---

### 2-2-3. 참고자료

- [Element Plus Autocomplete Component](https://element-plus.org/en-US/component/autocomplete.html) - 공식 자동완성 안내 가이드

## 2-3. 대시보드

### 2-3-1. 개요

대시보드 모니터링 화면은 사용자가 직관적으로 시스템 상황을 인지할 수 있도록 그리드, 차트, 할 일 목록 등의 위젯 단위를 효과적으로 레이아웃하고 제어하는 포틀릿(Portlet) 아키텍처 기반의 모니터링 기능입니다.

주요 특징 및 제공 컴포넌트는 다음과 같습니다.

#### 2-3-1-1. 포틀릿 컴포넌트 구조
- **기본 포틀릿 (`basicPortlet.vue`)**: 카드 형태로 구성된 독립적인 위젯 단위로, 공통 헤더 요소(새로고침, 상세보기, 툴팁, 단일/멀티 셀렉트 박스 등)를 동적으로 배치할 수 있습니다.
- **접이식 포틀릿 (`collapsiblePortlet.vue`)**: 영역을 접거나 펼칠 수 있는 구조로, 화면 면적을 유연하게 활용해야 하는 복잡한 위젯 구성 시 사용합니다.

#### 2-3-1-2. 제공 공통 상태 정보
- **데이터 유무 예외 처리**: API 로딩 상태(`loading`), 데이터 부재(`isExistData === false`), 통신 에러(`isError === true`) 상태를 위젯 단위에서 내장 처리하여 일관된 예외 UI를 노출합니다.
- **편집 모드**: 사용자가 대시보드 배치를 조작하거나 위젯을 개별 삭제(`emit('delete')`)할 수 있는 UI 스위치를 포함합니다.

---

### 2-3-2. 코드 설명

#### 2-3-2-1. 공통 포틀릿 (`basicPortlet.vue`) 사용법

```html
<template>
  <!-- 대시보드 그리드 영역 내 1x1 또는 1x2 규격의 포틀릿 배치 예제 -->
  <basicPortlet
    title="접속 통계 모니터링"
    className="unit-1x1"
    :loading="chartLoading"
    :isExistData="chartData.length > 0"
    :isError="hasError"
    :headerObj="['refresh', 'navigate', 'tooltip']"
    tooltipText="접속 OS 통계를 표시합니다."
    @refresh="fetchChartData"
    @navigate="goToAnalyticsDetail"
  >
    <template #content>
      <!-- 포틀릿 내부에 들어갈 콘텐츠 정의 (차트, 리스트 등) -->
      <div id="chartdiv" style="height: 100%; width: 100%"></div>
    </template>
  </basicPortlet>
</template>
```

#### 2-3-2-2. 주요 Props 명세

- `className`: 포틀릿 크기를 조정하는 CSS 클래스 (예: `unit-1x1`, `unit-1x2`, `unit-2x2`)
- `title`: 위젯 상단에 표출할 제목
- `loading`: 위젯별 자체 로딩 스피너 활성화 상태 (Boolean)
- `headerObj`: 헤더에 노출할 컴포넌트 목록 배열 (예: `['tooltip', 'select', 'multiple', 'refresh', 'navigate']`)
- `isExistData`: 바인딩할 데이터의 존재 여부 (없을 경우 '데이터 없음' 공통 메시지 렌더링)
- `isError`: 백엔드 통신 에러 여부 (에러 시 '불러오기 실패' 공통 메시지 렌더링)

---

### 2-3-3. 참고자료

- `src/components/Dashboard/basicPortlet.vue` 공통 컴포넌트 파일 코드 및 옵션 구성 참조

## 2-4. 2FA 인증

### 2-4-1. 개요

2FA(Two-Factor Authentication, 2단계 인증) 기능은 일반 ID/PW 로그인 성공 후 사용자의 보안 강화를 위해 등록된 SMS(휴대폰 번호) 또는 E-Mail을 연동하여 인증 코드를 발송하고, 입력된 인증 코드를 추가 검증하는 로그인 2단계 보안 솔루션입니다.

LT Framework에서는 기본적으로 다음과 같은 흐름으로 인증을 구성합니다.

#### 2-4-1-1. 주요 연동 시나리오
- **Cognito / Okta SSO 연동**: Cognito Hosted UI 및 Okta(OAuth2.0/OpenID Connect)의 다요소 인증 설정을 기반으로 하며, 필요시 내부 이메일/SMS 발송 인프라(AWS SNS/SES 등)와 연동됩니다.
- **인증 메커니즘**:
  1. 1차 로그인 성공 후 발급 임시 토큰 상태 확인
  2. 2FA 화면으로 리다이렉트
  3. SMS 또는 E-Mail로 난수 형태의 인증 코드(Verification Code) 발송 요청
  4. 사용자로부터 입력받은 코드를 백엔드 검증 API를 통해 대조 및 최종 Access Token 발급

---

### 2-4-2. 코드 설명

#### 2-4-2-1. 2FA 인증 폼 템플릿(HTML) 예제

```html
<el-form ref="verifyFormRef" :model="verifyForm" :rules="verifyRules" class="login-form">
  <h3 class="title">2단계 추가 인증 (2FA)</h3>
  <p class="login-tip">계정 보호를 위해 등록된 이메일 또는 SMS로 발송된 인증코드를 입력해 주세요.</p>
  
  <el-form-item prop="code">
    <el-input
      v-model="verifyForm.code"
      auto-complete="off"
      placeholder="인증코드 6자리 입력"
      style="width: 63%"
      @keyup.enter="handleVerify"
    >
      <template #prefix>
        <svg-icon icon-class="validCode" class="el-input__icon input-icon" />
      </template>
    </el-input>
    <div class="login-code">
      <!-- 재발송 요청 버튼 -->
      <el-button size="small" type="primary" @click="sendVerificationCode">인증코드 발송</el-button>
    </div>
  </el-form-item>

  <el-form-item style="width: 100%">
    <el-button :loading="loading" type="success" style="width: 100%" @click="handleVerify">
      <span>인증 확인</span>
    </el-button>
  </el-form-item>
</el-form>
```

#### 2-4-2-2. 2FA 요청 및 검증 로직 예제 (Vue 3 script)

```javascript
import { ref, reactive } from 'vue';
import { ElMessage } from 'element-plus';
import { send2FaCode, verify2FaCode } from '@/api/login';
import router from '@/router';

const loading = ref(false);
const verifyForm = reactive({
  username: 'nychoi',
  code: '',
  uuid: 'auth-session-uuid-xxxx'
});

// 인증코드 발송 (이메일/SMS)
const sendVerificationCode = async () => {
  try {
    await send2FaCode(verifyForm.username);
    ElMessage.success('등록된 보안 수단으로 인증코드가 발송되었습니다.');
  } catch (error) {
    ElMessage.error('인증코드 발송에 실패했습니다.');
  }
};

// 인증코드 검증 및 최종 로그인 처리
const handleVerify = async () => {
  if (!verifyForm.code) {
    ElMessage.warning('인증코드를 입력해 주세요.');
    return;
  }
  loading.value = true;
  try {
    const response = await verify2FaCode(verifyForm);
    if (response.code === 200) {
      ElMessage.success('2차 인증이 완료되었습니다.');
      // 최종 JWT 토큰 로컬 쿠키/스토어 저장 후 이동
      store.commit('SET_TOKEN', response.token);
      router.push({ path: '/' });
    }
  } catch (error) {
    ElMessage.error('인증번호가 일치하지 않거나 만료되었습니다.');
  } finally {
    loading.value = false;
  }
};
```

---

### 2-4-3. 참고자료

- Cognito User Pool Multi-Factor Authentication(MFA) 및 Okta 2FA API 개발 문서 참조

## 2-5. 다국어

### 2-5-1. 개요

다국어(Internationalization, i18n) 기능은 애플리케이션의 모든 텍스트 요소를 하드코딩하지 않고, 사용자가 선택한 언어 코드(예: 한국어 'ko', 영어 'en', 중국어 'zh')에 맞춰 실시간으로 해당 언어의 번역 리소스로 매핑하여 렌더링하는 다국어 지원 프레임워크 기능입니다.

LT Framework에서는 기본적으로 다음과 같이 설계 및 적용되어 있습니다.

#### 2-5-1-1. 주요 구성 및 API

- `vue-i18n`: Vue 3에 최적화된 다국어 라이브러리
- `setI18n / getTextByLngeCd`: 백엔드 DB의 다국어 텍스트 테이블로부터 실시간으로 번역 리소스(언어 키-값 매핑 데이터)를 로드하여 다국어 인스턴스를 동적으로 생성 및 초기화
- `useCommonHooks / useI18n`: 컴포넌트 내에서 번역 펑션(`t` 또는 `$t`)을 참조하여 렌더링에 사용

#### 2-5-1-2. 다국어 언어 변경 동적 바인딩
- 로그인 화면 또는 사용자 설정에서 언어 코드를 변경하면, `this.$i18n.locale`을 변경 처리하고 전역 상태(`Vuex/Pinia`) 및 로컬 스토리지에 동기화합니다.
- 화면 컴포넌트 뿐만 아니라 IBSheet8 등 외부 그리드 라이브러리 설정 시에도 해당 번역 데이터가 초기화 인자로 활용됩니다.

---

### 2-5-2. 코드 설명

#### 2-5-2-1. i18n 초기화 및 번역 데이터 동적 바인딩 예제 (`src/i18n/index.js`)

```javascript
import { createI18n } from 'vue-i18n'
import { getTextByLngeCd } from '@/api/login'
import { getLngeCd } from '@/utils/auth'

export const setI18n = async () => {
  let messages = {}
  
  // 백엔드 DB로부터 다국어 리소스 조회
  let result = await getTextByLngeCd();
  
  if (result && result.data) {
    const langTypes = [...new Set(result.data.map((item) => item.lngeCd))]
    langTypes.map((item) => {
      messages[item] = {}
    })
    // 언어코드별로 번역 키-값 구성
    result.data.forEach((item) => {
      messages[item.lngeCd][item.textKy] = item.textVlu
    })
  }

  const i18n = createI18n({
    legacy: false,
    locale: getLngeCd('lngeCd') ? getLngeCd('lngeCd') : 'ko',
    fallbackLocale: 'ko',
    messages,
  })

  return i18n
}
```

#### 2-5-2-2. UI 템플릿(HTML)에서 번역 적용 예제

```html
<!-- 다국어 번역 키를 사용하여 버튼 및 라벨 렌더링 -->
<el-form-item :label="$t('LBL_NTC_SBJT')" prop="sampleNtcTitl">
  <el-input 
    v-model="form.sampleNtcTitl" 
    :placeholder="`${$t('LBL_NTC_SBJT')}${$t('MSG_IPUT')}`" 
  />
</el-form-item>

<el-button type="primary" @click="resetQuery">
  {{ $t('LBL_INIT') }} <!-- 초기화 번역 출력 -->
</el-button>
```

#### 2-5-2-3. Script (Composition API) 번역 적용 예제

```javascript
import { useCommonHooks } from '@/composables/useCommonHooks';
import { ElMessageBox } from 'element-plus';

const { t } = useCommonHooks(); // 혹은 const { t } = useI18n();

const confirmDelete = () => {
  ElMessageBox.confirm(
    t('MSG_SAVE_CONFIRM'), // "저장하시겠습니까?" 와 같은 다국어 메시지 출력
    {
      confirmButtonText: t('LBL_VRF'), // "확인"
      cancelButtonText: t('LBL_CNCL'), // "취소"
      buttonSize: 'small',
      showClose: false,
    }
  );
};
```

---

### 2-5-3. 참고자료

- [Vue I18n 공식 문서](https://vue-i18n.intlify.dev/) - Vue 3 국제화 상세 가이드

## 2-6. 그리드

### 2-6-1. 개요

그리드 기능은 대량의 테이블 데이터를 효율적으로 렌더링하고, 정렬, 필터링, 컬럼 고정, 엑셀 익스포트, 행 추가/삭제 등의 복잡한 상호작용을 처리하기 위한 IBSheet8 데이터 그리드 라이브러리 연동 모듈입니다.

LT Framework에서는 기본적으로 다음과 같이 설계 및 구현되어 있습니다.

#### 2-6-1-1. 주요 연동 아키텍처

- `<IBSheetVue>`: IBSheet8의 Vue 3 래퍼 컴포넌트
- `convertIBSheetSortToApiSort`: IBSheet 정렬 이벤트를 백엔드 API 쿼리 파라미터(`sort`) 포맷으로 변환해 주는 공통 유틸리티
- `initializeDictForIbSheet`: 다국어 및 공통 코드 데이터를 IBSheet 컬럼 사전(Dictionary) 포맷으로 변환하여 매핑하는 공통 훅

---

### 2-6-2. 코드 설명

#### 2-6-2-1. 템플릿(HTML) 선언 예제

```html
<!-- IBSheetVue 컴포넌트에 옵션 및 스타일 바인딩 -->
<IBSheetVue 
  v-if="createSheet" 
  :options="options" 
  :style="customStyle" 
/>
```

#### 2-6-2-2. IBSheet8 설정 및 데이터 로드 예제 (Composition API)

```javascript
import { ref, shallowRef, onMounted } from 'vue';
import IBSheetVue from '@ibsheet/ibsheet-vue'; // 프로젝트 구성에 맞춰 import
import convertIBSheetSortToApiSort from '@/utils/grid/sort';
import { listSampleGrid } from '@/api/sample/grid';

const ibsheet = shallowRef(null);
const createSheet = ref(false);
const total = ref(0);
const sampleGridList = ref([]);

// IBSheet 옵션 및 컬럼 정의
const options = {
  Cfg: {
    SearchMode: 2,
    HeaderMerge: 3,
  },
  Cols: [
    { Header: "공지 ID", Name: "sampleNtcId", Type: "Text", Width: 80, Align: "Center", CanEdit: 0 },
    { Header: "제목", Name: "sampleNtcTitl", Type: "Text", Width: 250, Align: "Left", CanEdit: 1 },
    { Header: "구분", Name: "sampleNtcTy", Type: "Enum", Width: 100, Align: "Center", Enum: "|공지|이벤트", EnumKeys: "|01|02" },
    { Header: "작성일", Name: "crtnDt", Type: "Date", Format: "yyyy-MM-dd", Width: 120, Align: "Center", CanEdit: 0 },
  ],
  Events: {
    onRenderFirstFinish: (evt) => {
      ibsheet.value = evt.sheet;
      getList(); // 초기 데이터 로드
    },
    onSort: (evt) => {
      // IBSheet 정렬 데이터를 백엔드 API용 정렬 스트링 배열로 변환
      queryParams.value.sort = convertIBSheetSortToApiSort(evt.sheet.getSort());
      getList();
    }
  }
};

// 백엔드 조회 데이터 그리드 바인딩
const getList = async () => {
  const response = await listSampleGrid(queryParams.value);
  sampleGridList.value = response.rows;
  total.value = response.total;
  
  // IBSheet8 데이터 로드 API 호출
  if (ibsheet.value) {
    ibsheet.value.loadSearchData({
      data: sampleGridList.value,
    });
  }
};

onMounted(() => {
  createSheet.value = true;
});
```

---

### 2-6-3. 참고자료

- `src/views/sample/crud/ibsheet.vue` 샘플 구현 코드 참조
- IBSheet8 공식 개발 가이드 및 API 레퍼런스 문서 참조

## 2-7. Chart

### 2-7-1. 개요

이 컴포넌트는 Vue.js와 amCharts 라이브러리를 사용하여 바 차트와 파이 차트를 생성하고 데이터를 시각화하는 예제입니다.

- Bar Chart 는 로그인 기록 테이블에서 접속한 OS 중 윈도우10과 맥OS 의 접속 횟수를 추출햐여 일자별로 표시하는 차트입니다
- Pie Chartm는 로그인 기록 테이블에서 접속한 OS 중 윈도우10과 맥OS 의 접속 횟수를 추출하여 퍼센티지로 표시하는 차트입니다.

주요 기능은 다음과 같습니다.

#### 2-7-1-1. 주요 라이브러리

- @amcharts/amcharts5: amCharts 5 라이브러리
- @amcharts/amcharts5/xy: XY 차트 모듈
- @amcharts/amcharts5/percent: 퍼센트 차트 모듈
- @amcharts/amcharts5/themes/Animated: 애니메이션 테마
- @/api/sample/chart: 차트 데이터를 가져오는 API 모듈

#### 2-7-1-2. 주요 데이터 및 변수

- dateRange: 날짜 범위
- chartdiv1: 바 차트를 렌더링할 div 요소 참조
- data1: 바 차트 데이터
- chartdiv2: 파이 차트를 렌더링할 div 요소 참조
- data2: 파이 차트 데이터

### 2-7-1-3. 주요 함수

- getBarChartData: 바 차트 데이터를 가져오는 함수
- getPieChartData: 파이 차트 데이터를 가져오는 함수

### 2-7-1-4. 차트 초기화 및 데이터 바인딩

- onMounted: 컴포넌트가 마운트될 때 차트를 초기화하고 데이터를 설정합니다.
- onBeforeUnmount: 컴포넌트가 언마운트될 때 차트 리소스를 해제커합니다.
- watch: 데이터 변경 시 차트 데이터를 업데이트합니다

### 2-7-2. 코드 설명

#### 2-7-2-1. 라이브러리 임포트

```javascript
import * as am5 from "@amcharts/amcharts5";
import * as am5xy from "@amcharts/amcharts5/xy";
import * as am5percent from "@amcharts/amcharts5/percent";
import am5themes_Animated from "@amcharts/amcharts5/themes/Animated";
```

- amChart에 해당하는 컴포넌트 입니다. 필요에 따라 추가 및 삭제될 수 있습니다.

#### 2-7-2-2. Chart 관련 변수 선언

```javascript
// 바 차트 데이터
const chartdiv1 = ref(null);
const data1 = ref([]);
let root1;
let chart1, xAxis, barseries1, barseries2;

// 파이 차트 데이터
const chartdiv2 = ref(null);
const data2 = ref([]);
let root2;
let chart2, pieseries;
```

- 2-7-1-2 를 참조해주세요.

#### 2-7-2-3 Chart 생성 및 데이터 설정

```javascript
/**************** 바 차트 시작 *************************************/
root1 = am5.Root.new(chartdiv1.value);
root1.setThemes([am5themes_Animated.new(root1)]);

// 바 차트 생성
chart1 = root1.container.children.push(
  am5xy.XYChart.new(root1, {
    panY: false,
    layout: root1.verticalLayout,
  })
);

// 축 생성
let yAxis = chart1.yAxes.push(
  am5xy.ValueAxis.new(root1, { renderer: am5xy.AxisRendererY.new(root1, {}) })
);

xAxis = chart1.xAxes.push(
  am5xy.CategoryAxis.new(root1, {
    renderer: am5xy.AxisRendererX.new(root1, {}),
    categoryField: "lognDt",
  })
);

// 시리즈 생성
barseries1 = chart1.series.push(
  am5xy.ColumnSeries.new(root1, {
    name: "Windows 10",
    xAxis: xAxis,
    yAxis: yAxis,
    valueYField: "windows10",
    categoryXField: "lognDt",
  })
);

barseries2 = chart1.series.push(
  am5xy.ColumnSeries.new(root1, {
    name: "Mac OS X",
    xAxis: xAxis,
    yAxis: yAxis,
    valueYField: "macOsX",
    categoryXField: "lognDt",
  })
);

// 범례 생성
let legend1 = chart1.children.push(am5.Legend.new(root1, {}));
legend1.data.setAll(chart1.series.values);

chart1.set("cursor", am5xy.XYCursor.new(root1, {}));

// 데이터 가져오기
await getBarChartData();

// 데이터 설정
xAxis.data.setAll(data1.value);
barseries1.data.setAll(data1.value);
barseries2.data.setAll(data1.value);
/**************** 바 차트 끝 *************************************/
```

- 편의상 Bar Chart 관련해서만 서술하였습니다. Pie 차트는 소스를 확인해 주세요.
- Vue3의 onmounted 안에서 작성되었습니다. 데이터 가져오기는 상황에 따라 변경될 수 있습니다

#### 2-7-2-4 데이터 가져오기

```javascript
/** 바 차트 데이터 조회 */
const getBarChartData = async () => {
  try {
    loading.value = true;

    const response = await barChartlist(addDateRange({}, dateRange.value));
    data1.value = response.rows;
  } catch (error) {
    console.error("getBarChartData", error);
  } finally {
    loading.value = false;
  }
};
```

- backend로 부터 json데이터를 취득하여 data1변수에 설정합니다.
- backend 호출은 src/api/sample/chart.js 에서 선언하고 있습니다.

#### 2-7-2-5 차트 리소스 해제

```javascript
onBeforeUnmount(() => {
  if (root1) {
    root1.dispose();
  }
  if (root2) {
    root2.dispose();
  }
});
```

- Vue의 onBeforemount 에서 해제합니다.

#### 2-7-2-6 데이터 변경감지

```javascript
watch(data1, (newData) => {
  xAxis.data.setAll(newData);
  barseries1.data.setAll(newData);
  barseries2.data.setAll(newData);
});

watch(data2, (newData) => {
  pieseries.data.setAll(newData);
});
```

- watch를 이용하여 데이터가 변경되었는지를 감지하여 변경된 데이터를 표시합니다. 검색 기능에서 활용합니다.

### 2-7-3. 참고자료

- https://www.amcharts.com/docs/v5/ → amchart v5의 도큐먼트
- https://www.amcharts.com/docs/v5/getting-started/integrations/vue/ → Vue 환경에서의 amchart 샘플 및 도큐먼트


## 2-8. Editor
### 2-8-1. 개요

이 문서는 CrossEditor를 적용한 샘플을 설명하는 문서 입니다.

### 2-8-2. 전제 조건

#### Editor 프로그램 설치

Editor가 최초로 표시될 때 Setup 프로그램이 다운로드 됩니다. 클라이언트 PC에 설치해주세요.

#### 폰트

폰트를 사용하기 위해서는 클라이언트 PC에 설치되어 있어야 합니다. 클라이언트 PC에 설치가 되어있지 않은 경우에는 시스템 디폴트 폰트로 표시됩니다.

### 2-8-3. Editor 샘플 코드

Editor의 표시, Editor에 Html 데이타 설정과 Html 데이타 취득까지의 범위를 샘플로 제공합니다.

#### 샘플 코드 설명

- **HTML 소스**
  ```html
  <CrossEditor
    id="crosseditor1"
    ref="crossEditorRef"
    v-model="inputHtmlData"
    :readOnly="true"
  />
  ```

  1. (필수) id: Editor의 아이디를 설정합니다.
  2. (필수) ref: Editor의 참조 변수를 설정합니다.
  3. (필수) v-model: Editor로 넘겨주기 위한 데이터 변수를 설정합니다.
  4. (옵션) readOnly: true의 경우, 미리보기 모드로 표시됩니다.
  5. (옵션) width: 넓이를 지정하고자 할 때 설정합니다. 숫자 또는 % 지정 가능(예: 800, 50%) default 100%
  6. (옵션) height: 높이를 지정하고자 할 때 설정합니다. 숫자 지정 가능 (예: 800) default 800

- **script 소스**

  ```html
  <script setup>
    import CrossEditor from '@/components/CrossEditor';
    ...

    // 에디터 폼
    const crossEditorRef = ref(null);
    const inputHtmlData = ref('');

    ...
  </script>
  ```

  1. 모듈화되어있는 CrossEditor를 import합니다.
  2. Editor의 참조 변수와 데이터변수를 선언합니다.

### 2-8-4. CrossEditor 설정

```javascript
inputHtmlData.value = 설정값;
```

1. Editor의 데이터변수에 값을 설정하면 Editor에 반영됩니다.

### 2-8-5. HTML 취득처리

```javascript
// 경우에 따라서 선택해서 사용하세요.
crossEditorRef.value.GetBodyValue(); // 현재 편집 중인 문서의 BODY 정보(<body>~</body>)를 취득합니다.
crossEditorRef.value.GetValue(); // 현재 편집 중인 문서의 전체 내용을 HTML 형식으로 취득합니다.
```

1. Editor의 참조변수를 이용하여 GetBodyValue() 메소드를 호출합니다. 에디터에서 제공하는 Html Body 데이터를 취득할 수 있습니다. 일반적으로 본문 데이타를 DB에 저장 시 사용합니다.
2. Html 전체 header부터 취득하고자 할 시에는 GetValue() 를 사용합니다. html 전체를 취득하여 메일 템플리트로 이용할 경우 사용합니다.

### 2-8-6. Text 취득처리

```javascript
crossEditorRef.value.GetTextValue()
```

1. Editor의 참조변수를 이용하여 GetTextValue() 메소드를 호출합니다. Editor에서 제공하는 텍스트 데이터를 취득할 수 있습니다.
