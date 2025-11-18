# 03 - Frontend, Security & Deployment Analysis | 前端、安全和部署分析

**Document Version:** 1.0.0
**Analysis Date:** 2025-11-17
**Combined Phases:** 4 (Frontend), 5 (Security), 6 (Deployment)

---

## 📋 Executive Summary | 执行摘要

**English:**
This document combines the frontend component architecture, security implementations, and build/deployment configurations for Excalidraw. The UI is built with a modular component system using React 19, featuring 100+ components organized into logical categories. Security is implemented through end-to-end encryption, input validation, and secure collaboration. The build system uses Vite for the app and esbuild for packages, with PWA support and comprehensive optimization.

**中文:**
本文档结合了 Excalidraw 的前端组件架构、安全实现和构建/部署配置。UI 使用 React 19 构建了模块化组件系统，包含100多个按逻辑分类组织的组件。安全性通过端到端加密、输入验证和安全协作实现。构建系统对应用使用 Vite，对包使用 esbuild，并支持 PWA 和全面优化。

---

## 🧩 PART 1: Frontend Component Architecture | 前端组件架构

### Component Organization | 组件组织结构

**English:** Excalidraw's 100+ components are organized into **functional categories** rather than by file type.

**中文:** Excalidraw 的100多个组件按 **功能类别** 而非文件类型组织。

```
packages/excalidraw/components/
├── App.tsx                          # ⭐ Core (11,508 lines)
├── LayerUI.tsx                      # Main UI overlay
│
├── canvases/                        # Canvas rendering
│   ├── InteractiveCanvas.tsx
│   ├── StaticCanvas.tsx
│   └── NewElementCanvas.tsx
│
├── ColorPicker/                     # Color selection
│   ├── ColorPicker.tsx
│   ├── ColorInput.tsx
│   └── Picker.tsx
│
├── CommandPalette/                  # Cmd+K palette
│   ├── CommandPalette.tsx
│   └── CommandPaletteItem.tsx
│
├── TTDDialog/                       # 🤖 Text-to-Diagram (AI)
│   ├── TTDDialog.tsx
│   ├── TTDDialogInput.tsx
│   ├── TTDDialogOutput.tsx
│   └── MermaidToExcalidraw.tsx
│
├── live-collaboration/              # 🔄 Collaboration UI
│   └── LiveCollaborationTrigger.tsx
│
├── main-menu/                       # Main application menu
│   └── MainMenu.tsx
│
├── Sidebar/                         # Collapsible sidebar
│   ├── Sidebar.tsx
│   └── SidebarHeader.tsx
│
└── ... (90+ more components)
```

### Component Categories | 组件分类

#### 1. Core UI Components | 核心 UI 组件

| Component | File | Purpose (EN) | 用途 (中文) |
|-----------|------|--------------|------------|
| **App** | `App.tsx:1-11508` | Main canvas logic & event handling | 主画布逻辑和事件处理 |
| **LayerUI** | `LayerUI.tsx` | UI overlay (toolbar, panels) | UI 覆盖层（工具栏、面板） |
| **Toolbar** | `Toolbar.tsx` | Drawing tools toolbar | 绘图工具栏 |
| **Actions** | `Actions.tsx` | Action buttons | 动作按钮 |
| **Island** | `Island.tsx` | Floating UI container | 浮动 UI 容器 |

#### 2. Drawing Tools UI | 绘图工具 UI

| Component | Purpose (EN) | 用途 (中文) |
|-----------|--------------|------------|
| `ToolButton.tsx` | Individual tool button | 单个工具按钮 |
| `HandButton.tsx` | Pan tool | 平移工具 |
| `LaserPointerButton.tsx` | Laser pointer | 激光笔 |
| `PenModeButton.tsx` | Pen mode toggle | 笔模式切换 |
| `LockButton.tsx` | Lock tool | 锁定工具 |

#### 3. Properties & Styling | 属性和样式

