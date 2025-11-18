# 00 - Excalidraw Project Overview | 项目概览

**Document Version:** 1.0.0
**Analysis Date:** 2025-11-17
**Project Version:** v0.18.0 (@excalidraw/excalidraw)
**Total Source Files:** 565 TypeScript/JavaScript files

---

## 📋 Executive Summary | 项目简介

**English:**
Excalidraw is an open-source virtual whiteboard application for sketching hand-drawn diagrams. It consists of a reusable React component library published to npm (`@excalidraw/excalidraw`) and a full-featured web application (excalidraw.com). The project uses a monorepo architecture managed by Yarn workspaces, separating core functionality into focused packages.

**中文:**
Excalidraw 是一个开源的虚拟白板应用，用于绘制手绘风格的图表。它由两部分组成：一个发布到 npm 的可复用 React 组件库（`@excalidraw/excalidraw`）和一个功能完整的 Web 应用（excalidraw.com）。项目采用 monorepo 架构，使用 Yarn workspaces 管理，将核心功能分离到多个专注的包中。

**Project Type:** Frontend Library + Web Application (Monorepo)
**Architecture:** Modular Monorepo with Yarn Workspaces
**License:** MIT
**Repository:** https://github.com/excalidraw/excalidraw

---

## 🏗️ Technology Stack | 技术栈详情

### Core Frontend Technologies | 核心前端技术

| Technology | Version | Purpose (EN) | 用途 (中文) | File Reference |
|-----------|---------|--------------|------------|----------------|
| **React** | 19.0.0 | UI framework | UI 框架 | `package.json:36` |
| **React DOM** | 19.0.0 | DOM rendering | DOM 渲染 | `package.json:37` |
| **TypeScript** | 4.9.4 | Type safety | 类型安全 | `package.json:37` |
| **Jotai** | 2.11.0 | Atomic state management | 原子化状态管理 | `packages/excalidraw/package.json:98` |

### Build & Development Tools | 构建和开发工具

| Tool | Version | Purpose (EN) | 用途 (中文) | Configuration File |
|------|---------|--------------|------------|-------------------|
| **Vite** | 5.0.12 | App bundler & dev server | 应用打包和开发服务器 | `excalidraw-app/vite.config.mts` |
| **esbuild** | 0.19.10 | Package bundler (fast) | 包打包器（快速） | `scripts/buildPackage.js` |
| **Vitest** | 3.0.6 | Testing framework | 测试框架 | `vitest.config.mts` |
| **ESLint** | - | Code linting | 代码检查 | `.eslintrc.json` |
| **Prettier** | 2.6.2 | Code formatting | 代码格式化 | `package.json:50` |
| **Husky** | 7.0.4 | Git hooks | Git 钩子 | `.husky/` |

### Drawing & Rendering Libraries | 绘图和渲染库

| Library | Version | Purpose (EN) | 用途 (中文) |
|---------|---------|--------------|------------|
| **RoughJS** | 4.6.4 | Hand-drawn style rendering | 手绘风格渲染 |
| **Perfect Freehand** | 1.2.0 | Smooth freehand drawing | 平滑手绘 |
| **Pica** | 7.1.1 | Image resizing | 图片缩放 |
| **Points-on-Curve** | 1.0.1 | Curve calculations | 曲线计算 |

### Collaboration & Backend | 协作和后端

| Service | Version | Purpose (EN) | 用途 (中文) | Integration File |
|---------|---------|--------------|------------|-----------------|
| **Socket.IO Client** | 4.7.2 | Real-time WebSocket | 实时 WebSocket | `excalidraw-app/package.json:38` |
| **Firebase** | 11.3.1 | Authentication & storage | 认证和存储 | `excalidraw-app/data/firebase.ts` |
| **Sentry** | 9.0.1 | Error tracking | 错误追踪 | `excalidraw-app/sentry.ts` |

### UI Component Libraries | UI 组件库

