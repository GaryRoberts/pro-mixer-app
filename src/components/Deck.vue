<template>
  <div class="deck" :class="{ 
    'deck-active': isPlaying, 
    'neon-blue': isPlaying && id === 'deck1',
    'neon-red': isPlaying && id === 'deck2'
  }" :id="id"
    @dragover.prevent="onDragOver"
    @dragleave="onDragLeave"
    @drop="onDrop($event)"
  >
    <div class="deck-title">{{ id === 'deck1' ? 'Deck 1' : 'Deck 2' }}</div>
    
    <div class="deck-content">
      <!-- Turntable section -->
      <div class="turntable-section">
        <div 
          class="record-container" 
          v-if="selectedVideo"
          @dragover.prevent="onDragOver"
          @dragleave="onDragLeave"
          @drop="onDrop($event)"
        >
          <div class="record" :class="{ 
              'record-spinning': isPlaying && !isPullingUp, 
              'record-reverse-spin': isPullingUp 
            }" ref="recordElement">
            <div class="record-label">
              <img 
                :src="selectedVideo.snippet.thumbnails.medium.url" 
                :alt="selectedVideo.snippet.title"
                class="record-artwork"
              >
              <div class="record-center"></div>
            </div>
          </div>
          
          <!-- Drop area overlay for the record -->
          <div 
            class="drop-area-record" 
            v-if="isDraggingOver"
          >
            <div class="drop-message">Drop to load</div>
          </div>
        </div>
        <div 
          v-else 
          class="empty-turntable"
          @dragover.prevent="onDragOver"
          @dragleave="onDragLeave"
          @drop="onDrop($event)"
        >
          <div class="empty-record">
            <div class="record-center"></div>
          </div>
          
          <!-- Drop area overlay for empty turntable -->
          <div 
            class="drop-area-record" 
            v-if="isDraggingOver"
          >
            <div class="drop-message">Drop to load</div>
          </div>
        </div>
        
        <div class="video-title" v-if="selectedVideo">{{ selectedVideo.snippet.title }}</div>
        <div class="empty-title" v-else>No track loaded</div>
        
        <!-- Timeline slider and time display -->
        <div class="timeline-container" v-if="selectedVideo">
          <div class="time-display">
            <span>{{ formatTime(currentTime) }}</span>
            <span>{{ formatTime(duration - currentTime) }}</span>
          </div>
          <input 
            type="range" 
            min="0" 
            :max="duration" 
            v-model="sliderValue" 
            class="timeline-slider"
            @mousedown="onSliderDragStart"
            @mouseup="onSliderDragEnd"
            :disabled="!playerReady || !selectedVideo"
          >
        </div>
        
        <div class="player-status" v-if="!playerReady">Loading player...</div>
        
        <div class="deck-controls">
          <button @click="playVideo" :disabled="isPlaying || !selectedVideo || !playerReady" class="play-btn">
            ▶ Play
          </button>
          <button @click="pauseVideo" :disabled="!isPlaying || !playerReady" class="pause-btn">
            ❙❙ Pause
          </button>
          <button @click="stopVideo" :disabled="!selectedVideo || !playerReady" class="stop-btn">
            ■ Stop
          </button>
          <button @click="ejectVideo" :disabled="!selectedVideo" class="eject-btn">
            ⏏ Eject
          </button>
          <button @click="pullUpVideo" :disabled="!isPlaying || !playerReady" class="pullup-btn">
            ↺ Pull Up
          </button>
        </div>
        
        <div class="volume-controls">
          <label>Volume: {{ Math.round(actualVolume * 100) }}%</label>
          <div class="volume-slider-container">
            <input 
              type="range" 
              min="0" 
              max="100" 
              v-model="deckVolume" 
              class="volume-slider"
            >
          </div>
        </div>
        
        <!-- Tempo and Pitch Controls -->
        <div class="controls-container">
          <div class="tempo-pitch-controls">
            <div class="tempo-control">
              <label>Tempo: {{ tempoValue }}%</label>
              <div class="control-slider-container">
                <input 
                  type="range" 
                  min="50" 
                  max="150" 
                  v-model="tempoValue" 
                  class="tempo-slider"
                  :disabled="!isPlaying"
                  @input="updatePlaybackRate"
                >
              </div>
              <button class="reset-btn" @click="resetTempo" :disabled="!isPlaying || tempoValue === 100">Reset</button>
            </div>
            
            <div class="pitch-control">
              <label>Pitch: {{ pitchValue > 0 ? '+' : '' }}{{ pitchValue }}%</label>
              <div class="control-slider-container">
                <input 
                  type="range" 
                  min="-50" 
                  max="50" 
                  v-model="pitchValue" 
                  class="pitch-slider"
                  :disabled="!isPlaying"
                  @input="updatePlaybackRate"
                >
                <div class="pitch-markers">
                  <span class="marker low">-50%</span>
                  <span class="marker">-25%</span>
                  <span class="marker center">0%</span>
                  <span class="marker">+25%</span>
                  <span class="marker high">+50%</span>
                </div>
                <div class="pitch-indicator" :style="{ left: `${((Number(pitchValue) + 50) / 100) * 100}%` }"></div>
              </div>
              <button class="reset-btn" @click="resetPitch" :disabled="!isPlaying || pitchValue === 0">Reset</button>
            </div>
          </div>
          
          <!-- Sound Effects Buttons -->
          <div class="sound-effects">
            <div class="sound-effects-title">Sound FX</div>
            <div class="sound-effects-buttons">
              <button @click="playSound('airhorn')" class="sound-effect-btn airhorn-btn">
                Airhorn
              </button>
              <button @click="playSound('scratch')" class="sound-effect-btn scratch-btn">
                Scratch
              </button>
              <button @click="playSound('bass')" class="sound-effect-btn bass-btn">
                Bass
              </button>
              <button @click="playSound('siren')" class="sound-effect-btn siren-btn">
                Siren
              </button>
            </div>
          </div>
        </div>
      </div>
      
      <!-- Search section - always visible -->
      <div class="search-section">
        <div class="search-container">
          <input 
            v-model="searchQuery" 
            type="text" 
            :placeholder="`Search tracks (Deck ${id === 'deck1' ? '1' : '2'})`"
            @keypress.enter="searchVideos"
          >
          <button @click="searchVideos" :disabled="isSearching">Search</button>
        </div>
        
        <div class="search-results" v-if="videos.length > 0">
          <div 
            v-for="video in videos" 
            :key="video.id.videoId"
            class="search-result-item"
            @click="selectVideo(video)"
            draggable="true"
            @dragstart="onDragStart($event, video)"
          >
            <img :src="video.snippet.thumbnails.default.url" :alt="video.snippet.title">
            <div class="search-result-info">
              <div class="search-result-title">{{ video.snippet.title }}</div>
            </div>
          </div>
        </div>
        
        <div v-if="searchError" class="search-error">{{ searchError }}</div>
        <div v-if="isSearching" class="search-loading">Searching...</div>
      </div>
    </div>
    
    <!-- YouTube iframe (hidden) -->
    <div class="youtube-player-container">
      <div :id="`youtube-player-${id}`"></div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  name: 'Deck',
  props: {
    id: {
      type: String,
      required: true
    },
    apiKey: {
      type: String,
      required: true
    },
    currentCrossfade: {
      type: Number,
      default: 50
    },
    otherDeckPlaying: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      searchQuery: '',
      videos: [],
      selectedVideo: null,
      isSearching: false,
      searchError: '',
      isPlaying: false,
      isPaused: false,
      player: null,
      playerReady: false,
      deckVolume: 100,
      // Tempo and pitch controls
      tempoValue: 100, // 100% = normal speed
      pitchValue: 0,   // 0 = normal pitch
      // Timeline-related variables
      duration: 0,
      currentTime: 0,
      sliderValue: 0,
      timeUpdateInterval: null,
      isDraggingSlider: false,
      // YouTube API variables
      ytApiLoaded: false,
      playerInitialized: false,
      isDraggingOver: false,
      // Sound effect variables
      soundEffects: {
        airhorn: null,
        scratch: null,
        bass: null,
        siren: null,
        vinyl_stop: null
      },
      userInteracted: false, // Track if user has interacted with the page
      audioContext: null,    // Web Audio API context
      audioBuffers: {},       // Store decoded audio buffers
      isPullingUp: false,
    };
  },
  computed: {
    actualVolume() {
      // Calculate the actual volume based on deck volume and crossfade
      // currentCrossfade is already properly inverted in the parent component
      // 0 = fully faded out, 100 = full volume
      const crossfadeEffect = this.currentCrossfade / 100;
      return (this.deckVolume / 100) * crossfadeEffect;
    }
  },
  watch: {
    actualVolume(newVolume) {
      if (this.player && this.playerReady) {
        console.log(`Setting ${this.id} volume to ${Math.round(newVolume * 100)}%`);
        this.player.setVolume(newVolume * 100);
      }
    },
    currentCrossfade(newValue) {
      if (this.player && this.playerReady) {
        // Make sure volume is updated when crossfade changes
        // Use the computed actualVolume to ensure consistency
        console.log(`${this.id} crossfade changed to ${newValue}%`);
        this.player.setVolume(this.actualVolume * 100);
      }
    },
    sliderValue(newValue) {
      // Only seek when user is dragging the slider, otherwise it will jump around during playback
      if (this.isDraggingSlider && this.player && this.playerReady) {
        this.player.seekTo(Number(newValue), true);
      }
    }
  },
  mounted() {
    this.loadYouTubeApi();
    this.loadSoundEffects();
    
    // Add user interaction listener
    document.addEventListener('click', this.handleUserInteraction, { once: true });
    document.addEventListener('keydown', this.handleUserInteraction, { once: true });
  },
  beforeUnmount() {
    // Clean up interval and listeners when component is destroyed
    this.clearTimeUpdateInterval();
    document.removeEventListener('click', this.handleUserInteraction);
    document.removeEventListener('keydown', this.handleUserInteraction);
    
    // Clean up audio context if created
    if (this.audioContext) {
      this.audioContext.close().catch(() => {});
    }
  },
  methods: {
    loadYouTubeApi() {
      // Check if YouTube API is already loading or loaded
      if (window.YT && window.YT.Player) {
        this.ytApiLoaded = true;
        this.initYouTubePlayer();
        return;
      }
      
      // Check if the API script is already being loaded
      if (!document.getElementById('youtube-api')) {
        // Create script element
        const tag = document.createElement('script');
        tag.src = 'https://www.youtube.com/iframe_api';
        tag.id = 'youtube-api';
        const firstScriptTag = document.getElementsByTagName('script')[0];
        firstScriptTag.parentNode.insertBefore(tag, firstScriptTag);
      }
      
      // Setup global callback for YouTube API ready event
      if (!window.onYouTubeIframeAPIReady) {
        window.onYouTubeIframeAPIReady = () => {
          this.ytApiLoaded = true;
          
          // Call initialization for all deck instances
          document.dispatchEvent(new Event('youtube-api-ready'));
          
          // Initialize this player
          this.initYouTubePlayer();
        };
      }
      
      // Listen for the YouTube API ready event
      document.addEventListener('youtube-api-ready', () => {
        if (!this.playerInitialized) {
          this.initYouTubePlayer();
        }
      });
    },
    initYouTubePlayer() {
      if (this.playerInitialized) return;
      
      // Create YouTube player
      this.player = new window.YT.Player(`youtube-player-${this.id}`, {
        height: '0',
        width: '0',
        playerVars: {
          autoplay: 0,
          controls: 0,
          disablekb: 1,
          fs: 0,
          iv_load_policy: 3,
          modestbranding: 1,
          rel: 0
        },
        events: {
          'onReady': this.onPlayerReady,
          'onStateChange': this.onPlayerStateChange
        }
      });
      
      this.playerInitialized = true;
    },
    onPlayerReady(event) {
      this.playerReady = true;
      this.player.setVolume(this.actualVolume * 100);
      
      // If a video was selected before player was ready, load it now
      if (this.selectedVideo) {
        this.player.cueVideoById(this.selectedVideo.id.videoId);
      }
    },
    onPlayerStateChange(event) {
      // When video is playing (1) or buffering (3), start tracking time
      if (event.data === 1 || event.data === 3) {
        this.startTimeUpdateInterval();
        
        // Update duration only when we get a valid value
        const newDuration = this.player.getDuration();
        if (newDuration > 0) {
          this.duration = newDuration;
        }
      }
      
      // When video is paused (2) or ended (0), stop tracking time
      if (event.data === 2 || event.data === 0) {
        this.clearTimeUpdateInterval();
      }
      
      // When video ends
      if (event.data === 0) {
        this.isPlaying = false;
        this.$emit('playing', false);
      }
    },
    startTimeUpdateInterval() {
      // Clear any existing interval first
      this.clearTimeUpdateInterval();
      
      // Update time every 500ms
      this.timeUpdateInterval = setInterval(() => {
        if (this.player && this.playerReady && !this.isDraggingSlider) {
          this.currentTime = this.player.getCurrentTime() || 0;
          this.sliderValue = this.currentTime;
        }
      }, 500);
    },
    clearTimeUpdateInterval() {
      if (this.timeUpdateInterval) {
        clearInterval(this.timeUpdateInterval);
        this.timeUpdateInterval = null;
      }
    },
    formatTime(seconds) {
      if (!seconds || isNaN(seconds)) return '0:00';
      
      seconds = Math.floor(seconds);
      const minutes = Math.floor(seconds / 60);
      const remainingSeconds = seconds % 60;
      return `${minutes}:${remainingSeconds < 10 ? '0' : ''}${remainingSeconds}`;
    },
    onSliderDragStart() {
      this.isDraggingSlider = true;
    },
    onSliderDragEnd() {
      this.isDraggingSlider = false;
      if (this.player && this.playerReady) {
        this.player.seekTo(Number(this.sliderValue), true);
      }
    },
    async searchVideos() {
      const query = this.searchQuery.trim();
      if (!query) {
        this.searchError = 'Please enter a search query.';
        return;
      }

      this.isSearching = true;
      this.videos = [];
      this.searchError = '';

      try {
        const url = `https://www.googleapis.com/youtube/v3/search?part=snippet&type=video&maxResults=5&q=${encodeURIComponent(query)}&key=${this.apiKey}`;
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
    selectVideo(video) {
      const wasPlaying = this.isPlaying;
      this.selectedVideo = video;
      
      // Reset timeline related values
      this.duration = 0;
      this.currentTime = 0;
      this.sliderValue = 0;
      
      // Reset tempo and pitch to default values
      this.resetTempo();
      this.resetPitch();
      
      if (this.player && this.playerReady) {
        if (wasPlaying) {
          // If a song was already playing, load and play the new one immediately
          this.player.loadVideoById(video.id.videoId);
          this.isPlaying = true;
          this.$emit('playing', true);
          console.log(`Loading and playing video ID: ${video.id.videoId}`);
        } else {
          // Otherwise just cue the video without playing
          this.player.cueVideoById(video.id.videoId);
          console.log(`Loading video ID: ${video.id.videoId}`);
        }
      } else {
        console.log('Player not ready yet, video will be loaded when player is ready');
      }
    },
    playVideo() {
      if (this.player && this.playerReady && this.selectedVideo) {
        console.log(`Playing video on ${this.id}`);
        this.player.playVideo();
        this.isPlaying = true;
        this.isPaused = false;
        this.$emit('playing', true);
        
        // Apply current tempo and pitch when playing
        this.updatePlaybackRate();
      } else {
        console.log('Cannot play: player ready =', this.playerReady, 'selected video =', !!this.selectedVideo);
      }
    },
    pauseVideo() {
      if (this.player && this.playerReady) {
        this.player.pauseVideo();
        this.isPlaying = false;
        this.isPaused = true;
        this.$emit('playing', false);
      }
    },
    stopVideo() {
      if (this.player && this.playerReady) {
        this.player.stopVideo();
        this.isPlaying = false;
        this.isPaused = false;
        this.$emit('playing', false);
        
        // Reset timeline
        this.currentTime = 0;
        this.sliderValue = 0;
        
        // Reset tempo and pitch
        this.resetTempo();
        this.resetPitch();
      }
    },
    ejectVideo() {
      this.stopVideo();
      this.selectedVideo = null;
      this.videos = [];
      this.searchQuery = '';
      
      // Reset timeline related values
      this.duration = 0;
      this.currentTime = 0;
      this.sliderValue = 0;
      this.clearTimeUpdateInterval();
      
      // Reset tempo and pitch (redundant, but ensures it's reset)
      this.resetTempo();
      this.resetPitch();

      // Remove video from localStorage
      localStorage.removeItem(`dj-${this.id}-video`);
      localStorage.removeItem(`dj-${this.id}-playing`);
    },
    pullUpVideo() {
      if (this.player && this.playerReady && this.isPlaying) {
        // Flag to track ongoing pull-up effect
        this.isPullingUp = true;
        
        // Pause the video 
        this.player.pauseVideo();
        
        // Play the vinyl stop sound
        this.playSound('vinyl_stop');
        
        // Set a longer default duration to ensure we catch the full vinyl stop sound
        const vinylStopDuration = 2.5; // Increased duration in seconds
        
        // Set a timeout to restart the video when the vinyl_stop sound is about to end
        setTimeout(() => {
          if (this.player && this.playerReady) {
            // Reset the pull-up flag before starting playback
            this.isPullingUp = false;
            
            // Seek to beginning and play
            this.player.seekTo(0, true);
            
            // Force play state
            const playPromise = this.player.playVideo();
            
            // Set playing state flags
            this.isPlaying = true;
            this.$emit('playing', true);
            
            console.log('Restarting track after vinyl stop effect');
          }
        }, (vinylStopDuration * 0.9) * 1000); // Restart at 90% of the vinyl stop sound duration
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
      event.dataTransfer.effectAllowed = 'copyMove';
      
      // Clean up the temporary element after a short delay
      setTimeout(() => {
        document.body.removeChild(dragImage);
      }, 100);
    },
    onDragOver(event) {
      event.preventDefault();
      this.isDraggingOver = true;
      // Add some visual feedback
      event.dataTransfer.dropEffect = 'move';
    },
    onDragLeave(event) {
      const rect = event.currentTarget.getBoundingClientRect();
      // Only set isDraggingOver to false if we're leaving the record/turntable
      // and not just moving between its child elements
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
          this.selectVideo(video);
          
          // Show a quick highlight effect on successful drop
          const target = event.currentTarget;
          target.classList.add('drop-highlight');
          setTimeout(() => {
            target.classList.remove('drop-highlight');
          }, 500);
        }
      } catch (error) {
        console.error('Error processing dropped video:', error);
      }
    },
    // New methods for tempo and pitch control
    updatePlaybackRate() {
      if (!this.player || !this.playerReady || !this.isPlaying) return;
      
      // Calculate effective playback rate based on tempo and pitch
      // Tempo: direct scaling factor (100% = normal, 50% = half speed, 150% = 1.5x speed)
      // Pitch: semitones adjustment (each semitone is ~5.95% change in frequency)
      
      // Base rate from tempo
      const tempoRate = this.tempoValue / 100;
      
      // Pitch adjustment (approximate conversion from percentage to semitones)
      // Each semitone is roughly 5.95% change
      const pitchFactor = Math.pow(2, this.pitchValue / 100 / 12);
      
      // Combine both for final rate
      // YouTube API allows rates from 0.25 to 2.0, so we need to clamp our values
      const finalRate = Math.max(0.25, Math.min(2.0, tempoRate * pitchFactor));
      
      console.log(`Setting playback rate to ${finalRate.toFixed(3)} (tempo: ${this.tempoValue}%, pitch: ${this.pitchValue}%)`);
      this.player.setPlaybackRate(finalRate);
      
      // Apply visual feedback based on pitch
      this.applyPitchVisualFeedback();
    },
    
    applyPitchVisualFeedback() {
      // Update record rotation speed based on pitch
      if (this.$refs.recordElement) {
        const record = this.$refs.recordElement;
        const speedFactor = Math.max(0.5, Math.min(3.0, 1 + (this.pitchValue / 100)));
        record.style.animationDuration = `${2 / speedFactor}s`;
      }
    },
    
    resetTempo() {
      this.tempoValue = 100;
      this.updatePlaybackRate();
    },
    
    resetPitch() {
      this.pitchValue = 0;
      this.updatePlaybackRate();
    },
    
    loadSoundEffects() {
      // Initialize Web Audio API when possible
      try {
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        console.log('Audio context initialized for sound effects');
        
        // Load audio files from the public directory
        const soundFiles = {
          airhorn: require('@/assets/sounds/airhorn.mp3'),
          scratch: require('@/assets/sounds/scratch.mp3'),
          bass: require('@/assets/sounds/bass.mp3'),
          siren: require('@/assets/sounds/siren.mp3'),
          vinyl_stop: require('@/assets/sounds/vinyl_stop.mp3')
        };
        
        // Load and decode each sound file
        Object.entries(soundFiles).forEach(([name, url]) => {
          this.loadSound(name, url);
        });
      } catch (e) {
        console.error('Error initializing audio context:', e);
        
        // Fallback to simple Audio API
        this.soundEffects.airhorn = new Audio(require('@/assets/sounds/airhorn.mp3'));
        this.soundEffects.scratch = new Audio(require('@/assets/sounds/scratch.mp3'));
        this.soundEffects.bass = new Audio(require('@/assets/sounds/bass.mp3'));
        this.soundEffects.siren = new Audio(require('@/assets/sounds/siren.mp3'));
        this.soundEffects.vinyl_stop = new Audio(require('@/assets/sounds/vinyl_stop.mp3'));
        
        Object.values(this.soundEffects).forEach(sound => {
          sound.load();
        });
      }
    },
    
    loadSound(name, url) {
      // Fetch audio file data
      fetch(url)
        .then(response => response.arrayBuffer())
        .then(arrayBuffer => {
          // Decode audio data
          return this.audioContext.decodeAudioData(arrayBuffer);
        })
        .then(audioBuffer => {
          // Store the decoded buffer
          this.audioBuffers[name] = audioBuffer;
          console.log(`Sound loaded: ${name}`);
        })
        .catch(error => {
          console.error(`Error loading sound ${name}:`, error);
        });
    },
    
    handleUserInteraction() {
      this.userInteracted = true;
      console.log('User has interacted with the page, audio can now play');
      
      // Resume audio context if it was suspended (browsers often require user interaction)
      if (this.audioContext && this.audioContext.state === 'suspended') {
        this.audioContext.resume().then(() => {
          console.log('AudioContext resumed successfully');
        }).catch(e => {
          console.error('Error resuming AudioContext:', e);
        });
      }
    },
    
    playSound(soundName) {
      console.log(`Attempting to play ${soundName} sound effect`);
      
      // Ensure user has interacted first (needed for most browsers)
      if (!this.userInteracted) {
        console.warn('Audio playback requires user interaction first');
        this.handleUserInteraction();
      }
      
      // Try to use Web Audio API first (better performance)
      if (this.audioContext && this.audioBuffers[soundName]) {
        try {
          // Create source node
          const source = this.audioContext.createBufferSource();
          source.buffer = this.audioBuffers[soundName];
          
          // Create gain node for volume control
          const gainNode = this.audioContext.createGain();
          gainNode.gain.value = 1.0; // Full volume
          
          // Connect nodes
          source.connect(gainNode);
          gainNode.connect(this.audioContext.destination);
          
          // Play the sound
          source.start(0);
          console.log(`Playing ${soundName} via Web Audio API`);
          return;
        } catch (e) {
          console.error(`Error playing ${soundName} with Web Audio API:`, e);
        }
      }
      
      // Fallback to Audio API
      if (this.soundEffects[soundName]) {
        try {
          // Create a new audio element each time for overlapping sounds
          const sound = new Audio(this.soundEffects[soundName].src);
          sound.volume = 1.0;
          
          // Play the sound
          const playPromise = sound.play();
          
          // Handle play promise (required for modern browsers)
          if (playPromise !== undefined) {
            playPromise.catch(error => {
              console.error(`Error playing ${soundName} with Audio API:`, error);
              
              // When autoplay is prevented, we need to manually enable it after user interaction
              document.addEventListener('click', () => {
                sound.play().catch(e => console.error('Retry failed:', e));
              }, { once: true });
            });
          }
        } catch (e) {
          console.error(`Error setting up ${soundName} with Audio API:`, e);
          
          // Last resort - create dynamic audio
          this.createSyntheticSound(soundName);
        }
      } else {
        // If all else fails, generate synthetic sound
        this.createSyntheticSound(soundName);
      }
    },
    
    createSyntheticSound(soundName) {
      // Create synthetic sound effects using oscillators as absolute fallback
      console.log(`Creating synthetic ${soundName} sound`);
      
      try {
        // Create audio context if it doesn't exist
        if (!this.audioContext) {
          this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        }
        
        // Generate different sounds based on name
        switch (soundName) {
          case 'airhorn':
            this.createAirhornSound();
            break;
          case 'scratch':
            this.createScratchSound();
            break;
          case 'bass':
            this.createBassSound();
            break;
          case 'siren':
            this.createSirenSound();
            break;
        }
      } catch (e) {
        console.error('Failed to create synthetic sound:', e);
      }
    },
    
    createAirhornSound() {
      const ctx = this.audioContext;
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      
      osc.type = 'square';
      osc.frequency.setValueAtTime(466, ctx.currentTime);
      
      gain.gain.setValueAtTime(0, ctx.currentTime);
      gain.gain.linearRampToValueAtTime(0.8, ctx.currentTime + 0.1);
      gain.gain.linearRampToValueAtTime(0.8, ctx.currentTime + 0.4);
      gain.gain.linearRampToValueAtTime(0, ctx.currentTime + 0.5);
      
      osc.connect(gain);
      gain.connect(ctx.destination);
      
      osc.start();
      osc.stop(ctx.currentTime + 0.5);
    },
    
    createScratchSound() {
      const ctx = this.audioContext;
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      const filter = ctx.createBiquadFilter();
      
      osc.type = 'sawtooth';
      osc.frequency.setValueAtTime(1000, ctx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(50, ctx.currentTime + 0.2);
      
      filter.type = 'lowpass';
      filter.frequency.value = 1500;
      filter.Q.value = 15;
      
      gain.gain.setValueAtTime(0.7, ctx.currentTime);
      gain.gain.linearRampToValueAtTime(0, ctx.currentTime + 0.2);
      
      osc.connect(filter);
      filter.connect(gain);
      gain.connect(ctx.destination);
      
      osc.start();
      osc.stop(ctx.currentTime + 0.2);
    },
    
    createBassSound() {
      const ctx = this.audioContext;
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      
      osc.type = 'sine';
      osc.frequency.setValueAtTime(100, ctx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(40, ctx.currentTime + 0.4);
      
      gain.gain.setValueAtTime(1, ctx.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.4);
      
      osc.connect(gain);
      gain.connect(ctx.destination);
      
      osc.start();
      osc.stop(ctx.currentTime + 0.4);
    },
    
    createSirenSound() {
      const ctx = this.audioContext;
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      
      osc.type = 'triangle';
      osc.frequency.setValueAtTime(440, ctx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(880, ctx.currentTime + 0.25);
      osc.frequency.exponentialRampToValueAtTime(440, ctx.currentTime + 0.5);
      
      gain.gain.setValueAtTime(0, ctx.currentTime);
      gain.gain.linearRampToValueAtTime(0.7, ctx.currentTime + 0.1);
      gain.gain.linearRampToValueAtTime(0.7, ctx.currentTime + 0.4);
      gain.gain.linearRampToValueAtTime(0, ctx.currentTime + 0.5);
      
      osc.connect(gain);
      gain.connect(ctx.destination);
      
      osc.start();
      osc.stop(ctx.currentTime + 0.5);
    },
  }
};
</script>

<style scoped>
.deck {
  position: relative;
  display: flex;
  flex-direction: column;
  min-height: 420px;
  transition: all 0.3s ease;
  width: 100%;
  border-radius: 10px;
  background: #2a2a2a;
  border: 1px solid transparent;
}

.deck-active {
  /* Adding a subtle glow to the active deck */
  box-shadow: 0 0 5px rgba(255, 255, 255, 0.2);
}

.neon-blue, .neon-red {
  position: relative;
  border-radius: 10px;
}

.neon-blue::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 0;
  border-radius: 10px;
  pointer-events: none;
  box-shadow: inset 0 0 2px 1px rgba(0, 119, 255, 0.3);
}

.neon-red::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 0;
  border-radius: 10px;
  pointer-events: none;
  box-shadow: inset 0 0 2px 1px rgba(255, 0, 85, 0.3);
}

.neon-blue::after, .neon-red::after {
  content: '';
  position: absolute;
  width: 10px;
  height: 10px;
  border-radius: 50%;
  filter: blur(2.5px);
  z-index: 3;
  pointer-events: none;
  /* Use linear animation timing to ensure consistent speed */
  animation: neon-border-travel 30s linear infinite;
}

.neon-blue::after {
  background: rgb(56, 255, 255);
  box-shadow: 
    0 0 6px 2px rgb(56, 255, 255),
    0 0 12px 4px rgba(0, 225, 255, 1);
}

.neon-red::after {
  background: rgb(255, 56, 112);
  box-shadow: 
    0 0 6px 2px rgb(255, 56, 112),
    0 0 12px 4px rgba(255, 0, 85, 1);
}

@keyframes neon-border-travel {
  /* Top edge: left to right (0-25%) */
  0% {
    top: 0;
    left: 0;
    transform: translateY(-50%);
  }
  6.25% {
    top: 0;
    left: 25%;
    transform: translateY(-50%);
  }
  12.5% {
    top: 0;
    left: 50%;
    transform: translateY(-50%);
  }
  18.75% {
    top: 0;
    left: 75%;
    transform: translateY(-50%);
  }
  25% {
    top: 0;
    left: 100%;
    transform: translateY(-50%);
  }

  /* Right edge: top to bottom (25-50%) */
  31.25% {
    top: 25%;
    left: 100%;
    transform: translateX(-50%);
  }
  37.5% {
    top: 50%;
    left: 100%;
    transform: translateX(-50%);
  }
  43.75% {
    top: 75%;
    left: 100%;
    transform: translateX(-50%);
  }
  50% {
    top: 100%;
    left: 100%;
    transform: translate(-50%, -50%);
  }
  
  /* Bottom edge: right to left (50-75%) */
  56.25% {
    top: 100%;
    left: 75%;
    transform: translateY(-50%);
  }
  62.5% {
    top: 100%;
    left: 50%;
    transform: translateY(-50%);
  }
  68.75% {
    top: 100%;
    left: 25%;
    transform: translateY(-50%);
  }
  75% {
    top: 100%;
    left: 0;
    transform: translateY(-50%);
  }
  
  /* Left edge: bottom to top (75-100%) */
  81.25% {
    top: 75%;
    left: 0;
    transform: translateX(-50%);
  }
  87.5% {
    top: 50%;
    left: 0;
    transform: translateX(-50%);
  }
  93.75% {
    top: 25%;
    left: 0;
    transform: translateX(-50%);
  }
  100% {
    top: 0;
    left: 0;
    transform: translateY(-50%);
  }
}

.deck-title {
  text-align: center;
  font-size: 1.3rem;
  font-weight: bold;
  margin-bottom: 12px;
  padding: 5px;
  background: linear-gradient(to right, #ff0000, #0066ff);
  border-radius: 5px;
  color: white;
}

.deck-content {
  position: relative;
  z-index: 2;
  display: flex;
  flex: 1;
  gap: 12px;
  flex-wrap: wrap;
}

/* Turntable section styles */
.turntable-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  min-width: 160px;
  padding-bottom: 15px;
}

.record-container, .empty-turntable {
  position: relative;
  width: 130px;
  height: 130px;
  margin: 0 auto 12px;
}

.record, .empty-record {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border-radius: 50%;
  background: linear-gradient(145deg, #111, #333);
  display: flex;
  justify-content: center;
  align-items: center;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.7);
  transition: transform 0.5s ease;
}

.empty-record {
  background: linear-gradient(145deg, #222, #444);
}

.record-spinning {
  animation: spin 2s linear infinite;
}

.record-reverse-spin {
  animation: reverse-spin 0.05s linear infinite;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

@keyframes reverse-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(-360deg);
  }
}

.record-label {
  position: relative;
  width: 70px;
  height: 70px;
  border-radius: 50%;
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
}

.record-artwork {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.record-center {
  position: absolute;
  width: 18px;
  height: 18px;
  background-color: #ddd;
  border-radius: 50%;
  border: 2px solid #555;
}

.video-title, .empty-title {
  text-align: center;
  font-size: 12px;
  margin-bottom: 10px;
  height: 30px;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  width: 100%;
  max-width: 200px;
}

.empty-title {
  color: #777;
  font-style: italic;
}

.deck-controls {
  display: flex;
  justify-content: center;
  gap: 4px;
  margin-bottom: 8px;
  flex-wrap: wrap;
  width: 100%;
  max-width: 200px;
}

.deck-controls button {
  padding: 5px 7px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 11px;
  font-weight: bold;
}

.deck-controls button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.play-btn {
  background-color: #1db954;
  color: white;
}

.pause-btn {
  background-color: #ffcc00;
  color: #333;
}

.stop-btn {
  background-color: #ff3333;
  color: white;
}

.eject-btn {
  background-color: #666;
  color: white;
}

.pullup-btn {
  background: linear-gradient(to bottom, #ff6600, #cc3300);
  color: white;
  text-shadow: 0 0 3px rgba(0, 0, 0, 0.5);
  font-weight: bold;
  border: 1px solid #ff2200;
}

.pullup-btn:hover {
  background: linear-gradient(to bottom, #ff7722, #dd4400);
}

.volume-controls {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  font-size: 12px;
  max-width: 200px;
}

.volume-slider-container {
  width: 90%;
}

.volume-slider {
  width: 100%;
  -webkit-appearance: none;
  height: 6px;
  background: linear-gradient(to right, #ff0000, #00ff00);
  outline: none;
  border-radius: 3px;
}

.volume-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 12px;
  height: 12px;
  background: #ddd;
  border-radius: 50%;
  cursor: pointer;
}

/* Search section styles */
.search-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 140px;
}

.search-container {
  display: flex;
  margin-bottom: 8px;
}

.search-container input {
  flex: 1;
  padding: 6px;
  font-size: 12px;
  border: 1px solid #444;
  background-color: #222;
  color: white;
  border-radius: 4px 0 0 4px;
}

.search-container button {
  padding: 6px 10px;
  background-color: #ff0000;
  color: white;
  border: none;
  border-radius: 0 4px 4px 0;
  cursor: pointer;
  font-size: 12px;
}

.search-container button:disabled {
  background-color: #555;
}

.search-results {
  flex: 1;
  overflow-y: auto;
  background-color: #222;
  border-radius: 4px;
  padding: 4px;
  max-height: 250px;
}

.search-result-item {
  display: flex;
  padding: 6px;
  border-bottom: 1px solid #333;
  cursor: grab;
  transition: all 0.2s ease;
}

.search-result-item:hover {
  background-color: #333;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.search-result-item:active {
  cursor: grabbing;
}

.search-result-item img {
  width: 45px;
  height: 34px;
  object-fit: cover;
  border-radius: 3px;
}

.search-result-info {
  margin-left: 8px;
  overflow: hidden;
}

.search-result-title {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  font-size: 12px;
}

.search-error, .search-loading {
  text-align: center;
  margin-top: 8px;
  font-size: 12px;
}

.search-error {
  color: #ff5555;
}

.player-status {
  text-align: center;
  margin-bottom: 8px;
  color: #ffcc00;
  font-size: 12px;
  font-style: italic;
}

.youtube-player-container {
  position: absolute;
  opacity: 0;
  pointer-events: none;
}

/* Media queries for responsiveness */
@media (max-width: 640px) {
  .deck-content {
    flex-direction: column;
  }
  
  .turntable-section, .search-section {
    width: 100%;
  }
  
  .search-section {
    max-height: 180px;
  }
  
  .search-results {
    max-height: 120px;
  }

  /* Adjust controls layout for medium screens */
  .controls-container {
    flex-direction: row;
    justify-content: center;
    width: 100%;
    margin-left: 0;
  }
}

@media (max-width: 480px) {
  .deck-title {
    font-size: 1.1rem;
    margin-bottom: 8px;
    padding: 4px;
  }
  
  .record-container, .empty-turntable {
    width: 110px;
    height: 110px;
  }
  
  .record-label {
    width: 60px;
    height: 60px;
  }

  /* Stack controls vertically on very small screens */
  .controls-container {
    flex-direction: column;
    align-items: center;
    flex-wrap: wrap;
    margin-left: 0;
  }

  .tempo-pitch-controls, .sound-effects {
    width: 100%;
    max-width: 200px;
  }
}

/* Timeline styles */
.timeline-container {
  width: 100%;
  max-width: 200px;
  margin: 8px 0;
}

.time-display {
  display: flex;
  justify-content: space-between;
  font-size: 10px;
  color: #ddd;
  margin-bottom: 3px;
}

.timeline-slider {
  width: 100%;
  height: 8px;
  -webkit-appearance: none;
  background: linear-gradient(to right, #ff9900, #ff0099);
  border-radius: 4px;
  outline: none;
  opacity: 0.7;
  transition: opacity 0.2s;
}

.timeline-slider:hover {
  opacity: 1;
}

.timeline-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: #fff;
  cursor: pointer;
  box-shadow: 0 0 3px rgba(0, 0, 0, 0.5);
}

.timeline-slider::-moz-range-thumb {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: #fff;
  cursor: pointer;
  box-shadow: 0 0 3px rgba(0, 0, 0, 0.5);
}

/* Fire animation styles removed */

/* End of styles */

.drop-area {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  border-radius: 10px;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;
  border: 3px dashed #fff;
  pointer-events: none;
}

.drop-area-record {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.6);
  border-radius: 50%;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;
  border: 2px dashed #fff;
  pointer-events: none;
  animation: pulse 1.5s infinite alternate;
}

@keyframes pulse {
  from {
    box-shadow: 0 0 5px rgba(255, 255, 255, 0.5);
  }
  to {
    box-shadow: 0 0 15px rgba(255, 255, 255, 0.8);
  }
}

.drop-message {
  color: white;
  font-size: 1rem;
  font-weight: bold;
  text-shadow: 0 0 5px rgba(0, 0, 0, 0.5);
  background-color: rgba(0, 0, 0, 0.5);
  padding: 5px 10px;
  border-radius: 10px;
}

.drop-highlight {
  animation: highlight-drop 0.5s ease-out;
}

@keyframes highlight-drop {
  0% {
    transform: scale(1);
    box-shadow: 0 0 0 rgba(255, 255, 255, 0);
  }
  50% {
    transform: scale(1.05);
    box-shadow: 0 0 20px rgba(255, 255, 255, 0.8);
  }
  100% {
    transform: scale(1);
    box-shadow: 0 0 0 rgba(255, 255, 255, 0);
  }
}

/* Tempo and Pitch control styles */
.tempo-pitch-controls {
  flex: 0 0 auto;
  width: calc(50% - 6px);
  display: flex;
  flex-direction: column;
  gap: 12px;
  background: rgba(30, 30, 30, 0.5);
  padding: 10px;
  border-radius: 8px;
  max-width: 180px;
}

.tempo-control, .pitch-control {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}

.tempo-control label, .pitch-control label {
  font-size: 12px;
  margin-bottom: 5px;
  font-weight: bold;
  width: 100%;
  display: flex;
  justify-content: space-between;
}

.control-slider-container {
  width: 100%;
  position: relative;
  margin-bottom: 5px;
}

.tempo-slider, .pitch-slider {
  width: 100%;
  -webkit-appearance: none;
  height: 6px;
  outline: none;
  border-radius: 3px;
  margin-bottom: 14px; /* Space for markers */
}

.tempo-slider {
  background: linear-gradient(to right, #4444ff, #00ccff, #44ff44);
}

.pitch-slider {
  background: linear-gradient(to right, #ff0000, #ffff00, #ffffff, #00ffff, #0000ff);
  z-index: 2;
  position: relative;
}

.pitch-markers {
  position: absolute;
  bottom: -2px;
  left: 0;
  right: 0;
  display: flex;
  justify-content: space-between;
  font-size: 8px;
  color: #aaa;
}

.marker {
  position: relative;
}

.marker::before {
  content: '';
  position: absolute;
  top: -14px;
  left: 50%;
  width: 1px;
  height: 4px;
  background-color: #aaa;
  transform: translateX(-50%);
}

.marker.center::before {
  height: 6px;
  background-color: #fff;
}

.marker.low {
  color: #ff4444;
}

.marker.high {
  color: #4444ff;
}

.pitch-indicator {
  position: absolute;
  width: 2px;
  height: 10px;
  background-color: #fff;
  top: -2px;
  transform: translateX(-50%);
  z-index: 3;
  box-shadow: 0 0 3px rgba(255, 255, 255, 0.8);
}

.tempo-slider::-webkit-slider-thumb, .pitch-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: #ddd;
  cursor: pointer;
  box-shadow: 0 0 3px rgba(0, 0, 0, 0.5);
  z-index: 3;
  position: relative;
}

.tempo-slider:disabled, .pitch-slider:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.tempo-slider:disabled::-webkit-slider-thumb, .pitch-slider:disabled::-webkit-slider-thumb {
  cursor: not-allowed;
}

.reset-btn {
  background-color: #555;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 2px 6px;
  font-size: 10px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.reset-btn:hover:not(:disabled) {
  background-color: #777;
}

.reset-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Removed grab cursor styles */

/* Removed scratch visual feedback styles */

/* Removed scratch tip tooltip styles */

/* Sound Effects Styles */
.sound-effects {
  flex: 0 0 auto;
  width: calc(50% - 6px);
  margin-top: 0;
  background: rgba(30, 30, 30, 0.5);
  padding: 10px;
  border-radius: 8px;
  max-width: 180px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.sound-effects-title {
  font-size: 12px;
  font-weight: bold;
  margin-bottom: 8px;
  text-align: center;
  color: #ddd;
}

.sound-effects-buttons {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 8px;
  margin-top: auto;
}

.sound-effect-btn {
  padding: 6px 0;
  border: none;
  border-radius: 4px;
  font-size: 10px;
  font-weight: bold;
  color: white;
  cursor: pointer !important;
  transition: transform 0.2s, box-shadow 0.2s;
  position: relative;
  z-index: 2;
}

.sound-effect-btn:hover {
  transform: scale(1.05);
  box-shadow: 0 0 5px rgba(255, 255, 255, 0.5);
  cursor: pointer !important;
}

.sound-effect-btn:active {
  transform: scale(0.95);
  cursor: pointer !important;
}

.airhorn-btn {
  background: linear-gradient(to bottom, #ff4500, #cc3700);
}

.scratch-btn {
  background: linear-gradient(to bottom, #4da6ff, #0066cc);
}

.bass-btn {
  background: linear-gradient(to bottom, #9933ff, #6600cc);
}

.siren-btn {
  background: linear-gradient(to bottom, #ffcc00, #cc9900);
}

/* Controls container to hold tempo/pitch and effects side by side */
.controls-container {
  display: flex;
  flex-direction: row;
  justify-content: flex-start;
  gap: 12px;
  width: 100%;
  margin-top: 12px;
  margin-bottom: 10px;
  margin-left: 20px;
  flex-wrap: nowrap;
}
</style> 