<template>
  <view class="container">
    <view class="navbar-fixed">
      <Navbar title="出库详情"></Navbar>
    </view>
    
    <scroll-view class="content-container" scroll-y="true">
      <view v-if="loading" class="loading-container">
        <Loading :show="loading"></Loading>
      </view>

      <view v-else class="table-container">
        <view class="table-header">
          <view class="header-row">
            <view class="header-cell check-col">勾选</view>
            <view class="header-cell">库位</view>
            <view class="header-cell item-col">业务料号</view>
            <view class="header-cell">库位数量</view>
            <view class="header-cell last-col">本次出库</view>
          </view>
        </view>
        
        <view class="table-body">
          <view 
            class="table-row" 
            v-for="(item, index) in listData" 
            :key="index"
            :class="{ 'row-highlight': item.highlight }" 
            @click="toggleRow(index)"
          >
            <!-- 勾选框 -->
            <view class="table-cell check-col">
              <!-- 
                 修改点：
                 1. 添加 color="#000000"
                 2. class 改为 custom-checkbox
              -->
              <checkbox 
                :checked="item.checked" 
                color="#000000"
                class="custom-checkbox"
                style="transform:scale(0.7); pointer-events: none;" 
              />
            </view>
            <view class="table-cell">{{ item.ST_NAME }}</view>
            <view class="table-cell item-col">{{ item.SA_ItemCode }}</view>
            <view class="table-cell">{{ item.InQty }}</view>
            <view class="table-cell last-col">{{ item.OutQty }}</view>
          </view>
        </view>
      </view>
      
      <view style="height: 120rpx;"></view>
    </scroll-view>

    <!-- 底部按钮 -->
	<view class="bottom-bar">
		<button class="btn-back" @click="goBack">返回</button>
		
		<button 
			class="btn-confirm" 
			:class="{ 'btn-disabled': !hasPermission || selectedCount === 0 }"
			@click="handleSubmit"
			:disabled="!hasPermission || selectedCount === 0"
		>
			{{ hasPermission ? `确认出库 (${selectedCount})` : '没有修改权限' }}
		</button>
	</view>
  </view>
</template>

<script setup>
import { ref, computed } from 'vue'
import { onLoad } from '@dcloudio/uni-app'
import Navbar from "@/components/Navbar.vue" 
import Loading from "@/components/Loading.vue"

// ==========================================
// 🛠️ SN 配置器
// ==========================================
const IS_TEST_MODE = false   
const TEST_SN = "1a9960ec5662f946" 
// ==========================================
// 新增：权限控制变量，默认为 false (无权限)，接口返回 true 后开启
const hasPermission = ref(false)

const listData = ref([])
const loading = ref(true)
const currentDocNum = ref('')
const currentSN = ref('')

const selectedCount = computed(() => {
  return listData.value.filter(item => item.highlight).length
})

onLoad((options) => {
  if (options.docnum) {
    currentDocNum.value = decodeURIComponent(options.docnum)
  }
  if (IS_TEST_MODE) {
    currentSN.value = TEST_SN
  } else {
    currentSN.value = options.SN ? decodeURIComponent(options.SN) : ''
  }
  fetchDetailData()
})

const fetchDetailData = () => {
  if (!currentDocNum.value || !currentSN.value) {
    uni.showToast({ title: '参数缺失', icon: 'none' })
    loading.value = false
    return
  }

  loading.value = true
  
  const payload = {
    "SN": currentSN.value,
    "SQL": "SCDATA_YLT_20250930",
    "docnum": currentDocNum.value
  }

  uni.request({
    url: 'http://13.94.38.44:8080/OutStockConfirm/GetOutStockDetail',
    method: 'POST',
    data: payload,
    success: (res) => {
      let result = res.data
      if (typeof result === 'string') {
        try { result = JSON.parse(result.trim()) } catch (e) {}
      }

      if (res.statusCode === 200 && result && !result.isError) {
		  //获取权限状态
		hasPermission.value = result.falg === true;  
		console.log("是否有查看权限",hasPermission.value);
        const rawList = result.dt || []
        listData.value = rawList.map(item => ({
          ...item,
          checked: item.isfinish === 1, 
          highlight: false              
        }))
      } else {
        uni.showToast({ title: result.errMsg || '无数据', icon: 'none' })
      }
    },
    fail: () => {
      uni.showToast({ title: '网络请求失败', icon: 'none' })
    },
    complete: () => {
      loading.value = false
    }
  })
}

const toggleRow = (index) => {
  const item = listData.value[index]
  item.highlight = !item.highlight 
  item.checked = !item.checked     
}

