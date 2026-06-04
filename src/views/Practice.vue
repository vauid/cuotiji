<template>
  <div class="practice pb-16 min-h-screen">
    <van-nav-bar title="刷题模式" left-arrow @click-left="onClickLeft" fixed placeholder :border="false" />

    <!-- 配置阶段 -->
    <div v-if="!isPracticing" class="p-4">
      <!-- 智能复习模式 -->
      <div class="glass-panel p-4 mb-4">
        <div class="flex justify-between items-center mb-2">
          <h3 class="font-bold text-lg" style="color: var(--text-main)">智能复习 (今日任务)</h3>
          <van-tag round size="medium" class="custom-tag">{{ dueQuestions.length }} 题待复习</van-tag>
        </div>
        <p class="text-sm mb-4" style="color: var(--text-sub)">基于艾宾浩斯遗忘曲线，智能安排每日复习任务，高效巩固错题。</p>
        <van-button 
          block 
          round 
          @click="startSmartPractice" 
          :disabled="dueQuestions.length === 0"
          class="custom-btn"
        >
          {{ dueQuestions.length > 0 ? '开始今日复习' : '今日任务已完成' }}
        </van-button>
      </div>

      <div class="glass-panel p-4 mb-4">
        <h3 class="font-bold text-lg mb-4" style="color: var(--text-main)">按范围刷题</h3>
        
        <van-empty v-if="categories.length === 0" description="暂无分类，请先添加错题" />
        
        <van-checkbox-group v-model="selectedScope" v-else>
          <div v-for="cat in categories" :key="cat.id" class="mb-4">
            <div class="font-bold mb-2 border-b pb-1" style="color: var(--text-main); border-color: var(--glass-border)">{{ cat.name }}</div>
            
            <!-- 如果没有章节，直接选科目 -->
            <van-checkbox 
              v-if="!cat.chapters || cat.chapters.length === 0" 
              :name="`cat_${cat.id}`" 
              class="mb-2 ml-2"
            >
              全部 {{ cat.name }}
            </van-checkbox>
            
            <!-- 如果有章节，列出章节供多选 -->
            <template v-else>
              <van-checkbox 
                v-for="chap in cat.chapters" 
                :key="chap.id" 
                :name="`chap_${chap.id}`" 
                class="mb-2 ml-2"
              >
                {{ chap.name }}
              </van-checkbox>
            </template>
          </div>
        </van-checkbox-group>
      </div>

      <van-button 
        block 
        round 
        @click="startPractice" 
        :disabled="selectedScope.length === 0"
        class="custom-btn"
      >
        开始刷题
      </van-button>
    </div>

    <!-- 答题阶段 -->
    <div v-else class="p-4">
      <div v-if="practiceQuestions.length > 0 && currentIndex < practiceQuestions.length">
        <!-- 进度条 -->
        <div class="flex justify-between items-center mb-4 text-sm" style="color: var(--text-sub)">
          <span>进度: {{ currentIndex + 1 }} / {{ practiceQuestions.length }}</span>
          <span>正确率: {{ correctCount }} / {{ currentIndex }}</span>
        </div>

        <!-- 题目卡片 -->
        <div class="glass-panel p-4 mb-4">
          <div class="flex gap-2 mb-3 flex-wrap">
            <van-tag class="custom-tag">{{ currentQuestion.type || '其他' }}</van-tag>
          </div>
          
          <!-- 题干渲染 -->
          <div class="text-lg mb-4 leading-relaxed" style="color: var(--text-main)">
            <template v-if="isNewFillInTheBlank(currentQuestion)">
              <span v-for="(part, index) in splitContent(currentQuestion.content)" :key="index">
                {{ part }}
                <input 
                  v-if="index < splitContent(currentQuestion.content).length - 1"
                  v-model="userAnswers[index]"
                  type="text"
                  class="border-b-2 outline-none px-1 mx-1 w-20 text-center bg-transparent"
                  :style="showResult ? (blankResults[index] ? 'border-color: #10b981; color: #10b981' : 'border-color: #ef4444; color: #ef4444') : 'border-color: var(--text-main); color: var(--text-main)'"
                  :disabled="showResult"
                />
              </span>
            </template>
            <template v-else>
              <p class="whitespace-pre-wrap">{{ currentQuestion.content }}</p>
            </template>
          </div>
          
          <!-- 选项 (如果是选择题) -->
          <div v-if="currentQuestion.options" class="mt-4">
            <p class="whitespace-pre-wrap" style="color: var(--text-main)">{{ currentQuestion.options }}</p>
          </div>

          <!-- 图片 -->
          <div v-if="currentQuestion.image" class="mt-4">
            <van-image :src="currentQuestion.image" fit="contain" @click="showImagePreview([currentQuestion.image])" />
          </div>
        </div>

        <!-- 交互区 -->
        <div v-if="!showResult" class="mt-6">
          <!-- 判断题 -->
          <div v-if="currentQuestion.type === '判断题'" class="flex gap-4">
            <van-button block @click="handleTrueFalse('正确')" class="custom-btn">√ 正确</van-button>
            <van-button block @click="handleTrueFalse('错误')" class="custom-btn">× 错误</van-button>
          </div>

          <!-- 单选题 -->
          <div v-else-if="currentQuestion.type === '单选题'" class="flex flex-wrap gap-3 justify-center">
            <van-button 
              v-for="opt in availableOptions" 
              :key="opt"
              class="w-16 custom-btn"
              @click="handleSingleChoice(opt)"
            >
              {{ opt }}
            </van-button>
          </div>

          <!-- 多选题 -->
          <div v-else-if="currentQuestion.type === '多选题'" class="flex flex-col items-center">
            <div class="flex flex-wrap gap-3 justify-center mb-4">
              <van-button 
                v-for="opt in availableOptions" 
                :key="opt"
                class="w-16 custom-btn"
                @click="toggleMultiChoice(opt)"
                :style="userMultiChoice.includes(opt) ? 'opacity: 0.5' : ''"
              >
                {{ opt }}
              </van-button>
            </div>
            <van-button block round @click="submitAnswer" :disabled="userMultiChoice.length === 0" class="w-full custom-btn">确定选项</van-button>
          </div>

          <!-- 填空题提交按钮 -->
          <van-button v-else-if="isNewFillInTheBlank(currentQuestion)" block round @click="submitAnswer" class="custom-btn">提交答案</van-button>

          <!-- 主观题自评按钮 -->
          <van-button v-if="isSubjective(currentQuestion)" block round :class="{'mt-4': isNewFillInTheBlank(currentQuestion)}" @click="showAnswerDirectly" class="custom-btn">直接看答案 (自评)</van-button>
        </div>

        <!-- 结果展示区 -->
        <div v-if="showResult" class="mt-4">
          <div class="glass-panel p-4 mb-4" :style="isCorrect ? 'border-color: #10b981' : 'border-color: #ef4444'">
            <div class="font-bold text-lg mb-2" :style="isCorrect ? 'color: #10b981' : 'color: #ef4444'">
              {{ isCorrect ? '回答正确！' : '回答错误' }}
            </div>
            <div v-if="!isCorrect || isSubjective(currentQuestion)">
              <div class="mb-2">
                <span class="font-bold" style="color: var(--text-sub)">标准答案：</span>
                <span class="whitespace-pre-wrap" style="color: var(--text-main)">{{ currentQuestion.answer || '暂无' }}</span>
              </div>
              <div>
                <span class="font-bold" style="color: var(--text-sub)">解析：</span>
                <span class="whitespace-pre-wrap" style="color: var(--text-main)">{{ currentQuestion.analysis || '暂无' }}</span>
              </div>
            </div>
          </div>

          <!-- 自评按钮 (如果是主观题或旧版填空题直接看答案的情况) -->
          <div v-if="isSelfEval" class="flex gap-4 mb-4">
            <van-button block @click="recordSelfEval(true)" style="background: #10b981; color: white; border: none">我答对了</van-button>
            <van-button block @click="recordSelfEval(false)" style="background: #ef4444; color: white; border: none">我答错了</van-button>
          </div>

          <van-button v-else block round @click="nextQuestion" class="custom-btn">
            {{ currentIndex < practiceQuestions.length - 1 ? '下一题' : '完成刷题' }}
          </van-button>
        </div>
      </div>

      <!-- 刷题完成 -->
      <div v-else-if="practiceQuestions.length > 0 && currentIndex >= practiceQuestions.length" class="text-center py-10 glass-panel">
        <van-icon name="checked" size="60" color="var(--text-main)" />
        <h2 class="text-xl font-bold mt-4 mb-2" style="color: var(--text-main)">刷题完成！</h2>
        <p class="mb-6" style="color: var(--text-sub)">本次共刷题 {{ practiceQuestions.length }} 道，答对 {{ correctCount }} 道。</p>
        <van-button round @click="resetPractice" class="custom-btn">再刷一次</van-button>
        <van-button round class="ml-4 custom-btn" @click="onClickLeft">返回首页</van-button>
      </div>
      
      <van-empty v-else description="所选范围内没有错题" />
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import localforage from 'localforage'
import { showToast, showImagePreview } from 'vant'