| Library | Version | Purpose (EN) | 用途 (中文) |
|---------|---------|--------------|------------|
| **Radix UI - Popover** | 1.1.6 | Accessible popovers | 无障碍弹窗 |
| **Radix UI - Tabs** | 1.1.3 | Tab components | 标签页组件 |
| **Tunnel Rat** | 0.1.2 | Portal rendering | 传送门渲染 |
| **Open Color** | 1.9.1 | Color system | 颜色系统 |

### Data Processing | 数据处理

| Library | Version | Purpose (EN) | 用途 (中文) |
|---------|---------|--------------|------------|
| **Pako** | 2.0.3 | Compression (gzip) | 压缩（gzip） |
| **Nanoid** | 3.3.3 | Unique ID generation | 唯一 ID 生成 |
| **Browser FS Access** | 0.29.1 | File system access | 文件系统访问 |
| **Fractional Indexing** | 3.2.0 | Z-index management | Z 轴索引管理 |

### PWA & Performance | PWA 和性能

| Tool | Version | Purpose (EN) | 用途 (中文) |
|------|---------|--------------|------------|
| **vite-plugin-pwa** | 0.21.1 | Progressive Web App | 渐进式 Web 应用 |
| **pwacompat** | 2.0.17 | PWA compatibility | PWA 兼容性 |
| **Workbox** | (via plugin) | Service worker caching | Service Worker 缓存 |

### AI & Special Features | AI 和特殊功能

| Package | Version | Purpose (EN) | 用途 (中文) |
|---------|---------|--------------|------------|
| **@excalidraw/mermaid-to-excalidraw** | 1.1.3 | Mermaid diagram import | Mermaid 图表导入 |
| **@excalidraw/laser-pointer** | 1.3.1 | Presentation laser pointer | 演示激光笔 |
| **Fuzzy** | 0.1.3 | Fuzzy search | 模糊搜索 |

### Font Processing (WASM) | 字体处理（WebAssembly）

| Library | Version | Purpose (EN) | 用途 (中文) |
|---------|---------|--------------|------------|
| **HarfBuzzJS** | 0.3.6 | Text shaping (WASM) | 文本塑形（WASM） |
| **FontEditor Core** | 2.4.1 | Font editing | 字体编辑 |

---

## 📁 Project Structure | 项目结构

### Monorepo Architecture | Monorepo 架构

