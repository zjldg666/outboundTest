<template>
  <view class="container">
    <!-- 1. 固定导航栏 -->
    <view class="navbar-fixed">
      <Navbar title="通知单明细"></Navbar>
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
        <text class="error-text">{{ error || '暂无明细数据' }}</text>
        <button class="retry-btn" @click="fetchDetailData">刷新</button>
      </view>

      <!-- 表格区域 -->
      <view v-else class="table-container">
        <!-- 固定表头 -->
        <view class="table-header">
          <view class="header-row">
            <view class="header-cell col-1">业务料号</view>
            <view class="header-cell col-2">客户料号</view>
            <view class="header-cell col-3 last-col">送货数量</view>
          </view>
        </view>
        
        <!-- 表格内容 -->
        <view class="table-body">
          <view 
            class="table-row" 
            v-for="(item, index) in listData" 
            :key="index"
          >
            <!-- 业务料号 -->
            <view class="table-cell col-1">{{ item['业务料号'] }}</view>
            
            <!-- 客户料号 -->
            <view class="table-cell col-2">{{ item['客户料号'] }}</view>
            
            <!-- 送货数量 -->
            <view class="table-cell col-3 last-col">{{ item['送货数量'] }}</view>
          </view>
        </view>
      </view>
      
      <!-- 底部占位 -->
      <view style="height: 120rpx;"></view>
    </scroll-view>

    <!-- 3. 底部固定按钮 -->
    <view class="bottom-bar">
          <button class="btn-back" @click="goBack">返回</button>
          
          <button 
            class="btn-confirm" 
            :class="{ 'btn-disabled': !hasPermission }"
            :disabled="!hasPermission"
            @click="handlePickup"
          >
            {{ hasPermission ? '客户提货' : '无提货权限' }}
          </button>
    </view>
  </view>
</template>

<script setup>
import { ref } from 'vue'
import { onLoad } from '@dcloudio/uni-app'
import Navbar from "@/components/Navbar.vue"
import Loading from "@/components/Loading.vue"
// ==========================================
// 🛠️ SN 配置器
// ==========================================
const IS_TEST_MODE = false  
const TEST_SN = "1a9960ec5662f946" 
// ==========================================
// 权限变量，默认为 false 
const hasPermission = ref(false)
// --- 数据定义 ---
const listData = ref([])
const loading = ref(true)
const error = ref(null)
const currentDocNum = ref('')
const currentSN = ref('')

// --- 生命周期 ---
onLoad((options) => {
  if (options.docnum) {
    currentDocNum.value = decodeURIComponent(options.docnum)
  }

  if (IS_TEST_MODE) {
    currentSN.value = TEST_SN
    console.log('⚠️ [KehuQueding-2] 测试模式 SN:', currentSN.value)
  } else {
    currentSN.value = options.SN ? decodeURIComponent(options.SN) : ''
    console.log('✅ [KehuQueding-2] 生产模式 SN:', currentSN.value)
  }

  fetchDetailData()
})

// --- 1. 获取明细数据 ---
const fetchDetailData = () => {
  if (!currentDocNum.value || !currentSN.value) {
    error.value = "参数缺失"
    loading.value = false
    return
  }

  loading.value = true
  error.value = null

  const payload = {
    "SN": currentSN.value,
    "SQL": "SCDATA_YLT_20250930",
    "docnum": currentDocNum.value
  }

  uni.request({
    url: 'http://13.94.38.44:8080/CP_Sale/bindDgvDetail_Notify',
    method: 'POST',
    data: payload,
    success: (res) => {
          console.log('明细接口返回:', res.data)
          let result = res.data
          if (typeof result === 'string') {
            try { result = JSON.parse(result.trim()) } catch (e) {}
          }
    
          // 🔥🔥🔥 修改开始：解析 flag 和 dtDetail 🔥🔥🔥
          
          // 1. 获取权限 (注意：这个接口返回的是 flag，不是 falg)
          if (result && typeof result.flag !== 'undefined') {
            hasPermission.value = result.flag === true;
          } else {
            hasPermission.value = false; // 没返回就默认没权限
          }
    
          // 2. 获取列表数据 (优先找 dtDetail，找不到再找 dt)
          let finalData = []
          if (result && Array.isArray(result.dtDetail)) {
            finalData = result.dtDetail
          } else if (result && Array.isArray(result.dt)) {
            finalData = result.dt
          } else if (Array.isArray(result)) {
            finalData = result
          }
    
          if (finalData.length > 0) {
            listData.value = finalData
          } else {
            if (result && result.isError) {
                 error.value = result.errMsg || '获取失败'
            } else {
                 listData.value = [] 
            }
          }
        },
    fail: (err) => {
      console.error(err)
      error.value = '网络请求失败'
    },
    complete: () => {
      loading.value = false
    }
  })
}

// --- 2. 交互逻辑 ---

const goBack = () => {
  uni.navigateBack()
}

