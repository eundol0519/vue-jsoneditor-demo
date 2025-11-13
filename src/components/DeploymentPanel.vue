<template>
  <div class="deployment-panel">
    <div class="panel-header">
      <h2>🚀 배포 관리</h2>
      <p>업데이트된 JSON을 업로드하고 변경사항을 검토하세요</p>
    </div>

    <!-- Step 1: JSON 업로드 -->
    <div class="upload-section">
      <h3>📤 Step 1: 업데이트 JSON 업로드</h3>
      <div class="upload-area">
        <input 
          type="file" 
          ref="fileInput"
          @change="handleFileUpload"
          accept=".json"
          id="json-upload"
          style="display: none"
        />
        <button @click="$refs.fileInput.click()" class="btn btn-upload">
          📁 JSON 파일 선택
        </button>
        <span v-if="uploadedFileName" class="file-name">
          {{ uploadedFileName }}
        </span>
      </div>
      <button 
        v-if="newMasterData"
        @click="analyzeChanges"
        class="btn btn-primary"
      >
        🔍 변경사항 분석
      </button>
    </div>

    <!-- Step 2: 분석 결과 -->
    <div v-if="analysisResult" class="analysis-section">
      <h3>📊 Step 2: 변경사항 분석 결과</h3>
      
      <!-- 요약 카드 -->
      <div class="summary-cards">
        <div class="summary-card safe">
          <div class="card-icon">✅</div>
          <div class="card-content">
            <h4>안전 적용 가능</h4>
            <p class="count">{{ safeChanges.length }}건</p>
            <p class="description">새로 추가되거나 수정되지 않은 항목</p>
          </div>
        </div>

        <div class="summary-card warning">
          <div class="card-icon">⚠️</div>
          <div class="card-content">
            <h4>충돌 (검토 필요)</h4>
            <p class="count">{{ conflictChanges.length }}건</p>
            <p class="description">현재 데이터와 업데이트 모두 수정됨</p>
          </div>
        </div>
      </div>

      <!-- 필터 -->
      <div class="filters">
        <label>
          <input type="checkbox" v-model="filters.safe" />
          안전 적용 가능 ({{ safeChanges.length }})
        </label>
        <label>
          <input type="checkbox" v-model="filters.conflicts" />
          충돌 ({{ conflictChanges.length }})
        </label>
      </div>

      <!-- 변경사항 목록 -->
      <div class="changes-list">
        <div 
          v-for="change in filteredChanges"
          :key="change.id"
          class="change-item"
          :class="[change.type.toLowerCase(), { expanded: change.expanded }]"
        >
          <!-- 헤더 -->
          <div class="change-header" @click="toggleExpand(change)">
            <div class="change-left">
              <input 
                type="checkbox" 
                v-model="change.selected"
                :disabled="change.type === 'CONFLICT' && !change.resolved"
                @click.stop
              />
              <span class="change-icon">{{ getIcon(change.type) }}</span>
              <div class="change-info">
                <strong>{{ change.target }}</strong>
                <span class="change-type-badge">{{ getTypeLabel(change.type) }}</span>
              </div>
            </div>
            <div class="change-right">
              <button 
                v-if="change.type === 'CONFLICT'"
                @click.stop="resolveConflict(change)"
                class="btn btn-resolve"
                :disabled="change.resolved"
              >
                {{ change.resolved ? '✅ 해결됨' : '🔧 해결하기' }}
              </button>
              <span class="expand-icon">
                {{ change.expanded ? '▼' : '▶' }}
              </span>
            </div>
          </div>

          <!-- 상세 내용 -->
          <div v-if="change.expanded" class="change-body">
            <!-- NEW -->
            <div v-if="change.type === 'NEW_FEATURE' || change.type === 'NEW_OPTION'">
              <div class="section-title">✨ 추가될 내용</div>
              <pre class="code-block">{{ formatJSON(change.data) }}</pre>
            </div>

            <!-- UPDATE -->
            <div v-if="change.type === 'UPDATE'">
              <div class="diff-container">
                <div class="diff-side old">
                  <div class="section-title">❌ 현재</div>
                  <pre class="code-block">{{ formatJSON(change.old) }}</pre>
                </div>
                <div class="diff-arrow">→</div>
                <div class="diff-side new">
                  <div class="section-title">✅ 업데이트</div>
                  <pre class="code-block">{{ formatJSON(change.new) }}</pre>
                </div>
              </div>
              <div class="diff-summary">
                <strong>변경된 필드:</strong>
                <ul>
                  <li v-for="(val, key) in change.diff" :key="key">
                    <code>{{ key }}</code>: 
                    <del>{{ formatValue(val.old) }}</del> → 
                    <ins>{{ formatValue(val.new) }}</ins>
                  </li>
                </ul>
              </div>
            </div>

            <!-- CONFLICT -->
            <div v-if="change.type === 'CONFLICT'" class="conflict-resolver">
              <div class="conflict-warning">
                ⚠️ 현재 데이터와 업데이트 모두 수정되었습니다. 어떤 버전을 사용할지 선택하세요.
              </div>

              <div class="conflict-options">
                <div 
                  class="conflict-option"
                  :class="{ selected: change.resolution === 'new' }"
                  @click="change.resolution = 'new'"
                >
                  <input 
                    type="radio" 
                    :name="`resolution-${change.id}`"
                    value="new"
                    v-model="change.resolution"
                  />
                  <div class="option-content">
                    <strong>📥 업데이트 버전 사용</strong>
                    <pre class="code-block small">{{ formatJSON(change.new) }}</pre>
                  </div>
                </div>

                <div 
                  class="conflict-option"
                  :class="{ selected: change.resolution === 'current' }"
                  @click="change.resolution = 'current'"
                >
                  <input 
                    type="radio" 
                    :name="`resolution-${change.id}`"
                    value="current"
                    v-model="change.resolution"
                  />
                  <div class="option-content">
                    <strong>📌 현재 버전 유지</strong>
                    <pre class="code-block small">{{ formatJSON(change.current) }}</pre>
                  </div>
                </div>

                <div 
                  class="conflict-option manual"
                  :class="{ selected: change.resolution === 'manual' }"
                >
                  <input 
                    type="radio" 
                    :name="`resolution-${change.id}`"
                    value="manual"
                    v-model="change.resolution"
                  />
                  <div class="option-content">
                    <strong>✏️ 직접 편집</strong>
                    <textarea 
                      v-if="change.resolution === 'manual'"
                      v-model="change.manualEdit"
                      rows="8"
                      class="manual-edit"
                      placeholder="JSON 형식으로 입력..."
                    ></textarea>
                  </div>
                </div>
              </div>

              <button 
                @click="saveResolution(change)"
                :disabled="!change.resolution"
                class="btn btn-primary"
              >
                ✅ 해결 완료
              </button>
            </div>

            <!-- CUSTOM -->
            <div v-if="change.type === 'CUSTOM'">
              <div class="custom-note">
                ℹ️ {{ change.note }}
              </div>
              <pre class="code-block">{{ formatJSON(change.data) }}</pre>
            </div>
          </div>
        </div>

        <div v-if="filteredChanges.length === 0" class="empty-state">
          필터 조건에 맞는 변경사항이 없습니다.
        </div>
      </div>

      <!-- Step 3: 적용 -->
      <div class="apply-section">
        <h3>🎯 Step 3: 변경사항 적용</h3>
        <div class="apply-info">
          <div class="info-item">
            <span class="label">선택된 항목:</span>
            <span class="value">{{ selectedChanges.length }}건</span>
          </div>
        </div>
        <div class="apply-buttons">
          <button 
            @click="previewApply"
            :disabled="selectedChanges.length === 0"
            class="btn btn-secondary"
          >
            👁️ 미리보기
          </button>
          <button 
            @click="applyChanges"
            :disabled="selectedChanges.length === 0 || unresolvedConflicts.length > 0"
            class="btn btn-primary btn-large"
          >
            ✅ 적용하기 ({{ selectedChanges.length }}건)
          </button>
        </div>
        <p v-if="unresolvedConflicts.length > 0" class="warning-text">
          ⚠️ 모든 충돌을 해결해야 적용할 수 있습니다
        </p>
      </div>
    </div>

    <!-- 미리보기 모달 -->
    <teleport to="body">
      <div v-if="showPreviewModal" class="modal-overlay" @click="showPreviewModal = false">
        <div class="modal-content modal-large" @click.stop>
          <div class="modal-header">
            <h3>👁️ 적용 미리보기</h3>
            <button class="btn-close" @click="showPreviewModal = false">✕</button>
          </div>
          <div class="modal-body">
            <pre class="code-block">{{ formatJSON(previewData) }}</pre>
          </div>
          <div class="modal-actions">
            <button @click="showPreviewModal = false" class="btn btn-secondary">
              닫기
            </button>
          </div>
        </div>
      </div>
    </teleport>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';