```
excalidraw/                                  # Root directory (monorepo)
├── 📦 packages/                            # Core packages (workspace)
│   ├── excalidraw/                         # Main library [@excalidraw/excalidraw]
│   │   ├── index.tsx                       # Library entry point (309 lines)
│   │   ├── components/                     # React components (20+ categories)
│   │   │   ├── App.tsx                     # ⚠️ HUGE: 11,508 lines - Core app logic
│   │   │   ├── LayerUI.tsx                 # Main UI overlay
│   │   │   ├── canvases/                   # Canvas rendering
│   │   │   │   ├── InteractiveCanvas.tsx   # Interactive drawing canvas
│   │   │   │   ├── StaticCanvas.tsx        # Static rendering (export/preview)
│   │   │   │   └── NewElementCanvas.tsx    # Preview for new elements
│   │   │   ├── ColorPicker/                # Color selection UI
│   │   │   ├── CommandPalette/             # Cmd+K command palette
│   │   │   ├── TTDDialog/                  # 🤖 Text-to-Diagram (AI)
│   │   │   ├── DiagramToCodePlugin/        # 🤖 Diagram-to-Code
│   │   │   ├── Sidebar/                    # Sidebar panels
│   │   │   ├── Stats/                      # Statistics display
│   │   │   ├── main-menu/                  # Main menu
│   │   │   ├── welcome-screen/             # Welcome screen
│   │   │   ├── live-collaboration/         # Collaboration UI
│   │   │   └── footer/                     # Footer components
│   │   ├── actions/                        # Action handlers (~45 files)
│   │   │   ├── actionCanvas.tsx            # Canvas actions (zoom, pan)
│   │   │   ├── actionProperties.tsx        # Element properties
│   │   │   ├── actionDeleteSelected.tsx    # Delete action
│   │   │   ├── actionExport.tsx            # Export actions
│   │   │   ├── actionHistory.tsx           # Undo/redo
│   │   │   ├── actionAlign.tsx             # Alignment
│   │   │   └── actionDistribute.tsx        # Distribution
│   │   ├── data/                           # Data handling
│   │   │   ├── ai/                         # AI-related types
│   │   │   ├── blob.ts                     # File/blob operations
│   │   │   ├── json.ts                     # JSON serialization
│   │   │   ├── library.ts                  # Shape library management
│   │   │   ├── restore.ts                  # Data restoration
│   │   │   ├── reconcile.ts                # 🔄 Element reconciliation (collab)
│   │   │   ├── encryption.ts               # 🔐 End-to-end encryption
│   │   │   └── EditorLocalStorage.ts       # Local storage
│   │   ├── scene/                          # Scene management
│   │   │   ├── Renderer.ts                 # Main renderer
│   │   │   ├── export.ts                   # Scene export
│   │   │   ├── zoom.ts                     # Zoom logic
│   │   │   └── scroll.ts                   # Scroll logic
│   │   ├── renderer/                       # Rendering implementations
│   │   │   ├── staticScene.ts              # Static scene rendering
│   │   │   ├── staticSvgScene.ts           # SVG rendering
│   │   │   └── interactiveScene.ts         # Interactive rendering
│   │   ├── hooks/                          # React hooks (11 files)
│   │   ├── fonts/                          # Font files (10+ families)
│   │   │   ├── Virgil/                     # Default hand-drawn font
│   │   │   ├── Cascadia/                   # Monospace font
│   │   │   ├── Assistant/                  # Sans-serif font
│   │   │   └── ... (7 more font families)
│   │   ├── locales/                        # 🌍 i18n (60+ languages)
│   │   │   ├── en.json                     # English
│   │   │   ├── zh-CN.json                  # Chinese (Simplified)
│   │   │   ├── es-ES.json                  # Spanish
│   │   │   └── ... (57+ more languages)
│   │   ├── wysiwyg/                        # WYSIWYG text editor
│   │   ├── eraser/                         # Eraser tool
│   │   ├── lasso/                          # Lasso selection
│   │   ├── subset/                         # 🔧 Font subsetting (WASM)
│   │   │   ├── harfbuzz/                   # HarfBuzz WASM
│   │   │   └── woff2/                      # WOFF2 processing
│   │   ├── tests/                          # Unit tests
│   │   ├── css/                            # SCSS stylesheets
│   │   ├── context/                        # React contexts
│   │   └── package.json                    # v0.18.0
│   │
│   ├── common/                             # Common utilities [@excalidraw/common]
│   │   └── src/
│   │       ├── constants.ts                # Global constants
│   │       ├── utils.ts                    # Utility functions
│   │       ├── colors.ts                   # Color utilities
│   │       ├── points.ts                   # Point math
│   │       ├── emitter.ts                  # Event emitter
│   │       └── package.json                # v0.18.0
│   │
│   ├── element/                            # Element logic [@excalidraw/element]
│   │   └── src/
│   │       ├── types.ts                    # Element type definitions
│   │       ├── newElement.ts               # Element creation
│   │       ├── mutateElement.ts            # Element mutation
│   │       ├── bounds.ts                   # Bounding box calculations
│   │       ├── collision.ts                # Collision detection
│   │       ├── resizeElements.ts           # Resize logic
│   │       ├── dragElements.ts             # Drag logic
│   │       ├── linearElementEditor.ts      # Line/arrow editing
│   │       ├── binding.ts                  # 🔗 Arrow-to-shape binding
│   │       ├── frame.ts                    # Frame elements
│   │       ├── embeddable.ts               # Embeddable content
│   │       ├── flowchart.ts                # Flowchart features
│   │       └── package.json                # v0.18.0
│   │
│   ├── math/                               # Math utilities [@excalidraw/math]
│   │   └── src/
│   │       ├── point.ts                    # Point operations
│   │       ├── vector.ts                   # Vector math
│   │       ├── line.ts                     # Line math
│   │       ├── polygon.ts                  # Polygon operations
│   │       ├── ellipse.ts                  # Ellipse calculations
│   │       └── package.json                # v0.18.0
│   │
│   └── utils/                              # General utilities [@excalidraw/utils]
│       └── src/
│           ├── export.ts                   # Export utilities
│           ├── bbox.ts                     # Bounding box
│           ├── withinBounds.ts             # Boundary checking
│           └── package.json                # v0.18.0
│
├── 🌐 excalidraw-app/                      # Web application (excalidraw.com)
│   ├── index.tsx                           # App entry point (17 lines)
│   ├── index.html                          # HTML template
│   ├── App.tsx                             # Main app wrapper (1,183 lines)
│   ├── collab/                             # 🔄 Real-time collaboration
│   │   ├── Collab.tsx                      # Main collaboration logic
│   │   ├── Portal.tsx                      # Collaboration portal
│   │   └── CollabError.tsx                 # Error handling
│   ├── components/                         # App-specific components
│   │   ├── AppMainMenu.tsx                 # Custom main menu
│   │   ├── AppWelcomeScreen.tsx            # Custom welcome screen
│   │   └── ExportToExcalidrawPlus.tsx      # Excalidraw+ integration
│   ├── data/                               # App data handling
│   │   ├── firebase.ts                     # 🔥 Firebase integration
│   │   ├── FileManager.ts                  # File management
│   │   └── LocalData.ts                    # Local storage
│   ├── share/                              # Sharing features
│   ├── tests/                              # App tests
│   ├── vite.config.mts                     # Vite configuration
│   ├── sentry.ts                           # 🐛 Error tracking
│   └── package.json                        # v1.0.0 (private)
│
├── 📚 examples/                            # Integration examples
│   ├── with-nextjs/                        # Next.js example
│   │   ├── src/
│   │   ├── package.json
│   │   └── next.config.js
│   └── with-script-in-browser/             # Vanilla JS example
│       ├── components/
│       └── vite.config.mts
│
├── 📖 dev-docs/                            # Developer documentation
│   ├── docs/                               # Docusaurus docs
│   ├── docusaurus.config.js               # Doc site config
│   └── package.json
│
├── 🔧 scripts/                             # Build & utility scripts
│   ├── buildPackage.js                     # Package build script
│   ├── build-version.js                    # Version file generator
│   ├── release.js                          # Release automation
│   └── woff2/                              # Font processing scripts
│
├── 🌍 public/                              # Public assets
│   ├── Virgil.woff2                        # Font files
│   ├── Cascadia.woff2
│   ├── Assistant-Regular.woff2
│   ├── favicon.svg                         # Icons
│   ├── og-image-3.png                      # Social media image
│   └── ... (more assets)
│
├── 📝 Configuration Files                  # Root config files
│   ├── package.json                        # Root package (monorepo)
│   ├── tsconfig.json                       # TypeScript config
│   ├── vitest.config.mts                   # Test config
│   ├── .eslintrc.json                      # ESLint config
│   ├── .env.production                     # Production env vars
│   ├── .env.development                    # Development env vars
│   └── .husky/                             # Git hooks
│
└── 🔒 .git/                                # Git repository
```

