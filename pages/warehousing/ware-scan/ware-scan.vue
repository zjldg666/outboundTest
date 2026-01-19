<!-- 入库扫码 -->
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
          <text class="empty-text">暂无该订单的入库信息</text>
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
              <text class="item-label">待入库数：</text>
              <input 
                class="item-input font-bold text-primary" 
                :value="currentPendingQty || '请选择入库批次'"
                :disabled="true"
                :readonly="true"
                type="text"
              />
            </view>
          </view>

          <view class="input-section" v-if="inspectionList.length > 0">
            <text class="section-title">入库批次</text>
            <view class="batch-list">
              <view 
                v-for="(item, index) in inspectionList" 
                :key="index"
                class="batch-item"
                :class="{ 'active-batch': selectedInspectionIndex === index }"
                @click="selectInspection(index)"
              >
                <text class="batch-qty">{{ item['待入库数'] }}</text>
              </view>
            </view>
          </view>

          <view class="input-section">
            <view class="input-item">
              <text class="item-label">本次入库数：</text>
              <input 
                class="true-input" 
                v-model="formData.InQty"
                placeholder="请输入本次入库数"
                type="number"
                @input="onInQtyInput"
              />
            </view>
            
            <view class="input-item">
                <text class="item-label">入库板位：</text>
                
                <view class="store-btn-group">
                  <button 
                    class="ware-button common-btn" 
                    :class="{ 'text-primary font-bold': storeName }"
                    @click="scanStoreCode" 
                    :disabled="!hasEditPermission"
                  >
                    {{ storeName || '板位扫码' }}
                  </button>

                  <button 
                    class="ware-button common-btn" 
                    @click="openStoreSelect" 
                    :disabled="!hasEditPermission"
                  >
                    手动选择
                  </button>
                </view>
            </view>
                        
			<view v-if="showStoreModal" class="confirm-modal">
			  <view class="modal-content store-modal-content">
			    
			    <view class="modal-header">
			      <text class="modal-title">选择入库板位</text>
			      <text class="close-icon" @click="closeStoreSelect">×</text>
			    </view>
			
			    <view class="store-search-box">
			      <view class="search-input-wrapper">
			        <text class="search-icon">🔍</text>
			        <input 
			          class="search-input" 
			          v-model="storeSearchKey" 
			          placeholder="搜索板位名称或编码" 
			          confirm-type="search"
			        />
			        <text v-if="storeSearchKey" class="clear-icon" @click="storeSearchKey=''">×</text>
			      </view>
			    </view>
			    
			    <scroll-view v-if="storeSearchKey" scroll-y class="search-result-list">
			      <view 
			        v-for="(item, index) in searchResults" 
			        :key="item.ST_ID || index"
			        class="store-item search-item"
			        @click="confirmStoreSelection(item)"
			      >
			        <view class="item-main">{{ item.ST_NAME }}</view>
			        <view class="item-sub" v-if="item.ST_CODE">{{ item.ST_CODE }}</view>
			      </view>
			      <view v-if="searchResults.length === 0" class="empty-tip">
			        未找到相关板位
			      </view>
			    </scroll-view>
			
			    <view v-else class="store-picker-body">
			      <scroll-view scroll-y class="store-col left-col">
			        <view 
			          v-for="(parent, index) in storeTree" 
			          :key="parent.ST_ID"
			          class="store-item parent-item"
			          :class="{ 'active': activeParentIndex === index }"
			          @click="handleParentClick(index)"
			        >
			          {{ parent.ST_NAME }}
			        </view>
			      </scroll-view>
			      
			      <scroll-view scroll-y class="store-col right-col">
			        <view v-if="storeTree[activeParentIndex] && storeTree[activeParentIndex].children.length > 0">
			           <view 
			            v-for="(child, cIndex) in storeTree[activeParentIndex].children" 
			            :key="child.ST_ID"
			            class="store-item child-item"
			            @click="confirmStoreSelection(child)"
			          >
			            {{ child.ST_NAME }}
			          </view>
			        </view>
			        <view v-else class="empty-tip">无子级板位</view>
			      </scroll-view>
			    </view>
			
			  </view>
			</view>
				
            
            <view class="input-item">
              <text class="item-label">入货备注：</text>
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
                <view class="upload-btn" v-if="formData.FileList.length < 5 " @click="chooseImage">
                  <text class="upload-text">+</text>
                </view>
              </view>
            </view>

            <view class="submit-container">
              <button class="submit-btn" :disabled="!hasEditPermission" @click="showConfirmDialog">
                {{ hasEditPermission ? '确定入库' : '无入库权限' }}
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
          <text class="modal-title">确认入库</text>
        </view>
        <view class="modal-body">
          <text class="modal-text">您确定要进行入库操作吗？</text>
          <text class="modal-text">本次入库数：{{ formData.InQty }}</text>
          <text class="modal-text" v-if="storeName">入库板位：{{ storeName }}</text>
          <text class="modal-text" v-if="formData.ReceiveRemark">入库备注：{{ formData.ReceiveRemark }}</text>
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
// 图片压缩并转换为base64格式
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
          const base64 = `data:image/${type};base64,${uni.arrayBufferToBase64(res.data)}`;
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
              const base64 = `data:image/${type};base64,${readRes.data}`;
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
              const base64 = `data:image/${type};base64,${readRes.data}`;
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