const router = useRouter()

// 配置相关
const categories = ref([])
const selectedScope = ref([]) // 存放选中的 cat_id 或 chap_id
const isPracticing = ref(false)
const isSmartMode = ref(false) // 是否是智能复习模式

// 刷题数据
const allQuestions = ref([])
const practiceQuestions = ref([])
const dueQuestions = computed(() => {
  const now = Date.now()
  return allQuestions.value.filter(q => {
    // 如果没有 nextReviewDate，或者 nextReviewDate 小于等于当前时间，则需要复习
    return !q.nextReviewDate || q.nextReviewDate <= now
  })
})
const currentIndex = ref(0)
const correctCount = ref(0)

// 当前题目状态
const userAnswers = ref([]) // 填空题答案数组
const blankResults = ref([]) // 填空题每个空的判题结果
const userChoice = ref('') // 单选/判断题答案
const userMultiChoice = ref([]) // 多选题答案数组
const showResult = ref(false)
const isCorrect = ref(false)
const isSelfEval = ref(false) // 是否处于自评模式

const currentQuestion = computed(() => {
  return practiceQuestions.value[currentIndex.value] || {}
})

const availableOptions = computed(() => {
  const q = currentQuestion.value
  if (!q || !q.options) return ['A', 'B', 'C', 'D']
  
  const opts = ['A', 'B', 'C', 'D']
  const optionsText = q.options.toUpperCase()
  if (optionsText.includes('E')) opts.push('E')
  if (optionsText.includes('F')) opts.push('F')
  if (optionsText.includes('G')) opts.push('G')
  return opts
})

