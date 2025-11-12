# 📚 Page Option Editor - 기능 정의서

## 📌 개요

### 프로젝트 정보
- **프로젝트명**: Page Option Editor
- **버전**: 1.0.0
- **작성일**: 2024
- **기술 스택**: Vue 3 (Composition API), Element Plus, Vite

### 목적
`pageOption.json` 데이터를 JSON 에디터 라이브러리 없이 직관적인 UI로 편집할 수 있는 웹 애플리케이션

### 주요 특징
- ✅ 카드 기반 UI로 Feature와 Option을 시각적으로 관리
- ✅ 템플릿 시스템으로 재사용 가능한 Option 관리
- ✅ 실시간 데이터 검증 및 에러 방지
- ✅ LocalStorage 기반 커스텀 템플릿 영구 저장
- ✅ 변경사항 추적 및 되돌리기 기능

---

## 🎯 핵심 기능

### 1. Feature 관리

#### 1.1 Feature 목록 표시
**설명**: 모든 Feature를 카드 형태로 목록에 표시

**구성 요소**:
- Feature ID (대문자로 표시)
- Option 개수 표시
- 삭제 버튼 (🗑️)
- 선택 상태 표시 (보라색 그라데이션)

**동작**:
- 카드 클릭 시 해당 Feature의 Option 목록 표시
- 선택된 카드는 보라색으로 하이라이트
- hover 시 그림자 효과

#### 1.2 Feature 추가
**설명**: 새로운 Feature를 추가

**입력 요구사항**:
- Feature ID: 대문자로만 구성 (예: FEATURE001)
- 자동 대문자 변환 적용

**검증 규칙**:
- 중복 ID 체크
- 빈 값 입력 방지
- 특수문자 제한

**동작**:
1. "➕ Feature 추가" 버튼 클릭
2. ElMessageBox.prompt로 입력 모달 표시
3. Feature ID 입력
4. 중복 검사
5. Feature 추가 및 성공 메시지 표시

#### 1.3 Feature 삭제
**설명**: 선택한 Feature를 삭제

**검증**:
- 삭제 확인 다이얼로그 표시
- 관련된 모든 Option도 함께 삭제 경고

**동작**:
1. 🗑️ 버튼 클릭
2. 확인 다이얼로그 표시
3. 삭제 실행
4. 성공 메시지 및 UI 업데이트

---

### 2. Option 관리

#### 2.1 Option 목록 표시
**설명**: 선택된 Feature의 모든 Option을 카드로 표시

**구성 요소**:
- Option Key
- 설명 (desc)
- 현재 값 (value)
- 삭제 버튼 (🗑️)
- 선택 상태 표시 (핑크색 그라데이션)

#### 2.2 템플릿에서 Option 추가
**설명**: 미리 정의된 템플릿을 선택하여 Option 추가

**특징**:
- 이미 추가된 Option은 자동으로 목록에서 제외
- 검색 기능으로 원하는 템플릿 빠르게 찾기
- 템플릿 상세 정보 미리보기

**동작**:
1. "➕ 템플릿에서 추가" 버튼 클릭
2. 템플릿 선택 모달 표시
3. 검색어 입력 (선택 사항)
4. 템플릿 선택
5. Option 추가 및 성공 메시지

**중복 방지**:
- 현재 Feature에 이미 존재하는 Option Key는 템플릿 목록에서 자동 제외
- 모든 옵션이 추가된 경우 안내 메시지 표시

#### 2.3 Option 삭제
**설명**: 선택한 Option을 삭제

**동작**:
1. 🗑️ 버튼 클릭 (이벤트 버블링 방지)
2. 확인 다이얼로그 표시
3. Option 삭제
4. 성공 메시지

---

### 3. Option 상세 편집

#### 3.1 편집 모드
**설명**: 임시 저장 방식의 안전한 편집 시스템

**특징**:
- 실시간 반영이 아닌 **명시적 저장** 방식
- 변경사항 추적 및 경고
- 편집 취소 기능

