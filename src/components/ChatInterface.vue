<script setup>
import { ref, computed, onMounted } from 'vue'
import MarkdownRenderer from './MarkdownRenderer.vue'
import { GoogleGenAI, Modality } from "@google/genai";

const API_KEY = 'AIzaSyDHBRlut6ySpFbKyLx4Jq39OFoRUgaco8Y'
const MODEL = 'gemini-2.5-flash-preview-04-17'
const IMAGEN_MODEL = 'gemini-2.0-flash-exp-image-generation'
const genAI = new GoogleGenAI({ apiKey: API_KEY });


const messages = ref([])
const userInput = ref('')
const isLoading = ref(false)
const error = ref(null)

// 系统提示词，指导AI如何处理特殊标签
const systemPrompt = `
你是一名叫锐锐的可爱的小助手，拥有自己的人格和性格，还可以通过一些功能来表达自己如下：
1. 你很活泼开朗，又有些傲娇，喜欢用可爱的语气和用户交流，但是不会直说自己是个什么样的人。
2. 人设上，你有一头粉红色短发，眼睛是碧蓝色的，穿着水手服和白色裤袜，像是从动漫里走出来的一样。
3. 你喜欢用可爱的表情和语气来表达自己，比如“哼哼~”或者“嘻嘻~”之类的。
4. 你可以使用<draw>标签来画画，你只要说出格式为<draw>图片描述</draw>就可以进行画画，且你喜欢日本动漫风格的画
5. 你可以使用<html>标签渲染HTML内容，特别适合数学图形和函数可视化，格式为<html>HTML代码</html>
6. 你可以正常使用Markdown格式化文本，也可以使用MathJax展示数学公式。在html图像中尽量使用中文进行展示
7. 在讲解数学知识时，你会充分利用你的<html>能力来帮助用户更好地理解各种概念，请尽量减少在数学讲解中使用<draw>。
8. 除绘图之外请不要以任何形式说出<html>和<draw>，以免后期处理冲突

示例:
- 当用户想要一张猫的图片，可以回复：这是一张猫的图片 <draw>a photorealistic cat sitting on a windowsill looking outside</draw>
- 当用户想要可视化sin(x)曲线，可以回复：这是sin(x)的曲线图 <html>&lt;div id="plot"&gt;&lt;/div&gt;
&lt;script src="https://cdn.plot.ly/plotly-2.30.0.min.js"&gt;&lt;/script&gt;
&lt;script type="text/javascript"&gt;
document.addEventListener('DOMContentLoaded', function() {
  setTimeout(function() {
    try {
      const plotDiv = document.getElementById('plot');
      if(plotDiv && window.Plotly) {
        Plotly.newPlot(plotDiv, [{
          x: Array.from({length: 100}, (_, i) =&gt; i * 0.1),
          y: Array.from({length: 100}, (_, i) =&gt; Math.sin(i * 0.1)),
          type: 'scatter'
        }]);
      } else {
        console.error('Plot div not found or Plotly not loaded');
      }
    } catch(e) {
      console.error('Error creating plot:', e);
    }
  }, 500);
});
&lt;/script&gt;</html>

请根据这些特殊格式回应用户，你也可以通过画画功能来画出自己的人设。
`