const goBack = () => {
  uni.navigateBack()
}

const handleSubmit = () => {
  const selectedItems = listData.value.filter(item => item.highlight)
  if (selectedItems.length === 0) return

  uni.showModal({
    title: '确认出库',
    content: `已选中 ${selectedItems.length} 条数据，确认执行出库操作吗？`,
    showCancel: true,
    confirmText: '取消', 
    confirmColor: '#666666',
    cancelText: '确定',
    cancelColor: '#007aff', 
    success: (res) => {
      if (res.cancel) {
        executeOutboundRequest(selectedItems)
      }
    }
  })
}

const executeOutboundRequest = (selectedItems) => {
  const submitList = selectedItems.map(item => ({
    id: String(item.ID),
    type: item.checked ? "1" : "0"
  }))

  const payload = {
    "SN": currentSN.value,
    "SQL": "SCDATA_YLT_20250930",
    "list": submitList
  }

  uni.showLoading({ title: '提交中...', mask: true })
  
  uni.request({
    url: 'http://13.94.38.44:8080/OutStockConfirm/CheckFinish',
    method: 'POST',
    data: payload,
    success: (res) => {
      uni.hideLoading()
      let result = res.data
      if (typeof result === 'string') {
        try { result = JSON.parse(result.trim()) } catch (e) {}
      }

      if (res.statusCode === 200 && result && !result.isError) {
        uni.showToast({ title: '出库成功', icon: 'success' })
        setTimeout(() => { 
           uni.navigateBack() 
        }, 1500)
      } else {
        // 优先显示后端返回的具体错误信息
                const errorMsg = result ? (result.errMsg || result.Message || result.msg || '提交失败') : '服务器返回空数据'
                
                uni.showModal({
                  title: '提交失败',
                  content: errorMsg + '\n(状态码:' + res.statusCode + ')',
                  showCancel: false
                })
      }
    },
    fail: () => {
      uni.hideLoading()
      uni.showToast({ title: '网络失败', icon: 'none' })
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

.loading-container {
  display: flex; justify-content: center; align-items: center;
  height: 400rpx; color: #999;
}

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
  display: flex;
  border-bottom: 1rpx solid #eee;
  transition: background-color 0.1s;
}

.row-highlight {
  background-color: #DDECF6 !important; 
}

.table-cell {
  padding: 24rpx 2rpx;
  text-align: center; font-size: 26rpx;
  border-right: 1rpx solid #eee;
  display: flex; align-items: center; justify-content: center;
  word-break: break-all; overflow: hidden;
}

.check-col { width: 90rpx; flex: none; }
.header-cell:nth-child(2), .table-cell:nth-child(2) { flex: 1.5; }
.item-col { flex: 2.5; }
.header-cell:nth-child(4), .table-cell:nth-child(4), .last-col { flex: 1.2; }
.last-col { border-right: none; }

.bottom-bar {
  position: fixed; bottom: 0; left: 0; width: 100%; height: 120rpx;
  background-color: #fff; border-top: 1rpx solid #ccc;
  display: flex; align-items: center; justify-content: space-between;
  padding: 0 40rpx; box-sizing: border-box; z-index: 1000;
  box-shadow: 0 -2rpx 10rpx rgba(0,0,0,0.05);
}

.btn-back { width: 30%; background-color: #f1f1f1; color: #333; font-size: 30rpx; }
.btn-confirm { width: 65%; background-color: #007aff; color: white; font-size: 30rpx; transition: all 0.3s; }
.btn-disabled { background-color: #ccc !important; color: #fff; }
.btn-confirm:active { opacity: 0.8; }

/* 
   🔥🔥🔥 重点修改：checkbox 纯黑样式 🔥🔥🔥 
*/

/* 1. 未选中 */
uni-checkbox .uni-checkbox-input,
checkbox .wx-checkbox-input {
  background-color: #ffffff !important; 
  border-color: #999999 !important;     
  border-radius: 6rpx !important;       
}

/* 2. 选中：背景强制白，边框强制黑 */
uni-checkbox .uni-checkbox-input.uni-checkbox-input-checked,
checkbox .wx-checkbox-input.wx-checkbox-input-checked {
  background-color: #ffffff !important; 
  border-color: #000000 !important;     
}

/* 3. 选中后的钩：强制黑色 */
uni-checkbox .uni-checkbox-input.uni-checkbox-input-checked::before,
checkbox .wx-checkbox-input.wx-checkbox-input-checked::before {
  color: #000000 !important; 
  font-size: 34rpx;
  font-weight: bold;
}
</style>