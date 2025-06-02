<template>
  <div class="flex flex-col w-full h-full">
    <main class="flex-1 overflow-hidden">
      <div class="h-full overflow-y-auto">
        <div class="w-full max-w-4xl m-auto p-4">
          <div class="text-center mb-6">
            <h1 class="text-2xl font-bold text-gray-800 dark:text-white mb-2">
              <SvgIcon icon="ri:tools-fill" class="inline mr-2 text-blue-500" />
              AI 代码修复工具
            </h1>
            <p class="text-gray-600 dark:text-gray-300">上传代码文件，描述遇到的问题，获取AI修复建议</p>
          </div>

          <NSpace vertical size="large">
            <NCard title="配置信息" class="shadow-sm">
              <NSpace vertical size="medium">
                <NSpace vertical size="small">
                  <NInputGroup>
                    <NInputGroupLabel>DeepSeek API Key</NInputGroupLabel>
                    <NInput 
                      v-model:value="deepseekApiKey" 
                      type="password" 
                      placeholder="请输入DeepSeek API密钥"
                      show-password-on="click"
                    />
                  </NInputGroup>
                  
                  <NInputGroup>
                    <NInputGroupLabel>阿里云 Embedding Key</NInputGroupLabel>
                    <NInput 
                      v-model:value="aliyunEmbeddingKey" 
                      type="password" 
                      placeholder="请输入阿里云Embedding API密钥"
                      show-password-on="click"
                    />
                  </NInputGroup>
                </NSpace>

                <NSpace vertical size="small">
                  <NText strong>代码文件上传</NText>
                  <NUpload
                    :file-list="[]"
                    :max="1"
                    accept=".zip"
                    @change="handleFileChange"
                  >
                    <NButton>选择ZIP文件</NButton>
                  </NUpload>
                  <NText v-if="codeFile" depth="3" class="text-sm">
                    已选择: {{ codeFile.name }}
                  </NText>
                </NSpace>

                <NInputGroup>
                  <NInputGroupLabel>目标ID</NInputGroupLabel>
                  <NInput 
                    v-model:value="targetId" 
                    placeholder="请输入目标ID（通常与文件名相同）"
                  />
                </NInputGroup>

                <NSpace vertical size="small">
                  <NText strong>问题描述</NText>
                  <NInput
                    v-model:value="bugDescription"
                    type="textarea"
                    placeholder="请详细描述遇到的问题，包括错误信息、期望行为等"
                    :rows="4"
                  />
                </NSpace>

                <NButton 
                  type="primary" 
                  size="large" 
                  :loading="loading" 
                  :disabled="!isFormValid"
                  @click="handleSubmit"
                  block
                >
                  <template #icon>
                    <SvgIcon icon="ri:magic-fill" />
                  </template>
                  {{ loading ? '正在分析修复...' : '开始修复' }}
                </NButton>
              </NSpace>
            </NCard>

            <NCard v-if="showResult" title="修复建议" class="shadow-sm">
              <div v-html="formattedResult" class="formatted-content"></div>
              
              <template #action>
                <NButton @click="copyResult" size="small">
                  <template #icon>
                    <SvgIcon icon="ri:file-copy-line" />
                  </template>
                  复制结果
                </NButton>
              </template>
            </NCard>

            <NCard title="使用说明" class="shadow-sm">
              <NSpace vertical size="small">
                <NAlert type="info" :show-icon="false">
                  <strong>使用步骤：</strong>
                  <ol class="list-decimal list-inside mt-2 space-y-1">
                    <li>填写DeepSeek和阿里云的API密钥</li>
                    <li>上传包含代码的ZIP文件</li>
                    <li>输入目标ID（通常与文件名相同）</li>
                    <li>详细描述遇到的问题</li>
                    <li>点击"开始修复"获取AI建议</li>
                  </ol>
                </NAlert>
                
                <NAlert type="warning" :show-icon="false">
                  <strong>注意事项：</strong>
                  <ul class="list-disc list-inside mt-2 space-y-1">
                    <li>请确保后端服务已启动（127.0.0.1:5000）</li>
                    <li>上传的ZIP文件应包含完整的代码结构</li>
                    <li>问题描述越详细，修复建议越准确</li>
                    <li>请妥善保管API密钥，避免泄露</li>
                  </ul>
                </NAlert>
              </NSpace>
            </NCard>
          </NSpace>
        </div>
      </div>
    </main>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { NButton, NInput, NUpload, NCard, NSpace, NAlert, useMessage, NInputGroup, NInputGroupLabel, NText } from 'naive-ui' // Removed NSpin, NDivider if not used
