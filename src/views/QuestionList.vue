<template>
  <div class="question-list pb-16 min-h-screen" :class="{ 'pb-24': isMultiSelectMode }">
    <van-nav-bar 
      :title="isMultiSelectMode ? `已选择 ${selectedIds.length} 项` : pageTitle" 
      :left-text="isMultiSelectMode ? '取消' : ''"
      :left-arrow="!isMultiSelectMode" 
      @click-left="onClickLeft" 
      fixed 
      placeholder 
      :border="false"
    >
      <template #right v-if="isMultiSelectMode">
        <span @click="toggleAll" style="color: var(--text-main)">{{ selectedIds.length === questions.length ? '全不选' : '全选' }}</span>
      </template>
    </van-nav-bar>
    
    <div class="p-4">
      <van-empty v-if="questions.length === 0" description="该分类下暂无此题型的错题" class="glass-panel" />
      
      <div v-else class="space-y-4">
        <div 
          v-for="q in questions" 
          :key="q.id" 
          class="glass-panel p-4 cursor-pointer flex items-center gap-3 select-none" 
          @click="onItemClick(q.id)"
          @touchstart="onTouchStart(q.id)"
          @touchend="onTouchEnd"
          @touchmove="onTouchMove"
        >
          <van-checkbox 
            v-if="isMultiSelectMode" 
            :name="q.id" 
            :model-value="selectedIds.includes(q.id)"
            @click.stop="toggleSelect(q.id)"
            checked-color="var(--text-main)"
          />
          <div class="flex-1 min-w-0">
            <div class="flex justify-between items-start mb-2">
              <div class="flex gap-2 flex-wrap">
                <van-tag class="custom-tag">{{ q.type || '其他' }}</van-tag>
              </div>
              <span class="text-sm whitespace-nowrap ml-2" style="color: var(--text-sub)">{{ formatDate(q.date) }}</span>
            </div>
            <p class="line-clamp-3" style="color: var(--text-main)">{{ q.content }}</p>
            <div class="mt-2 text-xs flex justify-between" style="color: var(--text-sub)">
              <span>回顾次数: {{ q.reviewCount || 0 }}</span>
              <span>错误次数: {{ q.errorCount || 0 }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 底部操作栏 -->
    <div v-if="isMultiSelectMode" class="fixed bottom-0 left-0 right-0 flex justify-between p-3 px-4 z-10 pb-safe" style="background: var(--nav-bg); backdrop-filter: var(--nav-blur); border-top: 1px solid var(--panel-border)">
      <van-button plain class="flex-1 mr-2" :disabled="selectedIds.length === 0" @click="batchDelete" style="border-color: #ef4444; color: #ef4444">删除</van-button>
      <van-button class="flex-1 ml-2 custom-btn" :disabled="selectedIds.length === 0" @click="openMovePicker">转移</van-button>
    </div>

    <!-- 转移分类选择器 -->
    <van-popup v-model:show="showMovePicker" position="bottom">
      <van-picker
        title="选择目标分类"
        :columns="categoryColumns"
        @confirm="onMoveConfirm"
        @cancel="showMovePicker = false"
      />
    </van-popup>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import localforage from 'localforage'
import { showConfirmDialog, showToast } from 'vant'

const route = useRoute()
const router = useRouter()
const questions = ref([])
const categoryName = ref('')
const chapterName = ref('')

// 多选相关状态
const isMultiSelectMode = ref(false)
const selectedIds = ref([])
let pressTimer = null

// 转移相关状态
const showMovePicker = ref(false)
const allCategories = ref([])

const categoryColumns = computed(() => {
  return allCategories.value.map(c => {
    const children = (c.chapters || []).map(chap => ({
      text: chap.name,
      value: chap.id
    }))
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

const pageTitle = computed(() => {
  const type = route.query.type || '全部'
  let title = ''
  if (categoryName.value) {
    title += categoryName.value
    if (chapterName.value) {
      title += ` - ${chapterName.value}`
    }
    title += ` - ${type}`
  } else {
    title = type
  }
  return title
})

const onClickLeft = () => {
  if (isMultiSelectMode.value) {
    isMultiSelectMode.value = false
    selectedIds.value = []
  } else {
    router.back()
  }
}

const goToDetail = (id) => {
  router.push(`/detail/${id}`)
}

const onItemClick = (id) => {
  if (isMultiSelectMode.value) {
    toggleSelect(id)
  } else {
    goToDetail(id)
  }
}

const onTouchStart = (id) => {
  if (isMultiSelectMode.value) return
  pressTimer = setTimeout(() => {
    isMultiSelectMode.value = true
    selectedIds.value = [id]
    if (navigator.vibrate) {
      navigator.vibrate(50)
    }
  }, 500)
}

const onTouchEnd = () => {
  if (pressTimer) {
    clearTimeout(pressTimer)
    pressTimer = null
  }
}

const onTouchMove = () => {
  if (pressTimer) {
    clearTimeout(pressTimer)
    pressTimer = null
  }
}

const toggleSelect = (id) => {
  const index = selectedIds.value.indexOf(id)
  if (index > -1) {
    selectedIds.value.splice(index, 1)
  } else {
    selectedIds.value.push(id)
  }
}

const toggleAll = () => {
  if (selectedIds.value.length === questions.value.length) {
    selectedIds.value = []
  } else {
    selectedIds.value = questions.value.map(q => q.id)
  }
}

const batchDelete = () => {
  showConfirmDialog({
    title: '批量删除',
    message: `确定要删除选中的 ${selectedIds.value.length} 道错题吗？此操作不可恢复。`,
  }).then(async () => {
    try {
      const storedQuestions = await localforage.getItem('questions') || []
      const newQuestions = storedQuestions.filter(q => !selectedIds.value.includes(q.id))
      await localforage.setItem('questions', newQuestions)
      
      questions.value = questions.value.filter(q => !selectedIds.value.includes(q.id))
      
      showToast('删除成功')
      isMultiSelectMode.value = false
      selectedIds.value = []
    } catch (err) {
      console.error('Failed to delete questions:', err)
      showToast('删除失败')
    }
  }).catch(() => {})
}

const openMovePicker = async () => {
  if (allCategories.value.length === 0) {
    allCategories.value = await localforage.getItem('categories') || []
  }
  showMovePicker.value = true
}

const onMoveConfirm = async ({ selectedOptions }) => {
  if (selectedOptions.length > 0) {
    const targetCategoryId = selectedOptions[0].value
    const targetChapterId = selectedOptions.length > 1 && selectedOptions[1].value !== 'none' ? selectedOptions[1].value : null
    
    try {
      const storedQuestions = await localforage.getItem('questions') || []
      
      const updatedQuestions = storedQuestions.map(q => {
        if (selectedIds.value.includes(q.id)) {
          return { ...q, categoryId: targetCategoryId, chapterId: targetChapterId }
        }
        return q
      })
      
      await localforage.setItem('questions', updatedQuestions)
      
      const categoryId = route.query.categoryId
      const chapterId = route.query.chapterId
      const type = route.query.type
      
      let filtered = updatedQuestions.filter(q => q.type === type)
      if (categoryId) {
        filtered = filtered.filter(q => q.categoryId === categoryId)
      }
      if (chapterId) {
        filtered = filtered.filter(q => q.chapterId === chapterId)
      }
      questions.value = filtered.sort((a, b) => b.date - a.date)
      
      showToast('转移成功')
      showMovePicker.value = false
      isMultiSelectMode.value = false
      selectedIds.value = []
    } catch (err) {
      console.error('Failed to move questions:', err)
      showToast('转移失败')
    }
  }
}

onMounted(async () => {
  const categoryId = route.query.categoryId
  const chapterId = route.query.chapterId
  const type = route.query.type
  
  try {
    // 获取分类和章节名称
    const categories = await localforage.getItem('categories') || []
    const category = categories.find(c => c.id === categoryId)
    if (category) {
      categoryName.value = category.name
      if (chapterId && category.chapters) {
        const chapter = category.chapters.find(c => c.id === chapterId)
        if (chapter) {
          chapterName.value = chapter.name
        }
      }
    }

    // 获取题目
    const storedQuestions = await localforage.getItem('questions') || []
    let filtered = storedQuestions.filter(q => q.type === type)
    
    if (categoryId) {
      filtered = filtered.filter(q => q.categoryId === categoryId)
    }
    if (chapterId) {
      filtered = filtered.filter(q => q.chapterId === chapterId)
    }
    
    questions.value = filtered.sort((a, b) => b.date - a.date)
  } catch (err) {
    console.error('Failed to load questions:', err)
  }
})

const formatDate = (timestamp) => {
  const date = new Date(timestamp)
  return `${date.getMonth() + 1}-${date.getDate()}`
}
</script>
