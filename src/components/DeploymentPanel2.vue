<template>
  <div class="deployment-panel-v2">
    <div class="panel-header">
      <h2>🔀 버전 관리 (Git 스타일)</h2>
      <p>GitHub 스타일로 변경사항을 검토하고 적용하세요</p>
    </div>

    <!-- Step 1: JSON 업로드 -->
    <div class="upload-section">
      <h3>📤 업데이트 JSON 업로드</h3>
      <div class="upload-area">
        <input 
          type="file" 
          ref="fileInput"
          @change="handleFileUpload"
          accept=".json"
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

    <!-- Step 2: Git-Style Diff -->
    <div v-if="analysisResult" class="diff-section">
      <h3>🔀 변경사항 검토</h3>
      
      <!-- Feature별로 그룹화 -->
      <div 
        v-for="group in groupedChanges" 
        :key="group.featureId"
        class="feature-group"
      >
        <div class="feature-header">
          <h4>📦 {{ group.featureId }}</h4>
          <span class="change-count">{{ group.changes.length }}개 변경</span>
        </div>

        <!-- 각 변경사항 -->
        <div 
          v-for="change in group.changes"
          :key="change.id"
          class="change-block"
          :class="change.type.toLowerCase()"
        >
          <!-- 새로 추가된 항목 -->
          <div v-if="change.type === 'NEW_FEATURE' || change.type === 'NEW_OPTION'" class="new-addition">
            <div class="diff-header">
              <span class="icon">✨</span>
              <strong>{{ change.target }}</strong>
              <span class="badge new">새로 추가됨</span>
            </div>
            <div class="diff-body new-content">
              <div class="line-header">추가될 내용:</div>
              <pre class="code-block">{{ formatJSON(change.data) }}</pre>
              <div class="action-buttons">
                <button 
                  @click="acceptChange(change, 'add')"
                  :class="{ selected: change.decision === 'add' }"
                  class="btn btn-accept"
                >
                  ✅ 추가 적용
                </button>
                <button 
                  @click="rejectChange(change)"
                  :class="{ selected: change.decision === 'reject' }"
                  class="btn btn-reject"
                >
                  ❌ 무시
                </button>
              </div>
            </div>
          </div>

          <!-- 업데이트 (충돌 없음) -->
          <div v-else-if="change.type === 'UPDATE'" class="update-change">
            <div class="diff-header">
              <span class="icon">📝</span>
              <strong>{{ change.target }}</strong>
              <span class="badge update">수정됨</span>
            </div>
            <div class="diff-body">
              <div class="git-conflict-style">
                <!-- Current (제거될 내용) -->
                <div class="conflict-section current">
                  <div class="section-label">
                    <span class="label-text">현재 버전</span>
                  </div>
                  <pre class="code-block">{{ formatJSON(change.old) }}</pre>
                </div>

                <div class="conflict-divider"></div>

                <!-- Incoming (새로운 내용) -->
                <div class="conflict-section incoming">
                  <div class="section-label">
                    <span class="label-text">업데이트 버전</span>
                  </div>
                  <pre class="code-block">{{ formatJSON(change.new) }}</pre>
                </div>
              </div>

              <div class="action-buttons">
                <button 
                  @click="acceptChange(change, 'new')"
                  :class="{ selected: change.decision === 'new' }"
                  class="btn btn-accept"
                >
                  📥 업데이트 버전 적용
                </button>
                <button 
                  @click="acceptChange(change, 'current')"
                  :class="{ selected: change.decision === 'current' }"
                  class="btn btn-keep"
                >
                  📌 현재 버전 유지
                </button>
              </div>
            </div>
          </div>

          <!-- 충돌 (양쪽 모두 수정됨) -->
          <div v-else-if="change.type === 'CONFLICT'" class="conflict-change">
            <div class="diff-header conflict">
              <span class="icon">⚠️</span>
              <strong>{{ change.target }}</strong>
              <span class="badge conflict">병합 충돌</span>
            </div>
            <div class="diff-body">
              <div class="conflict-warning">
                ⚠️ 이 항목은 양쪽에서 모두 수정되었습니다. 어떤 버전을 사용할지 선택하세요.
              </div>
              
              <div class="git-conflict-style">
                <!-- Current -->
                <div class="conflict-section current">
                  <div class="section-label conflict-label">
                    <span class="label-text"><<<<<<< HEAD (현재 버전)</span>
                  </div>
                  <pre class="code-block">{{ formatJSON(change.current) }}</pre>
                </div>

                <!-- Separator -->
                <div class="conflict-separator">
                  <span>=======</span>
                </div>

                <!-- Incoming -->
                <div class="conflict-section incoming">
                  <div class="section-label conflict-label">
                    <span class="label-text">>>>>>>> incoming (업데이트 버전)</span>
                  </div>
                  <pre class="code-block">{{ formatJSON(change.new) }}</pre>
                </div>
              </div>

              <div class="action-buttons">
                <button 
                  @click="acceptChange(change, 'new')"
                  :class="{ selected: change.decision === 'new' }"
                  class="btn btn-accept"
                >
                  📥 업데이트 버전 적용
                </button>
                <button 
                  @click="acceptChange(change, 'current')"
                  :class="{ selected: change.decision === 'current' }"
                  class="btn btn-keep"
                >
                  📌 현재 버전 유지
                </button>
                <button 
                  @click="acceptChange(change, 'both')"
                  :class="{ selected: change.decision === 'both' }"
                  class="btn btn-both"
                >
                  🔀 양쪽 모두 적용
                </button>
                <button 
                  @click="openManualEdit(change)"
                  :class="{ selected: change.decision === 'manual' }"
                  class="btn btn-manual"
                >
                  ✏️ 직접 편집
                </button>
              </div>

              <!-- 직접 편집 모드 -->
              <div v-if="change.showManualEdit" class="manual-edit-area">
                <div class="manual-header">
                  <strong>✏️ 직접 편집</strong>
                  <button @click="change.showManualEdit = false" class="btn-close">닫기</button>
                </div>
                <textarea 
                  v-model="change.manualEdit"
                  rows="10"
                  class="manual-textarea"
                ></textarea>
                <button 
                  @click="saveManualEdit(change)"
                  class="btn btn-primary"
                >
                  💾 저장
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 적용 섹션 -->
      <div class="apply-section">
        <div class="summary">
          <h4>📊 결정된 항목</h4>
          <div class="summary-stats">
            <span class="stat-item">
              <span class="stat-label">적용할 변경:</span>
              <span class="stat-value">{{ decidedChanges.length }}건</span>
            </span>
            <span class="stat-item">
              <span class="stat-label">미결정:</span>
              <span class="stat-value">{{ undecidedChanges.length }}건</span>
            </span>
          </div>
        </div>

        <div class="apply-buttons">
          <button 
            @click="previewMerge"
            :disabled="decidedChanges.length === 0"
            class="btn btn-secondary"
          >
            👁️ 미리보기
          </button>
          <button 
            @click="applyMerge"
            :disabled="undecidedChanges.length > 0"
            class="btn btn-primary btn-large"
          >
            ✅ 병합 적용 ({{ decidedChanges.length }}건)
          </button>
        </div>
        <p v-if="undecidedChanges.length > 0" class="warning-text">
          ⚠️ 모든 변경사항에 대해 결정해야 병합을 적용할 수 있습니다
        </p>
      </div>
    </div>

    <!-- 미리보기 모달 -->
    <teleport to="body">
      <div v-if="showPreviewModal" class="modal-overlay" @click="showPreviewModal = false">
        <div class="modal-content preview-modal" @click.stop>
          <div class="modal-header">
            <h3>👁️ 병합 미리보기</h3>
            <button @click="showPreviewModal = false" class="btn-close">✕</button>
          </div>
          <div class="modal-body">
            <pre class="preview-json">{{ previewData }}</pre>
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

