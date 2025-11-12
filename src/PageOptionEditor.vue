<template>
  <div class="page-option-editor">
    <div class="header">
      <h1>📄 Page Option Editor</h1>
      <p>페이지 옵션 데이터 편집기</p>
    </div>

    <div class="editor-layout">
      <!-- 1. Feature 리스트 영역 -->
      <div class="feature-list-panel">
        <div class="panel-header">
          <h2>Feature 목록</h2>
          <button class="btn btn-add" @click="addFeature">
            ➕ Feature 추가
          </button>
        </div>
        <div class="feature-cards">
          <div
            v-for="(feature, index) in pageOptions"
            :key="feature._id?.$oid || index"
            class="feature-card"
            :class="{ active: selectedFeatureIndex === index }"
            @click="selectFeature(index)"
          >
            <div class="feature-card-header">
              <strong>{{ feature.featureId }}</strong>
              <button 
                class="btn-icon btn-delete" 
                @click.stop.prevent="deleteFeature(index, $event)" 
                title="삭제"
              >
                🗑️
              </button>
            </div>
            <div class="feature-card-info">
              옵션: {{ Object.keys(feature.option || {}).length }}개
            </div>
          </div>
          <div v-if="pageOptions.length === 0" class="empty-state">
            Feature가 없습니다. 추가해주세요.
          </div>
        </div>
      </div>

      <!-- 2. Option 리스트 영역 -->
      <div class="option-list-panel">
        <div class="panel-header">
          <h2>Option 목록</h2>
          <button 
            class="btn btn-add-template" 
            @click="addOptionFromTemplate"
            :disabled="selectedFeatureIndex === null"
          >
            ➕ 템플릿에서 추가
          </button>
        </div>
        <div v-if="selectedFeature" class="option-items">
          <div
            v-for="(option, key) in selectedFeature.option"
            :key="key"
            class="option-item"
            :class="{ active: selectedOptionKey === key }"
            @click="selectOption(key)"
          >
            <div class="option-item-header">
              <span class="option-key">{{ key }}</span>
              <button 
                class="btn-icon btn-delete" 
                @click.stop.prevent="deleteOption(key, $event)" 
                title="삭제"
              >
                🗑️
              </button>
            </div>
            <div class="option-item-desc">{{ option.desc }}</div>
            <div class="option-item-value">
              현재값: <code>{{ option.value }}</code>
            </div>
            <div class="option-item-list">
              리스트: {{ (option.list || []).length }}개 항목
            </div>
          </div>
          <div v-if="!selectedFeature.option || Object.keys(selectedFeature.option).length === 0" class="empty-state">
            Option이 없습니다. 추가해주세요.
          </div>
        </div>
        <div v-else class="empty-state">
          Feature를 선택해주세요.
        </div>
      </div>

      <!-- 3. Option 상세 편집 영역 -->
      <div class="option-detail-panel">
        <div class="panel-header">
          <h2>Option 상세 편집</h2>
        </div>
        <div v-if="selectedFeature && selectedOptionKey && selectedFeature.option && selectedFeature.option[selectedOptionKey]" class="detail-form">
          <div class="form-group">
            <label>Option Key</label>
            <input 
              type="text" 
              v-model="editingOptionKey" 
              class="form-input"
              @change="updateOptionKey"
            />
          </div>

          <div class="form-group">
            <label>설명 (desc)</label>
            <textarea 
              :value="selectedFeature.option[selectedOptionKey].desc"
              @input="selectedFeature.option[selectedOptionKey].desc = $event.target.value"
              class="form-textarea"
              rows="3"
            ></textarea>
          </div>

          <div class="form-group">
            <label>현재 값 (value)</label>
            <input 
              type="text" 
              :value="selectedFeature.option[selectedOptionKey].value"
              @input="selectedFeature.option[selectedOptionKey].value = $event.target.value"
              class="form-input"
            />
          </div>

          <div class="form-group">
            <div class="form-group-header">
              <label>리스트 항목 (템플릿 관리에서만 수정 가능)</label>
            </div>
            <div class="list-items readonly-list">
              <div 
                v-for="(item, index) in selectedFeature.option[selectedOptionKey].list" 
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
              <div v-if="!selectedFeature.option[selectedOptionKey].list || selectedFeature.option[selectedOptionKey].list.length === 0" class="empty-state-small">
                리스트 항목이 없습니다.
              </div>
            </div>
            <div class="info-message">
              💡 리스트 항목을 수정하려면 <strong>"📚 템플릿 관리"</strong>에서 템플릿을 수정하거나 새로운 템플릿을 만드세요.
            </div>
          </div>
        </div>
        <div v-else class="empty-state">
          Option을 선택해주세요.
        </div>
      </div>
    </div>

    <div class="action-bar">
      <button class="btn btn-primary" @click="saveChanges">
        💾 변경사항 저장
      </button>
      <button class="btn btn-secondary" @click="resetChanges">
        ↺ 초기화
      </button>
      <button class="btn btn-export" @click="exportJSON">
        📥 JSON 내보내기
      </button>
      <button class="btn btn-template" @click="showTemplateManager">
        📚 템플릿 관리
      </button>
    </div>

    <!-- 템플릿 선택 모달 -->
    <teleport to="body">
      <div v-if="showTemplateModal" class="modal-overlay" @click="closeTemplateModal">
        <div class="modal-content" @click.stop>
          <div class="modal-header">
            <h2>📋 Option 템플릿 선택</h2>
            <button class="btn-close" @click="closeTemplateModal">✕</button>
          </div>
          <div class="modal-body">
            <div class="template-search">
              <input 
                v-model="templateSearch" 
                type="text" 
                placeholder="템플릿 검색..." 
                class="form-input"
              />
            </div>
            <div class="template-list">
              <div 
                v-for="template in filteredTemplates" 
                :key="template.key"
                class="template-item"
                @click="selectTemplate(template)"
              >
                <div class="template-item-header">
                  <strong>{{ template.key }}</strong>
                  <span class="template-count">{{ template.count }}회 사용됨</span>
                </div>
                <div class="template-item-desc">{{ template.desc }}</div>
                <div class="template-item-preview">
                  리스트: {{ template.listCount }}개 항목
                </div>
              </div>
              <div v-if="filteredTemplates.length === 0" class="empty-state-small">
                검색 결과가 없습니다.
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 템플릿 관리 모달 -->
      <div v-if="showTemplateManagerModal" class="modal-overlay" @click="closeTemplateManagerModal">
        <div class="modal-content modal-large" @click.stop>
          <div class="modal-header">
            <h2>📚 템플릿 라이브러리</h2>
            <button class="btn-close" @click="closeTemplateManagerModal">✕</button>
          </div>
          <div class="modal-body">
            <div class="template-info">
              <p>현재 시스템에서 사용 중인 모든 Option 템플릿입니다. 새 템플릿을 추가하거나 관리할 수 있습니다.</p>
            </div>
            <div class="template-controls">
              <button class="btn btn-add" @click="showCreateTemplateModal = true">
                ➕ 새 템플릿 만들기
              </button>
              <input 
                v-model="templateManagerSearch" 
                type="text" 
                placeholder="템플릿 검색..." 
                class="form-input template-search-input"
              />
            </div>
            <div class="template-manager-list">
              <div 
                v-for="template in filteredManagerTemplates" 
                :key="template.key"
                class="template-manager-item"
              >
                <div class="template-manager-header">
                  <div>
                    <strong class="template-key">{{ template.key }}</strong>
                    <span class="template-badge">{{ template.count }}개 Feature에서 사용</span>
                  </div>
                </div>
                <div class="template-manager-body">
                  <div class="template-field">
                    <label>설명:</label>
                    <p>{{ template.desc }}</p>
                  </div>
                  <div class="template-field">
                    <label>기본값:</label>
                    <code>{{ template.defaultValue }}</code>
                  </div>
                  <div class="template-field">
                    <label>리스트 항목 ({{ template.listCount }}개):</label>
                    <ul class="template-list-preview">
                      <li v-for="(item, idx) in template.sampleList" :key="idx">
                        <code>{{ item.listValue }}</code> - {{ item.listDesc }}
                      </li>
                    </ul>
                  </div>
                  <div class="template-usage">
                    <label>사용 위치:</label>
                    <div class="usage-tags">
                      <span v-for="featureId in template.usedIn" :key="featureId" class="usage-tag">
                        {{ featureId }}
                      </span>
                    </div>
                  </div>
                </div>
              </div>
              <div v-if="filteredManagerTemplates.length === 0" class="empty-state-small">
                검색 결과가 없습니다.
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 템플릿 생성 모달 -->
      <div v-if="showCreateTemplateModal" class="modal-overlay" @click="closeCreateTemplateModal">
        <div class="modal-content" @click.stop>
          <div class="modal-header">
            <h2>➕ 새 템플릿 만들기</h2>
            <button class="btn-close" @click="closeCreateTemplateModal">✕</button>
          </div>
          <div class="modal-body">
            <div class="form-group">
              <label>템플릿 Key *</label>
              <input 
                v-model="newTemplate.key" 
                type="text" 
                placeholder="예: excelInputAbleOption"
                class="form-input"
              />
            </div>
            <div class="form-group">
              <label>설명 *</label>
              <textarea 
                v-model="newTemplate.desc" 
                placeholder="템플릿 설명을 입력하세요"
                class="form-textarea"
                rows="2"
              ></textarea>
            </div>
            <div class="form-group">
              <div class="form-group-header">
                <label>리스트 항목</label>
                <button class="btn btn-add-small" @click="addNewTemplateListItem">
                  ➕ 항목 추가
                </button>
              </div>
              <div class="list-items">
                <div 
                  v-for="(item, index) in newTemplate.list" 
                  :key="index"
                  class="list-item"
                >
                  <div class="list-item-fields">
                    <div class="list-field">
                      <label>listValue</label>
                      <input 
                        type="text" 
                        v-model="item.listValue" 
                        class="form-input-small"
                      />
                    </div>
                    <div class="list-field">
                      <label>listDesc</label>
                      <input 
                        type="text" 
                        v-model="item.listDesc" 
                        class="form-input-small"
                      />
                    </div>
                    <button 
                      class="btn-icon btn-delete" 
                      @click="deleteNewTemplateListItem(index)"
                      title="삭제"
                    >
                      🗑️
                    </button>
                  </div>
                </div>
              </div>
            </div>
            <div class="modal-actions">
              <button class="btn btn-secondary" @click="closeCreateTemplateModal">
                취소
              </button>
              <button class="btn btn-primary" @click="createTemplate">
                템플릿 생성
              </button>
            </div>
          </div>
        </div>
      </div>
    </teleport>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';

