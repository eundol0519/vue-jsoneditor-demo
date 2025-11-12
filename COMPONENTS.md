# 🧩 컴포넌트 구조 문서

## 📂 프로젝트 구조

```
src/
├── App.vue                      # 메인 앱 컨테이너
├── PageOptionEditor.vue         # 메인 에디터 페이지 (부모 컴포넌트)
└── components/
    ├── FeatureCard.vue          # Feature 카드 컴포넌트
    ├── OptionCard.vue           # Option 카드 컴포넌트
    └── OptionDetail.vue         # Option 상세 편집 컴포넌트
```

---

## 📦 컴포넌트 목록

### 1. FeatureCard.vue

**역할**: Feature를 카드 형태로 표시

**Props**:
- `feature` (Object, required): Feature 데이터 객체
  ```typescript
  {
    featureId: string,
    option: {
      [key: string]: {
        desc: string,
        value: string,
        list: Array
      }
    }
  }
  ```
- `isSelected` (Boolean, default: false): 선택 상태

**Emits**:
- `select`: 카드 클릭 시 발생
- `delete`: 삭제 버튼 클릭 시 발생

**사용 예시**:
```vue
<FeatureCard
  v-for="(feature, index) in pageOptions"
  :key="index"
  :feature="feature"
  :is-selected="selectedFeatureIndex === index"
  @select="selectFeature(index)"
  @delete="deleteFeature(index)"
/>
```

**스타일**:
- 기본: 흰색 배경, 회색 테두리
- Hover: 보라색 테두리, 그림자 효과
- Selected: 보라색 그라데이션 배경

---

### 2. OptionCard.vue

**역할**: Option을 카드 형태로 표시

**Props**:
- `optionKey` (String, required): Option의 키 값
- `option` (Object, required): Option 데이터 객체
  ```typescript
  {
    desc: string,
    value: string,
    list: Array<{
      listValue: string,
      listDesc: string
    }>
  }
  ```
- `isSelected` (Boolean, default: false): 선택 상태

**Emits**:
- `select`: 카드 클릭 시 발생
- `delete`: 삭제 버튼 클릭 시 발생

**사용 예시**:
```vue
<OptionCard
  v-for="(option, key) in selectedFeature.option"
  :key="key"
  :option-key="key"
  :option="option"
  :is-selected="selectedOptionKey === key"
  @select="selectOption(key)"
  @delete="deleteOption(key)"
/>
```

**스타일**:
- 기본: 흰색 배경, 회색 테두리
- Hover: 핑크색 테두리, 그림자 효과
- Selected: 핑크색 그라데이션 배경

---

### 3. OptionDetail.vue

**역할**: 선택된 Option의 상세 정보를 편집하는 폼

**Props**:
- `editingData` (Object, default: null): 편집 중인 Option 데이터 (임시 버퍼)
- `editingKey` (String, default: ''): 편집 중인 Option Key
- `hasChanges` (Boolean, default: false): 변경사항 존재 여부

**Emits**:
- `update:editingKey`: editingKey 값이 변경될 때 발생
- `update:editingData`: editingData 값이 변경될 때 발생
- `dataChange`: desc 또는 value가 변경될 때 발생
- `keyChange`: Option Key가 변경될 때 (blur 이벤트)
- `save`: "💾 Option 저장" 버튼 클릭 시
- `reset`: "↺ Option 초기화" 버튼 클릭 시

**사용 예시**:
```vue
<OptionDetail
  v-model:editingKey="editingOptionKey"
  v-model:editingData="editingOptionData"
  :has-changes="hasOptionChanges"
  @data-change="hasOptionChanges = true"
  @key-change="updateOptionKey"
  @save="saveOptionChanges"
  @reset="resetOptionChanges"
/>
```

**기능**:
- Option Key 수정 (입력 필드)
- 설명 (desc) 수정 (textarea)
- 현재 값 (value) 수정 (입력 필드)
- 리스트 항목 보기 (읽기 전용)
- 저장/초기화 버튼

**편집 모드**:
- 변경사항은 즉시 실제 데이터에 반영되지 **않음**
- 임시 버퍼 (`editingData`)에 저장
- "💾 Option 저장" 버튼 클릭 시 실제 데이터에 반영
- "↺ Option 초기화" 버튼으로 변경사항 취소 가능

