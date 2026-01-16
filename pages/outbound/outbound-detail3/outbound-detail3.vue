<template>
  <!-- 保持原有模板不变 -->
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
        <button class="retry-btn" @click="fetchDetailData">重新加载</button>
      </view>

      <view v-else-if="!detailData" class="empty-container">
        <text class="empty-text">暂无数据</text>
      </view>

      <view v-else class="order-card">
        <view class="order-item">
          <text class="item-label">业务料号:</text>
          <input 
            class="item-input" 
            :value="detailData.SA_ItemCode || ''" 
            disabled
            placeholder="暂无数据"
          />
        </view>
        
        <view class="order-item">
          <text class="item-label">客户料号:</text>
          <input 
            class="item-input" 
            :value="detailData.CustomerItemCode || ''" 
            disabled
            placeholder="暂无数据"
          />
        </view>
        
        <view class="order-item">
          <text class="item-label">需出库数:</text>
          <input 
            class="item-input" 
            :value="detailData.OutTotalQty || ''" 
            disabled
            placeholder="暂无数据"
          />
        </view>

        <view class="stock-header">
          <view class="stock-cell">
            <text class="cell-text">库位</text>
          </view>
          <view class="stock-cell">
            <text class="cell-text">入库日期</text>
          </view>
          <view class="stock-cell">
            <text class="cell-text">库位数量</text>
          </view>
          <view class="stock-cell">
            <text class="cell-text">本次出库</text>
          </view>
        </view>

        <view v-for="(item, index) in detailData.dtNew" :key="index" class="stock-row">
          <view class="stock-cell">
            <text class="cell-text">{{ item['库位'] || '' }}</text>
          </view>
          <view class="stock-cell">
            <text class="cell-text">{{ item['InTime'] || '' }}</text>
          </view>
          <view class="stock-cell">
            <text class="cell-text">{{ item['库位数量'] || '' }}</text>
          </view>
          <view class="stock-cell">
            <input 
              class="true-input" 
              :value="outboundQtyArray[index] || ''"
              @input="(e) => handleOutQtyChange(e, index)"
              placeholder="出库数"
              type="number"
            />
          </view>
        </view>

        <view class="start-outbound-container">
          <button class="start-outbound-btn" :disabled="!hasEditPermission" @click="confirmStartOutbound">
            {{ hasEditPermission ? '确认出库' : '无出库权限' }}
          </button>
        </view>
      </view>
    </scroll-view>

   <!-- 确认弹窗 --> 
    <view v-if="showConfirm" class="confirm-modal">
      <view class="modal-content">
        <view class="modal-header">
          <text class="modal-title">确认出库</text>
        </view>
        <view class="modal-body">
          <text class="modal-text">您确定要进行出库操作吗？</text>
         <!-- 出库的汇总信息 --> 
          <text class="modal-text">业务料号：{{ detailData?.SA_ItemCode || 'N/A' }}</text>
          <text class="modal-text">本次总出库数：{{ totalOutboundQty || 0 }}</text>
        </view>
        <view class="modal-footer">
          <button class="cancel-btn" @click="cancelConfirm">取消</button>
          <button class="confirm-btn" :disabled="!hasEditPermission" @click="confirmSubmit">确定</button>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup>
import { ref, watch } from 'vue'
import Navbar from "@/components/Navbar.vue"
import { onLoad } from '@dcloudio/uni-app';
import Loading from "@/components/Loading.vue";

const detailData = ref(null)
const loading = ref(true)
const error = ref(null)
// 添加权限状态变量
const hasEditPermission = ref(false);
// 新增：控制自定义弹窗显示
const showConfirm = ref(false);

const apiParams = ref({
  SN: '',
  SQL: 'SCDATA_YLT_20250930',
  ItemCode: '',
  Do_Code: '',
  check_str: ''
});

const outboundQtyArray = ref([]);

onLoad((options) => {
  console.log('接收的参数:', options);

  const SN = options.SN;
  const ItemCode = options.ItemCode;
  const Do_Code = options.Do_Code;
  const check_str = options.check_str;
  
  if (!SN || !ItemCode || !Do_Code || !check_str) {
    error.value = '参数错误：缺少必要参数';
    loading.value = false;
    return;
  }
  
  apiParams.value.SN = SN;
  apiParams.value.ItemCode = ItemCode;
  apiParams.value.Do_Code = Do_Code;
  apiParams.value.check_str = check_str;
  
  fetchDetailData();
});

