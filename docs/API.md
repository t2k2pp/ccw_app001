# API設計書

## 目次
1. [概要](#概要)
2. [アーキテクチャ方針](#アーキテクチャ方針)
3. [認証](#認証)
4. [共通仕様](#共通仕様)
5. [AI Proxy API](#ai-proxy-api)
6. [システムデータAPI](#システムデータapi)
7. [設定API](#設定api)
8. [エクスポートAPI](#エクスポートapi)
9. [WebSocket API](#websocket-api)
10. [エラーハンドリング](#エラーハンドリング)

## 概要

### ベースURL
```
http://localhost:3001/api/v1
```

### プロトコル
- REST API（HTTP/HTTPS）
- WebSocket（リアルタイム通知用、Phase 3以降）

### データフォーマット
- リクエスト: JSON
- レスポンス: JSON
- ファイルアップロード: multipart/form-data（アイコン、画像生成結果など）

## アーキテクチャ方針

### クライアントファースト設計

**重要**: このアプリケーションは**クライアントファースト・アーキテクチャ**を採用しています。

#### ユーザーデータの保存場所
- **クライアント側（100%）**:
  - IndexedDB: プロジェクト、スライド、要素、アセット（Blob）、履歴
  - localStorage: UI設定、AI設定、最近使ったプロジェクトリスト
- **サーバー側**:
  - ユーザーデータは**一切保存しない**
  - システムデータのみ（プリセットテンプレート、アイコンライブラリ、テーマ - 読み取り専用）

#### サーバーの役割
サーバーは以下の機能のみを提供します：

1. **AI Proxy**: LLM、画像生成、TTS APIへのプロキシ（クライアントから直接アクセスできない場合）
2. **システムデータ提供**: プリセットテンプレート、アイコンライブラリ、テーマの配信
3. **設定管理**: AI接続設定の保存・取得
4. **エクスポート支援**: サーバー側レンダリングが必要な場合（オプション）

#### データポータビリティ
- ユーザーデータはクライアント側でエクスポート/インポート可能
- 対応フォーマット: JSON、ZIP、Marp
- 別環境への移行: エクスポート → インポートで実現

### セキュリティとプライバシー
- **ユーザーデータの漏洩リスクゼロ**: サーバーに保存しないため
- **オフライン動作**: ネットワーク不要（AI機能を除く）
- **データ主権**: ユーザーが完全にデータを管理

## 認証

MVP版では認証なし（シングルユーザー、ローカル環境想定）

将来拡張（Phase 4以降）でJWT認証を実装予定：
```
Authorization: Bearer <token>
```

## 共通仕様

### リクエストヘッダー
```
Content-Type: application/json
Accept: application/json
```

### レスポンス形式

#### 成功レスポンス
```json
{
  "success": true,
  "data": { /* レスポンスデータ */ }
}
```

#### エラーレスポンス
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーメッセージ",
    "details": { /* 詳細情報（オプション） */ }
  }
}
```

#### ページネーション（システムデータAPI用）
```json
{
  "success": true,
  "data": {
    "items": [ /* データの配列 */ ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 100,
      "totalPages": 5
    }
  }
}
```

### HTTPステータスコード
- `200 OK`: 成功
- `201 Created`: 作成成功
- `204 No Content`: 削除成功
- `400 Bad Request`: リクエストエラー
- `404 Not Found`: リソースが見つからない
- `422 Unprocessable Entity`: バリデーションエラー
- `500 Internal Server Error`: サーバーエラー
- `502 Bad Gateway`: 外部AI APIエラー
- `504 Gateway Timeout`: AI APIタイムアウト

## AI Proxy API

サーバーは、クライアントから各種AI API（Ollama、Stable Diffusion、TTSなど）へのプロキシとして機能します。

### 1. LLM - コンテンツ生成

#### スライド構成生成
```
POST /ai/generate-structure
```

**リクエストボディ**:
```json
{
  "prompt": "新製品の発表プレゼンテーション",
  "parameters": {
    "audience": "投資家",
    "slideCount": 10,
    "tone": "formal",
    "language": "ja"
  }
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "slides": [
      {
        "title": "新製品紹介",
        "bulletPoints": [
          "市場の課題",
          "ソリューション",
          "競合優位性"
        ],
        "suggestedLayoutId": "template_content",
        "notes": "冒頭で市場の課題を明確に"
      }
    ]
  }
}
```

#### スライドコンテンツ生成
```
POST /ai/generate-content
```

**リクエストボディ**:
```json
{
  "prompt": "新製品の特徴を3つの箇条書きで",
  "context": {
    "slideTitle": "製品の特徴",
    "previousSlideContent": "市場の課題について説明しました"
  },
  "parameters": {
    "maxLength": 500,
    "tone": "persuasive",
    "language": "ja",
    "format": "bullets"
  }
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "content": "- 革新的なAI技術により作業効率が3倍向上\n- 直感的なUIで誰でもすぐに使いこなせる\n- 24時間365日の自動運用で人的コスト削減",
    "alternatives": [
      "- 最先端のAI技術搭載\n- 使いやすさにこだわった設計\n- 完全自動化による省人化"
    ]
  }
}
```

#### コンテンツ改善
```
POST /ai/improve-content
```

**リクエストボディ**:
```json
{
  "currentContent": "このプロダクトはすごいです",
  "improvementType": "professional",
  "parameters": {
    "tone": "formal",
    "language": "ja",
    "targetLength": "medium"
  }
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "original": "このプロダクトはすごいです",
    "improved": "本製品は、革新的な機能と卓越した性能により、市場に大きな価値を提供します。",
    "alternatives": [
      "本製品は業界をリードする技術革新を実現しています。",
      "当社の新製品は、お客様のビジネスを飛躍的に向上させる画期的なソリューションです。"
    ]
  }
}
```

#### トークスクリプト生成
```
POST /ai/generate-talk-script
```

**リクエストボディ**:
```json
{
  "slideContent": {
    "title": "新製品発表",
    "elements": [
      {
        "type": "text",
        "content": "革新的なAI技術\n直感的なUI\n完全自動化"
      }
    ]
  },
  "parameters": {
    "length": "standard",
    "tone": "formal",
    "targetAudience": "business",
    "language": "ja",
    "estimatedDuration": 60
  }
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "script": "本日は新製品の発表にお越しいただき、誠にありがとうございます。この製品は、革新的なAI技術により、お客様の業務効率を大幅に向上させます。また、直感的なUIにより、誰でもすぐに使いこなすことができます。さらに、完全自動化により、24時間365日の運用が可能となり、人的コストを大幅に削減できます。",
    "estimatedDuration": 58,
    "wordCount": 120,
    "alternatives": [
      "皆様、本日はお忙しい中お集まりいただき、ありがとうございます..."
    ]
  }
}
```

#### テンプレート選択支援
```
POST /ai/suggest-layout
```

**リクエストボディ**:
```json
{
  "content": {
    "title": "売上推移",
    "text": "過去3年間の売上は順調に成長しています。",
    "hasImage": true,
    "hasBullets": false
  },
  "availableTemplates": ["template_content", "template_image_left", "template_image_full"]
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "suggestedTemplateId": "template_image_left",
    "score": 0.92,
    "reasoning": "画像（グラフ）を左側に配置し、説明テキストを右側に配置するレイアウトが最適です。",
    "alternatives": [
      {
        "templateId": "template_content",
        "score": 0.75,
        "reasoning": "テキスト中心のレイアウトも可能ですが、視覚的インパクトが弱まります。"
      }
    ]
  }
}
```

### 2. 画像生成（Stable Diffusion）

#### 画像生成
```
POST /ai/generate-image
```

**リクエストボディ**:
```json
{
  "prompt": "modern office with technology, professional, bright lighting",
  "parameters": {
    "width": 1024,
    "height": 768,
    "steps": 30,
    "cfgScale": 7.5,
    "negativePrompt": "blurry, low quality, distorted",
    "style": "realistic",
    "seed": null,
    "sampler": "Euler a"
  }
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "imageBase64": "data:image/png;base64,iVBORw0KGgoAAAANS...",
    "metadata": {
      "prompt": "modern office with technology, professional, bright lighting",
      "seed": 42,
      "model": "stable-diffusion-v1.5",
      "steps": 30,
      "cfgScale": 7.5,
      "size": {
        "width": 1024,
        "height": 768
      }
    }
  }
}
```

#### アイコン提案（AI支援）
```
POST /ai/suggest-icons
```

**リクエストボディ**:
```json
{
  "context": "financial growth and analytics",
  "parameters": {
    "maxSuggestions": 5,
    "iconSets": ["heroicons", "feather"]
  }
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "suggestions": [
      {
        "iconId": "icon_chart_bar",
        "iconName": "chart-bar",
        "iconSet": "heroicons",
        "relevance": 0.95,
        "reasoning": "棒グラフは成長を視覚的に表現するのに最適です"
      },
      {
        "iconId": "icon_trending_up",
        "iconName": "trending-up",
        "iconSet": "feather",
        "relevance": 0.92,
        "reasoning": "上昇トレンドは金融成長を直接的に示します"
      }
    ]
  }
}
```

### 3. TTS（音声合成）

#### 音声合成（リアルタイム）
```
POST /ai/tts/synthesize
```

**リクエストボディ**:
```json
{
  "text": "こんにちは、これはテスト音声です。",
  "voiceSettings": {
    "voice": "ja-JP-Neural2-B",
    "rate": 1.0,
    "pitch": 1.0,
    "volume": 1.0
  },
  "outputFormat": "mp3"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "audioBase64": "data:audio/mp3;base64,SUQzBAAAAAAAI1RTU0UAAAA...",
    "duration": 3.5,
    "format": "mp3",
    "sampleRate": 22050
  }
}
```

#### 利用可能な音声一覧
```
GET /ai/tts/voices
```

**クエリパラメータ**:
- `language` (string, optional): 言語フィルター（ja, en等）
- `gender` (string, optional): 性別フィルター（male, female）

**レスポンス**:
```json
{
  "success": true,
  "data": [
    {
      "id": "ja-JP-Neural2-B",
      "name": "Japanese Female",
      "language": "ja-JP",
      "gender": "female",
      "provider": "web-speech"
    },
    {
      "id": "ja-JP-Neural2-C",
      "name": "Japanese Male",
      "language": "ja-JP",
      "gender": "male",
      "provider": "coqui-tts"
    }
  ]
}
```

### 4. AI設定テスト

#### AI接続テスト
```
POST /ai/test-connection
```

**リクエストボディ**:
```json
{
  "type": "llm",
  "config": {
    "provider": "ollama",
    "apiUrl": "http://localhost:11434",
    "model": "llama3.2"
  }
}
```

**レスポンス（成功）**:
```json
{
  "success": true,
  "data": {
    "connected": true,
    "latency": 150,
    "modelInfo": {
      "name": "llama3.2",
      "size": "3B",
      "quantization": "Q4_K_M"
    }
  }
}
```

**レスポンス（失敗）**:
```json
{
  "success": false,
  "error": {
    "code": "AI_CONNECTION_ERROR",
    "message": "Ollamaへの接続に失敗しました",
    "details": {
      "provider": "ollama",
      "apiUrl": "http://localhost:11434",
      "errorMessage": "Connection refused"
    }
  }
}
```

## システムデータAPI


サーバーは、プリセットテンプレート、アイコンライブラリ、テーマなどのシステムデータを提供します。
これらは**読み取り専用**で、サーバー側のSQLiteデータベースに格納されています。

### 1. Templates（テンプレート）

#### テンプレート一覧取得
```
GET /templates
```

**クエリパラメータ**:
- `category` (string, optional): カテゴリフィルター（title, content, image, bullets等）
- `tags` (string, optional): タグフィルター（カンマ区切り）
- `page` (number, optional): ページ番号（デフォルト: 1）
- `limit` (number, optional): 1ページあたりの件数（デフォルト: 20）

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "template_title_modern",
        "name": "モダンタイトルスライド",
        "description": "シンプルでモダンなタイトルスライド。プレゼンテーションの冒頭に最適。",
        "category": "title",
        "tags": ["modern", "simple", "professional", "business"],
        "size": {
          "width": 1920,
          "height": 1080,
          "aspectRatio": "16:9"
        },
        "areas": [
          {
            "id": "title",
            "type": "title",
            "position": { "x": "10%", "y": "30%", "width": "80%", "height": "20%" },
            "constraints": {
              "minWidth": 600,
              "maxCharacters": 60,
              "maxLines": 2,
              "required": true
            },
            "defaultStyle": {
              "text": {
                "fontSize": 72,
                "fontFamily": "Arial, sans-serif",
                "fontWeight": "bold",
                "color": "#1a1a1a",
                "align": "center"
              }
            }
          }
        ],
        "selectionCriteria": {
          "category": "title",
          "tags": ["modern", "simple"],
          "contentFit": {
            "textAmount": "minimal",
            "imageCount": "none",
            "hasBullets": false
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
        "previewUrl": "/api/v1/templates/template_title_modern/preview.jpg"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 50,
      "totalPages": 3
    }
  }
}
```

