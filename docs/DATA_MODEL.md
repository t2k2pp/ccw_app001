# データモデル設計

## 目次
1. [概要](#概要)
2. [データベーススキーマ](#データベーススキーマ)
3. [TypeScript型定義](#typescript型定義)
4. [データ関連図](#データ関連図)
5. [バリデーションルール](#バリデーションルール)

## 概要

Local AI Slide Creatorのデータモデルは、以下の主要エンティティで構成されます：

- **Project**: プレゼンテーションプロジェクト
- **Slide**: 個別のスライド
- **Element**: スライド内の要素（テキスト、画像など）
- **Template**: レイアウトテンプレート
- **Theme**: デザインテーマ
- **Asset**: 画像やその他のメディアファイル

## データベーススキーマ

### ER図（テキスト表現）

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   Project    │1    n│    Slide     │1    n│   Element    │
│──────────────│───────│──────────────│───────│──────────────│
│ id           │       │ id           │       │ id           │
│ name         │       │ projectId    │       │ slideId      │
│ description  │       │ order        │       │ type         │
│ themeId      │       │ templateId   │       │ content      │
│ settings     │       │ title        │       │ position     │
│ createdAt    │       │ notes        │       │ size         │
│ updatedAt    │       │ background   │       │ style        │
└──────────────┘       │ createdAt    │       │ zIndex       │
                       │ updatedAt    │       │ createdAt    │
                       └──────────────┘       │ updatedAt    │
                              │               └──────────────┘
                              │n
                              │
                       ┌──────┴───────┐
                       │   Template   │
                       │──────────────│
                       │ id           │
                       │ name         │
                       │ category     │
                       │ layout       │
                       │ preview      │
                       └──────────────┘

┌──────────────┐       ┌──────────────┐
│    Theme     │       │    Asset     │
│──────────────│       │──────────────│
│ id           │       │ id           │
│ name         │       │ projectId    │
│ category     │       │ type         │
│ colors       │       │ filename     │
│ fonts        │       │ filepath     │
│ typography   │       │ size         │
│ spacing      │       │ metadata     │
│ shadows      │       │ createdAt    │
└──────────────┘       └──────────────┘
```

### SQLスキーマ

#### projects テーブル
```sql
CREATE TABLE projects (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT,
  theme_id TEXT,
  settings JSON NOT NULL DEFAULT '{}',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (theme_id) REFERENCES themes(id) ON DELETE SET NULL
);

CREATE INDEX idx_projects_updated_at ON projects(updated_at DESC);
```

#### slides テーブル
```sql
CREATE TABLE slides (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  template_id TEXT,
  title TEXT,
  notes TEXT,
  talk_script TEXT,
  order_index INTEGER NOT NULL,
  background JSON,
  display_duration INTEGER DEFAULT 5,
  transition JSON,
  voice_settings JSON,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE,
  FOREIGN KEY (template_id) REFERENCES templates(id) ON DELETE SET NULL
);

CREATE INDEX idx_slides_project_id ON slides(project_id);
CREATE INDEX idx_slides_order ON slides(project_id, order_index);
```

#### elements テーブル
```sql
CREATE TABLE elements (
  id TEXT PRIMARY KEY,
  slide_id TEXT NOT NULL,
  type TEXT NOT NULL CHECK(type IN ('text', 'image', 'shape', 'icon', 'group')),
  content JSON NOT NULL,
  position JSON NOT NULL,
  size JSON NOT NULL,
  style JSON NOT NULL DEFAULT '{}',
  z_index INTEGER NOT NULL DEFAULT 0,
  locked BOOLEAN NOT NULL DEFAULT 0,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (slide_id) REFERENCES slides(id) ON DELETE CASCADE
);

CREATE INDEX idx_elements_slide_id ON elements(slide_id);
CREATE INDEX idx_elements_z_index ON elements(slide_id, z_index);
```

#### templates テーブル
```sql
CREATE TABLE templates (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT,
  category TEXT NOT NULL,
  layout JSON NOT NULL,
  preview_url TEXT,
  is_system BOOLEAN NOT NULL DEFAULT 0,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_templates_category ON templates(category);
```

#### themes テーブル
```sql
CREATE TABLE themes (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT,
  category TEXT NOT NULL,
  colors JSON NOT NULL,
  fonts JSON NOT NULL,
  typography JSON NOT NULL,
  spacing JSON NOT NULL,
  border_radius JSON NOT NULL,
  shadows JSON NOT NULL,
  is_system BOOLEAN NOT NULL DEFAULT 0,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_themes_category ON themes(category);
```

#### assets テーブル
```sql
CREATE TABLE assets (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  type TEXT NOT NULL CHECK(type IN ('image', 'video', 'audio', 'font', 'svg')),
  filename TEXT NOT NULL,
  filepath TEXT NOT NULL,
  size INTEGER NOT NULL,
  mime_type TEXT NOT NULL,
  description TEXT,
  tags TEXT,
  metadata JSON,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE
);

CREATE INDEX idx_assets_project_id ON assets(project_id);
CREATE INDEX idx_assets_type ON assets(type);
CREATE INDEX idx_assets_tags ON assets(tags);
```

#### icon_library テーブル
```sql
CREATE TABLE icon_library (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  icon_set TEXT NOT NULL,
  category TEXT,
  tags TEXT,
  svg_content TEXT NOT NULL,
  keywords TEXT,
  is_system BOOLEAN NOT NULL DEFAULT 1,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_icon_library_set ON icon_library(icon_set);
CREATE INDEX idx_icon_library_category ON icon_library(category);
CREATE INDEX idx_icon_library_keywords ON icon_library(keywords);
CREATE VIRTUAL TABLE icon_library_fts USING fts5(name, tags, keywords, content=icon_library);
```

#### ai_generations テーブル
```sql
CREATE TABLE ai_generations (
  id TEXT PRIMARY KEY,
  project_id TEXT,
  slide_id TEXT,
  type TEXT NOT NULL CHECK(type IN ('slide_structure', 'content', 'image', 'talk_script', 'icon_suggestion')),
  prompt TEXT NOT NULL,
  parameters JSON,
  result JSON,
  status TEXT NOT NULL CHECK(status IN ('pending', 'processing', 'completed', 'failed')),
  error TEXT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  completed_at DATETIME,
  FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE,
  FOREIGN KEY (slide_id) REFERENCES slides(id) ON DELETE CASCADE
);

CREATE INDEX idx_ai_generations_project_id ON ai_generations(project_id);
CREATE INDEX idx_ai_generations_status ON ai_generations(status);
```

## TypeScript型定義

### 基本型

```typescript
// 共通型
type ID = string;
type Timestamp = string; // ISO 8601形式

// 色
type Color = string; // HEX形式 "#RRGGBB"

// 位置
interface Position {
  x: number; // ピクセル単位
  y: number;
}

// サイズ
interface Size {
  width: number;
  height: number;
}

// 矩形
interface Rect extends Position, Size {}

// 背景
interface Background {
  type: 'solid' | 'gradient' | 'image';
  color?: Color;
  gradient?: {
    type: 'linear' | 'radial';
    angle?: number; // linear用（度）
    colors: Array<{ color: Color; offset: number }>;
  };
  image?: {
    url: string;
    fit: 'cover' | 'contain' | 'fill' | 'tile';
    opacity?: number;
  };
}
```

### Project（プロジェクト）

```typescript
interface Project {
  id: ID;
  name: string;
  description?: string;
  themeId?: ID;
  settings: ProjectSettings;
  createdAt: Timestamp;
  updatedAt: Timestamp;

  // リレーション（取得時）
  slides?: Slide[];
  theme?: Theme;
  assets?: Asset[];
}

interface ProjectSettings {
  // スライド設定
  slideSize: {
    width: number;
    height: number;
    aspectRatio: '16:9' | '4:3' | '1:1' | 'custom';
  };

  // デフォルト設定
  defaults: {
    background: Background;
    font: {
      family: string;
      size: number;
      color: Color;
    };
  };

  // AI設定
  ai?: {
    llm: {
      provider: 'ollama' | 'lmstudio' | 'llamacpp' | 'textgen-webui';
      apiUrl: string;
      model: string;
      temperature?: number;
      maxTokens?: number;
    };
    imageGen?: {
      provider: 'stable-diffusion' | 'comfyui' | 'custom';
      apiUrl: string;
      model?: string;
      steps?: number;
      cfgScale?: number;
    };
    tts?: {
      provider: 'web-speech' | 'coqui' | 'piper' | 'voicevox' | 'custom';
      apiUrl?: string; // ローカルTTSの場合
      defaultVoice?: string;
      defaultRate?: number;
      defaultPitch?: number;
    };
  };

  // デジタルサイネージ設定
  signage?: {
    defaultDisplayDuration: number; // 秒
    defaultTransition: TransitionSettings;
    autoPlay: boolean;
    loop: boolean;
  };

  // エクスポート設定
  export?: {
    pdf?: {
      pageSize: 'A4' | 'letter' | 'slide';
      orientation: 'landscape' | 'portrait';
      quality: 'low' | 'medium' | 'high' | 'maximum';
    };
    video?: {
      format: 'mp4' | 'webm' | 'mov';
      fps: number;
      quality: 'low' | 'medium' | 'high';
    };
  };
}
```

### Slide（スライド）

```typescript
interface Slide {
  id: ID;
  projectId: ID;
  templateId?: ID;
  title: string;
  notes?: string;
  talkScript?: string;
  orderIndex: number;
  background?: Background;
  displayDuration?: number; // 秒単位（サイネージモード用）
  transition?: TransitionSettings;
  voiceSettings?: VoiceSettings;
  createdAt: Timestamp;
  updatedAt: Timestamp;

  // リレーション（取得時）
  elements?: Element[];
  template?: Template;
}

// トランジション設定
interface TransitionSettings {
  type: 'none' | 'fade' | 'slide-left' | 'slide-right' | 'slide-up' | 'slide-down' | 'zoom' | 'flip' | 'cube';
  duration: number; // ミリ秒
  easing?: 'linear' | 'ease' | 'ease-in' | 'ease-out' | 'ease-in-out';
}

// 音声設定
interface VoiceSettings {
  voice?: string; // 音声ID（Web Speech APIまたはローカルTTS）
  rate?: number;  // 0.5 - 2.0
  pitch?: number; // 0.5 - 2.0
  volume?: number; // 0.0 - 1.0
  language?: string; // 'ja-JP', 'en-US' など
}

// スライド作成用の入力型
interface CreateSlideInput {
  projectId: ID;
  templateId?: ID;
  title?: string;
  orderIndex?: number;
  background?: Background;
}

// スライド更新用の入力型
interface UpdateSlideInput {
  title?: string;
  notes?: string;
  talkScript?: string;
  orderIndex?: number;
  background?: Background;
  templateId?: ID;
  displayDuration?: number;
  transition?: TransitionSettings;
  voiceSettings?: VoiceSettings;
}
```

### Element（要素）

```typescript
type ElementType = 'text' | 'image' | 'shape' | 'icon' | 'group';

interface BaseElement {
  id: ID;
  slideId: ID;
  type: ElementType;
  position: Position;
  size: Size;
  style: ElementStyle;
  zIndex: number;
  locked: boolean;
  createdAt: Timestamp;
  updatedAt: Timestamp;
}

interface ElementStyle {
  opacity?: number;
  rotation?: number; // 度
  shadow?: {
    offsetX: number;
    offsetY: number;
    blur: number;
    color: Color;
  };
  border?: {
    width: number;
    color: Color;
    style: 'solid' | 'dashed' | 'dotted';
    radius?: number;
  };
}

// テキスト要素
interface TextElement extends BaseElement {
  type: 'text';
  content: {
    text: string;
    format: 'plain' | 'markdown' | 'html';
    styles: TextStyle;
  };
}

interface TextStyle {
  fontFamily: string;
  fontSize: number;
  fontWeight: number | 'normal' | 'bold';
  fontStyle: 'normal' | 'italic';
  color: Color;
  backgroundColor?: Color;
  textAlign: 'left' | 'center' | 'right' | 'justify';
  verticalAlign: 'top' | 'middle' | 'bottom';
  lineHeight?: number;
  letterSpacing?: number;
  textDecoration?: 'none' | 'underline' | 'line-through';
  padding?: {
    top: number;
    right: number;
    bottom: number;
    left: number;
  };
}

// 画像要素
interface ImageElement extends BaseElement {
  type: 'image';
  content: {
    url: string;
    assetId?: ID;
    alt?: string;
    fit: 'cover' | 'contain' | 'fill' | 'none';
    crop?: {
      x: number;
      y: number;
      width: number;
      height: number;
    };
    filter?: {
      brightness?: number; // 0-200, default 100
      contrast?: number;   // 0-200, default 100
      saturation?: number; // 0-200, default 100
      grayscale?: boolean;
      sepia?: boolean;
      blur?: number;       // 0-10
    };
  };
}

// 図形要素
interface ShapeElement extends BaseElement {
  type: 'shape';
  content: {
    shapeType: 'rectangle' | 'circle' | 'triangle' | 'polygon' | 'line' | 'arrow';
    fill?: Color;
    stroke?: {
      color: Color;
      width: number;
    };
    // polygon用
    points?: Position[];
    // line/arrow用
    startPoint?: Position;
    endPoint?: Position;
  };
}

// アイコン要素
interface IconElement extends BaseElement {
  type: 'icon';
  content: {
    iconName: string;
    iconSet: string; // 'heroicons', 'feather', etc.
    color: Color;
    strokeWidth?: number;
  };
}

// グループ要素
interface GroupElement extends BaseElement {
  type: 'group';
  content: {
    elements: Element[];
  };
}

// 統合型
type Element = TextElement | ImageElement | ShapeElement | IconElement | GroupElement;
```

### Template（テンプレート）

```typescript
interface Template {
  id: ID;
  name: string;
  description?: string;
  category: TemplateCategory;
  layout: TemplateLayout;
  previewUrl?: string;
  isSystem: boolean;
  createdAt: Timestamp;
  updatedAt: Timestamp;
}

type TemplateCategory =
  | 'title'
  | 'section'
  | 'content'
  | 'comparison'
  | 'image'
  | 'quote'
  | 'summary'
  | 'blank';

interface TemplateLayout {
  // プレースホルダー定義
  placeholders: Placeholder[];

  // レイアウトのメタデータ
  metadata: {
    columns?: number;
    rows?: number;
    gridTemplate?: string;
  };
}

interface Placeholder {
  id: string;
  type: 'text' | 'image' | 'content'; // content=text or image
  position: Position;
  size: Size;
  defaultContent?: {
    text?: string;
    placeholder?: string;
  };
  constraints?: {
    minSize?: Size;
    maxSize?: Size;
    aspectRatio?: number;
  };
  style?: Partial<ElementStyle>;
}
```

### Theme（テーマ）

```typescript
interface Theme {
  id: ID;
  name: string;
  description?: string;
  category: ThemeCategory;
  colors: ColorScheme;
  fonts: FontScheme;
  typography: Typography;
  spacing: SpacingScale;
  borderRadius: BorderRadiusScale;
  shadows: ShadowScale;
  isSystem: boolean;
  createdAt: Timestamp;
  updatedAt: Timestamp;
}

type ThemeCategory = 'business' | 'creative' | 'academic' | 'minimal' | 'custom';

interface ColorScheme {
  // プライマリーカラー
  primary: Color;
  primaryLight: Color;
  primaryDark: Color;

  // セカンダリーカラー
  secondary: Color;
  secondaryLight: Color;
  secondaryDark: Color;

  // アクセントカラー
  accent: Color;

  // 背景
  background: Color;
  backgroundAlt: Color;

  // テキスト
  text: Color;
  textSecondary: Color;
  textOnPrimary: Color;

  // その他
  border: Color;
  success: Color;
  warning: Color;
  error: Color;
}

interface FontScheme {
  heading: FontDefinition;
  body: FontDefinition;
  code: FontDefinition;
}

interface FontDefinition {
  family: string;
  weights: number[];
  fallback: string[];
  source?: 'system' | 'google' | 'custom';
  url?: string; // custom font用
}

interface Typography {
  h1: TypographyStyle;
  h2: TypographyStyle;
  h3: TypographyStyle;
  body: TypographyStyle;
  small: TypographyStyle;
}

interface TypographyStyle {
  fontSize: string; // "48px", "2rem", etc.
  lineHeight: string;
  fontWeight: number;
  letterSpacing?: string;
}

interface SpacingScale {
  xs: string;  // "4px"
  sm: string;  // "8px"
  md: string;  // "16px"
  lg: string;  // "24px"
  xl: string;  // "32px"
  '2xl': string; // "48px"
}

interface BorderRadiusScale {
  none: string; // "0"
  sm: string;   // "2px"
  md: string;   // "4px"
  lg: string;   // "8px"
  xl: string;   // "16px"
  full: string; // "9999px"
}

interface ShadowScale {
  none: string;
  sm: string;   // "0 1px 2px rgba(0,0,0,0.05)"
  md: string;   // "0 4px 6px rgba(0,0,0,0.1)"
  lg: string;   // "0 10px 15px rgba(0,0,0,0.1)"
  xl: string;   // "0 20px 25px rgba(0,0,0,0.1)"
  '2xl': string;
}
```

### Asset（アセット）

```typescript
interface Asset {
  id: ID;
  projectId: ID;
  type: AssetType;
  filename: string;
  filepath: string;
  size: number; // bytes
  mimeType: string;
  description?: string;
  tags?: string[]; // タグ配列
  metadata?: AssetMetadata;
  createdAt: Timestamp;
}

type AssetType = 'image' | 'video' | 'audio' | 'font' | 'svg';

interface AssetMetadata {
  // 画像の場合
  width?: number;
  height?: number;
  format?: string; // 'png', 'jpg', 'webp', etc.

  // AI生成画像の場合
  aiGenerated?: {
    prompt: string;
    negativePrompt?: string;
    model: string;
    seed?: number;
    steps?: number;
    cfgScale?: number;
  };

  // SVG の場合
  svgContent?: string; // SVGソースコード

  // その他
  [key: string]: any;
}
```

### Icon Library（アイコンライブラリ）

```typescript
interface IconLibrary {
  id: ID;
  name: string;
  iconSet: IconSet;
  category?: string;
  tags: string[];
  svgContent: string;
  keywords: string[];
  isSystem: boolean;
  createdAt: Timestamp;
}

type IconSet = 'heroicons' | 'feather' | 'material-icons' | 'font-awesome' | 'custom';

// アイコン検索結果
interface IconSearchResult {
  icon: IconLibrary;
  relevance: number; // 0-1の関連度スコア
}
```

### AI Generation（AI生成履歴）

```typescript
interface AIGeneration {
  id: ID;
  projectId?: ID;
  slideId?: ID;
  type: AIGenerationType;
  prompt: string;
  parameters?: Record<string, any>;
  result?: any;
  status: 'pending' | 'processing' | 'completed' | 'failed';
  error?: string;
  createdAt: Timestamp;
  completedAt?: Timestamp;
}

type AIGenerationType = 'slide_structure' | 'content' | 'image' | 'talk_script' | 'icon_suggestion';

// スライド構成生成の結果
interface SlideStructureResult {
  slides: Array<{
    title: string;
    bulletPoints: string[];
    suggestedLayout: TemplateCategory;
    notes?: string;
  }>;
}

// コンテンツ生成の結果
interface ContentGenerationResult {
  content: string;
  alternatives?: string[];
}

// 画像生成の結果
interface ImageGenerationResult {
  imageUrl: string;
  assetId: ID;
  metadata: {
    prompt: string;
    seed: number;
    model: string;
  };
}

// トークスクリプト生成の結果
interface TalkScriptGenerationResult {
  script: string;
  estimatedDuration: number; // 秒
  alternatives?: string[];
}

// アイコン提案の結果
interface IconSuggestionResult {
  suggestions: Array<{
    iconId: ID;
    iconName: string;
    iconSet: IconSet;
    relevance: number; // 0-1
    reasoning?: string; // なぜこのアイコンが適切か
  }>;
}
```

## データ関連図

### プロジェクトとスライドの関係

```
Project (1) ──< (n) Slide ──< (n) Element
   │                  │
   │                  └─> (0..1) Template
   │
   ├─> (0..1) Theme
   │
   └──< (n) Asset
```

### テンプレート適用の流れ

```
1. Template を選択
   ↓
2. Template.layout.placeholders を読み込み
   ↓
3. 各 Placeholder に対応する Element を作成
   ↓
4. Slide に Element を追加
```

### テーマ適用の流れ

```
1. Theme を選択
   ↓
2. Project.themeId を更新
   ↓
3. すべての Slide と Element にテーマのスタイルを適用
   - colors からカラースキームを適用
   - typography からフォントスタイルを適用
   - spacing, shadows などをスタイルに反映
```

## バリデーションルール

### Project

```typescript
const projectValidation = {
  name: {
    required: true,
    minLength: 1,
    maxLength: 255,
  },
  description: {
    maxLength: 1000,
  },
  'settings.slideSize.width': {
    min: 100,
    max: 10000,
  },
  'settings.slideSize.height': {
    min: 100,
    max: 10000,
  },
};
```

### Slide

```typescript
const slideValidation = {
  title: {
    maxLength: 255,
  },
  notes: {
    maxLength: 5000,
  },
  orderIndex: {
    required: true,
    min: 0,
  },
};
```

### Element

```typescript
const elementValidation = {
  type: {
    required: true,
    enum: ['text', 'image', 'shape', 'icon', 'group'],
  },
  'position.x': {
    required: true,
    type: 'number',
  },
  'position.y': {
    required: true,
    type: 'number',
  },
  'size.width': {
    required: true,
    min: 1,
  },
  'size.height': {
    required: true,
    min: 1,
  },
  zIndex: {
    required: true,
    type: 'number',
  },
};
```

### TextElement

```typescript
const textElementValidation = {
  ...elementValidation,
  'content.text': {
    required: true,
    maxLength: 10000,
  },
  'content.styles.fontSize': {
    required: true,
    min: 1,
    max: 500,
  },
};
```

### ImageElement

```typescript
const imageElementValidation = {
  ...elementValidation,
  'content.url': {
    required: true,
    type: 'url',
  },
  'content.fit': {
    enum: ['cover', 'contain', 'fill', 'none'],
  },
};
```

## データマイグレーション

### バージョン管理

```typescript
interface DataVersion {
  version: string; // semantic versioning
  migratedAt: Timestamp;
}

// プロジェクトファイルに含める
interface ProjectFile {
  version: string;
  data: Project;
  migrations?: DataVersion[];
}
```

### マイグレーション例

```typescript
// v1.0.0 から v1.1.0 へのマイグレーション
function migrateV1_0_to_V1_1(project: any): Project {
  // 新しいフィールドの追加
  if (!project.settings.export) {
    project.settings.export = {
      pdf: {
        pageSize: 'slide',
        orientation: 'landscape',
        quality: 'high',
      },
    };
  }

  return project;
}
```

## ファイルフォーマット

### プロジェクトファイル（.lasc）

```json
{
  "version": "1.0.0",
  "type": "local-ai-slide-creator-project",
  "data": {
    "project": { /* Project オブジェクト */ },
    "slides": [ /* Slide オブジェクトの配列 */ ],
    "elements": [ /* Element オブジェクトの配列 */ ],
    "assets": [ /* Asset オブジェクトの配列 */ ],
    "theme": { /* Theme オブジェクト (埋め込みの場合) */ }
  },
  "meta": {
    "createdWith": "Local AI Slide Creator v1.0.0",
    "createdAt": "2025-10-24T00:00:00Z",
    "exportedAt": "2025-10-24T00:00:00Z"
  }
}
```

### テンプレートファイル（.last）

```json
{
  "version": "1.0.0",
  "type": "local-ai-slide-creator-template",
  "data": {
    "template": { /* Template オブジェクト */ }
  }
}
```

### テーマファイル（.lastheme）

```json
{
  "version": "1.0.0",
  "type": "local-ai-slide-creator-theme",
  "data": {
    "theme": { /* Theme オブジェクト */ }
  }
}
```

## インデックス戦略

### パフォーマンス最適化のためのインデックス

```sql
-- 頻繁に使用されるクエリ
-- 1. プロジェクトの最近の更新順取得
CREATE INDEX idx_projects_updated_at ON projects(updated_at DESC);

-- 2. プロジェクトのスライド一覧（順序付き）
CREATE INDEX idx_slides_order ON slides(project_id, order_index);

-- 3. スライドの要素一覧（Z順）
CREATE INDEX idx_elements_z_index ON elements(slide_id, z_index);

-- 4. AI生成ステータス確認
CREATE INDEX idx_ai_generations_status ON ai_generations(status, created_at);
```

## キャッシング戦略

### メモリキャッシュ

```typescript
interface CacheStrategy {
  // よく使用されるデータをメモリキャッシュ
  projects: LRUCache<ID, Project>; // 最大100件
  slides: LRUCache<ID, Slide>;     // 最大1000件
  templates: Map<ID, Template>;    // すべてキャッシュ
  themes: Map<ID, Theme>;          // すべてキャッシュ
}
```

### ファイルキャッシュ

```typescript
interface FileCacheStrategy {
  // 画像のサムネイル
  thumbnails: {
    path: string;
    maxSize: number; // bytes
    maxAge: number;  // milliseconds
  };

  // エクスポート結果
  exports: {
    path: string;
    maxAge: number;
  };
}
```

---

**最終更新**: 2025-10-24
**スキーマバージョン**: 1.0.0
