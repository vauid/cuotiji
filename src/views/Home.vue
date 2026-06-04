<template>
  <div class="home pb-16 min-h-screen">
    <van-nav-bar title="我的错题本" fixed placeholder :border="false">
      <template #left>
        <van-icon name="wap-nav" size="24" @click="showSidebar = true" />
      </template>
    </van-nav-bar>
    
    <div class="p-4">
      <!-- 今日复习任务卡片 -->
      <div class="glass-panel p-5 mb-6 relative overflow-hidden">
        <div class="relative z-10">
          <h3 class="text-lg font-bold mb-1" style="color: var(--text-main)">今日复习任务</h3>
          <p class="text-sm mb-4" style="color: var(--text-sub)">基于艾宾浩斯遗忘曲线智能安排</p>
          <div class="flex items-end justify-between">
            <div style="color: var(--text-main)">
              <span class="text-4xl font-black">{{ dueQuestionsCount }}</span>
              <span class="text-sm ml-1">题待复习</span>
            </div>
            <van-button 
              round 
              size="small" 
              class="custom-btn font-bold px-4"
              to="/practice"
            >
              去复习
            </van-button>
          </div>
        </div>
        <!-- 装饰背景 -->
        <van-icon name="clock-o" class="absolute -right-4 -bottom-4 text-8xl opacity-10" style="color: var(--text-main)" />
      </div>

      <div class="flex justify-between items-center mb-4 px-1">
        <h2 class="text-xl font-bold" style="color: var(--text-main)">
          {{ currentCategory ? currentCategory.name : '全部错题' }}
          {{ currentChapter ? ` - ${currentChapter.name}` : '' }}
        </h2>
        <van-button round size="small" to="/add" class="custom-btn">录入新题</van-button>
      </div>
      
      <van-empty v-if="categories.length === 0" description="还没有分类，请先在左侧菜单添加分类！" />
      <van-empty v-else-if="typeStats.length === 0" description="该分类下还没有错题，快去录入吧！" />
      
      <div v-else class="grid grid-cols-2 gap-4">
        <div v-for="stat in typeStats" :key="stat.type" class="glass-panel p-4 cursor-pointer flex flex-col items-center justify-center" @click="goToList(stat.type)">
          <div class="text-lg font-bold mb-2" style="color: var(--text-main)">{{ stat.type }}</div>
          <div class="text-3xl font-black mb-2" style="color: var(--text-main)">{{ stat.count }}</div>
          <div class="text-xs" style="color: var(--text-sub)">
            回顾错误率: <span :class="stat.errorRate > 50 ? 'text-red-500' : 'text-green-500'">{{ stat.errorRate }}%</span>
          </div>
        </div>
      </div>
    </div>
    
    <van-tabbar route>
      <van-tabbar-item replace to="/" icon="home-o">首页</van-tabbar-item>
      <van-tabbar-item replace to="/add" icon="plus">录入</van-tabbar-item>
      <van-tabbar-item replace to="/practice" icon="edit">刷题</van-tabbar-item>
      <van-tabbar-item replace to="/settings" icon="setting-o">设置</van-tabbar-item>
    </van-tabbar>

    <!-- 侧边栏 -->
    <van-popup v-model:show="showSidebar" position="left" :style="{ width: '80%', height: '100%' }">
      <div class="flex flex-col h-full">
        <div class="p-4 bg-blue-500 text-white text-lg font-bold">
          科目与章节
        </div>
        
        <div class="flex-1 overflow-y-auto">
          <van-cell 
            title="全部错题" 
            clickable 
            @click="selectCategory(null, null)"
            :class="{ 'bg-blue-50': !currentCategory }"
            title-class="text-gray-800 font-bold"
          >
            <template #right-icon v-if="!currentCategory">
              <van-icon name="success" class="text-blue-500 leading-inherit" />
            </template>
          </van-cell>

          <van-collapse v-model="activeCollapse" accordion>
            <van-collapse-item 
              v-for="cat in categories" 
              :key="cat.id" 
              :name="cat.id"
            >
              <template #title>
                <div class="flex items-center justify-between w-full" 
                     @click.stop="selectCategory(cat, null)"
                     @touchstart="handleTouchStart(cat, 'category')"
                     @touchend="handleTouchEnd"
                     @touchmove="handleTouchMove"
                     @contextmenu.prevent="handleContextMenu(cat, 'category')"
                >
                  <span :class="{ 'text-blue-600 font-bold': currentCategory && currentCategory.id === cat.id && !currentChapter }">
                    {{ cat.name }}
                  </span>
                  <van-icon v-if="currentCategory && currentCategory.id === cat.id && !currentChapter" name="success" class="text-blue-500 mr-2" />
                </div>
              </template>
              
              <van-cell-group :border="false">
                <van-cell 
                  v-for="chap in (cat.chapters || [])" 
                  :key="chap.id" 
                  :title="chap.name" 
                  clickable 
                  @click="selectCategory(cat, chap)"
                  @touchstart="handleTouchStart(chap, 'chapter', cat)"
                  @touchend="handleTouchEnd"
                  @touchmove="handleTouchMove"
                  @contextmenu.prevent="handleContextMenu(chap, 'chapter', cat)"
                  :class="{ 'bg-blue-50': currentChapter && currentChapter.id === chap.id }"
                  title-class="text-gray-600 text-sm pl-4"
                >
                  <template #right-icon v-if="currentChapter && currentChapter.id === chap.id">
                    <van-icon name="success" class="text-blue-500 leading-inherit" />
                  </template>
                </van-cell>
              </van-cell-group>
              
              <div class="mt-2 pl-4 pr-2 flex items-center">
                <van-field
                  v-model="newChapterNames[cat.id]"
                  placeholder="新增章节..."
                  size="small"
                  class="bg-gray-50 rounded px-2 py-1 flex-1"
                  :border="false"
                />
                <van-button size="mini" type="primary" plain class="ml-2" @click="addChapter(cat)" :disabled="!newChapterNames[cat.id]">添加</van-button>
              </div>
            </van-collapse-item>
          </van-collapse>
        </div>
        
        <div class="p-4 border-t bg-gray-50">
          <div class="text-sm text-gray-500 mb-2">新增科目分类</div>
          <van-field
            v-model="newCategoryName"
            center
            clearable
            placeholder="如：数学、物理"
            class="border rounded"
          >
            <template #button>
              <van-button size="small" type="primary" @click="addCategory" :disabled="!newCategoryName.trim()">新增</van-button>
            </template>
          </van-field>
        </div>
      </div>
    </van-popup>

    <!-- 动作面板 -->
    <van-action-sheet
      v-model:show="showActionSheet"
      :actions="actionSheetActions"
      cancel-text="取消"
      close-on-click-action
      @select="onActionSelect"
    />

    <!-- 重命名弹窗 -->
    <van-dialog
      v-model:show="showRenameDialog"
      title="重命名"
      show-cancel-button
      @confirm="confirmRename"
    >
      <div class="p-4">
        <van-field
          v-model="renameValue"
          placeholder="请输入新名称"
          class="border rounded"
          clearable
        />
      </div>
    </van-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import localforage from 'localforage'