const pageOptions = ref([]);
const originalData = ref(null);
const selectedFeatureIndex = ref(null);
const selectedOptionKey = ref(null);
const editingOptionKey = ref('');

// 템플릿 관련 상태
const showTemplateModal = ref(false);
const showTemplateManagerModal = ref(false);
const showCreateTemplateModal = ref(false);
const templateSearch = ref('');
const templateManagerSearch = ref('');
const newTemplate = ref({
  key: '',
  desc: '',
  list: []
});

// 커스텀 템플릿 저장소 (localStorage 사용)
const customTemplates = ref([]);

// 선택된 Feature 계산
const selectedFeature = computed(() => {
  if (selectedFeatureIndex.value !== null && pageOptions.value[selectedFeatureIndex.value]) {
    return pageOptions.value[selectedFeatureIndex.value];
  }
  return null;
});

// 모든 option들을 템플릿으로 수집 (기존 템플릿 + 커스텀 템플릿)
const optionTemplates = computed(() => {
  const templates = {};
  
  // 기존 페이지 옵션에서 템플릿 수집
  pageOptions.value.forEach(feature => {
    if (!feature.option) return;
    
    Object.keys(feature.option).forEach(optionKey => {
      const option = feature.option[optionKey];
      
      if (!templates[optionKey]) {
        templates[optionKey] = {
          key: optionKey,
          desc: option.desc || '',
          list: option.list || [],
          defaultValue: (option.list && option.list.length > 0) ? option.list[0].listValue : '',
          count: 0,
          usedIn: [],
          listCount: (option.list || []).length,
          sampleList: option.list || [],
          isCustom: false
        };
      }
      
      templates[optionKey].count++;
      templates[optionKey].usedIn.push(feature.featureId);
    });
  });
  
  // 커스텀 템플릿 추가
  customTemplates.value.forEach(template => {
    if (!templates[template.key]) {
      templates[template.key] = {
        ...template,
        count: 0,
        usedIn: [],
        isCustom: true
      };
    }
  });
  
  return Object.values(templates).sort((a, b) => b.count - a.count);
});