#### テンプレート詳細取得
```
GET /templates/:id
```

**パスパラメータ**:
- `id`: テンプレートID

**レスポンス**: 完全なテンプレートオブジェクト（areas、selectionCriteria、styleGuide を含む）

**レスポンス例**:
```json
{
  "success": true,
  "data": {
    "id": "template_title_modern",
    "name": "モダンタイトルスライド",
    "description": "シンプルでモダンなタイトルスライド",
    "category": "title",
    "tags": ["modern", "simple", "professional"],
    "size": {
      "width": 1920,
      "height": 1080,
      "aspectRatio": "16:9"
    },
    "areas": [ /* 詳細な area 定義 */ ],
    "selectionCriteria": { /* AI選択用メタデータ */ },
    "styleGuide": {
      "colorPalette": {
        "primary": "#667eea",
        "secondary": "#764ba2",
        "accent": "#f093fb",
        "background": "#FFFFFF",
        "text": "#1a1a1a"
      },
      "typography": {
        "headingFont": "Arial, sans-serif",
        "bodyFont": "Noto Sans JP, sans-serif",
        "codeFont": "Fira Code, monospace"
      },
      "spacing": {
        "baseUnit": 8,
        "gutters": 24,
        "margins": 48
      }
    },
    "previewUrl": "/api/v1/templates/template_title_modern/preview.jpg"
  }
}
```