import { showToast, showConfirmDialog } from 'vant'

const router = useRouter()
const questions = ref([])
const categories = ref([])
const showSidebar = ref(false)
const currentCategory = ref(null)
const currentChapter = ref(null)
const newCategoryName = ref('')
const newChapterNames = ref({})
const activeCollapse = ref('')

// 长按相关状态
const showActionSheet = ref(false)
const actionSheetActions = [
  { name: '重命名' },
  { name: '删除', color: '#ee0a24' }
]
const currentLongPressItem = ref(null)
const currentLongPressType = ref('')
const currentLongPressParent = ref(null)

// 重命名相关状态
const showRenameDialog = ref(false)
const renameValue = ref('')

let pressTimer = null
let isLongPress = false

const handleTouchStart = (item, type, parent = null) => {
  isLongPress = false
  pressTimer = setTimeout(() => {
    isLongPress = true
    currentLongPressItem.value = item
    currentLongPressType.value = type
    currentLongPressParent.value = parent
    showActionSheet.value = true
  }, 800)
}

const handleTouchEnd = () => {
  if (pressTimer) {
    clearTimeout(pressTimer)
    pressTimer = null
  }
}

const handleTouchMove = () => {
  if (pressTimer) {
    clearTimeout(pressTimer)
    pressTimer = null
  }
}

