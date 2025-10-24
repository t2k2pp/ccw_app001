# ストレージ戦略設計書

## 目次
1. [概要](#概要)
2. [ストレージ分類](#ストレージ分類)
3. [IndexedDB設計](#indexeddb設計)
4. [localStorage設計](#localstorage設計)
5. [エクスポート/インポート仕様](#エクスポートインポート仕様)
6. [データマイグレーション](#データマイグレーション)
7. [容量管理](#容量管理)
8. [エラーハンドリング](#エラーハンドリング)

## 概要

Local AI Slide Creatorは、**完全にクライアント側**でデータを管理するアーキテクチャを採用しています。
ユーザーデータはサーバーに一切送信されず、ブラウザ/Electronアプリ内のストレージに保存されます。

### 設計原則

```
1. プライバシー第一
   → ユーザーデータはクライアント側のみ
   → サーバーに送信・保存しない

2. データ所有権
   → ユーザーが完全にコントロール
   → エクスポート/インポートで自由に移行

3. オフライン優先
   → ネットワーク不要で動作
   → AI機能のみオンライン必須

4. 効率的な容量管理
   → 適切なストレージ使い分け
   → 自動クリーンアップ
```

## ストレージ分類

### データ保存先の使い分け

| ストレージ | 容量制限 | 用途 | データ型 | パフォーマンス |
|----------|----------|------|---------|--------------|
| **localStorage** | ~5-10MB | 設定、UI状態 | Key-Value（String） | 高速（同期） |
| **IndexedDB** | ~50MB-数GB | プロジェクト、アセット | Object, Blob | 高速（非同期） |
| **Server SQLite** | ~1GB | システムデータ、キャッシュ | Relational | N/A（サーバー側） |

### ストレージ選択フローチャート

```
データを保存する
     │
     ↓
┌─────────────────────┐
│ サイズは5MB以下？    │
└─────────────────────┘
     │ Yes         │ No
     ↓             ↓
┌─────────────┐  ┌──────────────┐
│バイナリ？    │  │ IndexedDB    │
└─────────────┘  │ (大容量データ)│
     │ No        └──────────────┘
     ↓
┌─────────────────┐
│ localStorage    │
│ (設定・UI状態)   │
└─────────────────┘
```

## IndexedDB設計

### データベース構造

```typescript
// Database Name: 'LocalAISlideCreator'
// Version: 1

interface IndexedDBSchema {
  // 1. Projects Store
  projects: {
    keyPath: 'id';
    indexes: {
      'by-updated': 'metadata.updatedAt';
      'by-created': 'metadata.createdAt';
      'by-name': 'name';
    };
  };

  // 2. Slides Store
  slides: {
    keyPath: 'id';
    indexes: {
      'by-project': 'projectId';
      'by-order': ['projectId', 'orderIndex']; // Compound index
    };
  };

  // 3. Assets Store (Images, SVG, Videos)
  assets: {
    keyPath: 'id';
    indexes: {
      'by-project': 'projectId';
      'by-type': 'type';
      'by-created': 'metadata.createdAt';
    };
  };

  // 4. Templates Store (Custom templates only)
  templates: {
    keyPath: 'id';
    indexes: {
      'by-category': 'category';
      'by-created': 'metadata.createdAt';
    };
  };

  // 5. History Store (Undo/Redo)
  history: {
    keyPath: 'id';
    indexes: {
      'by-project': 'projectId';
      'by-timestamp': ['projectId', 'timestamp']; // Compound index
    };
  };
}
```

### 詳細スキーマ定義

#### 1. Projects Store

```typescript
interface Project {
  id: string; // UUID v4
  name: string;
  description?: string;
  themeId: string; // Reference to theme
  settings: {
    defaultTransition: TransitionType;
    defaultDisplayDuration: number; // seconds
    aspectRatio: '16:9' | '4:3' | '1:1';
    resolution: { width: number; height: number };
  };
  metadata: {
    createdAt: Date;
    updatedAt: Date;
    slideCount: number;
    totalSize: number; // bytes (calculated)
    tags?: string[];
  };
}

// 使用例
const project: Project = {
  id: 'proj_a1b2c3d4',
  name: '2025年度 事業計画',
  description: '新規事業の提案資料',
  themeId: 'theme_modern_blue',
  settings: {
    defaultTransition: 'fade',
    defaultDisplayDuration: 5,
    aspectRatio: '16:9',
    resolution: { width: 1920, height: 1080 },
  },
  metadata: {
    createdAt: new Date('2025-10-24T10:00:00Z'),
    updatedAt: new Date('2025-10-24T15:30:00Z'),
    slideCount: 15,
    totalSize: 2500000, // 2.5MB
    tags: ['business', 'proposal', '2025'],
  },
};
```

#### 2. Slides Store

```typescript
interface Slide {
  id: string; // UUID v4
  projectId: string; // Foreign key to projects
  templateId?: string; // Reference to template
  title: string;
  notes?: string; // Speaker notes
  talkScript?: string; // TTS script
  orderIndex: number; // 0-based
  elements: Element[]; // Slide elements (text, image, shape, etc.)
  background: Background;
  displayDuration: number; // seconds (for signage mode)
  transition: TransitionSettings;
  voiceSettings?: VoiceSettings;
  metadata: {
    createdAt: Date;
    updatedAt: Date;
  };
}

interface Element {
  id: string;
  type: 'text' | 'image' | 'shape' | 'icon' | 'chart';
  x: number; // pixels
  y: number;
  width: number;
  height: number;
  rotation: number; // degrees
  zIndex: number;
  locked: boolean;
  visible: boolean;

  // Type-specific properties
  properties: TextProperties | ImageProperties | ShapeProperties | IconProperties | ChartProperties;
}

interface TextProperties {
  content: string;
  fontSize: number;
  fontFamily: string;
  fontWeight: 'normal' | 'bold' | '100' | '200' | ... | '900';
  fontStyle: 'normal' | 'italic';
  color: string; // hex color
  align: 'left' | 'center' | 'right' | 'justify';
  lineHeight: number;
  letterSpacing: number;
}

interface ImageProperties {
  assetId: string; // Reference to assets store
  objectFit: 'contain' | 'cover' | 'fill' | 'none';
  opacity: number; // 0-1
  filters?: {
    brightness?: number;
    contrast?: number;
    blur?: number;
  };
}

interface Background {
  type: 'solid' | 'gradient' | 'image';
  color?: string; // for solid
  gradient?: {
    type: 'linear' | 'radial';
    colors: string[];
    angle?: number; // for linear
  };
  imageAssetId?: string; // for image background
}

interface TransitionSettings {
  type: 'none' | 'fade' | 'slide-left' | 'slide-right' | 'slide-up' | 'slide-down' | 'zoom' | 'flip' | 'cube';
  duration: number; // milliseconds
  easing?: 'linear' | 'ease' | 'ease-in' | 'ease-out' | 'ease-in-out';
}

interface VoiceSettings {
  voice?: string; // e.g., 'ja-JP-Neural2-B'
  rate?: number; // 0.5 - 2.0
  pitch?: number; // 0.5 - 2.0
  volume?: number; // 0.0 - 1.0
  language?: string; // e.g., 'ja-JP'
}
```

#### 3. Assets Store

```typescript
interface Asset {
  id: string; // UUID v4
  projectId?: string; // null = global asset
  name: string;
  type: 'image' | 'svg' | 'video' | 'audio';
  mimeType: string; // e.g., 'image/png', 'image/svg+xml'
  blob: Blob; // Binary data
  thumbnailBlob?: Blob; // Thumbnail (200px width)
  size: number; // bytes
  dimensions?: { width: number; height: number }; // for images/videos
  duration?: number; // for videos/audio (seconds)
  metadata: {
    createdAt: Date;
    source: 'upload' | 'ai-generated' | 'icon-library' | 'url-import';
    aiPrompt?: string; // if AI-generated
    originalUrl?: string; // if url-import
    tags?: string[];
  };
}

// 使用例：アップロードした画像
const uploadedImage: Asset = {
  id: 'asset_img_001',
  projectId: 'proj_a1b2c3d4',
  name: 'company-logo.png',
  type: 'image',
  mimeType: 'image/png',
  blob: new Blob([...], { type: 'image/png' }),
  thumbnailBlob: new Blob([...], { type: 'image/jpeg' }),
  size: 150000, // 150KB
  dimensions: { width: 1200, height: 600 },
  metadata: {
    createdAt: new Date(),
    source: 'upload',
    tags: ['logo', 'brand'],
  },
};

// 使用例：AI生成画像
const aiGeneratedImage: Asset = {
  id: 'asset_img_002',
  projectId: 'proj_a1b2c3d4',
  name: 'ai-abstract-background.png',
  type: 'image',
  mimeType: 'image/png',
  blob: new Blob([...], { type: 'image/png' }),
  thumbnailBlob: new Blob([...], { type: 'image/jpeg' }),
  size: 800000, // 800KB
  dimensions: { width: 1920, height: 1080 },
  metadata: {
    createdAt: new Date(),
    source: 'ai-generated',
    aiPrompt: 'Abstract blue gradient background, modern, tech style',
    tags: ['background', 'ai-generated'],
  },
};
```

#### 4. Templates Store (Custom only)

```typescript
interface CustomTemplate {
  id: string; // UUID v4
  name: string;
  description?: string;
  category: string; // 'title', 'content', 'image-text', 'two-column', etc.
  thumbnailAssetId?: string; // Reference to assets
  layout: LayoutDefinition;
  isCustom: true; // Always true for IndexedDB templates
  metadata: {
    createdAt: Date;
    updatedAt: Date;
    usageCount: number;
    favorite: boolean;
  };
}

interface LayoutDefinition {
  width: number; // e.g., 1920
  height: number; // e.g., 1080
  areas: LayoutArea[];
}

interface LayoutArea {
  id: string;
  type: 'text' | 'image' | 'shape' | 'placeholder';
  x: number; // percentage or pixels
  y: number;
  width: number;
  height: number;
  constraints?: {
    minWidth?: number;
    maxWidth?: number;
    aspectRatio?: number;
  };
  defaultProperties?: Partial<Element>;
}
```

#### 5. History Store (Undo/Redo)

```typescript
interface HistoryEntry {
  id: string; // UUID v4
  projectId: string;
  action: ActionType;
  snapshot: StateSnapshot; // Before state
  timestamp: Date;
}

type ActionType =
  | 'create_slide'
  | 'delete_slide'
  | 'update_slide'
  | 'move_slide'
  | 'create_element'
  | 'delete_element'
  | 'update_element'
  | 'update_project';

interface StateSnapshot {
  type: 'slide' | 'element' | 'project';
  data: Slide | Element | Project;
}

// 使用例
const historyEntry: HistoryEntry = {
  id: 'hist_001',
  projectId: 'proj_a1b2c3d4',
  action: 'delete_element',
  snapshot: {
    type: 'element',
    data: {
      // Element data before deletion
      id: 'elem_text_001',
      type: 'text',
      // ...
    },
  },
  timestamp: new Date(),
};
```

### IndexedDB 操作クラス

```typescript
// services/storage/indexedDBService.ts

import { openDB, DBSchema, IDBPDatabase } from 'idb';

class IndexedDBService {
  private dbName = 'LocalAISlideCreator';
  private version = 1;
  private db: IDBPDatabase<AppDBSchema> | null = null;

  /**
   * データベースの初期化
   */
  async init(): Promise<void> {
    this.db = await openDB<AppDBSchema>(this.dbName, this.version, {
      upgrade(db, oldVersion, newVersion, transaction) {
        // Projects Store
        if (!db.objectStoreNames.contains('projects')) {
          const projectStore = db.createObjectStore('projects', { keyPath: 'id' });
          projectStore.createIndex('by-updated', 'metadata.updatedAt');
          projectStore.createIndex('by-created', 'metadata.createdAt');
          projectStore.createIndex('by-name', 'name');
        }

        // Slides Store
        if (!db.objectStoreNames.contains('slides')) {
          const slideStore = db.createObjectStore('slides', { keyPath: 'id' });
          slideStore.createIndex('by-project', 'projectId');
          slideStore.createIndex('by-order', ['projectId', 'orderIndex']);
        }

        // Assets Store
        if (!db.objectStoreNames.contains('assets')) {
          const assetStore = db.createObjectStore('assets', { keyPath: 'id' });
          assetStore.createIndex('by-project', 'projectId');
          assetStore.createIndex('by-type', 'type');
          assetStore.createIndex('by-created', 'metadata.createdAt');
        }

        // Templates Store
        if (!db.objectStoreNames.contains('templates')) {
          const templateStore = db.createObjectStore('templates', { keyPath: 'id' });
          templateStore.createIndex('by-category', 'category');
          templateStore.createIndex('by-created', 'metadata.createdAt');
        }

        // History Store
        if (!db.objectStoreNames.contains('history')) {
          const historyStore = db.createObjectStore('history', { keyPath: 'id' });
          historyStore.createIndex('by-project', 'projectId');
          historyStore.createIndex('by-timestamp', ['projectId', 'timestamp']);
        }
      },
    });
  }

  /**
   * データの追加・更新
   */
  async put<T extends StoreName>(storeName: T, data: StoreData<T>): Promise<void> {
    await this.db!.put(storeName, data);
  }

  /**
   * データの取得
   */
  async get<T extends StoreName>(storeName: T, id: string): Promise<StoreData<T> | undefined> {
    return await this.db!.get(storeName, id);
  }

  /**
   * 全データの取得
   */
  async getAll<T extends StoreName>(storeName: T): Promise<StoreData<T>[]> {
    return await this.db!.getAll(storeName);
  }

  /**
   * インデックスによる検索
   */
  async getAllByIndex<T extends StoreName>(
    storeName: T,
    indexName: string,
    query?: IDBKeyRange | IDBValidKey
  ): Promise<StoreData<T>[]> {
    return await this.db!.getAllFromIndex(storeName, indexName, query);
  }

  /**
   * プロジェクトのスライドを取得（orderIndex順）
   */
  async getSlidesByProject(projectId: string): Promise<Slide[]> {
    const slides = await this.getAllByIndex('slides', 'by-project', projectId);
    return slides.sort((a, b) => a.orderIndex - b.orderIndex);
  }

  /**
   * プロジェクトのアセットを取得
   */
  async getAssetsByProject(projectId: string): Promise<Asset[]> {
    return await this.getAllByIndex('assets', 'by-project', projectId);
  }

  /**
   * データの削除
   */
  async delete<T extends StoreName>(storeName: T, id: string): Promise<void> {
    await this.db!.delete(storeName, id);
  }

  /**
   * プロジェクトとその関連データを削除
   */
  async deleteProject(projectId: string): Promise<void> {
    const tx = this.db!.transaction(['projects', 'slides', 'assets', 'history'], 'readwrite');

    // プロジェクト削除
    await tx.objectStore('projects').delete(projectId);

    // スライド削除
    const slides = await tx.objectStore('slides').index('by-project').getAllKeys(projectId);
    for (const key of slides) {
      await tx.objectStore('slides').delete(key);
    }

    // アセット削除（プロジェクト固有のみ）
    const assets = await tx.objectStore('assets').index('by-project').getAllKeys(projectId);
    for (const key of assets) {
      await tx.objectStore('assets').delete(key);
    }

    // 履歴削除
    const history = await tx.objectStore('history').index('by-project').getAllKeys(projectId);
    for (const key of history) {
      await tx.objectStore('history').delete(key);
    }

    await tx.done;
  }

  /**
   * バッチ保存（トランザクション）
   */
  async saveBatch<T extends StoreName>(
    storeName: T,
    items: StoreData<T>[]
  ): Promise<void> {
    const tx = this.db!.transaction(storeName, 'readwrite');
    await Promise.all(items.map((item) => tx.objectStore(storeName).put(item)));
    await tx.done;
  }

  /**
   * 容量の取得（概算）
   */
  async estimateSize(): Promise<StorageEstimate> {
    if ('storage' in navigator && 'estimate' in navigator.storage) {
      return await navigator.storage.estimate();
    }
    return { usage: 0, quota: 0 };
  }
}

export const indexedDBService = new IndexedDBService();
```

## localStorage設計

### データ構造

```typescript
// localStorage Keys (Prefixed)

const STORAGE_KEYS = {
  // アプリ設定
  APP_SETTINGS: 'app:settings',

  // AI設定
  AI_CONFIG: 'ai:config',

  // 最近のプロジェクト
  RECENT_PROJECTS: 'app:recent-projects',

  // UI状態
  UI_STATE: 'ui:state',

  // オンボーディング状態
  ONBOARDING: 'app:onboarding',
} as const;
```

### 詳細スキーマ

#### 1. アプリ設定

```typescript
interface AppSettings {
  language: 'ja' | 'en' | 'auto';
  theme: 'light' | 'dark' | 'auto';
  autoSave: boolean;
  autoSaveInterval: number; // seconds (default: 30)
  showGrid: boolean;
  gridSize: number; // pixels (default: 10)
  snapToGrid: boolean;
  defaultAspectRatio: '16:9' | '4:3' | '1:1';
  defaultResolution: {
    width: number;
    height: number;
  };
}

// デフォルト値
const DEFAULT_APP_SETTINGS: AppSettings = {
  language: 'auto',
  theme: 'auto',
  autoSave: true,
  autoSaveInterval: 30,
  showGrid: true,
  gridSize: 10,
  snapToGrid: true,
  defaultAspectRatio: '16:9',
  defaultResolution: { width: 1920, height: 1080 },
};
```

#### 2. AI設定

```typescript
interface AIConfig {
  llm: {
    provider: 'ollama' | 'lm-studio' | 'llama-cpp' | 'text-gen-webui';
    endpoint: string; // e.g., 'http://localhost:11434'
    model: string; // e.g., 'llama3.1:8b'
    temperature: number; // 0.0 - 1.0 (default: 0.7)
    maxTokens: number; // default: 500
    systemPrompt?: string;
  };
  imageGen: {
    provider: 'sd-webui' | 'comfyui' | 'none';
    endpoint: string; // e.g., 'http://localhost:7860'
    enabled: boolean;
    defaultParams: {
      steps: number; // default: 20
      width: number; // default: 512
      height: number; // default: 512
      cfgScale: number; // default: 7.0
      negativePrompt: string;
    };
  };
  tts: {
    provider: 'web-speech' | 'coqui' | 'piper' | 'voicevox' | 'none';
    endpoint?: string;
    enabled: boolean;
    defaultVoice?: string;
    defaultRate: number; // 0.5 - 2.0 (default: 1.0)
    defaultPitch: number; // 0.5 - 2.0 (default: 1.0)
  };
}

// デフォルト値
const DEFAULT_AI_CONFIG: AIConfig = {
  llm: {
    provider: 'ollama',
    endpoint: 'http://localhost:11434',
    model: 'llama3.1:8b',
    temperature: 0.7,
    maxTokens: 500,
  },
  imageGen: {
    provider: 'sd-webui',
    endpoint: 'http://localhost:7860',
    enabled: false,
    defaultParams: {
      steps: 20,
      width: 512,
      height: 512,
      cfgScale: 7.0,
      negativePrompt: 'blurry, low quality, distorted',
    },
  },
  tts: {
    provider: 'web-speech',
    enabled: false,
    defaultRate: 1.0,
    defaultPitch: 1.0,
  },
};
```

#### 3. 最近のプロジェクト

```typescript
interface RecentProject {
  id: string;
  name: string;
  lastOpened: Date;
  thumbnailDataUrl?: string; // Base64 encoded thumbnail
}

// 最大10件まで保持
const MAX_RECENT_PROJECTS = 10;

// localStorage に保存するデータ
type RecentProjectsData = RecentProject[];
```

#### 4. UI状態

```typescript
interface UIState {
  // エディタ
  sidebarCollapsed: boolean;
  sidebarWidth: number; // pixels
  zoom: number; // percentage (50-200)

  // テーマ
  selectedThemeId: string;

  // グリッド
  gridEnabled: boolean;

  // ツールバー
  selectedTool: 'select' | 'text' | 'image' | 'shape' | 'draw';

  // パネル
  inspectorTab: 'properties' | 'style' | 'animations';

  // その他
  lastOpenedProjectId?: string;
}
```

### localStorage操作クラス

```typescript
// services/storage/localStorageService.ts

class LocalStorageService {
  /**
   * データの保存
   */
  set<T>(key: string, value: T): void {
    try {
      const serialized = JSON.stringify(value);
      localStorage.setItem(key, serialized);
    } catch (error) {
      console.error('localStorage.setItem failed:', error);

      // 容量エラーの場合
      if (error instanceof DOMException && error.name === 'QuotaExceededError') {
        this.handleQuotaExceeded(key);
      }
    }
  }

  /**
   * データの取得
   */
  get<T>(key: string, defaultValue?: T): T | undefined {
    try {
      const item = localStorage.getItem(key);
      if (item === null) {
        return defaultValue;
      }
      return JSON.parse(item) as T;
    } catch (error) {
      console.error('localStorage.getItem failed:', error);
      return defaultValue;
    }
  }

  /**
   * データの削除
   */
  remove(key: string): void {
    localStorage.removeItem(key);
  }

  /**
   * すべてのデータをクリア
   */
  clear(): void {
    localStorage.clear();
  }

  /**
   * 容量超過時の処理
   */
  private handleQuotaExceeded(key: string): void {
    // 最近のプロジェクトリストを削減
    if (key === STORAGE_KEYS.RECENT_PROJECTS) {
      const recentProjects = this.get<RecentProject[]>(STORAGE_KEYS.RECENT_PROJECTS, []);
      const reduced = recentProjects.slice(0, 5); // 10 → 5 に削減
      this.set(STORAGE_KEYS.RECENT_PROJECTS, reduced);
    }

    // それでもダメな場合は警告
    console.warn('localStorage quota exceeded. Consider reducing data size.');
  }
}

export const localStorageService = new LocalStorageService();
```

## エクスポート/インポート仕様

### エクスポート形式

#### 1. JSON形式（小規模プロジェクト向け）

```typescript
interface ExportDataJSON {
  version: string; // e.g., '1.0.0'
  exportedAt: string; // ISO 8601
  type: 'single-project' | 'all-data';

  data: {
    projects: Project[];
    slides: Slide[];
    assets: ExportedAsset[]; // Blob → Base64変換済み
    templates?: CustomTemplate[];
    settings?: AppSettings;
  };

  metadata: {
    appVersion: string;
    projectCount: number;
    slideCount: number;
    assetCount: number;
    totalSize: number; // bytes
  };
}

interface ExportedAsset {
  id: string;
  projectId?: string;
  name: string;
  type: string;
  mimeType: string;
  dataUrl: string; // Base64 encoded (data:image/png;base64,...)
  thumbnailDataUrl?: string;
  size: number;
  dimensions?: { width: number; height: number };
  metadata: Asset['metadata'];
}
```

**JSON形式の使用例**:
```json
{
  "version": "1.0.0",
  "exportedAt": "2025-10-24T15:30:00Z",
  "type": "single-project",
  "data": {
    "projects": [
      {
        "id": "proj_a1b2c3d4",
        "name": "2025年度 事業計画",
        "description": "新規事業の提案資料",
        "themeId": "theme_modern_blue",
        "settings": { ... },
        "metadata": { ... }
      }
    ],
    "slides": [ ... ],
    "assets": [
      {
        "id": "asset_img_001",
        "projectId": "proj_a1b2c3d4",
        "name": "company-logo.png",
        "type": "image",
        "mimeType": "image/png",
        "dataUrl": "data:image/png;base64,iVBORw0KGgoAAAANS...",
        "size": 150000,
        "dimensions": { "width": 1200, "height": 600 },
        "metadata": { ... }
      }
    ]
  },
  "metadata": {
    "appVersion": "1.0.0",
    "projectCount": 1,
    "slideCount": 15,
    "assetCount": 8,
    "totalSize": 2500000
  }
}
```

#### 2. ZIP形式（大規模プロジェクト向け、推奨）

```
project-export-2025-10-24.zip
├── manifest.json          # メタデータ
├── projects/
│   └── proj_a1b2c3d4.json
├── slides/
│   ├── slide_001.json
│   ├── slide_002.json
│   └── ...
├── assets/
│   ├── asset_img_001.png
│   ├── asset_img_002.jpg
│   ├── asset_svg_001.svg
│   └── ...
├── templates/             # カスタムテンプレート（オプション）
│   └── template_001.json
└── settings.json          # アプリ設定（オプション）
```

**manifest.json**:
```json
{
  "version": "1.0.0",
  "exportedAt": "2025-10-24T15:30:00Z",
  "type": "single-project",
  "appVersion": "1.0.0",
  "metadata": {
    "projectCount": 1,
    "slideCount": 15,
    "assetCount": 8,
    "totalSize": 2500000
  },
  "files": {
    "projects": ["projects/proj_a1b2c3d4.json"],
    "slides": ["slides/slide_001.json", "slides/slide_002.json", ...],
    "assets": ["assets/asset_img_001.png", ...]
  }
}
```

### エクスポート実装

```typescript
// services/export/exportService.ts

import JSZip from 'jszip';

export class ExportService {
  /**
   * JSON形式でエクスポート
   */
  static async exportAsJSON(projectId?: string): Promise<Blob> {
    const storageManager = new StorageManager();

    // データ収集
    const projects = projectId
      ? [await storageManager.getProject(projectId)]
      : await storageManager.getAllProjects();

    const slides = projectId
      ? await storageManager.getSlidesByProject(projectId)
      : await storageManager.getAllSlides();

    const assets = projectId
      ? await storageManager.getAssetsByProject(projectId)
      : await storageManager.getAllAssets();

    // アセットをBase64に変換
    const exportedAssets = await Promise.all(
      assets.map(async (asset) => ({
        ...asset,
        dataUrl: await this.blobToDataURL(asset.blob),
        thumbnailDataUrl: asset.thumbnailBlob
          ? await this.blobToDataURL(asset.thumbnailBlob)
          : undefined,
        blob: undefined, // Remove blob from export
        thumbnailBlob: undefined,
      }))
    );

    const exportData: ExportDataJSON = {
      version: '1.0.0',
      exportedAt: new Date().toISOString(),
      type: projectId ? 'single-project' : 'all-data',
      data: {
        projects,
        slides,
        assets: exportedAssets,
      },
      metadata: {
        appVersion: '1.0.0', // TODO: Get from package.json
        projectCount: projects.length,
        slideCount: slides.length,
        assetCount: assets.length,
        totalSize: exportedAssets.reduce((sum, a) => sum + a.size, 0),
      },
    };

    const json = JSON.stringify(exportData, null, 2);
    return new Blob([json], { type: 'application/json' });
  }

  /**
   * ZIP形式でエクスポート
   */
  static async exportAsZIP(projectId?: string): Promise<Blob> {
    const storageManager = new StorageManager();
    const zip = new JSZip();

    // データ収集
    const projects = projectId
      ? [await storageManager.getProject(projectId)]
      : await storageManager.getAllProjects();

    const slides = projectId
      ? await storageManager.getSlidesByProject(projectId)
      : await storageManager.getAllSlides();

    const assets = projectId
      ? await storageManager.getAssetsByProject(projectId)
      : await storageManager.getAllAssets();

    // Manifest作成
    const manifest = {
      version: '1.0.0',
      exportedAt: new Date().toISOString(),
      type: projectId ? 'single-project' : 'all-data',
      appVersion: '1.0.0',
      metadata: {
        projectCount: projects.length,
        slideCount: slides.length,
        assetCount: assets.length,
        totalSize: assets.reduce((sum, a) => sum + a.size, 0),
      },
      files: {
        projects: projects.map((p) => `projects/${p.id}.json`),
        slides: slides.map((s) => `slides/${s.id}.json`),
        assets: assets.map((a) => `assets/${a.id}.${this.getExtension(a.mimeType)}`),
      },
    };

    zip.file('manifest.json', JSON.stringify(manifest, null, 2));

    // プロジェクト
    const projectsFolder = zip.folder('projects')!;
    for (const project of projects) {
      projectsFolder.file(`${project.id}.json`, JSON.stringify(project, null, 2));
    }

    // スライド
    const slidesFolder = zip.folder('slides')!;
    for (const slide of slides) {
      slidesFolder.file(`${slide.id}.json`, JSON.stringify(slide, null, 2));
    }

    // アセット
    const assetsFolder = zip.folder('assets')!;
    for (const asset of assets) {
      const extension = this.getExtension(asset.mimeType);
      assetsFolder.file(`${asset.id}.${extension}`, asset.blob);
    }

    // ZIP生成
    return await zip.generateAsync({
      type: 'blob',
      compression: 'DEFLATE',
      compressionOptions: { level: 6 },
    });
  }

  /**
   * Blob to Data URL
   */
  private static blobToDataURL(blob: Blob): Promise<string> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onloadend = () => resolve(reader.result as string);
      reader.onerror = reject;
      reader.readAsDataURL(blob);
    });
  }

  /**
   * MIME Type から拡張子を取得
   */
  private static getExtension(mimeType: string): string {
    const map: Record<string, string> = {
      'image/png': 'png',
      'image/jpeg': 'jpg',
      'image/jpg': 'jpg',
      'image/gif': 'gif',
      'image/webp': 'webp',
      'image/svg+xml': 'svg',
      'video/mp4': 'mp4',
      'audio/mp3': 'mp3',
    };
    return map[mimeType] || 'bin';
  }
}
```

### インポート実装

```typescript
// services/export/importService.ts

import JSZip from 'jszip';

export class ImportService {
  /**
   * ファイル形式の判定とインポート
   */
  static async import(file: File): Promise<ImportResult> {
    if (file.name.endsWith('.json')) {
      return await this.importJSON(file);
    } else if (file.name.endsWith('.zip')) {
      return await this.importZIP(file);
    } else {
      throw new Error(`Unsupported file format: ${file.name}`);
    }
  }

  /**
   * JSON形式のインポート
   */
  private static async importJSON(file: File): Promise<ImportResult> {
    const text = await file.text();
    const data: ExportDataJSON = JSON.parse(text);

    // バージョンチェック
    if (!this.isCompatibleVersion(data.version)) {
      throw new Error(`Incompatible version: ${data.version}`);
    }

    const storageManager = new StorageManager();

    // プロジェクトをインポート
    for (const project of data.data.projects) {
      await storageManager.saveProject(project);
    }

    // スライドをインポート
    for (const slide of data.data.slides) {
      await storageManager.saveSlide(slide);
    }

    // アセットをインポート（Base64 → Blob変換）
    for (const asset of data.data.assets) {
      const blob = await this.dataURLToBlob(asset.dataUrl);
      const thumbnailBlob = asset.thumbnailDataUrl
        ? await this.dataURLToBlob(asset.thumbnailDataUrl)
        : undefined;

      await storageManager.saveAsset({
        ...asset,
        blob,
        thumbnailBlob,
      });
    }

    return {
      success: true,
      projectCount: data.data.projects.length,
      slideCount: data.data.slides.length,
      assetCount: data.data.assets.length,
    };
  }

  /**
   * ZIP形式のインポート
   */
  private static async importZIP(file: File): Promise<ImportResult> {
    const zip = await JSZip.loadAsync(file);

    // Manifest読み込み
    const manifestFile = zip.file('manifest.json');
    if (!manifestFile) {
      throw new Error('Invalid ZIP: manifest.json not found');
    }

    const manifestText = await manifestFile.async('string');
    const manifest = JSON.parse(manifestText);

    // バージョンチェック
    if (!this.isCompatibleVersion(manifest.version)) {
      throw new Error(`Incompatible version: ${manifest.version}`);
    }

    const storageManager = new StorageManager();

    // プロジェクトをインポート
    for (const projectPath of manifest.files.projects) {
      const file = zip.file(projectPath);
      if (file) {
        const text = await file.async('string');
        const project = JSON.parse(text);
        await storageManager.saveProject(project);
      }
    }

    // スライドをインポート
    for (const slidePath of manifest.files.slides) {
      const file = zip.file(slidePath);
      if (file) {
        const text = await file.async('string');
        const slide = JSON.parse(text);
        await storageManager.saveSlide(slide);
      }
    }

    // アセットをインポート
    for (const assetPath of manifest.files.assets) {
      const file = zip.file(assetPath);
      if (file) {
        const blob = await file.async('blob');

        // メタデータは assetPath から asset ID を取得して別途読み込み
        // （実装簡略化のため、ここでは省略）

        await storageManager.saveAsset({
          // ... asset metadata
          blob,
        });
      }
    }

    return {
      success: true,
      projectCount: manifest.metadata.projectCount,
      slideCount: manifest.metadata.slideCount,
      assetCount: manifest.metadata.assetCount,
    };
  }

  /**
   * バージョン互換性チェック
   */
  private static isCompatibleVersion(version: string): boolean {
    // 1.x.x は互換性あり
    return version.startsWith('1.');
  }

  /**
   * Data URL to Blob
   */
  private static async dataURLToBlob(dataUrl: string): Promise<Blob> {
    const response = await fetch(dataUrl);
    return await response.blob();
  }
}

interface ImportResult {
  success: boolean;
  projectCount: number;
  slideCount: number;
  assetCount: number;
  errors?: string[];
}
```

## データマイグレーション

### バージョン管理

```typescript
// Database version
const DB_VERSION = 1;

// Migration definitions
const MIGRATIONS: Record<number, MigrationFunction> = {
  1: migrate_v1,
  // 2: migrate_v2, // 将来のバージョン
};

type MigrationFunction = (
  db: IDBDatabase,
  transaction: IDBTransaction
) => void;

/**
 * Version 1: 初期バージョン
 */
function migrate_v1(db: IDBDatabase, transaction: IDBTransaction): void {
  // Object Stores作成（上記の init() と同じ）
  // ...
}

/**
 * Version 2（将来）: 新しいフィールド追加など
 */
// function migrate_v2(db: IDBDatabase, transaction: IDBTransaction): void {
//   // 既存のObject Storeにインデックス追加など
//   const slideStore = transaction.objectStore('slides');
//   slideStore.createIndex('by-tags', 'metadata.tags', { multiEntry: true });
// }
```

## 容量管理

### 容量監視

```typescript
export class StorageManager {
  /**
   * 使用容量の取得
   */
  async getUsage(): Promise<StorageUsage> {
    const estimate = await navigator.storage.estimate();

    return {
      usage: estimate.usage || 0, // bytes
      quota: estimate.quota || 0, // bytes
      percentage: estimate.usage && estimate.quota
        ? (estimate.usage / estimate.quota) * 100
        : 0,
    };
  }

  /**
   * 容量警告の確認
   */
  async shouldWarnUser(): Promise<boolean> {
    const usage = await this.getUsage();
    return usage.percentage > 80; // 80%超えたら警告
  }

  /**
   * 古い履歴データのクリーンアップ
   */
  async cleanupHistory(projectId: string, keepCount: number = 100): Promise<void> {
    const db = await indexedDBService.init();
    const historyEntries = await indexedDBService.getAllByIndex(
      'history',
      'by-project',
      projectId
    );

    // 古い順にソート
    historyEntries.sort((a, b) => a.timestamp.getTime() - b.timestamp.getTime());

    // 保持数を超えた分を削除
    const toDelete = historyEntries.slice(0, historyEntries.length - keepCount);
    for (const entry of toDelete) {
      await indexedDBService.delete('history', entry.id);
    }
  }
}

interface StorageUsage {
  usage: number; // bytes
  quota: number; // bytes
  percentage: number; // 0-100
}
```

## エラーハンドリング

### 主なエラーと対処

```typescript
export class StorageError extends Error {
  constructor(
    message: string,
    public code: StorageErrorCode,
    public originalError?: Error
  ) {
    super(message);
    this.name = 'StorageError';
  }
}

export enum StorageErrorCode {
  QUOTA_EXCEEDED = 'QUOTA_EXCEEDED',
  NOT_FOUND = 'NOT_FOUND',
  CORRUPTED_DATA = 'CORRUPTED_DATA',
  MIGRATION_FAILED = 'MIGRATION_FAILED',
  IMPORT_FAILED = 'IMPORT_FAILED',
  EXPORT_FAILED = 'EXPORT_FAILED',
}

// 使用例
try {
  await storageManager.saveProject(project);
} catch (error) {
  if (error instanceof StorageError) {
    switch (error.code) {
      case StorageErrorCode.QUOTA_EXCEEDED:
        // 容量不足の処理
        showQuotaExceededDialog();
        break;
      case StorageErrorCode.CORRUPTED_DATA:
        // データ破損の処理
        showDataCorruptionDialog();
        break;
      default:
        // その他のエラー
        showGenericErrorDialog(error.message);
    }
  }
}
```

---

**最終更新**: 2025-10-24
**レビュー**: 未実施
**次のステップ**: DATA_MODEL.md, API.md, FEATURES.md の更新
