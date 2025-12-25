<template>
  <div id="app">
    <h1>立法委員質詢統計</h1>

    <!-- 篩選面板 -->
    <div class="filter-panel">
      <div class="filter-group">
        <label>委員會篩選</label>
        <div style="margin-bottom: 10px;">
          <button @click="selectAllCommittees">全選常設委員會</button>
          <button @click="clearCommittees" style="margin-left: 10px; background-color: #6c757d;">清除</button>
        </div>
        <div class="checkbox-group">
          <label v-for="committee in committees" :key="committee.id">
            <input type="checkbox" :value="committee.id" v-model="selectedCommittees" />
            {{ committee.name }}
          </label>
        </div>
      </div>
    </div>

    <!-- 統計摘要 -->
    <div class="stats-summary">
      <div class="stat-item">
        <span class="stat-label">總委員數</span>
        <span class="stat-value">{{ filteredStats.length }}</span>
      </div>
      <div class="stat-item">
        <span class="stat-label">總發言次數</span>
        <span class="stat-value">{{ totalSpeeches.toLocaleString() }}</span>
      </div>
      <div class="stat-item">
        <span class="stat-label">總發言時長</span>
        <span class="stat-value">{{ formatDuration(totalDuration) }}</span>
      </div>
    </div>

    <div class="stats-table">
      <table v-if="!loading && filteredStats.length > 0">
        <thead>
          <tr>
            <th @click="sort('rank')" :class="getSortClass('rank')">排名</th>
            <th @click="sort('speaker')" :class="getSortClass('speaker')">委員姓名</th>
            <th @click="sort('total_speeches')" :class="getSortClass('total_speeches')">發言次數</th>
            <th @click="sort('total_duration')" :class="getSortClass('total_duration')">總發言時長</th>
            <th @click="sort('avg_duration')" :class="getSortClass('avg_duration')">平均時長</th>
            <th>參與委員會</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in paginatedStats" :key="item.speaker">
            <td>{{ item.rank }}</td>
            <td><strong>{{ item.speaker }}</strong></td>
            <td>{{ item.total_speeches.toLocaleString() }}</td>
            <td>{{ formatDuration(item.total_duration) }}</td>
            <td>{{ formatDuration(item.avg_duration) }}</td>
            <td>{{ item.committees_text }}</td>
          </tr>
        </tbody>
      </table>
      <div v-else-if="loading" class="loading">載入中...</div>
      <div v-else class="no-data">無資料</div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const loading = ref(true)
const rawData = ref(null)
const committees = ref([])
const selectedCommittees = ref([])
const sortKey = ref('total_speeches')
const sortOrder = ref('desc')

onMounted(async () => {
  try {
    const committeesRes = await fetch('/data/committees.json')
    const committeesData = await committeesRes.json()
    committees.value = committeesData.filter(c => c.type === 'standing')

    selectedCommittees.value = committees.value.map(c => c.id)

    const statsRes = await fetch('/data/aggregated.json')
    rawData.value = await statsRes.json()

    loading.value = false
  } catch (error) {
    console.error('載入資料失敗:', error)
    loading.value = false
  }
})

// 篩選和計算統計
const filteredStats = computed(() => {
  if (!rawData.value) return []

  const result = []

  for (const [speaker, data] of Object.entries(rawData.value)) {
    // 篩選委員會
    const filteredCommittees = {}
    let totalSpeeches = 0
    let totalDuration = 0

    for (const [org, stats] of Object.entries(data.committees)) {
      if (selectedCommittees.value.includes(org)) {
        filteredCommittees[org] = stats
        totalSpeeches += stats.count
        totalDuration += stats.duration
      }
    }

    if (totalSpeeches === 0) continue

    // 取得委員會名稱
    const committeeNames = Object.keys(filteredCommittees).map(id => {
      const committee = committees.value.find(c => c.id === id)
      return committee ? committee.name.replace('委員會', '') : id
    })

    result.push({
      speaker,
      total_speeches: totalSpeeches,
      total_duration: totalDuration,
      avg_duration: totalDuration / totalSpeeches,
      committees: filteredCommittees,
      committees_text: committeeNames.join('、')
    })
  }

  // 排序
  result.sort((a, b) => {
    let aVal = a[sortKey.value]
    let bVal = b[sortKey.value]

    if (sortKey.value === 'speaker') {
      return sortOrder.value === 'asc'
        ? aVal.localeCompare(bVal, 'zh-TW')
        : bVal.localeCompare(aVal, 'zh-TW')
    }

    return sortOrder.value === 'asc' ? aVal - bVal : bVal - aVal
  })

  // 加入排名
  result.forEach((item, index) => {
    item.rank = index + 1
  })

  return result
})

// 分頁顯示（先顯示全部，之後可加入分頁）
const paginatedStats = computed(() => filteredStats.value)

// 總計
const totalSpeeches = computed(() => {
  return filteredStats.value.reduce((sum, item) => sum + item.total_speeches, 0)
})

const totalDuration = computed(() => {
  return filteredStats.value.reduce((sum, item) => sum + item.total_duration, 0)
})

// 排序
function sort(key) {
  if (sortKey.value === key) {
    sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc'
  } else {
    sortKey.value = key
    sortOrder.value = key === 'speaker' ? 'asc' : 'desc'
  }
}

function getSortClass(key) {
  const classes = ['sortable']
  if (sortKey.value === key) {
    classes.push(sortOrder.value === 'asc' ? 'sorted-asc' : 'sorted-desc')
  }
  return classes.join(' ')
}

// 委員會選擇
function selectAllCommittees() {
  selectedCommittees.value = committees.value.map(c => c.id)
}

function clearCommittees() {
  selectedCommittees.value = []
}

// 時長格式化
function formatDuration(seconds) {
  const hours = Math.floor(seconds / 3600)
  const minutes = Math.floor((seconds % 3600) / 60)
  const secs = Math.floor(seconds % 60)

  if (hours > 0) {
    return `${hours}:${String(minutes).padStart(2, '0')}:${String(secs).padStart(2, '0')}`
  } else {
    return `${minutes}:${String(secs).padStart(2, '0')}`
  }
}
</script>
