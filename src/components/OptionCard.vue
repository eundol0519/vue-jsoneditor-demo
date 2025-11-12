<template>
  <div 
    class="option-card" 
    :class="{ selected: isSelected }"
    @click="$emit('select')"
  >
    <div class="option-header">
      <strong class="option-key">{{ optionKey }}</strong>
      <button 
        class="btn-icon btn-delete" 
        @click.stop="$emit('delete')"
        title="Option 삭제"
      >
        🗑️
      </button>
    </div>
    <div class="option-desc">{{ option.desc || '설명 없음' }}</div>
    <div class="option-value">
      <span class="value-label">현재 값:</span>
      <code>{{ option.value }}</code>
    </div>
  </div>
</template>

<script setup>
defineProps({
  optionKey: {
    type: String,
    required: true
  },
  option: {
    type: Object,
    required: true
  },
  isSelected: {
    type: Boolean,
    default: false
  }
});

defineEmits(['select', 'delete']);
</script>

<style scoped>
.option-card {
  background: white;
  border-radius: 10px;
  padding: 16px;
  margin-bottom: 10px;
  cursor: pointer;
  transition: all 0.3s ease;
  border: 2px solid #e5e7eb;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.05);
}

.option-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.1);
  border-color: #ec4899;
}

.option-card.selected {
  background: linear-gradient(135deg, #ec4899 0%, #f472b6 100%);
  color: white;
  border-color: #ec4899;
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(236, 72, 153, 0.3);
}

.option-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.option-key {
  font-size: 16px;
  font-weight: 600;
}

.option-desc {
  font-size: 13px;
  opacity: 0.85;
  margin-bottom: 8px;
  line-height: 1.4;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.option-value {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
}

.value-label {
  opacity: 0.7;
  font-weight: 500;
}

code {
  background: rgba(255, 255, 255, 0.2);
  padding: 2px 8px;
  border-radius: 4px;
  font-family: 'Courier New', monospace;
  font-size: 11px;
}

.option-card:not(.selected) code {
  background: #f3f4f6;
  color: #374151;
}

.btn-icon {
  background: none;
  border: none;
  font-size: 16px;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 6px;
  transition: all 0.2s ease;
  opacity: 0.6;
}

.btn-icon:hover {
  opacity: 1;
  background: rgba(255, 255, 255, 0.2);
  transform: scale(1.1);
}

.btn-delete:hover {
  background: rgba(239, 68, 68, 0.2);
}

.option-card:not(.selected) .btn-delete:hover {
  background: rgba(239, 68, 68, 0.1);
}
</style>

