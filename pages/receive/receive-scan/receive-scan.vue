<!-- 收货扫码 -->
<template>
  <view class="container">
    <view class="navbar-fixed">
      <navbar></navbar>
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
          <text class="empty-text">暂无该订单的收货信息</text>
        </view>

        <block v-else>
          <view class="order-card">
            <view class="order-item" v-for="(value, key) in commonOrderInfo" :key="key">
              <text class="item-label">{{ key }}：</text>
              <input 
                class="item-input" 
                :class="{ 'font-bold': key === '订单号' }"
                :value="formatValue(value, key)"
                :disabled="true"
                :readonly="true"
                type="text"
              />
            </view>
            <view class="order-item">
              <text class="item-label">待收货数：</text>
              <input 
                class="item-input font-bold text-primary" 
                :value="currentPendingQty || '请选择下方批次'"
                :disabled="true"
                :readonly="true"
                type="text"
              />
            </view>
          </view>

          <view class="input-section" v-if="batchList.length > 0">
            <text class="section-title">请选择通知单批次：(按照待收货数分批次)</text>
            <view class="batch-list">
              <view 
                v-for="(item, index) in batchList" 
                :key="index"
                class="batch-item"
                :class="{ 'active-batch': selectedBatchIndex === index }"
                @click="selectBatch(index)"
              >
                <text class="batch-qty">{{ item['待收货数'] }}</text>
              </view>
            </view>
          </view>

          <view class="input-section">
            <view class="input-item">
              <text class="item-label">本次收货数：</text>
              <input 
                class="true-input" 
                v-model="formData.InQty"
                placeholder="请输入本次收货数"
                type="number"
                @input="onInQtyInput"
              />
            </view>
            
            <view class="input-item">
              <text class="item-label">收货备注：</text>
              <input 
                class="true-input" 
                v-model="formData.ReceiveRemark"
                placeholder="请输入收货备注"
              />
            </view>

            <view class="input-item">
              <text class="item-label">上传图片：</text>
              <view class="image-upload-container">
                <view class="image-preview" v-for="(image, index) in formData.FileList" :key="index">
                  <image :src="image" mode="aspectFill" class="preview-image" @click="previewImage(image)"></image>
                  <view class="delete-btn" @click="deleteImage(index)">×</view>
                </view>
                <view class="upload-btn" v-if="formData.FileList.length < 5" @click="chooseImage">
                  <text class="upload-text">+</text>
                </view>
              </view>
            </view>

            <view class="submit-container">
              <button class="submit-btn" :disabled="!hasEditPermission" @click="showConfirmDialog">
                {{ hasEditPermission ? '确定收货' : '无收货权限' }}
              </button>
            </view>
          </view>
        </block>
        
        <view style="height: 40rpx;"></view>
      </view>
    </scroll-view>
    
    <view v-if="showConfirm" class="confirm-modal">
      <view class="modal-content">
        <view class="modal-header">
          <text class="modal-title">确认收货</text>
        </view>
        <view class="modal-body">
          <text class="modal-text">您确定要进行收货操作吗？</text>
          <text class="modal-text">本次收货数：{{ formData.InQty }}</text>
          <text class="modal-text" v-if="formData.ReceiveRemark">收货备注：{{ formData.ReceiveRemark }}</text>
          <text class="modal-text" v-if="formData.FileList.length > 0">已上传图片：{{ formData.FileList.length }}张</text>
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
import { ref, reactive } from 'vue';
import Navbar from "@/components/Navbar.vue";
import { onLoad } from '@dcloudio/uni-app';
import Loading from "@/components/Loading.vue"

// --- 1. 图片处理函数 (完全保持您原有的代码，一行未动) ---
/**
 * 图片压缩并转换为base64格式
 * @param {string} url 图片地址 可网络地址、本地相对路径
 * @param {string} type base64图片类型 默认png
 * @param {number} quality 压缩质量 1-100，默认80
 */
