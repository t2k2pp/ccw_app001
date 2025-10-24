# API設計書

## 目次
1. [概要](#概要)
2. [認証](#認証)
3. [共通仕様](#共通仕様)
4. [エンドポイント](#エンドポイント)
5. [WebSocket API](#websocket-api)
6. [エラーハンドリング](#エラーハンドリング)

## 概要

### ベースURL
```
http://localhost:3000/api/v1
```

### プロトコル
- REST API（HTTP/HTTPS）
- WebSocket（リアルタイム更新用、将来拡張）

### データフォーマット
- リクエスト: JSON
- レスポンス: JSON
- ファイルアップロード: multipart/form-data

## 認証

MVP版では認証なし（シングルユーザー想定）

将来拡張でJWT認証を実装予定：
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

#### ページネーション
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

## エンドポイント

### 1. Projects（プロジェクト）

#### プロジェクト一覧取得
```
GET /projects
```

**クエリパラメータ**:
- `page` (number, optional): ページ番号（デフォルト: 1）
- `limit` (number, optional): 1ページあたりの件数（デフォルト: 20）
- `sort` (string, optional): ソート順（`createdAt`, `updatedAt`, `name`）
- `order` (string, optional): 昇順/降順（`asc`, `desc`）

**レスポンス例**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "proj_123",
        "name": "新製品発表",
        "description": "Q4新製品のプレゼンテーション",
        "themeId": "theme_001",
        "settings": { /* ProjectSettings */ },
        "createdAt": "2025-10-24T10:00:00Z",
        "updatedAt": "2025-10-24T15:30:00Z",
        "slideCount": 15,
        "thumbnail": "/thumbnails/proj_123.jpg"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 5,
      "totalPages": 1
    }
  }
}
```

#### プロジェクト詳細取得
```
GET /projects/:id
```

**パスパラメータ**:
- `id`: プロジェクトID

**クエリパラメータ**:
- `include` (string, optional): 含めるリレーション（カンマ区切り）
  - `slides`: スライドを含める
  - `theme`: テーマを含める
  - `assets`: アセットを含める

**レスポンス例**:
```json
{
  "success": true,
  "data": {
    "id": "proj_123",
    "name": "新製品発表",
    "description": "Q4新製品のプレゼンテーション",
    "themeId": "theme_001",
    "settings": {
      "slideSize": {
        "width": 1920,
        "height": 1080,
        "aspectRatio": "16:9"
      },
      "defaults": {
        "background": {
          "type": "solid",
          "color": "#FFFFFF"
        },
        "font": {
          "family": "Noto Sans JP",
          "size": 16,
          "color": "#333333"
        }
      }
    },
    "createdAt": "2025-10-24T10:00:00Z",
    "updatedAt": "2025-10-24T15:30:00Z",
    "slides": [ /* Slide配列（include=slidesの場合） */ ],
    "theme": { /* Theme（include=themeの場合） */ }
  }
}
```

#### プロジェクト作成
```
POST /projects
```

**リクエストボディ**:
```json
{
  "name": "新規プロジェクト",
  "description": "プロジェクトの説明",
  "themeId": "theme_001",
  "templateId": "template_title",
  "settings": {
    "slideSize": {
      "width": 1920,
      "height": 1080,
      "aspectRatio": "16:9"
    }
  }
}
```

**レスポンス**: `201 Created` + プロジェクトオブジェクト

#### プロジェクト更新
```
PUT /projects/:id
```

**リクエストボディ**:
```json
{
  "name": "更新後のプロジェクト名",
  "description": "更新後の説明",
  "themeId": "theme_002"
}
```

**レスポンス**: `200 OK` + 更新後のプロジェクトオブジェクト

#### プロジェクト削除
```
DELETE /projects/:id
```

**レスポンス**: `204 No Content`

#### プロジェクト複製
```
POST /projects/:id/duplicate
```

**リクエストボディ**:
```json
{
  "name": "コピー - 新製品発表"
}
```

**レスポンス**: `201 Created` + 複製されたプロジェクトオブジェクト

### 2. Slides（スライド）

#### スライド一覧取得
```
GET /projects/:projectId/slides
```

**クエリパラメータ**:
- `include` (string, optional): `elements` で要素も含める

**レスポンス例**:
```json
{
  "success": true,
  "data": [
    {
      "id": "slide_001",
      "projectId": "proj_123",
      "templateId": "template_title",
      "title": "新製品発表",
      "notes": "プレゼンターノート",
      "orderIndex": 0,
      "background": {
        "type": "solid",
        "color": "#FFFFFF"
      },
      "createdAt": "2025-10-24T10:00:00Z",
      "updatedAt": "2025-10-24T15:30:00Z",
      "thumbnail": "/thumbnails/slide_001.jpg"
    }
  ]
}
```

#### スライド詳細取得
```
GET /slides/:id
```

**クエリパラメータ**:
- `include` (string, optional): `elements`, `template`

**レスポンス**: スライドオブジェクト（要素を含む）

#### スライド作成
```
POST /projects/:projectId/slides
```

**リクエストボディ**:
```json
{
  "templateId": "template_content",
  "title": "スライドタイトル",
  "orderIndex": 5,
  "background": {
    "type": "solid",
    "color": "#F0F0F0"
  }
}
```

**レスポンス**: `201 Created` + スライドオブジェクト

#### スライド更新
```
PUT /slides/:id
```

**リクエストボディ**:
```json
{
  "title": "更新後のタイトル",
  "notes": "更新後のノート",
  "background": {
    "type": "gradient",
    "gradient": {
      "type": "linear",
      "angle": 45,
      "colors": [
        { "color": "#667eea", "offset": 0 },
        { "color": "#764ba2", "offset": 1 }
      ]
    }
  }
}
```

**レスポンス**: `200 OK` + 更新後のスライドオブジェクト

#### スライド削除
```
DELETE /slides/:id
```

**レスポンス**: `204 No Content`

#### スライド並び替え
```
PUT /projects/:projectId/slides/reorder
```

**リクエストボディ**:
```json
{
  "slideIds": ["slide_003", "slide_001", "slide_002"]
}
```

**レスポンス**: `200 OK`

#### スライド複製
```
POST /slides/:id/duplicate
```

**レスポンス**: `201 Created` + 複製されたスライドオブジェクト

### 3. Elements（要素）

#### 要素一覧取得
```
GET /slides/:slideId/elements
```

**レスポンス**: 要素の配列

#### 要素作成
```
POST /slides/:slideId/elements
```

**リクエストボディ（テキスト要素の例）**:
```json
{
  "type": "text",
  "position": { "x": 100, "y": 100 },
  "size": { "width": 400, "height": 100 },
  "content": {
    "text": "Hello World",
    "format": "plain",
    "styles": {
      "fontFamily": "Noto Sans JP",
      "fontSize": 32,
      "fontWeight": "bold",
      "color": "#333333",
      "textAlign": "center"
    }
  },
  "zIndex": 1
}
```

**リクエストボディ（画像要素の例）**:
```json
{
  "type": "image",
  "position": { "x": 200, "y": 200 },
  "size": { "width": 600, "height": 400 },
  "content": {
    "url": "/assets/image_001.jpg",
    "assetId": "asset_001",
    "fit": "cover"
  },
  "zIndex": 0
}
```

**レスポンス**: `201 Created` + 要素オブジェクト

#### 要素更新
```
PUT /elements/:id
```

**リクエストボディ**: 更新するフィールドのみ

**レスポンス**: `200 OK` + 更新後の要素オブジェクト

#### 要素削除
```
DELETE /elements/:id
```

**レスポンス**: `204 No Content`

#### 複数要素の一括更新
```
PUT /slides/:slideId/elements/batch
```

**リクエストボディ**:
```json
{
  "updates": [
    {
      "id": "elem_001",
      "position": { "x": 150, "y": 150 }
    },
    {
      "id": "elem_002",
      "zIndex": 5
    }
  ]
}
```

**レスポンス**: `200 OK` + 更新後の要素配列

### 4. Templates（テンプレート）

#### テンプレート一覧取得
```
GET /templates
```

**クエリパラメータ**:
- `category` (string, optional): カテゴリフィルター

**レスポンス**:
```json
{
  "success": true,
  "data": [
    {
      "id": "template_title",
      "name": "タイトルスライド",
      "description": "プレゼンテーションのタイトルスライド",
      "category": "title",
      "layout": {
        "placeholders": [
          {
            "id": "title",
            "type": "text",
            "position": { "x": 960, "y": 400 },
            "size": { "width": 800, "height": 150 },
            "defaultContent": {
              "placeholder": "タイトルを入力"
            }
          }
        ]
      },
      "previewUrl": "/templates/previews/title.jpg"
    }
  ]
}
```

#### テンプレート詳細取得
```
GET /templates/:id
```

**レスポンス**: テンプレートオブジェクト

### 5. Themes（テーマ）

#### テーマ一覧取得
```
GET /themes
```

**クエリパラメータ**:
- `category` (string, optional): カテゴリフィルター

**レスポンス**: テーマの配列

#### テーマ詳細取得
```
GET /themes/:id
```

**レスポンス**: テーマオブジェクト

#### カスタムテーマ作成（Phase 3）
```
POST /themes
```

**リクエストボディ**: Themeオブジェクト

**レスポンス**: `201 Created` + テーマオブジェクト

### 6. Assets（アセット）

#### アセット一覧取得
```
GET /projects/:projectId/assets
```

**クエリパラメータ**:
- `type` (string, optional): アセットタイプフィルター

**レスポンス**: アセットの配列

#### アセットアップロード
```
POST /projects/:projectId/assets
```

**リクエスト**: `multipart/form-data`
- `file`: ファイル
- `metadata`: JSON文字列（オプション）

**レスポンス**: `201 Created` + アセットオブジェクト

#### アセット削除
```
DELETE /assets/:id
```

**レスポンス**: `204 No Content`

### 7. AI（AI生成）

#### スライド構成生成
```
POST /ai/generate-structure
```

**リクエストボディ**:
```json
{
  "projectId": "proj_123",
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
    "generationId": "gen_001",
    "status": "processing"
  }
}
```

**生成完了後のポーリング**:
```
GET /ai/generations/:generationId
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "id": "gen_001",
    "status": "completed",
    "result": {
      "slides": [
        {
          "title": "新製品紹介",
          "bulletPoints": [
            "市場の課題",
            "ソリューション",
            "競合優位性"
          ],
          "suggestedLayout": "content",
          "notes": "冒頭で市場の課題を明確に"
        }
      ]
    }
  }
}
```

#### コンテンツ生成
```
POST /ai/generate-content
```

**リクエストボディ**:
```json
{
  "slideId": "slide_001",
  "type": "text",
  "prompt": "新製品の特徴を3つの箇条書きで",
  "parameters": {
    "maxLength": 500,
    "tone": "persuasive",
    "language": "ja"
  }
}
```

**レスポンス**: 生成IDとステータス

#### 画像生成
```
POST /ai/generate-image
```

**リクエストボディ**:
```json
{
  "slideId": "slide_002",
  "prompt": "modern office with technology",
  "parameters": {
    "width": 1024,
    "height": 768,
    "steps": 30,
    "cfgScale": 7.5,
    "negativePrompt": "blurry, low quality",
    "style": "realistic"
  }
}
```

**レスポンス**: 生成IDとステータス

**生成完了後**:
```json
{
  "success": true,
  "data": {
    "id": "gen_002",
    "status": "completed",
    "result": {
      "imageUrl": "/assets/generated_001.jpg",
      "assetId": "asset_123",
      "metadata": {
        "prompt": "modern office with technology",
        "seed": 42,
        "model": "stable-diffusion-v1.5"
      }
    }
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
  "elementId": "elem_001",
  "currentContent": "このプロダクトはすごいです",
  "improvementType": "professional",
  "parameters": {
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
  "slideId": "slide_001",
  "parameters": {
    "length": "standard",
    "tone": "formal",
    "targetAudience": "business",
    "language": "ja"
  }
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "generationId": "gen_003",
    "status": "processing"
  }
}
```

**生成完了後**:
```json
{
  "success": true,
  "data": {
    "id": "gen_003",
    "status": "completed",
    "result": {
      "script": "本日は新製品の発表にお越しいただき、誠にありがとうございます。この製品は、お客様のビジネス課題を解決するために開発されました。",
      "estimatedDuration": 15,
      "alternatives": [
        "皆様、本日はお忙しい中お集まりいただき、ありがとうございます...",
        "ご来場の皆様、この度は弊社の新製品発表会にご参加いただき..."
      ]
    }
  }
}
```

#### アイコン提案
```
POST /ai/suggest-icons
```

**リクエストボディ**:
```json
{
  "slideId": "slide_002",
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

### 8. Icons（アイコン）

#### アイコン検索
```
GET /icons/search
```

**クエリパラメータ**:
- `q` (string, required): 検索キーワード
- `iconSet` (string, optional): アイコンセット（heroicons, feather, material-icons, font-awesome）
- `category` (string, optional): カテゴリフィルター
- `limit` (number, optional): 最大結果数（デフォルト: 20）

**レスポンス**:
```json
{
  "success": true,
  "data": [
    {
      "id": "icon_001",
      "name": "check-circle",
      "iconSet": "heroicons",
      "category": "actions",
      "tags": ["check", "success", "done", "complete"],
      "svgContent": "<svg>...</svg>"
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
- `page` (number, optional): ページ番号
- `limit` (number, optional): 1ページあたりの件数

**レスポンス**: アイコンリストとページネーション情報

### 9. Assets（アセット）拡張

#### アセット検索（セマンティック検索）
```
POST /assets/search
```

**リクエストボディ**:
```json
{
  "projectId": "proj_123",
  "query": "business meeting",
  "type": "image",
  "useAI": true
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": [
    {
      "asset": {
        "id": "asset_456",
        "filename": "conference_room.jpg",
        "description": "Modern conference room with people",
        "tags": ["business", "meeting", "office"]
      },
      "relevance": 0.88
    }
  ]
}
```

#### アセットタグ更新
```
PUT /assets/:id/tags
```

**リクエストボディ**:
```json
{
  "tags": ["business", "teamwork", "collaboration"],
  "description": "Team collaboration in modern office"
}
```

**レスポンス**: 更新後のアセットオブジェクト

### 10. Signage（デジタルサイネージ）

#### サイネージ設定取得
```
GET /projects/:projectId/signage-settings
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "autoPlay": true,
    "loop": true,
    "defaultDisplayDuration": 10,
    "defaultTransition": {
      "type": "fade",
      "duration": 500,
      "easing": "ease-in-out"
    }
  }
}
```

#### サイネージ設定更新
```
PUT /projects/:projectId/signage-settings
```

**リクエストボディ**:
```json
{
  "autoPlay": true,
  "loop": true,
  "defaultDisplayDuration": 15,
  "defaultTransition": {
    "type": "slide-left",
    "duration": 800
  }
}
```

**レスポンス**: 更新後の設定

### 11. TTS（音声合成）

#### 音声合成（リアルタイム）
```
POST /tts/speak
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
  }
}
```

**レスポンス**: 音声データ（ストリーミング）または音声ファイルURL

#### 音声ファイル生成
```
POST /tts/generate-audio
```

**リクエストボディ**:
```json
{
  "slideId": "slide_001",
  "format": "mp3"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "audioUrl": "/audio/slide_001_voice.mp3",
    "duration": 15.5
  }
}
```

#### プロジェクト全体の音声生成
```
POST /tts/generate-project-audio
```

**リクエストボディ**:
```json
{
  "projectId": "proj_123",
  "format": "mp3",
  "mergeAll": true
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "jobId": "audio_job_001",
    "status": "processing"
  }
}
```

#### 利用可能な音声一覧
```
GET /tts/voices
```

**クエリパラメータ**:
- `language` (string, optional): 言語フィルター（ja, en等）

**レスポンス**:
```json
{
  "success": true,
  "data": [
    {
      "id": "ja-JP-Neural2-B",
      "name": "Japanese Female",
      "language": "ja-JP",
      "gender": "female"
    },
    {
      "id": "ja-JP-Neural2-C",
      "name": "Japanese Male",
      "language": "ja-JP",
      "gender": "male"
    }
  ]
}
```

### 12. Export（エクスポート）

#### PDF エクスポート
```
POST /export/pdf
```

**リクエストボディ**:
```json
{
  "projectId": "proj_123",
  "options": {
    "pageSize": "slide",
    "orientation": "landscape",
    "quality": "high",
    "includeNotes": false,
    "slideRange": {
      "start": 1,
      "end": 10
    }
  }
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "exportId": "export_001",
    "status": "processing"
  }
}
```

**エクスポート完了確認**:
```
GET /export/:exportId
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "id": "export_001",
    "status": "completed",
    "fileUrl": "/downloads/export_001.pdf",
    "fileSize": 2048576,
    "expiresAt": "2025-10-25T10:00:00Z"
  }
}
```

#### PPTX エクスポート（Phase 3）
```
POST /export/pptx
```

**リクエストボディ**: PDF と同様

#### HTML エクスポート（Phase 3）
```
POST /export/html
```

**リクエストボディ**:
```json
{
  "projectId": "proj_123",
  "options": {
    "format": "reveal.js",
    "theme": "black",
    "transition": "slide",
    "standalone": true
  }
}
```

#### 動画エクスポート（音声付き）（Phase 4）
```
POST /export/video
```

**リクエストボディ**:
```json
{
  "projectId": "proj_123",
  "options": {
    "format": "mp4",
    "resolution": "1080p",
    "fps": 30,
    "quality": "high",
    "includeAudio": true,
    "audioSource": "talkScript",
    "includeSubtitles": false
  }
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "exportId": "export_video_001",
    "status": "processing",
    "estimatedTime": 120
  }
}
```

**エクスポート完了確認**:
```
GET /export/:exportId
```

**レスポンス（完了時）**:
```json
{
  "success": true,
  "data": {
    "id": "export_video_001",
    "status": "completed",
    "fileUrl": "/downloads/presentation_with_audio.mp4",
    "fileSize": 52428800,
    "duration": 180,
    "expiresAt": "2025-10-25T10:00:00Z"
  }
}
```

### 13. Settings（設定）

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
        "maxTokens": 2000
      },
      "imageGen": {
        "provider": "stable-diffusion",
        "apiUrl": "http://localhost:7860",
        "model": "sd-v1.5",
        "steps": 30,
        "cfgScale": 7.5
      },
      "tts": {
        "provider": "web-speech",
        "defaultVoice": "ja-JP-Neural2-B",
        "defaultRate": 1.0,
        "defaultPitch": 1.0
      }
    },
    "signage": {
      "defaultDisplayDuration": 10,
      "defaultTransition": {
        "type": "fade",
        "duration": 500
      },
      "autoPlay": false,
      "loop": false
    },
    "editor": {
      "gridEnabled": true,
      "snapToGrid": true,
      "gridSize": 10,
      "undoLimit": 50
    },
    "export": {
      "defaultFormat": "pdf",
      "defaultQuality": "high"
    }
  }
}
```

#### 設定更新
```
PUT /settings
```

**リクエストボディ**: 更新するフィールド

**レスポンス**: 更新後の設定オブジェクト

#### AI接続テスト
```
POST /settings/ai/test
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

**レスポンス**:
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

## WebSocket API

### 接続
```
ws://localhost:3000/ws
```

### イベント

#### プロジェクトに参加
```json
{
  "type": "join_project",
  "payload": {
    "projectId": "proj_123"
  }
}
```

#### スライド更新通知（受信）
```json
{
  "type": "slide_updated",
  "payload": {
    "slideId": "slide_001",
    "updatedBy": "user_001",
    "changes": {
      "title": "新しいタイトル"
    }
  }
}
```

#### 要素更新通知（受信）
```json
{
  "type": "element_updated",
  "payload": {
    "elementId": "elem_001",
    "slideId": "slide_001",
    "changes": {
      "position": { "x": 200, "y": 200 }
    }
  }
}
```

#### AI生成完了通知（受信）
```json
{
  "type": "ai_generation_completed",
  "payload": {
    "generationId": "gen_001",
    "status": "completed",
    "result": { /* 生成結果 */ }
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
