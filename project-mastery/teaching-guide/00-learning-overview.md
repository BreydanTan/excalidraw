# Excalidraw Learning Guide Overview | 学习指南概览

**Target Audience (EN):** Developers and non-developers who want to understand and potentially recreate Excalidraw.
**目标受众 (中文):** 希望理解并可能重建 Excalidraw 的开发者和非开发者。

---

## Learning Path | 学习路径

### Timeline | 时间线
- **Total Duration (总时长):** 4-8 weeks
- **Daily Study (每日学习):** 2-3 hours
- **Difficulty (难度):** Intermediate to Advanced

### 5-Stage Learning Journey | 5阶段学习之旅

```
Stage 1: Foundation (第1阶段：基础) - Week 1
└─ Understand project structure, run the app, make simple changes

Stage 2: Core Concepts (第2阶段：核心概念) - Week 2
└─ Canvas rendering, state management, element system

Stage 3: Advanced Features (第3阶段：高级功能) - Weeks 3-4
└─ Collaboration, encryption, actions system

Stage 4: Full Recreation (第4阶段：完整重建) - Weeks 5-6
└─ Build a simplified version from scratch

Stage 5: Extensions (第5阶段：扩展) - Weeks 7-8
└─ Add custom features, integrate into your project
```

---

## Stage 1: Foundation (Week 1) | 第1阶段：基础

### 🎯 Learning Objectives | 学习目标

By the end of Stage 1, you will:
- ✅ Run Excalidraw locally
- ✅ Understand the monorepo structure
- ✅ Know where to find any feature's code
- ✅ Make simple UI modifications

### 📚 Prerequisites | 前置知识

**Required (必需):**
- Basic JavaScript/TypeScript knowledge
- Familiarity with React
- Understanding of npm/yarn

**Helpful (有帮助):**
- HTML Canvas API basics
- WebSocket concepts
- State management (Redux/Zustand/Jotai)

### 🛠️ Practical Tasks | 实践任务

**Task 1.1: Environment Setup**
```bash
# Clone and run
git clone https://github.com/excalidraw/excalidraw
cd excalidraw
yarn install
yarn start
```

**Success Criteria (成功标准):**
- ✅ App opens in browser
- ✅ Can draw a rectangle
- ✅ No console errors

**Task 1.2: Modify UI Text**
Find and change:
- Toolbar button labels
- Menu text
- Welcome screen message

**Files to edit:**
- `packages/excalidraw/locales/en.json`
- `packages/excalidraw/components/welcome-screen/WelcomeScreen.tsx`

**Task 1.3: Change Default Colors**
Modify default stroke/fill colors in:
- `packages/excalidraw/constants.ts` → `DEFAULT_ELEMENT_PROPS`

---

## Stage 2: Core Concepts (Week 2) | 第2阶段：核心概念

### 🎯 Learning Objectives | 学习目标

- ✅ Understand the triple-canvas architecture
- ✅ Know how elements are created and stored
- ✅ Understand the rendering pipeline
- ✅ Grasp state management with Jotai

### 📊 Key Concepts | 关键概念

**Concept 1: Triple-Canvas System**
```
Interactive Canvas (top)     ← Selection UI, cursors
New Element Canvas (middle)  ← Preview while drawing
Static Canvas (bottom)       ← Finalized elements
```

**Analogy (类比):** Like layers in Photoshop - each canvas is a separate layer.

**Concept 2: Element Data Structure**
```javascript
{
  id: "abc123",           // Unique ID
  type: "rectangle",      // What shape?
  x: 100, y: 200,        // Position
  width: 300, height: 150, // Size
  strokeColor: "#000",    // Border color
  // ... 20+ more properties
}
```

**Analogy (类比):** Like a recipe - lists all ingredients (properties) needed to create the dish (element).

**Concept 3: Scene Class**
- Stores all elements
- Provides fast lookup (Map)
- Triggers re-renders when changed

**Analogy (类比):** Like a database - stores and manages all your data.

### 🛠️ Practical Tasks | 实践任务

**Task 2.1: Trace Element Creation**
1. Set breakpoint in `packages/element/src/newElement.ts`
2. Draw a rectangle
3. Step through code to see how element is created

**Task 2.2: Modify Rendering**
Change how rectangles are rendered:
- File: `packages/excalidraw/renderer/renderElement.ts`
- Make all rectangles have rounded corners
- Add a custom pattern

**Task 2.3: Explore State**
Open browser console:
```javascript
// Access global scene
window.Excalidraw.scene.getNonDeletedElements()

// See all element IDs
elements.map(el => el.id)
```

---

## Stage 3: Advanced Features (Weeks 3-4) | 第3阶段：高级功能

### 🎯 Learning Objectives | 学习目标

- ✅ Understand end-to-end encryption
- ✅ Know how real-time collaboration works
- ✅ Master the actions system
- ✅ Learn text editing implementation

### 📊 Key Concepts | 关键概念

**Concept 1: Encryption Flow**
```
User edits → Encrypt (AES-GCM) → Send → Decrypt → Display
```

**Concept 2: Collaboration Architecture**
```
Client A                Server              Client B
   │                      │                    │
   ├──── Encrypted ───────►                    │
   │      changes         │                    │
   │                      ├──── Broadcast ────►│
   │                      │                    │
   │                      │      Decrypt       │
   │                      │      Apply         │
```

**Concept 3: Actions Pattern**
```javascript
const action = {
  name: "deleteSelected",
  perform: (elements) => /* delete selected */,
  keyTest: (event) => event.key === "Delete",
  PanelComponent: /* UI button */
}
```

### 🛠️ Practical Tasks | 实践任务

