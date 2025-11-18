# Reconstruction Prompts | 重建提示词

**Purpose (EN):** Step-by-step prompts to recreate Excalidraw from scratch.
**用途 (中文):** 从头重建 Excalidraw 的分步提示词。

---

## Phase 1: Foundation Setup | 基础设置

### Prompt 1.1: Initialize Monorepo
```
Create a monorepo project for a canvas-based drawing application similar to Excalidraw.

Requirements:
- Package manager: Yarn workspaces
- Node version: >= 18.0.0
- TypeScript: 4.9.4
- React: 19.0.0

Structure:
excalidraw/
├── package.json (root with workspaces)
├── packages/
│   ├── excalidraw/      (main library)
│   ├── common/          (shared utilities)
│   ├── element/         (element logic)
│   ├── math/            (math utilities)
│   └── utils/           (general utils)
├── excalidraw-app/      (web application)
└── tsconfig.json        (shared TypeScript config)

Root package.json workspaces:
["excalidraw-app", "packages/*"]
```

### Prompt 1.2: Setup TypeScript Configuration
```
Configure TypeScript for the monorepo with the following requirements:

tsconfig.json:
- target: ESNext
- lib: ["dom", "esnext"]
- strict: true
- jsx: "react-jsx"
- module: "ESNext"
- moduleResolution: "node"

Path aliases:
- @excalidraw/common → packages/common/src
- @excalidraw/element → packages/element/src
- @excalidraw/math → packages/math/src
- @excalidraw/utils → packages/utils/src
- @excalidraw/excalidraw → packages/excalidraw

Include packages/ and excalidraw-app/
Exclude node_modules, dist, build
```

### Prompt 1.3: Install Core Dependencies
```
Install the following exact versions for the main library (packages/excalidraw):

Core:
- react@19.0.0
- react-dom@19.0.0
- typescript@4.9.4

Drawing:
- roughjs@4.6.4 (hand-drawn rendering)
- perfect-freehand@1.2.0 (smooth paths)

State Management:
- jotai@2.11.0 (atomic state)
- jotai-scope@0.7.2 (isolated scopes)

UI Components:
- @radix-ui/react-popover@1.1.6
- @radix-ui/react-tabs@1.1.3

Utilities:
- nanoid@3.3.3 (ID generation)
- pako@2.0.3 (compression)
- lodash.throttle@4.1.1
- lodash.debounce@4.0.8
- fractional-indexing@3.2.0 (z-index)

Dev Dependencies:
- vite@5.0.12
- vitest@3.0.6
- @vitejs/plugin-react@3.1.0
- esbuild@0.19.10
```

---

## Phase 2: Core Architecture | 核心架构

### Prompt 2.1: Create Scene Class
```
Create a Scene class in packages/element/src/Scene.ts that manages the element collection.

Requirements:
1. Store elements in a readonly array
2. Maintain fast lookup via Map<id, element>
3. Separate deleted/non-deleted elements
4. Track scene version via nonce
5. Trigger callbacks on updates

Interface:
class Scene {
  private elements: readonly OrderedExcalidrawElement[]
  private elementsMap: Map<string, OrderedExcalidrawElement>
  private nonDeletedElementsMap: Map<string, NonDeletedExcalidrawElement>
  private sceneNonce: number | undefined

  replaceAllElements(elements: ElementsMapOrArray): void
  insertElement(element: ExcalidrawElement): void
  getElement(id: string): ExcalidrawElement | null
  getNonDeletedElements(): readonly NonDeletedExcalidrawElement[]
  getElementsIncludingDeleted(): readonly OrderedExcalidrawElement[]

  addCallback(cb: SceneUpdateCallback): void
  triggerUpdate(): void
}

When elements change:
1. Increment sceneNonce
2. Rebuild maps
3. Fire all callbacks
```

### Prompt 2.2: Implement Triple-Canvas System
```
Create three separate canvas layers for optimal rendering performance:

1. Static Canvas (packages/excalidraw/components/canvases/StaticCanvas.tsx):
   - Renders all finalized elements
   - Throttled to 60fps using requestAnimationFrame
   - Memoized with React.memo (only re-render on scene/viewport changes)
   - Uses RoughJS for hand-drawn aesthetic

2. Interactive Canvas (packages/excalidraw/components/canvases/InteractiveCanvas.tsx):
   - Always rendering (RAF loop)
   - Shows selection boxes, transform handles, cursors
   - Transparent background
   - Uses AnimationController for continuous loop

3. New Element Canvas:
   - Preview layer for element being created
   - Rendered on-demand

Layer them with CSS:
position: absolute
z-index: static=0, new=1, interactive=2
```

### Prompt 2.3: Create Element Factory Functions
```
Implement element creation factories in packages/element/src/newElement.ts:

Base factory _newElementBase<T>(type, opts):
- Generate id: nanoid()
- Set default props (strokeColor, backgroundColor, etc.)
- Initialize version: 1, versionNonce: randomInteger()
- Set updated: Date.now()
- Validate coordinates (< 1e6)

Specialized factories:
1. newElement(type: "rectangle" | "ellipse" | "diamond")
2. newLinearElement(type: "line" | "arrow", points)
3. newTextElement(text, fontSize, fontFamily)
4. newFreeDrawElement(pressures, points)
5. newImageElement(fileId, status)
6. newFrameElement(name)

Each factory should:
- Call _newElementBase with type-specific defaults
- Add type-specific properties
- Return typed element
```

