<template>
    <div class="app-container">
        <div class="navigation-tabs">
            <button 
                class="nav-tab" 
                :class="{ active: currentView === 'jsoneditor' }"
                @click="currentView = 'jsoneditor'"
            >
                📝 JSONEditor
            </button>
            <button 
                class="nav-tab" 
                :class="{ active: currentView === 'pageoption' }"
                @click="currentView = 'pageoption'"
            >
                📄 Page Option Editor
            </button>
        </div>

        <PageOptionEditor v-if="currentView === 'pageoption'" />
        
        <div v-else class="container">
            <div class="header">
                <h1>📝 JSONEditor</h1>
                <p>josdejong/jsoneditor 사용 예시</p>
            </div>

            <div class="content">
                <!-- JSONEditor 패널 -->
                <div class="panel" style="grid-column: 1 / -1">
                    <div class="panel-title">📝 JSONEditor (편집 가능)</div>
                    <div class="panel-content">
                        <div class="info-box">✨ josdejong/jsoneditor - 완벽한 JSON 에디터</div>
                        <div ref="editorContainer" class="jsoneditor-container"></div>
                    </div>
                </div>
            </div>

            <div class="controls">
                <div class="select-group">
                    <label for="file-select">📂 테스트 데이터 선택:</label>
                    <select id="file-select" v-model="selectedFile" @change="handleFileChange" class="select-input">
                        <option value="">-- 선택하세요 --</option>
                        <option value="aiCamObjectType">AI 카메라 객체 타입</option>
                        <option value="aiCamObjectTypeOption">AI 카메라 객체 타입 옵션</option>
                        <option value="pageOption">페이지 옵션</option>
                    </select>
                </div>
                <button class="btn btn-danger" @click="clearData">🗑️ 초기화</button>
                <button 
                    class="btn btn-save" 
                    @click="saveData"
                    :disabled="!hasChanges"
                >
                    💾 저장
                </button>
                <button 
                    class="btn btn-cancel" 
                    @click="cancelEdit"
                    :disabled="!hasChanges"
                >
                    ❌ 취소
                </button>
            </div>

            <div class="status">
                <span>현재 JSON 크기: {{ currentSize }} bytes</span>
                <span v-if="hasChanges" class="status-warning">⚠️ 수정된 내용이 있습니다</span>
                <span>최근 업데이트: {{ lastUpdate }}</span>
            </div>

            <div class="footer">
                <p>
                    💡 팁: 드롭다운에서 데이터를 선택하면 에디터에 로드됩니다. Tree, Form, Code 모드로 자유롭게 편집하세요!
                </p>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue';
import JSONEditor from 'jsoneditor';
import 'jsoneditor/dist/jsoneditor.css';
import { ElMessageBox, ElMessage } from 'element-plus';
import PageOptionEditor from './PageOptionEditor.vue';

const currentView = ref('pageoption');

const editorContainer = ref(null);
let editor = null;

const lastUpdate = ref('방금 전');
const selectedFile = ref('');
const previousFile = ref('');
const originalData = ref(null);
const hasChanges = ref(false);

const currentSize = computed(() => {
    if (editor) {
        try {
            return JSON.stringify(editor.get()).length;
        } catch {
            return 0;
        }
    }
    return 0;
});

// 샘플 데이터
const sampleData = {
    name: 'Vue JSON Demo',
    version: '1.0.0',
    description: 'JSONEditor와 vue-json-pretty 사용 예시',
    author: 'Developer',
    features: [
        { name: 'JSONEditor', editable: true, description: '완벽한 JSON 에디터' },
        { name: 'vue-json-pretty', editable: false, description: '예쁜 JSON 뷰어' },
    ],
    settings: {
        theme: 'light',
        language: 'ko',
        autoSave: true,
    },
    data: {
        users: [
            { id: 1, name: 'Alice', email: 'alice@example.com' },
            { id: 2, name: 'Bob', email: 'bob@example.com' },
        ],
    },
    createdAt: new Date().toISOString(),
};

// 테스트 데이터 맵핑 (fetch API 사용)
const testDataMap = {
    aiCamObjectType: () => fetch('/data/dkant.aiCamObjectType.json').then((r) => r.json()),
    aiCamObjectTypeOption: () => fetch('/data/dkant.aiCamObjectTypeOption.json').then((r) => r.json()),
    pageOption: () => fetch('/data/dkant.pageOption.json').then((r) => r.json()),
};

// JSONEditor 초기화
onMounted(() => {
    const container = editorContainer.value;
    const options = {
        mode: 'tree',
        modes: ['tree', 'form', 'code'],
        onError: (err) => console.error('JSONEditor 에러:', err),
        onTextSelectionChange: () => checkChanges(),
        onChange: () => checkChanges(),
    };

    editor = new JSONEditor(container, options, sampleData);
    originalData.value = JSON.parse(JSON.stringify(sampleData));
});

// 변경 사항 확인
const checkChanges = () => {
    if (editor && originalData.value) {
        try {
            const currentData = editor.get();
            const currentStr = JSON.stringify(currentData);
            const originalStr = JSON.stringify(originalData.value);
            hasChanges.value = currentStr !== originalStr;
        } catch (err) {
            hasChanges.value = false;
        }
    }
};

