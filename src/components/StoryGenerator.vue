<script setup lang="ts">
import { ref, computed } from 'vue'
import html2canvas from 'html2canvas'
import JSZip from 'jszip'
import { saveAs } from 'file-saver'
import coverBg from '../assets/书名封面.png'
import contentBg from '../assets/故事背景页.png'

const title = ref('')
const content = ref('')
const previewMode = ref<'tile' | 'page'>('page') // 默认使用翻页模式
const currentPageIndex = ref(0)
const isSubmitted = ref(false)

// 计算分页内容
const CHARS_PER_PAGE = 250 // 每页理想字符数
const pages = computed(() => {
  const contentText = content.value
    .trim()
    .replace(/\n\s*\n/g, '\n') // 将多个空行替换为单个换行
    .replace(/^\s*[\r\n]/gm, '') // 移除开头的空行
  
  if (!contentText) return []
  
  // 先按句号分割成句子
  const sentences = contentText.split(/(?<=。)/)
  const pages: string[] = []
  let currentPage = ''
  let currentLength = 0
  
  for (let i = 0; i < sentences.length; i++) {
    const sentence = sentences[i]
    const sentenceLength = sentence.length
    
    // 如果当前页面为空，直接添加句子
    if (currentLength === 0) {
      currentPage = sentence
      currentLength = sentenceLength
      continue
    }
    
    // 判断添加这个句子是否会超过页面容量
    if (currentLength + sentenceLength <= CHARS_PER_PAGE) {
      // 如果不超过，添加到当前页
      currentPage += sentence
      currentLength += sentenceLength
    } else {
      // 如果超过，当前页结束，开始新页面
      pages.push(currentPage)
      currentPage = sentence
      currentLength = sentenceLength
    }
  }
  
  // 添加最后一页
  if (currentPage) {
    pages.push(currentPage)
  }
  
  return pages
})

// 切换预览模式
const togglePreviewMode = () => {
  previewMode.value = previewMode.value === 'tile' ? 'page' : 'tile'
  currentPageIndex.value = 0 // 切换模式时重置页码
}

// 翻页控制
const totalPages = computed(() => pages.value.length + 1) // 总页数（内容页 + 封面）

const goToPage = (index: number) => {
  currentPageIndex.value = Math.max(0, Math.min(index, totalPages.value - 1))
}

const goToPrevPage = () => {
  goToPage(currentPageIndex.value - 1)
}

const goToNextPage = () => {
  goToPage(currentPageIndex.value + 1)
}

// 下载所有页面
const downloadAllPages = async () => {
  if (!title.value.trim() || !content.value.trim()) {
    alert('请填写故事标题和内容')
    return
  }
  try {
    // 创建一个临时容器来渲染每一页
    const tempContainer = document.createElement('div')
    tempContainer.style.position = 'absolute'
    tempContainer.style.left = '-9999px'
    tempContainer.style.top = '-9999px'
    document.body.appendChild(tempContainer)
    
    const zip = new JSZip()
    const storyTitle = title.value || '晚安故事'
    
    // 添加封面
    const coverCanvas = await renderPage('cover')
    if (coverCanvas) {
      const coverBlob = await new Promise<Blob>(resolve => {
        coverCanvas.toBlob(blob => resolve(blob!), 'image/png')
      })
      zip.file(`${storyTitle}_封面.png`, coverBlob)
    }
    
    // 添加内容页
    for (let i = 0; i < pages.value.length; i++) {
      const canvas = await renderPage('content', i)
      if (canvas) {
        const blob = await new Promise<Blob>(resolve => {
          canvas.toBlob(blob => resolve(blob!), 'image/png')
        })
        zip.file(`${storyTitle}_第${i + 1}页.png`, blob)
      }
      // 添加一点延迟，避免浏览器卡顿
      await new Promise(resolve => setTimeout(resolve, 100))
    }
    
    // 生成并下载压缩包
    const content = await zip.generateAsync({ type: 'blob' })
    saveAs(content, `${storyTitle}.zip`)
    
    document.body.removeChild(tempContainer)
  } catch (error) {
    console.error('导出图片失败:', error)
  }
}

