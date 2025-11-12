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
          <FeatureCard
            v-for="(feature, index) in pageOptions"
            :key="feature._id?.$oid || index"
            :feature="feature"
            :is-selected="selectedFeatureIndex === index"
            @select="selectFeature(index)"
            @delete="deleteFeature(index, $event)"
          />
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
          <OptionCard
            v-for="(option, key) in selectedFeature.option"
            :key="key"
            :option-key="key"
            :option="option"
            :is-selected="selectedOptionKey === key"
            @select="selectOption(key)"
            @delete="deleteOption(key, $event)"
          />
          <div v-if="!selectedFeature.option || Object.keys(selectedFeature.option).length === 0" class="empty-state">
            Option이 없습니다. 추가해주세요.
          </div>
        </div>
        <div v-else class="empty-state">
          Feature를 선택해주세요.
        </div>
      </div>

      <!-- 3. Option 상세 편집 영역 -->
      <OptionDetail
        v-model:editingKey="editingOptionKey"
        v-model:editingData="editingOptionData"
        :has-changes="hasOptionChanges"
        @data-change="hasOptionChanges = true"
        @key-change="updateOptionKey"
        @save="saveOptionChanges"
        @reset="resetOptionChanges"
      />
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
                <template v-if="templateSearch">
                  검색 결과가 없습니다.
                </template>
                <template v-else>
                  추가 가능한 템플릿이 없습니다.<br/>
                  <small>현재 Feature에 모든 옵션이 추가되었거나, 새 템플릿을 만들어야 합니다.</small>
                </template>
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
                :class="{ expanded: expandedTemplates.includes(template.key) }"
              >
                <div class="template-manager-header" @click="toggleTemplate(template.key)">
                  <div class="template-header-left">
                    <span class="expand-icon">{{ expandedTemplates.includes(template.key) ? '▼' : '▶' }}</span>
                    <strong class="template-key">{{ template.key }}</strong>
                    <span class="template-badge">{{ template.count }}개 Feature에서 사용</span>
                  </div>
                  <div class="template-actions" @click.stop>
                    <button class="btn btn-edit-small" @click="editTemplate(template)" title="수정">
                      ✏️ 수정
                    </button>
                    <button 
                      class="btn btn-delete-small" 
                      @click="deleteTemplate(template)" 
                      title="삭제"
                    >
                      🗑️ 삭제
                    </button>
                  </div>
                </div>
                <transition name="slide-down">
                  <div v-if="expandedTemplates.includes(template.key)" class="template-manager-body">
                    <div class="template-field">
                      <label>설명:</label>
                      <p>{{ template.desc || '설명 없음' }}</p>
                    </div>
                    <div class="template-field">
                      <label>기본값:</label>
                      <code>{{ template.defaultValue || '값 없음' }}</code>
                    </div>
                    <div class="template-field">
                      <label>리스트 항목 ({{ template.listCount || 0 }}개):</label>
                      <ul class="template-list-preview" v-if="template.sampleList && template.sampleList.length > 0">
                        <li v-for="(item, idx) in template.sampleList" :key="idx">
                          <code>{{ item.listValue }}</code> - {{ item.listDesc }}
                        </li>
                      </ul>
                      <p v-else class="no-usage">리스트 항목이 없습니다</p>
                    </div>
                    <div class="template-usage">
                      <label>사용 위치:</label>
                      <div class="usage-tags">
                        <span v-for="featureId in template.usedIn" :key="featureId" class="usage-tag">
                          {{ featureId }}
                        </span>
                        <span v-if="!template.usedIn || template.usedIn.length === 0" class="no-usage">사용 중인 Feature 없음</span>
                      </div>
                    </div>
                  </div>
                </transition>
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
            <h2>{{ editingTemplate ? '✏️ 템플릿 수정' : '➕ 새 템플릿 만들기' }}</h2>
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
                :disabled="!!editingTemplate"
              />
              <div v-if="editingTemplate" class="input-hint">
                템플릿 Key는 수정할 수 없습니다.
              </div>
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
                <label>리스트 항목 * (최소 1개 필수)</label>
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
                        placeholder="값을 입력하세요"
                      />
                    </div>
                    <div class="list-field">
                      <label>listDesc</label>
                      <input 
                        type="text" 
                        v-model="item.listDesc" 
                        class="form-input-small"
                        placeholder="설명을 입력하세요"
                      />
                    </div>
                    <button 
                      class="btn-icon btn-delete" 
                      @click="deleteNewTemplateListItem(index)"
                      title="삭제"
                      :disabled="newTemplate.list.length <= 1"
                    >
                      🗑️
                    </button>
                  </div>
                </div>
              </div>
              <div v-if="newTemplate.list.length === 0" class="empty-state-small">
                최소 1개의 리스트 항목이 필요합니다.
              </div>
            </div>
            <div class="form-group">
              <label>기본값 (value) *</label>
              <select 
                v-model="newTemplate.value" 
                class="form-input"
                :disabled="newTemplate.list.length === 0"
              >
                <option value="">리스트 항목 중 선택하세요</option>
                <option 
                  v-for="(item, index) in newTemplate.list" 
                  :key="index"
                  :value="item.listValue"
                >
                  {{ item.listValue }} - {{ item.listDesc }}
                </option>
              </select>
              <div class="input-hint" v-if="newTemplate.list.length === 0">
                먼저 리스트 항목을 추가하세요.
              </div>
            </div>
            <div class="modal-actions">
              <button class="btn btn-secondary" @click="closeCreateTemplateModal">
                취소
              </button>
              <button class="btn btn-primary" @click="editingTemplate ? updateTemplate() : createTemplate()">
                {{ editingTemplate ? '수정 완료' : '템플릿 생성' }}
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
import FeatureCard from './components/FeatureCard.vue';
import OptionCard from './components/OptionCard.vue';
import OptionDetail from './components/OptionDetail.vue';

