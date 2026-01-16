<template>
  <view>
    <!-- 这个组件没有实际的UI，只是提供扫码功能 -->
  </view>
</template>

<script setup>
import { getCurrentInstance } from 'vue'

const instance = getCurrentInstance()

// 扫码功能
const scanCode = (options = {}) => {
  const {
    deviceId,           // 设备ID
    successCallback,    // 扫码成功的回调函数
    failCallback,       // 扫码失败的回调函数
    parseRule = '@',    // 解析规则，默认用@分割取第二部分
    checkDeviceId = true // 是否检查设备ID
  } = options

  // 1. 检查设备码是否已获取（避免无SN无法调用接口）
  if (checkDeviceId && (!deviceId || deviceId.includes('unknown'))) {
    uni.showToast({ title: '设备码获取失败，请重试', icon: 'none' });
    return;
  }

  uni.scanCode({
    onlyFromCamera: true, 
    scanType: ['qrCode'], // 支持二维码和条形码
    success: (scanRes) => {
      // 扫码成功：获取DocNum（扫码结果）
      const scanResult = scanRes.result; // 扫码得到的完整字符串
      console.log("二维码内容:", scanResult);
      
      if (!scanResult) {
        uni.showToast({ title: '扫码结果为空，请重新扫码', icon: 'none' });
        return;
      }

      // 解析二维码内容
      let docNum = '';
      if (parseRule && scanResult.includes(parseRule)) {
        // 按指定规则分割字符串
        const parts = scanResult.split(parseRule);
        // 获取第二部分（索引为1）
        if (parts.length >= 2) {
          docNum = parts[1].trim(); // 获取第二部分并去除首尾空白字符
        } else {
          // 如果分割后少于2部分，使用第一个部分
          docNum = parts[0].trim();
        }
      } else {
        // 如果没有指定规则或不包含规则符号，直接使用扫码结果作为订单号
        docNum = scanResult.trim();
      }

      if (!docNum) {
        uni.showToast({ title: '无法解析订单号，请重新扫码', icon: 'none' });
        return;
      }

      // 调用成功回调函数
      if (successCallback && typeof successCallback === 'function') {
        successCallback({
          docNum,
          scanResult,
          deviceId
        });
      }
    },
    fail: (err) => {
      // 扫码失败（如用户取消、权限拒绝）
      if (err.errMsg !== 'scanCode:fail cancel') { // 排除"用户主动取消"的错误提示
        uni.showToast({ title: '扫码失败：' + err.errMsg, icon: 'none' });
      }
      
      // 调用失败回调函数
      if (failCallback && typeof failCallback === 'function') {
        failCallback(err);
      }
    }
  });
}

// 暴露方法给父组件使用
defineExpose({
  scanCode
})
</script>

<style>
/* 组件样式 */
</style>