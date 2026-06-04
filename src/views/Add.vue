<template>
  <div class="add-question pb-20 min-h-screen">
    <van-nav-bar title="录入错题" left-arrow @click-left="onClickLeft" fixed placeholder :border="false" />
    
    <div class="p-4">
      <!-- 分类选择 -->
      <div class="glass-panel mb-4 p-4">
        <div class="flex items-center justify-between mb-2">
          <span class="font-bold" style="color: var(--text-main)">选择科目与章节 <span class="text-red-500">*</span></span>
          <van-button size="mini" round @click="showCategoryPicker = true" class="custom-btn">
            {{ selectedCategory ? selectedCategory.name : '请选择' }}{{ selectedChapter ? ` - ${selectedChapter.name}` : '' }}
          </van-button>
        </div>
        <div v-if="categories.length === 0" class="text-xs text-red-500">
          暂无分类，请先在首页侧边栏添加分类。
        </div>
      </div>

      <!-- 顶部操作区 -->
      <div class="glass-panel mb-4 overflow-hidden" :class="{ 'opacity-50 pointer-events-none': !selectedCategory }">
        <van-tabs v-model:active="activeTab" type="card" class="pt-4 px-4" background="transparent" color="var(--text-main)" title-active-color="var(--app-bg)" title-inactive-color="var(--text-main)">
          <van-tab title="拍照识图">
            <div class="py-4">
              <div class="flex justify-between items-center mb-4">
                <span style="color: var(--text-main)">上传包含错题的图片</span>
                <div class="flex gap-2">
                  <van-uploader v-model="fileList" :max-count="1" accept="image/*" capture="camera" :after-read="afterRead" @delete="onDeleteImage">
                    <van-button icon="photograph" round size="small" :loading="recognizing" class="custom-btn">拍照</van-button>
                  </van-uploader>
                  <van-uploader v-model="fileList" :max-count="1" accept="image/*" :after-read="afterRead" @delete="onDeleteImage">
                    <van-button icon="photo" round size="small" :loading="recognizing" class="custom-btn">选图</van-button>
                  </van-uploader>
                </div>
              </div>
              <div class="flex items-center justify-between text-sm mb-2">
                <span style="color: var(--text-sub)">保存原图 (复习时可查看)</span>
                <van-switch v-model="saveImage" size="20px" active-color="var(--text-main)" />
              </div>
              <div class="text-xs" style="color: var(--text-sub)">
                提示：AI 会自动过滤做对的题，只提取有红笔批改或打叉的错题。
              </div>
            </div>
          </van-tab>
          
          <van-tab title="文本导入">
            <div class="py-4">
              <van-field
                v-model="importText"
                type="textarea"
                rows="6"
                placeholder="请粘贴题库文本。支持自动识别题号(如 1. 或 1、)和答案/解析关键字。"
                class="border rounded-md mb-4 !bg-transparent"
                style="border-color: var(--glass-border)"
              />
              <van-button block round @click="parseText" :disabled="!importText.trim()" class="custom-btn">
                正则解析文本
              </van-button>
            </div>
          </van-tab>
        </van-tabs>
      </div>

      <!-- 识别结果列表 -->
      <div v-if="questions.length > 0" class="space-y-4">
        <div class="flex justify-between items-center">
          <h3 class="font-bold" style="color: var(--text-main)">待保存错题 ({{ questions.length }})</h3>
          <van-button round size="mini" icon="plus" @click="addManualQuestion" class="custom-btn">手动添加一题</van-button>
        </div>

        <div v-for="(q, index) in questions" :key="q.tempId" class="glass-panel overflow-hidden">
          <div class="px-4 py-2 flex justify-between items-center border-b" style="border-color: var(--glass-border); background: rgba(0,0,0,0.05)">
            <span class="font-bold" style="color: var(--text-main)">题目 {{ index + 1 }}</span>
            <van-icon name="cross" class="p-1" style="color: var(--text-sub)" @click="removeQuestion(index)" />
          </div>
          
          <van-form class="!bg-transparent">
            <van-field
              v-model="q.type"
              is-link
              readonly
              label="题型"
              placeholder="选择题型"
              @click="openTypePicker(index)"
            />
            
            <van-field
              v-model="q.content"
              label="题干"
              type="textarea"
              rows="3"
              autosize
              placeholder="请输入题干内容"
            />
            
            <van-field
              v-if="q.type === '单选题' || q.type === '多选题'"
              v-model="q.options"
              label="选项"
              type="textarea"
              rows="3"
              autosize
              placeholder="请输入选项内容"
            />

            <van-field
              v-model="q.answer"
              label="答案"
              type="textarea"
              rows="2"
              autosize
              placeholder="请输入正确答案"
            />
            
            <van-field
              v-model="q.analysis"
              label="解析"
              type="textarea"
              rows="2"
              autosize
              placeholder="请输入解析"
            />
          </van-form>
        </div>
      </div>

      <!-- 空状态 -->
      <div v-else class="text-center py-10 glass-panel">
        <van-empty description="暂无待保存的错题" />
        <van-button round class="mt-4 custom-btn" @click="addManualQuestion">手动录入</van-button>
      </div>

      <!-- 底部保存按钮 -->
      <div v-if="questions.length > 0" class="fixed bottom-0 left-0 right-0 p-4 z-10" style="background: var(--nav-bg); backdrop-filter: var(--nav-blur); border-top: 1px solid var(--panel-border)">
        <van-button round block @click="saveAll" :loading="saving" class="custom-btn" style="background: var(--text-main) !important; color: var(--app-bg) !important;">
          保存全部 {{ questions.length }} 道错题
        </van-button>
      </div>
    </div>

    <!-- 题型选择器 -->
    <van-popup v-model:show="showTypePicker" position="bottom">
      <van-picker
        :columns="typeColumns"
        @confirm="onTypeConfirm"
        @cancel="showTypePicker = false"
      />
    </van-popup>

    <!-- 分类选择器 -->
    <van-popup v-model:show="showCategoryPicker" position="bottom">
      <van-picker
        :columns="categoryColumns"
        @confirm="onCategoryConfirm"
        @cancel="showCategoryPicker = false"
      />
    </van-popup>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import localforage from 'localforage'
