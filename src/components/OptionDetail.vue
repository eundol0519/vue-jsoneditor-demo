<template>
  <div class="option-detail-panel">
    <div class="panel-header">
      <h2>Option 상세 편집</h2>
    </div>
    <div v-if="editingData" class="detail-form">
      <div class="form-group">
        <label>Option Key</label>
        <input 
          type="text" 
          :value="editingKey" 
          @input="$emit('update:editingKey', $event.target.value)"
          @change="$emit('keyChange')"
          class="form-input"
        />
      </div>

      <div class="form-group">
        <label>설명 (desc)</label>
        <textarea 
          :value="editingData.desc"
          @input="handleInput('desc', $event.target.value)"
          class="form-textarea"
          rows="3"
        ></textarea>
      </div>

      <div class="form-group">
        <label>현재 값 (value)</label>
        <input 
          type="text" 
          :value="editingData.value"
          @input="handleInput('value', $event.target.value)"
          class="form-input"
        />
      </div>

      <div class="form-group">
        <div class="form-group-header">
          <label>리스트 항목 (템플릿 관리에서만 수정 가능)</label>
        </div>
        <div class="list-items readonly-list">
          <div 
            v-for="(item, index) in editingData.list" 
            :key="index"
            class="list-item readonly"
          >
            <div class="list-item-fields-readonly">
              <div class="list-field-readonly">
                <label>listValue</label>
                <div class="readonly-value">{{ item.listValue }}</div>
              </div>
              <div class="list-field-readonly">
                <label>listDesc</label>
                <div class="readonly-value">{{ item.listDesc }}</div>
              </div>
            </div>
          </div>
          <div v-if="!editingData.list || editingData.list.length === 0" class="empty-state-small">
            리스트 항목이 없습니다.
          </div>
        </div>
        <div class="info-message">
          💡 리스트 항목을 수정하려면 <strong>"📚 템플릿 관리"</strong>에서 템플릿을 수정하거나 새로운 템플릿을 만드세요.
        </div>
      </div>

      <!-- 저장/초기화 버튼 -->
      <div class="option-edit-actions">
        <button 
          class="btn btn-option-save" 
          @click="$emit('save')"
          :disabled="!hasChanges"
        >
          💾 Option 저장
        </button>
        <button 
          class="btn btn-option-reset" 
          @click="$emit('reset')"
          :disabled="!hasChanges"
        >
          ↺ Option 초기화
        </button>
      </div>
    </div>
    <div v-else class="empty-state">
      Option을 선택해주세요.
    </div>
  </div>
</template>

<script setup>
const props = defineProps({
  editingData: {
    type: Object,
    default: null
  },
  editingKey: {
    type: String,
    default: ''
  },
  hasChanges: {
    type: Boolean,
    default: false
  }
});

const emit = defineEmits(['update:editingKey', 'update:editingData', 'dataChange', 'keyChange', 'save', 'reset']);

const handleInput = (field, value) => {
  const updated = { ...props.editingData, [field]: value };
  emit('update:editingData', updated);
  emit('dataChange');
};
</script>

<style scoped>
.option-detail-panel {
  background: white;
  border-radius: 16px;
  padding: 24px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  overflow-y: auto;
}

.panel-header h2 {
  font-size: 20px;
  font-weight: 700;
  color: #1f2937;
  margin: 0 0 20px 0;
}

.detail-form {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.form-group label {
  font-weight: 600;
  color: #374151;
  font-size: 14px;
}

.form-input,
.form-textarea {
  padding: 10px 14px;
  border: 2px solid #e5e7eb;
  border-radius: 8px;
  font-size: 14px;
  transition: all 0.2s ease;
  font-family: inherit;
}

.form-input:focus,
.form-textarea:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-textarea {
  resize: vertical;
  min-height: 80px;
}

.list-items {
  display: flex;
  flex-direction: column;
  gap: 10px;
  max-height: 300px;
  overflow-y: auto;
}

.readonly-list .list-item {
  background: #f9fafb;
  border: 2px solid #e5e7eb;
  padding: 12px;
  border-radius: 8px;
}

.list-item-fields-readonly {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.list-field-readonly {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.list-field-readonly label {
  font-size: 12px;
  font-weight: 600;
  color: #6b7280;
  text-transform: uppercase;
}

.readonly-value {
  padding: 8px 12px;
  background: white;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 14px;
  color: #374151;
}

.empty-state,
.empty-state-small {
  text-align: center;
  padding: 40px 20px;
  color: #9ca3af;
  font-size: 14px;
}

.empty-state-small {
  padding: 20px;
}

.info-message {
  background-color: #e8f4fd;
  border-left: 4px solid #409eff;
  padding: 12px 16px;
  margin-top: 12px;
  border-radius: 4px;
  color: #606266;
  font-size: 14px;
}

.info-message strong {
  color: #409eff;
  font-weight: 600;
}

.option-edit-actions {
  display: flex;
  gap: 12px;
  margin-top: 24px;
  padding-top: 16px;
  border-top: 2px solid #e4e7ed;
}

.btn {
  flex: 1;
  padding: 12px 20px;
  border: none;
  border-radius: 8px;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
}

.btn-option-save {
  background: linear-gradient(135deg, #67c23a 0%, #85ce61 100%);
  color: white;
}

.btn-option-save:hover:not(:disabled) {
  background: linear-gradient(135deg, #85ce61 0%, #67c23a 100%);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(103, 194, 58, 0.4);
}

.btn-option-reset {
  background: linear-gradient(135deg, #909399 0%, #b3b8bd 100%);
  color: white;
}

.btn-option-reset:hover:not(:disabled) {
  background: linear-gradient(135deg, #b3b8bd 0%, #909399 100%);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(144, 147, 153, 0.4);
}

.btn:disabled {
  background: #f5f7fa;
  color: #c0c4cc;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}
</style>

