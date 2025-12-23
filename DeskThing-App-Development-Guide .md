# DeskThing App Development Guide
## Building Apps for the Spotify Car Thing

**Version:** 1.0  
**Date:** December 2025  
**Based on:** DeskThing Server v0.11.17, SDK v0.11.6

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Project Structure](#project-structure)
4. [Critical Requirements](#critical-requirements)
5. [Server Code Patterns](#server-code-patterns)
6. [Client Code Patterns](#client-code-patterns)
7. [Manifest Configuration](#manifest-configuration)
8. [Build Process](#build-process)
9. [Common Pitfalls & Solutions](#common-pitfalls--solutions)
10. [Complete Working Example](#complete-working-example)
11. [Resources](#resources)

---

## Overview

DeskThing is a platform that repurposes the Spotify Car Thing as a customizable desk display. Apps consist of two parts:

- **Server**: Runs on your computer, handles API calls, data processing
- **Client**: Runs on the Car Thing, displays UI

The Car Thing is essentially a "dumb display" - it cannot access external APIs or the internet directly. All data must flow through the server.

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│   External API  │ ──────► │  Your Server    │ ──────► │   Car Thing     │
│   (Internet)    │         │  (Your PC)      │         │   (Display)     │
└─────────────────┘         └─────────────────┘         └─────────────────┘
                            DeskThing.send()            DeskThing.on()
```

---

## Architecture

### Data Flow

1. **Server fetches data** from external APIs (runs on your PC)
2. **Server sends data** to client via `DeskThing.send()`
3. **Client receives data** via `DeskThing.on('eventType', callback)`
4. **Client renders UI** using React

### Key Constraints

- ❌ Client CANNOT access external APIs
- ❌ Client CANNOT import external CSS/JS libraries from CDN
- ❌ Client CANNOT access the filesystem
- ✅ Client CAN receive data from server via DeskThing
- ✅ Client CAN send requests to server via DeskThing

---

## Project Structure

```
my-app/
├── deskthing/
│   └── manifest.json          # App metadata (REQUIRED)
├── public/
│   └── icons/
│       └── my-app.svg         # App icon (REQUIRED - must match app ID)
├── server/
│   └── index.ts               # Server-side code
├── src/
│   ├── App.tsx                # Main React component
│   └── main.tsx               # React entry point
├── index.html                 # HTML template
├── package.json               # Dependencies
├── tsconfig.json              # TypeScript config
└── vite.config.ts             # Vite build config
```

---

## Critical Requirements

### ⚠️ THESE ARE MANDATORY - Your app WILL crash without them:

### 1. Use DESKTHING_EVENTS Enum (Not Strings!)

```typescript
// ❌ WRONG - Will cause crashes
DeskThing.on('start', start)
DeskThing.on('stop', stop)

// ✅ CORRECT - Use the enum from @deskthing/types
import { DESKTHING_EVENTS } from '@deskthing/types'
DeskThing.on(DESKTHING_EVENTS.START, start)
DeskThing.on(DESKTHING_EVENTS.STOP, stop)
```

### 2. Call initSettings() in Start Handler

```typescript
// ❌ WRONG - Missing initSettings
const start = async () => {
  console.log('Starting')
  doStuff()
}

// ✅ CORRECT - Always call initSettings first
const start = async () => {
  console.log('Starting')
  await initializeSettings()  // REQUIRED!
  doStuff()
}
```

### 3. Use Async Handlers

```typescript
// ❌ WRONG - Sync handlers may cause issues
const start = () => { }
const stop = () => { }

// ✅ CORRECT - Use async handlers
const start = async () => { }
const stop = async () => { }
```

### 4. Only Register START and STOP Listeners at Module Level

```typescript
// ❌ WRONG - Extra listeners at module level cause crashes
DeskThing.on('get', handleGet)      // DON'T DO THIS
DeskThing.on('someEvent', handler)  // DON'T DO THIS
DeskThing.on(DESKTHING_EVENTS.START, start)
DeskThing.on(DESKTHING_EVENTS.STOP, stop)

// ✅ CORRECT - Only START and STOP at module level
DeskThing.on(DESKTHING_EVENTS.START, start)
DeskThing.on(DESKTHING_EVENTS.STOP, stop)
```

### 5. Wrap Async API Calls in Sync Functions for Intervals

```typescript
// ❌ WRONG - Async function directly in interval
setInterval(async () => {
  await fetch(API_URL)
}, 30000)

// ✅ CORRECT - Sync wrapper that fires async work
const doFetch = async () => {
  const response = await fetch(API_URL)
  // ... process
}

const fetchData = () => {
  doFetch()  // Fire and forget - don't await!
}

setInterval(fetchData, 30000)
```

### 6. Include package.json with type: module

Both root and server directories need `package.json` with ESM type:

```json
{
  "type": "module"
}
```

### 7. Icon Must Match App ID

If your manifest has `"id": "my-app"`, your icon must be at:
`public/icons/my-app.svg`

---

## Server Code Patterns

### Minimal Working Server Template

```typescript
import { DeskThing } from '@deskthing/server'
import { DESKTHING_EVENTS, SETTING_TYPES } from '@deskthing/types'

// ============================================
// CONFIGURATION
// ============================================
const API_URL = 'https://api.example.com/data'
const REFRESH_INTERVAL = 30000  // 30 seconds

// ============================================
// STATE
// ============================================
let data: any[] = []
let refreshInterval: ReturnType<typeof setInterval> | null = null

// ============================================
// SETTINGS (REQUIRED!)
// ============================================
const initializeSettings = async () => {
  const settings = {
    'refresh_rate': {
      id: 'refresh_rate',
      type: SETTING_TYPES.NUMBER,
      value: REFRESH_INTERVAL,
      label: 'Refresh Rate (ms)',
      min: 10000,
      max: 300000,
      step: 5000,
      description: 'How often to fetch new data'
    }
  }
  await DeskThing.initSettings(settings)
}

// ============================================
// DATA FETCHING
// ============================================
const doFetch = async () => {
  try {
    console.log('[MyApp] Fetching data...')
    const response = await fetch(API_URL)
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    data = await response.json()
    console.log(`[MyApp] Got ${data.length} items`)
    
    // Send to client
    DeskThing.send({ type: 'data', payload: data })
  } catch (error) {
    console.error('[MyApp] Fetch error:', error)
  }
}

// Sync wrapper for interval (IMPORTANT!)
const fetchData = () => {
  doFetch()  // Don't await!
}

// ============================================
// LIFECYCLE HANDLERS
// ============================================
const start = async () => {
  console.log('[MyApp] Started the server')
  
  // REQUIRED: Initialize settings first
  await initializeSettings()
  
  // Initial fetch
  fetchData()
  
  // Set up refresh interval
  refreshInterval = setInterval(fetchData, REFRESH_INTERVAL)
}

const stop = async () => {
  console.log('[MyApp] Stopped the server')
  if (refreshInterval) {
    clearInterval(refreshInterval)
    refreshInterval = null
  }
}

// ============================================
// EVENT REGISTRATION (Only START and STOP!)
// ============================================
DeskThing.on(DESKTHING_EVENTS.START, start)
DeskThing.on(DESKTHING_EVENTS.STOP, stop)
```

### Sending Data to Client

```typescript
// Send typed data
DeskThing.send({ 
  type: 'myDataType',    // Client listens for this
  payload: myData        // The actual data
})

// Examples:
DeskThing.send({ type: 'spots', payload: spotArray })
DeskThing.send({ type: 'weather', payload: weatherData })
DeskThing.send({ type: 'time', payload: { utc: '...', local: '...' } })
```

---

## Client Code Patterns

### Minimal Working Client Template

```typescript
import { useEffect, useState } from 'react'
import { DeskThing } from '@deskthing/client'

// Get the DeskThing instance
const deskthing = DeskThing

interface MyData {
  // Define your data structure
  id: string
  name: string
  value: number
}

function App() {
  const [data, setData] = useState<MyData[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // Listen for data from server
    const offData = deskthing.on('data', (event: any) => {
      console.log('[Client] Received data:', event)
      if (event?.payload && Array.isArray(event.payload)) {
        setData(event.payload)
        setLoading(false)
      }
    })

    // Request initial data after short delay
    setTimeout(() => {
      deskthing.send({ type: 'get', request: 'data' })
    }, 500)

    // Cleanup
    return () => {
      offData()
    }
  }, [])

  if (loading) {
    return (
      <div className="h-screen w-screen bg-gray-900 text-white flex items-center justify-center">
        <div className="text-2xl">Loading...</div>
      </div>
    )
  }

  return (
    <div className="h-screen w-screen bg-gray-900 text-white">
      {/* Your UI here */}
      {data.map((item, index) => (
        <div key={index}>{item.name}: {item.value}</div>
      ))}
    </div>
  )
}

export default App
```

### Listening for Events

```typescript
// Listen for specific data type
deskthing.on('spots', (data) => {
  setSpots(data.payload)
})

// Listen for scroll/knob events
deskthing.on('scroll', (data) => {
  const direction = data?.payload?.direction
  if (direction === 'up') { /* scroll up */ }
  if (direction === 'down') { /* scroll down */ }
})

// Listen for button presses
deskthing.on('button', (data) => {
  console.log('Button pressed:', data)
})
```

### Sending Requests to Server

```typescript
// Request data refresh
deskthing.send({ type: 'get', request: 'data' })

// Send user action
deskthing.send({ type: 'action', payload: { action: 'refresh' } })
```

---

## Manifest Configuration

### deskthing/manifest.json

```json
{
  "label": "My App Name",
  "tags": [],
  "requires": [],
  "version": "1.0.0",
  "requiredVersions": {
    "server": ">=0.11.0",
    "client": ">=0.11.2"
  },
  "description": "What my app does",
  "author": "Your Name",
  "platforms": ["windows", "linux", "mac"],
  "repository": "",
  "homepage": "",
  "updateUrl": "",
  "id": "my-app-id",
  "template": "full"
}
```

### Important Manifest Fields

| Field | Description | Example |
|-------|-------------|---------|
| `id` | Unique app identifier (lowercase, hyphens) | `"pota-thing"` |
| `label` | Display name | `"POTA Thing"` |
| `requiredVersions.server` | Minimum DeskThing server version | `">=0.11.0"` |
| `template` | App type | `"full"` |

---

## Build Process

### package.json

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "npx @deskthing/cli package",
    "preview": "vite preview"
  },
  "dependencies": {
    "@deskthing/client": "^0.11.2",
    "@deskthing/server": "^0.11.6",
    "@deskthing/types": "^0.11.6",
    "react": "^19.1.0",
    "react-dom": "^19.1.0"
  },
  "devDependencies": {
    "@deskthing/cli": "^0.11.15",
    "@types/react": "^19.1.4",
    "@types/react-dom": "^19.1.5",
    "@vitejs/plugin-react": "^4.5.0",
    "@vitejs/plugin-legacy": "^5.4.3",
    "terser": "^5.x",
    "esbuild": "^0.x",
    "typescript": "^5.8.3",
    "vite": "^6.4.1"
  }
}
```

### vite.config.ts

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import legacy from '@vitejs/plugin-legacy'

export default defineConfig({
  plugins: [
    react(),
    legacy({
      targets: ['chrome >= 69'],
      additionalLegacyPolyfills: ['regenerator-runtime/runtime']
    })
  ],
  base: './'
})
```

### Build Commands

```bash
# Install dependencies
npm install

# Build for production
npm run build

# Output will be in dist/ folder
# - dist/my-app-v1.0.0.zip  <- This is your app package
```

### Post-Build: Add ESM Package Files

After building, before installing, add these files to the ZIP:

**Root package.json:**
```json
{"type": "module"}
```

**server/package.json:**
```json
{"type": "module"}
```

---

## Common Pitfalls & Solutions

### Problem: "Cannot read properties of undefined (reading 'start')"

**Cause:** Using string literals instead of DESKTHING_EVENTS enum, or having extra listeners at module level.

**Solution:**
```typescript
// Use enum
import { DESKTHING_EVENTS } from '@deskthing/types'
DeskThing.on(DESKTHING_EVENTS.START, start)

// Only START and STOP at module level
```

### Problem: "DeskThing.sendLog is not a function"

**Cause:** Wiki documentation shows features not in published SDK.

**Solution:** Use `console.log()` instead of `DeskThing.sendLog()`

### Problem: Server crashes immediately after "Started"

**Cause:** Missing `initializeSettings()` call.

**Solution:**
```typescript
const start = async () => {
  await initializeSettings()  // Add this!
  // ... rest of code
}
```

### Problem: App crashes when making API calls

**Cause:** Async functions directly in setInterval, or awaiting in sync context.

**Solution:**
```typescript
const doFetch = async () => { /* async work */ }
const fetchData = () => { doFetch() }  // Sync wrapper
setInterval(fetchData, 30000)
```

### Problem: Black screen on Car Thing

**Cause:** Client not receiving data, or React not rendering.

**Solution:** 
- Check server logs for "Sending data" messages
- Add console.log in client's `on()` handler
- Ensure `setLoading(false)` is called after data received

### Problem: "ENOENT: no such file or directory... icons/my-app.svg"

**Cause:** Missing icon file or wrong filename.

**Solution:** Create `public/icons/{app-id}.svg` where `{app-id}` matches manifest ID.

### Problem: Module errors / ESM issues

**Cause:** Missing `"type": "module"` in package.json files.

**Solution:** Add `{"type": "module"}` to both root and server/ package.json.

---

## Complete Working Example

Here's the complete POTA (Parks on the Air) app that fetches ham radio spots:

### Server (server/index.ts)

```typescript
import { DeskThing } from '@deskthing/server'
import { DESKTHING_EVENTS, SETTING_TYPES } from '@deskthing/types'

const POTA_API = 'https://api.pota.app/spot/activator'

let spotData: any[] = []
let refreshInterval: ReturnType<typeof setInterval> | null = null

const initializeSettings = async () => {
  const settings = {
    'refresh_rate': {
      id: 'refresh_rate',
      type: SETTING_TYPES.NUMBER,
      value: 30000,
      label: 'Refresh Rate (ms)',
      min: 10000,
      max: 300000,
      step: 5000,
      description: 'How often to fetch new POTA spots'
    }
  }
  await DeskThing.initSettings(settings)
}

const doFetch = async () => {
  try {
    console.log('[POTA] Fetching spots...')
    const response = await fetch(POTA_API)
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    spotData = await response.json()
    console.log(`[POTA] Got ${spotData.length} spots`)
    DeskThing.send({ type: 'spots', payload: spotData })
  } catch (error) {
    console.error('[POTA] Fetch error:', error)
  }
}

const fetchSpots = () => {
  doFetch()
}

const start = async () => {
  console.log('[POTA] Started the server')
  await initializeSettings()
  fetchSpots()
  refreshInterval = setInterval(fetchSpots, 30000)
}

const stop = async () => {
  console.log('[POTA] Stopped the server')
  if (refreshInterval) {
    clearInterval(refreshInterval)
    refreshInterval = null
  }
}

DeskThing.on(DESKTHING_EVENTS.START, start)
DeskThing.on(DESKTHING_EVENTS.STOP, stop)
```

### Client (src/App.tsx)

```typescript
import { useEffect, useState } from 'react'
import { DeskThing } from '@deskthing/client'

const deskthing = DeskThing

interface POTASpot {
  activator: string
  frequency: string
  mode: string
  reference: string
  parkName: string
  locationDesc: string
  spotTime: string
}

function App() {
  const [spots, setSpots] = useState<POTASpot[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    const offSpots = deskthing.on('spots', (data: any) => {
      if (data?.payload && Array.isArray(data.payload)) {
        setSpots(data.payload)
        setLoading(false)
      }
    })

    setTimeout(() => {
      deskthing.send({ type: 'get', request: 'spots' })
    }, 500)

    return () => { offSpots() }
  }, [])

  if (loading) {
    return (
      <div className="h-screen w-screen bg-gray-900 text-white flex items-center justify-center">
        Loading POTA spots...
      </div>
    )
  }

  return (
    <div className="h-screen w-screen bg-gray-900 text-white flex flex-col">
      <div className="p-4 bg-gray-800 text-2xl font-bold">
        📻 POTA Spots ({spots.length})
      </div>
      <div className="flex-1 overflow-y-auto">
        {spots.map((spot, i) => (
          <div key={i} className="p-4 border-b border-gray-700">
            <div className="text-xl font-bold">{spot.activator}</div>
            <div className="text-gray-400">
              {spot.frequency} - {spot.reference}
            </div>
          </div>
        ))}
      </div>
    </div>
  )
}

export default App
```

---

## Resources

### Official Links
- **DeskThing Website:** https://deskthing.app
- **DeskThing Discord:** https://deskthing.app/discord
- **GitHub - DeskThing:** https://github.com/ItsRiprod/DeskThing
- **GitHub - Apps:** https://github.com/ItsRiprod/Deskthing-Apps
- **GitHub - Server SDK:** https://github.com/ItsRiprod/deskthing-app-server
- **GitHub - Client SDK:** https://github.com/ItsRiprod/deskthing-app-client

### NPM Packages
- `@deskthing/server` - Server-side SDK
- `@deskthing/client` - Client-side SDK  
- `@deskthing/types` - TypeScript types
- `@deskthing/cli` - Build tools

### Reference Apps
- **UltimateClock** - Good reference for settings and patterns
- **Weather** - Good reference for API fetching

---

## Quick Start Checklist

- [ ] Create project structure with `deskthing/manifest.json`
- [ ] Set manifest `id` field (lowercase, hyphens)
- [ ] Create icon at `public/icons/{id}.svg`
- [ ] Import `DESKTHING_EVENTS` from `@deskthing/types`
- [ ] Use `DeskThing.on(DESKTHING_EVENTS.START, start)` (not string!)
- [ ] Make `start` and `stop` handlers `async`
- [ ] Call `await initializeSettings()` first in `start`
- [ ] Wrap async API calls in sync functions for intervals
- [ ] Only register START and STOP listeners at module level
- [ ] Build with `npm run build`
- [ ] Add `{"type": "module"}` to root and server/ package.json in ZIP
- [ ] Install via DeskThing UI

---

*This guide was created after extensive debugging of the DeskThing SDK v0.11.6 with DeskThing Server v0.11.17. Some SDK behaviors may change in future versions.*
