<template>
  <div class="result-section">
    <h3>轉換結果</h3>
    <div class="conversion-summary">
      <p><strong>轉換完成！</strong></p>
      <p>成功轉換：{{ results.successCount }}/{{ results.totalCount }} 首歌曲</p>
      <p>播放清單名稱：{{ playlistName }}</p>
    </div>
    <div class="song-list">
      <div 
        v-for="song in results.songs" 
        :key="song.name"
        class="song-item"
      >
        <span>{{ song.name }}</span>
        <span :class="['song-status', song.status === 'success' ? 'status-success' : 'status-failed']">
          {{ song.status === 'success' ? '✅ 成功' : '❌ 未找到' }}
        </span>
      </div>
    </div>
    <button 
      class="btn btn-spotify" 
      style="display:inline-flex; align-items:center;"
      @click="openPlaylist"
    >
      <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" style="margin-right: 8px" fill="currentColor" class="bi bi-spotify" viewBox="0 0 16 16"><path d="M8 0a8 8 0 1 0 0 16A8 8 0 0 0 8 0m3.669 11.538a.5.5 0 0 1-.686.165c-1.879-1.147-4.243-1.407-7.028-.77a.499.499 0 0 1-.222-.973c3.048-.696 5.662-.397 7.77.892a.5.5 0 0 1 .166.686m.979-2.178a.624.624 0 0 1-.858.205c-2.15-1.321-5.428-1.704-7.972-.932a.625.625 0 0 1-.362-1.194c2.905-.881 6.517-.454 8.986 1.063a.624.624 0 0 1 .206.858m.084-2.268C10.154 5.56 5.9 5.419 3.438 6.166a.748.748 0 1 1-.434-1.432c2.825-.857 7.523-.692 10.492 1.07a.747.747 0 1 1-.764 1.288"/></svg>
      在 Spotify 中開啟播放清單
    </button>
  </div>
</template>

<script setup>
const props = defineProps({
  results: {
    type: Object,
    required: true
  },
  playlistName: {
    type: String,
    required: true
  }
})

const openPlaylist = () => {
  if (props.results.playlistUrl) {
    window.open(props.results.playlistUrl, '_blank')
  } else {
    console.warn('未找到播放清單 URL')
  }
}
</script>

<style scoped>
.result-section {
  margin-top: 30px;
  padding: 20px;
  background: #f8f9fa;
  border-radius: 12px;
}

.conversion-summary {
  margin-bottom: 20px;
}

.conversion-summary p {
  margin-bottom: 5px;
}

.song-list {
  max-height: 300px;
  overflow-y: auto;
  margin: 15px 0;
}

.song-item {
  padding: 10px;
  border-bottom: 1px solid #eee;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.song-status {
  font-size: 0.9rem;
  padding: 4px 8px;
  border-radius: 4px;
}

.status-success {
  background: #d4edda;
  color: #155724;
}

.status-failed {
  background: #f8d7da;
  color: #721c24;
}

.btn {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  margin-top: 15px;
}

.btn-spotify {
  background: #1db954;
  color: white;
}

.btn-spotify:hover {
  background: #1ed760;
  transform: translateY(-2px);
}
</style>

main.js

import { createApp } from 'vue'
import App from './App.vue'
import './style.css'

createApp(App).mount('#app')