const toggleMultiChoice = (opt) => {
  const index = userMultiChoice.value.indexOf(opt)
  if (index === -1) {
    userMultiChoice.value.push(opt)
  } else {
    userMultiChoice.value.splice(index, 1)
  }
}

const handleSingleChoice = (opt) => {
  userChoice.value = opt
  submitAnswer()
}

const handleTrueFalse = (val) => {
  userChoice.value = val
  submitAnswer()
}

onMounted(async () => {
  try {
    categories.value = await localforage.getItem('categories') || []
    allQuestions.value = await localforage.getItem('questions') || []
  } catch (err) {
    console.error('Failed to load data:', err)
  }
})

const onClickLeft = () => {
  if (isPracticing.value && currentIndex.value < practiceQuestions.length) {
    if (confirm('正在刷题中，确定要退出吗？进度将不会保存。')) {
      router.push('/')
    }
  } else {
    router.push('/')
  }
}

const startSmartPractice = () => {
  if (dueQuestions.value.length === 0) {
    showToast('今日任务已完成')
    return
  }
  
  practiceQuestions.value = [...dueQuestions.value].sort(() => Math.random() - 0.5)
  currentIndex.value = 0
  correctCount.value = 0
  isPracticing.value = true
  isSmartMode.value = true
  resetCurrentState()
}

const startPractice = () => {
  // 根据 selectedScope 筛选题目
  let filtered = []
  
  const selectedCatIds = selectedScope.value.filter(s => s.startsWith('cat_')).map(s => s.replace('cat_', ''))
  const selectedChapIds = selectedScope.value.filter(s => s.startsWith('chap_')).map(s => s.replace('chap_', ''))

  filtered = allQuestions.value.filter(q => {
    // 如果题目所属的科目被全选
    if (selectedCatIds.includes(q.categoryId)) return true
    // 如果题目所属的章节被选中
    if (selectedChapIds.includes(q.chapterId)) return true
    return false
  })

  if (filtered.length === 0) {
    showToast('所选范围内没有错题')
    return
  }

  // 打乱顺序
  practiceQuestions.value = filtered.sort(() => Math.random() - 0.5)
  currentIndex.value = 0
  correctCount.value = 0
  isPracticing.value = true
  resetCurrentState()
}

const resetCurrentState = () => {
  userAnswers.value = []
  blankResults.value = []
  userChoice.value = ''
  userMultiChoice.value = []
  showResult.value = false
  isCorrect.value = false
  isSelfEval.value = false
}

// 判断是否是新版填空题（包含 ___）
const isNewFillInTheBlank = (q) => {
  return q.type === '填空题' && q.content && q.content.includes('___')
}

// 判断是否是主观题或旧版填空题（难以自动判题）
const isSubjective = (q) => {
  if (q.type === '简答题' || q.type === '论述题' || q.type === '其他') return true
  if (q.type === '填空题' && !isNewFillInTheBlank(q)) return true
  return false
}

// 分割题干用于渲染填空输入框
const splitContent = (content) => {
  if (!content) return []
  return content.split('___')
}

