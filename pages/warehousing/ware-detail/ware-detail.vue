<!-- 待入库明细 -->
<template>
  <view class="container">
    <view class="navbar-fixed">
      <Navbar></Navbar>
    </view>
    
    <scroll-view class="content-container" scroll-y="true">
      <view v-if="loading" class="loading-container">
        <Loading :show="loading"></Loading>
      </view>

      <view v-else-if="error" class="error-container">
        <text class="error-text">{{ error }}</text>
        <button class="retry-btn" @click="fetchReceiveData">重新加载</button>
      </view>

      <view v-else class="table-container">
        <view class="table-header">
          <view class="header-row">
            <view class="header-cell clickable" @click="openSearch('供应商', '供应商')">
              供应商 <text class="filter-icon">▼</text>
            </view>
            
            <view class="header-cell clickable" @click="openSearch('订单号', '订单号', '供应商')">
              订单号 <text class="filter-icon">▼</text>
            </view>
            
            <view class="header-cell">业务料号</view>
            
            <view class="header-cell">待入库数</view>
          </view>
        </view>
        
        <scroll-view class="table-body" scroll-y="true">
          <view 
            class="table-row" 
            v-for="(item, index) in receiveData" 
            :key="index"
            :class="{ 'selected-row': selectedIndex === index }"
            @click="selectRow(index)"
          >
            <view class="table-cell">{{ item['供应商'] }}</view>
            <view class="table-cell">{{ item['订单号'] }}</view>
            <view class="table-cell">{{ item['业务料号'] }}</view>
            <view class="table-cell">{{ item['待入库数'] }}</view>
          </view>
        </scroll-view>
        
        <view style="height: 140rpx; width: 100%;"></view>
      </view>
    </scroll-view>

    <view class="bottom-bar">
      <button 
        class="btn-half reset-btn" 
        :class="{ 'btn-disabled': receiveData.length === allReceiveData.length }"
        :disabled="receiveData.length === allReceiveData.length"
        @click="resetFilter"
      >
        {{ receiveData.length !== allReceiveData.length ? '重置筛选' : '未进行筛选' }}
      </button>

      <button 
        class="btn-half action-btn" 
        :class="{ 'btn-disabled': selectedIndex === -1 }"
        :disabled="selectedIndex === -1"
        @click="handleStartStockIn"
      >
        {{ selectedIndex !== -1 ? '开始入库' : '选择订单' }}
      </button>
    </view>

    <SearchModal
      v-model:show="showSearchModal"
      :title="searchTitle"
      :list="searchModalList"
      :searchKey="searchField"
      :subKey="searchSubField"
      @select="handleSearchSelect"
    />
  </view>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import Navbar from "@/components/Navbar.vue"
import { onLoad } from '@dcloudio/uni-app';
import Loading from "@/components/Loading.vue"
// 【新增】引入搜索组件
import SearchModal from "@/components/SearchModal.vue"

// 响应式数据
const receiveData = ref([])     // 页面展示的数据（可能是筛选后的）
const allReceiveData = ref([])  // 【新增】原始全量数据（仓库）
const loading = ref(true)
const error = ref(null)

// 【新增】交互与筛选相关的变量
const selectedIndex = ref(-1)   // 当前选中的行索引
const showSearchModal = ref(false)
const searchTitle = ref('')
const searchField = ref('')
const searchSubField = ref('')
const searchModalList = ref([]) // 传给弹窗的去重列表

const apiParams = ref({
  SN: '',
  SQL: 'SCDATA_YLT_20250930'
});

onLoad((options) => {
  console.log('接收的参数:', options);
  const SN = options.SN;
  if (!SN) {
    // 注意：这里原代码中 errorMsg 未定义，建议用 error 或 uni.showToast
    error.value = '参数错误：缺少订单号或设备码'; 
    return;
  }
  apiParams.value.SN = SN;
  fetchReceiveData();
});

