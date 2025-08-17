<template>
  <div class="dj-app">
    <!-- Add the splash screen - IMPORTANT: Keep this always rendered until explicitly hidden -->
    <SplashScreen 
      :loadingComplete="appLoaded" 
      v-if="!splashScreenHidden" 
    />
    
    <!-- Club Vibe video background (always rendered but conditionally displayed) -->
    <div class="club-vibe-background" :class="{ active: clubVibeActive }">
      <div id="clubVibePlayer"></div>
      <div class="club-vibe-overlay"></div>
    </div>
    
    <!-- Only show main app content when splash screen is hidden -->
    <template v-if="splashScreenHidden">
      <div class="app-header">
        <img src="./assets/images/logo.png" alt="Pro Mixer Logo" class="app-logo" />
      </div>
      
      <div class="turntable-setup">
        <div class="deck-container">
          <Deck 
            id="deck1" 
            ref="deck1"
            :apiKey="apiKey" 
            :currentCrossfade="100 - crossfadeValue" 
            :otherDeckPlaying="deck2Playing"
            @playing="handleDeck1Playing"
          />
        </div>
        
        <div class="mixer-section">
          <div class="crossfader-container">
            <label>Crossfade</label>
            <input 
              type="range" 
              min="0" 
              max="100" 
              v-model="crossfadeValue" 
              class="crossfader"
            />
            <div class="crossfade-display">
              <span>Deck 1</span>
              <span>{{ crossfadeValue }}%</span>
              <span>Deck 2</span>
            </div>
            <div class="auto-fade-controls">
              <button class="auto-fade-btn left" @click="autoFadeLeft" title="Auto fade to Deck 1">
                <span class="arrow">&#9664;</span>
              </button>
              <button class="auto-fade-btn right" @click="autoFadeRight" title="Auto fade to Deck 2">
                <span class="arrow">&#9654;</span>
              </button>
            </div>
          </div>
          
          <!-- Club Vibe Switch -->
          <div class="club-vibe-switch">
            <label class="switch">
              <input type="checkbox" v-model="clubVibeActive">
              <span class="slider round"></span>
            </label>
            <span class="club-vibe-label">Club Vibe</span>
          </div>
          
          <!-- Add playlist button -->
          <button class="playlist-btn" 
            @click="openPlaylistModal" 
            @dragover.prevent="onPlaylistBtnDragOver"
            @dragleave="onPlaylistBtnDragLeave"
            @drop="onPlaylistBtnDrop($event)"
            :class="{ 'playlist-btn-drag-over': isPlaylistBtnDragOver }"
            title="Open Playlist">
            <span class="playlist-icon">▤</span> Playlist
          </button>
        </div>
        
        <div class="deck-container">
          <Deck 
            id="deck2" 
            ref="deck2"
            :apiKey="apiKey" 
            :currentCrossfade="crossfadeValue"
            :otherDeckPlaying="deck1Playing"
            @playing="handleDeck2Playing"
          />
        </div>
      </div>
      
      <!-- Playlist Modal -->
      <div class="modal-overlay" v-if="showPlaylistModal" @click.self="showPlaylistModal = false">
        <div class="modal-container">
          <div class="modal-header">
            <h2>Playlist</h2>
            <button class="modal-close-btn" @click="showPlaylistModal = false">&times;</button>
          </div>
          <div class="modal-body">
            <Playlist @load-to-deck="handleLoadToDeck" @close-modal="showPlaylistModal = false" />
          </div>
        </div>
      </div>
    </template>
  </div>
</template>

<script>
import Deck from './components/Deck.vue';
import SplashScreen from './components/SplashScreen.vue';
import Playlist from './components/Playlist.vue';