// State
const fileInput = ref(null);
const uploadedFileName = ref('');
const newMasterData = ref(null);
const analysisResult = ref(null);
const showPreviewModal = ref(false);
const previewData = ref('');

// File Upload
const handleFileUpload = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  uploadedFileName.value = file.name;

  try {
    const text = await file.text();
    newMasterData.value = JSON.parse(text);
    ElMessage.success('JSON 파일을 성공적으로 로드했습니다');
  } catch (error) {
    ElMessage.error('JSON 파일을 읽는 중 오류가 발생했습니다');
    console.error(error);
  }
};

// Analyze Changes
const analyzeChanges = () => {
  if (!newMasterData.value) return;

  const changes = [];
  let changeId = 0;

  // 업데이트 JSON의 각 feature 분석
  newMasterData.value.forEach(newFeature => {
    const currentFeature = props.currentData.find(f => f.featureId === newFeature.featureId);

    if (!currentFeature) {
      // 새로운 Feature
      changes.push({
        id: changeId++,
        type: 'NEW_FEATURE',
        featureId: newFeature.featureId,
        target: `Feature: ${newFeature.featureId}`,
        data: newFeature,
        decision: null
      });
    } else {
      // 기존 Feature - Option 비교
      Object.entries(newFeature.option).forEach(([optionKey, newOption]) => {
        const currentOption = currentFeature.option[optionKey];

        if (!currentOption) {
          // 새로운 Option
          changes.push({
            id: changeId++,
            type: 'NEW_OPTION',
            featureId: newFeature.featureId,
            target: `${newFeature.featureId} > ${optionKey}`,
            optionKey: optionKey,
            data: newOption,
            decision: null
          });
        } else {
          // 기존 Option - 내용 비교
          const hasChanges = JSON.stringify(currentOption) !== JSON.stringify(newOption);
          
          if (hasChanges) {
            // 양쪽 모두 수정되었는지 확인 (간단히 내용이 다르면 충돌로 간주)
            const isConflict = 
              currentOption.desc !== newOption.desc || 
              currentOption.value !== newOption.value ||
              JSON.stringify(currentOption.list) !== JSON.stringify(newOption.list);

            changes.push({
              id: changeId++,
              type: isConflict ? 'CONFLICT' : 'UPDATE',
              featureId: newFeature.featureId,
              target: `${newFeature.featureId} > ${optionKey}`,
              optionKey: optionKey,
              old: currentOption,
              current: currentOption,
              new: newOption,
              decision: null,
              showManualEdit: false,
              manualEdit: ''
            });
          }
        }
      });
    }
  });

  analysisResult.value = changes;
  
  const visibleCount = changes.length;
  ElMessage.success(`분석 완료: ${visibleCount}건의 변경사항 발견`);
};

