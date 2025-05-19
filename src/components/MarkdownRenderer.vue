<script setup>
import { ref, onMounted, watch } from 'vue'
import { marked } from 'marked'
import DOMPurify from 'dompurify'

const props = defineProps({
  content: {
    type: String,
    required: true
  },
  generateImage: {
    type: Function,
    required: true
  },
  messageId: { // 添加消息ID属性以区分不同消息
    type: String,
    required: true
  }
})

const renderedContent = ref('')
const processingComplete = ref(false)
const imageElements = ref([]) // 存储图像元素引用
const contentCopy = ref('') // 存储处理中的内容副本

// 配置DOMPurify允许data URLs
onMounted(() => {
  // 配置DOMPurify允许data URLs
  DOMPurify.addHook('afterSanitizeAttributes', function(node) {
    // 对于图片标签，允许data URL
    if (node.tagName === 'IMG' && node.getAttribute('src')) {
      const src = node.getAttribute('src');
      if (src.startsWith('data:image/')) {
        // 允许data URL
        return node;
      }
    }
  });

  // 加载MathJax库并初始化
  if (!window.MathJax) {
    const script = document.createElement('script')
    script.src = 'https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js'
    script.async = true
    
    // 配置MathJax
    window.MathJax = {
      tex: {
        inlineMath: [['$', '$'], ['\\(', '\\)']],
        displayMath: [['$$', '$$'], ['\\[', '\\]']],
        processEscapes: true
      },
      svg: {
        fontCache: 'global'
      }
    }
    
    document.head.appendChild(script)
  }
})

// 渲染数学公式
function renderMathJax() {
  if (window.MathJax) {
    if (window.MathJax.typesetPromise) {
      window.MathJax.typesetPromise()
        .catch((err) => console.error('MathJax处理失败:', err))
    } else if (window.MathJax.Hub) {
      // MathJax 2.x 版本
      window.MathJax.Hub.Queue(['Typeset', window.MathJax.Hub])
    }
  }
}

// 处理并渲染内容
async function renderContent() {
  // 创建一个内容的副本，以便在处理过程中修改
  let content = props.content
  contentCopy.value = content
  processingComplete.value = false
  
  // 清空图像元素缓存
  imageElements.value = []
  
  // 处理<draw>标签
  const drawRegex = /<draw>(.*?)<\/draw>/gs
  const drawMatches = [...content.matchAll(drawRegex)]
  
  // 创建一个映射表来存储每个占位符ID与其对应的替换内容
  const placeholderMap = new Map()
  
  // 首先，为所有的<draw>标签生成唯一ID和占位符
  for (let i = 0; i < drawMatches.length; i++) {
    const fullMatch = drawMatches[i][0]
    const promptText = drawMatches[i][1]
    
    // 生成真正唯一的ID，包含消息ID和索引
    const imageId = `img-${props.messageId}-${i}-${Date.now()}`
    const placeholder = `<div id="${imageId}" class="image-placeholder loading">正在生成图像...</div>`
    
    // 存储映射关系
    placeholderMap.set(fullMatch, {
      id: imageId,
      placeholder: placeholder,
      promptText: promptText
    })
    
    // 替换内容中的标签为占位符
    contentCopy.value = contentCopy.value.replace(fullMatch, placeholder)
  }
  
  // 更新渲染内容，显示所有占位符
  renderedContent.value = await parseMarkdown(contentCopy.value)
  
  // 处理<html>标签
  const htmlRegex = /<html>(.*?)<\/html>/gs
  const htmlMatches = [...contentCopy.value.matchAll(htmlRegex)]
  
  // 同样为所有HTML标签生成唯一ID和占位符
  for (let i = 0; i < htmlMatches.length; i++) {
    const fullMatch = htmlMatches[i][0]
    const htmlContent = htmlMatches[i][1]
    
    // 生成真正唯一的ID
    const divId = `html-${props.messageId}-${i}-${Date.now()}`
    const placeholder = `<div id="${divId}" class="html-container"></div>`
    
    // 替换HTML标签为占位符
    contentCopy.value = contentCopy.value.replace(fullMatch, placeholder)
    
    // 更新渲染内容
    renderedContent.value = await parseMarkdown(contentCopy.value)
    
    // 处理HTML内容
    setTimeout(() => {
      insertHtmlToDom(divId, htmlContent)
    }, 500)
  }
  
  // 如果没有特殊标签，直接渲染Markdown
  if (drawMatches.length === 0 && htmlMatches.length === 0) {
    renderedContent.value = await parseMarkdown(content)
    // 渲染公式
    setTimeout(renderMathJax, 100)
  }
  
  // 现在开始处理每个图像
  for (const [fullMatch, data] of placeholderMap.entries()) {
    try {
      // 已经有占位符了，现在生成图像
      const imageData = await props.generateImage(data.promptText)
      
      if (imageData) {
        // 存储图像信息
        imageElements.value.push({
          id: data.id,
          data: imageData,
          alt: data.promptText
        })
        
        // 插入图像
        setTimeout(() => {
          insertImageToDom(data.id, imageData, data.promptText)
        }, 100)
      } else {
        // 替换占位符为错误信息
        const errorDiv = document.getElementById(data.id)
        if (errorDiv) {
          errorDiv.className = 'image-error'
          errorDiv.textContent = `图像生成失败: "${data.promptText}"`
        }
      }
    } catch (error) {
      console.error('处理图像标签时出错:', error)
    }
  }
  
  // 最后渲染公式
  setTimeout(renderMathJax, 300)
  processingComplete.value = true
}