---

## Phase 3: Rendering Pipeline | 渲染管道

### Prompt 3.1: Implement Viewport Culling
```
Create viewport culling in packages/excalidraw/scene/export.ts:

Function getVisibleCanvasElements():
Input:
- elementsMap: Map of all elements
- zoom: Current zoom level
- scrollX, scrollY: Viewport offset
- width, height: Canvas dimensions
- offsetLeft, offsetTop: Canvas position

Algorithm:
1. Calculate viewport bounds in scene coordinates:
   minX = -offsetLeft / zoom + scrollX
   minY = -offsetTop / zoom + scrollY
   maxX = (-offsetLeft + width) / zoom + scrollX
   maxY = (-offsetTop + height) / zoom + scrollY

2. For each element:
   - Get element bounds [x1, y1, x2, y2]
   - Check intersection with viewport:
     if (x2 >= minX && x1 <= maxX && y2 >= minY && y1 <= maxY):
       add to visible elements

3. Return visible elements array

This prevents rendering off-screen elements, crucial for performance with 100+ elements.
```

### Prompt 3.2: Create RoughJS Renderer
```
Implement element rendering using RoughJS in packages/excalidraw/renderer/renderElement.ts:

For each element type:

Rectangle:
  rc.rectangle(x, y, width, height, {
    stroke: element.strokeColor,
    fill: element.backgroundColor,
    fillStyle: element.fillStyle, // hachure, solid, cross-hatch, zigzag
    strokeWidth: element.strokeWidth,
    roughness: element.roughness, // 0=architect, 1=artist, 2=cartoonist
    seed: element.seed, // Deterministic randomness
  })

Line/Arrow:
  rc.linearPath(points, options)
  // Add arrowheads manually using canvas path

Free-draw:
  Use perfect-freehand library for smooth paths

Text:
  Use canvas.fillText() (not RoughJS)

Image:
  canvas.drawImage()

Apply transformations:
- ctx.save()
- ctx.translate(x + width/2, y + height/2)
- ctx.rotate(angle)
- ctx.translate(-width/2, -height/2)
- render element
- ctx.restore()
```

---

## Phase 4: State Management | 状态管理

### Prompt 4.1: Setup Jotai Stores
```
Create Jotai stores for state management:

1. Editor Store (packages/excalidraw/editor-jotai.ts):
   import { createStore, atom } from "jotai"
   import { createIsolation } from "jotai-scope"

   const jotai = createIsolation()
   export const editorJotaiStore = createStore()
   export const { useAtom, useSetAtom, useAtomValue } = jotai
   export const EditorJotaiProvider = jotai.Provider

2. Key Atoms:
   export const libraryItemsAtom = atom({
     status: "loaded",
     isInitialized: false,
     libraryItems: [],
   })

3. App Store (excalidraw-app/app-jotai.ts):
   export const appJotaiStore = createStore()
   export const collabAPIAtom = atom(null)
   export const isCollaboratingAtom = atom(false)
```

### Prompt 4.2: Implement Local Storage Auto-Save
```
Create auto-save system in excalidraw-app/data/LocalData.ts:

Features:
1. Debounced save (300ms) to localStorage
2. IndexedDB for binary files (images)
3. Quota exceeded handling

Implementation:
class LocalData {
  private static _save = debounce(async (elements, appState, files) => {
    // 1. Save to localStorage
    localStorage.setItem("excalidraw", JSON.stringify(
      clearElementsForLocalStorage(elements)
    ))
    localStorage.setItem("excalidraw-state", JSON.stringify(
      clearAppStateForLocalStorage(appState)
    ))

    // 2. Save files to IndexedDB
    await saveFilesToIndexedDB(files)
  }, 300)

  static save(elements, appState, files) {
    if (!this.isSavePaused()) {
      this._save(elements, appState, files)
    }
  }
}

Handle quota errors:
catch (error) {
  if (error.name === "QuotaExceededError") {
    appJotaiStore.set(localStorageQuotaExceededAtom, true)
  }
}
```

---

## Phase 5: Collaboration | 协作

### Prompt 5.1: Implement End-to-End Encryption
```
Create encryption system in packages/excalidraw/data/encryption.ts using Web Crypto API:

Algorithm: AES-GCM-256 with PBKDF2 key derivation

1. Key Derivation:
   const deriveKey = async (passphrase: string) => {
     const keyMaterial = await crypto.subtle.importKey(
       "raw", encoder.encode(passphrase), "PBKDF2", false, ["deriveKey"]
     )
     return await crypto.subtle.deriveKey(
       {
         name: "PBKDF2",
         salt: encoder.encode("excalidraw-salt"),
         iterations: 100000,
         hash: "SHA-256",
       },
       keyMaterial,
       { name: "AES-GCM", length: 256 },
       false,
       ["encrypt", "decrypt"],
     )
   }

2. Encryption:
   const encryptData = async (passphrase, data) => {
     const key = await deriveKey(passphrase)
     const iv = crypto.getRandomValues(new Uint8Array(16))
     const encrypted = await crypto.subtle.encrypt(
       { name: "AES-GCM", iv },
       key,
       data,
     )
     return { encryptedBuffer: encrypted, iv }
   }

3. Decryption:
   const decryptData = async (iv, encrypted, passphrase) => {
     const key = await deriveKey(passphrase)
     const decrypted = await crypto.subtle.decrypt(
       { name: "AES-GCM", iv },
       key,
       encrypted,
     )
     return { decryptedBuffer: decrypted }
   }
```

