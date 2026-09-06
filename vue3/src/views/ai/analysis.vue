<template>
  <div class="analysis-container">
    <!-- 左侧面板 -->
    <div class="analysis-sidebar">
      <div class="sidebar-section">
        <div class="section-title">分析模板</div>
        <div class="template-list">
          <div
            v-for="tpl in templates"
            :key="tpl.type"
            class="template-item"
            :class="{ active: currentType === tpl.type && !isAnalyzing }"
            @click="selectTemplate(tpl.type)"
          >
            <div class="template-icon" :style="{ background: tpl.color }">
              <span>{{ tpl.icon }}</span>
            </div>
            <div class="template-info">
              <div class="template-name">{{ tpl.label }}</div>
              <div class="template-desc">{{ tpl.desc }}</div>
            </div>
          </div>
        </div>
      </div>

      <div class="sidebar-section history-section">
        <div class="section-title">
          历史记录
          <el-button text size="small" @click="loadHistory" :loading="historyLoading">
            <el-icon><Refresh /></el-icon>
          </el-button>
        </div>
        <div class="history-list" v-loading="historyLoading">
          <div
            v-for="record in historyList"
            :key="record.id"
            class="history-item"
            :class="{ active: currentRecordId === record.id }"
            @click="viewHistory(record)"
          >
            <div class="history-type">
              <el-tag :type="getTagType(record.analysisType)" size="small" effect="plain">
                {{ getTypeLabel(record.analysisType) }}
              </el-tag>
            </div>
            <div class="history-content">{{ record.analysisContent || '自动分析' }}</div>
            <div class="history-time">{{ formatTime(record.createdTime) }}</div>
          </div>
          <el-empty v-if="!historyLoading && historyList.length === 0" description="暂无记录" :image-size="60" />
        </div>
      </div>
    </div>

    <!-- 右侧主内容区 -->
    <div class="analysis-main">
      <!-- 顶部标题栏 -->
      <div class="analysis-header">
        <div class="header-left">
          <el-icon :size="20" color="#409eff"><DataAnalysis /></el-icon>
          <span class="header-title">{{ currentTemplate?.label || '智能分析' }}</span>
          <el-tag v-if="currentRecordId" type="success" size="small" effect="plain">历史查看</el-tag>
        </div>
        <div class="header-right" v-if="isAnalyzing">
          <el-tag type="warning" size="small" effect="dark">
            <el-icon class="is-loading"><Loading /></el-icon>
            分析中...
          </el-tag>
        </div>
      </div>

      <!-- 分析内容区 -->
      <div class="analysis-content" ref="contentRef">
        <!-- 欢迎页 / 模板选择 -->
        <div v-if="!currentResult && !isAnalyzing" class="welcome-panel">
          <div class="welcome-icon">
            <el-icon :size="64" color="#409eff"><MagicStick /></el-icon>
          </div>
          <h2>AI 智能分析</h2>
          <p>选择左侧分析模板，或输入自定义分析需求</p>
          <div class="quick-actions">
            <el-button
              v-for="tpl in templates"
              :key="tpl.type"
              :type="currentType === tpl.type ? 'primary' : 'default'"
              @click="selectTemplate(tpl.type)"
            >
              {{ tpl.icon }} {{ tpl.label }}
            </el-button>
          </div>
        </div>

        <!-- 分析进行中 / 结果展示 -->
        <div v-if="currentResult || isAnalyzing" class="result-panel">
          <div class="result-markdown" v-html="renderMarkdown(currentResult)"></div>
          <div v-if="isAnalyzing && !currentResult" class="analyzing-hint">
            <span class="thinking-text">正在调用AI分析引擎</span>
            <span class="loading-dots">
              <span>.</span><span>.</span><span>.</span>
            </span>
          </div>
        </div>

        <!-- 分析失败 -->
        <div v-if="errorMessage" class="error-panel">
          <el-alert :title="errorMessage" type="error" show-icon :closable="false" />
        </div>
      </div>

      <!-- 底部输入区 -->
      <div class="analysis-input" v-if="currentType">
        <div class="input-hint" v-if="currentTemplate">
          {{ currentTemplate.inputHint }}
        </div>
        <div class="input-row">
          <el-input
            v-model="userInput"
            type="textarea"
            :rows="2"
            :placeholder="currentTemplate?.inputPlaceholder || '输入分析需求...'"
            :disabled="isAnalyzing"
            @keydown.enter.ctrl="startAnalysis"
          />
          <el-button
            type="primary"
            :loading="isAnalyzing"
            :disabled="!currentType"
            @click="startAnalysis"
            class="send-btn"
          >
            <el-icon v-if="!isAnalyzing"><VideoPlay /></el-icon>
            {{ isAnalyzing ? '分析中' : '开始分析' }}
          </el-button>
        </div>
        <div class="input-tip">Ctrl + Enter 发送</div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, nextTick, computed } from 'vue'
