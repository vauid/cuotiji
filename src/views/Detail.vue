<template>
  <div class="question-detail pb-16 min-h-screen">
    <van-nav-bar title="错题详情" left-arrow @click-left="onClickLeft" fixed placeholder :border="false" />
    
    <div v-if="question" class="p-4">
      <!-- 练习模式开关 -->
      <div class="flex justify-end mb-4">
        <span class="mr-2" style="color: var(--text-sub)">练习模式 (隐藏解析)</span>
        <van-switch v-model="practiceMode" size="20px" active-color="var(--text-main)" />
      </div>

      <!-- 题目内容 -->
      <div class="glass-panel p-4 mb-4">
        <div class="flex gap-2 mb-3 flex-wrap">
          <van-tag v-if="categoryName" class="custom-tag">{{ categoryName }}</van-tag>
          <van-tag v-if="chapterName" class="custom-tag">{{ chapterName }}</van-tag>
          <van-tag class="custom-tag">{{ question.type || '其他' }}</van-tag>
        </div>
        <h3 class="font-bold text-lg mb-2" style="color: var(--text-main)">题目：</h3>
        <p class="whitespace-pre-wrap" style="color: var(--text-main)">{{ question.content }}</p>
        
        <div v-if="question.options" class="mt-4">
          <h4 class="font-bold text-md mb-1" style="color: var(--text-main)">选项：</h4>
          <p class="whitespace-pre-wrap" style="color: var(--text-main)">{{ question.options }}</p>
        </div>

        <div v-if="question.image" class="mt-4">
          <van-image :src="question.image" fit="contain" @click="showImagePreview([question.image])" />
        </div>
      </div>

      <!-- 解析内容 (练习模式下隐藏) -->
      <div v-show="!practiceMode" class="glass-panel p-4 mb-4">
        <div class="mb-4">
          <h3 class="font-bold text-lg mb-2" style="color: var(--text-main)">答案：</h3>
          <p class="whitespace-pre-wrap" style="color: var(--text-main)">{{ question.answer || '暂无答案' }}</p>
        </div>
        <div>
          <h3 class="font-bold text-lg mb-2" style="color: var(--text-main)">解析：</h3>
          <p class="whitespace-pre-wrap" style="color: var(--text-main)">{{ question.analysis || '暂无解析' }}</p>
        </div>
      </div>

      <!-- AI 助教聊天区 -->
      <div v-show="!practiceMode" class="glass-panel overflow-hidden mt-6">
        <div class="p-3 font-bold flex items-center justify-between" style="background: rgba(0,0,0,0.05); color: var(--text-main); border-bottom: 1px solid var(--glass-border)">
          <div class="flex items-center">
            <van-icon name="chat-o" class="mr-2" /> AI 助教答疑
          </div>
          <van-button size="mini" plain @click="clearChatHistory" style="border-color: var(--text-sub); color: var(--text-sub)">清空记录</van-button>
        </div>
        
        <div class="p-4 h-64 overflow-y-auto" ref="chatContainer">
          <div v-if="chatHistory.length === 0" class="text-center mt-10" style="color: var(--text-sub)">
            对这道题还有疑问？直接问我吧！
          </div>
          
          <div v-for="(msg, index) in chatHistory" :key="index" class="mb-4">
            <div v-if="msg.role === 'user'" class="text-right" style="color: var(--text-main)">
              <span class="font-bold">问：</span>{{ msg.content }}
            </div>
            <div v-else class="text-left w-full" style="color: var(--text-main)">
              <span class="font-bold">答：</span>{{ msg.content }}
            </div>
          </div>
          <div v-if="aiThinking" class="text-sm ml-2" style="color: var(--text-sub)">AI 思考中...</div>
        </div>
        
        <div class="p-2 flex items-center" style="border-top: 1px solid var(--panel-border); background: rgba(0,0,0,0.02)">
          <van-field v-model="chatInput" placeholder="输入你的疑问..." class="flex-1 rounded-full px-4 py-1 mr-2 !bg-transparent" :border="false" @keyup.enter="sendChatMessage" style="border: 1px solid var(--panel-border)" />
          <van-button size="small" round @click="sendChatMessage" :disabled="!chatInput.trim() || aiThinking" class="custom-btn">发送</van-button>
        </div>
      </div>
      
      <div class="mt-8 flex gap-4">
        <van-button block plain @click="deleteQuestion" style="border-color: #ef4444; color: #ef4444">删除此题</van-button>
      </div>
    </div>
    <van-empty v-else description="未找到该错题" class="glass-panel" />
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import localforage from 'localforage'
import { showImagePreview, showDialog, showToast } from 'vant'

