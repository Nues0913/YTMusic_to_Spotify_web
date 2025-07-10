<template>
  <div class="step">
    <div class="step-title">
      <span class="step-number">2</span>
      輸入 YouTube Music 播放清單連結
    </div>
    
    <div class="input-group">
      <label for="yt-playlist-url">YouTube Music 播放清單連結：</label>
      <div class="input-row">
        <input 
          type="url" 
          id="yt-playlist-url" 
          v-model="playlistUrl"
          placeholder="https://music.youtube.com/playlist?list=..."
          class="url-input"
        >
        <button 
          class="btn btn-fetch" 
          @click="fetchPlaylist"
          :disabled="!playlistUrl.trim() || isFetching"
        >
          {{ fetchButtonText }}
        </button>
      </div>
    </div>
    
    <PlaylistPreview 
      v-if="showPreview"
      :playlist-data="playlistData"
    />
    
    <div class="input-group">
      <label for="spotify-playlist-name">新 Spotify 播放清單名稱：</label>
      <input 
        type="text" 
        id="spotify-playlist-name" 
        :value="playlistName"
        @input="$emit('update:playlistName', $event.target.value)"
        placeholder="我的新播放清單"
        class="name-input"
      >
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import PlaylistPreview from './PlaylistPreview.vue'

const props = defineProps({
  isAuthenticated: Boolean,
  playlistName: String
})

const emit = defineEmits(['playlist-fetched', 'update:playlistName'])

const playlistUrl = ref('')
const isFetching = ref(false)
const isFetched = ref(false)
const showPreview = ref(false)
const playlistData = ref(null)

const fetchButtonText = computed(() => {
  if (isFetched.value && !showPreview.value) return '取得失敗'
  if (isFetched.value) return '已取得'
  if (isFetching.value) return '取得中...'
  return '取得曲目'
})

const fetchPlaylist = async () => {
  if (!playlistUrl.value.trim()) {
    alert('請先輸入 YouTube Music 播放清單連結')
    return
  }

  const match = playlistUrl.value.match(/[?&]list=([a-zA-Z0-9_-]+)/)
  const playlistId = match ? match[1] : null
  isFetching.value = true
  
  if (!playlistId) {
    isFetched.value = true
    showPreview.value = false
  }
  try {
    const apiKey = import.meta.env.VITE_YOUTUBE_DATA_API_V3_API_KEY
    const tracksUrl = `https://www.googleapis.com/youtube/v3/playlistItems?part=snippet&playlistId=${playlistId}&maxResults=50&key=${apiKey}`
    const titleUrl = `https://www.googleapis.com/youtube/v3/playlists?part=snippet&id=${playlistId}&key=${apiKey}`
    const titleResponse = await fetch(titleUrl)
    const tracksResponse = await fetch(tracksUrl)
    const titleData = await titleResponse.json()
    const tracksData = await tracksResponse.json()
    console.log("取得播放清單資料:", titleData)
    console.log("取得播放清單資料:", tracksData)

    playlistData.value = {
      title: titleData.items[0].snippet.title || '未知播放清單',
      description: titleData.items[0].snippet.description || '未知描述',
      trackCount: tracksData.items.length || 0,
      tracks: tracksData.items.map(item => ({
        title: item.snippet.title || '未知歌曲',
        artist: item.snippet.videoOwnerChannelTitle || '未知歌手',
      }))
  }
    showPreview.value = true
    isFetched.value = true
    isFetching.value = false
    emit('playlist-fetched', playlistData.value)
  } catch (e) {
    playlistData.value = null
    showPreview.value = false
    isFetched.value = true
    isFetching.value = false
    emit('playlist-fetched', playlistData.value)
    console.error("取得播放清單時發生錯誤:", e)
  }  
}
</script>

<style scoped>
.step {
  margin-bottom: 30px;
  padding: 20px;
  border-radius: 12px;
  border: 2px solid #f0f0f0;
  transition: all 0.3s ease;
}

.step:hover {
  border-color: #1db954;
  transform: translateY(-2px);
}

.step-number {
  display: inline-block;
  width: 30px;
  height: 30px;
  background: #2d3974;
  color: white;
  border-radius: 50%;
  text-align: center;
  line-height: 30px;
  font-weight: bold;
  margin-right: 15px;
}

.step-title {
  font-size: 1.2rem;
  font-weight: bold;
  margin-bottom: 10px;
  color: #333;
}

.input-group {
  margin: 15px 0;
}

.input-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: 500;
  color: #444444;
}

.input-row {
  display: flex;
  gap: 10px;
}

.url-input, .name-input {
  width: 100%;
  padding: 12px 16px;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  font-size: 1rem;
  transition: border-color 0.3s ease;
}

.url-input {
  flex: 1;
}

.url-input:focus, .name-input:focus {
  outline: none;
  border-color: #1db954;
}

.btn {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  white-space: nowrap;
}

.btn-fetch {
  background: #ff6b6b;
  color: white;
}

.btn-fetch:hover:not(:disabled) {
  background: #ff5252;
  transform: translateY(-2px);
}

.btn:disabled {
  background: #ccc;
  cursor: not-allowed;
  transform: none;
}
</style>