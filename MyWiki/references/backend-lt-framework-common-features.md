---
title: 1. Backend
category: references
tags: ['lt-framework-v3', 'backend', 'library', 'guide']
sources: ["/Users/hongtaegi/project/htg/ai/htg-memory/03_Analysis_Architecture/To-Be/3. 개발 표준/2. 개발 표준 가이드/1. backend/LT Framework 공통 기능 가이드.md"]
created: 2026-06-26
updated: 2026-06-26
summary: "1. Backend"
lifecycle: reviewed
base_confidence: 1.0
---

# 1. Backend
## 1-1. 문자열 마스킹

### 1-1-1. 개요

애플리케이션 내의 개인정보 보호(Privacy) 및 보안 거버넌스(Security Governance)를 준수하기 위해 로그인 ID, 이메일, 전화번호, 이름 등 민감한 문자열을 마스킹하여 반환하는 기능입니다.

LT Framework에서는 크게 두 가지 계층에서 마스킹 처리를 자동화합니다.

#### 1-1-1-1. MyBatis ResultInterceptor 기반 자동 마스킹
- MyBatis를 사용해 DB 데이터를 조회할 때, 도메인/VO 클래스의 특정 필드에 `@Masked` 어노테이션을 부착하여 자동으로 마스킹된 값을 결과 셋에 바인딩합니다.
- `MaskingResultSetHandlerInterceptor` 클래스에서 이 처리를 수행합니다.

#### 1-1-1-2. RestController ResponseBodyAdvice 기반 API Response 마스킹
- API 컨트롤러가 클라이언트(Frontend)로 데이터를 반환할 때, 최종 JSON 결과물 내의 민감 필드를 캐싱된 리플렉션 기술로 찾아 마스킹합니다.
- `ResponseMaskedAdvice` 클래스에서 이 처리를 수행합니다.

---

### 1-1-2. 코드 설명

#### 1-1-2-1. 공통 마스킹 유틸리티 (`MaskingUtil.java`)

```java
package com.valuesplay.common.utils;

public class MaskingUtil {
    // 아이디 마스킹 (뒷부분 지정된 자릿수 마스킹)
    public static String maskId(String id, int digit) {
        if (id == null) return null;
        if (id.length() <= digit) {
            return id.substring(0, 1) + "*".repeat(id.length() - 1);
        }
        return id.replaceAll(".{" + digit + "}$", "*".repeat(id.length() - 1));
    }

    // 이메일 마스킹 (아이디 뒷 3자리 마스킹)
    public static String maskEmail(String email) {
        if (email == null) return null;
        int atIndex = email.indexOf('@');
        if (atIndex == -1) return email;

        String idPart = email.substring(0, atIndex);
        String domainPart = email.substring(atIndex);

        String maskedIdPart = idPart.length() <= 5 
            ? idPart.substring(0, 2) + "*".repeat(idPart.length() - 2)
            : idPart.substring(0, idPart.length() - 3) + "***";

        return maskedIdPart + domainPart;
    }

    // 핸드폰 번호 마스킹 (가운데 4자리 마스킹)
    public static String maskPhoneNumber(String phoneNumber) {
        if (phoneNumber == null) return null;
        return phoneNumber.replaceAll("(\\d{3})(\\d{4})(\\d{4})", "$1****$3");
    }

    // 이름 마스킹 (건너뛰며 * 처리)
    public static String maskName(String name) {
        if (name == null || StringUtils.isEmpty(name)) return name;
        StringBuilder mask = new StringBuilder();
        for (int i = 0; i < name.length(); i += 2) {
            mask.append(name.charAt(i));
            if (i + 1 < name.length()) {
                mask.append("*");
            }
        }
        return mask.toString();
    }
}
```

#### 1-1-2-2. 도메인 필드 어노테이션 정의 (`@Masked`)

```java
package com.valuesplay.framework.interceptor.annotation;

import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Masked {
    MaskType type();

    enum MaskType {
        ID, EMAIL, PHONE, NAME
    }
}
```

#### 1-1-2-3. 도메인 적용 예제 (`User.java`)

```java
public class User {
    private Long userId;

    @Masked(type = Masked.MaskType.ID)
    private String empLognId; // 사용자 아이디 (마스킹 적용)

    @Masked(type = Masked.MaskType.NAME)
    private String empNm;     // 사용자 이름 (마스킹 적용)

    @Masked(type = Masked.MaskType.EMAIL)
    private String empEmail;  // 이메일 (마스킹 적용)
}
```

---

### 1-1-3. 참고자료

- `com.valuesplay.common.utils.MaskingUtil`
- `com.valuesplay.framework.interceptor.MaskingResultSetHandlerInterceptor`

## 1-2. 문자열 암복호화 (AES256)

### 1-2-1. 개요

데이터베이스에 주민등록번호, 카드번호, 계좌번호 등 민감 정보를 저장하거나 조회할 때 보안 규정을 충족하기 위해 단방향(해시) 또는 양방향 암호화를 수행하는 기능입니다. 

LT Framework에서는 기본적으로 비밀번호 수준의 민감 키를 Jasypt 설정 패스워드와 연동하여 SHA-256 기반의 해싱 키와 CBC 모드의 AES-256 알고리즘을 사용해 데이터를 양방향 암복호화 처리합니다.

#### 1-2-1-1. 주요 암호화 컴포넌트

- `AES256Util`: AES-256 알고리즘(AES/CBC/PKCS5Padding) 및 SHA-256 단방향 키 유도 함수를 내장한 양방향 암복호화 유틸리티
- **동작 원리**:
  1. 기동 시 yml의 `jasypt.encryptor.password` 설정 값을 가져와 SHA-256 단방향 해싱하여 256비트(32바이트) 비밀 키 생성
  2. 암호화 시 `SecureRandom`을 사용해 매번 임의의 16바이트 IV(초기화 벡터) 생성
  3. IV와 암호화된 본문 데이터를 결합하여 최종 Base64 인코딩 스트링 생성
  4. 복호화 시 Base64 디코딩 후 앞 16바이트 IV를 분리해 안전하게 평문 복구

---

### 1-2-2. 코드 설명

#### 1-2-2-1. 양방향 암복호화 유틸리티 (`AES256Util.java`)

```java
package com.valuesplay.common.utils;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.crypto.Cipher;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.Base64;

@Component
public class AES256Util {

    private static final String ALGORITHM = "AES/CBC/PKCS5Padding";
    private final byte[] secretKeyBytes;

    // Jasypt의 패스워드 설정을 암호 키의 재료로 사용
    public AES256Util(@Value("${jasypt.encryptor.password:}") String secretKey) {
        if (secretKey == null || secretKey.isBlank()) {
            throw new IllegalArgumentException("AES-256 키가 비어있습니다.");
        }
        this.secretKeyBytes = sha256(secretKey);
    }

    private static byte[] sha256(String value) {
        try {
            return MessageDigest.getInstance("SHA-256").digest(value.getBytes(StandardCharsets.UTF_8));
        } catch (Exception e) {
            throw new IllegalStateException("SHA-256 초기화 실패", e);
        }
    }

    /**
     * 문자열 암호화
     */
    public String encrypt(String plainText) throws Exception {
        byte[] iv = new byte[16];
        new SecureRandom().nextBytes(iv); // 매번 임의의 IV 생성
        IvParameterSpec ivSpec = new IvParameterSpec(iv);

        SecretKeySpec keySpec = new SecretKeySpec(secretKeyBytes, "AES");

        Cipher cipher = Cipher.getInstance(ALGORITHM);
        cipher.init(Cipher.ENCRYPT_MODE, keySpec, ivSpec);

        byte[] encrypted = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));

        // IV(16바이트)와 암호문 결합
        byte[] combined = new byte[iv.length + encrypted.length];
        System.arraycopy(iv, 0, combined, 0, iv.length);
        System.arraycopy(encrypted, 0, combined, iv.length, encrypted.length);

        return Base64.getEncoder().encodeToString(combined);
    }

    /**
     * 문자열 복호화
     */
    public String decrypt(String cipherText) throws Exception {
        byte[] combined = Base64.getDecoder().decode(cipherText);

        // IV 추출
        byte[] iv = new byte[16];
        System.arraycopy(combined, 0, iv, 0, iv.length);
        IvParameterSpec ivSpec = new IvParameterSpec(iv);

        // 암호문 분리
        int encryptedSize = combined.length - iv.length;
        byte[] encrypted = new byte[encryptedSize];
        System.arraycopy(combined, iv.length, encrypted, 0, encryptedSize);

        SecretKeySpec keySpec = new SecretKeySpec(secretKeyBytes, "AES");

        Cipher cipher = Cipher.getInstance(ALGORITHM);
        cipher.init(Cipher.DECRYPT_MODE, keySpec, ivSpec);

        byte[] decrypted = cipher.doFinal(encrypted);
        return new String(decrypted, StandardCharsets.UTF_8);
    }
}
```