export default {
  name: 'App',
  components: {
    Deck,
    SplashScreen,
    Playlist
  },
  data() {
    return {
      apiKey: 'AIzaSyAapqSEk6tO4BwJ4fbXd9hFNQTcBEl4h4Q',
      crossfadeValue: 50,
      deck1Playing: false,
      deck2Playing: false,
      fadeAnimationId: null,
      fadeSpeed: 0.5, // Smaller value for smoother transition
      appLoaded: false,
      splashScreenHidden: false,
      splashScreenTimer: null,
      showPlaylistModal: false,
      isPlaylistBtnDragOver: false,
      clubVibeActive: false,
      clubVibePlayer: null,
      savedDeck1Video: null, // Store deck1 video
      savedDeck2Video: null  // Store deck2 video
    };
  },
  mounted() {
    console.log("App mounted - Splash screen displayed");
    
    // Set a 5-second delay before starting to fade out splash screen
    setTimeout(() => {
      console.log("5-second timer completed - Starting splash screen fade-out");
      this.appLoaded = true;
      
      // Add a delay after fade out animation before actually hiding the splash screen
      setTimeout(() => {
        console.log("Fade animation complete - Removing splash screen from DOM");
        this.splashScreenHidden = true;
        
        // Load saved state after splash screen is hidden
        this.loadSavedState();
      }, 2000); // This should match the transition time in SplashScreen.vue
    }, 5000);
    
    // Initialize app components in the background
    this.initializeApp();
    
    // Load YouTube IFrame API
    this.loadYouTubeApi();
  },
  watch: {
    clubVibeActive(newValue) {
      if (newValue && this.clubVibePlayer) {
        // Play the video when Club Vibe is activated
        this.clubVibePlayer.playVideo();
        // Unmute the video and set volume to 50%
        this.clubVibePlayer.unMute();
        this.clubVibePlayer.setVolume(50);
        // Add a class to the body to indicate Club Vibe is active
        document.body.classList.add('club-vibe-mode');
      } else if (this.clubVibePlayer) {
        // Pause the video when Club Vibe is deactivated
        this.clubVibePlayer.pauseVideo();
        // Remove the class from body
        document.body.classList.remove('club-vibe-mode');
      }
      
      // Save club vibe state to localStorage
      localStorage.setItem('dj-club-vibe-active', newValue);
    },
    
    // Watch for crossfade value changes
    crossfadeValue(newValue) {
      // Save crossfade value to localStorage
      localStorage.setItem('dj-crossfade-value', newValue);
    }
  },
  beforeDestroy() {
    // Clean up timers
    if (this.splashScreenTimer) {
      clearTimeout(this.splashScreenTimer);
    }
    if (this.fadeAnimationId) {
      cancelAnimationFrame(this.fadeAnimationId);
    }
  },
  methods: {
    // Load YouTube IFrame API
    loadYouTubeApi() {
      // Create script element for YouTube API
      const tag = document.createElement('script');
      tag.src = "https://www.youtube.com/iframe_api";
      const firstScriptTag = document.getElementsByTagName('script')[0];
      firstScriptTag.parentNode.insertBefore(tag, firstScriptTag);
      
      // Set up callback when API is ready
      window.onYouTubeIframeAPIReady = () => this.onYouTubeIframeAPIReady();
    },
    
    // Initialize YouTube player when API is ready
    onYouTubeIframeAPIReady() {
      this.clubVibePlayer = new YT.Player('clubVibePlayer', {
        videoId: 'bWY4x_q_3v0',
        playerVars: {
          autoplay: 0,
          controls: 0,
          disablekb: 1,
          fs: 0,
          iv_load_policy: 3,
          modestbranding: 1,
          rel: 0,
          showinfo: 0,
          loop: 1,
          playlist: 'bWY4x_q_3v0', // Required for looping
          mute: 1, // Mute to comply with autoplay policies
          playsinline: 1,
          enablejsapi: 1,
          origin: window.location.origin,
          widget_referrer: window.location.href,
          autohide: 1,
          cc_load_policy: 0,
          color: 'white'
        },
        events: {
          'onReady': this.onPlayerReady,
          'onStateChange': this.onPlayerStateChange
        }
      });
    },
    
    // When player is ready
    onPlayerReady(event) {
      console.log("Club Vibe player ready");
      
      // Apply additional masking to hide YouTube branding
      this.hideYouTubeElements();
      
      // If club vibe is already active when player loads, start playing
      if (this.clubVibeActive) {
        event.target.playVideo();
      }
    },
    
    // Handle player state changes
    onPlayerStateChange(event) {
      // Apply masking each time the state changes
      this.hideYouTubeElements();
      
      // If video ended (state = 0) and Club Vibe is active, replay it
      if (event.data === 0 && this.clubVibeActive) {
        event.target.playVideo();
      }
    },
    
    // Method to hide all YouTube elements
    hideYouTubeElements() {
      // Get the iframe and add custom styles
      const iframe = document.querySelector('#clubVibePlayer iframe');
      if (iframe) {
        // Add extra CSS to the iframe to ensure no YouTube elements show
        const playerDoc = iframe.contentDocument || iframe.contentWindow.document;
        
        // If we can access the iframe content (security depends on implementation)
        if (playerDoc) {
          try {
            // Try to add a style tag to the iframe
            const styleTag = playerDoc.createElement('style');
            styleTag.textContent = `
              .ytp-chrome-top, .ytp-chrome-bottom, .ytp-watermark, 
              .ytp-pause-overlay, .ytp-youtube-button, .ytp-gradient-top, 
              .ytp-gradient-bottom, .ytp-show-cards-title, .ytp-spinner,
              .ytp-cued-thumbnail-overlay, .ytp-large-play-button,
              .html5-video-player:not(.ytp-hide-info-bar) .html5-info-bar,
              .ytp-iv-player-content, .ytp-ce-element, .ytp-player-content, 
              .ytp-endscreen-content, .ytp-share-button {
                display: none !important;
                opacity: 0 !important;
                visibility: hidden !important;
              }
              video {
                object-fit: cover !important;
              }
            `;
            playerDoc.head.appendChild(styleTag);
          } catch (e) {
            console.log("Could not apply custom styles to YouTube iframe", e);
          }
        }
      }
    },
    
    // Load saved state from localStorage
    loadSavedState() {
      // Load Club Vibe state
      const savedClubVibeState = localStorage.getItem('dj-club-vibe-active');
      if (savedClubVibeState === 'true') {
        this.clubVibeActive = true;
      }
      
      // Load saved crossfade value
      const savedCrossfade = localStorage.getItem('dj-crossfade-value');
      if (savedCrossfade !== null) {
        this.crossfadeValue = parseFloat(savedCrossfade);
      }
      
      // Load saved deck videos - with a delay to ensure components are fully mounted
      setTimeout(() => {
        try {
          // Load deck 1 video
          const savedDeck1 = localStorage.getItem('dj-deck1-video');
          if (savedDeck1) {
            this.savedDeck1Video = JSON.parse(savedDeck1);
            if (this.$refs.deck1 && this.savedDeck1Video) {
              try {
                // Try to use the parameter for not auto-playing
                this.$refs.deck1.selectVideo(this.savedDeck1Video, false);
              } catch (e) {
                // If that fails, use the regular method and then try to stop it
                this.$refs.deck1.selectVideo(this.savedDeck1Video);
              }
              
              // Store playing state separately
              localStorage.setItem('dj-deck1-playing', 'false');
            }
          }
          
          // Load deck 2 video
          const savedDeck2 = localStorage.getItem('dj-deck2-video');
          if (savedDeck2) {
            this.savedDeck2Video = JSON.parse(savedDeck2);
            if (this.$refs.deck2 && this.savedDeck2Video) {
              try {
                // Try to use the parameter for not auto-playing
                this.$refs.deck2.selectVideo(this.savedDeck2Video, false);
              } catch (e) {
                // If that fails, use the regular method and then try to stop it
                this.$refs.deck2.selectVideo(this.savedDeck2Video);
              }
              
              // Store playing state separately
              localStorage.setItem('dj-deck2-playing', 'false');
            }
          }
          
          // Add one more delay to force pause both players after initialization
          setTimeout(() => {
            // Force pause deck1 player if it exists
            if (this.$refs.deck1) {
              // Try different possible property names for the player
              const deck1Player = this.$refs.deck1.player || 
                                this.$refs.deck1.youtubePlayer || 
                                this.$refs.deck1.ytPlayer;
              
              if (deck1Player && typeof deck1Player.pauseVideo === 'function') {
                deck1Player.pauseVideo();
                console.log('Forced pause on deck1 player');
              } else {
                // Try calling a method directly on the component
                if (typeof this.$refs.deck1.pauseVideo === 'function') {
                  this.$refs.deck1.pauseVideo();
                  console.log('Called pauseVideo on deck1 component');
                } else if (typeof this.$refs.deck1.stopVideo === 'function') {
                  this.$refs.deck1.stopVideo();
                  console.log('Called stopVideo on deck1 component');
                }
              }
              // Reset playing state
              this.deck1Playing = false;
            }
            
            // Force pause deck2 player if it exists
            if (this.$refs.deck2) {
              // Try different possible property names for the player
              const deck2Player = this.$refs.deck2.player || 
                                this.$refs.deck2.youtubePlayer || 
                                this.$refs.deck2.ytPlayer;
              
              if (deck2Player && typeof deck2Player.pauseVideo === 'function') {
                deck2Player.pauseVideo();
                console.log('Forced pause on deck2 player');
              } else {
                // Try calling a method directly on the component
                if (typeof this.$refs.deck2.pauseVideo === 'function') {
                  this.$refs.deck2.pauseVideo();
                  console.log('Called pauseVideo on deck2 component');
                } else if (typeof this.$refs.deck2.stopVideo === 'function') {
                  this.$refs.deck2.stopVideo();
                  console.log('Called stopVideo on deck2 component');
                }
              }
              // Reset playing state
              this.deck2Playing = false;
            }
          }, 1000); // Increased delay to ensure videos are fully loaded
          
        } catch (error) {
          console.error('Error loading saved deck videos:', error);
        }
      }, 500); // Short delay to ensure components are fully mounted
    },
    
    // Simulate app initialization (replace with real initialization)
    initializeApp() {
      console.log("App is initializing...");
      // Here you would load resources, initialize components, etc.
      // For now, we just wait for the splash screen timer
    },
    
    handleDeck1Playing(isPlaying) {
      this.deck1Playing = isPlaying;
      
      // Save deck 1 video when it starts playing
      if (this.$refs.deck1 && this.$refs.deck1.selectedVideo) {
        localStorage.setItem('dj-deck1-video', JSON.stringify(this.$refs.deck1.selectedVideo));
        localStorage.setItem('dj-deck1-playing', isPlaying.toString());
      }
    },
    handleDeck2Playing(isPlaying) {
      this.deck2Playing = isPlaying;
      
      // Save deck 2 video when it starts playing
      if (this.$refs.deck2 && this.$refs.deck2.selectedVideo) {
        localStorage.setItem('dj-deck2-video', JSON.stringify(this.$refs.deck2.selectedVideo));
        localStorage.setItem('dj-deck2-playing', isPlaying.toString());
      }
    },
    autoFadeLeft() {
      // Cancel any existing fade animation
      if (this.fadeAnimationId) {
        cancelAnimationFrame(this.fadeAnimationId);
      }
      
      // If Deck 1 is not playing but has a song loaded, start playing it
      if (!this.deck1Playing && this.$refs.deck1 && this.$refs.deck1.selectedVideo) {
        this.$refs.deck1.playVideo();
      }
      
      // Start automatic fade to Deck 1 (0%)
      this.startFadeAnimation(0);
    },
    autoFadeRight() {
      // Cancel any existing fade animation
      if (this.fadeAnimationId) {
        cancelAnimationFrame(this.fadeAnimationId);
      }
      
      // If Deck 2 is not playing but has a song loaded, start playing it
      if (!this.deck2Playing && this.$refs.deck2 && this.$refs.deck2.selectedVideo) {
        this.$refs.deck2.playVideo();
      }
      
      // Start automatic fade to Deck 2 (100%)
      this.startFadeAnimation(100);
    },
    startFadeAnimation(targetValue) {
      // Store the starting value to calculate progress
      const startValue = Number(this.crossfadeValue);
      const targetVal = Number(targetValue);
      const totalDistance = Math.abs(targetVal - startValue);
      const startTime = performance.now();
      const duration = 8000; // 8 seconds for a full fade (slower than before)
      
      const animate = (currentTime) => {
        // Calculate time progress
        const elapsed = currentTime - startTime;
        const progress = Math.min(elapsed / duration, 1); // Cap at 1
        
        if (progress >= 1) {
          // Animation complete, snap to the exact target value
          this.crossfadeValue = targetVal;
          this.fadeAnimationId = null;
          return;
        }
        
        // Use easeInOutQuad for smoother animation
        let easedProgress;
        if (progress < 0.5) {
          easedProgress = 2 * progress * progress;
        } else {
          easedProgress = 1 - Math.pow(-2 * progress + 2, 2) / 2;
        }
        
        // Calculate the new value based on the eased progress
        const newValue = startValue + (targetVal - startValue) * easedProgress;
        
        // Ensure newValue is a number before using toFixed
        this.crossfadeValue = parseFloat(newValue.toString()).toFixed(1);
        
        // Continue the animation
        this.fadeAnimationId = requestAnimationFrame(animate);
      };
      
      this.fadeAnimationId = requestAnimationFrame(animate);
    },
    handleLoadToDeck({ song, deckId }) {
      if (deckId === 'deck1' && this.$refs.deck1) {
        this.$refs.deck1.selectVideo(song);
        // Save to localStorage when loading a video to deck 1
        localStorage.setItem('dj-deck1-video', JSON.stringify(song));
      } else if (deckId === 'deck2' && this.$refs.deck2) {
        this.$refs.deck2.selectVideo(song);
        // Save to localStorage when loading a video to deck 2
        localStorage.setItem('dj-deck2-video', JSON.stringify(song));
      }
    },
    
    // New methods for playlist button drag and drop
    onPlaylistBtnDragOver(event) {
      event.preventDefault();
      this.isPlaylistBtnDragOver = true;
      event.dataTransfer.dropEffect = 'copy';
    },
    
    onPlaylistBtnDragLeave(event) {
      this.isPlaylistBtnDragOver = false;
    },
    
    onPlaylistBtnDrop(event) {
      event.preventDefault();
      this.isPlaylistBtnDragOver = false;
      
      try {
        const videoData = event.dataTransfer.getData('videoData');
        if (videoData) {
          const video = JSON.parse(videoData);
          this.addSongToPlaylist(video);
        }
      } catch (error) {
        console.error('Error processing dropped song:', error);
      }
    },
    
    addSongToPlaylist(song) {
      // Get current playlist from localStorage
      let playlist = [];
      const savedPlaylist = localStorage.getItem('dj-playlist');
      if (savedPlaylist) {
        try {
          playlist = JSON.parse(savedPlaylist);
        } catch (error) {
          console.error('Error loading playlist from storage:', error);
          playlist = [];
        }
      }
      
      // Check if song is already in playlist
      const exists = playlist.some(item => 
        item.id.videoId === song.id.videoId
      );
      
      if (!exists) {
        // Add song to playlist
        playlist.push(song);
        localStorage.setItem('dj-playlist', JSON.stringify(playlist));
        
        // Show alert
        this.showAlert(`"${song.snippet.title}" added to playlist`);
      } else {
        this.showAlert(`"${song.snippet.title}" is already in playlist`);
      }
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
    },
    
    openPlaylistModal() {
      this.showPlaylistModal = true;
      // Dispatch event to notify the playlist component to refresh
      window.dispatchEvent(new Event('playlist-modal-opened'));
    }
  }
};
</script>

