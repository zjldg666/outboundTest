<template>
  <view class="container">
    <!-- 固定在顶部的导航栏 -->
    <view class="navbar-fixed">
      <!-- 这里的 title 可以根据你的 Navbar 组件支持的属性进行修改 -->
      <Navbar></Navbar>
    </view>
    
    <!-- 页面内容区域，给上边距避开导航栏 -->
    <scroll-view class="content-container" scroll-y="true">
      <!-- 数据加载状态 -->
      <view v-if="loading" class="loading-container">
        <!-- 如果没有Loading组件，可以临时用 text 替代，或者确保引入了组件 -->
        <Loading :show="loading"></Loading>
      </view>

      <!-- 错误状态 -->
      <view v-else-if="error" class="error-container">
        <text class="error-text">{{ error }}</text>
        <button class="retry-btn" @click="fetchOutboundData">重新加载</button>
      </view>

      <!-- 暂无数据状态 -->
      <view v-else-if="listData.length === 0" class="error-container">
        <text class="error-text">暂无待出库数据</text>
        <button class="retry-btn" @click="fetchOutboundData">刷新</button>
      </view>

      <!-- 数据表格 -->
      <view v-else class="table-container">
        <!-- 固定表头 -->
        <view class="table-header">
          <view class="header-row">
            <view class="header-cell">送货客户简称</view>
            <view class="header-cell">通知单号</view>
            <view class="header-cell">需求出库时间</view>
          </view>
        </view>
        
        <!-- 可滚动的内容区域 -->
        <scroll-view class="table-body" scroll-y="true">
          <view 
            class="table-row" 
            v-for="(item, index) in listData" 
            :key="index"
            :class="{ 'highlight-yellow': item.num > 0 }"
			@click="goToDetail(item)" 
          >
            <view class="table-cell">{{ item['客户简称'] }}</view>
            <view class="table-cell">{{ item['送货单号'] }}</view>
            <view class="table-cell">{{ formatTime(item['需求出库时间']) }}</view>
          </view>
        </scroll-view>
      </view>
    </scroll-view>
  </view>
</template>

<script setup>
import { ref } from 'vue'
import { onLoad, onShow, onPullDownRefresh } from '@dcloudio/uni-app'
import Navbar from "@/components/Navbar.vue"
import Loading from "@/components/Loading.vue"

// ==========================================
// 🛠️ 配置区域
// ==========================================
const IS_TEST_MODE = false   // true = 使用固定SN测试; false = 使用首页传来的SN
const TEST_SN = "1a9960ec5662f946" 
// ==========================================

// 响应式数据
const listData = ref([])
const loading = ref(true)
const error = ref(null)
const currentSN = ref('')

// --- 1. onLoad: 初始化 SN ---
onLoad((options) => {
  // 仅负责初始化参数，不负责请求数据（交给 onShow）
  if (IS_TEST_MODE) {
    currentSN.value = TEST_SN
    console.log('⚠️ [列表页] 当前处于测试模式，使用固定SN:', currentSN.value)
  } else {
    // 生产模式：优先使用传来的SN
    currentSN.value = options.SN ? decodeURIComponent(options.SN) : ''
    console.log('✅ [列表页] 当前处于生产模式，接收到的SN:', currentSN.value)
  }
})

// --- 2. onShow: 每次页面显示都刷新数据 ---
// 无论是第一次进入，还是从 outbound-List1 返回，都会触发这里
onShow(() => {
  if (currentSN.value) {
    console.log('👀 页面显示，正在刷新数据...')
    fetchOutboundData()
  } else {
    // 如果没有SN，停止加载动画并提示
    loading.value = false
    if (!IS_TEST_MODE) error.value = "未获取到设备SN，请重新扫描"
  }
})

// --- 3. 下拉刷新 ---
onPullDownRefresh(() => {
  fetchOutboundData()
})

// --- 获取数据核心逻辑 ---
const fetchOutboundData = async () => {
  // 开启加载状态 (如果你不想每次返回都闪烁Loading，可以注释掉下面这行)
  loading.value = true 
  error.value = null

  const requestPayload = {
    "SN": currentSN.value, 
    "SQL": "SCDATA_YLT_20250930"
  }

  try {
    const response = await uni.request({
      url: 'http://13.94.38.44:8080/OutStockConfirm/GetOutStockList',
      method: 'POST',
      data: requestPayload,
      timeout: 10000 // 10秒超时
    })
    
    uni.stopPullDownRefresh() // 停止下拉刷新动画
    console.log("列表接口数据:", response.data);

    if (response.statusCode === 200) {
      // 1. 处理可能返回的字符串格式 JSON
      let result = response.data
      if (typeof result === 'string') {
        try {
          // 清理可能的特殊字符并解析
          result = JSON.parse(result.trim())
        } catch (e) {
          throw new Error('服务器返回数据格式异常')
        }
      }

      // 2. 检查业务逻辑错误
      if (result.isError === false) {
        // 赋值，如果没有dt则给空数组
        listData.value = result.dt || []
      } else {
        // 只有明确报错才显示错误，数据为空不算错
        // listData.value = [] // 可以在这里清空，或者保留旧数据
        console.error(result.errMsg)
        // 只有当列表完全为空时才显示全屏错误，否则用 Toast
        if (listData.value.length === 0) {
           error.value = result.errMsg || '数据库无此账号，请联系管理员配置权限'
        }
      }
    } else {
      throw new Error(`HTTP错误: ${response.statusCode}`)
    }
  } catch (err) {
    console.error('请求异常:', err)
    error.value = '网络请求失败，请检查网络'
    uni.stopPullDownRefresh()
  } finally {
    loading.value = false
  }
}

