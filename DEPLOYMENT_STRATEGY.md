# 🚀 지자체별 PageOption 배포 전략

## 📋 목차
1. [문제 정의](#-문제-정의)
2. [핵심 요구사항](#-핵심-요구사항)
3. [해결책: 3-Way Merge 시스템](#-해결책-3-way-merge-시스템)
4. [구현 방안](#-구현-방안)
5. [사용자 워크플로우](#-사용자-워크플로우)
6. [API 명세](#-api-명세)
7. [배포 시나리오 예시](#-배포-시나리오-예시)

---

## 🎯 문제 정의

### 현재 상황
- 지자체마다 독립적인 pageOption DB 보유
- 중앙에서 수정 발생 시 각 지자체에 수동 포팅 필요
- 누락/실수 가능성 높음
- 시간 소모 큼

### 핵심 문제
**지자체마다 커스터마이징이 되어 있어서 무조건 덮어쓸 수 없음!**

예시:
- 서울시: FEATURE001의 option1을 자체적으로 수정함
- 부산시: FEATURE002를 추가함
- 대구시: 기본 템플릿 그대로 사용 중

→ 중앙에서 FEATURE001을 수정했다면?
- 서울시: 충돌 발생 (양쪽 다 수정)
- 부산시: 안전하게 적용 가능
- 대구시: 안전하게 적용 가능

---

## 🎯 핵심 요구사항

### 1. 선택적 적용
- ✅ 새로 추가된 것만 자동 적용
- ✅ 지자체가 수정하지 않은 것만 업데이트
- ⚠️ 충돌 발생 시 사용자가 선택

### 2. 안전성
- 지자체 커스터마이징 보존
- 변경사항 미리보기
- 롤백 가능

### 3. 효율성
- 한 번에 여러 지자체 처리
- 자동/반자동 처리
- 명확한 리포트

---

## 💡 해결책: 3-Way Merge 시스템

### 개념

```
베이스 버전 (Base)
    ↓
┌───────────────────────────┐
│   중앙 마스터 (Master)    │ ← 중앙에서 수정
└───────────────────────────┘
            ↓
    [변경사항 분석]
            ↓
┌───────────────────────────┐
│   지자체 데이터 (City)    │ ← 지자체가 수정
└───────────────────────────┘
            ↓
    [충돌 감지 & 해결]
            ↓
      [선택적 적용]
```

### 변경사항 타입 분류

| 타입 | 설명 | 처리 방법 |
|------|------|----------|
| **NEW_FEATURE** | 완전히 새로운 Feature 추가 | ✅ 자동 적용 |
| **NEW_OPTION** | 완전히 새로운 Option 추가 | ✅ 자동 적용 |
| **MASTER_UPDATE** | 중앙만 수정, 지자체는 그대로 | ✅ 자동 적용 |
| **CITY_CUSTOM** | 지자체만 수정, 중앙은 그대로 | ℹ️ 유지 |
| **CONFLICT** | 양쪽 다 수정 | ⚠️ 사용자 선택 필요 |

---

## 🔧 구현 방안

### 1. 변경사항 분석 엔진

```javascript
/**
 * 3-Way 머지 분석
 * @param {Array} baseVersion - 베이스 버전 (마지막 동기화 시점)
 * @param {Array} newMaster - 현재 중앙 마스터
 * @param {Array} cityData - 지자체 현재 데이터
 * @returns {Array} 변경사항 목록
 */
const analyzeChanges = (baseVersion, newMaster, cityData) => {
  const changes = [];
  
  // 1. Feature 레벨 체크
  newMaster.forEach(feature => {
    const baseFeature = baseVersion.find(f => f.featureId === feature.featureId);
    const cityFeature = cityData.find(f => f.featureId === feature.featureId);
    
    // Case 1: 완전히 새로운 Feature
    if (!baseFeature && !cityFeature) {
      changes.push({
        type: 'NEW_FEATURE',
        action: 'ADD',
        target: feature.featureId,
        data: feature,
        safe: true,
        autoApply: true
      });
      return;
    }
    
    // Case 2: Feature는 존재, Option 레벨 체크
    if (baseFeature && cityFeature) {
      Object.keys(feature.option).forEach(optionKey => {
        const baseOption = baseFeature.option?.[optionKey];
        const masterOption = feature.option[optionKey];
        const cityOption = cityFeature.option?.[optionKey];
        
        // Case 2-1: 완전히 새로운 Option
        if (!baseOption && !cityOption) {
          changes.push({
            type: 'NEW_OPTION',
            action: 'ADD',
            target: `${feature.featureId} > ${optionKey}`,
            data: masterOption,
            safe: true,
            autoApply: true
          });
        }
        
        // Case 2-2: Option 존재, 수정 여부 체크
        else if (baseOption && cityOption) {
          const masterModified = JSON.stringify(baseOption) !== JSON.stringify(masterOption);
          const cityModified = JSON.stringify(baseOption) !== JSON.stringify(cityOption);
          
          // 충돌: 양쪽 다 수정
          if (masterModified && cityModified) {
            changes.push({
              type: 'CONFLICT',
              action: 'MERGE_REQUIRED',
              target: `${feature.featureId} > ${optionKey}`,
              base: baseOption,
              master: masterOption,
              city: cityOption,
              safe: false,
              autoApply: false,
              needsReview: true
            });
          }
          
          // 중앙만 수정: 안전하게 업데이트
          else if (masterModified && !cityModified) {
            changes.push({
              type: 'MASTER_UPDATE',
              action: 'UPDATE',
              target: `${feature.featureId} > ${optionKey}`,
              old: baseOption,
              new: masterOption,
              safe: true,
              autoApply: true,
              diff: calculateDiff(baseOption, masterOption)
            });
          }
          
          // 지자체만 수정: 유지
          else if (!masterModified && cityModified) {
            changes.push({
              type: 'CITY_CUSTOM',
              action: 'KEEP',
              target: `${feature.featureId} > ${optionKey}`,
              data: cityOption,
              safe: true,
              autoApply: false,
              note: '지자체 커스터마이징 유지'
            });
          }
        }
      });
    }
  });
  
  return changes;
};

/**
 * Diff 계산
 */
const calculateDiff = (oldObj, newObj) => {
  const diff = {};
  
  Object.keys(newObj).forEach(key => {
    if (JSON.stringify(oldObj[key]) !== JSON.stringify(newObj[key])) {
      diff[key] = {
        old: oldObj[key],
        new: newObj[key]
      };
    }
  });
  
  return diff;
};
```

---

### 2. UI 컴포넌트

#### 2.1 메인 화면 구조

```vue
<template>
  <div class="smart-merge-system">
    <!-- 헤더 -->
    <div class="header">
      <h2>🔄 중앙 업데이트 → 지자체 배포</h2>
      <p>변경사항을 분석하고 안전하게 적용합니다</p>
    </div>
    
    <!-- 지자체 선택 -->
    <div class="city-selector">
      <label>대상 지자체:</label>
      <select v-model="selectedCity" @change="analyzeForCity">
        <option v-for="city in cities" :key="city.id" :value="city">
          {{ city.name }}
        </option>
      </select>
    </div>

    <!-- 분석 결과 -->
    <div v-if="analysisResult" class="analysis-result">
      <!-- 요약 카드 -->
      <div class="summary-cards">
        <div class="summary-card safe">
          <h4>✅ 안전 적용 가능</h4>
          <p class="count">{{ safeChanges.length }}건</p>
          <p class="description">자동으로 적용 가능한 변경사항</p>
          <button @click="autoApplyAll" class="btn btn-success">
            자동 적용
          </button>
        </div>
        
        <div class="summary-card warning">
          <h4>⚠️ 검토 필요</h4>
          <p class="count">{{ conflictChanges.length }}건</p>
          <p class="description">충돌이 발생한 항목</p>
          <button @click="reviewConflicts" class="btn btn-warning">
            검토하기
          </button>
        </div>
        
        <div class="summary-card info">
          <h4>ℹ️ 지자체 커스텀</h4>
          <p class="count">{{ customChanges.length }}건</p>
          <p class="description">지자체 고유 설정</p>
          <button @click="viewCustoms" class="btn btn-info">
            보기
          </button>
        </div>
      </div>

      <!-- 필터 -->
      <div class="filters">
        <label>
          <input type="checkbox" v-model="showSafe" checked />
          안전 적용 가능
        </label>
        <label>
          <input type="checkbox" v-model="showConflicts" checked />
          충돌
        </label>
        <label>
          <input type="checkbox" v-model="showCustoms" />
          지자체 커스텀
        </label>
      </div>

      <!-- 변경사항 리스트 -->
      <div class="change-list">
        <ChangeItem
          v-for="change in filteredChanges"
          :key="change.id"
          :change="change"
          @resolve="resolveConflict"
          @toggle-expand="toggleExpand"
        />
      </div>

      <!-- 액션 바 -->
      <div class="action-bar">
        <div class="selection-summary">
          선택된 변경사항: <strong>{{ selectedChanges.length }}건</strong>
        </div>
        <div class="action-buttons">
          <button 
            @click="previewChanges"
            :disabled="selectedChanges.length === 0"
            class="btn btn-secondary"
          >
            📋 미리보기
          </button>
          <button 
            @click="applyChanges"
            :disabled="selectedChanges.length === 0 || hasUnresolvedConflicts"
            class="btn btn-primary"
          >
            ✅ {{ selectedCity.name }}에 적용
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
```

#### 2.2 ChangeItem 컴포넌트 (개별 변경사항)

```vue
<template>
  <div 
    class="change-item"
    :class="[change.type.toLowerCase(), { expanded: change.expanded }]"
  >
    <!-- 헤더 -->
    <div class="change-header" @click="$emit('toggle-expand', change)">
      <div class="change-icon">{{ getIcon(change.type) }}</div>
      <div class="change-info">
        <strong class="change-target">{{ change.target }}</strong>
        <span class="change-type-label">{{ getTypeLabel(change.type) }}</span>
      </div>
      <div class="change-actions" @click.stop>
        <input 
          type="checkbox" 
          v-model="change.selected"
          :disabled="!change.safe && !change.resolved"
        />
        <button 
          v-if="change.type === 'CONFLICT'"
          @click="$emit('resolve', change)"
          class="btn-resolve"
        >
          해결하기
        </button>
      </div>
    </div>

    <!-- 상세 내용 (펼쳤을 때만) -->
    <div v-if="change.expanded" class="change-body">
      <!-- NEW_FEATURE / NEW_OPTION -->
      <template v-if="change.type === 'NEW_FEATURE' || change.type === 'NEW_OPTION'">
        <div class="new-content">
          <h5>✨ 추가될 내용:</h5>
          <CodeBlock :code="change.data" />
        </div>
      </template>

      <!-- MASTER_UPDATE -->
      <template v-if="change.type === 'MASTER_UPDATE'">
        <div class="diff-view">
          <div class="diff-column old">
            <h5>❌ 현재 (기존)</h5>
            <CodeBlock :code="change.old" />
          </div>
          <div class="diff-arrow">→</div>
          <div class="diff-column new">
            <h5>✅ 변경 후 (중앙)</h5>
            <CodeBlock :code="change.new" />
          </div>
        </div>
        <div class="diff-summary">
          <h5>📝 변경사항 요약:</h5>
          <ul>
            <li v-for="(value, key) in change.diff" :key="key">
              <code>{{ key }}</code>: 
              <del class="old-value">{{ formatValue(value.old) }}</del> → 
              <ins class="new-value">{{ formatValue(value.new) }}</ins>
            </li>
          </ul>
        </div>
      </template>

      <!-- CONFLICT (충돌 해결) -->
      <template v-if="change.type === 'CONFLICT'">
        <ConflictResolver
          :change="change"
          @save="saveResolution"
        />
      </template>

      <!-- CITY_CUSTOM -->
      <template v-if="change.type === 'CITY_CUSTOM'">
        <div class="custom-info">
          <p class="note">{{ change.note }}</p>
          <CodeBlock :code="change.data" />
        </div>
      </template>
    </div>
  </div>
</template>

<script setup>
const props = defineProps({
  change: {
    type: Object,
    required: true
  }
});

const emit = defineEmits(['resolve', 'toggle-expand']);

const getIcon = (type) => {
  const icons = {
    'NEW_FEATURE': '➕',
    'NEW_OPTION': '🆕',
    'MASTER_UPDATE': '🔄',
    'CONFLICT': '⚠️',
    'CITY_CUSTOM': 'ℹ️'
  };
  return icons[type] || '•';
};

const getTypeLabel = (type) => {
  const labels = {
    'NEW_FEATURE': '새 Feature 추가',
    'NEW_OPTION': '새 Option 추가',
    'MASTER_UPDATE': '중앙 업데이트',
    'CONFLICT': '충돌 - 해결 필요',
    'CITY_CUSTOM': '지자체 커스터마이징'
  };
  return labels[type] || type;
};

const formatValue = (val) => {
  if (typeof val === 'object') return JSON.stringify(val);
  return String(val);
};
</script>
```

#### 2.3 ConflictResolver 컴포넌트 (충돌 해결)

```vue
<template>
  <div class="conflict-resolver">
    <h5>⚠️ 충돌 해결이 필요합니다</h5>
    <p class="conflict-description">
      중앙과 {{ cityName }}에서 동시에 수정되었습니다. 
      어떤 버전을 사용할지 선택하거나 직접 편집하세요.
    </p>

    <!-- 옵션 1: 중앙 버전 -->
    <div 
      class="conflict-option"
      :class="{ selected: change.resolution === 'master' }"
      @click="selectResolution('master')"
    >
      <div class="option-header">
        <input 
          type="radio" 
          :name="`conflict-${change.id}`"
          value="master"
          v-model="change.resolution"
        />
        <strong>🏢 중앙 버전 사용</strong>
      </div>
      <CodeBlock :code="change.master" />
    </div>

    <!-- 옵션 2: 지자체 버전 -->
    <div 
      class="conflict-option"
      :class="{ selected: change.resolution === 'city' }"
      @click="selectResolution('city')"
    >
      <div class="option-header">
        <input 
          type="radio" 
          :name="`conflict-${change.id}`"
          value="city"
          v-model="change.resolution"
        />
        <strong>🏙️ {{ cityName }} 버전 유지</strong>
      </div>
      <CodeBlock :code="change.city" />
    </div>

    <!-- 옵션 3: 수동 편집 -->
    <div 
      class="conflict-option manual"
      :class="{ selected: change.resolution === 'manual' }"
    >
      <div class="option-header">
        <input 
          type="radio" 
          :name="`conflict-${change.id}`"
          value="manual"
          v-model="change.resolution"
          @change="initManualEdit"
        />
        <strong>✏️ 직접 편집</strong>
      </div>
      <textarea 
        v-if="change.resolution === 'manual'"
        v-model="change.manualEdit"
        rows="12"
        class="manual-edit"
        placeholder="JSON 형식으로 입력하세요..."
      ></textarea>
      <div v-if="manualEditError" class="error-message">
        ❌ {{ manualEditError }}
      </div>
    </div>

    <!-- 저장 버튼 -->
    <button 
      @click="saveResolution"
      :disabled="!change.resolution"
      class="btn btn-primary btn-save-resolution"
    >
      ✅ 해결 완료
    </button>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';
import { ElMessage } from 'element-plus';

const props = defineProps({
  change: {
    type: Object,
    required: true
  },
  cityName: {
    type: String,
    default: '지자체'
  }
});

const emit = defineEmits(['save']);

const manualEditError = ref('');

const selectResolution = (type) => {
  props.change.resolution = type;
  if (type === 'manual') {
    initManualEdit();
  }
};

const initManualEdit = () => {
  if (!props.change.manualEdit) {
    // 중앙 버전을 기본값으로
    props.change.manualEdit = JSON.stringify(props.change.master, null, 2);
  }
};

const saveResolution = () => {
  if (!props.change.resolution) {
    ElMessage.warning('해결 방법을 선택해주세요');
    return;
  }
  
  // resolution에 따라 최종 데이터 결정
  let finalData;
  
  if (props.change.resolution === 'master') {
    finalData = props.change.master;
  } else if (props.change.resolution === 'city') {
    finalData = props.change.city;
  } else if (props.change.resolution === 'manual') {
    try {
      finalData = JSON.parse(props.change.manualEdit);
      manualEditError.value = '';
    } catch (e) {
      manualEditError.value = 'JSON 형식이 올바르지 않습니다: ' + e.message;
      return;
    }
  }
  
  props.change.finalData = finalData;
  props.change.resolved = true;
  props.change.selected = true;
  
  emit('save', props.change);
  ElMessage.success('충돌이 해결되었습니다');
};
</script>
```

---

### 3. 스크립트 로직

```javascript
// PageOptionEditor.vue 또는 별도 컴포넌트

import { ref, computed } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';

// 상태 관리
const cities = ref([
  { id: 'seoul', name: '서울시', baseVersion: '1.2.0' },
  { id: 'busan', name: '부산시', baseVersion: '1.1.5' },
  { id: 'daegu', name: '대구시', baseVersion: '1.2.0' },
  // ...
]);

const selectedCity = ref(null);
const analysisResult = ref(null);
const showSafe = ref(true);
const showConflicts = ref(true);
const showCustoms = ref(false);

// 분석 실행
const analyzeForCity = async () => {
  if (!selectedCity.value) return;
  
  try {
    ElMessage.info('변경사항을 분석 중입니다...');
    
    // 1. 베이스 버전 가져오기 (지자체가 마지막으로 동기화한 시점)
    const baseVersion = await fetchBaseVersion(selectedCity.value.id);
    
    // 2. 현재 중앙 마스터
    const newMaster = pageOptions.value;
    
    // 3. 지자체 현재 데이터
    const cityData = await fetchCityData(selectedCity.value.id);
    
    // 4. 변경사항 분석
    const changes = analyzeChanges(baseVersion, newMaster, cityData);
    
    // 5. 결과 저장
    analysisResult.value = {
      city: selectedCity.value,
      baseVersion: baseVersion.version,
      masterVersion: getCurrentMasterVersion(),
      timestamp: new Date().toISOString(),
      changes: changes.map((c, idx) => ({
        ...c,
        id: idx,
        selected: c.autoApply,
        expanded: false,
        resolution: null,
        manualEdit: '',
        resolved: false
      }))
    };
    
    ElMessage.success(`분석 완료: ${changes.length}건의 변경사항 발견`);
  } catch (error) {
    console.error('분석 실패:', error);
    ElMessage.error('분석 실패: ' + error.message);
  }
};

// 필터링
const filteredChanges = computed(() => {
  if (!analysisResult.value) return [];
  
  return analysisResult.value.changes.filter(change => {
    if (showSafe.value && change.safe && change.type !== 'CITY_CUSTOM') return true;
    if (showConflicts.value && change.type === 'CONFLICT') return true;
    if (showCustoms.value && change.type === 'CITY_CUSTOM') return true;
    return false;
  });
});

// 카테고리별 변경사항
const safeChanges = computed(() => 
  analysisResult.value?.changes.filter(c => 
    c.safe && c.type !== 'CITY_CUSTOM'
  ) || []
);

const conflictChanges = computed(() => 
  analysisResult.value?.changes.filter(c => c.type === 'CONFLICT') || []
);

const customChanges = computed(() => 
  analysisResult.value?.changes.filter(c => c.type === 'CITY_CUSTOM') || []
);

// 선택된 변경사항
const selectedChanges = computed(() => 
  analysisResult.value?.changes.filter(c => c.selected) || []
);

// 미해결 충돌 체크
const hasUnresolvedConflicts = computed(() => 
  selectedChanges.value.some(c => c.type === 'CONFLICT' && !c.resolved)
);

// 자동 적용
const autoApplyAll = () => {
  safeChanges.value.forEach(change => {
    change.selected = true;
  });
  ElMessage.success(`${safeChanges.value.length}건을 자동 선택했습니다`);
};

// 미리보기
const previewChanges = () => {
  // 미리보기 모달 표시
  const preview = generatePreview(selectedChanges.value);
  // ...
};

// 변경사항 적용
const applyChanges = async () => {
  if (hasUnresolvedConflicts.value) {
    ElMessage.error('먼저 모든 충돌을 해결해주세요');
    return;
  }
  
  try {
    await ElMessageBox.confirm(
      `${selectedCity.value.name}에 ${selectedChanges.value.length}건의 변경사항을 적용하시겠습니까?\n\n이 작업은 되돌릴 수 없습니다.`,
      '적용 확인',
      {
        confirmButtonText: '적용',
        cancelButtonText: '취소',
        type: 'warning'
      }
    );
    
    // API 호출
    const payload = {
      cityId: selectedCity.value.id,
      changes: selectedChanges.value.map(c => ({
        type: c.type,
        target: c.target,
        action: c.action,
        data: c.finalData || c.data || c.new
      })),
      newBaseVersion: getCurrentMasterVersion(),
      timestamp: new Date().toISOString()
    };
    
    const response = await fetch(
      `/api/municipalities/${selectedCity.value.id}/apply-changes`,
      {
        method: 'POST',
        headers: { 
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${getAuthToken()}`
        },
        body: JSON.stringify(payload)
      }
    );
    
    if (!response.ok) {
      throw new Error('적용 실패: ' + response.statusText);
    }
    
    const result = await response.json();
    
    ElMessage.success('✅ 적용 완료!');
    
    // 적용 리포트 표시
    showApplyReport(result);
    
    // 상태 초기화
    analysisResult.value = null;
    selectedCity.value = null;
    
  } catch (error) {
    if (error !== 'cancel') {
      console.error('적용 실패:', error);
      ElMessage.error('적용 실패: ' + error.message);
    }
  }
};

// API 호출 함수들
const fetchBaseVersion = async (cityId) => {
  const response = await fetch(`/api/municipalities/${cityId}/base-version`);
  return await response.json();
};

const fetchCityData = async (cityId) => {
  const response = await fetch(`/api/municipalities/${cityId}/pageOptions`);
  return await response.json();
};

const getCurrentMasterVersion = () => {
  // 중앙 마스터의 현재 버전
  return '1.3.0'; // 실제로는 버전 관리 시스템에서 가져옴
};

const getAuthToken = () => {
  // 인증 토큰 가져오기
  return localStorage.getItem('authToken');
};
```

---

## 👤 사용자 워크플로우

### 일반적인 배포 프로세스

```
1. 중앙에서 pageOption 수정
   ↓
2. "지자체 배포" 메뉴 선택
   ↓
3. 대상 지자체 선택
   ↓
4. 자동 분석 실행
   ├─ ✅ 안전: 23건
   ├─ ⚠️ 충돌: 3건
   └─ ℹ️ 커스텀: 5건
   ↓
5. 안전한 것들 자동 선택 ("자동 적용" 버튼)
   ↓
6. 충돌 항목 하나씩 검토
   ├─ Option 1: 중앙 버전 선택
   ├─ Option 2: 지자체 버전 유지
   └─ Option 3: 직접 편집
   ↓
7. 미리보기 확인
   ↓
8. 적용 실행
   ↓
9. 완료 리포트 확인
   ↓
10. 다음 지자체 선택하여 반복
```

### 대량 배포 프로세스

```
1. 모든 지자체 일괄 분석
   ↓
2. 지자체별 변경사항 요약 확인
   ├─ 서울시: 안전 20건, 충돌 2건
   ├─ 부산시: 안전 23건, 충돌 0건
   └─ 대구시: 안전 23건, 충돌 1건
   ↓
3. 충돌 없는 지자체 자동 적용
   (부산시 → 자동 완료)
   ↓
4. 충돌 있는 지자체만 수동 처리
   (서울시, 대구시)
   ↓
5. 전체 완료 리포트
```

---

## 📡 API 명세

### 1. 베이스 버전 조회

**Endpoint**: `GET /api/municipalities/:cityId/base-version`

**Response**:
```json
{
  "cityId": "seoul",
  "version": "1.2.0",
  "lastSyncDate": "2024-01-15T10:30:00Z",
  "data": [
    // pageOption 전체 데이터
  ]
}
```

---

### 2. 지자체 현재 데이터 조회

**Endpoint**: `GET /api/municipalities/:cityId/pageOptions`

**Response**:
```json
{
  "cityId": "seoul",
  "currentVersion": "1.2.0-seoul-custom",
  "lastModified": "2024-01-20T14:20:00Z",
  "data": [
    // 지자체의 현재 pageOption 데이터
  ]
}
```

---

### 3. 변경사항 적용

**Endpoint**: `POST /api/municipalities/:cityId/apply-changes`

**Request**:
```json
{
  "cityId": "seoul",
  "changes": [
    {
      "type": "NEW_OPTION",
      "target": "FEATURE001 > newOption",
      "action": "ADD",
      "data": {
        "desc": "새로운 옵션",
        "value": "default",
        "list": [...]
      }
    },
    {
      "type": "MASTER_UPDATE",
      "target": "FEATURE002 > option1",
      "action": "UPDATE",
      "data": {
        "desc": "수정된 설명",
        "value": "updated",
        "list": [...]
      }
    },
    {
      "type": "CONFLICT",
      "target": "FEATURE003 > option2",
      "action": "UPDATE",
      "data": {
        // 사용자가 선택한 최종 데이터
      },
      "resolution": "master" // or "city" or "manual"
    }
  ],
  "newBaseVersion": "1.3.0",
  "timestamp": "2024-01-25T09:00:00Z"
}
```

**Response**:
```json
{
  "success": true,
  "cityId": "seoul",
  "appliedChanges": 23,
  "newVersion": "1.3.0-seoul-custom",
  "details": [
    {
      "target": "FEATURE001 > newOption",
      "status": "success",
      "message": "추가 완료"
    },
    {
      "target": "FEATURE002 > option1",
      "status": "success",
      "message": "업데이트 완료"
    }
  ],
  "timestamp": "2024-01-25T09:01:23Z"
}
```

---

### 4. 롤백

**Endpoint**: `POST /api/municipalities/:cityId/rollback`

**Request**:
```json
{
  "cityId": "seoul",
  "targetVersion": "1.2.0",
  "reason": "적용 오류 발생"
}
```

**Response**:
```json
{
  "success": true,
  "cityId": "seoul",
  "rolledBackTo": "1.2.0",
  "timestamp": "2024-01-25T10:00:00Z"
}
```

---

## 📊 배포 시나리오 예시

### 시나리오 1: 새 옵션 추가

**상황**: 
- 중앙에서 `FEATURE001`에 `newOption` 추가
- 모든 지자체에 배포 필요

**분석 결과**:
- 모든 지자체: `NEW_OPTION` (안전 적용 가능)

**처리**:
```
1. 전체 지자체 선택
2. "자동 적용" 클릭
3. 일괄 배포 완료 ✅
```

---

### 시나리오 2: 기존 옵션 수정

**상황**:
- 중앙에서 `FEATURE001 > option1`의 `desc` 수정
- 서울시는 동일 항목을 자체적으로 수정함

**분석 결과**:
- 서울시: `CONFLICT` ⚠️
- 부산시: `MASTER_UPDATE` ✅
- 대구시: `MASTER_UPDATE` ✅

**처리**:
```
1. 부산시, 대구시: 자동 적용 ✅
2. 서울시: 충돌 해결 필요
   - 중앙 버전 확인: "전국 공통 설명"
   - 서울시 버전 확인: "서울시 맞춤 설명"
   - 선택: 서울시 버전 유지 (지역 특성 반영)
3. 적용 완료 ✅
```

---

### 시나리오 3: 복합 변경

**상황**:
- `FEATURE001 > option1` 수정 (중앙)
- `FEATURE002` 추가 (중앙)
- `FEATURE003 > option3` 수정 (중앙 + 서울시 동시)

**서울시 분석 결과**:
```
✅ 안전 적용 가능: 2건
  - FEATURE001 > option1 (UPDATE)
  - FEATURE002 (ADD)

⚠️ 충돌: 1건
  - FEATURE003 > option3 (CONFLICT)

ℹ️ 지자체 커스텀: 3건
  - FEATURE999 (서울시 전용)
  - FEATURE888 > customOption
  - FEATURE777 > anotherOption
```

**처리**:
```
1. 안전한 2건 자동 선택
2. FEATURE003 > option3 충돌 해결
   - Diff 확인:
     중앙: desc="전국 공통", value="A"
     서울: desc="서울 맞춤", value="B"
   - 선택: 직접 편집
     desc="서울 맞춤", value="A" (절충)
3. 적용 완료 ✅
4. 지자체 커스텀 3건은 자동 유지
```

---

## 🔒 안전장치

### 1. 백업
모든 적용 전에 자동 백업:
```json
{
  "cityId": "seoul",
  "backupVersion": "1.2.0",
  "backupDate": "2024-01-25T09:00:00Z",
  "data": [...]
}
```

### 2. 롤백
문제 발생 시 즉시 롤백 가능

### 3. 로그
모든 변경사항 기록:
```json
{
  "action": "APPLY_CHANGES",
  "cityId": "seoul",
  "user": "admin@example.com",
  "timestamp": "2024-01-25T09:00:00Z",
  "changes": [...],
  "result": "success"
}
```

### 4. 권한 관리
- 중앙 관리자: 모든 지자체 배포 가능
- 지자체 관리자: 자신의 지자체만 확인 가능

---

## 🎯 핵심 장점

1. **안전성**: 
   - 자동 충돌 감지
   - 선택적 적용
   - 롤백 가능

2. **효율성**:
   - 안전한 것은 자동 처리
   - 대량 배포 지원
   - 시간 절약

3. **유연성**:
   - 지자체 커스터마이징 보존
   - 다양한 해결 방법 제공
   - 세밀한 제어 가능

4. **추적성**:
   - 모든 변경 기록
   - 버전 관리
   - 리포트 생성

---

**작성일**: 2024  
**버전**: 1.0.0

