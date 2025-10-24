# システムアーキテクチャ設計書

## 目次
1. [システム概要](#システム概要)
2. [アーキテクチャ図](#アーキテクチャ図)
3. [レイヤー構成](#レイヤー構成)
4. [コンポーネント設計](#コンポーネント設計)
5. [データフロー](#データフロー)
6. [技術スタック詳細](#技術スタック詳細)
7. [セキュリティ設計](#セキュリティ設計)
8. [パフォーマンス設計](#パフォーマンス設計)

## システム概要

Local AI Slide Creatorは、ローカル環境で動作するAI支援スライド作成アプリケーションです。
3層アーキテクチャを採用し、フロントエンド、バックエンド、AI層の明確な責務分離を実現します。

### アーキテクチャの特徴
- **完全ローカル動作**: すべての処理がユーザーのマシン内で完結
- **モジュラー設計**: 各機能が独立したモジュールとして実装
- **AI抽象化層**: 異なるAIバックエンドへの切り替えが容易
- **プラグインアーキテクチャ**: 拡張性の高い設計

## アーキテクチャ図

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Presentation │  │    Editor    │  │   Template   │         │
│  │     UI       │  │    Canvas    │  │   Manager    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │    Theme     │  │   Export     │  │    Asset     │         │
│  │   Manager    │  │   Manager    │  │   Manager    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Signage    │  │     TTS      │  │   Settings   │         │
│  │    Mode      │  │  Controller  │  │      UI      │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
                    HTTP/WebSocket
                              │
┌─────────────────────────────────────────────────────────────────┐
│                        Backend Layer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Slide      │  │   Project    │  │    User      │         │
│  │  Controller  │  │  Controller  │  │  Controller  │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Template   │  │    Theme     │  │   Export     │         │
│  │   Service    │  │   Service    │  │   Service    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Database   │  │  File Store  │  │    Cache     │         │
│  │   Manager    │  │   Manager    │  │   Manager    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
                         REST API
                              │
┌─────────────────────────────────────────────────────────────────┐
│                          AI Layer                               │
│  ┌─────────────────────────────────────────────────────┐       │
│  │              AI Abstraction Layer                   │       │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────┐   │       │
│  │  │  LLM Adapter │  │  Image Gen   │  │   TTS   │   │       │
│  │  │   Interface  │  │   Adapter    │  │ Adapter │   │       │
│  │  │              │  │  Interface   │  │Interface│   │       │
│  │  └──────────────┘  └──────────────┘  └─────────┘   │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Ollama     │  │  LM Studio   │  │ llama.cpp    │         │
│  │   Adapter    │  │   Adapter    │  │   Adapter    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  SD WebUI    │  │   ComfyUI    │  │   Custom     │         │
│  │   Adapter    │  │   Adapter    │  │   Adapter    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Web Speech   │  │  Coqui TTS   │  │    Piper     │         │
│  │   Adapter    │  │   Adapter    │  │   Adapter    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
                     HTTP/REST API
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      External AI Services                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Ollama     │  │  LM Studio   │  │ Text Gen UI  │         │
│  │   (Local)    │  │   (Local)    │  │   (Local)    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Stable      │  │   ComfyUI    │  │ Web Speech   │         │
│  │  Diffusion   │  │   (Local)    │  │     API      │         │
│  │  (Local)     │  │              │  │  (Browser)   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Coqui TTS   │  │  Piper TTS   │  │  Voicevox    │         │
│  │   (Local)    │  │   (Local)    │  │   (Local)    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

## レイヤー構成

### 1. Frontend Layer（プレゼンテーション層）

**責務**:
- ユーザーインターフェースの提供
- ユーザー入力の処理
- スライドの表示と編集
- バックエンドとの通信

**主要技術**:
- React 18+ (TypeScript)
- Fabric.js または Konva.js（キャンバスエディタ）
- TailwindCSS（スタイリング）
- Zustand（状態管理）
- React Query（API通信）

**モジュール構成**:
```
src/
├── components/           # UIコンポーネント
│   ├── editor/          # エディタ関連
│   │   ├── Canvas/      # キャンバスコンポーネント
│   │   ├── Toolbar/     # ツールバー
│   │   ├── Sidebar/     # サイドバー
│   │   └── Inspector/   # プロパティインスペクター
│   ├── slides/          # スライド表示
│   │   ├── SlideList/   # スライド一覧
│   │   ├── SlidePreview/# プレビュー
│   │   └── SlideView/   # スライドビュー
│   ├── templates/       # テンプレート関連
│   ├── themes/          # テーマ関連
│   └── common/          # 共通コンポーネント
├── hooks/               # カスタムフック
│   ├── useSlide.ts
│   ├── useTemplate.ts
│   ├── useTheme.ts
│   ├── useAI.ts
│   └── useExport.ts
├── stores/              # 状態管理
│   ├── slideStore.ts
│   ├── projectStore.ts
│   ├── templateStore.ts
│   ├── themeStore.ts
│   └── uiStore.ts
├── services/            # API通信
│   ├── api/
│   │   ├── slideApi.ts
│   │   ├── templateApi.ts
│   │   ├── themeApi.ts
│   │   └── aiApi.ts
│   └── websocket/
│       └── wsClient.ts
├── types/               # 型定義
│   ├── slide.ts
│   ├── template.ts
│   ├── theme.ts
│   └── common.ts
└── utils/               # ユーティリティ
    ├── canvas/          # キャンバス操作
    ├── layout/          # レイアウト計算
    └── helpers/         # ヘルパー関数
```

### 2. Backend Layer（アプリケーション層）

**責務**:
- ビジネスロジックの実装
- データの永続化
- AI層との連携
- ファイル管理

**主要技術**:
- Node.js + Express（または Python + FastAPI）
- TypeScript
- SQLite（開発）/ PostgreSQL（本番）
- WebSocket（リアルタイム通信）

**モジュール構成**:
```
server/
├── controllers/         # コントローラー
│   ├── slideController.ts
│   ├── projectController.ts
│   ├── templateController.ts
│   ├── themeController.ts
│   ├── exportController.ts
│   └── aiController.ts
├── services/            # サービス層
│   ├── slideService.ts
│   ├── projectService.ts
│   ├── templateService.ts
│   ├── themeService.ts
│   ├── exportService.ts
│   ├── layoutEngine.ts  # レイアウトエンジン
│   └── aiOrchestrator.ts# AI処理オーケストレーター
├── models/              # データモデル
│   ├── Slide.ts
│   ├── Project.ts
│   ├── Template.ts
│   ├── Theme.ts
│   └── User.ts
├── repositories/        # データアクセス層
│   ├── slideRepository.ts
│   ├── projectRepository.ts
│   ├── templateRepository.ts
│   └── themeRepository.ts
├── middleware/          # ミドルウェア
│   ├── auth.ts
│   ├── errorHandler.ts
│   ├── validator.ts
│   └── rateLimit.ts
├── routes/              # ルーティング
│   ├── slides.ts
│   ├── projects.ts
│   ├── templates.ts
│   ├── themes.ts
│   ├── export.ts
│   └── ai.ts
├── db/                  # データベース
│   ├── migrations/
│   ├── seeds/
│   └── connection.ts
└── utils/               # ユーティリティ
    ├── fileManager.ts
    ├── cacheManager.ts
    └── logger.ts
```

### 3. AI Layer（AI統合層）

**責務**:
- AI APIとの通信
- プロンプト生成と管理
- レスポンスのパース
- エラーハンドリングとリトライ

**主要技術**:
- Node.js / Python
- Axios（HTTP通信）
- プロンプトテンプレート管理

**モジュール構成**:
```
ai-layer/
├── adapters/            # AIアダプター
│   ├── base/
│   │   ├── LLMAdapter.ts      # LLM基底クラス
│   │   └── ImageGenAdapter.ts # 画像生成基底クラス
│   ├── llm/
│   │   ├── OllamaAdapter.ts
│   │   ├── LMStudioAdapter.ts
│   │   ├── LlamaCppAdapter.ts
│   │   └── TextGenWebUIAdapter.ts
│   └── image/
│       ├── StableDiffusionAdapter.ts
│       ├── ComfyUIAdapter.ts
│       └── CustomAdapter.ts
├── services/            # AIサービス
│   ├── contentGenerator.ts    # コンテンツ生成
│   ├── imageGenerator.ts      # 画像生成
│   ├── layoutSuggester.ts     # レイアウト提案
│   └── styleAnalyzer.ts       # スタイル分析
├── prompts/             # プロンプトテンプレート
│   ├── slideGeneration/
│   ├── contentImprovement/
│   └── imageGeneration/
├── types/               # 型定義
│   ├── llm.ts
│   └── imageGen.ts
└── utils/               # ユーティリティ
    ├── promptBuilder.ts
    ├── responseParser.ts
    └── retryHandler.ts
```

## コンポーネント設計

### 主要コンポーネント

#### 1. Slide Editor Canvas
**役割**: スライドの編集キャンバス

**機能**:
- 要素の配置、移動、リサイズ
- テキスト編集
- 画像配置
- レイヤー管理
- グリッドとガイド

**技術選定**: Fabric.js vs Konva.js

| 観点 | Fabric.js | Konva.js |
|------|-----------|----------|
| パフォーマンス | 良好 | より高速 |
| React統合 | react-fabricjs | react-konva（公式） |
| 機能性 | 豊富 | 豊富 |
| SVGサポート | 優秀 | 限定的 |
| ドキュメント | 充実 | 充実 |

**推奨**: Konva.js（Reactとの統合が優れ、パフォーマンスも良好）

#### 2. Template Engine
**役割**: スライドレイアウトの管理と適用

**機能**:
- テンプレートの読み込み
- レイアウトの自動選択
- 要素の配置計算
- レスポンシブ調整

**レイアウトアルゴリズム**:
```typescript
interface LayoutEngine {
  // コンテンツに基づいて最適なレイアウトを選択
  selectLayout(content: SlideContent): LayoutTemplate;

  // レイアウトに要素を配置
  applyLayout(layout: LayoutTemplate, elements: Element[]): PlacedElements;

  // レイアウトの妥当性を検証
  validateLayout(layout: LayoutTemplate): ValidationResult;
}
```

#### 3. AI Content Generator
**役割**: AIを使用したコンテンツ生成

**機能**:
- スライド構成の生成
- テキストコンテンツの生成
- 画像プロンプトの生成
- コンテンツの改善提案

**処理フロー**:
```
1. ユーザー入力（トピック、要件）
2. プロンプト生成
3. LLM API呼び出し
4. レスポンスのパース
5. スライドデータへの変換
6. プレビュー表示
7. ユーザー確認・編集
```

#### 4. Theme Manager
**役割**: テーマの管理と適用

**機能**:
- カラースキームの適用
- フォント管理
- スタイルの一貫性維持
- カスタムテーマ作成

**テーマ構造**:
```typescript
interface Theme {
  id: string;
  name: string;
  colors: {
    primary: string;
    secondary: string;
    accent: string;
    background: string;
    text: string;
    textSecondary: string;
  };
  fonts: {
    heading: FontFamily;
    body: FontFamily;
    code: FontFamily;
  };
  spacing: SpacingScale;
  borderRadius: BorderRadiusScale;
  shadows: ShadowScale;
}
```

#### 5. Export Manager
**役割**: スライドのエクスポート

**機能**:
- PDF生成
- PPTX生成
- HTML生成
- 画像シーケンス生成

**エクスポートパイプライン**:
```
1. スライドデータの取得
2. フォーマット別の変換処理
3. 埋め込みリソースの処理
4. ファイル生成
5. ダウンロード/保存
```

## データフロー

### 1. スライド作成フロー（AI支援）

```
User Input → Frontend
               ↓
        Validate Input
               ↓
        API Request (POST /api/ai/generate-slides)
               ↓
        Backend Controller
               ↓
        AI Orchestrator
               ↓
        LLM Adapter → Ollama/LM Studio/etc.
               ↓
        Parse Response
               ↓
        Create Slide Objects
               ↓
        Save to Database
               ↓
        Return to Frontend
               ↓
        Display in Editor
```

### 2. 画像生成フロー

```
User Request → Frontend
                 ↓
          Generate Prompt (Optional: LLM支援)
                 ↓
          API Request (POST /api/ai/generate-image)
                 ↓
          Backend Controller
                 ↓
          Image Generator Service
                 ↓
          Image Gen Adapter → Stable Diffusion/ComfyUI
                 ↓
          Save Image File
                 ↓
          Create Image Record
                 ↓
          Return Image URL
                 ↓
          Display in Editor
```

### 3. エクスポートフロー

```
User Action → Frontend
                ↓
         Select Format
                ↓
         API Request (POST /api/export)
                ↓
         Backend Controller
                ↓
         Export Service
                ↓
      ┌──────────┴────────────┐
      ↓                       ↓
   PDF Export            PPTX Export
   (jsPDF)              (pptxgenjs)
      ↓                       ↓
   Generate File        Generate File
      ↓                       ↓
      └──────────┬────────────┘
                 ↓
         Return File/URL
                 ↓
         Download on Frontend
```

### 4. リアルタイム編集フロー（将来拡張）

```
User Edit → Frontend
              ↓
        Update Local State
              ↓
        WebSocket Send
              ↓
        Backend WebSocket Handler
              ↓
        Broadcast to Other Clients
              ↓
        Update Database (Debounced)
```

## 技術スタック詳細

### Frontend

#### React + TypeScript
- **バージョン**: React 18.2+, TypeScript 5.0+
- **理由**:
  - 型安全性
  - コンポーネント再利用性
  - 豊富なエコシステム

#### キャンバスライブラリ: Konva.js
- **バージョン**: 9.0+
- **理由**:
  - 優れたパフォーマンス
  - React統合（react-konva）
  - レイヤー管理
  - イベント処理

#### 状態管理: Zustand
- **バージョン**: 4.0+
- **理由**:
  - シンプルなAPI
  - 軽量
  - TypeScript対応
  - Redux DevTools対応

#### スタイリング: TailwindCSS
- **バージョン**: 3.3+
- **理由**:
  - ユーティリティファースト
  - カスタマイズ性
  - パフォーマンス

#### API通信: React Query
- **バージョン**: 5.0+
- **理由**:
  - キャッシング
  - 自動リフェッチ
  - 楽観的更新

### Backend

#### Option A: Node.js + Express
```typescript
// 推奨構成
{
  "runtime": "Node.js 20 LTS",
  "framework": "Express 4.18+",
  "language": "TypeScript 5.0+",
  "orm": "Prisma 5.0+" または "TypeORM 0.3+",
  "validation": "Zod 3.0+",
  "testing": "Jest + Supertest"
}
```

**利点**:
- JavaScriptエコシステムとの統合
- 開発者フレンドリー
- 豊富なライブラリ

#### Option B: Python + FastAPI
```python
# 推奨構成
{
  "runtime": "Python 3.11+",
  "framework": "FastAPI 0.104+",
  "orm": "SQLAlchemy 2.0+",
  "validation": "Pydantic 2.0+",
  "testing": "pytest + httpx"
}
```

**利点**:
- AIライブラリとの親和性
- 型ヒント
- 自動ドキュメント生成

**推奨**: Node.js + Express（フロントエンドとの統合を優先）

### Database

#### 開発環境: SQLite
- ファイルベース
- セットアップ不要
- 軽量

#### 本番環境: PostgreSQL
- ACID準拠
- JSON型サポート
- 拡張性

### AI Integration

#### LLM APIs
1. **Ollama** (推奨)
   - セットアップ簡単
   - 複数モデル対応
   - OpenAI互換API

2. **LM Studio**
   - GUI管理
   - モデル切り替え容易

3. **llama.cpp server**
   - 軽量
   - カスタマイズ性高

#### Image Generation APIs
1. **Stable Diffusion WebUI** (AUTOMATIC1111)
   - 豊富な機能
   - 拡張性
   - コミュニティサポート

2. **ComfyUI**
   - ノードベース
   - 高度なワークフロー
   - 効率的

## セキュリティ設計

### 1. データ保護
- **ローカルストレージ**: すべてのデータはローカルに保存
- **暗号化**: オプションでデータベース暗号化（SQLCipher）
- **アクセス制御**: ファイルシステムレベルの権限管理

### 2. API セキュリティ
- **認証**: JWT トークンベース（マルチユーザー時）
- **CORS**: 厳格なCORS設定
- **レート制限**: API呼び出しのレート制限

### 3. AI API通信
- **検証**: API URLの検証
- **タイムアウト**: 適切なタイムアウト設定
- **エラーハンドリング**: 安全なエラーハンドリング

## パフォーマンス設計

### 1. フロントエンド最適化
- **コード分割**: React.lazy + Suspense
- **画像最適化**: 遅延読み込み、WebP形式
- **仮想化**: 大量スライドの仮想スクロール
- **メモ化**: useMemo, useCallback の適切な使用

### 2. バックエンド最適化
- **キャッシング**: Redis（オプション）またはメモリキャッシュ
- **データベースインデックス**: 適切なインデックス設定
- **非同期処理**: AI処理の非同期化
- **コネクションプーリング**: データベース接続プール

### 3. AI処理最適化
- **バッチ処理**: 複数リクエストのバッチ化
- **キャッシング**: プロンプトとレスポンスのキャッシング
- **並列処理**: 独立した処理の並列実行
- **タイムアウト管理**: 適切なタイムアウトとリトライ

### 4. メモリ管理
- **画像圧縮**: 適切な解像度とフォーマット
- **ガベージコレクション**: メモリリーク対策
- **リソース解放**: 未使用リソースの適切な解放

## スケーラビリティ

### 垂直スケーリング（現行設計）
- より強力なマシンでの実行
- メモリ増設
- GPU活用（画像生成）

### 水平スケーリング（将来拡張）
- マルチユーザー対応
- クラウド展開オプション
- 分散AI処理

## 監視とログ

### ログ設計
```typescript
interface LogEntry {
  timestamp: Date;
  level: 'debug' | 'info' | 'warn' | 'error';
  component: string;
  message: string;
  metadata?: Record<string, any>;
}
```

### メトリクス
- API レスポンスタイム
- AI処理時間
- エラー率
- メモリ使用量

## テスト戦略

### ユニットテスト
- すべてのサービス層
- ユーティリティ関数
- カバレッジ目標: 80%+

### 統合テスト
- API エンドポイント
- データベース操作
- AI統合

### E2Eテスト
- 主要ユーザーフロー
- クリティカルパス
- ツール: Playwright

## デプロイメント

### ローカル実行
```bash
# 開発モード
npm run dev

# 本番ビルド
npm run build

# 本番実行
npm start
```

### パッケージング
- Electron（デスクトップアプリ化）
- Docker（コンテナ化）

## まとめ

このアーキテクチャは以下の原則に基づいて設計されています：

1. **関心の分離**: 各層が明確な責務を持つ
2. **拡張性**: プラグインアーキテクチャで機能追加が容易
3. **保守性**: モジュラー設計で保守が容易
4. **パフォーマンス**: 最適化ポイントが明確
5. **セキュリティ**: ローカル動作でプライバシーを保護

---

**最終更新**: 2025-10-24
**レビュー**: 未実施
