# 01 - State & Data Management Analysis | 状态和数据管理分析

**Document Version:** 1.0.0
**Analysis Date:** 2025-11-17
**Related Document:** `00-project-overview.md`

---

## 📋 Executive Summary | 执行摘要

**English:**
Excalidraw uses a sophisticated multi-layered state and data management architecture. As a frontend-focused application, it doesn't have a traditional backend database, but instead employs:
- **Jotai** for reactive state management
- **localStorage** for quick persistence
- **IndexedDB** for binary file storage
- **Firebase Firestore** for collaboration data
- **End-to-end encryption** for collaborative sessions

The architecture balances local-first performance with cloud-based collaboration, featuring automatic conflict resolution and offline support.

**中文:**
Excalidraw 使用复杂的多层状态和数据管理架构。作为一个以前端为中心的应用，它没有传统的后端数据库，而是采用：
- **Jotai** 用于响应式状态管理
- **localStorage** 用于快速持久化
- **IndexedDB** 用于二进制文件存储
- **Firebase Firestore** 用于协作数据
- **端到端加密** 用于协作会话

该架构平衡了本地优先性能和基于云的协作，具有自动冲突解决和离线支持功能。

---

## 🏗️ State Management Architecture | 状态管理架构

### 1. Jotai Atomic State Management | Jotai 原子状态管理

**English:** Excalidraw uses **Jotai** for global state management, allowing isolated state scopes to support multiple editor instances on the same page.

**中文:** Excalidraw 使用 **Jotai** 进行全局状态管理，允许隔离的状态作用域以支持同一页面上的多个编辑器实例。

#### Core Setup | 核心设置

**File:** `packages/excalidraw/editor-jotai.ts`
```typescript
import { atom, createStore } from "jotai";
import { createIsolation } from "jotai-scope";

// Create isolated scope for multiple editor instances
const jotai = createIsolation();

export { atom, PrimitiveAtom, WritableAtom };
export const { useAtom, useSetAtom, useAtomValue, useStore } = jotai;
export const EditorJotaiProvider = jotai.Provider;
export const editorJotaiStore = createStore();
```

**File:** `excalidraw-app/app-jotai.ts`
```typescript
import { atom, Provider, createStore } from "jotai";

// App-level Jotai store
export const appJotaiStore = createStore();
export { atom, Provider, useAtom, useAtomValue, useSetAtom };
```

#### Key Global Atoms | 关键全局原子

| Atom | File Location | Purpose (EN) | 用途 (中文) |
|------|--------------|--------------|------------|
| `libraryItemsAtom` | `packages/excalidraw/data/library.ts` | Shape library state | 形状库状态 |
| `localStorageQuotaExceededAtom` | `excalidraw-app/data/LocalData.ts` | Storage quota warning | 存储配额警告 |
| `collabAPIAtom` | `excalidraw-app/collab/Collab.tsx` | Collaboration API instance | 协作 API 实例 |
| `isCollaboratingAtom` | `excalidraw-app/collab/Collab.tsx` | Collaboration status | 协作状态 |
| `isOfflineAtom` | `excalidraw-app/collab/Collab.tsx` | Network status | 网络状态 |
| `activeRoomLinkAtom` | `excalidraw-app/collab/Collab.tsx` | Active room URL | 活动房间 URL |

#### Library State Atom Example | 库状态原子示例

**File:** `packages/excalidraw/data/library.ts:42`
```typescript
export const libraryItemsAtom = atom<{
  status: "loading" | "loaded";      // Loading state
  isInitialized: boolean;             // First load complete?
  libraryItems: LibraryItems;         // Actual library content
}>({
  status: "loaded",
  isInitialized: false,
  libraryItems: []
});
```

#### State Update Pattern | 状态更新模式

**English:** Atoms can be updated imperatively or through React hooks:

**中文:** 原子可以通过命令式或 React 钩子更新：

```typescript
// Imperative updates (outside React components)
const currentValue = appJotaiStore.get(someAtom);
appJotaiStore.set(someAtom, newValue);

// Functional updates
appJotaiStore.set(someAtom, (prev) => ({ ...prev, newField: value }));

// React hook updates (inside components)
const [value, setValue] = useAtom(someAtom);
setValue(newValue);
```

#### State-to-Render Flow | 状态到渲染流程

```
┌─────────────────────┐
│  Action Triggered   │ (e.g., user clicks button)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Update Atom        │ appJotaiStore.set(atom, newValue)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Subscribers        │ Components using useAtom(atom)
│  Notified           │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  React Re-render    │ Only affected components re-render
└─────────────────────┘
```

---

## 💾 Local Storage & Persistence | 本地存储和持久化

### 2. Browser localStorage

**English:** localStorage is used for fast, synchronous persistence of scene data and app preferences.

**中文:** localStorage 用于快速、同步持久化场景数据和应用偏好。

#### Storage Keys | 存储键

**File:** `excalidraw-app/app_constants.ts:17-28`
```typescript
export const STORAGE_KEYS = {
  LOCAL_STORAGE_ELEMENTS: "excalidraw",                // Canvas elements
  LOCAL_STORAGE_APP_STATE: "excalidraw-state",         // App state (UI, viewport)
  LOCAL_STORAGE_COLLAB: "excalidraw-collab",           // Collab username
  LOCAL_STORAGE_THEME: "excalidraw-theme",             // Theme preference
  LOCAL_STORAGE_DEBUG: "excalidraw-debug",             // Debug flags
  VERSION_DATA_STATE: "version-dataState",             // Data version
  VERSION_FILES: "version-files",                      // Files version
  IDB_LIBRARY: "excalidraw-library",                   // Library (in IndexedDB)
  __LEGACY_LOCAL_STORAGE_LIBRARY: "excalidraw-library",// Legacy library key
} as const;
```