// 원본과 수정본의 차이점 비교
const getChangeSummary = (original, current, prefix = '') => {
    const changes = {
        modified: [],
        added: [],
        removed: [],
    };

    // 추가된 키 찾기
    for (const key in current) {
        if (!(key in original)) {
            changes.added.push(`  • ${prefix}${key}`);
        }
    }

    // 수정되거나 제거된 키 찾기
    for (const key in original) {
        const currentPath = prefix ? `${prefix}${key}` : key;
        if (!(key in current)) {
            changes.removed.push(`  • ${currentPath}`);
        } else if (JSON.stringify(original[key]) !== JSON.stringify(current[key])) {
            if (typeof original[key] === 'object' && original[key] !== null) {
                // 중첩된 객체는 간단히 표시
                changes.modified.push(`  • ${currentPath}`);
            } else {
                changes.modified.push(`  • ${currentPath}: "${original[key]}" → "${current[key]}"`);
            }
        }
    }

    return changes;
};

// 저장 버튼 클릭
const saveData = () => {
    try {
        if (!editor || !originalData.value) return;

        const currentData = editor.get();
        const changes = getChangeSummary(originalData.value, currentData);

        let messageContent = '변경된 내용:\n\n';

        if (changes.modified.length > 0) {
            messageContent += '수정된 필드:\n' + changes.modified.join('\n') + '\n\n';
        }

        if (changes.added.length > 0) {
            messageContent += '추가된 필드:\n' + changes.added.join('\n') + '\n\n';
        }

        if (changes.removed.length > 0) {
            messageContent += '제거된 필드:\n' + changes.removed.join('\n') + '\n\n';
        }

        messageContent += '\n정말 저장하시겠습니까?';

        ElMessageBox.confirm(
            messageContent,
            '변경 사항 저장',
            {
                confirmButtonText: '저장',
                cancelButtonText: '취소',
                type: 'warning',
                dangerouslyUseHTMLString: true,
                customClass: 'change-summary-dialog',
            }
        ).then(() => {
            // 저장 완료
            originalData.value = JSON.parse(JSON.stringify(currentData));
            hasChanges.value = false;
            updateTimestamp();
            ElMessage.success('변경 사항이 저장되었습니다.');
        }).catch(() => {
            // 취소됨
        });
    } catch (err) {
        console.error('저장 오류:', err);
        ElMessage.error('저장 중 오류가 발생했습니다.');
    }
};

// 취소 버튼 클릭
const cancelEdit = () => {
    if (!hasChanges.value) return;

    ElMessageBox.confirm(
        '수정된 내용이 있는데 정말 취소하시겠습니까?\n모든 변경 사항이 사라집니다.',
        '편집 취소',
        {
            confirmButtonText: '취소',
            cancelButtonText: '계속 편집',
            type: 'warning',
        }
    ).then(() => {
        // 취소 확인
        if (editor && originalData.value) {
            editor.set(JSON.parse(JSON.stringify(originalData.value)));
            hasChanges.value = false;
            updateTimestamp();
            ElMessage.info('변경 사항이 취소되었습니다.');
        }
    }).catch(() => {
        // 계속 편집
    });
};

// 초기화
const clearData = () => {
    if (!hasChanges.value) {
        ElMessage.info('변경된 내용이 없습니다.');
        return;
    }

    ElMessageBox.confirm(
        '저장된 시점의 원본 데이터로 초기화하시겠습니까?',
        '데이터 초기화',
        {
            confirmButtonText: '초기화',
            cancelButtonText: '취소',
            type: 'warning',
        }
    ).then(() => {
        if (editor && originalData.value) {
            // 저장된 원본 데이터로 복원
            editor.set(JSON.parse(JSON.stringify(originalData.value)));
            hasChanges.value = false;
            updateTimestamp();
            ElMessage.success('저장된 상태로 초기화되었습니다.');
        }
    }).catch(() => {
        // 취소됨
    });
};

// 파일 선택 변경 처리
const handleFileChange = () => {
    if (hasChanges.value) {
        ElMessageBox.confirm(
            '수정 중이던 데이터가 있습니다.\n\n지금 여기 나가면 초기화됩니다.\n정말 저장하지 않고 나가시겠습니까?',
            '데이터 변경 확인',
            {
                confirmButtonText: '나가기',
                cancelButtonText: '계속 편집',
                type: 'warning',
            }
        ).then(() => {
            // 확인 -> 새 파일 로드
            previousFile.value = selectedFile.value;
            loadSelectedFile();
        }).catch(() => {
            // 취소 -> select box 값 이전으로 되돌림
            selectedFile.value = previousFile.value;
        });
    } else {
        // 변경사항 없으면 바로 로드
        previousFile.value = selectedFile.value;
        loadSelectedFile();
    }
};

// 테스트 데이터 로드
const loadSelectedFile = async () => {
    try {
        if (!selectedFile.value) return;

        const loader = testDataMap[selectedFile.value];
        if (loader) {
            const data = await loader();
            if (editor) {
                originalData.value = JSON.parse(JSON.stringify(data));
                editor.set(data);
                hasChanges.value = false;
                updateTimestamp();
            }
        }
    } catch (err) {
        console.error('테스트 데이터 로드 에러:', err);
        ElMessage.error('데이터를 로드할 수 없습니다.');
    }
};

// 타임스탐프 업데이트
const updateTimestamp = () => {
    const now = new Date();
    lastUpdate.value = now.toLocaleTimeString('ko-KR');
};
</script>

<style scoped></style>
