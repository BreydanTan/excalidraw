# 02 - Architecture & Core Logic Analysis | 架构和核心逻辑分析

**Document Version:** 1.0.0
**Analysis Date:** 2025-11-17
**Related Documents:** `00-project-overview.md`, `01-state-data-management-analysis.md`

---

## 📋 Executive Summary | 执行摘要

**English:**
Excalidraw's architecture is built on a sophisticated multi-layer canvas rendering system, an immutable state pattern with version tracking, and a plugin-based action system. The core architecture emphasizes performance through viewport culling, memoization, and intelligent caching strategies. The rendering pipeline separates static scene rendering from interactive overlays, enabling smooth 60fps performance even with hundreds of elements.

**中文:**
Excalidraw 的架构建立在复杂的多层画布渲染系统、带版本跟踪的不可变状态模式和基于插件的动作系统之上。核心架构通过视口裁剪、记忆化和智能缓存策略强调性能。渲染管道将静态场景渲染与交互式覆盖层分离，即使有数百个元素也能实现流畅的 60fps 性能。

---

## 🎨 Triple-Canvas Rendering Architecture | 三层画布渲染架构

### Overview | 概述

**English:** Excalidraw uses **three separate HTML canvas elements** layered on top of each other to optimize rendering performance.

**中文:** Excalidraw 使用 **三个独立的 HTML canvas 元素** 层叠在一起以优化渲染性能。

```
┌─────────────────────────────────────────────────┐
│        Interactive Canvas (Top Layer)          │  ← Selection UI, handles, cursors
│          - Always rendering (RAF loop)          │
│          - Transparent background               │
├─────────────────────────────────────────────────┤
│      New Element Canvas (Middle Layer)         │  ← Preview of element being created
│          - Renders on-demand                    │
├─────────────────────────────────────────────────┤
│         Static Canvas (Base Layer)             │  ← All finalized elements
│          - Throttled rendering (60fps)          │
│          - Main scene with RoughJS              │
└─────────────────────────────────────────────────┘
```

---

### 1. Static Canvas | 静态画布

**File:** `packages/excalidraw/components/canvases/StaticCanvas.tsx`

**Purpose (EN):** Renders all finalized elements with hand-drawn aesthetic using RoughJS.

**用途 (中文):** 使用 RoughJS 渲染所有已完成的元素，呈现手绘风格。

#### Component Structure | 组件结构

```typescript
// StaticCanvas.tsx:20-80
export const StaticCanvas = React.memo(
  (props: StaticCanvasProps) => {
    const canvas = useRef<HTMLCanvasElement>(null);

    useEffect(() => {
      const canvasElement = canvas.current;
      if (!canvasElement) return;

      // Render static scene with throttling
      renderStaticSceneThrottled(
        {
          canvas: canvasElement,
          rc: props.rc,                    // RoughJS renderer
          scale: props.scale,              // Device pixel ratio
          elementsMap: props.elementsMap,  // Renderable elements map
          allElementsMap: props.allElementsMap,
          visibleElements: props.visibleElements,
          appState: props.appState,
          renderConfig: props.renderConfig,
        },
        isRenderThrottlingEnabled(),
      );
    });

    return (
      <canvas
        className="excalidraw__canvas static"
        ref={canvas}
        width={props.width}
        height={props.height}
      />
    );
  },
  // Memo comparison - only re-render on actual changes
  (prevProps, nextProps) => {
    return (
      prevProps.sceneNonce === nextProps.sceneNonce &&
      prevProps.width === nextProps.width &&
      prevProps.height === nextProps.height &&
      prevProps.appState.zoom === nextProps.appState.zoom &&
      prevProps.appState.scrollX === nextProps.appState.scrollX &&
      prevProps.appState.scrollY === nextProps.appState.scrollY
      // ... more comparison checks
    );
  },
);
```

#### Rendering Pipeline | 渲染管道

**File:** `packages/excalidraw/renderer/staticScene.ts:40-200`

```typescript
export const renderStaticScene = (
  renderConfig: StaticSceneRenderConfig,
  throttle: boolean,
) => {
  const {
    canvas,
    rc,
    scale,
    elementsMap,
    visibleElements,
    appState,
  } = renderConfig;

  const context = canvas.getContext("2d")!;

  // 1. Clear canvas
  context.clearRect(0, 0, canvas.width, canvas.height);

  // 2. Save context state
  context.save();

  // 3. Apply transformations
  context.scale(scale * appState.zoom.value, scale * appState.zoom.value);
  context.translate(
    appState.scrollX * scale,
    appState.scrollY * scale,
  );

  // 4. Render grid (if enabled)
  if (appState.gridModeEnabled) {
    renderGrid(context, appState, scale);
  }

  // 5. Render visible elements
  for (const element of visibleElements) {
    try {
      renderElement(element, elementsMap, allElementsMap, rc, context, appState);
    } catch (error) {
      console.error("Failed to render element", element, error);
    }
  }

  // 6. Render bound text
  for (const element of visibleElements) {
    if (hasBoundTextElement(element)) {
      const textElement = getBoundTextElement(element, elementsMap);
      if (textElement) {
        renderElement(textElement, elementsMap, allElementsMap, rc, context, appState);
      }
    }
  }

  // 7. Render link icons
  renderLinkIcons(visibleElements, context, appState, elementsMap);

  // 8. Restore context
  context.restore();
};

// Throttled version - limits to 60fps
export const renderStaticSceneThrottled = throttleRAF(
  renderStaticScene,
  { trailing: true },
);
```