const props = defineProps({
  currentData: {
    type: Array,
    required: true
  }
});

const emit = defineEmits(['apply']);

// 상태
const fileInput = ref(null);
const uploadedFileName = ref('');
const newMasterData = ref(null);
const analysisResult = ref(null);
const showPreviewModal = ref(false);
const previewData = ref(null);

const filters = ref({
  safe: true,
  conflicts: true
});

// 파일 업로드
const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  uploadedFileName.value = file.name;
  
  const reader = new FileReader();
  reader.onload = (e) => {
    try {
      newMasterData.value = JSON.parse(e.target.result);
      ElMessage.success('JSON 파일이 업로드되었습니다');
    } catch (error) {
      ElMessage.error('JSON 파일 형식이 올바르지 않습니다');
      uploadedFileName.value = '';
      newMasterData.value = null;
    }
  };
  reader.readAsText(file);
};

// 변경사항 분석
const analyzeChanges = () => {
  if (!newMasterData.value) {
    ElMessage.warning('먼저 JSON 파일을 업로드하세요');
    return;
  }

  const changes = [];
  let idCounter = 0;

  // 기존 데이터를 Map으로 변환
  const currentMap = new Map();
  props.currentData.forEach(feature => {
    currentMap.set(feature.featureId, feature);
  });

  // 새 데이터를 Map으로 변환
  const newMap = new Map();
  newMasterData.value.forEach(feature => {
    newMap.set(feature.featureId, feature);
  });

  // 1. 새 데이터 순회
  newMasterData.value.forEach(newFeature => {
    const currentFeature = currentMap.get(newFeature.featureId);

    // Case 1: 완전히 새로운 Feature
    if (!currentFeature) {
      changes.push({
        id: idCounter++,
        type: 'NEW_FEATURE',
        target: `Feature: ${newFeature.featureId}`,
        data: newFeature,
        selected: true,
        expanded: false,
        safe: true
      });
      return;
    }

    // Case 2: Feature 존재, Option 체크
    const currentOptions = currentFeature.option || {};
    const newOptions = newFeature.option || {};

    Object.keys(newOptions).forEach(optionKey => {
      const currentOption = currentOptions[optionKey];
      const newOption = newOptions[optionKey];

      // Case 2-1: 새로운 Option
      if (!currentOption) {
        changes.push({
          id: idCounter++,
          type: 'NEW_OPTION',
          target: `${newFeature.featureId} > ${optionKey}`,
          data: newOption,
          selected: true,
          expanded: false,
          safe: true
        });
      }
      // Case 2-2: Option 수정
      else {
        const currentStr = JSON.stringify(currentOption);
        const newStr = JSON.stringify(newOption);

        if (currentStr !== newStr) {
          // 충돌로 간주 (실제로는 베이스 버전과 비교해야 하지만 간소화)
          changes.push({
            id: idCounter++,
            type: 'CONFLICT',
            target: `${newFeature.featureId} > ${optionKey}`,
            current: currentOption,
            new: newOption,
            old: currentOption, // 간소화
            diff: calculateDiff(currentOption, newOption),
            selected: false,
            expanded: false,
            safe: false,
            resolved: false,
            resolution: null,
            manualEdit: ''
          });
        }
      }
    });
  });

  // 2. 현재 데이터에만 있는 것 (커스텀)
  props.currentData.forEach(currentFeature => {
    const newFeature = newMap.get(currentFeature.featureId);

    if (!newFeature) {
      // Feature 전체가 커스텀
      changes.push({
        id: idCounter++,
        type: 'CUSTOM',
        target: `Feature: ${currentFeature.featureId}`,
        data: currentFeature,
        note: '업데이트 JSON에는 없고, 현재 데이터에만 있는 Feature입니다. 자동으로 유지됩니다.',
        selected: false,
        expanded: false,
        safe: true
      });
    } else {
      // Option 레벨 체크
      const currentOptions = currentFeature.option || {};
      const newOptions = newFeature.option || {};

      Object.keys(currentOptions).forEach(optionKey => {
        if (!newOptions[optionKey]) {
          changes.push({
            id: idCounter++,
            type: 'CUSTOM',
            target: `${currentFeature.featureId} > ${optionKey}`,
            data: currentOptions[optionKey],
            note: '업데이트 JSON에는 없고, 현재 데이터에만 있는 Option입니다. 자동으로 유지됩니다.',
            selected: false,
            expanded: false,
            safe: true
          });
        }
      });
    }
  });

  analysisResult.value = changes;
  ElMessage.success(`분석 완료: ${changes.length}건의 변경사항 발견`);
};

