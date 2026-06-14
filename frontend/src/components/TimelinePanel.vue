<template>
  <div class="h-full flex flex-col bg-gray-900 border-l border-gray-800 transition-all duration-300" :class="collapsed ? 'w-12' : 'w-72'">
    <div class="flex items-center justify-between p-3 border-b border-gray-800">
      <div v-if="!collapsed" class="flex items-center gap-2">
        <svg class="w-5 h-5 text-amber-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" />
        </svg>
        <h3 class="text-amber-300 font-bold text-sm">操作时间轴</h3>
      </div>
      <button @click="collapsed = !collapsed" class="p-1 hover:bg-gray-800 rounded text-gray-400 hover:text-white">
        <svg v-if="collapsed" class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
        </svg>
        <svg v-else class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
        </svg>
      </button>
    </div>

    <div v-if="!collapsed" class="flex-1 overflow-hidden flex flex-col">
      <div class="p-2 border-b border-gray-800">
        <div class="flex flex-wrap gap-1">
          <button v-for="filter in filters" :key="filter.type" @click="toggleFilter(filter.type)"
            class="px-2 py-1 rounded text-xs transition-colors"
            :class="activeFilters.includes(filter.type) ? filter.bgClass + ' ' + filter.textClass : 'bg-gray-800 text-gray-500 hover:bg-gray-700'">
            {{ filter.label }}
          </button>
        </div>
      </div>

      <div class="flex-1 overflow-y-auto p-3">
        <div v-if="filteredEvents.length === 0" class="text-gray-500 text-xs text-center py-8">
          暂无操作记录
        </div>
        <div v-else class="relative">
          <div class="absolute left-3 top-2 bottom-2 w-0.5 bg-gray-700"></div>
          <div v-for="(event, index) in filteredEvents" :key="event.id" class="relative pl-8 pb-4 last:pb-0">
            <div class="absolute left-0 w-6 h-6 rounded-full flex items-center justify-center" :class="getEventIconBg(event.type)">
              <component :is="getEventIcon(event.type)" class="w-3.5 h-3.5" :class="getEventIconColor(event.type)" />
            </div>
            <div class="bg-gray-800 rounded-lg p-2 hover:bg-gray-700 transition-colors cursor-pointer group" @click="toggleExpand(event.id)">
              <div class="flex items-start justify-between gap-2">
                <div class="flex-1 min-w-0">
                  <div class="text-xs text-white font-medium truncate">{{ event.description }}</div>
                  <div class="text-xs text-gray-500 mt-0.5">{{ formatTime(event.timestamp) }}</div>
                </div>
                <span class="text-xs px-1.5 py-0.5 rounded shrink-0" :class="getEventTypeBadge(event.type)">
                  {{ getEventTypeLabel(event.type) }}
                </span>
              </div>
              <div v-if="expandedEvents.includes(event.id) && event.details && Object.keys(event.details).length > 0" class="mt-2 pt-2 border-t border-gray-700">
                <div class="text-xs text-gray-400 space-y-0.5">
                  <div v-for="(value, key) in event.details" :key="key" class="flex gap-2">
                    <span class="text-gray-500 shrink-0">{{ formatDetailKey(key) }}:</span>
                    <span class="text-gray-300 truncate">{{ formatDetailValue(value) }}</span>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="p-2 border-t border-gray-800 text-xs text-gray-500 flex items-center justify-between">
        <span>共 {{ store.currentTimeline.length }} 条记录</span>
        <button v-if="store.currentTimeline.length > 0" @click="clearTimeline" class="text-red-400 hover:text-red-300">
          清空记录
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, h } from 'vue'
import { useOcrStore } from '../store/ocr'
import type { TimelineEventType } from '../types'

const store = useOcrStore()
const collapsed = ref(false)
const expandedEvents = ref<string[]>([])

const filters = [
  { type: 'all' as const, label: '全部', bgClass: 'bg-amber-600', textClass: 'text-white' },
  { type: 'ocr' as const, label: '识别', bgClass: 'bg-blue-600', textClass: 'text-white' },
  { type: 'correction' as const, label: '校改', bgClass: 'bg-yellow-600', textClass: 'text-white' },
  { type: 'annotation_add' as const, label: '标注', bgClass: 'bg-green-600', textClass: 'text-white' },
  { type: 'annotation_remove' as const, label: '删除', bgClass: 'bg-red-600', textClass: 'text-white' },
  { type: 'document_create' as const, label: '创建', bgClass: 'bg-purple-600', textClass: 'text-white' },
]

type FilterType = TimelineEventType | 'all'
const activeFilters = ref<FilterType[]>(['all'])