**Key Optimizations:**
- ✅ **Throttled to 60fps** - Prevents excessive rendering
- ✅ **Viewport culling** - Only renders visible elements
- ✅ **Memoized component** - Only re-renders on actual state changes

---

### 2. Interactive Canvas | 交互式画布

**File:** `packages/excalidraw/components/canvases/InteractiveCanvas.tsx`

**Purpose (EN):** Renders interactive UI overlays: selection boxes, transform handles, cursors, and editing controls.

**用途 (中文):** 渲染交互式 UI 覆盖层：选择框、变换手柄、光标和编辑控件。

#### Animation Loop | 动画循环

```typescript
// InteractiveCanvas.tsx:169-220
useEffect(() => {
  const canvas = canvasRef.current;
  if (!canvas) return;

  // Start animation loop if not already running
  if (!AnimationController.running(INTERACTIVE_SCENE_ANIMATION_KEY)) {
    AnimationController.start<InteractiveSceneRenderAnimationState>(
      INTERACTIVE_SCENE_ANIMATION_KEY,
      ({ deltaTime, state }) => {
        // Render interactive scene
        const nextAnimationState = renderInteractiveScene({
          canvas,
          elementsMap: renderParams.current!.elementsMap,
          visibleElements: renderParams.current!.visibleElements,
          selectedElements: renderParams.current!.selectedElements,
          appState: renderParams.current!.appState,
          scale: renderParams.current!.scale,
          renderConfig: renderParams.current!.renderConfig,
          deltaTime,
          animationState: state,
        }).animationState;

        // Return state to continue, or undefined to stop
        return nextAnimationState;
      },
    );
  }

  return () => {
    // Cleanup animation loop on unmount
    AnimationController.stop(INTERACTIVE_SCENE_ANIMATION_KEY);
  };
}, []);
```

#### Animation Controller | 动画控制器

**File:** `packages/excalidraw/renderer/animation.ts:40-90`

```typescript
export class AnimationController {
  private static animations = new Map<string, AnimationData>();

  static start<T>(key: string, animation: AnimationFn<T>) {
    if (AnimationController.animations.has(key)) {
      console.warn(`Animation "${key}" already running`);
      return;
    }

    AnimationController.animations.set(key, {
      animation,
      state: undefined,
      lastTime: 0,
    });

    // Start tick loop if first animation
    if (AnimationController.animations.size === 1) {
      AnimationController.tick();
    }
  }

  private static tick() {
    if (AnimationController.animations.size === 0) {
      return; // All animations stopped
    }

    for (const [key, data] of AnimationController.animations) {
      const now = performance.now();
      const deltaTime = data.lastTime === 0 ? 0 : now - data.lastTime;

      // Execute animation frame
      const nextState = data.animation({
        deltaTime,
        state: data.state,
      });

      if (!nextState) {
        // Animation finished
        AnimationController.animations.delete(key);
      } else {
        // Update state for next frame
        data.lastTime = now;
        data.state = nextState;
      }
    }

    // Schedule next tick
    if (isRenderThrottlingEnabled()) {
      requestAnimationFrame(AnimationController.tick);
    } else {
      setTimeout(AnimationController.tick, 0); // Immediate (no throttle)
    }
  }

  static stop(key: string) {
    AnimationController.animations.delete(key);
  }

  static running(key: string): boolean {
    return AnimationController.animations.has(key);
  }
}
```

**English:** The animation loop runs continuously via `requestAnimationFrame`, rendering selection UI, handles, and cursors at maximum frame rate.

**中文:** 动画循环通过 `requestAnimationFrame` 持续运行，以最大帧率渲染选择 UI、手柄和光标。

---

### 3. Renderer Class | 渲染器类

**File:** `packages/excalidraw/scene/Renderer.ts:20-200`

**Purpose (EN):** Central coordinator for rendering - handles viewport culling and element filtering.

**用途 (中文):** 渲染的中央协调器 - 处理视口裁剪和元素过滤。

```typescript
export class Renderer {
  private scene: Scene;

  constructor(scene: Scene) {
    this.scene = scene;
  }

  /**
   * Get elements that should be rendered (memoized)
   */
  public getRenderableElements = memoize(
    ({
      zoom,
      offsetLeft,
      offsetTop,
      scrollX,
      scrollY,
      height,
      width,
      editingTextElement,
      newElementId,
      sceneNonce, // Cache key
    }: RenderableElementsParams) => {
      const elements = this.scene.getNonDeletedElements();

      // Filter out editing text element and new element (rendered separately)
      const elementsMap = getRenderableElements({
        elements,
        editingTextElement,
        newElementId,
      });

      // Viewport culling - only return visible elements
      const visibleElements = getVisibleCanvasElements({
        elementsMap,
        zoom,
        offsetLeft,
        offsetTop,
        scrollX,
        scrollY,
        height,
        width,
      });

      return {
        elementsMap,
        visibleElements,
      };
    },
    // Cache key - only recalculate if these change
    (params) => [
      params.sceneNonce,
      params.zoom.value,
      params.scrollX,
      params.scrollY,
      params.width,
      params.height,
      params.editingTextElement?.id,
      params.newElementId,
    ].join(","),
  );
}
```

**Viewport Culling Algorithm** (`packages/excalidraw/scene/export.ts:350-420`):