// 필터링된 템플릿 (검색용)
const filteredTemplates = computed(() => {
  if (!templateSearch.value) return optionTemplates.value;
  
  const search = templateSearch.value.toLowerCase();
  return optionTemplates.value.filter(template => 
    template.key.toLowerCase().includes(search) ||
    template.desc.toLowerCase().includes(search)
  );
});

// 템플릿 관리자용 필터링
const filteredManagerTemplates = computed(() => {
  if (!templateManagerSearch.value) return optionTemplates.value;
  
  const search = templateManagerSearch.value.toLowerCase();
  return optionTemplates.value.filter(template => 
    template.key.toLowerCase().includes(search) ||
    template.desc.toLowerCase().includes(search)
  );
});

// 데이터 로드
onMounted(async () => {
  try {
    const response = await fetch('/data/dkant.pageOption.json');
    const data = await response.json();
    pageOptions.value = data;
    originalData.value = JSON.parse(JSON.stringify(data));
    
    // 커스텀 템플릿 로드
    loadCustomTemplates();
    
    ElMessage.success('데이터를 성공적으로 로드했습니다.');
  } catch (err) {
    console.error('데이터 로드 실패:', err);
    ElMessage.error('데이터 로드에 실패했습니다.');
  }
});

// 커스텀 템플릿 로드
const loadCustomTemplates = () => {
  try {
    const saved = localStorage.getItem('pageOptionCustomTemplates');
    if (saved) {
      customTemplates.value = JSON.parse(saved);
    }
  } catch (err) {
    console.error('커스텀 템플릿 로드 실패:', err);
  }
};