**Editor-Specific Keys** (`packages/common/src/constants.ts:88-92`):
```typescript
export const EDITOR_LS_KEYS = {
  OAI_API_KEY: "excalidraw-oai-api-key",                    // OpenAI API key
  MERMAID_TO_EXCALIDRAW: "mermaid-to-excalidraw",           // Mermaid settings
  PUBLISH_LIBRARY: "publish-library-data",                  // Library publish data
} as const;
```

#### EditorLocalStorage Wrapper | 编辑器本地存储包装器

**File:** `packages/excalidraw/data/EditorLocalStorage.ts:1-67`

```typescript
export class EditorLocalStorage {
  /**
   * Check if key exists in localStorage
   */
  static has(key: typeof EDITOR_LS_KEYS[keyof typeof EDITOR_LS_KEYS]): boolean {
    try {
      return !!window.localStorage.getItem(key);
    } catch (error: any) {
      console.warn(`localStorage.getItem error: ${error.message}`);
      return false;
    }
  }

  /**
   * Get value from localStorage (parsed as JSON)
   */
  static get<T extends JSONValue>(
    key: typeof EDITOR_LS_KEYS[keyof typeof EDITOR_LS_KEYS],
  ): T | null {
    try {
      const value = window.localStorage.getItem(key);
      if (value) {
        return JSON.parse(value) as T;
      }
      return null;
    } catch (error: any) {
      console.warn(`localStorage.get error: ${error.message}`);
      return null;
    }
  }

  /**
   * Set value in localStorage (stringified as JSON)
   */
  static set(
    key: typeof EDITOR_LS_KEYS[keyof typeof EDITOR_LS_KEYS],
    value: JSONValue,
  ): boolean {
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
      return true;
    } catch (error: any) {
      console.error(`localStorage.set error: ${error.message}`);
      return false;
    }
  }

  /**
   * Remove key from localStorage
   */
  static delete(key: typeof EDITOR_LS_KEYS[keyof typeof EDITOR_LS_KEYS]): boolean {
    try {
      window.localStorage.removeItem(key);
      return true;
    } catch (error: any) {
      console.error(`localStorage.delete error: ${error.message}`);
      return false;
    }
  }
}
```

#### What Data is Persisted | 持久化的数据

**File:** `excalidraw-app/data/localStorage.ts`

**Elements** (`STORAGE_KEYS.LOCAL_STORAGE_ELEMENTS`):
- All canvas elements (shapes, text, images, etc.)
- Cleared of transient state before saving
- Deleted elements removed

**AppState** (`STORAGE_KEYS.LOCAL_STORAGE_APP_STATE`):
- Browser-specific UI preferences
- Viewport position (scrollX, scrollY, zoom)
- Selected tool
- Theme
- Grid settings

**Data Cleaning Configuration** (`packages/excalidraw/appState.ts`):
```typescript
const APP_STATE_STORAGE_CONF = {
  showWelcomeScreen: { browser: true, export: false, server: false },
  theme: { browser: true, export: false, server: false },
  gridSize: { browser: true, export: true, server: true },
  gridModeEnabled: { browser: true, export: true, server: true },
  viewBackgroundColor: { browser: true, export: true, server: true },
  zoom: { browser: true, export: false, server: false },
  scrollX: { browser: true, export: false, server: false },
  scrollY: { browser: true, export: false, server: false },
  // ... 50+ more keys
};
```

**English:** This configuration determines which properties are saved to:
- **browser**: localStorage
- **export**: .excalidraw file
- **server**: Firebase/collaboration

**中文:** 此配置决定哪些属性保存到：
- **browser**: 浏览器本地存储
- **export**: .excalidraw 文件
- **server**: Firebase/协作服务器

#### Auto-Save Implementation | 自动保存实现

**File:** `excalidraw-app/data/LocalData.ts:98-140`

```typescript
const SAVE_TO_LOCAL_STORAGE_TIMEOUT = 300; // 300ms debounce

export class LocalData {
  /**
   * Debounced save to localStorage
   */
  private static _save = debounce(
    async (
      elements: readonly ExcalidrawElement[],
      appState: AppState,
      files: BinaryFiles,
      onFilesSaved: () => void,
    ) => {
      try {
        // 1. Save elements and app state to localStorage
        saveDataStateToLocalStorage(elements, appState);

        // 2. Save binary files to IndexedDB
        await this.fileStorage.saveFiles({ elements, files });

        // 3. Callback after files saved
        onFilesSaved();
      } catch (error: any) {
        console.error("Failed to save to localStorage", error);
      }
    },
    SAVE_TO_LOCAL_STORAGE_TIMEOUT,
  );

  /**
   * Public save method (respects pause state)
   */
  static save(
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    files: BinaryFiles,
    onFilesSaved: () => void,
  ) {
    // Skip saving if paused (e.g., during collaboration)
    if (!this.isSavePaused()) {
      this._save(elements, appState, files, onFilesSaved);
    }
  }

  /**
   * Pause/resume auto-save (for collaboration)
   */
  private static savePaused = false;

  static pause() {
    this.savePaused = true;
  }

  static resume() {
    this.savePaused = false;
  }

  static isSavePaused() {
    return this.savePaused;
  }
}
```

#### Storage Quota Handling | 存储配额处理

**File:** `excalidraw-app/data/LocalData.ts:68-90`