### Prompt 5.2: Create WebSocket Collaboration
```
Implement real-time collaboration using Socket.IO:

1. Connect to WebSocket server:
   const socket = io("https://collab-server.com", {
     transports: ["websocket"],
   })

2. Broadcast scene updates:
   const broadcastScene = async (elements) => {
     // Only send changed elements
     const changedElements = elements.filter(el =>
       el.version > lastBroadcastedVersion.get(el.id)
     )

     // Encrypt payload
     const json = JSON.stringify({ type: "UPDATE", elements: changedElements })
     const { encryptedBuffer, iv } = await encryptData(roomKey, encoder.encode(json))

     // Emit to server
     socket.emit("server-broadcast", roomId, encryptedBuffer, iv)

     // Track versions
     changedElements.forEach(el => lastBroadcastedVersion.set(el.id, el.version))
   }

3. Receive remote updates:
   socket.on("server-broadcast", async (encrypted, iv) => {
     // Decrypt
     const { decryptedBuffer } = await decryptData(iv, encrypted, roomKey)
     const data = JSON.parse(decoder.decode(decryptedBuffer))

     // Reconcile with local elements
     const reconciled = reconcileElements(localElements, data.elements, appState)

     // Update scene
     scene.replaceAllElements(reconciled)
   })
```

---

## Phase 6: Actions System | 动作系统

### Prompt 6.1: Create Action Manager
```
Implement action registration and execution in packages/excalidraw/actions/:

1. Action Interface:
   interface Action {
     name: ActionName
     label: string
     icon?: ReactNode
     perform: (elements, appState, formData, app) => ActionResult
     keyTest?: (event, appState, elements) => boolean
     predicate?: (elements, appState) => boolean
     PanelComponent?: React.FC
   }

2. Action Manager:
   class ActionManager {
     actions: Record<ActionName, Action> = {}

     registerAction(action: Action) {
       this.actions[action.name] = action
     }

     executeAction(name: ActionName, formData) {
       const action = this.actions[name]
       if (!action || !action.predicate(elements, appState)) return
       const result = action.perform(elements, appState, formData, app)
       this.updater(result)
     }

     handleKeyDown(event) {
       const matching = Object.values(this.actions).filter(a =>
         a.keyTest?.(event, appState, elements)
       )
       if (matching.length === 1) {
         event.preventDefault()
         this.updater(matching[0].perform(elements, appState, null, app))
       }
     }
   }

3. Register actions:
   export const actionDeleteSelected = register({
     name: "deleteSelectedElements",
     label: "Delete",
     keyTest: (e) => e.key === "Delete" || e.key === "Backspace",
     perform: (elements, appState) => ({
       elements: elements.map(el =>
         appState.selectedElementIds[el.id] ? {...el, isDeleted: true} : el
       ),
       appState: { ...appState, selectedElementIds: {} },
     }),
   })
```

---

## Phase 7: UI Components | UI 组件

### Prompt 7.1: Create Toolbar
```
Build a responsive toolbar in packages/excalidraw/components/Toolbar.tsx:

Requirements:
1. Tool buttons for each drawing tool:
   - Selection
   - Rectangle
   - Diamond
   - Ellipse
   - Arrow
   - Line
   - Free-draw
   - Text
   - Image
   - Eraser
   - Hand (pan)

2. Active tool highlighting

3. Keyboard shortcuts display on hover

4. Mobile-responsive (stack vertically on narrow screens)

Example button:
<ToolButton
  icon={<RectangleIcon />}
  type="rectangle"
  active={activeTool.type === "rectangle"}
  onClick={() => setActiveTool({ type: "rectangle" })}
  title="Rectangle — R"
  aria-label="Rectangle tool"
/>

Styles:
- Island container (floating panel)
- Hover states
- Active state (highlighted)
- Disabled state (grayed out)
```

---

## Summary | 总结

**English:** These prompts provide a structured approach to rebuilding Excalidraw from scratch. Each prompt focuses on a specific architectural component and includes implementation details, code examples, and technical requirements. Follow the phases sequentially for best results.

**中文:** 这些提示词提供了从头重建 Excalidraw 的结构化方法。每个提示词专注于特定的架构组件，并包含实现细节、代码示例和技术要求。按顺序遵循各阶段以获得最佳结果。

**Total Phases:** 7
**Estimated Time:** 4-8 weeks (depending on experience level)
**Difficulty:** Advanced