```typescript
const getVisibleCanvasElements = ({
  elementsMap,
  zoom,
  offsetLeft,
  offsetTop,
  scrollX,
  scrollY,
  height,
  width,
}: ViewportParams): readonly NonDeletedExcalidrawElement[] => {
  // Calculate viewport bounds in scene coordinates
  const viewportBounds: Bounds = [
    -offsetLeft / zoom.value + scrollX, // minX
    -offsetTop / zoom.value + scrollY,  // minY
    (-offsetLeft + width) / zoom.value + scrollX,  // maxX
    (-offsetTop + height) / zoom.value + scrollY,  // maxY
  ];

  const visibleElements: NonDeletedExcalidrawElement[] = [];

  for (const element of elementsMap.values()) {
    const elementBounds = getElementBounds(element, elementsMap);

    // Check if element bounds intersect viewport
    if (
      elementBounds[2] >= viewportBounds[0] &&  // element maxX >= viewport minX
      elementBounds[0] <= viewportBounds[2] &&  // element minX <= viewport maxX
      elementBounds[3] >= viewportBounds[1] &&  // element maxY >= viewport minY
      elementBounds[1] <= viewportBounds[3]     // element minY <= viewport maxY
    ) {
      visibleElements.push(element);
    }
  }

  return visibleElements;
};
```

**English:** This optimization prevents rendering elements outside the viewport, dramatically improving performance with large scenes.

**中文:** 此优化防止渲染视口外的元素，显著提高大型场景的性能。

---

## 🔧 Element Creation & Manipulation | 元素创建和操作

### Element Factory Pattern | 元素工厂模式

**File:** `packages/element/src/newElement.ts:78-250`

#### Base Element Factory | 基础元素工厂

```typescript
const _newElementBase = <T extends ExcalidrawElement>(
  type: T["type"],
  {
    x = 0,
    y = 0,
    strokeColor = DEFAULT_ELEMENT_PROPS.strokeColor,
    backgroundColor = DEFAULT_ELEMENT_PROPS.backgroundColor,
    fillStyle = DEFAULT_ELEMENT_PROPS.fillStyle,
    strokeWidth = DEFAULT_ELEMENT_PROPS.strokeWidth,
    strokeStyle = DEFAULT_ELEMENT_PROPS.strokeStyle,
    roughness = DEFAULT_ELEMENT_PROPS.roughness,
    opacity = DEFAULT_ELEMENT_PROPS.opacity,
    width = 0,
    height = 0,
    angle = 0 as Radians,
    groupIds = [],
    frameId = null,
    index = null,
    roundness = null,
    boundElements = null,
    link = null,
    locked = DEFAULT_ELEMENT_PROPS.locked,
    ...rest
  }: ElementConstructorOpts,
) => {
  // Validate size/position (prevent extreme values)
  if (
    x < -MAX_ELEMENT_COORD || x > MAX_ELEMENT_COORD ||
    y < -MAX_ELEMENT_COORD || y > MAX_ELEMENT_COORD ||
    width < -MAX_ELEMENT_SIZE || width > MAX_ELEMENT_SIZE ||
    height < -MAX_ELEMENT_SIZE || height > MAX_ELEMENT_SIZE
  ) {
    console.error("Element size or position out of bounds");
  }

  return {
    id: rest.id || nanoid(),
    type,
    x,
    y,
    width,
    height,
    angle,
    strokeColor,
    backgroundColor,
    fillStyle,
    strokeWidth,
    strokeStyle,
    roughness,
    opacity,
    groupIds: makeArrayReadonly(groupIds),
    frameId,
    index,
    roundness,
    seed: rest.seed ?? randomInteger(),
    version: rest.version || 1,
    versionNonce: rest.versionNonce ?? 0,
    isDeleted: false,
    boundElements,
    updated: getUpdatedTimestamp(),
    link,
    locked,
    customData: rest.customData,
  } as T;
};
```

#### Specialized Factories | 专用工厂

**Rectangle/Ellipse/Diamond:**
```typescript
export const newElement = (
  opts: {
    type: "rectangle" | "ellipse" | "diamond";
  } & ElementConstructorOpts,
): NonDeleted<ExcalidrawRectangleElement | ExcalidrawEllipseElement | ExcalidrawDiamondElement> => {
  return _newElementBase(opts.type, opts);
};
```

**Text Element:**
```typescript
// newElement.ts:360-470
export const newTextElement = (
  opts: {
    text: string;
    fontSize?: number;
    fontFamily?: FontFamilyValues;
    textAlign?: TextAlign;
    verticalAlign?: VerticalAlign;
    containerId?: ExcalidrawTextContainer["id"] | null;
    lineHeight?: number & { _brand: "unitlessLineHeight" };
    autoResize?: boolean;
  } & ElementConstructorOpts,
): NonDeleted<ExcalidrawTextElement> => {
  const textElement = newElementWith(
    {
      ..._newElementBase("text", opts),
      fontSize: opts.fontSize || DEFAULT_FONT_SIZE,
      fontFamily: opts.fontFamily || DEFAULT_FONT_FAMILY,
      text: opts.text,
      textAlign: opts.textAlign || DEFAULT_TEXT_ALIGN,
      verticalAlign: opts.verticalAlign || DEFAULT_VERTICAL_ALIGN,
      containerId: opts.containerId || null,
      originalText: opts.text,
      autoResize: opts.autoResize ?? true,
      lineHeight: opts.lineHeight || getDefaultLineHeight(opts.fontFamily || DEFAULT_FONT_FAMILY),
    },
    {},
  );

  // Measure text and set dimensions
  const { width, height, text } = refreshTextDimensions(
    textElement,
    getContainerElement(textElement, elementsMap),
    elementsMap,
  );

  return { ...textElement, width, height, text };
};
```