// Diff 계산
const calculateDiff = (oldObj, newObj) => {
  const diff = {};
  const allKeys = new Set([...Object.keys(oldObj), ...Object.keys(newObj)]);

  allKeys.forEach(key => {
    const oldVal = JSON.stringify(oldObj[key]);
    const newVal = JSON.stringify(newObj[key]);
    if (oldVal !== newVal) {
      diff[key] = {
        old: oldObj[key],
        new: newObj[key]
      };
    }
  });

  return diff;
};

// 필터링된 변경사항
const filteredChanges = computed(() => {
  if (!analysisResult.value) return [];

  return analysisResult.value.filter(change => {
    if (filters.value.safe && (change.type === 'NEW_FEATURE' || change.type === 'NEW_OPTION' || change.type === 'UPDATE')) {
      return true;
    }
    if (filters.value.conflicts && change.type === 'CONFLICT') {
      return true;
    }
    // CUSTOM 타입은 필터링에서 제외 (표시하지 않음)
    return false;
  });
});

// 카테고리별 변경사항
const safeChanges = computed(() => 
  analysisResult.value?.filter(c => c.safe && c.type !== 'CUSTOM') || []
);

const conflictChanges = computed(() => 
  analysisResult.value?.filter(c => c.type === 'CONFLICT') || []
);

