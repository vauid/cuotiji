<template>
  <div class="settings pb-16 min-h-screen">
    <van-nav-bar title="设置" fixed placeholder :border="false" />
    
    <div class="p-4">
      <van-cell-group inset title="外观设置" class="mb-4 !bg-transparent">
        <van-cell title="主题风格" class="glass-panel mb-2">
          <template #right-icon>
            <van-radio-group v-model="currentTheme" direction="horizontal" @change="onThemeChange">
              <van-radio name="light">极简纯白</van-radio>
              <van-radio name="scenery">风景通透</van-radio>
            </van-radio-group>
          </template>
        </van-cell>
      </van-cell-group>

      <van-form @submit="onSubmit">
        <van-cell-group inset title="AI 配置" class="!bg-transparent">
          <van-field
            v-model="currentPresetName"
            is-link
            readonly
            label="配置预设"
            placeholder="选择或保存预设"
            @click="showPresetPicker = true"
            class="glass-panel mb-2"
          />
          
          <van-field
            v-model="apiUrl"
            name="apiUrl"
            label="API URL"
            placeholder="例如: https://api.openai.com/v1"
            :rules="[{ required: true, message: '请填写API URL' }]"
            class="glass-panel mb-2"
          />
          <van-field
            v-model="apiKey"
            name="apiKey"
            label="API Key"
            type="password"
            placeholder="请输入您的 API Key"
            :rules="[{ required: true, message: '请填写API Key' }]"
            class="glass-panel mb-2"
          />
          
          <van-field
            v-model="modelName"
            is-link
            readonly
            name="modelName"
            label="模型名称"
            placeholder="点击选择模型"
            @click="showModelPicker = true"
            :rules="[{ required: true, message: '请选择模型' }]"
            class="glass-panel mb-2"
          >
            <template #button>
              <van-button size="small" type="primary" plain @click.stop="fetchModels" :loading="fetchingModels">
                获取模型
              </van-button>
            </template>
          </van-field>
        </van-cell-group>
        
        <div class="mt-4 flex justify-between px-4">
          <van-button size="small" type="primary" plain @click="saveAsPreset">保存为新预设</van-button>
          <van-button size="small" type="danger" plain @click="deletePreset" v-if="currentPresetName">删除当前预设</van-button>
        </div>

        <div class="mt-8">
          <van-button round block type="primary" native-type="submit">
            保存设置
          </van-button>
        </div>
      </van-form>
      
      <div class="mt-8">
        <van-button round block type="danger" plain @click="clearData">
          清除所有数据
        </van-button>
      </div>
    </div>
    
    <van-popup v-model:show="showModelPicker" position="bottom">
      <van-picker
        :columns="modelColumns"
        @confirm="onModelConfirm"
        @cancel="showModelPicker = false"
      />
    </van-popup>

    <van-popup v-model:show="showPresetPicker" position="bottom">
      <van-picker
        :columns="presetColumns"
        @confirm="onPresetConfirm"
        @cancel="showPresetPicker = false"
      />
    </van-popup>

    <van-dialog v-model:show="showSavePresetDialog" title="保存预设" show-cancel-button @confirm="confirmSavePreset">
      <van-field v-model="newPresetName" label="预设名称" placeholder="请输入预设名称" />
    </van-dialog>

    <van-tabbar route>
      <van-tabbar-item replace to="/" icon="home-o">首页</van-tabbar-item>
      <van-tabbar-item replace to="/add" icon="plus">录入</van-tabbar-item>
      <van-tabbar-item replace to="/practice" icon="edit">刷题</van-tabbar-item>
      <van-tabbar-item replace to="/settings" icon="setting-o">设置</van-tabbar-item>
    </van-tabbar>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { showToast, showDialog } from 'vant'
import localforage from 'localforage'

const apiUrl = ref('https://api.openai.com/v1')
const apiKey = ref('')
const modelName = ref('gpt-4o')

const showModelPicker = ref(false)
const modelColumns = ref([
  { text: 'gpt-4o', value: 'gpt-4o' },
  { text: 'gpt-3.5-turbo', value: 'gpt-3.5-turbo' }
])
const fetchingModels = ref(false)

const presets = ref([])
const currentPresetName = ref('')
const showPresetPicker = ref(false)
const showSavePresetDialog = ref(false)
const newPresetName = ref('')

const currentTheme = ref('light')

const presetColumns = computed(() => {
  return presets.value.map(p => ({ text: p.name, value: p.name }))
})