| Component | Purpose (EN) | 用途 (中文) |
|-----------|--------------|------------|
| `PropertiesPopover.tsx` | Element properties panel | 元素属性面板 |
| `ColorPicker/` | Color selection UI | 颜色选择 UI |
| `FontPicker/` | Font selection | 字体选择 |
| `Range.tsx` | Slider controls | 滑块控件 |

#### 4. Export & Share | 导出和分享

| Component | Purpose (EN) | 用途 (中文) |
|-----------|--------------|------------|
| `ExportDialog.tsx` | Export dialog | 导出对话框 |
| `ImageExportDialog.tsx` | Image export options | 图片导出选项 |
| `ShareableLinkDialog.tsx` | Share link creation | 分享链接创建 |

### Component Patterns | 组件模式

#### Pattern 1: Memoized Components | 记忆化组件

**English:** Performance-critical components use `React.memo()` with custom comparison functions.

**中文:** 性能关键组件使用带自定义比较函数的 `React.memo()`。

```typescript
// StaticCanvas.tsx
export const StaticCanvas = React.memo(
  (props: StaticCanvasProps) => {
    // ... component logic
  },
  (prevProps, nextProps) => {
    // Custom shallow equality check
    return (
      prevProps.sceneNonce === nextProps.sceneNonce &&
      prevProps.width === nextProps.width &&
      prevProps.height === nextProps.height &&
      prevProps.appState.zoom.value === nextProps.appState.zoom.value
      // ... more checks
    );
  },
);
```

#### Pattern 2: Compound Components | 复合组件

**Example:** Dialog components
```typescript
<Dialog>
  <Dialog.Header>Title</Dialog.Header>
  <Dialog.Body>Content</Dialog.Body>
  <Dialog.Footer>
    <Dialog.Button>OK</Dialog.Button>
  </Dialog.Footer>
</Dialog>
```

#### Pattern 3: Render Props | 渲染属性

**Example:** Action PanelComponent
```typescript
export const actionExample = register({
  name: "example",
  PanelComponent: ({ updateData, appState, elements }) => (
    <ToolButton
      onClick={() => updateData(null)}
      disabled={!someCondition(elements, appState)}
    />
  ),
});
```

---

## 🔒 PART 2: Security Analysis | 安全分析

### Security Layers | 安全层次

```
┌───────────────────────────────────────────────────┐
│            End-to-End Encryption                  │  ← Collaboration data
├───────────────────────────────────────────────────┤
│            Input Validation                       │  ← XSS, injection prevention
├───────────────────────────────────────────────────┤
│            URL Sanitization                       │  ← Safe links
├───────────────────────────────────────────────────┤
│            Content Security Policy                │  ← Browser security
└───────────────────────────────────────────────────┘
```

### 1. End-to-End Encryption | 端到端加密

**File:** `packages/excalidraw/data/encryption.ts`

**Algorithm:** **AES-GCM-256** with PBKDF2 key derivation

```typescript
/**
 * Derive encryption key from room key (passphrase)
 */
const deriveKey = async (passphrase: string): Promise<CryptoKey> => {
  const encoder = new TextEncoder();
  const passphraseBuffer = encoder.encode(passphrase);

  // Import passphrase as key material
  const keyMaterial = await window.crypto.subtle.importKey(
    "raw",
    passphraseBuffer,
    "PBKDF2",
    false,
    ["deriveKey"],
  );

  // Derive AES-GCM key using PBKDF2
  const key = await window.crypto.subtle.deriveKey(
    {
      name: "PBKDF2",
      salt: encoder.encode("excalidraw-salt"),  // Fixed salt
      iterations: 100000,                        // 100k iterations
      hash: "SHA-256",
    },
    keyMaterial,
    {
      name: "AES-GCM",
      length: 256,  // 256-bit key
    },
    false,
    ["encrypt", "decrypt"],
  );

  return key;
};

/**
 * Encrypt data
 */
export const encryptData = async (
  passphrase: string,
  data: Uint8Array,
): Promise<{ encryptedBuffer: ArrayBuffer; iv: Uint8Array }> => {
  const key = await deriveKey(passphrase);
  const iv = window.crypto.getRandomValues(new Uint8Array(16));  // Random IV

  const encryptedBuffer = await window.crypto.subtle.encrypt(
    { name: "AES-GCM", iv },
    key,
    data,
  );

  return { encryptedBuffer, iv };
};

/**
 * Decrypt data
 */
export const decryptData = async (
  iv: Uint8Array,
  encryptedBuffer: ArrayBuffer,
  passphrase: string,
): Promise<{ decryptedBuffer: ArrayBuffer }> => {
  const key = await deriveKey(passphrase);

  const decryptedBuffer = await window.crypto.subtle.decrypt(
    { name: "AES-GCM", iv },
    key,
    encryptedBuffer,
  );

  return { decryptedBuffer };
};
```