import { Refresh, Loading, VideoPlay, DataAnalysis, MagicStick } from '@element-plus/icons-vue'
import { marked } from 'marked'
import { getToken } from '@/utils/auth'
import { getAnalysisList } from '@/api/ai/analysis'
import type { AiAnalysisRecord } from '@/api/ai/analysis'

interface Template {
  type: string
  label: string
  icon: string
  desc: string
  color: string
  inputHint: string
  inputPlaceholder: string
}

const templates: Template[] = [
  {
    type: 'sales',
    label: '销售分析',
    icon: '📊',
    desc: '销售趋势、热销排名、平台对比',
    color: '#409eff',
    inputHint: '可补充：关注的时间范围、特定平台、特定商品等',
    inputPlaceholder: '如：只分析淘宝平台最近3天的销售情况'
  },
  {
    type: 'inventory',
    label: '库存优化',
    icon: '📦',
    desc: '库存预警、周转分析、补货建议',
    color: '#67c23a',
    inputHint: '可补充：预警阈值、关注的仓库、特定品类等',
    inputPlaceholder: '如：分析服装类目的库存周转情况'
  },
  {
    type: 'customer',
    label: '客户洞察',
    icon: '👥',
    desc: '客户画像、复购分析、售后统计',
    color: '#e6a23c',
    inputHint: '可补充：关注的客户群体、特定时间段等',
    inputPlaceholder: '如：分析最近30天的客户复购情况'
  },
  {
    type: 'operation',
    label: '运营效率',
    icon: '⚡',
    desc: '订单处理、物流时效、人效分析',
    color: '#f56c6c',
    inputHint: '可补充：关注的环节、特定店铺等',
    inputPlaceholder: '如：分析各店铺的发货及时率'
  },
  {
    type: 'purchase',
    label: '采购分析',
    icon: '🏭',
    desc: '采购周期、供应商评估、库存匹配',
    color: '#909399',
    inputHint: '可补充：关注的供应商、采购品类等',
    inputPlaceholder: '如：分析最近的采购订单执行情况'
  }
]

const currentType = ref('')
const userInput = ref('')
const isAnalyzing = ref(false)
const currentResult = ref('')
const errorMessage = ref('')
const currentRecordId = ref<number | null>(null)
const historyList = ref<AiAnalysisRecord[]>([])
const historyLoading = ref(false)
const contentRef = ref<HTMLElement>()
const abortRef = ref<AbortController | null>(null)

const currentTemplate = computed(() => templates.find(t => t.type === currentType.value))

function selectTemplate(type: string) {
  if (isAnalyzing.value) return
  currentType.value = type
  currentResult.value = ''
  errorMessage.value = ''
  currentRecordId.value = null
}