watch(() => detailData.value, (newData) => {
  if (newData && newData.dtNew) {
    outboundQtyArray.value = Array(newData.dtNew.length).fill('');
    console.log('数组初始化完成，长度:', newData.dtNew.length, '数组内容:', outboundQtyArray.value);
  }
}, { immediate: true });

// 新增：计算总出库数，用于弹窗显示
const totalOutboundQty = computed(() => {
  if (!outboundQtyArray.value || !detailData.value?.dtNew) return 0;
  return outboundQtyArray.value.reduce((sum, qty, index) => {
    const num = parseFloat(qty) || 0;
    // 只累加有效的正整数
    if (num > 0 && Number.isInteger(num)) {
      return sum + num;
    }
    return sum;
  }, 0);
});

const fetchDetailData = async () => {
  loading.value = true
  error.value = null
  // 重置权限状态
  hasEditPermission.value = false;

  try {
    const response = await uni.request({
      url: 'http://13.94.38.44:8080/YLT_CpCk/fill_Info',
      method: 'POST',
      data: apiParams.value,
      timeout: 10000
    })
    
    console.log("出库详情接口数据", response);

    if (response.statusCode === 200) {
      let result
      console.log("成功")
      if (typeof response.data === 'string') {
        result = JSON.parse(response.data)
        console.log("result", result);
      } else {
        result = response.data
      }

      if (result.isError === false) {
        detailData.value = result
        // 从响应中获取权限信息
        hasEditPermission.value = result.isEdit === true;
        console.log('detailData更新完成', detailData.value);
        console.log('设置出库权限:', hasEditPermission.value);
      } else {
        throw new Error(result.msg || '数据格式错误')
      }
    }  else {
      throw new Error(`HTTP错误: ${response.statusCode}`)
    }
  } catch (err) {
    console.error('获取出库详情数据失败:', err)
    error.value = err.message || '获取数据失败，请检查网络连接'
  } finally {
    loading.value = false
  }
}

// 修改：处理本次出库数的输入变化，只允许正整数
const handleOutQtyChange = (event, index) => {
  console.log('handleOutQtyChange事件:', event, '索引:', index);
  
  let value = '';
  if (event && event.detail && event.detail.value !== undefined) {
    value = event.detail.value;
  } else if (event && event.target && event.target.value !== undefined) {
    value = event.target.value;
  } else {
    value = event;
  }
  
  console.log('获取到的输入值:', value);
  
  // 过滤输入值，只保留数字
  let filteredValue = value.replace(/[^\d]/g, '');
  
  // 如果输入了多个0开头的数字，只保留一个0
  if (filteredValue.startsWith('0') && filteredValue.length > 1) {
    filteredValue = filteredValue.replace(/^0+/, '');
    if (filteredValue === '') filteredValue = '0';
  }
  
  console.log('过滤后的值:', filteredValue);
  
  // 更新数组中对应索引的值
  if (outboundQtyArray.value && index < outboundQtyArray.value.length) {
    outboundQtyArray.value[index] = filteredValue;
    console.log('更新数组索引', index, '值为:', filteredValue, '当前数组:', outboundQtyArray.value);
  }
}

// 修改：不再直接调用 uni.showModal，改为显示自定义弹窗
const confirmStartOutbound = () => {
  // 验证权限
  if (!hasEditPermission.value) {
    uni.showToast({
      title: '您没有出库权限',
      icon: 'none'
    });
    return; // 如果没有权限，直接返回，不执行后续操作
  }

  if (!validateOutboundData()) {
    return;
  }

  // 准备数据（可选：在弹窗中显示）
  const checkArray = detailData.value.dtNew.map((item, index) => {
    const outQty = outboundQtyArray.value[index] || '0';
    console.log("构建checkArray时的outQty", outQty);
    return {
      StockID: item['ID'],
      InQty: String(item['InQty'] || ''),
      InTime: item['InTime'] || '',
      WorkNum: item['WorkNum'] || '',
      OutQty: String(item['OutQty'] || ''),
      OutTime: item['OutTime'] || '',
      BCOutQty: outQty,
      stockqtye: String(item['库位数量'] || ''),
      txtSA_ItemCode: detailData.value.SA_ItemCode
    };
  });
  
  const outboundData = {
    ItemCode: detailData.value.SA_ItemCode,
    SN: apiParams.value.SN,
    check_str: apiParams.value.check_str,
    SQL: apiParams.value.SQL,
    Do_Code:apiParams.value.Do_Code,
    check_array: formatCheckArray(checkArray) // 使用新的格式化函数
  };
  
  console.log('确认出库数据准备就绪，等待用户确认:', outboundData);

  // 显示自定义确认弹窗
  showConfirm.value = true;
}

