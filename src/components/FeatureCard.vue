<template>
  <div 
    class="feature-card" 
    :class="{ selected: isSelected }"
    @click="$emit('select')"
  >
    <div class="feature-header">
      <strong class="feature-id">{{ feature.featureId }}</strong>
      <button 
        class="btn-icon btn-delete" 
        @click.stop="$emit('delete')"
        title="Feature 삭제"
      >
        🗑️
      </button>
    </div>
    <div class="feature-info">
      <span class="option-count">{{ optionCount }}개의 Option</span>
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue';

const props = defineProps({
  feature: {
    type: Object,
    required: true
  },
  isSelected: {
    type: Boolean,
    default: false
  }
});

defineEmits(['select', 'delete']);

const optionCount = computed(() => {
  return props.feature.option ? Object.keys(props.feature.option).length : 0;
});
</script>

<style scoped>
.feature-card {
  background: white;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 12px;
  cursor: pointer;
  transition: all 0.3s ease;
  border: 2px solid #e5e7eb;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.feature-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
  border-color: #9333ea;
}

.feature-card.selected {
  background: linear-gradient(135deg, #9333ea 0%, #a855f7 100%);
  color: white;
  border-color: #9333ea;
  transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(147, 51, 234, 0.3);
}

.feature-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.feature-id {
  font-size: 18px;
  font-weight: 700;
  text-transform: uppercase;
}

.feature-info {
  display: flex;
  gap: 8px;
  font-size: 13px;
  opacity: 0.8;
}

.option-count {
  background: rgba(255, 255, 255, 0.2);
  padding: 4px 10px;
  border-radius: 12px;
  font-weight: 500;
}

.feature-card:not(.selected) .option-count {
  background: #f3f4f6;
  color: #6b7280;
}

.btn-icon {
  background: none;
  border: none;
  font-size: 18px;
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

.feature-card:not(.selected) .btn-delete:hover {
  background: rgba(239, 68, 68, 0.1);
}
</style>

