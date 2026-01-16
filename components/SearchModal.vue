<template>
  <view v-if="show" class="modal-overlay" @click="close">
    <view class="modal-content" @click.stop>
      <view class="modal-header">
        <text class="modal-title">搜索 {{ title }}</text>
        <view class="close-btn" @click="close">×</view>
      </view>
      
      <view class="modal-search-box">
        <view class="search-input-wrapper">
          <text class="search-icon">🔍</text>
          <input 
            class="search-input" 
            v-model="keyword" 
            :placeholder="'请输入' + title" 
            focus
          />
          <text v-if="keyword" class="clear-icon" @click="keyword=''">×</text>
        </view>
      </view>

      <scroll-view scroll-y="true" class="modal-list" :show-scrollbar="true">
        <view 
          class="modal-item" 
          v-for="(item, index) in filteredList" 
          :key="index"
          @click="selectItem(item)"
        >
          <view class="item-main">
            {{ item[searchKey] }}
          </view>
          
          <view class="item-sub" v-if="subKey && item[subKey]">
            {{ subKey }}: {{ item[subKey] }}
          </view>
        </view>

        <view v-if="filteredList.length === 0" class="empty-tip">
          未找到相关数据
        </view>
      </scroll-view>
    </view>
  </view>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  show: { type: Boolean, default: false }, // 控制显示
  title: { type: String, default: '内容' }, // 标题
  list: { type: Array, default: () => [] }, // 数据源
  searchKey: { type: String, required: true }, // 搜索匹配的字段名
  subKey: { type: String, default: '' } // 辅助显示的字段名（可选）
})

const emit = defineEmits(['update:show', 'select'])

const keyword = ref('')

// 监听弹窗打开，清空上次的搜索词
watch(() => props.show, (val) => {
  if (val) keyword.value = '',
  console.log('【组件内部】接收到 show 变化:', val)
})


// 计算过滤结果
const filteredList = computed(() => {
  // 如果没有关键词，返回空数组，而不是全部数据
  if (!keyword.value) return [] 
  
  const key = keyword.value.toLowerCase()
  return props.list.filter(item => {
    // 安全获取字段值并转为字符串
    const val = String(item[props.searchKey] || '').toLowerCase()
    return val.includes(key)
  })
})


const close = () => {
  emit('update:show', false)
}

const selectItem = (item) => {
  emit('select', item)
  close()
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  z-index: 2000;
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-content {
  width: 85%;
  height: 65vh;
  background-color: #fff;
  border-radius: 20rpx;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  box-shadow: 0 4rpx 20rpx rgba(0,0,0,0.2);
}

.modal-header {
  padding: 24rpx 30rpx;
  background-color: #f8f8f8;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1rpx solid #eee;
}

.modal-title {
  font-size: 32rpx;
  font-weight: bold;
  color: #333;
}

.close-btn {
  font-size: 44rpx;
  color: #999;
  line-height: 1;
  padding: 0 10rpx;
}

.modal-search-box {
  padding: 20rpx 30rpx;
  border-bottom: 1rpx solid #f5f5f5;
}

.search-input-wrapper {
  background-color: #f0f0f0;
  border-radius: 40rpx;
  padding: 0 24rpx;
  height: 72rpx;
  display: flex;
  align-items: center;
}

.search-icon { font-size: 28rpx; margin-right: 10rpx; }
.clear-icon { font-size: 32rpx; color: #999; padding: 10rpx; }
.search-input { flex: 1; font-size: 28rpx; height: 100%; }

.modal-list { 
  flex: 1; 
  /* 新增下面这两行 */
  height: 0;   /* 关键：强制高度由 flex 分配，防止被内容撑开 */
  width: 100%; /* 确保宽度占满 */
  
  padding: 10rpx 0; 
}
.modal-item {
  padding: 24rpx 30rpx;
  border-bottom: 1rpx solid #f9f9f9;
}
.modal-item:active { background-color: #f0f0f0; }

.item-main { font-size: 30rpx; color: #333; font-weight: 500; margin-bottom: 6rpx; }
.item-sub { font-size: 24rpx; color: #999; }

.empty-tip { text-align: center; color: #ccc; margin-top: 100rpx; font-size: 28rpx; }
.modal-list ::-webkit-scrollbar {
  width: 12rpx !important;    /* 纵向滚动条的宽度 */
  height: 12rpx !important;   /* 横向滚动条的高度 */
  background-color: transparent; /* 背景透明 */
  display: block;             /* 强制显示 */
}

/* 滚动条滑块（那个移动的小条） */
.modal-list ::-webkit-scrollbar-thumb {
  background-color: #ccc;     /* 灰色 */
  border-radius: 6rpx;        /* 圆角 */
}

/* 滚动条轨道（滑块跑动的背景） */
.modal-list ::-webkit-scrollbar-track {
  background-color: #f5f5f5;  /* 浅色背景，让轨道也隐约可见，方便对准 */
  /* 如果想要透明轨道，可以把上面这行改为 background-color: transparent; */
}
</style>