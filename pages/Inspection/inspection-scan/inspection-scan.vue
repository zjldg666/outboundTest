<template>
  <view class="container">
    <view class="navbar-fixed">
      <Navbar title="检验扫码"></Navbar>
    </view>
    
    <scroll-view class="content-container" scroll-y="true">
      <view class="order-container">
        <view class="empty-or-error" v-if="errorMsg">
          <text class="error-text">{{ errorMsg }}</text>
          <button class="retry-btn" @click="fetchOrderInfo">重新获取</button>
        </view>
        
        <view v-else-if="loading" class="loading-container">
          <Loading :show="loading"></Loading>
        </view>
        
        <view class="empty-or-error" v-else-if="!loading && Object.keys(commonOrderInfo).length === 0">
          <text class="empty-text">暂无该单据的检验信息</text>
        </view>

        <block v-else>
          <view class="order-card">
            <view class="order-item">
              <text class="item-label">订单号：</text>
              <input class="item-input font-bold" :value="commonOrderInfo['订单号']" disabled />
            </view>
            <view class="order-item">
              <text class="item-label">供应商：</text>
              <input class="item-input" :value="commonOrderInfo['供应商']" disabled />
            </view>
            <view class="order-item">
              <text class="item-label">业务料号：</text>
              <input class="item-input" :value="commonOrderInfo['业务料号']" disabled />
            </view>
            <view class="order-item">
              <text class="item-label">客户料号：</text>
              <input class="item-input" :value="commonOrderInfo['客户料号']" disabled />
            </view>
            <view class="order-item">
              <text class="item-label">检验单号：</text>
              <input class="item-input" :value="commonOrderInfo['检验单号']" disabled />
            </view>
            
            <view class="order-item">
              <text class="item-label">待检验数：</text>
              <input 
                class="item-input font-bold text-primary" 
                :value="currentPendingQty || '请选择批次'"
                disabled
              />
            </view>
          </view>

          <view class="input-section" v-if="batchList.length > 0">
            <text class="section-title">待检批次 ({{ batchList.length }})</text>
            <view class="batch-list">
              <view 
                v-for="(item, index) in batchList" 
                :key="index"
                class="batch-item"
                :class="{ 'active-batch': selectedBatchIndex === index }"
                @click="selectBatch(index)"
              >
                <text class="batch-qty">{{ item['本次收货数'] }}</text>
                <text class="batch-id" v-if="item['检验单号']">
                  {{ item['检验单号'].slice(-4) }}
                </text>
              </view>
            </view>
          </view>

          <view class="input-section">
            <view class="input-item">
              <text class="item-label">本次检验数：</text>
              <input 
                class="true-input" 
                v-model="formData.CheckQty"
                placeholder="请输入检验数量"
                type="number"
                @input="onQtyInput"
              />
            </view>
            
            <view class="input-item">
              <text class="item-label">检验备注：</text>
              <input 
                class="true-input" 
                v-model="formData.Remark"
                placeholder="请输入备注"
              />
            </view>

            <view class="submit-container">
              <button class="submit-btn" :disabled="!hasEditPermission">
                {{ hasEditPermission ? '确定检验' : '无检验权限' }}
              </button>
            </view>
          </view>
        </block>
        
        <view style="height: 40rpx;"></view>
      </view>
    </scroll-view>
  </view>
</template>

<script setup>
import { ref, reactive } from 'vue';
import Navbar from "@/components/Navbar.vue";
import { onLoad } from '@dcloudio/uni-app';
import Loading from "@/components/Loading.vue"

// 响应式状态
const loading = ref(false);
const errorMsg = ref('');

const commonOrderInfo = ref({}); // 主卡片显示的信息
const batchList = ref([]);       // 批次列表
const selectedBatchIndex = ref(-1); // 当前选中的批次索引
const currentPendingQty = ref('');  // 当前显示的待检验数
const hasEditPermission = ref(false); // 是否有编辑权限

// 接口参数
const apiParams = ref({
  DocNum: '',
  SN: '',
  SQL: 'SCDATA_YLT_20250930'
});

// 表单数据
const formData = reactive({
  CheckQty: '',
  Remark: ''
});

onLoad((options) => {
  console.log('检验扫码接收参数:', options);
  
  const DocNum = options.DocNum;
  const SN = options.SN;
  
  if (!DocNum || !SN) {
    errorMsg.value = '参数错误：缺少单号或设备码';
    return;
  }

  apiParams.value.DocNum = DocNum;
  apiParams.value.SN = SN;
  
  fetchOrderInfo();
});