// 解析Markdown
async function parseMarkdown(text) {
  return DOMPurify.sanitize(marked.parse(text), {
    ADD_TAGS: ['div', 'style', 'img'],
    ADD_ATTR: ['id', 'class', 'style', 'src']
  })
}

// 直接在DOM中插入图像元素
function insertImageToDom(id, imageData, altText) {
  const container = document.getElementById(id)
  if (container) {
    // 清除加载指示器相关类
    container.classList.remove('loading', 'image-placeholder')
    container.classList.add('image-container')
    container.textContent = ''
    
    // 创建图像元素
    const img = document.createElement('img')
    img.src = `data:image/jpeg;base64,${imageData}`
    img.alt = altText
    img.className = 'generated-image'
    
    // 添加到容器中
    container.appendChild(img)
  } else {
    console.error('找不到图像容器:', id)
  }
}

// 在DOM中插入HTML内容
function insertHtmlToDom(id, htmlContent) {
  try {
    const container = document.getElementById(id)
    if (container) {
      // 为了解决Plotly的加载问题，先提取所有script标签
      let scriptContents = []
      let mainContent = htmlContent
      
      // 分离非脚本内容和脚本内容
      const scriptRegex = /<script\b[^>]*>([\s\S]*?)<\/script>/gi
      mainContent = htmlContent.replace(scriptRegex, (match, scriptContent) => {
        const scriptTag = document.createElement('div')
        scriptTag.innerHTML = match
        const script = scriptTag.firstChild
        
        // 收集脚本内容和属性
        scriptContents.push({
          src: script.getAttribute('src'),
          type: script.getAttribute('type'),
          content: scriptContent
        })
        
        return '' // 从HTML中删除script标签
      })
      
      // 首先设置非脚本HTML内容
      const sanitizedHtml = DOMPurify.sanitize(mainContent, {
        ADD_TAGS: ['div', 'style', 'svg', 'path', 'rect', 'circle', 'line', 'polyline', 'polygon', 'g', 'defs', 'marker'],
        ADD_ATTR: ['id', 'class', 'style', 'viewBox', 'd', 'cx', 'cy', 'r', 'x', 'y', 'width', 'height', 
                   'fill', 'stroke', 'stroke-width', 'points', 'transform', 'font-size', 'font-family', 'text-anchor']
      })
      container.innerHTML = sanitizedHtml
      
      // 创建一个独立的处理上下文，避免多个HTML块之间的冲突
      const containerContext = {
        container: container,
        id: id,
        completedScripts: 0,
        totalScripts: scriptContents.length
      }
      
      // 处理外部脚本和内联脚本
      const loadExternalScripts = (context, index = 0) => {
        if (index >= context.totalScripts) return
        
        const scriptInfo = scriptContents[index]
        const newScript = document.createElement('script')
        
        if (scriptInfo.type) newScript.type = scriptInfo.type
        
        // 为每个脚本添加唯一标识，防止冲突
        newScript.setAttribute('data-container-id', context.id)
        
        if (scriptInfo.src) {
          // 外部脚本
          newScript.src = scriptInfo.src
          newScript.onload = () => {
            context.completedScripts++
            // 加载下一个脚本
            setTimeout(() => loadExternalScripts(context, index + 1), 50)
          }
          newScript.onerror = () => {
            console.error(`脚本加载错误: ${scriptInfo.src}`)
            context.completedScripts++
            setTimeout(() => loadExternalScripts(context, index + 1), 50)
          }
          // 将脚本附加到容器内而不是body，隔离作用域
          container.appendChild(newScript)
        } else {
          // 内联脚本 - 确保脚本在正确的容器上下文中执行
          try {
            let scriptContent = scriptInfo.content
            
            // 替换可能的全局选择器，限制在当前容器内
            scriptContent = scriptContent.replace(/document\.querySelector\(['"]([^'"]+)['"]\)/g, 
              `document.getElementById('${context.id}').querySelector('$1')`)
            scriptContent = scriptContent.replace(/document\.querySelectorAll\(['"]([^'"]+)['"]\)/g, 
              `document.getElementById('${context.id}').querySelectorAll('$1')`)
            scriptContent = scriptContent.replace(/document\.getElementById\(['"]([^'"]+)['"]\)/g, 
              `(function(id) { const el = document.getElementById(id); return el && (el.closest('#${context.id}') ? el : null); })('$1')`)
            
            // 创建一个沙箱环境，避免全局变量污染
            const sandboxScript = `
              (function() {
                try {
                  // 创建沙箱环境
                  const containerElement = document.getElementById('${context.id}');
                  if (!containerElement) {
                    console.error('容器元素不存在:', '${context.id}');
                    return;
                  }
                  
                  const sandbox = {
                    container: containerElement,
                    window: window,
                    document: {
                      // 重写选择器方法，限制在当前容器内
                      querySelector: function(sel) { return containerElement.querySelector(sel); },
                      querySelectorAll: function(sel) { return containerElement.querySelectorAll(sel); },
                      getElementById: function(id) { 
                        const el = document.getElementById(id);
                        return el && (el.closest('#${context.id}') ? el : null);
                      },
                      // 确保事件监听器正确传递
                      addEventListener: function(type, listener, options) {
                        if (type === 'DOMContentLoaded') {
                          // 对于DOMContentLoaded事件，立即执行，因为此时DOM已经加载完成
                          setTimeout(listener, 0);
                        } else {
                          // 将事件监听器限制在当前容器内
                          containerElement.addEventListener(type, listener, options);
                        }
                      },
                      removeEventListener: function(type, listener, options) {
                        containerElement.removeEventListener(type, listener, options);
                      },
                      // 保留对原始document其他属性的访问
                      ...document
                    }
                  };
                  
                  // 在沙箱中执行脚本
                  with (sandbox) {
                    ${scriptContent}
                  }
                } catch (error) {
                  console.error('执行内联脚本时出错:', error, error.stack);
                }
              })();
            `;
            
            // 创建文本节点并添加到脚本
            newScript.textContent = sandboxScript;
            
            // 将脚本附加到容器
            container.appendChild(newScript);
            
            context.completedScripts++;
            // 继续下一个
            setTimeout(() => loadExternalScripts(context, index + 1), 50);
          } catch (error) {
            console.error('处理内联脚本时出错:', error, error.stack);
            context.completedScripts++;
            // 继续下一个，即使当前脚本失败
            setTimeout(() => loadExternalScripts(context, index + 1), 50);
          }
        }
      }
      
      // 开始加载脚本，使用创建的上下文
      if (scriptContents.length > 0) {
        setTimeout(() => loadExternalScripts(containerContext), 100)
      }
    }
  } catch (error) {
    console.error('处理HTML标签时出错:', error, error.stack)
  }
}
// 监听内容变化
watch(() => props.content, renderContent, { immediate: true })
</script>

