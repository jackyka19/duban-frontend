<template>
  <div class="app-container">
    <el-container>
      <el-header style="background: #409EFF; color: white; text-align: center; line-height: 60px;">
        <h1 style="margin: 0;">督辦事項進度統計系統</h1>
      </el-header>
      <el-main>
        <!-- Filters -->
        <el-card class="filter-card">
          <el-form :inline="true" :model="filters">
            <el-form-item label="時間範圍">
              <el-date-picker
                v-model="filters.dateRange"
                type="monthrange"
                range-separator="至"
                start-placeholder="開始月份"
                end-placeholder="結束月份"
                value-format="YYYY-MM"
              />
            </el-form-item>
            <el-form-item label="進度狀態">
              <el-select v-model="filters.progresses" multiple placeholder="選擇進度" style="width: 200px">
                <el-option label="未開展" value="未開展" />
                <el-option label="推進中" value="推進中" />
                <el-option label="已完成" value="已完成" />
              </el-select>
            </el-form-item>
            <el-form-item>
              <el-checkbox v-model="filters.onlyUnreported">僅顯示未反饋/超期事項</el-checkbox>
            </el-form-item>
            <el-form-item>
              <el-button type="primary" @click="loadData">查詢</el-button>
              <el-button @click="resetFilters">重置</el-button>
              <el-button type="success" @click="exportExcel">導出Excel</el-button>
            </el-form-item>
          </el-form>
        </el-card>

        <!-- Stats Cards -->
        <el-row :gutter="20" style="margin: 20px 0;">
          <el-col :span="6">
            <el-card><div class="stat-item"><div class="stat-num">{{ stats.total }}</div><div>總事項</div></div></el-card>
          </el-col>
          <el-col :span="6">
            <el-card><div class="stat-item"><div class="stat-num" style="color:#E6A23C">{{ stats.notStarted }}</div><div>未開展</div></div></el-card>
          </el-col>
          <el-col :span="6">
            <el-card><div class="stat-item"><div class="stat-num" style="color:#409EFF">{{ stats.inProgress }}</div><div>推進中</div></div></el-card>
          </el-col>
          <el-col :span="6">
            <el-card><div class="stat-item"><div class="stat-num" style="color:#67C23A">{{ stats.completed }}</div><div>已完成</div></div></el-card>
          </el-col>
        </el-row>

        <!-- Charts -->
        <el-row :gutter="20">
          <el-col :span="12">
            <el-card>
              <div ref="pieChart" style="height: 350px;"></div>
            </el-card>
          </el-col>
          <el-col :span="12">
            <el-card>
              <div ref="barChart" style="height: 350px;"></div>
            </el-card>
          </el-col>
        </el-row>

        <!-- Table -->
        <el-card style="margin-top: 20px;">
          <template #header>
            <span>督辦事項列表</span>
            <span style="float: right; color: #F56C6C;" v-if="overdueCount > 0">
              超期未反饋: {{ overdueCount }} 項
            </span>
          </template>
          <el-table :data="tableData" stripe border style="width: 100%">
            <el-table-column prop="id" label="ID" width="60" />
            <el-table-column prop="name" label="事項名稱" min-width="200" />
            <el-table-column prop="startDate" label="發起日期" width="100" />
            <el-table-column prop="deadline" label="時限要求" width="100" />
            <el-table-column prop="progress" label="當前進度" width="100">
              <template #default="{ row }">
                <el-tag :type="getProgressType(row.progress)">{{ row.progress }}</el-tag>
              </template>
            </el-table-column>
            <el-table-column prop="department" label="責任部門" width="100" />
            <el-table-column prop="feedbackDeadline" label="反饋時限" width="100" />
            <el-table-column label="狀態" width="120">
              <template #default="{ row }">
                <el-tag v-if="row.overdue" type="danger">超期未反饋</el-tag>
                <el-tag v-else type="success">正常</el-tag>
              </template>
            </el-table-column>
          </el-table>
        </el-card>
      </el-main>
    </el-container>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, nextTick } from 'vue'