#### 1-2-2-2. 비즈니스 서비스 레이어 적용 예제 (`UserServiceImpl.java`)

```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private AES256Util aes256Util;

    @Override
    public int insertUser(User user) {
        try {
            // 개인정보 보호 대상인 전화번호 데이터 양방향 암호화 후 DB 저장
            if (user.getEmpPhonNo() != null) {
                String encryptedPhone = aes256Util.encrypt(user.getEmpPhonNo());
                user.setEmpPhonNo(encryptedPhone);
            }
        } catch (Exception e) {
            throw new ServiceException("개인정보 암호화 중 오류가 발생했습니다.");
        }
        return userMapper.insertUser(user);
    }
}
```

---

### 1-2-3. 참고자료

- `com.valuesplay.common.utils.AES256Util`

## 1-3. 파일 업로드/다운로드

샘플 소스의 CRUD 메뉴에서 추가 팝업을 예로서 설명합니다.

![upload_01.png](/project/upload/upload_01.png)

### 1-3-1. 첨부 파일 업로드
#### 1-3-1-1. Frontend 소스 설명

소스명: src/views/sample/crud/popupSampleGrid.vue

1. 파일 업로드 창


```html
<el-form-item :label="$t('LBL_FILE_UPLD')" prop="sampleFile">
    <el-upload
    class="upload--inline"
    ref="uploadRef"
    v-model:file-list="fileList"
    :headers="upload.headers"
    :action="upload.url"
    :disabled="upload.isUploading"
    :on-preview="handleFilePreview"
    :on-progress="handleFileUploadProgress"
    :on-success="handleFileSuccess"
    :on-error="handleFileError"
    :on-remove="handleFileRemove"
    multiple
    >
    <el-button type="default" size="small">{{ $t('LBL_FILE_CHOC') }}</el-button>
    </el-upload>
</el-form-item>
```
- element ui 의 upload 를 기반으로 합니다.  https://element-plus.org/en-US/component/upload
- 각 파라미터의 설명은 상기 url을 참조해주세요.

2. 파일 업로드 처리

```javascript
// 업로드 파라미터
const upload = ref({
  // 업로드 비활성화 여부
  isUploading: false,
  // 업로드 주소
  url: `${process.env.VUE_APP_BASE_API}/file/upload?fileMaxSize=100000`,
  // 업로드 헤더
  headers: { Authorization: `Bearer ${getToken()}` },
});
```

업로드 주소는

```javascript
`${process.env.VUE_APP_BASE_API}/file/upload?fileMaxSize=100000`,
```
이며, 파일 선택과 동시에 업로드 됩니다.  업로드 후 백엔드 처리는 아래에서 설명합니다.

fileMaxSize에 최대 파일 사이즈를 지정할 수 있으며, 지정하지 않을 시 백엔드에서 지정한 디폴트 맥스 사이즈가 지정됩니다.

```javascript
// 업로드 파일 리스트
const fileList = ref([]);
...
 
// 파일 업로드 처리 완료
const handleFileSuccess = (response) => {
  upload.value.isUploading = false;
  if (response.code === 200) {
    uploadChange.value = true;
  } else {
    if (fileList.value.length > 0) {
      fileList.value.pop();
    }
    ElMessage.error(response.msg);
  }
};
```
처리가 성공적으로 완료시 fileList 에 해당 파일이 들어오며 업로드 결과값은 fileList[i].response 에 저장됩니다.

```javascript
// 파일 ID 취합
const getAtchFileId = () => {
  let atchFileId = '';
  fileList.value.forEach((file) => {
    const id = file.atchFileId ? file.atchFileId : file.response.atchFileId;
    atchFileId += `${id},`;
  });
  if (atchFileId.endsWith(',')) {
    atchFileId = atchFileId.slice(0, -1);
  }
  return atchFileId;
};
```

fileList 의 response.atchFileId 에 저장된 파일 DB 테이블 (tble_atch_file) 의 키가 저장되어 있으므로 콤마(,) 로 연결된 스트링 형식으로 취합합니다.

```
/** 저장 버튼 이벤트 */
const submitForm = () => {
  if (formRef.value) {
    formRef.value.validate((valid) => {
      if (valid) {
        if (form.value.sampleNtcId) {
          // 수정
          if (uploadChange.value) {
            // 파일 변경 시 파일 ID 취합
            form.value.atchFileId = getAtchFileId();
          }
...
```

데이터 등록시 상기에서 취합한 atchFileId 를 추가하여 백엔드 서버에 전달합니다.

#### 1-3-1-2. Backend 소스 설명

1. 파일이 선택될 때 마다 업로드 처리

소스명: com.oliveyoung.project.common.controller.CommonAtchFileController

```java
/**
* 파일 업로드
*/
@PostMapping("/upload")
public AjaxResult uploadFile(@RequestParam("file") MultipartFile file,
                                @RequestParam(value = "fileMaxSize", required = false) Long fileMaxSize) throws Exception {

    try {
        if (fileMaxSize == null) {
            fileMaxSize = defaultMaxSize;
        }

        long fileMaxRequestSize = Long.valueOf(file.getSize()).intValue();
        if (fileMaxRequestSize > fileMaxSize) {
            // 업로드한 파일이 허용된 파일 크기를 초과합니다. 허용된 파일의 최대 크기는: {0}MB
            return error(databaseMessageSource.getMessage("MSG_UPLD_XCSS_MAXSIZE",
                    new String[] { String.valueOf(fileMaxRequestSize) }));
        }

        String filePath = ValuesplayConfig.getUploadPath();
        String fileName = FileUploadUtils.upload(filePath, file, fileMaxSize);

        CommonAtchFile atchFile = new CommonAtchFile();
        atchFile.setAtchOrglFileNm(file.getOriginalFilename());
        atchFile.setAtchFileNm(fileName);
        atchFile.setAtchFileSize(Long.valueOf(file.getSize()).intValue());
        atchFile.setCrtnId(getUsername());

        int result = atchFileService.createCommonAtchFile(atchFile);
        long atchFileId = 0;
        if (result > 0) {
            atchFileId = atchFile.getAtchFileId();
        }

        AjaxResult ajaxResult = toAjax(result);
        ajaxResult.put("atchFileId", atchFileId);

        return ajaxResult;

    } catch (Exception e) {
        return error(e.getMessage());
    }
}
```

2. 데이터 등록 시 파일 업로드 처리

소스명: com.oliveyoung.project.sample.service.SampleGridServiceImpl

```java
@Override
public int modifySampleGrid(SampleGrid grid) {
    int result = 0;

    // 첨부파일이 있을 경우 첨부파일 그룹 생성
    result = createAtchFileGrp(grid, true);
    if (result < 1) {
        return result;
    }

    result = sampleGridCommandMapper.updateSampleGrid(grid);
    return result;
}

 
/**
    * 첨부파일 그룹 생성
    * @param grid
    * @param isUpdate
    * @return
*/
private int createAtchFileGrp(SampleGrid grid, boolean isUpdate) {
    int result = 1;
    // 첨부파일이 있을 경우 첨부파일 그룹 생성
    if(StringUtils.isNotNull(grid.getAtchFileId()) && !grid.getAtchFileId().isEmpty()) {
        if(isUpdate) {
            // 업데이트인 경우 기존 첨부파일 그룹 삭제
            commonAtchFileCommandMapper.deleteCommonAtchFileGrp(grid.getAtchFileGrpId());
        }
        String[] atchFieldIds = grid.getAtchFileId().split(",");
        long atchFileGrpId = commonAtchFileCommandMapper.getNextAtchFileGrpId();
        List<CommonAtchFile> atchFileList = new ArrayList<>();
        for(String atchFileId : atchFieldIds) {
            CommonAtchFile atchFile = new CommonAtchFile();
            atchFile.setAtchFileId(Long.parseLong(atchFileId));
            atchFile.setAtchFileGrpId(atchFileGrpId);
            atchFileList.add(atchFile);
        }
        result = commonAtchFileCommandMapper.insertCommonAtchFileGrp(atchFileList);
        if (result > 0) {
            grid.setAtchFileGrpId(atchFileGrpId);
        }
    }
    return result;
}
```
화면에서 추가된 atchFileId 를 split 하여 tble_atch_file_grp 테이블에 그룹 ID로 파일을 그룹화 하여 저장합니다.

