<template>
  <div class="sync-page">
    <!-- 页面头部：标题 + 操作区 -->
    <el-page-header content="优惠数据同步与差异对比">
      <template #extra>
        <!-- 同步按钮：根据状态禁用/切换文本 -->
        <el-button 
          type="primary" 
          :disabled="syncStore.syncStatus === 'processing'"
          @click="handleStartSync"
          icon="el-icon-refresh"
        >
          <template v-if="syncStore.syncStatus === 'processing'">
            <el-loading-spinner size="16"></el-loading-spinner>
            同步中...
          </template>
          <template v-else>
            发起数据同步
          </template>
        </el-button>
        <!-- 刷新差异按钮：仅同步完成后可点击 -->
        <el-button 
          type="default" 
          :disabled="!syncStore.lastSyncTime"
          @click="fetchDiffData"
          icon="el-icon-search"
          class="ml-2"
        >
          刷新差异结果
        </el-button>
      </template>
    </el-page-header>

    <!-- 同步进度条：仅同步中显示 -->
    <el-progress 
      v-if="syncStore.syncStatus === 'processing'"
      :percentage="syncStore.syncProgress"
      status="success"
      class="my-4"
    ></el-progress>

    <!-- 同步状态提示：显示最后同步时间/状态 -->
    <el-alert 
      :title="statusText" 
      :type="statusType" 
      show-icon 
      class="my-4"
    ></el-alert>

    <!-- 核心内容区：左侧图表统计 + 右侧差异列表 -->
    <div class="content-container">
      <!-- 左侧：差异数据可视化 -->
      <div class="chart-wrapper">
        <h3 class="section-title">数据核对结果统计</h3>
        <!-- ECharts 饼图：展示匹配/差异/待核对数量占比 -->
        <v-chart 
          :option="chartOption" 
          class="chart"
          :loading="!syncStore.diffData.length"
        />
      </div>

      <!-- 右侧：差异详情表格 -->
      <div class="table-wrapper">
        <h3 class="section-title">差异项详情（共 {{ syncStore.diffCount }} 条）</h3>
        <!-- 无差异时显示空状态 -->
        <el-empty 
          v-if="!syncStore.diffData.length && syncStore.lastSyncTime"
          description="暂无差异数据"
          class="empty-state"
        ></el-empty>
        
        <!-- 差异表格：高亮显示差异字段 -->
        <el-table 
          v-else
          :data="syncStore.diffData"
          border
          stripe
          :loading="syncStore.syncStatus === 'processing'"
          style="width: 100%"
        >
          <el-table-column 
            prop="discountId" 
            label="优惠ID" 
            width="120"
          ></el-table-column>
          <el-table-column 
            prop="discountName" 
            label="优惠名称" 
            min-width="200"
          ></el-table-column>
          <el-table-column 
            prop="source" 
            label="数据来源" 
            width="120"
          ></el-table-column>
          <el-table-column 
            prop="diffField" 
            label="差异字段" 
            width="120"
          >
            <template #default="scope">
              <el-tag type="warning">{{ scope.row.diffField }}</el-tag>
            </template>
          </el-table-column>
          <el-table-column 
            prop="localValue" 
            label="本地数据" 
            min-width="150"
          ></el-table-column>
          <el-table-column 
            prop="remoteValue" 
            label="远程数据" 
            min-width="150"
          >
            <template #default="scope">
              <span class="diff-value">{{ scope.row.remoteValue }}</span>
            </template>
          </el-table-column>
          <el-table-column 
            label="操作" 
            width="120"
          >
            <template #default="scope">
              <el-button 
                type="text" 
                @click="handleViewDetail(scope.row)"
                size="small"
              >
                查看详情
              </el-button>
            </template>
          </el-table-column>
        </el-table>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
// 1. 导入依赖（Vue API、Pinia、组件、工具函数等）
import { ref, computed, onMounted } from 'vue'
import { ElMessage, ElEmpty, ElAlert, ElProgress, ElButton, ElTable, ElTableColumn, ElTag, ElPageHeader, ElLoadingSpinner } from 'element-plus'
import { VChart, PieChart, Pie, Cell, Tooltip, Legend } from 'vue-echarts'
import { useSyncStore } from '@/store/sync' // 导入Pinia状态
import { syncApi } from '@/services/syncApi' // 导入同步相关API
import type { DiffItem } from '@/types/sync' // 导入差异数据类型定义

// 2. 初始化状态与依赖
const syncStore = useSyncStore() // 获取同步状态
let ws: WebSocket | null = null // WebSocket实例（用于监听实时进度）

// 3. 计算属性：根据同步状态生成提示文本和类型
const statusText = computed(() => {
  if (syncStore.syncStatus === 'processing') {
    return `同步中，当前进度：${syncStore.syncProgress}%`
  }
  if (syncStore.lastSyncTime) {
    return `最后同步时间：${syncStore.lastSyncTime.toLocaleString()}，共发现 ${syncStore.diffCount} 条差异`
  }
  return '未发起过同步，请点击“发起数据同步”开始'
})