```typescript
const saveDataStateToLocalStorage = (
  elements: readonly ExcalidrawElement[],
  appState: AppState,
) => {
  try {
    // Save elements
    localStorage.setItem(
      STORAGE_KEYS.LOCAL_STORAGE_ELEMENTS,
      JSON.stringify(clearElementsForLocalStorage(elements)),
    );

    // Save app state
    localStorage.setItem(
      STORAGE_KEYS.LOCAL_STORAGE_APP_STATE,
      JSON.stringify(clearAppStateForLocalStorage(appState)),
    );
  } catch (error: any) {
    // Check if quota exceeded
    if (isQuotaExceededError(error)) {
      console.error("localStorage quota exceeded");

      // Set atom to show warning to user
      appJotaiStore.set(localStorageQuotaExceededAtom, true);
    } else {
      console.error("Failed to save to localStorage", error);
    }
  }
};

const isQuotaExceededError = (error: Error) => {
  return (
    error.name === "QuotaExceededError" ||
    error.name === "NS_ERROR_DOM_QUOTA_REACHED"
  );
};
```

---

### 3. IndexedDB (idb-keyval)

**English:** IndexedDB is used for storing large binary files (images) that would exceed localStorage quota.

**中文:** IndexedDB 用于存储大型二进制文件（图片），这些文件会超出 localStorage 配额。

#### Setup | 设置

**File:** `excalidraw-app/data/LocalData.ts:142-180`

```typescript
import { createStore, entries, del, getMany, set, setMany, get } from "idb-keyval";

// Create dedicated IndexedDB store for files
const filesStore = createStore("files-db", "files-store");

class FileManager {
  /**
   * Save files to IndexedDB
   */
  async saveFiles({
    elements,
    files,
  }: {
    elements: readonly ExcalidrawElement[];
    files: BinaryFiles;
  }) {
    const erroredFiles = new Map<FileId, true>();
    const savedFiles = new Map<FileId, true>();

    // Get all file IDs from elements
    const fileIds = new Set<FileId>();
    for (const element of elements) {
      if (
        isInitializedImageElement(element) &&
        files[element.fileId]
      ) {
        fileIds.add(element.fileId);
      }
    }

    // Save files
    for (const id of fileIds) {
      try {
        const fileData = files[id];
        if (fileData && !savedFiles.has(id)) {
          // Save to IndexedDB with metadata
          await set(
            id,
            {
              ...fileData,
              lastRetrieved: Date.now(),
            },
            filesStore,
          );
          savedFiles.set(id, true);
        }
      } catch (error: any) {
        erroredFiles.set(id, true);
        console.error(`Failed to save file ${id}`, error);
      }
    }

    return { savedFiles, erroredFiles };
  }

  /**
   * Load files from IndexedDB
   */
  async loadFiles(fileIds: readonly FileId[]) {
    const loadedFiles: BinaryFileData[] = [];
    const erroredFiles = new Map<FileId, true>();

    try {
      const filesData = await getMany(fileIds, filesStore);

      for (let i = 0; i < filesData.length; i++) {
        const id = fileIds[i];
        const fileData = filesData[i];

        if (fileData) {
          // Update last retrieved timestamp
          await set(
            id,
            { ...fileData, lastRetrieved: Date.now() },
            filesStore,
          );

          loadedFiles.push({
            ...fileData,
            id,
          });
        }
      }
    } catch (error: any) {
      console.error("Failed to load files from IndexedDB", error);
      for (const id of fileIds) {
        erroredFiles.set(id, true);
      }
    }

    return { loadedFiles, erroredFiles };
  }

  /**
   * Cleanup old files (older than 24 hours)
   */
  async cleanupOldFiles() {
    const ONE_DAY_MS = 24 * 60 * 60 * 1000;

    try {
      const allEntries = await entries(filesStore);

      for (const [id, fileData] of allEntries) {
        if (Date.now() - fileData.lastRetrieved > ONE_DAY_MS) {
          await del(id, filesStore);
          console.log(`Deleted old file: ${id}`);
        }
      }
    } catch (error: any) {
      console.error("Failed to cleanup old files", error);
    }
  }
}
```

#### Library Storage Migration | 库存储迁移

**File:** `excalidraw-app/data/LocalData.ts:210-250`

**English:** The library was migrated from localStorage to IndexedDB to avoid quota issues.

**中文:** 库从 localStorage 迁移到 IndexedDB 以避免配额问题。

```typescript
export class LibraryIndexedDBAdapter {
  private static idb_name = STORAGE_KEYS.IDB_LIBRARY;
  private static key = "libraryData";

  private static store = createStore(
    `${LibraryIndexedDBAdapter.idb_name}-db`,
    `${LibraryIndexedDBAdapter.idb_name}-store`,
  );

  /**
   * Load library from IndexedDB
   */
  static async load(): Promise<LibraryPersistedData | null> {
    try {
      const data = await get<LibraryPersistedData>(
        LibraryIndexedDBAdapter.key,
        LibraryIndexedDBAdapter.store,
      );
      return data || null;
    } catch (error: any) {
      console.error("Failed to load library from IndexedDB", error);
      return null;
    }
  }

  /**
   * Save library to IndexedDB
   */
  static async save(data: LibraryPersistedData): Promise<boolean> {
    try {
      await set(
        LibraryIndexedDBAdapter.key,
        data,
        LibraryIndexedDBAdapter.store,
      );
      return true;
    } catch (error: any) {
      console.error("Failed to save library to IndexedDB", error);
      return false;
    }
  }

  /**
   * Migrate from legacy localStorage
   */
  static async migrateLegacy() {
    try {
      const legacyData = localStorage.getItem(
        STORAGE_KEYS.__LEGACY_LOCAL_STORAGE_LIBRARY,
      );

      if (legacyData) {
        const parsed = JSON.parse(legacyData);
        await LibraryIndexedDBAdapter.save(parsed);

        // Remove legacy key
        localStorage.removeItem(STORAGE_KEYS.__LEGACY_LOCAL_STORAGE_LIBRARY);

        console.log("Library migrated from localStorage to IndexedDB");
      }
    } catch (error: any) {
      console.error("Failed to migrate library", error);
    }
  }
}
```