// 响应式数据修改
const loading = ref(false);
const errorMsg = ref('');

const commonOrderInfo = ref({}); // 存放公共信息（供应商、料号等）
const inspectionList = ref([]);  // 存放检验单列表（待入库数、检验单号）
const selectedInspectionIndex = ref(-1); // 当前选中的索引
const currentPendingQty = ref(''); // 当前显示的待入库数
const showConfirm = ref(false); // 控制确认弹窗显示
const hasEditPermission = ref(false); // 新增：存储入库权限状态

const showStoreModal = ref(false); // 控制弹窗显示
const storeTree = ref([]); // 存放处理后的树形数据
const activeParentIndex = ref(0); // 当前选中的父级索引
const allStoreList = ref([]); // 存放扁平化的原始数据
const storeSearchKey = ref(''); // 搜索关键词
import { computed } from 'vue'; // 确保引入 computed

const searchResults = computed(() => {
  if (!storeSearchKey.value) return [];
  const key = storeSearchKey.value.toLowerCase();
  
  // 从全量数据中过滤
  return allStoreList.value.filter(item => {
    const name = String(item.ST_NAME || '').toLowerCase();
    const code = String(item.ST_CODE || '').toLowerCase();
    return name.includes(key) || code.includes(key);
  });
});

const apiParams = ref({
  DocNum: '',
  SN: '',
  SQL: 'SCDATA_YLT_20250930'
});

// 板位相关数据
const storeName = ref(''); // 存储板位名称
const storeCode = ref(''); // 存储板位编码
const DocNum = ref('');//提交入库信息时的订单
// 表单数据
const formData = reactive({
  InQty: '',
  ReceiveRemark: '',
  FileList: [] // 存储图片的base64数组
});

onLoad((options) => {
  console.log('接收的参数:', options);
  
  // 1. 获取参数（建议改个局部变量名，比如 docNumVal，或者直接处理）
  const docNumVal = options.DocNum; 
  const snVal = options.SN;
  
  if (!docNumVal || !snVal) {
    errorMsg.value = '参数错误：缺少订单号或设备码';
    return;
  }

  // 2. ✅ 正确赋值给顶部的响应式变量 (注意要加 .value)
  DocNum.value = docNumVal; 

  // 3. 同时赋值给接口参数
  apiParams.value.DocNum = docNumVal;
  apiParams.value.SN = snVal;
  
  fetchOrderInfo();
});


// 格式化字段值：处理null/undefined，区分数字和字符串
const formatValue = (value, key) => {
  // 数字类型字段（如数量）空值显示0
  if (['订单数量', '已收数量', '已收可入数', '已退货数', '待收货数', '待入库数'].includes(key)) { // 添加了待入库数
    return value || 0;
  }
  // 其他字段空值显示"无"
  return value || '无';
};

//选择检验单批次
const selectInspection = (index) => {
  selectedInspectionIndex.value = index;
  const item = inspectionList.value[index];
  
  // 1. 将选中的“待入库数”填入上方显示框
  currentPendingQty.value = item['待入库数'];
  
  // 2. 【核心】更新主卡片显示的详细信息
  const displayInfo = { ...item };
  
  // 移除不需要在顶部重复显示的字段
  delete displayInfo['待入库数']; // 下方输入框旁已有显示
  delete displayInfo['检验单号']; // 按钮上已有显示
  // 保留：供应商、业务料号、客户料号、订单号 等信息
  
  commonOrderInfo.value = displayInfo;
};

