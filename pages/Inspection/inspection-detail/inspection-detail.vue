<template>
  <view class="container">
    <view class="navbar-fixed">
      <Navbar title="待检验明细"></Navbar>
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
        <button class="retry-btn" @click="fetchData">重新加载</button>
      </view>

      <view v-else class="table-container">
        
        <scroll-view scroll-x="true" class="table-scroll-x">
          
          <view class="table-header">
            <view class="header-row min-width-row">
              <view class="header-cell clickable" @click="openSearch('收货单号', '收货单号', '供应商')">
                收货单号 <text class="filter-icon">▼</text>
              </view>
              
              <view class="header-cell clickable" @click="openSearch('供应商', '供应商')">
                供应商 <text class="filter-icon">▼</text>
              </view>
              
              <view class="header-cell clickable" @click="openSearch('业务料号', '业务料号')">
                业务料号 <text class="filter-icon">▼</text>
              </view>
              
              <view class="header-cell">待检验数</view>
            </view>
          </view>
          
          <view class="table-body">
            <view 
              class="table-row min-width-row" 
              v-for="(item, index) in listData" 
              :key="index"
              :id="'row-' + index" 
              :class="{ 'selected-row': selectedIndex === index }"
              @click="selectRow(index)"
            >
              <view class="table-cell">{{ item['收货单号'] }}</view>
              <view class="table-cell">{{ item['供应商'] }}</view>
              <view class="table-cell">{{ item['业务料号'] }}</view>
              <view class="table-cell">{{ item['待检验数'] }}</view>
            </view>
          </view>
          
        </scroll-view>
        <view style="height: 140rpx; width: 100%;"></view>
      </view>
    </scroll-view>

    <view class="bottom-bar">
      <button 
        class="btn-half reset-btn" 
        :class="{ 'btn-disabled': listData.length === allData.length }"
        :disabled="listData.length === allData.length"
        @click="resetFilter"
      >
        {{ listData.length !== allData.length ? '重置筛选' : '未进行筛选' }}
      </button>

      <button 
        class="btn-half action-btn" 
        :class="{ 'btn-disabled': selectedIndex === -1 }"
        :disabled="selectedIndex === -1"
        @click="handleStartInspection"
      >
        {{ selectedIndex !== -1 ? '开始检验' : '选择单据' }}
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
import Loading from "@/components/Loading.vue";
import { onLoad, onUnload } from '@dcloudio/uni-app'
import SearchModal from "@/components/SearchModal.vue"

// 响应式数据
const listData = ref([])       // 当前展示的数据
const allData = ref([])        // 原始全量数据
const loading = ref(true)
const error = ref(null)
const selectedIndex = ref(-1)  // 当前选中行的索引
const scrollId = ref('')

// 搜索相关
const showSearchModal = ref(false)
const searchTitle = ref('')
const searchField = ref('')
const searchSubField = ref('')
const searchModalList = ref([])

const apiParams = ref({
  SN: '',
  SQL: 'SCDATA_YLT_20250930'
});

onLoad((options) => {
  console.log('待检验明细接收参数:', options);
  const SN = options.SN;
  
  if (SN) {
    apiParams.value.SN = SN;
  }
  
  fetchData();

  // 监听刷新事件（例如从扫码页返回时）
  uni.$on('refreshInspectionList', () => {
    console.log('收到刷新信号，开始刷新待检验列表...');
    fetchData();
  });
});

onUnload(() => {
  uni.$off('refreshInspectionList');
});

// 行点击
const selectRow = (index) => {
  if (selectedIndex.value === index) {
    selectedIndex.value = -1
  } else {
    selectedIndex.value = index
  }
}

// 获取数据
const fetchData = async () => {
  loading.value = true
  error.value = null
  selectedIndex.value = -1 // 重置选中

  try {
    const response = await uni.request({
      url: 'http://13.94.38.44:8080/YLT_SalIQC_Check/dt_Mainper',
      method: 'POST',
      data: apiParams.value,
      timeout: 10000
    })
    
    console.log("待检验接口返回:", response);

    if (response.statusCode === 200) {
      let result
      if (typeof response.data === 'string') {
        try {
           result = JSON.parse(response.data)
        } catch(e) {
           result = response.data
        }
      } else {
        result = response.data
      }

      if (result.isError === false && result.dt && Array.isArray(result.dt)) {
        allData.value = result.dt
        listData.value = result.dt
      } else {
        // 如果数据为空或出错
        if(result.dt === null || result.dt === undefined) {
             allData.value = []
             listData.value = []
        } else {
             throw new Error(result.msg || '数据格式错误')
        }
      }
    } else {
      throw new Error(`HTTP错误: ${response.statusCode}`)
    }
  } catch (err) {
    console.error('获取待检验数据失败:', err)
    error.value = err.message || '获取数据失败，请检查网络连接'
  } finally {
    loading.value = false
  }
}

// 打开搜索
const openSearch = (field, title, sub = '') => {
  if (allData.value.length === 0) {
    uni.showToast({ title: '暂无数据', icon: 'none' })
    return
  }
  
  searchField.value = field
  searchTitle.value = title
  searchSubField.value = sub
  
  // 数据去重处理
  if (field === '供应商' || field === '业务料号') {
    const allValues = allData.value.map(item => item[field])
    const uniqueValues = [...new Set(allValues)]
    searchModalList.value = uniqueValues.map(val => ({
      [field]: val 
    }))
  } else {
    // 收货单号等唯一值直接显示
    searchModalList.value = allData.value
  }

  showSearchModal.value = true
}