---

## 🔥 Firebase Integration (Collaboration) | Firebase 集成（协作）

### 4. Firebase Firestore & Storage

**English:** Firebase provides cloud storage for real-time collaboration with end-to-end encryption.

**中文:** Firebase 提供云存储以支持端到端加密的实时协作。

#### Firebase Configuration | Firebase 配置

**File:** `excalidraw-app/data/firebase.ts:1-30`

```typescript
import { initializeApp } from "firebase/app";
import { getFirestore, enableIndexedDbPersistence } from "firebase/firestore";
import { getStorage } from "firebase/storage";

// Parse config from environment variable
let FIREBASE_CONFIG: Record<string, any>;
try {
  FIREBASE_CONFIG = JSON.parse(import.meta.env.VITE_APP_FIREBASE_CONFIG);
} catch (error: any) {
  console.error("Failed to parse Firebase config", error);
  FIREBASE_CONFIG = {};
}

// Initialize Firebase app (lazy)
let firebaseApp: any = null;
let firestore: any = null;
let firebaseStorage: any = null;

const _getFirebaseApp = () => {
  if (!firebaseApp) {
    firebaseApp = initializeApp(FIREBASE_CONFIG);
  }
  return firebaseApp;
};

const _getFirestore = () => {
  if (!firestore) {
    firestore = getFirestore(_getFirebaseApp());

    // Enable offline persistence
    enableIndexedDbPersistence(firestore).catch((error) => {
      console.warn("Failed to enable Firebase persistence", error);
    });
  }
  return firestore;
};

const _getFirebaseStorage = () => {
  if (!firebaseStorage) {
    firebaseStorage = getStorage(_getFirebaseApp());
  }
  return firebaseStorage;
};
```

#### Firestore Data Model | Firestore 数据模型

**Collection:** `scenes`
**Document ID:** `{roomId}` (collaboration room ID)

**Document Structure:**
```typescript
type FirebaseStoredScene = {
  sceneVersion: number;          // Scene version number
  iv: Bytes;                     // Initialization vector (for encryption)
  ciphertext: Bytes;             // Encrypted element data
};
```

**Example Document:**
```json
{
  "sceneVersion": 42,
  "iv": <Uint8Array[16]>,
  "ciphertext": <Encrypted ArrayBuffer>
}
```

#### Save Scene to Firebase | 保存场景到 Firebase

**File:** `excalidraw-app/data/firebase.ts:100-180`

```typescript
export const saveToFirebase = async (
  portal: Portal,
  elements: readonly SyncableExcalidrawElement[],
  appState: AppState,
) => {
  const { roomId, roomKey } = portal;
  const firestore = _getFirestore();
  const docRef = doc(firestore, "scenes", roomId);

  // Use transaction for atomic read-modify-write
  await runTransaction(firestore, async (transaction) => {
    const snapshot = await transaction.get(docRef);

    if (snapshot.exists()) {
      // Document exists - reconcile with existing data
      const prevStoredScene = snapshot.data() as FirebaseStoredScene;

      // Decrypt existing elements
      const prevElements = await decryptElements(prevStoredScene, roomKey);

      // Reconcile local changes with remote
      const reconciledElements = reconcileElements(
        elements,
        prevElements,
        appState,
      );

      // Create encrypted scene document
      const storedScene = await createFirebaseSceneDocument(
        reconciledElements,
        roomKey,
      );

      // Update document
      transaction.update(docRef, storedScene);
    } else {
      // New document - create
      const storedScene = await createFirebaseSceneDocument(
        elements,
        roomKey,
      );

      transaction.set(docRef, storedScene);
    }
  });
};

/**
 * Create encrypted scene document
 */
const createFirebaseSceneDocument = async (
  elements: readonly ExcalidrawElement[],
  roomKey: string,
): Promise<FirebaseStoredScene> => {
  // Serialize elements to JSON
  const json = JSON.stringify(elements);
  const encoded = new TextEncoder().encode(json);

  // Encrypt data
  const { encryptedBuffer, iv } = await encryptData(roomKey, encoded);

  return {
    sceneVersion: getSceneVersion(elements),
    iv: Array.from(iv),           // Convert Uint8Array to Array for Firestore
    ciphertext: Array.from(new Uint8Array(encryptedBuffer)),
  };
};
```

#### Load Scene from Firebase | 从 Firebase 加载场景

**File:** `excalidraw-app/data/firebase.ts:200-250`

```typescript
export const loadFromFirebase = async (
  roomId: string,
  roomKey: string,
): Promise<readonly ExcalidrawElement[] | null> => {
  const firestore = _getFirestore();
  const docRef = doc(firestore, "scenes", roomId);

  try {
    const snapshot = await getDoc(docRef);

    if (!snapshot.exists()) {
      return null;
    }

    const storedScene = snapshot.data() as FirebaseStoredScene;

    // Decrypt elements
    const elements = await decryptElements(storedScene, roomKey);

    return elements;
  } catch (error: any) {
    console.error("Failed to load from Firebase", error);
    throw error;
  }
};

/**
 * Decrypt scene elements
 */
const decryptElements = async (
  storedScene: FirebaseStoredScene,
  roomKey: string,
): Promise<readonly ExcalidrawElement[]> => {
  const { iv, ciphertext } = storedScene;

  // Convert Arrays back to Uint8Array/ArrayBuffer
  const ivArray = new Uint8Array(iv);
  const ciphertextBuffer = new Uint8Array(ciphertext).buffer;

  // Decrypt
  const { decryptedBuffer } = await decryptData(ivArray, ciphertextBuffer, roomKey);

  // Parse JSON
  const json = new TextDecoder().decode(decryptedBuffer);
  const elements = JSON.parse(json) as ExcalidrawElement[];

  return elements;
};
```