const submitAnswer = async () => {
  const q = currentQuestion.value
  let correct = false

  if (isNewFillInTheBlank(q)) {
    // 新版填空题判题
    const standardAnswers = (q.answer || '').split('||').map(s => s.trim())
    
    let allMatch = true
    let userAnswerIndex = 0
    blankResults.value = []
    
    for (let i = 0; i < standardAnswers.length; i++) {
      const saPart = standardAnswers[i]
      
      // 检查是否是并列项
      if (saPart.includes('&&')) {
        const parallelAnswers = saPart.split('&&').map(s => s.trim())
        const parallelCount = parallelAnswers.length
        
        // 获取用户对应的几个答案
        const userParallelAnswers = []
        for (let j = 0; j < parallelCount; j++) {
          userParallelAnswers.push((userAnswers.value[userAnswerIndex + j] || '').trim())
        }
        
        let availableSA = [...parallelAnswers]
        let parallelMatch = true
        
        for (let j = 0; j < parallelCount; j++) {
          const ua = userParallelAnswers[j]
          const matchIndex = availableSA.indexOf(ua)
          if (matchIndex !== -1) {
            blankResults.value[userAnswerIndex + j] = true
            availableSA.splice(matchIndex, 1)
          } else {
            blankResults.value[userAnswerIndex + j] = false
            parallelMatch = false
          }
        }
        
        if (!parallelMatch) {
          allMatch = false
        }
        
        userAnswerIndex += parallelCount
      } else {
        // 普通顺序项
        const ua = (userAnswers.value[userAnswerIndex] || '').trim()
        if (ua === saPart) {
          blankResults.value[userAnswerIndex] = true
        } else {
          blankResults.value[userAnswerIndex] = false
          allMatch = false
        }
        userAnswerIndex++
      }
    }
    
    // 确保用户填写的答案数量与标准答案要求的总空数一致
    correct = allMatch && standardAnswers.length > 0 && userAnswerIndex === userAnswers.value.length
  } else if (q.type === '单选题') {
    const ua = userChoice.value.trim().toUpperCase()
    const sa = (q.answer || '').trim().toUpperCase()
    // 提取标准答案中的字母
    const saMatch = sa.match(/[A-G]/)
    const saLetter = saMatch ? saMatch[0] : sa
    correct = ua === saLetter && saLetter !== ''
  } else if (q.type === '多选题') {
    const ua = [...userMultiChoice.value].sort().join('')
    const sa = (q.answer || '').toUpperCase().replace(/[^A-G]/g, '').split('').sort().join('')
    correct = ua === sa && sa !== ''
  } else if (q.type === '判断题') {
    const ua = userChoice.value // '正确' 或 '错误'
    const sa = (q.answer || '').trim().toUpperCase()
    
    const isSaTrue = /对|√|T|正确|是/.test(sa)
    const isSaFalse = /错|×|X|F|错误|否/.test(sa)
    
    if (ua === '正确' && isSaTrue) correct = true
    else if (ua === '错误' && isSaFalse) correct = true
    else correct = false
  } else {
    // 其他题型，强制进入自评
    showAnswerDirectly()
    return
  }

  isCorrect.value = correct
  showResult.value = true

  await updateQuestionStats(correct)

  if (correct) {
    correctCount.value++
    showToast('回答正确')
    // 答对自动下一题，延迟一下让用户看到正确提示
    setTimeout(() => {
      nextQuestion()
    }, 1000)
  }
}

const showAnswerDirectly = () => {
  showResult.value = true
  isSelfEval.value = true
}

const recordSelfEval = async (correct) => {
  isCorrect.value = correct
  isSelfEval.value = false // 结束自评状态，显示下一题按钮
  
  await updateQuestionStats(correct)
  
  if (correct) {
    correctCount.value++
    nextQuestion()
  }
}

const updateQuestionStats = async (correct) => {
  const q = currentQuestion.value
  
  // 更新内存中的数据
  const indexInAll = allQuestions.value.findIndex(item => item.id === q.id)
  if (indexInAll !== -1) {
    const targetQ = allQuestions.value[indexInAll]
    targetQ.reviewCount = (targetQ.reviewCount || 0) + 1
    if (!correct) {
      targetQ.errorCount = (targetQ.errorCount || 0) + 1
    }
    
    // 遗忘曲线逻辑
    if (isSmartMode.value) {
      const intervals = [1, 2, 4, 7, 15, 30] // 复习间隔天数
      let currentStage = targetQ.reviewStage || 0
      
      if (correct) {
        currentStage = Math.min(currentStage + 1, intervals.length - 1)
      } else {
        currentStage = Math.max(currentStage - 1, 0) // 答错退回上一阶段
      }
      
      targetQ.reviewStage = currentStage
      targetQ.lastReviewDate = Date.now()
      const daysToAdd = intervals[currentStage]
      targetQ.nextReviewDate = Date.now() + daysToAdd * 24 * 60 * 60 * 1000
    }
    
    // 保存到 localforage
    try {
      await localforage.setItem('questions', JSON.parse(JSON.stringify(allQuestions.value)))
    } catch (err) {
      console.error('Failed to update stats:', err)
    }
  }
}

const nextQuestion = () => {
  currentIndex.value++
  resetCurrentState()
}

const resetPractice = () => {
  isPracticing.value = false
  isSmartMode.value = false
  selectedScope.value = []
}
</script>