<template>
  <div class="markdown-container">
    <div v-html="renderedContent"></div>
    <div v-if="!processingComplete" class="processing-indicator">
      <span>处理中...</span>
    </div>
  </div>
</template>

<style>
.image-container {
  min-height: 100px;
  margin: 15px 0;
  display: flex;
  justify-content: center;
  align-items: center;
}

.markdown-container {
  line-height: 1.6;
  word-wrap: break-word;
  text-align: left;
  width: 100%;
}

.markdown-container h1,
.markdown-container h2,
.markdown-container h3,
.markdown-container h4,
.markdown-container h5,
.markdown-container h6 {
  margin-top: 24px;
  margin-bottom: 16px;
  font-weight: 600;
  line-height: 1.25;
  text-align: left;
  color: #1a73e8;
}

.markdown-container p {
  margin: 0 0 16px;
  text-align: left;
}

.markdown-container ul,
.markdown-container ol {
  padding-left: 2em;
  margin-bottom: 16px;
  text-align: left;
}

.markdown-container li {
  margin-bottom: 0.5em;
  text-align: left;
}

.markdown-container code {
  padding: 0.2em 0.4em;
  margin: 0;
  font-size: 90%;
  background-color: rgba(175, 184, 193, 0.2);
  border-radius: 6px;
  font-family: 'Fira Code', 'Consolas', monospace;
}

.markdown-container pre {
  padding: 16px;
  overflow: auto;
  font-size: 90%;
  line-height: 1.45;
  background-color: rgba(246, 248, 250, 0.8);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  border-radius: 10px;
  margin-bottom: 16px;
  border: 1px solid rgba(0,0,0,0.05);
  box-shadow: 0 2px 10px rgba(0,0,0,0.05);
}