#### File Storage in Firebase Storage | Firebase 存储中的文件

**File Path Pattern:** `files/rooms/{roomId}/{fileId}`

**Save Files** (`excalidraw-app/data/firebase.ts:300-350`):
```typescript
export const saveFilesToFirebase = async ({
  prefix,
  files,
}: {
  prefix: string;  // "files/rooms/{roomId}"
  files: { id: FileId; buffer: Uint8Array }[];
}) => {
  const storage = await _getFirebaseStorage();

  // Upload all files in parallel
  await Promise.all(
    files.map(async ({ id, buffer }) => {
      const storageRef = ref(storage, `${prefix}/${id}`);

      // Upload with caching headers
      await uploadBytes(storageRef, buffer, {
        cacheControl: `public, max-age=${FILE_CACHE_MAX_AGE_SEC}`, // 30 days
      });
    }),
  );
};
```

**Load Files** (`excalidraw-app/data/firebase.ts:380-450`):
```typescript
export const loadFilesFromFirebase = async (
  prefix: string,
  decryptionKey: string,
  fileIds: readonly FileId[],
): Promise<{ loadedFiles: BinaryFileData[]; erroredFiles: Map<FileId, true> }> => {
  const loadedFiles: BinaryFileData[] = [];
  const erroredFiles = new Map<FileId, true>();

  // Load all files in parallel
  await Promise.all(
    fileIds.map(async (id) => {
      try {
        // Construct Firebase Storage URL
        const url = `https://firebasestorage.googleapis.com/v0/b/${
          FIREBASE_CONFIG.storageBucket
        }/o/${encodeURIComponent(prefix)}%2F${id}`;

        // Fetch file
        const response = await fetch(`${url}?alt=media`);

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`);
        }

        const arrayBuffer = await response.arrayBuffer();

        // Decompress and decrypt
        const { data, metadata } = await decompressData(
          new Uint8Array(arrayBuffer),
          { decryptionKey },
        );

        // Convert to data URL
        const dataURL = new TextDecoder().decode(data);

        loadedFiles.push({
          mimeType: metadata.mimeType || MIME_TYPES.binary,
          id,
          dataURL,
          created: metadata?.created || Date.now(),
        });
      } catch (error: any) {
        console.error(`Failed to load file ${id}`, error);
        erroredFiles.set(id, true);
      }
    }),
  );

  return { loadedFiles, erroredFiles };
};
```

---

## 🔐 End-to-End Encryption | 端到端加密

### 5. Encryption System

**English:** All collaboration data (scenes and files) is encrypted end-to-end using AES-GCM before being sent to Firebase.

**中文:** 所有协作数据（场景和文件）在发送到 Firebase 之前都使用 AES-GCM 进行端到端加密。

**File:** `packages/excalidraw/data/encryption.ts`

#### Key Derivation | 密钥派生

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
      salt: encoder.encode("excalidraw-salt"), // Fixed salt (not ideal but simple)
      iterations: 100000,
      hash: "SHA-256",
    },
    keyMaterial,
    {
      name: "AES-GCM",
      length: 256,
    },
    false,
    ["encrypt", "decrypt"],
  );

  return key;
};
```

#### Encryption | 加密

```typescript
/**
 * Encrypt data using AES-GCM
 */
export const encryptData = async (
  passphrase: string,
  data: Uint8Array,
): Promise<{ encryptedBuffer: ArrayBuffer; iv: Uint8Array }> => {
  const key = await deriveKey(passphrase);

  // Generate random initialization vector (IV)
  const iv = window.crypto.getRandomValues(new Uint8Array(16));

  // Encrypt using AES-GCM
  const encryptedBuffer = await window.crypto.subtle.encrypt(
    {
      name: "AES-GCM",
      iv,
    },
    key,
    data,
  );

  return { encryptedBuffer, iv };
};
```

#### Decryption | 解密

```typescript
/**
 * Decrypt data using AES-GCM
 */
export const decryptData = async (
  iv: Uint8Array,
  encryptedBuffer: ArrayBuffer,
  passphrase: string,
): Promise<{ decryptedBuffer: ArrayBuffer }> => {
  const key = await deriveKey(passphrase);

  // Decrypt using AES-GCM
  const decryptedBuffer = await window.crypto.subtle.decrypt(
    {
      name: "AES-GCM",
      iv,
    },
    key,
    encryptedBuffer,
  );

  return { decryptedBuffer };
};
```

---

## 🔄 Real-Time Collaboration | 实时协作

### 6. WebSocket Communication

**English:** Real-time updates are broadcast via WebSocket (Socket.IO) with encrypted payloads.

**中文:** 实时更新通过 WebSocket（Socket.IO）广播，使用加密负载。

#### WebSocket Events | WebSocket 事件

**File:** `excalidraw-app/app_constants.ts:30-40`

```typescript
export const WS_EVENTS = {
  SERVER_VOLATILE: "server-volatile-broadcast",  // Volatile (not guaranteed delivery)
  SERVER: "server-broadcast",                    // Reliable delivery
  USER_FOLLOW_CHANGE: "user-follow",             // User follow status
  USER_FOLLOW_ROOM_CHANGE: "user-follow-room-change",
} as const;

export enum WS_SUBTYPES {
  INVALID_RESPONSE = "INVALID_RESPONSE",
  INIT = "SCENE_INIT",                          // Initial scene load
  UPDATE = "SCENE_UPDATE",                      // Incremental update
  MOUSE_LOCATION = "MOUSE_LOCATION",            // Cursor position
  IDLE_STATUS = "IDLE_STATUS",                  // User idle/active
  USER_VISIBLE_SCENE_BOUNDS = "USER_VISIBLE_SCENE_BOUNDS",
}
```