#### 3.2 편집 가능 필드

##### Option Key
- 입력 필드로 수정 가능
- blur 이벤트로 변경 감지
- 중복 검사 및 성공 메시지

##### 설명 (desc)
- Textarea로 여러 줄 입력 가능
- 입력 시 변경사항 플래그 활성화

##### 값 (value)
- 텍스트 입력 필드
- 입력 시 변경사항 플래그 활성화

#### 3.3 읽기 전용 필드

##### 리스트 항목 (list)
- **편집 불가** (템플릿 관리에서만 수정 가능)
- 회색 배경으로 읽기 전용 표시
- 안내 메시지: "템플릿 관리에서 수정하세요"

**구조**:
```json
{
  "listValue": "값",
  "listDesc": "설명"
}
```

#### 3.4 저장/초기화 버튼

##### 💾 Option 저장
- 변경사항이 있을 때만 활성화
- 클릭 시 실제 데이터에 반영
- 성공 메시지 표시

##### ↺ Option 초기화
- 변경사항이 있을 때만 활성화
- 편집 내용을 원본으로 복원
- 정보 메시지 표시

#### 3.5 변경사항 경고
**상황 1**: 저장하지 않고 다른 Option 선택
- 경고 다이얼로그 표시
- "예": 변경사항 버리고 이동
- "아니오": 현재 Option에 머무름

**상황 2**: 저장하지 않고 다른 Feature 선택
- 동일한 경고 메커니즘

---

### 4. 템플릿 관리

#### 4.1 템플릿 라이브러리
**설명**: 모든 사용 가능한 Option 템플릿을 관리하는 중앙 저장소

**템플릿 소스**:
1. **시스템 템플릿**: 기존 pageOption.json에서 자동 수집
2. **커스텀 템플릿**: 사용자가 생성한 템플릿 (localStorage 저장)

**표시 정보**:
- 템플릿 Key
- 설명
- 기본값
- 리스트 항목 개수
- 사용 중인 Feature 개수
- 사용 위치 (Feature ID 목록)

#### 4.2 아코디언 UI
**설명**: 템플릿을 접었다 펼칠 수 있는 인터페이스

**동작**:
- 헤더 클릭 시 펼치기/접기
- ▶ / ▼ 아이콘으로 상태 표시
- slide-down 애니메이션

**펼친 상태에서 표시**:
- 설명
- 기본값
- 리스트 항목 전체 (listValue, listDesc)
- 사용 위치 태그

#### 4.3 템플릿 검색
**기능**: 템플릿 Key와 설명에서 검색

**동작**:
- 실시간 필터링
- 대소문자 구분 없음
- 검색 결과 즉시 반영

#### 4.4 새 템플릿 만들기

**입력 필드**:
1. **템플릿 Key** (필수)
   - 영문, 숫자, 언더스코어만 허용
   - 정규식: `^[a-zA-Z0-9_]+$`
   - 중복 불가

2. **설명** (필수)
   - 자유 텍스트
   - 최소 1자 이상

3. **리스트 항목** (최소 1개 필수)
   - listValue (필수)
   - listDesc (필수)
   - 항목이 1개만 있을 때 삭제 버튼 비활성화

4. **기본값** (필수)
   - 드롭다운에서 리스트 항목 중 선택
   - 리스트 항목이 없으면 비활성화

**검증 규칙**:
- ❌ Key 누락: "템플릿 Key를 입력해주세요"
- ❌ 잘못된 Key 형식: "영문, 숫자, 언더스코어만 가능합니다"
- ❌ 설명 누락: "템플릿 설명을 입력해주세요"
- ❌ 리스트 항목 없음: "리스트 항목을 최소 1개 이상 추가해주세요"
- ❌ 리스트 항목 빈 값: "모든 리스트 항목의 값과 설명을 입력해주세요"
- ❌ 기본값 미선택: "기본값을 선택해주세요"
- ❌ Key 중복: "이미 존재하는 템플릿 Key입니다"

