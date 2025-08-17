<template>
  <div class="playlist-container" 
    @dragover.prevent="onDragOver"
    @dragleave="onDragLeave"
    @drop="onDrop($event)"
    :class="{ 'drag-over': isDraggingOver }">
    
    <!-- Add search/filter input for playlist -->
    <div class="playlist-filter">
      <input 
        v-model="filterQuery" 
        type="text" 
        placeholder="Filter playlist..."
        class="filter-input"
      >
    </div>
    
    <!-- YouTube search toggle header -->
    <div class="search-toggle-header" @click="toggleSearchArea">
      <span>Search Music</span>
      <button class="toggle-btn" :class="{ 'open': isSearchAreaOpen }">
        <span class="toggle-icon">{{ isSearchAreaOpen ? '▼' : '▶' }}</span>
      </button>
    </div>
    
    <!-- Add YouTube search functionality -->
    <div class="youtube-search" v-if="isSearchAreaOpen">
      <div class="search-container">
        <input 
          v-model="searchQuery" 
          type="text" 
          placeholder="Search for a song..."
          @keypress.enter="searchVideos"
          class="search-input"
        >
        <button @click="searchVideos" :disabled="isSearching" class="search-button">
          Search
        </button>
      </div>
      
      <!-- YouTube search results -->
      <div class="search-results" v-if="videos.length > 0">
        <div 
          v-for="video in videos" 
          :key="video.id.videoId"
          class="search-result-item"
          draggable="true"
          @dragstart="onDragStart($event, video)"
          @click="addToPlaylist(video)"
        >
          <img :src="video.snippet.thumbnails.default.url" :alt="video.snippet.title" class="result-thumbnail">
          <div class="search-result-info">
            <div class="search-result-title">{{ video.snippet.title }}</div>
          </div>
          <button class="add-btn" @click.stop="addToPlaylist(video)" title="Add to playlist">+</button>
        </div>
      </div>
      
      <div v-if="searchError" class="search-error">{{ searchError }}</div>
      <div v-if="isSearching" class="search-loading">Searching...</div>
    </div>
    
    <div class="playlist-drop-message" v-if="playlistSongs.length === 0">
      <p>Drop songs from search results here to add to playlist</p>
    </div>
    
    <div class="playlist-controls" v-if="playlistSongs.length > 0">
      <button class="clear-btn" @click="clearPlaylist">
        Clear All
      </button>
    </div>
    
    <div class="playlist-songs" v-if="playlistSongs.length > 0">
      <div class="playlist-song" v-for="(song, index) in filteredSongs" :key="index">
        <img :src="song.snippet.thumbnails.default.url" :alt="song.snippet.title" class="song-thumbnail">
        <div class="song-info">
          <div class="song-title">{{ song.snippet.title }}</div>
        </div>
        <div class="song-actions">
          <button class="deck-btn deck1-btn" @click="loadToDeck(song, 'deck1')" title="Load to Deck 1">
            Deck 1
          </button>
          <button class="deck-btn deck2-btn" @click="loadToDeck(song, 'deck2')" title="Load to Deck 2">
            Deck 2
          </button>
          <button class="delete-btn" @click="removeSong(index)" title="Remove from playlist">
            ✕
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  name: 'Playlist',
  data() {
    return {
      playlistSongs: [],
      isDraggingOver: false,
      filterQuery: '',
      searchQuery: '',
      videos: [],
      isSearching: false,
      searchError: '',
      isSearchAreaOpen: true // Default to open
    };
  },
  computed: {
    filteredSongs() {
      if (!this.filterQuery.trim()) {
        return this.playlistSongs;
      }
      
      const query = this.filterQuery.toLowerCase().trim();
      return this.playlistSongs.filter(song => 
        song.snippet.title.toLowerCase().includes(query)
      );
    }
  },
  emits: ['load-to-deck', 'close-modal'],
  created() {
    // Load playlist from localStorage when component is created
    this.loadPlaylistFromStorage();
  },
  mounted() {
    // Add event listener to refresh playlist when modal opens
    window.addEventListener('playlist-modal-opened', this.loadPlaylistFromStorage);
  },
  beforeUnmount() {
    // Clean up event listener
    window.removeEventListener('playlist-modal-opened', this.loadPlaylistFromStorage);
  },
  methods: {
    loadPlaylistFromStorage() {
      const savedPlaylist = localStorage.getItem('dj-playlist');
      if (savedPlaylist) {
        try {
          this.playlistSongs = JSON.parse(savedPlaylist);
        } catch (error) {
          console.error('Error loading playlist from storage:', error);
          this.playlistSongs = [];
        }
      }
    },
    savePlaylistToStorage() {
      localStorage.setItem('dj-playlist', JSON.stringify(this.playlistSongs));
    },
    onDragOver(event) {
      event.preventDefault();
      this.isDraggingOver = true;
      event.dataTransfer.dropEffect = 'copy';
    },
    onDragLeave(event) {
      const rect = event.currentTarget.getBoundingClientRect();
      if (
        event.clientX <= rect.left ||
        event.clientX >= rect.right ||
        event.clientY <= rect.top ||
        event.clientY >= rect.bottom
      ) {
        this.isDraggingOver = false;
      }
    },
    onDrop(event) {
      event.preventDefault();
      this.isDraggingOver = false;
      
      try {
        const videoData = event.dataTransfer.getData('videoData');
        if (videoData) {
          const video = JSON.parse(videoData);
          this.addToPlaylist(video);
          
          // Show a quick highlight effect on successful drop
          const target = event.currentTarget;
          target.classList.add('drop-highlight');
          setTimeout(() => {
            target.classList.remove('drop-highlight');
          }, 500);
        }
      } catch (error) {
        console.error('Error processing dropped song:', error);
      }
    },
    addToPlaylist(song) {
      // Check if song is already in playlist
      const exists = this.playlistSongs.some(item => 
        item.id.videoId === song.id.videoId
      );
      
      if (!exists) {
        this.playlistSongs.push(song);
        this.savePlaylistToStorage();
        this.showAlert(`"${song.snippet.title}" added to playlist`);
      } else {
        this.showAlert(`"${song.snippet.title}" is already in playlist`);
      }
    },
    removeSong(index) {
      // Find the actual song in the original array based on the filtered index
      if (this.filterQuery.trim()) {
        const songToRemove = this.filteredSongs[index];
        const originalIndex = this.playlistSongs.findIndex(s => 
          s.id.videoId === songToRemove.id.videoId
        );
        if (originalIndex !== -1) {
          this.playlistSongs.splice(originalIndex, 1);
        }
      } else {
        this.playlistSongs.splice(index, 1);
      }
      this.savePlaylistToStorage();
    },
    clearPlaylist() {
      this.playlistSongs = [];
      this.savePlaylistToStorage();
    },
    loadToDeck(song, deckId) {
      this.$emit('load-to-deck', { song, deckId });
      // Close the modal after loading song to a deck
      this.$emit('close-modal');
    },
    closeModal() {
      this.$emit('close-modal');
    },
    // Get API key from parent component through props
    getApiKey() {
      // Access API key from parent App component
      return document.querySelector('.dj-app').__vue__?.apiKey || 'AIzaSyAapqSEk6tO4BwJ4fbXd9hFNQTcBEl4h4Q';
    },
    // Toggle search area visibility
    toggleSearchArea() {
      this.isSearchAreaOpen = !this.isSearchAreaOpen;
    },
    // YouTube search methods
    async searchVideos() {
      const query = this.searchQuery.trim();
      if (!query) {
        this.searchError = 'Please enter a search query.';
        return;
      }

      // Ensure search area is open when searching
      this.isSearchAreaOpen = true;
      this.isSearching = true;
      this.videos = [];
      this.searchError = '';

      try {
        const apiKey = this.getApiKey();
        const url = `https://www.googleapis.com/youtube/v3/search?part=snippet&type=video&maxResults=5&q=${encodeURIComponent(query)}&key=${apiKey}`;
        const response = await axios.get(url);
        
        if (response.data.items && response.data.items.length > 0) {
          this.videos = response.data.items;
        } else {
          this.searchError = 'No videos found.';
        }
      } catch (error) {
        this.searchError = `Error: ${error.message}`;
      } finally {
        this.isSearching = false;
      }
    },
    onDragStart(event, video) {
      event.dataTransfer.setData('videoId', video.id.videoId);
      event.dataTransfer.setData('videoData', JSON.stringify(video));
      
      // Set a custom drag image for better UX
      const dragImage = document.createElement('div');
      dragImage.innerHTML = `<img src="${video.snippet.thumbnails.default.url}" width="50" style="border-radius: 3px;" />`;
      dragImage.style.position = 'absolute';
      dragImage.style.top = '-1000px';
      document.body.appendChild(dragImage);
      
      // Use the custom drag image
      event.dataTransfer.setDragImage(dragImage.firstChild, 25, 25);
      
      // Set cursor to indicate dragging record
      event.dataTransfer.effectAllowed = 'copy';
      
      // Clean up the temporary element after a short delay
      setTimeout(() => {
        document.body.removeChild(dragImage);
      }, 100);
    },
    showAlert(message) {
      // Create alert element
      const alertEl = document.createElement('div');
      alertEl.className = 'playlist-alert';
      alertEl.textContent = message;
      
      // Add alert to document
      document.body.appendChild(alertEl);
      
      // Trigger animation
      setTimeout(() => {
        alertEl.classList.add('show');
      }, 10);
      
      // Remove alert after timeout
      setTimeout(() => {
        alertEl.classList.remove('show');
        setTimeout(() => {
          document.body.removeChild(alertEl);
        }, 300);
      }, 3000);
    }
  }
};
</script>