**Linear Element (Lines & Arrows):**
```typescript
// newElement.ts:520-600
export const newLinearElement = (
  opts: {
    type: "line" | "arrow";
    startBinding?: PointBinding | null;
    endBinding?: PointBinding | null;
    points?: readonly LocalPoint[];
  } & ElementConstructorOpts,
): NonDeleted<ExcalidrawLinearElement> => {
  return {
    ..._newElementBase(opts.type, opts),
    points: opts.points || [
      pointFrom(0, 0),
      pointFrom(0, 0),
    ],
    lastCommittedPoint: null,
    startBinding: opts.startBinding || null,
    endBinding: opts.endBinding || null,
    startArrowhead: null,
    endArrowhead: opts.type === "arrow" ? "arrow" : null,
  };
};
```

---

### Element Mutation System | 元素变更系统

**File:** `packages/element/src/mutateElement.ts:37-200`

**English:** Elements are **immutable** - mutations create new versions with incremented version numbers for collaboration tracking.

**中文:** 元素是 **不可变的** - 变更会创建带递增版本号的新版本以进行协作跟踪。

```typescript
export const mutateElement = <TElement extends Mutable<ExcalidrawElement>>(
  element: TElement,
  elementsMap: ElementsMap,
  updates: ElementUpdate<TElement>,
  options?: { isDragging?: boolean },
): TElement => {
  let didChange = false;

  // Special handling for elbow arrows
  if (
    isElbowArrow(element) &&
    (typeof updates.points !== "undefined" ||
      typeof updates.fixedSegments !== "undefined")
  ) {
    updates = {
      ...updates,
      angle: 0 as Radians,
      ...updateElbowArrowPoints(element, elementsMap, updates, options),
    };
  }

  // Apply all updates
  for (const key in updates) {
    const value = updates[key];

    // Deep equality check to avoid unnecessary updates
    if (element[key] !== value) {
      (element as any)[key] = value;
      didChange = true;
    }
  }

  if (!didChange) {
    return element; // No changes, return as-is
  }

  // Invalidate shape cache if geometry changed
  if (
    typeof updates.height !== "undefined" ||
    typeof updates.width !== "undefined" ||
    typeof updates.points !== "undefined" ||
    typeof updates.scale !== "undefined"
  ) {
    ShapeCache.delete(element);
  }

  // Invalidate bounds cache if needed
  if (
    typeof updates.x !== "undefined" ||
    typeof updates.y !== "undefined" ||
    typeof updates.width !== "undefined" ||
    typeof updates.height !== "undefined" ||
    typeof updates.angle !== "undefined" ||
    typeof updates.points !== "undefined"
  ) {
    ElementBounds.invalidate(element);
  }

  // Bump version for collaboration
  element.version = updates.version ?? element.version + 1;
  element.versionNonce = updates.versionNonce ?? randomInteger();
  element.updated = getUpdatedTimestamp();

  return element;
};
```

**Key Features:**
- ✅ **Immutability** - Original element unchanged
- ✅ **Version tracking** - Collaboration-friendly
- ✅ **Cache invalidation** - Smart cache management
- ✅ **Deep equality** - Avoid unnecessary updates

---

### Bounds Calculation & Caching | 边界计算和缓存

**File:** `packages/element/src/bounds.ts:100-250`

```typescript
export class ElementBounds {
  // Cache: element → { version, bounds }
  private static boundsCache = new WeakMap<
    ExcalidrawElement,
    { version: number; bounds: Bounds }
  >();

  private static nonRotatedBoundsCache = new WeakMap<
    ExcalidrawElement,
    { version: number; bounds: Bounds }
  >();

  /**
   * Get element bounds (cached)
   */
  static getBounds(
    element: ExcalidrawElement,
    elementsMap: ElementsMap,
    nonRotated: boolean = false,
  ): Bounds {
    const cache = nonRotated
      ? ElementBounds.nonRotatedBoundsCache
      : ElementBounds.boundsCache;

    const cached = cache.get(element);

    // Cache hit: version matches and not bound to container
    if (
      cached?.version === element.version &&
      !isBoundToContainer(element)
    ) {
      return cached.bounds;
    }

    // Cache miss: calculate bounds
    const bounds = ElementBounds.calculateBounds(element, elementsMap, nonRotated);

    // Cache result
    cache.set(element, {
      version: element.version,
      bounds,
    });

    return bounds;
  }

  /**
   * Calculate actual bounds based on element type
   */
  private static calculateBounds(
    element: ExcalidrawElement,
    elementsMap: ElementsMap,
    nonRotated: boolean,
  ): Bounds {
    if (isLinearElement(element)) {
      return getLinearElementBounds(element, elementsMap);
    }

    if (isFreeDrawElement(element)) {
      return getFreeDrawBounds(element);
    }

    if (isTextElement(element)) {
      const container = getContainerElement(element, elementsMap);
      if (container) {
        return getBoundTextElementBounds(container, element);
      }
    }

    // Standard rectangular bounds
    let [x1, y1, x2, y2, cx, cy] = getElementAbsoluteCoords(element, elementsMap);

    if (nonRotated || element.angle === 0) {
      return [x1, y1, x2, y2];
    }

    // Rotated bounds - calculate rotated corners
    const corners = [
      pointFrom(x1, y1),
      pointFrom(x2, y1),
      pointFrom(x2, y2),
      pointFrom(x1, y2),
    ].map((point) => pointRotateRads(point, pointFrom(cx, cy), element.angle));

    const xs = corners.map((p) => p[0]);
    const ys = corners.map((p) => p[1]);

    return [Math.min(...xs), Math.min(...ys), Math.max(...xs), Math.max(...ys)];
  }

  /**
   * Invalidate cache for element
   */
  static invalidate(element: ExcalidrawElement) {
    ElementBounds.boundsCache.delete(element);
    ElementBounds.nonRotatedBoundsCache.delete(element);
  }
}
```