import { SvgIcon } from '@/components/common'
// import { useBasicLayout } from '@/hooks/useBasicLayout' // Removed if isMobile not used, or keep if needed elsewhere

const ms = useMessage()
// const { isMobile } = useBasicLayout() // Removed if isMobile not used

// 表单数据
const codeFile = ref<File | null>(null)
const bugDescription = ref('')
const deepseekApiKey = ref('')
const aliyunEmbeddingKey = ref('')
const targetId = ref('')

// 状态管理
const loading = ref(false)
const result = ref<string>('')
const showResult = ref(false)

// 文件上传处理
const handleFileChange = (data: { file: any, fileList: any[], event?: Event }) => {
  const uploadFile = data.file
  if (uploadFile && uploadFile.file) {
    codeFile.value = uploadFile.file
    const fileName = uploadFile.file.name
    if (fileName.endsWith('.zip')) {
      targetId.value = fileName.slice(0, -4)
    } else {
      targetId.value = fileName
    }
  }
  return false 
}

// 表单验证
const isFormValid = computed(() => {
  return codeFile.value && 
         bugDescription.value.trim() && 
         deepseekApiKey.value.trim() && 
         aliyunEmbeddingKey.value.trim() && 
         targetId.value.trim()
})

// 提交修复请求
const handleSubmit = async () => {
  if (!isFormValid.value) {
    ms.error('请填写完整信息')
    return
  }

  loading.value = true
  showResult.value = false
  
  try {
    const formData = new FormData()
    
    formData.append('code_zip', codeFile.value!)
    
    const apiData = {
      API_KEY: deepseekApiKey.value,
      target_id: targetId.value,
      API_KEY_EMBEDDING: aliyunEmbeddingKey.value
    }
    formData.append('api_data', JSON.stringify(apiData))
    
    const bugDescriptionData = {
      problem_statement: bugDescription.value
    }
    formData.append('bug_description', JSON.stringify(bugDescriptionData))
    
    const response = await fetch('http://127.0.0.1:5000/process_bug_fix', {
      method: 'POST',
      body: formData
    })
    
    let responseData
    try {
      responseData = await response.json()
    } catch (error: any) {
      throw new Error(`响应解析失败: ${error.message}`)
    }
    
    if (response.ok && responseData.status === 'success') {
      if (responseData.data && responseData.data.agentless_context && responseData.data.agentless_context.length > 0) {
        result.value = responseData.data.agentless_context[0]
        showResult.value = true
        ms.success('代码修复建议生成成功!')
      } else {
        result.value = '未能生成有效的修复建议，请检查输入信息或重试。'
        showResult.value = true
        ms.warning('生成结果为空，请检查输入信息')
      }
    } else if (responseData.status === 'failed') {
      let content = ''
      
      if (responseData.data && responseData.data.agentless_context) {
        if (Array.isArray(responseData.data.agentless_context)) {
          content = responseData.data.agentless_context[0] || ''
        } else if (responseData.data.agentless_context.message) {
          try {
            const messageContent = JSON.parse(responseData.data.agentless_context.message)
            content = Array.isArray(messageContent) ? messageContent[0] : messageContent
          } catch (parseError) {
            console.error('JSON解析失败:', parseError)
            content = responseData.data.agentless_context.message
          }
        } else {
          content = JSON.stringify(responseData.data.agentless_context, null, 2)
        }
      }
      
      if (content) {
        result.value = content
        showResult.value = true
        ms.warning(`处理失败，但返回了部分结果: ${responseData.message || '请查看下方返回内容'}`)
      } else {
        result.value = `处理失败: ${responseData.message || '未知错误'}`
        showResult.value = true
        ms.error('代码修复失败')
      }
    } else {
      throw new Error(`HTTP错误: ${response.status} - ${responseData.message || '服务器返回错误'}`)
    }
    
  } catch (error: any) {
    console.error('请求失败:', error)
    ms.error(`请求失败: ${error.message || '未知错误'}`)
    result.value = `错误信息: ${error.message || '请求失败，请检查网络连接和后端服务'}`
    showResult.value = true
  } finally {
    loading.value = false
  }
}