// Feature별로 그룹화
const groupedChanges = computed(() => {
  if (!analysisResult.value) return [];

  const groups = {};
  analysisResult.value.forEach(change => {
    if (!groups[change.featureId]) {
      groups[change.featureId] = {
        featureId: change.featureId,
        changes: []
      };
    }
    groups[change.featureId].changes.push(change);
  });

  return Object.values(groups);
});

// 결정된/미결정 변경사항
const decidedChanges = computed(() => {
  return analysisResult.value?.filter(c => c.decision !== null) || [];
});

const undecidedChanges = computed(() => {
  return analysisResult.value?.filter(c => c.decision === null) || [];
});

// Actions
const acceptChange = (change, decision) => {
  change.decision = decision;
  change.showManualEdit = false;
};

const rejectChange = (change) => {
  change.decision = 'reject';
};

const openManualEdit = (change) => {
  change.decision = 'manual';
  change.showManualEdit = true;
  if (!change.manualEdit) {
    change.manualEdit = formatJSON(change.current);
  }
};

const saveManualEdit = (change) => {
  try {
    JSON.parse(change.manualEdit);
    change.showManualEdit = false;
    ElMessage.success('수동 편집이 저장되었습니다');
  } catch (error) {
    ElMessage.error('유효한 JSON 형식이 아닙니다');
  }
};