<style>
body {
  font-family: Arial, sans-serif;
  margin: 0;
  padding: 0;
  background-color: #121212;
  background-image: url('./assets/images/background.jpg');
  background-size: cover;
  background-position: center;
  background-attachment: fixed;
  background-repeat: no-repeat;
  color: white;
  overflow-x: hidden;
}

.dj-app {
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
  padding: 10px;
  box-sizing: border-box;
}

.app-header {
  text-align: center;
  margin-bottom: 20px;
}

.app-logo {
  width: 180px;
  height: auto;
  filter: drop-shadow(0 0 10px rgba(255, 0, 0, 0.5));
}

h1 {
  text-align: center;
  color: #ff0000;
  margin-bottom: 20px;
  font-size: 2rem;
  text-shadow: 0 0 10px rgba(255, 0, 0, 0.5);
}

.turntable-setup {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  flex-wrap: wrap;
  gap: 15px;
}

.deck-container {
  flex: 1;
  min-width: 280px;
  background: #2a2a2a;
  border-radius: 15px;
  padding: 15px;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.5);
}

.mixer-section {
  width: 120px;
  background: #333;
  padding: 15px;
  border-radius: 10px;
  display: flex;
  flex-direction: column;
  align-items: center;
  box-shadow: 0 0 15px rgba(0, 0, 0, 0.7);
}