#### テンプレートプレビュー画像取得
```
GET /templates/:id/preview.jpg
```

**レスポンス**: 画像ファイル（JPEG）

#### デフォルトテンプレート一覧
システムには以下の10種類のデフォルトテンプレートが含まれています：

| ID | 名前 | カテゴリ | 説明 |
|----|------|----------|------|
| template_title | タイトルスライド | title | プレゼンテーションの冒頭 |
| template_content | コンテンツスライド | content | 一般的なコンテンツ |
| template_two_column | 2カラムレイアウト | content | 比較、並列情報 |
| template_image_left | 画像左配置 | image | 画像+テキスト説明 |
| template_image_right | 画像右配置 | image | 画像+テキスト説明 |
| template_image_full | 画像フル | image | 視覚的に強いスライド |
| template_bullets | 箇条書き | bullets | リスト、要点 |
| template_quote | 引用 | quote | 引用、強調 |
| template_section | セクションヘッダー | section | 章の区切り |
| template_comparison | 比較レイアウト | comparison | Before/After、対比 |

### 2. Icons（アイコンライブラリ）

#### アイコン検索
```
GET /icons/search
```

**クエリパラメータ**:
- `q` (string, required): 検索キーワード（日本語・英語対応）
- `iconSet` (string, optional): アイコンセット（heroicons, feather, material-icons, font-awesome）
- `category` (string, optional): カテゴリフィルター（actions, arrows, business, communication等）
- `style` (string, optional): スタイル（outline, solid）
- `limit` (number, optional): 最大結果数（デフォルト: 20）