// 1. 打开板位选择弹窗
const openStoreSelect = async () => {
  if (!hasEditPermission.value) { /* ... */ return; }
  
  storeSearchKey.value = ''; // 【新增】每次打开重置搜索词
  
  if (storeTree.value.length === 0) {
    await fetchStoreList();
  }
  if (storeTree.value.length > 0) {
    showStoreModal.value = true;
  }
};

// 2. 关闭弹窗
const closeStoreSelect = () => {
  showStoreModal.value = false;
};

// 3. 获取并处理板位列表数据
const fetchStoreList = () => {
  uni.showLoading({ title: '加载板位...' });
  return new Promise((resolve) => {
    uni.request({
      url: 'http://13.94.38.44:8080/YLT_CpRk/Show_AllStoreName',
      method: 'POST',
      data: { "SQL": "SCDATA_YLT_20250930" },
      success: (res) => {
        let result = res.data;
        if (typeof result === 'string') {
          try { result = JSON.parse(result); } catch (e) {}
        }

        if (result && !result.isError && result.dt) {
          allStoreList.value = result.dt; // 【新增】保存原始数据供搜索使用
          processStoreData(result.dt);    // 处理树形结构
        } else {
          uni.showToast({ title: '获取板位失败', icon: 'none' });
        }
      },
      fail: (err) => { /* ...保持原有错误处理... */ },
      complete: () => {
        uni.hideLoading();
        resolve();
      }
    });
  });
};

// 4. 数据处理逻辑 (核心算法)
const processStoreData = (dataList) => {
  // 第一步：找到所有目录 (ST_PARENT 为 "TREETOP")
  const roots = dataList.filter(item => item.ST_PARENT === 'TREETOP');
  
  // 第二步：遍历目录，寻找其子级
  const tree = roots.map(root => {
    // 找到 ST_PARENT 等于当前目录 ST_CODE 的数据
    const children = dataList.filter(child => child.ST_PARENT === root.ST_CODE);
    return {
      ...root,
      children: children
    };
  });
  
  storeTree.value = tree;
  console.log('构建的板位树:', tree);
};

// 5. 点击父级分类
const handleParentClick = (index) => {
  activeParentIndex.value = index;
};

// 6. 确认选择子级板位
const confirmStoreSelection = (childItem) => {
  storeName.value = childItem.ST_NAME;
  storeCode.value = childItem.ST_CODE; // 记录板位编码
  
  showStoreModal.value = false; // 关闭弹窗
  
  uni.showToast({
    title: `已选择: ${childItem.ST_NAME}`,
    icon: 'none'
  });
};

// 输入验证：只允许正整数
const onInQtyInput = (e) => {
  let value = e.detail.value;
  // 只允许正整数
  value = value.replace(/[^\d]/g, '');
  if (value.startsWith('0') && value.length > 1) {
    // 移除前导零，但保留单个0
    value = value.replace(/^0+/, '');
  }
  formData.InQty = value;
};