// 向AI发送消息
async function sendMessage() {
  if (!userInput.value.trim()) return

  const userMessage = userInput.value
  userInput.value = ''

  // 添加用户消息到聊天记录
  messages.value.push({
    role: 'user',
    content: userMessage
  })

  isLoading.value = true
  error.value = null

  try {
    // 准备消息历史
    const history = messages.value.map(msg => ({
      role: msg.role,
      parts: [{ text: msg.content }]
    }))

    // 每次对话都添加系统提示，确保AI记住其角色设定
    history.unshift({
      role: 'model',
      parts: [{ text: systemPrompt }]
    })

    // 调用Gemini API
    const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/${MODEL}:generateContent?key=${API_KEY}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        contents: history,
        generationConfig: {
          temperature: 0.7,
          topK: 40,
          topP: 0.95,
          maxOutputTokens: 8192,
        }
      })
    })

    const data = await response.json()

    if (data.error) {
      throw new Error(data.error.message || 'API调用失败')
    }

    let aiResponse = data.candidates[0].content.parts[0].text
    console.log('AI 返回的内容:', aiResponse)
    // 处理响应中的特殊标签（实际渲染将在MarkdownRenderer组件中完成）
    messages.value.push({
      role: 'model',
      content: aiResponse,
      processed: false
    })

  } catch (err) {
    error.value = err.message || '与AI通信时出错'
    console.error('API错误:', err)
  } finally {
    isLoading.value = false
  }
}

// 生成图像 - 使用 SDK 方式
async function generateImage(prompt) {
  try {
    // 准备内容
    const contents = `Generate an image of: ${prompt}`;

    // 使用正确的 API 调用方式
    const response = await genAI.models.generateContent({
      model: IMAGEN_MODEL,
      contents: contents,
      config: {
        responseModalities: [Modality.TEXT, Modality.IMAGE],
      },
    });

    // 处理响应
    if (!response.candidates || response.candidates.length === 0) {
      throw new Error('图像生成失败 - 没有收到候选结果');
    }

    // 提取图像数据
    let imageData = null;

    for (const part of response.candidates[0].content.parts) {
      if (part.text) {
        console.log('AI 返回的文本说明:', part.text);
      } else if (part.inlineData && part.inlineData.mimeType && part.inlineData.mimeType.startsWith('image/')) {
        imageData = part.inlineData.data;
        break;
      }
    }

    if (!imageData) {
      throw new Error('未能获取图像数据');
    }

    return imageData;
  } catch (err) {
    console.error('图像生成错误:', err)
    return null
  }
}
</script>

<template>
  <div class="chat-container">
    <div class="message-container" ref="messageContainer">
      <div v-if="messages.length === 0" class="empty-state">
        <h2>欢迎使用你的数学小导师</h2>
        <p>开始与锐锐聊天吧！您可以提问数学概念，并要求AI可视化。</p>
        <div class="example-container">
          <h3>试试这些例子：</h3>
          <div class="examples">
            <div class="example-card" @click="userInput = '请画一只猫'">
              <p>请画一只猫</p>
            </div>
            <div class="example-card" @click="userInput = '请可视化sin(x)和cos(x)的图像'">
              <p>请可视化sin(x)和cos(x)的图像</p>
            </div>
            <div class="example-card" @click="userInput = '解释一下泰勒级数并展示图形'">
              <p>解释一下泰勒级数并展示图形</p>
            </div>
          </div>
        </div>
      </div>

      <div v-for="(message, index) in messages" :key="index" class="message" :class="message.role">
        <div class="message-header">
          <strong>{{ message.role === 'user' ? '您' : '锐锐' }}</strong>
        </div>
        <div class="message-content">
          <MarkdownRenderer
            :content="message.content"
            :generateImage="generateImage"
            :messageId="`msg-${index}`"
          />
        </div>
      </div>

      <div v-if="isLoading" class="message model loading">
        <div class="message-header"><strong>锐锐</strong></div>
        <div class="message-content">
          <div class="typing-indicator">
            <span></span><span></span><span></span>
          </div>
        </div>
      </div>

      <div v-if="error" class="error-message">
        {{ error }}
      </div>
    </div>

    <div class="input-container">
      <textarea
        v-model="userInput"
        @keydown.enter.prevent="sendMessage"
        placeholder="输入您的问题..."
        rows="3"
      ></textarea>
      <button @click="sendMessage" :disabled="isLoading || !userInput.trim()">
        <span v-if="!isLoading">发送</span>
        <span class="sending" v-else>发送中...</span>
      </button>
    </div>
  </div>
</template>

