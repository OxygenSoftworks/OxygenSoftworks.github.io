# Lampa Plugin Development Guide

## Overview

This guide explains how Lampa plugins work, including the architecture, API methods, and best practices for creating plugins that extract and play direct video streams (m3u8, mp4, ts) instead of using iframes.

## Lampa Architecture

### Core Components

1. **Lampa.Controller** - Manages navigation and focus
2. **Lampa.Scroll** - Handles scrolling through lists
3. **Lampa.Player** - Native video player that supports HLS (m3u8), MP4, and other formats
4. **Lampa.Activity** - Manages screen states and transitions
5. **Lampa.Notifier** - Shows toast notifications
6. **Lampa.Explorer** - Creates file explorer-like interfaces

### Key Concepts

#### Controller System
```javascript
Lampa.Controller.add('content', {
    toggle: function() { /* Called when controller is activated */ },
    up: function() { /* Handle up navigation */ },
    down: function() { /* Handle down navigation */ },
    left: function() { /* Handle left navigation */ },
    right: function() { /* Handle right navigation */ },
    ok: function() { /* Handle OK/Enter button */ },
    back: function() { /* Handle back button */ }
});

// Activate controller
Lampa.Controller.toggle('content');
```

#### Scroll Navigation
The Lampa.Scroll class does NOT have a `scrollToIndex` method. Instead, you must:
1. Calculate the position of the target element
2. Use `scroll.wheel(delta)` to scroll programmatically
3. Or use touch events on mobile devices

Example:
```javascript
function scrollToItem(index, items, scroll) {
    var item = items[index][0]; // Get DOM element
    var itemTop = item.offsetTop;
    var currentScroll = -scroll.position();
    var targetScroll = itemTop - 200; // Offset for visibility
    
    scroll.wheel(targetScroll - currentScroll);
}
```

#### Player API
```javascript
// Play direct stream (m3u8/mp4)
Lampa.Player.play({
    url: 'https://example.com/stream.m3u8',
    title: 'Movie Title',
    quality: 'auto' // For HLS streams
});

// Play via iframe (not recommended for TV devices)
Lampa.Player.play({
    url: 'https://example.com/embed/player',
    title: 'Movie Title',
    iframe: true
});
```

## Stream Extraction Techniques

### Why Extract Streams?

Iframes are problematic in Lampa because:
1. **TV Compatibility**: Many TV browsers don't support iframes well
2. **Navigation Issues**: Remote controls can't interact with iframe content
3. **Performance**: Iframes add overhead and may be blocked by CORS
4. **Quality Control**: Can't select quality or enable subtitles properly

### Extraction Methods

#### Method 1: Fetch and Parse HTML
```javascript
async function extractStreamFromIframe(iframeUrl) {
    const response = await fetch(iframeUrl);
    const html = await response.text();
    
    // Pattern 1: Direct m3u8 URLs
    const m3u8Pattern = /https?:\/\/[^"'\s]+\.m3u8[^"'\s]*/gi;
    const matches = html.match(m3u8Pattern);
    
    if (matches && matches.length > 0) {
        return { type: 'hls', url: matches[0] };
    }
    
    // Pattern 2: Video element src
    const videoSrcPattern = /<video[^>]*src=["']([^"']+)["']/i;
    const videoMatch = html.match(videoSrcPattern);
    
    if (videoMatch && videoMatch[1]) {
        return { type: 'hls', url: videoMatch[1] };
    }
    
    // Pattern 3: Player configuration (JSON in script)
    const configPattern = /file["']\s*:\s*["']([^"']+)["']/i;
    const configMatch = html.match(configPattern);
    
    if (configMatch && configMatch[1]) {
        return { type: 'hls', url: configMatch[1] };
    }
    
    return null;
}
```

