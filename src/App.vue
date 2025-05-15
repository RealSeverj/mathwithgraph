<script setup>
import { ref, onMounted } from 'vue'
import ChatInterface from './components/ChatInterface.vue'

const appTitle = ref('你的数学小导师')
const showDetails = ref(false)

// 切换详情弹窗的显示状态
function toggleDetails() {
  showDetails.value = !showDetails.value
}

// 关闭详情弹窗
function closeDetails() {
  showDetails.value = false
}

// 点击弹窗外部时关闭弹窗
function handleOutsideClick(event) {
  if (showDetails.value && event.target.classList.contains('details-modal')) {
    closeDetails()
  }
}
</script>

<template>
  <div class="app-container">
    <div class="background-overlay"></div>
    <header>
      <h1>{{ appTitle }}</h1>
      <p class="subtitle">基于Gemini API</p>
      <button class="info-button" @click="toggleDetails" title="查看详情">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <circle cx="12" cy="12" r="10"></circle>
          <line x1="12" y1="16" x2="12" y2="12"></line>
          <line x1="12" y1="8" x2="12.01" y2="8"></line>
        </svg>
      </button>
    </header>
    <main>
      <ChatInterface />
    </main>
    
    <!-- 详情弹窗 -->
    <div v-if="showDetails" class="details-modal" @click="handleOutsideClick">
      <div class="details-content">
        <button class="close-button" @click="closeDetails">×</button>
        <h3>关于应用</h3>
        <p>MathwithGraph 是一个基于 Gemini API 的数学学习助手，旨在通过可视化和交互方式帮助用户理解数学概念。</p>
        <p>无论是函数图像，还是几何概念，锐锐都能为您提供直观的理解方式。</p>
        <div class="details-footer">
          <p>© 2025 MathwithGraph</p>
          <p>版本: 1.0.0</p>
          <p>开发者: Severj</p>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app-container {
  width: 100%;
  min-height: 100vh;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  font-family: Arial, sans-serif;
  position: relative;
  overflow: hidden;
}

.background-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image: url('./assets/background.jpg');
  background-size: cover;
  background-position: center;
  background-attachment: fixed;
  filter: brightness(0.8);
  z-index: -1;
}

header {
  text-align: center;
  padding: 20px 0;
  background-color: rgba(245, 245, 245, 0.85);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  box-shadow: 0 2px 15px rgba(0,0,0,0.1);
  position: relative;
  z-index: 1;
}

h1 {
  color: #1a73e8;
  margin: 0;
  font-size: 2.2rem;
  text-shadow: 0 2px 4px rgba(250, 250, 250, 0.1);
  font-weight: 700;
  letter-spacing: -0.5px;
}

.subtitle {
  color: #444;
  margin: 5px 0 0;
  font-size: 1rem;
  font-weight: 500;
}

main {
  flex: 1;
  padding: 0;
  display: flex;
  flex-direction: column;
  position: relative;
  z-index: 1;
  margin-bottom: 20px; /* 为了在没有底栏时增加一些底部空间 */
}

/* 信息按钮样式 */
.info-button {
  position: absolute;
  top: 20px;
  right: 20px;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  border: none;
  cursor: pointer;
  color: #1a73e8;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  transition: all 0.3s ease;
  padding: 0;
}

.info-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 15px rgba(0,0,0,0.15);
  background-color: rgba(255, 255, 255, 0.9);
}

.info-button:active {
  transform: translateY(0);
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

/* 详情弹窗样式 */
.details-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(3px);
  -webkit-backdrop-filter: blur(3px);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  opacity: 0;
  animation: fadeIn 0.3s ease forwards;
}

.details-content {
  width: 90%;
  max-width: 500px;
  background-color: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border-radius: 15px;
  padding: 30px;
  position: relative;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
  transform: translateY(20px);
  animation: slideUp 0.4s ease forwards;
}

.close-button {
  position: absolute;
  top: 15px;
  right: 15px;
  width: 30px;
  height: 30px;
  border-radius: 50%;
  background-color: rgba(0, 0, 0, 0.05);
  border: none;
  font-size: 20px;
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  color: #666;
  transition: all 0.3s;
  padding: 0; /* 确保没有内边距影响形状 */
  line-height: 0; /* 修复行高导致的形状问题 */
  text-align: center; /* 确保文本居中 */
  font-family: Arial, sans-serif; /* 确保字体一致 */
}

.close-button:hover {
  background-color: rgba(0, 0, 0, 0.1);
  color: #333;
}

.details-content h3 {
  color: #1a73e8;
  margin-top: 0;
  font-size: 1.6rem;
  margin-bottom: 20px;
}

.details-content p {
  color: #444;
  line-height: 1.6;
  margin-bottom: 15px;
}

.details-footer {
  margin-top: 30px;
  padding-top: 20px;
  border-top: 1px solid rgba(0, 0, 0, 0.1);
  font-size: 0.9em;
  color: #666;
}

.details-footer p {
  margin: 5px 0;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes slideUp {
  from { transform: translateY(20px); opacity: 0; }
  to { transform: translateY(0); opacity: 1; }
}

/* 深色模式适配 */
@media (prefers-color-scheme: dark) {
  .background-overlay {
    filter: brightness(0.4) saturate(0.8);
  }
  
  header {
    background-color: rgba(26, 26, 26, 0.7);
    box-shadow: 0 2px 15px rgba(0,0,0,0.3);
  }
  
  h1 {
    color: #8ab4f8;
    text-shadow: 0 2px 4px rgba(0,0,0,0.2);
  }
  
  .subtitle {
    color: #ddd;
  }
  
  .info-button {
    background-color: rgba(42, 42, 42, 0.8);
    color: #8ab4f8;
    box-shadow: 0 2px 10px rgba(0,0,0,0.2);
  }
  
  .info-button:hover {
    background-color: rgba(60, 60, 60, 0.9);
    box-shadow: 0 4px 15px rgba(0,0,0,0.3);
  }
  
  .details-modal {
    background-color: rgba(0, 0, 0, 0.6);
  }
  
  .details-content {
    background-color: rgba(30, 30, 30, 0.95);
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.4);
  }
  
  .close-button {
    background-color: rgba(255, 255, 255, 0.1);
    color: #aaa;
  }
  
  .close-button:hover {
    background-color: rgba(255, 255, 255, 0.15);
    color: #ddd;
  }
  
  .details-content h3 {
    color: #8ab4f8;
  }
  
  .details-content p {
    color: #bbb;
  }
  
  .details-footer {
    border-top-color: rgba(255, 255, 255, 0.1);
    color: #999;
  }
}
</style>