import axios from 'axios'
import * as echarts from 'echarts'
import { ElMessage } from 'element-plus'

const filters = ref({
  dateRange: [],
  progresses: [],
  onlyUnreported: false
})

const tableData = ref([])
const stats = ref({ total: 0, notStarted: 0, inProgress: 0, completed: 0, overdue: 0 })

const pieChart = ref(null)
const barChart = ref(null)
let pieInstance = null
let barInstance = null

const overdueCount = computed(() => tableData.value.filter(i => i.overdue).length)

const getProgressType = (progress) => {
  if (progress === '已完成') return 'success'
  if (progress === '推進中') return 'primary'
  return 'warning'
}

const loadData = async () => {
  try {
    const params = {
      startDate: filters.value.dateRange?.[0] || '',
      endDate: filters.value.dateRange?.[1] || '',
      progresses: filters.value.progresses,
      onlyUnreported: filters.value.onlyUnreported
    }
    const res = await axios.get('/api/supervision/items', { params })
    tableData.value = res.data
    await loadStats()
    await nextTick()
    renderCharts()
  } catch (e) {
    ElMessage.error('加載數據失敗')
  }
}

const loadStats = async () => {
  try {
    const res = await axios.get('/api/supervision/stats')
    stats.value = res.data
  } catch (e) {}
}

const renderCharts = () => {
  // Pie chart - progress distribution
  if (!pieInstance) pieInstance = echarts.init(pieChart.value)
  const progressCount = { '未開展': 0, '推進中': 0, '已完成': 0 }
  tableData.value.forEach(i => progressCount[i.progress]++)
  
  pieInstance.setOption({
    title: { text: '進度分佈', left: 'center' },
    tooltip: { trigger: 'item' },
    legend: { bottom: 0 },
    series: [{
      type: 'pie',
      radius: '60%',
      data: [
        { value: progressCount['未開展'], name: '未開展', itemStyle: { color: '#E6A23C' } },
        { value: progressCount['推進中'], name: '推進中', itemStyle: { color: '#409EFF' } },
        { value: progressCount['已完成'], name: '已完成', itemStyle: { color: '#67C23A' } }
      ]
    }]
  })

  // Bar chart - by department
  if (!barInstance) barInstance = echarts.init(barChart.value)
  const deptMap = {}
  tableData.value.forEach(i => {
    deptMap[i.department] = (deptMap[i.department] || 0) + 1
  })
  const depts = Object.keys(deptMap)
  const counts = depts.map(d => deptMap[d])

  barInstance.setOption({
    title: { text: '部門事項分佈', left: 'center' },
    tooltip: { trigger: 'axis' },
    xAxis: { type: 'category', data: depts },
    yAxis: { type: 'value' },
    series: [{ type: 'bar', data: counts, itemStyle: { color: '#409EFF' } }]
  })
}

const resetFilters = () => {
  filters.value = { dateRange: [], progresses: [], onlyUnreported: false }
  loadData()
}

const exportExcel = () => {
  const params = new URLSearchParams()
  if (filters.value.dateRange?.[0]) params.append('startDate', filters.value.dateRange[0])
  if (filters.value.dateRange?.[1]) params.append('endDate', filters.value.dateRange[1])
  filters.value.progresses.forEach(p => params.append('progresses', p))
  params.append('onlyUnreported', filters.value.onlyUnreported)
  
  window.open(`/api/supervision/export/excel?${params.toString()}`)
}

onMounted(() => {
  loadData()
  window.addEventListener('resize', () => {
    pieInstance?.resize()
    barInstance?.resize()
  })
})
</script>

<style>
.app-container { min-height: 100vh; background: #f5f7fa; }
.filter-card { margin-bottom: 20px; }
.stat-item { text-align: center; padding: 10px 0; }
.stat-num { font-size: 28px; font-weight: bold; }
</style>