import { showToast, showLoadingToast, closeToast, showDialog } from 'vant'

const router = useRouter()
const activeTab = ref(0)
const fileList = ref([])
const saveImage = ref(false)
const recognizing = ref(false)
const saving = ref(false)
const importText = ref('')

// 分类相关
const categories = ref([])
const selectedCategory = ref(null)
const selectedChapter = ref(null)
const showCategoryPicker = ref(false)

const categoryColumns = computed(() => {
  return categories.value.map(c => {
    const children = (c.chapters || []).map(chap => ({
      text: chap.name,
      value: chap.id
    }))
    
    // 如果没有章节，添加一个默认的“无章节”选项，以便级联选择器能正常工作
    if (children.length === 0) {
      children.push({ text: '无章节', value: 'none' })
    }
    
    return {
      text: c.name,
      value: c.id,
      children
    }
  })
})

onMounted(async () => {
  try {
    const storedCategories = await localforage.getItem('categories') || []
    categories.value = storedCategories
    if (categories.value.length > 0) {
      // 默认选中第一个
      selectedCategory.value = categories.value[0]
      if (categories.value[0].chapters && categories.value[0].chapters.length > 0) {
        selectedChapter.value = categories.value[0].chapters[0]
      }
    }
  } catch (err) {
    console.error('Failed to load categories:', err)
  }
})

// 待保存的题目列表
const questions = ref([])

// 题型选择器相关
const showTypePicker = ref(false)
const currentEditIndex = ref(-1)
const typeColumns = [
  { text: '单选题', value: '单选题' },
  { text: '多选题', value: '多选题' },
  { text: '判断题', value: '判断题' },
  { text: '填空题', value: '填空题' },
  { text: '简答题', value: '简答题' },
  { text: '论述题', value: '论述题' },
  { text: '其他', value: '其他' }
]

const onClickLeft = () => {
  if (questions.value.length > 0) {
    showDialog({
      title: '提示',
      message: '有未保存的错题，确定要离开吗？',
      showCancelButton: true,
    }).then(() => {
      router.push('/')
    }).catch(() => {})
  } else {
    router.push('/')
  }
}