const fetchOrderInfo = () => {
  loading.value = true;
  errorMsg.value = '';
  
  // 重置相关数据
  commonOrderInfo.value = {}; 
  inspectionList.value = [];  
  selectedInspectionIndex.value = -1; 
  currentPendingQty.value = ''; 
  hasEditPermission.value = false;

  uni.request({
    url: 'http://13.94.38.44:8080/YLT_CpRk/fill_Info',
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
        // 设置权限状态
        hasEditPermission.value = response.isEdit === true;
        console.log('设置入库权限:', hasEditPermission.value);

        const rawList = Array.isArray(response.dt) ? response.dt : [];
        
        if (rawList.length > 0) {
          // --- 核心修改：只初始化订单号，不初始化其他可能变化的信息 ---
          
          // 1. 初始化 commonOrderInfo，只放一个绝对确定的字段：订单号
          // 这样主卡片初始时只显示订单号，不会显示错误的供应商
          commonOrderInfo.value = { 
            '订单号': rawList[0]['订单号'] || apiParams.value.DocNum 
          };
          
          // 2. 构建批次列表 (保留所有字段，以便选择时回填)
          inspectionList.value = rawList.map(item => ({
            ...item, // 保留全部原始数据(含供应商、料号等)，供 selectInspection 使用
            '待入库数': item['待入库数'],
            '检验单号': item['检验单号'] 
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
        
        // 特殊处理"无相关权限"错误
        if (response?.msg === '无相关权限') {
          uni.navigateBack({
            delta: 1,
            success: () => {
              setTimeout(() => {
                uni.showModal({
                  title: '权限提示',
                  content: '无相关权限，请点击下方获取设备码，并发给管理员',
                  showCancel: false,
                  confirmText: '确定'
                });
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

// 选择图片
const chooseImage = () => {
  // 检查权限
  if (!hasEditPermission.value) {
    uni.showToast({
      title: '您没有入库权限',
      icon: 'none'
    });
    return;
  }

  uni.chooseImage({
    count: 5 - formData.FileList.length, // 最多上传5张
    sizeType: ['compressed', 'original'], // 优先使用压缩图
    sourceType: ['album', 'camera'], // 从相册选择或拍照
    success: (res) => {
      // 将选择的图片压缩并转换为base64
      compressImagesToBase64(res.tempFilePaths);
    },
    fail: (err) => {
      console.error('选择图片失败:', err);
      uni.showToast({
        title: '选择图片失败',
        icon: 'none'
      });
    }
  });
};

// 将图片路径压缩并转换为 Base64 编码
const compressImagesToBase64 = (imagePaths) => {
  const promises = imagePaths.map(imagePath => {
    return compressImageToBase64(imagePath, 'jpeg', 80).catch(err => {
      console.error('压缩并转换图片失败，使用原始路径:', err, imagePath);
      // 如果转换失败，直接使用原始路径
      return imagePath;
    });
  });

  Promise.all(promises).then(base64Images => {
    // 更新 FileList 数组中的 Base64 编码
    formData.FileList.push(...base64Images);
    console.log('压缩后的图片base64数组:', formData.FileList);
  }).catch(err => {
    console.error('批量压缩图片失败:', err);
  });
};

// 预览图片
const previewImage = (current) => {
  // 检查权限
  if (!hasEditPermission.value) {
    uni.showToast({
      title: '您没有入库权限',
      icon: 'none'
    });
    return;
  }
  uni.previewImage({
    current: current,
    urls: formData.FileList
  });
};

// 删除图片
const deleteImage = (index) => {
  // 检查权限
  if (!hasEditPermission.value) {
    uni.showToast({
      title: '您没有入库权限',
      icon: 'none'
    });
    return;
  }
  formData.FileList.splice(index, 1);
};

// 扫码获取板位信息
const scanStoreCode = async () => {
  // 检查权限
  if (!hasEditPermission.value) {
    uni.showToast({
      title: '您没有入库权限',
      icon: 'none'
    });
    return;
  }

  uni.scanCode({
    onlyFromCamera: true,
    scanType: ['qrCode'],
    success: async (scanRes) => {
      const stCode = scanRes.result;
      if (!stCode) {
        uni.showToast({
          title: '扫码结果为空',
          icon: 'none'
        });
        return;
      }

      try {
        // 调用接口获取板位名称
        const response = await uni.request({
          url: 'http://13.94.38.44:8080/YLT_CpRk/Show_STOREName',
          method: 'POST',
          header: {
            'content-type': 'application/json'
          },
          data: {
            SN: apiParams.value.SN,
            SQL: apiParams.value.SQL,
            st_code: stCode
          }
        });

        let result;
        if (typeof response.data === 'string') {
          result = JSON.parse(response.data);
        } else {
          result = response.data;
        }

        if (result.isError === false) {
          storeName.value = result.return_stname || stCode; // 如果没有返回名称，显示原始编码
          storeCode.value = stCode; // 保存板位编码
          uni.showToast({
            title: '板位获取成功',
            icon: 'success'
          });
        } else {
          uni.showToast({
            title: result.msg || '获取板位信息失败',
            icon: 'none'
          });
        }
      } catch (err) {
        console.error('获取板位信息失败:', err);
        uni.showToast({
          title: '获取板位信息失败，请重试',
          icon: 'none'
        });
      }
    },
    fail: (err) => {
      if (err.errMsg !== 'scanCode:fail cancel') {
        uni.showToast({
          title: '扫码失败：' + err.errMsg,
          icon: 'none'
        });
      }
    }
  });
};

// 显示确认弹窗
const showConfirmDialog = () => {
  // 验证权限
  if (!hasEditPermission.value) {
    uni.showToast({
      title: '您没有入库权限',
      icon: 'none'
    });
    return;
  }

  // 验证必填字段
  if (!formData.InQty) {
    uni.showToast({
      title: '请输入本次入库数',
      icon: 'none'
    });
    return;
  }

  // 验证是否为正整数
  const inQty = parseInt(formData.InQty);
  if (isNaN(inQty) || inQty <= 0) {
    uni.showToast({
      title: '本次入库数必须为正整数',
      icon: 'none'
    });
    return;
  }
  
// 4. 【新增】验证是否选择了入库批次
  if (selectedInspectionIndex.value === -1) {
    uni.showToast({ title: '请先在下方选择一个入库批次', icon: 'none' });
    return;
  }
  
  showConfirm.value = true;
};

// 取消确认
const cancelConfirm = () => {
  showConfirm.value = false;
};

// 确认提交
const confirmSubmit = async () => {
    // 再次检查权限（防止弹窗期间权限改变，虽然可能性小）
    if (!hasEditPermission.value) {
        uni.showToast({
        title: '权限已更改，无法入库',
        icon: 'none'
        });
        showConfirm.value = false;
        return;
    }

  showConfirm.value = false; // 隐藏弹窗
  await submitReceive();
};


// 提交入库信息
const submitReceive = async () => {
  // 验证权限
  if (!hasEditPermission.value) {
    uni.showToast({ title: '您没有入库权限', icon: 'none' });
    return;
  }

  // 【新增】验证是否选择了入库批次
  if (selectedInspectionIndex.value === -1) {
    uni.showToast({ title: '请先在下方选择一个入库批次', icon: 'none' });
    return;
  }

  // 验证必填字段
  if (!formData.InQty) {
    uni.showToast({ title: '请输入本次入库数', icon: 'none' });
    return;
  }

  // 验证是否为正整数
  const inQty = parseInt(formData.InQty);
  if (isNaN(inQty) || inQty <= 0) {
    uni.showToast({ title: '本次入库数必须为正整数', icon: 'none' });
    return;
  }

  // 验证板位是否已选择
  if (!storeCode.value) {
    uni.showToast({ title: '请先选择入库板位', icon: 'none' });
    return;
  }
  
  // 获取当前选中的检验单号
  const selectedItem = inspectionList.value[selectedInspectionIndex.value];
  const IQCDocnum = selectedItem['检验单号']; // 获取检验单号

  // 准备提交数据
  const submitData = {
    DocNum: DocNum.value,
    SN: apiParams.value.SN,
    SQL: apiParams.value.SQL,
    st_code: storeCode.value, // 板位编码
    InQty: formData.InQty,
    ReceiveRemark: formData.ReceiveRemark,
    FileList: formData.FileList, // base64格式的图片数组
    IQCDocnum: IQCDocnum // 【关键】新增检验单号字段，请确认后端接收字段名为 CheckNo 或 InspectionNo
  };

  console.log('提交数据:', submitData);

  uni.showLoading({ title: '提交中...', mask: true });

  try {
    const response = await uni.request({
      url: 'http://13.94.38.44:8080/YLT_CpRk/insertData', // 入库接口
      method: 'POST',
      header: { 'content-type': 'application/json' },
      data: submitData
    });
    
    let result;
    console.log(response);
    if (typeof response.data === 'string') {
      result = JSON.parse(response.data);
    } else {
      result = response.data;
    }

	if (result.isError === false) {
      uni.showToast({ title: '收货成功', icon: 'success' });
      
      uni.$emit('refreshReceiveList');

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
  
  overflow: hidden;
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
  margin-top: 90rpx; 
  height: calc(100vh - 90rpx);
}

.order-container {

  padding: 30rpx;
  display: flex;
  flex-direction: column;
  gap: 20rpx; 
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

.ware-button{
  width: 500rpx;
  height: 80rpx;
  font-size: 26rpx;
  text-align: left;
  line-height: 80rpx; /* 行高和容器高度一样时，文本上下居中 */
  /* 添加禁用状态样式 */
  opacity: 0.6;
  cursor: not-allowed;
}

.ware-button:not([disabled]) {
  opacity: 1;
  cursor: pointer;
}

.retry-btn {
  width: 200rpx;
  height: 70rpx;
  background-color: #029999;
  color: white;
  font-size: 26rpx;
  border-radius: 10rpx;
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

.common-btn.text-primary {
  background-color: #f0f8ff !important; /* 对应 item-input.text-primary 的浅蓝背景 */
  border-color: #0066cc !important;     /* 对应 item-input.text-primary 的深蓝边框 */
  color: #0066cc !important;            /* 深蓝文字 */
  font-weight: bold;
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
  color: #0066cc;
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
  /* 添加禁用状态样式 */
  opacity: 0.6;
  cursor: not-allowed;
}

.upload-btn:not([disabled]) {
  opacity: 1;
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
  background-color: #007aff;
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
  background-color:#007aff;
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

/* --- 批次选择区域样式 --- */

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


/* --- 按钮组容器 --- */
.store-btn-group {
  flex: 1;               /* 填满右侧剩余空间 */
  display: flex;         /* 启用 Flex 布局 */
  gap: 20rpx;            /* 两个按钮之间的间距 */
  align-items: center;   /* 垂直居中 */
}

/* --- 统一按钮样式 --- */
.common-btn {
  flex: 1;               /* 核心：两个按钮平分宽度 */
  width: auto !important;/* 覆盖原 .ware-button 的固定宽度 */
  height: 80rpx;         /* 统一高度 */
  line-height: 80rpx;    /* 统一行高 */
  font-size: 26rpx;      /* 统一字体大小 */
  padding: 0 10rpx;      /* 左右内边距 */
  margin: 0;             /* 去除默认 margin */
  
  /* 统一外观风格 (仿照原来的输入框风格) */
  background-color: #f9f9f9; 
  border: 1rpx solid #e0e0e0;
  border-radius: 8rpx;
  color: #333;
  
  /* 文本样式 */
  text-align: center;    /* 建议居中显示，比左对齐更美观 */
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* 点击时的反馈效果 */
.common-btn:active {
  background-color: #e0e0e0;
}

/* 覆盖原有的 .ware-button 样式冲突 (如果原样式中写了 width: 500rpx) */
.ware-button {
  width: auto; 
}

/* 扫码按钮样式 (左边，稍微宽一点显示名称) */
.half-btn {
  flex: 1.5;
  font-size: 26rpx;
  background-color: #f9f9f9; /* 默认灰色 */
  color: #333;
  border: 1rpx solid #e0e0e0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* 选择按钮样式 (右边，蓝色) */
.select-btn {
  flex: 0.8;
  background-color: #007aff;
  color: white;
  font-size: 26rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 10rpx;
}

/* --- 弹窗样式 --- */
.store-modal-content {
  width: 90%;
  height: 70vh; /* 弹窗高度 */
  display: flex;
  flex-direction: column;
  padding: 0;
  overflow: hidden;
}

.store-picker-body {
  flex: 1;
  display: flex;
  border-top: 1rpx solid #eee;
  height: 0; /* 配合 flex:1 实现滚动 */
}

.store-col {
  height: 100%;
}

/* 左侧父级列表 */
.left-col {
  flex: 1;
  background-color: #f5f5f5;
  border-right: 1rpx solid #eee;
}

/* 右侧子级列表 */
.right-col {
  flex: 1.5;
  background-color: #fff;
}

.store-item {
  padding: 24rpx 20rpx;
  font-size: 28rpx;
  border-bottom: 1rpx solid #f0f0f0;
}

/* 选中的父级样式 */
.parent-item.active {
  background-color: #fff;
  color: #007aff;
  font-weight: bold;
  border-left: 6rpx solid #007aff;
}

.child-item:active {
  background-color: #f0f8ff;
}

.modal-header {
  position: relative;
  padding: 30rpx;
  text-align: center;
}

.close-icon {
  position: absolute;
  right: 30rpx;
  top: 20rpx;
  font-size: 40rpx;
  color: #999;
  padding: 10rpx;
}

.empty-tip {
  padding: 30rpx;
  text-align: center;
  color: #999;
  font-size: 26rpx;
}
/* 在 style 底部添加 */

/* --- 搜索框样式 --- */
.store-search-box {
  padding: 10rpx 20rpx;
  background-color: #fff;
  border-bottom: 1rpx solid #f0f0f0;
}

.search-input-wrapper {
  background-color: #f5f5f5;
  border-radius: 30rpx;
  padding: 0 20rpx;
  height: 70rpx;
  display: flex;
  align-items: center;
}

.search-icon { 
  font-size: 28rpx; 
  margin-right: 10rpx; 
  color: #999;
}

.search-input { 
  flex: 1; 
  font-size: 28rpx; 
  height: 100%; 
}

.clear-icon { 
  font-size: 32rpx; 
  color: #999; 
  padding: 10rpx; 
}

/* --- 搜索结果列表样式 --- */
.search-result-list {
  flex: 1; /* 占据剩余空间 */
  height: 0; /* 配合 flex:1 */
  background-color: #fff;
}

.search-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24rpx 30rpx !important; /* 覆盖 .store-item 的 padding */
}

.item-main {
  font-size: 28rpx;
  color: #333;
}

.item-sub {
  font-size: 24rpx;
  color: #999;
}
</style>