**저장**:
- localStorage에 `customTemplates` 키로 저장
- JSON.stringify로 직렬화

#### 4.5 템플릿 수정

**기능**: 기존 템플릿의 내용을 수정

**제약**:
- 템플릿 Key는 수정 불가 (비활성화)
- 나머지 필드는 생성과 동일한 검증

**동작**:
1. ✏️ 버튼 클릭
2. 수정 모달에 기존 값 pre-fill
3. 설명, 리스트 항목, 기본값 수정
4. 검증 통과 후 저장
5. **중요**: 해당 템플릿을 사용하는 모든 Feature의 Option도 자동 업데이트

**업데이트 전파**:
```javascript
// 모든 Feature를 순회하며 해당 템플릿 사용 Option 업데이트
pageOptions.forEach(feature => {
  if (feature.option[templateKey]) {
    feature.option[templateKey].desc = updatedTemplate.desc;
    feature.option[templateKey].list = updatedTemplate.list;
    // value는 유지 (사용자가 설정한 값)
  }
});
```

#### 4.6 템플릿 삭제

**기능**: 템플릿을 삭제하고 관련 Option도 제거

**경고**:
- 사용 중인 템플릿 삭제 시: "현재 N개의 Feature에서 사용 중입니다"
- 삭제 시 관련 Option도 함께 삭제됨을 명시

**동작**:
1. 🗑️ 버튼 클릭
2. 확인 다이얼로그 (z-index: 3000으로 모달 위에 표시)
3. 커스텀 템플릿이면 localStorage에서 제거
4. 모든 Feature의 해당 Option 제거
5. 성공 메시지

---

### 5. 데이터 관리

#### 5.1 데이터 구조
```json
{
  "featureId": "FEATURE_NAME",
  "option": {
    "optionKey": {
      "desc": "설명",
      "value": "현재값",
      "list": [
        {
          "listValue": "값1",
          "listDesc": "설명1"
        }
      ]
    }
  }
}
```

#### 5.2 변경사항 저장
**설명**: 현재 편집 중인 데이터를 originalData에 반영

**동작**:
1. "💾 변경사항 저장" 버튼 클릭
2. 확인 다이얼로그
3. `originalData.value = JSON.parse(JSON.stringify(pageOptions.value))`
4. 성공 메시지

#### 5.3 초기화
**설명**: 모든 변경사항을 취소하고 원본 데이터로 복원

**동작**:
1. "↺ 초기화" 버튼 클릭
2. 확인 다이얼로그
3. `pageOptions.value = JSON.parse(JSON.stringify(originalData.value))`
4. 선택 상태 초기화
5. 정보 메시지

#### 5.4 JSON 내보내기
**설명**: 현재 데이터를 JSON 파일로 다운로드

**파일명**: `pageOption_export.json`

**동작**:
1. "📥 JSON 내보내기" 버튼 클릭
2. Blob 객체 생성
3. 다운로드 링크 생성 및 클릭
4. 성공 메시지

---

## 🎨 UI/UX 사양

### 레이아웃
- **3-패널 구조**: Feature 목록 | Option 목록 | Option 상세
- **패널 비율**: 30% | 30% | 40%
- **반응형**: 1200px 이하에서 세로 배치