**レスポンス**:
```json
{
  "success": true,
  "data": [
    {
      "id": "heroicons_check_circle",
      "name": "check-circle",
      "iconSet": "heroicons",
      "category": "actions",
      "style": "outline",
      "tags": ["check", "success", "done", "complete", "確認", "完了"],
      "svgContent": "<svg xmlns=\"http://www.w3.org/2000/svg\" fill=\"none\" viewBox=\"0 0 24 24\" stroke=\"currentColor\"><path stroke-linecap=\"round\" stroke-linejoin=\"round\" stroke-width=\"2\" d=\"M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z\" /></svg>",
      "previewUrl": "/api/v1/icons/heroicons_check_circle/preview.svg"
    }
  ]
}
```

#### アイコン一覧取得
```
GET /icons
```

**クエリパラメータ**:
- `iconSet` (string, optional): アイコンセット
- `category` (string, optional): カテゴリ
- `style` (string, optional): スタイル
- `page` (number, optional): ページ番号
- `limit` (number, optional): 1ページあたりの件数（デフォルト: 50）

**レスポンス**: アイコンリストとページネーション情報

#### アイコン詳細取得
```
GET /icons/:id
```

**パスパラメータ**:
- `id`: アイコンID

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "id": "heroicons_check_circle",
    "name": "check-circle",
    "iconSet": "heroicons",
    "category": "actions",
    "style": "outline",
    "tags": ["check", "success", "done", "complete"],
    "svgContent": "<svg>...</svg>",
    "variants": [
      {
        "style": "outline",
        "svgContent": "<svg>...</svg>"
      },
      {
        "style": "solid",
        "svgContent": "<svg>...</svg>"
      }
    ],
    "license": "MIT",
    "attribution": "Heroicons by Tailwind Labs"
  }
}
```

#### アイコンカテゴリ一覧
```
GET /icons/categories
```

**レスポンス**:
```json
{
  "success": true,
  "data": [
    { "id": "actions", "name": "Actions", "count": 120 },
    { "id": "arrows", "name": "Arrows & Directions", "count": 80 },
    { "id": "business", "name": "Business", "count": 150 },
    { "id": "communication", "name": "Communication", "count": 90 },
    { "id": "devices", "name": "Devices", "count": 60 },
    { "id": "media", "name": "Media", "count": 70 },
    { "id": "social", "name": "Social", "count": 50 }
  ]
}
```

#### 利用可能なアイコンセット一覧
```
GET /icons/sets
```

**レスポンス**:
```json
{
  "success": true,
  "data": [
    {
      "id": "heroicons",
      "name": "Heroicons",
      "version": "2.0",
      "iconCount": 292,
      "styles": ["outline", "solid", "mini"],
      "license": "MIT",
      "url": "https://heroicons.com"
    },
    {
      "id": "feather",
      "name": "Feather Icons",
      "version": "4.29",
      "iconCount": 287,
      "styles": ["outline"],
      "license": "MIT",
      "url": "https://feathericons.com"
    },
    {
      "id": "material-icons",
      "name": "Material Icons",
      "version": "3.0",
      "iconCount": 2000+,
      "styles": ["filled", "outlined", "rounded", "sharp", "two-tone"],
      "license": "Apache 2.0",
      "url": "https://fonts.google.com/icons"
    }
  ]
}
```

### 3. Themes（テーマ）

#### テーマ一覧取得
```
GET /themes
```

**クエリパラメータ**:
- `category` (string, optional): カテゴリフィルター（business, creative, academic, minimal等）
- `page` (number, optional): ページ番号
- `limit` (number, optional): 1ページあたりの件数

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "theme_professional",
        "name": "Professional",
        "description": "ビジネス向けのプロフェッショナルなテーマ",
        "category": "business",
        "colors": {
          "primary": "#667eea",
          "secondary": "#764ba2",
          "accent": "#f093fb",
          "background": "#FFFFFF",
          "surface": "#F7FAFC",
          "text": {
            "primary": "#1a202c",
            "secondary": "#4a5568",
            "disabled": "#a0aec0"
          },
          "border": "#e2e8f0",
          "error": "#f56565",
          "warning": "#ed8936",
          "success": "#48bb78",
          "info": "#4299e1"
        },
        "typography": {
          "fontFamilies": {
            "heading": "Inter, system-ui, sans-serif",
            "body": "Inter, system-ui, sans-serif",
            "mono": "Fira Code, monospace"
          },
          "fontSizes": {
            "h1": 72,
            "h2": 56,
            "h3": 40,
            "h4": 32,
            "body": 16,
            "small": 14,
            "tiny": 12
          },
          "fontWeights": {
            "light": 300,
            "regular": 400,
            "medium": 500,
            "semibold": 600,
            "bold": 700
          },
          "lineHeights": {
            "tight": 1.2,
            "normal": 1.5,
            "relaxed": 1.75
          }
        },
        "spacing": {
          "baseUnit": 8,
          "scale": [0, 4, 8, 12, 16, 24, 32, 48, 64, 96, 128]
        },
        "borderRadius": {
          "none": 0,
          "sm": 4,
          "md": 8,
          "lg": 16,
          "full": 9999
        },
        "shadows": {
          "sm": "0 1px 2px 0 rgba(0, 0, 0, 0.05)",
          "md": "0 4px 6px -1px rgba(0, 0, 0, 0.1)",
          "lg": "0 10px 15px -3px rgba(0, 0, 0, 0.1)",
          "xl": "0 20px 25px -5px rgba(0, 0, 0, 0.1)"
        },
        "previewUrl": "/api/v1/themes/theme_professional/preview.jpg"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 15,
      "totalPages": 1
    }
  }
}
```