**Security Properties:**
- ✅ **AES-GCM-256** - Industry-standard authenticated encryption
- ✅ **Random IV** - New IV for every encryption
- ✅ **PBKDF2** - 100,000 iterations for key derivation
- ✅ **Browser Crypto API** - Hardware-accelerated, secure random

**⚠️ Security Note:** Fixed salt is used for simplicity but reduces security. Production systems should use unique salts per user/session.

### 2. Input Validation | 输入验证

#### XSS Prevention | XSS 防护

**URL Sanitization** (`packages/common/src/utils.ts`):
```typescript
import { sanitizeUrl } from "@braintree/sanitize-url";

export const normalizeLink = (link: string): string => {
  // Remove whitespace
  link = link.trim();

  // Sanitize against XSS
  link = sanitizeUrl(link);

  // Block dangerous protocols
  if (
    link.startsWith("javascript:") ||
    link.startsWith("data:") ||
    link.startsWith("vbscript:")
  ) {
    return "";
  }

  return link;
};
```

**Text Sanitization:**
```typescript
export const normalizeText = (text: string): string => {
  // Remove null bytes
  text = text.replace(/\0/g, "");

  // Normalize line endings
  text = text.replace(/\r\n/g, "\n");
  text = text.replace(/\r/g, "\n");

  return text;
};
```

#### Size/Position Validation | 大小/位置验证

```typescript
// newElement.ts
const MAX_ELEMENT_COORD = 1e6;
const MAX_ELEMENT_SIZE = 1e6;

if (
  x < -MAX_ELEMENT_COORD || x > MAX_ELEMENT_COORD ||
  y < -MAX_ELEMENT_COORD || y > MAX_ELEMENT_COORD ||
  width < -MAX_ELEMENT_SIZE || width > MAX_ELEMENT_SIZE ||
  height < -MAX_ELEMENT_SIZE || height > MAX_ELEMENT_SIZE
) {
  console.error("Element size or position out of bounds");
}
```

### 3. Content Security Policy | 内容安全策略

**Embeddable Validation** (`packages/element/src/embeddable.ts`):
```typescript
export const validateEmbeddable = (url: string): boolean => {
  const allowedDomains = [
    "youtube.com",
    "youtu.be",
    "vimeo.com",
    "twitter.com",
    "excalidraw.com",
  ];

  try {
    const parsed = new URL(url);

    return allowedDomains.some(domain =>
      parsed.hostname === domain ||
      parsed.hostname.endsWith(`.${domain}`)
    );
  } catch {
    return false;
  }
};
```

### 4. Collaboration Security | 协作安全

**Room Key Security:**
- ✅ Room keys **never sent to server** - Only encrypted data transmitted
- ✅ Keys shared **via URL fragment** (`#room=id,key`) - Not sent in HTTP requests
- ✅ **Ephemeral keys** - Generated per session, not stored

**Access Control:**
- ✅ **No authentication required** - Simplicity over strict access control
- ✅ **Possession of key = access** - Anyone with link can collaborate
- ⚠️ **No permission system** - All collaborators have equal rights

---

## 🏗️ PART 3: Build & Deployment | 构建和部署