// 获取检验信息
const fetchOrderInfo = () => {
  loading.value = true;
  errorMsg.value = '';
  
  // 重置状态
  commonOrderInfo.value = {};
  batchList.value = [];
  selectedBatchIndex.value = -1;
  currentPendingQty.value = '';
  hasEditPermission.value = false;

  uni.request({
    url: 'http://13.94.38.44:8080/YLT_SalIQC_Check/fill_Info',
    method: 'POST',
    header: { 'Content-Type': 'application/json' },
    data: apiParams.value,
    success: (res) => {
      console.log('检验接口返回:', res.data);
      
      let result = null;
      try {
        result = typeof res.data === 'string' ? JSON.parse(res.data) : res.data;
      } catch (e) {
        errorMsg.value = '数据解析失败';
        return;
      }

      if (result && result.isError === false) {
        // 设置权限
        hasEditPermission.value = result.isEdit === true;
        
        // 处理列表数据
        const dt = Array.isArray(result.dt) ? result.dt : [];
        
        if (dt.length > 0) {
          batchList.value = dt;
          
          // 默认不选中，提示用户选择，或者也可以默认选中第一个
          // 这里为了引导用户确认，先不选中，只显示主单号信息
          // 如果想默认选中第一个，可以调用 selectBatch(0)
          
          // 初始化显示基础信息（取第一条数据的通用字段）
          commonOrderInfo.value = {
            '订单号': dt[0]['订单号'],
            '供应商': dt[0]['供应商'],
            '业务料号': dt[0]['业务料号'],
            '客户料号': dt[0]['客户料号'],
            '检验单号': '请选择下方批次' 
          };
          
        } else {
          errorMsg.value = '未找到待检验明细';
        }
      } else {
        errorMsg.value = result ? result.msg : '接口请求失败';
      }
    },
    fail: (err) => {
      errorMsg.value = '网络请求失败';
      console.error(err);
    },
    complete: () => {
      loading.value = false;
    }
  });
};

// 批次选择逻辑
const selectBatch = (index) => {
  selectedBatchIndex.value = index;
  const item = batchList.value[index];
  
  // 更新主卡片显示的信息（切换到对应批次的详情）
  commonOrderInfo.value = {
    '订单号': item['订单号'],
    '供应商': item['供应商'],
    '业务料号': item['业务料号'],
    '客户料号': item['客户料号'],
    '检验单号': item['检验单号']
  };
  
  // 更新待检验数量（取字段：本次收货数）
  currentPendingQty.value = item['本次收货数'];
  
  // 自动填入本次检验数（可选体验优化）
  // formData.CheckQty = item['本次收货数']; 
};

// 输入过滤：只允许正整数
const onQtyInput = (e) => {
  let val = e.detail.value.replace(/[^\d]/g, '');
  if (val.startsWith('0') && val.length > 1) val = val.replace(/^0+/, '');
  formData.CheckQty = val;
};

</script>

<style scoped>
/* 复用之前的通用样式 */
.container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background-color: #f5f5f5;
  overflow: hidden;
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

.order-container {
  padding: 30rpx;
  display: flex;
  flex-direction: column;
  gap: 20rpx;
}

.empty-or-error {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40rpx;
  gap: 30rpx;
}

.error-text { color: #ff4d4f; font-size: 28rpx; }
.empty-text { color: #999; font-size: 28rpx; }
.retry-btn { background-color: #029999; color: white; font-size: 26rpx; }

.loading-container {
  flex: 1; display: flex; align-items: center; justify-content: center; margin-top: 90rpx;
}

/* 卡片样式 */
.order-card {
  background-color: white;
  border-radius: 20rpx;
  padding: 20rpx;
  box-shadow: 0 4rpx 15rpx rgba(0, 0, 0, 0.05);
}

.order-item {
  display: flex;
  margin-bottom: 10rpx;
  align-items: center;
  min-height: 80rpx;
}

.item-label {
  font-size: 26rpx;
  color: #666;
  width: 160rpx;
  flex-shrink: 0;
}

.item-input {
  flex: 1;
  font-size: 26rpx;
  color: #333;
  background-color: #f9f9f9;
  border: 1rpx solid #e0e0e0;
  border-radius: 8rpx;
  padding: 15rpx 20rpx;
  height: 50rpx;
  line-height: 50rpx;
}

.font-bold { font-weight: bold; color: #029999; }
.text-primary { color: #0066cc; }

/* 批次选择区 */
.input-section {
  background-color: white;
  border-radius: 20rpx;
  padding: 20rpx;
  box-shadow: 0 4rpx 15rpx rgba(0, 0, 0, 0.05);
}

.section-title {
  font-size: 30rpx;
  font-weight: bold;
  color: #333;
  margin-bottom: 20rpx;
  display: block;
}

.batch-list {
  display: flex;
  flex-wrap: wrap;
  gap: 20rpx;
}

.batch-item {
  width: calc((100% - 40rpx) / 3);
  background-color: white;
  border: 2rpx solid #e0e0e0;
  border-radius: 15rpx;
  padding: 20rpx 5rpx;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}

.active-batch {
  border-color: #007aff;
  background-color: #f0f8ff;
  box-shadow: 0 4rpx 12rpx rgba(0, 122, 255, 0.2);
}

.batch-qty { font-size: 34rpx; font-weight: bold; color: #333; }
.active-batch .batch-qty { color: #007aff; }

.batch-id { font-size: 22rpx; color: #999; margin-top: 6rpx; }

/* 输入区 */
.input-item {
  display: flex;
  margin-bottom: 20rpx;
  align-items: center;
}

.true-input {
  flex: 1;
  font-size: 26rpx;
  color: #333;
  background-color: #f9f9f9;
  border: 1rpx solid #e0e0e0;
  border-radius: 8rpx;
  padding: 15rpx 20rpx;
  height: 60rpx;
  line-height: 60rpx;
}

.submit-container {
  margin-top: 30rpx;
}

.submit-btn {
  width: 100%;
  height: 80rpx;
  background-color: #007aff;
  color: white;
  font-size: 32rpx;
  font-weight: bold;
  border-radius: 15rpx;
  border: none;
}
.submit-btn:disabled {
  background-color: #ccc;
  opacity: 0.7;
}
</style>