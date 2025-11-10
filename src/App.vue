<template>
  <div class="container">
    <div class="header">
      <h1>📝 JSONEditor</h1>
      <p>josdejong/jsoneditor 사용 예시</p>
    </div>

    <div class="content">
      <!-- JSONEditor 패널 -->
      <div class="panel" style="grid-column: 1 / -1;">
        <div class="panel-title">📝 JSONEditor (편집 가능)</div>
        <div class="panel-content">
          <div class="info-box">
            ✨ josdejong/jsoneditor - 완벽한 JSON 에디터
          </div>
          <div ref="editorContainer" class="jsoneditor-container"></div>
        </div>
      </div>
    </div>

    <div class="controls">
      <div class="select-group">
        <label for="file-select">📂 테스트 데이터 선택:</label>
        <select id="file-select" v-model="selectedFile" @change="loadSelectedFile" class="select-input">
          <option value="">-- 선택하세요 --</option>
          <option value="aiCamObjectType">AI 카메라 객체 타입</option>
          <option value="aiCamObjectTypeOption">AI 카메라 객체 타입 옵션</option>
          <option value="pageOption">페이지 옵션</option>
        </select>
      </div>
      <button class="btn btn-danger" @click="clearData">🗑️ 초기화</button>
    </div>

    <div class="status">
      현재 JSON 크기: {{ currentSize }} bytes | 최근 업데이트: {{ lastUpdate }}
    </div>

    <div class="footer">
      <p>💡 팁: 드롭다운에서 데이터를 선택하면 에디터에 로드됩니다. Tree, Form, Code 모드로 자유롭게 편집하세요!</p>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import JSONEditor from 'jsoneditor'
import 'jsoneditor/dist/jsoneditor.css'

const editorContainer = ref(null)
let editor = null

const lastUpdate = ref('방금 전')
const selectedFile = ref('')

const currentSize = computed(() => {
  if (editor) {
    try {
      return JSON.stringify(editor.get()).length
    } catch {
      return 0
    }
  }
  return 0
})

// 샘플 데이터
const sampleData = {
  name: 'Vue JSON Demo',
  version: '1.0.0',
  description: 'JSONEditor와 vue-json-pretty 사용 예시',
  author: 'Developer',
  features: [
    { name: 'JSONEditor', editable: true, description: '완벽한 JSON 에디터' },
    { name: 'vue-json-pretty', editable: false, description: '예쁜 JSON 뷰어' }
  ],
  settings: {
    theme: 'light',
    language: 'ko',
    autoSave: true
  },
  data: {
    users: [
      { id: 1, name: 'Alice', email: 'alice@example.com' },
      { id: 2, name: 'Bob', email: 'bob@example.com' }
    ]
  },
  createdAt: new Date().toISOString()
}

// 테스트 데이터 맵핑 (fetch API 사용)
const testDataMap = {
  aiCamObjectType: () => fetch('/data/dkant.aiCamObjectType.json').then(r => r.json()),
  aiCamObjectTypeOption: () => fetch('/data/dkant.aiCamObjectTypeOption.json').then(r => r.json()),
  pageOption: () => fetch('/data/dkant.pageOption.json').then(r => r.json())
}

// JSONEditor 초기화
onMounted(() => {
  const container = editorContainer.value
  const options = {
    mode: 'tree',
    modes: ['tree', 'form', 'code'],
    onError: (err) => console.error('JSONEditor 에러:', err),
    onTextSelectionChange: () => updateTimestamp()
  }

  editor = new JSONEditor(container, options, sampleData)
})


// 초기화
const clearData = () => {
  const defaultData = { message: '모든 데이터가 초기화되었습니다.' }
  if (editor) {
    editor.set(defaultData)
    updateTimestamp()
  }
}

// 테스트 데이터 로드
const loadSelectedFile = async () => {
  try {
    if (!selectedFile.value) return

    const loader = testDataMap[selectedFile.value]
    if (loader) {
      const data = await loader()
      if (editor) {
        editor.set(data)
        updateTimestamp()
      }
    }
  } catch (err) {
    console.error('테스트 데이터 로드 에러:', err)
    alert('데이터를 로드할 수 없습니다.')
  }
}

// 타임스탐프 업데이트
const updateTimestamp = () => {
  const now = new Date()
  lastUpdate.value = now.toLocaleTimeString('ko-KR')
}
</script>

<style scoped>
</style>