// --- 跳转到详情页 (outbound-List1) ---
const goToDetail = (item) => {
  const docNum = item['送货单号'];
  if (!docNum) return;

  // 携带 docnum 和 当前使用的 SN 跳转
  uni.navigateTo({
    url: `/pages/outbound/outbound-List1/outbound-List1?docnum=${encodeURIComponent(docNum)}&SN=${encodeURIComponent(currentSN.value)}`
  })
}

// --- 时间格式化 ---
const formatTime = (val) => {
  if (!val) return ''
  return val 
}
</script>

<style>
.container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background-color: white;
  position: relative;
}

/* 固定导航栏样式 */
.navbar-fixed {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 1000;
  width: 100%;
}

/* 内容容器，避开固定导航栏 */
.content-container {
  flex: 1;
  margin-top: 90rpx; /* 根据实际Navbar高度调整 */
  height: calc(100vh - 90rpx);
}

.loading-container {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 90rpx;
  height: 50vh;
}

.error-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40rpx;
  margin-top: 90rpx;
}

.error-text {
  font-size: 28rpx;
  color: #ff4757;
  text-align: center;
  margin-bottom: 30rpx;
  line-height: 1.5;
}
.retry-btn {
  margin-top: 20rpx;
  font-size: 28rpx;
  background-color: #007aff;
  color: #fff;
}

/* 表格容器 */
.table-container {
  padding-top: 20rpx;
  flex: 1;
  display: flex;
  flex-direction: column;
}

/* 固定表头样式 */
.table-header {
  position: sticky;
  top: 0;
  z-index: 999;
  background-color: #f5f5f5;
  border: 1rpx solid #ddd;
  border-bottom: none;
  box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
}

.header-row {
  display: flex;
  background-color: #f5f5f5;
}

/* --- 表头单元格通用样式 --- */
.header-cell {
  padding: 24rpx 2rpx; /* 减少左右内边距，挤出更多空间 */
  text-align: center;
  font-weight: bold;
  font-size: 28rpx; /* 表头字体稍微调小一点点 */
  border-right: 1rpx solid #eee;
  border-bottom: 1rpx solid #000; /* 加深下边框 */
  background-color: #f5f5f5;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* --- 表格主体样式 --- */
.table-body {
  flex: 1;
  border: 1rpx solid #ddd;
  border-top: none;
  background-color: #fff;
  box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
}

.table-row {
  display: flex;
  border-bottom: 1rpx solid #eee;
}

.table-row:last-child {
  border-bottom: none;
}

/* --- 数据单元格通用样式 --- */
.table-cell {
  padding: 24rpx 2rpx;
  text-align: center;
  font-size: 26rpx; /* 数据字体调小，确保能显示更多内容 */
  border-right: 1rpx solid #eee;
  
  display: flex;
  align-items: center;
  justify-content: center;
  
  /* 强制不换行，超出部分显示省略号 */
  white-space: nowrap; 
  overflow: hidden; 
  text-overflow: ellipsis;
}

/* --- 核心修改：按比例分配列宽 --- */

/* 1. 送货客户简称 (内容较短，占比最小) */
.header-cell:nth-child(1),
.table-cell:nth-child(1) {
  flex: 2.5; /* 权重 2 */
}

/* 2. 通知单号 (内容中等) */
.header-cell:nth-child(2),
.table-cell:nth-child(2) {
  flex: 3.5; /* 权重 3 */
}

/* 3. 需求出库时间 (内容最长，给予最大空间) */
.header-cell:nth-child(3),
.table-cell:nth-child(3) {
  flex: 4; /* 权重 4.5 */
  border-right: none; /* 最后一列去掉右边框 */
}

/* --- 高亮样式 --- */
.highlight-yellow {
  background-color: #DDECF6 !important; /* 强制黄色背景 */
}

/* 响应式适配超小屏幕 */
@media screen and (max-width: 360px) {
  .header-cell,
  .table-cell {
    font-size: 22rpx; /* 小屏幕进一步缩小字体 */
  }
}
</style>