.crossfader-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}

.crossfader-container label {
  margin-bottom: 10px;
  font-weight: bold;
  color: white;
}

.crossfader {
  width: 100%;
  height: 20px;
  -webkit-appearance: none;
  background: linear-gradient(to right, #ff0000, #0066ff);
  outline: none;
  border-radius: 10px;
  margin-bottom: 10px;
}

.crossfader::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 25px;
  height: 25px;
  background: #ffffff;
  border-radius: 50%;
  cursor: pointer;
}

.crossfade-display {
  display: flex;
  justify-content: space-between;
  width: 100%;
  font-size: 0.8rem;
}

.auto-fade-controls {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-top: 12px;
  width: 100%;
}

.auto-fade-btn {
  background: #333;
  border: 2px solid;
  border-radius: 50%;
  width: 36px;
  height: 36px;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  transition: all 0.3s ease;
  position: relative;
  overflow: hidden;
}

.auto-fade-btn:hover {
  transform: scale(1.1);
}

.auto-fade-btn:active {
  transform: scale(0.95);
}

.auto-fade-btn.left {
  border-color: #ff0000;
  box-shadow: 0 0 5px #ff0000;
}

.auto-fade-btn.right {
  border-color: #0066ff;
  box-shadow: 0 0 5px #0066ff;
}