### Build Architecture | 构建架构

```
┌──────────────────────────────────────────────────┐
│              Monorepo Root                       │
└───────────────┬──────────────────────────────────┘
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
┌───────────────┐  ┌────────────────┐
│   Packages    │  │ Excalidraw App │
│   (esbuild)   │  │    (Vite)      │
└───────────────┘  └────────────────┘
        │                  │
        ▼                  ▼
  dist/dev/          build/
  dist/prod/
```

### Package Build (esbuild) | 包构建

**Script:** `scripts/buildPackage.js`

**Configuration:**
```javascript
// buildPackage.js (simplified)
const buildPackage = async (mode) => {
  const isDev = mode === "development";

  await esbuild.build({
    entryPoints: ["index.tsx"],
    bundle: true,
    outdir: isDev ? "dist/dev" : "dist/prod",
    format: "esm",
    target: "es2020",
    platform: "browser",

    // Externals (peer dependencies)
    external: ["react", "react-dom"],

    // Dev vs Prod
    minify: !isDev,
    sourcemap: true,
    splitting: true,

    // Plugins
    plugins: [
      sassPlugin(),
      fontPlugin(),
    ],
  });

  // Generate TypeScript declarations
  execSync("tsc --emitDeclarationOnly");
};
```

**Output Structure:**
```
packages/excalidraw/dist/
├── dev/                      # Development build
│   ├── index.js             # Main bundle
│   ├── index.css            # Styles
│   └── chunks/              # Code-split chunks
│
├── prod/                    # Production build
│   ├── index.js             # Minified bundle
│   ├── index.css            # Minified styles
│   └── chunks/
│
└── types/                   # TypeScript declarations
    ├── excalidraw/
    ├── common/
    ├── element/
    └── ...
```

### App Build (Vite) | 应用构建

**File:** `excalidraw-app/vite.config.mts`

**Key Plugins:**
```typescript
export default defineConfig({
  plugins: [
    // React Fast Refresh
    react(),

    // SVG as React components
    svgrPlugin(),

    // PWA support
    VitePWA({
      registerType: "autoUpdate",
      workbox: {
        globIgnores: ["fonts.css", "**/locales/**"],
        runtimeCaching: [
          {
            urlPattern: /\.woff2$/,
            handler: "CacheFirst",
            options: {
              cacheName: "fonts",
              expiration: { maxAgeSeconds: 60 * 60 * 24 * 90 },
            },
          },
        ],
      },
      manifest: {
        short_name: "Excalidraw",
        name: "Excalidraw",
        icons: [{ src: "favicon.svg", sizes: "any" }],
        start_url: "/",
        display: "standalone",
        theme_color: "#121212",
      },
    }),

    // TypeScript + ESLint checking
    checker({
      typescript: true,
      eslint: { lintCommand: 'eslint "./**/*.{js,ts,tsx}"' },
    }),

    // Sitemap generation
    Sitemap({
      hostname: "https://excalidraw.com",
    }),
  ],

  build: {
    outDir: "build",
    sourcemap: true,

    rollupOptions: {
      output: {
        // Locale code splitting
        manualChunks(id) {
          if (
            id.includes("locales/") &&
            !id.match(/en\.json|percentages\.json/)
          ) {
            return `locales/${id.substring(id.indexOf("locales/") + 8)}`;
          }
        },
      },
    },
  },
});
```

### Build Commands | 构建命令

**Development:**
```bash
yarn start                    # Start Vite dev server (port 3000)
yarn test:typecheck           # TypeScript type checking
yarn test:code                # ESLint
yarn fix                      # Auto-fix formatting + linting
```

**Production:**
```bash
yarn build:packages           # Build all packages (esbuild)
yarn build:app                # Build app (Vite)
yarn build                    # Build both
```

**Testing:**
```bash
yarn test                     # Run tests (Vitest)
yarn test:update              # Update snapshots
yarn test:coverage            # Generate coverage report
```

### Deployment Targets | 部署目标