// 커스텀 템플릿 저장
const saveCustomTemplates = () => {
  try {
    localStorage.setItem('pageOptionCustomTemplates', JSON.stringify(customTemplates.value));
  } catch (err) {
    console.error('커스텀 템플릿 저장 실패:', err);
  }
};

// Feature 선택
const selectFeature = (index) => {
  selectedFeatureIndex.value = index;
  selectedOptionKey.value = null;
  editingOptionKey.value = '';
};

// Option 선택
const selectOption = (key) => {
  selectedOptionKey.value = key;
  editingOptionKey.value = key;
};

// Feature 추가
const addFeature = async () => {
  try {
    const { value } = await ElMessageBox.prompt('새 Feature ID를 입력하세요', 'Feature 추가', {
      confirmButtonText: '추가',
      cancelButtonText: '취소',
      inputPattern: /^[A-Z0-9]+$/,
      inputErrorMessage: 'Feature ID는 대문자와 숫자만 가능합니다',
      inputPlaceholder: '예: F99000',
    });

    if (!value || value.trim() === '') {
      ElMessage.error('Feature ID를 입력해주세요.');
      return;
    }

    // 이미 존재하는지 확인
    const exists = pageOptions.value.some(f => f.featureId === value);
    if (exists) {
      ElMessage.error('이미 존재하는 Feature ID입니다.');
      return;
    }

    const newFeature = {
      _id: {
        $oid: generateObjectId(),
      },
      featureId: value,
      option: {},
    };
    
    pageOptions.value.push(newFeature);
    selectedFeatureIndex.value = pageOptions.value.length - 1;
    selectedOptionKey.value = null;
    editingOptionKey.value = '';
    ElMessage.success(`Feature "${value}"가 추가되었습니다.`);
  } catch (err) {
    // 취소됨 또는 에러
    if (err !== 'cancel') {
      console.error('Feature 추가 오류:', err);
    }
  }
};

// Feature 삭제
const deleteFeature = async (index, event) => {
  // 이벤트 전파 중지 (이미 @click.stop이 있지만 추가 보장)
  if (event) {
    event.stopPropagation();
    event.preventDefault();
  }

  const feature = pageOptions.value[index];
  if (!feature) {
    ElMessage.error('삭제할 Feature를 찾을 수 없습니다.');
    return;
  }

  try {
    await ElMessageBox.confirm(
      `Feature "${feature.featureId}"를 삭제하시겠습니까?\n모든 옵션이 함께 삭제됩니다.`,
      '삭제 확인',
      {
        confirmButtonText: '삭제',
        cancelButtonText: '취소',
        type: 'warning',
      }
    );

    // 삭제 실행
    pageOptions.value.splice(index, 1);
    
    // 선택된 인덱스 업데이트
    if (selectedFeatureIndex.value === index) {
      selectedFeatureIndex.value = null;
      selectedOptionKey.value = null;
      editingOptionKey.value = '';
    } else if (selectedFeatureIndex.value > index) {
      selectedFeatureIndex.value--;
    }
    
    ElMessage.success('Feature가 삭제되었습니다.');
  } catch (err) {
    // 취소됨
    console.log('Feature 삭제 취소');
  }
};

// 템플릿에서 Option 추가
const addOptionFromTemplate = () => {
  if (!selectedFeature.value) {
    ElMessage.warning('Feature를 먼저 선택해주세요.');
    return;
  }

  if (optionTemplates.value.length === 0) {
    ElMessage.warning('사용 가능한 템플릿이 없습니다.');
    return;
  }

  showTemplateModal.value = true;
};

// 템플릿 선택
const selectTemplate = (template) => {
  if (!selectedFeature.value) return;

  // 이미 존재하는 키인지 확인
  if (selectedFeature.value.option && selectedFeature.value.option[template.key]) {
    ElMessageBox.alert(
      `"${template.key}" Option은 이미 이 Feature에서 사용 중입니다.\n중복으로 추가할 수 없습니다.`,
      '이미 사용 중인 옵션',
      {
        confirmButtonText: '확인',
        type: 'warning',
      }
    );
    return;
  }
  
  applyTemplate(template);
};