// Preview
const previewMerge = () => {
  const result = JSON.parse(JSON.stringify(props.currentData));

  decidedChanges.value.forEach(change => {
    if (change.decision === 'reject') return;

    const featureIndex = result.findIndex(f => f.featureId === change.featureId);

    if (change.type === 'NEW_FEATURE') {
      if (change.decision === 'add') {
        result.push(change.data);
      }
    } else if (change.type === 'NEW_OPTION') {
      if (change.decision === 'add' && featureIndex !== -1) {
        result[featureIndex].option[change.optionKey] = change.data;
      }
    } else if (change.type === 'UPDATE' || change.type === 'CONFLICT') {
      if (featureIndex !== -1) {
        if (change.decision === 'new') {
          result[featureIndex].option[change.optionKey] = change.new;
        } else if (change.decision === 'current') {
          // 현재 유지 (변경 없음)
        } else if (change.decision === 'both') {
          // Both는 현재로는 new로 처리 (실제로는 병합 로직 필요)
          result[featureIndex].option[change.optionKey] = change.new;
        } else if (change.decision === 'manual') {
          result[featureIndex].option[change.optionKey] = JSON.parse(change.manualEdit);
        }
      }
    }
  });

  previewData.value = formatJSON(result);
  showPreviewModal.value = true;
};

// Apply
const applyMerge = async () => {
  if (undecidedChanges.value.length > 0) {
    ElMessage.warning('모든 변경사항에 대해 결정해주세요');
    return;
  }

  try {
    await ElMessageBox.confirm(
      `${decidedChanges.value.length}건의 변경사항을 적용하시겠습니까?`,
      '병합 적용',
      {
        confirmButtonText: '적용',
        cancelButtonText: '취소',
        type: 'warning'
      }
    );

    const result = JSON.parse(JSON.stringify(props.currentData));

    decidedChanges.value.forEach(change => {
      if (change.decision === 'reject') return;

      const featureIndex = result.findIndex(f => f.featureId === change.featureId);

      if (change.type === 'NEW_FEATURE') {
        if (change.decision === 'add') {
          result.push(change.data);
        }
      } else if (change.type === 'NEW_OPTION') {
        if (change.decision === 'add' && featureIndex !== -1) {
          result[featureIndex].option[change.optionKey] = change.data;
        }
      } else if (change.type === 'UPDATE' || change.type === 'CONFLICT') {
        if (featureIndex !== -1) {
          if (change.decision === 'new') {
            result[featureIndex].option[change.optionKey] = change.new;
          } else if (change.decision === 'current') {
            // 현재 유지
          } else if (change.decision === 'both') {
            result[featureIndex].option[change.optionKey] = change.new;
          } else if (change.decision === 'manual') {
            result[featureIndex].option[change.optionKey] = JSON.parse(change.manualEdit);
          }
        }
      }
    });

    emit('apply', result);
    ElMessage.success('병합이 성공적으로 적용되었습니다');

    // Reset
    analysisResult.value = null;
    newMasterData.value = null;
    uploadedFileName.value = '';
  } catch (error) {
    if (error !== 'cancel') {
      console.error(error);
    }
  }
};

// Utilities
const formatJSON = (obj) => {
  return JSON.stringify(obj, null, 2);
};
</script>

<style scoped>
.deployment-panel-v2 {
  padding: 24px;
  background: #f8f9fa;
  border-radius: 12px;
  max-height: 85vh;
  overflow-y: auto;
}

.panel-header {
  margin-bottom: 32px;
  text-align: center;
}

.panel-header h2 {
  font-size: 28px;
  margin-bottom: 8px;
  color: #2c3e50;
}

.panel-header p {
  color: #6c757d;
  font-size: 14px;
}

/* Upload Section */
.upload-section {
  background: white;
  padding: 24px;
  border-radius: 8px;
  margin-bottom: 24px;
}

.upload-section h3 {
  font-size: 18px;
  margin-bottom: 16px;
  color: #2c3e50;
}

.upload-area {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 16px;
}

.file-name {
  color: #28a745;
  font-weight: 500;
}

/* Diff Section */
.diff-section {
  margin-top: 24px;
}

.diff-section > h3 {
  font-size: 20px;
  margin-bottom: 20px;
  color: #2c3e50;
}

