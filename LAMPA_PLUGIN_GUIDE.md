# Lampa Plugin Development Guide

## Overview
This guide explains how to develop plugins for Lampa, a popular media player application for smart TVs and set-top boxes. It includes detailed information about stream extraction, mass searching, and best practices.

## Table of Contents
1. [Lampa Architecture](#lampa-architecture)
2. [Stream Extraction Techniques](#stream-extraction-techniques)
3. [Mass Search Implementation](#mass-search-implementation)
4. [Common Regex Patterns](#common-regex-patterns)
5. [CORS Proxy Solutions](#cors-proxy-solutions)
6. [Player Integration](#player-integration)
7. [UI Components](#ui-components)
8. [Debugging Tips](#debugging-tips)

---

## Lampa Architecture

### Core Components

#### 1. Lampa.Component
Register custom components that can be loaded via Activity:
```javascript
Lampa.Component.add('my_component', MyComponentFunction);
```

#### 2. Lampa.Activity
Manages navigation stack and screens:
```javascript
Lampa.Activity.push({
    url: '',
    title: 'My Screen',
    component: 'my_component',
    movie: movieObject,
    page: 1
});
```

#### 3. Lampa.Controller
Handles remote control input (OK, Back, Up, Down, Left, Right):
```javascript
Lampa.Controller.add('content', {
    toggle: function() { /* ... */ },
    up: function() { /* ... */ },
    down: function() { /* ... */ },
    left: function() { /* ... */ },
    right: function() { /* ... */ },
    ok: function() { /* ... */ },
    back: function() { /* ... */ }
});
Lampa.Controller.toggle('content');
```

#### 4. Lampa.Scroll
Creates scrollable containers:
```javascript
var scroll = new Lampa.Scroll({ mask: true, over: true });
scroll.append(content);
```

#### 5. Lampa.Player
Native video player for direct streams:
```javascript
Lampa.Player.play({
    url: 'https://example.com/stream.m3u8',
    title: 'Movie Title',
    quality: 'auto'
});
```

#### 6. Lampa.Listener
Hooks into existing Lampa events:
```javascript
Lampa.Listener.follow('full', function(e) {
    if (e.type !== 'complite') return;
    // Modify the full screen
});
```

---

## Stream Extraction Techniques

### Why Extract Streams?
Many embed sources use iframes which are not supported on all TV platforms. Extracting direct .m3u8 (HLS) or .mp4 links allows native playback in Lampa.Player.

### Method 1: Fetch Iframe HTML and Parse
```javascript
async function extractStreamFromIframe(iframeUrl) {
    const html = await fetchWithProxy(iframeUrl);
    
    // Find m3u8 URLs
    const m3u8Pattern = /https?:\/\/[^"'\s]+\.m3u8[^"'\s]*/gi;
    const matches = html.match(m3u8Pattern);
    
    if (matches && matches.length > 0) {
        return { type: 'hls', url: matches[0] };
    }
    
    return null;
}
```

### Method 2: Use Provider APIs
Some providers like MultiEmbed have direct APIs:
```javascript
async function getDirectStreamFromMultiEmbed(tmdbId, isSeries, season, episode) {
    let apiUrl;
    if (isSeries) {
        apiUrl = `https://multiembed.mov/directstream.php?video_id=${tmdbId}&tmdb=1&s=${season}&e=${episode}`;
    } else {
        apiUrl = `https://multiembed.mov/?video_id=${tmdbId}&tmdb=1`;
    }
    
    const response = await fetch(apiUrl, { redirect: 'follow' });
    
    // Check for JSON response
    const data = await response.json();
    if (data.url && data.url.endsWith('.m3u8')) {
        return { type: 'hls', url: data.url };
    }
    
    // Check redirect URL
    if (response.url.endsWith('.m3u8')) {
        return { type: 'hls', url: response.url };
    }
    
    return null;
}
```

### Method 3: Video Element Parsing
Look for `<video>` tags with src attributes:
```javascript
const videoSrcPattern = /<video[^>]*src=["']([^"']+)["'][^>]*>/i;
const videoMatch = html.match(videoSrcPattern);
if (videoMatch && videoMatch[1]) {
    return { type: 'hls', url: videoMatch[1] };
}
```

### Method 4: Source Element Parsing
Look for `<source>` tags:
```javascript
const sourcePattern = /<source[^>]*src=["']([^"']+)[\"'][^>]*>/gi;
let sourceMatch;
while ((sourceMatch = sourcePattern.exec(html)) !== null) {
    const ext = sourceMatch[1].split('.').pop().toLowerCase();
    if (ext === 'm3u8' || ext === 'mp4') {
        return { type: ext === 'm3u8' ? 'hls' : 'mp4', url: sourceMatch[1] };
    }
}
```

### Method 5: Player Configuration JSON
Many players embed config in script tags:
```javascript
const configPatterns = [
    /file["']\s*:\s*["']([^"']+)["']/i,
    /url["']\s*:\s*["']([^"']+)["']/i,
    /src["']\s*:\s*["']([^"']+)["']/i
];

for (const pattern of configPatterns) {
    const configMatch = html.match(pattern);
    if (configMatch && configMatch[1]) {
        const streamUrl = configMatch[1].replace(/\\/g, '');
        const ext = streamUrl.split('.').pop().toLowerCase();
        if (ext === 'm3u8' || ext === 'mp4') {
            return { type: ext === 'm3u8' ? 'hls' : 'mp4', url: streamUrl };
        }
    }
}
```

---

## Mass Search Implementation

### What is Mass Search?
Mass search queries multiple embed sources simultaneously to find working streams quickly.

### Implementation Pattern
```javascript
async function massSearch(sources) {
    const results = [];
    const promises = sources.map(async (source) => {
        try {
            const streamInfo = await extractStreamFromIframe(source.url);
            if (streamInfo) {
                results.push({
                    name: source.name,
                    url: streamInfo.url,
                    type: streamInfo.type,
                    working: true
                });
            }
        } catch (e) {
            console.log('Source failed:', source.name);
        }
    });
    
    await Promise.allSettled(promises);
    return results.filter(r => r.working);
}
```

### Parallel Processing
Use `Promise.allSettled()` to wait for all sources even if some fail:
```javascript
const results = await Promise.allSettled(
    sources.map(source => testSource(source))
);
```

---

## Common Regex Patterns

### M3U8 URLs
```javascript
/https?:\/\/[^"'\s]+\.m3u8[^"'\s]*/gi
```

### MP4 URLs
```javascript
/https?:\/\/[^"'\s]+\.mp4[^"'\s]*/gi
```

### TS Segment Files
```javascript
/https?:\/\/[^"'\s]+\.ts[^"'\s]*/gi
```

### Video Tag with Src
```javascript
/<video[^>]*src=["']([^"']+)["'][^>]*>/i
```

### Source Elements
```javascript
/<source[^>]*src=["']([^"']+)[\"'][^>]*>/gi
```

### Player Config (file/url/src)
```javascript
/file["']\s*:\s*["']([^"']+)["']/i
/url["']\s*:\s*["']([^"']+)["']/i
/src["']\s*:\s*["']([^"']+)["']/i
```

### Master Playlist Detection
```javascript
/https?:\/\/[^"'\s]*master[^"'\s]*\.m3u8[^"'\s]*/gi
```

---

## CORS Proxy Solutions

### Why Proxies Are Needed
Browser security (CORS) prevents fetching cross-origin iframe content directly. Proxies bypass this.

### Recommended Proxies
```javascript
const CORS_PROXIES = [
    'https://api.allorigins.win/raw?url=',
    'https://corsproxy.io/?',
    'https://thingproxy.freeboard.io/fetch/'
];

async function fetchWithProxy(url) {
    for (const proxy of CORS_PROXIES) {
        try {
            const response = await fetch(proxy + encodeURIComponent(url), {
                method: 'GET',
                headers: {
                    'Accept': 'text/html,application/xhtml+xml',
                    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
                }
            });
            if (response.ok) {
                return await response.text();
            }
        } catch (e) {
            console.log('Proxy failed:', proxy);
            continue;
        }
    }
    throw new Error('All proxies failed');
}
```

---

## Player Integration

### Native HLS Playback
```javascript
Lampa.Player.play({
    url: 'https://example.com/stream.m3u8',
    title: 'Movie Title',
    quality: 'auto'  // Let Lampa choose quality
});
```

### MP4 Playback
```javascript
Lampa.Player.play({
    url: 'https://example.com/video.mp4',
    title: 'Movie Title'
});
```

### Iframe Fallback
When direct stream extraction fails:
```javascript
Lampa.Player.play({
    url: 'https://embed.source.com/embed/movie/123',
    title: 'Movie Title',
    iframe: true
});
```

---

## UI Components

### Scroll Component
```javascript
var scroll = new Lampa.Scroll({ mask: true, over: true });
scroll.append(content);
```

### Smooth Scrolling to Item
```javascript
function scrollToItem(index, items, scroll) {
    if (!items[index]) return;
    
    var itemElement = items[index][0];
    var scrollContainer = scroll.render()[0];
    
    var itemTop = itemElement.offsetTop;
    var itemHeight = itemElement.offsetHeight;
    var containerHeight = scrollContainer.clientHeight;
    var currentScroll = scrollContainer.scrollTop;
    
    var targetScroll = itemTop - (containerHeight / 2) + (itemHeight / 2);
    var maxScroll = scrollContainer.scrollHeight - containerHeight;
    targetScroll = Math.max(0, Math.min(targetScroll, maxScroll));
    
    var distance = targetScroll - currentScroll;
    var duration = 300;
    var startTime = null;
    
    function easeInOutQuad(t) {
        return t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
    }
    
    function animate(currentTime) {
        if (!startTime) startTime = currentTime;
        var elapsed = currentTime - startTime;
        var progress = Math.min(elapsed / duration, 1);
        var eased = easeInOutQuad(progress);
        
        scrollContainer.scrollTop = currentScroll + (distance * eased);
        
        if (progress < 1) {
            requestAnimationFrame(animate);
        }
    }
    
    requestAnimationFrame(animate);
}
```

### Focus Management
```javascript
function updateFocus(items, selectedIndex) {
    items.forEach((item, idx) => {
        if (idx === selectedIndex) {
            item.addClass('focus');
        } else {
            item.removeClass('focus');
        }
    });
}
```

---

## Debugging Tips

### Console Logging
```javascript
console.log('[Plugin] Message:', data);
```

### Notifier for User Feedback
```javascript
Lampa.Notifier.show({
    title: 'Testing Source',
    text: 'Checking availability...',
    time: 2000
});
```

### Error Handling
```javascript
try {
    // Your code
} catch (error) {
    console.error('[Plugin] Error:', error.message);
    Lampa.Notifier.show({
        title: 'Error',
        text: error.message,
        time: 3000
    });
}
```

### Testing Source Availability
```javascript
async function testSource(source) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 8000);
    
    try {
        const response = await fetch(source.url, {
            method: 'HEAD',
            mode: 'no-cors',
            signal: controller.signal
        });
        clearTimeout(timeout);
        return { working: true };
    } catch (error) {
        clearTimeout(timeout);
        return { working: false, error: error.message };
    }
}
```

---

## Best Practices

1. **Always try direct stream extraction first** - iframes don't work on all TV platforms
2. **Use multiple CORS proxies** - some may be blocked or rate-limited
3. **Implement fallback mechanisms** - if extraction fails, fall back to iframe
4. **Provide user feedback** - use Lampa.Notifier to show what's happening
5. **Handle errors gracefully** - don't crash the plugin on single source failures
6. **Prioritize sources** - try the most reliable sources first
7. **Cache results when possible** - reduce API calls and improve performance
8. **Respect rate limits** - add delays between requests if needed

---

## Example Complete Plugin Structure

```javascript
(function() {
    'use strict';
    if (window.MyPlugin) return;
    window.MyPlugin = true;

    // 1. Define sources
    function getSources(movie) {
        // Return array of source objects
    }

    // 2. Stream extraction
    async function extractStreamFromIframe(url) {
        // Fetch and parse iframe
    }

    // 3. Test sources
    async function testSource(source) {
        // Check if source is working
    }

    // 4. UI Component
    function MyComponent(object) {
        var scroll = new Lampa.Scroll({ mask: true, over: true });
        var files = new Lampa.Explorer(object);
        
        this.create = function() { /* ... */ };
        this.render = function() { /* ... */ };
        this.start = function() { /* ... */ };
        this.back = function() { /* ... */ };
        this.destroy = function() { /* ... */ };
    }

    // 5. Register component
    Lampa.Component.add('my_component', MyComponent);

    // 6. Add button to movie page
    Lampa.Listener.follow('full', function(e) {
        if (e.type !== 'complite') return;
        // Add button
    });

    console.log('[MyPlugin] Initialized');
})();
```

---

## Direct Stream Resolver Used by `ry`

Lampa players should be given playable media URLs instead of iframe embed pages. The `ry` plugin now follows this order when the user selects a source:

1. Try provider-specific direct APIs first, currently MultiEmbed's directstream endpoint for series and its TMDB movie endpoint.
2. If the configured source URL is already a media URL, accept it directly.
3. Fetch the embed page through a CORS proxy and scan the response for direct media links.
4. Play only extracted `.m3u8`, `.mp4`, or `.mpd` URLs with `Lampa.Player.play({ url })`.
5. If no direct stream is found, show a notification instead of falling back to iframe playback.

The core helper is `getDirectStreamUrl(source, movie)`. It delegates response parsing to `parseDirectStreamPayload(payload, baseUrl)`, which handles JSON payloads, player config keys such as `file`, `url`, `src`, `link`, `hls`, and `playlist`, HTML `<source>`/`<video>` tags, escaped URLs, HTML entities, and relative links. This keeps iframe pages out of Lampa's player while still allowing providers that expose HLS, MP4, or DASH links to work natively.