// 新增格式化函数，生成接口要求的字符串格式
const formatCheckArray = (array) => {
  if (!array || !Array.isArray(array)) return '[]';
  
  const formattedItems = array.map(item => {
    return `{` +
      `StockID:'${item.StockID}',` +
      `InQty:'${item.InQty}',` +
      `InTime:'${item.InTime}',` +
      `WorkNum:'${item.WorkNum}',` +
      `OutQty:'${item.OutQty}',` +
      `OutTime:'${item.OutTime}',` +
      `BCOutQty:'${item.BCOutQty}',` +
      `stockqtye:'${item.stockqtye}',` +
      `txtSA_ItemCode:'${item.txtSA_ItemCode}'` +
    `}`;
  });
  
  return `[${formattedItems.join(',')}]`;
};

// 修改验证逻辑
const validateOutboundData = () => {
  console.log('开始验证出库数据');
  console.log('detailData:', detailData.value);
  console.log('outboundQtyArray:', outboundQtyArray.value);
  
  if (!detailData.value || !detailData.value.dtNew || detailData.value.dtNew.length === 0) {
    uni.showToast({
      title: '没有可出库的数据',
      icon: 'none'
    });
    return false;
  }

  if (!outboundQtyArray.value || outboundQtyArray.value.length !== detailData.value.dtNew.length) {
    uni.showToast({
      title: '数据初始化错误',
      icon: 'none'
    });
    return false;
  }

  // 首先检查所有输入的有效性
  for (let i = 0; i < detailData.value.dtNew.length; i++) {
    const outQty = outboundQtyArray.value[i];
    console.log('验证索引', i, '值为:', outQty);
    
    if (outQty && outQty !== '' && outQty !== '0') {
      const num = parseFloat(outQty);
      // 检查是否为正整数
      if (isNaN(num) || num <= 0 || !Number.isInteger(num)) {
        uni.showToast({
          title: `第${i + 1}行出库数必须为正整数`,
          icon: 'none'
        });
        return false;
      }
      
      // 检查出库数是否超过库位数量
      const stockQty = parseFloat(detailData.value.dtNew[i]['库位数量']);
      console.log("库位数", stockQty);
      if (num > stockQty) {
        uni.showToast({
          title: `第${i + 1}行出库数不能超过库位数量`,
          icon: 'none'
        });
        return false;
      }
    }
  }

  // 然后检查是否有至少一个有效的出库数
  let hasValidOutbound = false;
  for (let i = 0; i < detailData.value.dtNew.length; i++) {
    const outQty = outboundQtyArray.value[i];
    if (outQty && outQty !== '' && outQty !== '0') {
      const num = parseFloat(outQty);
      if (num > 0 && Number.isInteger(num)) {
        hasValidOutbound = true;
        break;
      }
    }
  }

  if (!hasValidOutbound) {
    uni.showToast({
      title: '至少需要有一个库位输入出库数',
      icon: 'none'
    });
    return false;
  }

  console.log('验证通过');
  return true;
}

// 新增：取消确认弹窗
const cancelConfirm = () => {
  showConfirm.value = false;
};

// 新增：确认提交（从弹窗调用）
const confirmSubmit = () => {
  // 再次检查权限（防止弹窗期间权限改变，虽然可能性小）
  if (!hasEditPermission.value) {
    uni.showToast({
      title: '权限已更改，无法出库',
      icon: 'none'
    });
    showConfirm.value = false;
    return;
  }

  // 隐藏弹窗
  showConfirm.value = false;

  // 执行实际的出库操作
  startOutbound();
};