onMounted(() => {
  currentTheme.value = localStorage.getItem('app-theme') || 'light'
  
  const settings = JSON.parse(localStorage.getItem('ai_settings') || '{}')
  if (settings.apiUrl) apiUrl.value = settings.apiUrl
  if (settings.apiKey) apiKey.value = settings.apiKey
  if (settings.modelName) modelName.value = settings.modelName
  if (settings.models && Array.isArray(settings.models)) {
    modelColumns.value = settings.models.map(m => ({ text: m, value: m }))
  }
  
  const savedPresets = JSON.parse(localStorage.getItem('ai_presets') || '[]')
  presets.value = savedPresets
  if (settings.currentPresetName) {
    currentPresetName.value = settings.currentPresetName
  }
})

const saveAsPreset = () => {
  if (!apiUrl.value || !apiKey.value || !modelName.value) {
    showToast('请先填写完整的配置信息')
    return
  }
  newPresetName.value = ''
  showSavePresetDialog.value = true
}

const confirmSavePreset = () => {
  if (!newPresetName.value.trim()) {
    showToast('预设名称不能为空')
    return
  }
  
  const existingIndex = presets.value.findIndex(p => p.name === newPresetName.value.trim())
  const newPreset = {
    name: newPresetName.value.trim(),
    apiUrl: apiUrl.value,
    apiKey: apiKey.value,
    modelName: modelName.value
  }
  
  if (existingIndex >= 0) {
    presets.value[existingIndex] = newPreset
  } else {
    presets.value.push(newPreset)
  }
  
  localStorage.setItem('ai_presets', JSON.stringify(presets.value))
  currentPresetName.value = newPreset.name
  showToast('预设保存成功')
}

const onPresetConfirm = ({ selectedOptions }) => {
  if (selectedOptions && selectedOptions.length > 0) {
    const selectedName = selectedOptions[0].value
    const preset = presets.value.find(p => p.name === selectedName)
    if (preset) {
      currentPresetName.value = preset.name
      apiUrl.value = preset.apiUrl
      apiKey.value = preset.apiKey
      modelName.value = preset.modelName
    }
  }
  showPresetPicker.value = false
}

const deletePreset = () => {
  showDialog({
    title: '提示',
    message: `确定要删除预设 "${currentPresetName.value}" 吗？`,
    showCancelButton: true,
  }).then(() => {
    presets.value = presets.value.filter(p => p.name !== currentPresetName.value)
    localStorage.setItem('ai_presets', JSON.stringify(presets.value))
    currentPresetName.value = ''
    showToast('预设已删除')
  }).catch(() => {})
}

const fetchModels = async () => {
  if (!apiUrl.value || !apiKey.value) {
    showToast('请先填写 API URL 和 API Key')
    return
  }
  
  fetchingModels.value = true
  try {
    const response = await fetch(`${apiUrl.value}/models`, {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${apiKey.value}`
      }
    })
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }
    
    const data = await response.json()
    if (data && data.data && Array.isArray(data.data)) {
      const models = data.data.map(m => m.id).sort()
      modelColumns.value = models.map(m => ({ text: m, value: m }))
      
      // 保存模型列表到本地
      const settings = JSON.parse(localStorage.getItem('ai_settings') || '{}')
      settings.models = models
      localStorage.setItem('ai_settings', JSON.stringify(settings))
      
      showToast('获取模型成功')
      showModelPicker.value = true
    } else {
      showToast('获取模型失败：返回格式不正确')
    }
  } catch (error) {
    console.error('Fetch models error:', error)
    showToast('获取模型失败，请检查网络或配置')
  } finally {
    fetchingModels.value = false
  }
}

const onModelConfirm = ({ selectedOptions }) => {
  if (selectedOptions && selectedOptions.length > 0) {
    modelName.value = selectedOptions[0].value
  }
  showModelPicker.value = false
}

const onThemeChange = (val) => {
  localStorage.setItem('app-theme', val)
  document.body.setAttribute('data-theme', val)
}

const onSubmit = (values) => {
  const settings = JSON.parse(localStorage.getItem('ai_settings') || '{}')
  settings.apiUrl = apiUrl.value
  settings.apiKey = apiKey.value
  settings.modelName = modelName.value
  settings.currentPresetName = currentPresetName.value
  localStorage.setItem('ai_settings', JSON.stringify(settings))
  showToast('设置已保存')
}

const clearData = () => {
  showDialog({
    title: '警告',
    message: '确定要清除所有错题数据吗？此操作不可恢复。',
    showCancelButton: true,
  }).then(async () => {
    await localforage.clear()
    showToast('数据已清除')
  }).catch(() => {
    // on cancel
  })
}
</script>