const onDeleteImage = () => {
  // 如果用户删除了图片，可以选择清空识别结果，或者保留让用户手动编辑
  // 这里选择保留结果，仅清空图片
}

const addManualQuestion = () => {
  questions.value.push({
    tempId: Date.now().toString() + Math.random(),
    type: '单选题',
    content: '',
    options: '',
    answer: '',
    analysis: ''
  })
}

const removeQuestion = (index) => {
  questions.value.splice(index, 1)
}

const openTypePicker = (index) => {
  currentEditIndex.value = index
  showTypePicker.value = true
}

const onTypeConfirm = ({ selectedOptions }) => {
  if (currentEditIndex.value !== -1 && selectedOptions.length > 0) {
    questions.value[currentEditIndex.value].type = selectedOptions[0].value
  }
  showTypePicker.value = false
}

const onCategoryConfirm = ({ selectedOptions }) => {
  if (selectedOptions.length > 0) {
    const catId = selectedOptions[0].value
    selectedCategory.value = categories.value.find(c => c.id === catId)
    
    if (selectedOptions.length > 1 && selectedOptions[1].value !== 'none') {
      const chapId = selectedOptions[1].value
      selectedChapter.value = selectedCategory.value.chapters.find(chap => chap.id === chapId)
    } else {
      selectedChapter.value = null
    }
  }
  showCategoryPicker.value = false
}