// 템플릿 적용
const applyTemplate = (template) => {
  if (!selectedFeature.value) return;

  // 직접 배열에서 feature 가져오기
  const feature = pageOptions.value[selectedFeatureIndex.value];
  
  if (!feature.option) {
    feature.option = {};
  }

  // list를 깊은 복사
  const listCopy = JSON.parse(JSON.stringify(template.list));
  
  // value는 list의 첫 번째 항목의 listValue로 설정
  const defaultValue = (listCopy.length > 0) ? listCopy[0].listValue : '';

  feature.option[template.key] = {
    desc: template.desc,
    value: defaultValue,
    list: listCopy,
  };

  selectedOptionKey.value = template.key;
  editingOptionKey.value = template.key;
  
  showTemplateModal.value = false;
  templateSearch.value = '';
  
  ElMessage.success(`템플릿 "${template.key}"가 적용되었습니다.`);
};

// 템플릿 모달 닫기
const closeTemplateModal = () => {
  showTemplateModal.value = false;
  templateSearch.value = '';
};

// 템플릿 관리자 열기
const showTemplateManager = () => {
  showTemplateManagerModal.value = true;
};

// 템플릿 관리자 모달 닫기
const closeTemplateManagerModal = () => {
  showTemplateManagerModal.value = false;
  templateManagerSearch.value = '';
};

// 템플릿 생성 모달 닫기
const closeCreateTemplateModal = () => {
  showCreateTemplateModal.value = false;
  newTemplate.value = {
    key: '',
    desc: '',
    list: []
  };
};

// 새 템플릿에 리스트 항목 추가
const addNewTemplateListItem = () => {
  newTemplate.value.list.push({
    listValue: '',
    listDesc: ''
  });
};

// 새 템플릿에서 리스트 항목 삭제
const deleteNewTemplateListItem = (index) => {
  newTemplate.value.list.splice(index, 1);
};

// 템플릿 생성
const createTemplate = () => {
  if (!newTemplate.value.key || !newTemplate.value.key.trim()) {
    ElMessage.error('템플릿 Key를 입력해주세요.');
    return;
  }

  if (!newTemplate.value.key.match(/^[a-zA-Z0-9_]+$/)) {
    ElMessage.error('템플릿 Key는 영문, 숫자, 언더스코어만 가능합니다.');
    return;
  }

  if (!newTemplate.value.desc || !newTemplate.value.desc.trim()) {
    ElMessage.error('템플릿 설명을 입력해주세요.');
    return;
  }

  // 중복 체크
  const exists = customTemplates.value.some(t => t.key === newTemplate.value.key);
  if (exists) {
    ElMessage.error('이미 존재하는 템플릿 Key입니다.');
    return;
  }

  const template = {
    key: newTemplate.value.key.trim(),
    desc: newTemplate.value.desc.trim(),
    list: JSON.parse(JSON.stringify(newTemplate.value.list)),
    defaultValue: (newTemplate.value.list.length > 0) ? newTemplate.value.list[0].listValue : '',
    listCount: newTemplate.value.list.length,
    sampleList: JSON.parse(JSON.stringify(newTemplate.value.list))
  };

  customTemplates.value.push(template);
  saveCustomTemplates();
  
  closeCreateTemplateModal();
  ElMessage.success(`템플릿 "${template.key}"가 생성되었습니다.`);
};

// Option 삭제
const deleteOption = async (key, event) => {
  // 이벤트 전파 중지
  if (event) {
    event.stopPropagation();
    event.preventDefault();
  }

  try {
    await ElMessageBox.confirm(
      `Option "${key}"를 삭제하시겠습니까?`,
      '삭제 확인',
      {
        confirmButtonText: '삭제',
        cancelButtonText: '취소',
        type: 'warning',
      }
    );

    const feature = pageOptions.value[selectedFeatureIndex.value];
    if (feature && feature.option) {
      delete feature.option[key];
      
      if (selectedOptionKey.value === key) {
        selectedOptionKey.value = null;
        editingOptionKey.value = '';
      }
      
      ElMessage.success('Option이 삭제되었습니다.');
    }
  } catch (err) {
    // 취소됨
    console.log('Option 삭제 취소');
  }
};