const handleContextMenu = (item, type, parent = null) => {
  currentLongPressItem.value = item
  currentLongPressType.value = type
  currentLongPressParent.value = parent
  showActionSheet.value = true
}

const onActionSelect = (action) => {
  if (action.name === '重命名') {
    renameValue.value = currentLongPressItem.value.name
    showRenameDialog.value = true
  } else if (action.name === '删除') {
    handleDelete()
  }
}

const confirmRename = async () => {
  const newName = renameValue.value.trim()
  if (!newName) {
    showToast('名称不能为空')
    return
  }
  
  if (currentLongPressType.value === 'category') {
    if (categories.value.some(c => c.id !== currentLongPressItem.value.id && c.name === newName)) {
      showToast('分类已存在')
      return
    }
    currentLongPressItem.value.name = newName
  } else {
    const parent = currentLongPressParent.value
    if (parent.chapters.some(c => c.id !== currentLongPressItem.value.id && c.name === newName)) {
      showToast('章节已存在')
      return
    }
    currentLongPressItem.value.name = newName
  }
  
  await localforage.setItem('categories', JSON.parse(JSON.stringify(categories.value)))
  showToast('重命名成功')
}

const handleDelete = () => {
  const itemName = currentLongPressItem.value.name
  const typeName = currentLongPressType.value === 'category' ? '科目' : '章节'
  
  showConfirmDialog({
    title: '确认删除',
    message: `确定要删除${typeName}【${itemName}】吗？该${typeName}下的所有错题将被一并删除！`,
  }).then(async () => {
    if (currentLongPressType.value === 'category') {
      categories.value = categories.value.filter(c => c.id !== currentLongPressItem.value.id)
      questions.value = questions.value.filter(q => q.categoryId !== currentLongPressItem.value.id)
      
      if (currentCategory.value && currentCategory.value.id === currentLongPressItem.value.id) {
        selectCategory(null, null)
      }
    } else {
      const parent = currentLongPressParent.value
      parent.chapters = parent.chapters.filter(c => c.id !== currentLongPressItem.value.id)
      questions.value = questions.value.filter(q => q.chapterId !== currentLongPressItem.value.id)
      
      if (currentChapter.value && currentChapter.value.id === currentLongPressItem.value.id) {
        selectCategory(parent, null)
      }
    }
    
    await localforage.setItem('categories', JSON.parse(JSON.stringify(categories.value)))
    await localforage.setItem('questions', JSON.parse(JSON.stringify(questions.value)))
    showToast('删除成功')
  }).catch(() => {})
}

const dueQuestionsCount = computed(() => {
  const now = Date.now()
  return questions.value.filter(q => !q.nextReviewDate || q.nextReviewDate <= now).length
})