/* Feature Group */
.feature-group {
  background: white;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  border: 2px solid #e9ecef;
}

.feature-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-bottom: 16px;
  border-bottom: 2px solid #e9ecef;
  margin-bottom: 16px;
}

.feature-header h4 {
  font-size: 18px;
  color: #2c3e50;
  margin: 0;
}

.change-count {
  background: #6c757d;
  color: white;
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 13px;
  font-weight: 600;
}

/* Change Block */
.change-block {
  margin-bottom: 20px;
  border-radius: 8px;
  overflow: hidden;
  border: 2px solid #dee2e6;
}

.change-block:last-child {
  margin-bottom: 0;
}

/* Diff Header */
.diff-header {
  padding: 12px 16px;
  background: #f8f9fa;
  display: flex;
  align-items: center;
  gap: 12px;
  border-bottom: 1px solid #dee2e6;
}

.diff-header.conflict {
  background: #fff3cd;
  border-color: #ffc107;
}

.diff-header .icon {
  font-size: 20px;
}

.diff-header strong {
  flex: 1;
  color: #2c3e50;
}

.badge {
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.badge.new {
  background: #d4edda;
  color: #155724;
}

.badge.update {
  background: #d1ecf1;
  color: #0c5460;
}

.badge.conflict {
  background: #f8d7da;
  color: #721c24;
}

/* Diff Body */
.diff-body {
  padding: 16px;
  background: white;
}

/* New Addition */
.new-content {
  background: #f0fff4 !important;
  border-left: 4px solid #38a169;
}

.line-header {
  font-weight: 600;
  margin-bottom: 8px;
  color: #2c3e50;
}

/* Git Conflict Style */
.git-conflict-style {
  margin-bottom: 16px;
  border: 1px solid #dee2e6;
  border-radius: 6px;
  overflow: hidden;
}

.conflict-section {
  padding: 12px;
}

.conflict-section.current {
  background: #ffebee;
  border-left: 4px solid #e53e3e;
}

.conflict-section.incoming {
  background: #e8f5e9;
  border-left: 4px solid #38a169;
}

.section-label {
  font-weight: 600;
  margin-bottom: 8px;
  padding: 4px 8px;
  border-radius: 4px;
  display: inline-block;
}

.conflict-section.current .section-label {
  background: #ef5350;
  color: white;
}

.conflict-section.incoming .section-label {
  background: #66bb6a;
  color: white;
}

.conflict-label {
  font-family: 'Courier New', monospace;
  font-size: 13px;
  background: #495057;
  color: white;
  padding: 6px 12px;
  border-radius: 4px;
}

.conflict-divider {
  height: 2px;
  background: #dee2e6;
}

.conflict-separator {
  padding: 8px;
  background: #f8f9fa;
  text-align: center;
  font-family: 'Courier New', monospace;
  font-weight: bold;
  color: #6c757d;
}

.conflict-warning {
  background: #fff3cd;
  border: 1px solid #ffc107;
  border-radius: 6px;
  padding: 12px;
  margin-bottom: 16px;
  color: #856404;
  font-weight: 500;
}

/* Code Block */
.code-block {
  background: #2d3748;
  color: #e2e8f0;
  padding: 12px;
  border-radius: 6px;
  font-family: 'Courier New', monospace;
  font-size: 13px;
  line-height: 1.6;
  margin: 0;
  overflow-x: auto;
}

.conflict-section .code-block {
  background: transparent;
  color: #2c3e50;
  border: 1px solid rgba(0, 0, 0, 0.1);
}

/* Action Buttons */
.action-buttons {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  margin-top: 10px;
}

.btn {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 14px;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-upload {
  background: #6c757d;
  color: white;
}

.btn-upload:hover {
  background: #5a6268;
}

.btn-primary {
  background: #007bff;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #0056b3;
}

.btn-secondary {
  background: #6c757d;
  color: white;
}

.btn-secondary:hover:not(:disabled) {
  background: #5a6268;
}

.btn-accept {
  background: #28a745;
  color: white;
  border: 2px solid transparent;
}

.btn-accept:hover {
  background: #218838;
}

.btn-accept.selected {
  background: #1e7e34;
  border-color: #155724;
  box-shadow: 0 0 0 3px rgba(40, 167, 69, 0.3);
}

.btn-keep {
  background: #ffc107;
  color: #212529;
  border: 2px solid transparent;
}

.btn-keep:hover {
  background: #e0a800;
}

.btn-keep.selected {
  background: #d39e00;
  border-color: #c69500;
  box-shadow: 0 0 0 3px rgba(255, 193, 7, 0.3);
}

.btn-reject {
  background: #dc3545;
  color: white;
  border: 2px solid transparent;
}

.btn-reject:hover {
  background: #c82333;
}

.btn-reject.selected {
  background: #bd2130;
  border-color: #b21f2d;
  box-shadow: 0 0 0 3px rgba(220, 53, 69, 0.3);
}

.btn-both {
  background: #17a2b8;
  color: white;
  border: 2px solid transparent;
}

.btn-both:hover {
  background: #138496;
}

.btn-both.selected {
  background: #117a8b;
  border-color: #10707f;
  box-shadow: 0 0 0 3px rgba(23, 162, 184, 0.3);
}

.btn-manual {
  background: #6f42c1;
  color: white;
  border: 2px solid transparent;
}

.btn-manual:hover {
  background: #5a32a3;
}

.btn-manual.selected {
  background: #4e2a8e;
  border-color: #432377;
  box-shadow: 0 0 0 3px rgba(111, 66, 193, 0.3);
}

.btn-large {
  padding: 12px 24px;
  font-size: 16px;
}

/* Manual Edit */
.manual-edit-area {
  margin-top: 16px;
  padding: 16px;
  background: #f8f9fa;
  border-radius: 6px;
  border: 2px solid #6f42c1;
}

.manual-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.manual-header strong {
  color: #6f42c1;
}

.btn-close {
  background: transparent;
  border: none;
  color: #6c757d;
  cursor: pointer;
  padding: 4px 8px;
  font-size: 14px;
}

.btn-close:hover {
  color: #2c3e50;
}

.manual-textarea {
  width: 100%;
  padding: 12px;
  border: 1px solid #ced4da;
  border-radius: 6px;
  font-family: 'Courier New', monospace;
  font-size: 13px;
  resize: vertical;
  margin-bottom: 12px;
}

/* Apply Section */
.apply-section {
  background: white;
  padding: 24px;
  border-radius: 8px;
  margin-top: 24px;
  border: 2px solid #28a745;
}

.summary h4 {
  font-size: 18px;
  margin-bottom: 12px;
  color: #2c3e50;
}

.summary-stats {
  display: flex;
  gap: 24px;
  margin-bottom: 20px;
}

.stat-item {
  display: flex;
  gap: 8px;
}

.stat-label {
  color: #6c757d;
  font-weight: 500;
}

.stat-value {
  color: #28a745;
  font-weight: 700;
  font-size: 16px;
}

.apply-buttons {
  display: flex;
  gap: 12px;
  margin-bottom: 12px;
}

.warning-text {
  color: #dc3545;
  font-weight: 500;
  margin: 0;
  font-size: 14px;
}

/* Modal */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
}

.modal-content {
  background: white;
  border-radius: 12px;
  max-width: 90%;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
}

.preview-modal {
  width: 800px;
}

.modal-header {
  padding: 20px;
  border-bottom: 2px solid #e9ecef;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.modal-header h3 {
  margin: 0;
  font-size: 20px;
  color: #2c3e50;
}

.modal-header .btn-close {
  font-size: 24px;
  font-weight: bold;
}

.modal-body {
  padding: 20px;
  overflow-y: auto;
  flex: 1;
}

.preview-json {
  background: #2d3748;
  color: #e2e8f0;
  padding: 16px;
  border-radius: 8px;
  font-family: 'Courier New', monospace;
  font-size: 13px;
  line-height: 1.6;
  margin: 0;
  overflow-x: auto;
}
</style>