<style scoped>
.playlist-container {
  position: relative;
  min-height: 200px;
  transition: all 0.3s ease;
  border-radius: 5px;
  display: flex;
  flex-direction: column;
  max-height: 70vh;
}

/* Filter input styles */
.playlist-filter {
  margin-bottom: 15px;
}

.filter-input {
  width: 100%;
  padding: 8px 12px;
  background: #333;
  border: 1px solid #444;
  border-radius: 4px;
  color: white;
  font-size: 14px;
}

.filter-input:focus {
  outline: none;
  border-color: #0088ff;
  box-shadow: 0 0 0 2px rgba(0, 136, 255, 0.25);
}

/* Search toggle header styling */
.search-toggle-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 15px;
  background: linear-gradient(to right, #2a2a2a, #3a3a3a);
  border-radius: 5px;
  margin-bottom: 15px;
  cursor: pointer;
  user-select: none;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
  transition: all 0.2s ease;
  border-left: 3px solid #ff9900;
  font-weight: bold;
  color: #eee;
}

.search-toggle-header:hover {
  background: linear-gradient(to right, #333, #444);
}

.toggle-btn {
  background: transparent;
  border: none;
  color: #ff9900;
  cursor: pointer;
  font-size: 14px;
  padding: 5px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.2s ease;
}

.toggle-icon {
  transition: transform 0.3s ease;
}

.toggle-btn.open .toggle-icon {
  transform: rotate(0deg);
}

.toggle-btn:not(.open) .toggle-icon {
  transform: rotate(-90deg);
}

/* YouTube search styles */
.youtube-search {
  margin-bottom: 20px;
  background: rgba(0, 0, 0, 0.2);
  padding: 15px;
  border-radius: 5px;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.search-container {
  display: flex;
  gap: 10px;
  margin-bottom: 15px;
}

.search-input {
  flex: 1;
  padding: 8px 12px;
  background: #333;
  border: 1px solid #444;
  border-radius: 4px;
  color: white;
  font-size: 14px;
}

.search-input:focus {
  outline: none;
  border-color: #ff9900;
  box-shadow: 0 0 0 2px rgba(255, 153, 0, 0.25);
}

.search-button {
  padding: 8px 15px;
  background: linear-gradient(to right, #ff6600, #ff9900);
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.2s;
}

.search-button:hover {
  background: linear-gradient(to right, #ff7722, #ffaa33);
  transform: translateY(-2px);
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.3);
}

.search-button:active {
  transform: translateY(0);
}

.search-button:disabled {
  background: #555;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

/* Search results styling */
.search-results {
  max-height: 250px;
  overflow-y: auto;
  border-radius: 4px;
  background: rgba(50, 50, 50, 0.5);
  margin-bottom: 10px;
}

.search-result-item {
  display: flex;
  align-items: center;
  padding: 8px 10px;
  background-color: #333;
  margin-bottom: 5px;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
  border: 1px solid transparent;
}

.search-result-item:hover {
  background-color: #3a3a3a;
  border-color: #555;
  transform: translateY(-1px);
}

.result-thumbnail {
  width: 60px;
  height: 45px;
  object-fit: cover;
  border-radius: 3px;
}

.search-result-info {
  flex: 1;
  margin-left: 10px;
  overflow: hidden;
}

.search-result-title {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  font-size: 13px;
  color: #eee;
}

.add-btn {
  width: 24px;
  height: 24px;
  background: #0088ff;
  color: white;
  border: none;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  cursor: pointer;
  transition: all 0.2s;
}

.add-btn:hover {
  background: #00aaff;
  transform: scale(1.1);
}

.search-error, .search-loading {
  padding: 10px;
  text-align: center;
  font-size: 14px;
  color: #aaa;
}

.search-error {
  color: #ff5555;
}

.playlist-controls {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 15px;
}

.clear-btn {
  background-color: #555;
  color: white;
  border: none;
  padding: 6px 12px;
  border-radius: 3px;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.2s;
}

.clear-btn:hover {
  background-color: #777;
  transform: translateY(-1px);
}

.clear-btn:active {
  transform: translateY(1px);
}

.playlist-drop-message {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 150px;
  color: #aaa;
  border: 2px dashed #555;
  border-radius: 8px;
  background-color: rgba(0, 0, 0, 0.2);
  text-align: center;
  padding: 20px;
}

.drag-over {
  border: 2px dashed #00ccff;
  box-shadow: 0 0 15px rgba(0, 204, 255, 0.3);
}

.drop-highlight {
  background-color: rgba(0, 204, 255, 0.2);
}

.playlist-songs {
  flex: 1;
  overflow-y: auto;
  border-radius: 5px;
  padding: 5px;
  background-color: rgba(0, 0, 0, 0.2);
  max-height: 300px;
}

.playlist-song {
  display: flex;
  align-items: center;
  padding: 10px;
  margin-bottom: 8px;
  background-color: #333;
  border-radius: 6px;
  transition: all 0.2s ease;
  border: 1px solid #444;
}

.playlist-song:hover {
  background-color: #3a3a3a;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.song-thumbnail {
  width: 50px;
  height: 38px;
  object-fit: cover;
  border-radius: 4px;
}

.song-info {
  flex: 1;
  margin-left: 12px;
  overflow: hidden;
}

.song-title {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  font-size: 14px;
  color: #eee;
}

.song-actions {
  display: flex;
  gap: 6px;
}

.deck-btn {
  padding: 4px 8px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  color: white;
  transition: all 0.2s;
}

.deck-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.deck-btn:active {
  transform: translateY(1px);
}

.deck1-btn {
  background-color: #0077ff;
}

.deck1-btn:hover {
  background-color: #0088ff;
}

.deck2-btn {
  background-color: #ff0055;
}

.deck2-btn:hover {
  background-color: #ff2266;
}

.delete-btn {
  background-color: transparent;
  color: #999;
  border: none;
  cursor: pointer;
  font-size: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  transition: all 0.2s;
}

.delete-btn:hover {
  color: #ff3333;
  background-color: rgba(255, 51, 51, 0.1);
  transform: scale(1.1);
}

.delete-btn:active {
  transform: scale(0.95);
}

/* Scrollbar styling */
.playlist-songs::-webkit-scrollbar,
.search-results::-webkit-scrollbar {
  width: 8px;
}

.playlist-songs::-webkit-scrollbar-track,
.search-results::-webkit-scrollbar-track {
  background: #222;
  border-radius: 4px;
}

.playlist-songs::-webkit-scrollbar-thumb,
.search-results::-webkit-scrollbar-thumb {
  background: #555;
  border-radius: 4px;
}

.playlist-songs::-webkit-scrollbar-thumb:hover,
.search-results::-webkit-scrollbar-thumb:hover {
  background: #666;
}

@media (max-width: 600px) {
  .playlist-song {
    padding: 8px;
  }
  
  .song-thumbnail {
    width: 40px;
    height: 30px;
  }
  
  .song-title {
    font-size: 12px;
  }
  
  .deck-btn {
    padding: 3px 6px;
    font-size: 11px;
  }
}
</style> 