#### テーマ詳細取得
```
GET /themes/:id
```

**パスパラメータ**:
- `id`: テーマID

**レスポンス**: 完全なテーマオブジェクト

#### デフォルトテーマ一覧
システムには以下のデフォルトテーマが含まれています：

| ID | 名前 | カテゴリ | 説明 |
|----|------|----------|------|
| theme_professional | Professional | business | ビジネス向けプロフェッショナル |
| theme_creative | Creative | creative | クリエイティブで鮮やか |
| theme_minimal | Minimal | minimal | ミニマルでシンプル |
| theme_academic | Academic | academic | 学術・教育向け |
| theme_dark | Dark Mode | general | ダークモード |

## 設定API

サーバー側で管理されるAI接続設定などを取得・更新します。
ユーザーのUI設定やプロジェクトデータはクライアント側（localStorage/IndexedDB）で管理されるため、サーバーAPIには含まれません。

#### 設定取得
```
GET /settings
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "ai": {
      "llm": {
        "provider": "ollama",
        "apiUrl": "http://localhost:11434",
        "model": "llama3.2",
        "temperature": 0.7,
        "maxTokens": 2000,
        "availableModels": ["llama3.2", "llama3.1", "mistral", "codellama"]
      },
      "imageGen": {
        "provider": "stable-diffusion",
        "apiUrl": "http://localhost:7860",
        "model": "sd-v1.5",
        "steps": 30,
        "cfgScale": 7.5,
        "availableModels": ["sd-v1.5", "sd-xl", "sd-v2.1"]
      },
      "tts": {
        "provider": "web-speech",
        "defaultVoice": "ja-JP-Neural2-B",
        "defaultRate": 1.0,
        "defaultPitch": 1.0
      }
    }
  }
}
```

