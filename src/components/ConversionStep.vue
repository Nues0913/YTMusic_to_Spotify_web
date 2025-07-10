<template>
  <div class="step">
    <div class="step-title">
      <span class="step-number">3</span>
      開始轉換
    </div>
    <button 
      class="btn btn-primary" 
      @click="startConversion"
      style="display:inline-flex; align-items:center;"  
      :disabled="!canConvert || isConverting"
    >
      <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" style="margin-right: 8px" fill="none" stroke="#ffffff" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M20 11a8.1 8.1 0 0 0 -15.5 -2m-.5 -4v4h4" /><path d="M4 13a8.1 8.1 0 0 0 15.5 2m.5 4v-4h-4" /></svg>
      {{ buttonText }}
    </button>
    
    <div v-if="showProgress" class="progress-section">
      <div class="progress-bar">
        <div 
          class="progress-fill" 
          :style="{ width: progress + '%' }"
        ></div>
      </div>
      <div class="progress-text">
        {{ progressText }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const currentToken = {
  get access_token() { return localStorage.getItem('access_token') || null; },
  get refresh_token() { return localStorage.getItem('refresh_token') || null; },
  get expires_in() { return localStorage.getItem('refresh_in') || null },
  get expires() { return localStorage.getItem('expires') || null },
};

const props = defineProps({
  canConvert: Boolean,
  ytPlaylistData: {
    type: Object,
    required: true
  },
  playlistName: String
})

const emit = defineEmits(['conversion-complete'])

const isConverting = ref(false)
const showProgress = ref(false)
const progress = ref(0)
const progressText = ref('準備中...')

const buttonText = computed(() => {
  return isConverting.value ? '轉換中...' : '開始轉換播放清單'
})

const startConversion = async () => {
  isConverting.value = true
  showProgress.value = true

  progress.value = 25
  progressText.value = '正在建立 Spotify 播放清單...'
  const playlistId = await createSpotifyPlaylist(props.playlistName)
  progress.value = 50
  progressText.value = '正在搜尋 Spotify 中的對應歌曲...'
  const mappedTracks = await mapYtTracksToSpotifyTracks(props.ytPlaylistData)
  progress.value = 75
  progressText.value = '正在新增歌曲到播放清單...'
  const spotifyTracks = mappedTracks.filter(t => t.spotifyTrack).map(t => t.spotifyTrack)
  await playlistAddTracks(playlistId, spotifyTracks)
  progress.value = 100
  progressText.value = '轉換完成！'
  

  const results = {
    successCount: spotifyTracks.length,
    totalCount: props.ytPlaylistData.tracks.length,
    playlistId: playlistId,
    playlistUrl: `https://open.spotify.com/playlist/${playlistId}`,
    songs: mappedTracks.map(t => ({
      name: `${t.ytTrack.title} - ${t.ytTrack.artist || '未知藝術家'}`,
      status: t.spotifyTrack ? 'success' : 'failed'
    }))
  }
  
  emit('conversion-complete', results)
  isConverting.value = false
}

/**
 * 建立 Spotify 播放清單
 * @param playlistName 播放清單名稱
 * @return {Promise<string|null>} 返回新建立的播放清單 ID
 */
async function createSpotifyPlaylist(playlistName) {
  const accessToken = currentToken.access_token
  if (!accessToken) {
    console.error('尚未取得 Spotify 存取權杖')
    return
  }
  if (!playlistName || !playlistName.trim()) {
    console.error('缺少必要的參數: playlistName')
    return
  }

  const url = 'https://api.spotify.com/v1/me/playlists'
  const res = await fetch(url, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${accessToken}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      name: playlistName,
      description: 'Created by YouTube Music to Spotify Converter',
      public: false
    })
  })

  if (!res.ok) {
    console.error(`建立播放清單失敗: ${res.status} ${res.statusText}`)
    return null
  }

  const data = await res.json()
  return data.id
}

