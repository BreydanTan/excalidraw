# Excalidraw 项目规格文档

**版本:** 1.0.0
**日期:** 2025-11-17
**项目:** Excalidraw v0.18.0

---

## 目录

1. [项目概述](#项目概述)
2. [技术架构](#技术架构)
3. [核心功能](#核心功能)
4. [数据模型](#数据模型)
5. [API接口](#api接口)
6. [部署指南](#部署指南)
7. [开发指南](#开发指南)

---

## 项目概述

### 项目简介

Excalidraw 是一个**开源虚拟白板应用**，让您轻松绘制手绘风格的图表。它由两部分组成：

1. **@excalidraw/excalidraw** - 可复用的 React 组件库（发布到 npm）
2. **excalidraw.com** - 功能完整的 Web 应用

**核心特点:**
- ✏️ 手绘风格的视觉效果
- 🎨 简单直观的界面
- 🔄 实时协作（带端到端加密）
- 📱 PWA 支持（可离线使用）
- 🌍 支持 60+ 种语言
- 🤖 AI 功能（文本转图表）

### 适用场景

- 🎯 **绘制流程图** - 业务流程、系统架构
- 📊 **制作图表** - 思维导图、组织结构图
- 🎨 **快速草图** - 设计稿、线框图
- 📝 **技术说明** - 系统设计、API 文档
- 👥 **团队协作** - 实时协同绘图

### 技术亮点

| 特性 | 实现方式 | 优势 |
|------|---------|------|
| **手绘风格** | RoughJS 库 | 自然、友好的视觉效果 |
| **高性能渲染** | 三层画布系统 + 视口裁剪 | 流畅 60fps，支持大型场景 |
| **实时协作** | WebSocket + 端到端加密 | 安全的多人协作 |
| **离线支持** | PWA + Service Worker | 随时随地可用 |
| **组件化设计** | React 19 + TypeScript | 易于集成和扩展 |

---

## 技术架构

### 整体架构图

```
                  用户界面 (React 组件)
                          │
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
   工具栏/菜单        画布渲染           属性面板
         │                │                │
         └────────────────┼────────────────┘
                          │
                    核心逻辑层
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
    元素管理          动作系统          状态管理
  (Scene类)        (ActionManager)      (Jotai)
         │                │                │
         └────────────────┼────────────────┘
                          │
                    数据持久层
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
   localStorage     IndexedDB          Firebase
   (场景数据)        (图片)           (协作)
```

### 技术栈

**前端框架:**
- React 19.0.0
- TypeScript 4.9.4
- Jotai 2.11.0 (状态管理)

**绘图库:**
- RoughJS 4.6.4 (手绘渲染)
- Perfect Freehand 1.2.0 (平滑绘图)

**协作:**
- Socket.IO Client 4.7.2 (WebSocket)
- Firebase 11.3.1 (云存储)

**构建工具:**
- Vite 5.0.12 (应用构建)
- esbuild 0.19.10 (包构建)
- Vitest 3.0.6 (测试)

### 渲染架构

**三层画布系统** (性能优化的核心)：

```
┌────────────────────────────────────┐
│  交互式画布 (InteractiveCanvas)     │  ← 选择框、手柄、光标
│  - 持续渲染 (RAF循环)                │
│  - 透明背景                         │
├────────────────────────────────────┤
│  新元素画布 (NewElementCanvas)      │  ← 正在创建的元素预览
│  - 按需渲染                         │
├────────────────────────────────────┤
│  静态画布 (StaticCanvas)            │  ← 所有已完成的元素
│  - 限流渲染 (60fps)                 │
│  - RoughJS 手绘效果                 │
└────────────────────────────────────┘
```

**为什么使用三层画布?**
- **性能优化** - 分离静态内容和动态交互
- **减少重绘** - 只有必要的层重新渲染
- **流畅体验** - 即使有数百个元素也能保持 60fps

---

## 核心功能

### 1. 绘图工具

| 工具 | 快捷键 | 说明 | 文件位置 |
|------|--------|------|---------|
| **选择** | V 或 1 | 选择和移动元素 | `App.tsx:6457` |
| **矩形** | R 或 2 | 绘制矩形 | `newElement.ts:150` |
| **菱形** | D 或 3 | 绘制菱形 | `newElement.ts:165` |
| **椭圆** | O 或 4 | 绘制椭圆/圆形 | `newElement.ts:180` |
| **箭头** | A 或 5 | 绘制箭头 | `newLinearElement.ts:520` |
| **直线** | L 或 6 | 绘制直线 | `newLinearElement.ts:540` |
| **自由绘制** | P 或 7 | 手绘路径 | `App.tsx:5900` |
| **文本** | T 或 8 | 添加文本 | `newTextElement.ts:360` |
| **图片** | I 或 9 | 插入图片 | `newImageElement.ts:600` |
| **橡皮擦** | E 或 0 | 擦除元素 | `App.tsx:6100` |

### 2. 元素操作

**变换操作:**
- 🔄 旋转 - 拖动旋转手柄
- 📐 调整大小 - 拖动边角手柄
- 📋 复制/粘贴 - Ctrl+C / Ctrl+V
- ♻️ 撤销/重做 - Ctrl+Z / Ctrl+Shift+Z
- 🗑️ 删除 - Delete 或 Backspace

**对齐和分布:**
```
Ctrl+Shift+↑   对齐到顶部
Ctrl+Shift+↓   对齐到底部
Ctrl+Shift+←   对齐到左侧
Ctrl+Shift+→   对齐到右侧
Ctrl+Shift+H   水平居中
Ctrl+Shift+V   垂直居中
```

**组操作:**
- Ctrl+G - 组合元素
- Ctrl+Shift+G - 取消组合
- Ctrl+] - 置于顶层
- Ctrl+[ - 置于底层

### 3. 实时协作

**工作流程:**
```
1. 点击"协作"按钮
2. 生成分享链接（包含房间ID和加密密钥）
3. 分享链接给团队成员
4. 自动同步绘图内容
```

**安全特性:**
- 🔐 端到端加密（AES-GCM-256）
- 🔑 密钥仅在URL片段中（不发送到服务器）
- 🔄 冲突自动解决
- 👥 实时光标显示

**数据流:**
```
用户编辑
    ↓
加密更改 (AES-GCM)
    ↓
WebSocket 发送
    ↓
协作服务器广播
    ↓
其他用户接收
    ↓
解密并应用更改
    ↓
调和冲突
    ↓
更新画布
```

### 4. 导出功能

**支持的格式:**
- 🖼️ PNG - 带/不带背景
- 📄 SVG - 可缩放矢量图
- 📋 剪贴板 - 直接粘贴到其他应用
- 💾 .excalidraw - 原生格式（可重新编辑）

**导出选项:**
- 选择导出范围（所有元素/仅选中）
- 调整缩放比例
- 嵌入场景数据到PNG/SVG（可重新打开）
- 深色/浅色主题

### 5. AI 功能

**文本转图表 (Text-to-Diagram):**
```
输入：创建一个登录流程图
输出：自动生成带有框图和箭头的流程图

支持：
- 自然语言描述
- Mermaid 语法
- 流程图、序列图、类图
```

**文件位置:** `packages/excalidraw/components/TTDDialog/`

---

## 数据模型

### 元素数据结构

**基础元素属性 (所有元素共有):**
```typescript
{
  id: string                    // 唯一标识符 (nanoid)
  type: string                  // 元素类型
  x: number                     // X 坐标
  y: number                     // Y 坐标
  width: number                 // 宽度
  height: number                // 高度
  angle: number                 // 旋转角度（弧度）
  strokeColor: string           // 描边颜色 (hex)
  backgroundColor: string       // 填充颜色 (hex)
  fillStyle: string             // 填充样式 (hachure/solid/cross-hatch/zigzag)
  strokeWidth: number           // 描边宽度 (1/2/4)
  strokeStyle: string           // 描边样式 (solid/dashed/dotted)
  roughness: number             // 粗糙度 (0-2)
  opacity: number               // 不透明度 (0-100)
  seed: number                  // 随机种子（用于确定性渲染）
  version: number               // 版本号（协作用）
  versionNonce: number          // 版本随机数（冲突解决）
  index: string                 // Z轴索引（分数索引）
  isDeleted: boolean            // 软删除标志
  groupIds: string[]            // 所属组ID
  frameId: string | null        // 所属框架ID
  boundElements: object[]       // 绑定的元素（文本/箭头）
  updated: number               // 最后更新时间戳
  link: string | null           // 超链接
  locked: boolean               // 锁定（禁止编辑）
}
```

**矩形元素示例:**
```json
{
  "type": "rectangle",
  "id": "abc123xyz",
  "x": 100,
  "y": 200,
  "width": 300,
  "height": 150,
  "angle": 0,
  "strokeColor": "#000000",
  "backgroundColor": "#ced4da",
  "fillStyle": "hachure",
  "strokeWidth": 1,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "roundness": { "type": 3 },
  "seed": 1234567890,
  "version": 5,
  "versionNonce": 987654321,
  "index": "a0",
  "isDeleted": false,
  "groupIds": [],
  "frameId": null,
  "boundElements": null,
  "updated": 1700000000000,
  "link": null,
  "locked": false
}
```

**文本元素特有属性:**
```typescript
{
  text: string              // 文本内容
  fontSize: number          // 字体大小
  fontFamily: number        // 字体家族 (1-4)
  textAlign: string         // 对齐方式 (left/center/right)
  verticalAlign: string     // 垂直对齐 (top/middle)
  containerId: string       // 容器元素ID（文本框内文本）
  autoResize: boolean       // 自动调整大小
  lineHeight: number        // 行高
}
```

**线性元素（线条/箭头）特有属性:**
```typescript
{
  points: [[x, y], ...]     // 点数组
  startBinding: object      // 起点绑定到的形状
  endBinding: object        // 终点绑定到的形状
  startArrowhead: string    // 起点箭头类型
  endArrowhead: string      // 终点箭头类型
}
```

### 场景文件格式 (.excalidraw)

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [
    /* 元素数组 */
  ],
  "appState": {
    "viewBackgroundColor": "#ffffff",
    "gridSize": null,
    "zoom": { "value": 1 }
    /* 更多UI状态 */
  },
  "files": {
    /* 嵌入的图片（base64） */
  }
}
```

---

## API接口

### 协作 API

**WebSocket 事件:**

| 事件 | 方向 | 数据格式 | 说明 |
|------|------|----------|------|
| `server-broadcast` | 双向 | 加密的元素数据 | 场景更新（可靠传输） |
| `server-volatile-broadcast` | 双向 | 光标位置等 | 易失数据（不保证传输） |

**示例：广播场景更新**
```javascript
// 1. 准备数据
const data = {
  type: "SCENE_UPDATE",
  payload: {
    elements: changedElements
  }
}

// 2. 加密
const json = JSON.stringify(data)
const encoded = new TextEncoder().encode(json)
const { encryptedBuffer, iv } = await encryptData(roomKey, encoded)

// 3. 发送
socket.emit("server-broadcast", roomId, encryptedBuffer, iv)
```

### Firebase API

**保存场景到 Firestore:**
```javascript
const docRef = doc(firestore, "scenes", roomId)

await setDoc(docRef, {
  sceneVersion: 42,
  iv: Array.from(iv),
  ciphertext: Array.from(encryptedData)
})
```

**保存文件到 Storage:**
```javascript
const storageRef = ref(storage, `files/rooms/${roomId}/${fileId}`)

await uploadBytes(storageRef, fileBuffer, {
  cacheControl: "public, max-age=2592000"  // 30天缓存
})
```

---

## 部署指南

### 环境要求

- Node.js >= 18.0.0
- Yarn 1.22.22
- 现代浏览器（支持 Canvas, WebSocket, Crypto API）

### 本地开发

```bash
# 1. 克隆仓库
git clone https://github.com/excalidraw/excalidraw
cd excalidraw

# 2. 安装依赖
yarn install

# 3. 启动开发服务器
yarn start

# 浏览器自动打开 http://localhost:3000
```

### 生产构建

```bash
# 构建所有包
yarn build:packages

# 构建应用
yarn build:app

# 输出目录: excalidraw-app/build/
```

### Docker 部署

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

COPY . .
RUN yarn build:app:docker

EXPOSE 80
CMD ["npx", "http-server", "excalidraw-app/build", "-p", "80"]
```

**构建和运行:**
```bash
docker build -t excalidraw .
docker run -p 80:80 excalidraw
```

### Vercel 部署

**vercel.json:**
```json
{
  "buildCommand": "yarn build",
  "outputDirectory": "excalidraw-app/build",
  "framework": "vite"
}
```

**一键部署:**
```bash
npx vercel
```

---

## 开发指南

### 项目结构

```
excalidraw/
├── packages/              # 核心包
│   ├── excalidraw/       # 主库
│   ├── common/           # 共享工具
│   ├── element/          # 元素逻辑
│   ├── math/             # 数学工具
│   └── utils/            # 通用工具
│
├── excalidraw-app/       # Web 应用
│   ├── index.tsx         # 入口
│   ├── App.tsx           # 主组件
│   ├── collab/           # 协作
│   └── data/             # 数据层
│
├── public/               # 静态资源
└── scripts/              # 构建脚本
```

### 添加新元素类型

**步骤:**
1. 在 `packages/element/src/types.ts` 定义类型
2. 在 `packages/element/src/newElement.ts` 创建工厂函数
3. 在 `packages/excalidraw/renderer/renderElement.ts` 添加渲染逻辑
4. 在 `packages/excalidraw/components/Toolbar.tsx` 添加工具按钮

**示例：添加三角形**
```typescript
// 1. 定义类型
export type ExcalidrawTriangleElement = _ExcalidrawElementBase & {
  type: "triangle"
}

// 2. 工厂函数
export const newTriangleElement = (opts) => {
  return _newElementBase("triangle", opts)
}

// 3. 渲染逻辑
if (element.type === "triangle") {
  const [x1, y1, x2, y2] = getElementAbsoluteCoords(element)
  const path = `M ${x1 + (x2-x1)/2} ${y1} L ${x2} ${y2} L ${x1} ${y2} Z`
  rc.path(path, options)
}
```

### 添加新动作

```typescript
// packages/excalidraw/actions/actionExample.tsx
import { register } from "./register"

export const actionExample = register({
  name: "example",
  label: "示例动作",
  icon: ExampleIcon,

  // 执行逻辑
  perform: (elements, appState, formData, app) => {
    const newElements = elements.map(el => ({
      ...el,
      strokeColor: "#ff0000"  // 所有元素变红
    }))

    return {
      elements: newElements,
      appState,
      captureUpdate: CaptureUpdateAction.IMMEDIATELY,
    }
  },

  // 键盘快捷键
  keyTest: (event) =>
    event.ctrlKey && event.key === "e",

  // UI 组件
  PanelComponent: ({ updateData }) => (
    <ToolButton
      icon={ExampleIcon}
      onClick={() => updateData(null)}
      title="示例 — Ctrl+E"
    />
  ),
})
```

### 调试技巧

**启用调试模式:**
```javascript
// 浏览器控制台
localStorage.setItem("excalidraw-debug", "true")
```

**查看场景状态:**
```javascript
// 访问全局实例
window.Excalidraw.scene.getNonDeletedElements()
```

**性能分析:**
```javascript
// React DevTools Profiler
// 1. 安装 React DevTools 扩展
// 2. 打开 Profiler 标签页
// 3. 开始录制
// 4. 执行操作
// 5. 分析火焰图
```

---

## 常见问题 FAQ

### Q: 如何自定义默认颜色？
A: 修改 `packages/excalidraw/constants.ts` 中的 `DEFAULT_ELEMENT_PROPS`

### Q: 如何禁用某些工具？
A: 使用 `UIOptions` prop:
```tsx
<Excalidraw
  UIOptions={{
    tools: {
      image: false,  // 禁用图片工具
    }
  }}
/>
```

### Q: 如何集成到自己的应用？
A: 安装包并导入组件:
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

### Q: 协作需要自己的服务器吗？
A: 可以使用官方服务器，也可以部署 [excalidraw-room](https://github.com/excalidraw/excalidraw-room) 服务

---

## 附录

### 技术术语表

| 术语 | 解释 |
|------|------|
| **RoughJS** | 手绘风格渲染库 |
| **RAF** | requestAnimationFrame - 浏览器动画 API |
| **Jotai** | 原子化React状态管理库 |
| **PWA** | Progressive Web App - 渐进式 Web 应用 |
| **WebSocket** | 双向实时通信协议 |
| **AES-GCM** | 高级加密标准-伽罗瓦计数器模式 |
| **Viewport Culling** | 视口裁剪 - 只渲染可见元素 |
| **Fractional Indexing** | 分数索引 - 稳定的Z轴排序 |

### 学习资源

- 📖 官方文档: https://docs.excalidraw.com
- 🐙 GitHub仓库: https://github.com/excalidraw/excalidraw
- 💬 Discord社区: https://discord.gg/UexuTaE
- 📺 视频教程: https://www.youtube.com/@excalidraw

### 版本历史

- v0.18.0 (2024) - AI功能、改进协作
- v0.17.0 (2023) - 框架支持、性能优化
- v0.16.0 (2023) - 实时协作稳定版
- v0.15.0 (2022) - PWA支持

---

**文档版本:** 1.0.0
**最后更新:** 2025-11-17
**维护者:** Project Mastery Team
