<template>
  <view class="container">
    <!-- 固定在顶部的导航栏 -->
    <view class="navbar-fixed">
      <Navbar></Navbar>
    </view>
    
    <!-- 页面内容区域，需要给上边距避开导航栏 -->
    <scroll-view class="content-container" scroll-y="true">
      <!-- 数据加载状态 -->
      <view v-if="loading" class="loading-container">
        <Loading :show="loading"></Loading>
      </view>

      <!-- 错误状态 -->
      <view v-else-if="error" class="error-container">
        <text class="error-text">{{ error }}</text>
        <button class="retry-btn" @click="fetchDetailData">重新加载</button>
      </view>

      <!-- 数据为空状态 -->
      <view v-else-if="detailData.length === 0" class="empty-container">
        <text class="empty-text">暂无数据</text>
      </view>

      <!-- 数据表格 -->
      <view v-else class="table-container">
        <!-- 固定表头 -->
        <view class="table-header">
          <view class="header-row">
            <view class="header-cell">业务料号</view>
            <view class="header-cell">需求出库数</view>
            <view class="header-cell">当前库存数</view>
          </view>
        </view>
        
        <!-- 可滚动的内容区域 -->
        <scroll-view class="table-body" scroll-y="true">
          <view class="clickable-row" 
                v-for="(item, index) in detailData" 
                :key="index"
                @click="navigateToDetail(item)">
            <view class="table-cell">{{ item['业务料号'] }}</view>
            <view class="table-cell">{{ item['需求出库数'] }}</view>
            <view class="table-cell">{{ item['当前库存数'] }}</view>
          </view>
        </scroll-view>
      </view>
    </scroll-view>
  </view>
</template>

<script setup>
import { ref } from 'vue'
import Navbar from "@/components/Navbar.vue"
import { onLoad,onShow } from '@dcloudio/uni-app';
import Loading from "@/components/Loading.vue"
// 响应式数据
const detailData = ref([])
const loading = ref(true)
const error = ref(null)
const apiParams = ref({
  SN: '',
  SQL: 'SCDATA_YLT_20250930',
  Do_Code: ''
});

onShow(()=>{
	fetchDetailData(); // 重新获取数据
})
onLoad((options) => {
  console.log('接收的参数:', options);

  const SN = options.SN;
  const Do_Code = options.Do_Code;
  
  if (!SN || !Do_Code) {
    error.value = '参数错误：缺少设备码或送货单号';
    loading.value = false;
    return;
  }
  
  apiParams.value.SN = SN;
  apiParams.value.Do_Code = Do_Code;
  
  fetchDetailData();
});

// 获取详情数据 
const fetchDetailData = async () => {
  loading.value = true
  error.value = null

  try {
    const response = await uni.request({
      url: 'http://13.94.38.44:8080/YLT_CpCk/dt_ShowDO_Code',
      method: 'POST',
      data: apiParams.value,
      timeout: 10000 // 10秒超时
    })
    
    console.log("出库详情接口数据", response);

    if (response.statusCode === 200) {
      // 接口返回的是字符串格式的JSON，需要先解析
      let result
      console.log("成功")
      if (typeof response.data === 'string') {
        result = JSON.parse(response.data)
        console.log("result", result);
      } else {
        result = response.data
      }

      // 检查是否包含错误
      if (result.isError === false && result.dt && Array.isArray(result.dt)) {
        detailData.value = result.dt
      } else {
        throw new Error(result.msg || '数据格式错误')
      }
    } else {
      throw new Error(`HTTP错误: ${response.statusCode}`)
    }
  } catch (err) {
    console.error('获取出库详情数据失败:', err)
    error.value = err.message || '获取数据失败，请检查网络连接'
  } finally {
    loading.value = false
  }
}

// 跳转到详情页面
const navigateToDetail = (item) => {
  const SN = apiParams.value.SN;
  const ItemCode = item['业务料号'];
  const Do_Code = item['送货单号'];
  const requiredQty = item['需求出库数'];
  const currentStock = item['当前库存数'];
  
  // 生成 check_str
  const check_str = `${ItemCode}${requiredQty}${currentStock}`;
  
  if (!SN) {
    uni.showToast({
      title: '设备码参数错误',
      icon: 'none'
    });
    return;
  }
  
  if(currentStock<=0){
	  uni.showToast({
	  	title:'当前库存数为0，无法出库',
		icon:'none'
	  });
	  return;
  }else{
	  uni.navigateTo({
	    url: `/pages/outbound/outbound-detail3/outbound-detail3?ItemCode=${encodeURIComponent(ItemCode)}&Do_Code=${encodeURIComponent(Do_Code)}&SN=${encodeURIComponent(SN)}&check_str=${encodeURIComponent(check_str)}`
	  });
  }
  

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

/* 数据为空的容器样式 */
.empty-container {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 90rpx;
}

.empty-text {
  font-size: 28rpx;
  color: #999;
  text-align: center;
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
  flex: 1;
  padding: 16rpx 1rpx;
  text-align: center;
  font-weight: bold;
  font-size: 34rpx;
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

/* 可点击的行样式 - 改进后的样式 */
.clickable-row {
  display: flex;
  background-color: #ffffff;
  margin: 10rpx 20rpx;
  border-radius: 16rpx;
  border: 2rpx solid #e8e8e8;
  transition: all 0.2s ease;
  box-shadow: 0 2rpx 8rpx rgba(0,0,0,0.08);
  cursor: pointer;
}

.clickable-row:active {
  background-color: #e3f2fd;
  border-color: #2196f3;
  transform: scale(0.98);
  box-shadow: 0 4rpx 12rpx rgba(33, 150, 243, 0.2);
}

.clickable-row:hover {
  background-color: #f0f8ff;
  border-color: #bbdefb;
  box-shadow: 0 4rpx 12rpx rgba(33, 150, 243, 0.15);
}

/* 交替行颜色（可选） */
.clickable-row:nth-child(even) {
  background-color: #fafafa;
}

.clickable-row:nth-child(even):hover {
  background-color: #f0f8ff;
}

.clickable-row:nth-child(even):active {
  background-color: #e3f2fd;
}

.table-row {
  display: flex;
  border-bottom: 1rpx solid #eee;
}

.table-row:last-child {
  border-bottom: none;
}

.table-cell {
  flex: 1;
  padding: 16rpx 1rpx;
  text-align: center;
  font-size: 30rpx;
  border-right: 1rpx solid #eee;
  word-break: break-all;
  overflow-wrap: break-word;
  white-space: nowrap;
}

.table-cell:last-child {
  border-right: none;
}

/* 响应式适配小屏幕 */
@media screen and (max-width: 768px) {
  .header-cell,
  .table-cell {
    padding: 16rpx 1rpx;
    font-size: 34rpx;
  }
  
  .clickable-row {
    margin: 14rpx 12rpx;
    padding: 0;
  }
}
</style>