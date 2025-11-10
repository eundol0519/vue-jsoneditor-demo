# Vue 3 + JSONEditor + 동적 데이터 로더 데모

Vue 3와 JSONEditor를 활용하여 JSON 데이터를 실시간으로 편집하고 관리할 수 있는 웹 애플리케이션입니다.

## 🎯 주요 기능

- **JSONEditor 통합**: Tree, Form, Code 3가지 모드로 JSON 편집
- **동적 데이터 로더**: 드롭다운에서 선택한 JSON 파일 자동 로드
- **실시간 편집**: JSONEditor에서 수정 시 즉시 반영
- **데이터 관리**: JSON 크기 및 업데이트 시간 실시간 표시

## 🚀 빠른 시작

### 설치
```bash
npm install
```

### 개발 서버 실행
```bash
npm run dev
```

브라우저에서 `http://localhost:5173`으로 접속하세요.

### 프로덕션 빌드
```bash
npm run build
```

## 📁 프로젝트 구조

```
vue-jsoneditor-demo/
├── src/
│   ├── main.js           # Vue 앱 진입점
│   └── App.vue           # 메인 컴포넌트 (JSONEditor 로직)
├── data/
│   ├── dkant.aiCamObjectType.json
│   ├── dkant.aiCamObjectTypeOption.json
│   └── dkant.pageOption.json
├── index.html            # HTML 진입점
├── style.css             # 전역 스타일
├── vite.config.js        # Vite 설정
├── package.json          # 의존성 관리
└── README.md             # 문서
```

## 🛠️ 기술 스택

- **Frontend Framework**: Vue 3
- **JSON Editor**: JSONEditor (josdejong/jsoneditor)
- **Build Tool**: Vite
- **Data Loading**: Fetch API

## ✨ 사용 방법

1. **데이터 선택**: 드롭다운에서 테스트 데이터 선택
   - AI 카메라 객체 타입
   - AI 카메라 객체 타입 옵션
   - 페이지 옵션

2. **모드 선택**: JSONEditor 상단에서 모드 전환
   - **Tree 모드**: 구조화된 트리 뷰
   - **Form 모드**: 폼 입력
   - **Code 모드**: JSON 직접 편집

3. **데이터 편집**: 선택한 모드에서 자유롭게 편집

4. **초기화**: 🗑️ 버튼으로 데이터 초기화

## 📊 JSONEditor의 장점

✅ **구조화된 편집**: 트리 뷰로 직관적인 JSON 관리  
✅ **다중 모드**: Tree, Form, Code 중 선택  
✅ **실시간 검증**: JSON 형식 자동 검증  
✅ **사용자 친화적**: 마우스로 데이터 추가/삭제 가능  

## 🔗 참고 자료

- [JSONEditor GitHub](https://github.com/josdejong/jsoneditor)
- [Vue 3 Documentation](https://vuejs.org/)
- [Vite Documentation](https://vitejs.dev/)

## 📝 라이선스

이 프로젝트는 학습 및 데모 목적입니다.

## 💡 팁

- JSONEditor에서 직접 JSON을 수정할 수 있습니다
- 드롭다운으로 다양한 테스트 데이터를 빠르게 로드할 수 있습니다
- Code 모드에서 복잡한 JSON을 한눈에 볼 수 있습니다