// 渲染单页
const renderPage = async (type: 'cover' | 'content', pageIndex: number = 0) => {
  const tempContainer = document.createElement('div')
  tempContainer.style.width = '534px'
  tempContainer.style.height = '712px'
  tempContainer.innerHTML = `
    <div class="relative w-full aspect-[3/4]">
      <img 
        src="${type === 'cover' ? coverBg : contentBg}" 
        class="absolute inset-0 w-full h-full object-cover"
      />
      <div class="relative z-10 h-full flex flex-col">
        ${type === 'cover' ? `
          <div class="absolute inset-0 flex items-center justify-center">
            <span class="text-[32px] font-medium text-center story-title top-[30%] absolute" style="color: white; -webkit-text-stroke: 4px #9256A7; paint-order: stroke fill;">
              << ${title.value || '晚安故事'} >>
            </span>
          </div>
        ` : `
          <div class="absolute inset-0 flex flex-col px-10 pt-16 pb-2">
            <p class="text-[24px] leading-[1.8] text-[#333333] story-text" style="white-space: normal; text-align: justify; letter-spacing: 0.5px;">
              ${pages.value[pageIndex] || ''}
            </p>
          </div>
        `}
      </div>
    </div>
  `
  
  document.body.appendChild(tempContainer)
  
  try {
    const canvas = await html2canvas(tempContainer.firstElementChild as HTMLElement, {
      scale: 2,
      useCORS: true,
      backgroundColor: null,
    })
    return canvas
  } finally {
    document.body.removeChild(tempContainer)
  }
}
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-purple-50 to-pink-50 py-6 px-4">
    <div class="max-w-7xl mx-auto">

      <div class="grid grid-cols-1 lg:grid-cols-2 gap-12">
        <!-- 输入表单 -->
        <div class="space-y-8">
           <h1 class="text-3xl font-bold text-gray-800 mb-2">晚安故事生成器</h1>
        <p class="text-gray-500">创建美好的睡前故事，留下温暖的回忆</p>
          <div class="card p-8">
            <div class="space-y-6">
              <div>
                <label class="block text-sm font-medium text-gray-700 mb-2">故事标题</label>
                <input
                  v-model="title"
                  type="text"
                  class="input-base"
                  :class="{'ring-2 ring-red-500': !title.trim() && isSubmitted}"
                  placeholder="请输入一个温暖的标题"
                  required
                >
              </div>
              
              <div>
                <label class="block text-sm font-medium text-gray-700 mb-2">故事内容</label>
                <textarea
                  v-model="content"
                  rows="8"
                  class="input-base"
                  :class="{'ring-2 ring-red-500': !content.trim() && isSubmitted}"
                  placeholder="在这里书写您的故事..."
                  required
                ></textarea>
              </div>
            </div>
          </div>

          <!-- 控制面板 -->
          <div class="card p-6">
            <div class="flex items-center justify-between mb-6">
              <div class="flex items-center space-x-2">
                <span class="text-sm font-medium text-gray-600">预计页数：</span>
                <span class="px-3 py-1 bg-purple-100 text-purple-700 rounded-full text-sm font-medium">
                  {{ pages.length ? pages.length + 1 : 1 }} 页
                </span>
              </div>
              <button
                @click="togglePreviewMode"
                class="btn-secondary"
              >
                {{ previewMode === 'tile' ? '切换到翻页模式' : '切换到平铺模式' }}
              </button>
            </div>
            
            <button
              @click="downloadAllPages"
              class="btn w-full flex items-center justify-center space-x-2"
            >
              <span>导出故事</span>
              <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M3 17a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm3.293-7.707a1 1 0 011.414 0L9 10.586V3a1 1 0 112 0v7.586l1.293-1.293a1 1 0 111.414 1.414l-3 3a1 1 0 01-1.414 0l-3-3a1 1 0 010-1.414z" clip-rule="evenodd" />
              </svg>
            </button>
          </div>
        </div>
        
        <!-- 预览区域 -->
        <div class="space-y-6">
          <!-- 翻页控制器 -->
          <div v-if="previewMode === 'page'" class="flex items-center justify-center space-x-6 mb-6">
            <button
              @click="goToPrevPage"
              class="group flex items-center space-x-2 px-5 py-2.5 text-gray-600 hover:text-purple-600 disabled:opacity-40 disabled:hover:text-gray-600 transition-colors duration-200"
              :disabled="currentPageIndex === 0"
            >
              <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 transition-transform group-hover:-translate-x-0.5" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M12.707 5.293a1 1 0 010 1.414L9.414 10l3.293 3.293a1 1 0 01-1.414 1.414l-4-4a1 1 0 010-1.414l4-4a1 1 0 011.414 0z" clip-rule="evenodd" />
              </svg>
              <span class="font-medium">上一页</span>
            </button>

            <div class="px-4 py-2 bg-purple-50 rounded-xl text-purple-600 font-medium min-w-[80px] text-center">
              {{ currentPageIndex + 1 }} / {{ totalPages }}
            </div>

            <button
              @click="goToNextPage"
              class="group flex items-center space-x-2 px-5 py-2.5 text-gray-600 hover:text-purple-600 disabled:opacity-40 disabled:hover:text-gray-600 transition-colors duration-200"
              :disabled="currentPageIndex === totalPages - 1"
            >
              <span class="font-medium">下一页</span>
              <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 transition-transform group-hover:translate-x-0.5" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M7.293 14.707a1 1 0 010-1.414L10.586 10 7.293 6.707a1 1 0 011.414-1.414l4 4a1 1 0 010 1.414l-4 4a1 1 0 01-1.414 0z" clip-rule="evenodd" />
              </svg>
            </button>
          </div>
          
          <!-- 平铺模式 -->
          <div v-if="previewMode === 'tile'" class="space-y-8">
            <!-- 封面预览 -->
            <div class="relative aspect-[3/4] bg-white rounded-lg shadow-lg overflow-hidden">
              <img 
                :src="coverBg" 
                class="absolute inset-0 w-full h-full object-cover"
              />
              <div class="relative z-10 h-full flex flex-col">
                <div class="absolute inset-0 flex items-center justify-center">
                  <span class="text-[32px] font-medium text-center story-title top-[30%] absolute" style="color: white; -webkit-text-stroke: 4px #9256A7; paint-order: stroke fill;">
                    << {{ title || '晚安故事' }} >>
                  </span>
                </div>
              </div>
            </div>
            
            <!-- 内容页预览 -->
            <template v-if="pages.length > 0">
              <div 
                v-for="(page, index) in pages"
                :key="index"
                class="relative aspect-[3/4] bg-white rounded-lg shadow-lg overflow-hidden"
              >
                <img 
                  :src="contentBg" 
                  class="absolute inset-0 w-full h-full object-cover"
                />
                <div class="relative z-10 h-full flex flex-col">
                  <div class="absolute inset-0 flex flex-col px-10 pt-16 pb-2">
                    <p class="text-[24px] leading-[1.8] text-[#333333] story-text" style="white-space: normal; text-align: justify; letter-spacing: 0.5px;">
                      {{ page }}
                    </p>
                  </div>
                </div>
              </div>
            </template>
          </div>
          
          <!-- 翻页模式 -->
          <div v-if="previewMode === 'page'" class="card overflow-hidden"
              style="width: 534px; height: 712px;"
          >
            <div class="relative aspect-[3/4]"
                  style="width: 534px; height: 712px;"
            >
              <!-- 封面 -->
              <template v-if="currentPageIndex === 0">
                <img 
                  :src="coverBg" 
                  class="absolute inset-0 w-full h-full object-cover"
                />
                <div class="relative z-10 h-full flex flex-col">
                  <div class="absolute inset-0 flex items-center justify-center">
                    <span class="text-[32px] font-medium text-center story-title top-[30%] absolute" style="color: white; -webkit-text-stroke: 4px #9256A7; paint-order: stroke fill;">
                      << {{ title || '晚安故事' }} >>
                    </span>
                  </div>
                </div>
              </template>
              
              <!-- 内容页 -->
              <template v-else>
                <img 
                  :src="contentBg" 
                  class="absolute inset-0 w-full h-full object-cover"
                  style="width: 534px; height: 712px;"
                />
                <div class="relative z-10 h-full flex flex-col">
                  <div class="absolute inset-0 flex flex-col px-10 pt-16 pb-2">
                    <p class="text-[24px] leading-[1.8] text-[#333333] story-text" style="white-space: normal; text-align: justify; letter-spacing: 0.5px;">
                      {{ pages[currentPageIndex - 1] }}
                    </p>
                  </div>
                </div>
              </template>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.preview-page {
  break-inside: avoid;
  page-break-inside: avoid;
}

/* 添加翻页按钮禁用状态样式 */
.btn-secondary:disabled {
  @apply opacity-50 cursor-not-allowed hover:bg-white hover:shadow-none;
}
</style> 