**English:** Bounds are cached using WeakMap (automatic garbage collection) and invalidated when element version changes.

**中文:** 边界使用 WeakMap 缓存（自动垃圾回收），并在元素版本更改时失效。

---

## ⚡ Action System | 动作系统

### Action Manager | 动作管理器

**File:** `packages/excalidraw/actions/manager.tsx:40-250`

```typescript
export class ActionManager {
  actions = {} as Record<ActionName, Action>;
  updater: (actionResult: ActionResult | Promise<ActionResult>) => void;
  getAppState: () => Readonly<AppState>;
  getElementsIncludingDeleted: () => readonly OrderedExcalidrawElement[];
  app: AppClassProperties;

  constructor(
    updater: (actionResult: ActionResult | Promise<ActionResult>) => void,
    getAppState: () => Readonly<AppState>,
    getElementsIncludingDeleted: () => readonly OrderedExcalidrawElement[],
    app: AppClassProperties,
  ) {
    this.updater = (actionResult) => {
      if (isPromiseLike(actionResult)) {
        actionResult.then((result) => updater(result));
      } else {
        updater(actionResult);
      }
    };
    this.getAppState = getAppState;
    this.getElementsIncludingDeleted = getElementsIncludingDeleted;
    this.app = app;
  }

  /**
   * Register an action
   */
  registerAction(action: Action) {
    this.actions[action.name] = action;
  }

  /**
   * Execute action by name
   */
  executeAction(
    actionName: ActionName,
    formData?: any,
    source?: ActionSource,
  ) {
    const action = this.actions[actionName];
    if (!action) {
      console.error(`Action "${actionName}" not found`);
      return;
    }

    const elements = this.getElementsIncludingDeleted();
    const appState = this.getAppState();

    // Check predicate (action available?)
    if (action.predicate && !action.predicate(elements, appState, null, this.app)) {
      return;
    }

    // Track analytics
    trackAction(action, source || "ui", appState, elements, this.app, formData);

    // Execute action
    const result = action.perform(elements, appState, formData, this.app);

    // Update app state
    this.updater(result);
  }

  /**
   * Handle keyboard shortcuts
   */
  handleKeyDown(event: React.KeyboardEvent | KeyboardEvent): boolean {
    const elements = this.getElementsIncludingDeleted();
    const appState = this.getAppState();

    // Find matching actions (sorted by priority)
    const matchingActions = Object.values(this.actions)
      .filter(
        (action) =>
          action.keyTest &&
          action.keyTest(event, appState, elements, this.app),
      )
      .sort((a, b) => (b.keyPriority || 0) - (a.keyPriority || 0));

    if (matchingActions.length !== 1) {
      if (matchingActions.length > 1) {
        console.warn(
          "Multiple actions match this shortcut:",
          matchingActions.map((a) => a.name),
        );
      }
      return false; // No match or ambiguous
    }

    const action = matchingActions[0];

    // Check predicate
    if (action.predicate && !action.predicate(elements, appState, null, this.app)) {
      return false;
    }

    // Execute action
    event.preventDefault();
    event.stopPropagation();

    trackAction(action, "keyboard", appState, elements, this.app, null);
    this.updater(action.perform(elements, appState, null, this.app));

    return true;
  }
}
```

---

### Action Structure | 动作结构

**File:** `packages/excalidraw/actions/types.ts:160-250`

```typescript
export interface Action {
  name: ActionName;                    // Unique identifier
  label: string | LabelFn;             // Display label (can be dynamic)
  keywords?: string[];                 // Search keywords (for command palette)
  icon?: ReactNode;                    // Toolbar icon
  viewMode?: boolean;                  // Available in view mode?
  trackEvent: false | TrackEventData;  // Analytics tracking

  /**
   * Main action logic
   */
  perform: (
    elements: readonly OrderedExcalidrawElement[],
    appState: Readonly<AppState>,
    formData: any,
    app: AppClassProperties,
  ) => ActionResult | Promise<ActionResult>;

  /**
   * Keyboard shortcut test
   */
  keyTest?: (
    event: React.KeyboardEvent | KeyboardEvent,
    appState: AppState,
    elements: readonly OrderedExcalidrawElement[],
    app: AppClassProperties,
  ) => boolean;

  /**
   * Keyboard shortcut priority (higher = checked first)
   */
  keyPriority?: number;

  /**
   * Action availability predicate
   */
  predicate?: (
    elements: readonly OrderedExcalidrawElement[],
    appState: AppState,
    appProps: AppProps,
    app: AppClassProperties,
  ) => boolean;

  /**
   * Checked state (for toggle buttons)
   */
  checked?: (appState: AppState) => boolean;

  /**
   * Optional UI component for toolbar/properties panel
   */
  PanelComponent?: React.FC<PanelComponentProps>;
}

/**
 * Action result - what changed?
 */
export type ActionResult = {
  elements?: readonly OrderedExcalidrawElement[] | null;
  appState?: Partial<AppState> | null;
  files?: BinaryFiles | null;
  captureUpdate?: CaptureUpdateAction;  // History capture strategy
  replaceFiles?: boolean;
};
```