이때 업무 DB의 ATCH_FILE_GRP_ID 컬럼에 파일 그룹ID를 저장합니다.

### 1-3-2. 그리드 첨부 파일 업로드 / 다운로드
#### 1-3-2-1. 그리드 첨부 파일 업로드
##### 1-3-2-1-1. Frontend 소스 설명
소스: /sample/grid2/index.vue

```javascript
<!-- 파일 업로드 팝업 -->
    <PopupUploadFile
      v-model:visible="popupUploadVisible"
      :popupParams="popupUploadParams"
      @setAtchFile="handleSetAtchFile"
    />
 
...
 
// 팝업 업로드 파라메터
const popupUploadParams = ref({
  // 팝업 레이어 제목
  popupTitle: '',
  // 이미지 미리보기인 경우 설정(picture-card)
  // listType: 'picture-card',
  // 업로드의 필수 헤더 설정
  headers: { Authorization: `Bearer ${getToken()}` },
  // 업로드 주소
  url: `${process.env.VUE_APP_BASE_API}/file/upload`,
  // 업로드 파일 최대 크기
  fileMaxSize: 10000000, // 10MB
  // 업로드 파일 확장자
  accept: '.jpg, .jpeg, .png .gif, .bmp',
  // 가이드 문자열
  guide: [
    '파일 1개의 용량은 10MB를 초과할 수 없습니다.',
    '파일형식은 png, jpg, jpeg, gif, bmp만 등록할 수 있습니다.',
    '파일명은 앞자리가 88로 시작하는 11자리 숫자로 작성합니다. (예시 : 8800000000000.jpg)',
  ],
});
 
...
 
/** 파일 업로드 클릭 */
const handleImport = (row) => {
  const data = row.getData();
  popupUploadVisible.value = true;
  popupUploadParams.value.popupTitle = `${t('LBL_IMPT')}`;
  popupUploadParams.value.selectedId = data.sampleNtcId;
  popupUploadParams.value.selectedIndex = row.getPosition();
  popupUploadParams.value.getAtchFileList = getSampleGrid;
};
...
 
// 파일 업로드 처리
const handleSetAtchFile = (selectedId, selectedIndex, fileId) => {
  let cnt = 0;
  tabulator.value.getRows().forEach((row) => {
    const rowData = row.getData();
    if (selectedId !== undefined) {
      // 수정
      if (rowData.sampleNtcId === selectedId) {
        rowData.atchFileId = fileId;
        rowData.rowState = 'v';
        row.update(rowData);
      }
    } else if (selectedIndex - 1 === cnt) {
      // 추가
      rowData.atchFileId = fileId;
      rowData.rowState = '+';
      row.update(rowData);
    }
    cnt += 1;
  });
  popupUploadVisible.value = false;
};
```

 1. 파일업로드 팝업에서는 setAtchFile 이라는 emit 펑션을 작성합니다.
 2. 팝업 파라미터는 상기와 같은 파라미터를 지정할 수 있습니다. 확장자를 '' 로 설정하면 모든 파일을 업로드 가능합니다.
 3. 파일 업로드 클릭시 수정 화면의 경우를 위하여 파일 취득 api 함수를 파라미터로 설정합니다. (상기 getSampleGrid)
 4. setAtchFile 펑션에서는 각 행에 업로드한 파일id 를 1,2,5. 형식으로 행의 atchFileId 컬럼 값으로 추가합니다.
 5. 이미지 미리보기(썸네일) 스타일인 경우 listType을 'picture-card'를 설정합니다.

 ##### 1-3-2-1-2. Backend 소스 설명

 소스: SampleGridServiceImpl.java

 ```
 @Override
@Transactional
public int saveSampleGrid(SampleGrid sampleGrid) {
    int result = 0;
    result = sampleGridCommandMapper.updateSampleGrp(sampleGrid);
    if (result < 1) {
        return result;
    }
    if(sampleGrid.getSampleGridList() != null) {
        for (SampleGrid rowSampleGrid : sampleGrid.getSampleGridList()) {
            // 첨부파일이 있을 경우 첨부파일 그룹 생성
            sampleGrid.setAtchFileGrpId(commonAtchFileService.createAtchFileGrp(sampleGrid.getAtchFileId()));

            if (rowSampleGrid.getRowState().equals("+")) {
                // 신규인 경우
                result = createSampleGrid(rowSampleGrid);
            } else if (rowSampleGrid.getRowState().equals("-")) {
                // 삭제인 경우
                result = removeSampleGridById(rowSampleGrid.getSampleNtcId());
            } else {
                // 수정인 경우
                result = modifySampleGrid(rowSampleGrid);
            }
        }
    }
    return result;
}
 ```

첨부파일 그룹 테이블에 값을 입력 후 해당 그룹 id 를 업무 테이블에 저장합니다. (예. tble_sample_ntc. ATCH_FILE_GRP_ID)

## 1-4. 엑셀 업로드 / 다운로드
### 1-4-1. 엑셀 업로드(일반 - 10,000건 이하)
#### 1-4-1-1. Frontend 소스 설명
CRUD 샘플의 엑셀 업로드 화면을 예로서 설명합니다.
엑셀업로드는 화면에서 업로드한 엑셀을 Redis 캐쉬 서버에 저장 후 리스트에 에러가 없는 지 확인 후 DB에 저장하는 형식으로 되어 있습니다.

1. 팝업클릭
엑셀 업로드 팝업에 파라미터를 설정합니다.

```javascript
// 팝업 엑셀 업로드 파라메터
const popupExcelUploadParams = ref({
  // 팝업 레이어 제목
  popupTitle: '',
  // 엑셀 업로드의 필수 헤더 설정
  headers: { Authorization: `Bearer ${getToken()}` },
  // 그리드 컬럼
  columns: [],
  // 그리드 파라미터 (저장 키)
  gridParams: {},
  // 업로드 주소
  url: `${process.env.VUE_APP_BASE_API}/sample/grid/import-data`,
  // 캐쉬 엑셀 저장 함수
  saveApi: addSampleGridCacheExcel,
  // 템플릿 다운로드 URL
  templateUrl: 'sample/grid/import-template',
  // 템플릿 다운로드 파일명
  templateFile: `sample_template_${new Date().getTime()}.xlsx`,
});
...
 
// 그리드 개인 정보 파라미터
const gridParams = ref({
  empId: undefined,
  routePath: undefined,
  gridId: undefined,
});
...
 
/** 엑셀 업로드 클릭 */
const handleExcelImport = () => {
  popupExcelUploadParams.value.popupTitle = `${t('MSG_NTC')} ${t('LBL_IMPT')}`;
  popupExcelUploadParams.value.columns = columns.value;
  popupExcelUploadParams.value.gridParams = gridParams.value;
  popupExcelUploadVisible.value = true;
};
```

- 엑셀 업로드 클릭시 팝업 파라미터에 이하의 값을 설정합니다.
- columns: 엑셀 업로드 팝업에 표시할 그리드 컬럼값 (샘플은 메인 화면의 그리드와 동일하기 때문에 메인화면의 column을 그대로 전달)
- gridParams: Redis 에 저장을 위한 키값(empid: 유저아이디, routhPath: 화면경로, gridid: 그리드아이디)
- url: 엑셀 업로드 후 체크, Redis에 저장하는 api 의 url. 업무마다 작성
- saveApi: 엑셀 업로드 후 체크, Redis에 저장하는 api 함수. 업무마다 작성

2. 엑셀 업로드 팝업

![upload_02.png](/project/upload/upload_02.png)

- 파일선택 탭: 엑셀파일을 선택한 후 업로드 버튼을 클릭합니다.
- 선택한 엑셀 파일은 DRM 복호화를 거쳐 Redis에 저장됩니다.

#### 1-4-1-2. Backend 소스 설명
1. 업로드 파일 Redis 저장

상기 url 에 지정한 백엔드 api가 호출됩니다.

