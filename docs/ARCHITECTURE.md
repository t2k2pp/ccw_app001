# システムアーキテクチャ設計書

## 目次
1. [システム概要](#システム概要)
2. [設計原則](#設計原則)
3. [アーキテクチャ図](#アーキテクチャ図)
4. [ストレージ戦略](#ストレージ戦略)
5. [レイヤー構成](#レイヤー構成)
6. [コンポーネント設計](#コンポーネント設計)
7. [データフロー](#データフロー)
8. [技術スタック詳細](#技術スタック詳細)
9. [デプロイメント戦略](#デプロイメント戦略)
10. [セキュリティ設計](#セキュリティ設計)
11. [パフォーマンス設計](#パフォーマンス設計)

## システム概要

Local AI Slide Creatorは、**完全にクライアント中心**のアーキテクチャを採用したAI支援スライド作成アプリケーションです。

### 基本コンセプト

```
┌──────────────────────────────────────────────────┐
│  クライアント中心設計（Client-First Architecture）│
├──────────────────────────────────────────────────┤
│  ✅ ユーザーデータ: 100% クライアント側            │
│  ✅ ビジネスロジック: 主にクライアント側           │
│  ✅ サーバー: 最小限（AI Proxy + システムデータ）  │
│  ✅ オフライン動作: 完全対応（AI以外）             │
└──────────────────────────────────────────────────┘
```

### アーキテクチャの特徴

- **プライバシー最優先**: ユーザーデータはサーバーに送信・保存されない
- **完全オフライン動作**: AI機能以外はネットワーク不要
- **セキュリティリスク最小化**: データ漏洩リスクゼロ
- **軽量サーバー**: AI Proxyとシステムデータ提供のみ
- **エクスポート/インポート**: 環境間のデータ移行が容易

## 設計原則

### 1. データ所有権の原則
```
ユーザーデータ = ユーザーのブラウザ/デバイス内のみ
システムデータ = サーバー側（読み取り専用）
```

### 2. 責務分離の原則
```
Client Side:
  - ユーザーデータの永続化（IndexedDB）
  - ビジネスロジックの実行
  - UI/UX処理
  - エクスポート/インポート

Server Side:
  - AI API Proxy
  - システムデータ提供（テンプレート、アイコン等）
  - キャッシュ管理
```

### 3. オフライン優先の原則
```
すべての機能は可能な限りオフラインで動作
AI機能のみネットワーク必須（ローカルAIサーバーへ接続）
```

## アーキテクチャ図

### 全体アーキテクチャ

```
┌──────────────────────────────────────────────────────────────────┐
│                    User's Browser / Electron App                 │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                  React Frontend (SPA)                      │ │
│  │                                                            │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │ │
│  │  │  Slide   │  │ Template │  │  Asset   │  │  Export  │  │ │
│  │  │  Editor  │  │ Manager  │  │ Manager  │  │ Manager  │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │ │
│  │  │  Signage │  │   TTS    │  │   AI     │  │ Settings │  │ │
│  │  │   Mode   │  │Controller│  │  Client  │  │    UI    │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │           Client-Side Storage & Business Logic            │ │
│  │                                                            │ │
│  │  ┌───────────────────┐    ┌────────────────────────────┐  │ │
│  │  │  localStorage     │    │      IndexedDB             │  │ │
│  │  │  ───────────────  │    │  ────────────────────────  │  │ │
│  │  │  • Settings       │    │  • Projects (JSON)         │  │ │
│  │  │  • UI State       │    │  • Slides (JSON)           │  │ │
│  │  │  • AI Config      │    │  • Assets (Blob)           │  │ │
│  │  │  • Theme Prefs    │    │  • Templates (Custom)      │  │ │
│  │  │  • Recent Items   │    │  • History (Undo/Redo)     │  │ │
│  │  └───────────────────┘    └────────────────────────────┘  │ │
│  │                                                            │ │
│  │  ┌────────────────────────────────────────────────────┐   │ │
│  │  │         Business Logic (Client-Side)               │   │ │
│  │  │  • Layout Engine  • Validation  • Export Logic    │   │ │
│  │  └────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
                              │
                    HTTP (AI API & System Data)
                              │
┌──────────────────────────────────────────────────────────────────┐
│              Lightweight Backend (Optional)                      │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │            Express Server (Node.js + TypeScript)           │ │
│  │                                                            │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │ │
│  │  │   AI Proxy   │  │   System     │  │   Cache      │    │ │
│  │  │  Controller  │  │    Data      │  │   Manager    │    │ │
│  │  │              │  │   Provider   │  │              │    │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              SQLite (System Data Only)                     │ │
│  │                                                            │ │
│  │  • Preset Templates (Read-Only)                           │ │
│  │  • Icon Library Metadata (Read-Only)                      │ │
│  │  • Default Themes (Read-Only)                             │ │
│  │  • AI Response Cache (Temporary)                          │ │
│  │  • Usage Statistics (Optional)                            │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
                              │
                         HTTP / REST
                              │
┌──────────────────────────────────────────────────────────────────┐
│              Local AI Services (User's Machine)                  │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Ollama     │  │  SD WebUI    │  │  Coqui TTS   │          │
│  │   :11434     │  │    :7860     │  │    :5002     │          │
│  │   (GPU)      │  │    (GPU)     │  │   (CPU/GPU)  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└──────────────────────────────────────────────────────────────────┘
```

## ストレージ戦略

### データ分類と保存先

| データカテゴリ | 保存先 | 容量制限 | 読み書き | 用途 |
|--------------|--------|---------|---------|------|
| **アプリ設定** | localStorage | ~5MB | Read/Write | UI状態、AI接続設定 |
| **プロジェクトデータ** | IndexedDB | ~50MB-数GB | Read/Write | スライド、要素、メタデータ |
| **アセット（画像等）** | IndexedDB (Blob) | ~数GB | Read/Write | 画像、SVG、動画 |
| **カスタムテンプレート** | IndexedDB | ~10MB | Read/Write | ユーザー作成テンプレート |
| **履歴データ** | IndexedDB | ~100MB | Read/Write | Undo/Redo、編集履歴 |
| **プリセットテンプレート** | Server SQLite | - | Read-Only | システム提供テンプレート |
| **アイコンライブラリ** | Server SQLite | - | Read-Only | Heroicons, Feather等 |
| **AIキャッシュ** | Server SQLite | ~1GB | Read/Write | AI生成結果のキャッシュ |

### localStorage 構造

```typescript
// localStorage Schema
interface LocalStorageData {
  // アプリ設定
  'app:settings': {
    language: 'ja' | 'en';
    theme: 'light' | 'dark' | 'auto';
    autoSave: boolean;
    autoSaveInterval: number; // seconds
  };

  // AI設定
  'ai:config': {
    llm: {
      provider: 'ollama' | 'lm-studio' | 'llama-cpp';
      endpoint: string;
      model: string;
      temperature: number;
      maxTokens: number;
    };
    imageGen: {
      provider: 'sd-webui' | 'comfyui';
      endpoint: string;
      enabled: boolean;
    };
    tts: {
      provider: 'web-speech' | 'coqui' | 'piper';
      endpoint?: string;
      enabled: boolean;
    };
  };

  // 最近使用したプロジェクト
  'app:recent-projects': string[]; // project IDs

  // UI状態
  'ui:state': {
    sidebarCollapsed: boolean;
    selectedThemeId: string;
    zoom: number;
    gridEnabled: boolean;
  };
}
```

### IndexedDB スキーマ

```typescript
// IndexedDB Database: 'LocalAISlideCreator'
// Version: 1

interface IndexedDBSchema {
  // Object Store: projects
  projects: {
    key: string; // project_id (UUID)
    value: {
      id: string;
      name: string;
      description?: string;
      themeId: string;
      settings: ProjectSettings;
      metadata: {
        createdAt: Date;
        updatedAt: Date;
        slideCount: number;
        totalSize: number; // bytes
      };
    };
    indexes: {
      'by-updated': Date;
      'by-created': Date;
      'by-name': string;
    };
  };

  // Object Store: slides
  slides: {
    key: string; // slide_id (UUID)
    value: {
      id: string;
      projectId: string;
      templateId?: string;
      title: string;
      notes?: string;
      talkScript?: string;
      orderIndex: number;
      elements: Element[];
      background: Background;
      displayDuration: number; // for signage mode
      transition: TransitionSettings;
      voiceSettings?: VoiceSettings;
      metadata: {
        createdAt: Date;
        updatedAt: Date;
      };
    };
    indexes: {
      'by-project': string; // projectId
      'by-order': [string, number]; // [projectId, orderIndex]
    };
  };

  // Object Store: assets
  assets: {
    key: string; // asset_id (UUID)
    value: {
      id: string;
      projectId?: string; // null = global asset
      name: string;
      type: 'image' | 'svg' | 'video' | 'audio';
      mimeType: string;
      blob: Blob; // binary data
      thumbnailBlob?: Blob; // thumbnail for preview
      size: number; // bytes
      dimensions?: { width: number; height: number };
      metadata: {
        createdAt: Date;
        source?: 'upload' | 'ai-generated' | 'icon-library';
        aiPrompt?: string; // if AI-generated
      };
    };
    indexes: {
      'by-project': string;
      'by-type': string;
      'by-created': Date;
    };
  };

  // Object Store: templates (custom templates only)
  templates: {
    key: string; // template_id (UUID)
    value: {
      id: string;
      name: string;
      description?: string;
      category: string;
      thumbnailAssetId?: string;
      layout: LayoutDefinition;
      isCustom: true; // always true (preset templates are from server)
      metadata: {
        createdAt: Date;
        updatedAt: Date;
        usageCount: number;
      };
    };
    indexes: {
      'by-category': string;
      'by-created': Date;
    };
  };

  // Object Store: history (undo/redo)
  history: {
    key: string; // history_id (UUID)
    value: {
      id: string;
      projectId: string;
      action: string; // 'create_slide', 'delete_element', etc.
      snapshot: any; // state snapshot
      timestamp: Date;
    };
    indexes: {
      'by-project': string;
      'by-timestamp': [string, Date]; // [projectId, timestamp]
    };
    // Auto-cleanup: keep last 100 entries per project
  };
}
```

### Server-Side SQLite スキーマ

```sql
-- System Data Only (Read-Only for clients)

-- Preset Templates
CREATE TABLE templates (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT,
  category TEXT NOT NULL,
  thumbnail_url TEXT,
  layout_json TEXT NOT NULL, -- JSON string
  is_system BOOLEAN NOT NULL DEFAULT 1,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_templates_category ON templates(category);

-- Icon Library Metadata
CREATE TABLE icon_library (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  icon_set TEXT NOT NULL, -- 'heroicons', 'feather', etc.
  category TEXT,
  tags TEXT, -- comma-separated
  svg_content TEXT NOT NULL,
  keywords TEXT, -- for search
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_icons_set ON icon_library(icon_set);
CREATE INDEX idx_icons_category ON icon_library(category);

-- Full-Text Search for icons
CREATE VIRTUAL TABLE icon_library_fts USING fts5(
  name, tags, keywords, content=icon_library
);

-- Default Themes
CREATE TABLE themes (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT,
  colors_json TEXT NOT NULL,
  fonts_json TEXT NOT NULL,
  is_system BOOLEAN NOT NULL DEFAULT 1,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- AI Response Cache
CREATE TABLE ai_cache (
  id TEXT PRIMARY KEY,
  cache_key TEXT NOT NULL UNIQUE, -- hash(prompt + params)
  prompt TEXT NOT NULL,
  params_json TEXT NOT NULL,
  response_json TEXT NOT NULL,
  provider TEXT NOT NULL, -- 'ollama', 'sd-webui', etc.
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  accessed_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  access_count INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_cache_key ON ai_cache(cache_key);
CREATE INDEX idx_cache_accessed ON ai_cache(accessed_at);

-- Auto-cleanup: DELETE WHERE accessed_at < NOW() - 30 days

-- Usage Statistics (Optional)
CREATE TABLE usage_stats (
  id TEXT PRIMARY KEY,
  event_type TEXT NOT NULL, -- 'template_used', 'slide_created', etc.
  data_json TEXT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_stats_type ON usage_stats(event_type);
CREATE INDEX idx_stats_created ON usage_stats(created_at);
```

## レイヤー構成

### 1. Frontend Layer（メインレイヤー）

**責務**:
- ✅ ユーザーインターフェース
- ✅ **ビジネスロジックの実行**（クライアント側）
- ✅ **データ永続化**（IndexedDB/localStorage）
- ✅ AI API呼び出し（サーバー経由またはダイレクト）
- ✅ エクスポート/インポート処理

**技術スタック**:
- React 18+ (TypeScript)
- Konva.js（キャンバスエディタ）
- Zustand（状態管理）
- TailwindCSS（スタイリング）
- idb（IndexedDB wrapper）
- JSZip（エクスポート/インポート）

**ディレクトリ構造**:
```
src/
├── components/              # UIコンポーネント
│   ├── editor/
│   │   ├── Canvas/         # Konvaキャンバス
│   │   ├── Toolbar/
│   │   ├── Sidebar/
│   │   └── Inspector/
│   ├── slides/
│   │   ├── SlideList/
│   │   ├── SlidePreview/
│   │   └── SlideView/
│   ├── templates/
│   │   ├── TemplateGallery/
│   │   └── TemplateCard/
│   ├── assets/
│   │   ├── AssetManager/
│   │   ├── AssetUpload/
│   │   └── IconPicker/
│   ├── export/
│   │   ├── ExportDialog/
│   │   └── ImportDialog/
│   ├── signage/
│   │   └── SignageMode/
│   └── common/
│       ├── Button/
│       ├── Modal/
│       └── Toast/
│
├── stores/                  # Zustand stores
│   ├── projectStore.ts      # プロジェクト管理
│   ├── slideStore.ts        # スライド管理
│   ├── assetStore.ts        # アセット管理
│   ├── templateStore.ts     # テンプレート管理
│   ├── historyStore.ts      # Undo/Redo
│   └── settingsStore.ts     # アプリ設定
│
├── services/                # ビジネスロジック（クライアント側）
│   ├── storage/
│   │   ├── indexedDBService.ts   # IndexedDB操作
│   │   ├── localStorageService.ts
│   │   └── storageManager.ts     # 統合管理
│   ├── export/
│   │   ├── exportService.ts      # エクスポート処理
│   │   ├── importService.ts      # インポート処理
│   │   ├── pdfExporter.ts
│   │   ├── pptxExporter.ts
│   │   └── htmlExporter.ts
│   ├── ai/
│   │   ├── aiClient.ts           # AI API Client
│   │   ├── llmService.ts
│   │   ├── imageGenService.ts
│   │   └── ttsService.ts
│   ├── layout/
│   │   ├── layoutEngine.ts       # レイアウト計算
│   │   └── layoutSelector.ts     # レイアウト自動選択
│   └── validation/
│       └── schemaValidator.ts
│
├── repositories/            # データアクセス層（クライアント側）
│   ├── projectRepository.ts
│   ├── slideRepository.ts
│   ├── assetRepository.ts
│   ├── templateRepository.ts
│   └── historyRepository.ts
│
├── hooks/                   # カスタムフック
│   ├── useProject.ts
│   ├── useSlide.ts
│   ├── useAsset.ts
│   ├── useTemplate.ts
│   ├── useExport.ts
│   ├── useImport.ts
│   ├── useAI.ts
│   └── useHistory.ts        # Undo/Redo
│
├── types/                   # 型定義
│   ├── project.ts
│   ├── slide.ts
│   ├── element.ts
│   ├── template.ts
│   ├── theme.ts
│   ├── asset.ts
│   └── common.ts
│
└── utils/
    ├── canvas/              # キャンバス操作
    ├── layout/              # レイアウト計算
    ├── export/              # エクスポートヘルパー
    └── helpers/
```

### 2. Backend Layer（最小限）

**責務**:
- ✅ AI API Proxy（CORS対策、リトライ処理）
- ✅ システムデータ提供（テンプレート、アイコン）
- ✅ AIレスポンスキャッシュ
- ❌ ユーザーデータ保存（行わない）
- ❌ 認証・認可（不要）

**技術スタック**:
- Node.js 20 LTS
- Express 4.18+
- TypeScript 5.0+
- better-sqlite3（SQLite）
- Zod（バリデーション）

**ディレクトリ構造**:
```
server/
├── controllers/
│   ├── aiProxyController.ts      # AI Proxy
│   ├── templateController.ts     # テンプレート提供
│   ├── iconController.ts         # アイコンライブラリ提供
│   └── themeController.ts        # テーマ提供
│
├── services/
│   ├── aiProxyService.ts         # AI API呼び出し
│   ├── cacheService.ts           # キャッシュ管理
│   └── searchService.ts          # FTS検索
│
├── repositories/
│   ├── templateRepository.ts     # SQLite読み取り
│   ├── iconRepository.ts
│   ├── themeRepository.ts
│   └── cacheRepository.ts
│
├── middleware/
│   ├── cors.ts
│   ├── errorHandler.ts
│   └── rateLimit.ts
│
├── routes/
│   ├── ai.ts                     # /api/ai/*
│   ├── templates.ts              # /api/templates/*
│   ├── icons.ts                  # /api/icons/*
│   └── themes.ts                 # /api/themes/*
│
├── db/
│   ├── schema.sql
│   ├── seeds/
│   │   ├── templates.sql
│   │   ├── icons.sql
│   │   └── themes.sql
│   └── connection.ts
│
└── index.ts
```

## コンポーネント設計

### 主要コンポーネント

#### 1. Storage Manager（クライアント側）

```typescript
// services/storage/storageManager.ts

export class StorageManager {
  private indexedDB: IndexedDBService;
  private localStorage: LocalStorageService;

  constructor() {
    this.indexedDB = new IndexedDBService();
    this.localStorage = new LocalStorageService();
  }

  // プロジェクトの保存
  async saveProject(project: Project): Promise<void> {
    await this.indexedDB.put('projects', project);
    this.updateRecentProjects(project.id);
  }

  // スライドの保存
  async saveSlide(slide: Slide): Promise<void> {
    await this.indexedDB.put('slides', slide);
  }

  // アセットの保存（Blob）
  async saveAsset(asset: Asset, blob: Blob): Promise<void> {
    const assetData = {
      ...asset,
      blob,
      size: blob.size,
    };
    await this.indexedDB.put('assets', assetData);
  }

  // 全データのエクスポート
  async exportAllData(): Promise<Blob> {
    const projects = await this.indexedDB.getAll('projects');
    const slides = await this.indexedDB.getAll('slides');
    const assets = await this.indexedDB.getAll('assets');

    // ZIP形式でエクスポート
    return await ExportService.createZIP({
      projects,
      slides,
      assets,
    });
  }

  // データのインポート
  async importData(file: File): Promise<ImportResult> {
    const data = await ImportService.parse(file);

    // IndexedDBに保存
    for (const project of data.projects) {
      await this.saveProject(project);
    }
    for (const slide of data.slides) {
      await this.saveSlide(slide);
    }
    for (const asset of data.assets) {
      await this.saveAsset(asset.metadata, asset.blob);
    }

    return {
      success: true,
      projectCount: data.projects.length,
      slideCount: data.slides.length,
      assetCount: data.assets.length,
    };
  }
}
```

#### 2. Export/Import Service

```typescript
// services/export/exportService.ts

export class ExportService {
  // JSON形式でエクスポート
  static async exportAsJSON(projectId?: string): Promise<Blob> {
    const storageManager = new StorageManager();

    const data: ExportData = {
      version: '1.0.0',
      exportedAt: new Date().toISOString(),
      type: projectId ? 'single-project' : 'all-data',
      data: {
        projects: projectId
          ? [await storageManager.getProject(projectId)]
          : await storageManager.getAllProjects(),
        slides: projectId
          ? await storageManager.getSlidesByProject(projectId)
          : await storageManager.getAllSlides(),
        assets: projectId
          ? await storageManager.getAssetsByProject(projectId)
          : await storageManager.getAllAssets(),
      },
    };

    // アセットをBase64に変換
    const jsonData = await this.convertAssetsToBase64(data);
    const json = JSON.stringify(jsonData, null, 2);

    return new Blob([json], { type: 'application/json' });
  }

  // ZIP形式でエクスポート
  static async exportAsZIP(projectId?: string): Promise<Blob> {
    const storageManager = new StorageManager();
    const zip = new JSZip();

    // メタデータ
    const data = await this.gatherData(storageManager, projectId);
    zip.file('data.json', JSON.stringify(data.metadata, null, 2));

    // プロジェクト
    const projectsFolder = zip.folder('projects');
    for (const project of data.projects) {
      projectsFolder.file(`${project.id}.json`, JSON.stringify(project, null, 2));
    }

    // スライド
    const slidesFolder = zip.folder('slides');
    for (const slide of data.slides) {
      slidesFolder.file(`${slide.id}.json`, JSON.stringify(slide, null, 2));
    }

    // アセット（バイナリファイル）
    const assetsFolder = zip.folder('assets');
    for (const asset of data.assets) {
      const extension = asset.mimeType.split('/')[1];
      assetsFolder.file(`${asset.id}.${extension}`, asset.blob);
    }

    return await zip.generateAsync({
      type: 'blob',
      compression: 'DEFLATE',
      compressionOptions: { level: 6 },
    });
  }
}
```

#### 3. Layout Engine（クライアント側）

```typescript
// services/layout/layoutEngine.ts

export class LayoutEngine {
  /**
   * コンテンツに基づいて最適なレイアウトを選択
   */
  selectLayout(content: SlideContent): LayoutTemplate {
    const score = this.calculateLayoutScores(content);

    // スコアが最も高いレイアウトを選択
    const bestLayout = this.templates.reduce((best, current) => {
      return score[current.id] > score[best.id] ? current : best;
    });

    return bestLayout;
  }

  /**
   * レイアウトスコアの計算
   */
  private calculateLayoutScores(content: SlideContent): Record<string, number> {
    const scores: Record<string, number> = {};

    for (const template of this.templates) {
      let score = 0;

      // テキスト量
      const textLength = content.text?.length || 0;
      if (textLength < 100 && template.layout.textAreas.length === 1) {
        score += 30; // 短文は1カラム
      } else if (textLength > 300 && template.layout.textAreas.length > 1) {
        score += 30; // 長文は複数カラム
      }

      // 画像数
      const imageCount = content.images?.length || 0;
      if (imageCount === 0 && template.layout.imageAreas.length === 0) {
        score += 20; // 画像なしはテキストのみレイアウト
      } else if (imageCount > 0 && template.layout.imageAreas.length > 0) {
        score += 20;
      }

      // 箇条書き
      const hasBullets = content.bullets && content.bullets.length > 0;
      if (hasBullets && template.layout.supportsBullets) {
        score += 15;
      }

      scores[template.id] = score;
    }

    return scores;
  }
}
```

#### 4. AI Client（サーバー経由またはダイレクト）

```typescript
// services/ai/aiClient.ts

export class AIClient {
  private config: AIConfig;

  constructor(config: AIConfig) {
    this.config = config;
  }

  /**
   * LLMでコンテンツ生成
   */
  async generateContent(params: GenerateParams): Promise<string> {
    // サーバー経由（CORS対策、キャッシュ）
    if (this.config.useProxy) {
      const response = await fetch('/api/ai/generate-content', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(params),
      });

      const data = await response.json();
      return data.content;
    }

    // ダイレクト接続（Electronの場合）
    const response = await fetch(`${this.config.llm.endpoint}/api/generate`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: this.config.llm.model,
        prompt: params.prompt,
        temperature: params.temperature || 0.7,
        max_tokens: params.maxTokens || 500,
      }),
    });

    const data = await response.json();
    return data.response;
  }

  /**
   * 画像生成
   */
  async generateImage(params: ImageGenParams): Promise<Blob> {
    if (this.config.useProxy) {
      const response = await fetch('/api/ai/generate-image', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(params),
      });

      return await response.blob();
    }

    // Stable Diffusion WebUI API
    const response = await fetch(`${this.config.imageGen.endpoint}/sdapi/v1/txt2img`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        prompt: params.prompt,
        negative_prompt: params.negativePrompt,
        steps: params.steps || 20,
        width: params.width || 512,
        height: params.height || 512,
      }),
    });

    const data = await response.json();
    const base64 = data.images[0];

    // Base64 to Blob
    const binary = atob(base64);
    const array = new Uint8Array(binary.length);
    for (let i = 0; i < binary.length; i++) {
      array[i] = binary.charCodeAt(i);
    }

    return new Blob([array], { type: 'image/png' });
  }
}
```

## データフロー

### 1. プロジェクト作成フロー

```
User Action (New Project)
         ↓
  UI Component
         ↓
  projectStore.createProject()
         ↓
  ProjectRepository.save()
         ↓
  IndexedDB.put('projects', data)
         ↓
  localStorage.set('recent-projects', [id, ...])
         ↓
  UI Update (React re-render)
```

### 2. AI スライド生成フロー

```
User Input (Topic, Params)
         ↓
  UI Component
         ↓
  aiService.generateSlides()
         ↓
  AIClient.generateContent() → Server Proxy → Ollama
         ↓
  Parse AI Response
         ↓
  LayoutEngine.selectLayout()
         ↓
  slideStore.addSlides()
         ↓
  SlideRepository.saveAll()
         ↓
  IndexedDB.put('slides', data)
         ↓
  UI Update (Display slides)
```

### 3. エクスポートフロー

```
User Action (Export Project)
         ↓
  ExportDialog (Select format)
         ↓
  ExportService.exportAsZIP()
         ↓
  StorageManager.getAllData()
         ↓
  IndexedDB.getAll() → Projects, Slides, Assets
         ↓
  JSZip.generate() → Create ZIP file
         ↓
  Download ZIP file to user's device
```

### 4. インポートフロー

```
User Action (Import File)
         ↓
  ImportDialog (Drag & Drop or Browse)
         ↓
  ImportService.parse(file)
         ↓
  Validate file format
         ↓
  Extract data (JSON or ZIP)
         ↓
  StorageManager.import(data)
         ↓
  IndexedDB.put() → Save all data
         ↓
  UI Update (Reload project list)
```

## 技術スタック詳細

### Frontend

| カテゴリ | 技術 | バージョン | 用途 |
|---------|------|-----------|------|
| **フレームワーク** | React | 18.2+ | UIフレームワーク |
| **言語** | TypeScript | 5.0+ | 型安全な開発 |
| **ビルドツール** | Vite | 5.0+ | 高速ビルド |
| **状態管理** | Zustand | 4.0+ | グローバル状態管理 |
| **キャンバス** | Konva.js + react-konva | 9.0+ | スライドエディタ |
| **スタイリング** | TailwindCSS | 3.3+ | ユーティリティCSS |
| **ストレージ** | idb | 8.0+ | IndexedDB wrapper |
| **エクスポート** | JSZip | 3.10+ | ZIP生成 |
| **PDF生成** | jsPDF | 2.5+ | PDF出力 |
| **PPTX生成** | pptxgenjs | 3.12+ | PowerPoint出力 |
| **バリデーション** | Zod | 3.22+ | スキーマ検証 |
| **テスト** | Vitest | 1.0+ | ユニットテスト |
| **E2Eテスト** | Playwright | 1.40+ | E2Eテスト |

### Backend（最小限）

| カテゴリ | 技術 | バージョン | 用途 |
|---------|------|-----------|------|
| **ランタイム** | Node.js | 20 LTS | サーバー実行環境 |
| **フレームワーク** | Express | 4.18+ | Webフレームワーク |
| **言語** | TypeScript | 5.0+ | 型安全な開発 |
| **データベース** | better-sqlite3 | 9.0+ | SQLite（システムデータ） |
| **バリデーション** | Zod | 3.22+ | リクエスト検証 |
| **HTTP Client** | axios | 1.6+ | AI API呼び出し |

### Electron（本番推奨）

| カテゴリ | 技術 | バージョン | 用途 |
|---------|------|-----------|------|
| **フレームワーク** | Electron | 28+ | デスクトップアプリ化 |
| **ビルダー** | electron-builder | 24+ | パッケージング |
| **更新** | electron-updater | 6.0+ | 自動アップデート |

## デプロイメント戦略

### 開発環境

#### Docker Compose

```yaml
# docker-compose.yml

version: '3.8'

services:
  # バックエンド（AI Proxy + システムデータ）
  backend:
    build: ./server
    ports:
      - "3001:3001"
    volumes:
      - ./server:/app
      - /app/node_modules
      - ./data:/app/data
    environment:
      NODE_ENV: development
      PORT: 3001
      DATABASE_PATH: /app/data/system.db
      OLLAMA_API_URL: http://host.docker.internal:11434
      SD_API_URL: http://host.docker.internal:7860
      COQUI_TTS_URL: http://host.docker.internal:5002
    extra_hosts:
      - "host.docker.internal:host-gateway"

  # フロントエンド開発サーバー
  frontend:
    build: ./frontend
    ports:
      - "5173:5173"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      VITE_API_URL: http://localhost:3001
    depends_on:
      - backend

# データベースは不要（SQLiteファイル）
```

**起動方法**:
```bash
# 開発環境起動
docker-compose up

# アクセス
Frontend: http://localhost:5173
Backend API: http://localhost:3001
```

### 本番環境（Electron推奨）

#### Electron アプリ構成

```
Local-AI-Slide-Creator.app (or .exe)
├── resources/
│   ├── app/
│   │   ├── frontend/          # ビルド済みReactアプリ
│   │   ├── backend/           # バンドル済みNode.jsサーバー（オプション）
│   │   └── system.db          # システムデータ（テンプレート等）
│   └── icon.icns
└── MacOS/
    └── Local-AI-Slide-Creator # Electronバイナリ
```

#### パッケージング

```json
// package.json

{
  "name": "local-ai-slide-creator",
  "version": "1.0.0",
  "main": "electron/main.js",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "electron:dev": "electron .",
    "electron:build": "electron-builder"
  },
  "build": {
    "appId": "com.localai.slidecreator",
    "productName": "Local AI Slide Creator",
    "files": [
      "dist/**/*",
      "electron/**/*",
      "server/dist/**/*",
      "server/data/system.db"
    ],
    "directories": {
      "buildResources": "build"
    },
    "mac": {
      "target": ["dmg", "zip"],
      "category": "public.app-category.productivity"
    },
    "win": {
      "target": ["nsis", "portable"]
    },
    "linux": {
      "target": ["AppImage", "deb"]
    }
  }
}
```

### Web版（オプション）

```
静的ホスティング（Netlify, Vercel等）
  ├── Frontend（React SPA）
  └── Backend（Node.js）← Serverlessまたは小規模VPS
```

**注意**: Web版の場合、IndexedDBの容量制限に注意（ブラウザにより異なる）

## セキュリティ設計

### 1. データプライバシー

```
✅ ユーザーデータはクライアント側のみ
✅ サーバーにユーザーデータを送信しない
✅ エクスポート/インポートでユーザーが完全管理
✅ 削除時はIndexedDBから完全削除
```

### 2. AI API通信のセキュリティ

```typescript
// AI API呼び出し時の検証

export class AIClient {
  async generateContent(params: GenerateParams): Promise<string> {
    // URLの検証
    const url = new URL(this.config.llm.endpoint);
    if (!['http:', 'https:'].includes(url.protocol)) {
      throw new Error('Invalid protocol');
    }

    // タイムアウト設定
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 30000); // 30秒

    try {
      const response = await fetch(url.toString(), {
        method: 'POST',
        signal: controller.signal,
        // ...
      });

      return await response.json();
    } finally {
      clearTimeout(timeoutId);
    }
  }
}
```

### 3. XSS対策

```typescript
// ユーザー入力のサニタイズ（テキスト要素）

import DOMPurify from 'dompurify';

export function sanitizeHTML(html: string): string {
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'u', 'br', 'p'],
    ALLOWED_ATTR: [],
  });
}
```

## パフォーマンス設計

### 1. IndexedDB最適化

```typescript
// バッチ書き込み

export class IndexedDBService {
  async saveBatch(storeName: string, items: any[]): Promise<void> {
    const db = await this.getDB();
    const tx = db.transaction(storeName, 'readwrite');
    const store = tx.objectStore(storeName);

    // 並列書き込み
    await Promise.all(items.map(item => store.put(item)));

    await tx.done;
  }
}
```

### 2. 画像の最適化

```typescript
// サムネイル生成（保存時）

export async function generateThumbnail(
  blob: Blob,
  maxWidth: number = 200
): Promise<Blob> {
  const img = await createImageBitmap(blob);
  const scale = maxWidth / img.width;

  const canvas = document.createElement('canvas');
  canvas.width = maxWidth;
  canvas.height = img.height * scale;

  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(img, 0, 0, canvas.width, canvas.height);

  return new Promise((resolve) => {
    canvas.toBlob((blob) => resolve(blob!), 'image/jpeg', 0.8);
  });
}
```

### 3. 仮想化（大量スライド）

```typescript
// react-virtual を使用

import { useVirtual } from '@tanstack/react-virtual';

function SlideList({ slides }: { slides: Slide[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const rowVirtualizer = useVirtual({
    size: slides.length,
    parentRef,
    estimateSize: useCallback(() => 150, []),
    overscan: 5, // 前後5アイテムを事前レンダリング
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: `${rowVirtualizer.totalSize}px` }}>
        {rowVirtualizer.virtualItems.map((virtualRow) => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            <SlidePreview slide={slides[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 4. エクスポートの最適化

```typescript
// ストリーミング生成（大量データ）

export class ExportService {
  static async exportAsZIPStream(projectId: string): Promise<Blob> {
    const zip = new JSZip();

    // チャンク単位でデータを取得
    const CHUNK_SIZE = 100;
    const slideCount = await storageManager.getSlideCount(projectId);

    for (let i = 0; i < slideCount; i += CHUNK_SIZE) {
      const slides = await storageManager.getSlides(projectId, i, CHUNK_SIZE);

      for (const slide of slides) {
        zip.file(`slides/${slide.id}.json`, JSON.stringify(slide));
      }
    }

    return await zip.generateAsync({
      type: 'blob',
      streamFiles: true, // ストリーミング
    });
  }
}
```

---

**最終更新**: 2025-10-24
**レビュー**: 未実施
**次のステップ**: docs/STORAGE.md の詳細設計