function compressImageToBase64(url, type = 'png', quality = 50) {
  return new Promise((resolve, reject) => {
    if (url.startsWith('http')) {
      // 网络地址 使用request方式
      uni.request({
        url: url,
        method: 'GET',
        responseType: 'arraybuffer',
        success: (res) => {
          // 将arraybuffer转换为base64
          const base64 = `image/${type};base64,${uni.arrayBufferToBase64(res.data)}`;
          resolve(base64);
        },
        fail: (err) => {
          reject(new Error(`Request failed: ${err}`));
        }
      });
    } else {
      // 本地图片路径，先压缩再转换为base64
      // #ifdef APP-PLUS
      // APP环境使用plus.io处理
      plus.io.resolveLocalFileSystemURL(url, (entry) => {
        entry.file((file) => {
          // 压缩图片
          uni.compressImage({
            src: url,
            quality: quality, // 压缩质量
            width: '800px', // 限制最大宽度
            success: (compressRes) => {
              const compressedPath = compressRes.tempFilePath;
              // 读取压缩后的文件
              plus.io.resolveLocalFileSystemURL(compressedPath, (compressedEntry) => {
                compressedEntry.file((compressedFile) => {
                  const fileReader = new plus.io.FileReader();
                  fileReader.onload = (r) => {
                    resolve(r.target.result);
                  };
                  fileReader.onerror = (error) => {
                    reject(new Error(`File read failed: ${error.message}`));
                  };
                  fileReader.readAsDataURL(compressedFile);
                }, (error) => {
                  reject(new Error(`Compressed file resolution failed: ${error.message}`));
                });
              }, (error) => {
                reject(new Error(`Compressed file system resolution failed: ${error.message}`));
              });
            },
            fail: (compressErr) => {
              console.warn('压缩失败，使用原始图片:', compressErr);
              // 如果压缩失败，使用原始文件
              const fileReader = new plus.io.FileReader();
              fileReader.onload = (r) => {
                resolve(r.target.result);
              };
              fileReader.onerror = (error) => {
                reject(new Error(`Original file read failed: ${error.message}`));
              };
              fileReader.readAsDataURL(file);
            }
          });
        }, (error) => {
          reject(new Error(`Original file resolution failed: ${error.message}`));
        });
      }, (error) => {
        reject(new Error(`File system resolution failed: ${error.message}`));
      });
      // #endif

      // #ifndef APP-PLUS
      // 非APP环境，微信小程序等
      uni.compressImage({
        src: url,
        quality: quality, // 压缩质量
        width: '800px', // 限制最大宽度
        success: (compressRes) => {
          const compressedPath = compressRes.tempFilePath;
          // 将压缩后的图片转换为base64
          const fs = uni.getFileSystemManager();
          fs.readFile({
            filePath: compressedPath,
            encoding: 'base64',
            success: (readRes) => {
              const base64 = `image/${type};base64,${readRes.data}`;
              resolve(base64);
            },
            fail: (readErr) => {
              console.warn('读取压缩图片失败，使用原始路径:', readErr);
              // 如果读取失败，直接返回原始路径
              resolve(url);
            }
          });
        },
        fail: (compressErr) => {
          console.warn('压缩失败，使用原始图片:', compressErr);
          // 如果压缩失败，尝试直接读取原始图片
          const fs = uni.getFileSystemManager();
          fs.readFile({
            filePath: url,
            encoding: 'base64',
            success: (readRes) => {
              const base64 = `image/${type};base64,${readRes.data}`;
              resolve(base64);
            },
            fail: (readErr) => {
              console.warn('读取原始图片失败:', readErr);
              // 如果都失败，直接返回原始路径
              resolve(url);
            }
          });
        }
      });
      // #endif
    }
  });
}

// --- 2. 变量定义 ---
const loading = ref(false);
const errorMsg = ref('');
// 核心变更：不再用 orderList，改为 commonOrderInfo(主信息) 和 batchList(批次列表)
const commonOrderInfo = ref({}); 
const batchList = ref([]);       
const selectedBatchIndex = ref(-1); // 选中的批次索引
const currentPendingQty = ref('');  // 当前显示的待收货数

const showConfirm = ref(false); 
const hasEditPermission = ref(false); 
const apiParams = ref({
  DocNum: '',
  SN: '',
  SQL: 'SCDATA_YLT_20250930'
});

const formData = reactive({
  InQty: '',
  ReceiveRemark: '',
  FileList: [] 
});

