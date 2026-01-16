<template>
  <view class="container">
    <!-- 1. 固定导航栏 -->
    <view class="navbar-fixed">
      <Navbar title="客户确定列表"></Navbar>
    </view>
    
    <!-- 2. 内容区域 -->
    <scroll-view class="content-container" scroll-y="true">
      
      <!-- Loading 状态 -->
      <view v-if="loading" class="loading-container">
        <!-- 如果没有Loading组件，可以临时用 text 替代，或者确保引入了组件 -->
        <Loading :show="loading"></Loading>
      </view>

      <!-- 错误/空状态 -->
      <view v-else-if="error || listData.length === 0" class="error-container">
        <text class="error-text">{{ error  }}</text>
        <button class="retry-btn" @click="fetchData">刷新</button>
      </view>

      <!-- 表格区域 -->
      <view v-else class="table-container">
        <!-- 固定表头 -->
        <view class="table-header">
          <view class="header-row">
            <view class="header-cell col-1">客户简称</view>
            <view class="header-cell col-2">通知单号</view>
            <view class="header-cell col-3">送货客户简称</view>
            <view class="header-cell col-4 last-col">创建人</view>
          </view>
        </view>
        
        <!-- 表格内容 -->
        <view class="table-body">
          <view 
            class="table-row" 
            v-for="(item, index) in listData" 
            :key="index"
            @click="handleRowClick(item)"
          >
            <!-- 客户简称 -->
            <view class="table-cell col-1">{{ item['客户简称'] }}</view>
            
            <!-- 通知单号 -->
            <view class="table-cell col-2">{{ item['通知单号'] }}</view>
            
            <!-- 送货客户简称 (分两行显示：日期在上，时分秒在下) -->
            <view class="table-cell col-3 multi-line">
              <text class="date-text">{{ getDatePart(item['送货客户简称']) }}</text>
              <text class="time-text">{{ getTimePart(item['送货客户简称']) }}</text>
            </view>
            
            <!-- 创建人 -->
            <view class="table-cell col-4 last-col">{{ item['创建人'] }}</view>
          </view>
        </view>
      </view>
      
      <!-- 底部防遮挡占位 -->
      <view style="height: 40rpx;"></view>
    </scroll-view>
  </view>
</template>

<script setup>
import { ref } from 'vue'
import { onLoad, onShow, onPullDownRefresh } from '@dcloudio/uni-app'
import Navbar from "@/components/Navbar.vue"
import Loading from "@/components/Loading.vue"
// ==========================================
// 🛠️ SN 配置器 (测试/生产模式切换)
// ==========================================
const IS_TEST_MODE = false   // true = 使用固定SN; false = 使用上页传来SN
const TEST_SN = "1a9960ec5662f9" 
// ==========================================

// --- 数据定义 ---
const listData = ref([])
const loading = ref(true)
const error = ref(null)
const currentSN = ref('')

// --- 1. onLoad: 初始化参数 ---
onLoad((options) => {
  if (IS_TEST_MODE) {
    currentSN.value = TEST_SN
    console.log('⚠️ [KehuQueding-1] 测试模式，使用固定SN:', currentSN.value)
  } else {
    currentSN.value = options.SN ? decodeURIComponent(options.SN) : ''
    console.log('✅ [KehuQueding-1] 生产模式，接收SN:', currentSN.value)
  }
})

// --- 2. onShow: 每次显示页面刷新数据 ---
onShow(() => {
  if (currentSN.value) {
    fetchData()
  } else {
    loading.value = false
    if (!IS_TEST_MODE) error.value = "未获取到设备SN"
  }
})

// --- 3. 下拉刷新 ---
onPullDownRefresh(() => {
  fetchData()
})

// --- 核心：获取数据 ---
const fetchData = async () => {
  loading.value = true
  error.value = null

  const payload = {
    "SN": currentSN.value,
    "SQL": "SCDATA_YLT_20250930"
  }

  try {
    const res = await uni.request({
      url: 'http://13.94.38.44:8080/CP_Sale/GetSalDoStockList',
      method: 'POST',
      data: payload,
      timeout: 10000
    })

    uni.stopPullDownRefresh()
    console.log('客户确定列表返回:', res.data)

    if (res.statusCode === 200) {
      let result = res.data
      // JSON 字符串解析容错处理
      if (typeof result === 'string') {
        try { result = JSON.parse(result.trim()) } catch (e) {}
      }

      if (result && !result.isError) {
        listData.value = result.dt || []
      } else {
        // 如果出错或者 dt 为空
        console.error(result.errMsg)
        if (!listData.value.length) error.value = result.errMsg || '此设备无权限查看，请联系管理员'
      }
    } else {
      throw new Error(`HTTP状态码: ${res.statusCode}`)
    }
  } catch (err) {
    console.error('请求异常:', err)
    uni.stopPullDownRefresh()
    error.value = '网络请求失败'
  } finally {
    loading.value = false
  }
}