const loadData = async () => {
  try {
    const storedCategories = await localforage.getItem('categories') || []
    // 确保旧数据有 chapters 数组
    categories.value = storedCategories.map(c => ({
      ...c,
      chapters: c.chapters || []
    }))
    
    const storedQuestions = await localforage.getItem('questions') || []
    
    // 数据迁移：为旧数据添加遗忘曲线字段
    let needsSave = false
    const now = Date.now()
    const migratedQuestions = storedQuestions.map(q => {
      if (q.nextReviewDate === undefined) {
        needsSave = true
        return {
          ...q,
          createdAt: q.date || now,
          lastReviewDate: null,
          nextReviewDate: now, // 旧题直接设为今天需要复习
          reviewStage: 0
        }
      }
      return q
    })
    
    if (needsSave) {
      await localforage.setItem('questions', migratedQuestions)
    }
    
    questions.value = migratedQuestions

    // 恢复上次选择的分类和章节
    const lastCategoryId = localStorage.getItem('lastCategoryId')
    const lastChapterId = localStorage.getItem('lastChapterId')
    
    if (lastCategoryId) {
      const cat = categories.value.find(c => c.id === lastCategoryId)
      if (cat) {
        currentCategory.value = cat
        if (lastChapterId) {
          const chap = cat.chapters.find(c => c.id === lastChapterId)
          if (chap) {
            currentChapter.value = chap
          }
        }
      }
    }
  } catch (err) {
    console.error('Failed to load data:', err)
  }
}

onMounted(() => {
  loadData()
})

const addCategory = async () => {
  const name = newCategoryName.value.trim()
  if (!name) return
  
  if (categories.value.some(c => c.name === name)) {
    showToast('分类已存在')
    return
  }
  
  const newCat = {
    id: Date.now().toString(),
    name: name,
    chapters: []
  }
  
  categories.value.push(newCat)
  // 使用深拷贝确保 Vue 的响应式代理对象被正确序列化
  await localforage.setItem('categories', JSON.parse(JSON.stringify(categories.value)))
  newCategoryName.value = ''
  showToast('添加成功')
}

const addChapter = async (cat) => {
  const chapterName = (newChapterNames.value[cat.id] || '').trim()
  if (!chapterName) return
  
  if (!cat.chapters) {
    cat.chapters = []
  }
  
  if (cat.chapters.some(c => c.name === chapterName)) {
    showToast('章节已存在')
    return
  }
  
  cat.chapters.push({
    id: Date.now().toString(),
    name: chapterName
  })
  
  // 使用深拷贝确保 Vue 的响应式代理对象被正确序列化
  await localforage.setItem('categories', JSON.parse(JSON.stringify(categories.value)))
  newChapterNames.value[cat.id] = ''
  showToast('添加章节成功')
}

const selectCategory = (cat, chap) => {
  if (isLongPress) {
    isLongPress = false
    return
  }
  
  currentCategory.value = cat
  currentChapter.value = chap
  showSidebar.value = false
  
  // 保存选择状态到 localStorage
  if (cat) {
    localStorage.setItem('lastCategoryId', cat.id)
  } else {
    localStorage.removeItem('lastCategoryId')
  }
  
  if (chap) {
    localStorage.setItem('lastChapterId', chap.id)
  } else {
    localStorage.removeItem('lastChapterId')
  }
}

const typeStats = computed(() => {
  let filteredQuestions = questions.value
  if (currentCategory.value) {
    filteredQuestions = filteredQuestions.filter(q => q.categoryId === currentCategory.value.id)
  }
  if (currentChapter.value) {
    filteredQuestions = filteredQuestions.filter(q => q.chapterId === currentChapter.value.id)
  }
  
  const statsMap = {}
  
  filteredQuestions.forEach(q => {
    const type = q.type || '其他'
    if (!statsMap[type]) {
      statsMap[type] = {
        type: type,
        count: 0,
        totalReview: 0,
        totalError: 0
      }
    }
    statsMap[type].count++
    statsMap[type].totalReview += (q.reviewCount || 0)
    statsMap[type].totalError += (q.errorCount || 0)
  })
  
  return Object.values(statsMap).map(stat => {
    let errorRate = 0
    if (stat.totalReview > 0) {
      errorRate = Math.round((stat.totalError / stat.totalReview) * 100)
    }
    return {
      ...stat,
      errorRate
    }
  }).sort((a, b) => b.count - a.count)
})

const goToList = (type) => {
  const query = { type }
  if (currentCategory.value) {
    query.categoryId = currentCategory.value.id
  }
  if (currentChapter.value) {
    query.chapterId = currentChapter.value.id
  }
  router.push({ path: '/list', query })
}
</script>