function toggleFilter(type: FilterType) {
  if (type === 'all') {
    activeFilters.value = ['all']
    return
  }
  activeFilters.value = activeFilters.value.filter(f => f !== 'all')
  const idx = activeFilters.value.indexOf(type)
  if (idx > -1) {
    activeFilters.value.splice(idx, 1)
    if (activeFilters.value.length === 0) {
      activeFilters.value = ['all']
    }
  } else {
    activeFilters.value.push(type)
  }
}

const filteredEvents = computed(() => {
  if (activeFilters.value.includes('all')) {
    return store.currentTimeline
  }
  return store.currentTimeline.filter(e => activeFilters.value.includes(e.type))
})

function toggleExpand(id: string) {
  const idx = expandedEvents.value.indexOf(id)
  if (idx > -1) {
    expandedEvents.value.splice(idx, 1)
  } else {
    expandedEvents.value.push(id)
  }
}

function formatTime(isoString: string): string {
  const date = new Date(isoString)
  const now = new Date()
  const diffMs = now.getTime() - date.getTime()
  const diffMins = Math.floor(diffMs / 60000)
  const diffHours = Math.floor(diffMs / 3600000)
  const diffDays = Math.floor(diffMs / 86400000)

  if (diffMins < 1) return '刚刚'
  if (diffMins < 60) return `${diffMins} 分钟前`
  if (diffHours < 24) return `${diffHours} 小时前`
  if (diffDays < 7) return `${diffDays} 天前`
  
  return date.toLocaleString('zh-CN', {
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit'
  })
}

function getEventTypeLabel(type: TimelineEventType): string {
  const labels: Record<TimelineEventType, string> = {
    ocr: '识别',
    correction: '校改',
    annotation_add: '标注',
    annotation_remove: '删除',
    document_create: '创建'
  }
  return labels[type]
}

function getEventTypeBadge(type: TimelineEventType): string {
  const badges: Record<TimelineEventType, string> = {
    ocr: 'bg-blue-900 text-blue-300',
    correction: 'bg-yellow-900 text-yellow-300',
    annotation_add: 'bg-green-900 text-green-300',
    annotation_remove: 'bg-red-900 text-red-300',
    document_create: 'bg-purple-900 text-purple-300'
  }
  return badges[type]
}

function getEventIconBg(type: TimelineEventType): string {
  const bgs: Record<TimelineEventType, string> = {
    ocr: 'bg-blue-500',
    correction: 'bg-yellow-500',
    annotation_add: 'bg-green-500',
    annotation_remove: 'bg-red-500',
    document_create: 'bg-purple-500'
  }
  return bgs[type]
}

function getEventIconColor(type: TimelineEventType): string {
  return 'text-white'
}

function getEventIcon(type: TimelineEventType) {
  const icons: Record<TimelineEventType, any> = {
    ocr: () => h('svg', { fill: 'none', stroke: 'currentColor', viewBox: '0 0 24 24', 'stroke-width': '2' }, [
      h('path', { 'stroke-linecap': 'round', 'stroke-linejoin': 'round', d: 'M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z' })
    ]),
    correction: () => h('svg', { fill: 'none', stroke: 'currentColor', viewBox: '0 0 24 24', 'stroke-width': '2' }, [
      h('path', { 'stroke-linecap': 'round', 'stroke-linejoin': 'round', d: 'M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z' })
    ]),
    annotation_add: () => h('svg', { fill: 'none', stroke: 'currentColor', viewBox: '0 0 24 24', 'stroke-width': '2' }, [
      h('path', { 'stroke-linecap': 'round', 'stroke-linejoin': 'round', d: 'M12 4v16m8-8H4' })
    ]),
    annotation_remove: () => h('svg', { fill: 'none', stroke: 'currentColor', viewBox: '0 0 24 24', 'stroke-width': '2' }, [
      h('path', { 'stroke-linecap': 'round', 'stroke-linejoin': 'round', d: 'M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16' })
    ]),
    document_create: () => h('svg', { fill: 'none', stroke: 'currentColor', viewBox: '0 0 24 24', 'stroke-width': '2' }, [
      h('path', { 'stroke-linecap': 'round', 'stroke-linejoin': 'round', d: 'M9 13h6m-3-3v6m5 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z' })
    ])
  }
  return icons[type]
}

function formatDetailKey(key: string): string {
  const keyMap: Record<string, string> = {
    resultId: '识别ID',
    oldValue: '原值',
    newValue: '新值',
    annotationId: '标注ID',
    type: '类型',
    label: '标签',
    content: '内容',
    documentId: '文档ID',
    resultCount: '识别行数'
  }
  return keyMap[key] || key
}

function formatDetailValue(value: any): string {
  if (typeof value === 'string') return value
  if (typeof value === 'number') return value.toString()
  return JSON.stringify(value)
}

function clearTimeline() {
  if (confirm('确定要清空当前文档的所有时间轴记录吗？')) {
    if (store.currentDoc) {
      store.timelineEvents = store.timelineEvents.filter(e => e.documentId !== store.currentDoc!.id)
    }
  }
}
</script>