const customChanges = computed(() => 
  analysisResult.value?.filter(c => c.type === 'CUSTOM') || []
);

const selectedChanges = computed(() => 
  analysisResult.value?.filter(c => c.selected) || []
);

const unresolvedConflicts = computed(() => 
  selectedChanges.value.filter(c => c.type === 'CONFLICT' && !c.resolved) || []
);

// 액션
const toggleExpand = (change) => {
  change.expanded = !change.expanded;
};

const selectAllSafe = () => {
  safeChanges.value.forEach(c => c.selected = true);
  ElMessage.success(`${safeChanges.value.length}건 선택됨`);
};

const showConflictsOnly = () => {
  filters.value.safe = false;
  filters.value.conflicts = true;
};

const resolveConflict = (change) => {
  change.expanded = true;
};

const saveResolution = (change) => {
  if (!change.resolution) {
    ElMessage.warning('해결 방법을 선택하세요');
    return;
  }

  if (change.resolution === 'manual') {
    try {
      JSON.parse(change.manualEdit);
    } catch (e) {
      ElMessage.error('JSON 형식이 올바르지 않습니다');
      return;
    }
  }

  change.resolved = true;
  change.selected = true;
  change.expanded = false; // 해결 완료 후 자동으로 접기
  ElMessage.success('충돌이 해결되었습니다');
};