### Package Dependency Graph | 包依赖关系图

```
┌─────────────────────────────────────────────────────────┐
│                   excalidraw-app                        │
│                  (Web Application)                      │
│              uses library as dependency                 │
└──────────────────────┬──────────────────────────────────┘
                       │ depends on
                       ↓
┌─────────────────────────────────────────────────────────┐
│            @excalidraw/excalidraw (Main Library)        │
│                  Component Library                      │
└──┬───────────────┬───────────────┬────────────────┬─────┘
   │               │               │                │
   │ uses          │ uses          │ uses           │ uses
   ↓               ↓               ↓                ↓
┌────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐
│  @common   │ │ @element │ │  @math   │ │    @utils    │
│ Constants  │ │  Element │ │   Math   │ │   Utilities  │
│  Utilities │ │  Logic   │ │Operations│ │Export/BBox  │
└────────────┘ └──────────┘ └──────────┘ └──────────────┘
```

---

## 🎯 Project Architecture Patterns | 项目架构模式

### 1. Monorepo Architecture | Monorepo 架构

**English:**
- Uses **Yarn Workspaces** to manage multiple packages in a single repository
- Shared dependencies hoisted to root `node_modules`
- Independent versioning for published packages
- Path aliases for cross-package imports

**中文:**
- 使用 **Yarn Workspaces** 在单个仓库中管理多个包
- 共享依赖提升到根目录 `node_modules`
- 发布的包独立版本控制
- 跨包导入使用路径别名