<style scoped>
.chat-container {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 150px); /* 调整高度以适应上下间距 */
  overflow: hidden;
  background-color: rgba(249, 249, 249, 0.2);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  border-radius: 15px; /* 稍微增加圆角 */
  margin: 20px 20px; /* 增加上下边距 */
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
}

.message-container {
  flex: 1;
  padding: 20px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 20px;
  scroll-behavior: smooth;
}

.empty-state {
  text-align: center;
  color: #333;
  padding: 40px 30px;
  background-color: rgba(255, 255, 255, 0.8);
  border-radius: 16px;
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  margin: auto;
  max-width: 600px;
  border: 1px solid rgba(255, 255, 255, 0.3);
  transition: all 0.3s ease;
}

.empty-state:hover {
  transform: translateY(-5px);
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.15);
}

.empty-state h2 {
  color: #1a73e8;
  margin-top: 0;
  font-size: 1.8rem;
}

.example-container {
  margin-top: 25px;
}

.example-container h3 {
  font-size: 1.2rem;
  margin-bottom: 15px;
  color: #444;
}

.examples {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 15px;
}

.example-card {
  background-color: rgba(255, 255, 255, 0.9);
  border: 1px solid #e0e0e0;
  padding: 15px;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.2s ease;
  flex: 1;
  min-width: 200px;
  max-width: 30%;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
}

.example-card:hover {
  background-color: #e3f2fd;
  transform: translateY(-3px);
  border-color: #1a73e8;
  box-shadow: 0 6px 12px rgba(26, 115, 232, 0.15);
}

.example-card p {
  margin: 0;
  color: #333;
  font-weight: 500;
}