const previewApply = () => {
  const result = JSON.parse(JSON.stringify(props.currentData));
  
  selectedChanges.value.forEach(change => {
    if (change.type === 'NEW_FEATURE') {
      result.push(change.data);
    } else if (change.type === 'NEW_OPTION') {
      const [featureId, optionKey] = change.target.split(' > ');
      const feature = result.find(f => f.featureId === featureId);
      if (feature) {
        if (!feature.option) feature.option = {};
        feature.option[optionKey] = change.data;
      }
    } else if (change.type === 'CONFLICT') {
      const [featureId, optionKey] = change.target.split(' > ');
      const feature = result.find(f => f.featureId === featureId);
      if (feature && feature.option) {
        if (change.resolution === 'new') {
          feature.option[optionKey] = change.new;
        } else if (change.resolution === 'manual') {
          feature.option[optionKey] = JSON.parse(change.manualEdit);
        }
        // 'current'면 그대로 유지
      }
    }
  });

  previewData.value = result;
  showPreviewModal.value = true;
};

const applyChanges = async () => {
  try {
    await ElMessageBox.confirm(
      `${selectedChanges.value.length}건의 변경사항을 적용하시겠습니까?\n\n이 작업은 되돌릴 수 없습니다.`,
      '적용 확인',
      {
        confirmButtonText: '적용',
        cancelButtonText: '취소',
        type: 'warning'
      }
    );

    const result = JSON.parse(JSON.stringify(props.currentData));
    
    selectedChanges.value.forEach(change => {
      if (change.type === 'NEW_FEATURE') {
        result.push(change.data);
      } else if (change.type === 'NEW_OPTION') {
        const [featureId, optionKey] = change.target.split(' > ');
        const feature = result.find(f => f.featureId === featureId);
        if (feature) {
          if (!feature.option) feature.option = {};
          feature.option[optionKey] = change.data;
        }
      } else if (change.type === 'CONFLICT') {
        const [featureId, optionKey] = change.target.split(' > ');
        const feature = result.find(f => f.featureId === featureId);
        if (feature && feature.option) {
          if (change.resolution === 'new') {
            feature.option[optionKey] = change.new;
          } else if (change.resolution === 'manual') {
            feature.option[optionKey] = JSON.parse(change.manualEdit);
          }
        }
      }
    });

    emit('apply', result);
    ElMessage.success('✅ 변경사항이 적용되었습니다');
    
    // 초기화
    analysisResult.value = null;
    newMasterData.value = null;
    uploadedFileName.value = '';

  } catch (error) {
    if (error !== 'cancel') {
      ElMessage.error('적용 실패');
    }
  }
};

// 유틸리티
const getIcon = (type) => {
  const icons = {
    'NEW_FEATURE': '➕',
    'NEW_OPTION': '🆕',
    'UPDATE': '🔄',
    'CONFLICT': '⚠️',
    'CUSTOM': 'ℹ️'
  };
  return icons[type] || '•';
};

const getTypeLabel = (type) => {
  const labels = {
    'NEW_FEATURE': '새 Feature',
    'NEW_OPTION': '새 Option',
    'UPDATE': '업데이트',
    'CONFLICT': '충돌',
    'CUSTOM': '여기에만 있음'
  };
  return labels[type] || type;
};

const formatJSON = (obj) => {
  return JSON.stringify(obj, null, 2);
};

const formatValue = (val) => {
  if (typeof val === 'object') return JSON.stringify(val);
  return String(val);
};
</script>

<style scoped>
.deployment-panel {
  padding: 24px;
  background: #f8f9fa;
  border-radius: 12px;
}

.panel-header {
  margin-bottom: 32px;
  text-align: center;
}

.panel-header h2 {
  font-size: 28px;
  margin-bottom: 8px;
}