// 执行筛选
const handleSearchSelect = (selectedItem) => {
  const field = searchField.value
  const value = selectedItem[field]

  uni.showLoading({ title: '筛选中...' })

  const filtered = allData.value.filter(item => {
    return item[field] === value
  })

  listData.value = filtered
  selectedIndex.value = -1 
  uni.hideLoading()
  
  uni.showToast({ title: `已筛选: ${value}`, icon: 'none' })
}

// 重置筛选
const resetFilter = () => {
  listData.value = [...allData.value]
  selectedIndex.value = -1
  uni.showToast({ title: '已显示全部', icon: 'none' })
}

// 开始检验跳转
const handleStartInspection = () => {
  if (selectedIndex.value === -1) return

  const selectedItem = listData.value[selectedIndex.value]
  
  const docNum = selectedItem['订单号']

  if (!docNum) {
    uni.showToast({ title: '单号无效', icon: 'none' })
    return
  }

  // 跳转到检验扫码页 (注意：这里假设扫码页路径为 inspection-scan，请确保该页面已创建)
  uni.navigateTo({
    url: `/pages/Inspection/inspection-scan/inspection-scan?DocNum=${encodeURIComponent(docNum)}&SN=${encodeURIComponent(apiParams.value.SN)}`
  })
}

onMounted(() => {
  // 可以在这里做一些初始化
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

.navbar-fixed {
  position: fixed;
  top: 0; left: 0; right: 0;
  z-index: 1000;
  width: 100%;
}

.content-container {
  flex: 1;
  margin-top: 90rpx;
  height: calc(100vh - 90rpx); 
}

.loading-container, .error-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin-top: 90rpx;
  padding: 40rpx;
}

.error-text {
  font-size: 28rpx;
  color: #ff4757;
  text-align: center;
  margin-bottom: 30rpx;
}

.retry-btn {
  background-color: #007aff;
  color: white;
  border: none;
  border-radius: 10rpx;
  padding: 20rpx 40rpx;
  font-size: 28rpx;
}

/* 表格容器 */
.table-container {
  padding-top: 40rpx;
  flex: 1;
  display: flex;
  flex-direction: column;
}

.table-scroll-x {
  width: 100%;
  overflow-x: auto;
}

/* 强制行宽 */
.min-width-row {
  min-width: 100%; /* 4列内容不多，可以直接占满屏幕，如果内容长可改为固定像素如 900rpx */
  display: flex;
}

.table-header {
  position: sticky;
  top: 0;
  z-index: 999;
  background-color: #f5f5f5;
  border: 1rpx solid #ddd;
  border-bottom: none;
  width: fit-content;
  min-width: 100%;
}

.header-row { display: flex; background-color: #f5f5f5; }

.header-cell {
  padding: 24rpx 2rpx;
  text-align: center;
  font-weight: bold;
  font-size: 28rpx;
  border-right: 1rpx solid #eee;
  border-bottom: 1rpx solid #000;
  background-color: #f5f5f5;
  display: flex; align-items: center; justify-content: center;
}

.table-body {
  width: fit-content;
  min-width: 100%;
  border: 1rpx solid #ddd;
  border-top: none;
  background-color: #fff;
}

.table-row {
  display: flex;
  border-bottom: 1rpx solid #eee;
}

.table-cell {
  padding: 24rpx 2rpx;
  text-align: center;
  font-size: 26rpx;
  border-right: 1rpx solid #eee;
  display: flex; align-items: center; justify-content: center;
  white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
}

/* --- 列宽分配 (4列) --- */

/* 1. 收货单号 (较长) */
.header-cell:nth-child(1), .table-cell:nth-child(1) {
  flex: 3;
}

/* 2. 供应商 */
.header-cell:nth-child(2), .table-cell:nth-child(2) {
  flex: 2;
}

/* 3. 业务料号 (较长) */
.header-cell:nth-child(3), .table-cell:nth-child(3) {
  flex: 3;
}

/* 4. 待检验数 (较短) */
.header-cell:nth-child(4), .table-cell:nth-child(4) {
  flex: 1.5;
  border-right: none;
}

/* 底部栏 */
.bottom-bar {
  position: fixed; bottom: 0; left: 0;
  width: 100%; height: 120rpx;
  background-color: #fff;
  border-top: 1rpx solid #ccc;
  display: flex; align-items: center; justify-content: space-between;
  gap: 20rpx; padding: 0 40rpx;
  box-sizing: border-box; z-index: 1000;
  box-shadow: 0 -2rpx 10rpx rgba(0,0,0,0.05);
}

.btn-half {
  flex: 1; height: 80rpx; line-height: 80rpx;
  font-size: 30rpx; border-radius: 10rpx;
  border: none; padding: 0;
}

.reset-btn { background-color: #ff9f43; color: white; }
.action-btn { background-color: #007aff; color: white; }
.btn-disabled { background-color: #ccc !important; color: #fff !important; }

.selected-row { background-color: #DDECF6 !important; }

.clickable { color: #007aff; }
.clickable:active { background-color: #e0e0e0; }
.filter-icon { font-size: 20rpx; margin-left: 6rpx; color: #666; }
</style>