---

### Example: Align Action | 示例：对齐动作

**File:** `packages/excalidraw/actions/actionAlign.tsx:70-150`

```typescript
export const actionAlignTop = register({
  name: "alignTop",
  label: "labels.alignTop",
  icon: AlignTopIcon,
  trackEvent: { category: "element" },

  predicate: (elements, appState, appProps, app) => {
    // Only available when multiple elements selected
    const selectedElements = app.scene.getSelectedElements(appState);
    return selectedElements.length > 1;
  },

  perform: (elements, appState, _, app) => {
    return {
      appState,
      elements: alignSelectedElements(elements, appState, app, {
        position: "start",  // Align to top
        axis: "y",          // Vertical axis
      }),
      captureUpdate: CaptureUpdateAction.IMMEDIATELY,
    };
  },

  keyTest: (event) =>
    event[KEYS.CTRL_OR_CMD] &&
    event.shiftKey &&
    event.key === KEYS.ARROW_UP,

  PanelComponent: ({ updateData, app }) => {
    const selectedElements = app.scene.getSelectedElements(app.state);
    const disabled = selectedElements.length < 2;

    return (
      <ToolButton
        icon={AlignTopIcon}
        onClick={() => updateData(null)}
        title={`${t("labels.alignTop")} — ${getShortcutKey("CtrlOrCmd+Shift+Up")}`}
        disabled={disabled}
        aria-label={t("labels.alignTop")}
      />
    );
  },
});
```

**Align Algorithm** (`packages/excalidraw/actions/actionAlign.tsx:200-300`):

```typescript
const alignSelectedElements = (
  elements: readonly OrderedExcalidrawElement[],
  appState: AppState,
  app: AppClassProperties,
  { position, axis }: { position: "start" | "center" | "end"; axis: "x" | "y" },
): readonly OrderedExcalidrawElement[] => {
  const selectedElements = app.scene.getSelectedElements(appState);

  // Get common bounds of all selected elements
  const [minX, minY, maxX, maxY] = getCommonBounds(selectedElements);

  // Calculate alignment target
  let targetValue: number;
  if (axis === "x") {
    if (position === "start") targetValue = minX;
    else if (position === "center") targetValue = (minX + maxX) / 2;
    else targetValue = maxX;
  } else {
    if (position === "start") targetValue = minY;
    else if (position === "center") targetValue = (minY + maxY) / 2;
    else targetValue = maxY;
  }

  // Apply alignment to each element
  return elements.map((element) => {
    if (!selectedElements.includes(element)) {
      return element;
    }

    const [x1, y1, x2, y2] = getElementAbsoluteCoords(element, elementsMap);
    let offsetX = 0;
    let offsetY = 0;

    if (axis === "x") {
      if (position === "start") offsetX = targetValue - x1;
      else if (position === "center") offsetX = targetValue - (x1 + x2) / 2;
      else offsetX = targetValue - x2;
    } else {
      if (position === "start") offsetY = targetValue - y1;
      else if (position === "center") offsetY = targetValue - (y1 + y2) / 2;
      else offsetY = targetValue - y2;
    }

    return mutateElement(
      element,
      elementsMap,
      {
        x: element.x + offsetX,
        y: element.y + offsetY,
      },
      app.scene,
    );
  });
};
```

---

## 🖱️ Event Handling | 事件处理

### Unified Pointer Events | 统一指针事件

**English:** Excalidraw uses the **Pointer Events API** to handle both mouse and touch input with a single code path.

**中文:** Excalidraw 使用 **指针事件 API** 通过单一代码路径处理鼠标和触摸输入。

**File:** `packages/excalidraw/components/App.tsx:5800-6500`

#### Main Event Handlers | 主要事件处理器