소스명: com.oliveyoung.project.sample.controller.SampleGridController

```java
@Log(title = "샘플 EXCEL 업로드", businessType = BusinessType.IMPORT)
@Operation(summary = "EXCEL 업로드")
@PostMapping("/import-data")
public AjaxResult importData(@RequestParam("file") MultipartFile file, @RequestParam("saveKey") String saveKey) throws Exception {
    // 파일 복호화
    File decryptFile = new DrmUtil().decryptWithMultipartFile(file);

    ExcelDrmUtil<SampleGrid> util = new ExcelDrmUtil<>(SampleGrid.class);
    List<SampleGrid> sampleGridList = null;
    if (!decryptFile.exists()) {
        // 파일이 존재하지 않는 경우 복호화 실패 이므로 원본 파일로 처리
        sampleGridList = util.importExcel(file.getInputStream());
    } else {
        // 복호화 성공한 경우 복호화 파일로 처리
        sampleGridList = util.importExcel(decryptFile);
    }

    AjaxResult message = sampleGridService.createSampleGridList(sampleGridList, saveKey);
    return success(message);
}
```

소스명: com.oliveyoung.project.sample.service.SampleGridServiceImpl

```java
@Log(title = "샘플 EXCEL 업로드", businessType = BusinessType.IMPORT)
@Operation(summary = "EXCEL 업로드")
@PostMapping("/import-data")
public AjaxResult importData(@RequestParam("file") MultipartFile file, @RequestParam("saveKey") String saveKey) throws Exception {
    // 파일 복호화
    File decryptFile = new DrmUtil().decryptWithMultipartFile(file);

    ExcelDrmUtil<SampleGrid> util = new ExcelDrmUtil<>(SampleGrid.class);
    List<SampleGrid> sampleGridList = null;
    if (!decryptFile.exists()) {
        // 파일이 존재하지 않는 경우 복호화 실패 이므로 원본 파일로 처리
        sampleGridList = util.importExcel(file.getInputStream());
    } else {
        // 복호화 성공한 경우 복호화 파일로 처리
        sampleGridList = util.importExcel(decryptFile);
    }

    AjaxResult message = sampleGridService.createSampleGridList(sampleGridList, saveKey);
    return success(message);
}
```

소스명: com.oliveyoung.project.sample.service.SampleGridServiceImpl

```java
@Override
public AjaxResult createSampleGridList(List<SampleGrid> sampleGridList, String saveKey) {
    if (StringUtils.isNull(sampleGridList) || sampleGridList.isEmpty()) {
        throw new ServiceException(databaseMessageSource.getMessage("MSG_FILE_SHEET_NOT_XSTC"));
    }

    long successCount = sampleGridList.size();
    long failureCount = 0;

    try {
        // 엑셀데이터 체크
        for (SampleGrid sampleGrid : sampleGridList) {
            sampleGrid.setChkState(Constants.SUCCESS);

            if(StringUtils.isEmpty(sampleGrid.getSampleNtcTitl())) {
                sampleGrid.setChkState(Constants.FAIL);
                sampleGrid.setChkDesc("제목은 필수항목 입니다.");
                successCount--;
                failureCount++;
            }
        }

        // 캐시에 저장
        redisCache.setCacheObject(saveKey, sampleGridList, 5, TimeUnit.MINUTES);

    } catch (Exception e) {
        e.printStackTrace();
        throw new ServiceException(e.getMessage());
    }

    AjaxResult result = AjaxResult.success();
    result.put("successCount", successCount);
    result.put("failureCount", failureCount);
    return result;
}
```

- 첫번째 createSampleGridList는 Redis에 엑셀 데이터를 저장합니다. 이때 데이터를 체크 후 상태 및 에러내용을 같이 저장합니다.
- 엑셀 데이터의 체크는 업무마다 다르기 때문에 별개로 작성합니다.
- 체크가 끝난 리스트는 Redis 캐시에 저장합니다.

2. Redis 저장 데이터를 DB에 저장  
![upload_03.png](/project/upload/upload_03.png)

- 업로드 결과 탭에서 오류 내용을 확인 후 에러가 없을 시 등록 버튼을 클릭합니다.
- 등록 버튼을 클릭하면 상기의 saveApi 가 실행됩니다.

saveApi: api/sample/grid.js -> addSampleCacheExcel

```java
// 캐쉬 엑셀 리스트 저장
export function addSampleGridCacheExcel(saveKey) {
  return request({
    url: `/sample/grid/import-data-save/${saveKey}`,
    method: 'post',
  })
}
```

3. Redis 데이터를 DB에 저장
소스명: com.oliveyoung.project.sample.controller.SampleGridController

```java
@Log(title = "샘플 EXCEL 업로드 저장", businessType = BusinessType.IMPORT)
@Operation(summary = "EXCEL 업로드 저장")
@PostMapping("/import-data-save/{saveKey}")
public AjaxResult importDataSave(@PathVariable("saveKey") String saveKey) throws Exception {
    return toAjax(sampleGridService.createSampleGridList(saveKey));
}
```

소스명: com.oliveyoung.project.sample.service.SampleGridServiceImpl

```java
@Override
public int createSampleGridList(String saveKey) {
    List<SampleGrid> excelExtList = redisCache.getCacheObject(saveKey);
    if (StringUtils.isNull(excelExtList)) {
        throw new ServiceException(databaseMessageSource.getMessage("MSG_FILE_SHEET_NOT_XSTC"));
    }
    return sampleGridCommandMapper.insertSampleGridBatch(excelExtList);
}
```
    - 두번째 createSampleGridList는 Redis에 저장된 데이터를 전달된 키로 취득하여 DB에 저장합니다.

### 1-4-2. 엑셀 업로드 템플릿 파일 다운로드
#### 1-4-2-1. Backend 소스 설명
1. 상기 2.1 에 팝업 파라미터에 템플릿 관련 파라미터(url, 파일명) 을 설정합니다.
2. 백엔드를 작성합니다.
소스명: com.oliveyoung.project.sample.controller.SampleGridController

```
@Log(title = "샘플 EXCEL 템플릿 다운로드", businessType = BusinessType.IMPORT)
@PostMapping("/import-template")
@Operation(summary = "EXCEL 템플릿 다운로드")
public void exportTemplate(HttpServletResponse response) {
    ExcelDrmUtil<SampleGrid> util = new ExcelDrmUtil<SampleGrid>(SampleGrid.class);
    util.importTemplateExcel(response, "공지사항");
}
```

상기의 importTemple는 domain 의 '@Excel' 로 지정한 엑셀 출력 항목을 출력하도록 되어 있습니다.  (엑셀 다운로드와 동일)

## 1-5. JWT 생성 / 삭제 / 만료시간체크 / 토큰검증

### 1-5-1. 개요

무상태(Stateless) 아키텍처 기반의 안전한 사용자 인증 및 인가를 처리하기 위한 JWT(Json Web Token) 제어 모듈입니다.

LT Framework에서는 기본적으로 다음과 같이 설계 및 적용되어 있습니다.

#### 1-5-1-1. 주요 인증 메커니즘 및 API

- `TokenService`: JWT 생성, 파싱, 갱신 및 만료 시간 확인을 통합 처리하는 핵심 서비스 컴포넌트
- **동작 흐름**:
  1. **토큰 생성**: 로그인 성공 시 임의의 UUID를 생성해 Redis 캐시 서버 또는 RDB(`UserToken`)에 사용자 인증 정보(`LoginUser`)와 함께 바인딩하고, 이 UUID와 사용자 VO 객체를 Claims로 담아 HS512 알고리즘으로 서명된 JWT를 생성하여 클라이언트에 반환합니다.
  2. **필터 기반 검증**: 클라이언트 요청 시 HTTP Header(`Authorization: Bearer <Token>`)를 통해 수신된 JWT를 파싱하고 만료 시간을 체크하여, RDB/Redis 세션 데이터가 유효할 경우 Spring Security Context에 인증 객체(`UsernamePasswordAuthenticationToken`)를 주입합니다.
  3. **만료시간 연장 (Sliding Session)**: 요청이 유입될 때마다 토큰의 남은 유효 시간을 체크하여, 지정된 만료 시간 이내일 경우 세션 유효 기간을 자동으로 자동 연장(Refresh)합니다.

---

### 1-5-2. 코드 설명