#### 設定更新
```
PUT /settings
```

**リクエストボディ**: 更新するフィールドのみ

**例**:
```json
{
  "ai": {
    "llm": {
      "provider": "ollama",
      "apiUrl": "http://localhost:11434",
      "model": "llama3.2",
      "temperature": 0.8
    }
  }
}
```

**レスポンス**: 更新後の設定オブジェクト

**注意**: これはサーバー側のAI接続設定です。ユーザーのUI設定（グリッド表示、スナップ設定など）はクライアント側のlocalStorageで管理されます。

## エクスポートAPI

クライアント側でエクスポート処理を行うことが基本ですが、サーバー側でのレンダリングが必要な場合（複雑なPDF生成、PPTXなど）にこのAPIを使用します。

### 1. PDF エクスポート（サーバー側レンダリング）

#### PDFエクスポート開始
```
POST /export/pdf
```

**リクエストボディ**:
```json
{
  "slides": [
    {
      "id": "slide_001",
      "title": "タイトル",
      "elements": [ /* 要素の配列 */ ],
      "background": { /* 背景設定 */ }
    }
  ],
  "options": {
    "pageSize": "slide",
    "orientation": "landscape",
    "quality": "high",
    "includeNotes": false,
    "imageFit": "contain"
  }
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "exportId": "export_001",
    "status": "processing",
    "estimatedTime": 10
  }
}
```