// 获取待入库数据 
const fetchReceiveData = async () => {
  loading.value = true
  error.value = null
  selectedIndex.value = -1 // 重置选中状态

  try {
    const response = await uni.request({
      url: 'http://13.94.38.44:8080/YLT_CpRk/dt_Mainper',
      method: 'POST',
      data: apiParams.value,
      timeout: 10000
    })
    console.log("入库明细接口数据", response);
    if (response.statusCode === 200) {
      let result
      if (typeof response.data === 'string') {
        result = JSON.parse(response.data)
      } else {
        result = response.data
      }

      if (result.isError === false && result.dt && Array.isArray(result.dt)) {
        // 【核心修改】同时存入全量数据和展示数据
        allReceiveData.value = result.dt
        receiveData.value = result.dt
      } else {
        throw new Error(result.msg || '数据格式错误')
      }
    } else {
      throw new Error(`HTTP错误: ${response.statusCode}`)
    }
  } catch (err) {
    console.error('获取数据失败:', err)
    error.value = err.message || '获取数据失败，请检查网络连接'
  } finally {
    loading.value = false
  }
}

// 【新增】行选中处理
const selectRow = (index) => {
  if (selectedIndex.value === index) {
    selectedIndex.value = -1
  } else {
    selectedIndex.value = index
  }
}

// 【新增】打开搜索弹窗（含去重逻辑）
const openSearch = (field, title, sub = '') => {
  if (allReceiveData.value.length === 0) {
    uni.showToast({ title: '暂无数据', icon: 'none' })
    return
  }
  
  searchField.value = field
  searchTitle.value = title
  searchSubField.value = sub
  
  // 供应商去重逻辑
  if (field === '供应商') {
    const allSuppliers = allReceiveData.value.map(item => item['供应商'])
    const uniqueSuppliers = [...new Set(allSuppliers)]
    
    let list = uniqueSuppliers.map(name => ({
      [field]: name 
    }))
    // 添加“显示全部”选项
    list.unshift({ [field]: '显示全部', isReset: true })
    searchModalList.value = list
    
  } else {
    // 订单号直接显示（也可以根据需要去重）
    // 为了体验一致，也给订单号加一个“显示全部”
    const list = [...allReceiveData.value]
    list.unshift({ [field]: '显示全部', isReset: true, [sub]: '' })
    searchModalList.value = list
  }

  showSearchModal.value = true
}

// 【新增】处理筛选选择
const handleSearchSelect = (selectedItem) => {
  if (selectedItem.isReset) {
    resetFilter()
    return
  }

  const field = searchField.value
  const value = selectedItem[field]

  uni.showLoading({ title: '筛选中...' })

  // 从全量数据中筛选
  const filtered = allReceiveData.value.filter(item => {
    return item[field] === value
  })

  receiveData.value = filtered
  selectedIndex.value = -1 
  uni.hideLoading()
  
  uni.showToast({ title: `已筛选: ${value}`, icon: 'none' })
}

// 【新增】重置筛选
const resetFilter = () => {
  receiveData.value = [...allReceiveData.value]
  selectedIndex.value = -1
  uni.showToast({ title: '已显示全部', icon: 'none' })
}

// 【新增】开始入库（跳转逻辑）
const handleStartStockIn = () => {
  if (selectedIndex.value === -1) return

  const selectedItem = receiveData.value[selectedIndex.value]
  const docNum = selectedItem['订单号']

  if (!docNum) {
    uni.showToast({ title: '订单号无效', icon: 'none' })
    return
  }

  // 注意：请确认入库扫码页面的路径是否正确
  uni.navigateTo({
    url: `/pages/warehousing/ware-scan/ware-scan?DocNum=${encodeURIComponent(docNum)}&SN=${encodeURIComponent(apiParams.value.SN)}`
  })
}

// 页面加载时获取数据
onMounted(() => {
  fetchReceiveData()
})
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
  margin-top: 90rpx;
  height: calc(100vh - 90rpx);
}

.loading-container {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 90rpx;
}

