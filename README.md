# Pro Mixer DJ App

A professional DJ mixing application built with Vue.js that integrates with the YouTube Data API for music streaming and mixing capabilities.

## Project Setup

```
npm install
```

### Compile and Hot-Reload for Development

```
npm run serve
```

### Compile and Minify for Production

```
npm run build
```

## Features

- **Dual Deck DJ Interface**: Two independent audio decks for mixing
- **YouTube Music Integration**: Search and stream music directly from YouTube
- **Advanced Audio Controls**: Tempo, pitch, and crossfade controls
- **Sound Effects**: Airhorn, scratch, bass, siren, and vinyl stop effects
- **Playlist Management**: Create and manage music playlists
- **Professional UI**: Modern, responsive design with smooth animations
- **Real-time Mixing**: Live crossfading between decks

## Technical Details

- Built with Vue.js 3
- YouTube Data API v3 integration
- Web Audio API for sound effects
- Responsive design for all devices
- Local storage for playlist persistence

## API Requirements

This app requires a YouTube Data API key. Note that the free tier has daily quota limits (10,000 units/day) which may affect extended usage.

## Usage

1. Start the development server with `npm run serve`
2. Access the app at `http://localhost:8080`
3. Search for music using the YouTube integration
4. Load tracks into the dual decks
5. Mix and create your DJ set with crossfade controls
