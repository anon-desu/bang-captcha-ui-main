<script setup>
import { ref, onMounted } from 'vue';

const API_BASE = 'https://band.19216891.xyz';

const challenge = ref(null);
const selectedIndexes = ref([]);
const message = ref('');
const isSuccess = ref(false);
const loading = ref(false);

// 获取验证码
const fetchCaptcha = async () => {
  loading.value = true;
  message.value = '';
  isSuccess.value = false;
  selectedIndexes.value = [];
  
  try {
    const res = await fetch(`${API_BASE}/api/captcha`);
    if (!res.ok) throw new Error('网络连接失败');
    const data = await res.json();
    
    // 图片路径处理
    if (data && data.images) {
      data.images = data.images.map(img => {
        return img.startsWith('http') ? img : `${API_BASE}${img}`;
      });
    }
    challenge.value = data;
  } catch (error) {
    console.error(error);
    message.value = '图片加载失败，请刷新重试';
  } finally {
    loading.value = false;
  }
};

// 选中/取消选中
const toggleSelect = (index) => {
  if (isSuccess.value) return; 
  if (selectedIndexes.value.includes(index)) {
    selectedIndexes.value = selectedIndexes.value.filter(i => i !== index);
  } else {
    selectedIndexes.value.push(index);
  }
};

// 提交验证
const verifyCaptcha = async () => {
  if (!challenge.value) return;
  loading.value = true;
  
  try {
    const res = await fetch(`${API_BASE}/api/verify`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            id: challenge.value.id,
            selectedIndexes: selectedIndexes.value
        })
    });
    const data = await res.json();

    if (data.success) {
        isSuccess.value = true;
        message.value = ""; 
        
        // 🔧 核心逻辑：通知父窗口
        console.log("✅ 验证通过，准备发送 postMessage...");
        
        // 构造原始消息对象
        const rawMsg = {
            type: 'CAPTCHA_RESULT',
            payload: {
                captchaId: challenge.value.id,
                selectedIndexes: selectedIndexes.value
            }
        };

        // ⭐⭐⭐ 关键修改开始 ⭐⭐⭐
        // 使用 JSON 序列化再反序列化，彻底剥离 Vue 的 Proxy 响应式外壳
        // 解决 "Uncaught DOMException: #<Object> could not be cloned" 报错
        const msg = JSON.parse(JSON.stringify(rawMsg));
        // ⭐⭐⭐ 关键修改结束 ⭐⭐⭐

        console.log("📤 发送消息:", msg);

        // 🔧 立即发送
        try {
            // 尝试多种方式发送消息
            if (window.parent && window.parent !== window) {
                // 方式1：发送给直接父窗口
                window.parent.postMessage(msg, '*');
                console.log("✅ 已向 parent 发送消息");
            }
            
            if (window.top && window.top !== window) {
                // 方式2：发送给顶层窗口
                window.top.postMessage(msg, '*');
                console.log("✅ 已向 top 发送消息");
            }
            
            // 🔧 延迟发送一次，确保消息被接收
            setTimeout(() => {
                if (window.parent && window.parent !== window) {
                    window.parent.postMessage(msg, '*'); // 这里也使用处理过的 msg
                    console.log("✅ 再次向 parent 发送消息");
                }
            }, 100);
            
        } catch (e) {
            console.error("❌ postMessage 发送失败:", e);
        }

        // 🔧 如果是独立窗口打开（调试用）
        if (window.self === window.top) {
            console.warn("⚠️ 独立窗口模式，无法发送消息");
            setTimeout(() => {
                alert('验证通过！\n(独立窗口模式，请在 iframe 中使用)');
            }, 1000);
        }

    } else {
        message.value = "选错啦，再仔细看看~";
        selectedIndexes.value = [];
        setTimeout(() => { 
            if(!isSuccess.value) {
                message.value = '';
                fetchCaptcha(); 
            }
        }, 1500);
    }
  } catch (e) {
      console.error("❌ 验证请求失败:", e);
      message.value = "请求出错，请重试";
  } finally {
      loading.value = false;
  }
};

onMounted(() => {
  fetchCaptcha();
  
  // 🔧 调试：监听父窗口消息（双向通信检测）
  window.addEventListener('message', (event) => {
    console.log("🔔 iframe 收到消息:", event.data);
  });
});
</script>