.loading-text {
  font-size: 28rpx;
  color: #666;
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
  background-color: #007aff;
  color: white;
  border: none;
  border-radius: 10rpx;
  padding: 20rpx 40rpx;
  font-size: 28rpx;
}

.table-container {
  padding-top: 40rpx;
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
  border-radius: 12rpx 12rpx 0 0;
  box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
  overflow: hidden;
}

.header-row {
  display: flex;
  background-color: #f5f5f5;
}

.header-cell {
  padding: 24rpx 1rpx;
  text-align: center;
  font-weight: bold;
  font-size: 30rpx;
  border-right: 1rpx solid #eee;
  border-bottom: 1rpx solid #eee;
  background-color: #f5f5f5;
}

.header-cell:last-child {
  border-right: none;
}

/* 表格主体样式 */
.table-body {
  flex: 1;
  border: 1rpx solid #ddd;
  border-top: none;
  border-radius: 0 0 12rpx 12rpx;
  background-color: #fff;
  box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
  overflow: hidden;
}

.table-row {
  display: flex;
  border-bottom: 1rpx solid #eee;
}

.table-row:last-child {
  border-bottom: none;
}

.table-cell {
  padding: 24rpx 1rpx;
  text-align: center;
  font-size: 28rpx;
  border-right: 1rpx solid #eee;
  word-break: break-all;
  overflow-wrap: break-word;
  white-space: nowrap;
}

.table-cell:last-child {
  border-right: none;
}

/* --- 核心修改：按比例分配列宽 (针对4列数据) --- */

/* 1. 供应商 */
.header-cell:nth-child(1),
.table-cell:nth-child(1) {
  flex: 2; 
}

/* 2. 订单号 */
.header-cell:nth-child(2),
.table-cell:nth-child(2) {
  flex: 3; 
}

/* 3. 业务料号 (通常较长，给最大的比例) */
.header-cell:nth-child(3),
.table-cell:nth-child(3) {
  flex: 3.2; 
}

/* 4. 待入库数 (数字较短，给最小的比例) */
.header-cell:nth-child(4),
.table-cell:nth-child(4) {
  flex: 1.5; 
  border-right: none; /* 最后一列去掉右边框 */
}

/* 响应式适配小屏幕 */
@media screen and (max-width: 768px) {
  .header-cell,
  .table-cell {
    padding: 16rpx 1rpx;
    font-size: 26rpx;
  }
}

/* --- 新增交互样式 --- */

/* 可点击的表头 */
.clickable {
  color: #007aff;
  position: relative;
}

.clickable:active {
  background-color: #e0e0e0;
}

/* 筛选图标 */
.filter-icon {
  font-size: 20rpx;
  margin-left: 6rpx;
  color: #666;
  vertical-align: middle;
}

/* 选中行的背景色 */
.selected-row {
  background-color: #DDECF6 !important;
}

/* --- 底部操作栏样式 --- */

.bottom-bar {
  position: fixed;
  bottom: 0; left: 0;
  width: 100%;
  height: 120rpx;
  background-color: #fff;
  border-top: 1rpx solid #ccc;
  display: flex;
  align-items: center;
  justify-content: space-between; /* 按钮左右分布 */
  padding: 0 40rpx;
  box-sizing: border-box;
  z-index: 1000;
  box-shadow: 0 -2rpx 10rpx rgba(0,0,0,0.05);
  gap: 20rpx; /* 按钮间距 */
}

/* 通用半宽按钮 */
.btn-half {
  flex: 1;
  height: 80rpx;
  line-height: 80rpx;
  font-size: 30rpx;
  border-radius: 10rpx;
  border: none;
  padding: 0;
  color: white;
}

/* 重置按钮（橙色） */
.reset-btn {
  background-color: #ff9f43;
}

/* 入库按钮（蓝色） */
.action-btn {
  background-color: #007aff;
}

/* 禁用状态 */
.btn-disabled {
  background-color: #cccccc !important;
  color: #ffffff !important;
  opacity: 1;
}
</style>