#### 1-5-2-1. 토큰 생성 및 세션 캐싱 예제 (`TokenService.java`)

```java
@Component
public class TokenService {

    @Value("${token.header:Authorization}")
    private String header;

    @Value("${token.secret:}")
    private String secret;

    @Value("${token.expire-time:60}") // 기본 60분
    private int expireTime;

    @Autowired
    private RedisCache redisCache; // Redis 사용 시 캐시 모듈

    // JWT 생성 및 세션 등록
    public String createToken(LoginUser loginUser) {
        String token = IdUtils.fastUUID();
        loginUser.setToken(token);

        // Redis 세션 스토어에 사용자 세션 정보 및 만료시간 매핑 저장
        String userKey = getTokenKey(token);
        redisCache.setCacheObject(userKey, loginUser, expireTime, TimeUnit.MINUTES);

        // JWT Claims에 세션 키(UUID) 주입
        Map<String, Object> claims = new HashMap<>();
        claims.put(Constants.LOGIN_USER_KEY, token);

        return createToken(claims);
    }

    // JWT 실제 빌드 (HMAC-SHA512)
    private String createToken(Map<String, Object> claims) {
        SecretKey key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        return Jwts.builder()
                .setClaims(claims)
                .signWith(key, SignatureAlgorithm.HS512)
                .compact();
    }
}
```

#### 1-5-2-2. 토큰 파싱 및 만료시간 확인 예제

```java
@Component
public class TokenService {

    // JWT 파싱 및 클레임 추출
    private Claims parseToken(String token) {
        SecretKey key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        return Jwts.parserBuilder()
                .setSigningKey(key)
                .build()
                .parseClaimsJws(token)
                .getBody();
    }

    // 세션 만료 검증 및 슬라이딩 세션 갱신
    public void verifyToken(LoginUser loginUser) {
        long currentTime = System.currentTimeMillis();
        long cachedExpireTime = loginUser.getExpireTime();

        if (currentTime < cachedExpireTime) {
            // 유효할 경우 만료 기한 자동 연장 (Sliding Session)
            refreshToken(loginUser);
        } else {
            // 만료 시 세션 삭제 및 에러 반환
            delLoginUser(loginUser.getToken());
            throw new ServiceException("세션이 만료되었습니다. 다시 로그인 해주세요.", HttpStatus.UNAUTHORIZED);
        }
    }
}
```

---

### 1-5-3. 참고자료

- `com.valuesplay.framework.security.service.TokenService`
- `com.valuesplay.framework.security.filter.JwtAuthenticationTokenFilter`

## 1-6. Redis, ElastiCache

### 1-6-1. 개요

성능 병목을 방지하고 상태 비저장(Stateless) 인프라 환경에서 분산 세션 관리, 데이터 캐싱, 실시간 데이터 공유 및 임시 작업 큐를 처리하기 위한 고성능 In-memory Key-Value 스토어 연동 제어 기능입니다.

LT Framework에서는 기본적으로 다음과 같이 설계 및 적용되어 있습니다.

#### 1-6-1-1. 주요 아키텍처 및 역할

- `RedisCache`: Spring Data Redis의 `RedisTemplate`을 래핑하여 객체, 리스트, 맵, 셋 등의 Java 데이터를 Redis에 직렬화하여 저장하고 조회하는 공통 컴포넌트
- **사용 시나리오**:
  1. **SSO / JWT 세션 캐싱**: 무상태(Stateless) 아키텍처 상에서 유입되는 JWT 토큰에 해당하는 상세 사용자 데이터(`LoginUser`)를 Redis 세션 키(`login_tokens:<uuid>`)로 저장하여 DB 쿼리 부하를 방지합니다.
  2. **임시 엑셀 데이터 적재**: 대용량 엑셀 업로드 시, 임시 데이터를 DB에 직접 적재하지 않고 검증 완료될 때까지 Redis 캐시 서버에 5분 내외(`TimeUnit.MINUTES`)로 저장하여 메모리 부담을 경감시킵니다.
  3. **인증 캡차 및 인증 코드**: SMS/Email 2FA나 로그인 시 필요한 검증 난수를 특정 TTL(Time To Live)을 지정하여 적재하고 검증 완료 후 자동 삭제되도록 합니다.

---

### 1-6-2. 코드 설명

#### 1-6-2-1. 공통 캐시 클래스 인터페이스 (`RedisCache.java`)

```java
package com.valuesplay.framework.redis;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;
import java.util.Collection;
import java.util.List;
import java.util.concurrent.TimeUnit;

@Component
public class RedisCache {
    @Autowired
    public RedisTemplate redisTemplate;

    // 단일 객체 캐시 저장 (TTL 지정 가능)
    public <T> void setCacheObject(final String key, final T value, final Integer timeout, final TimeUnit timeUnit) {
        redisTemplate.opsForValue().set(key, value, timeout, timeUnit);
    }

    // 단일 객체 캐시 조회
    public <T> T getCacheObject(final String key) {
        return (T) redisTemplate.opsForValue().get(key);
    }

    // 키 삭제
    public boolean deleteObject(final String key) {
        return redisTemplate.delete(key);
    }

    // 리스트 형식 데이터 저장
    public <T> long setCacheList(final String key, final List<T> dataList) {
        Long count = redisTemplate.opsForList().rightPushAll(key, dataList);
        return count == null ? 0 : count;
    }

    // 리스트 형식 데이터 조회
    public <T> List<T> getCacheList(final String key) {
        return redisTemplate.opsForList().range(key, 0, -1);
    }
}
```

#### 1-6-2-2. 엑셀 업로드 임시 데이터 Redis 적재 비즈니스 적용 예제

```java
@Service
public class SampleGridServiceImpl implements SampleGridService {

    @Autowired
    private RedisCache redisCache;

    @Override
    public AjaxResult createSampleGridList(List<SampleGrid> sampleGridList, String saveKey) {
        // 1차 데이터 정합성 검증 수행
        for (SampleGrid sampleGrid : sampleGridList) {
            if (StringUtils.isEmpty(sampleGrid.getSampleNtcTitl())) {
                sampleGrid.setChkState(Constants.FAIL);
                sampleGrid.setChkDesc("제목은 필수 항목입니다.");
            }
        }
        
        // 검증이 완료된 임시 리스트 데이터를 Redis에 5분간 캐싱
        redisCache.setCacheObject(saveKey, sampleGridList, 5, TimeUnit.MINUTES);

        AjaxResult result = AjaxResult.success();
        result.put("successCount", sampleGridList.size());
        return result;
    }
}
```

---

### 1-6-3. 참고자료

- `com.valuesplay.framework.redis.RedisCache`

## 1-7. 동기/비동기 HTTP 클라이언트 (HttpUtils)

### 1-7-1. 개요

외부 시스템과의 원격 연동 또는 내부 API 서비스를 연동하여 동기 또는 비동기 형태로 HTTP/HTTPS 통신을 수행하기 위한 유틸리티 모듈입니다.

LT Framework에서는 기본적으로 표준 자바의 `URLConnection` 및 `HttpsURLConnection`을 최적화하여 래핑한 동기식 통신 유틸리티를 제공합니다.

#### 1-7-1-1. 주요 기능 및 컴포넌트

- `HttpUtils`: GET, POST, SSL POST 등의 통신 방식을 간결한 static 메서드로 호출할 수 있게 지원합니다.
- **제공 기능**:
  - `sendGet`: 지정된 URL에 쿼리 스트링 매개변수와 문자셋(UTF-8 등) 설정을 적용하여 HTTP GET 요청을 전송하고 응답 문자열을 반환합니다.
  - `sendPost`: HTTP POST 통신 바디에 파라미터를 인코딩하여 전송하고 결괏값을 수신합니다.
  - `sendSSLPost`: HTTPS 프로토콜 연동 시 보안 인증서 우회 검증 구조(`TrustAnyTrustManager`, `TrustAnyHostnameVerifier`)를 탑재하여 신뢰할 수 있는 SSL 통신을 연결합니다.

---

### 1-7-2. 코드 설명

#### 1-7-2-1. 공통 HTTP 전송 유틸리티 (`HttpUtils.java`)