#### Method 2: API Endpoints
Some embed providers offer direct APIs:
```javascript
async function getDirectStream(tmdbId, isSeries, season, episode) {
    let apiUrl = isSeries 
        ? `https://provider.com/api/tv/${tmdbId}/${season}/${episode}`
        : `https://provider.com/api/movie/${tmdbId}`;
    
    const response = await fetch(apiUrl);
    const data = await response.json();
    
    if (data.url && (data.url.includes('.m3u8') || data.url.includes('.mp4'))) {
        return { type: data.url.includes('.m3u8') ? 'hls' : 'mp4', url: data.url };
    }
    
    return null;
}
```

#### Method 3: Redirect Following
Some providers redirect to the actual stream:
```javascript
async function followRedirect(url) {
    const response = await fetch(url, { redirect: 'follow' });
    const finalUrl = response.url;
    
    if (finalUrl.includes('.m3u8') || finalUrl.includes('.mp4')) {
        return { 
            type: finalUrl.includes('.m3u8') ? 'hls' : 'mp4', 
            url: finalUrl 
        };
    }
    
    return null;
}
```

## Common Stream Patterns

### M3U8 (HLS) Streams
- Extension: `.m3u8`
- MIME type: `application/vnd.apple.mpegurl`
- Lampa handles these natively with quality selection

### MP4 Streams
- Extension: `.mp4`
- MIME type: `video/mp4`
- Direct playback without transcoding

### TS Segments
- Extension: `.ts`
- Usually referenced inside m3u8 playlists
- Don't play individual TS files directly

### Regex Patterns for Extraction

```javascript
// M3U8 URLs
const m3u8Pattern = /https?:\/\/[^"'\s]+\.m3u8[^"'\s]*/gi;

// MP4 URLs
const mp4Pattern = /https?:\/\/[^"'\s]+\.mp4[^"'\s]*/gi;

// TS segment URLs
const tsPattern = /https?:\/\/[^"'\s]+\.ts[^"'\s]*/gi;

// Master playlist (often contains "master" in URL)
const masterPattern = /https?:\/\/[^"'\s]*master[^"'\s]*\.m3u8[^"'\s]*/gi;

// Video source element
const sourcePattern = /<source[^>]*src=["']([^"']+)["'][^>]*>/gi;

// Player config patterns
const configPatterns = [
    /file["']\s*:\s*["']([^"']+)[\"']/i,
    /url["']\s*:\s*["']([^"']+)[\"']/i,
    /src["']\s*:\s*["']([^"']+)[\"']/i
];
```

## Best Practices

### 1. Always Provide Fallback
If stream extraction fails, fall back to iframe:
```javascript
if (streamInfo && streamInfo.url) {
    Lampa.Player.play({
        url: streamInfo.url,
        title: movie.title
    });
} else {
    Lampa.Player.play({
        url: iframeUrl,
        iframe: true
    });
}
```

### 2. Handle CORS
Many embed servers block cross-origin requests:
```javascript
try {
    const response = await fetch(url, {
        method: 'GET',
        mode: 'cors',
        headers: {
            'Accept': 'text/html'
        }
    });
} catch (error) {
    console.error('CORS error:', error);
    return null;
}
```

### 3. Add User Feedback
Use Notifier to inform users:
```javascript
Lampa.Notifier.show({
    title: 'Stream Found',
    text: 'Playing HLS stream directly',
    time: 2000
});
```

### 4. Cache Results
Avoid repeated extraction:
```javascript
const streamCache = {};

async function getStream(source) {
    if (streamCache[source.url]) {
        return streamCache[source.url];
    }
    
    const stream = await extractStreamFromIframe(source.url);
    streamCache[source.url] = stream;
    return stream;
}
```

### 5. Respect Rate Limits
Add delays between requests:
```javascript
async function testSources(sources) {
    for (let i = 0; i < sources.length; i++) {
        const result = await testSource(sources[i]);
        if (result.working) break;
        
        if (i < sources.length - 1) {
            await new Promise(r => setTimeout(r, 500)); // 500ms delay
        }
    }
}
```

## Troubleshooting

### Common Issues

1. **"scrollToIndex is not a function"**
   - Solution: Implement custom scroll function using `scroll.wheel()`

2. **CORS errors when fetching iframe**
   - Solution: Use no-cors mode or find alternative source

3. **No stream found in HTML**
   - Solution: The stream may be loaded dynamically via JavaScript
   - Try checking network requests or use provider's API

4. **Iframe works but direct stream doesn't**
   - Solution: The stream may require specific headers/referer
   - Fall back to iframe for this source

### Debug Tips

```javascript
// Log extracted streams
console.log('[Plugin] Found stream:', streamInfo);

// Check HTML content
console.log('[Plugin] HTML length:', html.length);
console.log('[Plugin] Contains m3u8:', html.includes('.m3u8'));

// Test URL accessibility
try {
    const test = await fetch(streamUrl, { method: 'HEAD' });
    console.log('[Plugin] Stream accessible:', test.ok);
} catch (e) {
    console.error('[Plugin] Stream error:', e.message);
}
```

## Example Plugin Structure

```javascript
(function() {
    'use strict';
    
    // 1. Configuration and sources
    function getSources(movie) { ... }
    
    // 2. Stream extraction functions
    async function extractStreamFromIframe(url) { ... }
    async function getDirectStreamFromAPI(id, params) { ... }
    
    // 3. UI Component
    function MyComponent(object) {
        var scroll = new Lampa.Scroll({ mask: true, over: true });
        var items = [];
        var selectedIndex = 0;
        
        // Custom scroll function
        function scrollToItem(index) { ... }
        
        this.start = function() {
            Lampa.Controller.add('content', {
                up: function() { 
                    if (selectedIndex > 0) {
                        selectedIndex--;
                        scrollToItem(selectedIndex);
                    }
                },
                down: function() {
                    if (selectedIndex < items.length - 1) {
                        selectedIndex++;
                        scrollToItem(selectedIndex);
                    }
                },
                ok: function() {
                    playSource(items[selectedIndex]);
                }
            });
        };
        
        async function playSource(source) {
            // Extract stream
            const stream = await extractStreamFromIframe(source.url);
            
            // Play direct or fallback to iframe
            if (stream && stream.url) {
                Lampa.Player.play({
                    url: stream.url,
                    title: object.movie.title
                });
            } else {
                Lampa.Player.play({
                    url: source.url,
                    iframe: true
                });
            }
        }
    }
    
    // 4. Register component
    Lampa.Component.add('my_component', MyComponent);
    
    // 5. Inject UI
    Lampa.Listener.follow('full', function(e) {
        // Add button to movie page
    });
    
})();
```

## Resources

- Lampa Source Code: https://github.com/yumata/lampa-source
- Lampa Documentation: Run `npm run doc` in lampa-source
- HLS Specification: https://tools.ietf.org/html/rfc8216
- MDN Fetch API: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API

## Legal Disclaimer

This plugin is provided for educational purposes only. Ensure compliance with:
- Your local laws and regulations
- Terms of service of streaming providers
- Copyright and licensing requirements

Do not use this plugin to access content you don't have rights to view.