.panel-header p {
  color: #6c757d;
  font-size: 14px;
}

/* Upload Section */
.upload-section {
  background: white;
  padding: 24px;
  border-radius: 12px;
  margin-bottom: 24px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
}

.upload-section h3 {
  font-size: 18px;
  margin-bottom: 16px;
  color: #333;
}

.upload-area {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-bottom: 16px;
}

.btn-upload {
  padding: 12px 24px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 15px;
  font-weight: 600;
  transition: all 0.3s ease;
}

.btn-upload:hover {
  background: #0056b3;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,123,255,0.3);
}

.file-name {
  color: #28a745;
  font-weight: 600;
}

/* Summary Cards */
.summary-cards {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 20px;
  margin: 24px 0;
}

.summary-card {
  background: white;
  border-radius: 12px;
  padding: 24px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.summary-card.safe {
  border-left: 4px solid #28a745;
}

.summary-card.warning {
  border-left: 4px solid #ffc107;
}

.summary-card.info {
  border-left: 4px solid #17a2b8;
}

.card-icon {
  font-size: 36px;
}

.card-content h4 {
  font-size: 16px;
  color: #333;
  margin-bottom: 8px;
}

.count {
  font-size: 32px;
  font-weight: bold;
  color: #333;
  margin: 8px 0;
}

.description {
  font-size: 13px;
  color: #6c757d;
  margin-bottom: 12px;
}

/* Filters */
.filters {
  display: flex;
  gap: 24px;
  padding: 16px;
  background: white;
  border-radius: 8px;
  margin-bottom: 16px;
}

.filters label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  font-size: 14px;
}

/* Changes List */
.changes-list {
  background: white;
  padding: 16px;
  border-radius: 12px;
  margin-bottom: 24px;
}

.change-item {
  border: 2px solid #e9ecef;
  border-radius: 8px;
  margin-bottom: 12px;
  overflow: hidden;
  transition: all 0.3s ease;
}

.change-item.new_feature,
.change-item.new_option {
  border-color: #28a745;
  background: #f0fff4;
}

.change-item.conflict {
  border-color: #ffc107;
  background: #fffbf0;
}

.change-item.custom {
  border-color: #17a2b8;
  background: #f0f9ff;
}

.change-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px;
  cursor: pointer;
  user-select: none;
}

.change-header:hover {
  background: rgba(0,0,0,0.02);
}

.change-left {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
}

.change-icon {
  font-size: 24px;
}

.change-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.change-type-badge {
  font-size: 12px;
  color: #6c757d;
}

.change-right {
  display: flex;
  align-items: center;
  gap: 12px;
}

.btn-resolve {
  padding: 6px 12px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
}

.btn-resolve:disabled {
  background: #6c757d;
  cursor: not-allowed;
}

.expand-icon {
  color: #6c757d;
  font-size: 14px;
}

.change-body {
  padding: 16px;
  border-top: 1px solid #e9ecef;
  background: white;
}

.section-title {
  font-weight: 600;
  margin-bottom: 12px;
  font-size: 14px;
}

.code-block {
  background: #2d2d2d;
  color: #f8f8f2;
  padding: 16px;
  border-radius: 8px;
  overflow-x: auto;
  font-size: 13px;
  font-family: 'Courier New', monospace;
  line-height: 1.6;
}

.code-block.small {
  max-height: 200px;
  overflow-y: auto;
}

.diff-container {
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  gap: 16px;
  margin: 16px 0;
}

.diff-side {
  padding: 12px;
  border-radius: 8px;
}

.diff-side.old {
  background: #ffe6e6;
  border: 2px solid #dc3545;
}

.diff-side.new {
  background: #e6ffe6;
  border: 2px solid #28a745;
}

.diff-arrow {
  display: flex;
  align-items: center;
  font-size: 24px;
  color: #6c757d;
}

.diff-summary {
  background: #f8f9fa;
  padding: 16px;
  border-radius: 8px;
  margin-top: 16px;
}

