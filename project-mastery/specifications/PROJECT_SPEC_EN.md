# Excalidraw Project Specification

**Version:** 1.0.0
**Date:** 2025-11-17
**Project:** Excalidraw v0.18.0

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Technical Architecture](#technical-architecture)
3. [Core Features](#core-features)
4. [Data Models](#data-models)
5. [API Reference](#api-reference)
6. [Deployment Guide](#deployment-guide)
7. [Development Guide](#development-guide)

---

## Project Overview

### Introduction

Excalidraw is an **open-source virtual whiteboard** for sketching hand-drawn diagrams. It consists of two parts:

1. **@excalidraw/excalidraw** - Reusable React component library (published to npm)
2. **excalidraw.com** - Full-featured web application

**Key Features:**
- ✏️ Hand-drawn visual style
- 🎨 Simple, intuitive interface
- 🔄 Real-time collaboration (with end-to-end encryption)
- 📱 PWA support (offline-capable)
- 🌍 60+ languages supported
- 🤖 AI features (text-to-diagram)

### Use Cases

- 🎯 **Flowcharts** - Business processes, system architecture
- 📊 **Diagrams** - Mind maps, org charts
- 🎨 **Quick Sketches** - Design mockups, wireframes
- 📝 **Technical Documentation** - System design, API docs
- 👥 **Team Collaboration** - Real-time co-drawing

---

## Technical Architecture

### System Architecture

```
              User Interface (React Components)
                          │
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
   Toolbar/Menus    Canvas Rendering   Properties Panel
         │                │                │
         └────────────────┼────────────────┘
                          │
                    Core Logic Layer
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
  Element Management  Action System    State Management
    (Scene class)    (ActionManager)      (Jotai)
         │                │                │
         └────────────────┼────────────────┘
                          │
                 Data Persistence Layer
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
   localStorage      IndexedDB          Firebase
  (scene data)       (images)        (collaboration)
```

### Technology Stack

**Frontend:**
- React 19.0.0
- TypeScript 4.9.4
- Jotai 2.11.0 (state management)

**Drawing:**
- RoughJS 4.6.4 (hand-drawn rendering)
- Perfect Freehand 1.2.0 (smooth drawing)

**Collaboration:**
- Socket.IO Client 4.7.2 (WebSocket)
- Firebase 11.3.1 (cloud storage)

**Build Tools:**
- Vite 5.0.12 (app bundler)
- esbuild 0.19.10 (package bundler)
- Vitest 3.0.6 (testing)

### Triple-Canvas Rendering

**Three-layer canvas system** (core performance optimization):

```
┌────────────────────────────────────┐
│  Interactive Canvas                 │  ← Selection boxes, handles, cursors
│  - Continuous rendering (RAF)       │
│  - Transparent background           │
├────────────────────────────────────┤
│  New Element Canvas                 │  ← Preview of element being created
│  - On-demand rendering              │
├────────────────────────────────────┤
│  Static Canvas                      │  ← All finalized elements
│  - Throttled rendering (60fps)     │
│  - RoughJS hand-drawn effect       │
└────────────────────────────────────┘
```

**Why three canvases?**
- **Performance** - Separate static content from dynamic interactions
- **Reduced repaints** - Only necessary layers re-render
- **Smooth experience** - Maintains 60fps even with hundreds of elements

---

## Core Features

### 1. Drawing Tools

| Tool | Shortcut | Description | File Location |
|------|----------|-------------|---------------|
| **Selection** | V or 1 | Select and move elements | `App.tsx:6457` |
| **Rectangle** | R or 2 | Draw rectangles | `newElement.ts:150` |
| **Diamond** | D or 3 | Draw diamonds | `newElement.ts:165` |
| **Ellipse** | O or 4 | Draw ellipses/circles | `newElement.ts:180` |
| **Arrow** | A or 5 | Draw arrows | `newLinearElement.ts:520` |
| **Line** | L or 6 | Draw lines | `newLinearElement.ts:540` |
| **Free-draw** | P or 7 | Freehand paths | `App.tsx:5900` |
| **Text** | T or 8 | Add text | `newTextElement.ts:360` |
| **Image** | I or 9 | Insert images | `newImageElement.ts:600` |
| **Eraser** | E or 0 | Erase elements | `App.tsx:6100` |

### 2. Real-Time Collaboration

**Workflow:**
```
1. Click "Collaborate" button
2. Generate share link (contains room ID + encryption key)
3. Share link with team members
4. Automatic synchronization of drawings
```

**Security Features:**
- 🔐 End-to-end encryption (AES-GCM-256)
- 🔑 Keys only in URL fragment (not sent to server)
- 🔄 Automatic conflict resolution
- 👥 Real-time cursor display

**Data Flow:**
```
User edits
    ↓
Encrypt changes (AES-GCM)
    ↓
Send via WebSocket
    ↓
Collaboration server broadcasts
    ↓
Other users receive
    ↓
Decrypt and apply changes
    ↓
Reconcile conflicts
    ↓
Update canvas
```

### 3. Export Functionality

**Supported Formats:**
- 🖼️ PNG - With/without background
- 📄 SVG - Scalable vector graphics
- 📋 Clipboard - Paste directly into other apps
- 💾 .excalidraw - Native format (re-editable)

**Export Options:**
- Select export range (all elements / selection only)
- Adjust scale
- Embed scene data in PNG/SVG (re-openable)
- Dark/light theme

---

## Data Models

### Element Data Structure

**Base Element Properties (shared by all elements):**
```typescript
{
  id: string                    // Unique identifier (nanoid)
  type: string                  // Element type
  x: number                     // X coordinate
  y: number                     // Y coordinate
  width: number                 // Width
  height: number                // Height
  angle: number                 // Rotation angle (radians)
  strokeColor: string           // Stroke color (hex)
  backgroundColor: string       // Fill color (hex)
  fillStyle: string             // Fill pattern (hachure/solid/cross-hatch/zigzag)
  strokeWidth: number           // Stroke width (1/2/4)
  strokeStyle: string           // Stroke pattern (solid/dashed/dotted)
  roughness: number             // Roughness level (0-2)
  opacity: number               // Opacity (0-100)
  seed: number                  // Random seed (deterministic rendering)
  version: number               // Version number (for collaboration)
  versionNonce: number          // Version nonce (conflict resolution)
  index: string                 // Z-index (fractional indexing)
  isDeleted: boolean            // Soft delete flag
  groupIds: string[]            // Group IDs
  frameId: string | null        // Parent frame ID
  boundElements: object[]       // Bound elements (text/arrows)
  updated: number               // Last update timestamp
  link: string | null           // Hyperlink
  locked: boolean               // Locked (prevent editing)
}
```

### Scene File Format (.excalidraw)

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [
    /* Element array */
  ],
  "appState": {
    "viewBackgroundColor": "#ffffff",
    "gridSize": null,
    "zoom": { "value": 1 }
    /* More UI state */
  },
  "files": {
    /* Embedded images (base64) */
  }
}
```

---

## API Reference

### Collaboration API

**WebSocket Events:**

| Event | Direction | Data Format | Description |
|-------|-----------|-------------|-------------|
| `server-broadcast` | Bidirectional | Encrypted element data | Scene updates (reliable) |
| `server-volatile-broadcast` | Bidirectional | Cursor positions | Volatile data (unreliable) |

**Example: Broadcasting Scene Update**
```javascript
// 1. Prepare data
const data = {
  type: "SCENE_UPDATE",
  payload: {
    elements: changedElements
  }
}

// 2. Encrypt
const json = JSON.stringify(data)
const encoded = new TextEncoder().encode(json)
const { encryptedBuffer, iv } = await encryptData(roomKey, encoded)

// 3. Send
socket.emit("server-broadcast", roomId, encryptedBuffer, iv)
```

---

## Deployment Guide

### Environment Requirements

- Node.js >= 18.0.0
- Yarn 1.22.22
- Modern browser (Canvas, WebSocket, Crypto API support)

### Local Development

```bash
# 1. Clone repository
git clone https://github.com/excalidraw/excalidraw
cd excalidraw

# 2. Install dependencies
yarn install

# 3. Start dev server
yarn start

# Browser opens automatically at http://localhost:3000
```

### Production Build

```bash
# Build all packages
yarn build:packages

# Build application
yarn build:app

# Output: excalidraw-app/build/
```

### Docker Deployment

```bash
docker build -t excalidraw .
docker run -p 80:80 excalidraw
```

---

## Development Guide

### Adding a New Element Type

**Steps:**
1. Define type in `packages/element/src/types.ts`
2. Create factory in `packages/element/src/newElement.ts`
3. Add rendering logic in `packages/excalidraw/renderer/renderElement.ts`
4. Add toolbar button in `packages/excalidraw/components/Toolbar.tsx`

### Adding a New Action

```typescript
// packages/excalidraw/actions/actionExample.tsx
export const actionExample = register({
  name: "example",
  label: "Example Action",

  perform: (elements, appState) => {
    // Your logic here
    return { elements, appState }
  },

  keyTest: (event) => event.ctrlKey && event.key === "e",

  PanelComponent: ({ updateData }) => (
    <ToolButton onClick={() => updateData(null)} />
  ),
})
```

---

## FAQ

### Q: How to customize default colors?
A: Modify `DEFAULT_ELEMENT_PROPS` in `packages/excalidraw/constants.ts`

### Q: How to integrate into my app?
A:
```bash
npm install @excalidraw/excalidraw
```
```tsx
import { Excalidraw } from "@excalidraw/excalidraw"
import "@excalidraw/excalidraw/index.css"

export default function App() {
  return <Excalidraw />
}
```

---

**Document Version:** 1.0.0
**Last Updated:** 2025-11-17
**Maintained by:** Project Mastery Team
