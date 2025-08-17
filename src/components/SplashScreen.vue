<template>
  <div 
    class="splash-screen" 
    :class="{ 'fade-out': startFadeOut }"
  >
    <!-- Particle background -->
    <div class="particles">
      <div v-for="n in 40" :key="`particle-${n}`" class="particle"></div>
    </div>
    
    <div class="laser-container">
      <div class="laser laser-1"></div>
      <div class="laser laser-2"></div>
      <div class="laser laser-3"></div>
      <div class="laser laser-4"></div>
      <div class="laser laser-5"></div>
    </div>
    
    <div class="content">
      <div class="logo-container">
        <img src="../assets/images/logo.png" alt="Pro Mixer Logo" class="app-logo" />
        <div class="waveform">
          <div v-for="n in 20" :key="n" class="bar"></div>
        </div>
        
        <!-- Pulse circles -->
        <div class="pulse-circles">
          <div class="pulse-circle"></div>
          <div class="pulse-circle"></div>
          <div class="pulse-circle"></div>
        </div>
      </div>
      
      <div class="loading-area">
        <div class="equalizer">
          <div v-for="n in 10" :key="n" class="eq-bar"></div>
        </div>
        <div class="loading-text">
          <span>L</span>
          <span>O</span>
          <span>A</span>
          <span>D</span>
          <span>I</span>
          <span>N</span>
          <span>G</span>
          <span>.</span>
          <span>.</span>
          <span>.</span>
        </div>
        <div class="loading-bar">
          <div class="progress"></div>
        </div>
      </div>
      
      <div class="turntable-animation">
        <div class="deck deck-left"></div>
        <div class="mixer"></div>
        <div class="deck deck-right"></div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'SplashScreen',
  props: {
    loadingComplete: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      startFadeOut: false,
      startTime: null
    };
  },
  watch: {
    loadingComplete(newVal) {
      console.log(`loadingComplete changed to: ${newVal}`);
      if (newVal) {
        console.log('Starting splash screen fade-out sequence');
        // Start the fade out animation immediately when loadingComplete becomes true
        this.startFadeOut = true;
      }
    }
  },
  mounted() {
    console.log('SplashScreen mounted - Will display for 10 seconds');
    // Track when the component is mounted
    this.startTime = Date.now();
    
    // Add dynamic waveform animation
    this.animateWaveform();
    
    // Animate equalizer bars
    this.animateEqualizer();
    
    // Initialize particles
    this.initParticles();
  },
  methods: {
    animateWaveform() {
      const bars = document.querySelectorAll('.waveform .bar');
      bars.forEach((bar, index) => {
        const height = Math.floor(Math.random() * 50) + 10;
        const delay = index * 60;
        
        setInterval(() => {
          const newHeight = Math.floor(Math.random() * 50) + 10;
          bar.style.height = `${newHeight}px`;
        }, 500);
        
        // Set initial height with delay
        setTimeout(() => {
          bar.style.height = `${height}px`;
        }, delay);
      });
    },
    
    animateEqualizer() {
      const eqBars = document.querySelectorAll('.equalizer .eq-bar');
      eqBars.forEach((bar) => {
        setInterval(() => {
          const height = Math.floor(Math.random() * 100);
          bar.style.height = `${height}%`;
        }, 200);
      });
    },
    
    initParticles() {
      const particles = document.querySelectorAll('.particle');
      particles.forEach(particle => {
        // Random position, size and animation delay
        const size = Math.random() * 6 + 2;
        const posX = Math.random() * 100;
        const posY = Math.random() * 100;
        const delay = Math.random() * 5;
        const duration = Math.random() * 10 + 10;
        
        particle.style.width = `${size}px`;
        particle.style.height = `${size}px`;
        particle.style.left = `${posX}%`;
        particle.style.top = `${posY}%`;
        particle.style.animationDelay = `${delay}s`;
        particle.style.animationDuration = `${duration}s`;
      });
    }
  }
};
</script>

