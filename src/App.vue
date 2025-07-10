<template>
  <div class="container">
    <AppHeader />
    <div class="main-card">
      <SpotifyAuth @authenticated="handleSpotifyAuth" @logout="handleSpotifyLogout" />
      <PlaylistInput 
      :is-authenticated="isSpotifyAuthenticated"
      @playlist-fetched="handlePlaylistFetched"
      v-model:playlist-name="playlistName"
      />
      <ConversionStep 
        :can-convert="canStartConversion"
        :yt-playlist-data="ytPlaylistData"
        :playlist-name="playlistName"
        @conversion-complete="handleConversionComplete"
      />
      <ConversionResults 
        v-if="showResults"
        :results="conversionResults"
        :playlist-name="playlistName"
        @open-playlist="openSpotifyPlaylist"
      />
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import AppHeader from './components/AppHeader.vue'
import SpotifyAuth from './components/SpotifyAuth.vue'
import PlaylistInput from './components/PlaylistInput.vue'
import ConversionStep from './components/ConversionStep.vue'
import ConversionResults from './components/ConversionResults.vue'

const isSpotifyAuthenticated = ref(false)
const ytPlaylistFetched = ref(false)
const playlistName = ref('')
const showResults = ref(false)
const conversionResults = ref(null)
const currentSpPlaylistId = ref(null)
const ytPlaylistData = ref(null)

const canStartConversion = computed(() => {
  return isSpotifyAuthenticated.value && 
         ytPlaylistFetched.value && 
         !!playlistName.value.trim()
})

const handleSpotifyAuth = () => {
  isSpotifyAuthenticated.value = true
}

const handleSpotifyLogout = () => {
  isSpotifyAuthenticated.value = false
  ytPlaylistFetched.value = false
  playlistName.value = ''
  showResults.value = false
  conversionResults.value = null
  currentSpPlaylistId.value = null
  ytPlaylistData.value = null
}

const handlePlaylistFetched = (data) => {
  if(data) {
    ytPlaylistFetched.value = true
    ytPlaylistData.value = data
    if (!playlistName.value.trim()) {
      playlistName.value = data.title
    }
  } else {
    ytPlaylistFetched.value = false
    ytPlaylistData.value = null
    playlistName.value = ''
  }
}

const handleConversionComplete = (results) => {
  if(results){
    conversionResults.value = results
    showResults.value = true
    currentSpPlaylistId.value = results.playlistId
  } else {
    showResults.value = false
    conversionResults.value = null
    currentSpPlaylistId.value = null
  }
}

const openSpotifyPlaylist = () => {
  if (conversionResults.value && conversionResults.value.playlistUrl) {
    window.open(conversionResults.value.playlistUrl, '_blank')
  }
}
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.main-card {
  background: rgba(255, 255, 255, 0.8);
  border-radius: 20px;
  padding: 40px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1);
  margin-bottom: 30px;
}

@media (max-width: 768px) {
  .container {
    padding: 10px;
  }
  
  .main-card {
    padding: 20px;
  }
}
</style>