#### Broadcasting Scene Updates | 广播场景更新

**File:** `excalidraw-app/collab/Portal.tsx:200-280`

```typescript
export class Portal {
  private socket: Socket | null = null;
  private roomId: string | null = null;
  private roomKey: string | null = null;

  // Track which element versions have been broadcast
  private broadcastedElementVersions = new Map<string, number>();

  /**
   * Broadcast scene changes to other collaborators
   */
  broadcastScene = async (
    updateType: WS_SUBTYPES.INIT | WS_SUBTYPES.UPDATE,
    elements: readonly OrderedExcalidrawElement[],
    syncAll: boolean,  // If true, sync all elements; if false, only changed
  ) => {
    if (!this.socket || !this.roomId || !this.roomKey) {
      return;
    }

    // Determine which elements need to be synced
    const syncableElements = elements.reduce((acc, element) => {
      const lastVersion = this.broadcastedElementVersions.get(element.id);

      if (
        syncAll ||
        !lastVersion ||
        element.version > lastVersion
      ) {
        acc.push(element);

        // Track this version as broadcast
        this.broadcastedElementVersions.set(element.id, element.version);
      }

      return acc;
    }, [] as SyncableExcalidrawElement[]);

    if (syncableElements.length === 0 && !syncAll) {
      return; // Nothing to sync
    }

    // Create payload
    const data = {
      type: updateType,
      payload: {
        elements: syncableElements,
      },
    };

    // Serialize to JSON
    const json = JSON.stringify(data);
    const encoded = new TextEncoder().encode(json);

    // Encrypt
    const { encryptedBuffer, iv } = await encryptData(this.roomKey, encoded);

    // Broadcast via WebSocket
    this.socket.emit(
      WS_EVENTS.SERVER,
      this.roomId,
      encryptedBuffer,
      iv,
    );
  };

  /**
   * Broadcast cursor position
   */
  broadcastMouseLocation = (payload: { pointer: { x: number; y: number }; button: "down" | "up" }) => {
    if (!this.socket || !this.roomId) {
      return;
    }

    this.socket.volatile.emit(
      WS_EVENTS.SERVER_VOLATILE,
      this.roomId,
      {
        type: WS_SUBTYPES.MOUSE_LOCATION,
        payload,
      },
    );
  };
}
```

#### Receiving Remote Updates | 接收远程更新

**File:** `excalidraw-app/collab/Portal.tsx:350-450`

```typescript
/**
 * Listen for scene updates from other collaborators
 */
socket.on(WS_EVENTS.SERVER, async (encryptedData, iv) => {
  // Decrypt payload
  const { decryptedBuffer } = await decryptData(
    new Uint8Array(iv),
    encryptedData,
    this.roomKey!,
  );

  // Parse JSON
  const json = new TextDecoder().decode(decryptedBuffer);
  const data = JSON.parse(json);

  if (data.type === WS_SUBTYPES.INIT || data.type === WS_SUBTYPES.UPDATE) {
    const remoteElements = data.payload.elements;

    // Reconcile with local elements
    const reconciledElements = reconcileElements(
      localElements,
      remoteElements,
      appState,
    );

    // Update scene
    scene.replaceAllElements(reconciledElements);
  }
});

/**
 * Listen for cursor updates (volatile)
 */
socket.on(WS_EVENTS.SERVER_VOLATILE, (data) => {
  if (data.type === WS_SUBTYPES.MOUSE_LOCATION) {
    const { pointer, button } = data.payload;

    // Update remote user cursor position
    updateCollaboratorCursor(data.socketId, pointer, button);
  }
});
```

---

## 🔁 Reconciliation & Conflict Resolution | 调和和冲突解决

### 7. Element Reconciliation

**English:** When multiple users edit simultaneously, Excalidraw reconciles changes using version-based conflict resolution.

**中文:** 当多个用户同时编辑时，Excalidraw 使用基于版本的冲突解决来调和更改。

**File:** `packages/excalidraw/data/reconcile.ts`

#### Conflict Resolution Logic | 冲突解决逻辑

```typescript
/**
 * Determine if remote element should be discarded
 */
export const shouldDiscardRemoteElement = (
  localAppState: AppState,
  local: OrderedExcalidrawElement | undefined,
  remote: RemoteExcalidrawElement,
): boolean => {
  if (!local) {
    return false; // Remote element is new, accept it
  }

  // Priority 1: Local element is being edited
  if (
    local.id === localAppState.editingTextElement?.id ||
    local.id === localAppState.resizingElement?.id ||
    local.id === localAppState.newElement?.id
  ) {
    return true; // Keep local version
  }

  // Priority 2: Version comparison
  if (local.version > remote.version) {
    return true; // Local is newer
  }

  if (local.version < remote.version) {
    return false; // Remote is newer
  }

  // Priority 3: Deterministic tiebreaker (same version)
  // Lower versionNonce wins (arbitrary but consistent)
  if (local.versionNonce < remote.versionNonce) {
    return true; // Keep local
  }

  return false; // Use remote
};
```

#### Full Reconciliation Algorithm | 完整调和算法