<style scoped>
.splash-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(135deg, #0a0a0a 0%, #1e1e1e 100%);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
  transition: opacity 2s ease-out; /* Shorter fade-out transition */
  overflow: hidden;
}

.splash-screen.fade-out {
  opacity: 0;
  pointer-events: none;
}

.content {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100%;
  max-width: 600px;
  perspective: 1000px;
  transform-style: preserve-3d;
  animation: subtle-rotate 10s ease-in-out infinite alternate;
  position: relative;
  z-index: 1;
}

.logo-container {
  position: relative;
  margin-bottom: 40px;
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.app-logo {
  width: 200px;
  height: auto;
  margin: 0 auto;
  filter: drop-shadow(0 0 20px rgba(255, 0, 0, 0.5)) drop-shadow(0 0 40px rgba(0, 0, 255, 0.3));
  animation: pulse-glow 5s ease-in-out infinite alternate;
  z-index: 1;
}

@keyframes pulse-glow {
  0% {
    filter: drop-shadow(0 0 15px rgba(255, 0, 0, 0.5)) drop-shadow(0 0 30px rgba(0, 0, 255, 0.3));
    transform: scale(1);
  }
  100% {
    filter: drop-shadow(0 0 25px rgba(255, 0, 0, 0.7)) drop-shadow(0 0 50px rgba(0, 0, 255, 0.5));
    transform: scale(1.05);
  }
}

.waveform {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 80px;
  margin-top: 30px;
  gap: 5px;
  width: 100%;
}

.waveform .bar {
  width: 4px;
  background: linear-gradient(to top, #00ccff, #ff00cc);
  border-radius: 2px;
  transition: height 0.2s ease;
}

.loading-area {
  width: 100%;
  max-width: 400px;
  margin-bottom: 30px;
}

.equalizer {
  display: flex;
  align-items: flex-end;
  justify-content: center;
  height: 40px;
  gap: 4px;
  margin-bottom: 20px;
}

.equalizer .eq-bar {
  width: 8px;
  background: linear-gradient(to top, #ff3300, #ffcc00);
  border-radius: 2px;
  transition: height 0.2s ease;
}

.loading-text {
  text-align: center;
  margin-bottom: 15px;
  font-size: 20px;
  color: white;
  letter-spacing: 4px;
  text-transform: uppercase;
  font-weight: bold;
  position: relative;
}

.loading-text::before {
  content: '';
  position: absolute;
  top: -5px;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent 0%, rgba(255, 255, 255, 0.2) 50%, transparent 100%);
  animation: loading-text-shine 2s linear infinite;
  z-index: 1;
  pointer-events: none;
}

@keyframes loading-text-shine {
  0% {
    transform: translateX(-100%);
  }
  100% {
    transform: translateX(100%);
  }
}

.loading-text span {
  display: inline-block;
  animation: pulse 1.5s infinite alternate;
  position: relative;
  backdrop-filter: blur(10px);
  padding: 0 3px;
}

.loading-text span:nth-child(1) { animation-delay: 0.0s; }
.loading-text span:nth-child(2) { animation-delay: 0.1s; }
.loading-text span:nth-child(3) { animation-delay: 0.2s; }
.loading-text span:nth-child(4) { animation-delay: 0.3s; }
.loading-text span:nth-child(5) { animation-delay: 0.4s; }
.loading-text span:nth-child(6) { animation-delay: 0.5s; }
.loading-text span:nth-child(7) { animation-delay: 0.6s; }
.loading-text span:nth-child(8) { animation-delay: 0.7s; }
.loading-text span:nth-child(9) { animation-delay: 0.8s; }
.loading-text span:nth-child(10) { animation-delay: 0.9s; }

.loading-bar {
  width: 100%;
  height: 10px;
  background: rgba(20, 20, 20, 0.5);
  border-radius: 5px;
  overflow: hidden;
  position: relative;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(5px);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.loading-bar .progress {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  width: 0%;
  background: linear-gradient(90deg, 
    #00ccff, 
    #0099ff, 
    #cc00ff, 
    #ff0099, 
    #ff3300, 
    #ffcc00, 
    #00ccff);
  background-size: 200% 100%;
  border-radius: 5px;
  animation: 
    progress 5s ease-in-out forwards,
    gradient-shift 3s linear infinite;
}

@keyframes gradient-shift {
  0% {
    background-position: 0% 50%;
  }
  100% {
    background-position: 100% 50%;
  }
}

.turntable-animation {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  margin-top: 20px;
  transform-style: preserve-3d;
  animation: float 4s ease-in-out infinite;
  filter: drop-shadow(0 10px 20px rgba(0, 0, 0, 0.7));
}

.deck {
  width: 120px;
  height: 120px;
  background: linear-gradient(135deg, #333, #111);
  border-radius: 10px;
  position: relative;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.5);
  transform-style: preserve-3d;
  transform: perspective(500px) rotateX(20deg);
}

.deck::after {
  content: '';
  position: absolute;
  top: 20px;
  left: 20px;
  width: 80px;
  height: 80px;
  background: #000;
  border-radius: 50%;
  box-shadow: 0 0 10px rgba(255, 255, 255, 0.2);
  animation: spin 3s linear infinite;
}

.deck.deck-left::after {
  background: linear-gradient(135deg, #ff0055, #111);
}

.deck.deck-right::after {
  background: linear-gradient(135deg, #0066ff, #111);
}

.mixer {
  width: 80px;
  height: 100px;
  background: linear-gradient(135deg, #444, #222);
  border-radius: 8px;
  margin: 0 15px;
  position: relative;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.5);
}

.mixer::before, .mixer::after {
  content: '';
  position: absolute;
  width: 60px;
  height: 10px;
  background: linear-gradient(90deg, #ff0055, #0066ff);
  border-radius: 5px;
  left: 10px;
}

.mixer::before {
  top: 25px;
  animation: slider 4s ease-in-out infinite alternate;
}

.mixer::after {
  bottom: 25px;
  animation: slider 3s ease-in-out infinite alternate-reverse;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

@keyframes anti-spin {
  from {
    transform: rotate(0deg) scale(1);
  }
  to {
    transform: rotate(-360deg) scale(1);
  }
}

@keyframes pulse {
  from {
    transform: scale(1);
    opacity: 0.5;
    color: #00ccff;
    text-shadow: 0 0 5px rgba(0, 204, 255, 0.7);
  }
  to {
    transform: scale(1.1);
    opacity: 1;
    color: #ff00cc;
    text-shadow: 0 0 10px rgba(255, 0, 204, 0.9);
  }
}

@keyframes progress {
  0% {
    width: 0%;
    box-shadow: 0 0 10px rgba(0, 204, 255, 0.5);
  }
  50% {
    width: 50%;
    box-shadow: 0 0 20px rgba(204, 0, 255, 0.7);
  }
  90% {
    width: 90%;
    box-shadow: 0 0 25px rgba(255, 165, 0, 0.8);
  }
  100% {
    width: 100%;
    box-shadow: 0 0 30px rgba(255, 255, 255, 0.9);
  }
}

@keyframes float {
  0% {
    transform: translateY(0px) rotateX(5deg);
  }
  50% {
    transform: translateY(-10px) rotateX(0deg);
  }
  100% {
    transform: translateY(0px) rotateX(5deg);
  }
}

@keyframes slider {
  from {
    transform: translateX(-10px);
  }
  to {
    transform: translateX(10px);
  }
}

.laser-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  z-index: -1;
}

.laser {
  position: absolute;
  height: 1px;
  width: 100%;
  background: rgba(255, 0, 98, 0.8);
  box-shadow: 0 0 10px 3px rgba(255, 0, 98, 0.8);
  transform-origin: 0 0;
  opacity: 0.6;
}

.laser-1 {
  top: 30%;
  left: -10%;
  width: 200%;
  transform: rotate(-5deg);
  animation: laser-move 7s ease-in-out infinite alternate;
  box-shadow: 0 0 10px 2px rgba(255, 0, 98, 0.8);
}

.laser-2 {
  top: 60%;
  left: -5%;
  width: 130%;
  transform: rotate(3deg);
  animation: laser-move 6s ease-in-out infinite alternate-reverse;
  box-shadow: 0 0 10px 2px rgba(0, 255, 234, 0.8);
  background: rgba(0, 255, 234, 0.8);
}

.laser-3 {
  top: 20%;
  left: -15%;
  width: 150%;
  transform: rotate(7deg);
  animation: laser-move 9s ease-in-out infinite alternate;
  animation-delay: 1s;
  box-shadow: 0 0 10px 2px rgba(174, 0, 255, 0.8);
  background: rgba(174, 0, 255, 0.8);
}

.laser-4 {
  top: 80%;
  left: -10%;
  width: 180%;
  transform: rotate(-8deg);
  animation: laser-move 8s ease-in-out infinite alternate-reverse;
  animation-delay: 2s;
  box-shadow: 0 0 10px 2px rgba(255, 234, 0, 0.8);
  background: rgba(255, 234, 0, 0.8);
}

.laser-5 {
  top: 40%;
  left: -5%;
  width: 150%;
  transform: rotate(2deg);
  animation: laser-move 10s ease-in-out infinite alternate;
  animation-delay: 1.5s;
  box-shadow: 0 0 10px 2px rgba(0, 255, 115, 0.8);
  background: rgba(0, 255, 115, 0.8);
}

@keyframes laser-move {
  0% {
    transform: rotate(var(--start-angle, -10deg));
  }
  100% {
    transform: rotate(var(--end-angle, 10deg));
  }
}

/* Particles */
.particles {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  z-index: 0;
}

.particle {
  position: absolute;
  background: white;
  border-radius: 50%;
  opacity: 0;
  animation: particle-float 15s linear infinite;
  z-index: 0;
}

@keyframes particle-float {
  0%, 100% {
    opacity: 0;
    transform: translateY(0) translateX(0) rotate(0deg);
  }
  10% {
    opacity: 0.8;
  }
  50% {
    opacity: 0.4;
  }
  90% {
    opacity: 0.8;
  }
  50% {
    transform: translateY(-100px) translateX(100px) rotate(180deg);
  }
}

/* Pulse circles */
.pulse-circles {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 0;
}

.pulse-circle {
  position: absolute;
  border-radius: 50%;
  box-sizing: border-box;
  border: 2px solid rgba(255, 255, 255, 0.2);
  animation: pulse-out 5s infinite;
  top: 0;
  left: 0;
  transform: translate(-50%, -50%);
}

.pulse-circle:nth-child(1) {
  width: 250px;
  height: 250px;
  border-color: rgba(255, 0, 128, 0.2);
  animation-delay: 0s;
}

.pulse-circle:nth-child(2) {
  width: 350px;
  height: 350px;
  border-color: rgba(0, 170, 255, 0.2);
  animation-delay: 1s;
}

.pulse-circle:nth-child(3) {
  width: 450px;
  height: 450px;
  border-color: rgba(255, 230, 0, 0.2);
  animation-delay: 2s;
}

@keyframes pulse-out {
  0% {
    transform: translate(-50%, -50%) scale(0.5);
    opacity: 1;
  }
  100% {
    transform: translate(-50%, -50%) scale(1.5);
    opacity: 0;
  }
}

/* 3D transformations */
@keyframes subtle-rotate {
  0% {
    transform: rotateX(3deg) rotateY(-3deg);
  }
  100% {
    transform: rotateX(-3deg) rotateY(3deg);
  }
}
</style> 