onLoad((options) => {
  console.log('接收的参数:', options);
  const DocNum = options.DocNum;
  const SN = options.SN;
  
  if (!DocNum || !SN) {
    errorMsg.value = '参数错误：缺少订单号或设备码';
    return;
  }

  apiParams.value.DocNum = DocNum;
  apiParams.value.SN = SN;
  fetchOrderInfo();
});

// 格式化字段值
const formatValue = (value, key) => {
  if (['订单数量', '已收数量', '已收可入数', '已退货数', '待收货数'].includes(key)) {
    return value || 0;
  }
  return value || '无';
};

// 选择批次
const selectBatch = (index) => {
  selectedBatchIndex.value = index;
  const item = batchList.value[index];
  
  // 1. 更新待收货数
  currentPendingQty.value = item['待收货数'];
  
  // 2. 更新主卡片显示的详细信息（供应商、预交日期等可能不一样）
  const displayInfo = { ...item };
  
  // 移除掉不需要在主列表重复显示的字段
  delete displayInfo['待收货数']; // 下方单独显示了
  delete displayInfo['通知单号']; // 按钮上显示了
  // 如果你想在主卡片也保留预交日期，就不要delete它；如果按钮上够了，可以delete
  // 这里建议保留在主卡片，因为它是重要信息
  
  commonOrderInfo.value = displayInfo;
};

// 输入验证
const onInQtyInput = (e) => {
  let value = e.detail.value;
  value = value.replace(/[^\d]/g, '');
  if (value.startsWith('0') && value.length > 1) {
    value = value.replace(/^0+/, '');
  }
  formData.InQty = value;
};

// --- 3. 获取数据 (逻辑修改：数据拆分) ---
const fetchOrderInfo = () => {
  loading.value = true;
  errorMsg.value = '';
  
  // 重置数据
  commonOrderInfo.value = {};
  batchList.value = [];
  selectedBatchIndex.value = -1;
  currentPendingQty.value = '';
  hasEditPermission.value = false;

  uni.request({
    url: 'http://13.94.38.44:8080/YLT_SalIQC/fill_Info',
    method: 'POST',
    header: {
      'Content-Type': 'application/json',
      'Accept': 'application/json'
    },
    data: apiParams.value,
    success: (res) => {
      console.log('原始响应体类型:', typeof res.data);
      let response = null;
      try {
        response = typeof res.data === 'string' ? JSON.parse(res.data) : res.data;
        console.log('解析后响应:', response);
      } catch (parseErr) {
        errorMsg.value = '接口响应格式错误：' + parseErr.message;
        console.error('JSON解析失败:', parseErr);
        // 保持您原有的错误处理风格
        uni.navigateBack({
          delta: 1,
          success: () => {
            setTimeout(() => {
              uni.showToast({ title: '获取订单信息失败：' + parseErr.message, icon: 'none', duration: 3000 });
            }, 500);
          }
        });
        return;
      }

      if (response && (response.isError === false || !response.isError)) {
        hasEditPermission.value = response.isEdit === true;
        console.log('设置收货权限:', hasEditPermission.value);

        const rawList = Array.isArray(response.dt) ? response.dt : [];
        
        if (rawList.length > 0) {
          // --- 修改开始：针对“供应商可能不同”的逻辑调整 ---
          
          // 1. 初始状态：只显示“订单号”
          // 既然供应商等其他信息都可能变，初始就不显示，避免误导用户。
          // 用户点击下方批次按钮后，selectBatch 会更新 commonOrderInfo 显示详情。
          commonOrderInfo.value = { 
            '订单号': rawList[0]['订单号'] || apiParams.value.DocNum 
          };
          
          // 2. 构建批次列表 (完全保留原始数据)
          // 这里保存所有字段，因为selectBatch时需要用到它们来更新顶部卡片
          batchList.value = rawList.map(item => ({
            ...item,
            '待收货数': item['待收货数'],
            '通知单号': item['通知单号'],
            '预交日期': item['预交日期']
          }));
          
          // --- 修改结束 ---
          
        } else {
          const emptyMsg = '接口返回为空，请确认订单号是否正确';
          errorMsg.value = emptyMsg;
          uni.navigateBack({
            delta: 1,
            success: () => {
              setTimeout(() => {
                uni.showToast({ title: emptyMsg, icon: 'none', duration: 3000 });
              }, 500);
            }
          });
        }
      } else {
        const errorMsgText = '接口返回错误：' + (response?.msg || '未知错误');
        errorMsg.value = errorMsgText;
        
        if (response?.msg === '无相关权限') {
          uni.navigateBack({
            delta: 1,
            success: () => {
              setTimeout(() => {
                uni.showModal({ title: '权限提示', content: '无相关权限，请点击下方获取设备码，并发给管理员', showCancel: false, confirmText: '确定' });
              }, 500);
            }
          });
        } else {
          uni.navigateBack({
            delta: 1,
            success: () => {
              setTimeout(() => {
                uni.showToast({ title: errorMsgText, icon: 'none', duration: 3000 });
              }, 500);
            }
          });
        }
      }
    },
    fail: (err) => {
      const failMsg = '请求失败：' + (err.errMsg || '网络异常，请检查网络');
      errorMsg.value = failMsg;
      uni.navigateBack({
        delta: 1,
        success: () => {
          setTimeout(() => {
            uni.showToast({ title: failMsg, icon: 'none', duration: 3000 });
          }, 500);
        }
      });
    },
    complete: () => {
      uni.hideLoading();
      loading.value = false;
    }
  });
};