---

## 🔄 데이터 흐름

### Feature 선택 흐름
```
사용자 클릭
  ↓
FeatureCard @select 이벤트 발생
  ↓
PageOptionEditor.selectFeature(index) 호출
  ↓
selectedFeatureIndex 업데이트
  ↓
computed selectedFeature 재계산
  ↓
OptionCard 목록 업데이트
```

### Option 선택 흐름
```
사용자 클릭
  ↓
OptionCard @select 이벤트 발생
  ↓
PageOptionEditor.selectOption(key) 호출
  ↓
변경사항이 있으면 경고 다이얼로그 표시
  ↓
loadOptionData(key) 호출
  ↓
editingOptionData에 깊은 복사
  ↓
OptionDetail 컴포넌트에 데이터 전달
```

### Option 편집 흐름
```
사용자가 desc/value 입력
  ↓
OptionDetail @data-change 이벤트 발생
  ↓
hasOptionChanges = true (버튼 활성화)
  ↓
"💾 Option 저장" 버튼 클릭
  ↓
OptionDetail @save 이벤트 발생
  ↓
PageOptionEditor.saveOptionChanges() 호출
  ↓
editingOptionData → pageOptions에 반영
  ↓
hasOptionChanges = false
```

### Option 삭제 흐름
```
사용자가 🗑️ 버튼 클릭
  ↓
@click.stop으로 이벤트 버블링 방지
  ↓
OptionCard @delete 이벤트 발생
  ↓
PageOptionEditor.deleteOption(key) 호출
  ↓
ElMessageBox.confirm으로 확인
  ↓
delete feature.option[key]
  ↓
성공 메시지 표시
```

---

## 🎨 컴포넌트 디자인 패턴

### 1. Presentation 컴포넌트
**FeatureCard, OptionCard**는 순수한 프레젠테이션 컴포넌트입니다:
- ✅ Props로만 데이터 수신
- ✅ Emit으로만 이벤트 전달
- ✅ 내부 상태 없음 (stateless)
- ✅ 재사용 가능
- ✅ 테스트 용이

### 2. Container 컴포넌트
**PageOptionEditor**는 컨테이너 컴포넌트입니다:
- ✅ 전체 상태 관리
- ✅ 비즈니스 로직 처리
- ✅ API 호출 (데이터 로드)
- ✅ 자식 컴포넌트 조율

### 3. Form 컴포넌트
**OptionDetail**은 폼 컴포넌트입니다:
- ✅ v-model을 통한 양방향 바인딩
- ✅ 폼 검증 로직
- ✅ 편집 상태 관리

---

## 🔧 컴포넌트 재사용 가이드

### FeatureCard를 다른 곳에서 사용하기
```vue
<template>
  <div class="my-feature-list">
    <FeatureCard
      v-for="feature in features"
      :key="feature.id"
      :feature="feature"
      :is-selected="feature.id === currentId"
      @select="handleSelect(feature)"
      @delete="handleDelete(feature)"
    />
  </div>
</template>

<script setup>
import FeatureCard from '@/components/FeatureCard.vue';

const features = ref([...]);
const currentId = ref(null);

const handleSelect = (feature) => {
  currentId.value = feature.id;
  // 추가 로직...
};

const handleDelete = (feature) => {
  // 삭제 로직...
};
</script>
```

### OptionCard를 다른 곳에서 사용하기
```vue
<template>
  <div class="my-option-list">
    <OptionCard
      v-for="[key, option] in Object.entries(options)"
      :key="key"
      :option-key="key"
      :option="option"
      @select="handleSelect(key)"
    />
  </div>
</template>

<script setup>
import OptionCard from '@/components/OptionCard.vue';

const options = ref({...});

const handleSelect = (key) => {
  // 선택 로직...
};
</script>
```

---

## 🚀 컴포넌트 확장 가능성

### 추가 가능한 컴포넌트

1. **TemplateModal.vue** (템플릿 선택 모달)
   - Props: `templates`, `show`
   - Emits: `select`, `close`

2. **TemplateManagerModal.vue** (템플릿 관리 모달)
   - Props: `templates`, `show`
   - Emits: `create`, `update`, `delete`, `close`

