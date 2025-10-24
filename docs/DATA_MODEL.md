# データモデル設計

## 目次
1. [概要](#概要)
2. [ストレージ戦略](#ストレージ戦略)
3. [IndexedDB スキーマ](#indexeddb-スキーマ)
4. [Server SQLite スキーマ](#server-sqlite-スキーマ)
5. [TypeScript型定義](#typescript型定義)
6. [レイアウトテンプレート定義](#レイアウトテンプレート定義)
7. [デフォルトレイアウトパターン集](#デフォルトレイアウトパターン集)
8. [データ関連図](#データ関連図)
9. [バリデーションルール](#バリデーションルール)

## 概要

Local AI Slide Creatorは、**クライアント中心設計**を採用し、ユーザーデータとシステムデータを明確に分離します。

### データ分類

```
┌─────────────────────────────────────────────────────┐
│  ユーザーデータ（IndexedDB - クライアント側）          │
│  - プロジェクト、スライド                              │
│  - アセット（画像等のBlob）                           │
│  - カスタムテンプレート                               │
│  - 編集履歴（Undo/Redo）                             │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  システムデータ（Server SQLite - サーバー側）          │
│  - プリセットテンプレート（読み取り専用）              │
│  - アイコンライブラリ（読み取り専用）                 │
│  - デフォルトテーマ（読み取り専用）                   │
│  - AIレスポンスキャッシュ                            │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  軽量設定（localStorage - クライアント側）            │
│  - アプリ設定、UI状態                                │
│  - AI接続設定                                       │
│  - 最近使用したプロジェクト                          │
└─────────────────────────────────────────────────────┘
```

## ストレージ戦略

### データの保存先

| エンティティ | 保存先 | 読み書き | 理由 |
|------------|--------|---------|------|
| Project | IndexedDB (Client) | Read/Write | ユーザーデータ |
| Slide | IndexedDB (Client) | Read/Write | ユーザーデータ |
| Asset (Blob) | IndexedDB (Client) | Read/Write | ユーザーデータ |
| Custom Template | IndexedDB (Client) | Read/Write | ユーザーデータ |
| History (Undo/Redo) | IndexedDB (Client) | Read/Write | ユーザーデータ |
| Preset Template | SQLite (Server) | Read-Only | システムデータ |
| Icon Library | SQLite (Server) | Read-Only | システムデータ |
| Default Theme | SQLite (Server) | Read-Only | システムデータ |
| AI Cache | SQLite (Server) | Read/Write | 一時データ |
| App Settings | localStorage (Client) | Read/Write | 軽量設定 |

## IndexedDB スキーマ

### Database: 'LocalAISlideCreator'

#### Object Stores

```typescript
interface IndexedDBSchema {
  projects: {
    keyPath: 'id';
    indexes: ['by-updated', 'by-created', 'by-name'];
  };
  slides: {
    keyPath: 'id';
    indexes: ['by-project', 'by-order'];
  };
  assets: {
    keyPath: 'id';
    indexes: ['by-project', 'by-type', 'by-created'];
  };
  templates: {
    keyPath: 'id';
    indexes: ['by-category', 'by-created'];
  };
  history: {
    keyPath: 'id';
    indexes: ['by-project', 'by-timestamp'];
  };
}
```

#### 1. projects Store

```typescript
interface Project {
  id: string; // UUID v4
  name: string;
  description?: string;
  themeId: string; // Reference to theme (system or custom)
  settings: ProjectSettings;
  metadata: {
    createdAt: Date;
    updatedAt: Date;
    slideCount: number;
    totalSize: number; // bytes
    tags?: string[];
  };
}

interface ProjectSettings {
  defaultTransition: TransitionType;
  defaultDisplayDuration: number; // seconds (for signage mode)
  aspectRatio: '16:9' | '4:3' | '1:1' | 'custom';
  resolution: { width: number; height: number };
  autoSave: boolean;
  autoSaveInterval: number; // seconds
}

type TransitionType =
  | 'none'
  | 'fade'
  | 'slide-left'
  | 'slide-right'
  | 'slide-up'
  | 'slide-down'
  | 'zoom'
  | 'flip'
  | 'cube';
```

#### 2. slides Store

```typescript
interface Slide {
  id: string; // UUID v4
  projectId: string; // Foreign key
  templateId?: string; // Reference to template (system or custom)
  title: string;
  notes?: string; // Speaker notes
  talkScript?: string; // TTS script
  orderIndex: number; // 0-based order
  elements: SlideElement[];
  background: SlideBackground;
  displayDuration: number; // seconds (for signage mode)
  transition: TransitionSettings;
  voiceSettings?: VoiceSettings;
  metadata: {
    createdAt: Date;
    updatedAt: Date;
    aiGenerated?: boolean;
    aiPrompt?: string; // Original prompt if AI-generated
  };
}

interface SlideElement {
  id: string; // UUID v4
  type: ElementType;
  x: number; // pixels or percentage
  y: number;
  width: number;
  height: number;
  rotation: number; // degrees
  zIndex: number;
  locked: boolean;
  visible: boolean;
  opacity: number; // 0-1

  // Type-specific properties
  properties:
    | TextProperties
    | ImageProperties
    | ShapeProperties
    | IconProperties
    | ChartProperties;
}

type ElementType = 'text' | 'image' | 'shape' | 'icon' | 'chart' | 'video';

interface TextProperties {
  content: string;
  fontSize: number; // px
  fontFamily: string;
  fontWeight: 'normal' | 'bold' | '100' | '200' | '300' | '400' | '500' | '600' | '700' | '800' | '900';
  fontStyle: 'normal' | 'italic';
  color: string; // hex color
  align: 'left' | 'center' | 'right' | 'justify';
  verticalAlign: 'top' | 'middle' | 'bottom';
  lineHeight: number; // multiplier (e.g., 1.5)
  letterSpacing: number; // px
  textDecoration: 'none' | 'underline' | 'line-through';
  textShadow?: string; // CSS text-shadow format
}

interface ImageProperties {
  assetId: string; // Reference to assets store
  objectFit: ImageFit;
  opacity: number; // 0-1
  filters?: ImageFilters;
  border?: BorderStyle;
  shadow?: ShadowStyle;
}

type ImageFit = 'contain' | 'cover' | 'fill' | 'none';

interface ImageFilters {
  brightness?: number; // 0-200 (100 = normal)
  contrast?: number; // 0-200 (100 = normal)
  saturation?: number; // 0-200 (100 = normal)
  blur?: number; // px
  grayscale?: number; // 0-100
  sepia?: number; // 0-100
}

interface ShapeProperties {
  shapeType: 'rectangle' | 'circle' | 'triangle' | 'polygon' | 'line' | 'arrow';
  fill?: string; // hex color
  stroke?: string; // hex color
  strokeWidth?: number; // px
  cornerRadius?: number; // px (for rectangle)
  shadow?: ShadowStyle;
}

interface IconProperties {
  iconId: string; // Reference to icon library
  svgContent: string; // SVG code
  color?: string; // hex color (if customizable)
  strokeWidth?: number;
}

interface ChartProperties {
  chartType: 'bar' | 'line' | 'pie' | 'doughnut' | 'scatter';
  data: ChartData;
  options: ChartOptions;
}

interface BorderStyle {
  width: number; // px
  color: string; // hex
  style: 'solid' | 'dashed' | 'dotted';
}

interface ShadowStyle {
  offsetX: number; // px
  offsetY: number; // px
  blur: number; // px
  color: string; // rgba
}

interface SlideBackground {
  type: 'solid' | 'gradient' | 'image';
  color?: string; // for solid
  gradient?: GradientBackground;
  imageAssetId?: string; // for image
  imageOpacity?: number; // 0-1
}

interface GradientBackground {
  type: 'linear' | 'radial';
  colors: GradientStop[];
  angle?: number; // degrees (for linear)
  position?: { x: number; y: number }; // for radial
}

interface GradientStop {
  color: string; // hex or rgba
  position: number; // 0-100 (percentage)
}

interface TransitionSettings {
  type: TransitionType;
  duration: number; // milliseconds
  easing?: EasingFunction;
}

type EasingFunction =
  | 'linear'
  | 'ease'
  | 'ease-in'
  | 'ease-out'
  | 'ease-in-out'
  | 'cubic-bezier';

interface VoiceSettings {
  voice?: string; // e.g., 'ja-JP-Neural2-B'
  rate?: number; // 0.5 - 2.0 (default: 1.0)
  pitch?: number; // 0.5 - 2.0 (default: 1.0)
  volume?: number; // 0.0 - 1.0 (default: 1.0)
  language?: string; // e.g., 'ja-JP', 'en-US'
}
```

#### 3. assets Store

```typescript
interface Asset {
  id: string; // UUID v4
  projectId?: string; // null = global asset
  name: string;
  type: AssetType;
  mimeType: string; // e.g., 'image/png'
  blob: Blob; // Binary data
  thumbnailBlob?: Blob; // Thumbnail (200px width)
  size: number; // bytes
  dimensions?: { width: number; height: number }; // for images/videos
  duration?: number; // seconds (for videos/audio)
  metadata: AssetMetadata;
}

type AssetType = 'image' | 'svg' | 'video' | 'audio';

interface AssetMetadata {
  createdAt: Date;
  source: AssetSource;
  aiPrompt?: string; // if AI-generated
  originalUrl?: string; // if url-import
  tags?: string[];
  description?: string;
  altText?: string; // for accessibility
}

type AssetSource = 'upload' | 'ai-generated' | 'icon-library' | 'url-import' | 'camera';
```

#### 4. templates Store (Custom templates only)

```typescript
interface CustomTemplate {
  id: string; // UUID v4
  name: string;
  description?: string;
  category: TemplateCategory;
  thumbnailAssetId?: string;
  layout: TemplateLayout;
  metadata: TemplateMetadata;
  isCustom: true; // Always true in IndexedDB
}

type TemplateCategory =
  | 'title'
  | 'content'
  | 'two-column'
  | 'image-text'
  | 'image-full'
  | 'bullets'
  | 'quote'
  | 'section-header'
  | 'comparison'
  | 'timeline'
  | 'custom';

interface TemplateMetadata {
  createdAt: Date;
  updatedAt: Date;
  usageCount: number;
  favorite: boolean;
  tags: string[];
  author?: string;
}
```

#### 5. history Store (Undo/Redo)

```typescript
interface HistoryEntry {
  id: string; // UUID v4
  projectId: string;
  action: HistoryAction;
  snapshot: StateSnapshot;
  timestamp: Date;
  description?: string; // Human-readable description
}

type HistoryAction =
  | 'create_slide'
  | 'delete_slide'
  | 'update_slide'
  | 'move_slide'
  | 'create_element'
  | 'delete_element'
  | 'update_element'
  | 'move_element'
  | 'update_project'
  | 'batch_operation';

interface StateSnapshot {
  type: 'slide' | 'element' | 'project' | 'batch';
  data: Slide | SlideElement | Project | BatchData;
}

interface BatchData {
  operations: Array<{
    type: HistoryAction;
    data: any;
  }>;
}
```

## Server SQLite スキーマ

### Database: 'system.db' (Server-side, Read-Only for clients)

#### 1. templates テーブル

```sql
CREATE TABLE templates (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  name_en TEXT, -- English name
  description TEXT,
  description_en TEXT,
  category TEXT NOT NULL,
  thumbnail_url TEXT,
  layout_json TEXT NOT NULL, -- JSON string of TemplateLayout
  metadata_json TEXT NOT NULL, -- JSON string of TemplateMetadata
  is_system BOOLEAN NOT NULL DEFAULT 1,
  version TEXT NOT NULL DEFAULT '1.0.0',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_templates_category ON templates(category);
CREATE INDEX idx_templates_system ON templates(is_system);
```

#### 2. icon_library テーブル

```sql
CREATE TABLE icon_library (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  icon_set TEXT NOT NULL, -- 'heroicons', 'feather', 'material-icons', 'font-awesome'
  category TEXT, -- 'actions', 'arrows', 'media', 'social', etc.
  tags TEXT, -- comma-separated
  svg_content TEXT NOT NULL,
  keywords TEXT, -- for search (comma-separated)
  is_system BOOLEAN NOT NULL DEFAULT 1,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_icons_set ON icon_library(icon_set);
CREATE INDEX idx_icons_category ON icon_library(category);

-- Full-Text Search
CREATE VIRTUAL TABLE icon_library_fts USING fts5(
  name, tags, keywords, content=icon_library
);
```

#### 3. themes テーブル

```sql
CREATE TABLE themes (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  name_en TEXT,
  description TEXT,
  description_en TEXT,
  colors_json TEXT NOT NULL,
  fonts_json TEXT NOT NULL,
  spacing_json TEXT,
  shadows_json TEXT,
  is_system BOOLEAN NOT NULL DEFAULT 1,
  preview_url TEXT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_themes_system ON themes(is_system);
```

#### 4. ai_cache テーブル

```sql
CREATE TABLE ai_cache (
  id TEXT PRIMARY KEY,
  cache_key TEXT NOT NULL UNIQUE, -- hash(prompt + params)
  prompt TEXT NOT NULL,
  params_json TEXT NOT NULL,
  response_json TEXT NOT NULL,
  provider TEXT NOT NULL, -- 'ollama', 'sd-webui', etc.
  model TEXT, -- e.g., 'llama3.1:8b'
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  accessed_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  access_count INTEGER NOT NULL DEFAULT 1,
  expires_at DATETIME -- optional TTL
);

CREATE INDEX idx_cache_key ON ai_cache(cache_key);
CREATE INDEX idx_cache_accessed ON ai_cache(accessed_at);
CREATE INDEX idx_cache_expires ON ai_cache(expires_at);

-- Auto-cleanup: DELETE WHERE expires_at < NOW() OR accessed_at < NOW() - 30 days
```

#### 5. usage_stats テーブル (Optional)

```sql
CREATE TABLE usage_stats (
  id TEXT PRIMARY KEY,
  event_type TEXT NOT NULL, -- 'template_used', 'slide_created', 'export_pdf', etc.
  data_json TEXT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_stats_type ON usage_stats(event_type);
CREATE INDEX idx_stats_created ON usage_stats(created_at);
```

## TypeScript型定義

### レイアウトテンプレート定義（詳細）

```typescript
/**
 * レイアウトテンプレートの定義
 *
 * デザイナーや開発者がこの形式でテンプレートを作成し、
 * AIがメタ情報を元に適切なテンプレートを選択できます。
 */
interface TemplateLayout {
  id: string;
  name: string;
  description?: string;

  // サイズ設定
  size: {
    width: number; // px (e.g., 1920)
    height: number; // px (e.g., 1080)
    aspectRatio: '16:9' | '4:3' | '1:1' | 'custom';
  };

  // レイアウト領域
  areas: LayoutArea[];

  // メタ情報（AI選択用）
  selectionCriteria: SelectionCriteria;

  // スタイルガイド
  styleGuide: StyleGuide;
}

/**
 * レイアウト領域の定義
 */
interface LayoutArea {
  id: string; // e.g., 'title', 'content-1', 'image-main'
  type: AreaType;

  // 位置とサイズ（パーセンテージまたは絶対値）
  position: AreaPosition;

  // 制約条件
  constraints: AreaConstraints;

  // デフォルトスタイル
  defaultStyle: AreaStyle;

  // AI配置ヒント
  placementHints: PlacementHints;
}

type AreaType =
  | 'title'        // タイトル領域
  | 'subtitle'     // サブタイトル領域
  | 'text'         // テキスト領域
  | 'bullets'      // 箇条書き領域
  | 'image'        // 画像領域
  | 'icon'         // アイコン領域
  | 'shape'        // 図形領域
  | 'chart'        // グラフ領域
  | 'quote'        // 引用領域
  | 'code'         // コード領域
  | 'video'        // 動画領域
  | 'placeholder'; // 汎用プレースホルダー

interface AreaPosition {
  // 位置（絶対値またはパーセンテージ）
  x: number | string; // e.g., 100 or '10%'
  y: number | string;
  width: number | string;
  height: number | string;

  // アンカーポイント（レスポンシブ対応）
  anchor?: AnchorPoint;

  // z-index
  zIndex?: number;
}

type AnchorPoint =
  | 'top-left'
  | 'top-center'
  | 'top-right'
  | 'middle-left'
  | 'middle-center'
  | 'middle-right'
  | 'bottom-left'
  | 'bottom-center'
  | 'bottom-right';

interface AreaConstraints {
  // サイズ制約
  minWidth?: number;
  maxWidth?: number;
  minHeight?: number;
  maxHeight?: number;

  // アスペクト比（画像用）
  aspectRatio?: number; // e.g., 16/9
  maintainAspectRatio?: boolean;

  // コンテンツ制約
  maxCharacters?: number; // テキスト用
  maxLines?: number; // テキスト用
  maxWords?: number; // テキスト用

  // 必須/オプション
  required: boolean;

  // リサイズ可能か
  resizable: boolean;

  // 移動可能か
  movable: boolean;

  // 削除可能か
  deletable: boolean;
}

interface AreaStyle {
  // テキストスタイル（text/title/subtitle用）
  text?: {
    fontSize: number; // px
    fontFamily: string;
    fontWeight: string;
    color: string; // hex
    align: 'left' | 'center' | 'right' | 'justify';
    lineHeight: number;
  };

  // 画像スタイル（image用）
  image?: {
    objectFit: ImageFit;
    border?: BorderStyle;
    shadow?: ShadowStyle;
    filter?: ImageFilters;
  };

  // 背景
  background?: {
    color?: string;
    opacity?: number;
  };

  // パディング
  padding?: {
    top: number;
    right: number;
    bottom: number;
    left: number;
  };

  // ボーダー
  border?: BorderStyle;

  // シャドウ
  shadow?: ShadowStyle;
}

interface PlacementHints {
  // コンテンツの優先度（AIが複数の領域がある場合に使用）
  priority: number; // 1-10 (10 = highest)

  // 推奨コンテンツタイプ
  suggestedContentType?: string; // e.g., 'short-text', 'long-text', 'image-portrait', 'image-landscape'

  // 推奨コンテンツ長
  suggestedContentLength?: {
    min: number;
    max: number;
    ideal: number;
  };

  // AI生成時のヒント
  aiHint?: string; // e.g., 'Place main point here', 'Use supporting image'
}

/**
 * テンプレート選択基準（AI用）
 */
interface SelectionCriteria {
  // カテゴリ
  category: TemplateCategory;

  // タグ
  tags: string[]; // e.g., ['business', 'simple', 'modern', 'colorful']

  // コンテンツ適合性
  contentFit: ContentFitCriteria;

  // 使用シーン
  usageScenario: string[]; // e.g., ['opening', 'agenda', 'content', 'closing']

  // スコアリングウェイト
  scoringWeights: ScoringWeights;
}

interface ContentFitCriteria {
  // テキスト量の適合度
  textAmount: 'none' | 'minimal' | 'moderate' | 'heavy';

  // 画像数の適合度
  imageCount: 'none' | 'single' | 'multiple';

  // 箇条書きの有無
  hasBullets: boolean;

  // 推奨テキスト文字数
  recommendedTextLength?: {
    min: number;
    max: number;
  };

  // 推奨画像サイズ
  recommendedImageSize?: {
    width: number;
    height: number;
  };
}

interface ScoringWeights {
  // テキスト量の一致度
  textAmountMatch: number; // 0-1

  // 画像数の一致度
  imageCountMatch: number; // 0-1

  // カテゴリの一致度
  categoryMatch: number; // 0-1

  // タグの一致度
  tagMatch: number; // 0-1

  // 使用頻度
  usageFrequency: number; // 0-1
}

interface StyleGuide {
  // 推奨カラーパレット
  colorPalette?: string[]; // hex colors

  // 推奨フォント
  fonts?: {
    heading: string;
    body: string;
    accent?: string;
  };

  // 余白ガイド
  spacing?: {
    small: number;
    medium: number;
    large: number;
  };

  // グリッド設定
  grid?: {
    columns: number;
    rows: number;
    gutter: number; // px
  };
}
```

## レイアウトテンプレート定義

### JSON形式の例

```json
{
  "id": "template_title_modern",
  "name": "モダンタイトルスライド",
  "description": "シンプルでモダンなタイトルスライド。プレゼンテーションの冒頭に最適。",
  "size": {
    "width": 1920,
    "height": 1080,
    "aspectRatio": "16:9"
  },
  "areas": [
    {
      "id": "title",
      "type": "title",
      "position": {
        "x": "10%",
        "y": "30%",
        "width": "80%",
        "height": "20%",
        "anchor": "middle-center",
        "zIndex": 2
      },
      "constraints": {
        "minWidth": 600,
        "maxCharacters": 60,
        "maxLines": 2,
        "required": true,
        "resizable": true,
        "movable": true,
        "deletable": false
      },
      "defaultStyle": {
        "text": {
          "fontSize": 72,
          "fontFamily": "Arial, sans-serif",
          "fontWeight": "bold",
          "color": "#1a1a1a",
          "align": "center",
          "lineHeight": 1.2
        }
      },
      "placementHints": {
        "priority": 10,
        "suggestedContentType": "short-text",
        "suggestedContentLength": {
          "min": 10,
          "max": 60,
          "ideal": 30
        },
        "aiHint": "プレゼンテーションの主題を簡潔に記述"
      }
    },
    {
      "id": "subtitle",
      "type": "subtitle",
      "position": {
        "x": "10%",
        "y": "55%",
        "width": "80%",
        "height": "10%",
        "anchor": "middle-center",
        "zIndex": 2
      },
      "constraints": {
        "maxCharacters": 100,
        "maxLines": 2,
        "required": false,
        "resizable": true,
        "movable": true,
        "deletable": true
      },
      "defaultStyle": {
        "text": {
          "fontSize": 36,
          "fontFamily": "Arial, sans-serif",
          "fontWeight": "normal",
          "color": "#666666",
          "align": "center",
          "lineHeight": 1.4
        }
      },
      "placementHints": {
        "priority": 8,
        "suggestedContentType": "short-text",
        "suggestedContentLength": {
          "min": 0,
          "max": 100,
          "ideal": 50
        },
        "aiHint": "サブタイトルまたは日付、発表者名など"
      }
    },
    {
      "id": "background-accent",
      "type": "shape",
      "position": {
        "x": "0%",
        "y": "0%",
        "width": "100%",
        "height": "30%",
        "zIndex": 1
      },
      "constraints": {
        "required": false,
        "resizable": true,
        "movable": false,
        "deletable": true
      },
      "defaultStyle": {
        "background": {
          "color": "#f0f4f8",
          "opacity": 0.8
        }
      },
      "placementHints": {
        "priority": 1,
        "aiHint": "装飾的な背景要素"
      }
    }
  ],
  "selectionCriteria": {
    "category": "title",
    "tags": ["modern", "simple", "professional", "business"],
    "contentFit": {
      "textAmount": "minimal",
      "imageCount": "none",
      "hasBullets": false,
      "recommendedTextLength": {
        "min": 10,
        "max": 160
      }
    },
    "usageScenario": ["opening", "section-header"],
    "scoringWeights": {
      "textAmountMatch": 0.3,
      "imageCountMatch": 0.2,
      "categoryMatch": 0.3,
      "tagMatch": 0.1,
      "usageFrequency": 0.1
    }
  },
  "styleGuide": {
    "colorPalette": ["#1a1a1a", "#666666", "#f0f4f8", "#3b82f6"],
    "fonts": {
      "heading": "Arial, sans-serif",
      "body": "Arial, sans-serif"
    },
    "spacing": {
      "small": 16,
      "medium": 32,
      "large": 64
    }
  }
}
```

## デフォルトレイアウトパターン集

以下の10種類のデフォルトレイアウトパターンを用意します：

### 1. Title Slide（タイトルスライド）

**用途**: プレゼンテーションの冒頭、セクション区切り

**構成**:
- タイトル領域（中央、大きめ）
- サブタイトル領域（タイトル下、中サイズ）
- 背景装飾（オプション）

**適合コンテンツ**:
- テキスト量: 最小（10-160文字）
- 画像: なし
- 箇条書き: なし

### 2. Content Slide（コンテンツスライド）

**用途**: 一般的なコンテンツスライド

**構成**:
- タイトル領域（上部、左寄せ）
- コンテンツ領域（中央、広め）

**適合コンテンツ**:
- テキスト量: 中程度（100-500文字）
- 画像: 0-2枚
- 箇条書き: 任意

### 3. Two Column（2カラム）

**用途**: 比較、対比、並列情報

**構成**:
- タイトル領域（上部）
- 左カラム領域（50%）
- 右カラム領域（50%）

**適合コンテンツ**:
- テキスト量: 中程度（各カラム100-300文字）
- 画像: 各カラムに0-1枚
- 箇条書き: 任意

### 4. Image Left / Right（画像＋テキスト）

**用途**: 画像を使った説明

**構成**:
- タイトル領域（上部）
- 画像領域（左 or 右、40-50%）
- テキスト領域（右 or 左、50-60%）

**適合コンテンツ**:
- テキスト量: 中程度（100-400文字）
- 画像: 1枚（メイン）
- 箇条書き: 任意

### 5. Image Full（画像メイン）

**用途**: 画像中心のスライド、ビジュアル重視

**構成**:
- 画像領域（全体の70-90%）
- キャプション領域（下部または上部、小さめ）

**適合コンテンツ**:
- テキスト量: 最小（0-100文字）
- 画像: 1枚（大きめ）
- 箇条書き: なし

### 6. Bullet Points（箇条書き）

**用途**: リスト、要点の列挙

**構成**:
- タイトル領域（上部）
- 箇条書き領域（中央、左寄せ）
- アイコン領域（オプション、各項目の左）

**適合コンテンツ**:
- テキスト量: 中程度（5-10項目、各50-100文字）
- 画像: 0-1枚（補助）
- 箇条書き: 必須

### 7. Quote（引用）

**用途**: 引用、重要な発言、名言

**構成**:
- 引用文領域（中央、大きめ）
- 引用元領域（下部、小さめ）
- 引用符装飾（オプション）

**適合コンテンツ**:
- テキスト量: 少ない（50-200文字）
- 画像: 0-1枚（人物写真等）
- 箇条書き: なし

### 8. Section Header（セクション見出し）

**用途**: 章・セクションの区切り

**構成**:
- セクション番号領域（大きめ、装飾的）
- セクションタイトル領域（中央または左）
- 背景装飾（大胆なカラーまたは画像）

**適合コンテンツ**:
- テキスト量: 最小（5-50文字）
- 画像: 0-1枚（背景）
- 箇条書き: なし

### 9. Comparison（比較）

**用途**: Before/After、Option A/B、対比

**構成**:
- タイトル領域（上部）
- 左側領域（45%、ラベル付き）
- 右側領域（45%、ラベル付き）
- 区切り線または vs アイコン（中央）

**適合コンテンツ**:
- テキスト量: 中程度（各側100-300文字）
- 画像: 各側に0-1枚
- 箇条書き: 任意

### 10. Timeline（タイムライン）

**用途**: 時系列、ロードマップ、プロセス

**構成**:
- タイトル領域（上部）
- タイムライン軸（横または縦）
- 各ステップ領域（3-6個）

**適合コンテンツ**:
- テキスト量: 中程度（各ステップ50-150文字）
- 画像: 各ステップに0-1枚（アイコン推奨）
- 箇条書き: 任意

### デフォルトテンプレート一覧表

| ID | 名前 | カテゴリ | テキスト量 | 画像数 | 箇条書き | タグ |
|----|------|----------|----------|--------|---------|------|
| `template_title_modern` | モダンタイトル | title | minimal | none | ✗ | modern, simple, professional |
| `template_content_standard` | 標準コンテンツ | content | moderate | 0-2 | ○ | versatile, standard |
| `template_two_column` | 2カラム | two-column | moderate | 0-2 | ○ | comparison, balanced |
| `template_image_left` | 画像左 | image-text | moderate | single | ○ | visual, explanation |
| `template_image_right` | 画像右 | image-text | moderate | single | ○ | visual, explanation |
| `template_image_full` | 画像フル | image-full | minimal | single | ✗ | visual, photo, impact |
| `template_bullets_simple` | シンプル箇条書き | bullets | moderate | 0-1 | ✓ | list, points, standard |
| `template_quote_elegant` | エレガント引用 | quote | minimal | 0-1 | ✗ | quote, emphasis, elegant |
| `template_section_bold` | ボールドセクション | section-header | minimal | 0-1 | ✗ | divider, bold, modern |
| `template_comparison` | 比較スライド | comparison | moderate | 0-2 | ○ | vs, before-after, options |
| `template_timeline_horizontal` | 横タイムライン | timeline | moderate | 0-6 | ○ | process, steps, roadmap |

## データ関連図

### エンティティ関係図（クライアント側）

```
┌──────────────────┐
│     Project      │
│──────────────────│
│ id (PK)          │
│ name             │
│ themeId          │
│ settings         │
└──────────────────┘
         │1
         │
         │n
         ↓
┌──────────────────┐       ┌──────────────────┐
│      Slide       │       │    Template      │
│──────────────────│       │──────────────────│
│ id (PK)          │──────→│ id (PK)          │
│ projectId (FK)   │n    1 │ name             │
│ templateId (FK)  │       │ category         │
│ title            │       │ layout           │
│ elements[]       │       └──────────────────┘
│ background       │
└──────────────────┘
         │1                ┌──────────────────┐
         │                 │      Asset       │
         │                 │──────────────────│
         │n                │ id (PK)          │
         │                 │ projectId (FK)   │
         │                 │ type             │
         ↓                 │ blob             │
┌──────────────────┐       │ metadata         │
│   SlideElement   │──────→└──────────────────┘
│──────────────────│n    1
│ id (PK)          │   (if type=image)
│ slideId (FK)     │
│ type             │
│ properties       │
└──────────────────┘
```

### データフロー図

```
┌─────────────┐
│    User     │
└─────────────┘
       │
       ↓ (Create Project)
┌─────────────────────────┐
│  IndexedDB: projects    │←─── themeId ───→ Server: themes
└─────────────────────────┘
       │
       ↓ (Create Slide with Template)
┌─────────────────────────┐
│  IndexedDB: slides      │←── templateId ──→ Server: templates
└─────────────────────────┘
       │
       ↓ (Add Elements)
┌─────────────────────────┐
│  IndexedDB: slide.      │
│  elements[]             │
└─────────────────────────┘
       │
       ↓ (Upload/Generate Image)
┌─────────────────────────┐
│  IndexedDB: assets      │
│  (Blob storage)         │
└─────────────────────────┘
       │
       ↓ (Export)
┌─────────────────────────┐
│  ZIP / JSON / PPTX /    │
│  PDF / Marp / HTML      │
└─────────────────────────┘
```

## バリデーションルール

### Project

```typescript
const projectSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(200),
  description: z.string().max(1000).optional(),
  themeId: z.string(),
  settings: z.object({
    defaultTransition: z.enum([
      'none', 'fade', 'slide-left', 'slide-right',
      'slide-up', 'slide-down', 'zoom', 'flip', 'cube'
    ]),
    defaultDisplayDuration: z.number().min(1).max(60),
    aspectRatio: z.enum(['16:9', '4:3', '1:1', 'custom']),
    resolution: z.object({
      width: z.number().min(640).max(7680),
      height: z.number().min(480).max(4320),
    }),
    autoSave: z.boolean(),
    autoSaveInterval: z.number().min(10).max(300),
  }),
  metadata: z.object({
    createdAt: z.date(),
    updatedAt: z.date(),
    slideCount: z.number().min(0),
    totalSize: z.number().min(0),
    tags: z.array(z.string()).optional(),
  }),
});
```

### Slide

```typescript
const slideSchema = z.object({
  id: z.string().uuid(),
  projectId: z.string().uuid(),
  templateId: z.string().optional(),
  title: z.string().min(1).max(200),
  notes: z.string().max(5000).optional(),
  talkScript: z.string().max(10000).optional(),
  orderIndex: z.number().min(0),
  elements: z.array(slideElementSchema),
  background: backgroundSchema,
  displayDuration: z.number().min(1).max(60),
  transition: transitionSchema,
  voiceSettings: voiceSettingsSchema.optional(),
  metadata: z.object({
    createdAt: z.date(),
    updatedAt: z.date(),
    aiGenerated: z.boolean().optional(),
    aiPrompt: z.string().optional(),
  }),
});
```

### SlideElement

```typescript
const slideElementSchema = z.object({
  id: z.string().uuid(),
  type: z.enum(['text', 'image', 'shape', 'icon', 'chart', 'video']),
  x: z.number(),
  y: z.number(),
  width: z.number().min(1),
  height: z.number().min(1),
  rotation: z.number().min(-180).max(180),
  zIndex: z.number().min(0).max(1000),
  locked: z.boolean(),
  visible: z.boolean(),
  opacity: z.number().min(0).max(1),
  properties: z.union([
    textPropertiesSchema,
    imagePropertiesSchema,
    shapePropertiesSchema,
    iconPropertiesSchema,
    chartPropertiesSchema,
  ]),
});
```

### TemplateLayout

```typescript
const templateLayoutSchema = z.object({
  id: z.string(),
  name: z.string().min(1).max(100),
  description: z.string().max(500).optional(),
  size: z.object({
    width: z.number().min(640).max(7680),
    height: z.number().min(480).max(4320),
    aspectRatio: z.enum(['16:9', '4:3', '1:1', 'custom']),
  }),
  areas: z.array(layoutAreaSchema).min(1).max(20),
  selectionCriteria: selectionCriteriaSchema,
  styleGuide: styleGuideSchema,
});

const layoutAreaSchema = z.object({
  id: z.string(),
  type: z.enum([
    'title', 'subtitle', 'text', 'bullets', 'image',
    'icon', 'shape', 'chart', 'quote', 'code',
    'video', 'placeholder'
  ]),
  position: z.object({
    x: z.union([z.number(), z.string()]),
    y: z.union([z.number(), z.string()]),
    width: z.union([z.number(), z.string()]),
    height: z.union([z.number(), z.string()]),
    anchor: z.enum([
      'top-left', 'top-center', 'top-right',
      'middle-left', 'middle-center', 'middle-right',
      'bottom-left', 'bottom-center', 'bottom-right'
    ]).optional(),
    zIndex: z.number().optional(),
  }),
  constraints: z.object({
    minWidth: z.number().optional(),
    maxWidth: z.number().optional(),
    minHeight: z.number().optional(),
    maxHeight: z.number().optional(),
    aspectRatio: z.number().optional(),
    maintainAspectRatio: z.boolean().optional(),
    maxCharacters: z.number().optional(),
    maxLines: z.number().optional(),
    maxWords: z.number().optional(),
    required: z.boolean(),
    resizable: z.boolean(),
    movable: z.boolean(),
    deletable: z.boolean(),
  }),
  defaultStyle: z.object({
    text: textStyleSchema.optional(),
    image: imageStyleSchema.optional(),
    background: backgroundStyleSchema.optional(),
    padding: paddingSchema.optional(),
    border: borderStyleSchema.optional(),
    shadow: shadowStyleSchema.optional(),
  }),
  placementHints: z.object({
    priority: z.number().min(1).max(10),
    suggestedContentType: z.string().optional(),
    suggestedContentLength: z.object({
      min: z.number(),
      max: z.number(),
      ideal: z.number(),
    }).optional(),
    aiHint: z.string().optional(),
  }),
});
```

---

**最終更新**: 2025-10-24
**レビュー**: 未実施
**次のステップ**: API.md, FEATURES.md の更新