```java
package com.valuesplay.common.utils.http;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.URL;
import java.net.URLConnection;
import java.nio.charset.StandardCharsets;

public class HttpUtils {

    // HTTP GET 요청 전송
    public static String sendGet(String url, String param) {
        StringBuilder result = new StringBuilder();
        BufferedReader in = null;
        try {
            String urlNameString = (param == null || param.isBlank()) ? url : url + "?" + param;
            URL realUrl = new URL(urlNameString);
            URLConnection connection = realUrl.openConnection();
            connection.setRequestProperty("accept", "*/*");
            connection.setRequestProperty("connection", "Keep-Alive");
            connection.setRequestProperty("user-agent", "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1;SV1)");
            connection.connect();
            
            in = new BufferedReader(new InputStreamReader(connection.getInputStream(), StandardCharsets.UTF_8));
            String line;
            while ((line = in.readLine()) != null) {
                result.append(line);
            }
        } catch (Exception e) {
            log.error("HttpUtils.sendGet Exception, url=" + url, e);
        } finally {
            closeReader(in);
        }
        return result.toString();
    }

    // HTTP POST 요청 전송
    public static String sendPost(String url, String param) {
        PrintWriter out = null;
        BufferedReader in = null;
        StringBuilder result = new StringBuilder();
        try {
            URL realUrl = new URL(url);
            URLConnection conn = realUrl.openConnection();
            conn.setRequestProperty("accept", "*/*");
            conn.setRequestProperty("connection", "Keep-Alive");
            conn.setDoOutput(true);
            conn.setDoInput(true);
            
            out = new PrintWriter(conn.getOutputStream());
            out.print(param);
            out.flush();
            
            in = new BufferedReader(new InputStreamReader(conn.getInputStream(), StandardCharsets.UTF_8));
            String line;
            while ((line = in.readLine()) != null) {
                result.append(line);
            }
        } catch (Exception e) {
            log.error("HttpUtils.sendPost Exception, url=" + url, e);
        } finally {
            closeWriter(out);
            closeReader(in);
        }
        return result.toString();
    }
}
```

#### 1-7-2-2. 외부 연동 API 비즈니스 적용 예제

```java
@Service
public class ExternalApiServiceImpl implements ExternalApiService {

    @Value("${api.external.url}")
    private String externalApiUrl;

    @Override
    public String getExternalData(String searchCode) {
        // 파라미터 조립
        String param = "code=" + searchCode + "&type=json";
        
        // HttpUtils 공통 컴포넌트를 이용해 동기 통신 수행
        String jsonResponse = HttpUtils.sendGet(externalApiUrl, param);
        
        if (jsonResponse == null || jsonResponse.isEmpty()) {
            throw new ServiceException("외부 시스템 연동에 실패했습니다.");
        }
        return jsonResponse;
    }
}
```

---

### 1-7-3. 참고자료

- `com.valuesplay.common.utils.http.HttpUtils`

## 1-8. 페이징 처리

### 1-8-1. 개요

대량의 조회 결과를 특정 크기 단위(Page)로 분할하여 클라이언트에게 필요한 구역의 데이터만 신속하게 반환하기 위한 페이징 및 정렬 처리 모듈입니다.

LT Framework에서는 기본적으로 MyBatis 오픈소스 페이징 플러그인인 `PageHelper`를 사용하여 간결하게 처리하도록 설계되어 있습니다.

#### 1-8-1-1. 주요 아키텍처 및 역할

- `PageUtils / BaseController`: HTTP Request 파라미터(`pageNum`, `pageSize`, `orderBy` 등)를 인터셉트하여 ThreadLocal 영역에 페이징 정보를 보관하고 자동으로 데이터베이스 쿼리에 물리적 페이징 절(PostgreSQL의 `LIMIT`, `OFFSET`)을 삽입합니다.
- **제공 API**:
  - `startPage()`: HTTP 요청으로부터 넘어온 현재 페이지 번호와 페이지당 레코드 수를 추출하여 페이징을 가동합니다.
  - `getDataTable(List<?> list)`: PageHelper를 통해 조회된 List 결과를 래핑하여 총 카운트(`total`), 결과 행 데이터(`rows`), 응답 코드(`code`) 등을 갖춘 표준 `TableDataInfo` 객체로 가공해 클라이언트(IBSheet8 등)에 전달합니다.

---

### 1-8-2. 코드 설명

#### 1-8-2-1. 공통 페이징 제어 메서드 (`BaseController.java`)

```java
package com.valuesplay.framework.web.controller;

import com.github.pagehelper.PageInfo;
import com.valuesplay.common.constant.HttpStatus;
import com.valuesplay.common.utils.PageUtils;
import com.valuesplay.framework.web.page.TableDataInfo;
import java.util.List;

public class BaseController {

    // 페이징 가동
    protected void startPage() {
        PageUtils.startPage();
    }

    // 결과 목록을 표준 페이징 구조로 패키징
    @SuppressWarnings({ "rawtypes", "unchecked" })
    protected TableDataInfo getDataTable(List<?> list) {
        TableDataInfo rspData = new TableDataInfo();
        rspData.setCode(HttpStatus.SUCCESS);
        rspData.setMsg("조회 성공");
        rspData.setRows(list);
        rspData.setTotal(new PageInfo(list).getTotal()); // 총 레코드 수 추출
        return rspData;
    }
}
```

#### 1-8-2-2. 컨트롤러 비즈니스 적용 예제 (`NoticeController.java`)

```java
@RestController
@RequestMapping("/system/notice")
public class NoticeController extends BaseController {

    @Autowired
    private NoticeService noticeService;

    // 공지사항 페이징 목록 조회
    @GetMapping("/list")
    public TableDataInfo list(Notice notice) {
        // 1. ThreadLocal에 현재 요청의 페이징 설정 바인딩 (자동 LIMIT/OFFSET 적용)
        startPage();
        
        // 2. 비즈니스 쿼리 실행
        List<Notice> list = noticeService.selectNoticeList(notice);
        
        // 3. PageInfo를 통해 데이터 가공 후 표준 결과 반환
        return getDataTable(list);
    }
}
```

---

### 1-8-3. 참고자료

- `com.valuesplay.common.utils.PageUtils`
- `com.valuesplay.framework.web.controller.BaseController`

## 1-9. Swagger (Springdoc)

### 1-9-1. 개요

애플리케이션의 RESTful API를 시각적으로 문서화하고 개발 및 검증 단계에서 즉시 테스트할 수 있도록 지원하는 Springdoc OpenAPI 기반 API 문서화 모듈입니다.
LT Framework에서는 기본적인 API 스펙 생성 기능 외에, Spring Security 환경 하에 보호되는 리소스에 접근하기 위한 **JWT Bearer 인증(Authorization) 설정**을 공통 탑재하여 편리한 테스트 환경을 제공합니다.

#### 1-9-1-1. 주요 아키텍처 및 설정
- `OpenApiConfig`: OpenAPI 객체를 정의하여 API 문서 제목/설명/버전을 관리하고, Security Components에 Bearer JWT 인증 설정을 주입합니다.
- `@Tag`, `@Operation`: 컨트롤러 및 개별 API 메서드 단위에 주석을 달아 Swagger UI 화면에 한글 매핑 및 세부 설명을 표출합니다.

---

### 1-9-2. 코드 설명

#### 1-9-2-1. OpenAPI 공통 설정 클래스 (`OpenApiConfig.java`)

```java
package com.valuesplay.framework.config;

import java.util.Arrays;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.security.SecurityRequirement;
import io.swagger.v3.oas.models.security.SecurityScheme;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI usersMicroserviceOpenAPI() {
        // Bearer Token 형식의 Security Scheme 설정 (JWT 전송용)
        SecurityScheme securityScheme = new SecurityScheme()
                .type(SecurityScheme.Type.HTTP).scheme("bearer").bearerFormat("JWT")
                .in(SecurityScheme.In.HEADER).name("Authorization");
        
        SecurityRequirement securityRequirement = new SecurityRequirement().addList("bearerAuth");
    	
        return new OpenAPI()
                .info(new Info().title("LT Framework API 명세서")
                                 .description("LT Framework 백엔드 공통 및 업무 API 가이드")
                                 .version("1.0"))
                .components(new Components().addSecuritySchemes("bearerAuth", securityScheme))
                .security(Arrays.asList(securityRequirement));
    }
}
```

#### 1-9-2-2. 컨트롤러 적용 예제 (`LoginController.java`)