const startOutbound = async () => {
  // 权限检查已在 confirmSubmit 中进行
  // if (!hasEditPermission.value) { ... } // 可以移除或保留双重检查

  try {
    const checkArray = detailData.value.dtNew.map((item, index) => {
      const outQty = outboundQtyArray.value[index] || '0';
      console.log("构建数据时的outQty", outQty);
      return {
        StockID: item['ID'],
        InQty: String(item['InQty'] || ''),
        InTime: item['InTime'] || '',
        WorkNum: item['WorkNum'] || '',
        OutQty: String(item['OutQty'] || ''),
        OutTime: item['OutTime'] || '',
        BCOutQty: outQty,
        stockqtye: String(item['库位数量'] || ''),
        txtSA_ItemCode: detailData.value.SA_ItemCode
      };
    });

    const outboundData = {
      ItemCode: detailData.value.SA_ItemCode,
      SN: apiParams.value.SN,
      check_str: apiParams.value.check_str,
      SQL: apiParams.value.SQL,
      Do_Code: apiParams.value.Do_Code,
      check_array: formatCheckArray(checkArray) // 使用格式化函数
    };

    console.log('出库请求数据:', outboundData);

    uni.showLoading({
      title: '出库中...'
    });

    const response = await uni.request({
      url: 'http://13.94.38.44:8080/YLT_CpCk/insertData',
      method: 'POST',
      data: outboundData, // 修复：使用 data 而不是 outboundData
      timeout: 15000
    });

    uni.hideLoading();

    console.log('出库接口响应:', response);

    if (response.statusCode === 200) {
      let result;
      if (typeof response.data === 'string') {
        result = JSON.parse(response.data);
      } else {
        result = response.data;
      }

      if (result.isError === false) {
        uni.showToast({
          title: '出库成功',
          icon: 'success'
        });
        
        // 出库成功后返回上一页并传递刷新参数
        setTimeout(() => {
          uni.navigateBack({
            delta: 1
          });
        }, 1500); // 1.5秒延迟
      } else {
        throw new Error(result.msg || '出库失败');
      }
    } else {
      throw new Error(`HTTP错误: ${response.statusCode}`);
    }
  } catch (err) {
    console.error('出库失败:', err);
    uni.hideLoading();
    uni.showToast({
      title: err.message || '出库失败，请检查网络连接',
      icon: 'none'
    });
  }
}

// 为了在模板中使用 totalOutboundQty，需要从 vue 导入 computed
import { computed } from 'vue';

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
  margin-top: 100rpx;
  height: calc(100vh - 90rpx);
  padding: 0;
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

.order-card {
  background-color: white;
  border-radius: 20rpx;
  
   box-shadow: 0 4rpx 15rpx rgba(0, 0, 0, 0.05);
  width: 98%;
  margin: 0 auto;
}

.order-item {
  display: flex;
  margin-top: 10rpx; /* 增加每个字段之间的间距 */
  align-items: center;
  min-height: 80rpx; /* 设置最小高度 */
  padding: 0rpx 10rpx;
}

.order-item:last-child {
  margin-bottom: 0;
}

.item-label {
  font-size: 30rpx;
  color: #666;
  width: 200rpx; /* 固定标签宽度，确保对齐 */
  flex-shrink: 0; /* 防止标签被压缩 */
}

.item-input {
  flex: 1;
  font-size: 30rpx;
  color: #333;
  background-color: #f9f9f9; 
  border: 1rpx solid #e0e0e0;
  border-radius: 8rpx; 
  padding: 15rpx 10rpx;
  height: 50rpx;
  line-height: 50rpx;
  text-align: left;
  pointer-events: none;
  outline: none;
  -webkit-appearance: none;
  appearance: none;
}

.true-input {
  flex: 1;
  font-size: 30rpx;
  color: #333;
  background-color: #f9f9f9; 
  border: 1rpx solid #e0e0e0;
  border-radius: 8rpx; 
  padding: 15rpx 10rpx;
  height: 50rpx;
  line-height: 50rpx;
  text-align: left;
  outline: none;
  -webkit-appearance: none;
  appearance: none;
}

.item-input:disabled {
  background-color: #f9f9f9;
  color: #333;
  opacity: 1;
}

