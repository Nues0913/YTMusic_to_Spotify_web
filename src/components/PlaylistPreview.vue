<template>
  <div class="playlist-preview">
    <h4 class="preview-title">播放清單預覽</h4>
    <div class="playlist-info">
      <div class="playlist-stats">
        <strong>{{ playlistData.title }}</strong><br>
        {{ playlistData.description }}<br>
        {{ playlistData.trackCount }} 首歌曲
      </div>
    </div>
    <div class="track-list">
      <div 
        v-for="(track, index) in playlistData.tracks" 
        :key="index"
        class="track-item"
      >
        <div class="track-number">{{ index + 1 }}</div>
        <div class="track-info">
          <div class="track-title">{{ track.title }}</div>
          <div class="track-artist">{{ track.artist }}</div>
        </div>
        <div class="track-duration">{{ track.duration }}</div>
      </div>
    </div>
  </div>
</template>

<script setup>
defineProps({
  playlistData: {
    type: Object,
    required: true
  }
})
</script>

<style scoped>
.playlist-preview {
  margin: 20px 0;
  padding: 20px;
  background: #f8f9fa;
  border-radius: 8px;
  border: 1px solid #e9ecef;
}

.preview-title {
  margin-bottom: 15px;
  color: #333;
}

.playlist-info {
  margin-bottom: 15px;
}

.playlist-stats {
  background: #e3f2fd;
  padding: 10px 15px;
  border-radius: 6px;
  color: #1565c0;
  font-size: 0.9rem;
}

.track-list {
  max-height: 250px;
  overflow-y: auto;
}

.track-item {
  padding: 12px;
  border-bottom: 1px solid #dee2e6;
  display: flex;
  align-items: center;
  gap: 12px;
}

.track-item:last-child {
  border-bottom: none;
}

.track-number {
  background: #6c757d;
  color: white;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.8rem;
  font-weight: bold;
  flex-shrink: 0;
}

.track-info {
  flex: 1;
}

.track-title {
  font-weight: 500;
  color: #333;
  margin-bottom: 2px;
}

.track-artist {
  color: #6c757d;
  font-size: 0.9rem;
}

.track-duration {
  color: #6c757d;
  font-size: 0.9rem;
}
</style>