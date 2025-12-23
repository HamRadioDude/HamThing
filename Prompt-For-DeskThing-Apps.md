# Claude Prompt for DeskThing App Development

Copy everything below the line and paste it into a new Claude chat:

---

I need help building a DeskThing app for the Spotify Car Thing. I have a working template and know the exact patterns required. Please follow these rules EXACTLY or the app will crash.

## Architecture

- **Server** (runs on PC): Fetches data from APIs, sends to client via `DeskThing.send()`
- **Client** (runs on Car Thing): Displays UI, receives data via `DeskThing.on()`
- The Car Thing CANNOT access external APIs - all data must flow through the server

## CRITICAL RULES (App WILL crash without these):

### 1. Use DESKTHING_EVENTS Enum - NOT Strings!
```typescript
// ❌ WRONG - WILL CRASH
DeskThing.on('start', start)

// ✅ CORRECT
import { DESKTHING_EVENTS } from '@deskthing/types'
DeskThing.on(DESKTHING_EVENTS.START, start)
```

### 2. Call initializeSettings() First in Start Handler
```typescript
const start = async () => {
  await initializeSettings()  // MUST be first!
  // then do other stuff
}
```

### 3. Use Async Handlers
```typescript
const start = async () => { }
const stop = async () => { }
```

### 4. Only START and STOP Listeners at Module Level
```typescript
// ❌ WRONG - extra listeners cause crashes
DeskThing.on('get', handler)  // DON'T DO THIS
DeskThing.on(DESKTHING_EVENTS.START, start)

// ✅ CORRECT - only these two
DeskThing.on(DESKTHING_EVENTS.START, start)
DeskThing.on(DESKTHING_EVENTS.STOP, stop)
```

### 5. Wrap Async API Calls in Sync Functions for Intervals
```typescript
// ❌ WRONG
setInterval(async () => { await fetch(url) }, 30000)

// ✅ CORRECT
const doFetch = async () => { await fetch(url) }
const fetchData = () => { doFetch() }  // sync wrapper, no await!
setInterval(fetchData, 30000)
```

### 6. Icon Must Match App ID
If manifest has `"id": "my-app"`, icon must be at `public/icons/my-app.svg`

## Working Server Template

```typescript
import { DeskThing } from '@deskthing/server'
import { DESKTHING_EVENTS, SETTING_TYPES } from '@deskthing/types'

const API_URL = 'https://your-api.com/endpoint'
const REFRESH_INTERVAL = 30000

let data: any[] = []
let refreshInterval: ReturnType<typeof setInterval> | null = null

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
      description: 'How often to fetch data'
    }
  }
  await DeskThing.initSettings(settings)
}

const doFetch = async () => {
  try {
    console.log('[MyApp] Fetching...')
    const response = await fetch(API_URL)
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    data = await response.json()
    console.log(`[MyApp] Got ${data.length} items`)
    DeskThing.send({ type: 'data', payload: data })
  } catch (error) {
    console.error('[MyApp] Fetch error:', error)
  }
}

const fetchData = () => { doFetch() }

const start = async () => {
  console.log('[MyApp] Started')
  await initializeSettings()
  fetchData()
  refreshInterval = setInterval(fetchData, REFRESH_INTERVAL)
}

const stop = async () => {
  console.log('[MyApp] Stopped')
  if (refreshInterval) {
    clearInterval(refreshInterval)
    refreshInterval = null
  }
}

DeskThing.on(DESKTHING_EVENTS.START, start)
DeskThing.on(DESKTHING_EVENTS.STOP, stop)
```

## Working Client Template

```typescript
import { useEffect, useState } from 'react'
import { DeskThing } from '@deskthing/client'

const deskthing = DeskThing

function App() {
  const [data, setData] = useState<any[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    const offData = deskthing.on('data', (event: any) => {
      if (event?.payload && Array.isArray(event.payload)) {
        setData(event.payload)
        setLoading(false)
      }
    })

    setTimeout(() => {
      deskthing.send({ type: 'get', request: 'data' })
    }, 500)

    return () => { offData() }
  }, [])

  if (loading) return <div>Loading...</div>

  return (
    <div className="h-screen w-screen bg-gray-900 text-white">
      {data.map((item, i) => (
        <div key={i}>{/* render item */}</div>
      ))}
    </div>
  )
}

export default App
```

## Manifest Template (deskthing/manifest.json)

```json
{
  "label": "My App Name",
  "id": "my-app-id",
  "version": "1.0.0",
  "description": "What it does",
  "author": "Your Name",
  "requiredVersions": {
    "server": ">=0.11.0",
    "client": ">=0.11.2"
  },
  "platforms": ["windows", "linux", "mac"],
  "template": "full",
  "tags": [],
  "requires": [],
  "repository": "",
  "homepage": "",
  "updateUrl": ""
}
```

## Project Structure

```
my-app/
├── deskthing/manifest.json
├── public/icons/my-app-id.svg
├── server/index.ts
├── src/App.tsx
├── src/main.tsx
├── src/index.css
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.js
└── postcss.config.js
```

## Build Process

1. `npm install`
2. `npm run build`
3. Add `{"type": "module"}` to root `package.json` in the ZIP
4. Add `{"type": "module"}` to `server/package.json` in the ZIP
5. Install via DeskThing UI

## What I Want to Build

[DESCRIBE YOUR APP HERE - what API, what data to display, what the UI should look like]

---

Please help me build this app following the exact patterns above. Generate the complete server/index.ts, src/App.tsx, and deskthing/manifest.json files. Do NOT deviate from the patterns - they are the result of extensive debugging and the app WILL crash otherwise.