**Workspace Configuration** (`package.json:5-9`):
```json
{
  "workspaces": [
    "excalidraw-app",
    "packages/*",
    "examples/*"
  ]
}
```

### 2. State Management Pattern | 状态管理模式

**English:**
Uses **Jotai** for atomic state management, avoiding complex Redux boilerplate. State is split into:
- **Component-local atoms**: UI state within components
- **Global atoms**: Shared application state
- **Derived atoms**: Computed values based on other atoms

**中文:**
使用 **Jotai** 进行原子化状态管理，避免复杂的 Redux 样板代码。状态分为：
- **组件本地原子**: 组件内的 UI 状态
- **全局原子**: 共享的应用程序状态
- **派生原子**: 基于其他原子计算的值

**Implementation** (`packages/excalidraw/editor-jotai.ts`):
```typescript
import { createStore } from 'jotai'
export const editorJotaiStore = createStore()
```

### 3. Rendering Architecture | 渲染架构

**English:**
Multi-layered canvas rendering system for performance:

**中文:**
多层画布渲染系统以提升性能：

```
┌─────────────────────────────────────────────────┐
│         LayerUI (React Components)              │  ← Toolbar, menus, dialogs
├─────────────────────────────────────────────────┤
│      InteractiveCanvas (User Interaction)       │  ← Mouse/touch events
├─────────────────────────────────────────────────┤
│       StaticCanvas (Scene Rendering)            │  ← Main scene with elements
├─────────────────────────────────────────────────┤
│      NewElementCanvas (Preview Layer)           │  ← Element being created
└─────────────────────────────────────────────────┘

Rendering Flow:
User Input → App.tsx (event handling) → Renderer.ts → staticScene.ts/interactiveScene.ts
                                                    → renderElement.ts → Canvas API + RoughJS
```

### 4. Event Handling System | 事件处理系统

**English:**
Custom event emitter pattern for decoupled communication between components.

**中文:**
自定义事件发射器模式，用于组件间的解耦通信。

**Implementation** (`packages/common/src/emitter.ts`):
```typescript
export class Emitter<T> {
  on(event: string, handler: (data: T) => void): void
  off(event: string, handler: (data: T) => void): void
  trigger(event: string, data: T): void
}
```

### 5. Module Pattern | 模块模式

**Package Exports** (`packages/excalidraw/package.json:8-33`):
```json
{
  "exports": {
    ".": {
      "types": "./dist/types/excalidraw/index.d.ts",
      "development": "./dist/dev/index.js",
      "production": "./dist/prod/index.js",
      "default": "./dist/prod/index.js"
    },
    "./index.css": {
      "development": "./dist/dev/index.css",
      "production": "./dist/prod/index.css"
    }
  }
}
```

**English:**
- Supports tree-shaking via ESM
- Separate dev/prod builds
- Conditional exports for different environments

**中文:**
- 通过 ESM 支持 tree-shaking
- 分离开发/生产构建
- 针对不同环境的条件导出

---

## 🔑 Key Configuration Files | 关键配置文件

### Build Configuration | 构建配置