```typescript
export const reconcileElements = (
  localElements: readonly OrderedExcalidrawElement[],
  remoteElements: readonly RemoteExcalidrawElement[],
  localAppState: AppState,
): ReconciledExcalidrawElement[] => {
  // Create map for O(1) lookup
  const localElementsMap = arrayToMap(localElements);
  const reconciledElements: OrderedExcalidrawElement[] = [];
  const added = new Set<string>();

  // Step 1: Process all remote elements
  for (const remoteElement of remoteElements) {
    const localElement = localElementsMap.get(remoteElement.id);

    // Decide which version to keep
    const discardRemote = shouldDiscardRemoteElement(
      localAppState,
      localElement,
      remoteElement,
    );

    if (localElement && discardRemote) {
      reconciledElements.push(localElement);
    } else {
      reconciledElements.push(remoteElement);
    }

    added.add(remoteElement.id);
  }

  // Step 2: Add local-only elements (not in remote)
  for (const localElement of localElements) {
    if (!added.has(localElement.id)) {
      reconciledElements.push(localElement);
      added.add(localElement.id);
    }
  }

  // Step 3: Sort by fractional index
  const orderedElements = orderByFractionalIndex(reconciledElements);

  // Step 4: Fix any invalid indices
  syncInvalidIndices(orderedElements);

  return orderedElements;
};
```

#### Reconciliation Flow Diagram | 调和流程图

```
┌─────────────────────────────────────────────────────┐
│        Remote Update Received                       │
│  (encrypted elements from other collaborator)       │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
           ┌─────────────────┐
           │ Decrypt Payload │
           └────────┬────────┘
                    │
                    ▼
       ┌────────────────────────┐
       │ Parse Remote Elements  │
       └────────┬───────────────┘
                │
                ▼
┌───────────────────────────────────────────────────┐
│          For Each Remote Element:                 │
├───────────────────────────────────────────────────┤
│  1. Find corresponding local element (by ID)      │
│  2. Check if local element is being edited        │
│  3. Compare versions                              │
│  4. Use deterministic tiebreaker if equal         │
│  5. Choose winning version                        │
└────────────────────┬──────────────────────────────┘
                     │
                     ▼
       ┌─────────────────────────┐
       │ Add Local-Only Elements │
       │  (not in remote)        │
       └────────┬────────────────┘
                │
                ▼
        ┌───────────────────┐
        │ Sort by Fractional│
        │     Index         │
        └────────┬──────────┘
                 │
                 ▼
        ┌───────────────────┐
        │ Fix Invalid       │
        │ Indices           │
        └────────┬──────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│      Update Scene with Reconciled Elements         │
│  scene.replaceAllElements(reconciledElements)      │
└────────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│             React Re-render                        │
│     (UI updates with merged changes)               │
└────────────────────────────────────────────────────┘
```

---

## 📦 Data Serialization Formats | 数据序列化格式

### 8. .excalidraw Format (Scene Export)

**File:** `packages/excalidraw/data/json.ts`

#### Data Structure | 数据结构

```typescript
export interface ExportedDataState {
  type: string;                    // "excalidraw"
  version: number;                 // Format version (currently 2)
  source: string;                  // Source URL (e.g., "https://excalidraw.com")
  elements: readonly ExcalidrawElement[];
  appState: ReturnType<typeof cleanAppStateForExport>;
  files: BinaryFiles | undefined;  // Embedded images (base64)
}
```

#### Serialization Function | 序列化函数

```typescript
export const serializeAsJSON = (
  elements: readonly ExcalidrawElement[],
  appState: Partial<AppState>,
  files: BinaryFiles,
  type: "local" | "database",
): string => {
  const data: ExportedDataState = {
    type: EXPORT_DATA_TYPES.excalidraw,  // "excalidraw"
    version: VERSIONS.excalidraw,         // 2
    source: getExportSource(),            // "https://excalidraw.com"

    // Clean elements (remove transient state)
    elements:
      type === "local"
        ? clearElementsForExport(elements)
        : clearElementsForDatabase(elements),

    // Clean app state (only save exportable keys)
    appState:
      type === "local"
        ? cleanAppStateForExport(appState)
        : clearAppStateForDatabase(appState),

    // Include files only for local export
    files: type === "local" ? filterOutDeletedFiles(elements, files) : undefined,
  };

  return JSON.stringify(data, null, 2);
};
```

#### Save to File | 保存到文件

```typescript
export const saveAsJSON = async (
  elements: readonly ExcalidrawElement[],
  appState: AppState,
  files: BinaryFiles,
  name: string,
) => {
  const serialized = serializeAsJSON(elements, appState, files, "local");

  const blob = new Blob([serialized], {
    type: MIME_TYPES.excalidraw,  // "application/vnd.excalidraw+json"
  });

  // Use File System Access API (with fallback)
  const fileHandle = await fileSave(blob, {
    fileName: name,
    extensions: [".excalidraw"],
    description: "Excalidraw file",
  });

  return { fileHandle };
};
```

---

### 9. .excalidrawlib Format (Library Export)

```typescript
export interface ExportedLibraryData {
  type: string;                           // "excalidrawlib"
  version: typeof VERSIONS.excalidrawLibrary;  // 2
  source: string;                         // "https://excalidraw.com"
  libraryItems: LibraryItems;
}

export const serializeLibraryAsJSON = (libraryItems: LibraryItems): string => {
  const data: ExportedLibraryData = {
    type: EXPORT_DATA_TYPES.excalidrawLibrary,
    version: VERSIONS.excalidrawLibrary,
    source: getExportSource(),
    libraryItems,
  };

  return JSON.stringify(data, null, 2);
};
```

---

## 📐 Element Data Model | 元素数据模型

### 10. Core Element Structure

**File:** `packages/element/src/types.ts:50-100`