// 复制结果
const copyResult = async () => {
  try {
    await navigator.clipboard.writeText(result.value) // Consider stripping HTML for plain text copy if result.value is raw response
    ms.success('复制成功 (原始内容)')
  } catch (error) {
    ms.error('复制失败')
  }
}

// HTML转义函数
const escapeHtml = (unsafe: string): string => {
  return unsafe
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#039;");
}

// 修复后的格式化结果计算属性
const formattedResult = computed(() => {
  if (!result.value) return '';

  let formatted = result.value;

  // 首先处理可能的数组格式数据 (如果后端有时会返回 ["string content"] 这样的字符串)
  if (formatted.startsWith("['") && formatted.endsWith("']")) {
    try {
      // Attempt to parse it as if it's a stringified array representation
      const parsedArrayAttempt = JSON.parse(formatted.replace(/'/g, '"'));
      if (Array.isArray(parsedArrayAttempt) && parsedArrayAttempt.length > 0) {
        formatted = parsedArrayAttempt[0];
      }
    } catch (e) {
      // Fallback for simple ['...'] string, extract content
      const match = formatted.match(/^\['(.*)'\]$/s);
      if (match) {
        formatted = match[1].replace(/\\'/g, "'"); // Handle escaped single quotes if any
      }
    }
  } else if (formatted.startsWith('[') && formatted.endsWith(']')) { // Standard JSON array of strings
    try {
      const parsed = JSON.parse(formatted);
      if (Array.isArray(parsed) && parsed.length > 0 && typeof parsed[0] === 'string') {
        formatted = parsed[0];
      }
    } catch (e) {
      // Not a valid JSON array string, proceed with `formatted` as is
    }
  }


  // 将后端返回的 \\n (双反斜杠n) 替换为实际的换行符 \n
  formatted = formatted.replace(/\\n/g, '\n');

  // 处理 SEARCH/REPLACE 块
  formatted = formatted.replace(/<<<<<<< SEARCH\s*\n?([\s\S]*?)\s*\n?=======\s*\n?([\s\S]*?)\s*\n?>>>>>>> REPLACE/g, (match, searchCode, replaceCode) => {
    const searchLines = searchCode.split('\n'); //保留所有行，包括空行
    const replaceLines = replaceCode.split('\n'); //保留所有行，包括空行

    let diffHtml = '<div class="diff-block">';
    diffHtml += '<div class="diff-header">代码修改建议</div>';

    if (searchLines.length > 0 && searchCode !== '') { // Check searchCode explicitly if needed
      searchLines.forEach((line: string) => {
        diffHtml += `<div class="diff-line removed"><span class="diff-marker">-</span><span class="diff-content">${escapeHtml(line)}</span></div>`;
      });
    }
    
    if (replaceLines.length > 0 && replaceCode !== '') { // Check replaceCode explicitly
      replaceLines.forEach((line: string) => {
        diffHtml += `<div class="diff-line added"><span class="diff-marker">+</span><span class="diff-content">${escapeHtml(line)}</span></div>`;
      });
    }
    diffHtml += '</div>';
    return diffHtml;
  });
  
  // 处理普通的代码块 (```python ... ```)
  formatted = formatted.replace(/```python\n([\s\S]*?)```/g, (match, code) => {
    return `<div class="code-block python"><pre>${escapeHtml(code.trim())}</pre></div>`;
  });
  
  // 处理通用的代码块 (``` ... ``` or ```lang\n...```)
  formatted = formatted.replace(/```(\w*\n)?([\s\S]*?)```/g, (match, langPrefix, code) => {
    const lang = langPrefix ? langPrefix.trim() : '';
    return `<div class="code-block ${escapeHtml(lang)}"><pre>${escapeHtml(code.trim())}</pre></div>`;
  });
  
  // 处理文件路径 (### 开头或📁 开头的行)
  formatted = formatted.replace(/^###\s*([^\n]+)/gm, '<div class="file-path">📄 $1</div>');
  formatted = formatted.replace(/📁\s*([^\n]+)/g, '<div class="file-path">📁 $1</div>');
  
  // 处理其他单独的diff格式行（确保这个规则不会与上面的SEARCH/REPLACE冲突）
  // This regex needs to be careful not to re-process existing diff-blocks.
  // The current implementation might be okay if diff-blocks are complex enough not to match this simple pattern.
  // It's generally better to process more specific patterns first.
  // This block is kept from original, but ensure it doesn't corrupt already formatted diffs.
  formatted = formatted.replace(/((?:^\s*[-+].*(?:\n|$))+)/gm, (m) => {
    if (m.includes('diff-marker')) return m; // Skip if it's part of an already formatted diff line

    const lines = m.trim().split('\n');
    let diffHtml = '<div class="diff-block">'; // This might create nested or incorrect diff blocks
    
    lines.forEach((line: string) => {
      const trimmedLine = line.trim();
      if (trimmedLine.startsWith('-')) {
        const content = trimmedLine.substring(1).trim();
        diffHtml += `<div class="diff-line removed"><span class="diff-marker">-</span><span class="diff-content">${escapeHtml(content)}</span></div>`;
      } else if (trimmedLine.startsWith('+')) {
        const content = trimmedLine.substring(1).trim();
        diffHtml += `<div class="diff-line added"><span class="diff-marker">+</span><span class="diff-content">${escapeHtml(content)}</span></div>`;
      } else {
         //diffHtml += `<div class="diff-line"><span class="diff-marker"> </span><span class="diff-content">${escapeHtml(trimmedLine)}</span></div>`; // Context line
      }
    });
    diffHtml += '</div>';
    return diffHtml;
  });

  // 移除了最后的全局 formatted = formatted.replace(/\n/g, '<br>')
  // CSS white-space: pre-line on .formatted-content will handle general newlines.
  // CSS white-space: pre-wrap on .diff-content and .code-block pre will handle newlines there.
  return formatted;
});

</script>

<style scoped>
/* :deep() is used to apply styles to content generated by v-html */
:deep(.n-card) { /* If NCard itself is not part of v-html, :deep isn't needed here unless styling its slots */
  margin-bottom: 0; /* This was .n-card, assuming it's a Naive UI component in the template */
}

:deep(.formatted-content) {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  line-height: 1.6;
  color: #333;
  white-space: pre-line; /* Handles \n in general text as line breaks */
}

:deep(.code-block) {
  background-color: #f8f9fa;
  border: 1px solid #e9ecef;
  border-radius: 6px;
  margin: 12px 0;
  overflow-x: auto;
}

:deep(.code-block pre) {
  margin: 0;
  padding: 12px;
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  font-size: 14px;
  line-height: 1.4;
  color: #212529;
  background: transparent;
  border: none;
  max-height: 400px;
  overflow-y: auto;
  white-space: pre-wrap; /* Preserves whitespace and newlines, wraps lines */
}

:deep(.code-block.python pre) {
  color: #0056b3; /* Or specific Python syntax highlighting color */
}

:deep(.diff-block) {
  background-color: #ffffff;
  border: 1px solid #d1ecf1; /* Light blue border for info context */
  border-radius: 6px;
  margin: 12px 0;
  overflow: hidden; /* Important for border-radius */
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  font-size: 14px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05); /* Softer shadow */
}

:deep(.diff-header) {
  background: linear-gradient(135deg, #4a90e2 0%, #357abd 100%); /* Example gradient */
  color: white;
  padding: 8px 16px;
  font-weight: 600;
  font-size: 13px; /* Slightly smaller header text */
  border-bottom: 1px solid #357abd; /* Match gradient end */
}

:deep(.diff-line) {
  display: flex;
  align-items: flex-start; /* Align marker and content to the top */
  padding: 4px 0px; /* Reduced vertical padding, no horizontal for marker */
  line-height: 1.4;
  border-left: 3px solid transparent; /* For the color bar */
}

:deep(.diff-line.removed) {
  background-color: #ffebee; /* Light red background */
  border-left-color: #f44336; /* Red accent bar */
  /* color: #c62828; Removed to let diff-content handle color */
}

:deep(.diff-line.added) {
  background-color: #e8f5e9; /* Light green background */
  border-left-color: #4caf50; /* Green accent bar */
  /* color: #2e7d32; Removed to let diff-content handle color */
}

:deep(.diff-marker) {
  display: inline-block;
  width: 24px; /* Width for the +/- symbol */
  text-align: center;
  font-weight: bold;
  flex-shrink: 0;
  user-select: none; /* Prevent selection of +/- */
  font-size: 16px; /* Slightly larger marker */
  padding: 0 4px; /* Padding around marker */
}

:deep(.diff-line.removed .diff-marker) {
  color: #d32f2f; /* Darker red for symbol */
  /* background-color: rgba(244, 67, 54, 0.1); Optional: very light bg for marker itself */
}

:deep(.diff-line.added .diff-marker) {
  color: #388e3c; /* Darker green for symbol */
  /* background-color: rgba(76, 175, 80, 0.1); Optional */
}

:deep(.diff-content) {
  flex: 1;
  padding-left: 8px;  /* Space between marker and text */
  padding-right: 12px; /* Padding at the end of the line */
  white-space: pre-wrap; /* Preserves whitespace/newlines, wraps text */
  word-break: break-all; /* Break long words if necessary */
  color: #24292f; /* Default text color for diff content */
}
/* Ensure removed/added lines have specific text color if needed, or rely on parent */
:deep(.diff-line.removed .diff-content) {
  color: #c62828; /* Text color for removed lines */
}
:deep(.diff-line.added .diff-content) {
  color: #2e7d32; /* Text color for added lines */
}


:deep(.file-path) {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); /* Example purple gradient */
  color: #ffffff;
  padding: 10px 16px;
  border-radius: 6px;
  font-weight: 600;
  margin: 16px 0 12px 0;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

/* 深色模式支持 */
@media (prefers-color-scheme: dark) {
  :deep(.formatted-content) {
    color: #e6e6e6;
    white-space: pre-line;
  }
  
  :deep(.code-block) {
    background-color: #1e1e1e;
    border-color: #3e3e3e;
  }
  
  :deep(.code-block pre) {
    color: #d4d4d4;
  }
  :deep(.code-block.python pre) {
    color: #87cefa; /* Lighter blue for dark mode Python */
  }
  
  :deep(.diff-block) {
    background-color: #0d1117; /* GitHub dark background */
    border-color: #30363d; /* GitHub dark border */
    box-shadow: 0 2px 8px rgba(0,0,0,0.2);
  }
  :deep(.diff-header) {
    background: #1f2428; /* Darker header */
    border-bottom-color: #30363d;
  }

  :deep(.diff-line.removed) {
    background-color: rgba(248, 81, 73, 0.15); /* More subtle red */
    border-left-color: #f85149;
  }
  
  :deep(.diff-line.added) {
    background-color: rgba(46, 160, 67, 0.15); /* More subtle green */
    border-left-color: #2ea043;
  }

  :deep(.diff-line.removed .diff-marker) {
    color: #f85149;
  }

  :deep(.diff-line.added .diff-marker) {
    color: #2ea043;
  }
  
  :deep(.diff-content) {
    color: #c9d1d9; /* GitHub dark mode text */
  }
  :deep(.diff-line.removed .diff-content) {
    color: #ff817b; /* Lighter red text for dark mode */
  }
  :deep(.diff-line.added .diff-content) {
    color: #85d893; /* Lighter green text for dark mode */
  }

  :deep(.file-path) {
    background: linear-gradient(135deg, #3e4095 0%, #583277 100%); /* Darker purple */
  }
}
</style>