### 색상 체계
- **Feature 선택**: 보라색 그라데이션 (#9333ea → #a855f7)
- **Option 선택**: 핑크색 그라데이션 (#ec4899 → #f472b6)
- **추가 버튼**: 녹색 (#10b981)
- **삭제 버튼**: 빨간색 (#ef4444)
- **저장 버튼**: 녹색 그라데이션 (#67c23a → #85ce61)
- **초기화 버튼**: 회색 그라데이션 (#909399 → #b3b8bd)

### 애니메이션
- **카드 hover**: transform: translateY(-2px), box-shadow 강화
- **모달**: slideIn 애니메이션
- **아코디언**: slide-down 트랜지션 (0.3s ease)
- **버튼**: hover 시 transform + shadow

### 타이포그래피
- **Feature ID**: 큰 볼드체
- **Option Key**: 중간 볼드체
- **설명**: 일반 텍스트
- **코드 값**: monospace 폰트

---

## 🔒 데이터 검증 및 보안

### 입력 검증
1. **Feature ID**: 대문자, 중복 체크
2. **Option Key**: 영문/숫자/언더스코어, 중복 체크
3. **Template Key**: 정규식 검증, 중복 체크
4. **필수 필드**: 빈 값 방지

### 에러 처리
- 모든 사용자 작업에 try-catch 적용
- ElMessage로 사용자 친화적 에러 메시지
- 콘솔 로깅으로 디버깅 지원

### 데이터 무결성
- 삭제 시 참조 무결성 확인
- 템플릿 업데이트 시 전파
- 변경사항 추적 및 되돌리기

---

## 💾 저장소

### LocalStorage
**키**: `customTemplates`

**구조**:
```json
[
  {
    "key": "myCustomOption",
    "desc": "커스텀 옵션 설명",
    "list": [...],
    "defaultValue": "값",
    "listCount": 3,
    "sampleList": [...]
  }
]
```

**동작**:
- 템플릿 생성/수정/삭제 시 자동 저장
- 페이지 로드 시 자동 로드
- 브라우저 저장 용량: 약 5MB

---

## 🔧 기술 스펙

### Vue 3 Composition API
- `ref`: 반응형 데이터
- `computed`: 파생 상태 (필터링, 선택된 Feature 등)
- `onMounted`: 초기 데이터 로드
- `watch`: 데이터 변경 감지 (선택 사항)

### Element Plus 컴포넌트
- `ElMessage`: 알림 메시지
- `ElMessageBox`: confirm, prompt 다이얼로그

### 주요 함수
- `selectFeature(index)`: Feature 선택
- `selectOption(key)`: Option 선택
- `addFeature()`: Feature 추가
- `deleteFeature(index)`: Feature 삭제
- `addOptionFromTemplate(template)`: 템플릿에서 Option 추가
- `deleteOption(key)`: Option 삭제
- `saveOptionChanges()`: Option 저장
- `resetOptionChanges()`: Option 초기화
- `createTemplate()`: 템플릿 생성
- `updateTemplate()`: 템플릿 수정
- `deleteTemplate(template)`: 템플릿 삭제
- `saveChanges()`: 전체 변경사항 저장
- `resetChanges()`: 전체 초기화
- `exportJSON()`: JSON 내보내기

---

## 📊 성능 고려사항

### 최적화
- 깊은 복사 최소화 (필요한 경우만 JSON.parse/stringify)
- computed로 파생 상태 캐싱
- 이벤트 버블링 방지 (삭제 버튼 등)

### 확장성
- Feature 100개, Option 50개/Feature까지 원활하게 동작
- 템플릿 검색은 O(n) 선형 탐색 (충분히 빠름)

---

## 🚀 향후 개선 사항

### 우선순위 높음
- [ ] Undo/Redo 기능
- [ ] Feature 순서 변경 (드래그앤드롭)
- [ ] Option 순서 변경
- [ ] 일괄 편집 기능

### 우선순위 중간
- [ ] 테마 변경 (다크 모드)
- [ ] 키보드 단축키
- [ ] JSON 가져오기
- [ ] 데이터 버전 관리

### 우선순위 낮음
- [ ] 사용자 설정 저장
- [ ] 다국어 지원
- [ ] 접근성 개선 (ARIA)

---

## 📞 지원 및 문의

**문제 보고**: GitHub Issues  
**문서 버전**: 1.0.0  
**최종 업데이트**: 2024

---

## 📄 변경 이력

### v1.0.0 (2024)
- ✅ 초기 릴리스
- ✅ Feature/Option 관리 기능
- ✅ 템플릿 시스템 구현
- ✅ Option 임시 저장 시스템
- ✅ 필수 입력 검증
- ✅ 중복 옵션 자동 필터링
- ✅ 아코디언 UI
- ✅ z-index 문제 해결