```typescript
type _ExcalidrawElementBase = Readonly<{
  id: string;                           // Unique identifier (nanoid)
  x: number;                            // X position (scene coordinates)
  y: number;                            // Y position (scene coordinates)
  strokeColor: string;                  // Hex color (e.g., "#000000")
  backgroundColor: string;              // Hex color with "transparent"
  fillStyle: FillStyle;                 // "hachure" | "cross-hatch" | "solid" | "zigzag"
  strokeWidth: number;                  // 1, 2, 4
  strokeStyle: StrokeStyle;             // "solid" | "dashed" | "dotted"
  roundness: null | { type: RoundnessType; value?: number };
  roughness: number;                    // 0 (architect), 1 (artist), 2 (cartoonist)
  opacity: number;                      // 0-100
  width: number;                        // Width in pixels
  height: number;                       // Height in pixels
  angle: Radians;                       // Rotation angle (radians)
  seed: number;                         // Random seed for rough.js
  version: number;                      // Incremented on each change
  versionNonce: number;                 // Random nonce for conflict resolution
  index: FractionalIndex | null;        // Z-index using fractional indexing
  isDeleted: boolean;                   // Soft delete flag
  groupIds: readonly GroupId[];         // Groups this element belongs to
  frameId: string | null;               // Parent frame ID
  boundElements: readonly BoundElement[] | null;  // Bound text/arrows
  updated: number;                      // Timestamp (ms) of last update
  link: string | null;                  // Hyperlink URL
  locked: boolean;                      // Prevent editing
  customData?: Record<string, any>;     // Custom metadata
}>;
```

#### Example Element (Rectangle) | 示例元素（矩形）

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

---

## 🎯 Data Flow Summary | 数据流总结

### Complete Data Flow | 完整数据流

```
┌───────────────────────────────────────────────────────────────┐
│                     User Interaction                          │
│  (mouse move, click, keyboard, etc.)                          │
└─────────────────────────┬─────────────────────────────────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ Event Handler   │
                 │ (App.tsx)       │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ Action/Mutation │
                 │ mutateElement() │
                 └────────┬────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                  Scene.replaceAllElements()                  │
│  - Updates elements array                                   │
│  - Rebuilds maps                                            │
│  - Increments scene nonce                                   │
│  - Triggers callbacks                                       │
└──────────┬──────────────────────────┬───────────────────────┘
           │                          │
           │                          ├─────────────────┐
           │                          │                 │
           ▼                          ▼                 ▼
  ┌─────────────┐         ┌──────────────────┐  ┌────────────┐
  │ React       │         │ Auto-save         │  │ Broadcast  │
  │ Re-render   │         │ (300ms debounce) │  │ (throttled)│
  └─────────────┘         └────────┬─────────┘  └─────┬──────┘
                                   │                   │
                          ┌────────┴────────┐         │
                          │                 │         │
                          ▼                 ▼         ▼
                  ┌──────────────┐  ┌──────────┐  ┌────────┐
                  │ localStorage │  │IndexedDB │  │WebSocket│
                  │  Elements    │  │  Files   │  │Encrypted│
                  │  AppState    │  └──────────┘  └────┬───┘
                  └──────────────┘                     │
                                                       ▼
                                              ┌────────────────┐
                                              │    Firebase    │
                                              │  Firestore +   │
                                              │    Storage     │
                                              └────────────────┘
```

---

## ⚡ Performance Optimizations | 性能优化

### Key Optimizations | 关键优化

| Optimization | Implementation | Impact |
|-------------|----------------|---------|
| **Debounced Saves** | 300ms debounce on localStorage writes | Reduces write frequency |
| **Throttled Broadcasts** | RAF throttling on collab updates | Limits network traffic |
| **Incremental Sync** | Only broadcast changed elements | Reduces payload size |
| **Lazy File Loading** | Files loaded on-demand from IndexedDB | Faster initial load |
| **Scene Nonce** | Cache invalidation without deep comparison | O(1) change detection |
| **Fractional Indices** | Stable ordering without re-indexing | Efficient z-index updates |
| **Element Maps** | O(1) element lookup by ID | Fast element access |
| **Offline Persistence** | Firebase IndexedDB persistence | Works offline |

---

## 📊 Storage Limits & Quotas | 存储限制和配额

| Storage Type | Typical Limit | Used For | Quota Handling |
|-------------|--------------|----------|----------------|
| **localStorage** | ~5-10 MB | Elements, AppState | Shows warning, disables auto-save |
| **IndexedDB** | ~50-100 MB+ | Binary files, library | Automatic cleanup (24h old files) |
| **Firebase Firestore** | Unlimited* | Collaboration scenes | Rate limiting, encryption |
| **Firebase Storage** | Unlimited* | Collaboration files | 30-day caching |

*Subject to Firebase pricing and quotas

---

## 🔍 Debugging & Monitoring | 调试和监控

### Storage Debugging | 存储调试

**Check localStorage Usage:**
```javascript
// In browser console
let total = 0;
for (let key in localStorage) {
  if (localStorage.hasOwnProperty(key)) {
    total += localStorage[key].length + key.length;
  }
}
console.log(`localStorage usage: ${(total / 1024).toFixed(2)} KB`);
```

**Check IndexedDB:**
```javascript
// List all IndexedDB databases
indexedDB.databases().then(console.log);

// Open specific database
const request = indexedDB.open("files-db");
request.onsuccess = () => {
  const db = request.result;
  console.log("ObjectStores:", db.objectStoreNames);
};
```

---

## 📚 Related Documents | 相关文档

**Previous:** `00-project-overview.md` - Project structure and technology stack
**Next:** `02-backend-analysis.md` - API endpoints and architecture (adapted for collaboration backend)

**Key Files Referenced:**
- `packages/excalidraw/editor-jotai.ts` - Jotai setup
- `excalidraw-app/data/LocalData.ts` - Local storage management
- `excalidraw-app/data/firebase.ts` - Firebase integration
- `packages/excalidraw/data/reconcile.ts` - Conflict resolution
- `packages/excalidraw/data/encryption.ts` - End-to-end encryption
- `packages/element/src/types.ts` - Element data model

---

**Analysis Completed:** Phase 2 ✅
**Next Phase:** Architecture and Core Logic Analysis