/**
 * 將 YouTube Music 播放清單轉換為 Spotify URI 列表
 * @param ytPlaylistData YouTube Music 播放清單資料
 * @return {Promise<Array>} 返回包含 YouTube Music track 和對應 Spotify track 的
 */
async function mapYtTracksToSpotifyTracks(ytPlaylistData) {
  if(!ytPlaylistData) {
    console.error('尚未取得ytMusic播放清單資料')
    return
  }
  // trackDatas = [{
  //   ytTrack: {...},
  //   spotifyTrack: {...} || null
  // }]
  const trackDatas = []
  for (const track of ytPlaylistData.tracks) {
    if (!track.title || !track.artist) {
      console.warn(`不明歌曲: ${JSON.stringify(track)}`)
      trackDatas.push({ ytTrack: track, spotifyTrack: null })
      continue
    }
    let searchResult = null
    searchResult = track.artist ? await searchTracks(track.title, track.artist) : await searchTracks(track.title)
    if (searchResult && searchResult.tracks.items.length > 0) {
      trackDatas.push({
        ytTrack: track,
        spotifyTrack: searchResult.tracks.items[0]
      })
    } else {
      trackDatas.push({ ytTrack: track, spotifyTrack: null })
      console.warn(`找不到 Spotify 中的對應歌曲: ${track.title} - ${track.artist}`)
    }
  }

  return trackDatas
}

/**
 * 將 Spotify 播放清單新增歌曲
 * @param playlistId 播放清單 ID
 * @param spotifyTracks 包含 Spotify track URI 的列表
 * @return {Promise<Array>} 返回成功新增的列表
 */
async function playlistAddTracks(playlistId, spotifyTracks) {
  const accessToken = currentToken.access_token
  if(!accessToken) {
    console.error('尚未取得 Spotify 存取權杖')
    return []
  }

  const url = `https://api.spotify.com/v1/playlists/${playlistId}/tracks`

  const uris = spotifyTracks.map(track => track.uri)

  const res = await fetch(url, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${accessToken}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      uris: uris
    })
  })
  if (!res.ok) {
    console.error(`Error adding track to playlist: ${res.statusText}`)
  }

  return uris
}

/**
 * 搜尋 Spotify 中的歌曲
 * @param trackName 歌曲名稱
 * @param artist 藝術家名稱(可選)
 * @return {Promise<Response|null>} 返回搜尋結果
 */
async function searchTracks(trackName, artist = null) {
  const accessToken = currentToken.access_token
  if (!accessToken) {
    console.error('尚未取得 Spotify 存取權杖')
    return
  }
  if (!trackName) {
    console.error('缺少必要的參數: trackName')
    return
  }
  let query = `track:"${trackName}"`
  if (artist) {
    if (artist.match(/\s*-\s*Topic$/i)) {
      artist = ''
    } else {
      query += ` artist:"${artist}"`
    }
  }

  const encodedQuery = encodeURIComponent(query)
  const url = `https://api.spotify.com/v1/search?q=${encodedQuery}&type=track`

  const res = await fetch(url, {
    headers: {
      'Authorization': `Bearer ${accessToken}`
    }
  })
  if (!res.ok) {
    console.error(`搜尋失敗: ${res.status} ${res.statusText}`)
    return null
  }

  return res.json()
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

.btn {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
}

.btn-primary {
  background: #667eea;
  color: white;
  width: 100%;
  margin-top: 20px;
}

.btn-primary:hover:not(:disabled) {
  background: #5a6fd8;
  transform: translateY(-2px);
}

.btn:disabled {
  background: #ccc;
  cursor: not-allowed;
  transform: none;
}

.progress-section {
  margin-top: 20px;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background: #f0f0f0;
  border-radius: 4px;
  overflow: hidden;
  margin: 20px 0;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #1db954, #1ed760);
  transition: width 0.3s ease;
}

.progress-text {
  text-align: center;
  margin-top: 10px;
  color: #444444;
}
</style>