```java
@Tag(name = "로그인관련")
@RestController
public class LoginController {

    @PostMapping("/login")
    @Operation(summary = "로그인관련 - 로그인 토큰조회")
    public AjaxResult login(@RequestBody LoginBody loginBody) {
        return loginTokenService.login(loginBody);
    }
}
```

---

### 1-9-3. 참고자료

- `com.valuesplay.framework.config.OpenApiConfig`
- `io.swagger.v3.oas.annotations` 라이브러리

## 1-10. 공통 유틸(String, Array, Date, ...)

### 1-10-1. 개요

개발 생산성을 향상시키고 예외 처리를 정형화하기 위해 문자열, 날짜, 데이터 구조 등을 손쉽게 다룰 수 있도록 래핑한 유틸리티 컴포넌트 모듈입니다.

#### 1-10-1-1. 주요 유틸리티 클래스
- `StringUtils`: `org.apache.commons.lang3.StringUtils`를 확장하여 Null 처리 및 빈 값 검증을 다각도로 처리합니다.
- `DateUtils`: `org.apache.commons.lang3.time.DateUtils`를 확장하여 현지 및 서버 기준 시각 포맷팅, 파싱 및 차이 연산을 처리합니다.

---

### 1-10-2. 코드 설명

#### 1-10-2-1. 문자열 유틸리티 (`StringUtils.java`)

```java
package com.valuesplay.common.utils;

public class StringUtils extends org.apache.commons.lang3.StringUtils {
    private static final String NULLSTR = "";

    // Null인 경우 Default Value 반환
    public static <T> T nvl(T value, T defaultValue) {
        return value != null ? value : defaultValue;
    }

    // 문자열 빈 값 확인
    public static boolean isEmpty(String str) {
        return isNull(str) || NULLSTR.equals(str.trim());
    }

    // 객체 Null 확인
    public static boolean isNull(Object object) {
        return object == null;
    }

    public static boolean isNotNull(Object object) {
        return !isNull(object);
    }
}
```

#### 1-10-2-2. 날짜 유틸리티 (`DateUtils.java`)

```java
package com.valuesplay.common.utils;

import java.text.SimpleDateFormat;
import java.util.Date;

public class DateUtils extends org.apache.commons.lang3.time.DateUtils {
    public static String YYYY_MM_DD = "yyyy-MM-dd";
    public static String YYYY_MM_DD_HH_MM_SS = "yyyy-MM-dd HH:mm:ss";

    // 현재 일자 취득 (yyyy-MM-dd)
    public static String getDate() {
        return dateTimeNow(YYYY_MM_DD);
    }

    // 현재 시간 취득 (yyyy-MM-dd HH:mm:ss)
    public static String getTime() {
        return dateTimeNow(YYYY_MM_DD_HH_MM_SS);
    }

    public static String dateTimeNow(final String format) {
        return new SimpleDateFormat(format).format(new Date());
    }
}
```

---

### 1-10-3. 참고자료

- `com.valuesplay.common.utils.StringUtils`
- `com.valuesplay.common.utils.DateUtils`

## 1-11. 다국어 처리

### 1-11-1. 개요

글로벌 비즈니스를 지원하기 위해 에러 메시지나 화면의 UI 명칭들을 데이터베이스(RDB)에 관리하고 사용자 로케일(Locale) 환경에 따라 다국어로 다이내믹하게 조회하는 메시지 리소스 소스 모듈입니다.

#### 1-11-1-1. 주요 다국어 메커니즘
- `DatabaseMessageSource`: Spring의 `AbstractMessageSource`를 상속하여 DB(`TextService / Text`)를 조회하고 로케일에 맞춰 메시지 문자열을 리턴합니다.
- `MessageUtils`: Spring Container 내의 `MessageSource` 빈을 정적 영역에서 래핑해 손쉽게 특정 코드의 번역 텍스트를 조회할 수 있게 돕습니다.

---

### 1-11-2. 코드 설명

#### 1-11-2-1. 데이터베이스 연계 다국어 메시지 소스 (`DatabaseMessageSource.java`)

```java
package com.valuesplay.common.utils;

import java.text.MessageFormat;
import java.util.Locale;
import org.springframework.context.support.AbstractMessageSource;
import org.springframework.stereotype.Component;
import com.valuesplay.project.db1.system.domain.Text;
import com.valuesplay.project.db1.system.service.TextService;
import lombok.RequiredArgsConstructor;

@RequiredArgsConstructor
@Component
public class DatabaseMessageSource extends AbstractMessageSource {

    private final TextService textService;
    private static final String DEFAULT_LOCALE_CODE = "ko";

    @Override
    protected MessageFormat resolveCode(String code, Locale locale) {
        // 로케일 언어코드에 부합하는 메시지 레코드 DB 조회
        Text message = textService.getMessageBundle(code, locale.getLanguage());
        if (message == null || StringUtils.isEmpty(message.getTextVlu())) {
            message = textService.getMessageBundle(code, DEFAULT_LOCALE_CODE);
        }
        return new MessageFormat(message.getTextVlu(), locale);
    }

    public String getMessage(String code) {
        return getMessage(code, new Object[0], DEFAULT_LOCALE_CODE);
    }
}
```

#### 1-11-2-2. 정적 다국어 번역 유틸리티 (`MessageUtils.java`)

```java
package com.valuesplay.common.utils;

import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import com.valuesplay.common.utils.spring.SpringUtils;

public class MessageUtils {
    public static String message(String code, Object... args) {
        MessageSource messageSource = SpringUtils.getBean(MessageSource.class);
        return messageSource.getMessage(code, args, LocaleContextHolder.getLocale());
    }
}
```

---

### 1-11-3. 참고자료

- `com.valuesplay.common.utils.DatabaseMessageSource`
- `com.valuesplay.common.utils.MessageUtils`

## 1-12. 즐겨찾기메뉴

### 1-12-1. 개요

사용자가 다수 업무 화면 중에서 신속한 업무 처리를 위하여 자주 접근하는 메뉴 목록을 북마크 형태로 저장하고 관리하는 기능입니다.

#### 1-12-1-1. 연동 구성 요소
- `MnuFavController`: 즐겨찾기 메뉴 추가, 삭제, 목록 조회를 처리하는 REST 컨트롤러입니다.
- `MnuFav`: 사용자 ID(`empId`), 라우터패스(`routePath`), 메뉴명(`menuTitle`) 등으로 구성된 즐겨찾기 정보 도메인 객체입니다.

---

### 1-12-2. 코드 설명

#### 1-12-2-1. 즐겨찾기 관리 컨트롤러 (`MnuFavController.java`)

```java
package com.valuesplay.project.db1.common;

@Tag(name = "즐겨찾기 메뉴 관리")
@RequestMapping("/mnuFav")
@RestController
@RequiredArgsConstructor
public class MnuFavController extends BaseController {

	private final MnuFavService mnuFavService;

	// 즐겨찾기 추가
	@PostMapping("/add")
	@Operation(summary = "즐겨찾기 메뉴 관리 - 즐겨찾기 메뉴 추가")
	public AjaxResult addMnuFav(@Validated @RequestBody MnuFav mnuFav) {
		mnuFav.setCrtnId(getUsername());
		mnuFav.setUpdtId(getUsername());
		return toAjax(mnuFavService.createMnuFav(mnuFav));
	}

	// 즐겨찾기 삭제
	@PostMapping("/del")
	@Operation(summary = "즐겨찾기 메뉴 관리 - 즐겨찾기 메뉴 삭제")
	public AjaxResult delMnuFav(@Validated @RequestBody MnuFav mnuFav) {
		return toAjax(mnuFavService.removeMnuFav(mnuFav));
	}

	// 즐겨찾기 목록 조회
	@GetMapping("/list")
	@Operation(summary = "즐겨찾기 메뉴 관리 - 즐겨찾기 메뉴 리스트 조회")
	public AjaxResult getGridDefList(MnuFav mnuFav) {
		return success(mnuFavService.getMnuFavList(mnuFav));
	}
}
```

---

### 1-12-3. 참고자료

- `com.valuesplay.project.db1.common.MnuFavController`
- `com.valuesplay.project.db1.common.domain.MnuFav`

## 1-13. 로그인

### 1-13-1. 개요

인증 및 보안 표준을 기반으로 계정의 타당성을 입증하고 권한/부서에 적절한 리소스 접근 토큰(JWT)과 개인 정보를 반환하는 모듈입니다.