async function startAnalysis() {
  if (!currentType.value || isAnalyzing.value) return

  isAnalyzing.value = true
  currentResult.value = ''
  errorMessage.value = ''
  currentRecordId.value = null

  try {
    const token = getToken()
    abortRef.value = new AbortController()

    const response = await fetch('/api/ai/analysis/execute', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`
      },
      body: JSON.stringify({
        analysisType: currentType.value,
        content: userInput.value || undefined
      }),
      signal: abortRef.value.signal
    })

    if (!response.ok) {
      const errText = await response.text().catch(() => '')
      errorMessage.value = `请求失败 (${response.status}): ${errText}`
      isAnalyzing.value = false
      return
    }

    const reader = response.body!.getReader()
    const decoder = new TextDecoder()
    let buffer = ''
    let fullContent = ''

    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      buffer += decoder.decode(value, { stream: true })
      const lines = buffer.split('\n')
      buffer = lines.pop() || ''

      for (const line of lines) {
        const t = line.trim()
        if (!t || !t.startsWith('data:')) continue

        const jsonStr = t.slice(5).trim()
        if (!jsonStr) continue

        try {
          const event = JSON.parse(jsonStr)
          const type = event.type
          const content = event.content || ''

          if (type === 'message') {
            fullContent += content
            currentResult.value = fullContent
            scrollToBottom()
          } else if (type === 'analysis_start') {
            if (content) currentRecordId.value = Number(content)
          } else if (type === 'done') {
            if (content) currentRecordId.value = Number(content)
          } else if (type === 'error') {
            errorMessage.value = content
          }
        } catch {
          // ignore non-JSON
        }
      }
    }

    if (fullContent) {
      currentResult.value = fullContent
    }

    // 刷新历史记录
    loadHistory()
  } catch (e: any) {
    if (e.name === 'AbortError') return
    errorMessage.value = '网络错误: ' + (e.message || '请检查网络连接')
  } finally {
    isAnalyzing.value = false
    abortRef.value = null
  }
}

async function loadHistory() {
  historyLoading.value = true
  try {
    const res = await getAnalysisList({ pageNum: 1, pageSize: 50 })
    if (res.data?.records) {
      historyList.value = res.data.records
    } else if (Array.isArray(res.data)) {
      historyList.value = res.data
    }
  } catch {
    // 静默失败
  } finally {
    historyLoading.value = false
  }
}

function viewHistory(record: AiAnalysisRecord) {
  if (isAnalyzing.value) return
  currentRecordId.value = record.id!
  currentType.value = record.analysisType
  currentResult.value = record.analysisResult || ''
  errorMessage.value = record.status === 2 ? (record.errorMessage || '分析失败') : ''
  userInput.value = record.analysisContent || ''
}

function getTagType(type: string): string {
  const map: Record<string, string> = {
    sales: '',
    inventory: 'success',
    customer: 'warning',
    operation: 'danger',
    purchase: 'info',
    custom: ''
  }
  return map[type] || ''
}

function getTypeLabel(type: string): string {
  const tpl = templates.find(t => t.type === type)
  return tpl?.label || type
}

function formatTime(time?: string): string {
  if (!time) return ''
  const d = new Date(time)
  const now = new Date()
  const diff = now.getTime() - d.getTime()
  if (diff < 60000) return '刚刚'
  if (diff < 3600000) return Math.floor(diff / 60000) + '分钟前'
  if (diff < 86400000) return Math.floor(diff / 3600000) + '小时前'
  if (diff < 604800000) return Math.floor(diff / 86400000) + '天前'
  return time.slice(0, 10)
}

function renderMarkdown(text: string): string {
  if (!text) return ''
  return marked.parse(text) as string
}

function scrollToBottom() {
  nextTick(() => {
    const el = contentRef.value
    if (el) el.scrollTop = el.scrollHeight
  })
}

onMounted(() => {
  loadHistory()
})
</script>

<style lang="scss" scoped>
.analysis-container {
  display: flex;
  height: calc(100vh - 84px);
  overflow: hidden;
}

// 左侧面板
.analysis-sidebar {
  width: 280px;
  min-width: 280px;
  border-right: 1px solid #ebeef5;
  display: flex;
  flex-direction: column;
  background: #fff;
}

.sidebar-section {
  padding: 16px;
}

.section-title {
  font-size: 14px;
  font-weight: 600;
  color: #303133;
  margin-bottom: 12px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.template-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.template-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
  border: 1px solid transparent;

  &:hover {
    background: #f5f7fa;
  }

  &.active {
    background: #ecf5ff;
    border-color: #409eff;
  }
}

.template-icon {
  width: 36px;
  height: 36px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
  color: #fff;
  flex-shrink: 0;
}

.template-info {
  flex: 1;
  min-width: 0;
}

.template-name {
  font-size: 14px;
  font-weight: 500;
  color: #303133;
}

.template-desc {
  font-size: 12px;
  color: #909399;
  margin-top: 2px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

// 历史记录
.history-section {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  border-top: 1px solid #ebeef5;
}

.history-list {
  flex: 1;
  overflow-y: auto;
}

.history-item {
  padding: 10px 12px;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
  margin-bottom: 4px;

  &:hover {
    background: #f5f7fa;
  }

  &.active {
    background: #ecf5ff;
  }
}

.history-type {
  margin-bottom: 4px;
}

.history-content {
  font-size: 13px;
  color: #606266;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.history-time {
  font-size: 12px;
  color: #c0c4cc;
  margin-top: 4px;
}

// 右侧主内容区
.analysis-main {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: #f9fafb;
}

.analysis-header {
  padding: 14px 24px;
  background: #fff;
  border-bottom: 1px solid #ebeef5;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 8px;
}

.header-icon {
  font-size: 20px;
  color: #409eff;
}

.header-title {
  font-size: 16px;
  font-weight: 600;
  color: #303133;
}

// 分析内容区
.analysis-content {
  flex: 1;
  overflow-y: auto;
  padding: 24px;
}

.welcome-panel {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  text-align: center;
}

.welcome-icon {
  font-size: 64px;
  color: #409eff;
  margin-bottom: 16px;
}

.welcome-panel h2 {
  font-size: 24px;
  color: #303133;
  margin: 0 0 8px;
}

.welcome-panel p {
  font-size: 14px;
  color: #909399;
  margin: 0 0 24px;
}

.quick-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  justify-content: center;
}

.result-panel {
  max-width: 900px;
  margin: 0 auto;
}

.result-markdown {
  background: #fff;
  border-radius: 12px;
  padding: 24px 32px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
  line-height: 1.8;
  font-size: 14px;
  color: #303133;

  :deep(h1) {
    font-size: 22px;
    margin: 24px 0 12px;
    padding-bottom: 8px;
    border-bottom: 1px solid #ebeef5;
  }

  :deep(h2) {
    font-size: 18px;
    margin: 20px 0 10px;
    color: #303133;
  }

  :deep(h3) {
    font-size: 16px;
    margin: 16px 0 8px;
    color: #303133;
  }

  :deep(p) {
    margin: 8px 0;
  }

  :deep(ul), :deep(ol) {
    padding-left: 20px;
    margin: 8px 0;
  }

  :deep(li) {
    margin: 4px 0;
  }

  :deep(table) {
    width: 100%;
    border-collapse: collapse;
    margin: 12px 0;
    font-size: 13px;
  }

  :deep(th), :deep(td) {
    border: 1px solid #ebeef5;
    padding: 8px 12px;
    text-align: left;
  }

  :deep(th) {
    background: #f5f7fa;
    font-weight: 600;
  }

  :deep(tr:hover) {
    background: #f5f7fa;
  }

  :deep(strong) {
    font-weight: 600;
    color: #303133;
  }

  :deep(code) {
    background: rgba(0, 0, 0, 0.06);
    padding: 1px 5px;
    border-radius: 3px;
    font-size: 13px;
    font-family: monospace;
  }

  :deep(pre) {
    background: #f5f7fa;
    padding: 12px 16px;
    border-radius: 6px;
    overflow-x: auto;
    margin: 12px 0;

    code {
      background: none;
      padding: 0;
    }
  }

  :deep(blockquote) {
    border-left: 4px solid #409eff;
    padding: 8px 16px;
    margin: 12px 0;
    background: #ecf5ff;
    color: #606266;
  }
}

.analyzing-hint {
  text-align: center;
  padding: 32px 0;
  color: #909399;
}

.thinking-text {
  font-size: 14px;
}

.loading-dots span {
  animation: dot-blink 1.4s infinite;
  font-weight: bold;
  font-size: 18px;
  line-height: 1;
  color: #409eff;
}

.loading-dots span:nth-child(2) {
  animation-delay: 0.2s;
}

.loading-dots span:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes dot-blink {
  0%, 20% { opacity: 0; }
  50% { opacity: 1; }
  100% { opacity: 0; }
}

.error-panel {
  max-width: 900px;
  margin: 0 auto;
}

// 底部输入区
.analysis-input {
  padding: 16px 24px;
  background: #fff;
  border-top: 1px solid #ebeef5;
}

.input-hint {
  font-size: 12px;
  color: #909399;
  margin-bottom: 8px;
}

.input-row {
  display: flex;
  align-items: flex-end;
  gap: 12px;
}

.input-row :deep(.el-textarea) {
  flex: 1;
}

.send-btn {
  height: 54px;
  min-width: 100px;
}

.input-tip {
  font-size: 12px;
  color: #c0c4cc;
  margin-top: 4px;
  text-align: right;
}

</style>