.auto-fade-btn.left:hover {
  box-shadow: 0 0 10px #ff0000, 0 0 15px rgba(255, 0, 0, 0.5);
}

.auto-fade-btn.right:hover {
  box-shadow: 0 0 10px #0066ff, 0 0 15px rgba(0, 102, 255, 0.5);
}

.auto-fade-btn .arrow {
  color: white;
  font-size: 14px;
  line-height: 1;
  text-shadow: 0 0 5px rgba(255, 255, 255, 0.7);
}

.auto-fade-btn.left .arrow {
  color: #ff0000;
}

.auto-fade-btn.right .arrow {
  color: #0066ff;
}

/* Media queries for responsiveness */
@media (max-width: 900px) {
  .turntable-setup {
    flex-direction: column;
    align-items: center;
  }
  
  .deck-container {
    width: 100%;
    max-width: 500px;
  }
  
  .mixer-section {
    width: 80%;
    max-width: 500px;
    margin: 10px 0;
    padding: 10px;
  }
  
  .crossfader {
    height: 30px;
  }
}

@media (max-width: 480px) {
  .dj-app {
    padding: 5px;
  }
  
  h1 {
    font-size: 1.5rem;
    margin-bottom: 10px;
  }
  
  .deck-container {
    padding: 10px;
  }
}