#### 1-13-1-1. 주요 API
- `POST /login`: ID, 패스워드 검증 및 세션 등록 후 JWT 토큰 발행
- `POST /expire-token`: 세션 파기 및 사용자 로그아웃 처리
- `GET /info`: 현재 로그인한 사용자의 프로필 정보, 부여된 역할(roles), 자원 접근 권한(permissions) 목록 반환
- `GET /routers/{roleId}-{empDeptId}`: 사용자 권한 및 소속 부서에 부합하는 메뉴 라우터 정보 반환

---

### 1-13-2. 코드 설명

#### 1-13-2-1. 로그인 API 제어 (`LoginController.java` 일부)

```java
@Tag(name = "로그인관련")
@RestController
@RequiredArgsConstructor
public class LoginController {

    private final LoginTokenService loginTokenService;
    private final UserService userService;
    private final SysPermissionService permissionService;

    // 로그인 실행
    @PostMapping("/login")
    @Operation(summary = "로그인관련 - 로그인 토큰조회")
    public AjaxResult login(@RequestBody LoginBody loginBody) {
        return loginTokenService.login(loginBody);
    }

    // 로그인된 사원의 상세 프로필 및 권한 목록 조회
    @GetMapping("info")
    @Operation(summary = "로그인관련 - 사용자정보조회")
    public AjaxResult getInfo() throws Exception {
        LoginUser loginUser = securityUtils.getLoginUser();
        User user = loginUser.getUser();

        User userInfo = userService.getUserById(user.getEmpId(), user.getSysId());
        Set<String> roles = permissionService.getRolePermission(user);
        Set<String> permissions = permissionService.getMenuPermission(user);

        AjaxResult ajax = AjaxResult.success();
        ajax.put("user", userInfo);
        ajax.put("roles", roles);
        ajax.put("permissions", permissions);
        return ajax;
    }
}
```

---

### 1-13-3. 참고자료

- `com.valuesplay.project.db1.system.controller.LoginController`

## 1-14. 메뉴 관리

### 1-14-1. 개요

시스템에서 접근할 수 있는 각 화면 단위(Router), 기능 버튼, 보호 대상이 되는 API URI 등을 생성하고 상하 계층구조(Tree)로 조회 및 관리하는 모듈입니다.

#### 1-14-1-1. 주요 기능
- 전체 메뉴 목록 및 특정 메뉴 상세 조회
- 2단계 또는 3단계(depth) 레이아웃의 메뉴 구조 지원
- 메뉴의 추가, 변경 시 메뉴 이름 및 코드 고유성(Unique) 유효성 검증

---

### 1-14-2. 코드 설명

#### 1-14-2-1. 메뉴 관리 컨트롤러 (`MenuController.java` 일부)

```java
@Tag(name = "메뉴관리")
@RequestMapping("/system/menus")
@RestController
@RequiredArgsConstructor
public class MenuController extends BaseController {

    private final MenuService menuService;

    // 메뉴 트리 구조 조회
    @GetMapping("/treeselect")
    public AjaxResult getTreeselect(Menu menu) {
        List<Menu> menus = menuService.getMenuList(menu, getUserId(), getUsername());
        return success(menuService.buildMenuTreeSelect(menus));
    }

    // 신규 메뉴 등록
    @PostMapping
    public AjaxResult createMenu(@Validated @RequestBody Menu menu) {
        if (!menuService.checkMenuNameUnique(menu)) {
            return warn("메뉴 등록 실패: 이미 등록된 메뉴 명칭입니다.");
        }
        menu.setCrtnId(getUsername());
        return toAjax(menuService.createMenu(menu));
    }
}
```

---

### 1-14-3. 참고자료

- `com.valuesplay.project.db1.system.controller.MenuController`

## 1-15. 권한 관리

### 1-15-1. 개요

사용자의 역할(Role)에 부합하는 메뉴 접근 한도와 기능 호출 제어를 위한 데이터 범위를 연결하고 제어하는 권한 거버넌스 모듈입니다.

#### 1-15-1-1. 주요 기능
- 권한(Role) 생성, 수정 및 일괄 제거
- 특정 권한에 부여된 메뉴 리스트 연동 및 관리
- 사용자별 권한 맵핑 테이블(`tble_user_role`) 제어

---

### 1-15-2. 코드 설명

#### 1-15-2-1. 권한 관리 컨트롤러 (`RoleController.java` 일부)

```java
@RequestMapping("/system/roles")
@RestController
@RequiredArgsConstructor
public class RoleController extends BaseController {

    private final RoleService roleService;

    // 권한 목록 페이징 조회
    @GetMapping("/lists")
    public TableDataInfo getRoleList(Role role) {
        startPage(); // 페이징 적용
        List<Role> list = roleService.getRoleList(role);
        return getDataTable(list);
    }

    // 신규 권한 등록
    @PostMapping
    public AjaxResult createRole(@Validated @RequestBody Role role) {
        if (!roleService.checkRoleNameUnique(role)) {
            return error("권한 등록 실패: 이미 존재하는 권한 명칭입니다.");
        }
        role.setCrtnId(getUsername());
        return toAjax(roleService.createRole(role));
    }
}
```

---

### 1-15-3. 참고자료

- `com.valuesplay.project.db1.system.controller.RoleController`

## 1-16. 사용자 관리

### 1-16-1. 개요

조직의 임직원 데이터를 기반으로 각 시스템에 로그인하여 업무를 수행할 수 있는 계정 정보를 구성하고 관리하는 관리형 모듈입니다.

#### 1-16-1-1. 주요 기능
- 사용자 정보 조회, 신규 생성, 회원 정보 수정 및 논리적/물리적 탈퇴 처리
- 개인 프로필 상세 설정(비밀번호 재설정, 아바타 이미지 등록 등)을 처리하는 `ProfileController` 지원

---

### 1-16-2. 코드 설명

#### 1-16-2-1. 사용자 관리 API 컨트롤러 (`UserController.java` 일부)

```java
@RequestMapping("/system/users")
@RestController
@RequiredArgsConstructor
public class UserController extends BaseController {

    private final UserService userService;

    // 사용자 목록 페이징 조회 (MyBatis PageHelper 연계)
    @GetMapping("/lists")
    public TableDataInfo getUserList(User user) {
        startPage();
        List<User> list = userService.getUserList(user);
        return getDataTable(list);
    }

    // 사용자 상세 정보 조회
    @GetMapping("/{empId}/{sysId}")
    public AjaxResult getUserById(@PathVariable("empId") Long empId, @PathVariable("sysId") Long sysId) {
        return success(userService.getUserById(empId, sysId));
    }
}
```

---

### 1-16-3. 참고자료

- `com.valuesplay.project.db1.system.controller.UserController`
- `com.valuesplay.project.db1.system.controller.ProfileController`

## 1-17. 공통코드 관리

### 1-17-1. 개요

자주 변경되지 않는 고정식 항목 구분 코드(예: 상태, 상태구분, 권한구분 등)의 대분류(Type) 및 개별 세부 데이터(Data)를 정의하여 전역 비즈니스 로직에 캐싱 연계하여 호출하는 기준 관리 모듈입니다.

#### 1-17-1-1. 주요 API 및 역할
- `CodeController` (`/system/dicts/type`): 공통 코드의 카테고리(대분류 키) 성격인 유형 정보 생성/조회 관리
- `CodeDataController` (`/system/dicts/data`): 대분류 카테고리 하위에 속하는 실제 매핑 데이터(상세 값, 한글명, 노출 우선순위 등) 관리

---

### 1-17-2. 코드 설명

#### 1-17-2-1. 공통코드 데이터 조회 및 캐싱 연계 (`CodeDataController.java` 일부)

```java
@RequestMapping("/system/dicts/data")
@RestController
@RequiredArgsConstructor
public class CodeDataController extends BaseController {

    private final CodeDataService codeDataService;

    // 대분류 구분 키(codeKy)에 매핑된 공통 상세코드 리스트 조회
    @GetMapping(value = "/type/{codeKy}")
    public AjaxResult getCodeDataByKy(@PathVariable("codeKy") String codeKy) {
        List<CodeData> data = codeDataService.getCodeDataByKy(codeKy);
        if (data == null) {
            data = new ArrayList<>();
        }
        return success(data);
    }
}
```

---

### 1-17-3. 참고자료

- `com.valuesplay.project.db1.system.controller.CodeController`
- `com.valuesplay.project.db1.system.controller.CodeDataController`

