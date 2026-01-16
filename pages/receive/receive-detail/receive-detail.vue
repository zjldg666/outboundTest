<!-- 待收货明细 -->
<template>
  <view class="container">
    <view class="navbar-fixed">
      <Navbar></Navbar>
    </view>
    
    <scroll-view 
          class="content-container" 
          scroll-y="true"
          :scroll-into-view="scrollId"
          scroll-with-animation
        >
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
                  <view class="header-cell">
                      预交日期
                    </view>
                    
                    <view class="header-cell clickable" @click="openSearch('供应商', '供应商')">
                      供应商 <text class="filter-icon">▼</text>
                    </view>
                    
                    <view class="header-cell clickable" @click="openSearch('订单号', '订单号', '供应商')">
                      订单号 <text class="filter-icon">▼</text>
                    </view>
                  <view class="header-cell">待收货数</view>
                </view>
              </view>
              
              <view class="table-body">
                  <view 
                    class="table-row" 
                    v-for="(item, index) in receiveData" 
                    :key="index"
                    :id="'row-' + index" 
                    :class="{ 'selected-row': selectedIndex === index }"
                    @click="selectRow(index)"
                  >
                    <view class="table-cell">{{ item['预交日期'] }}</view>
                    <view class="table-cell">{{ item['供应商'] }}</view>
                    <view class="table-cell">{{ item['订单号'] }}</view>
                    <view class="table-cell">{{ item['待收货数'] }}</view>
                  </view>
              </view>
        
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
            @click="handleStartReceive"
          >
            {{ selectedIndex !== -1 ? '开始收货' : '选择订单' }}
          </button>
    </view>
	<SearchModal
	      v-model:show="showSearchModal"
	      :title="searchTitle"
	      :list="searchModalList"   :searchKey="searchField"
	      :subKey="searchSubField"
	      @select="handleSearchSelect"
	/>
  </view>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import Navbar from "@/components/Navbar.vue"
import Loading from "@/components/Loading.vue";
import { onLoad } from '@dcloudio/uni-app';
// 搜索组件
import SearchModal from "@/components/SearchModal.vue"

// 响应式数据
const receiveData = ref([])
const loading = ref(true)
const error = ref(null)
// 记录当前选中行的索引，初始为 -1 (表示未选中)
const selectedIndex = ref(-1)
// 定义 scrollId 用于列表自动滚动定位 ---
const scrollId = ref('')
//搜索相关的变量和逻辑 ---
const showSearchModal = ref(false)
const searchTitle = ref('')
const searchField = ref('')     // 告诉组件搜哪个字段 (如 '订单号')
const searchSubField = ref('')  // 告诉组件辅助显示哪个字段 (如 '供应商')
const allReceiveData = ref([]) // 【用于存放接口返回的原始全量数据

// 专门给弹窗用的列表数据，避免直接污染 receiveData
const searchModalList = ref([])

const apiParams = ref({
  SN: '',
  SQL: 'SCDATA_YLT_20250930'
});

onLoad((options) => {
  console.log('接收的参数:', options);

  const SN = options.SN;
  
  if (!SN) {
    errorMsg.value = '参数错误：缺少订单号或设备码';
    return;
  }
  apiParams.value.SN = SN;
  fetchReceiveData();
});

// 行点击处理函数
const selectRow = (index) => {
  // 如果点击的是当前已选中的行，则取消选中；否则选中当前行
  if (selectedIndex.value === index) {
    selectedIndex.value = -1
  } else {
    selectedIndex.value = index
  }
}
// 获取待收货数据 
const fetchReceiveData = async () => {
  loading.value = true
  error.value = null

  try {
    const response = await uni.request({
      url: 'http://13.94.38.44:8080/YLT_SalIQC/dt_Mainper',
      method: 'POST', // 改为POST请求
    data:apiParams.value,
      timeout: 10000 // 10秒超时
    })
	console.log("收货明细接口数据",response);
    if (response.statusCode === 200) {
      // 接口返回的是字符串格式的JSON，需要先解析
      let result
      if (typeof response.data === 'string') {
        result = JSON.parse(response.data)
      } else {
        result = response.data
      }

      // 检查是否包含错误
      if (result.isError === false && result.dt && Array.isArray(result.dt)) {
		allReceiveData.value = result.dt // 先存一份原始数据
        receiveData.value = result.dt
      } else {
        throw new Error(result.msg || '数据格式错误')
      }
    } else {
      throw new Error(`HTTP错误: ${response.statusCode}`)
    }
  } catch (err) {
    console.error('获取待收货数据失败:', err)
    error.value = err.message || '获取数据失败，请检查网络连接'
  } finally {
    loading.value = false
  }
}
// 打开搜索弹窗的方法

const openSearch = (field, title, sub = '') => {
  // 1. 如果数据为空，拦截
  if (allReceiveData.value.length === 0) {
    uni.showToast({ title: '暂无数据', icon: 'none' })
    return
  }
  
  searchField.value = field
  searchTitle.value = title
  searchSubField.value = sub
  
  // 【关键】根据不同字段进行去重处理
  if (field === '供应商') {
    // 提取所有供应商名称
    const allSuppliers = allReceiveData.value.map(item => item['供应商'])
    // 使用 Set 去重
    const uniqueSuppliers = [...new Set(allSuppliers)]
    // 重新组装成对象数组，供 SearchModal 显示
    // 格式：[{ '供应商': '娃哈哈' }, { '供应商': '农夫山泉' }]
    searchModalList.value = uniqueSuppliers.map(name => ({
      [field]: name 
    }))
  } else if (field === '订单号') {
    // 订单号通常是唯一的，可以直接使用原始数据
    // 或者也做一次去重以防万一
    searchModalList.value = allReceiveData.value
  } else {
    // 其他情况
    searchModalList.value = allReceiveData.value
  }

  showSearchModal.value = true
}