```typescript
class App extends React.Component<AppProps, AppState> {
  /**
   * Pointer down - start interaction
   */
  private handleCanvasPointerDown = (
    event: React.PointerEvent<HTMLCanvasElement>,
  ) => {
    // Get pointer coordinates in scene space
    const { x: sceneX, y: sceneY } = viewportCoordsToSceneCoords(
      { clientX: event.clientX, clientY: event.clientY },
      this.state,
    );

    const toolType = this.state.activeTool.type;

    // Tool-specific handling
    if (toolType === "selection" || toolType === "lasso") {
      this.handleSelectionPointerDown(event, sceneX, sceneY);
    }
    else if (toolType === "text") {
      this.handleTextPointerDown(event, sceneX, sceneY);
    }
    else if (toolType === "arrow" || toolType === "line") {
      this.handleLinearElementCreation(event, toolType, sceneX, sceneY);
    }
    else if (toolType === "freedraw") {
      this.handleFreeDrawStart(event, sceneX, sceneY);
    }
    else if (toolType === "laser") {
      this.handleLaserPointerDown(event, sceneX, sceneY);
    }
    else if (["rectangle", "diamond", "ellipse"].includes(toolType)) {
      this.handleGenericElementCreation(event, toolType, sceneX, sceneY);
    }
    // ... more tools
  };

  /**
   * Pointer move - update interaction
   */
  private handleCanvasPointerMove = (
    event: React.PointerEvent<HTMLCanvasElement>,
  ) => {
    const { x: sceneX, y: sceneY } = viewportCoordsToSceneCoords(
      { clientX: event.clientX, clientY: event.clientY },
      this.state,
    );

    if (this.state.draggingElement) {
      this.handleElementDrag(event, sceneX, sceneY);
    }
    else if (this.state.resizingElement) {
      this.handleElementResize(event, sceneX, sceneY);
    }
    else if (this.state.selectedLinearElement?.isEditing) {
      LinearElementEditor.handlePointerMove(event, sceneX, sceneY, this);
    }
    // ... more states
  };

  /**
   * Pointer up - finalize interaction
   */
  private handleCanvasPointerUp = (
    event: React.PointerEvent<HTMLCanvasElement>,
  ) => {
    const { x: sceneX, y: sceneY } = viewportCoordsToSceneCoords(
      { clientX: event.clientX, clientY: event.clientY },
      this.state,
    );

    if (this.state.draggingElement) {
      this.finalizeElementDrag(sceneX, sceneY);
    }
    else if (this.state.multiElement) {
      this.finalizeLinearElement(sceneX, sceneY);
    }
    // ... finalization logic
  };
}
```

---

## ✏️ Drawing Tools Implementation | 绘图工具实现

### Linear Element Drawing (Lines & Arrows) | 线性元素绘制

**File:** `packages/element/src/linearElementEditor.ts:1000-1150`

**Drawing Flow:**
```
PointerDown
    ↓
Create linear element with 2 points: [0,0], [0,0]
Set as multiElement (being created)
    ↓
PointerMove
    ↓
Update last point to pointer position
If Alt key: Add intermediate points (multi-segment)
If Shift key: Constrain to 45° angles
    ↓
PointerUp or Enter
    ↓
Finalize element
Clear multiElement state
```

**Point Adding Logic:**
```typescript
// linearElementEditor.ts:1050-1120
static handlePointerMove(
  event: React.PointerEvent,
  scenePointerX: number,
  scenePointerY: number,
  app: AppClassProperties,
): LinearElementEditor | null {
  const { selectedLinearElement } = app.state;
  if (!selectedLinearElement) return null;

  const { elementId, lastUncommittedPoint } = selectedLinearElement;
  const element = LinearElementEditor.getElement(elementId, app.scene.getElementsMapIncludingDeleted());
  const points = element.points;
  const lastCommittedPoint = lastUncommittedPoint || points[points.length - 1];

  // Alt key: multi-segment mode
  if (!event.altKey) {
    // Remove uncommitted point if Alt released
    if (lastUncommittedPoint && points[points.length - 1] === lastUncommittedPoint) {
      LinearElementEditor.deletePoints(element, app, [points.length - 1]);
    }
    return selectedLinearElement;
  }

  let newPoint: LocalPoint;

  // Shift key: constrain angle
  if (shouldRotateWithDiscreteAngle(event) && points.length >= 2) {
    const [width, height] = LinearElementEditor._getShiftLockedDelta(
      element,
      app.scene.getElementsMapIncludingDeleted(),
      lastCommittedPoint,
      pointFrom(scenePointerX, scenePointerY),
      event[KEYS.CTRL_OR_CMD] ? null : app.getEffectiveGridSize(),
    );
    newPoint = pointFrom(
      width + lastCommittedPoint[0],
      height + lastCommittedPoint[1],
    );
  } else {
    // Free point
    newPoint = LinearElementEditor.createPointAt(
      element,
      app.scene.getElementsMapIncludingDeleted(),
      scenePointerX,
      scenePointerY,
      event[KEYS.CTRL_OR_CMD] ? null : app.getEffectiveGridSize(),
    );
  }

  // Update or add point
  if (lastUncommittedPoint && points[points.length - 1] === lastUncommittedPoint) {
    // Update existing uncommitted point
    LinearElementEditor.movePoints(
      element,
      app.scene,
      new Map([[points.length - 1, { point: newPoint }]]),
    );
  } else {
    // Add new uncommitted point
    LinearElementEditor.addPoints(element, app.scene, [newPoint]);
  }

  return {
    ...selectedLinearElement,
    lastUncommittedPoint: element.points[element.points.length - 1],
  };
}
```

---

### Rectangle/Ellipse/Diamond Drawing | 矩形/椭圆/菱形绘制

**Drag-to-Create Pattern:**
```
PointerDown
    ↓
Create element at pointer (width=0, height=0)
Set as draggingElement
    ↓
PointerMove (while dragging)
    ↓
Calculate width = currentX - startX
Calculate height = currentY - startY
If Shift: Lock aspect ratio (width = height)
If Alt: Draw from center (startX - width, startY - height)
Update element dimensions
    ↓
PointerUp
    ↓
Finalize element
Clear draggingElement state
```

---

## 📝 Text Editing (WYSIWYG) | 文本编辑

### WYSIWYG Overlay System | WYSIWYG 覆盖系统

**File:** `packages/excalidraw/wysiwyg/textWysiwyg.tsx:80-350`