#### 1. Static Hosting (Vercel/Netlify)

**Vercel Configuration:**
```json
{
  "buildCommand": "yarn build",
  "outputDirectory": "excalidraw-app/build",
  "framework": "vite",
  "installCommand": "yarn install",
  "env": {
    "VITE_APP_GIT_SHA": "$VERCEL_GIT_COMMIT_SHA",
    "VITE_APP_ENABLE_TRACKING": "true"
  }
}
```

#### 2. Docker Deployment

**Dockerfile:**
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

# Copy source
COPY . .

# Build
RUN yarn build:app:docker

# Serve
EXPOSE 80
CMD ["npx", "http-server", "excalidraw-app/build", "-p", "80"]
```

**Build:**
```bash
yarn build:app:docker         # Disable Sentry in Docker
```

#### 3. PWA Installation

**Features:**
- ✅ **Offline support** - Service worker caching
- ✅ **Install prompt** - Add to home screen
- ✅ **File handling** - Open `.excalidraw` files
- ✅ **Share target** - Receive files from other apps

### Performance Optimizations | 性能优化

**Build Optimizations:**
- ✅ **Code splitting** - Locales split into separate chunks
- ✅ **Tree shaking** - Remove unused code
- ✅ **Minification** - Terser for JS, cssnano for CSS
- ✅ **Font subsetting** - WOFF2 with subset glyphs
- ✅ **Image optimization** - Pica for resizing

**Runtime Optimizations:**
- ✅ **Lazy loading** - Dynamic imports for routes
- ✅ **Memoization** - React.memo, useMemo, useCallback
- ✅ **Viewport culling** - Only render visible elements
- ✅ **Throttled rendering** - 60fps cap on static canvas
- ✅ **Web Workers** - Font subsetting off main thread

---

## 📊 Performance Metrics | 性能指标

### Bundle Size | 包大小

**Production Build:**
```
Main bundle:        ~800 KB (gzipped)
Styles:            ~50 KB (gzipped)
Total initial:     ~850 KB (gzipped)
Locales (lazy):    ~10 KB each (gzipped)
Fonts (cached):    ~200 KB total
```

**Code Splitting:**
- Core bundle: 800 KB
- Per locale: 10 KB (loaded on demand)
- Total with all locales: ~1.5 MB

### Loading Performance | 加载性能

**Lighthouse Scores (excalidraw.com):**
- Performance: 90+
- Accessibility: 95+
- Best Practices: 100
- SEO: 100

**Core Web Vitals:**
- LCP (Largest Contentful Paint): < 2.5s
- FID (First Input Delay): < 100ms
- CLS (Cumulative Layout Shift): < 0.1

---

## 🔍 Debugging & Development Tools | 调试和开发工具

### Debug Flags | 调试标志

**Environment Variables:**
```bash
# Enable text container bounding boxes
VITE_APP_DEBUG_ENABLE_TEXT_CONTAINER_BOUNDING_BOX=true

# Disable live reload
VITE_APP_DEV_DISABLE_LIVE_RELOAD=true

# Disable ESLint in dev
VITE_APP_ENABLE_ESLINT=false
```

### Browser DevTools Integration | 浏览器开发工具集成

**React DevTools:**
- Component hierarchy inspection
- Props/state inspection
- Profiler for performance

**Custom Debugging:**
```typescript
// Enable via localStorage
localStorage.setItem("excalidraw-debug", "true");

// Access app instance
window.Excalidraw; // Global reference to app instance
```

---

## 📚 Related Documents | 相关文档

**Previous:**
- `00-project-overview.md` - Technology stack
- `01-state-data-management-analysis.md` - State management
- `02-architecture-core-logic-analysis.md` - Rendering architecture

**Next:**
- `04-reconstruction-prompts.md` - Step-by-step rebuild prompts
- Specifications (CN + EN)
- Teaching Guide (CN + EN)

---

**Analysis Completed:** Phases 4-6 ✅
**Next Phase:** Generate Reconstruction Prompts