const pageOptions = ref([]);
const originalData = ref(null);
const selectedFeatureIndex = ref(null);
const selectedOptionKey = ref(null);
const editingOptionKey = ref('');
const editingOptionData = ref(null); // 편집 중인 Option 데이터
const hasOptionChanges = ref(false); // Option 변경 여부

// 템플릿 관련 상태
const showTemplateModal = ref(false);
const showTemplateManagerModal = ref(false);
const showCreateTemplateModal = ref(false);
const templateSearch = ref('');
const templateManagerSearch = ref('');
const editingTemplate = ref(null);
const expandedTemplates = ref([]); // 확장된 템플릿 추적 (배열로 변경)
const newTemplate = ref({
  key: '',
  desc: '',
  list: [],
  value: ''
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
  // 현재 선택된 Feature에서 이미 사용 중인 옵션 키 목록
  const existingKeys = new Set();
  if (selectedFeatureIndex.value !== null) {
    const feature = pageOptions.value[selectedFeatureIndex.value];
    if (feature && feature.option) {
      Object.keys(feature.option).forEach(key => existingKeys.add(key));
    }
  }

  // 사용하지 않은 템플릿만 필터링
  let availableTemplates = optionTemplates.value.filter(template => 
    !existingKeys.has(template.key)
  );

  // 검색어로 추가 필터링
  if (templateSearch.value) {
    const search = templateSearch.value.toLowerCase();
    availableTemplates = availableTemplates.filter(template => 
      template.key.toLowerCase().includes(search) ||
      template.desc.toLowerCase().includes(search)
    );
  }

  return availableTemplates;
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
  // 변경사항이 있으면 경고
  if (hasOptionChanges.value) {
    ElMessageBox.confirm(
      '저장하지 않은 변경사항이 있습니다.\n변경사항을 버리고 다른 Feature를 선택하시겠습니까?',
      '변경사항 확인',
      {
        confirmButtonText: '예',
        cancelButtonText: '아니오',
        type: 'warning',
      }
    ).then(() => {
      selectedFeatureIndex.value = index;
      selectedOptionKey.value = null;
      editingOptionKey.value = '';
      editingOptionData.value = null;
      hasOptionChanges.value = false;
    }).catch(() => {
      // 취소, 아무것도 안함
    });
  } else {
    selectedFeatureIndex.value = index;
    selectedOptionKey.value = null;
    editingOptionKey.value = '';
    editingOptionData.value = null;
    hasOptionChanges.value = false;
  }
};

// Option 선택
const selectOption = (key) => {
  // 변경사항이 있으면 경고
  if (hasOptionChanges.value) {
    ElMessageBox.confirm(
      '저장하지 않은 변경사항이 있습니다.\n변경사항을 버리고 다른 Option을 선택하시겠습니까?',
      '변경사항 확인',
      {
        confirmButtonText: '예',
        cancelButtonText: '아니오',
        type: 'warning',
      }
    ).then(() => {
      loadOptionData(key);
    }).catch(() => {
      // 취소, 아무것도 안함
    });
  } else {
    loadOptionData(key);
  }
};

// Option 데이터 로드
const loadOptionData = (key) => {
  selectedOptionKey.value = key;
  editingOptionKey.value = key;
  
  const feature = pageOptions.value[selectedFeatureIndex.value];
  if (feature && feature.option && feature.option[key]) {
    // 깊은 복사로 편집용 데이터 생성
    editingOptionData.value = JSON.parse(JSON.stringify(feature.option[key]));
    hasOptionChanges.value = false;
  }
};

// Option 변경사항 저장
const saveOptionChanges = () => {
  if (!editingOptionData.value || selectedFeatureIndex.value === null || !selectedOptionKey.value) {
    ElMessage.error('저장할 데이터가 없습니다.');
    return;
  }

  try {
    const feature = pageOptions.value[selectedFeatureIndex.value];
    if (feature && feature.option) {
      // 실제 데이터에 반영
      feature.option[selectedOptionKey.value] = JSON.parse(JSON.stringify(editingOptionData.value));
      hasOptionChanges.value = false;
      ElMessage.success('Option이 저장되었습니다.');
    }
  } catch (error) {
    console.error('Option 저장 실패:', error);
    ElMessage.error('Option 저장 중 오류가 발생했습니다.');
  }
};

// Option 변경사항 초기화
const resetOptionChanges = () => {
  if (selectedFeatureIndex.value === null || !selectedOptionKey.value) {
    return;
  }

  const feature = pageOptions.value[selectedFeatureIndex.value];
  if (feature && feature.option && feature.option[selectedOptionKey.value]) {
    // 원본 데이터로 다시 복사
    editingOptionData.value = JSON.parse(JSON.stringify(feature.option[selectedOptionKey.value]));
    editingOptionKey.value = selectedOptionKey.value;
    hasOptionChanges.value = false;
    ElMessage.info('변경사항이 초기화되었습니다.');
  }
};

// Feature 추가
const addFeature = async () => {
  console.log('addFeature 호출됨');
  try {
    const result = await ElMessageBox.prompt('새 Feature ID를 입력하세요', 'Feature 추가', {
      confirmButtonText: '추가',
      cancelButtonText: '취소',
      inputPattern: /^[A-Z0-9]+$/,
      inputErrorMessage: 'Feature ID는 대문자와 숫자만 가능합니다',
      inputPlaceholder: '예: F99000',
    });

    console.log('prompt 결과:', result);

    if (!result.value || result.value.trim() === '') {
      ElMessage.error('Feature ID를 입력해주세요.');
      return;
    }

    // 이미 존재하는지 확인
    const exists = pageOptions.value.some(f => f.featureId === result.value);
    if (exists) {
      ElMessage.error('이미 존재하는 Feature ID입니다.');
      return;
    }

    const newFeature = {
      _id: {
        $oid: generateObjectId(),
      },
      featureId: result.value,
      option: {},
    };
    
    pageOptions.value.push(newFeature);
    selectedFeatureIndex.value = pageOptions.value.length - 1;
    selectedOptionKey.value = null;
    editingOptionKey.value = '';
    ElMessage.success(`Feature "${result.value}"가 추가되었습니다.`);
  } catch (err) {
    // 취소됨 또는 에러
    if (err !== 'cancel' && err !== 'close') {
      console.error('Feature 추가 오류:', err);
      ElMessage.error('Feature 추가 중 오류가 발생했습니다: ' + err.message);
    }
  }
};

// Feature 삭제
const deleteFeature = async (index, event) => {
  console.log('deleteFeature 호출됨, index:', index);
  
  // 이벤트 전파 중지
  if (event) {
    event.stopPropagation();
    event.preventDefault();
  }

  const feature = pageOptions.value[index];
  if (!feature) {
    console.error('Feature를 찾을 수 없음, index:', index);
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

    console.log('삭제 확인됨, 삭제 실행');
    
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
    console.log('Feature 삭제 취소 또는 에러:', err);
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
  // 처음 열 때는 모든 템플릿 접기
  expandedTemplates.value = [];
};

// 템플릿 관리자 모달 닫기
const closeTemplateManagerModal = () => {
  showTemplateManagerModal.value = false;
  templateManagerSearch.value = '';
  expandedTemplates.value = [];
};

// 템플릿 확장/축소 토글
const toggleTemplate = (key) => {
  console.log('=== toggleTemplate 호출 ===');
  console.log('클릭한 템플릿 key:', key);
  console.log('현재 expandedTemplates:', JSON.stringify(expandedTemplates.value));
  
  const index = expandedTemplates.value.indexOf(key);
  console.log('indexOf 결과:', index);
  
  if (index > -1) {
    // 이미 펼쳐져 있으면 접기
    expandedTemplates.value.splice(index, 1);
    console.log('✅ 템플릿 접기 완료:', key);
  } else {
    // 접혀 있으면 펼치기
    expandedTemplates.value.push(key);
    console.log('✅ 템플릿 펼치기 완료:', key);
  }
  
  console.log('업데이트 후 expandedTemplates:', JSON.stringify(expandedTemplates.value));
  console.log('includes 테스트:', expandedTemplates.value.includes(key));
  console.log('=========================');
};

// 템플릿 생성 모달 닫기
const closeCreateTemplateModal = () => {
  showCreateTemplateModal.value = false;
  editingTemplate.value = null;
  newTemplate.value = {
    key: '',
    desc: '',
    list: [],
    value: ''
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

// 템플릿 편집
const editTemplate = (template) => {
  editingTemplate.value = template.key;
  newTemplate.value = {
    key: template.key,
    desc: template.desc,
    list: JSON.parse(JSON.stringify(template.sampleList || template.list || [])),
    value: template.defaultValue || ''
  };
  showCreateTemplateModal.value = true;
};

// 템플릿 수정
const updateTemplate = () => {
  // 1. 설명 검증
  if (!newTemplate.value.desc || !newTemplate.value.desc.trim()) {
    ElMessage.error('템플릿 설명을 입력해주세요.');
    return;
  }

  // 2. 리스트 항목 검증
  if (!newTemplate.value.list || newTemplate.value.list.length === 0) {
    ElMessage.error('리스트 항목을 최소 1개 이상 추가해주세요.');
    return;
  }

  // 3. 리스트 항목 내용 검증
  const hasEmptyListItem = newTemplate.value.list.some(
    item => !item.listValue || !item.listValue.trim() || !item.listDesc || !item.listDesc.trim()
  );
  if (hasEmptyListItem) {
    ElMessage.error('모든 리스트 항목의 값과 설명을 입력해주세요.');
    return;
  }

  // 4. 기본값 검증
  if (!newTemplate.value.value || !newTemplate.value.value.trim()) {
    ElMessage.error('기본값을 선택해주세요.');
    return;
  }

  const templateKey = editingTemplate.value;
  
  // 커스텀 템플릿 찾아서 업데이트
  const customIndex = customTemplates.value.findIndex(t => t.key === templateKey);
  if (customIndex !== -1) {
    const updatedTemplate = {
      key: templateKey,
      desc: newTemplate.value.desc.trim(),
      list: JSON.parse(JSON.stringify(newTemplate.value.list)),
      defaultValue: newTemplate.value.value.trim(),
      listCount: newTemplate.value.list.length,
      sampleList: JSON.parse(JSON.stringify(newTemplate.value.list))
    };
    
    customTemplates.value[customIndex] = updatedTemplate;
    saveCustomTemplates();
    
    // 사용 중인 Feature들의 해당 Option도 업데이트
    updateFeaturesWithTemplate(templateKey, updatedTemplate);
  }
  
  closeCreateTemplateModal();
  ElMessage.success(`템플릿 "${templateKey}"가 수정되었습니다.`);
};

// Feature들의 템플릿 업데이트
const updateFeaturesWithTemplate = (templateKey, updatedTemplate) => {
  pageOptions.value.forEach(feature => {
    if (feature.option && feature.option[templateKey]) {
      // desc와 list를 업데이트, value는 유지
      feature.option[templateKey].desc = updatedTemplate.desc;
      feature.option[templateKey].list = JSON.parse(JSON.stringify(updatedTemplate.list));
    }
  });
};

// 템플릿 삭제
const deleteTemplate = async (template) => {
  try {
    let warningMessage = `템플릿 "${template.key}"를 삭제하시겠습니까?`;
    
    if (template.count > 0) {
      warningMessage += `\n\n⚠️ 이 템플릿은 현재 ${template.count}개의 Feature에서 사용 중입니다.\n삭제 시 해당 Feature들의 Option도 함께 삭제됩니다.`;
    }
    
    await ElMessageBox.confirm(
      warningMessage,
      '템플릿 삭제',
      {
        confirmButtonText: '삭제',
        cancelButtonText: '취소',
        type: 'warning',
        dangerouslyUseHTMLString: true,
      }
    );

    // 커스텀 템플릿에서 삭제
    const customIndex = customTemplates.value.findIndex(t => t.key === template.key);
    if (customIndex !== -1) {
      customTemplates.value.splice(customIndex, 1);
      saveCustomTemplates();
    }

    // 사용 중인 Feature들에서 해당 Option 삭제
    if (template.count > 0) {
      pageOptions.value.forEach(feature => {
        if (feature.option && feature.option[template.key]) {
          delete feature.option[template.key];
        }
      });
    }

    ElMessage.success(`템플릿 "${template.key}"가 삭제되었습니다.`);
  } catch (err) {
    console.log('템플릿 삭제 취소');
  }
};

// 템플릿 생성
const createTemplate = () => {
  // 1. 템플릿 Key 검증
  if (!newTemplate.value.key || !newTemplate.value.key.trim()) {
    ElMessage.error('템플릿 Key를 입력해주세요.');
    return;
  }

  if (!newTemplate.value.key.match(/^[a-zA-Z0-9_]+$/)) {
    ElMessage.error('템플릿 Key는 영문, 숫자, 언더스코어만 가능합니다.');
    return;
  }

  // 2. 설명 검증
  if (!newTemplate.value.desc || !newTemplate.value.desc.trim()) {
    ElMessage.error('템플릿 설명을 입력해주세요.');
    return;
  }

  // 3. 리스트 항목 검증
  if (!newTemplate.value.list || newTemplate.value.list.length === 0) {
    ElMessage.error('리스트 항목을 최소 1개 이상 추가해주세요.');
    return;
  }

  // 4. 리스트 항목 내용 검증
  const hasEmptyListItem = newTemplate.value.list.some(
    item => !item.listValue || !item.listValue.trim() || !item.listDesc || !item.listDesc.trim()
  );
  if (hasEmptyListItem) {
    ElMessage.error('모든 리스트 항목의 값과 설명을 입력해주세요.');
    return;
  }

  // 5. 기본값 검증
  if (!newTemplate.value.value || !newTemplate.value.value.trim()) {
    ElMessage.error('기본값을 선택해주세요.');
    return;
  }

  // 6. 중복 체크
  const exists = customTemplates.value.some(t => t.key === newTemplate.value.key);
  if (exists) {
    ElMessage.error('이미 존재하는 템플릿 Key입니다.');
    return;
  }

  const template = {
    key: newTemplate.value.key.trim(),
    desc: newTemplate.value.desc.trim(),
    list: JSON.parse(JSON.stringify(newTemplate.value.list)),
    defaultValue: newTemplate.value.value.trim(),
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
const saveChanges = async () => {
  console.log('saveChanges 호출됨');
  try {
    await ElMessageBox.confirm(
      '변경사항을 저장하시겠습니까?\n(실제 파일은 저장되지 않으며, 브라우저 세션에만 저장됩니다)',
      '저장 확인',
      {
        confirmButtonText: '저장',
        cancelButtonText: '취소',
        type: 'info',
      }
    );
    
    originalData.value = JSON.parse(JSON.stringify(pageOptions.value));
    ElMessage.success('변경사항이 저장되었습니다.');
    console.log('저장 완료');
  } catch (err) {
    // 취소됨
    console.log('저장 취소');
  }
};

// 초기화
const resetChanges = async () => {
  console.log('resetChanges 호출됨');
  try {
    await ElMessageBox.confirm(
      '모든 변경사항을 취소하고 원본 데이터로 되돌리시겠습니까?',
      '초기화 확인',
      {
        confirmButtonText: '초기화',
        cancelButtonText: '취소',
        type: 'warning',
      }
    );
    
    pageOptions.value = JSON.parse(JSON.stringify(originalData.value));
    selectedFeatureIndex.value = null;
    selectedOptionKey.value = null;
    editingOptionKey.value = '';
    ElMessage.success('원본 데이터로 초기화되었습니다.');
    console.log('초기화 완료');
  } catch (err) {
    // 취소됨
    console.log('초기화 취소');
  }
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
  grid-template-columns: 350px 500px 1fr;
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

/* Feature 및 Option 카드 스타일은 FeatureCard.vue, OptionCard.vue로 이동 */

/* Option 아이템 wrapper */
.option-items {
  padding: 15px;
  overflow-y: auto;
  flex: 1;
}

/* 상세 편집 폼 스타일은 OptionDetail.vue로 이동 */

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
  z-index: 2000;
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
  max-width: 1200px;
  max-height: 85vh;
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
  gap: 15px;
  max-height: 600px;
  overflow-y: auto;
}

.template-manager-item {
  background: white;
  border: 2px solid #e9ecef;
  border-radius: 8px;
  overflow: hidden;
  transition: all 0.3s ease;
}

.template-manager-item.expanded {
  border-color: #667eea;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.15);
}

.template-manager-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 18px 20px;
  background: #f8f9fa;
  cursor: pointer;
  transition: all 0.3s ease;
  border-bottom: 2px solid transparent;
}

.template-manager-header:hover {
  background: #e9ecef;
}

.template-manager-item.expanded .template-manager-header {
  background: linear-gradient(135deg, #f0f4ff 0%, #fdf0ff 100%);
  border-bottom-color: #667eea;
}

.template-header-left {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
}

.expand-icon {
  font-size: 14px;
  color: #667eea;
  font-weight: bold;
  min-width: 20px;
}

.template-actions {
  display: flex;
  gap: 8px;
}

.template-key {
  font-size: 18px;
  color: #667eea;
  font-weight: 700;
}

.template-badge {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.template-badge-custom {
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  color: white;
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.btn-edit-small {
  background: #3b82f6;
  color: white;
  padding: 6px 12px;
  font-size: 12px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
  transition: all 0.3s ease;
}

.btn-edit-small:hover {
  background: #2563eb;
  transform: translateY(-1px);
}

.btn-delete-small {
  background: #ef4444;
  color: white;
  padding: 6px 12px;
  font-size: 12px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
  transition: all 0.3s ease;
}

.btn-delete-small:hover:not(:disabled) {
  background: #dc2626;
  transform: translateY(-1px);
}

.btn-delete-small:disabled {
  background: #d1d5db;
  cursor: not-allowed;
  opacity: 0.6;
}

.input-hint {
  font-size: 12px;
  color: #666;
  margin-top: 5px;
  font-style: italic;
}

.template-manager-body {
  display: grid;
  gap: 15px;
  padding: 20px;
  background: white;
  animation: slideDown 0.3s ease;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Transition 애니메이션 */
.slide-down-enter-active {
  transition: all 0.3s ease;
}

.slide-down-leave-active {
  transition: all 0.2s ease;
}

.slide-down-enter-from {
  opacity: 0;
  transform: translateY(-10px);
}

.slide-down-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}

.template-field {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 6px;
  border: 1px solid #e9ecef;
}

.template-field label {
  display: block;
  font-weight: 700;
  color: #495057;
  margin-bottom: 8px;
  font-size: 13px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.template-field p {
  margin: 0;
  color: #666;
  font-size: 14px;
  line-height: 1.6;
}

.template-field code {
  background: #e3f2fd;
  color: #1565c0;
  padding: 6px 12px;
  border-radius: 6px;
  font-size: 14px;
  font-family: 'Monaco', 'Menlo', monospace;
  font-weight: 600;
}

.template-list-preview {
  list-style: none;
  padding: 0;
  margin: 8px 0 0;
}

.template-list-preview li {
  padding: 10px;
  background: white;
  border-radius: 6px;
  margin-bottom: 8px;
  font-size: 13px;
  border-left: 4px solid #667eea;
  transition: all 0.2s ease;
}

.template-list-preview li:hover {
  transform: translateX(5px);
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.1);
}

.template-usage {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 6px;
  border: 1px solid #e9ecef;
}

.template-usage label {
  display: block;
  font-weight: 700;
  color: #495057;
  margin-bottom: 8px;
  font-size: 13px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
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
  padding: 6px 14px;
  border-radius: 16px;
  font-size: 12px;
  font-weight: 600;
  transition: all 0.2s ease;
}

.usage-tag:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(245, 87, 108, 0.3);
}

.no-usage {
  color: #999;
  font-size: 13px;
  font-style: italic;
  padding: 6px 14px;
  background: #f1f3f5;
  border-radius: 16px;
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

/* Option 편집 액션 버튼 */
.option-edit-actions {
  display: flex;
  gap: 12px;
  margin-top: 24px;
  padding-top: 16px;
  border-top: 2px solid #e4e7ed;
}

.btn-option-save,
.btn-option-reset {
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

.btn-option-save:disabled,
.btn-option-reset:disabled {
  background: #f5f7fa;
  color: #c0c4cc;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

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
