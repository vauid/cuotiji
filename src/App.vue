<template>
  <router-view />
</template>

<script setup>
import { onMounted } from 'vue'

onMounted(() => {
  // 初始化主题
  const savedTheme = localStorage.getItem('app-theme') || 'light'
  document.body.setAttribute('data-theme', savedTheme)
})
</script>

<style>
/* 全局样式已移至 style.css */
</style>