/* Add styles for playlist section */
.playlist-wrapper {
  width: 100%;
  max-width: 1200px;
  margin: 20px auto;
  padding: 0 15px;
  box-sizing: border-box;
}

@media (max-width: 768px) {
  .playlist-wrapper {
    padding: 0 10px;
  }
  
  .turntable-setup {
    flex-direction: column;
    align-items: center;
  }
  
  .deck-container {
    margin-bottom: 20px;
  }
  
  .mixer-section {
    width: 100%;
    margin: 10px 0;
  }
}

/* Modal Styles */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
  backdrop-filter: blur(3px);
}

.modal-container {
  background-color: #2a2a2a;
  width: 90%;
  max-width: 800px;
  max-height: 80vh;
  border-radius: 10px;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.5);
  display: flex;
  flex-direction: column;
  animation: modal-appear 0.3s ease-out;
}

@keyframes modal-appear {
  from {
    opacity: 0;
    transform: translateY(-50px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  border-bottom: 1px solid #444;
}

.modal-header h2 {
  margin: 0;
  color: #fff;
  font-size: 1.5rem;
}

.modal-close-btn {
  background: none;
  border: none;
  color: #aaa;
  font-size: 24px;
  cursor: pointer;
  transition: color 0.2s;
}

.modal-close-btn:hover {
  color: #fff;
}

.modal-body {
  padding: 20px;
  overflow-y: auto;
}

/* Playlist button in mixer section */
.playlist-btn {
  width: 100%;
  margin-top: 20px;
  padding: 8px 0;
  background: linear-gradient(to right, #ff6600, #ff9900);
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 5px;
  transition: all 0.2s;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.3);
}

.playlist-btn:hover {
  background: linear-gradient(to right, #ff7722, #ffaa33);
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.4);
}

.playlist-btn:active {
  transform: translateY(1px);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
}

.playlist-icon {
  font-size: 16px;
}

/* Remove the old playlist wrapper styles */
.playlist-wrapper {
  display: none;
}

@media (max-width: 768px) {
  .modal-container {
    width: 95%;
    max-height: 90vh;
  }
  
  .playlist-btn {
    padding: 10px 0;
    font-size: 14px;
  }
}

/* Playlist button hover/active states */
.playlist-btn-drag-over {
  background: linear-gradient(to right, #ff8800, #ffcc00);
  transform: scale(1.05);
  box-shadow: 0 0 15px rgba(255, 136, 0, 0.7);
}

/* Alert styling */
.playlist-alert {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%) translateY(100px);
  padding: 12px 20px;
  background-color: rgba(0, 0, 0, 0.8);
  color: white;
  border-radius: 8px;
  font-size: 14px;
  max-width: 80%;
  text-align: center;
  z-index: 1000;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.5);
  opacity: 0;
  transition: transform 0.3s ease, opacity 0.3s ease;
  border-left: 4px solid #ff9900;
}

.playlist-alert.show {
  transform: translateX(-50%) translateY(0);
  opacity: 1;
}

/* Club Vibe Styles */
.club-vibe-background {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: -1;
  opacity: 0;
  transition: opacity 0.5s ease;
  pointer-events: none;
  overflow: hidden;
}

.club-vibe-background.active {
  opacity: 1;
}

.club-vibe-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 10;
  pointer-events: none;
  background: transparent;
}