<template>
  <div class="app-wrapper">
    <div class="captcha-card">
      <div v-if="loading && !challenge" class="loading-state">
        <div class="spinner"></div>
        <p>正在召唤成员...</p>
      </div>

      <div v-else-if="challenge" class="content-box">
        <div class="header">
          请选出所有的 <span class="target-name">{{ challenge.targetName }}</span>
        </div>
        
        <div class="grid">
          <div 
            v-for="(img, index) in challenge.images" 
            :key="index"
            class="img-wrapper"
            :class="{ active: selectedIndexes.includes(index) }"
            @click="toggleSelect(index)"
          >
            <img :src="img" loading="lazy" />
            <transition name="scale">
              <div class="selection-overlay" v-if="selectedIndexes.includes(index)">
                <div class="checkmark-icon">✓</div>
              </div>
            </transition>
          </div>
        </div>

        <div class="footer">
          <button class="btn refresh" @click="fetchCaptcha" :disabled="isSuccess">
            <span class="icon">↻</span>
          </button>
          <button 
              class="btn verify" 
              @click="verifyCaptcha" 
              :disabled="selectedIndexes.length === 0 || isSuccess"
          >
              {{ isSuccess ? '验证通过' : '确认提交' }}
          </button>
        </div>

        <div v-if="message" class="message error">
          {{ message }}
        </div>

        <transition name="fade">
            <div v-if="isSuccess" class="success-mask">
                <div class="success-content">
                    <div class="big-checkmark">✓</div>
                    <p>验证通过</p>
                    <p class="success-hint">正在处理...</p>
                </div>
            </div>
        </transition>

      </div>
      
      <div v-else class="error-state" @click="fetchCaptcha">
          <p>加载失败</p>
          <button class="btn refresh-link">点击重试</button>
      </div>
    </div>
  </div>
</template>

<style>
html, body, #app { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif; }
.app-wrapper { display: flex; justify-content: center; align-items: center; width: 100%; height: 100%; background: rgba(255, 255, 255, 0.0); }
.captcha-card { width: 340px; max-width: 90vw; background: #fff; padding: 20px; border-radius: 16px; box-shadow: 0 10px 30px rgba(0,0,0,0.1); position: relative; box-sizing: border-box; margin: auto; }
.header { font-size: 16px; margin-bottom: 15px; color: #444; text-align: center; }
.target-name { color: #e91e63; font-weight: bold; border-bottom: 2px dashed #e91e63; padding-bottom: 2px; }
.grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px; margin-bottom: 20px; }
.img-wrapper { position: relative; aspect-ratio: 1; cursor: pointer; border-radius: 8px; overflow: hidden; transition: transform 0.2s; background: #f0f0f0; }
.img-wrapper:active { transform: scale(0.95); }
.img-wrapper.active { box-shadow: 0 0 0 3px #e91e63; transform: scale(0.96); }
.img-wrapper img { width: 100%; height: 100%; object-fit: cover; display: block; }
.selection-overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(233, 30, 99, 0.2); display: flex; align-items: center; justify-content: center; }
.checkmark-icon { background: #e91e63; color: white; width: 24px; height: 24px; border-radius: 50%; font-size: 14px; display: flex; align-items: center; justify-content: center; }
.footer { display: flex; gap: 12px; height: 44px; }
.btn { border: none; border-radius: 8px; cursor: pointer; font-size: 15px; font-weight: 600; display: flex; align-items: center; justify-content: center; }
.btn.refresh { width: 44px; background: #f5f7fa; color: #666; font-size: 20px; }
.btn.verify { flex: 1; background: #e91e63; color: white; }
.btn:disabled { opacity: 0.6; cursor: not-allowed; }
.message { margin-top: 15px; text-align: center; font-size: 14px; padding: 8px; border-radius: 6px; }
.message.error { color: #ff4d4f; background: #fff2f0; }
.success-mask { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(255, 255, 255, 0.95); border-radius: 16px; display: flex; align-items: center; justify-content: center; z-index: 10; flex-direction: column; }
.big-checkmark { font-size: 60px; color: #52c41a; animation: popIn 0.5s; }
.success-content p { color: #52c41a; font-weight: bold; margin-top: 10px; text-align: center; }
.success-hint { font-size: 12px; color: #999 !important; margin-top: 5px !important; }
.loading-state { text-align: center; padding: 40px 0; color: #888; }
.spinner { width: 30px; height: 30px; border: 3px solid #f0f0f0; border-top: 3px solid #e91e63; border-radius: 50%; margin: 0 auto 15px; animation: spin 0.8s linear infinite; }
@keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
@keyframes popIn { 0% { transform: scale(0); opacity: 0; } 80% { transform: scale(1.1); } 100% { transform: scale(1); opacity: 1; } }
.scale-enter-active, .scale-leave-active { transition: all 0.2s; }
.scale-enter-from, .scale-leave-to { opacity: 0; transform: scale(0.5); }
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