// --- 交互逻辑 ---
// 在 KehuQueding-1.vue 中

const handleRowClick = (item) => {
  console.log('点击了行:', item)
  
  const docNum = item['通知单号']
  if (!docNum) return

  // 携带 docnum 和 SN 跳转到 KehuQueding-2
  uni.navigateTo({
    url: `/pages/kehu/KehuQueding-2/KehuQueding-2?docnum=${encodeURIComponent(docNum)}&SN=${encodeURIComponent(currentSN.value)}`
  })
}

// --- 时间处理工具函数 ---

// 1. 获取日期部分 (2025/09/29)
const getDatePart = (val) => {
  if (!val) return ''
  // 原始数据: "2025-09-29 10:00:00"
  // split(' ')[0] 拿到 "2025-09-29"
  // replace 变成 "2025/09/29"
  return val.split(' ')[0].replace(/-/g, '/')
}

// 2. 获取时间部分 (10:00:00)
const getTimePart = (val) => {
  if (!val) return ''
  const parts = val.split(' ')
  // 如果没有时间部分，返回空
  if (parts.length < 2) return ''
  // 返回 "10:00:00" (保留秒)
  return parts[1]
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

/* 1. 导航栏固定 */
.navbar-fixed {
  position: fixed;
  top: 0; left: 0; right: 0;
  z-index: 1000;
  width: 100%;
}

/* 2. 内容区域容器 */
.content-container {
  flex: 1;
  margin-top: 90rpx; /* 避开 Navbar */
  height: calc(100vh - 90rpx);
}

/* Loading / Error 样式 */
.loading-container, .error-container {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  height: 400rpx;
  color: #999;
}

.retry-btn {
  margin-top: 20rpx;
  font-size: 28rpx;
  background-color: #007aff;
  color: #fff;
}

/* --- 表格样式 --- */
.table-container {
  padding-top: 20rpx;
  display: flex;
  flex-direction: column;
}

/* 表头 Sticky */
.table-header {
  position: sticky;
  top: 0;
  z-index: 999;
  background-color: #f5f5f5;
  border-top: 1rpx solid #ddd;
  border-bottom: 1rpx solid #000; /* 加深下划线 */
  box-shadow: 0 2rpx 5rpx rgba(0,0,0,0.05);
}

.header-row {
  display: flex;
}

.header-cell {
  padding: 24rpx 2rpx;
  text-align: center;
  font-weight: bold;
  font-size: 28rpx;
  border-right: 1rpx solid #eee;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f5f5f5;
}

/* 表体 */
.table-body {
  background-color: #fff;
}

.table-row {
  display: flex;
  border-bottom: 1rpx solid #eee;
}

.table-row:active {
  background-color: #f0f0f0; /* 点击反馈 */
}

/* 单元格通用样式 */
.table-cell {
  padding: 24rpx 2rpx;
  text-align: center;
  font-size: 26rpx;
  border-right: 1rpx solid #eee;
  display: flex;
  align-items: center;
  justify-content: center;
  
  /* 默认不允许换行，防止高度撑开 */
  white-space: nowrap; 
  overflow: hidden; 
  text-overflow: ellipsis;
}

/* --- 特殊样式：日期时间列 (col-3) --- */
.col-3.multi-line {
  /* 允许内容垂直排列 */
  flex-direction: column !important;
  line-height: 1.3;
  /* 覆盖默认的 nowrap，但因为 flex-col，其实无需换行 */
  white-space: normal;
}

.date-text {
  font-size: 26rpx;

}

.time-text {
  font-size: 26rpx;

}

/* --- 核心：列宽分配 (Flex 布局) --- */
/* 
   总比例设定：
   1. 客户简称 (Flex 2)
   2. 通知单号 (Flex 3)
   3. 需求时间 (Flex 3.5)
   4. 创建人   (Flex 1.5)
*/

.col-1 { flex: 2.6; }    /* 客户简称 */
.col-2 { flex: 3.2; }    /* 通知单号 */
.col-3 { flex: 2.7; }  /* 送货客户简称 */
.col-4 { flex: 1.5; }  /* 创建人 */

/* 最后一列去掉右边框 */
.last-col {
  border-right: none;
}
</style>