const statusType = computed(() => {
  if (syncStore.syncStatus === 'processing') return 'info'
  if (syncStore.diffCount > 0) return 'warning'
  return 'success'
})

// 4. 图表配置：ECharts饼图选项（根据差异数据动态生成）
const chartOption = computed(() => {
  // 统计匹配/差异/待核对数量（假设diffData中包含status字段）
  const total = syncStore.diffData.length + (syncStore.diffData.length > 0 ? 100 : 0) // 模拟总数据量
  const matched = total - syncStore.diffCount // 匹配数量（模拟）
  const pending = 5 // 待核对数量（模拟）
  
  return {
    tooltip: { trigger: 'item' },
    legend: { bottom: 0, left: 'center' },
    series: [
      {
        name: '数据状态',
        type: 'pie',
        radius: ['40%', '70%'],
        avoidLabelOverlap: false,
        itemStyle: { borderRadius: 10, borderColor: '#fff', borderWidth: 2 },
        label: { show: false, position: 'center' },
        emphasis: {
          label: { show: true, fontSize: 16, fontWeight: 'bold' }
        },
        labelLine: { show: false },
        data: [
          { value: matched, name: '匹配项' },
          { value: syncStore.diffCount, name: '差异项' },
          { value: pending, name: '待核对项' }
        ],
        color: ['#10b981', '#f59e0b', '#6366f1'] // 绿/黄/紫：对应匹配/差异/待核对
      }
    ]
  }
})

// 5. 核心方法：发起同步、获取差异、监听WebSocket
const handleStartSync = async () => {
  try {
    // 1. 初始化同步状态
    syncStore.startSync()
    // 2. 发起同步请求（Axios）
    await syncApi.startSync()
    // 3. 建立WebSocket连接，监听实时进度和差异数据
    initWebSocket()
    ElMessage.success('同步任务已发起，正在处理...')
  } catch (err) {
    syncStore.syncStatus = 'error'
    ElMessage.error('同步发起失败，请重试')
    console.error('Sync error:', err)
  }
}

// 获取差异数据（手动刷新用）
const fetchDiffData = async () => {
  try {
    const res = await syncApi.getDiffData()
    syncStore.handleSyncResult({ diffData: res.data })
    ElMessage.success('差异数据已刷新')
  } catch (err) {
    ElMessage.error('刷新差异失败')
    console.error('Fetch diff error:', err)
  }
}

// 初始化WebSocket：监听同步进度和结果
const initWebSocket = () => {
  // 关闭已有连接（避免重复）
  if (ws) ws.close()
  
  // 建立新连接（使用环境变量中的WS地址）
  ws = new WebSocket(`ws://${import.meta.env.VITE_WS_HOST}/sync/progress`)
  
  // 监听消息：接收进度更新或同步完成通知
  ws.onmessage = (event) => {
    const data = JSON.parse(event.data)
    if (data.type === 'progress') {
      // 更新同步进度
      syncStore.updateProgress(data.value)
    } else if (data.type === 'complete') {
      // 同步完成：更新差异数据
      syncStore.handleSyncResult({ diffData: data.diffData })
      ElMessage.success('数据同步完成！')
    }
  }
  
  // 监听错误
  ws.onerror = (err) => {
    console.error('WebSocket error:', err)
    ElMessage.warning('实时进度监听异常，将手动刷新结果')
  }
  
  // 监听关闭
  ws.onclose = () => {
    console.log('WebSocket connection closed')
  }
}

// 查看差异详情（跳转/弹窗，此处简化为提示）
const handleViewDetail = (row: DiffItem) => {
  ElMessage.info(`查看优惠ID ${row.discountId} 的差异详情（实际项目中可跳转至详情页）`)
}

// 6. 生命周期：组件挂载时初始化WebSocket（可选，用于页面刷新后继续监听）
onMounted(() => {
  if (syncStore.syncStatus === 'processing') {
    initWebSocket()
  }
})

// 7. 生命周期：组件卸载时关闭WebSocket，避免内存泄漏
onUnmounted(() => {
  if (ws) ws.close()
})
</script>

<style scoped lang="scss">
.sync-page {
  padding: 20px;
  background-color: #f9fafb;
  min-height: 100vh;
}

.content-container {
  display: flex;
  gap: 20px;
  margin-top: 20px;
  .chart-wrapper, .table-wrapper {
    flex: 1;
    background: #fff;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  }
}

.section-title {
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 16px;
  color: #1f2937;
}

.chart {
  height: 300px;
}

.empty-state {
  height: 300px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.diff-value {
  color: #ef4444; // 红色高亮差异值
  font-weight: 500;
}
</style>
