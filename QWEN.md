# Multi-Embed Plugin for Lampa

## Product Overview

The Multi-Embed Plugin is a Lampa application extension that provides multiple streaming source options for movies and TV shows. It integrates directly into the Lampa media center interface, allowing users to select from various embed providers when standard playback options are unavailable.

## Key Features

### 1. Multiple Source Providers
The plugin aggregates content from 9 different embed sources:
- **VidSrc.to** (Priority 1) - Primary source, often provides direct HLS streams
- **VidSrc.xyz** (Priority 2) - Reliable fallback
- **Embed.su** (Priority 3) - Good quality streams
- **AutoEmbed.co** (Priority 4) - Alternative provider
- **2Embed.to** (Priority 5) - Additional backup
- **MultiEmbed.mov** (Priority 6) - Mixed type, may provide direct m3u8/HLS links
- **SmashyStream** (Priority 7) - Streaming alternative
- **SuperEmbed** (Priority 8) - Backup source
- **MoviesAPI** (Priority 9) - Final fallback

### 2. Source Testing Functionality
- Built-in source availability testing before playback
- 8-second timeout for unresponsive sources
- Visual feedback via Lampa Notifier system
- "Source Working" or "Source Failed" notifications

### 3. Improved Navigation
- Fixed scrolling issue - now properly navigates through all sources using up/down controls
- Focus management with visual highlighting
- OK button or Right directional button to test and play selected source
- Hover/enter also triggers test and play

### 4. Direct Link Support
- Sources are categorized by type:
  - **iframe**: Traditional embedded player
  - **mixed**: May provide direct HLS/m3u8 streams (e.g., MultiEmbed.mov)
- The plugin attempts to fetch direct stream URLs where available
- Uses Lampa's native iframe player for compatibility

## Technical Implementation

### Architecture
```javascript
(function() {
    // 1. getSources(movie) - Returns prioritized list of embed URLs
    // 2. testSource(source) - Async function to test source availability
    // 3. MultiEmbedComponent - Lampa UI component with navigation
    // 4. UI Integration - Button injection on movie/TV detail pages
})();
```

### Source Types Explained

| Type | Description | Example |
|------|-------------|---------|
| `iframe` | Embedded web player that loads in Lampa's iframe player | Most sources |
| `mixed` | May redirect to direct HLS/m3u8 or provide iframe | MultiEmbed.mov |

### URL Patterns

**For Movies:**
- VidSrc.to: `https://vidsrc.to/embed/movie/{tmdbId}`
- MultiEmbed: `https://multiembed.mov/?video_id={tmdbId}&tmdb=1`

**For TV Series:**
- VidSrc.to: `https://vidsrc.to/embed/tv/{tmdbId}/{season}/{episode}`
- MultiEmbed: `https://multiembed.mov/directstream.php?video_id={tmdbId}&tmdb=1&s={season}&e={episode}`

## Changes Made

### Bug Fixes
1. **Navigation Fix**: Implemented proper `selectedIndex` tracking with `updateFocus()` method to ensure down navigation works correctly through all sources
2. **Scroll Integration**: Added `scroll.scrollToIndex(selectedIndex)` to keep visible area synchronized with selection

### New Features
1. **Source Testing**: Added `testSource()` async function with 8-second timeout
2. **Priority System**: Sources sorted by reliability (VidSrc.to first)
3. **Visual Feedback**: Notifier messages show testing status and results
4. **Type Indicators**: UI shows whether source is iframe or may provide direct HLS
5. **Multiple Trigger Options**: Test/play on hover:enter, OK button, or Right button

### Code Improvements
- Removed problematic `Lampa.Utils.escape` calls
- Added proper focus state management
- Implemented cleanup in `destroy()` method
- Better error handling with AbortController for fetch timeouts

## Usage Instructions

1. **Installation**: Load the `ry` script in your Lampa application
2. **Navigation**: 
   - Open any movie or TV show detail page
   - Press the "🌐 Multi-Embed" button
   - Use UP/DOWN to navigate sources
   - Press OK or RIGHT to test and play a source
3. **Testing**: Sources are automatically tested when selected; working sources will begin playback

## Troubleshooting

### "Corrupted or Not Found" Errors
These typically occur when:
- The embed source is temporarily down
- Geographic restrictions block access
- The source requires specific headers/referers

**Solution**: Try alternative sources in the list; they're ordered by reliability.

### Navigation Issues
If you can't scroll down:
- Ensure you're in the content controller (should be automatic)
- Check that all sources loaded (check console for errors)

## Notes on Direct Streams

Some sources like **MultiEmbed.mov** may return direct HLS (.m3u8) or MP4 links instead of requiring an iframe. The plugin handles both types through Lampa's player. For true direct link extraction, additional parsing of the iframe response would be required, which is beyond the scope of this plugin due to CORS restrictions.

## License

This plugin is provided as-is for educational purposes. Ensure compliance with your local laws and terms of service when accessing streaming content.

## Direct Stream Update

The plugin no longer falls back to iframe playback for selected sources. The source picker now resolves a direct stream via `getDirectStreamUrl(source, movie)` and only calls `Lampa.Player.play` with extracted `.m3u8`, `.mp4`, or `.mpd` URLs. When no direct media URL is found, the UI displays a "No direct stream" notification because iframe embeds are not usable by the target Lampa players.

### Navigation and Resolver Fix

The source picker now builds the list before controller activation, tracks rendered items separately from provider metadata, and routes OK/RIGHT through `playSelected()` so the selected source object is resolved correctly. The resolver also scans encoded provider payloads, including URL-encoded and base64/base64url-looking strings, while keeping playback direct-media-only.