.markdown-container pre code {
  background-color: transparent;
  padding: 0;
}

.markdown-container img {
  max-width: 100%;
  height: auto;
  display: block;
  margin: 1.5em 0;
  border-radius: 10px;
  box-shadow: 0 4px 15px rgba(0,0,0,0.1);
  transition: transform 0.3s ease;
}

.markdown-container img:hover {
  transform: scale(1.01);
}

.markdown-container blockquote {
  padding: 0.5em 1.2em;
  color: #6a737d;
  border-left: 0.25em solid #1a73e8;
  background-color: rgba(230, 244, 255, 0.4);
  border-radius: 0 6px 6px 0;
  margin: 0 0 16px;
}

.image-placeholder {
  padding: 30px;
  background-color: rgba(240, 240, 240, 0.7);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  border-radius: 10px;
  margin: 15px 0;
  text-align: center;
  border: 1px dashed #ccc;
  box-shadow: 0 4px 10px rgba(0,0,0,0.05);
}

.image-placeholder.loading {
  animation: pulse 1.5s infinite;
}

.html-container {
  margin: 20px 0;
  padding: 20px;
  border: 1px solid #eee;
  border-radius: 10px;
  overflow-x: auto;
  max-width: 100%;
  box-sizing: border-box;
  background-color: rgba(255, 255, 255, 0.7);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  box-shadow: 0 4px 15px rgba(0,0,0,0.05);
  transition: all 0.3s ease;
}

.html-container:hover {
  box-shadow: 0 6px 18px rgba(0,0,0,0.08);
  border-color: #1a73e8;
}

.generated-image {
  max-width: 100%;
  border-radius: 10px;
  margin: 15px 0;
  box-shadow: 0 6px 20px rgba(0,0,0,0.1);
  transition: all 0.3s ease;
}

.generated-image:hover {
  transform: scale(1.02);
  box-shadow: 0 8px 25px rgba(0,0,0,0.15);
}

.image-error {
  padding: 15px;
  background-color: rgba(255, 235, 238, 0.7);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  color: #c62828;
  border-radius: 8px;
  margin: 15px 0;
  text-align: left;
  border-left: 4px solid #c62828;
  box-shadow: 0 4px 10px rgba(198, 40, 40, 0.1);
}

.processing-indicator {
  display: inline-block;
  padding: 10px 15px;
  background-color: rgba(240, 240, 240, 0.7);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  border-radius: 20px;
  font-size: 14px;
  color: #666;
  margin: 15px 0;
  animation: pulse 1.5s infinite;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}

table {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 20px;
  text-align: left;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0,0,0,0.05);
}

table th,
table td {
  padding: 12px 15px;
  border: 1px solid #dfe2e5;
}

table th {
  font-weight: 600;
  background-color: rgba(230, 244, 255, 0.6);
}

table tr:nth-child(even) {
  background-color: rgba(0, 0, 0, 0.02);
}

@keyframes pulse {
  0% { opacity: 0.6; }
  50% { opacity: 1; }
  100% { opacity: 0.6; }
}

/* 深色模式适配 */
@media (prefers-color-scheme: dark) {
  .markdown-container h1,
  .markdown-container h2,
  .markdown-container h3,
  .markdown-container h4,
  .markdown-container h5,
  .markdown-container h6 {
    color: #8ab4f8;
  }
  
  .markdown-container pre {
    background-color: rgba(30, 30, 30, 0.8);
    border-color: rgba(60, 60, 60, 0.5);
  }
  
  .markdown-container code {
    background-color: rgba(100, 100, 100, 0.3);
  }
  
  .markdown-container blockquote {
    color: #a0a0a0;
    border-left-color: #8ab4f8;
    background-color: rgba(26, 59, 92, 0.4);
  }
  
  .image-placeholder {
    background-color: rgba(42, 42, 42, 0.7);
    border-color: #555;
  }
  
  .html-container {
    border-color: #444;
    background-color: rgba(30, 30, 30, 0.7);
  }
  
  .html-container:hover {
    border-color: #8ab4f8;
  }
  
  .image-error {
    background-color: rgba(78, 40, 40, 0.7);
    color: #ff9a9a;
    border-left-color: #ff9a9a;
  }
  
  .processing-indicator {
    background-color: rgba(42, 42, 42, 0.7);
    color: #aaa;
  }
  
  table th,
  table td {
    border-color: #444;
  }
  
  table th {
    background-color: rgba(26, 59, 92, 0.6);
  }
  
  table tr:nth-child(even) {
    background-color: rgba(255, 255, 255, 0.03);
  }
}
</style>