// --- 4. 辅助功能 (完全保持原样) ---
const chooseImage = () => {
  uni.chooseImage({
    count: 5 - formData.FileList.length,
    sizeType: ['compressed', 'original'],
    sourceType: ['album', 'camera'],
    success: (res) => { compressImagesToBase64(res.tempFilePaths); },
    fail: (err) => { uni.showToast({ title: '选择图片失败', icon: 'none' }); }
  });
};

const compressImagesToBase64 = (imagePaths) => {
  const promises = imagePaths.map(imagePath => {
    return compressImageToBase64(imagePath, 'jpeg', 80).catch(err => imagePath);
  });
  Promise.all(promises).then(base64Images => {
    formData.FileList.push(...base64Images);
    console.log('压缩后的图片base64数组:', formData.FileList);
  });
};

const previewImage = (current) => {
  uni.previewImage({ current: current, urls: formData.FileList });
};

const deleteImage = (index) => {
  formData.FileList.splice(index, 1);
};

// --- 5. 提交逻辑 (逻辑修改：加入批次校验) ---
const showConfirmDialog = () => {
  if (!hasEditPermission.value) {
    uni.showToast({ title: '您没有收货权限', icon: 'none' });
    return;
  }
  
  // 【新增校验】必须选择一个批次
  if (selectedBatchIndex.value === -1) {
    uni.showToast({ title: '请先在下方选择一个通知单批次', icon: 'none' });
    return;
  }

  if (!formData.InQty) {
    uni.showToast({ title: '请输入本次收货数', icon: 'none' });
    return;
  }
  const inQty = parseInt(formData.InQty);
  if (isNaN(inQty) || inQty <= 0) {
    uni.showToast({ title: '本次收货数必须为正整数', icon: 'none' });
    return;
  }
  
  showConfirm.value = true;
};

const cancelConfirm = () => {
  showConfirm.value = false;
};

const confirmSubmit = async () => {
  if (!hasEditPermission.value) {
     uni.showToast({ title: '权限已更改，无法收货', icon: 'none' });
     showConfirm.value = false;
     return;
  }
  showConfirm.value = false;
  await submitReceive();
};