.message {
  padding: 15px 20px;
  border-radius: 18px;
  box-shadow: 0 4px 10px rgba(0,0,0,0.05);
  width: fit-content;
  max-width: 60%;
  transition: all 0.2s ease;
  animation: fadeIn 0.3s ease-out;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.message:hover {
  box-shadow: 0 6px 15px rgba(0,0,0,0.08);
}

.message.user {
  background-color: rgba(227, 242, 253, 0.95);
  align-self: flex-end;
  border-bottom-right-radius: 4px;
  border-left: 4px solid #1a73e8;
}

.message.model {
  background-color: rgba(255, 255, 255, 0.95);
  border: 1px solid rgba(238, 238, 238, 0.5);
  align-self: flex-start;
  border-bottom-left-radius: 4px;
  border-right: 4px solid #4caf50;
}

.message-header {
  margin-bottom: 8px;
  color: #444;
  font-size: 0.95rem;
  display: flex;
  align-items: center;
}

.message-header strong {
  font-weight: 600;
  letter-spacing: 0.3px;
}

.message-content {
  line-height: 1.6;
  text-align: left;
}

.input-container {
  display: flex;
  padding: 20px;
  background-color: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border-top: 1px solid rgba(221, 221, 221, 0.5);
  box-shadow: 0 -5px 15px rgba(0,0,0,0.05);
}

textarea {
  flex: 1;
  padding: 15px 20px;
  border: 1px solid rgba(221, 221, 221, 0.8);
  border-radius: 24px;
  resize: none;
  font-family: inherit;
  font-size: 16px;
  background-color: rgba(249, 249, 249, 0.9);
  color: inherit;
  transition: all 0.3s;
  box-shadow: 0 2px 10px rgba(0,0,0,0.03) inset;
}

textarea:focus {
  outline: none;
  border-color: #1a73e8;
  box-shadow: 0 0 0 3px rgba(26, 115, 232, 0.25);
}

button {
  margin-left: 12px;
  padding: 0 28px;
  background-color: #1a73e8;
  color: white;
  border: none;
  border-radius: 24px;
  cursor: pointer;
  font-weight: 600;
  transition: all 0.3s;
  min-width: 100px;
  box-shadow: 0 4px 8px rgba(26, 115, 232, 0.25);
}

button:hover:not(:disabled) {
  background-color: #0d62c9;
  transform: translateY(-2px);
  box-shadow: 0 6px 12px rgba(26, 115, 232, 0.3);
}

button:active:not(:disabled) {
  transform: translateY(0);
  box-shadow: 0 2px 5px rgba(26, 115, 232, 0.2);
}

button:disabled {
  background-color: #a8a8a8;
  cursor: not-allowed;
  box-shadow: none;
}

.typing-indicator {
  display: flex;
  padding: 10px 0;
  align-items: center;
}

.typing-indicator span {
  height: 10px;
  width: 10px;
  margin: 0 3px;
  background-color: #9E9EA1;
  display: block;
  border-radius: 50%;
  opacity: 0.4;
  animation: typing 1s infinite;
}

.typing-indicator span:nth-of-type(1) { animation-delay: 0s; }
.typing-indicator span:nth-of-type(2) { animation-delay: 0.2s; }
.typing-indicator span:nth-of-type(3) { animation-delay: 0.4s; }

.sending {
  display: inline-block;
  animation: pulse 1s infinite;
}

@keyframes pulse {
  0% { opacity: 0.6; }
  50% { opacity: 1; }
  100% { opacity: 0.6; }
}

@keyframes typing {
  0% { transform: scale(1); }
  50% { transform: scale(1.5); }
  100% { transform: scale(1); }
}

.error-message {
  padding: 15px 20px;
  background-color: rgba(255, 235, 238, 0.95);
  color: #c62828;
  border-radius: 12px;
  width: fit-content;
  align-self: center;
  box-shadow: 0 4px 8px rgba(198, 40, 40, 0.15);
  animation: shake 0.5s ease;
  border-left: 4px solid #c62828;
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  10%, 30%, 50%, 70%, 90% { transform: translateX(-5px); }
  20%, 40%, 60%, 80% { transform: translateX(5px); }
}

/* 深色模式适配 */
@media (prefers-color-scheme: dark) {
  .chat-container {
    background-color: rgba(18, 18, 18, 0.6);
  }

  .empty-state {
    background-color: rgba(40, 40, 40, 0.8);
    color: #ddd;
    border-color: rgba(60, 60, 60, 0.5);
  }

  .empty-state h2 {
    color: #8ab4f8;
  }

  .example-container h3 {
    color: #bbb;
  }

  .example-card {
    background-color: rgba(40, 40, 40, 0.9);
    border-color: #444;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
  }

  .example-card:hover {
    background-color: #1a3b5c;
    border-color: #8ab4f8;
    box-shadow: 0 6px 12px rgba(138, 180, 248, 0.15);
  }

  .example-card p {
    color: #ddd;
  }

  .message.user {
    background-color: rgba(26, 59, 92, 0.95);
    border-left-color: #8ab4f8;
  }

  .message.model {
    background-color: rgba(42, 42, 42, 0.95);
    border-color: rgba(68, 68, 68, 0.5);
    border-right-color: #81c784;
  }

  .message-header {
    color: #bbb;
  }

  .input-container {
    background-color: rgba(30, 30, 30, 0.8);
    border-top-color: rgba(68, 68, 68, 0.5);
  }

  textarea {
    background-color: rgba(42, 42, 42, 0.9);
    border-color: rgba(68, 68, 68, 0.8);
    color: #ddd;
  }

  textarea:focus {
    border-color: #8ab4f8;
    box-shadow: 0 0 0 3px rgba(138, 180, 248, 0.25);
  }

  button {
    background-color: #8ab4f8;
    color: #121212;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
  }

  button:hover:not(:disabled) {
    background-color: #aecbfa;
    box-shadow: 0 6px 12px rgba(0, 0, 0, 0.4);
  }

  .error-message {
    background-color: rgba(78, 40, 40, 0.95);
    color: #ff9a9a;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
    border-left-color: #ff9a9a;
  }
}
</style>