**English:** Text editing uses an actual HTML `<textarea>` overlay positioned on the canvas, providing native browser text editing capabilities.

**中文:** 文本编辑使用实际的 HTML `<textarea>` 覆盖在画布上，提供原生浏览器文本编辑功能。

```typescript
export const textWysiwyg = ({
  id,
  onChange,
  onSubmit,
  element,
  canvas,
  app,
  autoSelect = true,
}: WysiwygParams): (() => void) => {
  // Create textarea element
  const editable = document.createElement("textarea");

  editable.dir = "auto";
  editable.tabIndex = 0;
  editable.dataset.type = "wysiwyg";
  editable.wrap = "off"; // Prevent Safari auto-wrapping
  editable.classList.add("excalidraw-wysiwyg");

  // Set initial value
  editable.value = element.originalText;

  // Position and style to match canvas element
  const updateWysiwygStyle = () => {
    const elementsMap = app.scene.getElementsMapIncludingDeleted();
    const updatedElement = app.scene.getElement<ExcalidrawTextElement>(id);
    const container = getContainerElement(updatedElement, elementsMap);

    let coordX = updatedElement.x;
    let coordY = updatedElement.y;
    let width = updatedElement.width;
    let height = updatedElement.height;
    let maxWidth = width;
    let maxHeight = height;

    // Bound text positioning
    if (container && updatedElement.containerId) {
      if (isArrowElement(container)) {
        const boundTextCoords = LinearElementEditor.getBoundTextElementPosition(
          container,
          updatedElement,
          elementsMap,
        );
        coordX = boundTextCoords.x;
        coordY = boundTextCoords.y;
      }

      maxWidth = getBoundTextMaxWidth(container, updatedElement);
      maxHeight = getBoundTextMaxHeight(container, updatedElement);

      // Auto-grow container if text exceeds
      if (!isArrowElement(container) && height > maxHeight) {
        const targetHeight = computeContainerDimensionForBoundText(
          height,
          container.type,
        );
        app.scene.mutateElement(container, { height: targetHeight }, false);
      }
    }

    // Convert scene coords to viewport coords
    const [viewportX, viewportY] = sceneCoordsToViewportCoords(
      pointFrom(coordX, coordY),
      app.state,
    );

    // Apply styles
    Object.assign(editable.style, {
      font: getFontString(updatedElement),
      lineHeight: `${updatedElement.lineHeight}`,
      width: `${width}px`,
      height: `${height}px`,
      left: `${viewportX}px`,
      top: `${viewportY}px`,
      transform: getTransform(
        width,
        height,
        updatedElement.angle,
        app.state,
        maxWidth,
        maxHeight,
      ),
      textAlign: updatedElement.textAlign,
      verticalAlign: updatedElement.verticalAlign,
      color: updatedElement.strokeColor,
      opacity: (updatedElement.opacity / 100).toString(),
      filter: "var(--appearance-filter)",
    });
  };

  updateWysiwygStyle();

  // Text input handler
  editable.oninput = () => {
    const normalized = normalizeText(editable.value);
    onChange(normalized);
    updateWysiwygStyle(); // Update size on input
  };

  // Keyboard shortcuts
  editable.onkeydown = (event) => {
    if (event.key === KEYS.ESCAPE) {
      event.preventDefault();
      handleSubmit();
    } else if (event.key === KEYS.ENTER && event[KEYS.CTRL_OR_CMD]) {
      event.preventDefault();
      handleSubmit();
    } else if (event.key === KEYS.TAB) {
      event.preventDefault();
      if (event.shiftKey) {
        outdent(editable);
      } else {
        indent(editable);
      }
    }
  };

  // Submit on blur
  editable.onblur = handleSubmit;

  // Attach to DOM
  canvas.parentElement!.appendChild(editable);

  // Auto-select text
  if (autoSelect) {
    editable.select();
  }

  editable.focus();

  // Cleanup function
  const handleSubmit = () => {
    editable.remove();
    onSubmit(normalizeText(editable.value));
  };

  return handleSubmit;
};
```

**Key Features:**
- ✅ **Native editing** - Browser handles text input, cursor, selection
- ✅ **CSS transform** - Matches canvas rotation and zoom
- ✅ **Auto-resize** - Container grows with text
- ✅ **Tab support** - Custom tab/shift-tab indentation

---

## 📚 Related Documents | 相关文档

**Previous:**
- `00-project-overview.md` - Project structure and tech stack
- `01-state-data-management-analysis.md` - State management and persistence

**Next:**
- `03-frontend-analysis.md` - Component architecture and UI
- `04-security-analysis.md` - Security patterns and encryption

**Key Files Referenced:**
- `packages/excalidraw/scene/Renderer.ts` - Rendering coordinator
- `packages/excalidraw/renderer/staticScene.ts` - Static canvas rendering
- `packages/excalidraw/renderer/interactiveScene.ts` - Interactive canvas rendering
- `packages/element/src/newElement.ts` - Element creation
- `packages/element/src/mutateElement.ts` - Element mutation
- `packages/element/src/bounds.ts` - Bounds calculation
- `packages/excalidraw/actions/manager.tsx` - Action system
- `packages/element/src/linearElementEditor.ts` - Linear element editing
- `packages/excalidraw/wysiwyg/textWysiwyg.tsx` - Text editing

---

**Analysis Completed:** Phase 3 ✅
**Next Phase:** Frontend Component and Interaction Analysis