**Task 3.1: Test Collaboration Locally**
1. Open two browser windows
2. Start collaboration session
3. Draw in one window
4. Verify updates appear in other window

**Task 3.2: Create Custom Action**
Add action to change all elements to blue:
```typescript
export const actionMakeBlue = register({
  name: "makeBlue",
  label: "Make All Blue",
  perform: (elements) => ({
    elements: elements.map(el => ({
      ...el,
      strokeColor: "#0000ff"
    }))
  }),
  keyTest: (e) => e.ctrlKey && e.key === "b"
})
```

---

## Stage 4: Full Recreation (Weeks 5-6) | 第4阶段：完整重建

### 🎯 Learning Objectives | 学习目标

- ✅ Build a simplified Excalidraw from scratch
- ✅ Implement core drawing tools
- ✅ Add basic collaboration

### 🏗️ Build Plan | 构建计划

**Week 5: Minimal Version**
1. Day 1-2: Setup canvas and basic shapes
2. Day 3-4: Selection and transform
3. Day 5-7: Save/load to localStorage

**Week 6: Add Features**
1. Day 1-2: Text tool
2. Day 3-4: Export to PNG
3. Day 5-7: Basic collaboration

### 🛠️ Step-by-Step | 分步指南

**Step 1: Create Basic Canvas**
```html
<canvas id="canvas" width="800" height="600"></canvas>
```

```javascript
const canvas = document.getElementById("canvas")
const ctx = canvas.getContext("2d")

// Draw rectangle
ctx.fillRect(100, 100, 200, 150)
```

**Step 2: Add RoughJS**
```bash
npm install roughjs
```

```javascript
import rough from "roughjs"

const rc = rough.canvas(canvas)
rc.rectangle(100, 100, 200, 150, {
  roughness: 1,
  fill: "#ced4da"
})
```

**Step 3: Make Interactive**
```javascript
canvas.onmousedown = (e) => {
  const element = {
    type: "rectangle",
    x: e.offsetX,
    y: e.offsetY,
    width: 0,
    height: 0
  }
  elements.push(element)
}

canvas.onmousemove = (e) => {
  if (dragging) {
    const lastElement = elements[elements.length - 1]
    lastElement.width = e.offsetX - lastElement.x
    lastElement.height = e.offsetY - lastElement.y
    redraw()
  }
}
```

---

## Stage 5: Extensions (Weeks 7-8) | 第5阶段：扩展

### 🎯 Learning Objectives | 学习目标

- ✅ Add custom features
- ✅ Integrate Excalidraw into your project
- ✅ Deploy to production

### 💡 Extension Ideas | 扩展想法

**Idea 1: Custom Shapes**
- Triangle
- Star
- Hexagon
- Custom SVG paths

**Idea 2: Templates**
- Pre-made flowchart templates
- Organization chart templates
- Network diagram templates

**Idea 3: AI Features**
- Image-to-diagram conversion
- Auto-layout for diagrams
- Smart connectors

**Idea 4: Integrations**
- Notion integration
- Slack bot
- VS Code extension
- Figma plugin

---

## Resources | 学习资源

### Official Documentation | 官方文档
- 📖 Excalidraw Docs: https://docs.excalidraw.com
- 🐙 GitHub: https://github.com/excalidraw/excalidraw
- 💬 Discord: https://discord.gg/UexuTaE

### Learning Materials | 学习材料
- 📺 Canvas API Tutorial: MDN Canvas Tutorial
- 📚 React Docs: https://react.dev
- 🎓 TypeScript Handbook: https://www.typescriptlang.org

### Tools | 工具
- VS Code
- React DevTools
- Chrome DevTools
- Git/GitHub

---

## Self-Assessment Checklist | 自我评估清单

### Stage 1 Assessment
- [ ] Can run Excalidraw locally
- [ ] Understand monorepo structure
- [ ] Can find feature code locations
- [ ] Made successful UI modifications

**Score:** __ / 4  (3+ to proceed to Stage 2)

### Stage 2 Assessment
- [ ] Understand triple-canvas architecture
- [ ] Can explain element data structure
- [ ] Know how Scene class works
- [ ] Can modify rendering logic

**Score:** __ / 4  (3+ to proceed to Stage 3)

### Stage 3 Assessment
- [ ] Understand encryption flow
- [ ] Can explain collaboration architecture
- [ ] Created custom action
- [ ] Tested collaboration locally

**Score:** __ / 4  (3+ to proceed to Stage 4)

### Stage 4 Assessment
- [ ] Built simplified Excalidraw
- [ ] Implemented basic drawing tools
- [ ] Added save/load functionality
- [ ] (Optional) Added basic collaboration

**Score:** __ / 4  (3+ to proceed to Stage 5)

---

## Tips for Success | 成功提示

### For Non-Programmers | 给非程序员
1. **Start with concepts** - Don't worry about code details initially
2. **Use analogies** - Relate technical concepts to everyday things
3. **Take notes** - Keep a glossary of technical terms
4. **Ask questions** - Use AI assistants to explain unfamiliar concepts

### For Programmers | 给程序员
1. **Read the code** - Best way to understand is to read actual implementation
2. **Debug actively** - Set breakpoints and step through code
3. **Experiment** - Make changes and see what breaks
4. **Build features** - Best learning is by doing

### General Tips | 通用提示
1. **One concept at a time** - Don't try to learn everything at once
2. **Practice regularly** - 30 minutes daily > 4 hours weekly
3. **Join community** - Discord server is very helpful
4. **Track progress** - Use this guide's checklists

---

**Next Steps | 下一步:**
1. Choose your starting stage based on experience level
2. Set aside dedicated learning time
3. Prepare development environment
4. Begin with Stage 1 practical tasks

**Good luck on your learning journey! | 学习愉快！**