#clubVibePlayer {
  position: absolute;
  top: 50%;
  left: 50%;
  min-width: 100%;
  min-height: 100%;
  width: auto;
  height: auto;
  transform: translate(-50%, -50%);
  object-fit: cover;
}

/* Hide all YouTube UI elements */
.club-vibe-background iframe {
  border: none !important;
}

.club-vibe-background .ytp-chrome-top,
.club-vibe-background .ytp-chrome-bottom,
.club-vibe-background .ytp-watermark,
.club-vibe-background .ytp-pause-overlay,
.club-vibe-background .ytp-youtube-button,
.club-vibe-background .ytp-gradient-top,
.club-vibe-background .ytp-gradient-bottom,
.club-vibe-background .ytp-show-cards-title,
.club-vibe-background .ytp-spinner,
.club-vibe-background .ytp-cued-thumbnail-overlay,
.club-vibe-background .ytp-large-play-button,
.club-vibe-background .html5-video-player:not(.ytp-hide-info-bar) .html5-info-bar {
  display: none !important;
  opacity: 0 !important;
  visibility: hidden !important;
  pointer-events: none !important;
}

.club-vibe-switch {
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 15px 0;
  width: 100%;
}

.club-vibe-label {
  margin-left: 10px;
  font-weight: bold;
  color: #ff9900;
  text-shadow: 0 0 5px rgba(255, 153, 0, 0.7);
}

/* Switch styling */
.switch {
  position: relative;
  display: inline-block;
  width: 60px;
  height: 34px;
}

.switch input {
  opacity: 0;
  width: 0;
  height: 0;
}

.slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: #333;
  transition: .4s;
  box-shadow: inset 0 0 5px rgba(0, 0, 0, 0.5);
}

.slider:before {
  position: absolute;
  content: "";
  height: 26px;
  width: 26px;
  left: 4px;
  bottom: 4px;
  background-color: white;
  transition: .4s;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.5);
}

input:checked + .slider {
  background-color: #ff9900;
  box-shadow: 0 0 10px #ff9900;
}

input:checked + .slider:before {
  transform: translateX(26px);
}

.slider.round {
  border-radius: 34px;
}

.slider.round:before {
  border-radius: 50%;
}

/* Club Vibe Mode - Body class for when Club Vibe is active */
body.club-vibe-mode {
  background-image: none !important;
}

/* Make deck containers transparent when in Club Vibe mode */
body.club-vibe-mode .deck-container {
  background: rgba(42, 42, 42, 0.7);
  backdrop-filter: blur(3px);
  box-shadow: 0 0 25px rgba(0, 0, 0, 0.3);
  transition: background 0.5s ease, box-shadow 0.5s ease;
}
</style> 