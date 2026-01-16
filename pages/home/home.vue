<template>
  <view class="container">
    <navbar></navbar>
    
    <!-- 主要功能按钮区域：改为 Grid 布局 -->
    <view class="menu-grid">
      <view 
        class="square-button" 
        v-for="(item, index) in menuList" 
        :key="index"
      >
        <button class="icon-text-container" @click="item.action">
          <image class="icon" :src="item.icon"></image>
          <text class="text">{{ item.text }}</text>
        </button>
      </view>
    </view>

    <!-- 设备码功能区域 (保持不变) -->
    <view class="device-id-section">
      <view class="device-id-container">
        <view class="device-btn" @click="toggleDeviceId">
          <template v-if="showDeviceId">
            <text v-if="deviceId && !deviceIdLoading" class="device-id-text" selectable>{{ deviceId }}</text>
            <text v-else-if="deviceIdLoading">获取中...</text>
            <text v-else>获取失败</text>
            <view v-if="deviceId && !deviceIdLoading" class="copy-btn" @click.stop="copyDeviceId">
              <text class="copy-text">复制</text>
            </view>
          </template>
          <template v-else>
            <text class="device-btn-text">获取设备码</text>
          </template>
        </view>
      </view>
    </view>
    
    <!-- 扫码组件 -->
    <ScanCode ref="scanCodeRef" />
  </view>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import Navbar from "@/components/Navbar.vue"
import ScanCode from "@/components/ScanCode.vue"
import checkUpdate from '@/uni_modules/uni-upgrade-center-app/utils/check-update'

// 响应式数据
const showDeviceId = ref(false)
const deviceId = ref(null)
const deviceIdLoading = ref(false)
const hasAttemptedToFetchId = ref(false)
const scanCodeRef = ref(null)

// ----------------------
// 核心逻辑：菜单配置
// ----------------------

// 定义具体的跳转/点击方法
const navigateToReceiveDetail = () => {
  uni.navigateTo({ url: `/pages/receive/receive-detail/receive-detail?SN=${encodeURIComponent(deviceId.value)}` })
}

const navigateToWareDetail = () => {
  if (!checkDeviceId()) return;
  uni.navigateTo({ url: `/pages/warehousing/ware-detail/ware-detail?SN=${encodeURIComponent(deviceId.value)}` });
}

const navigateToOutboundDetail = () => {
  if (!checkDeviceId()) return;
  uni.navigateTo({ url: `/pages/outbound/outbound-detail1/outbound-detail1?SN=${encodeURIComponent(deviceId.value)}` });
}

const handleReceiveScan = () => {
  scanCodeRef.value.scanCode({
    deviceId: deviceId.value,
    successCallback: (result) => {
      uni.navigateTo({ url: `/pages/receive/receive-scan/receive-scan?DocNum=${encodeURIComponent(result.docNum)}&SN=${encodeURIComponent(result.deviceId)}` });
    }
  });
};

const handleWareScan = () => {
  scanCodeRef.value.scanCode({
    deviceId: deviceId.value,
    successCallback: (result) => {
      uni.navigateTo({ url: `/pages/warehousing/ware-scan/ware-scan?DocNum=${encodeURIComponent(result.docNum)}&SN=${encodeURIComponent(result.deviceId)}` });
    }
  });
};

const handleOutboundScan = () => {
  uni.showToast({ title: `暂无此功能`, icon: 'none' });
}

// 新增：待出库清单逻辑
const handleOutboundList = () => {
  if (!checkDeviceId()) return;
  uni.navigateTo({ url: `/pages/outbound/outbound-List/outbound-List?SN=${encodeURIComponent(deviceId.value)}` });
}

const handleKehuQueding = () =>{
	if (!checkDeviceId()) return;
	uni.navigateTo({ url: `/pages/kehu/KehuQueding-1/KehuQueding-1?SN=${encodeURIComponent(deviceId.value)}` });
}

// 辅助方法：检查设备ID
const checkDeviceId = () => {
  if (!deviceId.value || deviceId.value.includes('unknown')) {
    uni.showToast({ title: '设备码获取失败，请重试', icon: 'none' });
    return false;
  }
  return true;
}