const route = useRoute()
const router = useRouter()
const question = ref(null)
const practiceMode = ref(false)
const categoryName = ref('')
const chapterName = ref('')

// AI Chat
const chatInput = ref('')
const chatHistory = ref([])
const aiThinking = ref(false)
const chatContainer = ref(null)

const onClickLeft = () => {
  router.back()
}

onMounted(async () => {
  const id = route.params.id
  if (id) {
    const questions = await localforage.getItem('questions') || []
    question.value = questions.find(q => q.id === id)
    
    if (question.value && question.value.categoryId) {
      const categories = await localforage.getItem('categories') || []
      const cat = categories.find(c => c.id === question.value.categoryId)
      if (cat) {
        categoryName.value = cat.name
        if (question.value.chapterId && cat.chapters) {
          const chap = cat.chapters.find(c => c.id === question.value.chapterId)
          if (chap) {
            chapterName.value = chap.name
          }
        }
      }
    }
  }
})

const deleteQuestion = () => {
  showDialog({
    title: '确认删除',
    message: '确定要删除这道错题吗？',
    showCancelButton: true,
  }).then(async () => {
    const questions = await localforage.getItem('questions') || []
    const newQuestions = questions.filter(q => q.id !== question.value.id)
    await localforage.setItem('questions', newQuestions)
    showToast('已删除')
    router.back()
  }).catch(() => {})
}

const scrollToBottom = async () => {
  await nextTick()
  if (chatContainer.value) {
    chatContainer.value.scrollTop = chatContainer.value.scrollHeight
  }
}

const sendChatMessage = async () => {
  if (!chatInput.value.trim() || aiThinking.value) return
  
  const userMsg = chatInput.value
  chatInput.value = ''
  
  chatHistory.value.push({ role: 'user', content: userMsg })
  scrollToBottom()
  
  const settings = JSON.parse(localStorage.getItem('ai_settings') || '{}')
  if (!settings.apiUrl || !settings.apiKey) {
    chatHistory.value.push({ role: 'assistant', content: '请先在设置中配置 API URL 和 API Key。' })
    scrollToBottom()
    return
  }

  aiThinking.value = true
  
  try {
    // 构建系统提示词，包含当前题目的上下文
    const systemPrompt = `你是一个耐心的错题辅导老师。学生正在复习一道错题。
题目内容：${question.value.content}
选项：${question.value.options || '无'}
正确答案：${question.value.answer || '无'}
标准解析：${question.value.analysis || '无'}
请根据以上题目信息，解答学生的疑问。回答要简明扼要，通俗易懂。`

    const messages = [
      { role: 'system', content: systemPrompt },
      ...chatHistory.value.map(m => ({ role: m.role, content: m.content }))
    ]

    const response = await fetch(`${settings.apiUrl}/chat/completions`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${settings.apiKey}`
      },
      body: JSON.stringify({
        model: settings.modelName || 'gpt-4o',
        messages: messages
      })
    })

    if (!response.ok) throw new Error('API 请求失败')

    const data = await response.json()
    const aiReply = data.choices[0].message.content
    
    chatHistory.value.push({ role: 'assistant', content: aiReply })
  } catch (error) {
    console.error('Chat failed:', error)
    chatHistory.value.push({ role: 'assistant', content: '抱歉，网络请求失败，请检查 API 配置或稍后再试。' })
  } finally {
    aiThinking.value = false
    scrollToBottom()
  }
}
</script>