// Option Key 변경
const updateOptionKey = () => {
  if (editingOptionKey.value === selectedOptionKey.value) return;
  
  if (!selectedFeature.value || !selectedOptionKey.value) return;

  if (!editingOptionKey.value.match(/^[a-zA-Z0-9_]+$/)) {
    ElMessage.error('Option Key는 영문, 숫자, 언더스코어만 가능합니다.');
    editingOptionKey.value = selectedOptionKey.value;
    return;
  }

  const feature = pageOptions.value[selectedFeatureIndex.value];
  if (feature.option[editingOptionKey.value]) {
    ElMessage.error('이미 존재하는 Option Key입니다.');
    editingOptionKey.value = selectedOptionKey.value;
    return;
  }

  const oldKey = selectedOptionKey.value;
  const newKey = editingOptionKey.value;
  const optionData = feature.option[oldKey];

  // 새 키로 복사하고 이전 키 삭제
  feature.option[newKey] = optionData;
  delete feature.option[oldKey];

  selectedOptionKey.value = newKey;
  ElMessage.success(`Option Key가 "${oldKey}"에서 "${newKey}"로 변경되었습니다.`);
};

// List Item 추가
const addListItem = () => {
  if (!selectedFeature.value || !selectedOptionKey.value) return;
  
  const option = selectedFeature.value.option[selectedOptionKey.value];
  if (!option) return;
  
  if (!option.list) {
    option.list = [];
  }
  
  option.list.push({
    listValue: '',
    listDesc: '',
  });
};

// List Item 삭제
const deleteListItem = (index) => {
  if (!selectedFeature.value || !selectedOptionKey.value) return;
  
  const option = selectedFeature.value.option[selectedOptionKey.value];
  if (!option || !option.list) return;
  
  option.list.splice(index, 1);
};

// 변경사항 저장
const saveChanges = () => {
  ElMessageBox.confirm(
    '변경사항을 저장하시겠습니까?\n(실제 파일은 저장되지 않으며, 브라우저 세션에만 저장됩니다)',
    '저장 확인',
    {
      confirmButtonText: '저장',
      cancelButtonText: '취소',
      type: 'info',
    }
  ).then(() => {
    originalData.value = JSON.parse(JSON.stringify(pageOptions.value));
    ElMessage.success('변경사항이 저장되었습니다.');
  }).catch(() => {
    // 취소됨
  });
};

// 초기화
const resetChanges = () => {
  ElMessageBox.confirm(
    '모든 변경사항을 취소하고 원본 데이터로 되돌리시겠습니까?',
    '초기화 확인',
    {
      confirmButtonText: '초기화',
      cancelButtonText: '취소',
      type: 'warning',
    }
  ).then(() => {
    pageOptions.value = JSON.parse(JSON.stringify(originalData.value));
    selectedFeatureIndex.value = null;
    selectedOptionKey.value = null;
    editingOptionKey.value = '';
    ElMessage.success('원본 데이터로 초기화되었습니다.');
  }).catch(() => {
    // 취소됨
  });
};