| File | Purpose (EN) | 用途 (中文) | Key Features |
|------|--------------|------------|--------------|
| `vitest.config.mts` | Test configuration | 测试配置 | jsdom, coverage thresholds, mocks |
| `excalidraw-app/vite.config.mts` | App build config | 应用构建配置 | PWA, React, SVGR, sitemap, font optimization |
| `scripts/buildPackage.js` | Package build | 包构建 | esbuild, dev/prod, SASS compilation |
| `tsconfig.json` | TypeScript config | TypeScript 配置 | Strict mode, path aliases, ESNext |

**TypeScript Path Aliases** (`tsconfig.json:21-32`):
```json
{
  "paths": {
    "@excalidraw/common": ["./packages/common/src/index.ts"],
    "@excalidraw/common/*": ["./packages/common/src/*"],
    "@excalidraw/excalidraw": ["./packages/excalidraw/index.tsx"],
    "@excalidraw/excalidraw/*": ["./packages/excalidraw/*"],
    "@excalidraw/element": ["./packages/element/src/index.ts"],
    "@excalidraw/element/*": ["./packages/element/src/*"],
    "@excalidraw/math": ["./packages/math/src/index.ts"],
    "@excalidraw/math/*": ["./packages/math/src/*"]
  }
}
```

### Code Quality | 代码质量

| File | Purpose (EN) | 用途 (中文) | Rules |
|------|--------------|------------|-------|
| `.eslintrc.json` | ESLint config | ESLint 配置 | React best practices, import rules |
| `prettier.json` | Code formatting | 代码格式化 | Via `@excalidraw/prettier-config` |
| `.husky/pre-commit` | Git pre-commit hook | Git 提交前钩子 | Runs lint-staged |

### Development Tools | 开发工具

| File | Purpose (EN) | 用途 (中文) |
|------|--------------|------------|
| `.husky/` | Git hooks directory | Git 钩子目录 |
| `.vscode/settings.json` (if exists) | VS Code config | VS Code 配置 |
| `.github/workflows/` (if exists) | CI/CD pipelines | CI/CD 流水线 |

---

## 🌍 Environment Variables | 环境变量清单

### Production Environment | 生产环境 (`.env.production`)

| Variable | Value | Purpose (EN) | 用途 (中文) |
|----------|-------|--------------|------------|
| `MODE` | `"production"` | Environment mode | 环境模式 |
| `VITE_APP_BACKEND_V2_GET_URL` | `https://json.excalidraw.com/api/v2/` | Scene GET API | 场景获取 API |
| `VITE_APP_BACKEND_V2_POST_URL` | `https://json.excalidraw.com/api/v2/post/` | Scene POST API | 场景发布 API |
| `VITE_APP_LIBRARY_URL` | `https://libraries.excalidraw.com` | Library repository | 库仓库 |
| `VITE_APP_LIBRARY_BACKEND` | Firebase Functions URL | Library backend | 库后端 |
| `VITE_APP_WS_SERVER_URL` | `https://oss-collab.excalidraw.com` | 🔄 Collaboration WebSocket | 协作 WebSocket |
| `VITE_APP_AI_BACKEND` | `https://oss-ai.excalidraw.com` | 🤖 AI features backend | AI 功能后端 |
| `VITE_APP_PLUS_LP` | `https://plus.excalidraw.com` | Excalidraw+ landing | Excalidraw+ 落地页 |
| `VITE_APP_PLUS_APP` | `https://app.excalidraw.com` | Excalidraw+ app | Excalidraw+ 应用 |
| `VITE_APP_FIREBASE_CONFIG` | Firebase config JSON | 🔥 Firebase configuration | Firebase 配置 |
| `VITE_APP_ENABLE_TRACKING` | `false` | Analytics tracking | 分析追踪 |
| `VITE_APP_PLUS_EXPORT_PUBLIC_KEY` | RSA public key | Export encryption | 导出加密 |

### Development Environment | 开发环境 (`.env.development`)