const afterRead = async (file) => {
  const settings = JSON.parse(localStorage.getItem('ai_settings') || '{}')
  if (!settings.apiUrl || !settings.apiKey) {
    showToast('请先在设置中配置 API URL 和 API Key')
    return
  }

  recognizing.value = true
  const toast = showLoadingToast({
    message: 'AI 正在扫描错题...',
    forbidClick: true,
    duration: 0
  })

  try {
    const base64Image = file.content

    const prompt = `你是一个智能错题整理助手。用户上传了一张包含多道题目的试卷或作业图片。
请仔细观察图片，找出其中**有错误痕迹**（如红笔批改、打叉✗、订正痕迹、低分等）的题目。**忽略做对的题目**。
对于每一道错题，请提取以下信息，并严格以JSON数组的格式返回：
[
  {
    "type": "题型（如：单选题、多选题、判断题、填空题、简答题、论述题、其他）",
    "content": "题干完整内容。注意：如果是填空题，请将需要填空的地方统一替换为三个下划线 ___",
    "options": "选项完整内容（仅选择题需要，其他题型留空）",
    "answer": "正确答案。注意：如果是填空题，有多个空时，请严格使用双竖线 || 分隔每个空的答案（例如：答案1||答案2||答案3）。如果其中有几个空的顺序先后并不影响答案正确（如xx里包含A、B、C。此类型题目），请使用 && 连接这几个并列的答案（例如：A&&B&&C||固定答案3）。",
    "analysis": "根据原题做出的解析（不要根据图片中的错误答案做解析）"
  }
]
注意：
1. 必须返回JSON数组，即使只有一道错题。
2. 只返回JSON，不要包含任何其他说明文字或Markdown代码块标记（如\`\`\`json）。`

    const response = await fetch(`${settings.apiUrl}/chat/completions`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${settings.apiKey}`
      },
      body: JSON.stringify({
        model: settings.modelName || 'gpt-4o',
        messages: [
          {
            role: 'user',
            content: [
              { type: 'text', text: prompt },
              { type: 'image_url', image_url: { url: base64Image } }
            ]
          }
        ]
      })
    })

    if (!response.ok) {
      throw new Error(`API 请求失败: ${response.status}`)
    }

    const data = await response.json()
    let resultText = data.choices[0].message.content
    
    // 清理可能存在的 markdown 标记
    resultText = resultText.replace(/```json/g, '').replace(/```/g, '').trim()
    
    const resultArr = JSON.parse(resultText)
    
    if (Array.isArray(resultArr) && resultArr.length > 0) {
      // 将识别结果追加到列表中
      const newQuestions = resultArr.map(item => ({
        tempId: Date.now().toString() + Math.random(),
        type: item.type || '其他',
        content: item.content || '',
        options: item.options || '',
        answer: item.answer || '',
        analysis: item.analysis || ''
      }))
      
      questions.value = [...questions.value, ...newQuestions]
      showToast(`成功识别 ${newQuestions.length} 道错题`)
    } else {
      showToast('未在图片中发现错题')
    }
    
  } catch (error) {
    console.error('AI Recognition failed:', error)
    showToast('识别失败，请检查网络或API配置')
  } finally {
    recognizing.value = false
    closeToast()
  }
}

const parseText = () => {
  if (!importText.value.trim()) return
  
  const text = importText.value
  // 匹配题号，例如 "1." "1、" "1．" "一、"
  const questionRegex = /(?:^|\n)\s*(?:\d+|[一二三四五六七八九十]+)[\.、．]/g
  
  let match
  const indices = []
  while ((match = questionRegex.exec(text)) !== null) {
    indices.push(match.index)
  }
  
  if (indices.length === 0) {
    // 如果没有匹配到题号，就把整段文本作为一道题
    questions.value.push({
      tempId: Date.now().toString() + Math.random(),
      type: '其他',
      content: text.trim(),
      analysis: '',
      tagsInput: ''
    })
    showToast('未识别到题号，已作为单题导入')
    importText.value = ''
    return
  }
  
  const parsedQuestions = []
  for (let i = 0; i < indices.length; i++) {
    const start = indices[i]
    const end = i < indices.length - 1 ? indices[i + 1] : text.length
    let qText = text.substring(start, end).trim()
    
    // 尝试分离题目和解析
    // 匹配 "【答案】" "答案：" "【解析】" "解析："
    const ansRegex = /(?:^|\n)\s*(?:【答案】|答案[:：]|【解析】|解析[:：])/
    const ansMatch = ansRegex.exec(qText)
    
    let content = qText
    let analysis = ''
    
    if (ansMatch) {
      content = qText.substring(0, ansMatch.index).trim()
      analysis = qText.substring(ansMatch.index).trim()
    }
    
    // 简单推断题型
    let type = '其他'
    if (/[A-D][\.、．]/.test(content)) {
      type = '单选题' // 默认单选，用户可改
    } else if (content.includes('判断')) {
      type = '判断题'
    }
    
    parsedQuestions.push({
      tempId: Date.now().toString() + Math.random(),
      type,
      content,
      options: '',
      answer: '',
      analysis
    })
  }
  
  questions.value = [...questions.value, ...parsedQuestions]
  showToast(`成功解析 ${parsedQuestions.length} 道题目`)
  importText.value = '' // 清空输入框
}

const saveAll = async () => {
  if (!selectedCategory.value) {
    showToast('请先选择科目分类')
    return
  }

  // 简单校验
  const invalidIndex = questions.value.findIndex(q => !q.content.trim())
  if (invalidIndex !== -1) {
    showToast(`题目 ${invalidIndex + 1} 的内容不能为空`)
    return
  }

  saving.value = true
  try {
    const storedQuestions = await localforage.getItem('questions') || []
    const now = Date.now()
    
    // 获取图片内容（如果需要保存）
    const imageContent = saveImage.value && fileList.value.length > 0 ? fileList.value[0].content : null

    const newQuestionsToSave = questions.value.map((q, index) => {
      return {
        id: `${now}_${index}`,
        categoryId: selectedCategory.value.id,
        chapterId: selectedChapter.value ? selectedChapter.value.id : null,
        type: q.type,
        content: q.content,
        options: q.options,
        answer: q.answer,
        analysis: q.analysis,
        date: now,
        image: imageContent, // 批量录入的题目共享同一张原图
        reviewCount: 0,
        errorCount: 0,
        createdAt: now,
        lastReviewDate: null,
        nextReviewDate: now + 24 * 60 * 60 * 1000, // 默认明天复习
        reviewStage: 0
      }
    })
    
    const updatedQuestions = [...storedQuestions, ...newQuestionsToSave]
    await localforage.setItem('questions', updatedQuestions)
    
    showToast('保存成功')
    router.push('/')
  } catch (error) {
    console.error('Save failed:', error)
    showToast('保存失败')
  } finally {
    saving.value = false
  }
}
</script>