#### エクスポート状態確認
```
GET /export/:exportId
```

**パスパラメータ**:
- `exportId`: エクスポートID

**レスポンス（処理中）**:
```json
{
  "success": true,
  "data": {
    "id": "export_001",
    "status": "processing",
    "progress": 45
  }
}
```

**レスポンス（完了）**:
```json
{
  "success": true,
  "data": {
    "id": "export_001",
    "status": "completed",
    "fileUrl": "/api/v1/export/export_001/download",
    "fileSize": 2048576,
    "format": "pdf",
    "expiresAt": "2025-10-25T10:00:00Z"
  }
}
```

#### エクスポートファイルダウンロード
```
GET /export/:exportId/download
```

**レスポンス**: ファイル（application/pdf）

### 2. PPTX エクスポート（Phase 3）

#### PPTXエクスポート開始
```
POST /export/pptx
```

**リクエストボディ**: PDF と同様の構造

**レスポンス**: エクスポートIDとステータス

### 3. 動画エクスポート（音声付き）（Phase 4）

#### 動画エクスポート開始
```
POST /export/video
```

**リクエストボディ**:
```json
{
  "slides": [ /* スライドデータ */ ],
  "options": {
    "format": "mp4",
    "resolution": "1080p",
    "fps": 30,
    "quality": "high",
    "includeAudio": true,
    "audioTracks": [
      {
        "slideId": "slide_001",
        "audioBase64": "data:audio/mp3;base64,...",
        "duration": 15.5
      }
    ],
    "includeSubtitles": false
  }
}
```

**レスポンス**: エクスポートIDとステータス

**注意**: 動画エクスポートは処理時間が長いため、WebSocket通知を推奨します。

## WebSocket API

リアルタイム通知用（Phase 3以降）。AI生成の進捗、エクスポートの完了通知などに使用します。

### 接続
```
ws://localhost:3001/ws
```

### イベント

#### エクスポート完了通知（受信）
```json
{
  "type": "export_completed",
  "payload": {
    "exportId": "export_001",
    "status": "completed",
    "fileUrl": "/api/v1/export/export_001/download"
  }
}
```

#### AI生成完了通知（受信）
```json
{
  "type": "ai_generation_completed",
  "payload": {
    "generationId": "gen_001",
    "type": "content",
    "status": "completed",
    "result": { /* 生成結果 */ }
  }
}
```

#### AI生成進捗通知（受信）
```json
{
  "type": "ai_generation_progress",
  "payload": {
    "generationId": "gen_001",
    "progress": 45,
    "currentStep": "Generating slide 5 of 10"
  }
}
```

## エラーハンドリング

### エラーコード一覧

| コード | 説明 | HTTPステータス |
|--------|------|---------------|
| VALIDATION_ERROR | バリデーションエラー | 422 |
| NOT_FOUND | リソースが見つからない | 404 |
| DUPLICATE | 重複エラー | 409 |
| UNAUTHORIZED | 認証エラー | 401 |
| FORBIDDEN | 権限エラー | 403 |
| INTERNAL_ERROR | サーバー内部エラー | 500 |
| AI_ERROR | AI処理エラー | 500 |
| AI_TIMEOUT | AIタイムアウト | 504 |
| FILE_TOO_LARGE | ファイルサイズ超過 | 413 |
| UNSUPPORTED_FORMAT | 非サポート形式 | 415 |

### エラーレスポンス例