| Variable | Value | Purpose (EN) | 用途 (中文) |
|----------|-------|--------------|------------|
| `MODE` | `"development"` | Environment mode | 环境模式 |
| `VITE_APP_BACKEND_V2_GET_URL` | `https://json-dev.excalidraw.com/api/v2/` | Dev scene GET API | 开发场景 API |
| `VITE_APP_WS_SERVER_URL` | `http://localhost:3002` | Local collab server | 本地协作服务器 |
| `VITE_APP_AI_BACKEND` | `http://localhost:3015` | Local AI server | 本地 AI 服务器 |
| `VITE_APP_PLUS_APP` | `http://localhost:3000` | Local Excalidraw+ | 本地 Excalidraw+ |
| `VITE_APP_PORT` | `3000` | Dev server port | 开发服务器端口 |
| `VITE_APP_ENABLE_TRACKING` | `true` | Enable analytics | 启用分析 |
| `VITE_APP_ENABLE_ESLINT` | `true` | Enable ESLint in dev | 开发时启用 ESLint |
| `VITE_APP_ENABLE_PWA` | `false` | Enable PWA in dev | 开发时启用 PWA |
| `VITE_APP_COLLAPSE_OVERLAY` | `true` | Collapse error overlay | 折叠错误覆盖层 |

**English:** Environment variables with `VITE_APP_` prefix are injected at build time and accessible via `import.meta.env.VITE_APP_*` in the code.

**中文:** 带有 `VITE_APP_` 前缀的环境变量在构建时注入，可通过代码中的 `import.meta.env.VITE_APP_*` 访问。

---

## 🚀 Entry Points & Initialization | 入口点和初始化

### Library Entry Point | 库入口点

**File:** `packages/excalidraw/index.tsx:1-310`

**Component Hierarchy:**
```
ExcalidrawBase (Props validation, polyfills)
  └─ EditorJotaiProvider (Jotai state store)
      └─ InitializeApp (Theme & i18n setup)
          └─ App (Main canvas and logic)
```

**Key Exports:**
```typescript
// Main component
export const Excalidraw = React.memo(ExcalidrawBase, areEqual)

// Sub-components (customizable)
export { Sidebar, MainMenu, WelcomeScreen, Footer }

// Utility functions
export { exportToCanvas, exportToBlob, serializeAsJSON, restore }

// Constants
export { FONT_FAMILY, THEME, MIME_TYPES }
```

### Application Entry Point | 应用入口点

**File:** `excalidraw-app/index.tsx:1-17`

**Bootstrap Sequence:**
```
1. HTML loads (index.html) with dark mode initialization
2. Register service worker (PWA) via vite-plugin-pwa
3. Create React root with StrictMode
4. Mount <ExcalidrawApp />
5. Initialize Sentry error tracking
```

**Main App Component:** `excalidraw-app/App.tsx:1-1183`
- Wraps library component with app-specific features
- Adds Firebase collaboration
- Implements local storage persistence
- Provides custom UI components

---

## 🎨 Core Features Summary | 核心功能摘要

### Element Types | 元素类型 (13 types)

1. **Basic Shapes** - Rectangle, Diamond, Ellipse
2. **Lines & Arrows** - Line, Arrow, Elbow Arrow
3. **Freehand** - Pen drawing with pressure sensitivity
4. **Text** - Multi-font text with WYSIWYG editing
5. **Images** - Embedded images with resize
6. **Frames** - Container elements for grouping
7. **Embeddables** - YouTube, websites, etc.

### Drawing Features | 绘图功能

- ✏️ Hand-drawn aesthetic (RoughJS)
- 🎨 Color picker with opacity
- 📏 Precise alignment and distribution
- 🔄 Rotation and flipping
- 📐 Grid and snap-to-object
- ↩️ Undo/redo with history
- 📋 Copy/paste with formatting

### Collaboration Features | 协作功能

- 🔄 Real-time multiplayer (WebSocket)
- 🔐 End-to-end encryption
- 👥 User presence and cursors
- 💾 Auto-save to Firebase
- 🔀 Conflict resolution (operational transformation)