// --- 3. 客户提货 (核心功能 - 修改确认框位置) ---
const handlePickup = () => {
  if (!currentDocNum.value || !currentSN.value) {
    uni.showToast({ title: '参数缺失，无法提货', icon: 'none' })
    return
  }

  // 1. 弹出确认框 (使用反向逻辑调整按钮位置)
  uni.showModal({
    title: '确认提货',
    content: `单号：${currentDocNum.value}\n确认执行客户提货操作吗？`,
    showCancel: true,

    // --- 核心修改 ---
    // 左边按钮 (API的 confirm) -> 显示为 "取消" (灰色)
    confirmText: '取消',
    confirmColor: '#666666',

    // 右边按钮 (API的 cancel) -> 显示为 "确定" (蓝色)
    cancelText: '确定',
    cancelColor: '#007aff',

    success: (res) => {
      // 因为我们把“确定”文字放在了 cancel 按钮上
      // 所以这里要判断 res.cancel 为 true 时，才是用户点的“确定”
      if (res.cancel) {
        // 用户点击确定，执行请求
        executePickupRequest()
      } else if (res.confirm) {
        console.log('用户点击了取消')
      }
    }
  })
}

// 执行提货请求
const executePickupRequest = () => {
  uni.showLoading({ title: '提交中...', mask: true })

  const payload = {
    "SN": currentSN.value,
    "SQL": "SCDATA_YLT_20250930",
    "docnum": currentDocNum.value
  }
  
  console.log('正在提交提货参数:', payload)

  uni.request({
    url: 'http://13.94.38.44:8080/CP_Sale/SongHuo',
    method: 'POST',
    data: payload,
    success: (res) => {
      uni.hideLoading()
      console.log('提货接口返回:', res.data)
      
      let result = res.data
      if (typeof result === 'string') {
        try { result = JSON.parse(result.trim()) } catch (e) {}
      }

      if (res.statusCode === 200 && result && result.isError === false) {
        uni.showToast({ title: '提货成功', icon: 'success' })
        
        // 成功后延迟 1.5秒 返回上一页
        setTimeout(() => {
          uni.navigateBack()
        }, 1500)
        
      } else {
        uni.showToast({ 
          title: result.errMsg || result.msg || '提货失败', 
          icon: 'none' 
        })
      }
    },
    fail: (err) => {
      uni.hideLoading()
      console.error(err)
      uni.showToast({ title: '网络请求失败', icon: 'none' })
    }
  })
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

/* 导航栏固定 */
.navbar-fixed {
  position: fixed;
  top: 0; left: 0; right: 0;
  z-index: 1000;
  width: 100%;
}

/* 内容区域 */
.content-container {
  flex: 1;
  margin-top: 90rpx; 
  height: calc(100vh - 90rpx);
}

/* Loading / Error */
.loading-container, .error-container {
  display: flex; flex-direction: column;
  justify-content: center; align-items: center;
  height: 400rpx; color: #999;
}
.retry-btn { margin-top: 20rpx; font-size: 28rpx; background-color: #007aff; color: #fff; }

/* 表格样式 */
.table-container {
  padding-top: 20rpx;
  display: flex; flex-direction: column;
}

.table-header {
  position: sticky; top: 0; z-index: 999;
  background-color: #f5f5f5;
  border-top: 1rpx solid #ddd;
  border-bottom: 1rpx solid #000; 
  box-shadow: 0 2rpx 5rpx rgba(0,0,0,0.05);
}

.header-row { display: flex; }

.header-cell {
  padding: 24rpx 2rpx;
  text-align: center; font-weight: bold; font-size: 28rpx;
  border-right: 1rpx solid #eee;
  display: flex; align-items: center; justify-content: center;
  background-color: #f5f5f5;
}

.table-body { background-color: #fff; }

.table-row {
  display: flex; border-bottom: 1rpx solid #eee;
}

.table-cell {
  padding: 24rpx 4rpx;
  text-align: center; font-size: 26rpx;
  border-right: 1rpx solid #eee;
  display: flex; align-items: center; justify-content: center;
  word-break: break-all;
  overflow-wrap: break-word;
}

/* 列宽分配 */
.col-1 { flex: 2.5; }
.col-2 { flex: 3.5; }
.col-3 { flex: 1.5; }
.last-col { border-right: none; }

/* 底部按钮 */
.bottom-bar {
  position: fixed; bottom: 0; left: 0; width: 100%; height: 120rpx;
  background-color: #fff; border-top: 1rpx solid #ccc;
  display: flex; align-items: center; justify-content: space-between;
  padding: 0 40rpx; box-sizing: border-box; z-index: 1000;
  box-shadow: 0 -2rpx 10rpx rgba(0,0,0,0.05);
}

.btn-back {
  width: 30%; background-color: #f1f1f1; color: #333; font-size: 30rpx;
}

.btn-confirm {
  width: 65%; background-color: #007aff; color: white; font-size: 30rpx;
}

.btn-confirm:active { opacity: 0.8; }
/*  新增：灰色禁用样式 */
.btn-disabled {
  background-color: #ccc !important;
  color: #fff;
  cursor: not-allowed;
}
/* 防止禁用时点击还有透明度变化效果 */
.btn-disabled:active { opacity: 1; }
</style>