3. **TemplateFormModal.vue** (템플릿 생성/수정 폼)
   - Props: `template`, `show`, `mode`
   - Emits: `submit`, `cancel`

4. **ConfirmDialog.vue** (재사용 가능한 확인 다이얼로그)
   - Props: `message`, `show`
   - Emits: `confirm`, `cancel`

5. **FeatureList.vue** (Feature 목록 래퍼)
   - Props: `features`, `selectedIndex`
   - Emits: `select`, `add`, `delete`

6. **OptionList.vue** (Option 목록 래퍼)
   - Props: `options`, `selectedKey`
   - Emits: `select`, `add`, `delete`

---

## 📝 컴포넌트 네이밍 규칙

### 파일명
- **PascalCase** 사용: `FeatureCard.vue`
- 역할이 명확하게 드러나도록 작성
- 단일 책임 원칙 준수

### 컴포넌트 이름
```vue
<!-- Good -->
<FeatureCard />
<OptionDetail />
<TemplateModal />

<!-- Bad -->
<feature-card />  <!-- kebab-case는 템플릿 내에서만 -->
<Card />          <!-- 너무 일반적 -->
<MyComponent />   <!-- 모호함 -->
```

### Props 이름
```typescript
// Good
props: {
  isSelected: Boolean,
  featureId: String,
  optionKey: String,
  hasChanges: Boolean
}

// Bad
props: {
  selected: Boolean,  // 명확하지 않음
  id: String,         // 모호함
  key: String,        // 예약어
  changed: Boolean    // 명확하지 않음
}
```

---

## 🎯 Best Practices

### 1. Props는 불변으로 취급
```vue
<!-- Bad -->
<script setup>
const props = defineProps(['data']);
props.data.value = 'new value';  // ❌ Props 직접 변경
</script>

<!-- Good -->
<script setup>
const props = defineProps(['data']);
const localData = ref(JSON.parse(JSON.stringify(props.data)));
localData.value.value = 'new value';  // ✅ 로컬 복사본 변경
emit('update:data', localData.value);
</script>
```

### 2. 이벤트는 명확하게 명명
```vue
<!-- Good -->
emit('select', feature);
emit('delete', optionKey);
emit('save');
emit('update:editingData', newData);

<!-- Bad -->
emit('click');  // 모호함
emit('action'); // 무엇을 하는지 불명확
emit('do');     // 너무 일반적
```

### 3. 스타일은 scoped 사용
```vue
<style scoped>
.feature-card {
  /* 이 컴포넌트에만 적용 */
}
</style>
```

### 4. 컴포넌트 크기 제한
- 한 컴포넌트는 **200줄 이하** 권장
- 템플릿, 스크립트, 스타일 각각 관리 가능한 크기 유지
- 복잡해지면 더 작은 컴포넌트로 분리

---

## 🧪 테스트 가이드

### 단위 테스트 예시 (FeatureCard)
```typescript
import { mount } from '@vue/test-utils';
import FeatureCard from '@/components/FeatureCard.vue';

describe('FeatureCard', () => {
  it('renders feature ID correctly', () => {
    const feature = {
      featureId: 'TEST001',
      option: {}
    };
    const wrapper = mount(FeatureCard, {
      props: { feature, isSelected: false }
    });
    expect(wrapper.text()).toContain('TEST001');
  });

  it('emits select event when clicked', async () => {
    const feature = { featureId: 'TEST001', option: {} };
    const wrapper = mount(FeatureCard, {
      props: { feature, isSelected: false }
    });
    await wrapper.trigger('click');
    expect(wrapper.emitted('select')).toBeTruthy();
  });

  it('applies selected class when isSelected is true', () => {
    const feature = { featureId: 'TEST001', option: {} };
    const wrapper = mount(FeatureCard, {
      props: { feature, isSelected: true }
    });
    expect(wrapper.classes()).toContain('selected');
  });
});
```

---

## 📚 참고 자료

- [Vue 3 Composition API 공식 문서](https://vuejs.org/guide/extras/composition-api-faq.html)
- [Vue 3 컴포넌트 Props 문서](https://vuejs.org/guide/components/props.html)
- [Vue 3 Events 문서](https://vuejs.org/guide/components/events.html)
- [Element Plus 공식 문서](https://element-plus.org/)

---

**작성일**: 2024  
**버전**: 1.0.0