const submitReceive = async () => {
  if (selectedBatchIndex.value === -1) return; // 双重保险

  // 获取当前选中的批次数据
  const selectedItem = batchList.value[selectedBatchIndex.value];

  // 生成 check_str - 【关键】必须使用选中行的数据，而不是原来的 orderList[0]
  const orderNum = selectedItem['订单号'] || '';
  const orderQty = selectedItem['订单数量'] || 0;
  const receivedQty = selectedItem['已收数量'] || 0;
  const canReceiveQty = selectedItem['已收可入数'] || 0;
  const returnQty = selectedItem['已退货数'] || 0;
  const pendingQty = selectedItem['待收货数'] || 0;
  
  const checkStr = `${orderNum}${orderQty}${receivedQty}${canReceiveQty}${returnQty}${pendingQty}`;
  
  // 获取通知单号
  const TZ_DocNum = selectedItem['通知单号'];

  // 准备提交数据
  const submitData = {
    DocNum: apiParams.value.DocNum,
    SN: apiParams.value.SN,
    SQL: apiParams.value.SQL,
    InQty: formData.InQty,
    ReceiveRemark: formData.ReceiveRemark,
    check_str: checkStr,
    FileList: formData.FileList,
    TZ_DocNum: TZ_DocNum 
  };

  console.log('提交数据:', submitData);

  uni.showLoading({ title: '提交中...', mask: true });

  try {
    const response = await uni.request({
      url: 'http://13.94.38.44:8080/YLT_SalIQC/insertData', 
      method: 'POST',
      header: { 'content-type': 'application/json' },
      data: submitData
    });
    
    let result;
    if (typeof response.data === 'string') {
      result = JSON.parse(response.data);
    } else {
      result = response.data;
    }

    if (result.isError === false) {
      uni.showToast({ title: '收货成功', icon: 'success' });
      setTimeout(() => {
        uni.navigateBack({ delta: 1 });
      }, 1500); 
    } else {
      uni.showToast({ title: result.msg || '数量发生变化，请重新扫码', icon: 'none' });
    }
  } catch (err) {
    console.error('提交失败:', err);
    uni.showToast({ title: '提交失败，请检查网络', icon: 'none' });
  } finally {
    uni.hideLoading();
  }
};
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background-color: #f5f5f5;
  /* padding-bottom: 30rpx;  <-- 删除这行，padding交给内部控制 */
  overflow: hidden; /* 防止整个页面滚动 */
}

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
  margin-top: 90rpx; /* 避开 Navbar 的高度 */
  height: calc(100vh - 90rpx); /* 限制高度，确保 scroll-view 生效 */
}

.order-container {
  /* flex: 1; <-- 删除 flex:1，因为在 scroll-view 内部不需要撑开父容器 */
  padding: 30rpx;
  display: flex;
  flex-direction: column;
  gap: 20rpx; /* 稍微增加间距美观一点 */
}

.section-title {
  font-size: 32rpx;
  font-weight: bold;
  color: #333;
  margin-bottom: 10rpx;
}

.empty-or-error {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 30rpx;
  padding: 40rpx;
}

.error-text {
  font-size: 28rpx;
  color: #ff4d4f;
  text-align: center;
}

.empty-text {
  font-size: 28rpx;
  color: #999;
}

.retry-btn {
  width: 200rpx;
  height: 70rpx;
  background-color: #029999;
  color: white;
  font-size: 26rpx;
  border-radius: 10rpx;
}
.loading-container {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 90rpx;
}

.order-card {
  background-color: white;
  border-radius: 20rpx;
  padding: 20rpx;
  box-shadow: 0 4rpx 15rpx rgba(0, 0, 0, 0.05);
 /* margin-bottom: 20rpx; */
}

.order-item {
  display: flex;
  margin-bottom: 10rpx; /* 增加每个字段之间的间距 */
  align-items: center;
  min-height: 80rpx; /* 设置最小高度 */
}

.order-item:last-child {
  margin-bottom: 0;
}

.item-label {
  font-size: 26rpx;
  color: #666;
  width: 200rpx; /* 固定标签宽度，确保对齐 */
  flex-shrink: 0; /* 防止标签被压缩 */
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
  line-height: 50rpx; /* 行高 */
  text-align: left; /* 文字左对齐 */
  pointer-events: none; /* 禁用鼠标事件，防止聚焦 */
  outline: none; /* 移除聚焦时的边框 */
  -webkit-appearance: none; /* 移除iOS默认样式 */
  appearance: none; /* 移除默认样式 */
}
.true-input{
  flex: 1;
  font-size: 26rpx;
  color: #333;
  background-color: #f9f9f9; 
  border: 1rpx solid #e0e0e0;
  border-radius: 8rpx; 
  padding: 15rpx 20rpx; 
  height: 50rpx;
  line-height: 50rpx; /* 行高 */
  text-align: left; /* 文字左对齐 */
  
  outline: none; /* 移除聚焦时的边框 */
  -webkit-appearance: none; /* 移除iOS默认样式 */
  appearance: none; /* 移除默认样式 */
}
.item-input:disabled {
  background-color: #f9f9f9; /* 禁用状态背景色 */
  color: #333; /* 禁用状态文字颜色 */
  opacity: 1; /* 确保文字可见 */
}