#### バリデーションエラー
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力値が不正です",
    "details": {
      "fields": {
        "name": ["名前は必須です"],
        "email": ["有効なメールアドレスを入力してください"]
      }
    }
  }
}
```

#### AI エラー
```json
{
  "success": false,
  "error": {
    "code": "AI_ERROR",
    "message": "AI処理中にエラーが発生しました",
    "details": {
      "provider": "ollama",
      "apiError": "Connection refused",
      "suggestion": "AI サービスが起動していることを確認してください"
    }
  }
}
```

## レート制限

### 制限値
- 一般API: 100リクエスト/分
- AI生成API: 10リクエスト/分
- ファイルアップロード: 5リクエスト/分

### レスポンスヘッダー
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1635724800
```

### 制限超過時
```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "リクエスト制限を超過しました",
    "details": {
      "retryAfter": 60
    }
  }
}
```

## ベストプラクティス

### 1. ポーリング
AI生成などの長時間処理は、適切な間隔でポーリングしてください：
- 初回: 即座
- 2回目以降: 2秒間隔
- 最大: 60秒でタイムアウト

### 2. バッチ処理
複数要素の更新は、個別ではなくバッチAPIを使用してください：
```
PUT /slides/:slideId/elements/batch
```

### 3. キャッシング
テンプレートやテーマなど、変更頻度の低いデータはクライアント側でキャッシュしてください。

### 4. エラーハンドリング
すべてのAPIコールでエラーハンドリングを実装してください：
```typescript
try {
  const response = await fetch('/api/v1/projects');
  const data = await response.json();

  if (!data.success) {
    // エラー処理
    handleError(data.error);
  }

  // 成功処理
  return data.data;
} catch (error) {
  // ネットワークエラーなど
  handleNetworkError(error);
}
```

## API クライアント例

### TypeScript クライアント
```typescript
class APIClient {
  private baseUrl: string;

  constructor(baseUrl: string = 'http://localhost:3000/api/v1') {
    this.baseUrl = baseUrl;
  }

  async request<T>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
      ...options,
    });

    const data = await response.json();

    if (!data.success) {
      throw new APIError(data.error);
    }

    return data.data;
  }

  // Projects
  async getProjects(params?: GetProjectsParams): Promise<ProjectList> {
    const query = new URLSearchParams(params as any).toString();
    return this.request(`/projects?${query}`);
  }

  async getProject(id: string, include?: string[]): Promise<Project> {
    const query = include ? `?include=${include.join(',')}` : '';
    return this.request(`/projects/${id}${query}`);
  }

  async createProject(data: CreateProjectInput): Promise<Project> {
    return this.request('/projects', {
      method: 'POST',
      body: JSON.stringify(data),
    });
  }

  // Slides
  async createSlide(
    projectId: string,
    data: CreateSlideInput
  ): Promise<Slide> {
    return this.request(`/projects/${projectId}/slides`, {
      method: 'POST',
      body: JSON.stringify(data),
    });
  }

  // AI
  async generateStructure(
    data: GenerateStructureInput
  ): Promise<{ generationId: string; status: string }> {
    return this.request('/ai/generate-structure', {
      method: 'POST',
      body: JSON.stringify(data),
    });
  }

  async pollGeneration(generationId: string): Promise<AIGeneration> {
    return this.request(`/ai/generations/${generationId}`);
  }
}

class APIError extends Error {
  constructor(public error: ErrorResponse) {
    super(error.message);
    this.name = 'APIError';
  }
}
```

### 使用例
```typescript
const api = new APIClient();

// プロジェクト作成
const project = await api.createProject({
  name: '新規プレゼンテーション',
  themeId: 'theme_professional',
});

// AI でスライド構成生成
const { generationId } = await api.generateStructure({
  projectId: project.id,
  prompt: '会社紹介プレゼンテーション',
  parameters: {
    slideCount: 10,
    tone: 'formal',
  },
});

// ポーリング
const result = await pollUntilComplete(generationId);

// スライド作成
for (const slideData of result.slides) {
  await api.createSlide(project.id, {
    title: slideData.title,
    templateId: getTemplateForCategory(slideData.suggestedLayout),
  });
}
```

---

**最終更新**: 2025-10-24
**APIバージョン**: v1