### AI Features | AI 功能

- 🤖 Text-to-Diagram (Mermaid support)
- 📊 Diagram-to-Code conversion
- 🎯 Smart frames with auto-layout

### Export & Share | 导出和分享

- 🖼️ PNG export with background
- 📄 SVG export (scalable)
- 📋 Clipboard (native format)
- 🔗 Shareable links
- 💾 .excalidraw JSON format

---

## 📊 Project Statistics | 项目统计

| Metric | Count | Details |
|--------|-------|---------|
| **Total Source Files** | 565 | TypeScript/JavaScript files |
| **Main App Lines** | 11,508 | `packages/excalidraw/components/App.tsx` |
| **Core Packages** | 5 | excalidraw, common, element, math, utils |
| **Action Handlers** | ~45 | In `packages/excalidraw/actions/` |
| **Supported Languages** | 60+ | In `packages/excalidraw/locales/` |
| **Font Families** | 10+ | Hand-drawn, monospace, sans-serif, etc. |
| **React Components** | 100+ | Across all packages |
| **Test Files** | 50+ | Unit and integration tests |

---

## 🔧 Build System Details | 构建系统详情

### Package Build (esbuild) | 包构建

**Script:** `scripts/buildPackage.js`

**Process:**
```
Source (TS/TSX/SCSS)
      ↓
esbuild bundling
      ↓
├── Development Build
│   ├── Sourcemaps: Yes
│   ├── Minification: No
│   └── Output: dist/dev/
│
└── Production Build
    ├── Sourcemaps: Yes
    ├── Minification: Yes
    └── Output: dist/prod/
      ↓
TypeScript Type Generation (tsc)
      ↓
Output: dist/types/
```

### App Build (Vite) | 应用构建

**Config:** `excalidraw-app/vite.config.mts`

**Plugins:**
- ⚛️ `@vitejs/plugin-react` - React Fast Refresh
- 🎨 `vite-plugin-svgr` - SVG as React components
- 📱 `vite-plugin-pwa` - Progressive Web App
- 🗺️ `vite-plugin-sitemap` - Sitemap generation
- 🔍 `vite-plugin-checker` - TypeScript + ESLint checking
- 📝 `vite-plugin-ejs` - Template variables

**Optimizations:**
- Code splitting (locales, chunks)
- Font optimization (WOFF2)
- Image optimization (Pica)
- Tree-shaking (ESM)
- CSS extraction

---

## 📝 Next Steps | 下一步

**For Developers:**
1. Read `01-database-analysis.md` (State management deep dive)
2. Read `02-backend-analysis.md` (API and collaboration architecture)
3. Read `03-frontend-analysis.md` (Component architecture)

**For Integrators:**
1. See `/examples/with-nextjs/` for Next.js integration
2. See `/examples/with-script-in-browser/` for vanilla JS
3. Read library API documentation

**For Contributors:**
1. Run `yarn install` to set up workspace
2. Run `yarn test:typecheck` to verify setup
3. Run `yarn start` to launch dev server
4. Read `/dev-docs/` for contribution guidelines

---

## 📚 Document References | 文档引用

**Related Documents in this Series:**
- `01-database-analysis.md` - State and data management (frontend-focused)
- `02-backend-analysis.md` - API endpoints and collaboration backend
- `03-frontend-analysis.md` - Component architecture and rendering
- `04-security-analysis.md` - Security patterns and encryption
- `05-deployment-analysis.md` - Build and deployment configuration
- `prompts-generated/` - Step-by-step reconstruction prompts

**External Resources:**
- Repository: https://github.com/excalidraw/excalidraw
- Documentation: https://docs.excalidraw.com
- Library Docs: https://github.com/excalidraw/excalidraw/tree/master/packages/excalidraw
- Live App: https://excalidraw.com

---

**Analysis Completed:** Phase 1 ✅
**Next Phase:** State Management Analysis (adapted for frontend-only architecture)