.font-bold {
  font-weight: bold;
  color: #029999;
}

.text-primary {
  color: #0066cc;
}

/* 为特殊字段添加样式 */
.item-input.font-bold {
  background-color: #f0f8f8;
  border-color: #029999;
  font-weight: bold;
}

.item-input.font-bold.text-primary {
  background-color: #f0f8ff;
  border-color: #0066cc;
  font-weight: bold;
}

/* 新增字段区域样式 */
.input-section {
  background-color: white;
  border-radius: 20rpx;
  padding: 15rpx;
  box-shadow: 0 4rpx 15rpx rgba(0, 0, 0, 0.05);
 
}

.input-item {
  display: flex;
  margin-bottom: 10rpx;
  align-items: center;
  min-height: 100rpx; /* 设置最小高度，为图片上传留出空间 */
}

.input-item:last-child {
  margin-bottom: 0;
}

/* 图片上传样式 */
.image-upload-container {
  flex: 1;
  display: flex;
  flex-wrap: wrap;
  gap: 10rpx;
}

.image-preview {
  position: relative;
  width: 120rpx;
  height: 120rpx;
}

.preview-image {
  width: 100%;
  height: 100%;
  border-radius: 10rpx;
  border: 1rpx solid #e0e0e0;
}

.delete-btn {
  position: absolute;
  top: -10rpx;
  right: -10rpx;
  width: 40rpx;
  height: 40rpx;
  background-color: #ff4d4f;
  color: white;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24rpx;
  cursor: pointer;
}

.upload-btn {
  width: 120rpx;
  height: 120rpx;
  border: 2rpx dashed #d0d0d0;
  border-radius: 10rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #fafafa;
  cursor: pointer;
}

.upload-text {
  font-size: 48rpx;
  color: #999;
}

/* 提交按钮样式 */
.submit-container {
  margin-top: 20rpx;
  padding: 0 20rpx;
  display: flex;
  flex-direction: column; /* 确保按钮和提示文字垂直排列 */
  align-items: center; /* 居中对齐 */
}

.submit-btn {
  width: 100%;
  height: 80rpx;
  background-color:#007aff;
  color: white;
  font-size: 32rpx;
  font-weight: bold;
  border-radius: 15rpx;
  border: none;
  /* 添加禁用状态样式 */
  opacity: 0.6;
  cursor: not-allowed;
}

.submit-btn:not([disabled]) {
  opacity: 1;
  cursor: pointer;
}

/* 可选：权限提示文字样式 */
.permission-hint {
  font-size: 24rpx;
  color: #ff4d4f;
  margin-top: 10rpx;
  text-align: center;
}

/* 确认弹窗样式 */
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

.selection-section {
  margin-top: 20rpx;
  margin-bottom: 20rpx;
}

.batch-list {
  display: flex;
  flex-wrap: wrap;
  gap: 20rpx; /* 间隙保持 20rpx */
  margin-top: 10rpx;
  justify-content: flex-start; /* 确保左对齐，防止最后一行元素居中或分散 */
}

.batch-item {
  /* 计算宽度 */
  /* (100%总宽 - 2个间隙共40rpx) / 3个元素 */
  width: calc((100% - 40rpx) / 3);

  background-color: white;
  border: 2rpx solid #e0e0e0;
  border-radius: 15rpx;
  
  /* 调整内边距，防止内容过多撑破布局，建议加上 box-sizing */
  padding: 20rpx 5rpx; 
  box-sizing: border-box; 
  
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.02);
  transition: all 0.3s;
}


/* 选中状态的样式 */
.active-batch {
  border-color: #007aff; /* 选中变蓝 */
  background-color: #f0f8ff; /* 浅蓝色背景 */
  box-shadow: 0 4rpx 12rpx rgba(0, 122, 255, 0.2);
}

.batch-qty {
  font-size: 36rpx;
  font-weight: bold;
  color: #333;
}

.active-batch .batch-qty {
  color: #007aff;
}

.batch-id {
  font-size: 22rpx;
  color: #999;
  margin-top: 6rpx;
}

.active-batch .batch-id {
  color: #666;
}
</style>