// 处理搜索组件选中的结果
const handleSearchSelect = (selectedItem) => {
  const field = searchField.value       // 当前搜的是哪个字段，比如 '供应商'
  const value = selectedItem[field]     // 选中的值，比如 '娃哈哈'

  uni.showLoading({ title: '筛选中...' })

  // 【核心】从原始数据中过滤，只保留符合条件的
  const filtered = allReceiveData.value.filter(item => {
    return item[field] === value
  })

  // 更新展示列表
  receiveData.value = filtered
  
  // 重置选中状态（避免筛选后索引错乱）
  selectedIndex.value = -1 
  
  uni.hideLoading()
  
  // 提示用户
  uni.showToast({
    title: `已筛选: ${value}`,
    icon: 'none'
  })
}

// 【新增】重置筛选（可选，建议加上，否则用户筛完回不去）
const resetFilter = () => {
  receiveData.value = [...allReceiveData.value]
  selectedIndex.value = -1
  uni.showToast({ title: '已显示全部', icon: 'none' })
}

// 开始收货逻辑（增加校验）
const handleStartReceive = () => {
  // 1. 如果没有选中行，直接返回
  if (selectedIndex.value === -1) return

  // 2. 获取选中行的数据
  const selectedItem = receiveData.value[selectedIndex.value]
  
  // 3. 提取订单号 (根据你的表格显示，字段名是 '订单号')
  const docNum = selectedItem['订单号']

  if (!docNum) {
    uni.showToast({ title: '订单号无效', icon: 'none' })
    return
  }

  uni.navigateTo({
    url: `/pages/receive/receive-scan/receive-scan?DocNum=${encodeURIComponent(docNum)}&SN=${encodeURIComponent(apiParams.value.SN)}`
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
  box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
  overflow: hidden;
}

.header-row {
  display: flex;
  background-color: #f5f5f5;
}

/* 通用表头单元格样式 */
.header-cell {
  padding: 24rpx 2rpx;
  text-align: center;
  font-weight: bold;
  font-size: 28rpx;
  border-right: 1rpx solid #eee;
  border-bottom: 1rpx solid #000; /* 加深下边框 */
  background-color: #f5f5f5;
  display: flex;
  align-items: center;
  justify-content: center;
}


.table-body {
 
  width: 100%;
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

/* 通用表格数据单元格样式 */
.table-cell {
  padding: 24rpx 2rpx;
  text-align: center;
  font-size: 26rpx;
  border-right: 1rpx solid #eee;
  
  display: flex;
  align-items: center;
  justify-content: center;
  
  /* 文本溢出显示省略号 */
  white-space: nowrap; 
  overflow: hidden; 
  text-overflow: ellipsis;
}

/* --- 核心修改：按比例分配列宽 (针对4列数据) --- */

/* 1. 预交日期 */
.header-cell:nth-child(1),
.table-cell:nth-child(1) {
  flex: 2.5; 
}

/* 2. 供应商 */
.header-cell:nth-child(2),
.table-cell:nth-child(2) {
  flex: 2.5; 
}

/* 3. 订单号 */
.header-cell:nth-child(3),
.table-cell:nth-child(3) {
  flex: 3.2; 
}

/* 4. 待收货数 */
.header-cell:nth-child(4),
.table-cell:nth-child(4) {
  flex: 1.8; 
  border-right: none; /* 最后一列去掉右边框 */
}

/* 响应式适配小屏幕 */
@media screen and (max-width: 360px) {
  .header-cell,
  .table-cell {
    font-size: 22rpx;
  }
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
  justify-content: space-between; /* 两端对齐 */
  gap: 20rpx; /* 两个按钮之间的间距 */
  padding: 0 40rpx;
  box-sizing: border-box;
  z-index: 1000;
  box-shadow: 0 -2rpx 10rpx rgba(0,0,0,0.05);
}


.btn-half {
  flex: 1; 
  height: 80rpx;
  line-height: 80rpx;
  font-size: 30rpx;
  border-radius: 10rpx;
  border: none;
  padding: 0;
}
/* 左侧重置按钮样式 (激活状态) */
.reset-btn {
  background-color: #007aff; 
  color: white;
}

/* 右侧收货按钮样式 (激活状态) */
.action-btn {
  background-color: #007aff; /* 保持原来的蓝色 */
  color: white;
}

/* 按钮禁用状态（保持您原有的样式，但要确保优先级足够高） */
.btn-disabled {
  background-color: #cccccc !important; 
  color: #ffffff !important;
  opacity: 1; 
}

/* 开始收货按钮，选中行变为浅蓝色 */
.selected-row {
  background-color: #DDECF6 !important; 
}


.clickable {
  color: #007aff; 
  /* 如果不想一直蓝色，可以去掉上面这行，只保留下面的 active 效果 */
  position: relative;
}

/* 点击时的背景反馈 */
.clickable:active {
  background-color: #e0e0e0; 
}

/* 筛选小图标样式 */
.filter-icon {
  font-size: 20rpx;
  margin-left: 6rpx;
  color: #666;
  vertical-align: middle;
}
</style>