// JSON 내보내기
const exportJSON = () => {
  const json = JSON.stringify(pageOptions.value, null, 2);
  const blob = new Blob([json], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'pageOption_export.json';
  a.click();
  URL.revokeObjectURL(url);
  ElMessage.success('JSON 파일이 다운로드되었습니다.');
};

// ObjectId 생성 헬퍼
const generateObjectId = () => {
  const timestamp = Math.floor(Date.now() / 1000).toString(16);
  const random = Math.random().toString(16).substring(2, 18);
  return (timestamp + random).padEnd(24, '0');
};
</script>

<style scoped>
.page-option-editor {
  max-width: 1800px;
  margin: 0 auto;
  padding: 20px;
  background: #f5f5f5;
  min-height: 1000px;
}

.header {
  text-align: center;
  margin-bottom: 30px;
  padding: 20px;
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.header h1 {
  margin: 0;
  color: #333;
  font-size: 32px;
}

.header p {
  margin: 10px 0 0;
  color: #666;
  font-size: 16px;
}

.editor-layout {
  display: grid;
  grid-template-columns: 300px 400px 1fr;
  gap: 20px;
  margin-bottom: 20px;
}

/* 패널 공통 스타일 */
.feature-list-panel,
.option-list-panel,
.option-detail-panel {
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  overflow: hidden;
  display: flex;
  flex-direction: column;
  max-height: 700px;
}

.panel-header {
  padding: 20px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.panel-header h2 {
  margin: 0;
  font-size: 18px;
  font-weight: 600;
}

/* Feature 카드 */
.feature-cards {
  padding: 15px;
  overflow-y: auto;
  flex: 1;
}

.feature-card {
  background: #f8f9fa;
  border: 2px solid #e9ecef;
  border-radius: 8px;
  padding: 15px;
  margin-bottom: 10px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.feature-card:hover {
  border-color: #667eea;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.2);
}

.feature-card.active {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-color: #667eea;
  color: white;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.3);
}

.feature-card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.feature-card-header strong {
  font-size: 16px;
}

.feature-card-info {
  font-size: 13px;
  opacity: 0.8;
}

/* Option 아이템 */
.option-items {
  padding: 15px;
  overflow-y: auto;
  flex: 1;
}

.option-item {
  background: #f8f9fa;
  border: 2px solid #e9ecef;
  border-radius: 8px;
  padding: 15px;
  margin-bottom: 10px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.option-item:hover {
  border-color: #667eea;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.2);
}

.option-item.active {
  background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
  border-color: #f5576c;
  color: white;
  box-shadow: 0 4px 12px rgba(245, 87, 108, 0.3);
}

.option-item-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.option-key {
  font-weight: 600;
  font-size: 14px;
}

.option-item-desc {
  font-size: 13px;
  margin-bottom: 6px;
  opacity: 0.9;
}

.option-item-value {
  font-size: 12px;
  margin-bottom: 4px;
}

.option-item-value code {
  background: rgba(0,0,0,0.1);
  padding: 2px 6px;
  border-radius: 4px;
  font-family: monospace;
}

.option-item.active .option-item-value code {
  background: rgba(255,255,255,0.2);
}

.option-item-list {
  font-size: 12px;
  opacity: 0.8;
}

/* 상세 편집 폼 */
.detail-form {
  padding: 20px;
  overflow-y: auto;
  flex: 1;
}

.form-group {
  margin-bottom: 20px;
}

.form-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: 600;
  color: #333;
  font-size: 14px;
}

.form-group-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.form-input,
.form-textarea {
  width: 100%;
  padding: 10px;
  border: 2px solid #e9ecef;
  border-radius: 6px;
  font-size: 14px;
  transition: border-color 0.3s ease;
}

.form-input:focus,
.form-textarea:focus {
  outline: none;
  border-color: #667eea;
}

.form-textarea {
  resize: vertical;
  font-family: inherit;
}

/* 리스트 아이템 */
.list-items {
  max-height: 300px;
  overflow-y: auto;
}

.list-item {
  background: #f8f9fa;
  border: 1px solid #dee2e6;
  border-radius: 6px;
  padding: 12px;
  margin-bottom: 10px;
}

.list-item.readonly {
  background: #e9ecef;
  border: 1px solid #ced4da;
  cursor: not-allowed;
}

.list-item-fields {
  display: grid;
  grid-template-columns: 1fr 2fr auto;
  gap: 10px;
  align-items: end;
}

.list-item-fields-readonly {
  display: grid;
  grid-template-columns: 1fr 2fr;
  gap: 15px;
}

.list-field label {
  display: block;
  font-size: 12px;
  color: #666;
  margin-bottom: 4px;
}

.list-field-readonly label {
  display: block;
  font-size: 12px;
  color: #666;
  margin-bottom: 4px;
  font-weight: 600;
}

.readonly-value {
  padding: 8px;
  background: white;
  border-radius: 4px;
  font-size: 13px;
  color: #495057;
  border: 1px solid #ced4da;
  font-family: monospace;
}

.readonly-list {
  border: 2px dashed #ced4da;
  border-radius: 8px;
  padding: 10px;
  background: #f8f9fa;
}

.info-message {
  margin-top: 15px;
  padding: 12px;
  background: #fff3cd;
  border-left: 4px solid #ffc107;
  border-radius: 4px;
  font-size: 13px;
  color: #856404;
}

.info-message strong {
  color: #664d03;
}

.form-input-small {
  width: 100%;
  padding: 8px;
  border: 1px solid #ced4da;
  border-radius: 4px;
  font-size: 13px;
}

.form-input-small:focus {
  outline: none;
  border-color: #667eea;
}

/* 버튼 스타일 */
.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-add {
  background: #10b981;
  color: white;
  padding: 8px 16px;
  font-size: 13px;
}

.btn-add:hover:not(:disabled) {
  background: #059669;
}

.btn-add-small {
  background: #10b981;
  color: white;
  padding: 6px 12px;
  font-size: 12px;
}

.btn-add-small:hover {
  background: #059669;
}

.btn-add-template {
  background: #3b82f6;
  color: white;
  padding: 8px 16px;
  font-size: 13px;
}

.btn-add-template:hover:not(:disabled) {
  background: #2563eb;
}

.btn-template {
  background: #8b5cf6;
  color: white;
}

.btn-template:hover {
  background: #7c3aed;
}

.btn-icon {
  background: transparent;
  border: none;
  cursor: pointer;
  font-size: 16px;
  padding: 4px;
  transition: transform 0.2s ease;
}

.btn-icon:hover {
  transform: scale(1.2);
}

.btn-delete {
  opacity: 0.7;
}

.btn-delete:hover {
  opacity: 1;
}

/* 모달 스타일 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
  backdrop-filter: blur(4px);
}

.modal-content {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  width: 90%;
  max-width: 700px;
  max-height: 80vh;
  display: flex;
  flex-direction: column;
  animation: modalSlideIn 0.3s ease;
}

.modal-large {
  max-width: 1000px;
}

@keyframes modalSlideIn {
  from {
    opacity: 0;
    transform: translateY(-50px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.modal-header {
  padding: 25px 30px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-radius: 12px 12px 0 0;
}

.modal-header h2 {
  margin: 0;
  font-size: 22px;
}

.btn-close {
  background: rgba(255, 255, 255, 0.2);
  border: none;
  color: white;
  font-size: 24px;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

.btn-close:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: rotate(90deg);
}

.modal-body {
  padding: 25px 30px;
  overflow-y: auto;
  flex: 1;
}

.modal-actions {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid #e9ecef;
}

.template-info {
  background: #e3f2fd;
  border-left: 4px solid #2196f3;
  padding: 12px 15px;
  border-radius: 4px;
  margin-bottom: 20px;
  color: #1565c0;
}

.template-info p {
  margin: 0;
  font-size: 14px;
}

.template-controls {
  display: flex;
  gap: 15px;
  margin-bottom: 20px;
  align-items: center;
}

.template-search-input {
  flex: 1;
}

.template-search {
  margin-bottom: 20px;
}

.template-list {
  display: grid;
  gap: 12px;
  max-height: 400px;
  overflow-y: auto;
}

.template-item {
  background: #f8f9fa;
  border: 2px solid #e9ecef;
  border-radius: 8px;
  padding: 15px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.template-item:hover {
  border-color: #667eea;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.2);
  transform: translateY(-2px);
}

.template-item-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.template-item-header strong {
  font-size: 16px;
  color: #333;
}

.template-count {
  background: #667eea;
  color: white;
  padding: 3px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.template-item-desc {
  font-size: 13px;
  color: #666;
  margin-bottom: 6px;
}

.template-item-preview {
  font-size: 12px;
  color: #999;
}

/* 템플릿 관리자 스타일 */
.template-manager-list {
  display: grid;
  gap: 20px;
  max-height: 500px;
  overflow-y: auto;
}

.template-manager-item {
  background: #f8f9fa;
  border: 2px solid #e9ecef;
  border-radius: 8px;
  padding: 20px;
}

.template-manager-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
  padding-bottom: 15px;
  border-bottom: 2px solid #dee2e6;
}

.template-key {
  font-size: 18px;
  color: #667eea;
  margin-right: 10px;
}

.template-badge {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.template-manager-body {
  display: grid;
  gap: 15px;
}

.template-field {
  background: white;
  padding: 12px;
  border-radius: 6px;
  border: 1px solid #dee2e6;
}

.template-field label {
  display: block;
  font-weight: 600;
  color: #333;
  margin-bottom: 6px;
  font-size: 13px;
}

.template-field p {
  margin: 0;
  color: #666;
  font-size: 14px;
}

.template-field code {
  background: #e3f2fd;
  color: #1565c0;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 13px;
  font-family: 'Monaco', 'Menlo', monospace;
}

.template-list-preview {
  list-style: none;
  padding: 0;
  margin: 8px 0 0;
}

.template-list-preview li {
  padding: 8px;
  background: white;
  border-radius: 4px;
  margin-bottom: 6px;
  font-size: 13px;
  border-left: 3px solid #667eea;
}

.template-usage {
  background: white;
  padding: 12px;
  border-radius: 6px;
  border: 1px solid #dee2e6;
}

.usage-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 8px;
}

.usage-tag {
  background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
  color: white;
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.btn-primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.btn-secondary {
  background: #6c757d;
  color: white;
}

.btn-secondary:hover {
  background: #5a6268;
}

.btn-export {
  background: #f59e0b;
  color: white;
}

.btn-export:hover {
  background: #d97706;
}

/* 액션 바 */
.action-bar {
  display: flex;
  gap: 15px;
  justify-content: center;
  padding: 20px;
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

/* 빈 상태 */
.empty-state {
  text-align: center;
  padding: 40px 20px;
  color: #999;
  font-size: 14px;
}

.empty-state-small {
  text-align: center;
  padding: 20px;
  color: #999;
  font-size: 13px;
}

/* 반응형 */
@media (max-width: 1400px) {
  .editor-layout {
    grid-template-columns: 250px 350px 1fr;
  }
}

@media (max-width: 1200px) {
  .editor-layout {
    grid-template-columns: 1fr;
    grid-template-rows: auto auto auto;
  }

  .feature-list-panel,
  .option-list-panel,
  .option-detail-panel {
    max-height: 400px;
  }
}
</style>