// 菜单数据列表 (在这里配置按钮)
const menuList = [
  { text: '待收货明细', icon: '../../static/icon/Receive.png', action: navigateToReceiveDetail },
  { text: '待入库明细', icon: '../../static/icon/Into.png', action: navigateToWareDetail },
  { text: '待出库明细', icon: '../../static/icon/Outbound.png', action: navigateToOutboundDetail },
  { text: '收货扫码', icon: '../../static/icon/ReceiveScan.png', action: handleReceiveScan },
  { text: '入库扫码', icon: '../../static/icon/IntoScan.png', action: handleWareScan },
  { text: '出库扫码', icon: '../../static/icon/OutboundScan.png', action: handleOutboundScan },
  { text: '待出库清单', icon: '../../static/icon/OutboundList.png', action: handleOutboundList },
  { text: '客户提货确认', icon: '../../static/icon/KehuQueding.png', action: handleKehuQueding }
]

// ----------------------
// 生命周期与基础逻辑
// ----------------------

onMounted(() => {
  getDeviceId();
  checkUpdate();	 
})

const getDeviceId = () => {
  if (deviceId.value || hasAttemptedToFetchId.value) return
  hasAttemptedToFetchId.value = true
  deviceIdLoading.value = true
  uni.getSystemInfo({
    success: (systemInfo) => {
      if (systemInfo.deviceId) {
        deviceId.value = systemInfo.deviceId
      } else {
        deviceId.value = 'device_' + Date.now().toString(36).slice(-8)
      }
      deviceIdLoading.value = false
    },
    fail: (err) => {
      deviceId.value = 'unknown_' + Date.now().toString(36).slice(-8)
      deviceIdLoading.value = false
    }
  })
}

const toggleDeviceId = () => {
  showDeviceId.value = !showDeviceId.value
  if (showDeviceId.value && !deviceId.value && !hasAttemptedToFetchId.value) {
    getDeviceId()
  }
}

const copyDeviceId = () => {
  if (deviceIdLoading.value || !deviceId.value || deviceId.value.toString().includes('unknown')) {
    uni.showToast({ title: 'ID无效', icon: 'none' })
    return
  }
  uni.setClipboardData({
    data: deviceId.value,
    success: () => uni.showToast({ title: '已复制', icon: 'success' }),
    fail: () => uni.showToast({ title: '复制失败', icon: 'none' })
  })
}
</script>

<style>
.container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background-color: white;
}

/* --- 核心修改：Grid 布局 --- */
.menu-grid {
  flex: 1;
  display: grid;
  /* 创建3列，每列宽度相等 */
  grid-template-columns: repeat(3, 1fr); 
  /* 设置行间距和列间距 */
  gap: 20rpx; 
  /* 整体内边距 */
  padding: 40rpx;
  /* 内容顶部对齐，避免垂直拉伸 */
  align-content: start; 
}

.square-button {
  /* 在 Grid 中，不需要手动设置宽度为 32%，Grid 会自动处理 */
  width: 100%; 
}
/* ------------------------- */

.icon-text-container {
  width: 100%;
  aspect-ratio: 1 / 1;
  background-color: white;
  color: black;
  text-align: center;
  border-radius: 10px;
  cursor: pointer;
  box-shadow: 0 0 20rpx rgba(0, 0, 0, 0.1);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  line-height: 1.2;
  position: relative;
}

.icon-text-container:active {
  transform: scale(0.95) rotateZ(1.7deg);
}

.icon {
  width: 140rpx;
  height: 140rpx;
  margin-bottom: 10rpx;
}

.text {
  font-size: 15px;
}

/* 设备码样式保持不变 */
.device-id-section {
  width: 100%;
  padding: 40rpx;
}

.device-id-container {
  width: 90%;
}

.device-btn {
  width: 100%;
  height: 120rpx;
  background: rgba(217, 217, 217, 0.58);
  border: 1px solid white;
  box-shadow: 12px 17px 51px rgba(0, 0, 0, 0.22);
  backdrop-filter: blur(6px);
  border-radius: 17px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0 20rpx;
  box-sizing: border-box;
  cursor: pointer;
  transition: all 0.5s;
  position: relative;
}

.device-btn:hover {
  border: 1px solid black;
  transform: scale(1.02);
}

.device-btn:active {
  transform: scale(0.98) rotateZ(1.7deg);
}

.device-id-text {
  font-size: 26rpx;
  color: #0066cc;
  margin-right: 10rpx;
  flex: 1;
  word-break: break-all;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.copy-btn {
  width: 100rpx;
  height: 50rpx;
  background-color: #029999;
  border-radius: 8rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.copy-text {
  color: white;
  font-size: 24rpx;
  font-weight: bold;
}
</style>