/* 库位信息标题样式 */
.stock-info-title {
  margin: 30rpx 0 20rpx 0;
  padding: 0 20rpx;
}

.title-text {
  font-size: 28rpx;
  font-weight: bold;
  color: #333;
}

/* 库位信息表格样式 */
.stock-header {
  display: flex;
  background-color: #f0f0f0;
  border-radius: 8rpx;
  padding: 15rpx 0;
  margin: 40rpx 10rpx 10rpx;
}

.stock-row {
  display: flex;
  border: 1rpx solid #e0e0e0;
  border-radius: 8rpx;
  margin: 0 10rpx 10rpx;
  padding: 15rpx 0;
  background-color: #fff;
}

.stock-cell {
  padding: 0 10rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  /* 统一设置列宽比例，使表格更整齐 */
  flex: 1;
  min-width: 0; /* 防止内容溢出时缩小 */
}

/* 修正列宽比例，让表格更美观 */
.stock-cell:nth-child(1) { /* 库位 */
  flex: 1;
  
}

.stock-cell:nth-child(2) { /* 入库日期 */
  flex: 1.3;
  
}

.stock-cell:nth-child(3) { /* 库位数量 */
  flex: 1;
 
}

.stock-cell:nth-child(4) { /* 本次出库 */
  flex: 1;
  
}

.cell-text {
  font-size: 30rpx;
  color: #333;
  text-align: center;
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

/* 修正"本次出库"列的输入框样式 */
.stock-cell:nth-child(4) .true-input {
  width: 100%;
  padding: 10rpx;
  height: 40rpx;
  line-height: 40rpx;
  text-align: center;
  font-size: 28rpx;
}

/* 开始出库按钮容器 */
.start-outbound-container {
  margin-top: 30rpx;
  padding: 20rpx;
}

.start-outbound-btn {
  width: 100%;
  background-color: #007aff;
  color: white;
  border: none;
  border-radius: 10rpx;
  padding: 25rpx;
  font-size: 32rpx;
  font-weight: bold;
}

/* 响应式适配小屏幕 */
@media screen and (max-width: 768px) {
  .content-container {
    
  }
  
  .order-item {
    margin-bottom: 5rpx;
  }
  
  .item-label {
    font-size: 30rpx;
    width: 140rpx;
  }
  
  .item-input, .true-input {
    height: 70rpx;
    font-size: 30rpx;
    padding: 0 15rpx;
  }

  .stock-cell {
    padding: 0 5rpx;
  }

  .cell-text {
    font-size: 28rpx;
  }
  
  /* 小屏幕下调整列宽 */
  .stock-cell:nth-child(1) {
    flex: 1;
    min-width: 100rpx;
  }
  
  .stock-cell:nth-child(2) {
    flex: 1.5;
    min-width: 150rpx;
  }
  
  .stock-cell:nth-child(3) {
    flex: 1;
    min-width: 100rpx;
  }
  
  .stock-cell:nth-child(4) {
    flex: 1;
    min-width: 120rpx;
  }
}

/* 新增：自定义确认弹窗样式 */
.confirm-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  width: 80%;
  background-color: white;
  border-radius: 20rpx;
  padding: 30rpx;
  box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.2);
}

.modal-header {
  margin-bottom: 20rpx;
}

.modal-title {
  font-size: 32rpx;
  font-weight: bold;
  color: #333;
  text-align: center;
}

.modal-body {
  margin-bottom: 30rpx;
}

.modal-text {
  display: block;
  font-size: 28rpx;
  color: #666;
  margin-bottom: 10rpx;
  word-break: break-all;
}

.modal-footer {
  display: flex;
  justify-content: space-between;
}

.cancel-btn {
  flex: 1;
  height: 70rpx;
  background-color: #f0f0f0;
  color: #333;
  font-size: 28rpx;
  border-radius: 10rpx;
  margin-right: 15rpx;
  border: none;
}

.confirm-btn {
  flex: 1;
  height: 70rpx;
  background-color: #007aff;
  color: white;
  font-size: 28rpx;
  border-radius: 10rpx;
  margin-left: 15rpx;
  border: none;
  /* 添加禁用状态样式 */
  opacity: 0.6;
  cursor: not-allowed;
}

.confirm-btn:not([disabled]) {
  opacity: 1;
  cursor: pointer;
}
</style>