.diff-summary ul {
  list-style: none;
  padding: 0;
  margin: 8px 0 0 0;
}

.diff-summary li {
  margin: 8px 0;
  padding: 8px;
  background: white;
  border-radius: 4px;
}

del {
  background: #ffe6e6;
  color: #dc3545;
  padding: 2px 6px;
  border-radius: 3px;
  text-decoration: line-through;
}

ins {
  background: #e6ffe6;
  color: #28a745;
  padding: 2px 6px;
  border-radius: 3px;
  text-decoration: none;
}

/* Conflict Resolver */
.conflict-resolver {
  padding: 16px;
}

.conflict-warning {
  background: #fff3cd;
  border-left: 4px solid #ffc107;
  padding: 12px 16px;
  border-radius: 6px;
  margin-bottom: 16px;
  font-size: 14px;
  color: #856404;
}

.conflict-options {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 16px;
}

.conflict-option {
  border: 2px solid #e9ecef;
  border-radius: 8px;
  padding: 16px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.conflict-option:hover {
  border-color: #007bff;
  background: #f0f8ff;
}

.conflict-option.selected {
  border-color: #007bff;
  background: #e7f3ff;
}

.option-content {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.manual-edit {
  width: 100%;
  padding: 12px;
  border: 1px solid #ced4da;
  border-radius: 6px;
  font-family: 'Courier New', monospace;
  font-size: 13px;
  resize: vertical;
}

.custom-note {
  background: #d1ecf1;
  border-left: 4px solid #17a2b8;
  padding: 12px 16px;
  border-radius: 6px;
  margin-bottom: 12px;
  font-size: 14px;
  color: #0c5460;
}

/* Apply Section */
.apply-section {
  background: white;
  padding: 24px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
}

.apply-section h3 {
  font-size: 18px;
  margin-bottom: 16px;
}

.apply-info {
  display: flex;
  gap: 32px;
  margin-bottom: 16px;
  padding: 16px;
  background: #f8f9fa;
  border-radius: 8px;
}

.info-item {
  display: flex;
  gap: 8px;
  align-items: center;
}

.label {
  font-size: 14px;
  color: #6c757d;
}

.value {
  font-size: 18px;
  font-weight: bold;
  color: #333;
}

.value.error {
  color: #dc3545;
}

.apply-buttons {
  display: flex;
  gap: 12px;
  justify-content: center;
}

.btn {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 15px;
  font-weight: 600;
  transition: all 0.3s ease;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-sm {
  padding: 8px 16px;
  font-size: 13px;
}

.btn-large {
  padding: 16px 32px;
  font-size: 16px;
}

.btn-primary {
  background: #007bff;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #0056b3;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,123,255,0.3);
}

.btn-secondary {
  background: #6c757d;
  color: white;
}

.btn-secondary:hover:not(:disabled) {
  background: #5a6268;
}

.btn-success {
  background: #28a745;
  color: white;
}

.btn-success:hover {
  background: #218838;
}

.btn-warning {
  background: #ffc107;
  color: #333;
}

.btn-warning:hover {
  background: #e0a800;
}

.warning-text {
  text-align: center;
  color: #dc3545;
  font-size: 14px;
  margin-top: 12px;
}

.empty-state {
  text-align: center;
  padding: 40px;
  color: #6c757d;
  font-size: 14px;
}

/* Modal */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 2000;
}

.modal-content {
  background: white;
  border-radius: 12px;
  max-width: 800px;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  padding: 20px 24px;
  border-bottom: 1px solid #e9ecef;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.modal-header h3 {
  margin: 0;
  font-size: 20px;
}

.btn-close {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #6c757d;
  padding: 0;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
}

.btn-close:hover {
  background: #f8f9fa;
}

.modal-body {
  padding: 24px;
  overflow-y: auto;
  flex: 1;
}

.modal-actions {
  padding: 16px 24px;
  border-top: 1px solid #e9ecef;
  display: flex;
  justify-content: flex-end;
  gap: 12px;
}
</style>

